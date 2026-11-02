# KvClientStorageBackend 第一版最小实现设计（写 / 读 / 查 / 删）

本文给出 `KvClientStorageBackend` 第一版最小实现的设计方案，**本版实现写、读、查、删四步操作**。
它新增一个 `StorageBackendInterface` 实现类，参考 `NvmeKvStorageBackend` 的组织方式，
**在 backend 内直接调用 `kv::KvClient`**，把 Mooncake KV client 的 offload 数据面落到 ASU。

- 状态：Draft v1（写 / 读 / 查 / 删）
- 2026-11-02 裁决：`IsEnableOffloading` 在 `kReady` 后返回 true，未就绪返回错误；
  容量/配额控制只在 `BatchOffload` 内逐批/逐 key 做。以
  `kv-client-storage-backend-review.md` §0.3 / §5.2 为准。
- 2026-11-02 entry 构造改为对齐 asuStore 命名：`BlockId` / `Shard` / `TaskDesc` /
  `KVCacheRegistration` / `RegisteredPersistentRegion` / `FindPersistentHandle` /
  `BuildKvBuffers`；以 `kv-client-storage-backend-review.md` 的 3.1 / 3.2 节为准。
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
| `KvClientStorageBackend`（本文） | ASU（直接调用 `kv::KvClient`） | **本版不实现恢复**（`ScanMeta` 返回空） |

### 1.2 本版范围

**实现**：

| 操作 | 入口 | 说明 |
| --- | --- | --- |
| 写 | `BatchOffload` | 把 Mooncake 对象写入 ASU；逐 key 部分成功 |
| 读 | `BatchLoad` | 按 Master 提供的 key + size 从 ASU 读回；校验 `RecordHeader` 与 CRC |
| 查 | `IsExist` | 用 ASU `QueryAsync` 判断存在性 |
| 删 | `Client::Remove` / `Client::BatchRemove` → `BatchRemoveKeys` | **仅支持用户按 key 删除**；Master 先删元数据，再通知盘副本 owner 逐 key `DeleteAsync` |

**接口必需方法的本版行为**：

| 入口 | 本版行为 |
| --- | --- |
| `Init` | 创建 `kv::KvClient` → `Init` → `RegisterRegions(staging)` → 置零索引/计数；不打开设备、不建 backend 线程池；失败回滚 `initialized_` 为 `kUninit` 并 `Shutdown` client |
| `IsEnableOffloading` | `kReady` 后直接返回 true；`kUninit`/`kInitializing`/Init 失败返回错误；不读计数、不查 ASU 容量、不发 RPC。配额/容量控制由 `BatchOffload` 逐批/逐 key 做 |
| `ScanMeta` | 返回 `OK`，不枚举、不注册任何对象 |
| `ResetScanIterator` | 空实现 |
| `EvictAboveDiskWatermark` | **不实现**；返回空（本版不支持水位淘汰） |
| `RemoveAll` | **不实现**；空实现（本版不支持全清） |
| `SetTestFailurePredicate` | 保留，用于部分成功测试 |

**明确不在本版**：`RemoveByRegex`、`RemoveAll`、水位淘汰、ASU GC 对接、恢复 manifest、
`ScanMeta` 枚举、chunk 分片、多副本、压缩/加密/TTL、零拷贝、批量查询。

### 1.3 实现注意事项（语义不统一点）

| 关注点 | Mooncake 侧语义 | KvClient 侧语义（不统一点） | 直接照搬的后果 | 实现处理 |
| --- | --- | --- | --- | --- |
| 调用模型 | 同步，返回 `tl::expected` | 异步 task：`XxxAsync` 返回 `taskId`；`Wait` 是破坏性读，超时移除 task 但不取消底层任务 | 把提交当完成；`Wait` 期间持锁；超时后复用 slot 导致 use-after-free | 每块 submit 后只 `Wait` 一次；`Wait` 不持元数据锁；OK/PARTIAL 释放 slot，TIMEOUT/TASK_NOT_FOUND 隔离 |
| 失败/部分成功 | 逐 key 部分成功，返回成功数；`KEYS_ULTRA_LIMIT` 会永久关闭 offload | 聚合 `status` + 逐 entry `status`；`PARTIAL_FAILED` 时可能有“OK 但未发送”的 entry | 只看聚合状态或盲目发布 OK entry 会产生假成功；误用 `KEYS_ULTRA_LIMIT` 会永久停 offload | 顶层 OK 才发布写/读；`PARTIAL_FAILED` 本块不发布/不返回；删逐 key；0 成功返回 `expected(0)` |
| 批量大小 | 一次一个 map | `Store/Load` ioNum=1；`Batch*` ioNum 110；Delete 254；Query 256；内部按 ASU 路由再拆子批 | 用错 API 导致 SQE 数量过多/任务粒度过大 | 多 key 用 `Batch*`；按 `config.batch_size` 切 TaskId；内部 ioNum 不暴露 |
| key | Mooncake 真实 key：用户态 `ObjectKey = std::string`；到 backend 是 `scoped_key = TenantId + '\0' + local_key` | 8 字节 `asu_key`（`kv::CacheKey`），不可逆 | 无法从 ASU key 反解 `scoped_key`；64-bit 哈希碰撞未处理 | `XXH64("mooncake/kvclient/v1" + key_namespace + 0x00 + scoped_key)`；本版不做碰撞检测，默认不发生碰撞（见 §3.6） |
| 长度 | Master `data_size` 是权威 | Query 只有存在性，没有长度 | 想从 ASU 查长度会失败 | 写时计算；读时用 `slice.size`；头里 `payload_size` 交叉校验 |
| value/内存 | `Slice` 任意内存，无需注册 | `KVBuffer` 必须引用注册内存；长度/offset 512B 对齐 | `BUFFER_NOT_REGISTERED`/`INVALID_ARGUMENT` | staging arena 注册；`transfer_size = align_up(24+payload,512)` |
| 查询 | `IsExist` bool；`ScanMeta` 可枚举 | `QueryAsync` exists 位图；未命中是 OK+exists0+ENTRY_KEY_NOT_EXIST；分片失败时 exists 保持 0 | 只看 exists 会把分片失败当成不存在，probe 误删 Master 元数据 | 同时看 exists+entryStatus；非 ENTRY_KEY_NOT_EXIST 的错误返回 `unexpected`；`ScanMeta` 空实现 |
| task/内存生命周期 | backend 自身线程安全；Slice 调用方管理 | KvClient 内部 worker；调用方保证 task 完成前注册内存有效；Wait 超时后 task 被移除但可能仍在飞 | 提前 `Unregister`/复用 slot → 数据损坏 | slot 正常完成 Release；超时/任务丢失 Quarantined 不复用；Shutdown 前确保无在飞/隔离 |
| 生命周期 | `Init`；无 `Shutdown`；二次 Init 失败 | `Init` 非线程安全/失败不回滚；`Shutdown` drain 超时后仍注销内存 | 失败实例复用/在飞 IO 引用已注销内存 | 失败丢弃 client 重建；正常路径先等所有 task 完成再 Shutdown；CAS 保证二次 Init 失败 |
| 容量/配额 | `total_size_limit`、淘汰 | 无容量查询；ASU 自管 | 计数重启归零；超量写 | 计数仅参考；超配额跳过；不返回 `KEYS_ULTRA_LIMIT` |
| 删除/清空 | `Client::Remove`/`BatchRemove` 先删 Master 元数据；后端需要按 key 物理删除 | `DeleteAsync` 逐 key；entry raw 0=OK/raw 1=ENTRY_DELETE_FAILED；整体 CQE_KEY_NOT_EXIST=已不存在；无 entry KEY_NOT_EXIST | 调用方可能不是 owner；key_namespace 不同无法算 key；把 CQE_KEY_NOT_EXIST 当失败 | Master 按 key 通知 owner → `BatchRemoveKeys` → `DeleteAsync`；CQE_KEY_NOT_EXIST 归一成功；本版不支持 RemoveAll/正则/水位 |
| 恢复 | 启动 `ScanMeta` 重新注册 | 无枚举 | Master 重启后不可注册 | `ScanMeta` 返回空；读/查不依赖索引 |
| 完整性 | 各后端自选：CRC-32C / XXH32+verify_hash | 无 per-value checksum | 撕裂/串写被当有效数据 | 自建 `RecordHeader`（24B）+ 头 CRC + payload CRC |
| 错误码 | `ErrorCode` | `StatusCode` + `ENTRY_*`/`CQE_*` | 透传错误码触发错误分支 | backend 内映射；未知归 `INTERNAL_ERROR` |

## 2. 系统上下文与模块架构

### 2.1 参与者与部署边界

| 参与者 | 运行位置 | 职责 |
| --- | --- | --- |
| Application / vLLM | 客户端进程 | 调用 Store API |
| `StoreClient` / `Client` | 客户端进程 | 路由、内存副本、RPC |
| `FileStorage` | 客户端进程 | offload/restore 编排（默认写完即下沉）、心跳、缓冲、接收 owner 删除任务 |
| `KvClientStorageBackend` | 客户端进程 | 配额、索引、编解码、写/读/查/按 key 删；直接调用 `kv::KvClient` |
| UCM `KvClient` | 客户端进程 | ASU 连接、视图路由、task 管理、内存注册 |
| ASU View Service | 控制面 | 提供 ASU 视图/路由 |
| ASU 节点 | 进程外 | 持久化 KV，拥有自身容量/GC 策略 |
| Master | 控制面进程 | 逻辑对象与 `LOCAL_DISK` replica 元数据的权威 |



### 2.2 系统上下文图

```mermaid
flowchart LR
    App["Application / vLLM"] -->|"Put / Get / Remove"| Store["StoreClient API"]

    Store -->|"路由 / lease / offload 队列"| Master["Master Service"]
    Store -->|"Remove / BatchRemove（删）"| Master
    Store -->|"BatchGet（读）"| FS["FileStorage"]

    Master -->|"offload 任务（写）"| FS
    Master -->|"per-key delete task（删）"| FS

    FS -->|"BatchOffload（写）"| BE["KvClientStorageBackend"]
    FS -->|"BatchLoad（读）"| BE
    FS -->|"IsExist（查）"| BE
    FS -->|"BatchRemoveKeys（删）"| BE

    BE -->|"StoreAsync（写）"| KV["kv::KvClient"]
    BE -->|"LoadAsync（读）"| KV
    BE -->|"QueryAsync（查）"| KV
    BE -->|"DeleteAsync（删）"| KV

    BE -->|"complete_handler"| FS
    FS -->|"NotifyOffloadSuccess"| Master

    KV --> View["ASU View Service"]
    KV --> ASU["ASU nodes"]
```

四条主流程：

| 流程 | 路径 |
| --- | --- |
| 写 / offload | App → Store（Put）→ Master（路由/lease/offload 队列）→ FileStorage 心跳 → Backend `BatchOffload` → `KvClient::StoreAsync` → ASU；Backend `complete_handler` → FileStorage → Master `NotifyOffloadSuccess` |
| 读 / restore | App → Store（Get/BatchGet）→ Master（LOCAL_DISK 路由）→ FileStorage `BatchGet` → Backend `BatchLoad` → `KvClient::LoadAsync` → ASU；数据回填到调用方 buffer |
| 查 / probe | FileStorage `Exists`（dangling replica probe）→ Backend `IsExist` → `KvClient::QueryAsync` → ASU |
| 删 / per-key | App → Store（Remove/BatchRemove）→ Master（删元数据）→ per-key delete task → FileStorage `BatchRemoveKeys` → Backend `BatchRemoveKeys` → `KvClient::DeleteAsync` → ASU |

### 2.3 模块分层与依赖规则

```mermaid
flowchart TB
    subgraph L0["Mooncake 现有调用层"]
        FS["FileStorage"]
        Factory["CreateStorageBackend"]
    end

    subgraph L1["Mooncake 唯一新增实现"]
        BE["KvClientStorageBackend<br/>: StorageBackendInterface"]
        IDX["PayloadSizeIndex<br/>（内部缓存）"]
        CODEC["KeyCodec + RecordCodec<br/>（内部组件）"]
        CFG["KvClientBackendConfig<br/>（内部配置）"]
        STAGE["StagingArena<br/>（内部组件）"]
        DEL["BatchRemoveKeys<br/>（内部方法）"]
    end

    subgraph L2["现成依赖：UCM KvClient SDK"]
        KVC["kv::KvClient<br/>（现成接口）"]
        CREATE["CreateKvClient()<br/>（SDK 工厂）"]
        LIB["libkv_client / libkv_transport"]
    end

    subgraph L3["ASU"]
        VIEW["ASU View Service"]
        ASUN["ASU nodes"]
    end

    subgraph TEST["仅测试/CI，不进入生产部署形态"]
        REF["ReferenceKvClient<br/>（client_impl=kReference）"]
        FAKE["FakeKvClient<br/>（故障注入）"]
    end

    FS --> BE
    Factory --> BE
    BE --> IDX
    BE --> CODEC
    BE --> CFG
    BE --> STAGE
    BE --> DEL
    BE -->|"直接调用；UCM 头仅在 .cpp"| KVC
    KVC --> CREATE
    CREATE --> LIB
    LIB --> VIEW
    LIB --> ASUN
    REF -.->|"实现 kv::KvClient（CI）"| KVC
    FAKE -.->|"实现 kv::KvClient（测试注入）"| KVC
```

依赖规则：

- **Mooncake 只新增一个 `StorageBackendInterface` 实现**：`KvClientStorageBackend`。索引、codec、
  配置、staging、`BatchRemoveKeys` 都是它的内部组件/方法，不新增端口层或适配器层。
- `kv::KvClient` 是 **现成的 UCM SDK**；Mooncake 只调用 `CreateKvClient()`、`Init`、
  `BatchStoreAsync`/`BatchLoadAsync`/`QueryAsync`/`DeleteAsync`/`Wait`，不实现、不包装 SDK。
- UCM 头文件只出现在 `kv_client/backend.cpp`、`kv_client/reference_kv_client.cpp` 和测试
  `fake_kv_client.h`；`FileStorage`、`CreateStorageBackend` 和 Mooncake 生产公共头不感知 UCM。
- `ReferenceKvClient` 是仓库内的 `kv::KvClient` 参考实现，仅由 `client_impl=kReference` 在
  CI/单测选中；生产部署出现 `kReference` 视为配置错误。`FakeKvClient` 只用于测试注入。
- `StagingArena` 持有 ASU 注册句柄；`UnregisterRegions` 只在 shutdown 时、所有 task 完成后调用。

### 2.4 线程模型与所有权

| 线程 | 入口 | 与后端的关系 |
| --- | --- | --- |
| Heartbeat 线程 | `Heartbeat()` → `BatchOffload` / `PollDeleteTasks` → `BatchRemoveKeys` | 串行写、按 key 删除 |
| RPC worker | `BatchGet` → `BatchLoad` | 并发读 |
| dangling replica probe | `FileStorage::Exists` → `IsExist` | 低频并发查 |
| 重启/重注册 | `FileStorage::Init` → `ScanMeta` | 空实现，直接返回 |
| UCM KvClient worker | `KvClientImpl::WorkerLoop` | 提交 task、等待完成；不回调 Mooncake |

锁纪律：

1. `write_mutex_`：串行化 `BatchOffload`、`BatchRemoveKeys` 和 `Init`；保护跨 key 的原子序列和
   `complete_handler` 的顺序。
2. `metadata_mutex_`：读写锁，保护 `index_`、`total_keys_`、`total_size_`；**绝不跨 ASU
   `Wait` 或 Master 回调持有**。
3. `staging_mutex_`：只保护 slot 的 `Free/InUse/Quarantined` 状态与空闲队列，不覆盖 `Wait`。
4. 加锁顺序固定为 `write_mutex_` → `metadata_mutex_`；`staging_mutex_` 不参与嵌套。
5. 读/查只在需要更新/读取索引时短暂持有 `metadata_mutex_`；ASU I/O 期间不持锁。
6. `complete_handler` 约定不得重入本后端。

**每个锁的作用与必要性**

| 锁 | 保护什么 | 为什么必须 | 去掉会怎样 | 必要性 |
| --- | --- | --- | --- | --- |
| `write_mutex_` | 多步写序列（Prepare → ASU I/O → CommitIndex → `complete_handler`）、配额准入、Init 与写互斥 | 两个写者同时看到“还有空间”会超配额；两个写序列交错会让回调顺序/计数与 ASU 实际不一致；Init 与写并发会看到半初始化 client | 计数错、超配额、回调顺序乱、Init 期访问半初始化 client | **当前设计必须**。若能量化保证“只有一个写线程且 Init 在心跳前完成”，可降级为断言；但删除任务、未来 admin 操作都可能引入第二个写者，不建议去掉 |
| `metadata_mutex_` | `index_` + `total_keys_/total_size_` 的读一致性 | 读/查会读/删索引，写会改索引；`unordered_map` 并发读写是 UB；计数与索引需要一起更新 | 数据竞争/崩溃；读看到半更新计数 | **当前设计必须**。若读/查完全不碰索引、计数改成原子量，可去掉，但配额判断会变成近似值 |
| `staging_mutex_` | slot 的 `Free/InUse/Quarantined` 状态与空闲队列 | 多个 RPC worker 并发读、心跳线程写，都要从同一个池里取 slot；两个操作拿到同一 slot 会互相覆盖；Wait 超时后不能释放，只能隔离 | 数据串写、读到别人的数据；超时后复用 slot 会 use-after-free | **必须**。除非改成每线程私有 buffer，或 per-slot 原子队列 |
| 锁顺序 `write_mutex_ → metadata_mutex_` | 防止加锁顺序反转 | 未来新增路径若先拿 metadata 再拿 write，会死锁 | 潜在死锁 | **规则必须**；当前代码只有写路径嵌套这两把锁 |
| 短临界区（不跨 `Wait`/回调） | 可用性/尾延迟 | ASU `Wait` 最长 `wait_timeout_ms`；持锁会让所有读写排队、心跳超时 | 功能仍正确，但读写尾延迟和心跳可用性急剧下降 | **正确性上非必须，可用性上必须** |
| `complete_handler` 不重入 | 回调期间持有 `write_mutex_` | 回调若重入 backend 会自死锁 | 死锁 | **必须**；或者改为回调前释放锁，并重新定义顺序协议 |

