# Mooncake + KVClient L3 接入设计方案

版本：v3.0　更新日期：2026-11-10

---

# 1 概述

## 1.1 需求分析

### 1.1.1 背景

L3 当前的 `DistributedStorageBackend` 已支持 `hf3fs` / `posix` / `oss` / `s3` / `kvcs-lowlevel` 五种 `fs_adapter_type`。其中对象存储类后端（oss / s3 / kvcs-lowlevel）已完全适配器抽象化：数据面只依赖 `UsesObjectStorage()`，通过 `ObjectStorageAdapter` 接口按逻辑 key 读写，Master 只维护「key → 逻辑 DFS 副本」的目录，不做字节级 offset 分配。

现需将**远端 KV 存储**（经 KVClient SDK，Host RDMA 直连）接入为共享 L3。

### 1.1.2 与既有 KVCS 后端的差异

KVClient 与 KVCS 同为「网络可寻址 KV」，但底层能力差异决定了不能直接照搬：

| 维度 | KVCS（`kvcs-lowlevel`） | KVClient |
| --- | --- | --- |
| 传输 | gRPC over unix socket → 本机 EFC 代理 | Host RDMA 直连远端存储 |
| key | string，hex 编码 ≤256B，**单射无碰撞** | **8 字节定长** `CacheKey`，必须 hash |
| 内存注册 | 无此概念 | 每次 I/O 需已注册 region + `MRHandle` |
| 对象上限 | 4 MiB 可配 | **16,776,704 B** 硬限（24-bit length + 512 对齐） |
| 对齐 | 无要求 | 线格式 length 须 **512 B** 对齐；N 本身可任意，由适配器补零 |
| 批量上限 | 8，可配 | 写/读 110、删 254、查 256 |
| 容量查询 | `kvcs_ll_mp_stat` → 驱动水位淘汰 | **无接口** |

### 1.1.3 注意事项

| 内容 | 落地位置 |
| --- | --- |
| key 映射：XXH64 单 seed → 8B `CacheKey`，value 就是原始 N 字节。`EncodeKey` 作为 `KvClientObjectStorageAdapter` 的私有方法 | §3.1（§3.5） |
| 只增加 `UsesKvClient()`，与 `UsesKvcs()` **平级**；不引入 `UsesManagedKvStorage()`。Master 侧因此需要 `use_kvcs_` / `use_kvclient_` 两个独立 flag | §3.4（§3.6、§3.7） |
| 512 B 对齐发生在**传入 KvClient 时**：`object_size = N`，`aligned_size = AlignUp(N, 512)`，适配器在 staging slot 内把尾部 `[N, aligned_size)` 补零后提交。读回 `aligned_size` 字节，只把前 N 字节交还调用者，`GetBatch` 返回 **N 而非 `aligned_size`**。**上限判据是 `AlignUp(N,512) <= 16,776,704`，即 `N <= 16,776,704`，须在 `PrepareWriteStart` 显式校验**（Master 的 `kMaxSliceSize` 是 16,777,200，会放行越界值，或判断为KvClient，重置kMaxSliceSize） | §3.5（§3.2、§3.3、§3.6） |
| 两副本（memory + ASU）硬逻辑：ASU 写失败 → 整个 Put 失败 + memory 副本一起 revoke。**零代码改动**，`dfs > 0` 时现有 `RELIABLE_MULTI_REPLICA` 即是该语义； | §3.6 |
| **不支持**从 snapshot / OpLog 恢复对象目录。| §3.6 |
| `ReconcileMissingKvcsReplica()` 推广为 `ReconcileMissingKvReplica()`； | §3.7（§3.6） |
| 删除顺序：**先删 value，再删元数据**。连带必改 `RemoveAll` / `RemoveByRegex` 门控，且依赖 miss 自愈兜底中间态 | §3.7 |

## 1.2 用户使用场景分析

### 1.2.1 支持的场景

| 编号 | 场景 | 副本配置 | 说明 |
| --- | --- | --- | --- |
| S1 | 两副本写 + 跨 Client 读 L3 | `replica_num=1, nof=0, dfs=1` | 主场景。Put 同步写 memory + L3；memory 副本淘汰后，其他 Client 经 Master 目录读 L3 |
| S2 | DFS-only 写读 | `replica_num=0, nof=0, dfs=1` | 纯 L3 存储，不占内存池 |
| S3 | 显式删除 | — | `Remove` / `BatchRemove`，先删 value 再删元数据 |
| S4 | 整值替换 | — | `Upsert`，复用 Master 现有状态机；同尺寸走原地复用分支 |
| S5 | 读 miss 自愈 | — | 存储侧返回明确 `NOT_FOUND` 时，按 ReplicaID 条件清理失效的 DFS 记录 |
| S6 | 批量读写删 | — | `BatchPut` / `BatchGet` / `BatchRemove`，逐 key 结果与原下标严格对应 |

### 1.2.2 明确不支持的场景

以下能力在 KVClient 后端下返回 `NOT_SUPPORTED` 或启动期拒绝：

| 场景 | 原因 | 行为 |
| --- | --- | --- |
| 范围读 / 前缀读 | `BatchRead` 对象分支忽略 `range_read`，slice 容量 < object_size 即拒绝 | `INVALID_PARAMS`；上层需回退整对象读后截取 |
| key 枚举 | SDK 无 List 接口，`QueryResult` 只有 `exists` 位图 | `ListKeys` → `NOT_SUPPORTED` |
| `RemoveAll` / `RemoveByRegex` | 依赖 key 枚举；只删元数据会泄漏全部 L3 value | `NOT_SUPPORTED`，**元数据不删** |
| 元数据恢复 | 不支持恢复裁决 | `enable_oplog=true` 时 Master 启动 `LOG(FATAL)` |
| 容量水位淘汰 | SDK 无容量查询接口 | Master 侧降级：仅 WARNING，禁用淘汰 |
| N > 16,776,704 | SDK 线格式 24-bit length + 512 对齐；Master 的 `kMaxSliceSize`（16,777,200）更宽，会放行越界值 | `PrepareWriteStart` 返回 `INVALID_PARAMS`，不触达适配器 |

---

# 2 设计描述

## 2.1 设计

### 2.1.1 逻辑模型

KVClient 后端位于 Mooncake 数据面的最南端。新增的 `KvClientObjectStorageAdapter` 与既有 `KvcsObjectStorageAdapter` **平级**，同实现 `ObjectStorageAdapter` 接口，由工厂按 `fs_adapter_type` 二选一创建。

实线表示应用调用与存储 I/O，虚线表示元数据交互。灰色模块沿用现有职责，黄色模块需要调整，绿色模块为新增。

```mermaid
flowchart TB
    subgraph Request["请求端进程"]
        A["应用<br/>已聚合 CPU buffer / 连续 CPU slices"]
        R["RealClient<br/>应用 API、buffer 组织"]
        C["Client<br/>副本读写、逐 key 收尾<br/>value-first 删除、miss 清理"]
        RPC["MasterClient<br/>元数据 RPC"]
        TE["TransferWrite / TransferRead<br/>Transfer Engine / memcpy"]
        B["DistributedStorageBackend<br/>BatchWrite / BatchRead / BatchDeleteObjects<br/>长度校验"]
        O["ObjectStorageAdapter<br/>按逻辑 key 读写"]
        K["KvClientObjectStorageAdapter<br/>key 编码、staging、提交与完成等待、结果转换"]
        SDK["KVClient<br/>CacheKey 路由、Host RDMA"]

        A <-->|"Put / Get / Remove"| R
        R <--> C
        C -.-> RPC
        C <-->|"MEMORY 副本"| TE
        C <-->|"读写：key + 描述符 + CPU slices<br/>删除：key"| B
        B <-->|"PutBatch / GetBatch / DeleteBatch"| O
        O <--> K
        K <-->|"内存注册<br/>BatchStore / BatchLoad / Delete 与完成等待"| SDK
    end

    subgraph Master["Master 进程（不接 SDK）"]
        M["MasterService<br/>申请、发布、查询、元数据删除<br/>use_kvclient_ 分支"]
        META["对象目录<br/>key、长度、ReplicaID、状态、读取租约"]

        M -.->|"查询与更新"| META
    end

    RAM["Mooncake 内存池<br/>L2：可位于本机或远端"]
    S["远端 KV 存储<br/>L3：CacheKey 到 value 的定位、物理空间、内部冗余"]

    RPC -.->|"PutStart / PutEnd / PutRevoke<br/>GetReplicaList / Remove<br/>EvictDiskReplicaIfCurrent"| M
    M -.->|"副本描述符、租约、操作结果"| RPC
    TE <-->|"数据传输"| RAM
    SDK <-->|"KV 协议与 value 数据"| S

    classDef existing fill:#f3f4f6,stroke:#6b7280,color:#111827;
    classDef changed fill:#fff4cc,stroke:#b58105,color:#111827;
    classDef added fill:#e3f5e8,stroke:#28864b,color:#111827;
    class A,R,RPC,TE,B,O,META,RAM,S existing;
    class C,SDK,M changed;
    class K added;
```

**分层职责**：

| 层 | 职责 | 本方案改动 |
| --- | --- | --- |
| `Client` | 副本编排、逐 key 收尾、删除路由、miss 清理 | 删除顺序改 value-first；miss 清理门控放宽 |
| `DistributedStorageBackend` | slices → 对象请求转换、长度校验 | **零改动**（`UsesObjectStorage()` 已抽象化） |
| `ObjectStorageAdapter` | 对象存储统一契约 | **零改动**（接口不变） |
| `KvcsObjectStorageAdapter` | KVCS 后端实现 | **零改动**（与 KVClient 平级，互不影响） |
| `KvClientObjectStorageAdapter` | KVClient 后端实现：key 编码、staging、提交与完成等待、结果转换 | **新增** |

### 2.1.2 Client 侧类图

黄色表示修改现有类，绿色表示新增类，灰色表示复用接口或数据结构。类图列出相关成员，参数和返回类型作了简化；新增方法使用设计名称。

```mermaid
classDiagram
    direction TB

    class DistributedStorageConfig["DistributedStorageConfig（修改）"] {
        +fs_adapter_type
        +object_storage_config_path
        +UsesObjectStorage()
        +UsesKvcs()
        +UsesKvClient()
        +Validate()
    }
    class FileStorage["FileStorage（修改）"] {
        -storage_backend_
        -client_
        +FileStorage(config, client)
        +Init()
    }
    class Client["Client（修改）"] {
        -dfs_storage_backend_
        +SetDfsStorageBackend(backend)
        +Put(key, slices, config)
        +Get(key, slices)
        +Remove(key)
        +BatchRemove(keys, force)
        -ReconcileMissingKvReplica(key, query, replica)
    }
    class DistributedStorageBackend["DistributedStorageBackend（复用）"] {
        -object_storage_adapter_
        -distributed_config_
        +BatchWrite(requests)
        +BatchRead(requests)
        +BatchDeleteObjects(keys)
        +UsesObjectStorage()
        +UsesKvClient()
    }
    class ObjectStorageAdapter["ObjectStorageAdapter（复用）"] {
        +PutV(key, iov, iovcnt)
        +PutBatch(requests)
        +GetBatch(requests)
        +DeleteBatch(keys)
        +Exists(key)
        +ListKeys()
        +GetRange(key, buffer, len, offset)
        +Init()
        +GetName()
    }
   
    class KvClientObjectStorageAdapter["KvClientObjectStorageAdapter（新增）"] {
        -kv_client_
        +Init()
        +PutV(key, iov, iovcnt)
        +PutBatch(requests)
        +GetBatch(requests)
        +DeleteBatch(keys)
        -RegisterBuffers(buffers)
        -WaitForCompletion(task_id)
        -ReleaseRegistrations(handles)
    }

    class SdkKvClient["kv::KvClient"] {
        +Init(configPath)
        +BatchStoreAsync(entries, task_id)
        +BatchLoadAsync(entries, task_id)
        +DeleteAsync(keys, task_id)
        +QueryAsync(keys, task_id)
        +Wait(task_id, timeout, result)
    }

    FileStorage o-- DistributedStorageBackend : "共享持有后端"
    FileStorage --> Client : "注入 DFS 后端"
    Client o-- DistributedStorageBackend : "共享持有"
    DistributedStorageBackend *-- DistributedStorageConfig : "保存配置"
    DistributedStorageBackend *-- ObjectStorageAdapter : "持有对象适配器"
    ObjectStorageAdapter <|-- KvClientObjectStorageAdapter
    KvClientObjectStorageAdapter *-- SdkKvClient : "持有 SDK 实例"
    DistributedStorageConfig ..> KvClientObjectStorageAdapter : "工厂按配置创建"

    style DistributedStorageConfig fill:#fff4cc,stroke:#b58105,color:#111827
    style FileStorage fill:#fff4cc,stroke:#b58105,color:#111827
    style Client fill:#fff4cc,stroke:#b58105,color:#111827
    style KvClientObjectStorageAdapter fill:#e3f5e8,stroke:#28864b,color:#111827
    style StagingArena fill:#e3f5e8,stroke:#28864b,color:#111827
    style DistributedStorageBackend fill:#f3f4f6,stroke:#6b7280,color:#111827
    style ObjectStorageAdapter fill:#f3f4f6,stroke:#6b7280,color:#111827
    style SdkKvClient fill:#f3f4f6,stroke:#6b7280,color:#111827
```

| 类或函数 | 接口与改动说明 |
| --- | --- |
| `DistributedStorageConfig` | `UsesObjectStorage()` 列表加入 `"kvclient"`；新增 `UsesKvClient()`，与 `UsesKvcs()` 平级；`Validate()` 增加 KVClient 专属分支 |
| `CreateStorageBackend(config)` | 修改现有工厂函数，在 distributed 分支新增 `UsesKvClient()` case 创建新适配器 |
| `FileStorage` | 修改构造时的 DFS 启用判断，将 KVClient 后端注入 Client；沿用现有初始化入口 |
| `Client` | `ReconcileMissingKvcsReplica()` 推广为图中的 `ReconcileMissingKvReplica()` |
| `DistributedStorageBackend` | **零改动**。读写路径已由 `UsesObjectStorage()` 抽象化，仅新增 `UsesKvClient()` 组合判断供 Client 侧门控使用 |
| `ObjectStorageAdapter` | **零改动**。同步契约（payload 须在返回前保持有效）是 §2.3.2 staging 设计的来源 |
| `kv::KvClient` | **SDK 不改**。只用批量接口 `BatchStoreAsync` / `BatchLoadAsync` / `DeleteAsync` / `QueryAsync`，单条 `StoreAsync` / `LoadAsync` 在线格式层无批量能力（`io_scheduler.cc:82`） |

