# Mooncake + KVClient L3 接入设计方案

版本：v3.3　更新日期：2026-11-13

---

# 1 概述

## 1.1 需求分析

### 1.1.1 背景

L3 当前的 `DistributedStorageBackend` 已支持 `hf3fs` / `posix` / `oss` / `s3` / `kvcs-lowlevel` 五种 `fs_adapter_type`。其中对象存储类后端（oss / s3 / kvcs-lowlevel）已完全适配器抽象化：数据面只依赖 `UsesObjectStorage()`，通过 `ObjectStorageAdapter` 接口按逻辑 key 读写，Master 只维护「key → 逻辑 DFS 副本」的目录，不做字节级 offset 分配。

现需将**远端 KV 存储**（经 KVClient SDK，Host RDMA 直连）接入为共享 L3。

### 1.1.2 注意事项

| 内容 | 落地位置 |
| --- | --- |
| key 映射：XXH64 单 seed → 8B `kv::BinaryKey`，再包成 `kv::CacheKey::FromBytes(...)`；value 就是原始 N 字节。`MakeCacheKey` 作为 `KvClientObjectStorageAdapter` 的私有方法 | §3.1（§3.4） |
| 数据面按 512B 对齐：注册区间、对象槽位、线格式长度全部 512B 对齐（对象在 buffer 中占 `AlignUp(N,512)`，写路径尾部补零、读路径尾部忽略）。 | §3.2.3（§3.4） |
| SDK 侧 string key 哈希（`CacheKey::FromEngineKey`）**为预留能力，本方案不使用**。key 由 Mooncake 侧 XXH64 算好后以 `FromBytes` 传入。 | §3.1（SFMEA-26、SFMEA-39） |
| 只增加 `UsesKvClient()`，与 `UsesKvcs()` **平级**；不引入 `UsesManagedKvStorage()`。 | §3.3（§3.5、§3.6） |
| 双副本（memory + ASU）硬逻辑：ASU 写失败 → 整个 Put 失败 + memory 副本一起 revoke。 | §3.5 |
| **不支持**从 snapshot / OpLog 恢复对象目录。 | §3.5 |
| `ReconcileMissingKvcsReplica()` 推广为 `ReconcileMissingKvReplica()`； | §3.6（§3.5） |
| 删除顺序：**先删 value，再删元数据**。连带必改 `RemoveAll` / `RemoveByRegex` 门控，且依赖 miss 自愈兜底中间态 | §3.6 |

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
        K["KvClientObjectStorageAdapter<br/>key 编码、持久注册表、零拷贝读写<br/>提交与完成等待、结果转换"]
        SDK["KVClient<br/>CacheKey 路由、Host RDMA"]

        A <-->|"Put / Get / Remove"| R
        R <--> C
        C -.-> RPC
        C <-->|"MEMORY 副本"| TE
        C <-->|"读写：key + 描述符 + CPU slices<br/>删除：key"| B
        B <-->|"PutBatch / GetBatch / DeleteBatch"| O
        O <--> K
        K <-->|"内存注册 / 批量提交 / 完成等待"| SDK
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
| `DistributedStorageBackend` | slices → 对象请求转换、长度校验 | 仅新增 `UsesKvClient()` 组合判断 |
| `ObjectStorageAdapter` | 对象存储统一契约 | **零改动**（接口不变） |
| `KvcsObjectStorageAdapter` | KVCS 后端实现 | **零改动**（与 KVClient 平级，互不影响） |
| `KvClientObjectStorageAdapter` | KVClient 后端实现：key 编码、原 buffer 注册与注册表、零拷贝读写、提交与完成等待 | **新增**；注册与读写都在适配器内完成（§3.2.3） |

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
        +RegisterLocalMemory(addr, len, ...)
        +unregisterLocalMemory(addr)
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
        -persistentRegions_
        -persistentRegionsMu_
        +Init()
        +PutV(key, iov, iovcnt)
        +PutBatch(requests)
        +GetBatch(requests)
        +DeleteBatch(keys)
        -MakeCacheKey(logical_key)
        -PrepareKvBuffer(iov, object_size, is_store)
        -FindPersistentHandle(addr, len)
        -AddPersistentRegion(base, size, handle)
        -IsUnsafe(region, addr, len)
        -MarkUnsafe(region, addr, len)
        -SubmitAndWait(is_store, entries, plans, results)
    }

    class SdkKvClient["kv::KvClient"] {
        +LoadFromFile(path, config)$
        +Init(config)
        +StoreAsync(entries, task_id, options)
        +LoadAsync(entries, task_id, options)
        +DeleteAsync(keys, task_id, options)
        +QueryAsync(keys, task_id, options)
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
    style DistributedStorageBackend fill:#f3f4f6,stroke:#6b7280,color:#111827
    style ObjectStorageAdapter fill:#f3f4f6,stroke:#6b7280,color:#111827
    style SdkKvClient fill:#f3f4f6,stroke:#6b7280,color:#111827
```

| 类或函数 | 接口与改动说明 |
| --- | --- |
| `DistributedStorageConfig` | `UsesObjectStorage()` 列表加入 `"kvclient"`；新增 `UsesKvClient()`，与 `UsesKvcs()` 平级；`Validate()` 增加 KVClient 专属分支 |
| `CreateStorageBackend(config)` | 修改现有工厂函数，在 distributed 分支新增 `UsesKvClient()` case 创建新适配器 |
| `FileStorage` | 修改构造时的 DFS 启用判断，将 KVClient 后端注入 Client；沿用现有初始化入口 |
| `Client` | `ReconcileMissingKvcsReplica()` 推广为图中的 `ReconcileMissingKvReplica()`；**内存注册不经过 Client** |
| `DistributedStorageBackend` | 读写路径已由 `UsesObjectStorage()` 抽象化；仅新增 `UsesKvClient()` 组合判断，不参与注册 |
| `ObjectStorageAdapter` | **零改动**。同步契约（payload 须在返回前保持有效）是零拷贝路径成立的前提 |
| `kv::KvClient` | **SDK 不改**。只用 `StoreAsync` / `LoadAsync` / `DeleteAsync` / `QueryAsync` 四个公共方法——SDK 不暴露 `Batch*Async`；`StoreAsync` / `LoadAsync` 按 `entries.size() > 1 \|\| useBatchIoForSingleKey`（默认 true，故单条也走 `BATCH_*`）内部选择 opcode，适配器无需关心。四个方法均带 `const kv::IoOptions& options = {}`；本方案不填 `kvns`、不用 `prerequisite`（见 §3.2.1） |

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
        O["KvClientObjectStorageAdapter::PutBatch<br/>MakeCacheKey + 补零到 aligned_size"]
        D["⑦ PrepareKvBuffer → 注册/查表 → 清零尾部<br/>StoreAsync → Wait → 复用或标记 unsafe"]
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
        D["⑥ KvClientObjectStorageAdapter::GetBatch<br/>PrepareKvBuffer → LoadAsync → Wait<br/>设备直接写入原目标 buffer"]
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
        D["③ KvClientObjectStorageAdapter::DeleteBatch<br/>MakeCacheKey → DeleteAsync → Wait"]
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

初始化在 `CreateStorageBackend` 工厂内创建适配器，随后由后端 `Init()` 触发 SDK 连接。**`Init` 不注册任何内存**：原 buffer 由首次 I/O 惰性注册并常驻。

```mermaid
sequenceDiagram
    autonumber
    participant RC as RealClient
    participant FS as FileStorage
    participant F as CreateStorageBackend
    participant BE as DistributedStorageBackend
    participant AD as KvClientObjectStorageAdapter
    participant SDK as kv::KvClient

    RC->>FS: 构造（UsesKvClient 时强制 kDistributed）
    FS->>F: CreateStorageBackend(config)
    F->>F: DistributedStorageConfig::Validate()<br/>single_tenant / 无 eviction_check_interval 要求
    F->>AD: make_unique<KvClientObjectStorageAdapter>(config)
    F->>BE: DistributedStorageBackend(cfg, distributed_cfg, nullptr, adapter)
    BE->>AD: Init()
    AD->>SDK: CreateKvClient()
    AD->>SDK: KvClientConfig::LoadFromFile(configPath, cfg)
    AD->>AD: cfg.keyResolution = {}<br/>强制 GENERIC，不继承推理引擎配置
    AD->>AD: 校验 cfg.timeoutMs >= wait_timeout_ms<br/>transport deadline 不得短于 Wait 超时
    AD->>SDK: Init(cfg)
    SDK-->>AD: Status
    AD->>AD: CheckSharedProviderMode()<br/>sharedProviderMode == SHARED ?
    AD-->>BE: OK
    BE-->>FS: backend
    FS->>RC: Client::SetDfsStorageBackend(backend)
```

**注册表初始为空**：适配器内的注册表不参与 `Init`，由适配器在首次 I/O 时注册并**常驻到关闭**（同步 ASUStore 的 `persistentRegions_` 模型，不淘汰）。首次请求因此多一次 `RegisterRegions` 开销；注册失败则该请求**直接失败**，不回退 staging（§3.2.3）。

### 2.3.2 写入阶段

写路径只有一条：`PrepareKvBuffer` 先校验长度、连续性与槽位对齐，再按地址包含关系查注册表（未命中则注册一次并常驻），然后把调用方地址直接填进 `KVBuffer`，只清零尾部 `[N, wire_len)`，不拷贝 payload。**不连续、槽位未 512B 对齐、区间已标记 unsafe 或注册失败一律返回错误，不回退 staging。** 随后按对象边界切批，一次性提交 `StoreAsync`，再 `Wait`；task 终态为 `OK` / `PARTIAL_FAILED` 时逐条映射 `entryStatus`，`TIMEOUT` / `CANCELED` / `IN_PROGRESS` 的条目把该区间标记为 unsafe。

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant BE as DistributedStorageBackend
    participant AD as KvClientObjectStorageAdapter
    participant SDK as kv::KvClient
    participant S as 远端存储

    C->>BE: BatchWrite(key + descriptor + slices)
    BE->>BE: 校验 total_size == desc.object_size<br/>每 slice 转 iovec（不合并）
    BE->>AD: PutBatch(ObjectPutRequest{key, iov, iovcnt})
    loop 每个请求
        AD->>AD: MakeCacheKey(logical_key) → BinaryKey<br/>CacheKey::FromBytes(...)<br/>校验 iov 非空、同批重复 key 拒绝
        AD->>AD: n = Σiov_len；wire_len = AlignUp(n,512)<br/>校验 wire_len <= 16776704
        AD->>AD: PrepareKvBuffer：单段连续？槽位 512B 对齐？<br/>FindPersistentHandle（未命中则 RegisterRegions 后常驻）
        alt 全部满足
            AD->>AD: region.inflight++<br/>memset 尾部 [n, wire_len) 补零<br/>KVBuffer{key, addr, wire_len, handle, offset=0}
        else 不连续 / 未对齐 / 已标记 unsafe / 注册失败
            AD->>AD: results[i] = INVALID_PARAMS 或 DFS_SERVICE_UNAVAILABLE
        end
    end
    AD->>AD: 按 110 切批（不切开单个对象）<br/>记录 originalIndices
    AD->>SDK: StoreAsync(entries, taskId)
    SDK->>SDK: 按 nodeId 路由拆分 transport task
    SDK->>S: Host RDMA write（buffer.region.size = wire_len）
    SDK-->>AD: 提交结果
    alt 提交失败（含 key 解析失败，整批同步失败）
        AD->>AD: inflight--（未触达设备）
        AD-->>BE: 逐项错误
    else 已受理
        AD->>SDK: Wait(taskId, timeoutMs, result)
        alt task status = OK / PARTIAL_FAILED（终态，entryStatus 逐条可信）
            SDK-->>AD: TaskResult{status, entryStatus[]}
            AD->>AD: 逐项映射 entryStatus → ErrorCode<br/>PARTIAL_FAILED 不整批置错
            AD->>AD: 逐条处置（看 entry，不看 task）<br/>TIMEOUT/CANCELED/IN_PROGRESS → MarkUnsafe(range)<br/>其余 → inflight--
        else TIMEOUT / CANCELED / IN_PROGRESS
            Note over AD: ⚠️ 禁止读 entryStatus（初值 OK，假成功）
            AD->>AD: MarkUnsafe(全部区间)；逐项置 RPC_TIMEOUT / INTERNAL_ERROR
        end
        AD-->>BE: 逐项 expected
    end
    BE-->>C: results[index]
    C->>C: AllAllocatedTransfersSucceeded?<br/>→ PutEnd(ALL) 或 PutRevoke(ALL)
```

### 2.3.3 读取阶段

读路径与写路径对称：设备直接写入调用方 buffer 的 512B 槽位（容量 `AlignUp(N,512)`，尾部写在同一槽位内，调用方只取前 N 字节）。不连续、未对齐、已标记 unsafe 或注册失败一律返回错误。明确 `NOT_FOUND` 触发 miss 清理；超时则把该区间标记为 unsafe，且**不**触发清理。

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant BE as DistributedStorageBackend
    participant AD as KvClientObjectStorageAdapter
    participant SDK as kv::KvClient
    participant S as 远端存储

    C->>C: GetReplicaList → descriptor + lease
    C->>BE: BatchRead(key + descriptor + slices)
    BE->>BE: 校验 slice 容量 >= object_size<br/>size = desc.object_size = N
    BE->>AD: GetBatch(ObjectGetRequest{key, buffer, N})
    loop 每个请求
        AD->>AD: MakeCacheKey(logical_key) → BinaryKey<br/>CacheKey::FromBytes(...)<br/>wire_len = AlignUp(N,512)
        AD->>AD: PrepareKvBuffer：单段连续？槽位 512B 对齐？<br/>FindPersistentHandle（未命中则 RegisterRegions 后常驻）
        alt 全部满足
            AD->>AD: region.inflight++<br/>KVBuffer{key, 调用方 addr, wire_len, handle, offset=0}
        else 不连续 / 未对齐 / 已标记 unsafe / 注册失败
            AD->>AD: results[i] = INVALID_PARAMS 或 DFS_SERVICE_UNAVAILABLE
        end
    end
    AD->>SDK: LoadAsync(entries, taskId)<br/>buffer.region.size = wire_len
    SDK->>S: Host RDMA read
    AD->>SDK: Wait(taskId, timeoutMs, result)
    alt task status = OK / PARTIAL_FAILED（终态，entryStatus 逐条可信）
        SDK-->>AD: entryStatus[]（逐 key）
        alt entryStatus = OK
            AD->>AD: 设备已直接写入调用方 buffer；inflight--
            AD-->>BE: expected<size_t>(**N**)
            BE->>BE: 校验 *result == request.size(=N)<br/>（双层：BE :838-841 + 适配器）
        else entryStatus = ENTRY_KEY_NOT_FOUND
            AD->>AD: inflight--（不标记）
            AD-->>C: 逐 key OBJECT_NOT_FOUND
            C->>C: ReconcileMissingKvReplica(key, ReplicaID)
        else entryStatus = TIMEOUT / CANCELED
            AD->>AD: MarkUnsafe(range)（设备可能仍在写）；inflight--
            AD-->>C: RPC_TIMEOUT / INTERNAL_ERROR
            Note over C: 元数据保留，不触发清理
        end
    else TIMEOUT / CANCELED / IN_PROGRESS
        Note over AD: 禁止读 entryStatus
        AD->>AD: MarkUnsafe(全部区间)
        AD-->>C: RPC_TIMEOUT / INTERNAL_ERROR
        Note over C: 元数据保留，不触发清理
    end
```

### 2.3.4 删除阶段

删除只传 key、不传 buffer，因此不涉及注册与 buffer 处置。value 删除成功（含幂等 `NOT_FOUND`）的 key 才进入元数据删除集合；失败项两者同时保留，可安全重试。

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
    AD->>AD: MakeCacheKey → CacheKey[]
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

关闭必须先排空在途操作再注销全部注册区间。被标记 unsafe 的区间表示「设备可能仍可访问」，区间在途计数非零即为泄漏信号，需显式告警。

```mermaid
sequenceDiagram
    autonumber
    participant AD as KvClientObjectStorageAdapter
    participant SDK as kv::KvClient

    Note over AD: 析构 / Shutdown
    AD->>AD: 停止接受新提交
    AD->>SDK: Drain / 等待 in-flight task 结束
    AD->>AD: 检查全部区间 inflight == 0 ?
    alt 仍有占用
        AD->>AD: LOG(ERROR) 泄漏告警<br/>列出 unsafe 标记字节数
    else 已排空
        AD->>AD: 继续
    end
    AD->>SDK: UnregisterRegions(注册表中全部区间 handle)
    AD->>SDK: Shutdown()
```

**说明**：注册区间按 handle 逐个注销，注册与注销都由适配器负责，上层无感知。`UnsafeBytes()` 非零表示有区间因超时被降级，需在关闭日志中显式输出。

## 2.4 SFMEA 分析

只列会改变设计或需要专门用例的失效模式；等价分支合并到同一编号行，纯既有语义、范围外与不可达项已并入相关条目或删除。

| 编号 | 故障模式 | 故障影响 | 容错措施 | 故障注入 / 备注 |
| --- | --- | --- | --- | --- |
| SFMEA-01 | logical key XXH64 碰撞：两个不同 key 映射到同一 CacheKey | **数据损坏**：后写覆盖先写，读到错 value。默认（`MOONCAKE_STORE_CHECKSUM` 关闭）无从发现；开启后 Get 端以 `CHECKSUM_MISMATCH` 暴露，但 Put 侧仍是覆盖写、不报错 | 不做 fingerprint（默许碰撞）；可复用既有 object checksum 作为检测手段；部署文档须写明 key 数量级上限 | 缩小 seed 空间强制构造碰撞。概率 ≈ n²/2^65：10^8 key → 2.7e-4，10^9 → 2.7% |
| SFMEA-02/03 | 非法 logical key：为空，或同批内重复 | 空 key 无法编码；同批重复会让同一 CacheKey 在一个 task 内冲突，SDK 行为未定义 | `MakeCacheKey` 拒空（`INVALID_PARAMS`）；写侧拒绝同批重复，读/查/删去重后 fan-out | 去重需在新适配器内自行实现（可参照 `kvcs_object_storage_adapter.cpp:144-147`） |
| SFMEA-04 | 未按 `AlignUp(N,512)` 补零：直接把 N 当作线格式长度，且未清零尾部 | 提交被拒或行为未定义；槽位里上一次 I/O 的残留字节会被发到存储侧 | `wire_len = AlignUp(n, 512)`；写路径显式 `memset` 尾部 `[n, wire_len)` | 写 513 字节对象，断言 `buffer.region.size == 1024` 且尾部全 0。N 非 512 倍数是**合法输入**，故障在补零遗漏 |
| SFMEA-05/06 | 长度非法：N > 16,776,704，或 N == 0 | 超限会被 `PackSqe` 本地拒绝，预发送失败 **abort 整个 transport task**（`trans_task_executor.cc:227-232`）；N == 0 不被 SDK 接受 | 适配器逐请求预检 `wire_len <= kKvClientMaxValueBytes` 且 `n > 0`；Master `PrepareWriteStart` 第一道校验 | 不能指望「只失败超限那条」。Master 那道必需：`kMaxSliceSize`（16,777,200）更宽且只在 CACHELIB 下生效 |
| SFMEA-07/08 | 初始化门禁失败：`sharedProviderMode != SHARED`，或 SDK `Init` 失败 | 前者在 INDEPENDENT 下 SDK 会为每个 ASU 节点 `BindMemory` 全部已注册区域，而 AICPU 的 `BindMemory` 只接受 `MEM_DEVICE` → **非空 view 下注册必然 PARTIAL_FAILED**；后者适配器不可用 | `Init()` 中 `CheckSharedProviderMode` 硬门禁（缺省或为 0 直接 `DFS_SERVICE_UNAVAILABLE`）；SDK `Init` 失败则适配器 `Init` 失败 | 必须在 Init 阶段失败而非运行期 |
| SFMEA-10/11 | 提交前 buffer 元数据不合法：`buffer.handle` 为空或与注册表不符，调用方自填 `mrKey`，或区间未注册 | 非 0 但未注册的 handle 在提交阶段**不报错**，到 Wait 才以 `BUFFER_NOT_REGISTERED` 暴露且**连坐整 task**；`mrKey` 由 SDK 回填，自填值无效 | handle 一律取自适配器内的注册表，提交前预检非空且属于本适配器；未注册区间直接在建 plan 阶段失败；`mrKey` 一律留 `std::nullopt` | 注册经 SDK 完成，不接受调用方传入的 handle |
| SFMEA-12 | `Wait` 超时，设备仍在访问调用方 buffer | 缓冲区被复用 → 设备写入新数据 = **数据损坏** | 该区间标记为 unsafe，后续重叠请求**直接失败**；unsafe 标记常驻到进程重启（注册不淘汰） | **承重**。SDK 无面向业务的中止能力：`Cancel` 只由 `Shutdown` 调用，且只放弃任务、不停设备（`kv_transport_impl.cc:196-201`） |
| SFMEA-13 | 超时后读 `entryStatus` | 初值是 `Status::OK()`，`BuildResult` 在 `!Done()` 时原样返回 → **假成功**，数据未落盘却报成功 | 仅 `OK` / `PARTIAL_FAILED` 两个可信终态下映射 `entryStatus`；`TIMEOUT` / `CANCELED` / `IN_PROGRESS` 或本地 `Wait` 非 OK 时**不读**，整批置错 | 必须有专门用例覆盖，否则极易回归 |
| SFMEA-14 | task 级 `PARTIAL_FAILED`，逐项 entryStatus 部分失败 | 部分 key 成功部分失败 | 逐项映射 entryStatus → ErrorCode 并按 `originalIndices` 回填；区间按 **entry status** 处置：业务错误 → `inflight--`，`TIMEOUT` / `CANCELED` / `IN_PROGRESS` → `MarkUnsafe` | entryStatus 与输入严格一一对应；task 终态**不等于**可安全复用（SFMEA-40） |
| SFMEA-15/16 | 派发阶段失败：已派发后 `CANCELED`，或路由失败（`routed asu transport not found`） | `CANCELED` 时设备可能已接触 buffer；路由失败时未触达设备 | `CANCELED` → `MarkUnsafe`；路由失败 → `inflight--` | 两者都发生在提交阶段，但 buffer 安全性相反，**不能合并处置** |
| SFMEA-17/19/31 | 设备侧资源/连接错误：`CQE_OUT_OF_CREATE_SIZE`、`CONNECTION_ERROR`、存储空间耗尽 | 写入或读取失败；无容量查询与自动回收 | 分别映射为 `NO_AVAILABLE_HANDLE` / `DFS_SERVICE_UNAVAILABLE`；Master 不建 capacity provider | 无容量接口 → 只能事后失败；容量管理留待 SDK |
| SFMEA-18 | `CQE_KEY_NOT_EXIST` / `ENTRY_KEY_NOT_FOUND`（读时） | 对象在存储侧不存在 | 映射为 `OBJECT_NOT_FOUND` → 触发 `ReconcileMissingKvReplica` 条件清理 | **必须与超时严格区分**：超时**不**触发清理 |
| SFMEA-20/21/24 | 删除中间态：value 已删而元数据残留（Master `BatchRemove` 失败 / `CheckRemovable` 拒绝 / Client 崩溃） | 已发布对象不可读；`CheckRemovable` 拒绝时对象仍受租约保护，元数据不会被判为失效 | 后续读取返回 `NOT_FOUND` → miss 自愈清理 DFS 记录；无 pending-delete 持久化队列 | 已知窗口：语义安全（不返回错数据），但会丢已成功 Put 的数据。value-first 依赖 miss 清理，二者必须同时落地 |
| SFMEA-22 | 删除阶段：value 删除失败（超时 / 不可达） | value 与元数据**同时保留** | 该 key 不进元数据删除集合 → **无孤儿**；返回错误供上层重试 | value-first 的核心收益：失败态可安全重试 |
| SFMEA-23 | 迟到 Delete 误删新 value（同 key 重写） | 旧删除请求命中新写入的 value | 无专门机制（XXH64 下新旧 value 同 CacheKey）；调用方须串行组织同 key 的删除/重建/更新 | 调用方契约，非本方案引入 |
| SFMEA-25 | `RemoveAll` / `RemoveByRegex` 被调用 | 若只删元数据不删 value → **一次性泄漏整个 L3** | 门控放宽到 KVClient，返回 `NOT_SUPPORTED` 且**不删元数据** | `client_service.cpp:3843`、`:3864`，value-first 删除的连带必改项 |
| SFMEA-26/27 | 不支持元数据恢复：Master 重启（默认无 HA），或误配 `enable_oplog=true` | 重启后目录全丢、L3 value 不可发现；误配时恢复门会在 HA 切换时拒绝所有 DFS 描述符 | 检测到 `enable_oplog` 即启动期 `LOG(FATAL)`；恢复门保持 `use_kvcs_` only，**不改** | 把运行期失效提前到启动期；运维须知晓无恢复能力 |
| SFMEA-28 | 三副本 `memory + nof + ASU` | `dfs > 0` 时 `FLEXIBLE_DUAL_REPLICA` 不可达（`replica.h:315-318`）→ 静默落入 RELIABLE，**nof 变为必需**，nof 抖动导致整 Put 失败 | `PrepareWriteStart` 显式拒绝 `INVALID_PARAMS` | 有意与 KVCS 不对称：把运行期惊喜变成启动期可发现约束 |
| SFMEA-30 | miss 清理 CAS 不匹配（读到 R1，目录已是 R2） | 无 | `EvictDiskReplicaIfCurrent` 返回 false，R2 原样保留（`master_service.cpp:6463-6473`） | 既有 CAS 语义，仅需放宽白名单 |
| SFMEA-32 | `Shutdown` 时仍有 in-flight task | 注销 region 会导致设备访问已注销内存 | 先 `Drain` 排空，再断言各区间 `inflight==0`，非零则 `LOG(ERROR)` 并输出 `UnsafeBytes()` | 注册区间在 Shutdown 统一注销 |
| SFMEA-33 | 读回长度 != `object_size` | 数据不完整 | 双层校验：适配器内 + `DistributedStorageBackend:838-841`，返回 `FILE_READ_FAIL` | `GetBatch` 必须返回 **N**（= `object_size`）而非 `wire_len`，否则每次读取都误报 |
| SFMEA-37/39 | 启用 string key 路径的连带风险：`Resolve()` 提交阶段失败，或共用配置写了 `inference.engine=vllm_ascend` | 前者让整批同步失败（`taskId` 无效、返回 `INVALID_ARGUMENT`）；后者把 `@...@<数字>` 结尾的 key 静默重解释为分层 key（identity 截断 + `baseOffset`），**读写落到错误的 key/offset 且不报错** | 本方案只传 `FromBytes`（解析直接返回原值、恒成功），并在 `Init` 前把 `keyResolution` 复位为 `GENERIC`；提交阶段同步失败统一 `inflight--`（与 SFMEA-16 同路） | 换用 `FromEngineKey` 前这两道必须保留，并补解析失败用例 |
| SFMEA-38 | 全零 `CacheKey` | 协议层 `INVALID_ARGUMENT`（`kv_protocol.cc` 的 `IsCacheKeyAllZero`），预发送失败会 abort 整个 transport task | `EncodeBinaryKey` 内把全零结果替换为非零常量（概率 2^-64，但不值得留窗口） | 与 SFMEA-05 同一 abort 机制 |
| SFMEA-40 | 传输层 deadline 到期，以 **`PARTIAL_FAILED` + 逐条 `TIMEOUT`** 的形式返回（`trans_task_executor.cc:390-401` → `trans_task_manager.cc:35-41`）；`TransportTaskExecutor::Cancel` 同样只放弃任务、不停设备（`:258-275`） | 若按 **task status** 处置区间（把 `PARTIAL_FAILED` 当「可安全复用」）→ 设备仍在写的内存被重新使用 → **静默数据损坏** | 区间处置只看 **entry status**：`TIMEOUT` / `CANCELED` / `IN_PROGRESS` → `MarkUnsafe`，其余 → `inflight--`（§3.2.3 结果处置） | **承重**：`PARTIAL_FAILED` 既可能来自设备逐条报错（可复用），也可能来自 deadline（不可复用），二者在 task status 上无法区分。用例 T-57 |
| SFMEA-41 | SDK 配置的 transport deadline 小于适配器 `Wait` 超时：`waitTimeoutMs` 同时写 `defaultWaitTimeoutMs` 与 `timeoutMs`（`config_utils.cc:183-186`），后者进 `transportConfig.timeoutMs`（`:273`）；默认 `timeoutMs` 仅 100 ms | 健康但较慢的写（尤其 16 MiB 大对象）在 `Wait` 到期前被判 `PARTIAL_FAILED` + `TIMEOUT`，故障现象由「超时」伪装成「设备错误」；叠加 SFMEA-40 时还会把区间标记为 unsafe | `Init` 内校验 `sdk_config.timeoutMs >= wait_timeout_ms`，不满足拒绝启动（§3.3）；生产配置须显式设置 `waitTimeoutMs` | 用例 T-58 |
| SFMEA-42 | 对齐契约被破坏：槽位起点未对齐、槽位容量 < `wire_len`、区间未注册或注册失败、多段不连续 | 读会写出槽位边界；写会把槽位外的相邻内存发到 L3；或访问未注册内存 | 统一按 `wire_len = AlignUp(N,512)` 判定与提交；写路径清零 `[N, wire_len)`；`addr % 512 == 0` 且 `FindPersistentHandle` 整体包含，**任一不满足返回错误，不回退** | 用例 T-59 / T-60 |
| SFMEA-44 | unsafe 区间被复用 | 设备迟到写入 → **静默数据损坏** | `FindPersistentHandle` 拒绝与 unsafe 标记重叠的请求并返回错误；unsafe 标记常驻到进程重启；`UnsafeBytes()` 告警 | 用例 T-61 |
| SFMEA-45 | 注册失败或注册表只增不减：`RegisterRegions` 失败、请求地址覆盖过多区间 | 该请求失败；长期运行句柄/注册内存持续增长 | 注册失败返回 `DFS_SERVICE_UNAVAILABLE`，不回退；同区间复用同一 handle；用 `kvclient_registered_bytes` 监控增长 | 用例 T-63 |

## 2.5 可定位设计

### 2.5.1 统一关联字段

所有 driver 与适配器日志必须携带以下字段，使一次 I/O 可跨层串联：

| 字段 | 来源 | 用途 |
| --- | --- | --- |
| `logical_key` | 应用 | 业务侧关联（可能含敏感信息，按日志级别裁剪） |
| `cache_key` | `EncodeBinaryKey` 结果的 16 位 hex（`kv::BinaryKey`，不经 SDK 解析） | 碰撞排查的**主要**手段（默认配置下唯一；开启 object checksum 后可被 `CHECKSUM_MISMATCH` 直接发现）：两个不同 logical_key 出现同一 cache_key 即为碰撞 |
| `object_size` | descriptor | 对齐/上限问题定位 |
| `task_id` | SDK | 跨 client/transport 层关联 SDK 内部日志 |
| `replica_id` | descriptor | miss 清理 CAS 排查 |
| `region_id` | 适配器注册表 | 区间关联（注册、unsafe 标记、inflight） |
| `wire_len` | 适配器 | 线格式长度；与 `object_size` 的差值即补零量 |
| `unsafe` | 适配器 | 该请求是否因区间已标记 unsafe 被拒绝 |
| `batch_index` / `original_index` | 适配器 | 批量结果错位排查 |

### 2.5.2 日志与指标

| 级别 | 内容 |
| --- | --- |
| `ERROR` | Wait 超时（含 `task_id`、`op`、`region_id`、**unsafe 字节数**）；`RegisterRegions` 失败；`Shutdown` 时区间 `inflight != 0`；entryStatus 映射到未知 SDK 码 |
| `WARNING` | `CheckSharedProviderMode` 通过但 provider 类型非预期；单项 entryStatus 失败（非整 task）；请求因不连续/未对齐被拒 |
| `INFO` | `Init` 成功（含 SDK 版本、`sharedProviderMode`、`MaxValueSize`）；`Shutdown` 完成（含注册区间数与累计 unsafe 字节） |
| `VLOG(1)` | 每次 `PutBatch` / `GetBatch` / `DeleteBatch` 的 key 数、总字节、切批数、耗时分解 |
| `VLOG(2)` | 逐 entry 的 `logical_key` → `cache_key`、`region_id`、`inflight`、区间 unsafe 标记 |

| 指标 | 类型 | 说明 |
| --- | --- | --- |
| `kvclient_registered_bytes` | gauge | 已注册区间总量，**单调不减**（不淘汰） |
| `kvclient_registration_total{result}` | counter | 注册成功 / 失败 / 复用 |
| `kvclient_unsafe_bytes` | gauge | 被标记 unsafe 的区间字节数，常驻到重启 |

### 2.5.3 定位手册

| 症状 | 排查路径 |
| --- | --- |
| Init 失败 `DFS_SERVICE_UNAVAILABLE` | 查 `sharedProviderMode` 是否为 SHARED（SFMEA-07）；查 view service 可达性 |
| Init 失败 `PARTIAL_FAILED` | 查 SDK `Init` 日志；AICPU provider 下确认 SHARED 模式 |
| Put 全部失败但 SDK 日志无异常 | 查是否读了超时后的 `entryStatus`（SFMEA-13）；查 `Wait` 超时日志里的 `task_id` 与 `task_status`（§2.5.2） |
| 读到错误数据，无报错 | **优先怀疑 key 碰撞**（SFMEA-01）：grep VLOG(2) 日志，比对两个 logical_key 是否得到同一 `cache_key` |
| 读取恒返回 `FILE_READ_FAIL`，但数据其实写对了 | 查 `GetBatch` 是否误返回 `aligned_size` 而非 N（§3.4）——后端比对的是 `desc.object_size` |
| 读到的对象尾部有脏数据 | 查写路径是否漏了 `memset` 补零，或读路径 memcpy 是否误用了 `wire_len` 而非 N |
| 内存/句柄持续增长 | 注册表**不淘汰**，`kvclient_registered_bytes` 单调上升属预期；关注增长速度与覆盖区间数。`UnsafeBytes()` 非零说明有超时区间被永久标记（SFMEA-12/44） |
| 请求返回 `INVALID_PARAMS` | 多数是 buffer 侧不满足契约：多段不连续、槽位起点未 512B 对齐、或区间未注册且注册失败（§3.2.3） |
| 请求返回 `DFS_SERVICE_UNAVAILABLE` | 惰性注册失败（`kvclient_registration_total{result="failed"}`）；查 SDK 日志与 provider 状态 |
| 首次请求延迟偏高 | 首次 I/O 注册的固有成本；同一区间后续请求复用 handle，不再注册 |
| Remove 返回成功但存储侧仍有 value | 不可能——value-first 下成功即已删。若出现，查是否走了 `RemoveAll`/`RemoveByRegex`（SFMEA-25） |
| Remove 返回错误且对象变为不可读 | 已知窗口（SFMEA-21）：value 已删但 `CheckRemovable` 拒绝。等待后续读取触发 miss 自愈 |
| Get 持续 `OBJECT_NOT_FOUND` 但目录有记录 | 查 `EvictDiskReplicaIfCurrent` 是否返回 `INVALID_PARAMS`——说明 `use_kvclient_` 白名单漏改（§3.5） |
| 对象大于 4 MiB 写入失败 | 查 `kKvClientMaxValueBytes` 是否被误配；KVClient 上限是 16,776,704 而非 KVCS 的 4 MiB |
| N 在 16,776,705 ~ 16,777,200 之间的对象写入失败 | 预期行为。该区间能通过 Mooncake 的 `kMaxSliceSize` 但超 SDK 线格式上限，由 `PrepareWriteStart` 拒绝（§3.5） |
| Master 重启后 L3 全部读不到 | 预期行为（不支持恢复裁决，SFMEA-26）。目录不可恢复，value 不可枚举 |

---

# 3 shard 设计描述

## 3.1 shard-1：key 编码（`kvclient_key_codec`）

**职责**：logical key → 8 字节 `kv::BinaryKey` 的确定性映射，并包装成 SDK 的 `kv::CacheKey`。

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
// Deliberately the same value as the SDK's HASH_SEED_VALUE
// (kv_semantics/src/client/key_resolver.h). We never hand the SDK a string key,
// so this does not change the wire identity today; sharing one constant keeps a
// future switch to FromEngineKey a no-op and makes cache_key directly
// comparable with the UCM/ASUStore side.
inline constexpr XXH64_hash_t kCacheKeySeed = 5381ULL;

// Collisions are accepted by design (no fingerprint validation). Two distinct
// logical keys mapping to the same CacheKey silently overwrite each other.
// Probability is ~n^2/2^65; see SFMEA-01 for the deployment key-count bound.
inline kv::BinaryKey EncodeBinaryKey(std::string_view logical_key) {
    XXH64_hash_t hash =
        XXH64(logical_key.data(), logical_key.size(), kCacheKeySeed);
    // The protocol rejects an all-zero key (kv_protocol.cc IsCacheKeyAllZero)
    // and that rejection aborts the whole transport task, so the 2^-64 case
    // is mapped away instead of being left as a latent batch poisoner.
    if (hash == 0) hash = kCacheKeySeed;
    kv::BinaryKey raw{};
    std::memcpy(raw.data(), &hash, sizeof(hash));
    return raw;
}

// 16 hex chars. This is the only observable handle for collision diagnosis,
// so it must be logged unconditionally at VLOG(2). It takes BinaryKey rather
// than CacheKey because CacheKeyView/CacheKeyToHex live in the SDK's private
// src/trans/trans_types.h and are not available to external consumers.
inline std::string CacheKeyToLogString(const kv::BinaryKey& key) {
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

- **seed 选择**：取 `5381`，与 SDK 的 `HASH_SEED_VALUE`（`kv_semantics/src/client/key_resolver.h`）同值。本方案不使用 SDK 的 string key 解析，故 seed 不影响今天的 wire 身份；对齐它只是为了 (a) 将来若改用 `FromEngineKey`，wire key 逐字节不变，(b) 与 UCM/ASUStore 侧用同一函数校验时 `cache_key` 可直接比对。借鉴 `nvme_kv/key_codec.cpp:44-45` 的 XXH64 用法，但那里用双 seed 产出 16 字节（`low`/`high`）；本方案只需 8 字节，单 seed 即可。
- **纯函数约束**：映射不得混入 epoch / 进程 id / 随机种子。这既是跨 Client 读取的前提，也是不支持恢复裁决下唯一需要保证的稳定性。
- **SDK 侧 string key 哈希为预留能力，本方案不使用**（KVClient `#1641` 起 `kv::CacheKey::FromEngineKey(std::string)` 会在 `SubmitAsync` 内用 XXH64(seed=5381) 自行哈希）。
- **全零 key 必须映射掉**：协议层对全零 key 直接 `INVALID_ARGUMENT`，而预发送失败会 abort 整个 transport task（SFMEA-05 同一机制）。XXH64 输出全零的概率是 2^-64，但代价是一整批被毒化，不值得留窗口。

## 3.2 shard-2：适配器（`kvclient_object_storage_adapter`）

**职责**：实现 `ObjectStorageAdapter`，承载 key 编码、原 buffer 注册、零拷贝读写、提交/等待与结果映射。与 `KvcsObjectStorageAdapter` 平级，互不依赖。

**文件**：`mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h`、`src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp`

**注册表不单独成类**：它只有一个消费者，注册与注销本就依赖 `kv::KvClient`；故注册表、unsafe 标记与在途计数直接作为适配器私有成员持有。

### 3.2.1 类定义

```cpp
#pragma once

#include <atomic>
#include <memory>
#include <mutex>
#include <optional>
#include <shared_mutex>
#include <span>
#include <string>
#include <unordered_map>
#include <vector>

#include "storage/distributed/object_storage_adapter.h"

namespace mooncake {

inline constexpr char kKvClientAdapterName[] = "kvclient";

// Single-entry wire limit: SQE length is 24-bit and must be 512-aligned, so
// 0xFFFFFF & ~511 == 0xFFFE00. Not "about 16 MiB" -- the exact bound matters
// because Mooncake's kMaxSliceSize (16,777,200) is 496 bytes wider and will
// admit values that cannot go on the wire.
inline constexpr uint64_t kKvClientMaxValueBytes = 16'776'704;

struct KvClientAdapterConfig {
    std::string config_path;  // SDK client config
    uint64_t wait_timeout_ms = 30'000;
    // Escape hatch: when false the adapter rejects DFS I/O instead of
    // registering buffers. There is no staging fallback either way.
    bool zero_copy_enabled = true;
};

struct RegisteredPersistentRegion;  // §3.2.3

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

   private:
    struct PreparedKvBuffer {
        std::shared_ptr<RegisteredPersistentRegion> region;
        uintptr_t addr = 0;
        size_t n = 0;
        size_t wire_len = 0;
    };

    // Delegates to kvclient_key_codec.h's EncodeBinaryKey and returns the
    // 8-byte wire identity. Wrap with kv::CacheKey::FromBytes at the call site
    // so the same value stays available for cache_key logging (§3.1).
    kv::BinaryKey MakeCacheKey(std::string_view logical_key) const;
    // Validates contiguity and alignment, then registers the range on first
    // use and keeps it for the adapter lifetime (ASUStore model). Returns
    // nullopt and sets `error` on any contract violation; there is no staging
    // fallback (SFMEA-42/45).
    std::optional<PreparedKvBuffer> PrepareKvBuffer(const iovec* iov, int iovcnt,
                                 size_t object_size, bool is_store,
                                 ErrorCode& error);
    // Address-containment lookup, mirroring ASUStore::FindPersistentHandle.
    // Runs under a shared lock; returns nullptr when no range contains
    // [addr, addr + len).
    std::shared_ptr<RegisteredPersistentRegion> FindPersistentHandle(uintptr_t addr,
                                                           size_t len) const;
    // Inserts a freshly registered range. Ranges are never evicted.
    std::shared_ptr<RegisteredPersistentRegion> AddPersistentRegion(uintptr_t base, size_t size,
                                                kv::MRHandle handle);
    bool IsUnsafe(const RegisteredPersistentRegion& region, uintptr_t addr,
                   size_t len) const;
    // Retires [addr, addr + wire_len): the device may still write it. The mark
    // is permanent for this process because ranges are never re-registered
    // (SFMEA-44).
    void MarkUnsafe(const std::shared_ptr<RegisteredPersistentRegion>& region, uintptr_t addr,
               size_t wire_len);
    uint64_t UnsafeBytes() const;

    // Kept as a bool because kv::AsuOpType lives in the SDK's private
    // src/common/utils/task_context.h; the public API exposes only
    // StoreAsync / LoadAsync and picks the batch flavour itself.
    // Plans are parallel to entries; on a terminal state each is marked or
    // released according to its own entry status.
    void SubmitAndWait(bool is_store, std::vector<kv::KVBuffer>& entries,
                       const std::vector<size_t>& original_indices,
                       const std::vector<PreparedKvBuffer>& plans, BatchResults& results);
    static ErrorCode ToErrorCode(const kv::Status& status);
    bool CheckSharedProviderMode(const kv::KvClientConfig& sdk_config) const;
    void Shutdown();

    KvClientAdapterConfig config_;
    std::unique_ptr<kv::KvClient> kv_client_;
    bool initialized_ = false;

    // Persistent registrations: one entry per range, registered on first use
    // and kept until Shutdown (ASUStore model).
    mutable std::shared_mutex persistentRegionsMu_;
    std::unordered_map<uintptr_t, std::shared_ptr<RegisteredPersistentRegion>> persistentRegions_;
    std::atomic<uint64_t> nextPersistentRegionId_{1};
};

}  // namespace mooncake
```

### 3.2.2 无 staging 路径

本方案**不设 staging**：没有 slot 位图、没有补零缓冲、没有 quarantine。原 buffer 直接注册给 SDK，所有数据路径都走 §3.2.3。任何不满足注册或对齐契约的请求都以错误返回，而不是降级拷贝。

### 3.2.3 零拷贝数据路径

**职责**：把调用方 buffer 注册给 SDK 并直接读写；**注册与读写都在适配器内完成**，与 ASUStore 的 `persistentRegions_` + `FindPersistentHandle` 模型一致。

### 注册表

```cpp
struct UnsafeRange {
    uintptr_t begin;
    uintptr_t end;
};

struct RegisteredPersistentRegion {
    uintptr_t base{0};
    size_t size{0};
    kv::MRHandle handle{kv::kInvalidMRHandle};
    uint64_t id{0};                      // 日志 / 指标关联
    std::atomic<uint32_t> inflight{0};   // 在途 I/O 数
    std::vector<UnsafeRange> unsafe_ranges;  // 保序、合并
};

// 注册表是适配器的私有成员，不单独成类（§3.2.1）：
//   mutable std::shared_mutex persistentRegionsMu_;
//   std::unordered_map<uintptr_t, std::shared_ptr<RegisteredPersistentRegion>> persistentRegions_;
//   std::atomic<uint64_t> nextPersistentRegionId_{1};
// 方法：FindPersistentHandle / AddPersistentRegion / IsUnsafe / MarkUnsafe
```

- **注册一次常驻**：首次 I/O 注册 `[addr, AlignUp(len, 512))` 并插入；此后同区间请求直接复用 handle，**不淘汰**，直到 `Shutdown` 统一注销。这是 ASUStore 的持久注册语义。
- **查找语义**：`[addr, addr + len)` 必须整体落在一个已注册区间内（`addr >= base && addr - base <= size - len`），否则视为未注册。
- **重复请求**：同 `base` 复用；不同 `base` 但包含关系成立时，返回第一个匹配。
- **unsafe 标记常驻**：超时把 `[addr, addr + wire_len)` 标记为 unsafe，重叠请求一律失败，常驻到进程重启。

### 构造 KVBuffer 与注册

```cpp
// PreparedKvBuffer 定义见 §3.2.1。
// Returns nullopt and fills `error` when the request violates the contract.
std::optional<PreparedKvBuffer> KvClientObjectStorageAdapter::PrepareKvBuffer(
    const iovec* iov, int iovcnt, size_t object_size, bool is_store,
    ErrorCode& error) {
    const size_t n = is_store ? SumIov(iov, iovcnt) : object_size;
    if (n == 0) { error = ErrorCode::INVALID_PARAMS; return std::nullopt; }
    const size_t wire_len = AlignUp(n, kv::kAlignmentBytes);
    if (wire_len > kKvClientMaxValueBytes) {
        error = ErrorCode::INVALID_PARAMS;
        return std::nullopt;
    }
    if (!IsContiguous(iov, iovcnt)) {
        error = ErrorCode::INVALID_PARAMS;   // 不连续即失败，不拷入 staging
        return std::nullopt;
    }
    const auto addr = reinterpret_cast<uintptr_t>(iov[0].iov_base);
    if (addr % kv::kAlignmentBytes != 0) {
        error = ErrorCode::INVALID_PARAMS;   // 槽位起点未对齐即失败
        return std::nullopt;
    }

    auto region = FindPersistentHandle(addr, wire_len);
    if (!region) {
        // Register once and keep it for the adapter lifetime (ASUStore model).
        std::vector<kv::MemoryRegion> regions{
            {kv::MemoryType::HOST, addr, wire_len, /*deviceId=*/-1,
             /*numaNode=*/-1}};
        std::vector<kv::RegisteredMemory> out;
        if (!kv_client_->RegisterRegions(regions, out).ok()) {
            error = ErrorCode::DFS_SERVICE_UNAVAILABLE;  // 注册失败即失败
            return std::nullopt;
        }
        region = AddPersistentRegion(addr, wire_len, out.front().handle);
    }
    if (IsUnsafe(*region, addr, wire_len)) {
        error = ErrorCode::INVALID_PARAMS;
        return std::nullopt;
    }
    return PreparedKvBuffer{region, addr, n, wire_len};
}
```

**三条失败路径一律返回错误，不回退**：多段不连续 → `INVALID_PARAMS`；槽位起点未 512B 对齐 → `INVALID_PARAMS`；`RegisterRegions` 失败 → `DFS_SERVICE_UNAVAILABLE`。

### 构造与提交

```cpp
// 512B 契约：写路径清零尾部，读路径的尾部由设备写在同一区间内。
if (is_store && wire_len > n) {
    std::memset(reinterpret_cast<char*>(plan.addr) + n, 0, wire_len - n);
}

kv::KVBuffer entry;
entry.key = kv::CacheKey::FromBytes(MakeCacheKey(logical_key));
entry.buffer.region = {kv::MemoryType::HOST, plan.addr,
                       static_cast<uint64_t>(plan.wire_len),
                       /*deviceId=*/-1, /*numaNode=*/-1};
entry.buffer.handle = plan.region->handle;
entry.offset = 0;            // 512 对齐；本方案不做分层聚合
entry.mrKey = std::nullopt;  // SDK 按 handle 回填

plan.region->inflight.fetch_add(1, std::memory_order_acq_rel);
```

`entry.buffer.region.size` 必须等于 `wire_len`：SDK 没有独立的 KV 长度字段（§3.4）。

### 结果处置

| entry 状态 | 区间处置 |
| --- | --- |
| `OK` / 业务错误 | `inflight--` |
| `TIMEOUT` / `CANCELED` / `IN_PROGRESS` | `MarkUnsafe(range)` + `inflight--` |
| task 非终态 / 提交失败 | 全部 `MarkUnsafe` / 全部 `inflight--` |

`MarkUnsafe` 把 `[addr, addr + wire_len)` 并入区间的 unsafe 列表。区间不会被重新注册，因此该地址在进程生命周期内不可再用于快路径（SFMEA-44）。
## 3.3 shard-3：配置（`kvclient_config`）

**环境变量**（沿用 `Environ::ReadOr` 模式，`kvcs_capi_driver.cpp:39-75`）：

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `MOONCAKE_KVCLIENT_CONFIG` | 无（必填） | SDK client 配置路径 |
| `MOONCAKE_KVCLIENT_WAIT_TIMEOUT_MS` | 30000 | `Wait` 超时 |
| `MOONCAKE_KVCLIENT_ZERO_COPY` | true | 关闭后适配器拒绝 DFS I/O（没有 staging 可退） |

**校验**（`ReadConfig`）：`config_path` 非空；`wait_timeout_ms > 0`。

**跨配置校验**（`Init` 内，需先 `LoadFromFile` 才能拿到值）：`sdk_config.timeoutMs >= wait_timeout_ms`。SDK 的 `waitTimeoutMs` 配置键同时写 `defaultWaitTimeoutMs` 与 `timeoutMs`（`config_utils.cc:183-186`），后者成为 transport task 的 deadline（`:273`）；若它小于适配器自己的 `Wait` 超时，健康但较慢的写会在 `Wait` 到期前就被判成 `PARTIAL_FAILED` + 逐条 `TIMEOUT`，故障现象从「超时」变成「设备报错」。不满足即拒绝启动（`INVALID_PARAMS`）。

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

## 3.4 shard-4：padding 与长度语义

`object_size = N`，`wire_len = AlignUp(N, 512)` 是唯一的线格式长度（SDK 没有独立的 KV 长度字段，§3.2.3）。**512B 对齐是全局契约**：对象在 buffer 中占 `wire_len` 字节，起点 512B 对齐。

- **写路径**：设备读满 `wire_len`，适配器先清零尾部 `[N, wire_len)`（≤511 字节），不复制 payload。
- **读路径**：设备写满 `wire_len`，尾部落在同一槽位内，调用方只取前 N 字节。
- 不满足对齐契约的请求直接失败（§3.2.3），**没有 staging 分支**。

**写路径**：

```cpp
// PutBatch: 后端传来的 ObjectPutRequest{logical_key, iov, iovcnt}
auto plan = PrepareKvBuffer(request.iov, request.iovcnt, /*object_size=*/0,
                   /*is_store=*/true, error);
if (!plan) { results[i] = tl::make_unexpected(error); continue; }

// 512B 契约：清零尾部，避免把槽位里的残留字节发到 L3。
if (plan->wire_len > plan->n) {
    std::memset(reinterpret_cast<char*>(plan->addr) + plan->n, 0,
                plan->wire_len - plan->n);
}

kv::KVBuffer entry;
entry.key = kv::CacheKey::FromBytes(cache_key);
entry.buffer.region = kv::MemoryRegion{
    kv::MemoryType::HOST,
    static_cast<std::uint64_t>(plan->addr),
    static_cast<std::uint64_t>(plan->wire_len),   // 线格式长度就是它
    /*deviceId=*/-1,
    /*numaNode=*/-1,
};
entry.buffer.handle = plan->region->handle;
entry.offset = 0;
entry.mrKey = std::nullopt;
```

**读路径**：

```cpp
// GetBatch: 后端传来的 ObjectGetRequest{logical_key, buffer, size = object_size}
auto plan = PrepareKvBuffer(request.iov, request.iovcnt, request.size, /*is_store=*/false,
                   error);
if (!plan) { results[i] = tl::make_unexpected(error); continue; }
// 设备直接把 wire_len 字节写入调用方 buffer，调用方只消费前 N 字节。
return request.size;                              // 返回 N，不是 wire_len
```

⚠️ **返回值必须是 `n` 而非 `wire_len`**。`DistributedStorageBackend::BatchRead` 会比对 `*object_results[i] != object_requests[i].size`（`:838-841`），而 `request.size` 是 `desc.object_size` = N。返回 `wire_len` 会让每次读取都误报 `FILE_READ_FAIL`。

**`aligned_size` 在读路径不直接使用**：后端传给 `GetBatch` 的 `size` 取 `desc.object_size`（`:817-818`），不取 `aligned_size`。故 `aligned_size` 目前只是元数据记录（供诊断与未来扩展），适配器自行用 `AlignUp(object_size, 512)` 重算线格式长度。

**注册粒度**：首次请求按 `[addr, AlignUp(N,512))` 注册并常驻。同一对象重复读写命中同一区间；对象长度变化会注册新区间，注册表只增不减（§5）。

**长度校验清单**（三道）：

| 层 | 校验 | 位置 |
| --- | --- | --- |
| Master | `value_length <= 16,776,704` | `PrepareWriteStart`（§3.5） |
| 适配器写 / 读 | `AlignUp(N,512) <= kKvClientMaxValueBytes` | `PrepareKvBuffer` |
| 适配器读 | 返回长度 == `request.size`（N） | `GetBatch` |

Master 那道必需，因为 `kMaxSliceSize`（16,777,200）比 SDK 上限宽 496 字节，且只在 CACHELIB 分配器下生效（`master_service.cpp:5287-5293`）。
## 3.5 shard-5：Master 集成

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
    // itself may be arbitrary -- the adapter clears the slot tail, it does not
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

## 3.6 shard-6：Client 集成

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
| `distributed_storage_backend.h:70-71` | 新增 `bool UsesKvClient() const { return UsesObjectStorage() && distributed_config_.UsesKvClient(); }`。**注册不经过后端** |
| `client_service.{h,cpp}`、`real_client.cpp` | **零改动**：原 buffer 的注册与读写都在 `KvClientObjectStorageAdapter` 内完成 |
| `storage_backend.cpp:6506` 旁 | 工厂新增 `UsesKvClient()` 分支 |

**不改**：`DetermineFinalizeDecision`、`DetermineReplicaWriteMode`、`AllAllocatedTransfersSucceeded`、`HasExpectedReplicaAllocation`、`FinalizeBatchUpsert`——两副本硬逻辑裁决 的两副本硬逻辑现有代码已完全实现。

## 3.7 shard-7：构建与可选依赖

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

层级：**单元** = fake driver / fake SDK，无需真机；**集成** = Mooncake 端到端（fake SDK）；**真机** = fake 无法覆盖的确认项。只保留能区分设计正确性的用例，等价分支合并到同一编号行。

| 编号 | 层级 | 用例 | 预期结果 | 覆盖 |
| --- | --- | --- | --- | --- |
| T-01 / T-02 | 单元 | `EncodeBinaryKey` 幂等与单射：同 key 多次调用、10^5 个不同 key 抽样 | 逐字节相同 / 互不相同 | §3.1 纯函数约束 |
| T-03 | 单元 | 缩小 seed 空间强制构造碰撞，先写 k1 再写 k2 | k2 覆盖 k1，读 k1 得到 k2 的数据；默认配置（checksum 关闭）下**无错误返回**，开启 `MOONCAKE_STORE_CHECKSUM` 时返回 `CHECKSUM_MISMATCH` | SFMEA-01（验证碰撞后果与检测边界） |
| T-04 / T-05 | 单元 | 空 logical key；同批两个相同 logical key | 前者 `INVALID_PARAMS`；后者第二个 `INVALID_PARAMS` | SFMEA-02/03 |
| T-06 / T-25 / T-25b / T-09b | 单元 | N=513 写读；A(513)→B(1024) 复用同一槽位后再读 A | 成功，`buffer.region.size == 1024`，`[0,513)` 为原始数据、`[513,1024)` 全 0；复用后 A 不含 B 残留 | SFMEA-04、512 对齐裁决（N 任意是合法输入，不是拒绝条件） |
| T-07 | 单元 | N = 16,776,704（上限，本身 512 对齐） | 成功，`buffer.region.size == 16776704`，无补零 | §3.5 上限 |
| T-08 / T-08b | 单元 | N = 16,777,000（能过 `kMaxSliceSize`=16,777,200，但 `AlignUp` 后为 16,777,216 超上限）；N = 16,777,216 | Master `PrepareWriteStart` → `INVALID_PARAMS` 且适配器未被调用；绕过 Master 时适配器预检拒绝且**不提交 SDK** | SFMEA-05 双层防护（**关键**：Master 那道校验存在的理由） |
| T-09 | 单元 | N = 0 | `INVALID_PARAMS` | SFMEA-05/06 |
| T-09c / T-25c / T-27 | 单元 | `GetBatch` 对 N=513 对象的返回值；构造读回长度 != 请求长度 | 返回 **513**（不是 1024）；长度不符 → `FILE_READ_FAIL` | §3.4、SFMEA-33（返回 wire_len 会让每次读都误报） |
| T-10 / T-11 | 单元 | `sharedProviderMode = 0` 时 `Init()`；`RegisterRegions` 返回 PARTIAL_FAILED | 前者 `DFS_SERVICE_UNAVAILABLE` 且不进入 `RegisterRegions`；后者 `Init()` / `FileStorage::Init` 明确失败 | SFMEA-07/08 |
| T-13 | 单元 | 篡改 entry 的 `buffer.handle` 为非注册表内的非 0 值 | 提交前预检拒绝，不到 Wait | SFMEA-10 |
| T-14 | 单元 | fake SDK 让 `Wait` 返回 TIMEOUT 且 task 保持 in-flight | 区间被标记 unsafe：`UnsafeBytes()` 增加，同区间后续请求返回 `INVALID_PARAMS` | SFMEA-12/44 |
| T-15 | 单元 | fake SDK 让 `Wait` 超时且 `entryStatus` 保持初值 `OK` | 逐项结果为 `RPC_TIMEOUT`，**绝不为 OK** | SFMEA-13（回归防护，必须有） |
| T-16 | 单元 | fake SDK 返回 task status=`PARTIAL_FAILED`，第 2 项 entryStatus 为业务错误，其余 OK | 按 `originalIndices` 回填原下标 2、其余成功；区间按 entry status 处置（本例为业务错误 → `inflight--`） | SFMEA-14 |
| T-17 | 单元 | fake SDK 让整个 task 返回 `CANCELED`；另跑 `TIMEOUT` / `IN_PROGRESS` | 全部区间 `MarkUnsafe`，逐项 `INTERNAL_ERROR`（`TIMEOUT` 为 `RPC_TIMEOUT`） | SFMEA-15/13 |
| T-18 | 单元 | fake SDK 让 `StoreAsync` 提交阶段失败 | 区间 `inflight--`（不标记） | SFMEA-16/17 |
| T-19 | 单元 | fake SDK 返回 `CQE_OUT_OF_CREATE_SIZE` | 映射 `NO_AVAILABLE_HANDLE` | SFMEA-17 |
| T-20 / T-21 | 单元 | fake SDK 读时返回 `ENTRY_KEY_NOT_FOUND`；读时返回 `TIMEOUT` | 前者 `OBJECT_NOT_FOUND`；后者 `RPC_TIMEOUT` 且**不触发** `ReconcileMissingKvReplica` | SFMEA-18 正向 / 反向 |
| T-22 | 单元 | 跨对象合批：3 个对象共 250 entry，批上限 110 | 切成 3 批（110/110/30），无对象被切开，结果下标正确 | §2.3.2 |
| T-23 | 单元 | `DeleteBatch` 遇 `ENTRY_KEY_NOT_EXIST` | 视为幂等成功 | §3.2 |
| T-24 | 单元 | `Shutdown` 时仍有 in-flight | 先 Drain；若区间 `inflight != 0` 则 `LOG(ERROR)` 并输出 `UnsafeBytes()`；随后注销全部注册区间 | SFMEA-32 |
| T-26 | 单元 | `Exists()` / `ListKeys()` / `GetRange()` | 均 `NOT_SUPPORTED` | §1.2.2 |
| T-29 | 集成 | 两副本 Put 成功 → 淘汰 memory → 另一 Client Get | 读到原始数据，逐字节相同 | S1 |
| T-30 | 集成 | DFS-only Put / Get / Remove | 全流程成功 | S2 |
| T-31 / T-32 | 集成 | 注入 ASU 写失败；注入 memory 写失败 | 前者 Put 失败且 Master 侧查不到该对象（memory 副本被一起 revoke）；后者适配器侧提交计数为 0（ASU 零调用）、Put 失败 | 两副本硬逻辑裁决（含短路行为） |
| T-33 | 集成 | 配置 `memory + nof + ASU` 后 Put | `PrepareWriteStart` 返回 `INVALID_PARAMS` | SFMEA-28 |
| T-34 / T-35 | 集成 | 同尺寸对象 Upsert；仅 MEMORY 的对象做 Upsert 请求 dfs=1 | 前者原地复用、ReplicaID 不变、同 CacheKey 覆盖写并读到新值；后者失败（拓扑不符），需先 Remove 再 Put | §2.2.4 |
| T-36 / T-37 | 集成 | Remove：value 删除成功 + Master `BatchRemove` 失败；对象处于 lease 中 | 前者元数据保留 → 后续 Get `NOT_FOUND` → `ReconcileMissingKvReplica` 清理 DFS 记录；后者 `CheckRemovable` 拒绝且 Get 返回 `NOT_FOUND` 而非错数据，最终经 miss 自愈清理 | SFMEA-20/21/24（value-first + miss 清理联动） |
| T-38 / T-39 | 集成 | Remove：value 删除失败；`BatchRemove` 部分 key 失败 | 前者元数据**保留**、重试 Remove 可成功；后者成功项元数据被删、失败项两者均保留、结果与原下标对应 | SFMEA-22、S6 |
| T-40 | 集成 | `RemoveAll()` / `RemoveByRegex()` | 返回 `NOT_SUPPORTED`，**且 Master 侧元数据完全未变** | SFMEA-25（关键：不能只删元数据） |
| T-41 | 集成 | Client 观察到 R1 miss，Master 已发布 R2 | 条件清理返回 false，R2 原样保留且可读 | SFMEA-30 |
| T-42 / T-43 | 集成 | `enable_oplog=true` + KVClient 启动 Master；默认配置（无 HA）重启 Master | 前者 `LOG(FATAL)`，启动失败且错误信息明确不支持恢复；后者目录全丢、Get `OBJECT_NOT_FOUND`、无崩溃无脏数据 | SFMEA-27/26、不支持恢复裁决 |
| T-44 | 集成 | 范围读（窗口 < object_size） | `INVALID_PARAMS`，不触达适配器 | §1.2.2 |
| T-45 | 集成 | `PutEnd` 响应未知 | Client 返回提交错误；重复 Put 或查询可确认状态 | 既有时序（§2.2.1） |
| T-46 | 集成 | 无 KVClient SDK 时构建 | 全部目标构建成功；显式初始化 KVClient 后端返回 `NOT_SUPPORTED`；KVCS/oss/s3 后端功能不受影响 | §3.7 |
| T-47 | 集成 | 既有 KVCS 回归 | `kvcs_object_storage_adapter_test`、`object_storage_adapter_test`、`distributed_storage_config_test` 全部通过 | 平级架构下 **KVCS 代码零改动**；失败即说明改动越界 |
| T-48 | 真机 | ASU `Store` 是否支持**覆盖写**：同 CacheKey 连续两次 Store 不同 value | 读回应为第二次 | **阻塞**：决定 `PutBatch` 是否需先 Delete（KVCS 因 insert-only 必须这么做）。不支持则 Upsert 与同 key 重写都会失败 |
| T-49 | 真机 | `Wait` 超时后设备是否仍访问 buffer | 超时后立即用哨兵值覆写已注册区间，检查是否被设备改写 | **阻塞**：验证 unsafe 标记的必要性边界 |
| T-50 / T-51 | 真机 | 设备侧 `singleValueMaxBytes` 实际值；多节点部署下跨节点合批路由 | 读 SDK `Init` 日志（只打日志、不 clamp entry 长度）；`originalIndices` 回填正确 | 高：若上限 < 16,776,704 则实际上限更低；SDK 自动拆分未经真机验证 |
| T-55 | 单元 | 配置文件写入 `inference.engine=vllm_ascend` + `aggregation.layer_count/layer_slot_bytes`，Init 后对逻辑 key `model@hash@3` 做 Put | 写入的 entry `offset == 0`，`cache_key` 等于 `EncodeBinaryKey(logical_key)`——未被重解释为分层 key | §2.3.1 `keyResolution` 复位、SFMEA-39 |
| T-56 | 单元 | 注入返回 0 的 hasher 调 `EncodeBinaryKey` | 输出 `BinaryKey` 非全零（不触发协议层 all-zero 拒绝） | SFMEA-38 |
| T-57 | 单元 | fake SDK 让 transport task 到 deadline：`Wait` 返回 OK、`task_result.status == PARTIAL_FAILED`、部分 `entryStatus == TIMEOUT`，其余为 OK | 只有 TIMEOUT 那几条的区间被 `MarkUnsafe`；OK 那几条 `inflight--`；**不得**因 task status 是 PARTIAL_FAILED 就整批标记 unsafe | SFMEA-40（**关键**：PARTIAL_FAILED 不等于可安全复用） |
| T-58 | 单元 | SDK 配置 `waitTimeoutMs=100`（或任意 < `MOONCAKE_KVCLIENT_WAIT_TIMEOUT_MS`），调 `Init` | `Init` 返回 `INVALID_PARAMS`，不调用 SDK `Init`；错误日志同时打印两个超时值 | SFMEA-41 |
| T-59 | 单元 | 已注册、单段连续、槽位 512B 对齐；分别用 `N % 512 == 0` 与 `N % 512 != 0` 两种对象走 Put/Get | 均成功；写路径尾部被清零，读路径只返回前 N 字节，数据正确 | SFMEA-42 |
| T-60 | 单元 | 违约输入：多段不连续、槽位起点未 512B 对齐 | 均返回 `INVALID_PARAMS`，**不回退、不提交 SDK** | SFMEA-42 |
| T-61 | 单元 | 区间已标记 unsafe 时再次发起重叠请求 | 返回 `INVALID_PARAMS`；unsafe 标记常驻到进程重启 | SFMEA-44 |
| T-62 | 单元 | 同一区间连续发起两次 I/O | 第二次命中注册表、复用同一 handle，不重复调用 `RegisterRegions` | SFMEA-45 |
| T-63 | 单元 | `RegisterRegions` 失败后 Put/Get | 返回 `DFS_SERVICE_UNAVAILABLE`；不降级、不写半份数据 | SFMEA-45 |

## 4.1 验收标准

- 上表全部「单元」「集成」用例通过（fake SDK / 集成环境）。
- T-48 ~ T-51 真机验证通过后方可进入生产部署。
- 24 小时压测后 `UnsafeBytes()` 为 0（无区间降级）。
- 关闭 `MOONCAKE_KVCLIENT_ZERO_COPY` 时 DFS I/O 明确失败（不降级），KVCS / oss / s3 / hf3fs / posix 后端不受影响。
- 既有 KVCS / oss / s3 / hf3fs / posix 后端回归全部通过（T-47）。

---

# 5 已知限制

| 限制 | 说明 | 缓解 |
| --- | --- | --- |
| **key 碰撞静默损坏** | XXH64 单 hash，无 fingerprint（key 碰撞裁决）。碰撞概率 ≈ n²/2^65：10^7 key → 2.7e-6，10^9 → 2.7% | 默认仅能靠 §2.5.1 的 `cache_key` 日志事后排查；可开启 `MOONCAKE_STORE_CHECKSUM` 让 Get 端以 `CHECKSUM_MISMATCH` 发现（Put 仍会覆盖，不报错）。**部署须限定 key 数量级** |
| 删除孤儿窗口 | value-first 下 `CheckRemovable` 拒绝会留下不可读的已发布对象 | 语义安全（不返回错数据）；靠读取 miss 自愈 |
| 无容量管理 | SDK 无容量查询 API → Master 水位淘汰不生效 | 沿用 `:883-889` 降级；容量控制留待 SDK 提供接口 |
| 不支持元数据恢复 | Master 重启即目录全丢，L3 数据不可发现 | 启动期显式拒绝 `enable_oplog`；运维需知晓 |
| 不支持 key 枚举 | `ListKeys` / `RemoveAll` / `RemoveByRegex` 返回 `NOT_SUPPORTED` | 同 key 重写时覆盖 |
| 不支持范围读 | `BatchRead` 对象分支忽略 `range_read`（`:802-818`） | 上层回退整对象读后截取 |
| 适配器内 memcpy + 补零 | 已**取消 staging**：只清零槽位尾部 `[N, wire_len)`（≤511 字节），payload 不复制 | 尾部清零是 512B 契约的一部分，无法省去 |
| **无降级路径** | 不连续、槽位未 512B 对齐、区间已标记 unsafe、注册失败都直接返回错误 | 上层需保证单段连续、512B 对齐的槽位；这是与 v3.0 的主要行为差异 |
| **零拷贝要求 512B 槽位** | 对象必须在 buffer 中占一个 512B 对齐、容量 `AlignUp(N,512)` 的槽位；适配器只能看到 N，无法验证槽位容量 | 由上层分配保证（§3.2.3）；未遵守时读会覆盖相邻数据、写会发出相邻数据。这是本方案唯一无法在适配器内自证的前提 |
| **首次 I/O 多一次注册** | 区间首次使用时调用 `RegisterRegions`，之后复用 handle | 用 `kvclient_registration_total` 观察 |
| **注册表只增不减** | 区间注册后常驻到关闭，不淘汰；不同对象长度会产生新区间 | 用 `kvclient_registered_bytes` 监控增长 |
| **unsafe 标记常驻到重启** | 一次超时会把该区间永久标记，重叠请求一律失败 | 已知窗口：安全优先；`UnsafeBytes()` 告警提示 |
| **小对象补零放大** | N < 512 时线格式长度仍是 512，放大最高 512 倍（N=1）；带宽与存储双浪费 | 上层聚合小对象；补零开销可由 §2.5.1 的 `object_size` 与实际线格式长度对比估算 |
| 单对象上限 16,776,704 B | SDK 线格式 24-bit length + 512 对齐。比 Mooncake 的 `kMaxSliceSize`（16,777,200）窄 496 字节 | `PrepareWriteStart` 显式校验（§3.5）；无其他缓解 |
| Upsert 无法为仅 MEMORY 对象补 DFS 副本 | `master_service.cpp:6146-6181` 拓扑不符 | 先 Remove 再 Put |
| **namespace 为预留能力，当前不上线格式** | `IoOptions.kvns` 在 SDK 内被哈希成 GNSID 并挂在 `ClientTask` / `TransportTask` 上，但**没有消费方**：SQE 的 `kv_ns_id` 仍取 `transport.kv_ns_id` 配置。且默认 `IoOptions{}` 得到的是 `XXH32("")` 而非「回退配置值」 | 本方案不填 `kvns`，租户隔离仍靠 `single_tenant=true` + logical key。SDK 升级时须确认 GNSID 是否开始影响 `kv_ns_id`——那会是一次数据可见性迁移 |
| 推理引擎配置可污染 key 解析 | `KvClientConfig::LoadFromFile` 会解析 `inference.engine` / `aggregation.*`；`vllm_ascend` 下以 `@...@<数字>` 结尾的 key 会被重解释为分层 key | 适配器只传 `FromBytes`，并在 `Init` 前复位 `keyResolution`（§2.3.1、SFMEA-39） |