为什么不是一把锁：

- 一把锁从 `Prepare` 持有到 `Wait` 结束，会把所有读者阻塞最长 `wait_timeout_ms`（默认 5s），不可接受。
- 一把锁只在 `CommitIndex` 短暂持有，又无法串行化“看到有空位 → 写入 → 提交计数”的整段序列，配额会超卖。
- 把 `staging_mutex_` 并进 `metadata_mutex_`，会让“等一个空闲 slot”阻塞所有索引读者；分开后等待 slot 不影响读索引。

`staging_mutex_` 只在 `Acquire`/`Release`/`Quarantine` 的瞬间持有；slot 的所有权（不是 mutex）
覆盖 ASU I/O。正常完成 `Release`，Wait 超时/任务丢失 `Quarantine`（不再复用），所以它不与
`metadata_mutex_` 嵌套，不参与 `write_mutex_ → metadata_mutex_` 的顺序。

**与 NVMeKV 的锁设计对照**

NVMeKV 没有 `write_mutex_` / `metadata_mutex_` / `staging_mutex_` 这一组锁，它的并发控制是：

| NVMeKV 机制 | 证据 | 作用 |
| --- | --- | --- |
| `std::atomic<bool> initialized_` | `nvme_kv/backend.h:83`、`backend.cpp:460-472` | Init CAS；失败复位，允许重试 |
| `std::atomic<int64_t> total_size_/total_keys_`（relaxed） | `backend.h:88-89`、`backend.cpp:961-962, 1503-1514` | 计数；`IsEnableOffloading` 直接读原子量 |
| `std::shared_mutex manifest_cache_mutex_` | `backend.h:84,87`、`backend.cpp:352-399` | `FindCachedManifest` 用 shared，`CacheManifest` 用 unique |
| `ThreadPool` + `IndexQueue`（mutex+condvar） | `backend.h:17,97-99`、`backend.cpp:216-275, 30-63` | 内部 IO 并行与任务队列 |
| `connector_` 只在 Init 赋值，之后只读 | `backend.cpp:449-456` | 无锁读取 shared_ptr 副本 |

对照到本后端：

| 本后端 | NVMeKV 对应 | 差异与启示 |
| --- | --- | --- |
| `write_mutex_` | **无**；NVMeKV 依赖 FileStorage 心跳单写线程 | NVMeKV 的配额检查是“读原子量 → 判断 → 稍后 `fetch_add`”，并发 `BatchOffload` 会超卖；它接受这个前提。本后端若也保证“心跳单写”，可去掉 `write_mutex_`；若未来有 admin/淘汰线程，保留 |
| `metadata_mutex_` | `manifest_cache_mutex_` + 原子计数 | NVMeKV 把 map 保护和计数拆开：map 用 `shared_mutex`，计数用原子量。本后端可照做：`PayloadSizeIndex` 用 `shared_mutex`，`total_*` 用原子量；代价是计数与索引不是同一快照 |
| `staging_mutex_` | **无**；NVMeKV 由 connector/executor 管理设备 buffer | 本后端有共享 staging arena，必须保留；NVMeKV 没有可复用的等价物 |
| 锁顺序 | 无嵌套 | 本后端只有 `write → metadata` 一处嵌套；`staging_mutex_` 不嵌套 |

可参考的取舍：

- 若保持“心跳线程是唯一写者”（当前 `BatchOffload`、`BatchRemoveKeys` 都在心跳线程），可以照 NVMeKV：
  去掉 `write_mutex_`，用 `std::atomic` 计数 + `shared_mutex` 保护索引，配额判断接受近似值。
- 若要求精确配额/多写者，保留 `write_mutex_`；或者用 `total_size_` 的 CAS 预留 + 失败回滚，代替整段串行。
- `staging_mutex_` 不能照 NVMeKV 去掉，因为 NVMeKV 没有“跨读者/写者共享的注册 buffer 池”。
- **不要照搬** NVMeKV 配额不足返回 `KEYS_ULTRA_LIMIT` 的行为：该码会让 FileStorage 永久关闭 offload；
  本后端应跳过本批并返回 `expected(0)`。

### 2.5 对象状态机

```mermaid
stateDiagram-v2
    [*] --> Absent
    Absent --> Stored: StoreAsync 成功且 Wait OK
    Stored --> Indexed: 提交 index
    Indexed --> Indexed: 覆盖写（更新 payload_size）
    Indexed --> Deleting: BatchRemoveKeys 标记
    Deleting --> Absent: DeleteAsync 成功或 key 不存在
    Deleting --> Indexed: DeleteAsync 失败，恢复索引
    Indexed --> Absent: ASU 不存在且 IsExist/读 探测到
```

**状态机说明**

这张图描述的是 **backend 本地索引/删除簿记** 中单个 key 的生命周期。

### 2.6 副本 / 元数据 / 句柄：三层关系

这张图把 Mooncake 读路径上的三种投影画在一起，用来定位 `KvClientStorageBackend` 在链路中的确切位置：
**副本**回答"数据在哪里"，**元数据**回答"对象有哪些副本、多大、校验值是多少"，**句柄**回答"本进程怎么引用、何时释放"。
三者都不携带 payload——payload 始终在共享内存、本地盘或 ASU 上。

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
    OW2 -->|"storage_backend_->BatchLoad(key, slice.size)"| BE["KvClientStorageBackend"]
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
3. **`KvClientStorageBackend` 只在 owner 侧的 `BatchLoad` 出现一次**（图中唯一与 ASU 相邻的节点），
   输入只有 `(scoped_key, slice.size)`，不含 value 定位之外的任何上下文。
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
| F5 | 删 / 按 key | `Client::Remove` / `Client::BatchRemove` → `BatchRemoveKeys` | P0 | 仅用户按 key 删除；Master 先删元数据，再通知 owner；`DeleteAsync` 逐 key；失败恢复索引 |
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
void SetTestFailurePredicate(std::function<bool(const std::string& key)>);

// 本版新增：仅支持 Client::Remove / BatchRemove 触发的按 key 删除。
// 基类默认实现逐 key 返回 NOT_SUPPORTED；本后端重写。
virtual std::vector<tl::expected<void, ErrorCode>> BatchRemoveKeys(
    const std::vector<std::string>& scoped_keys);
```

不重写、沿用基类默认：

```cpp
void RemoveAll() {}  // 本版不支持全清
tl::expected<std::vector<std::string>, ErrorCode>
EvictAboveDiskWatermark(double, double, EvictionHandler = nullptr) {
    return std::vector<std::string>{};  // 本版不支持水位淘汰
}
```

内部辅助：

```cpp
struct RemoveOutcome {
    std::vector<std::string> removed;
    std::vector<std::string> not_found;
    std::vector<std::string> failed;
};
tl::expected<RemoveOutcome, ErrorCode> RemoveKeys(
    const std::vector<std::string>& scoped_keys);
```

关键语义：

- `BatchOffload` 的 `keys` 必须是调用方传入的 **`scoped_key`**（`tenant\0local_key`），不是 ASU key。
- `StorageObjectMetadata` 的 `data_size` 必须填 **payload 大小（不含 `RecordHeader`，24B）**。
- `BatchLoad` 的 `Slice` 由调用方按 `data_size` 预分配；后端不得改写 `slice.ptr`。
- `IsExist` 的“不存在”是 `expected(false)`；传输错误必须返回 `unexpected`。
- `ScanMeta` 本版返回 `OK` 且不调用 handler；`ResetScanIterator` 空实现。
- `BatchRemoveKeys` 只处理 **本 client 拥有的 ASU 对象**；调用前 Master 已删除元数据
  （master-first）；`NOT_FOUND` 视为成功；失败 key 返回错误并恢复索引。
- `BatchRemoveKeys` 的输入是 `scoped_key`；只有 owner 知道自己的 `key_namespace`，因此不能由任意
  client 直接算 ASU key。
- `RemoveAll`/`EvictAboveDiskWatermark` 本版不支持，继承基类空实现；`RemoveByRegex` 不支持。
- `RemoveKeys` 逐 entry 处理：`NOT_FOUND` 视为成功；失败 key 恢复索引并保留重试。

**本版 `Init` / `IsEnableOffloading` 的实现契约**：

`initialized_` 使用三态 `std::atomic<InitState>`（`kUninit` / `kInitializing` / `kReady`），
而不是 `bool`：只有 `kReady` 才允许对外服务，`kInitializing` 期间并发进入的调用一律失败，
不会把半初始化的实例当成可用。

```cpp
enum class InitState { kUninit, kInitializing, kReady };

tl::expected<void, ErrorCode> KvClientStorageBackend::Init() {
    InitState expected = InitState::kUninit;
    if (!initialized_.compare_exchange_strong(expected, InitState::kInitializing)) {
        return tl::make_unexpected(ErrorCode::INTERNAL_ERROR);  // 已初始化 / 并发 Init
    }
    if (!config_.Validate()) {
        initialized_.store(InitState::kUninit);
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }

    client_ = kv::CreateKvClient();
    auto init_status = client_->Init(config_.ucm_config_path);
    if (!init_status.ok()) {
        client_->Shutdown();
        client_.reset();
        initialized_.store(InitState::kUninit);
        return tl::make_unexpected(MapStatus(init_status.code));
    }

    auto staging_result = staging_.Init(config_.staging_bytes, slot_size);
    if (!staging_result) {
        staging_.Shutdown();
        client_->Shutdown();
        client_.reset();
        initialized_.store(InitState::kUninit);
        return staging_result;
    }

    total_keys_ = 0;
    total_size_ = 0;
    index_.clear();
    initialized_.store(InitState::kReady, std::memory_order_release);
    return {};
}