### 2.1.3 Master 侧类图

Master 在现有 `ObjectMetadata` 和 `Replica` 上管理 KVClient 的逻辑 DFS 副本。`PutStart` 创建描述符，`PutEnd` 发布副本，查询返回可读副本，Remove 和明确 miss 清理更新对象目录。Master **不接 SDK**，不创建 driver，也不建 capacity provider。

```mermaid
classDiagram
    direction TB

    class MasterService["MasterService（修改）"] {
        -use_kvcs_
        -use_kvclient_
        -enable_dfs_
        +PutStart(...)
        +PutEnd(...)
        +PutRevoke(...)
        +GetReplicaList(...)
        +Remove(...)
        +BatchRemove(...)
        +EvictDiskReplicaIfCurrent(...)
        -InitDfsAllocatorFromEnvironment(config)
        -PrepareWriteStart(...)
        -AllocateReplicas(...)
        -EvictDiskReplicaImpl(...)
    }
    class DistributedStorageConfig["DistributedStorageConfig（修改）"] {
        +UsesKvcs()
        +UsesKvClient()
        +FromEnvironment()
    }
    class ObjectMetadata["ObjectMetadata（复用）"] {
        +user_key
        +client_id
        +VisitReplicas(...)
        +GrantReadLease(...)
        +IsLeaseExpired(now)
    }
    class Replica["Replica（复用）"] {
        +id()
        +status()
        +is_dfs_replica()
        +get_dfs_descriptor()
    }
    class ReplicaDescriptor["Replica::Descriptor（复用）"] {
        +id
        +status
        +get_dfs_descriptor()
    }
    class DistributedFSDescriptor["DistributedFSDescriptor（复用）"] {
        +file_path
        +offset
        +object_size
        +aligned_size
        +shard_idx
    }

    MasterService ..> DistributedStorageConfig : "配置与后端选择"
    MasterService --> ObjectMetadata : "查询与更新对象目录"
    ObjectMetadata *-- Replica : "保存各类副本"
    MasterService ..> ReplicaDescriptor : "返回副本描述符"
    Replica ..> ReplicaDescriptor : "生成描述符"
    ReplicaDescriptor *-- DistributedFSDescriptor : "DFS 类型的描述信息"

    style MasterService fill:#fff4cc,stroke:#b58105,color:#111827
    style DistributedStorageConfig fill:#fff4cc,stroke:#b58105,color:#111827
    style ObjectMetadata fill:#f3f4f6,stroke:#6b7280,color:#111827
    style Replica fill:#f3f4f6,stroke:#6b7280,color:#111827
    style ReplicaDescriptor fill:#f3f4f6,stroke:#6b7280,color:#111827
    style DistributedFSDescriptor fill:#f3f4f6,stroke:#6b7280,color:#111827
```

Master 查询自身对象目录，Client 根据返回的描述符调用 driver。存储返回明确 miss 时，由 Client 请求 `EvictDiskReplicaIfCurrent` 条件清理。

| 类 | 改动与职责 |
| --- | --- |
| `MasterService` | 新增私有 `use_kvclient_`（与 `use_kvcs_` 平级）。改动 ：`enable_dfs_` 判定、初始化新增分支、descriptor 铸造、`PrepareWriteStart` 门与 N 上限校验 |
| `DistributedStorageConfig` | 解析 KVClient 配置，提供与 `UsesKvcs()` 平级的能力判断 |
| `ObjectMetadata` / `Replica` | 保存 key、写入 Client、副本身份、状态和读取租约，沿用既有元数据持久化方式 |
| `Replica::Descriptor` / `DistributedFSDescriptor` | 向 Client 返回对象长度、ReplicaID 和状态。KVClient 模式下 `file_path` 为空、`offset` 为 0、`shard_idx` 为 -1，`object_size = N`、`aligned_size = AlignUp(N, 512)` |



## 2.2 业务交互流程

### 2.2.1 写入流程

下图以新 key 的 Put 为例，按申请、数据写入和提交顺序展开。Master 的申请与收尾分别绘制，元数据 RPC 经 `MasterClient` 发送，数据由请求端访问内存池和远端存储。

```mermaid
flowchart TB
    C1["① 请求端 Client::Put<br/>key、CPU slices、replica config"]

    subgraph MasterStart["Master 进程：申请阶段"]
        START["② MasterService::PutStart / PrepareWriteStart<br/>校验 slice_length<=16776704"]
        ALLOC["分配 MEMORY 副本<br/>逻辑 DFS 描述符<br/>object_size=N, aligned_size=AlignUp(N,512), shard_idx=-1"]
        META["记录写入 Client、对象长度与 ReplicaID<br/>副本状态为 PROCESSING"]

        START --> ALLOC
        ALLOC --> META
    end

    subgraph Request["请求端进程：数据写入与结果汇总"]
        C2["③ Client<br/>接收 PROCESSING 副本描述符"]
        T["④ TransferWrite<br/>完成 MEMORY 传输"]
        GATE{"⑤ NonDfsTransfersSucceeded?"}
        W["⑥ WriteDfsReplicas<br/>构造 key + 描述符 + CPU slices"]
        B["DistributedStorageBackend::BatchWrite<br/>校验总长度等于 object_size"]
        O["KvClientObjectStorageAdapter::PutBatch<br/>EncodeKey + 补零到 aligned_size"]
        D["⑦ staging Acquire → memcpy<br/>BatchStoreAsync → Wait → Release/Quarantine"]
        J{"⑧ Client 逐 key 汇总<br/>AllAllocatedTransfersSucceeded?<br/>memory 与 dfs 均成功且零失败"}
        END["⑨ MasterClient::PutEnd<br/>key、对象校验和、ALL"]
        REVOKE["⑨ MasterClient::PutRevoke<br/>key、ALL"]

        C2 --> T
        T --> GATE
        GATE -->|"是"| W
        GATE -->|"否：跳过 DFS 写，ASU 零调用"| J
        W --> B
        B -->|"校验通过"| O
        B -->|"参数错误"| J
        O --> D
        D -->|"逐 key 数据结果经后端返回"| J
        J -->|"成功"| END
        J -->|"失败"| REVOKE
    end

    subgraph MasterFinish["Master 进程：提交或撤销"]
        COMMIT["⑩ MasterService::PutEnd<br/>核对写入者与副本状态<br/>发布 COMPLETE，更新访问记录"]
        ABORT["⑩ MasterService::PutRevoke<br/>核对写入 Client 与状态<br/>移除本轮 PROCESSING 副本（含 MEMORY）"]
    end

    OK["⑪ 请求端返回 Put 成功"]
    FAIL["⑪ 请求端返回该 key 的失败结果"]
    REJECT["请求端返回申请错误"]
    RAM["MEMORY 副本<br/>Mooncake 内存池 L1"]
    S["KVClient → 远端 KV 存储 L3<br/>按 CacheKey 写入完整 value"]

    C1 -->|"MasterClient RPC：PutStart<br/>client_id、key、总长度、配置"| START
    START -->|"申请失败"| REJECT
    META -->|"ReplicaID、PROCESSING、对象长度<br/>及 MEMORY 副本描述符"| C2
    T <-->|"写入并等待传输结果"| RAM
    D <-->|"注册内存与操作结果"| S
    END --> COMMIT
    COMMIT -->|"提交成功"| OK
    COMMIT -->|"提交错误或响应未知"| FAIL
    REVOKE --> ABORT
    ABORT -->|"撤销结果"| FAIL
```

**关键判定**（两副本硬逻辑裁决，零代码改动）：`DetermineReplicaWriteMode` 在 `dfs > 0` 时返回 `RELIABLE_MULTI_REPLICA`，故 `AllAllocatedTransfersSucceeded` 要求 memory 与 dfs **全部成功且零失败**，否则 `PutRevoke(ALL)`。

PutEnd 响应未知时，Client 返回提交错误；后续通过 Master 查询或符合条件的重复提交确认发布状态。

### 2.2.2 读取流程（S1 / S5）

下图展开完整对象 Get 选中 DFS 副本后的数据路径，Master 按查询阶段和 miss 清理阶段分别绘制。内存副本淘汰后，Master 仍保留已发布的 DFS 记录，读取 Client 按相同流程访问 L3。

```mermaid
flowchart TB
    Q["① 请求端 Client::Get → Query<br/>记录本地查询起始时间"]

    subgraph MasterQuery["Master 进程：查询阶段"]
        GET["② MasterService::GetReplicaList<br/>查找对象、筛选 COMPLETE 副本"]
        LEASE["GrantReadLease<br/>更新对象读取租约"]

        GET -->|"存在可读副本"| LEASE
    end

    subgraph Request["请求端进程：读取与校验"]
        QR["③ QueryResult<br/>可读副本、租约截止时间、可选校验和"]
        SELECT["④ Client 按入口策略选择副本<br/>本图选中 COMPLETE DFS"]
        READ["⑤ ReadDfsReplica<br/>key + DFS 描述符 + CPU 目标 slices"]
        B["DistributedStorageBackend::BatchRead<br/>size = desc.object_size<br/>校验 slice 容量 >= object_size"]
        D["⑥ KvClientObjectStorageAdapter::GetBatch<br/>staging Acquire → BatchLoadAsync → Wait<br/>→ memcpy 前 N 字节回原目标 buffer"]
        VERIFY["⑦ 确认完整读取 object_size 字节<br/>数据已写入原目标 buffer"]
        CHECK["⑧ Client 检查结果<br/>对象校验和、IsLeaseExpired"]
        MISS["⑨ 明确 NotFound<br/>携带本次观察到的 ReplicaID<br/>请求条件清理 DFS 记录"]

        QR --> SELECT
        SELECT --> READ
        READ --> B
        B -->|"logical_key + 原 buffer + object_size"| D
        D -->|"读取完成"| VERIFY
        D -->|"对象明确不存在"| MISS
        VERIFY -->|"完整读取成功"| CHECK
    end

    subgraph MasterCleanup["Master 进程：miss 清理"]
        EVICT["⑩ EvictDiskReplicaIfCurrent<br/>核对当前 DFS ReplicaID<br/>匹配时移除失效记录"]
    end

    RETURN["⑪ 请求端返回成功或错误<br/>租约到期返回 LEASE_EXPIRED"]
    ERROR["请求端返回 I/O 或校验错误<br/>元数据保留"]
    QUERY_FAIL["请求端返回查询错误"]
    MISS_RETURN["请求端返回本次读取 miss"]
    S["KVClient → 远端 KV 存储 L3<br/>按 CacheKey 读取 value"]

    Q -->|"MasterClient RPC：GetReplicaList(key)"| GET
    GET -->|"OBJECT_NOT_FOUND / REPLICA_IS_NOT_READY 等"| QUERY_FAIL
    LEASE -->|"描述符、lease_ttl_ms、可选校验和"| QR
    D <-->|"Load 到原目标 buffer，返回逐项结果"| S
    MISS -->|"MasterClient：条件清理请求"| EVICT
    EVICT --> MISS_RETURN
    CHECK --> RETURN
    D -->|"超时、连接或 I/O 错误"| ERROR
    B -->|"容量不足或参数错误"| ERROR
    VERIFY -->|"读取长度不符"| ERROR
```


普通 `Client::Query` 使用本地查询起始时间加 `lease_ttl_ms` 计算租约截止时间，完整对象 Get 在读取结束后检查有效期。

### 2.2.3 删除流程（S3，value-first）

删除顺序与 KVCS 相反：Client 先调用 KVClient 删除 value，再对删除成功的 key 请求 Master 移除对象元数据。

```mermaid
flowchart TB
    C["① Client::Remove / BatchRemove<br/>key 列表与删除参数"]

    subgraph Request["请求端进程：先删 value"]
        B["② DistributedStorageBackend<br/>BatchDeleteObjects(全部 keys)"]
        D["③ KvClientObjectStorageAdapter::DeleteBatch<br/>EncodeKey → DeleteAsync → Wait"]
        PICK["④ 选出 value 删除成功的项<br/>（含幂等 NOT_FOUND）"]

        B --> D
        D --> PICK
    end

    subgraph Master["Master 进程：后删元数据"]
        M["⑤ MasterService::BatchRemove<br/>检查对象状态与读取保护"]
        META["⑥ 移除对象元数据"]

        M -->|"允许删除"| META
    end

    RESULT["⑧ 回填原下标，汇总逐 key 结果"]
    RETURN["⑨ 向应用返回删除结果"]
    WINDOW["⑦ 元数据保留但 value 已失<br/>⚠️ 后续读取触发 miss 自愈"]
    S["远端 KV 存储 L3<br/>按 CacheKey 删除 value"]

    C -->|"key 列表"| B
    D <-->|"删除请求与结果"| S
    PICK -->|"成功项 metadata_keys"| M
    PICK -->|"失败项保留各自错误<br/>value 与元数据均保留"| RESULT
    M -->|"请求被拒绝或元数据错误"| WINDOW
    META -->|"元数据删除成功"| RESULT
    WINDOW --> RESULT
    RESULT --> RETURN
```

**顺序理由**：KVClient 的三条孤儿回收通道全断，metadata-first 的孤儿是**永久且无界**的。

**中间态自愈**：value 已删、元数据尚在 → 读取返回 `NOT_FOUND` → `ReconcileMissingKvReplica` → `EvictDiskReplicaIfCurrent` 按 ReplicaID 条件清理。**value-first 删除依赖 miss 清理同时落地**，否则中间态无人清理。

**失败态可安全重试**：value 删除失败的 key **不进入**元数据删除集合，故 value 与元数据同时保留。这是 value-first 相对 metadata-first 的核心优势。

### 2.2.4 Upsert 流程（S4）

Upsert 使用 `UpsertStart / UpsertEnd / UpsertRevoke` 管理本次更新。Master 按现有状态机处理对象可见性，Client 通过同一适配器写入新 value；内存与 DFS 的结果按 key 汇总，处理方式与 Put 一致。

