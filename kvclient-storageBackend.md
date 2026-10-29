# KVClientStorageBackend 第一版最小实现设计（写 / 读 / 查 / 删）

本文给出 `KVClientStorageBackend` 第一版最小实现的设计方案，**本版实现写、读、查、删四步操作**。
它新增一个 `StorageBackendInterface` 实现类，参考 `NvmeKvStorageBackend` 的组织方式，
**在 backend 内直接调用 `kv::KvClient`**，把 Mooncake KV client 的 offload 数据面落到 ASU。

- 状态：Draft v1（写 / 读 / 查 / 删）
- 适用代码：`mooncake-store`（C++）
- 接口基线：`mooncake-store/include/storage_backend.h`
- 参考实现：`NvmeKvStorageBackend`（KV 客户端型后端）
- ASU 访问工具：`kv::KvClient`，来自
  [ModelEngine-Group/unified-cache-management](https://github.com/ModelEngine-Group/unified-cache-management)

## 1. 概述

### 1.1 背景

Mooncake 的 KV client 通过 `StorageBackendInterface` 把 KV cache 对象下沉到 ASU，并在内存 miss
时按 `LOCAL_DISK` replica 读回。默认模式是**写完即下沉**，不是“因淘汰而 offload”：

1. `PutEnd` 成功后，Master 立即把对象加入 owner 的 offload 队列，并对源 `MEMORY` 副本
   `inc_refcnt`（`mooncake-store/src/master_service.cpp` 的 `PushOffloadingQueue` 分支，
   约 5785–5811 行）。
2. `refcnt > 0` 期间该内存副本不可淘汰（`IsEvictableMemoryReplica` 返回 false），淘汰线程碰不到它。
3. 下一次心跳由 `FileStorage` 调 `BatchOffload` 把对象写进 ASU。
4. `NotifyOffloadSuccess` 成功后 Master 对源副本 `dec_refcnt`（`master_service.cpp`，约
   9063–9069 行），这份内存副本才重新成为可淘汰候选。

因此，即使对象一直很热、永远不会被淘汰，它也会在写完后就下沉。`offload_on_evict=true` 时才把
下沉推迟到淘汰时刻，该开关默认 `false`（`mooncake-store/include/master_config.h`，约 184 行）。
本版后端只实现写、读、查、删，不感知触发时机，两种模式都适用。

现有后端都把对象写到本地文件或本地块设备：

| 后端 | 数据面 | 恢复方式 |
| --- | --- | --- |
| `StorageBackendAdaptor` | 每 key 一个本地文件 | 目录扫描 `ScanMeta` |
| `BucketStorageBackend` | bucket 数据文件 + 元数据文件 | 内存索引 + 持久化元数据 |
| `OffsetAllocatorStorageBackend` | 单数据文件 + extent 分配器 | allocator checkpoint 重建 |
| `NvmeKvStorageBackend` | 本地 NVMe-KV 设备 | 无（`ScanMeta` 为空实现） |
| `KVClientStorageBackend`（本文） | ASU（直接调用 `kv::KvClient`） | **本版不实现恢复**（`ScanMeta` 返回空） |

### 1.2 本版范围  **(todo：删除的逻辑需要再确认)**

**实现**：

| 操作 | 入口 | 说明 |
| --- | --- | --- |
| 写 | `BatchOffload` | 把 Mooncake 对象写入 ASU；逐 key 部分成功 |
| 读 | `BatchLoad` | 按 Master 提供的 key + size 从 ASU 读回；校验记录头与 CRC |
| 查 | `IsExist` | 用 ASU `QueryAsync` 判断存在性 |
| 删 | `EvictAboveDiskWatermark` / `RemoveAll` | master-first；FIFO 选 victim；`DeleteAsync` 逐 key；失败恢复索引 |

**接口必需但不实现业务逻辑**：

| 入口 | 本版行为 |
| --- | --- |
| `Init` | 初始化 `kv::KvClient`、注册 staging、置计数；二次 `Init` 失败 |
| `IsEnableOffloading` | 按本进程计数和配置上限返回 true/false |
| `ScanMeta` | 返回 `OK`，不枚举、不注册任何对象 |
| `ResetScanIterator` | 空实现 |
| `SetTestFailurePredicate` | 保留，用于部分成功测试 |




### 1.3 实现注意事项（语义不统一点）**(todo：每一条都要认真确认)**

| 关注点 | Mooncake 侧语义 | KvClient 侧语义（不统一点） | 直接照搬的后果 | 实现处理 |
| --- | --- | --- | --- | --- |
| 调用模型 | 同步，返回 `tl::expected` | 异步 task：`XxxAsync` 返回 `taskId`，`Check`/`Wait` 取结果 | 把提交当完成；`Wait` 期间持锁；缓冲提前复用 | 每批 `submit + Wait`；`Wait` 不持元数据锁；slot 结果消费后释放 |
| 失败/部分成功 | 逐 key 部分成功，返回成功数；`KEYS_ULTRA_LIMIT` 会永久关闭 offload | 聚合 `status` + 逐 entry `status`；`PARTIAL_FAILED` | 把聚合失败当整批失败；误用 `KEYS_ULTRA_LIMIT` | 只按逐 entry 判断；只上报成功 key；0 成功返回 `expected(0)` |
| 批量大小 | 一次一个 map | `Batch*Async` 受 server 上限约束 | 超限 `INVALID_ARGUMENT` | 有效批量 = `min(config.batch_size, 上限)`；切块保持下标对齐 |
| key | 变长 scoped key（`tenant\0local_key`） | 8 字节 `CacheKey`，不可逆 | 碰撞读错对象；无法反解 | `XXH64(domain + namespace + scoped_key)`；128-bit fingerprint |
| 长度 | Master `data_size` 是权威 | Query 只有存在性，没有长度 | 想从 ASU 查长度会失败 | 写时计算；读时用 `slice.size`；头里 `payload_size` 交叉校验 |
| value/内存 | `Slice` 任意内存，无需注册 | `KVBuffer` 必须引用注册内存；长度/offset 512B 对齐 | `BUFFER_NOT_REGISTERED`/`INVALID_ARGUMENT` | staging arena 注册；`transfer_size = align_up(40+payload,512)` |
| 查询 | `IsExist` bool；`ScanMeta` 可枚举 | `QueryAsync` exists 位图；无长度、无枚举 | 试图重建长度/恢复索引会失败 | `IsExist` 直接映射；`ScanMeta` 空实现 |
| task/内存生命周期 | backend 自身线程安全；Slice 调用方管理 | KvClient 内部 worker；调用方保证 task 完成前注册内存有效 | 提前 `Unregister`/复用 slot → 数据损坏 | slot 生命周期覆盖 `Wait`；shutdown 前 drain |
| 生命周期 | `Init`；无 `Shutdown`；二次 Init 失败 | `Init`/`Shutdown`；Shutdown drain task | 析构时 task 未完成 | 先 drain 再 `Shutdown`；CAS 保证二次 Init 失败 |
| 容量/配额 | `total_size_limit`、淘汰 | 无容量查询；ASU 自管 | 计数重启归零；超量写 | 计数仅参考；超配额跳过；不返回 `KEYS_ULTRA_LIMIT` |
| 删除/清空 | 基类 `RemoveAll`/`Evict` 期望物理清理 | `DeleteAsync` 逐 key，无枚举 API | 无法列出 ASU 全量对象；重启后 `RemoveAll` 只能删本进程已知 key | master-first；FIFO/索引选 victim；逐 entry `DeleteAsync`；失败恢复索引；重启限制文档化 |
| 恢复 | 启动 `ScanMeta` 重新注册 | 无枚举 | Master 重启后不可注册 | `ScanMeta` 返回空；读/查不依赖索引 |
| 完整性 | 各后端自选：CRC-32C / XXH32+verify_hash | 无 per-value checksum | 撕裂/串写/碰撞被当有效数据 | 自建 40B 头 + 头 CRC + payload CRC + fingerprint |
| 错误码 | `ErrorCode` | `StatusCode` + `ENTRY_*`/`CQE_*` | 透传错误码触发错误分支 | backend 内映射；未知归 `INTERNAL_ERROR` |

## 2. 系统上下文与模块架构

### 2.1 参与者与部署边界

| 参与者 | 运行位置 | 职责 |
| --- | --- | --- |
| Application / vLLM | 客户端进程 | 调用 Store API |
| `StoreClient` / `Client` | 客户端进程 | 路由、内存副本、RPC |
| `FileStorage` | 客户端进程 | offload/restore 编排（默认写完即下沉）、心跳、缓冲、水位淘汰 |
| `KVClientStorageBackend` | 客户端进程 | 配额、索引/FIFO、编解码、写/读/查/删；直接调用 `kv::KvClient` |
| UCM `KvClient` | 客户端进程 | ASU 连接、视图路由、task 管理、内存注册 |
| ASU View Service | 控制面 | 提供 ASU 视图/路由 |
| ASU 节点 | 进程外 | 持久化 KV，拥有自身容量/GC 策略 |
| Master | 控制面进程 | 逻辑对象与 `LOCAL_DISK` replica 元数据的权威 |



### 2.2 系统上下文图

```mermaid
flowchart LR
    App["Application / vLLM"] --> Store["StoreClient API"]
    Store --> Master["Master Service"]
    Store --> FS["FileStorage"]
    FS -->|"BatchOffload / BatchLoad / IsExist / Evict / RemoveAll"| BE["KVClientStorageBackend"]
    BE -->|complete_handler| FS
    BE -->|eviction_handler| FS
    FS -->|NotifyOffloadSuccess| Master
    FS -->|BatchEvictDiskReplica| Master
    BE -->|"store/load/query/delete (async task)"| KV["kv::KvClient"]
    KV --> View["ASU View Service"]
    KV --> ASU["ASU nodes"]
```

### 2.3 模块分层与依赖规则

```mermaid
flowchart TB
    subgraph L0["调用层（现有）"]
        FS["FileStorage"]
        Factory["CreateStorageBackend"]
    end
    subgraph L1["后端层（新增）"]
        BE["KVClientStorageBackend"]
        IDX["BackendIndex + FIFO（payload_size/淘汰顺序）"]
        CODEC["KeyCodec + RecordCodec"]
        CFG["KVClientBackendConfig"]
        STAGE["StagingArena"]
        DEL["RemoveKeys + pending delete"]
    end
    subgraph L2["UCM 接口（仅 backend .cpp）"]
        KVC["kv::KvClient"]
    end
    subgraph L3["实现"]
        UCM["CreateKvClient()"]
        MEM["MemoryKVClient（参考/测试）"]
        FAKE["FakeKvClient（测试）"]
        REMOTE["RemoteKvClient（可选）"]
    end
    subgraph L4["外部"]
        VIEW["ASU View Service"]
        ASUN["ASU nodes"]
    end
    FS --> BE
    Factory --> BE
    BE --> IDX
    BE --> CODEC
    BE --> CFG
    BE --> STAGE
    BE --> DEL
    BE --> KVC
    KVC --> UCM
    KVC --> MEM
    KVC --> FAKE
    KVC --> REMOTE
    UCM --> VIEW
    UCM --> ASUN
```

依赖规则：

- `KVClientStorageBackend` 直接依赖 `kv::KvClient`；UCM 头文件只在 `kv_client_backend.cpp`
  中包含，不进入 Mooncake 公共头。
- `FakeKvClient` / `MemoryKVClient` 实现 `kv::KvClient`，用于测试/参考。
- `RemoteKvClient` 是可选的远程服务实现；如果 ASU 侧提供 KV 服务，让服务客户端实现
  `kv::KvClient` 即可，backend 不变。
- `StagingArena` 持有 ASU 注册句柄；`UnregisterRegions` 只在 shutdown 时、所有 task 完成后调用。

### 2.4 线程模型与所有权

| 线程 | 入口 | 与后端的关系 |
| --- | --- | --- |
| Heartbeat 线程 | `Heartbeat()` → `BatchOffload` / `RunDiskWatermarkEviction` | 串行写、淘汰 |
| RPC worker | `BatchGet` → `BatchLoad` | 并发读 |
| dangling replica probe | `FileStorage::Exists` → `IsExist` | 低频并发查 |
| Master 触发的 `RemoveAll` | `FileStorage::RemoveAll` → `RemoveAll` | 低频删除 |
| 重启/重注册 | `FileStorage::Init` → `ScanMeta` | 空实现，直接返回 |
| UCM KvClient worker | `KvClientImpl::WorkerLoop` | 提交 task、等待完成；不回调 Mooncake |

锁纪律：**(todo：为什么这样设计锁，能讲明白原因？）**

1. `write_mutex_`：串行化 `BatchOffload`、`EvictAboveDiskWatermark`、`RemoveAll` 和 `Init`；
   保护跨 key 的原子序列和 `complete_handler`/`eviction_handler` 的顺序。
2. `metadata_mutex_`：读写锁，保护 `index_`、`fifo_`、`total_keys_`、`total_size_`、
   `pending_delete_`；**绝不跨 ASU `Wait` 或 Master 回调持有**。
3. `staging_mutex_`：只保护 slot 分配/释放，不覆盖 `Wait`。
4. 加锁顺序固定为 `write_mutex_` → `metadata_mutex_`。
5. 读/查只在需要更新/读取索引时短暂持有 `metadata_mutex_`；ASU I/O 期间不持锁。
6. `complete_handler` 约定不得重入本后端。

### 2.5 对象状态机

```mermaid
stateDiagram-v2
    [*] --> Absent
    Absent --> Stored: StoreAsync 成功且 Wait OK
    Stored --> Indexed: 提交 index
    Indexed --> Indexed: 覆盖写（更新 payload_size）
    Indexed --> Deleting: 淘汰 / RemoveAll 标记
    Deleting --> Absent: DeleteAsync 成功或 key 不存在
    Deleting --> Indexed: DeleteAsync 失败，恢复索引
    Indexed --> Absent: ASU 不存在且 IsExist/读 探测到
```

**状态机说明**

这张图描述的是 **backend 本地索引/删除簿记** 中单个 key 的生命周期。

### 2.6 副本 / 元数据 / 句柄：三层关系
**（todo：需要进一步了解三者的意义）**

这张图把 Mooncake 读路径上的三种投影画在一起，用来定位 `KVClientStorageBackend` 在链路中的确切位置：
**副本**回答"数据在哪里"，**元数据**回答"对象有哪些副本、多大、校验值是多少"，**句柄**回答"本进程怎么引用、何时释放"。
三者都不携带 value 字节——value 始终在共享内存、本地盘或 ASU 上。

```mermaid
flowchart TB
    subgraph M["Master 元数据（权威账本，默认纯内存）"]
        OM["ObjectMetadata / ObjectPart<br/>length_ = 权威 size<br/>checksum_"]
        R1["Replica(MEMORY)<br/>status / refcnt"]
        R2["Replica(LOCAL_DISK)<br/>client_id / object_size / transport_endpoint"]
        OM -->|"replicas_"| R1
        OM -->|"replicas_"| R2
    end

    R1 -.->|"持有（仅进程内）"| AB["AllocatedBuffer<br/>allocator / offset_handle / client_liveness"]
    R1 -->|"getDescriptorIfAvailable"| D["Replica::Descriptor<br/>id + variant + status"]
    R2 -->|"getDescriptorIfAvailable"| D

    D -->|"GetReplicaList RPC"| QR["QueryResult.replicas<br/>lease_timeout / object_checksum"]
    QR --> SEL["SelectBestReplica<br/>local MEMORY → local NOF → remote MEMORY<br/>→ remote NOF → LOCAL_DISK → DFS → DISK"]
    SEL -->|"FilterQueryResult"| P1["MEMORY / NOF"]
    SEL -->|"FilterQueryResult"| P2["LOCAL_DISK"]
    SEL -->|"FilterQueryResult"| P3["DISK / DFS"]

    P1 --> T1["TransferEngine READ / 本地 memcpy"]
    P2 --> T2["offload RPC<br/>batch_get_offload_object"]
    P3 --> T3["本地文件读 / DFS 读"]

    T1 --> H1["用户 BufferHandle（RAII）"]
    T2 --> OW["owner 进程：<br/>RealClient::batch_get_offload_object"]
    OW --> OW2["FileStorage::BatchGet → LoadBatch<br/>batch_id + AllocatedBatch"]
    OW2 -->|"storage_backend_->BatchLoad(key, slice.size)"| BE["KVClientStorageBackend"]
    BE -->|"kv::KvClient 读 ASU"| ASU["ASU"]
    BE -.->|"ScanMeta → NotifyOffloadSuccess<br/>（本版空实现，不重建账本）"| OM
    OW2 -->|"TE READ / kLocalAddress memcpy"| H2["用户 BufferHandle"]
    H2 -.->|"release_offload_buffer"| REL["FileStorage::ReleaseBuffer"]
    T3 --> H3["直接写入用户 slice"]
```

图上四条不可省略的边界：

1. **Master 从不访问客户端磁盘。** 它只保存 `Replica` 记录；`LOCAL_DISK` 分支里没有 file_path，
   只有 owner 的 `client_id` 与 `transport_endpoint`。
2. **`LOCAL_DISK` 不是"本机文件直读"。** 读方按 `transport_endpoint` 发 RPC 给 owner，由 owner 现读自己的
   本地 tier；只有当 owner 就是本进程且 pinned restore arena 可用时，才短路成
   `FileStorage::BatchGetLocal`。
3. **`KVClientStorageBackend` 只在 owner 侧的 `BatchLoad` 出现一次**（图中唯一与 ASU 相邻的节点），
   输入只有 `(scoped key, slice.size)`，不含 value 定位之外的任何上下文。
4. **`batch_id` 是唯一跨进程有生命周期的句柄。** owner 分配，reader 用完通过 `release_offload_buffer`
   归还，否则由 `client_buffer_gc_ttl_ms` 兜底回收。

三种投影的字段对照：

| 形态 | 所在位置 | 装什么 | 明确不含 |
| --- | --- | --- | --- |
| `Replica` | Master 内存 | 定位信息 + `status` + `refcnt` | value 字节 |
| `Replica::Descriptor` | RPC / `QueryResult` | `id` + 类型化定位信息 + `status` | `refcnt`、allocator 绑定、`client_liveness` |
| `ObjectMetadata` / `ObjectPart` | Master 内存 | `length_`（权威 size）、`checksum_`、`replicas_` | value 字节 |
| `StorageObjectMetadata` | backend → FileStorage → Master | `bucket_id` / `offset` / `key_size` / `data_size` / `transport_endpoint` | key 本身（在平行数组里）、value |
| `AllocatedBuffer` / `BufferHandle` | owner / 调用方进程 | 引用 + 生命周期（RAII） | value（仅通过 `ptr()` 访问） |
| `batch_id` | owner 进程 | 一批已读入 client buffer 的数据 | — |

对本后端的直接约束：

- 图中 `BatchLoad(key, slice.size)` 是唯一读入口，所以 backend 必须能**仅凭 key 与 Master 给的 size**
  定位并校验 value。ASU 的 `QueryAsync` 只有存在性、没有长度，长度必须由 backend 自己记录
  （对应 §3.6 记录布局中的 `payload_size`）。
- `ScanMeta` 是图中唯一一条从 backend 指回 Master 的虚线：它不参与稳态读写，只在启动/重注册时把
  ASU 上已有的 key/size 重新上报。本版为空实现，等价于放弃"Master 丢元数据后重建账本"的能力。
- backend 不接触 `Replica` / `Descriptor` / `BufferHandle` 中的任何一个；它看到的只有
  `StorageObjectMetadata`（上报）和 `Slice`（读写）。

## 3. 功能分解与接口契约

### 3.1 功能分解

| 编号 | 功能 | 入口 | 优先级 | 本版边界 |
| --- | --- | --- | --- | --- |
| F1 | 初始化与注册内存 | `Init` | P0 | 初始化 `kv::KvClient`、注册 staging、二次 Init 失败 |
| F2 | 写 / offload | `BatchOffload` | P0 | 单段记录；部分成功；跳过超限 key |
| F3 | 读 / restore | `BatchLoad` | P0 | 读入 staging、校验、拷贝到 Slice；索引可选 |
| F4 | 查 / 存在性 | `IsExist` | P0 | ASU Query 位图；传输错误不误报 false |
| F5 | 删 / 淘汰 | `EvictAboveDiskWatermark` / `RemoveAll` | P0 | master-first；FIFO/索引选 victim；`DeleteAsync`；失败恢复索引 |
| F6 | 配置与工厂 | `FromEnvironment` / `CreateStorageBackend` | P0 | 环境变量、校验、工厂分支 |
| F7 | staging 内存 | `StagingArena` | P0 | 注册、slot 分配、生命周期 |
| F8 | 可观测性 | 日志 / `SsdMetric` | P1 | 复用现有指标 |
| F9 | 空实现 | `ScanMeta` / `ResetScanIterator` | P0 | 返回 OK / 空操作 |


### 3.2 `StorageBackendInterface` 契约（本版）

本版实现以下方法：

```cpp
tl::expected<void, ErrorCode> Init();
tl::expected<int64_t, ErrorCode> BatchOffload(
    const std::unordered_map<std::string, std::vector<Slice>>& batch_object,
    std::function<ErrorCode(const std::vector<std::string>& keys,
                            std::vector<StorageObjectMetadata>& metadatas)>
        complete_handler,
    EvictionHandler eviction_handler = nullptr);
tl::expected<void, ErrorCode> BatchLoad(
    std::unordered_map<std::string, Slice>& batched_slices);
tl::expected<bool, ErrorCode> IsExist(const std::string& key);
tl::expected<bool, ErrorCode> IsEnableOffloading();
tl::expected<void, ErrorCode> ScanMeta(
    const std::function<ErrorCode(
        const std::vector<std::string>& keys,
        std::vector<StorageObjectMetadata>& metadatas)>& handler);
void ResetScanIterator();
tl::expected<std::vector<std::string>, ErrorCode> EvictAboveDiskWatermark(
    double high_watermark_ratio, double low_watermark_ratio,
    EvictionHandler eviction_handler = nullptr);
void RemoveAll();
void SetTestFailurePredicate(std::function<bool(const std::string& key)>);
```

内部辅助：

```cpp
struct RemoveOutcome {
    std::vector<std::string> removed;
    std::vector<std::string> missing;
    std::vector<std::string> failed;
};
tl::expected<RemoveOutcome, ErrorCode> RemoveKeys(
    const std::vector<std::string>& scoped_keys);
```

关键语义：

- `BatchOffload` 的 `keys` 必须是调用方传入的 **scoped key**（`tenant\0local_key`），不是 ASU key。
- `StorageObjectMetadata` 的 `data_size` 必须填 **payload 大小（不含 40 字节记录头）**。
- `BatchLoad` 的 `Slice` 由调用方按 `data_size` 预分配；后端不得改写 `slice.ptr`。
- `IsExist` 的“不存在”是 `expected(false)`；传输错误必须返回 `unexpected`。
- `ScanMeta` 本版返回 `OK` 且不调用 handler；`ResetScanIterator` 空实现。
- `EvictAboveDiskWatermark` 必须 master-first：`eviction_handler` 成功后才调 `DeleteAsync`；
  handler 为空或返回错误时零删除。
- `RemoveAll` 删除本进程索引中已知的 key；进程重启后索引为空，无法枚举 ASU 残留对象。
- `RemoveKeys` 逐 entry 处理：`NOT_FOUND` 视为成功；失败 key 恢复索引并保留重试。

### 3.3 直接使用 `kv::KvClient`

`KVClientStorageBackend` 直接调用 `kv::KvClient`，不再定义 Mooncake 自有的端口类型。
使用的 API：

| 用途 | `kv::KvClient` API / 类型 |
| --- | --- |
| 创建 | `std::unique_ptr<KvClient> CreateKvClient()` |
| 初始化 | `Status Init(const KvClientConfig&)` / `Status Init(const std::string& configPath)` |
| 注册内存 | `Status RegisterRegions(const std::vector<MemoryRegion>&, std::vector<RegisteredMemory>&)` |
| 注销内存 | `Status UnregisterRegions(const std::vector<MRHandle>&)` |
| 写 | `Status BatchStoreAsync(const std::vector<KVBuffer>&, TaskId&)` / `StoreAsync` |
| 读 | `Status BatchLoadAsync(const std::vector<KVBuffer>&, TaskId&)` / `LoadAsync` |
| 查 | `Status QueryAsync(const std::vector<CacheKey>&, TaskId&)` |
| 删 | `Status DeleteAsync(const std::vector<CacheKey>&, TaskId&)` |
| 等待 | `bool Check(TaskId)` / `Status Wait(TaskId, uint64_t timeoutMs, TaskResult&)` |
| 关停 | `Status Shutdown()` |
| key | `CacheKey = std::array<std::byte, 8>` |
| 值 | `KVBuffer{key, Buffer{MemoryRegion, MRHandle}, offset, std::optional<uint32_t> mrKey}` |
| 结果 | `TaskResult{Status status, vector<Status> entryStatus, optional<QueryResult> queryResult}` |

约束：

- `KVBuffer.region.size` 是本次传输长度，`region.size` 和 `offset` 都必须 512B 对齐。
- `QueryResult.exists` 是存在性位图，没有长度。
- `TaskResult.status` 是聚合状态；必须逐 `entryStatus` 判断成功/失败。
- `kv::Status` / `StatusCode` 在 backend 内映射为 Mooncake `ErrorCode`，不向外透传。
- `KvClient` 是纯虚接口；测试用 `FakeKvClient : public kv::KvClient` 注入，不需要生产 .so。
- UCM 头文件只在 `kv_client_backend.cpp` 中包含；backend 头文件只前置声明
  `namespace kv { class KvClient; }`。

### 3.4 错误码映射

| `kv::StatusCode`（旧分支 `UC::ASU::StatusCode`）/ 本地条件 | Mooncake `ErrorCode` | 说明 |
| --- | --- | --- |
| `NOT_FOUND`, `ENTRY_KEY_NOT_FOUND`, `ENTRY_DATA_NOT_EXIST`, `CQE_KEY_NOT_EXIST` | `OBJECT_NOT_FOUND` | 读时返回；查时 `false` |
| `INVALID_ARGUMENT` | `INVALID_PARAMS` | |
| `NOT_INITIALIZED` | `INTERNAL_ERROR` | |
| `TIMEOUT`, `CQE_IO_TIMEOUT` | `RPC_TIMEOUT` | 不无限重试 |
| `PARTIAL_FAILED` | 首个 entry 错误 | 部分成功继续 |
| `CONNECTION_ERROR` | `UNAVAILABLE_IN_CURRENT_STATUS` | |
| `IO_ERROR`, `CQE_WRITE_FAULT` | `FILE_WRITE_FAIL` | |
| `CQE_UNRECOVERED_READ_ERROR` | `FILE_READ_FAIL` | |
| `BUFFER_NOT_REGISTERED`, `BUFFER_NOT_SUPPORTED` | `INVALID_PARAMS` | staging 生命周期错误 |
| `RESOURCE_BUSY`, `IN_PROGRESS` | `UNAVAILABLE_IN_CURRENT_STATUS` | 可重试 |
| `CQE_OUT_OF_CREATE_SIZE`, `CQE_RESOURCE_BUSY` | `UNAVAILABLE_IN_CURRENT_STATUS` | ASU 容量/资源不足；本批跳过 |
| `UNSUPPORTED` | `NOT_SUPPORTED` | |
| `CANCELED` | `INTERNAL_ERROR` | |
| `ENTRY_RETRY_ADVISED` | 重试一次；仍失败按对应错误 | |
| `ENTRY_NO_RETRY_ADVISED` | `INTERNAL_ERROR` | |
| `ENTRY_KEY_EXIST`, `CQE_KEY_ALREADY_EXISTED` | 视为成功（upsert） | 本版不依赖条件写 |
| `ENTRY_DELETE_FAILED` | `FILE_WRITE_FAIL` | 删除失败；恢复索引 |
| 记录头 magic/version/length/fingerprint 不符 | `INVALID_READ` | 内容损坏或 key 碰撞 |
| 记录 CRC-32C 不符 | `CHECKSUM_MISMATCH` | 撕裂/位翻转 |
| 本地 `ptr == nullptr` / `size == 0` | `INVALID_PARAMS` | 读 |

### 3.5 语义规约

| 属性 | 规约 |
| --- | --- |
| 写原子性 | 单 key 一个 ASU task 条目；ASU 负责该条目的可见性 |
| 写幂等 | 同 key 同 payload 重复 Store 字节确定；Store 是 upsert |
| 批量原子性 | 无事务；`PARTIAL_FAILED`/逐 entry 失败是正常结果 |
| 读一致性 | 返回最近一次成功 Store 的完整 value；CRC 检测撕裂 |
| 读的输入 | key + `slice.size` 由 Master 提供；索引缺失时以 `slice.size` 为准 |
| 查语义 | 以 ASU `QueryResult.exists` 为准；索引不参与 |
| 长度来源 | Master metadata（读）、写路径计算（写）；ASU 不提供长度 |
| 可见性 | `complete_handler` 成功之前，Master 不知道该副本 |
| 容量计数 | `total_size_` 计 `align_up(40 + payload, 512)`；仅统计本进程写入，重启后从 0 开始 |
| 删除 | master-first；FIFO 水位选 victim；`DeleteAsync` 逐 key；`NOT_FOUND` 视为成功；失败恢复索引 |
| RemoveAll | 只删除本进程索引已知 key；重启后无法枚举 ASU 残留对象 |
| 恢复 | 本版不恢复；`ScanMeta` 返回 OK 空扫描 |
| 二次 Init | 返回 `INTERNAL_ERROR` |

### 3.6 Key 与记录布局

**ASU key**

```text
key64 = XXH64("mooncake/kvclient/v1" + namespace_id + 0x00 + scoped_key, seed=0)
```

- ASU key 固定 8 字节，按小端写入 `kv::CacheKey`。
- `namespace_id` 由配置必填；它参与哈希，不同命名空间的同一 scoped key 得到不同 ASU key。
- 64-bit 碰撞概率约 `N^2 / 2^65`；1 亿对象约 `5e-4`。记录头存 128-bit fingerprint 检测碰撞。

**记录布局（40 字节头 + payload）**

```text
0   u32 magic          0x4B564331 ("KVC1")
4   u16 version        1
6   u16 flags          bit0 chunked（本版必须 0）; bit1 has_payload_crc
8   u64 fingerprint_lo XXH64(scoped_key, seed=1)
16  u64 fingerprint_hi XXH64(scoped_key, seed=2)
24  u32 payload_size   单 value <= UINT32_MAX
28  u32 reserved       0
32  u32 payload_crc32c
36  u32 header_crc32c  对 [0,36) 计算，本字段按 0 参与
40  ... payload        各 Slice 按序拼接，无补齐
```

- `record_size = 40 + payload_size`；`transfer_size = align_up(record_size, 512)`。
- `KVBuffer`：`key = key64`、`offset = 0`、`region.size = transfer_size`、`handle = slot.handle`。
- 上报 Master 的 `data_size = payload_size`。
- 读校验顺序：`transfer_size >= 40` → magic → version → flags → `payload_size` 在
  `transfer_size - 40` 以内 → header CRC → fingerprint → payload CRC。
- 单对象上限：`min(config.max_value_size, ASU 单值上限) - 40`，再受 staging slot 大小约束；
  超限 key 在写路径跳过并记录。

**CRC-32C 的定位**：`mooncake-store/include/crc32c.h` 是 Mooncake 自己的 CRC-32C 实现；
它是标准算法，但不是 ASU 或 Mooncake 的全局强制格式。`NvmeKvStorageBackend` 使用
XXH32 + 32 字节 `verify_hash`，与本后端的 key/记录格式不互通。ASU 公共 `KvClient` 没有
per-value checksum；transport 校验不覆盖 at-rest 数据。因此本后端读到的每条 value 都必须按
`KVC1` 记录格式校验。

### 3.7 配置契约

**（这里的配置应该明确为KVClientStorageBackendConfig）**

```cpp
struct KVClientBackendConfig {
    std::string client_type;          // "ucm"（生产）/ "memory"（参考实现）
    std::string kv_client_config_path; // KvClient 的 view/transport 配置文件
    std::string client_id;            // KvClientConfig.clientId
    std::string namespace_id;         // 必填，参与 ASU key 哈希
    size_t staging_bytes = 128 * 1024 * 1024;
    size_t batch_size = 32;
    int64_t max_value_size = 16 * 1024 * 1024;
    int64_t wait_timeout_ms = 5000;
    int64_t query_timeout_ms = 500;

    static KVClientBackendConfig FromEnvironment();
    bool Validate() const;
};
```

环境变量：

| 变量 | 字段 |
| --- | --- |
| `MOONCAKE_KV_CLIENT_TYPE` | `client_type` |
| `MOONCAKE_KV_CLIENT_CONFIG_PATH` | `kv_client_config_path` |
| `MOONCAKE_KV_CLIENT_ID` | `client_id` |
| `MOONCAKE_KV_CLIENT_NAMESPACE` | `namespace_id` |
| `MOONCAKE_KV_CLIENT_STAGING_BYTES` | `staging_bytes` |
| `MOONCAKE_KV_CLIENT_BATCH_SIZE` | `batch_size` |
| `MOONCAKE_KV_CLIENT_MAX_VALUE_SIZE` | `max_value_size` |
| `MOONCAKE_KV_CLIENT_WAIT_TIMEOUT_MS` | `wait_timeout_ms` |
| `MOONCAKE_KV_CLIENT_QUERY_TIMEOUT_MS` | `query_timeout_ms` |

**`FileStorageConfig::Validate()` 的配套修改**：

- `kFilePerKey` / `kBucket` / `kOffsetAllocator` / `kDistributed`：保持现有本地路径校验。
- `kKVClient`：跳过 `ValidatePath`；`KVClientBackendConfig::Validate()` 只校验
  `namespace_id`、`staging_bytes`、`batch_size` 等。
- `total_keys_limit > 0` 与 `total_size_limit != 0` 仍要求。

### 3.8 能力与限制矩阵（本版）

| 能力/限制 | 本版处理 |
| --- | --- |
| key 8 字节 | 哈希映射 + 128-bit fingerprint 检测碰撞 |
| 无长度查询 | 写时计算；读时用 Master 的 `slice.size` |
| 无枚举 | `ScanMeta` 返回 OK 空扫描；不做恢复 |
| 512B 对齐 | `transfer_size = align_up(record_size, 512)`；`offset=0` |
| 注册内存 | staging arena；读写都经过 staging |
| 单 value 上限 | `min(config.max_value_size, ASU 上限) - 40` |
| 批量 key 上限 | `min(config.batch_size, ASU 上限)` |
| 异步 task | 每批提交后 `Wait`；超时 `RPC_TIMEOUT` |
| ASU 自身 GC | 无法观测；查/读遇到 `NOT_FOUND` 按缺失处理 |
| 删除/淘汰 | master-first + FIFO 水位；`DeleteAsync` 逐 key；失败恢复索引；重启后 `RemoveAll` 仅删已知 key |

## 4. 功能实现设计

### 4.1 公共类图

```mermaid
classDiagram
    class StorageBackendInterface {
        <<interface>>
        +Init() expected
        +BatchOffload(batch, completeHandler, evictionHandler) int64
        +BatchLoad(slices) expected
        +IsExist(key) expected
        +IsEnableOffloading() expected
        +ScanMeta(handler) expected
        +ResetScanIterator() void
        +RemoveAll() void
        +EvictAboveDiskWatermark(high, low, handler) expected
        +SetTestFailurePredicate(predicate) void
    }

    class KVClientStorageBackend {
        -config_ KVClientBackendConfig
        -client_ KvClient
        -staging_ StagingArena
        -write_mutex_ stdMutex
        -metadata_mutex_ sharedMutex
        -staging_mutex_ stdMutex
        -index_ BackendIndex
        -fifo_ map
        -pending_delete_ vector
        -total_keys_ int64
        -total_size_ int64
        -initialized_ bool
        +Init() expected
        +BatchOffload(batch, completeHandler, evictionHandler) int64
        +BatchLoad(slices) expected
        +IsExist(key) expected
        +IsEnableOffloading() expected
        +ScanMeta(handler) expected
        +ResetScanIterator() void
        +RemoveAll() void
        +EvictAboveDiskWatermark(high, low, handler) expected
        -PrepareBatch(batch) PreparedBatch
        -CommitIndex(scopedKey, payloadSize)
        -SelectVictims(targetBytes) vector
        -RemoveKeys(scopedKeys) RemoveOutcome
        -RestoreIndex(entries)
    }

    class KvClient {
        <<interface>>
        +Init(config) Status
        +Shutdown() Status
        +QueryAsync(keys, taskId) Status
        +StoreAsync(entries, taskId) Status
        +BatchStoreAsync(entries, taskId) Status
        +LoadAsync(entries, taskId) Status
        +BatchLoadAsync(entries, taskId) Status
        +DeleteAsync(keys, taskId) Status
        +Check(taskId) bool
        +Wait(taskId, timeoutMs, result) Status
        +RegisterRegions(regions, registered) Status
        +UnregisterRegions(handles) Status
    }

    class UcmKvClient {
        +UcmKvClient()
        +Init(config) Status
        +BatchStoreAsync(entries, taskId) Status
        +BatchLoadAsync(entries, taskId) Status
        +QueryAsync(keys, taskId) Status
        +DeleteAsync(keys, taskId) Status
        +Wait(taskId, timeoutMs, result) Status
    }

    class KVClientBackendConfig {
        +client_type string
        +kv_client_config_path string
        +client_id string
        +namespace_id string
        +staging_bytes size
        +batch_size size
        +max_value_size int64
        +wait_timeout_ms int64
        +query_timeout_ms int64
        +FromEnvironment() KVClientBackendConfig
        +Validate() bool
    }

    class KVClientKeyCodec {
        +EncodeKey(namespaceId, scopedKey) CacheKey
        +Fingerprint(scopedKey) Fingerprint
    }

    class KVClientRecordCodec {
        +Encode(scopedKey, slices) Record
        +Parse(buffer, expectedFingerprint) ParsedRecord
        +TransferSize(recordSize) size
    }

    class BackendIndex {
        +Upsert(scopedKey, entry) Delta
        +Snapshot(scopedKey) Entry
        +Remove(scopedKey) Entry
        +OldestVictims(bytes) vector
        +TotalKeys() int64
        +TotalSize() int64
    }

    class StagingArena {
        +Init(bytes, slotSize) expected
        +Acquire() Slot
        +Release(slot) void
        +RegisteredRegions() vector
        +Shutdown() void
    }

    class MemoryKVClient {
        +Init(config) Status
        +BatchStoreAsync(entries, taskId) Status
        +BatchLoadAsync(entries, taskId) Status
        +QueryAsync(keys, taskId) Status
        +DeleteAsync(keys, taskId) Status
        +Wait(taskId, timeoutMs, result) Status
    }

    class FakeKvClient {
        +FailNext(op, key)
        +DelayNext(op, ms)
        +RecordCalls()
    }

    StorageBackendInterface <|-- KVClientStorageBackend
    KVClientStorageBackend o-- KvClient
    KVClientStorageBackend o-- KVClientBackendConfig
    KVClientStorageBackend *-- BackendIndex
    KVClientStorageBackend *-- StagingArena
    KVClientStorageBackend ..> KVClientKeyCodec
    KVClientStorageBackend ..> KVClientRecordCodec
    KvClient <|.. UcmKvClient
    KvClient <|.. MemoryKVClient
    KvClient <|.. FakeKvClient
```

### 4.2 写：`BatchOffload`

#### 4.2.1 写操作类图

```mermaid
classDiagram
    class FileStorage {
        +Heartbeat()
        +OffloadObjects(batch)
        -BuildOffloadTasksFromStorageKeys(keys, metadatas)
    }
    class KVClientStorageBackend {
        +BatchOffload(batch, completeHandler, evictionHandler) int64
        -PrepareBatch(batch) PreparedBatch
        -CommitIndex(scopedKey, payloadSize)
    }
    class KVClientKeyCodec {
        +EncodeKey(namespaceId, scopedKey) CacheKey
        +Fingerprint(scopedKey) Fingerprint
    }
    class KVClientRecordCodec {
        +Encode(scopedKey, slices) Record
    }
    class StagingArena {
        +Acquire() Slot
        +Release(slot) void
    }
    class KvClient {
        +BatchStoreAsync(entries, taskId) Status
        +Wait(taskId, timeoutMs, result) Status
    }
    class BackendIndex {
        +Upsert(scopedKey, entry) Delta
    }
    FileStorage --> KVClientStorageBackend : calls
    KVClientStorageBackend --> KVClientKeyCodec : encode key
    KVClientStorageBackend --> KVClientRecordCodec : encode record
    KVClientStorageBackend --> StagingArena : acquire
    KVClientStorageBackend --> KvClient : store and wait
    KVClientStorageBackend --> BackendIndex : commit
```

#### 4.2.2 写时序图

```mermaid
sequenceDiagram
    autonumber
    participant HB as HeartbeatThread
    participant FS as FileStorage
    participant BE as Backend
    participant RC as RecordCodec
    participant SA as StagingArena
    participant KC as KvClient
    participant MS as Master

    HB->>FS: Heartbeat()
    FS->>BE: BatchOffload(scopedBatch, completeHandler, evictionHandler)
    BE->>BE: check initialized, empty, quota

    loop each scoped_key
        BE->>RC: Encode(scoped_key, slices)
        RC-->>BE: record, payload_size, transfer_size
    end

    BE->>BE: drop oversized / over-quota keys
    BE->>SA: Acquire(slots)
    SA-->>BE: registered slots
    BE->>BE: copy record into each slot, zero padding to 512B

    BE->>KC: BatchStoreAsync(entries)
    KC-->>BE: taskId
    BE->>KC: Wait(taskId, wait_timeout_ms)
    KC-->>BE: TaskResult with per-entry status

    alt entry success
        BE->>BE: CommitIndex
    else entry failed
        BE->>BE: record first error, skip key
    end

    BE->>SA: Release(slots)
    BE->>FS: completeHandler(success_scoped_keys, metadatas)
    FS->>MS: NotifyOffloadSuccess(tasks, metadatas)
    MS-->>FS: OK or error

    alt handler returned error
        BE-->>FS: unexpected(handler_error)
    else partial success
        BE-->>FS: success_count
    end
```

#### 4.2.3 实现步骤

触发时机不在后端：默认 `offload_on_evict=false`，Master 在 `PutEnd` 后就把对象放进 owner 的
offload 队列，下一次心跳调用本方法；`offload_on_evict=true` 时同一方法在淘汰时刻被调用。

1. `initialized_` 为假 → `INTERNAL_ERROR`。
2. `batch_object` 为空 → `INVALID_KEY`。
3. `IsEnableOffloading()` 为假 → 本批全部跳过，返回 `expected(0)`；不返回
   `KEYS_ULTRA_LIMIT`。
4. 逐 key 预处理：
   - `scoped_key` 为空 → 跳过并记录 `INVALID_KEY`。
   - 计算 8 字节 `key64` 和 128-bit fingerprint；payload 超
     `effective_max_value_size - 40`、超 `UINT32_MAX`、或 `transfer_size` 超过 staging slot
     → 跳过并记录 `INVALID_PARAMS`。
   - 拼接 Slice 为连续 payload；生成 40 字节记录头；尾部补零到 `transfer_size`。
   - 若 `total_size_ + transfer_size > total_size_limit` 或
     `total_keys_ >= total_keys_limit`，跳过该 key；本版不做淘汰来腾空间。
5. 从 `StagingArena` 分配 slot；把记录拷入 slot 的已注册内存。
6. 按 `min(config.batch_size, ASU 上限)` 切块：
   - 多 key 用 `BatchStoreAsync`，单 key 用 `StoreAsync`。
   - 每块提交后 `Wait(task_id, wait_timeout_ms, result)`。
   - 逐 entry 检查 `entryStatus`；聚合 `PARTIAL_FAILED` 不当作整批失败。
7. 对每个成功 entry，在 `metadata_mutex_` 独占锁内提交索引：
   - `old = index_[scoped_key]`；`total_size_ += transfer_size - old.transfer_size`；
     `total_keys_ += (old 存在 ? 0 : 1)`。
8. 只把成功 key 上报：

   ```cpp
   StorageObjectMetadata{
       .bucket_id = -1,
       .offset = 0,
       .key_size = static_cast<int64_t>(scoped_key.size()),
       .data_size = static_cast<int64_t>(payload_size),  // 不含 40B 头
       .transport_endpoint = "",
   }
   ```

9. `complete_handler` 返回非 OK → 原样返回该错误；已写入的数据不回滚。
10. 返回成功上报的 key 数量；若 0 个成功，仍返回 `expected(0)`。

失败处理：

- 单个 entry 的 ASU 错误：跳过，保留首错用于日志；不影响其他 entry。
- `Wait` 超时：记录 `RPC_TIMEOUT`，不通知 Master；ASU 侧可能稍后完成，形成一个未被 Master
  登记的孤儿对象。重试 Put 会覆盖它。
- `StagingArena` 分配失败/耗尽：本批剩余 key 跳过，返回已成功数量；不长期阻塞心跳。
- 超限 key 会被反复选中：进程内维护有界“跳过集合”，避免重复编码和日志。
- `BatchOffload` 本身不删除 key；删除由 `EvictAboveDiskWatermark`/`RemoveAll` 触发（§4.5），
  `eviction_handler` 只用于淘汰路径。

### 4.3 读：`BatchLoad`

读路径在 Mooncake 侧的完整分层（副本 / 元数据 / 句柄、owner 侧 offload RPC 与 client buffer）见 §2.6；
本节只展开图中 `KVClientStorageBackend` 这一个节点。

#### 4.3.1 读操作类图

```mermaid
classDiagram
    class FileStorage {
        +BatchGetLocal(keys, sizes)
        +LoadBatch(keys, sizes)
    }
    class KVClientStorageBackend {
        +BatchLoad(slices) expected
        -LookupIndex(scopedKey) optional
        -ValidateRecord(buffer, fingerprint) ParsedRecord
        -CommitIndex(scopedKey, payloadSize)
    }
    class BackendIndex {
        +Snapshot(scopedKey) Entry
        +Upsert(scopedKey, entry) Delta
        +Remove(scopedKey) Entry
    }
    class StagingArena {
        +Acquire() Slot
        +Release(slot) void
    }
    class KvClient {
        +BatchLoadAsync(entries, taskId) Status
        +Wait(taskId, timeoutMs, result) Status
    }
    class KVClientRecordCodec {
        +Parse(buffer, expectedFingerprint) ParsedRecord
    }
    FileStorage --> KVClientStorageBackend : calls
    KVClientStorageBackend --> BackendIndex : optional cache
    KVClientStorageBackend --> StagingArena : acquire
    KVClientStorageBackend --> KvClient : load and wait
    KVClientStorageBackend --> KVClientRecordCodec : validate
```

#### 4.3.2 读时序图

```mermaid
sequenceDiagram
    autonumber
    participant RPC as RpcWorker
    participant FS as FileStorage
    participant BE as Backend
    participant SA as StagingArena
    participant KC as KvClient
    participant RC as RecordCodec

    RPC->>FS: BatchGet(keys, sizes)
    FS->>BE: BatchLoad(batched_slices)

    loop each key
        BE->>BE: validate ptr and slice.size
        BE->>BE: optional index snapshot, release
        BE->>BE: payload_size = slice.size, transfer_size = align_up(40 + payload_size, 512)
    end

    BE->>SA: Acquire(slots)
    BE->>KC: BatchLoadAsync(entries, transfer_size)
    KC-->>BE: taskId
    BE->>KC: Wait(taskId, wait_timeout_ms)
    KC-->>BE: TaskResult with per-entry status

    loop each loaded entry
        alt not found
            BE->>BE: remove index entry if present
            BE-->>FS: OBJECT_NOT_FOUND (first_error)
        else header or fingerprint invalid
            BE-->>FS: INVALID_READ (first_error)
        else CRC mismatch
            BE-->>FS: CHECKSUM_MISMATCH (first_error)
        else valid
            BE->>RC: Parse(buffer, fingerprint)
            BE->>FS: memcpy payload into slice.ptr
            BE->>BE: CommitIndex (optional cache)
        end
    end

    BE->>SA: Release(slots)
    BE-->>FS: OK or first_error
```

#### 4.3.3 实现步骤

1. 空 map → `OK`。
2. 对每个输入 key：
   - `slice.ptr == nullptr` 或 `slice.size == 0` → `INVALID_PARAMS`。
   - **`payload_size = slice.size`**（Master 提供的权威长度）；`transfer_size =
     align_up(40 + payload_size, 512)`；超过 staging slot → `INVALID_PARAMS`。
   - 索引是可选缓存：命中时可用于日志/统计；**不要求命中**。
3. 分配 staging slot，构造 `KVBuffer{key, offset=0, region.size=transfer_size, handle}`。
4. 按 `batch_size` 切块，`BatchLoadAsync` + `Wait`；逐 entry 检查状态。
5. 逐结果校验：
   - `NOT_FOUND`：从索引删除（若存在），记 `OBJECT_NOT_FOUND`。
   - `TIMEOUT`/`CONNECTION_ERROR`/`IO_ERROR`：映射后记首错。
   - `transfer_size < 40`：`INVALID_READ`。
   - magic/version/flags 不符：`INVALID_READ`。
   - `payload_size` 超出 buffer 或与 `slice.size` 不一致：`INVALID_PARAMS`。
   - header CRC 不符：`INVALID_READ`。
   - fingerprint 不符：`INVALID_READ`（key 碰撞或串写）。
   - payload CRC 不符：`CHECKSUM_MISMATCH`。
   - 全部通过：`memcpy(slot + 40, slice.ptr, payload_size)`，并按 `payload_size` 更新索引。
6. 释放 slot；返回 `OK` 或首个错误；已经写入的 slice 不回滚。**不改写 `slice.ptr`**。

**为什么需要两层 CRC 和 fingerprint**：

- 头 CRC-32C 保护“怎么读”：magic/version/flags/payload_size 决定拷贝长度和记录类型。
- payload CRC-32C 保护“读到的内容”：检测撕裂、位翻转、写入中断。
- fingerprint 不是校验和，而是身份校验：ASU key 只有 64 bit，两个 scoped key 可能碰撞。
- CRC-32C 只检测偶然损坏，不防恶意篡改；512B padding 不参与 payload CRC。

### 4.4 查：`IsExist`

#### 4.4.1 查询类图

```mermaid
classDiagram
    class FileStorage {
        +Exists(scopedKey) expected
    }
    class KVClientStorageBackend {
        +IsExist(scopedKey) expected
        -RemoveIndexEntry(scopedKey)
    }
    class KvClient {
        +QueryAsync(keys, taskId) Status
        +Wait(taskId, timeoutMs, result) Status
    }
    class BackendIndex {
        +Remove(scopedKey) Entry
    }
    FileStorage --> KVClientStorageBackend : calls
    KVClientStorageBackend --> KvClient : query and wait
    KVClientStorageBackend --> BackendIndex : remove stale entry
```

#### 4.4.2 查询时序图

```mermaid
sequenceDiagram
    autonumber
    participant P as DanglingReplicaProbe
    participant FS as FileStorage
    participant BE as Backend
    participant KC as KvClient

    P->>FS: Exists(scoped_key)
    FS->>BE: IsExist(scoped_key)
    BE->>BE: EncodeKey

    BE->>KC: QueryAsync(one key)
    KC-->>BE: taskId
    BE->>KC: Wait(taskId, query_timeout_ms)
    KC-->>BE: TaskResult with exists bitmap

    alt not found
        BE->>BE: remove stale index entry
        BE-->>FS: expected(false)
    else found
        BE-->>FS: expected(true)
    else transport error
        BE-->>FS: unexpected(mapped_error)
    end
```

#### 4.4.3 实现步骤

1. 未初始化 → `INTERNAL_ERROR`。
2. 编码 8 字节 ASU key。
3. `QueryAsync({key})` + `Wait(query_timeout_ms)`：
   - `exists[0] == 0` → 删除索引条目（若存在）→ `expected(false)`。
   - `exists[0] != 0` → `expected(true)`。
4. 传输/超时/IO 错误 → `unexpected(mapped)`，**绝不能**返回 `false`。
5. 本版不做懒修复：Query 只返回存在性，没有长度；读路径以 Master 的 `slice.size` 为准。

### 4.5 删：`EvictAboveDiskWatermark` / `RemoveAll`

#### 4.5.1 删除类图

```mermaid
classDiagram
    class FileStorage {
        +RunDiskWatermarkEviction() expected
        +NotifyEvictedDiskReplicas(keys) expected
        +RemoveAll() void
    }
    class KVClientStorageBackend {
        +EvictAboveDiskWatermark(high, low, handler) expected
        +RemoveAll() void
        -SelectVictims(targetBytes) vector
        -RemoveKeys(scopedKeys) RemoveOutcome
        -RestoreIndex(entries)
    }
    class BackendIndex {
        +OldestVictims(bytes) vector
        +Remove(scopedKey) Entry
        +Upsert(scopedKey, entry) Delta
    }
    class KvClient {
        +DeleteAsync(keys, taskId) Status
        +Wait(taskId, timeoutMs, result) Status
    }
    class Master {
        +BatchEvictDiskReplica(keys, tenant) results
    }
    FileStorage --> KVClientStorageBackend : calls
    KVClientStorageBackend --> BackendIndex : mark and remove
    KVClientStorageBackend --> KvClient : delete and wait
    FileStorage --> Master : eviction_handler
```

#### 4.5.2 删除时序图

```mermaid
sequenceDiagram
    autonumber
    participant HB as HeartbeatThread
    participant FS as FileStorage
    participant BE as Backend
    participant KC as KvClient
    participant MS as Master

    HB->>FS: RunDiskWatermarkEviction()
    FS->>BE: EvictAboveDiskWatermark(high, low, handler)
    BE->>BE: write_mutex_ lock, select FIFO victims

    alt no victim
        BE-->>FS: empty
    else handler is null
        BE->>BE: warn, do not delete
        BE-->>FS: empty
    else handler present
        BE->>FS: handler(victim_scoped_keys)
        FS->>MS: BatchEvictDiskReplica(keys, tenant) per tenant
        MS-->>FS: per-key results
        alt handler returned error
            FS-->>BE: error
            BE-->>FS: unexpected, keep local keys for retry
        else handler OK
            BE->>BE: exclusive lock, remove index entries
            BE->>KC: DeleteAsync(victim keys)
            KC-->>BE: taskId
            BE->>KC: Wait(taskId, wait_timeout_ms)
            KC-->>BE: TaskResult with per-entry status
            alt entry deleted or not found
                BE->>BE: erase index, subtract counters
            else entry failed
                BE->>BE: restore index entry
            end
            BE-->>FS: removed_keys
        end
    end
```

#### 4.5.3 淘汰实现步骤

1. 配置未开启 `enable_disk_watermark_eviction` → 返回空。
2. 获取 `write_mutex_`。
3. `used = total_size_`；若 `used <= high_ratio * total_size_limit` → 返回空。
4. 在 `metadata_mutex_` 共享锁内按 FIFO 选择 victim，直到 `used <= low_ratio * total_size_limit`
   或没有候选；跳过 `deleting` 条目和 `fifo_seq` 已失效的惰性槽位。key 数水位作为附加条件。
5. `eviction_handler == nullptr` → 记录 WARNING，返回空，**不删除任何数据**。
6. 调用 `handler(victim_scoped_keys)`：
   - 返回 OK → 继续。
   - 返回错误 → 返回 `unexpected`，本地索引保持不变；下一轮重试。Master 侧
     `OBJECT_NOT_FOUND` 视为成功，因此跨 tenant 的部分成功会收敛。
7. 在 `metadata_mutex_` 独占锁内把 victim 标记为 `deleting` 并移出可淘汰集合；保留
   `pending_delete_` 中的 `Entry` 副本用于失败恢复。
8. 按 `batch_size` 切块调用 `DeleteAsync` + `Wait`：
   - 成功或 `NOT_FOUND` → 从索引删除，`total_size_ -= transfer_size`、`total_keys_--`。
   - 失败 → 恢复索引条目（若期间没有新的同名写覆盖），保留在 FIFO 的重试位置。
9. 返回实际删除的 scoped key 列表。

崩溃窗口：Master 已接受淘汰、`DeleteAsync` 前崩溃，重启后 `ScanMeta` 为空、索引为空，无法发现
这些残留对象；它们可能被后续写覆盖。这是本版接受的“删除复活窗口”，关闭它需要 tombstone 或
持久化索引（后续版本）。

#### 4.5.4 `RemoveAll` 实现步骤

1. 获取 `write_mutex_`。
2. 从 `index_` 复制本进程已知的 scoped keys；**没有枚举 ASU 的能力**。
3. 分块 `RemoveKeys`：成功/`NOT_FOUND` 删除索引并扣减计数；失败保留索引。
4. 失败只记日志（接口返回 `void`），下一轮 `RemoveAll`/`IsExist` 收敛。
5. 重启后索引为空，`RemoveAll` 不会删除 ASU 中已有的旧对象；这是本版的明确限制。
6. 不做租户级物理隔离；`FileStorage::RemoveAll` 的现有全局 wipe 语义保持不变。

### 4.6 `Init` 与空实现

#### 4.6.1 初始化时序图

```mermaid
sequenceDiagram
    autonumber
    participant FS as FileStorage
    participant BE as Backend
    participant KC as KvClient
    participant SA as StagingArena

    FS->>BE: Init()
    BE->>KC: CreateKvClient()
    BE->>KC: Init(kv_client_config_path)
    KC-->>BE: Status
    BE->>SA: Init(staging_bytes, slot_size)
    SA->>KC: RegisterRegions(staging)
    KC-->>SA: registered handles
    SA-->>BE: handles

    FS->>BE: ScanMeta(handler)
    BE->>BE: return OK without invoking handler
    BE-->>FS: OK
```

#### 4.6.2 `Init`

- `initialized_` 用 `compare_exchange(false, true)`：第二次 `Init` 返回 `INTERNAL_ERROR`。
- 校验 `KVClientBackendConfig::Validate()`；`client_ = kv::CreateKvClient()`；按
  `kv_client_config_path` 或 `KvClientConfig` 初始化。
- 计算 slot size：`align_up(40 + max_payload, 512)`；slot 数 = `staging_bytes / slot_size`，
  至少 1，且不超过 `max_batch_size`。`StagingArena::Init` 分配对齐内存并 `RegisterRegions`。
- 计数置 0，索引为空。
- 若 `KvClient::Init` 失败或注册内存失败，返回对应错误；不要以空索引运行。

#### 4.6.3 `ScanMeta` / `ResetScanIterator` / 恢复限制

- `ScanMeta`：返回 `expected<void>` OK，不调用 handler，不枚举 ASU；记一次 WARNING。
- `ResetScanIterator`：空实现。
- `RemoveAll`/`EvictAboveDiskWatermark` 已在 §4.5 实现；`RemoveAll` 只能删除本进程索引已知的
  key。
- 结果：进程重启后，Master 若仍有 `LOCAL_DISK` 元数据，读/查仍可用；若 Master 也丢失元数据，
  对象不会自动重新注册。这是本版的明确限制。

### 4.7 并发与一致性（写/读/查/删）

| 场景 | 结果 | 处理 |
| --- | --- | --- |
| Store 成功、handler 失败 | 数据在 ASU、本进程索引有记录、Master 不知道 | 不恢复；重试 Put 覆盖 |
| Store 成功、Wait 超时 | ASU 可能稍后完成 | 不通知 Master；孤儿对象；重试覆盖 |
| 覆盖写 | 单 key 一条 ASU value 原子替换；索引按 delta 更新 | last-writer-wins |
| 读与写并发 | 读到旧值或新值 | 二者都是完整对象；CRC 保证完整性 |
| 读时索引缺失 | 仍可读 | 以 Master 的 `slice.size` 为准；成功后补索引 |
| ASU 自身 GC/删除 | 索引暂时陈旧 | 读/查遇到 `NOT_FOUND` 时删索引 |
| 删除与并发读 | 读到旧值或 `OBJECT_NOT_FOUND` | 不允许半对象；CRC 保证完整性 |
| Master 已接受淘汰、`DeleteAsync` 前崩溃 | ASU 残留对象 | 重启后索引为空，无法发现；本版接受的复活窗口 |
| `DeleteAsync` 失败 | 索引恢复、计数不变 | 保留 FIFO 重试；下一轮再删 |
| `RemoveAll` 后重启 | 只能删除索引已知 key | 旧对象残留；文档化限制 |
| 重复删除 | `NOT_FOUND` 视为成功 | 幂等 |
| 进程重启 | 索引为空 | 读/查不依赖索引；写重新累计计数 |
| 二次 `Init` | `INTERNAL_ERROR` | 与现有共享契约一致 |
| 多客户端共享 ASU | key 哈希包含 namespace | 逻辑隔离；namespace 唯一性由部署保证 |

### 4.8 可观测性

- 复用 `SsdMetric`：`FileStorage` 已记录 read/write ops、bytes、latency，本版不新增指标文件。
- 结构化日志字段：`backend=kvclient`、`namespace`、`op`、`keys`、`entries`、`success`、
  `skipped`、`failed`、`first_error`、`task_id`。
- 建议的后端计数器（本版只记日志）：提交 task 数、Wait 超时数、逐 entry 失败数、
  staging 等待时间。

## 5. SFMEA 分析与测试设计

### 5.1 分析方法

- **范围**：写（`BatchOffload`）、读（`BatchLoad`）、查（`IsExist`），以及 Init、staging、配置、
  错误映射、并发和删除/淘汰。
- **FMEA 方法**：对每个功能列出失效模式、局部影响、系统影响、严重度、检测/缓解和对应测试。
  本版不计算 RPN，按严重度优先；发生概率和可检测性写在“检测/缓解”列。
- **严重度定义**：

| 级别 | 含义 |
| --- | --- |
| S1 | 数据损坏、读到错误对象、Master 元数据被误删；必须零容忍 |
| S2 | 操作不可用、offload 永久停止、重启后不可恢复；必须可检测并快速失败 |
| S3 | 性能/配额/部分功能退化，无数据损坏 |
| S4 | 仅可观测性/诊断问题 |

- **测试方法**：单元测试覆盖 codec/CRC；契约测试覆盖写/读/查/删；故障注入用
  `FakeKvClient : public kv::KvClient`；并发用 TSAN/ASAN；端到端用 `FileStorage` +
  `MemoryKVClient`（CI）和 UCM FAKE provider（可选集成）。

### 5.2 失效模式与影响（FMEA）

| ID | 功能 | 失效模式 | 局部影响 | 系统影响 | 严重度 | 检测/缓解 | 测试 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| FM-INIT-01 | Init | `KvClient::Init` 或握手失败 | 后端 Init 返回错误，无 staging | FileStorage Init 失败，节点不提供 offload | S2 | 错误码/日志；不进入可用状态 | TC-INIT-01 |
| FM-INIT-02 | Init | `RegisterRegions` 失败 | 无可用 staging slot | 写/读全部不可用 | S2 | 返回错误；不保留部分句柄 | TC-INIT-02 |
| FM-INIT-03 | Init | 二次 Init | 计数/索引被重置或重复注册 | 状态不一致、重复 task | S2 | CAS 保护；第二次返回 `INTERNAL_ERROR` | TC-INIT-03 |
| FM-INIT-04 | Init | namespace 为空或配置非法 | 使用无效 key 启动 | 写入错误对象或全部失败 | S2 | `Validate` 失败；`INVALID_PARAMS` | TC-INIT-04 |
| FM-WRITE-01 | 写 | 8B key 碰撞 | 两个 scoped key 覆盖同一 ASU key | 一个对象丢失或读错 | S1 | 128-bit fingerprint 读校验；日志 | TC-WRITE-01 |
| FM-WRITE-02 | 写 | 记录头编码错误 | 读端拒收合法数据 | offload 数据不可读 | S2 | 记录头单测；头 CRC | TC-WRITE-02 |
| FM-WRITE-03 | 写 | payload 超 slot/单值上限 | 该 key 跳过 | offload 不完整，Master 仍持有内存副本 | S3 | 跳过集合；日志；返回成功数 | TC-WRITE-03 |
| FM-WRITE-04 | 写 | staging 耗尽 | 批次阻塞或跳过 | 心跳延迟或丢批 | S3 | 有界等待/跳过；slot 计数 | TC-WRITE-04 |
| FM-WRITE-05 | 写 | `BatchStoreAsync` 提交失败 | 无 entry 写入 | 本批失败 | S2 | 错误映射；不更新索引 | TC-WRITE-05 |
| FM-WRITE-06 | 写 | `Wait` 超时 | task 可能稍后完成 | 不通知 Master；孤儿对象 | S2 | `RPC_TIMEOUT`；slot 释放；重试覆盖 | TC-WRITE-06 |
| FM-WRITE-07 | 写 | 把 `PARTIAL_FAILED` 当整批失败 | 成功 entry 被丢弃 | offload 回退/重试放大 | S2 | 逐 entry 判断 | TC-WRITE-07 |
| FM-WRITE-08 | 写 | 逐 entry 失败未跳过 | 整批返回错误 | 心跳失败或 offload 停用 | S2 | 只上报成功 key；0 成功返回 `expected(0)` | TC-WRITE-07 |
| FM-WRITE-09 | 写 | `complete_handler` 失败 | 数据已写、Master 不知 | 悬挂副本；重启不可见 | S2 | 返回错误；不回滚；日志 | TC-WRITE-08 |
| FM-WRITE-10 | 写 | 覆盖写计数 delta 错误 | 配额/水位漂移 | 过早停写或超配额 | S3 | 原子更新 index+counters | TC-WRITE-09 |
| FM-WRITE-11 | 写 | 空间不足返回 `KEYS_ULTRA_LIMIT` | FileStorage 永久关闭 offload | 节点不再 offload | S2 | 禁止该码；返回 success count | TC-WRITE-03 |
| FM-DEL-01 | 删除 | `eviction_handler` 为空 | 零删除 | 空间无法回收 | S2 | 返回空 + WARNING；不删任何 key | TC-DEL-01 |
| FM-DEL-02 | 删除 | `eviction_handler` 返回错误 | 零删除、索引保留 | 下一轮重试；可能反复失败 | S2 | master-first；不删；重试；Master 对 `OBJECT_NOT_FOUND` 幂等 | TC-DEL-02 |
| FM-DEL-03 | 删除 | `DeleteAsync` 逐 entry 失败 | 该 key 索引恢复 | 残留对象；计数不扣减 | S3 | 恢复索引；下轮重试；日志 | TC-DEL-03 |
| FM-DEL-04 | 删除 | Master 已接受淘汰、`DeleteAsync` 前崩溃 | ASU 残留对象 | 重启后无法发现；可能被后续写覆盖 | S2 | 文档化“复活窗口”；v2 tombstone/持久化索引 | TC-DEL-04 |
| FM-DEL-05 | RemoveAll | 重启后索引为空 | 只能删除已知 key | 旧对象残留；容量不释放 | S2 | 文档化；后续 manifest/Master RPC | TC-DEL-05 |
| FM-DEL-06 | 删除 | 删除与并发读 | 读到旧值或 `OBJECT_NOT_FOUND` | 不允许半对象 | S1 | 单 key 原子 + CRC；读到旧值可接受 | TC-DEL-06 |
| FM-DEL-07 | 删除 | 删除后计数 delta 错误 | 配额/水位漂移 | 过早停写或超配额 | S3 | 成功/缺失才扣减；失败恢复 | TC-DEL-07 |
| FM-DEL-08 | 删除 | 重复删除 | `NOT_FOUND` 被当失败 | 反复重试/日志噪声 | S3 | `NOT_FOUND` 视为成功 | TC-DEL-08 |
| FM-READ-01 | 读 | `slice.size` 与头 `payload_size` 不一致 | 返回 `INVALID_PARAMS` | 该对象读失败 | S2 | 交叉校验；不拷贝 | TC-READ-01 |
| FM-READ-02 | 读 | ASU `NOT_FOUND` | 返回 `OBJECT_NOT_FOUND` | Master 路由悬挂 | S2 | 清索引；probe 修复 | TC-READ-02 |
| FM-READ-03 | 读 | magic/version/flags 损坏 | `INVALID_READ` | 不返回错误数据 | S1 | 头校验 | TC-READ-03 |
| FM-READ-04 | 读 | header CRC 损坏 | `INVALID_READ` | 记录头不可信 | S1 | CRC-32C | TC-READ-03 |
| FM-READ-05 | 读 | fingerprint 不匹配 | `INVALID_READ` | key 碰撞/串写被发现 | S1 | 128-bit fingerprint | TC-READ-04 |
| FM-READ-06 | 读 | payload CRC 损坏 | `CHECKSUM_MISMATCH` | 不返回撕裂数据 | S1 | CRC-32C | TC-READ-05 |
| FM-READ-07 | 读 | 读回长度不足/超出 | `INVALID_READ`/`INVALID_PARAMS` | 不越界拷贝 | S1 | `transfer_size` 校验 | TC-READ-06 |
| FM-READ-08 | 读 | 索引缺失但对象存在 | 若强制查索引则读失败 | 重启后读不可用 | S2 | 以 `slice.size` 为准；成功后补索引 | TC-READ-07 |
| FM-READ-09 | 读 | 与覆盖写并发 | 读到旧值或新值 | 不允许半对象 | S1 | 单 key 原子 + CRC | TC-READ-08 |
| FM-QUERY-01 | 查 | 超时/传输错误返回 `false` | probe 误删 Master 元数据 | 数据不可路由 | S1 | 必须 `unexpected`，不得 `false` | TC-QUERY-01 |
| FM-QUERY-02 | 查 | 索引陈旧/假阳性 | 读时 `NOT_FOUND` | 读失败 | S2 | 以 Query 为准；读时清索引 | TC-QUERY-02 |
| FM-QUERY-03 | 查 | 编码 key 与写不一致 | 永远查不到 | 对象被视为不存在 | S1/S2 | 同一 codec；单测 | TC-QUERY-03 |
| FM-CONC-01 | 并发 | `Wait` 期间持有 `metadata_mutex_` | 读写互相阻塞/死锁 | 心跳超时 | S2 | 锁纪律；`Wait` 外置 | TC-CONC-01 |
| FM-CONC-02 | 并发 | task 未完成就复用 slot | 数据被覆盖 | 写坏对象 | S1 | slot 生命周期覆盖 `Wait` | TC-CONC-02 |
| FM-LIFE-01 | 关停 | 有在途 task 就 `Unregister`/`Shutdown` | 崩溃/数据损坏 | 进程异常 | S1 | 先 drain 再注销 | TC-LIFE-01 |
| FM-LIFE-02 | 生命周期 | 二次 Init/重新 Init | 状态/句柄错乱 | 行为不可预测 | S2 | CAS/禁止重入 | TC-INIT-03 |
| FM-STUB-01 | ScanMeta | 返回空 | Master 重启后不注册 | 冷数据不可达 | S2（本版接受） | 文档/日志 | TC-STUB-01 |
| FM-STUB-02 | RemoveAll | 重启后索引为空，无法枚举 ASU | 只能删除已知 key | 旧对象残留；容量不释放 | S2 | 文档/日志；后续 manifest/Master RPC | TC-DEL-05 |
| FM-STUB-03 | Evict | ASU 容量满/资源不足 | 写入失败 | offload 停止 | S2 | 映射 `RESOURCE_BUSY`/`OUT_OF_CREATE_SIZE`；跳过 key | TC-STUB-02 |
| FM-CAP-01 | 配额 | 重启后计数归零 | 配额失真 | 超量写或提前停写 | S3 | 计数仅参考；ASU 容量为准 | TC-CAP-01 |
| FM-CFG-01 | 工厂 | UCM 未编译但选了 `kKVClient` | 创建失败 | 节点启动失败 | S2 | 编译开关；清晰错误 | TC-CFG-01 |
| FM-CFG-02 | 工厂 | 生产误用 FAKE provider | 数据未落 ASU | 数据丢失 | S1 | 配置校验/健康检查 | TC-CFG-02 |
| FM-OBS-01 | 可观测性 | 无 task/entry 日志 | 无法定位 | 诊断困难 | S4 | 结构化日志 | TC-OBS-01 |

### 5.3 测试用例与通过标准

| TC | 覆盖失效模式 | 注入/前置 | 步骤 | 通过标准 |
| --- | --- | --- | --- | --- |
| TC-INIT-01 | FM-INIT-01 | `FakeKvClient::Init` 返回错误 | 调 `KVClientStorageBackend::Init` | 返回对应错误；无 staging 注册；无崩溃；FileStorage Init 失败 |
| TC-INIT-02 | FM-INIT-02 | `RegisterRegions` 返回失败 | 调 Init | 返回错误；无部分句柄泄漏；修复后重试成功 |
| TC-INIT-03 | FM-INIT-03 | 连续两次 Init | 两次调用 | 第一次 OK；第二次 `INTERNAL_ERROR`；索引/计数不变 |
| TC-INIT-04 | FM-INIT-04 | namespace 为空/配置非法 | 调 Validate/Init | Validate false；Init `INVALID_PARAMS`；不调用 KvClient |
| TC-WRITE-01 | FM-WRITE-01 | 注入两个 scoped key 映射到同一 8B key | 写两个对象，再读第一个 | 读返回 `INVALID_READ`；不返回错误字节；日志含碰撞提示 |
| TC-WRITE-02 | FM-WRITE-02 | 固定 payload/记录头向量 | 编码后再解码 | 字段/CRC/fingerprint 与向量一致；篡改任一字段解码失败 |
| TC-WRITE-03 | FM-WRITE-03/11 | payload 超 slot、超配额、空 key | 批量写 3 个 key | 超限 key 跳过；其余成功；handler 只收到成功 key；不返回 `KEYS_ULTRA_LIMIT` |
| TC-WRITE-04 | FM-WRITE-04 | staging slot=1，延长 Wait | 并发两个写 | 有界等待；无死锁；slot 全部归还；成功数正确 |
| TC-WRITE-05 | FM-WRITE-05 | `BatchStoreAsync` 返回错误 | 调 BatchOffload | 无 handler 调用；返回错误；索引/计数不变 |
| TC-WRITE-06 | FM-WRITE-06 | `Wait` 延迟超过 timeout | 调 BatchOffload | 返回 `RPC_TIMEOUT`；无 handler；slot 释放；重试写成功 |
| TC-WRITE-07 | FM-WRITE-07/08 | 3 个 key，entry 2 注入失败 | 调 BatchOffload | 返回 2；handler keys 为 k1/k3 的 scoped key；k2 不存在；计数 +2 |
| TC-WRITE-08 | FM-WRITE-09 | `complete_handler` 返回错误 | 调 BatchOffload | 数据仍可读；返回该错误；不回滚；日志记录 |
| TC-WRITE-09 | FM-WRITE-10 | 同 key 先写 4K 再写 8K | 两次写 | 读到 8K 新值；`total_size_` 按 delta 更新；`total_keys_` 不变 |
| TC-DEL-01 | FM-DEL-01 | `eviction_handler=nullptr` | 调 EvictAboveDiskWatermark | `DeleteAsync` 调用 0 次；索引/计数不变；返回空 |
| TC-DEL-02 | FM-DEL-02 | handler 返回错误 | 调 Evict | `DeleteAsync` 调用 0 次；索引不变；下一轮可重试 |
| TC-DEL-03 | FM-DEL-03 | 3 个 victim，entry 2 删除失败 | 调 Evict | 返回 2 个 removed；失败 key 索引恢复；计数只扣 2 个 |
| TC-DEL-04 | FM-DEL-04 | handler 成功、DeleteAsync 前模拟崩溃 | 重启 backend + ScanMeta | 进程不崩溃；残留对象不被发现（记录为已知窗口）；日志可追踪 |
| TC-DEL-05 | FM-DEL-05 | 写入 3 个 key 后重启 backend | 调 RemoveAll | 只删除索引已知 key；重启后索引为空时旧 key 残留；行为与文档一致 |
| TC-DEL-06 | FM-DEL-06 | 读与删除并发 | 多线程 | 读到旧值或 `OBJECT_NOT_FOUND`；不出现半对象；CRC 通过 |
| TC-DEL-07 | FM-DEL-07 | 覆盖写后再删除 | 写两次、删一次 | `total_keys_`/`total_size_` 归零；无重复扣减 |
| TC-DEL-08 | FM-DEL-08 | 连续删除同一 key | 两次 RemoveKeys | 第二次返回 `NOT_FOUND` 且视为成功；计数不变 |
| TC-READ-01 | FM-READ-01 | 写入 4096B，读时 `slice.size=4097` | 调 BatchLoad | 返回 `INVALID_PARAMS`；`slice.ptr` 内容不变 |
| TC-READ-02 | FM-READ-02 | ASU 无该 key | 调 BatchLoad | 返回 `OBJECT_NOT_FOUND`；索引条目被清理；无崩溃 |
| TC-READ-03 | FM-READ-03/04 | 篡改 magic/version/header CRC | 调 BatchLoad | 返回 `INVALID_READ`；不拷贝 payload；缓冲区不变 |
| TC-READ-04 | FM-READ-05 | 篡改 fingerprint | 调 BatchLoad | 返回 `INVALID_READ`；不返回错误字节 |
| TC-READ-05 | FM-READ-06 | 篡改 payload 一个字节 | 调 BatchLoad | 返回 `CHECKSUM_MISMATCH`；不拷贝 payload |
| TC-READ-06 | FM-READ-07 | fake 返回短读/超长 | 调 BatchLoad | 返回 `INVALID_READ`/`INVALID_PARAMS`；ASAN 无越界 |
| TC-READ-07 | FM-READ-08 | 清空索引，Master 提供 `slice.size` | 调 BatchLoad | 读成功；字节一致；索引被补回 |
| TC-READ-08 | FM-READ-09 | 读与覆盖写并发 | 多线程读写 | 读到旧值或新值；两者 CRC 均通过；无半对象 |
| TC-QUERY-01 | FM-QUERY-01 | Query 超时/传输错误 | 调 IsExist | 返回 `unexpected`（`RPC_TIMEOUT`/`UNAVAILABLE`）；绝不返回 `false`；Master 元数据不变 |
| TC-QUERY-02 | FM-QUERY-02 | 索引有陈旧条目，ASU 无数据 | 调 IsExist | 返回 `false`；索引条目删除；其他 key 不受影响 |
| TC-QUERY-03 | FM-QUERY-03 | 同一 scoped key 走写/读/查 | 比较编码结果 | 三次得到同一 8B key；无 mismatch |
| TC-CONC-01 | FM-CONC-01 | 阻塞 Wait + 并发写 | 两线程 | 写不被读的 Wait 阻塞（或仅在索引提交短暂等待）；TSAN 无数据竞争；无死锁 |
| TC-CONC-02 | FM-CONC-02 | 1 个 slot + 多读多写 | 并发压测 | ASAN/TSAN 干净；无 use-after-free；slot 全部归还 |
| TC-LIFE-01 | FM-LIFE-01 | 有在途 task 时 Shutdown | 调 Shutdown/析构 | 等待 task 完成；不提前 `UnregisterRegions`；无崩溃/泄漏 |
| TC-STUB-01 | FM-STUB-01 | 正常 backend | 调 ScanMeta | 返回 OK；handler 未被调用；不注册任何对象 |
| TC-STUB-02 | FM-STUB-03 | 模拟 ASU 容量不足 | 调写 | 该 key 跳过/可重试；不崩溃；其他 key 正常 |
| TC-CFG-01 | FM-CFG-01 | 未编译 UCM，选 `kKVClient` | 调 CreateStorageBackend | 返回清晰错误（`INVALID_PARAMS`/`NOT_SUPPORTED`）；无崩溃 |
| TC-CFG-02 | FM-CFG-02 | 生产配置使用 FAKE provider | 启动/健康检查 | 配置校验拒绝或显著告警；不静默写成功 |
| TC-CAP-01 | FM-CAP-01 | 重启 backend | 读计数并写新 key | 计数从 0 开始；写仍可用；ASU 容量错误被正确映射 |
| TC-OBS-01 | FM-OBS-01 | 注入一次 entry 失败 | 检查日志 | 日志含 `task_id`、entry 下标/状态、首错、key；可定位到具体 entry |
| TC-PERF-01 | 性能/批量 | N=100，batch=32 | 调 BatchOffload/BatchLoad | TaskId 数 ≤ `ceil(100/32)=4`；p99 在基线内；无 per-key task |

### 5.4 故障注入能力

`FakeKvClient : public kv::KvClient` 提供：

- `FailNext(op, key, StatusCode)`：对下一次 Query/Store/Load/Delete 注入错误。
- `DelayNext(op, ms)`：模拟 `Wait` 超时。
- `PartialFailure(indices)`：让指定 entry 在 `entryStatus` 中失败，聚合状态为 `PARTIAL_FAILED`。
- `FailDelete(key, StatusCode)`：删除失败，用于验证索引恢复。
- `CorruptValue(key, field)`：篡改 magic/version/header CRC/fingerprint/payload CRC。
- `RejectUnregisteredBuffer()`：模拟 `BUFFER_NOT_REGISTERED`。
- `RecordCalls()`：记录提交/等待顺序和 task 数。

### 5.5 覆盖结论与残余风险

- S1/S2 失效模式都有对应测试；S3 覆盖配额、staging、覆盖写；S4 只做日志断言。


## 6. Mooncake 改动量

### 6.1 文件级清单

| # | 文件 | 改动 | 行数（本版） | 优先级 |
| --- | --- | --- | --- | --- |
| 1 | `mooncake-store/include/storage_backend.h` | `StorageBackendType::kKVClient`；前置声明 | +5~10 | 必须 |
| 2 | `mooncake-store/src/storage_backend.cpp` | 工厂 `case kKVClient`；include | +15~25 | 必须 |
| 3 | `mooncake-store/src/config/file_storage_config.cpp` | `Validate()` 对 `kKVClient` 跳过本地路径 | +15~30 | 必须 |
| 4 | `mooncake-common/include/environment_variables.h` | `MOONCAKE_KV_CLIENT_*` 9 个变量 | +10~18 | 必须 |
| 5 | `mooncake-store/include/config/kv_client_backend_config.h` | 配置结构 | +50~70 | 必须 |
| 6 | `mooncake-store/src/config/kv_client_backend_config.cpp` | 环境变量解析 + `Validate` | +100~140 | 必须 |
| 7 | `mooncake-store/include/kvclient/kv_client_backend.h` | 后端声明；前置声明 `kv::KvClient` | +120~170 | 必须 |
| 8 | `mooncake-store/src/kvclient/kv_client_backend.cpp` | 写/读/查/删、索引/FIFO、task 编排、UCM 类型/错误映射 | +1,400~1,900 | 必须 |
| 9 | `mooncake-store/include/kvclient/kv_client_record.h` + `src/.../kv_client_record.cpp` | 40B 头、CRC、fingerprint | +140~200 | 必须 |
| 10 | `mooncake-store/include/kvclient/kv_client_key_codec.h` + `src/.../kv_client_key_codec.cpp` | 8B key + 128-bit fingerprint | +80~120 | 必须 |
| 11 | `mooncake-store/include/kvclient/kv_client_staging.h` + `src/.../kv_client_staging.cpp` | 注册内存、slot 池 | +180~280 | 必须 |
| 12 | `mooncake-store/include/kvclient/kv_client_factory.h` + `src/.../kv_client_factory.cpp`（含 `MemoryKVClient`） | 工厂 + 参考实现 | +200~300 | 必须 |
| 13 | `mooncake-store/src/CMakeLists.txt` | 源文件 + UCM include/link；`USE_UCM` | +20~40 | 必须 |
| 14 | `dependencies.sh` / `third_party` | UCM `kv_client`/`kv_transport` 引入方式 | +20~60 | 必须 |
| 15 | `mooncake-store/tests/CMakeLists.txt` | `add_store_test` | +4~6 | 必须 |
| 16 | `mooncake-store/tests/storage_backend_test_utils.h` + `storage_backend_test.cpp` | 抽取共享 helper | +80~140 / +10~20 | 必须 |
| 17 | `mooncake-store/tests/kvclient/kv_client_backend_test.cpp` | 写/读/查/删契约、故障、并发 | +900~1,300 | 必须 |
| 18 | `mooncake-store/tests/kvclient/kv_client_record_test.cpp` | 记录头测试 | +140~200 | 必须 |
| 19 | `mooncake-store/tests/kvclient/kv_client_key_codec_test.cpp` | key/fingerprint 测试 | +80~120 | 必须 |
| 20 | `mooncake-store/tests/kvclient/kv_client_staging_test.cpp` | 对齐/注册/池测试 | +100~160 | 必须 |
| 21 | `mooncake-store/tests/kvclient/fake_kv_client.h` | `FakeKvClient : public kv::KvClient` | +200~300 | 必须 |
| 22 | `mooncake-store/tests/kvclient/kv_client_config_test.cpp` | 配置/能力校验 | +100~150 | 必须 |
| 23 | `mooncake-store/tests/file_storage_config_test.cpp` | `Validate` 分支断言 | +30~50 | 必须 |
| 24 | `docs/source/design/store/kvclient-storageBackend.md` | 本设计页 | +250~400 | 必须 |
| 25 | `docs/source/design/store/mooncake-store.md` | toctree 条目 | +1 | 必须 |
| 26 | `mooncake-store/benchmarks/storage_backend_bench.cpp` | 性能基准 | +200~300 | 可延后 |
| 27 | `docs/source/deployment/ssd/ssd-offload.md` | 部署说明 | +40~80 | 可延后 |
| 28 | `mooncake-store/include/file_storage.h` + `src/file_storage.cpp` | 公共 `BatchRemove` 接线 | +10~60 | 后续版本 |

### 6.2 汇总与估算依据

- **核心实现（1–14）**：约 **2,400–3,500 行**。
- **测试（15–23）**：约 **1,650–2,450 行**。
- **文档（24–25）**：约 **250–400 行**。
- **本版总计**：约 **4,300–6,350 行**，新增文件约 20 个，修改文件约 10 个。


估算依据：

| 参照 | 行数 |
| --- | --- |
| `mooncake-store/src/nvme_kv/backend.cpp` | 1,527 |
| `mooncake-store/src/storage/distributed/distributed_storage_backend.cpp` | 906 |
| `mooncake-store/tests/storage_backend_test.cpp` | 5,392 |
| `kv_semantics/src/client/kv_client_impl.cc` | 约 990 |

区间误差约 ±30%。

### 6.3 分期

| 阶段 | 内容 | 退出条件 |
| --- | --- | --- |
| P0（本版） | 后端 + 直接调用 `kv::KvClient` + staging + 写/读/查/删契约测试 | 四步操作闭环；`make` + 测试通过 |
| P1 | 公共 `BatchRemove`/`BatchQueryStat`、性能基准 | 端到端压测通过 |
| P2 | 恢复 manifest、零拷贝、大对象分片、按 key 公共 `BatchRemove` | 单独设计评审 |



## 附录 A：术语

| 术语 | 含义 |
| --- | --- |
| ASU | UCM 管理的存储后端 |
| KVClient | 本文专指 UCM 的 `kv::KvClient`（旧分支 `UC::ASU::AsuClient`） |
| CacheKey | ASU 的 8 字节 key |
| KVBuffer | ASU 的一次读写描述：key + 注册内存区间 + offset |
| scoped key | `tenant_id + '\0' + local_key`，FileStorage/后端回调使用的 key |
| record | 40 字节头 + payload |
| payload | 各 `Slice` 按序拼接后的对象数据，等于上报 Master 的 `data_size` |
| transfer_size | `align_up(40 + payload_size, 512)`，ASU 实际传输长度 |
| staging arena | backend 持有、向 ASU 注册的读写中转内存 |