tl::expected<bool, ErrorCode> KvClientStorageBackend::IsEnableOffloading() {
    if (initialized_.load(std::memory_order_acquire) != InitState::kReady) {
        return tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
    }
    return true;
}
```

要点：

- `Init` **不打开设备、不建 backend 线程池**；ASU 连接、view 路由、task worker、transport
  线程由 `kv::KvClient` 内部管理。backend 只初始化 staging slot 池。
- `Init` 任一步失败都要 `Shutdown` client、`reset` 指针、把 `initialized_` 回滚为 `kUninit`，
  允许部署侧重试；不能留下半初始化状态。只有全部步骤成功才置 `kReady`。
- `IsEnableOffloading` **不查 ASU 容量、不发 RPC、不读计数**；`kReady` 后直接返回 true，
  未就绪返回错误让 `FileStorage::Init` 明确失败。
- `IsEnableOffloading` **只在 `FileStorage::Init` 调用一次**，返回值被锁存到 `enable_offloading_`；
  返回 false 等价于本进程直到重启都不再 offload。因此它是一次性特性门槛，不是实时配额探针。
- 配额/容量控制放在 `BatchOffload` 内逐批/逐 key 做：计数只统计本进程写入、重启后为 0，
  只作本地硬上限参考，不代表 ASU 实际容量；超限 key 跳过，返回 `expected(success_count)`/
  `expected(0)`，不返回 `KEYS_ULTRA_LIMIT`。瞬时容量不足由逐 key
  `RESOURCE_BUSY`/`CQE_OUT_OF_CREATE_SIZE` 处理。

### 3.3 直接使用 `kv::KvClient`

`KvClientStorageBackend` 直接调用 `kv::KvClient`，不再定义 Mooncake 自有的端口类型。
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
| 等待 | `bool Check(TaskId)` / `Status Wait(TaskId, uint64_t timeoutMs, TaskResult&)`；`Wait` 是破坏性读，超时/失败也会移除 task |
| 关停 | `Status Shutdown()` |
| key | `kv::CacheKey = std::array<std::byte, 8>` |
| 值 | `KVBuffer{key, Buffer{MemoryRegion, MRHandle}, offset, std::optional<uint32_t> mrKey}` |
| 结果 | `TaskResult{Status status, vector<Status> entryStatus, optional<QueryResult> queryResult}` |

约束：

- `KVBuffer.region.size` 是本次传输长度，`region.size` 和 `offset` 都必须 512B 对齐。
- `QueryResult.exists` 是存在性位图，没有长度。
- `TaskResult.status` 是聚合状态；必须逐 `entryStatus` 判断成功/失败。
- `kv::Status` / `StatusCode` 在 backend 内映射为 Mooncake `ErrorCode`，不向外透传。
- `KvClient` 是纯虚接口；测试用 `FakeKvClient : public kv::KvClient` 注入，CI 用仓库内的
  `ReferenceKvClient : public kv::KvClient`，两者都不需要生产 .so。
- UCM 头文件只出现在 `kv_client/backend.cpp`、`kv_client/reference_kv_client.cpp` 和测试
  `fake_kv_client.h`；backend 头文件只前置声明 `namespace kv { class KvClient; }`。

**`StagingArena::Acquire/Release` 的归属**

`Acquire()/Release()` **不是 `kv::KvClient` 的 API，而是 Mooncake 侧 `StagingArena` 的内部方法**。
UCM 只负责注册/注销内存和消费 `KVBuffer`，不向外分配 payload buffer：

| 事项 | Mooncake（StagingArena） | `kv::KvClient` |
| --- | --- | --- |
| 分配 payload 内存 | **是**：`staging_bytes` / `slot_size` 决定 slot | 否；公共接口没有 `Allocate/Acquire` |
| 注册内存 | 调用 `RegisterRegions` | 是：返回 `RegisteredMemory`/`MRHandle` |
| 组装 `KVBuffer` | **是**：填 `key`/`region`/`handle`/`offset` | 消费 `KVBuffer`，从 `handle` 解析 `mrKey` |
| 保证内存在 task 完成前有效 | **是**：slot 生命周期覆盖 `Wait` | 是：调用方必须保证；`UnregisterRegions` 只能在 task 完成后 |
| 注销内存 | 调用 `UnregisterRegions` | 是：执行注销 |

证据：

- `kv_semantics/include/kv_client.h:56-60` 只有 `RegisterRegions`/`UnregisterRegions`，没有
  `Acquire/Allocate/Free`。
- `kv_semantics/include/kv_types.h:109-134` 的 `MemoryRegion`/`KVBuffer`/`RegisteredMemory`
  都由调用方提供或由注册返回；`Buffer.handle` 是调用方注册时拿到的句柄。
- `kv_semantics/src/trans/buffer/buffer_manager.{h,cc}` 里的 `Allocate/Free` 是 transport 私有的
  SQE/CQE/注册槽管理，不在 `kv::KvClient` 公共头中，Mooncake 不能也不应直接使用。
- `kv_semantics/kv_test/src/buffer_allocator.cc` 与 `kv_client_runner.cc:175` 也是测试程序
  自己分配 buffer、再 `RegisterRegions`，说明 payload buffer 由调用方拥有。

因此：**Mooncake 必须自己做 `Acquire/Release`**，因为只有 Mooncake 知道 batch 大小、slot 预算、
`Wait` 何时完成、以及失败时如何复用/释放 slot。`KvClient` 只接受“已经注册好的内存”。

能否不做 staging？可以，但仍是 Mooncake 侧的工作：把已有 `ClientBuffer`/local segment 通过
`RegisterRegions` 注册一次，然后在已注册区域内分配 offset；这省掉一次拷贝，但不是“交给
KvClient 分配”，且要自己管理注册生命周期和 512B 对齐。v1 用独立 staging 最简单。

**`BatchStoreAsync` / `BatchLoadAsync` 的参数**

| 参数 | 传什么 | 不需要传什么 |
| --- | --- | --- |
| `entries` | `std::vector<KVBuffer>`；每个 entry：`key = asu_key`、`buffer.region = {type, addr, size = transfer_size}`、`buffer.handle = slot.handle`、`offset = 0`、`mrKey = nullopt` | 不需要 slot id、`scoped_key`、单独的 `payload_size`、batch index；KvClient 只看到“key + 已注册内存区间” |
| `taskId` | 输出参数；保存到 `Wait` | 不需要 batch id |
| `eventHandle` | 可选的前置设备事件；host staging 传 `0`，或直接用不带 event 的重载 | 不是 task id，也不是完成回调；不需要传 slot |
| 返回值 `Status` | 只表示“提交是否被接受” | 不是逐 key 结果；逐 key 结果在 `Wait` 后的 `TaskResult.entryStatus` |

补充：

- `eventHandle` 的语义是“在提交 Store 前先同步这个设备事件”（`kv_client_impl.cc:448, 559-560`；
  `runtime/ascend/ascend_device.cc:36-39` 调 `aclrtSynchronizeEvent`，simu 下 no-op）。只有当源内存在
  设备上、且由异步 kernel 生产时才有意义；我们的 HOST staging 传 0。
- `BatchLoadAsync` 没有 event 重载；提交后 `Wait` 完成再消费目标内存。
- `entries` 的向量顺序就是 `TaskResult.entryStatus` 的下标顺序；不需要额外传 batch index。
- `mrKey` 可留空：KvClient 从 `buffer.handle` 解析（`kv_types.h:122-127`）。

#### 3.3.1 KvClient 调用语义与后端必须遵守的约束

来源：`kvclient_deep_read.md` 第 3/7/8/9/10/11/13/14/15/17 章。

| 语义 | KvClient 事实 | 后端动作 |
| --- | --- | --- |
| 提交 | 成功只表示“已入队”；`RESOURCE_BUSY`=队列满，`NOT_INITIALIZED`=未初始化/Shutdown 中；`entries` 被拷贝进 `ClientTask`，但引用的内存仍必须有效 | 提交失败不产生 task，立即释放 slot；不提交空 entries |
| TaskId | 客户端任务表编号，非 0、不复用；一个 task 只能 `Wait` 一次 | 保存 taskId，严格一次 `Wait`；不缓存 taskId 跨批次 |
| Wait | **破坏性**：成功、失败、超时都会移除 task；超时**不取消**底层任务；超时后 `entryStatus` 只是当时快照；二次 `Wait` 得到 `TASK_NOT_FOUND` | 超时后不得复用/释放 slot，必须隔离；不得用超时快照提交索引/通知 Master |
| Check | 对未知/已完成 TaskId 都返回 true，无法区分“不存在”和“已完成” | v1 不用 Check 轮询；每 task 一次 `Wait` |
| 结果 | 顶层 `TaskResult.status` 通常是 `OK`/`PARTIAL_FAILED`；真实逐 key 结果在 `entryStatus`；`PARTIAL_FAILED` 时可能存在“OK 但未发送”的 entry | 逐 entry 判断；对 `PARTIAL_FAILED` 采取保守提交策略（见 §4.2.3/§4.3.3） |
| Query | 未命中不是失败：`TaskResult.status=OK`、`exists=0`、`entryStatus=ENTRY_KEY_NOT_EXIST`；transport 分片失败时该分片 `exists` 保持 0 | `IsExist` 必须同时看 `exists` 与 `entryStatus`；分片失败返回 `unexpected`，不能返回 false |
| Delete | key 路径，不需要注册 buffer；entry raw 0=OK，raw 1=`ENTRY_DELETE_FAILED`；没有 entry 级 `ENTRY_KEY_NOT_EXIST`；整个 CQE 为 `CQE_KEY_NOT_EXIST` 时所有 key 共享该状态 | `ENTRY_DELETE_FAILED` 视为失败；`CQE_KEY_NOT_EXIST` 视为“已不存在=成功”；`PARTIAL_FAILED` 按逐 key 处理 |
| 重试/取消 | 客户端不自动重试、不取消；`ENTRY_RETRY_ADVISED`/`NO_RETRY_ADVISED` 只是建议 | v1 不做自动重试；记日志并跳过，交给后续 offload/调用方 |
| 视图 | 任务在提交时钉住 `ViewSnapshot`；连接错误触发后台视图刷新；被移除的 transport 只 retire，Shutdown 才关闭；不重发任务 | 把 `CONNECTION_ERROR`/顶层 `NOT_FOUND` 映射为可重试错误；不自行重建 client |
| 批量 | `Store/Load` 的 ioNum=1（N 条=N 个 SQE）；`BatchStore/BatchLoad` ioNum 默认 110；Delete 254；Query 256；内部按 ASU 路由再拆子批 | 用 `Batch*` 提交多 key；`config.batch_size` 只控制客户端 TaskId 粒度；内部 ioNum 不暴露 |
| 内存 | `RegisterRegions` 返回 canonical handle；`KVBuffer.buffer.region` 可以是已注册区域的子区间；调用方传入的 `mrKey` 会被重置；`region.size` 是本地传输长度，`offset` 是远端 value 内偏移 | 注册整个 staging arena 一次，slot 用子区间；`mrKey` 留空；`offset=0` |
| 生命周期 | `Init` 非线程安全、失败不回滚；`Shutdown` 先停接受、再 drain；drain 超时后仍会注销内存，可能在飞 IO 引用已注销内存 | 失败后丢弃 client 对象重建；Shutdown 前确保没有在飞/隔离 slot；否则记录风险 |

**StagingSlot 状态机（后端侧）**

| 状态 | 含义 | 迁移 |
| --- | --- | --- |
| `Free` | 可分配 | `Acquire` → `InUse` |
| `InUse` | 已交给一个已提交 task | `Wait` 返回 OK/`PARTIAL_FAILED` 且结果消费完 → `Free`；`Submit` 失败 → `Free` |
| `Quarantined` | `Wait` 超时/`TASK_NOT_FOUND`/未知错误，底层任务可能仍在飞 | 不再复用、不释放；只在进程退出时随 arena 一起丢弃；计入隔离容量 |

- `staging_mutex_` 只保护 `Free` 队列和状态迁移；`Quarantined` slot 数量用于告警和 `BatchOffload` 的准入判断，不进入 `IsEnableOffloading`。
- `Acquire` 在无 `Free` slot 时做有界等待；超时则本批跳过/返回 `RESOURCE_BUSY`，不阻塞心跳。
- `Quarantined` slot 的内存不能 `UnregisterRegions`，因为底层 task 可能仍在引用；Shutdown 时 `KvClient::Shutdown` 会注销全部注册内存，这是已知风险，见 §4.6.2。

### 3.4 错误码映射

| `kv::StatusCode`（旧分支 `UC::ASU::StatusCode`）/ 本地条件 | Mooncake `ErrorCode` | 说明 |
| --- | --- | --- |
| `ENTRY_KEY_NOT_FOUND`, `ENTRY_DATA_NOT_EXIST`, `CQE_KEY_NOT_EXIST`（读） | `OBJECT_NOT_FOUND` | 读时返回；删时 `CQE_KEY_NOT_EXIST` 视为已不存在=成功 |
| **顶层** `NOT_FOUND` | `UNAVAILABLE_IN_CURRENT_STATUS` | 路由到的节点没有 transport / 路由失败，**不是** key 不存在 |
| `INVALID_ARGUMENT` | `INVALID_PARAMS` | |
| `NOT_INITIALIZED` | `INTERNAL_ERROR` | 未初始化/已 Shutdown |
| `TIMEOUT`（Wait）/ `CQE_IO_TIMEOUT` | `RPC_TIMEOUT` | Wait 超时**不取消**底层任务；slot 必须隔离 |
| `TASK_NOT_FOUND` | `INTERNAL_ERROR` | task 已被移除（重复 Wait/内部错误）；slot 隔离 |
| `PARTIAL_FAILED` | 首个失败 entry 的错误 | 逐 entry 继续；见 §4.2.3/§4.3.3 的保守提交规则 |
| `CONNECTION_ERROR` | `UNAVAILABLE_IN_CURRENT_STATUS` | 可重试；会触发后台视图刷新 |
| `IO_ERROR`, `CQE_WRITE_FAULT` | `FILE_WRITE_FAIL` | |
| `CQE_UNRECOVERED_READ_ERROR` | `FILE_READ_FAIL` | |
| `BUFFER_NOT_REGISTERED`, `BUFFER_NOT_SUPPORTED` | `INVALID_PARAMS` | staging 注册/handle 错误 |
| `RESOURCE_BUSY`, `IN_PROGRESS` | `UNAVAILABLE_IN_CURRENT_STATUS` | `RESOURCE_BUSY`：submit 队列/transport 队列满，稍后重试 |
| `CQE_OUT_OF_CREATE_SIZE`, `CQE_RESOURCE_BUSY` | `UNAVAILABLE_IN_CURRENT_STATUS` | ASU 容量/资源不足；本批跳过 |
| `UNSUPPORTED` | `NOT_SUPPORTED` | |
| `CANCELED` | `INTERNAL_ERROR` | 任务被取消（Shutdown/未派发） |
| `ENTRY_RETRY_ADVISED` | 不自动重试 | 记日志并跳过；交给后续 offload/调用方 |
| `ENTRY_NO_RETRY_ADVISED` | `INTERNAL_ERROR` | 记日志并跳过 |
| `ENTRY_KEY_EXIST`, `CQE_KEY_ALREADY_EXISTED` | 视为成功（upsert） | 本版不依赖条件写 |
| `ENTRY_DELETE_FAILED` | `FILE_WRITE_FAIL` | 删除失败；恢复索引 |
| Query `exists=1` + `ENTRY_KEY_EXIST` | `expected(true)` | Query 命中不是错误 |
| Query `exists=0` + `ENTRY_KEY_NOT_EXIST` | `expected(false)` | Query 未命中不是失败 |
| Query `exists=0` 但 `entryStatus` 为 `IO_ERROR`/其他错误 | `unexpected(mapped)` | **不能**当成“对象不存在” |
| `RecordHeader` magic/version/length 不符 | `INVALID_READ` | 内容损坏 |
| `RecordHeader` CRC-32C 不符 | `INVALID_READ` | 头不可信 |
| payload CRC-32C 不符 | `CHECKSUM_MISMATCH` | 撕裂/位翻转 |
| 本地 `ptr == nullptr` / `size == 0` | `INVALID_PARAMS` | 读 |

### 3.5 语义规约

| 属性 | 规约 |
| --- | --- |
| 写原子性 | 单 key 一个 ASU task 条目；ASU 负责该条目的可见性 |
| 写幂等 | 同 key 同 payload 重复 Store 字节确定；Store 是 upsert |
| 批量原子性 | 无事务；`PARTIAL_FAILED`/逐 entry 失败是正常结果。`PARTIAL_FAILED` 时可能存在“entryStatus=OK 但实际未发送”的子批，写/读必须保守处理（§4.2.3/§4.3.3） |
| 读一致性 | 返回最近一次成功 Store 的完整 value；CRC 检测撕裂 |
| 读的输入 | key + `slice.size` 由 Master 提供；索引缺失时以 `slice.size` 为准 |
| 查语义 | 同时看 `queryResult.exists[i]` 与 `entryStatus[i]`：exists=1+`ENTRY_KEY_EXIST`→true；exists=0+`ENTRY_KEY_NOT_EXIST`→false；其他→`unexpected`（分片失败不能当不存在） |
| 长度来源 | Master metadata（读）、写路径计算（写）；ASU 不提供长度 |
| 可见性 | `complete_handler` 成功之前，Master 不知道该副本 |
| 容量计数 | `total_size_` 计 `align_up(24 + payload, 512)`；仅统计本进程写入，重启后从 0 开始 |
| 删除 | master-first（Master 先删元数据）；`BatchRemoveKeys` 逐 key `DeleteAsync`；entry raw 0=成功、raw 1=`ENTRY_DELETE_FAILED` 失败；整个 CQE `CQE_KEY_NOT_EXIST` 视为已不存在=成功；无 entry 级 KEY_NOT_EXIST |
| RemoveAll | 本版不支持；继承空实现；`Client::RemoveAll` 不会物理删除 ASU |
| 恢复 | 本版不恢复；`ScanMeta` 返回 OK 空扫描 |
| 二次 Init | 返回 `INTERNAL_ERROR` |

### 3.6 Key 与记录布局

**Mooncake 的真实 key 形态**

| 层 | 类型/格式 | 说明 | 证据 |
| --- | --- | --- | --- |
| 用户 / Client API | `ObjectKey = std::string` | 用户传入的对象 key，不含 tenant | `types.h:201` |
| Master 元数据 | `ObjectIdentity { TenantId tenant_id; std::string user_key; }` | tenant 是独立字段，不是 `user_key` 的一部分 | `master_service.h:1201-1204, 1302-1305` |
| Master offload 任务 | `OffloadTaskItem { std::string tenant_id; std::string key; int64_t size; }` | 心跳返回的任务按 tenant + 裸 key 携带 | `types.h:268-277` |
| FileStorage → StorageBackend | `scoped_key = tenant_id + '\0' + local_key` | `TenantId::MakeScopedKey`；`BatchOffload`/`BatchLoad`/`IsExist`/`BatchRemoveKeys` 都用它 | `tenant_id.h:42-48`；`file_storage.cpp:367-370, 488, 582, 1399-1404` |
| Backend → ASU | `asu_key = XXH64(..., 8B)` | 不可逆 | 本设计 |

- `TenantId` 默认 `"default"`；tenant 只允许可打印 ASCII（`0x20..0x7e`）、不能以 `_` 开头；
  分隔符是 `'\0'`（`tenant_id.h`）。
- `StorageObjectMetadata.key_size = scoped_key.size()`；`data_size = payload_size`。
- 后端只看到 `scoped_key`，看不到裸 `ObjectKey`；不要假设 key 里没有 `'\0'`。

**`asu_key`（`kv::CacheKey`）**

```text
asu_key = XXH64("mooncake/kvclient/v1" + key_namespace + 0x00 + scoped_key, seed=0)
```

- `asu_key` 是 `kv::CacheKey`（8 字节），按小端写入。
- `key_namespace` 由配置必填；它参与哈希，不同命名空间的同一 `scoped_key` 得到不同 `asu_key`。
  它只是 ASU 侧的逻辑命名空间，与 Master 的 `TenantId` 无关（见 §3.7）。
- **本版不处理 64-bit 哈希碰撞**：默认不发生小概率碰撞。若后续需要，可增加 slot 探测或版本化 key。

**记录布局（`RecordHeader` + payload）**

```text
0   u32 magic          0x4B564331 ("KVC1")
4   u16 version        1
6   u16 flags          bit0 chunked（本版必须 0）; bit1 has_payload_crc
8   u32 payload_size   单 payload <= UINT32_MAX
12  u32 reserved       0
16  u32 payload_crc32c
20  u32 header_crc32c  对 [0,20) 计算，本字段按 0 参与
24  ... payload        各 Slice 按序拼接，无补齐
```

- `record_size = 24 + payload_size`；`transfer_size = align_up(record_size, 512)`。
- `KVBuffer`：`key = asu_key`、`offset = 0`、`region.size = transfer_size`、`handle = slot.handle`。
- 上报 Master 的 `data_size = payload_size`。
- 读校验顺序：`transfer_size >= 24` → magic → version → flags → `payload_size` 在
  `transfer_size - 24` 以内 → `RecordHeader` CRC → payload CRC。
- 单对象上限：`min(config.max_payload_bytes, ASU 单 value 上限 - 24)`，再受 staging slot 大小约束；
  超限 key 在写路径跳过并记录。

**`payload_size` / `transfer_size` 的区别与例子**

- `payload_size`：对象真实数据长度，等于各 `Slice.size` 之和；不含 `RecordHeader`。它就是上报
  Master 的 `data_size`，也是读路径里 `slice.size` 的期望值。
- `record_size = 24 + payload_size`：`RecordHeader` + payload 的逻辑长度。
- `transfer_size = align_up(record_size, 512)`：实际向 ASU 读/写的字节数，包含 24B 头和零填充；
  `KVBuffer.region.size` 必须用它，`offset = 0`。

| `payload_size` | `record_size` | `transfer_size` | 填充字节 |
| --- | --- | --- | --- |
| 0 | 24 | 512 | 488 |
| 471 | 495 | 512 | 17 |
| 488 | 512 | 512 | 0 |
| 489 | 513 | 1024 | 511 |
| 1000 | 1024 | 1024 | 0 |
| 4096 | 4120 | 4608 | 488 |

写一个 1000B 对象（两个 Slice：600B + 400B）：

- `payload_size = 1000`、`record_size = 1024`、`transfer_size = 1024`。
- staging slot 布局：`[24B RecordHeader][600B][400B]`（恰好 1024B，无填充）。
- `KVBuffer.region.size = 1024`、`offset = 0`；上报 Master 的 `data_size = 1000`。

读同一个对象（Master 给出 `slice.size = 1000`）：

- `payload_size = slice.size = 1000`、`transfer_size = 1024`。
- 从 ASU 读 1024B 到 staging；头里的 `payload_size` 必须等于 1000。
- 校验 `RecordHeader` CRC、payload CRC。
- 只 `memcpy` 1000B 到 `slice.ptr`。
- 索引记 `{payload_size=1000, transfer_size=1024}`；`total_size_` 按 1024 计。

填充字节不参与 payload CRC，也不返回给调用方；读时以头里的 `payload_size` 决定拷贝长度。

**CRC-32C 的定位**：`mooncake-store/include/crc32c.h` 是 Mooncake 自己的 CRC-32C 实现；
它是标准算法，但不是 ASU 或 Mooncake 的全局强制格式。`NvmeKvStorageBackend` 使用
XXH32 + 32 字节 `verify_hash`，与本后端的 key/记录格式不互通。ASU 公共 `KvClient` 没有
per-value checksum；transport 校验不覆盖 at-rest 数据。因此本后端读到的每条 value（一条 `Record`）
都必须按 `KVC1` 记录格式校验。

**是否必须**：

| 校验 | 本版 | 理由 |
| --- | --- | --- |
| `RecordHeader` CRC-32C（24B） | **必须** | 只覆盖 20B，成本极低；保护 `payload_size`/flags，防止按错误长度拷贝 |
| payload CRC-32C | **本版固定开启** | ASU 不提供 per-value checksum；撕裂/位翻转只能靠它发现。后续性能评估可加 `enable_payload_crc`，但 KV cache 场景不建议关闭 |
| 替代算法 | 不采用 | XXH32/XXH3 更快，但哈希不是纠错码；对 burst error 的检测弱于 CRC-32C |

**写批次（`PreparedOffloadBatch`）**

`PrepareBatch` 的返回值：一批已通过预检查、记录已编码进 staging slot 的 key 集合。

```cpp
struct PreparedOffloadBatch {
    struct PreparedEntry {
        std::string scoped_key;  // tenant\0local_key，用于索引与 complete_handler
        kv::CacheKey asu_key;    // 8B ASU key（哈希后）
        uint32_t payload_size;   // 不含 RecordHeader（24B）
        uint32_t transfer_size;  // align_up(24 + payload_size, 512)
        StagingSlot slot;        // 已注册内存；OK/PARTIAL 后 Release，TIMEOUT/TASK_NOT_FOUND 进入 Quarantined
    };
    std::vector<PreparedEntry> entries;  // 通过预检查、可提交 ASU 的 key
    std::vector<std::string> skipped;    // 超限/超配额/编码失败，仅用于日志与计数
};
```

- `entries` 与 `skipped` 互斥且不重不漏：每个输入 `scoped_key` 恰好落在一侧。
- `slot` 在 `Wait` 返回且 entry 状态消费后才 `Release`；`PreparedOffloadBatch` 析构不释放 slot。

**与 NVMeKV 是否互斥**：

- **同一 FileStorage / client 进程内互斥**：`FileStorageConfig.storage_backend_type` 是单值，
  `FileStorage` 只持有一个 `StorageBackendInterface`；`CreateStorageBackend` 按枚举选一个。
  NVMeKV 和 `KvClientStorageBackend` 可以同时编译进二进制，但运行时只实例化一个。
- **数据格式互斥**：NVMeKV 是 16B 物理 key + `NvmeKvObjectHeader`；本后端是 8B `asu_key` +
  `KVC1`。两者不能互读同一份数据，不能指向同一个 ASU namespace/设备。
- **集群层面**：Master 只看到 `LOCAL_DISK` replica，不关心后端类型；理论上不同 client 可以用
  不同后端，但删除/恢复/容量语义不同，运维复杂。建议同一个 pool 使用同一种后端。
- **迁移**：换后端必须 drain 旧后端并重新 offload，不能双读；不存在“NVMeKV 和
  `KvClientStorageBackend` 同时服务同一 key”的模式。

### 3.7 配置契约

```cpp
enum class KvClientImplType { kUcm, kReference };  // 选哪个 kv::KvClient 实现
std::ostream& operator<<(std::ostream& os, KvClientImplType impl);  // 日志输出 "ucm"/"reference"