```mermaid
flowchart TB
    C1["① 请求端 Client::Upsert<br/>key、CPU slices、replica config"]

    subgraph MasterStart["Master 进程：申请阶段"]
        START["② MasterService::UpsertStart<br/>PrepareWriteStart 校验（同 Put）"]
        BRANCH{"③ 对象是否已存在<br/>且尺寸相同？"}
        REUSE["原地复用分支<br/>旧 COMPLETED 副本标为 PROCESSING<br/>原样返回旧描述符、不重新分配"]
        ALLOC["分配新副本<br/>铸造新逻辑 DFS 描述符"]
        META["记录写入 Client 与 ReplicaID<br/>副本状态为 PROCESSING"]

        START --> BRANCH
        BRANCH -->|"是（bucket_allocator_ 为 null）"| REUSE
        BRANCH -->|"否"| ALLOC
        REUSE --> META
        ALLOC --> META
    end

    subgraph Request["请求端进程：数据写入与收尾"]
        C2["④ Client 接收 PROCESSING 描述符"]
        W["⑤ 写入 MEMORY 与 DFS<br/>路径同 §2.2.1"]
        J{"⑥ HasExpectedReplicaAllocation<br/>&& AllAllocatedTransfersSucceeded?"}
        END["⑦ MasterClient::UpsertEnd"]
        REVOKE["⑦ MasterClient::UpsertRevoke"]

        C2 --> W
        W -->|"所需数据写入均成功"| J
        J -->|"成功"| END
        J -->|"失败"| REVOKE
    end

    subgraph MasterFinish["Master 进程：提交或撤销"]
        COMMIT["⑧ MasterService::UpsertEnd<br/>发布 COMPLETE"]
        ABORT["⑧ MasterService::UpsertRevoke<br/>回滚本轮更新"]
    end

    OK["⑨ 请求端返回 Upsert 成功"]
    FAIL["⑨ 请求端返回该 key 的失败结果"]
    TOPO["请求端返回拓扑不符<br/>仅 MEMORY 对象无法补 DFS 副本"]

    C1 -->|"MasterClient RPC：UpsertStart"| START
    BRANCH -->|"已有对象无 DFS 且请求 dfs=1"| TOPO
    META -->|"描述符、PROCESSING"| C2
    END --> COMMIT
    COMMIT --> OK
    REVOKE --> ABORT
    ABORT --> FAIL
```

 **仅 MEMORY 的对象无法经 Upsert 补 DFS 副本**，需先 Remove 再 Put。

## 2.3 运行设计

本节按阶段展开适配器内部的时序。各阶段的判定依据与代码位置见 §3 的 shard 设计。

### 2.3.1 初始化阶段

初始化在 `CreateStorageBackend` 工厂内创建适配器，随后由后端 `Init()` 触发 SDK 连接与 staging region 注册。**注册是一次性的**，整个适配器生命周期只注册一个大 region（v1 的「按次注册、按次释放」不可行，理由见 §2.3.2）。

```mermaid
sequenceDiagram
    autonumber
    participant RC as RealClient
    participant FS as FileStorage
    participant F as CreateStorageBackend
    participant BE as DistributedStorageBackend
    participant AD as KvClientObjectStorageAdapter
    participant AR as StagingArena
    participant SDK as kv::KvClient

    RC->>FS: 构造（UsesKvClient 时强制 kDistributed）
    FS->>F: CreateStorageBackend(config)
    F->>F: DistributedStorageConfig::Validate()<br/>single_tenant / 无 eviction_check_interval 要求
    F->>AD: make_unique<KvClientObjectStorageAdapter>(config)
    F->>BE: DistributedStorageBackend(cfg, distributed_cfg, nullptr, adapter)
    BE->>AD: Init()
    AD->>SDK: CreateKvClient()
    AD->>SDK: Init(configPath)
    SDK-->>AD: Status
    AD->>AD: PreflightSharedProvider()<br/>sharedProviderMode == SHARED ?
    AD->>AR: Init(region_bytes, max_slot_bytes)
    AR->>SDK: RegisterRegions(单个大 region)
    SDK-->>AR: RegisteredMemory{handle, tokenId}
    AR->>AR: 记录 canonical handle<br/>按 512B 对齐切分 slot 位图
    AR-->>AD: OK
    AD-->>BE: OK
    BE-->>FS: backend
    FS->>RC: Client::SetDfsStorageBackend(backend)
```

**说明**：

- **一次性注册，非按次注册**。v1 的「按次注册、按次释放」不可行（§2.3.2）。整个适配器生命周期只注册一个大 region，所有 I/O 从中做 512B 对齐变长子分配。
- **`PreflightSharedProvider` 是硬门禁**。默认 `INDEPENDENT` 模式下 SDK 为每个 ASU 节点新建 provider 并 `BindMemory` 全部已注册区域（`kv_client_impl.cc:687-702`、`:719-731`），而 AICPU provider 的 `BindMemory` **只接受 `MEM_DEVICE`**（`aicpu_trans_provider.cc:1691-1699`）→ 非空 view 下注册必然 `PARTIAL_FAILED`。故要求 `sharedProviderMode=SHARED`，缺省或为 0 直接返回 `DFS_SERVICE_UNAVAILABLE`。
- **Master 侧初始化**（`master_service.cpp:872-907` 新增分支）：置 `enable_dfs_=true`、`use_kvclient_=true`，然后 return。若 `enable_oplog_` 为真则 `LOG(FATAL)`（不支持恢复）。

### 2.3.2 写入阶段

适配器从 staging arena 取得已注册内存，把调用者的多段 slices 拷入连续 slot（尾部补零到 `aligned_size`），按对象边界切批后一次性提交 `BatchStoreAsync`，再 `Wait` 完成。成功则归还 slot，超时或取消则 quarantine。

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant BE as DistributedStorageBackend
    participant AD as KvClientObjectStorageAdapter
    participant AR as StagingArena
    participant SDK as kv::KvClient
    participant S as 远端存储

    C->>BE: BatchWrite(key + descriptor + slices)
    BE->>BE: 校验 total_size == desc.object_size<br/>每 slice 转 iovec（不合并）
    BE->>AD: PutBatch(ObjectPutRequest{key, iov, iovcnt})
    loop 每个请求
        AD->>AD: EncodeKey(logical_key) → CacheKey<br/>校验 iov 非空、同批重复 key 拒绝
        AD->>AD: n = Σiov_len；wire_len = AlignUp(n,512)<br/>校验 wire_len <= 16776704
        AD->>AR: Acquire(wire_len)
        alt arena 耗尽
            AR-->>AD: NO_AVAILABLE_HANDLE
            AD->>AD: results[i] = 错误（不阻塞等待）
        else 成功
            AR-->>AD: Slot{ptr, wire_len, index}
            AD->>AD: memcpy(iov → slot.ptr)<br/>memset 尾部 [n, wire_len) 补零
        end
    end
    AD->>AD: 按 110 切批（不切开单个对象）<br/>记录 originalIndices
    AD->>SDK: BatchStoreAsync(entries, taskId)
    SDK->>SDK: 按 nodeId 路由拆分 transport task
    SDK->>S: Host RDMA write（entry.length = wire_len）
    SDK-->>AD: 提交结果
    alt 提交失败
        AD->>AR: Release(全部 slot)
        AD-->>BE: 逐项错误
    else 已受理
        AD->>SDK: Wait(taskId, timeoutMs, result)
        alt Wait 返回 OK
            SDK-->>AD: TaskResult{status, entryStatus[]}
            AD->>AD: 逐项映射 entryStatus → ErrorCode
            AD->>AR: Release(全部 slot)
        else Wait 超时 / CANCELED
            Note over AD,AR: ⚠️ 禁止读 entryStatus（初值 OK，假成功）
            AD->>AR: Quarantine(全部 slot)<br/>永不归还、永不注销
            AD->>AD: 逐项置 RPC_TIMEOUT / INTERNAL_ERROR
        end
        AD-->>BE: 逐项 expected
    end
    BE-->>C: results[index]
    C->>C: AllAllocatedTransfersSucceeded?<br/>→ PutEnd(ALL) 或 PutRevoke(ALL)
```

### 2.3.3 读取阶段

读路径与写路径对称：driver 从 arena 取 slot 承接设备写入，`Wait` 完成后把数据 memcpy 回调用者原 buffer。明确 `NOT_FOUND` 触发 miss 清理，超时则 quarantine slot 且**不**触发清理。

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant BE as DistributedStorageBackend
    participant AD as KvClientObjectStorageAdapter
    participant AR as StagingArena
    participant SDK as kv::KvClient
    participant S as 远端存储

    C->>C: GetReplicaList → descriptor + lease
    C->>BE: BatchRead(key + descriptor + slices)
    BE->>BE: 校验 slice 容量 >= object_size<br/>size = desc.object_size = N
    BE->>AD: GetBatch(ObjectGetRequest{key, buffer, N})
    loop 每个请求
        AD->>AD: EncodeKey(logical_key) → CacheKey<br/>wire_len = AlignUp(N,512)
        AD->>AR: Acquire(wire_len)
        AR-->>AD: Slot{ptr, wire_len}
    end
    AD->>SDK: BatchLoadAsync(entries, taskId)<br/>entry.length = wire_len
    SDK->>S: Host RDMA read
    AD->>SDK: Wait(taskId, timeoutMs, result)
    alt 成功
        SDK-->>AD: entryStatus[] 全 OK
        AD->>AD: memcpy(slot.ptr → 调用者 buffer, **仅前 N 字节**)<br/>尾部 padding 丢弃
        AD->>AR: Release(slot)
        AD-->>BE: expected<size_t>(**N**)
        BE->>BE: 校验 *result == request.size(=N)<br/>（双层：BE :838-841 + 适配器）
    else 明确 NOT_FOUND
        AD->>AR: Release(slot)
        AD-->>C: 逐 key OBJECT_NOT_FOUND
        C->>C: ReconcileMissingKvReplica(key, ReplicaID)
    else 超时 / CANCELED
        Note over AD,AR: 禁止读 entryStatus
        AD->>AR: Quarantine(slot)
        AD-->>C: RPC_TIMEOUT / INTERNAL_ERROR
        Note over C: 元数据保留，不触发清理
    end
```

**说明**：读路径多一次 memcpy（staging → 调用者 buffer），这是同步契约的代价。slot 按 `aligned_size`（= `AlignUp(N,512)`）分配以容纳设备写入，但只把前 N 字节交还调用者，且 `GetBatch` 返回 **N**——因为 `DistributedStorageBackend` 比对的是 `request.size`，而它取 `desc.object_size` = N（`:817-818`）；返回 `wire_len` 会让每次读取都误报 `FILE_READ_FAIL`（`:838-841`）。

**slot 必须按 `aligned_size` 而非 N 分配**，否则设备按 `entry.length = wire_len` 写入会越过 slot 边界，踩坏相邻 slot 的数据。

### 2.3.4 删除阶段

删除只传 key、不传 buffer，因此不涉及 staging，也没有 quarantine 风险。value 删除成功（含幂等 `NOT_FOUND`）的 key 才进入元数据删除集合；失败项两者同时保留，可安全重试。

```mermaid
sequenceDiagram
    autonumber
    participant APP as 应用
    participant C as Client
    participant BE as DistributedStorageBackend
    participant AD as KvClientObjectStorageAdapter
    participant SDK as kv::KvClient
    participant MC as MasterClient
    participant M as MasterService

    APP->>C: BatchRemove(keys, force)
    C->>BE: BatchDeleteObjects(全部 keys)
    BE->>AD: DeleteBatch(logical_keys)
    AD->>AD: EncodeKey → CacheKey[]
    AD->>SDK: DeleteAsync(keys, taskId)
    AD->>SDK: Wait(taskId, timeoutMs, result)
    alt 成功或 NOT_FOUND（幂等）
        SDK-->>AD: entryStatus OK / ENTRY_KEY_NOT_EXIST
        AD-->>C: 逐项 OK
    else 超时 / 失败
        AD-->>C: 逐项错误
        Note over C: 该 key 不进元数据删除集合<br/>value 与元数据均保留 → 无孤儿
    end
    C->>C: 选出删除成功项 metadata_keys
    C->>MC: BatchRemove(metadata_keys, force)
    MC->>M: RPC
    M->>M: CheckRemovable（lease / 就绪 / 复制任务）
    alt 允许
        M->>M: Erase 元数据
        M-->>C: OK
    else 拒绝（OBJECT_HAS_LEASE 等）
        M-->>C: 错误
        Note over C,M:  value 已删、元数据保留<br/>后续读取触发 miss 自愈
    end
    C->>C: 回填原下标，合并两阶段结果
    C-->>APP: 逐 key expected
```

**说明**：删除失败项**不进入**元数据删除集合，故 value 与元数据同时保留，无孤儿。这是 value-first 相对 metadata-first 的核心优势——metadata-first 下失败项元数据已毁、value 残留，且重试遇 `OBJECT_NOT_FOUND`（`master_service.cpp:7391-7393`）→ 不进 `removed_keys` → **永不再删，孤儿被焊死**。

删除路径**不使用 staging**（`DeleteAsync` 只传 key，无 buffer），故无 quarantine 风险。

`Remove` 单 key 路由到 `BatchRemove`（`client_service.cpp:3816-3820` 门控放宽）。`RemoveAll` / `RemoveByRegex` 必须同样返回 `NOT_SUPPORTED` 且**不删元数据**（`:3843`、`:3864`），否则一次性泄漏整个 L3。

### 2.3.5 miss 清理阶段

清理是条件性的：Client 携带本次读取**观察到**的 ReplicaID，Master 在 key 锁内核对，只有仍为当前记录时才移除。读到 R1 而目录已是 R2 时返回不匹配，R2 原样保留。

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant MC as MasterClient
    participant M as MasterService
    participant META as 对象目录

    Note over C: GetBatch 返回 OBJECT_NOT_FOUND<br/>（仅明确 miss，超时不触发）
    C->>C: ReconcileMissingKvReplica(key, query, descriptor)
    C->>C: 门控：UsesKvcs() || UsesKvClient()<br/>&& descriptor.is_dfs_replica()<br/>&& resolved_tenant_id 已解析
    C->>MC: EvictDiskReplicaIfCurrent(key, tenant, DFS, ReplicaID)
    MC->>M: RPC
    M->>M: 白名单校验<br/>DFS && expected_replica_id != nullptr<br/>&& (use_kvcs_ || use_kvclient_)
    M->>META: key 锁内取 metadata
    alt ReplicaID 匹配
        M->>META: EraseReplicasWithCacheTotalAccounting(target_pred)
        M->>M: !metadata.IsValid() → accessor.Erase()
        M-->>C: true
    else ReplicaID 不匹配（已被 R2 替换）
        M-->>C: false（eviction_precondition_miss）
        Note over META: R2 原样保留
    end
    C->>C: 记日志，本次读取仍返回原 miss
```

### 2.3.6 关闭阶段

关闭必须先排空在途操作再注销 region。quarantine 的 slot 永不归还，随 region 统一注销；`InUseSlots()` 非零是泄漏信号，需显式告警。

```mermaid
sequenceDiagram
    autonumber
    participant AD as KvClientObjectStorageAdapter
    participant AR as StagingArena
    participant SDK as kv::KvClient

    Note over AD: 析构 / Shutdown
    AD->>AD: 停止接受新提交
    AD->>SDK: Drain / 等待 in-flight task 结束
    AD->>AR: InUseSlots() == 0 ?
    alt 仍有占用
        AR-->>AD: 非零
        AD->>AD: LOG(ERROR) 泄漏告警<br/>列出 quarantined 字节数
    else 已排空
        AR-->>AD: 0
    end
    AD->>AR: Shutdown()
    AR->>SDK: UnregisterRegions(canonical handle)
    AD->>SDK: Shutdown()
```

**说明**：不显式逐个 `UnregisterRegions`；quarantine 的 slot 永不归还，`Shutdown` 时统一随 region 注销。`QuarantinedBytes()` 非零是**内存泄漏指标**，需在关闭日志中显式输出。

## 2.4 SFMEA 分析

| 编号 | 故障模式 | 故障影响 | 容错措施 | 故障注入 / 备注 |
| --- | --- | --- | --- | --- |
| SFMEA-01 | logical key XXH64 碰撞：两个不同 key 映射到同一 CacheKey | **静默数据损坏**：后写覆盖先写，读到错 value，无校验可发现 | 无（默许碰撞、不做 fingerprint）；部署文档须写明 key 数量级上限 | 缩小 seed 空间强制构造碰撞。概率 ≈ n²/2^65：10^8 key → 2.7e-4，10^9 → 2.7% |
| SFMEA-02 | logical key 为空 | 无法编码 | `EncodeKey` 入口返回 `INVALID_PARAMS` | hash 编码不受长度限制，只需拒空 |
| SFMEA-03 | 同批重复 logical key | 同 CacheKey 在一个 task 内冲突，SDK 行为未定义 | 写侧直接拒绝；读/查/删去重后 fan-out | 需在新适配器内自行实现（可参照 `kvcs_object_storage_adapter.cpp:144-147`） |
| SFMEA-04 | 未按 `AlignUp(N,512)` 补零，直接把 N 当作 `entry.length` | 提交被拒或行为未定义；slot 复用时把上次 I/O 的残留数据发到存储侧 | `wire_len = AlignUp(n, 512)`，`Acquire(wire_len)`，尾部 `[n, wire_len)` 显式 `memset` 补零 | 写 513 字节对象，断言 `entry.length == 1024` 且尾部全 0。N 非 512 倍数是**合法输入**，故障在补零遗漏 |
| SFMEA-05 | N > 16,776,704 | 超 24-bit length 上限，`PackSqe` 本地拒绝；预发送失败会 **abort 整个 transport task**（`trans_task_executor.cc:227-232`） | 适配器逐请求预检 `wire_len <= kKvClientMaxValueBytes`；Master `PrepareWriteStart` 第一道校验 | 不能指望「只失败超限那条」。Master 那道必需：`kMaxSliceSize`（16,777,200）更宽且只在 CACHELIB 下生效 |
| SFMEA-06 | N == 0 | SDK 不接受空 value | `n == 0 → INVALID_PARAMS` | 后端 `BatchWrite` 另有 `object_size == 0` 校验，双重保障 |
| SFMEA-07 | `sharedProviderMode != SHARED` | INDEPENDENT 下 SDK 为每个 ASU 节点新建 provider 并 `BindMemory` 全部已注册区域，而 AICPU 的 `BindMemory` 只接受 `MEM_DEVICE` → **非空 view 下注册必然 PARTIAL_FAILED** | `Init()` 中 `PreflightSharedProvider` 硬门禁，缺省或为 0 直接 `DFS_SERVICE_UNAVAILABLE` | 部署硬门禁，必须在 Init 阶段失败而非运行期 |
| SFMEA-08 | `RegisterRegions` 失败 | 无 staging 内存，所有 I/O 不可用 | `Init()` 失败 → 后端初始化失败 → `FileStorage::Init` 明确失败 | 部分成功时 SDK 已自行回滚（`kv_client_impl.cc:359-373`） |
| SFMEA-09 | Staging arena 耗尽 | 新 I/O 无法分配 slot | 逐项返回 `NO_AVAILABLE_HANDLE`，**不阻塞等待**；计入 metric | 不可 fail-silent：读路径静默失败会永久不可诊断 |
| SFMEA-10 | 提交时 `buffer.handle` 非 canonical | 非 0 但未注册的 handle 在提交阶段**不报错**，到 Wait 才以 `BUFFER_NOT_REGISTERED` 暴露且**连坐整 task** | 提交前预检 handle **精确等于** canonical handle（不是 `!= kInvalidMRHandle`） | 预检必须精确相等 |
| SFMEA-11 | `mrKey` 被调用方填入（不涉及） | 无影响 | `SubmitAsync` 无条件重置并按 `buffer.handle` 回填（`kv_client_impl.cc:452-458`） | 适配器侧一律留 `std::nullopt` |
| SFMEA-12 | `Wait` 超时，设备仍在访问 staging | 若归还 slot 并被复用 → 设备写入他人数据 = **数据损坏** | slot **quarantine**：永不归还、永不注销，直到排空；超时后不再触碰该 slot | **承重组件，不可裁剪**。SDK 无取消能力（`Cancel` 无生产调用者） |
| SFMEA-13 | 超时后读 `entryStatus` | 初值是 `Status::OK()`，`BuildResult` 在 `!Done()` 时原样返回 → **假成功**，数据未落盘却报成功 | `Wait` 非 OK 时**不读** `entryStatus`，直接逐项置错误 | 必须有专门用例覆盖，否则极易回归 |
| SFMEA-14 | task 级 `PARTIAL_FAILED`，逐项 entryStatus 部分失败 | 部分 key 成功部分失败 | 逐项映射 entryStatus → ErrorCode，按 `originalIndices` 回填原下标 | entryStatus 与输入严格一一对应 |
| SFMEA-15 | dispatch 失败导致整 task CANCELED | 所有 entry 被置 CANCELED（`task_manager.cc:216-223`） | CANCELED 一律 quarantine slot，逐项返回 `INTERNAL_ERROR` | 与 TIMEOUT 同等处理 |
| SFMEA-16 | 路由失败：`routed asu transport not found` | 整个 task 提交失败（`task_manager.cc:276-282`） | 提交阶段即返回错误，slot 全部 Release（未进入设备） | 与超时区分：提交失败可安全归还 slot |
| SFMEA-17 | `CQE_OUT_OF_CREATE_SIZE`（设备侧空间不足） | 该 key 写入失败 | 映射为 `NO_AVAILABLE_HANDLE`；两副本硬逻辑裁决下触发 `PutRevoke(ALL)` | 无容量查询接口 → 无法预判，只能事后失败 |
| SFMEA-18 | `CQE_KEY_NOT_EXIST` / `ENTRY_KEY_NOT_FOUND`（读时） | 对象在存储侧不存在 | 映射为 `OBJECT_NOT_FOUND` → 触发 `ReconcileMissingKvReplica` 条件清理 | **必须与超时严格区分**：超时**不**触发清理 |
| SFMEA-19 | 连接失败 / `CONNECTION_ERROR` | I/O 失败 | 映射为 `DFS_SERVICE_UNAVAILABLE`；元数据保留 | 与 KVCS `CallError` 映射对齐（`kvcs_capi_driver.cpp:77-100`） |
| SFMEA-20 | 删除阶段：value 删除成功但 Master `BatchRemove` 失败 | value 已失、元数据保留 → 已发布对象不可读 | 后续读取返回 `NOT_FOUND` → miss 自愈清理 DFS 记录 | value-first 删除依赖 miss 清理自愈，二者必须同时落地 |
| SFMEA-21 | 删除阶段：`CheckRemovable` 拒绝（lease / 未就绪 / 复制任务） | 同 SFMEA-20，但**无自愈**——对象仍受租约保护，元数据不会被判为失效 | 返回该 key 错误；元数据保留；靠后续读取 miss 自愈 | 已知窗口。语义安全（不返回错数据），但丢已成功 Put 的数据 |
| SFMEA-22 | 删除阶段：value 删除失败（超时 / 不可达） | value 与元数据**同时保留** | 该 key 不进元数据删除集合 → **无孤儿**；返回错误供上层重试 | value-first 的核心收益：失败态可安全重试 |
| SFMEA-23 | 迟到 Delete 误删新 value（同 key 重写） | 旧删除请求命中新写入的 value | 无专门机制（XXH64 下新旧 value 同 CacheKey）；调用方须串行组织同 key 的删除/重建/更新 | v1 §1.2 约定保留 |
| SFMEA-24 | 删除后 Client 崩溃（value 已删、元数据未删） | 同 SFMEA-20 | 重启后由读取 miss 自愈 | 无 pending-delete 持久化队列，依赖自愈 |
| SFMEA-25 | `RemoveAll` / `RemoveByRegex` 被调用 | 若只删元数据不删 value → **一次性泄漏整个 L3** | 门控放宽到 KVClient，返回 `NOT_SUPPORTED` 且**不删元数据** | `client_service.cpp:3843`、`:3864`，value-first 删除的连带必改项 |
| SFMEA-26 | Master 重启（默认无 HA） | 对象目录全丢；L3 value 仍在但**不可发现**（无枚举能力）→ 等同丢失且永久占用空间 | 无恢复能力（不支持恢复裁决）。运维需知晓；残留由存储侧或人工处理 | 默认 `enable_ha=false`、`enable_oplog=false` |
| SFMEA-27 | 误配 `enable_oplog=true` | 恢复门会在 HA 切换时返回 `DFS_SERVICE_UNAVAILABLE`（响亮失败，非静默丢数据） | `InitDfsAllocatorFromEnvironment` 启动期 `LOG(FATAL)`，把失败点提前到启动时 | 恢复门 3533/3825/4118 保持 `use_kvcs_` only，**不改** |
| SFMEA-28 | 三副本 `memory + nof + ASU` | `dfs > 0` 时 `FLEXIBLE_DUAL_REPLICA` 不可达（`replica.h:315-318`）→ 静默落入 RELIABLE，**nof 变为必需**，nof 抖动导致整 Put 失败 | `PrepareWriteStart` 显式拒绝 `INVALID_PARAMS` | 有意与 KVCS 不对称：把运行期惊喜变成启动期可发现约束 |
| SFMEA-29 | `PutEnd` 响应未知 | 数据已落 L3，但发布状态不明 | Client 返回提交错误；后续经查询或符合条件的重复提交确认（既有行为） | 沿用现有语义，不新增机制 |
| SFMEA-30 | miss 清理 CAS 不匹配（读到 R1，目录已是 R2） | 无 | `EvictDiskReplicaIfCurrent` 返回 false，R2 原样保留（`master_service.cpp:6463-6473`） | 既有 CAS 语义，仅需放宽白名单 |
| SFMEA-31 | 存储侧空间满，无水位淘汰 | 写入持续失败，无自动回收 | KVClient 初始化分支**不建** capacity provider，`RunKvcsDfsEviction` 因 provider 为 null 直接返回；写入按 SFMEA-17 失败 | 容量管理留待 SDK 提供接口 |
| SFMEA-32 | `Shutdown` 时仍有 in-flight task | 注销 region 会导致设备访问已注销内存 | 先 `Drain` 排空，再断言 `InUseSlots()==0`，非零则 `LOG(ERROR)` 并输出 `QuarantinedBytes()` | quarantine slot 永不归还，关闭时随 region 统一注销 |
| SFMEA-33 | 读回长度 != `object_size` | 数据不完整 | 双层校验：适配器内 + `DistributedStorageBackend:838-841`，返回 `FILE_READ_FAIL` | `GetBatch` 必须返回 **N**（= `object_size`）而非 `wire_len`，否则每次读取都误报 |
| SFMEA-34 | 范围读请求 | `BatchRead` 对象分支忽略 `range_read`/`value_offset`，slice 容量 < object_size → `INVALID_PARAMS`（`:802-818`） | 不支持。上层需回退整对象读后截取 | RealClient 的 DFS 范围读只分配窗口大小 staging，KVCS 下今天也会失败 |
| SFMEA-35 | 租约到期 | 读取结果可能已失效 | `IsLeaseExpired` 检查后返回 `LEASE_EXPIRED`（既有） | 沿用现有语义 |
| SFMEA-36 | 设备地址直达 / 设备 buffer（不涉及） | — | 不支持。`MemoryType` 保持 `HOST_PINNED`；SDK 把 HOST/HOST_PINNED 折叠为 `MEM_HOST` | 范围外 |

## 2.5 可定位设计

### 2.5.1 统一关联字段

所有 driver 与适配器日志必须携带以下字段，使一次 I/O 可跨层串联：

| 字段 | 来源 | 用途 |
| --- | --- | --- |
| `logical_key` | 应用 | 业务侧关联（可能含敏感信息，按日志级别裁剪） |
| `cache_key` | XXH64 结果的 16 位 hex | **碰撞排查的唯一手段**：两个不同 logical_key 出现同一 cache_key 即为碰撞 |
| `object_size` | descriptor | 对齐/上限问题定位 |
| `task_id` | SDK | 跨 client/transport 层关联 SDK 内部日志 |
| `replica_id` | descriptor | miss 清理 CAS 排查 |
| `slot_index` | StagingArena | 泄漏与 quarantine 追踪 |
| `batch_index` / `original_index` | driver | 批量结果错位排查 |

`cache_key` 是key 碰撞裁决（不做 fingerprint）下**唯一**的碰撞可观测点，必须在 VLOG(2) 无条件输出。

### 2.5.2 日志分级

| 级别 | 内容 |
| --- | --- |
| `ERROR` | Wait 超时（含 `task_id`、`op`、涉及 slot 数、**quarantine 字节数**）；`RegisterRegions` 失败；`Shutdown` 时 `InUseSlots() != 0`；entryStatus 映射到未知 SDK 码 |
| `WARNING` | arena 占用率 > 90%；`PreflightSharedProvider` 通过但 provider 类型非预期；单项 entryStatus 失败（非整 task） |
| `INFO` | `Init` 成功（含 SDK 版本、`sharedProviderMode`、region 大小、slot 数、`MaxValueSize`）；`Shutdown` 完成（含累计 quarantine 字节） |
| `VLOG(1)` | 每次 `BatchPut`/`BatchGet`/`BatchDelete` 的 key 数、总字节、切批数、耗时分解 |
| `VLOG(2)` | 逐 entry 的 `logical_key` → `cache_key` 映射、slot 分配/归还 |

**耗时分解**（`VLOG(1)`，对应用例 T-54）：`encode_us` / `acquire_us` / `memcpy_us` / `submit_us` / `wait_us` / `release_us`，用于确定推荐 I/O 粒度与定位 staging 拷贝占比。

### 2.5.3 指标

| 指标 | 类型 | 说明 |
| --- | --- | --- |
| `kvclient_adapter_ops_total{op,result}` | counter | op ∈ {put,get,delete,query}，result ∈ {ok,error,timeout,canceled} |
| `kvclient_adapter_bytes_total{op,direction}` | counter | direction ∈ {payload,padded}，分别统计 N 与 `aligned_size` 字节数，二者差值即补零开销 |
| `kvclient_adapter_latency_us{op,phase}` | histogram | phase ∈ {encode,acquire,memcpy,pad,submit,wait,release} |
| `kvclient_staging_slots_in_use` | gauge | 当前占用 slot 数 |
| `kvclient_staging_quarantined_bytes` | gauge | **泄漏指标**：单调不减，非零即有未排空的超时 slot |
| `kvclient_staging_exhausted_total` | counter | arena 耗尽次数 |
| `kvclient_entry_status_total{code}` | counter | 按 SDK StatusCode 分布，用于识别设备侧错误模式 |

`kvclient_staging_quarantined_bytes` 持续增长是**内存泄漏的确定信号**，应配告警。

`kvclient_adapter_bytes_total` 拆 payload / padded 两个方向是为了直接观测补零放大率：小对象（N 远小于 512）的放大最严重，N=1 时 padded 是 512，放大 512 倍。这是评估「是否值得让上层聚合小对象」的数据来源。

### 2.5.4 定位手册

| 症状 | 排查路径 |
| --- | --- |
| Init 失败 `DFS_SERVICE_UNAVAILABLE` | 查 `sharedProviderMode` 是否为 SHARED（SFMEA-07）；查 view service 可达性 |
| Init 失败 `PARTIAL_FAILED` | 查 `RegisterRegions` 日志；AICPU provider 下确认 SHARED 模式 |
| Put 全部失败但 SDK 日志无异常 | 查是否读了超时后的 `entryStatus`（SFMEA-13）；查 `kvclient_adapter_ops_total{result="timeout"}` |
| 读到错误数据，无报错 | **优先怀疑 key 碰撞**（SFMEA-01）：grep VLOG(2) 日志，比对两个 logical_key 是否得到同一 `cache_key` |
| 读取恒返回 `FILE_READ_FAIL`，但数据其实写对了 | 查 `GetBatch` 是否误返回 `aligned_size` 而非 N（§3.5）——后端比对的是 `desc.object_size` |
| 读到的对象尾部有脏数据 | 查写路径是否漏了 `memset` 补零，或读路径 memcpy 是否误用了 `wire_len` 而非 N |
| 内存持续增长不释放 | 查 `kvclient_staging_quarantined_bytes`；非零说明有超时 slot 未排空（SFMEA-12） |
| Remove 返回成功但存储侧仍有 value | 不可能——value-first 下成功即已删。若出现，查是否走了 `RemoveAll`/`RemoveByRegex`（SFMEA-25） |
| Remove 返回错误且对象变为不可读 | 已知窗口（SFMEA-21）：value 已删但 `CheckRemovable` 拒绝。等待后续读取触发 miss 自愈 |
| Get 持续 `OBJECT_NOT_FOUND` 但目录有记录 | 查 `EvictDiskReplicaIfCurrent` 是否返回 `INVALID_PARAMS`——说明 `use_kvclient_` 白名单漏改（§3.6） |
| 对象大于 4 MiB 写入失败 | 查 `kKvClientMaxValueBytes` 是否被误配；KVClient 上限是 16,776,704 而非 KVCS 的 4 MiB |
| N 在 16,776,705 ~ 16,777,200 之间的对象写入失败 | 预期行为。该区间能通过 Mooncake 的 `kMaxSliceSize` 但超 SDK 线格式上限，由 `PrepareWriteStart` 拒绝（§3.6） |
| Master 重启后 L3 全部读不到 | 预期行为（不支持恢复裁决，SFMEA-26）。目录不可恢复，value 不可枚举 |

### 2.5.5 诊断辅助接口

适配器暴露只读诊断方法（不在 `ObjectStorageAdapter` 接口上，通过独立头文件提供，运维工具 `dynamic_cast` 获取）：

```cpp
struct KvClientAdapterStats {
    size_t slots_total;
    size_t slots_in_use;
    size_t slots_quarantined;
    uint64_t quarantined_bytes;
    uint64_t submits_total;
    uint64_t timeouts_total;
    uint64_t canceled_total;
    uint64_t padded_bytes_total;   // 累计补零字节数
};
KvClientAdapterStats KvClientObjectStorageAdapter::GetStats() const;
```

另提供存在性核查工具方法（内部用 `QueryAsync`）供运维手工排查单个 key。注意：适配器层 `Exists` 仍返回 `NOT_SUPPORTED`（与 KVCS 一致），因为**物理存在性只能观测、不能当权威**——`QueryResult` 只有 `exists` 位图（`kv_types.h:98-101`），无长度信息，长度权威始终是 Master 元数据。

---

# 3 shard 设计描述

## 3.1 shard-1：key 编码（`kvclient_key_codec`）

**职责**：logical key → 8 字节 `CacheKey` 的确定性映射。

**文件**：`mooncake-store/include/storage/distributed/kvclient/kvclient_key_codec.h`（header-only）

```cpp
#pragma once