struct KvClientBackendConfig {
    KvClientImplType client_impl = KvClientImplType::kUcm;
    std::string ucm_config_path;      // UCM 的 view/transport 配置文件（不是 Mooncake 配置）
    std::string client_id;            // KvClientConfig.clientId
    std::string key_namespace;        // 必填，参与 ASU key 哈希；与 TenantId 无关
    int64_t staging_bytes = 128 * 1024 * 1024;
    size_t batch_size = 32;
    int64_t max_payload_bytes = 16 * 1024 * 1024;
    uint64_t wait_timeout_ms = 5000;
    uint64_t query_timeout_ms = 500;

    static KvClientBackendConfig FromEnvironment();
    bool Validate() const;
};
```

- `client_impl` 是枚举而不是字符串：`kUcm`（生产）/ `kReference`（参考实现，仅 CI/单测）。
  枚举名加 `Type` 后缀是为了避开 UCM 自己的实现类 `kv::KvClientImpl`；`operator<<` 让日志直接输出
  `ucm` / `reference`，与 `BucketEvictionPolicy`、`OffsetPersistMode` 的做法一致。生产部署出现
  `kReference` 视为配置错误（与 FM-CFG-02 一致）。
- `ucm_config_path` 是 UCM 的 view/transport 配置文件路径，不是 Mooncake 自己的配置文件。
- `key_namespace` 只用于 ASU key 哈希的域隔离；它**不是** Master 的 `TenantId`：租户隔离由
  `scoped_key` 的 `tenant\0local_key` 前缀表达，不参与 ASU key 计算。
- `max_payload_bytes` 是 payload 大小（不含 `RecordHeader`，24B），与上报 Master 的 `data_size` 同义。
- 字节配置沿用仓库的 `int64_t`（`bucket_size_limit`、`total_size_limit`）；两个超时用 `uint64_t`，
  因为 UCM `Wait` 接收 `uint64_t`。`Validate()` 检查 `staging_bytes`、`batch_size`、
  `max_payload_bytes`、`wait_timeout_ms`、`query_timeout_ms` 均 `> 0`，并检查 `key_namespace` 非空。
- **超时关系**：UCM 的 `transport.timeoutMs` 默认 100ms，`defaultWaitTimeoutMs` 默认 100ms；
  `wait_timeout_ms`/`query_timeout_ms` 必须大于它，或把 transport `timeoutMs` 设为 0。否则
  transport 先超时，`Wait` 可能拿到 `TIMEOUT`/entry 错误。`Validate()` 无法读取 UCM 配置文件，
  这一条需要在部署文档中明确。
- `batch_size` 只控制客户端 `TaskId` 的粒度（默认 32）；UCM 内部再按 `batchStoreIoNum`/
  `batchLoadIoNum`（默认 110）、`deleteIoNum`（254）、`queryIoNum`（256）切 SQE。
- UCM 的 `maxInflightTasks`（默认 1024）是客户端队列深度；本后端每块提交后立即 `Wait`，
  正常情况下同时只有 1 个在飞 task，不会压满队列。

环境变量（沿用现有 offload 配置的 `MOONCAKE_OFFLOAD_*` 前缀）：

| 变量 | 字段 |
| --- | --- |
| `MOONCAKE_OFFLOAD_KV_CLIENT_IMPL` | `client_impl`（`ucm` / `reference`） |
| `MOONCAKE_OFFLOAD_KV_CLIENT_UCM_CONFIG_PATH` | `ucm_config_path` |
| `MOONCAKE_OFFLOAD_KV_CLIENT_ID` | `client_id` |
| `MOONCAKE_OFFLOAD_KV_CLIENT_KEY_NAMESPACE` | `key_namespace` |
| `MOONCAKE_OFFLOAD_KV_CLIENT_STAGING_BYTES` | `staging_bytes` |
| `MOONCAKE_OFFLOAD_KV_CLIENT_BATCH_SIZE` | `batch_size` |
| `MOONCAKE_OFFLOAD_KV_CLIENT_MAX_PAYLOAD_BYTES` | `max_payload_bytes` |
| `MOONCAKE_OFFLOAD_KV_CLIENT_WAIT_TIMEOUT_MS` | `wait_timeout_ms` |
| `MOONCAKE_OFFLOAD_KV_CLIENT_QUERY_TIMEOUT_MS` | `query_timeout_ms` |

**`FileStorageConfig` 的 descriptor 与 `Validate()` 配套修改**：

- 新增 descriptor 字符串 `kv_client_storage_backend`（与 `file_per_key_storage_backend`、
  `nvme_kv_storage_backend` 同风格），在 `file_storage_config.cpp` 的解析分支里映射到
  `StorageBackendType::kKvClient`。
- `kFilePerKey` / `kBucket` / `kOffsetAllocator` / `kDistributed`：保持现有本地路径校验。
- `kKvClient`：跳过 `ValidatePath`；改为校验 `KvClientBackendConfig::Validate()` 的字段
  （`key_namespace`、`staging_bytes`、`batch_size`、`max_payload_bytes`、两个超时等）。
- `total_keys_limit > 0` 与 `total_size_limit != 0` 仍要求。

### 3.8 能力与限制矩阵（本版）

| 能力/限制 | 本版处理 |
| --- | --- |
| key 8 字节 | 哈希映射；本版不处理 64-bit 碰撞（默认不发生） |
| 无长度查询 | 写时计算；读时用 Master 的 `slice.size` |
| 无枚举 | `ScanMeta` 返回 OK 空扫描；不做恢复 |
| 512B 对齐 | `transfer_size = align_up(record_size, 512)`；`offset=0` |
| 注册内存 | staging arena；读写都经过 staging |
| 单 payload 上限 | `min(config.max_payload_bytes, ASU 单 value 上限 - 24)` |
| 批量 key 上限 | `config.batch_size`（默认 32）；UCM 内部再按 ioNum 110/110/254/256 切 SQE |
| 异步 task | 每块提交后只 `Wait` 一次；`TIMEOUT`/`TASK_NOT_FOUND` → `RPC_TIMEOUT` + slot `Quarantined` |
| Submit 队列满 | `RESOURCE_BUSY`：没有 task，立即释放 slot；下一轮重试 |
| Query 分片失败 | `exists=0` 不能当不存在；必须检查 `entryStatus` |
| ASU 自身 GC | 无法观测；读/查遇到明确的 `ENTRY_KEY_NOT_FOUND`/`CQE_KEY_NOT_EXIST` 才按缺失处理 |
| 删除 | 仅 `Client::Remove`/`BatchRemove` 触发的按 key 删除；无 `RemoveAll`/正则/水位淘汰 |

## 4. 功能实现设计

### 4.1 公共类图

```mermaid
classDiagram
    class StorageBackendInterface {
        <<interface>>
        +Init() expected
        +BatchOffload(batch_object, complete_handler, eviction_handler) int64
        +BatchLoad(slices) expected
        +IsExist(key) expected
        +IsEnableOffloading() expected
        +ScanMeta(handler) expected
        +ResetScanIterator() void
        +RemoveAll() void
        +EvictAboveDiskWatermark(high, low, handler) expected
        +SetTestFailurePredicate(predicate) void
        +BatchRemoveKeys(scoped_keys) vector
    }

    class KvClientStorageBackend {
        -config_ KvClientBackendConfig
        -client_ KvClient
        -staging_ StagingArena
        -write_mutex_ std::mutex
        -metadata_mutex_ std::shared_mutex
        -staging_mutex_ std::mutex
        -index_ PayloadSizeIndex
        -total_keys_ int64
        -total_size_ int64
        -initialized_ std::atomic~InitState~
        +Init() expected
        +BatchOffload(batch_object, complete_handler, eviction_handler) int64
        +BatchLoad(slices) expected
        +IsExist(key) expected
        +IsEnableOffloading() expected
        +ScanMeta(handler) expected
        +ResetScanIterator() void
        +BatchRemoveKeys(scoped_keys) vector
        -PrepareBatch(batch_object) PreparedOffloadBatch
        -CommitIndex(scoped_key, payload_size)
        -RemoveKeys(scoped_keys) RemoveOutcome
    }

    class InitState {
        <<enumeration>>
        kUninit
        kInitializing
        kReady
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

    class KvClientImpl {
        +KvClientImpl()
        +Init(config) Status
        +BatchStoreAsync(entries, taskId) Status
        +BatchLoadAsync(entries, taskId) Status
        +QueryAsync(keys, taskId) Status
        +DeleteAsync(keys, taskId) Status
        +Wait(taskId, timeoutMs, result) Status
    }

    note for KvClientImpl "UCM 侧实现（kv::KvClient 的具体实现）"

    class KvClientBackendConfig {
        +client_impl KvClientImplType
        +ucm_config_path string
        +client_id string
        +key_namespace string
        +staging_bytes int64
        +batch_size size
        +max_payload_bytes int64
        +wait_timeout_ms uint64
        +query_timeout_ms uint64
        +FromEnvironment() KvClientBackendConfig
        +Validate() bool
    }

    class KvClientImplType {
        <<enumeration>>
        kUcm
        kReference
    }

    class KvClientKeyCodec {
        +EncodeKey(key_namespace, scoped_key) kv::CacheKey
    }

    class KvClientRecordCodec {
        +Encode(scoped_key, slices) EncodedRecord
        +Parse(buffer) DecodedRecord
        +TransferSize(record_size) size
    }

    class PayloadSizeIndex {
        +Upsert(scoped_key, entry) IndexDelta
        +Snapshot(scoped_key) IndexEntry
        +Remove(scoped_key) IndexEntry
        +TotalKeys() int64
        +TotalSize() int64
    }

    class StagingArena {
        +Init(bytes, slot_size) expected
        +Acquire() optional
        +Release(slot) void
        +Quarantine(slot) void
        +RegisteredRegions() vector
        +Shutdown() void
    }

    class ReferenceKvClient {
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

    StorageBackendInterface <|-- KvClientStorageBackend
    KvClientStorageBackend o-- KvClient
    KvClientStorageBackend o-- KvClientBackendConfig
    KvClientBackendConfig ..> KvClientImplType
    KvClientStorageBackend *-- PayloadSizeIndex
    KvClientStorageBackend *-- StagingArena
    KvClientStorageBackend ..> KvClientKeyCodec
    KvClientStorageBackend ..> KvClientRecordCodec
    KvClientStorageBackend ..> InitState
    KvClient <|.. KvClientImpl
    KvClient <|.. ReferenceKvClient
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
    class KvClientStorageBackend {
        +BatchOffload(batch_object, complete_handler, eviction_handler) int64
        -PrepareBatch(batch_object) PreparedOffloadBatch
        -CommitIndex(scoped_key, payload_size)
    }
    class KvClientKeyCodec {
        +EncodeKey(key_namespace, scoped_key) kv::CacheKey
    }
    class KvClientRecordCodec {
        +Encode(scoped_key, slices) EncodedRecord
    }
    class StagingArena {
        +Acquire() optional
        +Release(slot) void
        +Quarantine(slot) void
    }
    class KvClient {
        +BatchStoreAsync(entries, taskId) Status
        +Wait(taskId, timeoutMs, result) Status
    }
    class PayloadSizeIndex {
        +Upsert(scoped_key, entry) IndexDelta
    }
    FileStorage --> KvClientStorageBackend : calls
    KvClientStorageBackend --> KvClientKeyCodec : encode key
    KvClientStorageBackend --> KvClientRecordCodec : encode record
    KvClientStorageBackend --> StagingArena : acquire
    KvClientStorageBackend --> KvClient : store and wait
    KvClientStorageBackend --> PayloadSizeIndex : commit
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
    FS->>BE: BatchOffload(scoped_batch, complete_handler, eviction_handler)
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

    alt Submit failed (RESOURCE_BUSY / NOT_INITIALIZED)
        BE->>SA: Release(slots)
        BE-->>FS: error
    else Wait TIMEOUT / TASK_NOT_FOUND
        BE->>SA: Quarantine(slots)
        BE-->>FS: RPC_TIMEOUT
    else Wait OK
        BE->>KC: Wait(taskId, wait_timeout_ms)
        KC-->>BE: TaskResult OK + per-entry status
        BE->>BE: CommitIndex for OK entries
        BE->>SA: Release(slots)
    else Wait PARTIAL_FAILED
        BE->>KC: Wait(taskId, wait_timeout_ms)
        KC-->>BE: TaskResult PARTIAL_FAILED
        BE->>BE: publish nothing for this chunk (pre-send OK pitfall)
        BE->>SA: Release(slots)
    end

    BE->>FS: complete_handler(published_scoped_keys, metadatas)
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

1. `initialized_ != kReady`（未初始化或初始化中）→ `INTERNAL_ERROR`。
2. `batch_object` 为空 → `INVALID_KEY`。
3. 配额/容量检查在本方法内逐 key 做（见步骤 4）；`IsEnableOffloading()` 只用于
   `FileStorage::Init` 的一次性门槛，不在每批调用，也不会因瞬时配额返回 false。
4. 逐 key 预处理：
   - `scoped_key` 为空 → 跳过并记录 `INVALID_KEY`。
   - 计算 `asu_key`（`kv::CacheKey`，8B）；payload 超有效上限
     （`min(config.max_payload_bytes, ASU 单 value 上限 - 24)`）、超 `UINT32_MAX`、或
     `transfer_size` 超过 staging slot → 跳过并记录 `INVALID_PARAMS`。
   - 拼接 Slice 为连续 payload；生成 `RecordHeader`（24B）；尾部补零到 `transfer_size`。
   - 若 `total_size_ + transfer_size > total_size_limit` 或
     `total_keys_ >= total_keys_limit`，跳过该 key；本版不做淘汰来腾空间。
5. 从 `StagingArena` 分配 slot；把记录拷入 slot 的已注册内存。slot 不足时按可用数量切块，
   或本批剩余 key 跳过。
6. 按 `config.batch_size` 切块；多 key 用 `BatchStoreAsync`，单 key 用 `StoreAsync`：
   - **Submit 失败**（`RESOURCE_BUSY`/`NOT_INITIALIZED`/`INVALID_ARGUMENT`）：没有 task，
     立即 `Release` 本块 slot；记录错误，继续/结束。
   - Submit 成功后每块**只 `Wait` 一次** `Wait(task_id, wait_timeout_ms, result)`。
7. 按 `Wait` 的结果分派：
   - `TaskResult.status == OK`：逐 entry 检查；`entryStatus[i]` 为 OK 的 entry 才可发布。
   - `TaskResult.status == PARTIAL_FAILED`：**本块整体不发布**（保守策略）。原因：KvClient 在预发送
     失败时可能留下“entryStatus=OK 但子批从未发送”的 entry；无法区分，发布会产生假成功。
     slot 在结果消费后 `Release`；记录逐 entry 错误。
   - `Wait` 返回 `TIMEOUT` / `TASK_NOT_FOUND` / 未知错误：task 已被移除、底层可能仍在飞；
     **本块所有 slot 进入 `Quarantined`**，不发布、不扣计数、不通知 Master；返回对应错误。
   - `CONNECTION_ERROR`/`IO_ERROR`/`CANCELED`：task 已结束；`Release` slot；不发布；返回错误。
8. 对每个可发布 entry，在 `metadata_mutex_` 独占锁内提交索引：
   - `old = index_[scoped_key]`；`total_size_ += transfer_size - old.transfer_size`；
     `total_keys_ += (old 存在 ? 0 : 1)`。
9. 只把可发布 key 上报：

   ```cpp
   StorageObjectMetadata{
       .bucket_id = -1,
       .offset = 0,
       .key_size = static_cast<int64_t>(scoped_key.size()),
       .data_size = static_cast<int64_t>(payload_size),  // 不含 RecordHeader（24B）
       .transport_endpoint = "",
   }
   ```

10. `complete_handler` 返回非 OK → 原样返回该错误；已写入的数据不回滚。
11. 返回成功上报的 key 数量；若 0 个成功，仍返回 `expected(0)`。

失败处理：

- 单个 entry 的 ASU 错误：该 entry 不发布，保留首错用于日志；同块其他 entry 若顶层是
  `PARTIAL_FAILED` 也按保守策略不发布。
- `Submit` 失败：没有 task，slot 立即释放；`RESOURCE_BUSY` 可稍后由下一轮心跳重试。
- `Wait` 超时：**不要复用/释放 slot**，置为 `Quarantined`；不通知 Master；ASU 侧可能稍后完成，
  形成未被 Master 登记的孤儿对象。重试 Put 会覆盖它。
- `PARTIAL_FAILED`：本块整体不发布；宁可由 Master 重新 offload，也不上报可能未发送的 entry。
- `StagingArena` 分配失败/耗尽：本批剩余 key 跳过，返回已成功数量；不长期阻塞心跳。
- 超限 key 会被反复选中：进程内维护有界“跳过集合”，避免重复编码和日志。
- `BatchOffload` 本身不删除 key；删除由 `Client::Remove`/`BatchRemove` 经 Master→owner 触发
  （§4.5）。

### 4.3 读：`BatchLoad`

读路径在 Mooncake 侧的完整分层（副本 / 元数据 / 句柄、owner 侧 offload RPC 与 client buffer）见 §2.6；
本节只展开图中 `KvClientStorageBackend` 这一个节点。

#### 4.3.1 读操作类图

```mermaid
classDiagram
    class FileStorage {
        +BatchGetLocal(keys, sizes)
        +LoadBatch(keys, sizes)
    }
    class KvClientStorageBackend {
        +BatchLoad(slices) expected
        -LookupIndex(scoped_key) optional
        -ValidateRecord(buffer) DecodedRecord
        -CommitIndex(scoped_key, payload_size)
    }
    class PayloadSizeIndex {
        +Snapshot(scoped_key) IndexEntry
        +Upsert(scoped_key, entry) IndexDelta
        +Remove(scoped_key) IndexEntry
    }
    class StagingArena {
        +Acquire() optional
        +Release(slot) void
        +Quarantine(slot) void
    }
    class KvClient {
        +BatchLoadAsync(entries, taskId) Status
        +Wait(taskId, timeoutMs, result) Status
    }
    class KvClientRecordCodec {
        +Parse(buffer) DecodedRecord
    }
    FileStorage --> KvClientStorageBackend : calls
    KvClientStorageBackend --> PayloadSizeIndex : optional cache
    KvClientStorageBackend --> StagingArena : acquire
    KvClientStorageBackend --> KvClient : load and wait
    KvClientStorageBackend --> KvClientRecordCodec : validate
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
        BE->>BE: payload_size = slice.size, transfer_size = align_up(24 + payload_size, 512)
    end

    BE->>SA: Acquire(slots)
    BE->>KC: BatchLoadAsync(entries)
    KC-->>BE: taskId

    alt Submit failed
        BE->>SA: Release(slots)
        BE-->>FS: error
    else Wait TIMEOUT / TASK_NOT_FOUND
        BE->>KC: Wait(taskId, wait_timeout_ms)
        KC-->>BE: TIMEOUT / task removed
        BE->>SA: Quarantine(slots)
        BE-->>FS: RPC_TIMEOUT
    else Wait PARTIAL_FAILED
        BE->>KC: Wait(taskId, wait_timeout_ms)
        KC-->>BE: PARTIAL_FAILED
        BE->>SA: Release(slots)
        BE-->>FS: error (no entry returned to caller)
    else Wait OK
        BE->>KC: Wait(taskId, wait_timeout_ms)
        KC-->>BE: TaskResult OK + per-entry status
        loop each loaded entry
            alt not found
                BE->>BE: remove index entry if present
                BE-->>FS: OBJECT_NOT_FOUND (first_error)
            else RecordHeader invalid
                BE-->>FS: INVALID_READ (first_error)
            else CRC mismatch
                BE-->>FS: CHECKSUM_MISMATCH (first_error)
            else valid
                BE->>RC: Parse(buffer)
                BE->>FS: memcpy payload into slice.ptr
                BE->>BE: CommitIndex (optional cache)
            end
        end
        BE->>SA: Release(slots)
        BE-->>FS: OK or first_error
    end
```

#### 4.3.3 实现步骤

1. 空 map → `OK`。
2. 对每个输入 key：
   - `slice.ptr == nullptr` 或 `slice.size == 0` → `INVALID_PARAMS`。
   - **`payload_size = slice.size`**（Master 提供的权威长度）；`transfer_size =
     align_up(24 + payload_size, 512)`；超过 staging slot → `INVALID_PARAMS`。
   - 索引是可选缓存：命中时可用于日志/统计；**不要求命中**。
3. 分配 staging slot，构造 `KVBuffer{key, offset=0, region.size=transfer_size, handle}`。
4. 按 `batch_size` 切块，`BatchLoadAsync`；每块**只 `Wait` 一次**：
   - **Submit 失败**：立即 `Release` 本块 slot；返回错误。
   - `TaskResult.status == OK`：逐 entry 校验/拷贝；slot 在结果消费后 `Release`。
   - `TaskResult.status == PARTIAL_FAILED`：**本块不向调用方提供任何 entry**（预发送失败时可能有
     “OK 但未写入目标 buffer”的 entry）；`Release` slot；返回首个失败 entry 的错误。
   - `Wait` 返回 `TIMEOUT`/`TASK_NOT_FOUND`/未知错误：**slot 进入 `Quarantined`**，不释放；
     返回 `RPC_TIMEOUT`/对应错误。
5. `OK` 结果下逐 entry 校验：
   - `ENTRY_KEY_NOT_FOUND`/`CQE_KEY_NOT_EXIST`：从索引删除（若存在），记 `OBJECT_NOT_FOUND`。
   - `CONNECTION_ERROR`/`IO_ERROR`/其他 entry 错误：映射后记首错；该 entry 不拷贝。
   - 读回长度 < 24（不足一个 `RecordHeader`）：`INVALID_READ`。
   - magic/version/flags 不符：`INVALID_READ`。
   - `payload_size` 超出 buffer 或与 `slice.size` 不一致：`INVALID_PARAMS`。
   - `RecordHeader` CRC 不符：`INVALID_READ`。
   - payload CRC 不符：`CHECKSUM_MISMATCH`。
   - 全部通过：`memcpy(slot + 24, slice.ptr, payload_size)`，并按 `payload_size` 更新索引。
6. `Release` slot（`TIMEOUT`/`TASK_NOT_FOUND` 除外，它们进入 `Quarantined`）；返回 `OK` 或首个错误；
   已经写入的 slice 不回滚。**不改写 `slice.ptr`**。

**为什么需要两层 CRC**：

- 头 CRC-32C 保护“怎么读”：magic/version/flags/payload_size 决定拷贝长度和记录类型。
- payload CRC-32C 保护“读到的内容”：检测撕裂、位翻转、写入中断。
- CRC-32C 只检测偶然损坏，不防恶意篡改；512B padding 不参与 payload CRC。

### 4.4 查：`IsExist`

#### 4.4.1 查询类图

```mermaid
classDiagram
    class FileStorage {
        +Exists(scoped_key) expected
    }
    class KvClientStorageBackend {
        +IsExist(scoped_key) expected
        -RemoveIndexEntry(scoped_key)
    }
    class KvClient {
        +QueryAsync(keys, taskId) Status
        +Wait(taskId, timeoutMs, result) Status
    }
    class PayloadSizeIndex {
        +Remove(scoped_key) IndexEntry
    }
    FileStorage --> KvClientStorageBackend : calls
    KvClientStorageBackend --> KvClient : query and wait
    KvClientStorageBackend --> PayloadSizeIndex : remove stale entry
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

    alt Submit failed or Wait TIMEOUT/TASK_NOT_FOUND
        BE-->>FS: unexpected(mapped_error)
    else no queryResult
        BE-->>FS: unexpected(INTERNAL_ERROR)
    else exists=1 and entryStatus=ENTRY_KEY_EXIST
        BE-->>FS: expected(true)
    else exists=0 and entryStatus=ENTRY_KEY_NOT_EXIST
        BE->>BE: remove stale index entry
        BE-->>FS: expected(false)
    else exists=0 but entryStatus=IO_ERROR/other
        BE-->>FS: unexpected(mapped_error)
    end
```

#### 4.4.3 实现步骤

1. `initialized_ != kReady`（未初始化或初始化中）→ `INTERNAL_ERROR`。
2. 编码 8 字节 `asu_key`。
3. `QueryAsync({key})`；`Submit` 失败直接返回错误（没有 task，不需要 `Wait`）。
4. **只 `Wait` 一次** `Wait(task_id, query_timeout_ms, result)`：
   - `TIMEOUT`/`TASK_NOT_FOUND`/其他错误 → `unexpected(mapped)`，**绝不能**返回 `false`。
   - `result.queryResult` 为空 → `unexpected(INTERNAL_ERROR)`。
5. 同时看 `exists[0]` 与 `entryStatus[0]`：
   - `exists[0] == 1` 且 `entryStatus[0] == ENTRY_KEY_EXIST` → `expected(true)`。
   - `exists[0] == 0` 且 `entryStatus[0] == ENTRY_KEY_NOT_EXIST` → 删除索引条目（若存在）→
     `expected(false)`。
   - `exists[0] == 0` 但 `entryStatus[0]` 是 `IO_ERROR`/`CONNECTION_ERROR`/其他错误 → 
     `unexpected(mapped)`；**不能**当成“不存在”，否则 probe 会误删 Master 元数据。
6. 本版不做懒修复：Query 只返回存在性，没有长度；读路径以 Master 的 `slice.size` 为准。
7. `prefixHitKeys` 只是“连续前缀命中数”，不是总命中数，本后端不使用。

### 4.5 删：仅支持 `Client::Remove` / `Client::BatchRemove`

本版删除只支持两种用户操作：

- `Client::Remove(key, force)`：删除单个对象。
- `Client::BatchRemove(keys, force)`：批量删除对象。

**不支持**：`Client::RemoveByRegex`、`Client::RemoveAll`、`EvictAboveDiskWatermark`、按 tenant/
前缀删除。接口里的 `RemoveAll`/`EvictAboveDiskWatermark` 不重写，沿用基类空实现。

#### 4.5.1 删除类图

```mermaid
classDiagram
    class Client {
        +Remove(key, force) expected
        +BatchRemove(keys, force) vector
    }
    class Master {
        +Remove(key, force) expected
        +BatchRemove(keys, force) vector
        -EnqueueDiskDeleteTask(owner, keys)
    }
    class FileStorage {
        +PollDeleteTasks() vector
        +BatchRemoveKeys(scoped_keys) vector
    }
    class KvClientStorageBackend {
        +BatchRemoveKeys(scoped_keys) vector
        -RemoveKeys(scoped_keys) RemoveOutcome
    }
    class KvClient {
        +DeleteAsync(keys, taskId) Status
        +Wait(taskId, timeoutMs, result) Status
    }
    class PayloadSizeIndex {
        +Remove(scoped_key) IndexEntry
        +Upsert(scoped_key, entry) IndexDelta
    }
    Client --> Master : Remove / BatchRemove
    Master --> FileStorage : per-key delete task (owner)
    FileStorage --> KvClientStorageBackend : BatchRemoveKeys
    KvClientStorageBackend --> KvClient : DeleteAsync + Wait
    KvClientStorageBackend --> PayloadSizeIndex : remove / restore
```

#### 4.5.2 删除时序图

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant C as Client
    participant MS as Master
    participant FS as FileStorage
    participant BE as Backend
    participant KC as KvClient

    U->>C: Remove(key) / BatchRemove(keys)
    C->>MS: Remove / BatchRemove RPC
    MS->>MS: lease / replica / replication task checks
    MS->>MS: EraseMetadata
    MS->>MS: collect LOCAL_DISK replica owners
    MS->>FS: enqueue per-key delete task (owner)
    MS-->>C: per-key result
    C-->>U: result

    FS->>BE: BatchRemoveKeys(removed_scoped_keys)
    BE->>BE: map scoped_key to asu_key, chunk by 254
    BE->>KC: DeleteAsync(keys)
    KC-->>BE: taskId

    alt Submit failed
        BE-->>FS: error (no task)
    else Wait TIMEOUT / TASK_NOT_FOUND
        BE->>KC: Wait(taskId, wait_timeout_ms)
        KC-->>BE: TIMEOUT / task removed
        BE->>BE: keep index (unknown completion)
        BE-->>FS: RPC_TIMEOUT
    else Wait OK / PARTIAL_FAILED
        BE->>KC: Wait(taskId, wait_timeout_ms)
        KC-->>BE: per-entry status
        alt whole CQE = CQE_KEY_NOT_EXIST
            BE->>BE: all keys already absent -> success
        else raw 0
            BE->>BE: remove index, subtract counters
        else ENTRY_DELETE_FAILED/IO_ERROR
            BE->>BE: restore index, log
        end
        BE-->>FS: per-key result
    end
```

#### 4.5.3 实现步骤

1. **Master 侧（元数据先删）**：
   - `Client::Remove`/`BatchRemove` 仍先走 Master RPC；Master 的 lease/副本完整性/复制任务检查
     不变。
   - `EraseMetadata` 成功后，Master 收集该对象 `LOCAL_DISK` 副本的 owner client。
   - 对每个成功删除的 key，向 owner 的本地 SSD 任务邮箱投递一条 **按 key 删除任务**。
2. **owner 侧（物理删除）**：
   - owner 心跳 `PollDeleteTasks` 取出任务，调用 `FileStorage::BatchRemoveKeys`。
   - `FileStorage` 透传给 `StorageBackendInterface::BatchRemoveKeys`。
   - 只删除本 client 命名空间下、自己拥有的 key；不处理其他 client 的 namespace。
3. **backend 侧**：
   - `BatchRemoveKeys` 输入 `scoped_key` 列表；为空返回空结果。
   - 每个 `scoped_key` 映射为 `asu_key`（`kv::CacheKey`）；按 254（`deleteIoNum` 默认值，
     协议硬上限）切块；删除不需要注册 buffer。
   - 每块 `DeleteAsync`；提交成功后才 `Wait` 一次：
     - `TaskResult.status == OK`：entry raw 0 → 删除成功；`ENTRY_DELETE_FAILED` → 失败。
     - 整个 CQE 为 `CQE_KEY_NOT_EXIST`（0x10701）：所有 key 视为“已不存在=成功”。
     - `PARTIAL_FAILED`：逐 key 处理；OK 的 key 视为删除成功，失败的 key 保留索引。
       **注意**：Delete 的 `CQE_CHECK_RESULT_BUFFER` 不归一化为 OK；预发送失败时可能存在
       “OK 但未发送”的 key，删除会留下孤儿数据；Master 元数据已删，残留由后续清理承担。
     - `TIMEOUT`/`TASK_NOT_FOUND`：task 已移除、底层可能仍在跑；**不删索引、不扣计数**，
       返回错误；下一次 delete 幂等重试。
   - 成功：从索引删除，`total_keys_--`、`total_size_ -= transfer_size`。
   - 失败：保留/恢复索引，记录 `first_error`；返回该 key 的错误。
4. **返回**：`BatchRemoveKeys` 返回与输入等长的 `vector<expected<void, ErrorCode>>`；
   `Client::BatchRemove` 的逐 key 结果可直接使用；`Client::Remove` 取单元素。
5. **失败处理**：Master 元数据已删，物理删除失败只记日志；本版没有持久化重试队列，失败对象会
   残留到后续覆盖写或带外清理。这是本版的明确限制。
6. **幂等**：删除不存在的 key 在 CQE 级返回 `CQE_KEY_NOT_EXIST`，backend 归一为成功；没有
   entry 级 `ENTRY_KEY_NOT_EXIST`；`ENTRY_DELETE_FAILED` 才是真正的失败。

#### 4.5.4 接线与 owner 路由（必须）

`Client::Remove` 的调用方不一定是 ASU 数据的 owner；而且 ASU key 包含 per-client
`key_namespace`，其他 client 无法算出同一个 key。因此不能只在 `KvClientStorageBackend` 内部删，
必须补三段接线：

| 位置 | 改动 |
| --- | --- |
| `StorageBackendInterface` | 新增 `BatchRemoveKeys(scoped_keys)`，默认逐 key 返回 `NOT_SUPPORTED` |
| `FileStorage` | 新增 `BatchRemoveKeys`，透传给 backend；提供 `PollDeleteTasks` |
| `local_ssd` manager / Master | 现有 `PollRemoveAll` 只有布尔位；新增按 key 的删除任务邮箱/队列 |
| `Master::Remove` / `BatchRemove` | 元数据删除成功后，收集 `LOCAL_DISK` owner 并投递按 key 删除任务 |
| `Client::Remove` / `BatchRemove` | 仍只发 Master RPC；不直接调用本地 backend |

如果不做 owner 路由，只有“调用方恰好是 owner 且 `key_namespace` 已知”时才能本地删；
多客户端/共享 ASU 场景会漏删或删错 namespace。

#### 4.5.5 Mooncake 删除形式与本后端支持矩阵

##### Mooncake 的删除形式（背景）

| 形式 | 触发方 | 物理删除 | 到 StorageBackend | 说明 |
| --- | --- | --- | --- | --- |
| `Client::Remove(key, force)` | 用户 | 否 | 否 | 只删 Master 元数据和 hot cache；`storage_backend_->RemoveFile(key)` 已被注释 |
| `Client::BatchRemove(keys, force)` | 用户 | 否 | 否 | 逐 key `EraseMetadata` |
| `Client::RemoveByRegex(pattern, force)` | 用户 | 否 | 否 | Master 扫 shard 删元数据；客户端物理 `RemoveByRegex` 已注释 |
| `Client::RemoveAll(force)` | 用户/Master 广播 | 是（视后端） | 是：`RemoveAll()` | Master 置 `PollRemoveAll`，client 心跳调 `FileStorage::RemoveAll` |
| `RemoveAll(force, tenant_id)` | RPC/HTTP | 部分 | 是：`RemoveAll()` | Master 只广播该租户 owner；client 侧仍是租户无关全局清空（已知限制） |
| `EvictDiskReplica` / `BatchEvictDiskReplica` | Master/owner | 否 | 否（只作 handler） | 只删 Master 副本元数据；是淘汰的 master-first 通知 |
| `EvictAboveDiskWatermark` | 心跳 | 是（视后端） | 是 | file/bucket 实现；其他后端默认返回空 |

其中只有 `RemoveAll`（全清广播）和 `EvictAboveDiskWatermark` 会在现有后端里触发物理删除；
`Client::Remove`/`BatchRemove`/`RemoveByRegex` 只删 Master 元数据（物理调用已注释）。本版若要
支持用户按 key 删除，必须新增 `BatchRemoveKeys` 并补 Master→owner 路由（见 §4.5.4）。

##### 本后端的支持矩阵

| Mooncake 形式 | 本后端 | 说明 |
| --- | --- | --- |
| `Client::Remove(key, force)` | **支持** | Master 元数据先删；owner 收到按 key 任务后 `DeleteAsync` |
| `Client::BatchRemove(keys, force)` | **支持** | 逐 key；成功删除的 key 才投递删除任务 |
| `Client::RemoveByRegex(pattern, force)` | 不支持 | 没有 ASU 枚举；本版不接 |
| `Client::RemoveAll(force)` | 不支持 | 没有枚举/drop；接口 `RemoveAll` 继承空实现 |
| `RemoveAll(force, tenant_id)` | 不支持 | 同上，且 ASU 没有 tenant 作用域 |
| `EvictAboveDiskWatermark` | 不支持 | 不重写，基类返回空 |
| `BatchOffload` reactive eviction | 不支持 | 不做写前淘汰 |
| 覆盖写 / `Upsert` | 不显式删除 | 同 key 覆盖；本后端用 `payload_size` 忽略旧尾部 |
| Offload / `complete_handler` 回滚 | 不在本版 | 可后续用 `DeleteAsync` 删本次写入 key |
| `PutRevoke` / 租约 / TTL / 内存淘汰 | 不涉及 | 只删元数据/内存副本 |
| dangling probe | 不涉及 | 物理丢失时删 Master 元数据，不删 ASU |
| ASU 自身 GC | 不可控 | 读/查遇到 `NOT_FOUND` 清理索引 |

**ASU `KvClient` 能力**：

| 能力 | 本后端用法 |
| --- | --- |
| `DeleteAsync({key})` | 按 8B key 删除整个对象（所有 segment/offset） |
| `DeleteAsync(keys)` | 批量；transport 按 `deleteIoNum` 默认 254 切 SQE；backend 仍按 254 切 TaskId |
| 逐 entry `entryStatus` | raw 0=成功、raw 1=`ENTRY_DELETE_FAILED`；没有 entry 级 `ENTRY_KEY_NOT_EXIST` |
| 整体 `CQE_KEY_NOT_EXIST` | 所有 key 共享该状态；视为“已不存在=成功” |
| `PARTIAL_FAILED` | 逐 key 处理；OK key 视为成功，失败 key 保留索引；可能有“OK 但未发送”的残留 |
| 前缀/tenant/namespace 删除 | 不支持 |
| `RemoveAll`/drop/scan | 不支持 |
| CAS/版本/条件删除 | 不支持 |

**NVMeKV 参照**：

| 形式 | NVMeKV |
| --- | --- |
| `RemoveAll` | 不重写，默认 no-op |
| `EvictAboveDiskWatermark` | 不重写，默认空 |
| 用户按 key 删除 | 不暴露；`Delete` 只用于写失败回滚 |
| 批量删除 | 无；`Delete` 是单 16B 物理 key |
| ScanMeta | 空实现，重启不可枚举 |

因此 NVMeKV 不是删除能力的模板；本版按 `Client::Remove`/`BatchRemove` 的按 key 语义单独实现。

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
    BE->>KC: Init(ucm_config_path)
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

- `initialized_` 用 `compare_exchange_strong(kUninit → kInitializing)`：已就绪或正在初始化的
  实例再次 `Init` 都返回 `INTERNAL_ERROR`；全部步骤成功后置 `kReady`。
- 校验 `KvClientBackendConfig::Validate()`；`client_ = kv::CreateKvClient()`；按
  `ucm_config_path` 或 `KvClientConfig` 初始化。**不打开设备、不建 backend 线程池**；
  UCM 内部管理连接/worker/transport 线程。
- 计算 slot size：`align_up(24 + max_payload, 512)`；slot 数 = `staging_bytes / slot_size`，
  至少 1；若小于 `config.batch_size`，写路径按可用 slot 数进一步分子批（有界等待，见 TC-WRITE-04）。
  `StagingArena::Init` 分配对齐内存并 `RegisterRegions`；
  失败时注销已注册的部分区域。
- 计数置 0，索引为空。
- 任一步失败：`client_->Shutdown()`（best-effort；KvClient 的 `Init` 本身不回滚，失败后可能留下
  部分 `viewServer_`/config 状态）、`client_.reset()`、`initialized_.store(InitState::kUninit)`，
  返回对应错误；重试时**新建** `KvClient` 对象，不复用失败实例。
- `staging_.Init` 注册**整块** staging arena 一次；每个 slot 是已注册区域的子区间
  （`KVBuffer.buffer.region.addr/size` 指向 slot，`handle` 用 arena 的 canonical handle）。
- `wait_timeout_ms` 必须大于 UCM transport 的 `timeoutMs`（默认 100ms），或把 transport
  `timeoutMs` 设为 0（无 deadline）；否则 transport 先超时，`Wait` 可能拿到 `TIMEOUT`/entry 错误。
- `IsEnableOffloading` 见 §3.2 的实现契约：`kReady` 后返回 true，未就绪返回错误；
  不读计数、不查 ASU、不发 RPC；配额/容量在 `BatchOffload` 内逐批做。

**Shutdown 前置条件**：

- 调 `KvClient::Shutdown` 前必须确保没有 `InUse`/`Quarantined` slot。KvClient 的 Shutdown 会
  `Drain(defaultWaitTimeoutMs)`；drain 超时后它仍会注销全部注册内存，而在飞 IO 可能仍在引用这些
  内存（`kvclient_deep_read.md` 第 2.6 节明确警告）。
- 如果存在 `Quarantined` slot（Wait 超时导致），无法确认底层 task 是否结束；此时调用 Shutdown
  属于已知风险：只能记录 ERROR、不再复用内存，并接受可能在飞 IO 引用已注销内存的后果。
- 后端析构/关停路径应先检查 slot 状态；正常路径（无超时）先等待所有 task `Wait` 完成再 Shutdown。

#### 4.6.2.1 `Init` 与 `IsEnableOffloading` 的实现边界

这两个方法都必须实现（都是纯虚函数），但“打开设备、建线程池”的 NVMeKV 式实现不能直接照搬：

| 项 | 本版实现 | 优势 | 风险/劣势 |
| --- | --- | --- | --- |
| `Init` | 创建 `kv::KvClient`、加载 view/transport 配置、`KvClient::Init`、注册 staging、初始化索引/计数、CAS 置 `kInitializing`，成功后置 `kReady` | 连接/版本/配置 fail-fast；staging 只注册一次；二次 Init 可拒绝；生命周期清晰 | 启动依赖 UCM/ASU 可用；失败会让 FileStorage Init 失败，需要部署侧重试；没有后台重连 |
| `Init` 中“打开设备/建线程池” | **不做**。ASU 连接、视图路由、task worker、transport 线程由 `kv::KvClient` 内部管理；backend 只持有 staging slot 池 | 不重复 UCM 的线程/连接管理；避免多套线程池和生命周期冲突 | 如果未来需要并行提交多个 chunk 的 `Wait`，再引入一个小的 `ThreadPool`；v1 串行 `Wait` 足够 |
| `IsEnableOffloading` | `kReady` 后直接返回 true；`kUninit`/`kInitializing`/Init 失败返回错误。不读计数、不查 ASU | 便宜、无 RPC；FileStorage Init 能明确决定是否 MountLocalDiskSegment；不会把瞬时状态变成永久关闭 | false/error 都会让本进程不参与 offload；因此不能在方法内做动态配额判断，否则一次瞬时超限会永久关闭 |
| `IsEnableOffloading` 查询 ASU 容量 | **不做**。ASU 公共 API 没有容量/水位查询；容量不足由 `BatchOffload` 的 `RESOURCE_BUSY`/`CQE_OUT_OF_CREATE_SIZE` 逐 key 反馈 | 不引入额外 RPC/依赖；不把瞬时容量当成永久状态 | 无法提前阻止写满；依赖逐 key 跳过和日志 |

关键结论：

- `IsEnableOffloading` 在 `FileStorage::Init` 里被调用一次，用来设置 `enable_offloading_` 并
  `MountLocalDiskSegment(enable_offloading_)`；它不是每次心跳重新求值。**`kReady` 后直接返回 true；
  未就绪返回错误让 Init 明确失败**，不要用计数/水位/ASU 瞬时容量做动态判断。
- **容量与配额控制只在 `BatchOffload` 内逐批/逐 key 做**：超限 key 跳过，返回
  `expected(success_count)`/`expected(0)`，下一轮心跳重试；不返回 `KEYS_ULTRA_LIMIT`。
- `Init` 不要重复创建 UCM 已经管理的连接/worker/transport 线程；backend 需要的是 staging slot
  池，不是线程池。

#### 4.6.3 `ScanMeta` / `ResetScanIterator` / 恢复限制

- `ScanMeta`：返回 `expected<void>` OK，不调用 handler，不枚举 ASU；记一次 WARNING。
- `ResetScanIterator`：空实现。
- `RemoveAll`/`EvictAboveDiskWatermark` 本版不支持，继承基类空实现；按 key 删除见 §4.5。
- 结果：进程重启后，Master 若仍有 `LOCAL_DISK` 元数据，读/查仍可用；若 Master 也丢失元数据，
  对象不会自动重新注册。这是本版的明确限制。

#### 4.6.4 现有五套 `ScanMeta` 实现对照

五套实现（`StorageBackendAdaptor` / `BucketStorageBackend` / `OffsetAllocatorStorageBackend` /
`DistributedStorageBackend` / `NvmeKvStorageBackend`）的对照表、逐后端要点与逐行代码注解，已移到
《FileStorage-offload-实现解读.md》的 6.9 节与附录 C——那里是它们真正的归属地。

本版只需记住结论：**只有 file-per-key、bucket、对象存储三种能在 Master 丢元数据后完整重建账本；
OffsetAllocator 取决于 `persist_mode`（默认不恢复）；NVMe KV 完全不恢复。**

### 4.7 并发与一致性（写/读/查/删）

| 场景 | 结果 | 处理 |
| --- | --- | --- |
| Store 成功、handler 失败 | 数据在 ASU、本进程索引有记录、Master 不知道 | 不恢复；重试 Put 覆盖 |
| Store 成功、Wait 超时 | task 被移除、底层可能仍在飞 | slot 进入 `Quarantined`；不通知 Master；孤儿对象；重试覆盖 |
| 覆盖写 | 单 key 一条 ASU value 原子替换；索引按 delta 更新 | last-writer-wins |
| 读与写并发 | 读到旧值或新值 | 二者都是完整对象；CRC 保证完整性 |
| 读时索引缺失 | 仍可读 | 以 Master 的 `slice.size` 为准；成功后补索引 |
| ASU 自身 GC/删除 | 索引暂时陈旧 | 读/查遇到 `NOT_FOUND` 时删索引 |
| 删除与并发读 | 读到旧值或 `OBJECT_NOT_FOUND` | 不允许半对象；CRC 保证完整性 |
| Master 已删元数据、owner 未收到删除任务 | ASU 残留对象 | 本版无持久化任务队列；后续覆盖/带外清理 |
| `DeleteAsync` 失败 | 索引恢复、计数不变 | `ENTRY_DELETE_FAILED`/IO 错误保留索引；本版只记日志 |
| 重复删除 | 整体 CQE 为 `CQE_KEY_NOT_EXIST` | 归一为成功；无 entry 级 KEY_NOT_EXIST |
| 进程重启 | 索引为空 | 读/查不依赖索引；写重新累计计数 |
| 二次 `Init` | `INTERNAL_ERROR` | 与现有共享契约一致 |
| Submit 返回 `RESOURCE_BUSY` | 队列/transport 满，没有 task | 立即释放 slot；本批跳过，下一轮心跳重试 |
| `PARTIAL_FAILED` | 可能有“OK 但未发送”的 entry | 写：本块不发布；读：本块不返回数据；删：逐 key 处理 |
| Query 分片失败、`exists=0` | 分片未合并，不能当“不存在” | 检查 `entryStatus`；非 `ENTRY_KEY_NOT_EXIST` 返回 `unexpected` |
| 视图刷新/transport retire | in-flight task 可能失败；不重发 | 映射为 `CONNECTION_ERROR`；交给下一轮 offload/调用方 |
| Shutdown 时有隔离 slot | KvClient drain 超时后仍注销内存 | 记录 ERROR；正常路径先等所有 task 完成；风险文档化 |
| 多客户端共享 ASU | key 哈希包含 `key_namespace` | 逻辑隔离；`key_namespace` 唯一性由部署保证 |

### 4.8 可观测性

- 复用 `SsdMetric`：`FileStorage` 已记录 read/write ops、bytes、latency，本版不新增指标文件。
- 结构化日志字段：`backend=kv_client_storage_backend`、`key_namespace`、`op`、`keys`、`entries`、`success`、
  `skipped`、`failed`、`first_error`、`task_id`、`task_status`、`entry_status`、`quarantined_slots`。
- 建议的后端计数器（本版只记日志）：提交 task 数、`RESOURCE_BUSY` 次数、Wait 超时数、`PARTIAL_FAILED`
  次数、`Quarantined` slot 数、逐 entry 失败数、staging 等待时间。
- `Quarantined` slot 数持续增长是明确告警：说明 Wait 超时/任务丢失在耗尽 staging。

## 5. SFMEA 分析与测试设计

### 5.1 分析方法

- **范围**：写（`BatchOffload`）、读（`BatchLoad`）、查（`IsExist`），以及 Init、staging、配置、
  错误映射、并发和按 key 删除。水位淘汰/RemoveAll 不在本版，因此不作为本版测试项。
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
  `ReferenceKvClient`（CI），以及 UCM fake transport provider（`BUILD_KV_CLIENT_PROVIDER_FAKE`，
  可选集成，不依赖真实 ASU 硬件）。
- **必须覆盖的 KvClient 特有语义**：破坏性 `Wait`（超时移除 task）、`PARTIAL_FAILED` 下
  “OK 但未发送”、Query 分片失败（`exists=0` + `entryStatus=IO_ERROR`）、Delete 整体
  `CQE_KEY_NOT_EXIST`、Submit `RESOURCE_BUSY`、`Quarantined` slot 不复用。

### 5.2 失效模式与影响（FMEA）

| ID | 功能 | 失效模式 | 局部影响 | 系统影响 | 严重度 | 检测/缓解 | 测试 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| FM-INIT-01 | Init | `KvClient::Init` 或握手失败 | 后端 Init 返回错误，无 staging | FileStorage Init 失败，节点不提供 offload | S2 | 错误码/日志；不进入可用状态 | TC-INIT-01 |
| FM-INIT-02 | Init | `RegisterRegions` 失败 | 无可用 staging slot | 写/读全部不可用 | S2 | 返回错误；不保留部分句柄 | TC-INIT-02 |
| FM-INIT-03 | Init | 二次 Init | 计数/索引被重置或重复注册 | 状态不一致、重复 task | S2 | CAS 保护；第二次返回 `INTERNAL_ERROR` | TC-INIT-03 |
| FM-INIT-04 | Init | `key_namespace` 为空或配置非法 | 使用无效 key 启动 | 写入错误对象或全部失败 | S2 | `Validate` 失败；`INVALID_PARAMS` | TC-INIT-04 |
| FM-INIT-05 | Init | 失败后未 `Shutdown`/未复位 `initialized_` | client/内存泄漏；无法重试 | 节点只能重启 | S2 | 失败路径统一 Shutdown + reset + 复位；单测覆盖 | TC-INIT-05 |
| FM-WRITE-02 | 写 | `RecordHeader` 编码错误 | 读端拒收合法数据 | offload 数据不可读 | S2 | `RecordHeader` 单测；头 CRC | TC-WRITE-02 |
| FM-WRITE-03 | 写 | payload 超 slot/单值上限 | 该 key 跳过 | offload 不完整，Master 仍持有内存副本 | S3 | 跳过集合；日志；返回成功数 | TC-WRITE-03 |
| FM-WRITE-04 | 写 | staging 耗尽 | 批次阻塞或跳过 | 心跳延迟或丢批 | S3 | 有界等待/跳过；slot 计数 | TC-WRITE-04 |
| FM-WRITE-05 | 写 | `BatchStoreAsync` 提交失败 | 无 entry 写入 | 本批失败 | S2 | 错误映射；不更新索引 | TC-WRITE-05 |
| FM-WRITE-06 | 写 | `Wait` 超时 | task 已被移除、底层可能仍在飞 | 不通知 Master；孤儿对象 | S2 | slot 进入 `Quarantined`，不得复用/释放；返回 `RPC_TIMEOUT` | TC-WRITE-06 |
| FM-WRITE-07 | 写 | `PARTIAL_FAILED` 下发布 OK entry | 预发送失败时 OK 但未发送的 entry 被当成功 | Master 指向不存在的数据 | S1 | 本块整体不发布；释放 slot；日志 | TC-WRITE-07 |
| FM-WRITE-08 | 写 | 逐 entry 失败未跳过 | 整批返回错误 | 心跳失败或 offload 停用 | S2 | 顶层 OK 时只上报成功 key；顶层 PARTIAL 时本块不发布；0 成功返回 `expected(0)` | TC-WRITE-07 |
| FM-WRITE-09 | 写 | `complete_handler` 失败 | 数据已写、Master 不知 | 悬挂副本；重启不可见 | S2 | 返回错误；不回滚；日志 | TC-WRITE-08 |
| FM-WRITE-10 | 写 | 覆盖写计数 delta 错误 | 配额/水位漂移 | 过早停写或超配额 | S3 | 原子更新 index+counters | TC-WRITE-09 |
| FM-WRITE-11 | 写 | 空间不足返回 `KEYS_ULTRA_LIMIT` | FileStorage 永久关闭 offload | 节点不再 offload | S2 | 禁止该码；返回 success count | TC-WRITE-03 |
| FM-DEL-01 | 删除 | Master 删元数据成功但 owner 未收到删除任务 | 元数据已删、ASU 残留 | 对象不可路由但占容量 | S2 | 任务队列/日志；带外清理 | TC-DEL-01 |
| FM-DEL-02 | 删除 | owner 路由错误 / `key_namespace` 不匹配 | 删不到或删错对象 | 数据丢失或残留 | S1 | Master 记录 owner；按 `scoped_key` 校验 `key_namespace`；审计 | TC-DEL-02 |
| FM-DEL-03 | 删除 | `DeleteAsync` 逐 entry 失败 | 该 key 索引保留/恢复 | 残留对象；计数不扣 | S3 | 逐 entry 状态；恢复索引；日志 | TC-DEL-03 |
| FM-DEL-04 | 删除 | 调用方非 owner 直接本地删 | 漏删 | ASU 残留 | S2 | 必须 Master→owner；caller 不直接删 | TC-DEL-02 |
| FM-DEL-05 | 删除 | 进程重启后在途任务丢失 | 删除任务丢失 | ASU 残留 | S2 | 本版接受；文档化；后续持久化队列 | TC-DEL-04 |
| FM-DEL-06 | 删除 | 删除与并发读 | 读到旧值或 `OBJECT_NOT_FOUND` | 不允许半对象 | S1 | 单 key 原子 + CRC；读到旧值可接受 | TC-DEL-06 |
| FM-DEL-07 | 删除 | 删除后计数 delta 错误 | 配额漂移 | 过早停写或超配额 | S3 | 成功/缺失才扣减；失败恢复 | TC-DEL-07 |
| FM-DEL-08 | 删除 | 重复删除 | 整体 `CQE_KEY_NOT_EXIST` 被当失败 | 误报错误 | S3 | `CQE_KEY_NOT_EXIST` 视为成功；无 entry 级 KEY_NOT_EXIST | TC-DEL-08 |
| FM-DEL-09 | 删除 | 单次 Delete 超过 254 | 客户端内部再切子批，但 TaskId 粒度过大/Wait 超时风险 | 部分 key 未删 | S3 | 后端按 254 切块 | TC-DEL-09 |
| FM-DEL-10 | 删除 | `PARTIAL_FAILED` 下把 OK key 视为已删 | 预发送失败时 key 实际未删 | ASU 残留，容量不释放 | S3 | 逐 key 处理；OK 视为成功、失败保留索引；残留由后续清理承担 | TC-DEL-10 |
| FM-READ-01 | 读 | `slice.size` 与头 `payload_size` 不一致 | 返回 `INVALID_PARAMS` | 该对象读失败 | S2 | 交叉校验；不拷贝 | TC-READ-01 |
| FM-READ-02 | 读 | ASU `NOT_FOUND` | 返回 `OBJECT_NOT_FOUND` | Master 路由悬挂 | S2 | 清索引；probe 修复 | TC-READ-02 |
| FM-READ-03 | 读 | magic/version/flags 损坏 | `INVALID_READ` | 不返回错误数据 | S1 | 头校验 | TC-READ-03 |
| FM-READ-04 | 读 | `RecordHeader` CRC 损坏 | `INVALID_READ` | `RecordHeader` 不可信 | S1 | CRC-32C | TC-READ-03 |
| FM-READ-06 | 读 | payload CRC 损坏 | `CHECKSUM_MISMATCH` | 不返回撕裂数据 | S1 | CRC-32C | TC-READ-05 |
| FM-READ-07 | 读 | 读回长度不足/超出 | `INVALID_READ`/`INVALID_PARAMS` | 不越界拷贝 | S1 | 读回长度校验 | TC-READ-06 |
| FM-READ-08 | 读 | 索引缺失但对象存在 | 若强制查索引则读失败 | 重启后读不可用 | S2 | 以 `slice.size` 为准；成功后补索引 | TC-READ-07 |
| FM-READ-09 | 读 | 与覆盖写并发 | 读到旧值或新值 | 不允许半对象 | S1 | 单 key 原子 + CRC | TC-READ-08 |
| FM-READ-10 | 读 | `PARTIAL_FAILED` 下把 OK entry 当已读 | 预发送失败时目标 buffer 未被写入 | 调用方拿到未初始化数据 | S1 | 本块不向调用方返回数据；返回错误；Release slot | TC-READ-09 |
| FM-QUERY-01 | 查 | 超时/传输错误返回 `false` | probe 误删 Master 元数据 | 数据不可路由 | S1 | 必须 `unexpected`，不得 `false` | TC-QUERY-01 |
| FM-QUERY-02 | 查 | 索引陈旧/假阳性 | 读时 `NOT_FOUND` | 读失败 | S2 | 以 Query 为准；读时清索引 | TC-QUERY-02 |
| FM-QUERY-03 | 查 | 编码 key 与写不一致 | 永远查不到 | 对象被视为不存在 | S1/S2 | 同一 codec；单测 | TC-QUERY-03 |
| FM-QUERY-04 | 查 | Query 分片失败时 `exists=0` 被当不存在 | 分片未合并，`exists` 保持 0 | probe 误删 Master 元数据 | S1 | 同时检查 `entryStatus`；非 `ENTRY_KEY_NOT_EXIST` 返回 `unexpected` | TC-QUERY-04 |
| FM-CONC-01 | 并发 | `Wait` 期间持有 `metadata_mutex_` | 读写互相阻塞/死锁 | 心跳超时 | S2 | 锁纪律；`Wait` 外置 | TC-CONC-01 |
| FM-CONC-02 | 并发 | task 未完成就复用 slot | 数据被覆盖 | 写坏对象 | S1 | 正常完成 Release；超时/任务丢失 Quarantined，不复用 | TC-CONC-02 |
| FM-LIFE-01 | 关停 | 有在途 task 就 `Unregister`/`Shutdown` | 崩溃/数据损坏 | 进程异常 | S1 | 先 drain 再注销 | TC-LIFE-01 |
| FM-LIFE-02 | 生命周期 | 二次 Init/重新 Init | 状态/句柄错乱 | 行为不可预测 | S2 | CAS/禁止重入 | TC-INIT-03 |
| FM-LIFE-03 | 关停 | 存在 `Quarantined` slot 时 Shutdown | KvClient drain 超时后仍注销内存 | 在飞 IO 引用已注销内存，崩溃/损坏 | S1 | 正常路径先等所有 task 完成；隔离场景记录 ERROR，文档化风险 | TC-LIFE-02 |
| FM-SUBMIT-01 | 提交 | Submit 返回 `RESOURCE_BUSY`/`NOT_INITIALIZED` | 没有 task，若未释放 slot | staging 泄漏/心跳阻塞 | S2 | 立即释放 slot，不 Wait；下一轮重试 | TC-SUBMIT-01 |
| FM-STUB-01 | ScanMeta | 返回空 | Master 重启后不注册 | 冷数据不可达 | S2（本版接受） | 文档/日志 | TC-STUB-01 |
| FM-STUB-02 | RemoveAll | 本版不支持，继承空实现 | 元数据可能已删 | ASU 数据残留；容量不释放 | S2（本版接受） | 文档/日志 | TC-DEL-05 |
| FM-STUB-03 | 容量 | ASU 容量满/资源不足 | 写入失败 | offload 停止 | S2 | 映射 `RESOURCE_BUSY`/`OUT_OF_CREATE_SIZE`；跳过 key | TC-STUB-02 |
| FM-CAP-01 | 配额 | 重启后计数归零 | 配额失真 | 超量写或提前停写 | S3 | 计数仅参考；ASU 容量为准 | TC-CAP-01 |
| FM-CAP-02 | IsEnableOffloading | 误把配额/容量做成返回值 | 本进程永久停 offload | 节点不再下沉 | S2 | `kReady` 后恒 true；未就绪返回错误；配额只在 `BatchOffload` 逐 key 跳过并返回 `expected(success_count)`/`expected(0)` | TC-CAP-02 |
| FM-CFG-01 | 工厂 | UCM 未编译但选了 `kKvClient` | 创建失败 | 节点启动失败 | S2 | 编译开关；清晰错误 | TC-CFG-01 |
| FM-CFG-02 | 工厂 | 生产误配 `client_impl=kReference`（参考实现，数据不落 ASU） | 数据未落 ASU | 数据丢失 | S1 | `Validate`/启动健康检查告警；部署校验 | TC-CFG-02 |
| FM-OBS-01 | 可观测性 | 无 task/entry 日志 | 无法定位 | 诊断困难 | S4 | 结构化日志 | TC-OBS-01 |

### 5.3 测试用例与通过标准

| TC | 覆盖失效模式 | 注入/前置 | 步骤 | 通过标准 |
| --- | --- | --- | --- | --- |
| TC-INIT-01 | FM-INIT-01 | `FakeKvClient::Init` 返回错误 | 调 `KvClientStorageBackend::Init` | 返回对应错误；无 staging 注册；无崩溃；FileStorage Init 失败 |
| TC-INIT-02 | FM-INIT-02 | `RegisterRegions` 返回失败 | 调 Init | 返回错误；无部分句柄泄漏；修复后重试成功 |
| TC-INIT-03 | FM-INIT-03 | 连续两次 Init | 两次调用 | 第一次 OK；第二次 `INTERNAL_ERROR`；索引/计数不变 |
| TC-INIT-04 | FM-INIT-04 | `key_namespace` 为空/配置非法 | 调 Validate/Init | Validate false；Init `INVALID_PARAMS`；不调用 KvClient |
| TC-INIT-05 | FM-INIT-05 | `RegisterRegions` 失败 | 调 Init，再重试 | 首次失败后 `initialized_ == kUninit`、client 已 Shutdown；第二次 Init 成功 |
| TC-WRITE-02 | FM-WRITE-02 | 固定 payload/`RecordHeader` 向量 | 编码后再解码 | 字段/CRC 与向量一致；篡改任一字段解码失败 |
| TC-WRITE-03 | FM-WRITE-03/11 | payload 超 slot、超配额、空 key | 批量写 3 个 key | 超限 key 跳过；其余成功；handler 只收到成功 key；不返回 `KEYS_ULTRA_LIMIT` |
| TC-WRITE-04 | FM-WRITE-04 | staging slot=1，延长 Wait | 并发两个写 | 有界等待；无死锁；slot 全部归还；成功数正确 |
| TC-WRITE-05 | FM-WRITE-05 | `BatchStoreAsync` 返回错误 | 调 BatchOffload | 无 handler 调用；返回错误；索引/计数不变 |
| TC-WRITE-06 | FM-WRITE-06 | `Wait` 延迟超过 timeout | 调 BatchOffload | 返回 `RPC_TIMEOUT`；无 handler；slot 为 `Quarantined` 且不被复用；后续批次使用剩余 slot |
| TC-WRITE-07 | FM-WRITE-07/08 | 3 个 key，顶层 `PARTIAL_FAILED` | 调 BatchOffload | 本块发布 0 个 key；handler 不被调用；slot 全部 Release；日志含逐 entry 错误 |
| TC-WRITE-08 | FM-WRITE-09 | `complete_handler` 返回错误 | 调 BatchOffload | 数据仍可读；返回该错误；不回滚；日志记录 |
| TC-WRITE-09 | FM-WRITE-10 | 同 key 先写 4K 再写 8K | 两次写 | 读到 8K 新值；`total_size_` 按 delta 更新；`total_keys_` 不变 |
| TC-DEL-01 | FM-DEL-01 | Master Remove 成功但删除任务未投递 | 模拟 owner 不收任务 | 元数据已删；ASU 残留；日志可追踪；不崩溃 |
| TC-DEL-02 | FM-DEL-02/04 | 两个 client，A 删 B 拥有的 key | 调 Client::Remove | Master 通知 B；B 的 backend 删除；A 不直接删；结果正确 |
| TC-DEL-03 | FM-DEL-03 | 3 个 key，entry 2 `DeleteAsync` 失败 | 调 BatchRemoveKeys | 返回 2 成功/1 失败；失败 key 索引保留；计数只扣 2 |
| TC-DEL-04 | FM-DEL-05 | 删除任务入队后重启 owner | 重启 + PollDeleteTasks | 在途任务丢失；ASU 残留；行为与文档一致；不崩溃 |
| TC-DEL-05 | FM-STUB-02 | 调用 `RemoveAll`/`RemoveByRegex` | 观察 ASU 数据与日志 | 元数据删除；ASU 数据不删；日志说明；不崩溃 |
| TC-DEL-06 | FM-DEL-06 | 读与删除并发 | 多线程 | 读到旧值或 `OBJECT_NOT_FOUND`；不出现半对象；CRC 通过 |
| TC-DEL-07 | FM-DEL-07 | 覆盖写后再删除 | 写两次、删一次 | `total_keys_`/`total_size_` 归零；无重复扣减 |
| TC-DEL-08 | FM-DEL-08 | 删除不存在的 key | 调 BatchRemoveKeys | 整体 `CQE_KEY_NOT_EXIST` 归一成功；计数不变 |
| TC-DEL-09 | FM-DEL-09 | 300 个 key | 调 BatchRemoveKeys | 分成 2 块（254+46）；逐 key 结果；TaskId 数=2 |
| TC-DEL-10 | FM-DEL-10 | fake 返回 PARTIAL_FAILED（部分 OK 未发送） | 调 BatchRemoveKeys | OK key 从索引删除；失败 key 保留索引；残留日志可见 |
| TC-READ-01 | FM-READ-01 | 写入 4096B，读时 `slice.size=4097` | 调 BatchLoad | 返回 `INVALID_PARAMS`；`slice.ptr` 内容不变 |
| TC-READ-02 | FM-READ-02 | ASU 无该 key | 调 BatchLoad | 返回 `OBJECT_NOT_FOUND`；索引条目被清理；无崩溃 |
| TC-READ-03 | FM-READ-03/04 | 篡改 magic/version/`RecordHeader` CRC | 调 BatchLoad | 返回 `INVALID_READ`；不拷贝 payload；缓冲区不变 |
| TC-READ-05 | FM-READ-06 | 篡改 payload 一个字节 | 调 BatchLoad | 返回 `CHECKSUM_MISMATCH`；不拷贝 payload |
| TC-READ-06 | FM-READ-07 | fake 返回短读/超长 | 调 BatchLoad | 返回 `INVALID_READ`/`INVALID_PARAMS`；ASAN 无越界 |
| TC-READ-09 | FM-READ-10 | fake 返回 PARTIAL_FAILED（部分 OK 未写入） | 调 BatchLoad | 返回错误；不把未写入的 buffer 当成功；slot Release |
| TC-READ-07 | FM-READ-08 | 清空索引，Master 提供 `slice.size` | 调 BatchLoad | 读成功；字节一致；索引被补回 |
| TC-READ-08 | FM-READ-09 | 读与覆盖写并发 | 多线程读写 | 读到旧值或新值；两者 CRC 均通过；无半对象 |
| TC-QUERY-01 | FM-QUERY-01 | Query 超时/传输错误 | 调 IsExist | 返回 `unexpected`（`RPC_TIMEOUT`/`UNAVAILABLE`）；绝不返回 `false`；Master 元数据不变 |
| TC-QUERY-02 | FM-QUERY-02 | 索引有陈旧条目，ASU 无数据 | 调 IsExist | 返回 `false`；索引条目删除；其他 key 不受影响 |
| TC-QUERY-03 | FM-QUERY-03 | 同一 `scoped_key` 走写/读/查 | 比较编码结果 | 三次得到同一 8B key；无 mismatch |
| TC-QUERY-04 | FM-QUERY-04 | fake 返回 exists=0 + entryStatus=IO_ERROR | 调 IsExist | 返回 `unexpected`；索引条目保留；Master 元数据不变 |
| TC-CONC-01 | FM-CONC-01 | 阻塞 Wait + 并发写 | 两线程 | 写不被读的 Wait 阻塞（或仅在索引提交短暂等待）；TSAN 无数据竞争；无死锁 |
| TC-CONC-02 | FM-CONC-02 | 1 个 slot + 多读多写 | 并发压测 | ASAN/TSAN 干净；无 use-after-free；slot 全部归还 |
| TC-LIFE-01 | FM-LIFE-01 | 有在途 task 时 Shutdown | 调 Shutdown/析构 | 等待 task 完成；不提前 `UnregisterRegions`；无崩溃/泄漏 |
| TC-LIFE-02 | FM-LIFE-03 | 存在 Quarantined slot 时 Shutdown | 调 Shutdown | 记录 ERROR；不崩溃（允许已知风险）；隔离 slot 不被复用 |
| TC-SUBMIT-01 | FM-SUBMIT-01 | fake 让 Submit 返回 RESOURCE_BUSY | 调 BatchOffload/BatchLoad | 无 Wait；slot 立即 Release；返回错误；无泄漏 |
| TC-STUB-01 | FM-STUB-01 | 正常 backend | 调 ScanMeta | 返回 OK；handler 未被调用；不注册任何对象 |
| TC-STUB-02 | FM-STUB-03 | 模拟 ASU 容量不足 | 调写 | 该 key 跳过/可重试；不崩溃；其他 key 正常 |
| TC-CFG-01 | FM-CFG-01 | 未编译 UCM，选 `kKvClient` | 调 CreateStorageBackend | 返回清晰错误（`INVALID_PARAMS`/`NOT_SUPPORTED`）；无崩溃 |
| TC-CFG-02 | FM-CFG-02 | 生产配置使用 `client_impl=kReference` | 启动/健康检查 | 配置校验拒绝或显著告警；不静默写成功 |
| TC-CAP-01 | FM-CAP-01 | 重启 backend | 读计数并写新 key | 计数从 0 开始；写仍可用；ASU 容量错误被正确映射 |
| TC-CAP-02 | FM-CAP-02 | 计数超过 hard limit | 调 IsEnableOffloading / BatchOffload | `IsEnableOffloading` 仍为 true 且不读计数；`BatchOffload` 跳过超限 key，返回 `expected(success_count)`/`expected(0)`，不返回 `KEYS_ULTRA_LIMIT`；未调用 FakeKvClient 的 Query |
| TC-OBS-01 | FM-OBS-01 | 注入一次 entry 失败 | 检查日志 | 日志含 `task_id`、entry 下标/状态、首错、key；可定位到具体 entry |
| TC-PERF-01 | 性能/批量 | N=100，batch=32 | 调 BatchOffload/BatchLoad | TaskId 数 ≤ `ceil(100/32)=4`；p99 在基线内；无 per-key task |

### 5.4 故障注入能力

`FakeKvClient : public kv::KvClient` 提供：

- `FailNext(op, key, StatusCode)`：对下一次 Query/Store/Load/Delete 注入错误。
- `SubmitBusy()`：下一次 `*Async` 返回 `RESOURCE_BUSY`，不创建 task。
- `DelayNext(op, ms)` / `WaitTimeoutDestructive()`：`Wait` 返回 `TIMEOUT` 并**移除** task；
  随后 `Check` 返回 true，模拟 KvClient 的破坏性 Wait。
- `PartialFailure(indices)`：让指定 entry 在 `entryStatus` 中失败，聚合状态为 `PARTIAL_FAILED`。
- `PartialFailureWithUnsentOk(indices)`：顶层 `PARTIAL_FAILED`，但部分 `entryStatus` 仍为 OK，
  且实际未写入/未读取目标内存；模拟“预发送失败导致 OK 但未发送”的坑。
- `QueryShardFailure(key)`：`exists=0` 且 `entryStatus=IO_ERROR`，模拟 Query 分片失败。
- `DeleteWholeCqe(StatusCode::CQE_KEY_NOT_EXIST)`：整个 Delete CQE 返回 KEY_NOT_EXIST。
- `FailDelete(key, StatusCode)`：删除 entry 失败，用于验证索引恢复。
- `CorruptValue(key, field)`：篡改 magic/version/`RecordHeader` CRC/payload CRC。
- `RejectUnregisteredBuffer()`：模拟 `BUFFER_NOT_REGISTERED`。
- `RetireTransport()`：模拟视图刷新/transport retire 后 in-flight task 失败。
- `RecordCalls()`：记录提交/等待顺序和 task 数。

### 5.5 覆盖结论与残余风险

- S1/S2 失效模式都有对应测试；S3 覆盖配额、staging、覆盖写；S4 只做日志断言。
- 残余风险：
  - 删除依赖 Master→owner 的按 key 任务；任务在途丢失/owner 重启后本版没有持久化重试队列。
  - `RemoveAll`/`RemoveByRegex`/水位淘汰不支持；这些操作只删元数据，ASU 数据残留。
  - 真实 ASU 对“删除不存在的 key”的返回码需实测；本版按整体 `CQE_KEY_NOT_EXIST` 归一成功（没有 entry 级 KEY_NOT_EXIST）。
  - owner 路由/`key_namespace` 配置错误会导致漏删或删错，需要审计与配置校验。
  - `asu_key` 只有 64 bit；本版不做碰撞检测，假设不发生小概率碰撞。若发生，后写对象会覆盖先写对象，
    读先写 key 可能返回后写对象的数据（该 value 自身 CRC 仍可能通过）。这是本版明确接受的假设。
  - `Wait` 超时后 KvClient 已移除 task，底层任务可能仍在飞；本版只能把 slot 永久隔离，反复超时会
    耗尽 staging 容量。没有取消/回收手段。
  - `PARTIAL_FAILED` 时可能存在“OK 但未发送”的 entry；本版对写/读采用“本块不发布/不返回”的保守
    策略，会牺牲部分真实成功，换取不产生假成功。
  - 存在 `Quarantined` slot 时调用 `KvClient::Shutdown`，KvClient 仍可能在 drain 超时后注销内存；
    这是已知的生命周期风险，正常路径必须先等所有 task 完成。
  - KvClient 不自动重试；`ENTRY_RETRY_ADVISED` 只记录，不做自动重发。


## 6. Mooncake 改动量

### 6.1 文件级清单

| # | 文件 | 改动 | 行数（本版） | 优先级 |
| --- | --- | --- | --- | --- |
| 1 | `mooncake-store/include/storage_backend.h` | `StorageBackendType::kKvClient`；`BatchRemoveKeys` 虚函数 | +10~20 | 必须 |
| 2 | `mooncake-store/src/storage_backend.cpp` | 工厂 `case kKvClient`；include | +15~25 | 必须 |
| 3 | `mooncake-store/src/config/file_storage_config.cpp` | descriptor `kv_client_storage_backend` 的解析分支；`Validate()` 对 `kKvClient` 跳过本地路径 | +25~40 | 必须 |
| 4 | `mooncake-common/include/environment_variables.h` | `MOONCAKE_OFFLOAD_KV_CLIENT_*` 9 个变量 | +10~18 | 必须 |
| 5 | `mooncake-store/include/config/kv_client_backend_config.h` | 配置结构 | +50~70 | 必须 |
| 6 | `mooncake-store/src/config/kv_client_backend_config.cpp` | 环境变量解析 + `Validate` | +100~140 | 必须 |
| 7 | `mooncake-store/include/kv_client/backend.h` | 后端声明；前置声明 `kv::KvClient`；`BatchRemoveKeys` | +140~190 | 必须 |
| 8 | `mooncake-store/src/kv_client/backend.cpp` | 写/读/查/按 key 删、slot 状态/隔离、task 编排、UCM 类型/错误映射 | +1,300~1,800 | 必须 |
| 9 | `mooncake-store/include/kv_client/record.h` + `mooncake-store/src/kv_client/record.cpp` | `KvClientRecordCodec`：`RecordHeader`（24B）、CRC | +100~160 | 必须 |
| 10 | `mooncake-store/include/kv_client/key_codec.h` + `mooncake-store/src/kv_client/key_codec.cpp` | `KvClientKeyCodec`：`scoped_key` → 8B `asu_key` | +50~80 | 必须 |
| 11 | `mooncake-store/include/kv_client/staging.h` + `mooncake-store/src/kv_client/staging.cpp` | `StagingArena`：整块注册、子区间 slot、Free/InUse/Quarantined | +240~360 | 必须 |
| 12 | `mooncake-store/include/kv_client/factory.h` + `mooncake-store/src/kv_client/factory.cpp`；`mooncake-store/include/kv_client/reference_kv_client.h` + `mooncake-store/src/kv_client/reference_kv_client.cpp`（`ReferenceKvClient`） | 工厂 + 参考实现（各自独立文件） | +200~300 | 必须 |
| 13 | `mooncake-store/src/CMakeLists.txt` | 源文件 + UCM include/link；`USE_UCM` | +20~40 | 必须 |
| 14 | `dependencies.sh` / `third_party` | UCM `kv_client`/`kv_transport` 引入方式 | +20~60 | 必须 |
| 15 | `mooncake-store/include/file_storage.h` + `src/file_storage.cpp` | `BatchRemoveKeys` 转发 + `PollDeleteTasks` | +40~80 | 必须（接线） |
| 16 | `mooncake-store/include/local_ssd/manager.h` + `src/local_ssd/manager.cpp` | 按 key 删除任务邮箱/队列（现有只有 RemoveAll 布尔位） | +120~220 | 必须（接线） |
| 17 | `mooncake-store/src/master_service.cpp` + 头文件 | `Remove`/`BatchRemove` 成功后收集 `LOCAL_DISK` owner 并投递删除任务 | +80~160 | 必须（接线） |
| 18 | `mooncake-store/src/client_service.cpp` | `Remove`/`BatchRemove` 不再尝试本地物理删除；日志/结果映射 | +0~20 | 必须（接线） |
| 19 | `mooncake-store/tests/CMakeLists.txt` | `add_store_test` | +4~6 | 必须 |
| 20 | `mooncake-store/tests/storage_backend_test_utils.h` + `storage_backend_test.cpp` | 抽取共享 helper | +80~140 / +10~20 | 必须 |
| 21 | `mooncake-store/tests/kv_client/backend_test.cpp` | 写/读/查/删契约、Wait 超时/隔离、PARTIAL、Query 分片、删除状态、并发、owner 路由 | +1,000~1,500 | 必须 |
| 22 | `mooncake-store/tests/kv_client/record_test.cpp` | `RecordHeader`/`Record` 测试 | +140~200 | 必须 |
| 23 | `mooncake-store/tests/kv_client/key_codec_test.cpp` | key 编码测试 | +50~80 | 必须 |
| 24 | `mooncake-store/tests/kv_client/staging_test.cpp` | 对齐/子区间/Free-InUse-Quarantined 迁移测试 | +140~220 | 必须 |
| 25 | `mooncake-store/tests/kv_client/fake_kv_client.h` | `FakeKvClient : public kv::KvClient` | +200~300 | 必须 |
| 26 | `mooncake-store/tests/kv_client/kv_client_backend_config_test.cpp` | 配置/能力校验 | +100~150 | 必须 |
| 27 | `mooncake-store/tests/file_storage_config_test.cpp` | `Validate` 分支断言 | +30~50 | 必须 |
| 28 | `docs/source/design/store/kv-client-storage-backend.md` | 本设计页 | +250~400 | 必须 |
| 29 | `docs/source/design/store/mooncake-store.md` | toctree 条目 | +1 | 必须 |
| 30 | `mooncake-store/benchmarks/storage_backend_bench.cpp` | 性能基准 | +200~300 | 可延后 |
| 31 | `docs/source/deployment/ssd/ssd-offload.md` | 部署说明 | +40~80 | 可延后 |
| 32 | `mooncake-store/include/storage_backend.h` | `RemoveAll`/`EvictAboveDiskWatermark`/`RemoveByRegex` 支持 | — | 后续版本 |

### 6.2 汇总与估算依据

- **核心实现（1–18）**：约 **2,550–3,800 行**。
- **测试（19–27）**：约 **1,800–2,650 行**。
- **文档（28–29）**：约 **250–400 行**。
- **本版总计**：约 **4,600–6,850 行**，新增文件约 20 个，修改文件约 14 个。
- **可延后项（30–31）**：约 **240–380 行**。
- **后续版本（32）**：`RemoveAll`/`EvictAboveDiskWatermark`/`RemoveByRegex` 支持。


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
| P0（本版） | 后端 + 直接调用 `kv::KvClient` + staging（含 Quarantined）+ owner 按 key 删除接线 + 破坏性 Wait/PARTIAL/Query 分片处理 + 写/读/查/删契约测试 | 四步操作闭环；`make` + 测试通过 |
| P1 | 性能基准；批量查询接口（如需要） | 端到端压测通过 |
| P2 | 删除任务持久化、`RemoveAll`/`RemoveByRegex`/水位淘汰、恢复 manifest、零拷贝、大对象分片 | 单独设计评审 |



## 附录 A：术语

| 术语 | 含义 |
| --- | --- |
| ASU | 存储节点/设备：持久化 KV，拥有自身的容量与 GC 策略 |
| UCM | 客户端库/接入层：提供 `kv::KvClient`、视图路由、task 管理与内存注册 |
| KVClient | 本文专指 UCM 的 `kv::KvClient`（旧分支 `UC::ASU::AsuClient`） |
| `ObjectKey` | 用户/Client API 的对象 key，`using ObjectKey = std::string`（`types.h:201`）；不含 tenant，Master 侧按 `TenantId` 区分 |
| `asu_key` | ASU key 变量名，类型 `kv::CacheKey`（8 字节）；由 `XXH64("mooncake/kvclient/v1" + key_namespace + 0x00 + scoped_key)` 得到 |
| `kv::CacheKey` | ASU key 的类型：`std::array<std::byte, 8>` |
| KVBuffer | ASU 的一次读写描述：key + 注册内存区间 + offset |
| `scoped_key` | `TenantId::MakeScopedKey(tenant, key)`，即 `tenant_id + '\0' + local_key`；backend 接口统一用该名称，`FileStorage` 现有的 `storage_key` 指同一对象 |
| key_namespace | 配置项，参与 ASU key 哈希的域隔离；与 Master 的 `TenantId` 无关 |
| TenantId | Master 的多租户标识，编码在 `scoped_key` 的 `tenant` 前缀中；不参与 ASU key 计算 |
| `RecordHeader` | 24 字节记录头：magic/version/flags/payload_size/CRC |
| `Record` | `RecordHeader` + payload，即写入 ASU 的一条 value |
| payload | 对象字节（value bytes），各 `Slice` 按序拼接，不含 `RecordHeader`；等于上报 Master 的 `data_size` |
| transfer_size | `align_up(24 + payload_size, 512)`，ASU 实际传输长度 |
| staging arena | backend 持有、向 ASU 注册的读写中转内存 |
| `TaskId` | KvClient 的客户端任务编号；非 0、不复用；一次 `Wait` 后失效 |
| `cid` | SQE/CQE 配对键；transport 内部 16 位编号，与 `TaskId` 无关 |
| `ClientTask` | 一次 API 调用对应的任务；持有 entries/keys、视图快照、`entryStatus` |
| `TransportTask` | 一个 ASU 节点上的任务；由 ClientTask 按 key 路由拆分 |
| `TransportSubBatch` | 一个 SQE 子批；由 transport 按 ioNum 切分 |
| canonical handle | `RegisterRegions` 返回的客户端账本句柄；`KVBuffer.buffer.handle` 用它 |
| provider-local handle | 每个 transport provider 上 `BindMemory` 得到的内部句柄，不暴露给调用方 |
| `entryStatus` | 逐 key 的真实结果；顶层 `TaskResult.status` 通常只是 OK/PARTIAL_FAILED |
| `PARTIAL_FAILED` | 至少一个子批/条目失败；部分 OK 可能来自“已构建但未发送”的子批 |
| `Quarantined` slot | Wait 超时/任务丢失后无法确认底层是否仍在用；后端永久隔离、不复用/释放 |
| `mrKey` | 硬件 token；由 `SubmitAsync` 从 `buffer.handle` 解析，调用方传入值会被重置 |
| `ReferenceKvClient` | 仓库内的 `kv::KvClient` 参考实现（内存后端），仅 `client_impl=kReference` 时在 CI/单测使用 |
| `FakeKvClient` | 测试替身，实现 `kv::KvClient`，用于故障注入；不进入生产 |