#include <cstdint>
#include <cstring>
#include <string>
#include <string_view>

#include <xxhash.h>

#include "kv_types.h"

namespace mooncake::kvclient {

// Fixed seed. The mapping must be a pure function of the logical key so that
// any client, at any time, derives the same CacheKey. Do not mix in epoch,
// process id, or random salt: that would break cross-client reads.
inline constexpr XXH64_hash_t kCacheKeySeed = 0x4D434B5643310001ULL;

// Collisions are accepted by design (no fingerprint validation). Two distinct
// logical keys mapping to the same CacheKey silently overwrite each other.
// Probability is ~n^2/2^65; see SFMEA-01 for the deployment key-count bound.
inline kv::CacheKey EncodeCacheKey(std::string_view logical_key) {
    const XXH64_hash_t hash =
        XXH64(logical_key.data(), logical_key.size(), kCacheKeySeed);
    kv::CacheKey key{};
    std::memcpy(key.data(), &hash, sizeof(hash));
    return key;
}

// 16 hex chars. This is the only observable handle for collision diagnosis,
// so it must be logged unconditionally at VLOG(2).
inline std::string CacheKeyToLogString(const kv::CacheKey& key) {
    static constexpr char kHex[] = "0123456789abcdef";
    std::string text;
    text.reserve(key.size() * 2);
    for (auto byte : key) {
        const auto value =
            static_cast<unsigned char>(std::to_integer<unsigned char>(byte));
        text.push_back(kHex[value >> 4]);
        text.push_back(kHex[value & 0x0F]);
    }
    return text;
}

}  // namespace mooncake::kvclient
```

**设计说明**：

- **seed 选择**：借鉴 `nvme_kv/key_codec.cpp:44-45` 的 XXH64 用法，但那里用双 seed 产出 16 字节（`low`/`high`）；本方案只需 8 字节，单 seed 即可。
- **不做 fingerprint**（key 碰撞裁决）：value 前 N 字节就是应用原始数据（尾部 `[N, aligned_size)` 是补零，读回时丢弃），故无需 value 内嵌校验头，§2.3.3 的双层长度校验得以保留。代价是碰撞不可检测，只能靠 §2.5.1 的 `cache_key` 日志事后排查。
- **纯函数约束**：映射不得混入 epoch / 进程 id / 随机种子。这既是跨 Client 读取的前提，也是不支持恢复裁决下唯一需要保证的稳定性。

## 3.2 shard-2：staging arena（`kvclient_staging_arena`）

**职责**：一个已注册 region + 512B 对齐变长子分配 + quarantine。

**文件**：`mooncake-store/include/storage/distributed/kvclient/kvclient_staging_arena.h`、`src/storage/distributed/kvclient/kvclient_staging_arena.cpp`

```cpp
#pragma once

#include <cstddef>
#include <cstdint>
#include <mutex>
#include <vector>

#include <ylt/util/tl/expected.hpp>

#include "kv_types.h"
#include "types.h"

namespace mooncake {
namespace kv {
class KvClient;
}

namespace kvclient {

// Owns the only memory the SDK is allowed to touch. The SDK never cancels an
// in-flight operation on Wait timeout (AsuTransportImpl::Cancel has no
// production caller), so a slot handed to a timed-out task can never be safely
// reused: Quarantine() retires it for the lifetime of the arena.
class StagingArena {
   public:
    struct Slot {
        void* ptr = nullptr;
        size_t size = 0;
        size_t index = 0;
    };

    StagingArena() = default;
    ~StagingArena();

    StagingArena(const StagingArena&) = delete;
    StagingArena& operator=(const StagingArena&) = delete;

    // Registers one contiguous HOST_PINNED region and records its handle as
    // the canonical handle every submitted entry must carry.
    tl::expected<void, ErrorCode> Init(kv::KvClient& client,
                                       size_t region_bytes,
                                       size_t max_slot_bytes);
    // Caller drains in-flight work first; logs an error if InUseSlots() != 0.
    void Shutdown(kv::KvClient& client);

    // bytes must be a multiple of 512. Returns NO_AVAILABLE_HANDLE when the
    // arena is exhausted; never blocks.
    tl::expected<Slot, ErrorCode> Acquire(size_t bytes);
    void Release(const Slot& slot);
    // Retires the slot permanently. Counted in QuarantinedBytes().
    void Quarantine(const Slot& slot);

    size_t InUseSlots() const;
    size_t QuarantinedSlots() const;
    uint64_t QuarantinedBytes() const;
    kv::MRHandle CanonicalHandle() const { return canonical_handle_; }

   private:
    enum class SlotState : uint8_t { kFree, kInUse, kQuarantined };

    mutable std::mutex mu_;
    void* base_ = nullptr;
    size_t region_bytes_ = 0;
    size_t slot_stride_ = 0;  // max_slot_bytes, already 512-aligned
    kv::MRHandle canonical_handle_ = kv::kInvalidMRHandle;
    std::vector<SlotState> states_;
    std::vector<size_t> free_list_;
    size_t quarantined_slots_ = 0;
    uint64_t quarantined_bytes_ = 0;
};

}  // namespace kvclient
}  // namespace mooncake
```

**关键实现**：

```cpp
tl::expected<StagingArena::Slot, ErrorCode> StagingArena::Acquire(size_t bytes) {
    if (bytes == 0 || bytes % kv::kAlignmentBytes != 0 || bytes > slot_stride_) {
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    std::lock_guard<std::mutex> lock{mu_};
    if (free_list_.empty()) {
        return tl::make_unexpected(ErrorCode::NO_AVAILABLE_HANDLE);
    }
    const size_t index = free_list_.back();
    free_list_.pop_back();
    states_[index] = SlotState::kInUse;
    return Slot{static_cast<char*>(base_) + index * slot_stride_, bytes, index};
}

void StagingArena::Quarantine(const Slot& slot) {
    std::lock_guard<std::mutex> lock{mu_};
    if (slot.index >= states_.size() || states_[slot.index] != SlotState::kInUse) {
        return;
    }
    states_[slot.index] = SlotState::kQuarantined;
    ++quarantined_slots_;
    quarantined_bytes_ += slot.size;
    // Deliberately not pushed back to free_list_: the device may still write
    // into this slot after Wait returned TIMEOUT.
}
```

**设计说明**：

- **为什么是变长子分配而非固定 16,776,704B slot**：固定 slot 下 128 MiB region 只能开 8 个，批深度不足以兑现跨对象合批。变长子分配以 `max_slot_bytes` 为 stride，`Acquire(bytes)` 只要求 `bytes <= slot_stride_`，小对象不浪费批深度。
- **`slot_stride_` 必须 512 对齐**：SDK 只约束 `length`/`offset` 的 512 对齐，**不校验 `buffer_addr`**；但 region 起始地址与 stride 都对齐可让每个 slot 天然满足地址对齐，避免依赖 SDK 的未校验行为。
- **quarantine 不注销**：quarantine 的 slot 仍属已注册 region，`Shutdown` 时随 region 统一注销。逐个注销会与 in-flight 设备操作竞争。
- **`HOST_PINNED`**：SDK 把 `HOST` / `HOST_PINNED` 都折叠为 `MEM_HOST`（`kv_client_impl.cc:350-354`），pinned 不由 SDK 分配兑现。若需要真正的 pinned 内存，arena 需自行分配（后续优化项，V1 用普通 host 内存）。
- **`tokenId` 只作诊断**：`RegisteredMemory.tokenId` 不入 `KVBuffer`、不当 handle 用；`mrKey` 必须留 `std::nullopt`（SDK 会按 `buffer.handle` 回填，见 SFMEA-11）。

## 3.3 shard-3：适配器（`kvclient_object_storage_adapter`）

**职责**：实现 `ObjectStorageAdapter`，承载 key 编码、staging、提交/等待、结果映射。与 `KvcsObjectStorageAdapter` 平级，互不依赖。

**文件**：`mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h`、`src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp`

```cpp
#pragma once

#include <memory>
#include <string>
#include <vector>

#include "storage/distributed/object_storage_adapter.h"
#include "storage/distributed/kvclient/kvclient_staging_arena.h"

namespace mooncake {

inline constexpr char kKvClientAdapterName[] = "kvclient";

// Single-entry wire limit: SQE length is 24-bit and must be 512-aligned, so
// 0xFFFFFF & ~511 == 0xFFFE00. Not "about 16 MiB" -- the exact bound matters
// because Mooncake's kMaxSliceSize (16,777,200) is 496 bytes wider and will
// admit values that cannot go on the wire.
inline constexpr uint64_t kKvClientMaxValueBytes = 16'776'704;

struct KvClientAdapterConfig {
    std::string config_path;  // SDK client config
    size_t staging_region_bytes = 128ULL * 1024 * 1024;
    size_t max_slot_bytes = kKvClientMaxValueBytes;
    uint64_t wait_timeout_ms = 30'000;
};

class KvClientObjectStorageAdapter final : public ObjectStorageAdapter {
   public:
    explicit KvClientObjectStorageAdapter(KvClientAdapterConfig config = {});
    ~KvClientObjectStorageAdapter() override;

    tl::expected<void, ErrorCode> Put(const std::string& logical_key,
                                      std::span<const char> data) override;
    tl::expected<void, ErrorCode> PutV(const std::string& logical_key,
                                       const iovec* iov, int iovcnt) override;
    std::vector<tl::expected<void, ErrorCode>> PutBatch(
        const std::vector<ObjectPutRequest>& requests) override;
    tl::expected<size_t, ErrorCode> Get(const std::string& logical_key,
                                        void* buf, size_t len) override;
    std::vector<tl::expected<size_t, ErrorCode>> GetBatch(
        const std::vector<ObjectGetRequest>& requests) override;
    tl::expected<bool, ErrorCode> Exists(
        const std::string& logical_key) override;
    tl::expected<void, ErrorCode> Delete(const std::string& logical_key) override;
    ObjectStorageIoResults DeleteBatch(
        std::span<const std::string> logical_keys) override;
    tl::expected<std::vector<KeyInfo>, ErrorCode> ListKeys() override;

    tl::expected<void, ErrorCode> Init() override;
    const char* GetName() const override { return kKvClientAdapterName; }

    KvClientAdapterStats GetStats() const;

   private:
    kv::CacheKey EncodeKey(std::string_view logical_key) const;
    void SubmitAndWait(kv::AsuOpType op, std::vector<kv::KVBuffer>& entries,
                       const std::vector<size_t>& original_indices,
                       const std::vector<StagingArena::Slot>& slots,
                       BatchResults& results);
    static ErrorCode MapEntryStatus(const kv::Status& status);
    bool PreflightSharedProvider() const;
    void Shutdown();

    KvClientAdapterConfig config_;
    std::unique_ptr<kv::KvClient> kv_client_;
    StagingArena arena_;
    bool initialized_ = false;
};

}  // namespace mooncake
```

**未覆写的接口**（沿用 `ObjectStorageAdapter` 基类默认）：

| 方法 | 基类默认 | 说明 |
| --- | --- | --- |
| `GetRange` | `NOT_SUPPORTED` | 范围读不支持（§1.2.2） |
| `CheckHealth` | `NOT_SUPPORTED` | 需要真机连通性探测，V1 不做 |
| `SupportsSinglePutAttempt` / `PutOnce` / `PutVOnce` | `false` / `NOT_SUPPORTED` | 仅 immutable-object backend 使用，KVClient 不走该路径 |

`Exists` 与 `ListKeys` 显式覆写为 `NOT_SUPPORTED`（与 KVCS 一致，理由见 §2.1.2 设计要点 4）。

**`Init` 实现**：

```cpp
tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::Init() {
    if (initialized_) return {};
    kv_client_ = kv::CreateKvClient();
    if (!kv_client_) return tl::make_unexpected(ErrorCode::INTERNAL_ERROR);

    const auto status = kv_client_->Init(config_.config_path);
    if (!status.ok()) return tl::make_unexpected(MapStatus(status));

    // INDEPENDENT mode makes the SDK BindMemory every registered region on
    // each per-node provider, and AICPU's BindMemory rejects non-device
    // memory -> registration always fails PARTIAL_FAILED once the view is
    // non-empty. Fail here rather than at first I/O.
    if (!PreflightSharedProvider()) {
        return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);
    }

    auto arena_init = arena_.Init(*kv_client_, config_.staging_region_bytes,
                                  config_.max_slot_bytes);
    if (!arena_init) return tl::make_unexpected(arena_init.error());

    initialized_ = true;
    LOG(INFO) << "action=kvclient_adapter_init"
              << ", staging_region_bytes=" << config_.staging_region_bytes
              << ", slot_stride=" << config_.max_slot_bytes
              << ", max_value_bytes=" << kKvClientMaxValueBytes
              << ", wait_timeout_ms=" << config_.wait_timeout_ms;
    return {};
}
```

**批量提交与超时处理骨架**（承重逻辑）：

```cpp
// Submits one sub-batch and maps results back through original_indices.
// On Wait failure the slots are quarantined, never released, and entryStatus
// is never read.
void KvClientObjectStorageAdapter::SubmitAndWait(
    kv::AsuOpType op, std::vector<kv::KVBuffer>& entries,
    const std::vector<size_t>& original_indices,
    const std::vector<StagingArena::Slot>& slots,
    BatchResults& results) {
    // Every entry must carry the canonical handle exactly. A non-zero but
    // unregistered handle is not rejected at submit time; it surfaces only at
    // Wait as BUFFER_NOT_REGISTERED and poisons the whole task.
    const auto canonical = arena_.CanonicalHandle();
    for (const auto& entry : entries) {
        if (entry.buffer.handle != canonical) {
            for (const auto& slot : slots) arena_.Release(slot);
            FillError(results, original_indices, ErrorCode::INTERNAL_ERROR);
            return;
        }
    }

    kv::TaskId task_id = kv::kInvalidTaskId;
    const auto submit = (op == kv::AsuOpType::BATCH_STORE)
                            ? kv_client_->BatchStoreAsync(entries, task_id)
                            : kv_client_->BatchLoadAsync(entries, task_id);
    if (!submit.ok()) {
        // Rejected before reaching the device: slots are safe to release.
        for (const auto& slot : slots) arena_.Release(slot);
        FillError(results, original_indices, MapStatus(submit));
        return;
    }

    kv::TaskResult task_result;
    const auto wait =
        kv_client_->Wait(task_id, config_.wait_timeout_ms, task_result);

    if (!wait.ok() || !task_result.status.ok()) {
        // TIMEOUT / CANCELED / PARTIAL_FAILED at task level.
        // entryStatus may still hold its Status::OK() initial value
        // (kv_client_impl.cc:460), so reading it here would report success
        // for data that never landed. Quarantine and fail every entry.
        for (const auto& slot : slots) arena_.Quarantine(slot);
        const auto code =
            (wait.code == kv::StatusCode::TIMEOUT ||
             task_result.status.code == kv::StatusCode::TIMEOUT)
                ? ErrorCode::RPC_TIMEOUT
                : ErrorCode::INTERNAL_ERROR;
        LOG(ERROR) << "action=kvclient_task_failed, op=" << AsuOpName(op)
                   << ", task_id=" << task_id
                   << ", entries=" << entries.size()
                   << ", quarantined_slots=" << slots.size()
                   << ", quarantined_bytes_total=" << arena_.QuarantinedBytes()
                   << ", code=" << toString(code);
        FillError(results, original_indices, code);
        return;
    }

    if (task_result.entryStatus.size() != entries.size()) {
        for (const auto& slot : slots) arena_.Quarantine(slot);
        FillError(results, original_indices, ErrorCode::INTERNAL_ERROR);
        return;
    }

    for (size_t i = 0; i < entries.size(); ++i) {
        results.Set(original_indices[i], MapEntryStatus(task_result.entryStatus[i]));
        arena_.Release(slots[i]);
    }
}
```

**错误码映射**（对齐 KVCS 的 `CallError` / `ItemError`，`kvcs_capi_driver.cpp:77-123`）：

```cpp
ErrorCode KvClientObjectStorageAdapter::MapEntryStatus(const kv::Status& status) {
    using SC = kv::StatusCode;
    switch (status.code) {
        case SC::OK:
            return ErrorCode::OK;
        case SC::ENTRY_KEY_NOT_FOUND:
        case SC::ENTRY_DATA_NOT_EXIST:
        case SC::CQE_KEY_NOT_EXIST:
        case SC::NOT_FOUND:
            return ErrorCode::OBJECT_NOT_FOUND;
        case SC::CQE_OUT_OF_CREATE_SIZE:
        case SC::RESOURCE_BUSY:
            return ErrorCode::NO_AVAILABLE_HANDLE;
        case SC::TIMEOUT:
        case SC::CQE_IO_TIMEOUT:
            return ErrorCode::RPC_TIMEOUT;
        case SC::CONNECTION_ERROR:
            return ErrorCode::DFS_SERVICE_UNAVAILABLE;
        case SC::BUFFER_NOT_REGISTERED:
        case SC::BUFFER_NOT_SUPPORTED:
        case SC::INVALID_ARGUMENT:
            return ErrorCode::INVALID_PARAMS;
        case SC::CANCELED:
            return ErrorCode::INTERNAL_ERROR;
        default:
            LOG(ERROR) << "action=kvclient_unmapped_status, code="
                       << static_cast<int>(status.code);
            return ErrorCode::INTERNAL_ERROR;
    }
}
```

⚠️ `OBJECT_NOT_FOUND` 与 `RPC_TIMEOUT` 的区分是**承重**的：只有前者触发 `ReconcileMissingKvReplica` 清理元数据，后者必须保留元数据（SFMEA-18）。

**`DeleteBatch` 的幂等处理**：

```cpp
// ENTRY_KEY_NOT_EXIST is idempotent success, matching the KVCS adapter
// (kvcs_capi_driver.cpp:358-366).
const auto code = MapEntryStatus(task_result.entryStatus[i]);
results[original_indices[i]] =
    (code == ErrorCode::OK || code == ErrorCode::OBJECT_NOT_FOUND)
        ? tl::expected<void, ErrorCode>()
        : tl::make_unexpected(code);
```

删除路径**不使用 staging**（`DeleteAsync` 只传 key，无 buffer），故无 quarantine 风险。

**切批上限**：`TransportConfig` 的 `batchLoadIoNum` / `batchStoreIoNum` = 110、`deleteIoNum` = 254、`queryIoNum` = 256（`kv_types.h:196-199`，均为**结构体成员字段**而非编译期常量，可被配置文件覆盖）。适配器不应硬编码这些值，而应：

- 优先从 SDK 的 `TransportConfig` 读取实际生效值；若 SDK 不暴露读取入口，则用同值作为适配器侧的保守默认，并在 `Init` 日志中打印所用批上限，便于与 SDK 配置不一致时排查。
- 三个上限不可统一成 110——那会让 Delete/Query 无谓降批；也不可放大，可能触发协议错误。
- 切批按对象边界，绝不切开单个对象（§2.3.2）。

## 3.4 shard-4：配置（`kvclient_config`）

**环境变量**（沿用 `Environ::ReadOr` 模式，`kvcs_capi_driver.cpp:39-75`）：

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `MOONCAKE_KVCLIENT_CONFIG` | 无（必填） | SDK client 配置路径 |
| `MOONCAKE_KVCLIENT_STAGING_BYTES` | 128 MiB | arena region 大小 |
| `MOONCAKE_KVCLIENT_MAX_SLOT_BYTES` | 16,776,704 | 单 slot 上限，须 512 对齐 |
| `MOONCAKE_KVCLIENT_WAIT_TIMEOUT_MS` | 30000 | `Wait` 超时 |

**校验**（`ReadConfig`）：`config_path` 非空；`staging_bytes >= max_slot_bytes`；`max_slot_bytes % 512 == 0 && <= 16776704`；`wait_timeout_ms > 0`。

**不设 `MAX_KEY_SIZE`**：hash 编码后 key 恒为 8 字节，与 logical key 长度无关，该配置无意义（这正是 `KvcsDriver::MaxKeySize()` 不适配 KVClient 的表现之一）。

`DistributedStorageConfig::Validate()` 新增分支（`distributed_storage_config.cpp:63-77` 旁）：

```cpp
if (UsesKvClient()) {
    if (!single_tenant) {
        LOG(ERROR) << "DistributedStorageConfig: KVClient requires "
                      "single_tenant=true";
        return false;
    }
    // No capacity API -> no watermark eviction -> eviction_check_interval is
    // meaningless. Deliberately not required, unlike the KVCS branch.
    return true;
}
```

`UsesObjectStorage()` 与新增谓词（`distributed_storage_config.h:27-31`）：

```cpp
bool UsesObjectStorage() const {
    return fs_adapter_type == "oss" || fs_adapter_type == "s3" ||
           fs_adapter_type == "kvcs-lowlevel" || fs_adapter_type == "kvclient";
}
bool UsesKvcs() const { return fs_adapter_type == "kvcs-lowlevel"; }
bool UsesKvClient() const { return fs_adapter_type == "kvclient"; }
```

## 3.5 shard-5：padding 与长度语义

512 对齐裁决下 `object_size = N`（任意），`aligned_size = AlignUp(N, 512)`（线格式长度）。二者的差值由适配器在 staging slot 内补零承担，**上层不可见**。

**写路径**：

```cpp
// PutBatch: 后端传来的 ObjectPutRequest{logical_key, iov, iovcnt}
const size_t n = SumIovecBytes(request.iov, request.iovcnt);
const size_t wire_len = AlignUp(n, kv::kAlignmentBytes);   // 512
if (wire_len > kKvClientMaxValueBytes) {
    results[i] = tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    continue;
}
auto slot = arena_.Acquire(wire_len);          // 按补零后的长度分配
if (!slot) { results[i] = tl::make_unexpected(slot.error()); continue; }

// 多段 iov 依次拷入连续 slot；尾部 [n, wire_len) 补零
size_t written = 0;
for (int j = 0; j < request.iovcnt; ++j) {
    std::memcpy(static_cast<char*>(slot->ptr) + written,
                request.iov[j].iov_base, request.iov[j].iov_len);
    written += request.iov[j].iov_len;
}
std::memset(static_cast<char*>(slot->ptr) + written, 0, wire_len - written);

entries.push_back(MakeEntry(cache_key, *slot, wire_len));  // length = wire_len
```

补零是**必需的**，不是可选优化：slot 复用时残留上一次 I/O 的数据，若不显式清零，尾部会把旧数据发到设备。同一 key 的后续读取只取前 N 字节，故残留不影响正确性，但会泄漏无关数据到存储侧——清零同时解决这两点。

**读路径**：

```cpp
// GetBatch: 后端传来的 ObjectGetRequest{logical_key, buffer, size = object_size}
const size_t n = request.size;                  // == desc.object_size
const size_t wire_len = AlignUp(n, kv::kAlignmentBytes);
auto slot = arena_.Acquire(wire_len);
// entry.length = wire_len，设备写入 wire_len 字节
...
// Wait 成功后：只把前 n 字节交还调用者，尾部 padding 丢弃
std::memcpy(request.buffer, slot->ptr, n);
arena_.Release(*slot);
return n;                                        // 返回 n，不是 wire_len
```

⚠️ **返回值必须是 `n` 而非 `wire_len`**。`DistributedStorageBackend::BatchRead` 会比对 `*object_results[i] != object_requests[i].size`（`:838-841`），而 `request.size` 是 `desc.object_size` = N。返回 `wire_len` 会让每次读取都误报 `FILE_READ_FAIL`。

**`aligned_size` 在读路径不直接使用**：后端传给 `GetBatch` 的 `size` 取 `desc.object_size`（`:817-818`），不取 `aligned_size`。故 `aligned_size` 目前只是元数据记录（供诊断与未来扩展），适配器自行用 `AlignUp(object_size, 512)` 重算线格式长度。这样即使 `aligned_size` 字段缺失或为 0（旧记录），读路径仍能工作。

**长度校验清单**（三道，缺一不可）：

| 层 | 校验 | 位置 |
| --- | --- | --- |
| Master | `value_length <= 16,776,704` | `PrepareWriteStart`（§3.6） |
| 适配器写 | `AlignUp(n,512) <= kKvClientMaxValueBytes` | `PutBatch` 逐请求 |
| 适配器读 | slot 容量 >= `AlignUp(n,512)`，返回长度 == n | `GetBatch` 逐请求 |

Master 那道必需，因为 `kMaxSliceSize`（16,777,200）比 SDK 上限宽 496 字节，且只在 CACHELIB 分配器下生效（`master_service.cpp:5287-5293`）。

## 3.6 shard-6：Master 集成

**文件**：`master_service.{h,cpp}`

**新增成员**：`bool use_kvclient_ = false;`（与 `use_kvcs_` 平级。谓词平级裁决要求 `UsesKvClient()` 与 `UsesKvcs()` 独立，不支持恢复裁决要求恢复门保持 KVCS-only，故不能合并为一个 flag）。

**改动点**：

| 位置 | 改动 |
| --- | --- |
| `master_service.cpp:847` | `enable_dfs_ = DfsEnablementConfig::FromEnvironment().enabled \|\| dfs_config.UsesKvcs() \|\| dfs_config.UsesKvClient();` |
| `:872` 之前 | 新增 `if (dfs_config.UsesKvClient()) { ... return; }` 分支 |
| `:5090` | `if (use_kvcs_ \|\| use_kvclient_) {` — descriptor 铸造。**KVClient 分支公式与 KVCS 不同**：`object_size = value_length`、`aligned_size = AlignUp(value_length, 512)`、`shard_idx = -1`（KVCS 分支 `:5092-5093` 两者都取 `value_length`，不改） |
| `:5249` | `(!use_kvcs_ && !use_kvclient_ && (!dfs_allocator_ \|\| ...))` |
| `:5248` 旁 | 新增 N 上限校验与三副本拒绝（见下）。**不校验 512 整除**——N 任意是合法输入，对齐由适配器补零承担 |
| `:6440` | `use_kvcs_ \|\| use_kvclient_` |
| `:6457` | `(use_kvcs_ \|\| use_kvclient_) && expected_replica_id != nullptr && ...` |
| **不改** `:3533`、`:3825`、`:4118` | 恢复门保持 `use_kvcs_` only（不支持恢复裁决） |

**`InitDfsAllocatorFromEnvironment` 新增分支**：

```cpp
if (dfs_config.UsesKvClient()) {
    if (enable_oplog_) {
        // Decision 6: no metadata recovery for the KVClient backend. The
        // restore gates at :3533/:3825/:4118 stay KVCS-only, so an HA
        // failover would reject every DFS descriptor. Fail at startup with a
        // clear message instead of at recovery time.
        LOG(FATAL) << "KVClient backend does not support metadata recovery; "
                      "enable_oplog must be false";
    }
    use_kvclient_ = true;
    // No EFC target, no capacity provider, no watermark eviction: the SDK has
    // no capacity query API. Master keeps the object directory only.
    LOG(INFO) << "KVClient distributed object storage initialized without DFS "
                 "extent allocator and without capacity provider, config={"
              << dfs_config.FormatStr() << "}";
    return;
}
```

**`PrepareWriteStart` 新增校验**（512 对齐裁决 + 两副本硬逻辑裁决）：

```cpp
// In PrepareWriteStart the length parameter is named slice_length
// (master_service.cpp:5225), not value_length.
if (config.dfs_replica_num > 0 && use_kvclient_) {
    // 512-alignment ruling: the wire format needs a 512B-aligned length, but N
    // itself may be arbitrary -- the adapter pads the staging slot, it does not
    // reject the object. The bound is on the padded length:
    //     AlignUp(slice_length, 512) <= 16'776'704
    // which is equivalent to slice_length <= 16'776'704 (the limit is itself a
    // multiple of 512). Mooncake's own kMaxSliceSize is 16'777'200, i.e. 496
    // bytes wider, and it is only enforced under the CACHELIB allocator
    // (master_service.cpp:5287-5293), so it will happily admit values that
    // cannot go on the wire.
    if (slice_length > 16'776'704) {
        LOG(ERROR) << "key=" << key << ", slice_length=" << slice_length
                   << ", error=kvclient_value_too_large";
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    // Two-replica ruling: only (memory + ASU) and DFS-only are supported.
    // With dfs > 0 the write mode is RELIABLE_MULTI_REPLICA, which would make
    // a NoF replica silently mandatory as well.
    if (config.nof_replica_num > 0) {
        LOG(ERROR) << "key=" << key << ", error=kvclient_rejects_nof_with_dfs";
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
}
```

放在 `PrepareWriteStart` 而非 `AllocateReplicas`，因为 Put 与 Upsert 都过这道门。

⚠️ **上限校验是必需的，不能省略**。`AlignUp(N,512) <= 16776704` 等价于 `N <= 16776704`（上限本身是 512 的倍数），但 Mooncake 自己的 `kMaxSliceSize = 16,777,200` 比它宽 496 字节：

```
16,776,704 = 512 × 32767   ← SDK 单 entry 上限
16,777,200 = kMaxSliceSize ← Mooncake 上限（types.h:469 + Slab.h:82,88）
16,777,216 = 512 × 32768   ← N = 16,777,000 补零后的线格式长度
```

N = 16,777,000 能通过 Master 的 slice 校验，补零后却是 16,777,216，超 SDK 上限。超限 entry 在 `PackSqe` 本地被拒，但预发送失败会 **abort 整个 transport task**，把所有 entryStatus 置 CANCELED（`trans_task_executor.cc:227-232`）——即一个越界对象会毒化整批。

⚠️ **Master 的 slice 校验还只在 CACHELIB 分配器下生效**（`master_service.cpp:5287-5293`），DFS-only 写 + 非 CACHELIB 时那道门根本不开。适配器侧须独立再校验一次。

**descriptor 铸造**（`:5090` 分支，KVClient 专属）：

```cpp
if (use_kvclient_) {
    DistributedFSDescriptor descriptor;
    descriptor.object_size = value_length;                    // N，原始长度
    descriptor.aligned_size = AlignUp(value_length, 512);     // 线格式长度
    descriptor.shard_idx = -1;
    replicas.emplace_back(std::move(descriptor), ReplicaStatus::PROCESSING);
}
```

与 KVCS 分支（`:5092-5093` 两者都取 `value_length`）的差异是本方案唯一需要改公式的地方，且落在 KVClient 专属分支内，**不影响 KVCS**。

`aligned_size` 只需满足 `>= object_size`（`IsDfsDescriptorRangeValid` 的 `:62`），不要求是 `config.alignment`（默认 4096）的倍数——因为对象存储分支在 `:664`/`:790` 提前 return，**完全跳过** `IsDfsDescriptorRangeValid`。故 512 对齐的 `aligned_size` 不会被该校验拒绝。

**Upsert 的门同样要放宽**：逻辑描述符分配只在 `use_kvcs_` 下（`:5090-5100`），`PrepareWriteStart` 同样要求（`:5248-5255`）。漏改则 KVClient 的 Upsert 直接 `DFS_SERVICE_UNAVAILABLE`。

## 3.7 shard-7：Client 集成

**文件**：`client_service.{h,cpp}`

**改动 1：`ReconcileMissingKvcsReplica` → `ReconcileMissingKvReplica`**（miss 清理裁决）

```cpp
void Client::ReconcileMissingKvReplica(
    const std::string& key, const QueryResult& query_result,
    const Replica::Descriptor& replica_descriptor) {
    if (!dfs_storage_backend_ ||
        !(dfs_storage_backend_->UsesKvcs() ||
          dfs_storage_backend_->UsesKvClient()) ||
        !replica_descriptor.is_dfs_replica()) {
        return;
    }
    // ... 其余不变（tenant 校验、EvictDiskReplicaIfCurrent、日志）
}
```

调用点 `:1626`、`:1936` 同步改名，声明 `client_service.h:1057` 同步。触发条件不变：仅 `OBJECT_NOT_FOUND`。

**改动 2：`BatchRemove` 的 KVClient 分支改 value-first**（value-first 删除裁决）

```cpp
std::vector<tl::expected<void, ErrorCode>> Client::BatchRemove(
    const std::vector<ObjectKey>& keys, bool force) {
    if (hot_cache_) hot_cache_->BumpKeyGenerations(keys);

    std::vector<tl::expected<void, ErrorCode>> results(keys.size());
    const bool kvclient =
        dfs_storage_backend_ && dfs_storage_backend_->UsesKvClient();

    std::vector<std::string> metadata_keys;
    std::vector<size_t> metadata_indices;
    metadata_keys.reserve(keys.size());
    metadata_indices.reserve(keys.size());

    if (kvclient) {
        // Value first. The KVClient backend has no orphan reclamation path at
        // all: no capacity API (so no watermark eviction), no key listing, and
        // the Master holds no driver. Deleting metadata first would leak the
        // value permanently, and a retry could never fix it because Remove on
        // a gone object returns OBJECT_NOT_FOUND and is skipped.
        const auto deleted = dfs_storage_backend_->BatchDeleteObjects(keys);
        if (deleted.size() != keys.size()) {
            for (size_t i = 0; i < keys.size(); ++i)
                results[i] = tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
            return results;
        }
        for (size_t i = 0; i < keys.size(); ++i) {
            // OBJECT_NOT_FOUND is idempotent success: the value is already
            // gone, which is the intended end state.
            if (deleted[i] ||
                deleted[i].error() == ErrorCode::OBJECT_NOT_FOUND) {
                metadata_keys.push_back(keys[i]);
                metadata_indices.push_back(i);
            } else {
                // Value and metadata both survive -> safe to retry.
                results[i] = tl::make_unexpected(deleted[i].error());
            }
        }
    } else {
        metadata_keys.assign(keys.begin(), keys.end());
        for (size_t i = 0; i < keys.size(); ++i) metadata_indices.push_back(i);
    }

    if (!metadata_keys.empty()) {
        const auto master_results =
            master_client_.BatchRemove(metadata_keys, force);
        for (size_t j = 0; j < metadata_indices.size(); ++j) {
            const size_t i = metadata_indices[j];
            results[i] = (j < master_results.size())
                             ? master_results[j]
                             : tl::make_unexpected(ErrorCode::RPC_FAIL);
        }
    }

    if (hot_cache_ && !metadata_keys.empty())
        hot_cache_->RemoveHotKeys(metadata_keys);
    return results;
}
```

`Remove` 单 key 门控放宽（`:3816`）：`UsesKvcs() || UsesKvClient()`。

**改动 3：`RemoveAll` / `RemoveByRegex` 门控放宽**（SFMEA-25，value-first 删除裁决 连带必改）

```cpp
// :3843 与 :3864
if (dfs_storage_backend_ &&
    (dfs_storage_backend_->UsesKvcs() ||
     dfs_storage_backend_->UsesKvClient()))
    return tl::unexpected(ErrorCode::NOT_SUPPORTED);
```

**必须在删元数据之前返回**，否则一次性泄漏整个 L3。

**改动 4：其余放宽点**

| 位置 | 改动 |
| --- | --- |
| `client_service.cpp:584` | `AttachHostId` 强制 `dfs_replica_num = 1`：`UsesKvcs() \|\| UsesKvClient()` |
| `real_client.cpp:1608` | 创建 `FileStorage` 并强制 `kDistributed` |
| `file_storage.cpp:126` | `enable_dfs` 判断与后端注入 |
| `file_storage_config.cpp:240` | 跳过本地目录校验 |
| `distributed_storage_backend.h:70-71` | 新增 `bool UsesKvClient() const { return UsesObjectStorage() && distributed_config_.UsesKvClient(); }` |
| `storage_backend.cpp:6506` 旁 | 工厂新增 `UsesKvClient()` 分支 |

**不改**：`DetermineFinalizeDecision`、`DetermineReplicaWriteMode`、`AllAllocatedTransfersSucceeded`、`HasExpectedReplicaAllocation`、`FinalizeBatchUpsert`——两副本硬逻辑裁决 的两副本硬逻辑现有代码已完全实现。

## 3.8 shard-8：构建与可选依赖

沿用 KVCS 的可选依赖模式（`src/CMakeLists.txt:179-197`）：

```cmake
set(MOONCAKE_KVCLIENT_SDK_FOUND OFF)
find_package(PkgConfig QUIET)
if(PkgConfig_FOUND)
  pkg_check_modules(KVCLIENT QUIET IMPORTED_TARGET "kvclient>=0.1.0")
endif()
if(TARGET PkgConfig::KVCLIENT)
  set(MOONCAKE_KVCLIENT_SDK_FOUND ON)
  message(STATUS "KVClient SDK found: kvclient backend enabled")
else()
  message(STATUS "KVClient SDK not found: kvclient backend builds with "
                 "unsupported stub")
endif()
set(MOONCAKE_KVCLIENT_SDK_FOUND ${MOONCAKE_KVCLIENT_SDK_FOUND} PARENT_SCOPE)
```

新增源文件（`MOONCAKE_STORE_SHARED_OBJECTS` / `mooncake_store`）：

```cmake
storage/distributed/kvclient/kvclient_object_storage_adapter.cpp
storage/distributed/kvclient/kvclient_staging_arena.cpp
```

条件编译与链接：

```cmake
if(MOONCAKE_KVCLIENT_SDK_FOUND)
  target_compile_definitions(mooncake_store_shared_objects
                             PRIVATE MOONCAKE_HAVE_KVCLIENT_SDK=1)
  target_link_libraries(mooncake_store_shared_objects
                        PRIVATE PkgConfig::KVCLIENT)
endif()
```

**`mooncake_store_master` 不链接 KVClient SDK**——不支持恢复裁决下 Master 无适配器实例、无 capacity provider，与 KVCS 需要链接（`:719-721`）不同。

SDK 缺失时工厂的 `UsesKvClient()` 分支返回 `NOT_SUPPORTED`（对照 `kvcs_capi_driver.cpp:404-407` 的 stub 模式），其他后端照常构建。

测试目标（`tests/CMakeLists.txt:416-424` 模式）：

```cmake
add_store_test(kvclient_object_storage_adapter_test
               kvclient_object_storage_adapter_test.cpp)
if(NOT MOONCAKE_KVCLIENT_SDK_FOUND)
  target_compile_definitions(kvclient_object_storage_adapter_test
                             PRIVATE MOONCAKE_KVCLIENT_TEST_NO_SDK=1)
endif()
```

---

# 4 开发自验证用例

层级：**单元** = fake driver / fake SDK，无需真机；**集成** = Mooncake 端到端（fake SDK）；**真机** = fake 无法覆盖的确认项。

| 编号 | 层级 | 用例 | 预期结果 | 覆盖 |
| --- | --- | --- | --- | --- |
| T-01 / T-02 | 单元 | `EncodeCacheKey` 幂等与单射：同 key 多次调用、10^5 个不同 key 抽样 | 逐字节相同 / 互不相同 | §3.1 纯函数约束 |
| T-03 | 单元 | 缩小 seed 空间强制构造碰撞，先写 k1 再写 k2 | k2 覆盖 k1，读 k1 得到 k2 的数据且**无错误返回** | SFMEA-01（验证碰撞后果符合「默许」定义） |
| T-04 / T-05 | 单元 | 空 logical key；同批两个相同 logical key | 前者 `INVALID_PARAMS`；后者第二个 `INVALID_PARAMS` | SFMEA-02/03 |
| T-06 / T-25 / T-25b / T-09b | 单元 | N=513 写读；A(513)→B(1024) 复用同一 slot 后再读 A | 成功，`entry.length == 1024`，`[0,513)` 为原始数据、`[513,1024)` 全 0；复用后 A 不含 B 残留 | SFMEA-04、512 对齐裁决（N 任意是合法输入，不是拒绝条件） |
| T-07 | 单元 | N = 16,776,704（上限，本身 512 对齐） | 成功，`entry.length == 16776704`，无补零 | §3.6 上限 |
| T-08 / T-08b | 单元 | N = 16,777,000（能过 `kMaxSliceSize`=16,777,200，但 `AlignUp` 后为 16,777,216 超上限）；N = 16,777,216 | Master `PrepareWriteStart` → `INVALID_PARAMS` 且适配器未被调用；绕过 Master 时适配器预检拒绝且**不提交 SDK** | SFMEA-05 双层防护（**关键**：Master 那道校验存在的理由） |
| T-09 | 单元 | N = 0 | `INVALID_PARAMS` | SFMEA-06 |
| T-09c / T-25c / T-27 | 单元 | `GetBatch` 对 N=513 对象的返回值；构造读回长度 != 请求长度 | 返回 **513**（不是 1024）；长度不符 → `FILE_READ_FAIL` | §3.5、SFMEA-33（返回 wire_len 会让每次读都误报） |
| T-10 / T-11 | 单元 | `sharedProviderMode = 0` 时 `Init()`；`RegisterRegions` 返回 PARTIAL_FAILED | 前者 `DFS_SERVICE_UNAVAILABLE` 且不进入 `RegisterRegions`；后者 `Init()` / `FileStorage::Init` 明确失败 | SFMEA-07/08 |
| T-12 | 单元 | arena 配 1 slot，并发 2 个请求 | 第 2 个立即 `NO_AVAILABLE_HANDLE`，**不阻塞** | SFMEA-09 |
| T-13 | 单元 | 篡改 entry 的 `buffer.handle` 为非 canonical 的非 0 值 | 提交前预检拒绝，不到 Wait | SFMEA-10 |
| T-14 | 单元 | fake SDK 让 `Wait` 返回 TIMEOUT 且 task 保持 in-flight | slot 进 quarantine：`InUseSlots()` 归零、`QuarantinedSlots()==1`、`QuarantinedBytes()` 增加、后续 `Acquire` 拿不到该 slot | SFMEA-12 |
| T-15 | 单元 | fake SDK 让 `Wait` 超时且 `entryStatus` 保持初值 `OK` | 逐项结果为 `RPC_TIMEOUT`，**绝不为 OK** | SFMEA-13（回归防护，必须有） |
| T-16 / T-17 | 单元 | fake SDK 让第 2 项 entryStatus 失败；让整个 task 返回 CANCELED | 前者按 `originalIndices` 回填原下标 2、其余成功；后者全部 slot quarantine 且逐项 `INTERNAL_ERROR` | SFMEA-14/15 |
| T-18 | 单元 | fake SDK 让 `BatchStoreAsync` 提交阶段失败 | slot 全部 **Release**（非 quarantine），`InUseSlots()` 归零 | SFMEA-16 |
| T-19 | 单元 | fake SDK 返回 `CQE_OUT_OF_CREATE_SIZE` | 映射 `NO_AVAILABLE_HANDLE` | SFMEA-17 |
| T-20 / T-21 | 单元 | fake SDK 读时返回 `ENTRY_KEY_NOT_FOUND`；读时返回 `TIMEOUT` | 前者 `OBJECT_NOT_FOUND`；后者 `RPC_TIMEOUT` 且**不触发** `ReconcileMissingKvReplica` | SFMEA-18 正向 / 反向 |
| T-22 | 单元 | 跨对象合批：3 个对象共 250 entry，批上限 110 | 切成 3 批（110/110/30），无对象被切开，结果下标正确 | §2.3.2 |
| T-23 | 单元 | `BatchDelete` 遇 `ENTRY_KEY_NOT_EXIST` | 视为幂等成功 | §3.3 |
| T-24 | 单元 | `Shutdown` 时仍有 in-flight | 先 Drain；若 `InUseSlots() != 0` 则 `LOG(ERROR)` 且输出 `QuarantinedBytes()` | SFMEA-32 |
| T-26 | 单元 | `Exists()` / `ListKeys()` / `GetRange()` | 均 `NOT_SUPPORTED` | §1.2.2 |
| T-28 | 单元 | 含 NUL 与全 0 字节的 logical key | 正常产出 8 字节 CacheKey，删除与查询路径不被截断 | §3.1（hash 编码天然二进制安全） |
| T-29 | 集成 | 两副本 Put 成功 → 淘汰 memory → 另一 Client Get | 读到原始数据，逐字节相同 | S1 |
| T-30 | 集成 | DFS-only Put / Get / Remove | 全流程成功 | S2 |
| T-31 / T-32 | 集成 | 注入 ASU 写失败；注入 memory 写失败 | 前者 Put 失败且 Master 侧查不到该对象（memory 副本被一起 revoke）；后者适配器侧提交计数为 0（ASU 零调用）、Put 失败 | 两副本硬逻辑裁决（含短路行为） |
| T-33 | 集成 | 配置 `memory + nof + ASU` 后 Put | `PrepareWriteStart` 返回 `INVALID_PARAMS` | SFMEA-28 |
| T-34 / T-35 | 集成 | 同尺寸对象 Upsert；仅 MEMORY 的对象做 Upsert 请求 dfs=1 | 前者原地复用、ReplicaID 不变、同 CacheKey 覆盖写并读到新值；后者失败（拓扑不符），需先 Remove 再 Put | §2.2.4 |
| T-36 / T-37 | 集成 | Remove：value 删除成功 + Master `BatchRemove` 失败；对象处于 lease 中 | 前者元数据保留 → 后续 Get `NOT_FOUND` → `ReconcileMissingKvReplica` 清理 DFS 记录；后者 `CheckRemovable` 拒绝且 Get 返回 `NOT_FOUND` 而非错数据，最终经 miss 自愈清理 | SFMEA-20/21（value-first + miss 清理联动） |
| T-38 / T-39 | 集成 | Remove：value 删除失败；`BatchRemove` 部分 key 失败 | 前者元数据**保留**、重试 Remove 可成功；后者成功项元数据被删、失败项两者均保留、结果与原下标对应 | SFMEA-22、S6 |
| T-40 | 集成 | `RemoveAll()` / `RemoveByRegex()` | 返回 `NOT_SUPPORTED`，**且 Master 侧元数据完全未变** | SFMEA-25（关键：不能只删元数据） |
| T-41 | 集成 | Client 观察到 R1 miss，Master 已发布 R2 | 条件清理返回 false，R2 原样保留且可读 | SFMEA-30 |
| T-42 / T-43 | 集成 | `enable_oplog=true` + KVClient 启动 Master；默认配置（无 HA）重启 Master | 前者 `LOG(FATAL)`，启动失败且错误信息明确不支持恢复；后者目录全丢、Get `OBJECT_NOT_FOUND`、无崩溃无脏数据 | SFMEA-27/26、不支持恢复裁决 |
| T-44 | 集成 | 范围读（窗口 < object_size） | `INVALID_PARAMS`，不触达适配器 | SFMEA-34 |
| T-45 | 集成 | `PutEnd` 响应未知 | Client 返回提交错误；重复 Put 或查询可确认状态 | SFMEA-29 |
| T-46 | 集成 | 无 KVClient SDK 时构建 | 全部目标构建成功；显式初始化 KVClient 后端返回 `NOT_SUPPORTED`；KVCS/oss/s3 后端功能不受影响 | §3.8 |
| T-47 | 集成 | 既有 KVCS 回归 | `kvcs_object_storage_adapter_test`、`object_storage_adapter_test`、`distributed_storage_config_test` 全部通过 | 平级架构下 **KVCS 代码零改动**；失败即说明改动越界 |
| T-48 | 真机 | ASU `Store` 是否支持**覆盖写**：同 CacheKey 连续两次 Store 不同 value | 读回应为第二次 | **阻塞**：决定 `PutBatch` 是否需先 Delete（KVCS 因 insert-only 必须这么做）。不支持则 Upsert 与同 key 重写都会失败 |
| T-49 | 真机 | `Wait` 超时后设备是否仍访问 buffer | 超时后立即用哨兵值覆写 staging slot，检查是否被设备改写 | **阻塞**：验证 quarantine 的必要性边界 |
| T-50 / T-51 | 真机 | 设备侧 `singleValueMaxBytes` 实际值；多节点部署下跨节点合批路由 | 读 SDK `Init` 日志（只打日志、不 clamp entry 长度）；`originalIndices` 回填正确 | 高：若上限 < 16,776,704 则实际上限更低；SDK 自动拆分未经真机验证 |
| T-52 / T-53 | 真机 | 非 512 对齐的 length / `buffer_addr` 直接提交 | 观察设备是否拒绝或容忍 | 中：防御性确认「补零确属必要」而非过度设计 |
| T-54 | 真机 | 推荐 I/O 粒度：分层测量 `encode/acquire/memcpy/submit/wait/release`（§2.5.2） | 得出推荐 I/O 粒度 | 低：性能调优，不阻塞功能 |

## 4.1 验收标准

- T-01 ~ T-47 全部通过（fake SDK / 集成环境）。
- T-48、T-49、T-50、T-51 真机验证通过后方可进入生产部署。
- `kvclient_staging_quarantined_bytes` 在 24 小时压测后为 0（无超时泄漏）。
- 既有 KVCS / oss / s3 / hf3fs / posix 后端回归全部通过（T-47）。

---

# 5 已知限制

| 限制 | 说明 | 缓解 |
| --- | --- | --- |
| **key 碰撞静默损坏** | XXH64 单 hash，无 fingerprint（key 碰撞裁决）。碰撞概率 ≈ n²/2^65：10^7 key → 2.7e-6，10^9 → 2.7% | 仅能靠 §2.5.1 的 `cache_key` 日志事后排查。**部署须限定 key 数量级** |
| 删除孤儿窗口 | value-first 下 `CheckRemovable` 拒绝会留下不可读的已发布对象 | 语义安全（不返回错数据）；靠读取 miss 自愈 |
| 无容量管理 | SDK 无容量查询 API → Master 水位淘汰不生效 | 沿用 `:883-889` 降级；容量控制留待 SDK 提供接口 |
| 不支持元数据恢复 | Master 重启即目录全丢，L3 数据不可发现 | 启动期显式拒绝 `enable_oplog`；运维需知晓 |
| 不支持 key 枚举 | `ListKeys` / `RemoveAll` / `RemoveByRegex` 返回 `NOT_SUPPORTED` | 同 key 重写时覆盖 |
| 不支持范围读 | `BatchRead` 对象分支忽略 `range_read`（`:802-818`） | 上层回退整对象读后截取 |
| 适配器内 memcpy + 补零 | §2.3.2 的同步契约强制 staging；512 对齐裁决又要求尾部补零 | 零拷贝需 SDK 提供真正的设备级中止能力 |
| **小对象补零放大** | N < 512 时线格式长度仍是 512，放大最高 512 倍（N=1）；带宽与存储双浪费 | 上层聚合小对象；用 `kvclient_adapter_bytes_total{direction}` 观测实际放大率（§2.5.3） |
| 单对象上限 16,776,704 B | SDK 线格式 24-bit length + 512 对齐。比 Mooncake 的 `kMaxSliceSize`（16,777,200）窄 496 字节 | `PrepareWriteStart` 显式校验（§3.6）；无其他缓解 |
| Upsert 无法为仅 MEMORY 对象补 DFS 副本 | `master_service.cpp:6146-6181` 拓扑不符 | 先 Remove 再 Put |

---

# 6 代码依据

Mooncake `77689203`；KVClient SDK `e6717d04`。

| 引用 | 路径 |
| --- | --- |
| 适配器接口 | `mooncake-store/include/storage/distributed/object_storage_adapter.h` |
| KVCS 适配器 | `mooncake-store/src/storage/distributed/kvcs/kvcs_object_storage_adapter.cpp` |
| driver 抽象 | `mooncake-store/include/storage/distributed/kvcs/kvcs_driver.h` |
| KVCS driver 实现 | `mooncake-store/src/storage/distributed/kvcs/kvcs_capi_driver.cpp` |
| KVCS 容量 provider | `mooncake-store/src/storage/distributed/kvcs/kvcs_capacity_provider.cpp` |
| 后端 | `mooncake-store/src/storage/distributed/distributed_storage_backend.cpp` |
| 后端配置 | `mooncake-store/include/config/distributed_storage_config.h`、`src/config/distributed_storage_config.cpp` |
| Client | `mooncake-store/src/client_service.cpp` |
| Master | `mooncake-store/src/master_service.cpp`、`src/master.cpp` |
| 副本与写模式 | `mooncake-store/include/replica.h` |
| Master 配置 | `mooncake-store/include/master_config.h` |
| 构建 | `mooncake-store/src/CMakeLists.txt`、`tests/CMakeLists.txt` |
| SDK 接口 / 类型 | `kv_semantics/include/kv_client.h`、`kv_types.h` |
| SDK 客户端实现 | `kv_semantics/src/client/kv_client_impl.cc` |
| SDK 任务管理 | `kv_semantics/src/client/task/task_manager.cc` |
| SDK 传输 | `kv_semantics/src/trans/task/trans_task_executor.cc`、`src/trans/task/io_scheduler.cc`、`src/trans/kv_transport_impl.cc`、`src/trans/conn/connection_manager.cc` |
| SDK provider | `kv_semantics/src/trans_provider/aicpu/aicpu_trans_provider.cc`、`src/trans_provider/aiv/aiv_trans_provider.h` |
