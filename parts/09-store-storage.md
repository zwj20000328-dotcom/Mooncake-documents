# 09 持久化与分层存储后端

> 本章基于仓库 `D:/xiaojun/work/Mooncake`（commit `67df8c41`）的实际源码精读写成。所有技术断言均带 `相对路径:行号`，可直接跳转核对。
>
> **路径约定**：本章所有相对路径以仓库根目录为基准。注意范围提示里的文件都位于 `mooncake-store/` 子目录下：例如 `mooncake-store/src/storage_backend.cpp`、`mooncake-store/src/file_storage.cpp`。仓库根目录**没有**独立的 `src/storage_backend.cpp`。另外 `master_store_backend.cpp` 与磁盘分层无关（见“设计取舍”一节），`local_ssd/` 与 `spdk/` 也不在 client 本地 offload 路径上。

---

## 定位与职责

Mooncake Store 的热数据在 DRAM 里（memory replica），冷数据需要一条**比 DRAM 便宜、但比对象存储快**的落盘通道。本子系统就是这条通道：

1. **容量扩展**：master 在内存压力下通过心跳点名若干 key，real client 把它们写到本地 SSD / NVMe KV / 共享 DFS / 对象存储。
2. **副本登记**：写成功后 client 调用 `NotifyOffloadSuccess`，master 给对象追加一个 `LOCAL_DISK` 副本，并在副本描述符里记录属主 client 的 RPC 地址。DFS 副本**不**走这条通知：它在 `PutStart` 时由 master 的 DFS allocator 分配 descriptor，直接以 PROCESSING 副本写进元数据（见流程 E）。
3. **读回落**：读路径在内存副本不可用时，向“拥有该盘上副本的 real client”发 RPC，由它从盘上读进预注册的 `ClientBuffer`，再由 Transfer Engine 用 RDMA/TCP 拉到请求方。
4. **自动回收**：盘用量超过水位时，后端先通知 master 删副本，master 接受后才真正删文件——**盘上数据的删除权掌握在 master 手里**。

一个关键认知：**`LOCAL_DISK` 副本不是“共享盘”，而是“某个 client 进程拥有的盘”**。读请求永远经过拥有者进程转发（`docs/source/deployment/object-storage-offload.md` 明确写了 S3/OSS 也是如此），所以拥有者进程下线要先 `DrainLocalDiskSegment` 做优雅摘除。

子系统内部有两条完全不同的数据面：

- **本地/对象 offload 数据面**：`FileStorage` 心跳线程 → `StorageBackendInterface::BatchOffload/BatchLoad`。覆盖 file-per-key、bucket、offset-allocator、NVMe KV、对象存储五种实现。
- **DFS 描述符数据面**：master 在 `PutStart` 时分配 `DistributedFSDescriptor`，client 拿描述符直接 `BatchWrite/BatchRead` 到共享文件系统。它**不走** `LOCAL_DISK` 心跳 offload（`mooncake-store/src/file_storage.cpp:142-147` 在 DFS 模式下直接 return，连心跳线程都不启动）。

---

## 目录结构速览

| 文件 | 一句话职责 |
|---|---|
| `mooncake-store/include/storage_backend.h` | 子系统总契约：`StorageBackendInterface`、`FileStorageConfig`、五种后端声明、`RecordHeader`(v3 盘上格式)、`BucketMetadata` |
| `mooncake-store/src/storage_backend.cpp` | 5663 行：file-per-key 原语、`StorageBackendAdaptor`、`BucketStorageBackend`、`OffsetAllocatorStorageBackend`、`CreateStorageBackend` 工厂 |
| `mooncake-store/include/file_interface.h` | `StorageFile` / `PosixFile` / `UringFile` / `FileLockRAII` |
| `mooncake-store/src/posix_file.cpp` | `preadv/pwritev/fdatasync`，写失败时析构 unlink 损坏文件 |
| `mooncake-store/src/uring_file.cpp` | 每线程一个 io_uring ring、进程级固定缓冲区注册、O_DIRECT bounce buffer、`batch_read` |
| `mooncake-store/src/uring_submit.h` | `submit_all_pending`：SQ 提交重试（EINTR/EAGAIN/ENOMEM） |
| `mooncake-store/include/file_storage.h` + `src/file_storage.cpp` | client 侧总协调者：心跳、offload/promotion、`BatchGet` 远程读、buffer 租约、优雅 drain |
| `mooncake-store/src/config/file_storage_config.cpp` | `MOONCAKE_OFFLOAD_STORAGE_BACKEND_DESCRIPTOR` 等环境变量解析 |
| `mooncake-store/include/storage/distributed/` | DFS/对象存储：`DistributedStorageBackend`、`FileSystemAdapter`、`ObjectStorageAdapter`、`ShardAllocator`、`ImmutableBucketAllocator` |
| `mooncake-store/src/storage/distributed/` | 上述实现，含 S3/OSS REST 引擎（HMAC 签名 + curl_multi） |
| `mooncake-store/include/nvme_kv/` + `src/nvme_kv/` | NVMe KV：16B 物理 key 编解码、冲突槽、inline/manifest 对象布局、三级线程池 |
| `mooncake-store/include/local_ssd/` + `src/local_ssd/` | **master 侧**：每 client 的 offload/promotion 任务邮箱与容量账本 |
| `mooncake-store/include/spdk/` + `src/spdk/` | NVMe-oF 远程 SSD 池（NoF）封装，**不在本地 offload 路径** |
| `mooncake-store/include/hf3fs/` + `src/storage/distributed/hf3fs_adapter.cpp` | 3FS USRBIO 资源管理与 DFS 适配器 |
| `mooncake-store/include/config/{bucket_backend,offset_allocator_backend,file_per_key}_config.h` | 各后端的策略开关 |
| `docs/source/design/store/ssd-offload.md` | 架构与数据流总览（建议先读） |
| `docs/source/deployment/ssd/{ssd-offload,nvme-kv,nvmf-ssd-deployment-guide}.md` | 部署与环境变量 |
| `docs/source/deployment/object-storage-offload.md` | OSS/S3 拓扑与凭据配置 |

---

## 核心数据结构

### 1. `StorageBackendInterface`：六个纯虚函数是核心契约

定义在 `mooncake-store/include/storage_backend.h:262-318`。六个纯虚方法如下：

| 方法 | 语义 |
|---|---|
| `Init()` | 打开设备/恢复盘上元数据；必须只调用一次 |
| `BatchOffload(batch_object, complete_handler, eviction_handler)` | 批量写；**写成功但 master 未被告知**时必须走回滚/上报失败 |
| `BatchLoad(batched_slices)` | 批量读到调用方预分配的 `Slice` |
| `IsExist(key)` | 存在性探测（用于 dangling replica 自愈） |
| `IsEnableOffloading()` | 是否还能继续 offload（配额/水位） |
| `ScanMeta(handler)` | 扫描盘上已有对象并批量回调，用于重启后向 master 重注册 |

除了这六个纯虚方法，接口里还有 4 个**带默认实现**的虚函数，它们在真实流程中同样会被调用，读代码时不能漏看：`ResetScanIterator()`（bucket 的 `ScanMeta` 基于游标，重扫前必须调用，见 `FileStorage::ReRegisterOffloadedObjects`）、`RemoveAll()`（master 要求清盘时由 `Heartbeat` → `PollRemoveAll` 触发）、`EvictAboveDiskWatermark(high, low, handler)`（主动水位驱逐）、`SetTestFailurePredicate(...)`（仅测试注入，默认 no-op）。

两个回调的事务语义是全章最容易读错的地方：

- `complete_handler(keys, metadatas)` 由 `FileStorage` 提供，内部调用 `client_->NotifyOffloadSuccess`（`mooncake-store/src/file_storage.cpp:342-367`）。它返回非 OK 意味着**数据已经在盘上、但 master 不知道**。此时 bucket 后端会 `RollbackCommittedBucket` 把本地索引和文件一起撤掉（`mooncake-store/src/storage_backend.cpp:1667-1680`），offset-allocator 后端则直接返回错误、依赖下次 `ScanMeta` 兜底（`storage_backend.cpp:5243-5255`）。
- `EvictionHandler(evicted_keys)` 由 `FileStorage` 提供，内部调用 `client_->BatchEvictDiskReplica(..., ReplicaType::LOCAL_DISK)`（`file_storage.cpp:579-626`）。**只有它返回成功，后端才被允许删文件**。这是“两阶段驱逐”的核心。

### 2. `FileStorageConfig`

`storage_backend.h:205-260`。要点：

- `storage_backend_type` 默认 `kBucket`；`storage_filepath` 默认 `/data/file_storage`，且 `Validate()` 强制**目录必须已存在、绝对路径、非符号链接、可写**（`mooncake-store/src/config/file_storage_config.cpp:146-205`）。即使是对象存储模式也要这个目录，只是不拿它缓存数据。
- `local_buffer_size` 默认 1280 MiB（`storage_backend.h:213`），这是 staging/读缓冲池大小，不是磁盘配额。
- `total_size_limit` 默认 2 TiB、`total_keys_limit` 默认 1000 万；不同后端对它们的解释不同（bucket 用 `max_total_size`，offset-allocator 直接当文件容量 `capacity_`）。
- `use_uring`：全局开关，决定 `PosixFile` 还是 `UringFile`。
- `enable_dfs` 不是环境变量直接配的，而是 `FileStorage` 构造时根据 `DistributedStorageBackend::UsesObjectStorage()` 反向设置（`mooncake-store/src/file_storage.cpp:86-93`）。

### 3. `StorageBackend`（file-per-key 原语）

`storage_backend.h:328-655`。它**不是** `StorageBackendInterface` 的实现，而是 `StorageBackendAdaptor` 的底层工具类，负责：

- 目录状态重建：`Init(quota_bytes)` 递归扫描、按创建时间排序、重建 FIFO 队列、计算 `used_space_/total_space_/available_space_`；配额缺省为磁盘容量的 90%（`storage_backend.cpp:95-206`）。
- 空间预留与 FIFO 驱逐：`EnsureDiskSpace` 先 `CheckDiskSpace` 预留，不足则循环 `PopFileToEvictByFIFO`，凑够后再调用 `eviction_handler`，master 接受后才 `DeleteEvictedFile`（`storage_backend.cpp:936-1020`）。
- 路径级 64 路分段锁 `file_path_mutexes_`（`storage_backend.h:518-519`，取锁 `storage_backend.cpp:85-88`），把“同一路径的写/删”串行化；`pending_eviction_paths_` 防止正在被驱逐的文件被重新写入。
- 物理文件选择：`create_file` 只对 **读** 加 `O_DIRECT`；`use_uring_` 打开时读、写都返回 `UringFile`，但写模式的 `use_direct_io=false`（buffered，不加 `O_DIRECT`）（`storage_backend.cpp:701-737`）。注释给的理由是“写延迟不敏感，且 O_DIRECT 写要 4KiB padding，会破坏 meta 解析、浪费空间”。

### 4. `BucketMetadata` / `BucketReadGuard` / `StorageObjectMetadata`

- `BucketMetadata`（`storage_backend.h:40-98`）：`meta_size`、`data_size`、`keys`、`metadatas`（每个对象的 `{offset,key_size,data_size}`）。注意 `YLT_REFL(BucketMetadata, data_size, keys, metadatas)` **没有序列化 `meta_size`**（`storage_backend.h:98`），`meta_size` 是写 `.meta` 文件后回填的序列化长度（`storage_backend.cpp:3383`）。
- `inflight_reads_` / `last_access_ns_` 是 `mutable std::atomic` 的**运行时字段**，拷贝/移动构造时显式清零（`storage_backend.h:56-96`），保证不会被元数据序列化带走。
- `BucketReadGuard`（`storage_backend.h:112-154`）：构造时 `fetch_add(1, relaxed)`，析构 `fetch_sub(1, release)`。它让删除方可以“先摘索引，再等 in-flight 读归零，最后删文件”。
- `StorageObjectMetadata`（`mooncake-store/include/types.h:539-547`）：`{bucket_id, offset, key_size, data_size, transport_endpoint}`。它是 client 与 master 之间的副本描述符载体；`transport_endpoint` 由 `FileStorage` 在调用 `complete_handler` 前填入本机 RPC 地址（`file_storage.cpp:347-349`、`184-199`）。
- `RecordHeader`（`storage_backend.h:1249-1359`）：offset-allocator 后端的盘上记录头，v3 布局：

```text
[u32 key_len][u32 value_len][u64 seq][u32 flags][u32 crc32][key][zero pad][value]
```

  `SIZE=24`，`value` 起点按 `key_len` 纯函数对齐到 4 KiB（`ValueOffsetInRecord`），为 GDS/DMA 预留；`seq` 是写入时的 `insert_seq_` 戳；`flags & kFlagHasCrc` 决定 `crc32` 是否可信（flag 定义见 `storage_backend.h:1268-1272`）；未知 flag 位在恢复扫描时会导致该 extent 被丢弃并释放（`storage_backend.cpp:4225-4232`）。

### 5. `BucketStorageBackend` 的内部状态

`storage_backend.h:1064-1131`：

- `mutex_`（`SharedMutex`）保护 `object_bucket_map_`、`buckets_`、`total_size_`、`lru_index_`、`pending_eviction_*`、`pending_write_*`。
- `object_bucket_map_`：key → `StorageObjectMetadata`（含 bucket_id/offset）。`buckets_`：`std::map<bucket_id, shared_ptr<BucketMetadata>>`，bucket_id 单调递增，所以 FIFO 候选就是 `begin()`。
- `lru_index_`：`std::set<pair<last_access_ns, bucket_id>>`。它是**惰性索引**：读路径只原子更新 `last_access_ns_`，不碰这个 set；驱逐时才在独占锁下修复（`storage_backend.cpp:2724-2755`）。
- `pending_eviction_keys_` / `pending_eviction_size_`：已从索引摘除、但 master 尚未确认的 key 与字节数；`pending_write_keys_` / `pending_write_size_`：正在写、尚未提交的 key。它们参与配额投影，避免并发 offload 把同一份空间算两遍。
- `file_cache_`：读 fd 缓存。**当前代码里 `GetOrOpenFile` 只有头文件声明（`storage_backend.h:1127`）和定义（`storage_backend.cpp:3488`），没有任何调用点**，`BatchLoad` 实际每次 `OpenFile`（`storage_backend.cpp:1786`）。`file_cache_` 的 erase 逻辑仍然存在（`FinalizeEviction`/`CleanupOrphanedBucket`），属于“为缓存预留、目前基本闲置”的代码，读时不要误以为热路径命中了缓存。

### 6. `OffsetAllocatorStorageBackend`

- 单个预分配文件 `kv_cache.data` + 元数据 `kv_cache.meta`（`storage_backend.cpp:3632-3638`）。
- 元数据分 1024 个 shard，每个 shard 一把 `SharedMutex` + 一个 `unordered_map`（`storage_backend.h:1425-1449`），`ShardForKey` 用 `hash & 1023`。
- `ObjectEntry{offset,total_size,value_size,AllocationPtr allocation,fifo_seq}`（`storage_backend.h:1388-1414`）。`AllocationPtr = shared_ptr<RefCountedAllocationHandle>`：**读到一半被驱逐也不会 UAF**，因为物理 extent 由最后一个引用释放。
- FIFO 索引 `fifo_index_`（`seq → key`）与 `insert_seq_`（全局单调）；`eviction_mutex_` 保护两者，且**必须先于 shard 锁获取**（`storage_backend.h:1494-1504`）。
- 持久化结构 `OffsetAllocatorPersistedMetadata`：`{version, allocator_state, insert_seq, fifo_entries, evicted_keys_this_batch}`（`storage_backend.h:185-193`），当前版本 3。

### 7. 文件 I/O 层：`StorageFile` / `PosixFile` / `UringFile`

- `StorageFile`（`file_interface.h:57-163`）：纯虚 `write/read/vector_write/vector_read`，虚 `datasync()`，`error_code_` 在写失败时被置为 `FILE_WRITE_FAIL`，析构据此决定是否 `unlink`（`posix_file.cpp:25-42`）。`SetDeleteOnWriteFail(false)` 用于 fresh-start 新建的数据文件与恢复目标数据文件，避免写失败误删可恢复数据（`storage_backend.cpp:3847-3849`、`4640`）。
- `PosixFile`：`preadv/pwritev` 一次系统调用完成 scatter/gather；`write` 循环处理 EINTR；`vector_write` 要求**一次写满**，否则返回 `FILE_WRITE_FAIL`（`posix_file.cpp:113-132`）。
- `UringFile`：见流程 C 的第 3 步与“设计取舍”第 4 条。

### 8. DFS / 对象存储的数据结构

- `DistributedStorageMode{kFileSystem,kObjectStorage}`（`distributed_storage_backend.h:20-23`）；`DfsWriteRequest/DfsReadRequest = {key, descriptor, slices}`（`:25-35`）。DFS 读写必须带 `DistributedFSDescriptor`，因为 key 本身无法定位 shard/offset。
- `DistributedFSDescriptor{file_path, offset, object_size, aligned_size, shard_idx}` 定义在 `mooncake-store/include/replica.h:264-270`。bucket 模式下 `shard_idx` 复用为 bucket id（`dfs_allocator_interface.h:39-44`）。
- `FileSystemAdapter`（`fs_adapter.h:34-176`）：文件管理和 fd 级 `WriteAt/ReadAt`，可选 `BatchWriteAt/BatchReadAt`（默认串行）、`SupportsFdReopen()`（POSIX 为 true）、`SupportsBatchIo()`（当前 3FS 为 true）。`FdIoRequest{fd, iov, iovcnt, offset}` 是批 I/O 的单元（`fs_adapter.h:20-25`）。
- `ObjectStorageAdapter`（`object_storage_adapter.h:41-104`）：逻辑 key 的 `Put/PutV/Get/PutBatch/GetBatch/Exists/Delete/ListKeys`。逻辑 key 可以为任意字节，物理 key 由适配器负责可逆编码。
- `ShardAllocator::ShardState`（`shard_allocator.h:120-163`）：每 shard 一个 `OffsetAllocator` + `offset_to_handle` + 独立 LRU + `pending_free` 延迟释放队列。
- `ImmutableBucketAllocator`（`immutable_bucket_allocator.h:25-190`）：`BucketEntryState{PENDING,COMMITTED,TOMBSTONE}` × `BucketLifecycle{ACTIVE,SEALED,FROZEN,EVICTING,RETIRED}` 双状态机；`BucketState` 只存在于内存，`Init` 拒绝任何已有 bucket 文件的根目录（`immutable_bucket_allocator.cpp:96-109`）。

### 9. NVMe KV 的数据结构

- `NvmeKvPhysicalKey = std::array<uint8_t,16>`（`key_codec.h:12`）。16 字节物理 key 由 canonical identity 连同 role/slot/chunk_index 一起用两个 XXH64 seed 派生（`key_codec.cpp:32-50`、`:91-107`），另有一个 32 字节 `verify_hash`（4 个 XXH64）写进对象头（`object_layout.h:18-28`）。
- `NvmeKvObjectHeader{magic,object_type,payload_size,verify_hash,payload_checksum,header_checksum,identity_metadata_size}`（`object_layout.h:18-28`）。
- 对象两种形态：`kInline`（header+identity+payload 一个 value）与 `kManifest`（header+identity+manifest，manifest 指向多个 chunk 物理 key）（`object_layout.h:13-16`、`41-50`）。
- `NvmeKvStorageBackend` 内部：`manifest_cache_`（`shared_mutex` 保护）、`connector_`、`total_size_/total_keys_` 原子计数、三组 `ThreadPool`（`backend.h:83-96`）。

### 10. master 侧 `LocalSsdManager` / `LocalSsdTaskMailbox`

`local_ssd/manager.h:21-151`：

- 每个注册的 real client 一条 `ClientRecord`：`total_capacity_bytes`（受 `stats_mutex` 保护）、`used_bytes`（原子）、`mailbox`。
- mailbox 内是 `pending_offloads_`（按 scoped key 去重）与 `pending_promotions_`（去重 + `seq` 重新排序）。`TakePromotions` 用 `partial_sort` 按 seq 降序取前 N（`manager.cpp:63-88`），保证积压时也是“最近被再次触达的优先”，不会退化成哈希序。
- `ExportPersistedState/RestorePersistedState` 让 **pending offload 队列**（连同 `enable_offloading` 与 `total_capacity_bytes`）进 HA 快照（`persisted_state.h`、`manager.cpp:356-394`）；注意 `pending_promotions_` **不** 被持久化，重启后 promotion 队列为空。
- 容量上报 `ReportCapacity` 返回 `{previous,current}`，供 master 更新 `file_total_capacity_`；`AdjustUsedBytes` 只在 `NotifyOffloadSuccess`（加）与 `ReleaseLocalDiskUsage`（减）两处被调用（`master_service.cpp:9148`、`2139`），promotion 不直接调整这个账本。

---

## 关键流程剖析

### 流程 A：启动与盘上状态恢复

```text
RealClient::setup (real_client.cpp:1484-1518)
  └─ FileStorage::FileStorage            // 建 allocator、选 backend、注册 io_uring 全局 buffer
  └─ FileStorage::Init                   // file_storage.cpp:129-228
       ├─ RegisterLocalMemory            // ClientBuffer 以 remote_accessible=true 注册
       ├─ storage_backend_->Init()
       │    ├─ BucketStorageBackend::Init          // storage_backend.cpp:1906-2169
       │    ├─ OffsetAllocatorStorageBackend::Init // storage_backend.cpp:3642-3913
       │    ├─ StorageBackendAdaptor::Init         // storage_backend.cpp:1161-1191
       │    └─ DistributedStorageBackend::Init     // distributed_storage_backend.cpp:174-214
       ├─ MountLocalDiskSegment(enable_offloading_)        // master 注册 LOCAL_DISK 段
       ├─ ReportSsdCapacity(total_size_limit)              // 仅用于展示分母
       ├─ ScanMeta(handler) → NotifyOffloadSuccess         // 重注册盘上对象
       └─ 启动 heartbeat_thread_ + client_buffer_gc_thread_
```

分后端看恢复语义，这是本章最值得精读的部分：

**Bucket**（`storage_backend.cpp:1906-2169`）：

1. 先在 `storage_path_` 下取 `.mooncake_local_disk.lock` 的**非阻塞独占 flock**；失败即 `FILE_OPEN_FAIL`（`:1914-1938`）。这确立了“一个目录只能有一个活 client”的契约，直接排除了两个进程写同一批 bucket_id 的可能。
2. 扫描所有 `.meta`：解析失败、或对应 `.bucket` 数据文件缺失/不是普通文件、或 `data_size/meta_size/keys/metadatas` 任一为空 → 把该 bucket 的数据与元数据都删掉并跳过（`:1970-2054`）。
3. 对每个有效 bucket 重建 `object_bucket_map_`、`total_size_`、`lru_index_`（时间戳置 0，因而 LRU 下“从未被读”的桶最先被淘汰，等价于 FIFO）。
4. 第二遍扫描 `.bucket` 文件，删掉没有对应 `.meta` 的孤儿文件（`:2072-2131`）。这是“数据写成功但元数据写失败”崩溃窗口的兜底。
5. `max_total_size` 未配置时取磁盘容量的 90%（`:2140-2150`）；`BucketIdGenerator` 从最大已有 id 续号（`:2152-2160`）。

**OffsetAllocator**（`storage_backend.cpp:3642-3913` + `4499-4678`）：`TryRecoverFromMetadata` 返回四种结果，语义差别巨大：

| 结果 | 触发条件 | 行为 |
|---|---|---|
| `kRecovered` | 元数据、allocator、数据文件大小全部校验通过 | 恢复索引与 FIFO，直接可用 |
| `kNoMeta` | 没有 `.meta` | 真正的首次启动，清空重建 |
| `kCorrupt` | 版本不匹配、解析失败、capacity 不匹配、数据文件大小不匹配、数据文件 ENOENT | **允许从零开始**（会 `O_TRUNC` 数据文件） |
| `kTransientError` | `FILE_OPEN_FAIL`、`bad_alloc`、fstat 失败、非 ENOENT 的 open 失败 | **绝不能擦数据**：`Init` 直接失败，让运维重试 |

恢复细节：`RebuildShardMapsFromAllocator` 遍历 allocator 里标记为 used 的 extent，逐条读 header/key/value；任何 `seq >= checkpoint.insert_seq` 的记录被判为“checkpoint 之后写的、可能是撕裂写”而丢弃并释放 extent（`storage_backend.cpp:4258-4272`）；带 CRC 的记录会分块校验（1 MiB chunk，`:4294-4337`）；同一 key 出现两份时保留 seq 更大的一份（`:4339-4397`）。随后 `RestoreAndRepairFifoIndex` 恢复 FIFO（`:4422-4493`）：A 阶段丢弃 seq >= insert_seq 的条目；B 阶段给 shard map 里没有 FIFO 槽的 key 补新 seq；C 阶段根据 `evicted_keys_this_batch` 删除“复活”的已驱逐 key。

**FilePerKey**（`storage_backend.cpp:1359-1444`）：扫描两级哈希目录，读每个文件、`struct_pb::from_pb` 出 `KVEntry`，回填 `total_keys/total_size`，并按 `scanmeta_iterator_keys_limit` 分批回调。注意它在扫描时**持有 `mutex_` 整个回调过程**，因此大目录启动会阻塞。

**DFS**：`DistributedStorageBackend::Init` 只建目录、初始化 adapter；**不扫描、不恢复任何分配状态**（`distributed_storage_backend.cpp:198-213`）。DFS 的分配器在 master 侧（`master_service.cpp:518-548`），并且 `enable_oplog` 与 DFS 互斥（`master_service.cpp:503-508`），`single_tenant=false` 也被拒绝（`:510-516`）。`ImmutableBucketAllocator` 更激进：根目录里已有任何 bucket 文件就拒绝启动（`immutable_bucket_allocator.cpp:96-109`）。

**对象存储**：`Init` 后可选 `CheckHealth()`（写读删探针，`distributed_storage_backend.cpp:180-195`）；`ScanMeta` 用 `ListKeys()` 枚举逻辑 key 并按批回调（`:863-904`）。

最后 `FileStorage::Init` 对 DFS 有一个**提前返回**：`config_.enable_dfs` 为真时只启动 buffer GC 线程就返回（`file_storage.cpp:142-147`）。这意味着 DFS 模式**不 mount LOCAL_DISK 段、不跑 offload 心跳、不做 ScanMeta 重注册**——DFS 副本由 master 的 allocator 直接管理，与 `LOCAL_DISK` 是两套东西。

### 流程 B：一次 offload（内存 → 盘）

```text
heartbeat_thread_ (每 heartbeat_interval_seconds)
 └─ FileStorage::Heartbeat                         // file_storage.cpp:662-812
     ├─ draining_? return
     ├─ (可选) 重试 metadata_resync_pending_ 的后台 rescan
     ├─ [offloading_mutex_] client_->OffloadObjectHeartbeat(enable_offloading_, tasks)
     │    └─ SEGMENT_NOT_FOUND? → MountLocalDiskSegment + ReportSsdCapacity + 重试
     │         + 异步 ReRegisterOffloadedObjects()
     ├─ client_->PollRemoveAll() → RemoveAll()
     ├─ OffloadObjects(tasks)                      // file_storage.cpp:307-577
     │    ├─ tenant scoped key 化
     │    ├─ BucketStorageBackend::AllocateOffloadingBuckets  // 按 bucket_keys_limit / bucket_size_limit 分桶；跳过盘上已存在的 key；flush_partial_buckets=false 时余下的 key 存进 ungrouped_offloading_objects_ 留到下一轮
     │    ├─ 每个 bucket: BatchQuerySegmentSlices(user_keys, tenant) → 内存 Slice
     │    ├─ 设备指针 → PinnedBufferPool D2H staging
     │    ├─ storage_backend_->BatchOffload(host_batch_object, complete, eviction)
     │    └─ 未成功/未上报的 key 收集进 failed_tasks
     ├─ failed_tasks → NotifyOffloadSuccess(tasks, data_size=-1 哨兵)
     ├─ ProcessPromotionTasks()
     └─ RunDiskWatermarkEviction()
```

几个必须看懂的点：

1. **`BatchQuerySegmentSlices` 是 best-effort**（`file_storage.cpp:1047-1073`）：它调 `client_->BatchQuery` 查副本列表，只挑“在本进程内存里”的副本，拿不到就当作失败任务。注释明确说它现在总是返回 OK（`:400-403`），失败靠 `failed_tasks` 表达。
2. **D2H staging**：`storage_backend` 的所有实现都假定收到的是 host 指针（`ConcatSlicesToString`/`BuildBucket`/`WriteBucket` 直接 memcpy）。因此 GPU/XPU slice 必须先拷到 `pinned_buffer_pool_`（`file_storage.cpp:418-453`）。
3. **错误分级**：`INVALID_READ` 与 `OBJECT_ALREADY_EXISTS` 被视为“本 bucket 软失败”，继续处理后续 bucket；`KEYS_ULTRA_LIMIT` 会把 `enable_offloading_ = false` 并中止本轮（`file_storage.cpp:302-305`、`521-547`）。这个分级修复过 issue #2827（注释里写了）。
4. **失败上报哨兵**：`failed_metadatas` 全部填 `StorageObjectMetadata{-1,0,0,-1,""}`（`file_storage.cpp:558-571`），master 看到 `data_size=-1` 就知道要清理 offloading 任务并释放源副本引用。不这么做，任务要等 TTL reaper 才回收。
5. **重复 key 是幂等 no-op**：`PrepareEviction` 对已在 `object_bucket_map_` 或 `pending_write_keys_` 的 key 放进 `skipped_keys`（`storage_backend.cpp:2818-2841`），提交阶段跳过；但**已被 `pending_eviction_keys_` 覆盖的 key 会返回 `OBJECT_ALREADY_EXISTS`**，因为此时 skip 可能丢掉唯一副本。如果整个 bucket 的 key 都已在盘上，`BatchOffload` 会 `CleanupOrphanedBucket` 删掉刚写的冗余文件并直接返回 `bucket_id`，**不调用 `complete_handler`**（`storage_backend.cpp:1628-1635`）。

Bucket 后端的 `BatchOffload` 内部顺序（`storage_backend.cpp:1514-1684`）值得逐行读：

1. `BuildBucket` 把 `[key bytes][value bytes][key bytes][value bytes]...` 拼成 iovec 数组，同时算出每个对象的 `offset/key_size/data_size`（`storage_backend.cpp:2407-2437`）。
2. `PrepareEviction(required_size, bucket->keys)`：独占锁下按 FIFO/LRU 选旧桶，从 `buckets_/object_bucket_map_` 摘除，累加 `pending_eviction_size_`，返回 `PendingEviction`。**此时不删任何文件**。
3. `eviction_handler(pending.keys)` 通知 master；失败则 `RestorePreparedEviction` 原样放回。
4. `CommitPreparedEviction` 清掉 `pending_eviction_keys_`；`FinalizeEviction` 才真正删文件：**先删 `.meta`，再等 `inflight_reads_` 归零，最后删 `.bucket`**（`storage_backend.cpp:3088-3158`）。先删 meta 是刻意的：即使后面删数据失败，重启 `Init` 也不会把这个已被 master 摘除的桶恢复出来。
5. `WriteBucket`：写数据 → `datasync()` → 写 `.meta`（`store_bucket_metadata`）。`datasync` 失败会 `CleanupOrphanedBucket` 并返回错误（`:2537-2553`）。
6. 提交本地索引（独占锁）：再次过滤重复 key，插入 `object_bucket_map_`、更新 `total_size_`、插 `buckets_` 与 `lru_index_`（`:1597-1659`）。**先提交索引再通知 master**，这样 master 一旦把读请求转过来，本地一定能命中。
7. `complete_handler` 失败 → `RollbackCommittedBucket`：摘索引 → 自旋等 in-flight 读 → 删文件（`:2622-2713`）。超时则**故意不删文件**，留给下次 `Init` 的孤儿扫描清理。

### 流程 C：一次 SSD 读（远程 client → 盘 → RDMA）

```text
请求方 Client::Get
 ├─ BatchGet 查 master，得到 LOCAL_DISK(rpc_addr=owner)
 ├─ (不同进程) RPC batch_get_offload_object → owner RealClient  (real_client.cpp:8694-8720)
 │    └─ FileStorage::BatchGet(keys, sizes)       // file_storage.cpp:267-287
 │         ├─ LoadBatch(prefer_pinned=false)
 │         │    ├─ AllocateBatch(client_buffer_allocator_)   // 每 key 一条 BufferHandle
 │         │    └─ storage_backend_->BatchLoad(allocated_batch->slices)
 │         ├─ 用 slice.ptr 回填 pointers（O_DIRECT 会对齐偏移修正）
 │         └─ 把 AllocatedBatch 放进 client_buffer_allocated_batches_[batch_id]
 ├─ 响应 {batch_id, pointers[], TE addr, gc_ttl_ms}
 ├─ Transfer Engine 从 ClientBuffer 拉数据
 └─ release_offload_buffer(batch_id) → FileStorage::ReleaseBuffer   // file_storage.cpp:1202-1214
```

- 同进程 GPU 恢复可走 `BatchGetLocal` + `pinned_restore_arena_`（`file_storage.cpp:289-300`、`1092-1167`），避免再经过一次远程 TE。但 `use_uring` 打开时 pinned arena 会被禁用（`file_storage.cpp:56-77`，代码只打印 “Pinned SSD restore is disabled with io_uring”）；推测原因是 O_DIRECT 读要求缓冲区与长度都 4 KiB 对齐，而 arena 的分配策略不保证这一点。
- `AllocateBatch` 在 `use_uring` 下会为每个对象多分配 `2*4096` 并把指针向上对齐（`file_storage.cpp:1120-1155`），这是给 O_DIRECT 的“头尾各一个页”留余量。
- 租约 GC：`client_buffer_allocated_batches_` 每条有 `lease_timeout = now + client_buffer_gc_ttl_ms`；`ClientBufferGCThreadFunc` 周期清理（`file_storage.cpp:1169-1192`），正常路径靠 `ReleaseBuffer` 主动释放。

Bucket 的 `BatchLoad`（`storage_backend.cpp:1703-1891`）是并发正确性最密集的读路径：

1. 共享锁下为每个 key 查 `object_bucket_map_` → `buckets_`，校验 `data_size == slice.size`，把读计划按 bucket 分组，并为每个 bucket 创建一个 `BucketReadGuard`（`:1722-1771`）。同时更新 `last_access_ns_`。
2. **释放锁**，按 bucket 打开文件做 I/O；guard 在整个 I/O 期间保证 `inflight_reads_ > 0`。
3. 如果是 `UringFile`（即 `use_uring` 且读模式），走 `batch_read`：对每个 key 计算 `actual_offset = offset + key_size`，向下/向上按 4 KiB 对齐成 `aligned_offset/aligned_size`，每次提交最多 `QUEUE_DEPTH = 32` 条独立读（`uring_file.cpp:53`、`:151-190`）；成功后把 `batch_object[key].ptr` 调整为 `dst + offset_in_buffer`（`:1794-1867`）。这里真正解决的是**文件偏移与读长度**的页对齐，而不是缓冲区对齐：`dst` 本身仍必须 4 KiB 对齐（由 `AllocateBatch` 保证，`UringFile::batch_read` 会拒绝未对齐的 buf/len/off，`uring_file.cpp:774-786`）。
4. 否则逐条 `vector_read`，要求读到恰好 `data_size` 字节。

注意这个读批次的**失败粒度是整批**：任何一个 key 在索引里找不到（`INVALID_KEY`）、所属 bucket 找不到（`BUCKET_NOT_FOUND`）或 `data_size` 与调用方 `Slice.size` 不符（`INVALID_PARAMS`），`BatchLoad` 都会立刻返回错误，已经读出的 key 也一起作废（`storage_backend.cpp:1722-1771`）。调用方必须按整个 batch 重试或降级。

### 流程 D：promotion（L2 盘 → L1 DRAM）与水位驱逐

`ProcessPromotionTasks`（`file_storage.cpp:868-1016`）是 offload 的反向流水线：

1. `client_->PromotionObjectHeartbeat(tasks)` 拉任务；master 每次最多返回一条（注释 `:898-903` 解释这是为了不超心跳存活窗口）。
2. `PromotionAllocStart` 在 master 侧分配一个 PROCESSING 的 MEMORY 副本。失败（通常 `NO_AVAILABLE_HANDLE`）要立刻 `NotifyPromotionFailure` 释放 admission 名额，否则名额被 TTL reaper 占满（`:915-936`）。
3. `AllocateBatch` 从 `client_buffer_allocator_` 拿 staging → `BatchLoad` 从盘读 → `PromotionWrite(memory_descriptor, tx_slices)` 用 TE 写入 DRAM 副本 → `NotifyPromotionSuccess` 提交。**任意一步失败都要 `NotifyPromotionFailure`**，注释明确说这是为了避免瞬时抖动把 `promotion_queue_limit_` 占满（`:938-952`）。
4. `RunDiskWatermarkEviction`（`file_storage.cpp:628-647`）与写路径驱逐复用同一套 `EvictAboveDiskWatermark`：bucket 后端把 `synthetic_required_size = max_total_size - target` 传给 `PrepareEviction`（`storage_backend.cpp:3204-3211`），file-per-key 后端则从 FIFO 队列直接选文件（`storage_backend.cpp:1022-1120`）。

### 流程 E：DFS 描述符数据面（控制面/数据面分离）

```text
PutStart (master_service.cpp:5589)
  └─ AllocateAndInsertMetadata (:5404) → AllocateReplicas (:5140)
       └─ dfs_allocator_->Allocate(key, value_length) → DistributedFSDescriptor   // :5300-5317
            └─ 作为 PROCESSING 的 DFS replica 写进元数据，随 PutStart 响应下发
Put 数据面 (client_service.cpp:2900-3010)
  ├─ 设备 slice D2H staging 到 pinned buffer
  └─ DistributedStorageBackend::BatchWrite(DfsWriteRequest[])   // distributed_storage_backend.cpp:633-720
       ├─ 校验 descriptor 范围（IsDfsDescriptorRangeValid / IsBucketDescriptorRangeValid）
       ├─ 校验 sum(slice.size) == descriptor.object_size
       ├─ adapter->SupportsBatchIo() ? 收集后批量提交 : ExecuteSingleIo
       └─ shard 模式：缓存 fd；bucket 模式：每批最多打开 64 个 bucket 文件
PutEnd (master_service.cpp:5725-5743)
  └─ bucket 模式额外 MarkCommitted(key, descriptor) 把 PENDING 翻成 COMMITTED
```

- **shard 模式**（`ShardAllocator`）：每个 shard 是预分配的 `dfs_shard_XX.data`，`Allocate` 先按 `hash(key) % shard_count` 选 shard，满了再线性探测其它 shard（`shard_allocator.cpp:255-300`）。`alloc_size = aligned_size + alignment - 1`，保证任意偏移都能向上对齐到 4 KiB。`Free` 不立即还空间，而是进 `pending_free` 延迟队列，默认 30 秒后才真正释放（`shard_allocator.cpp:398-401`、`588-611`），这是给“读请求手里还握着旧 descriptor”留的安全窗口。
- **bucket 模式**（`ImmutableBucketAllocator`）：只追加，不回收单条；条目布局就是 `[value][zero pad]`，没有头、没有校验（`bucket_entry_layout.h:12-19`）。`ReserveInBucketLocked` 永不回退 `append_offset`，即使 key 后来变成 TOMBSTONE（`immutable_bucket_allocator.cpp:186-216`）。驱逐是整桶删除：`PrepareEviction` 只选 `SEALED` 且 `pending_entries==0` 的桶，翻成 `FROZEN`；master 在**稳定顺序锁住所有相关元数据 shard** 后逐条校验 descriptor/lease/pin，全过才 `CommitEvictionLogical`（`master_service.cpp:8410-8567`）；物理删除失败则留在 `failed_deletions_` 等重试（`immutable_bucket_allocator.cpp:541-593`）。另外，`PutStart` 遇到 DFS 分配失败时会先重新 admit，再调用 `TryRecoverDfsSpaceAfterAllocationFailure()` **强制驱逐一个整桶**（`PrepareEvictionForAllocationFailure`，`manager` 侧见 `master_service.cpp:5576-5587`、`8400-8425`），成功后重试 admit；这是“盘满”时 bucket 模式的唯一在线回收路径。
- **对象存储模式**：`BatchOffload` 直接 `PutBatch` 整个逻辑对象（`:359-444`），`BatchLoad` 直接 `GetBatch`（`:815-845`），`ScanMeta` 用 `ListKeys`。它**不支持驱逐**（收到 `eviction_handler` 只打一次 warning，`:372-376`）。

### 流程 F：NVMe KV 的写与读

写（`nvme_kv/backend.cpp:479-973`）：

1. `BuildPayloadView` 把多 slice 拼成连续 payload（单 slice 零拷贝）。
2. 配额检查（keys/size）→ `IsEnableOffloading`。
3. 对每个对象 `BuildWritePlan(identity, payload, slot=0, max_value_size)`（`key_conflict_policy.cpp:32-123`）：payload 能塞进一个 value 就 `kInline`；否则按 `max_value_size` 切 chunk，再生成一个 `kManifest` 根对象。
4. 三级流水线：prepare（`prepare_concurrency_`）→ chunk 提交（`batch_submit_concurrency_` 个 lane，每个 lane 一个 `IndexQueue`）→ 根对象提交（`root_submit_concurrency_` 个 consumer，`root_queue`）。根对象**最后写**，它相当于可见性标记（`:628-830`）。
5. 失败回退：任何一个 chunk 失败会标记 `needs_fallback`，清理已写的 chunk/root，再逐对象走 `OffloadOne` 的“换 slot 重试”路径（`:832-916`、`975-1165`）。
6. `BuildWritePlan` 的 `slot` 与 `ResolveExistingObject` 配合实现 **store-if-not-exists + 冲突槽**：物理 key 已存在且内容相同 → 幂等成功；内容不同 → `OffloadOne` 换下一个 slot 重试，最多 `kNvmeKvMaxPhysicalKeySlots = 64` 个（常量在 `key_codec.h:18`，循环在 `backend.cpp:995`；`ResolveExistingObject` 在 `key_conflict_policy.cpp:125-145`）。

读（`backend.cpp:1167-1485`）：

1. `ResolveRoot` 逐个 slot 试 `Retrieve`，解析 header/identity，校验 `logical_key` 与 `resolved_physical_key/resolved_slot` 一致（`:147-184`）。
2. inline 对象直接校验 checksum 后 memcpy；manifest 对象解析 chunk 记录并缓存到 `manifest_cache_`。
3. chunk 读分两种：目标地址满足 DMA 对齐 → `RetrieveIntoBatch` 直接落盘；否则 `RetrieveBufferBatch` 再 memcpy（`:1336-1422`）。
4. `RunPipelinedIo` 让“读下一组”和“校验上一组”重叠（`:1433-1469`）。

---

## 与其他模块的边界

### 依赖谁

| 依赖 | 用途 | 证据 |
|---|---|---|
| `liburing` | `UringFile`、NVMe KV io_uring executor | `uring_file.cpp:18`、`file_interface.h:13`、`nvme_kv/executor_io_uring.cpp:9` |
| `offset_allocator` | offset-allocator 后端与 `ShardAllocator` 的空间分配 | `storage_backend.h:23`、`shard_allocator.h:17` |
| `yalantinglibs` | 元数据 `struct_pb` 序列化、`tl::expected` | `storage_backend.cpp:25`、`storage_backend.h:38` |
| `libcurl + OpenSSL` | OSS/S3 REST 引擎（HMAC 签名、curl_multi） | `rest_object_storage_adapter.cpp:253-254`、`:403-425`（curl）；HMAC 签名在 `storage/distributed/object_storage_signing.h:8,50-56`、`oss_adapter.cpp:431-442` |
| `3FS hf3fs_usrbio` | `Hf3fsAdapter` 数据面 | `hf3fs_adapter.cpp:15`、`hf3fs.h:8` |
| `SPDK + DPDK` | NoF 远程 SSD 池、SPDK 内存分配/注册 | `spdk_wrapper.h:83-108`、`src/common/client_buffer_allocation.cpp:128-131` |
| master 控制面 | `MountLocalDiskSegment`、`OffloadObjectHeartbeat`、`PromotionObjectHeartbeat`、`NotifyOffloadSuccess`、`BatchEvictDiskReplica`、`ReportSsdCapacity`、`PollRemoveAll` | `file_storage.cpp` 全文 |
| Transfer Engine | SSD→DRAM 的 promotion 写、SSD→远端读 | `file_storage.cpp:986-988` |

### 被谁依赖

- `RealClient`：创建/持有 `FileStorage`，并在析构前调 `DrainLocalDiskSegment`（`real_client.cpp:1484-1518`、`2362-2375`）。
- `Client`：通过 `SetDfsStorageBackend` 拿到 DFS 后端，在 Put/Get 路径直接 `BatchWrite/BatchRead`（`client_service.cpp:4275`、`2988`、`1798`）。
- `MasterService`：持有 `LocalSsdManager` 与 `DfsAllocatorInterface`，负责调度 offload/promotion/DFS 驱逐（`master_service.h:2156-2166`）。
- 监控：`SsdMetric`（`file_storage.cpp:470-498`、`1030-1043`）。

### 接口契约在哪

- client ↔ backend：`StorageBackendInterface`（`storage_backend.h:262-318`）。
- client ↔ master：`Client` 的 RPC 方法（`master_client.h`），语义由 `FileStorage` 的调用顺序定义。
- backend ↔ 盘：`StorageFile`（`file_interface.h:57-163`）与 `ObjectStorageAdapter`（`object_storage_adapter.h:41-104`）。
- master ↔ DFS 存储：`DfsAllocatorInterface`（`dfs_allocator_interface.h:45-79`）+ `DistributedFSDescriptor`（随副本描述符下发）。
- 数据怎么流过去：`StorageObjectMetadata` / `DistributedFSDescriptor` 都通过 `YLT_REFL` 序列化进副本描述符（`types.h:545`、`replica.h:270`）。

---

## 设计取舍与易踩坑点

1. **`master_store_backend.cpp` 与磁盘无关**。它实现的是 `WeightStoreBackend`（接口在 `mooncake-store/include/weight_store_backend.h:18-34`，`MasterStoreBackend final : public WeightStoreBackend` 在 `:38-58`），负责把 oplog 持久化回调与权重组快照桥接到 `MasterService`（`master_store_backend.cpp:9-78`）。不要把它归入存储后端。

2. **SPDK 不在本地 offload 路径上**。`SpdkWrapper` 服务的是 NVMe-oF 远程 SSD 池（`OpenNofSegment`/`SubmitRequest`，`spdk_wrapper.cpp:370`、`450`）以及 SPDK 内存分配/注册（`src/common/client_buffer_allocation.cpp:128-131`、`real_client.cpp:1023`）。本地落盘的文件路径只有 POSIX/io_uring（file-per-key、bucket、offset-allocator），另有 `nvme_kv/` 直连本地 NVMe 设备的独立路径；两者都不经过 SPDK NoF。`local_ssd/` 同理是 **master 侧** 的任务邮箱与容量账本（`local_ssd/manager.h:21-151`），不是 client 写盘代码。

3. **O_DIRECT 只用于读，不用于写**。file-per-key 在 `create_file` 里只对读加 `O_DIRECT`（`storage_backend.cpp:714-721`）；bucket 的 `OpenFile` 同样只对读返回 `UringFile`（`:3464-3484`）。由此产生一个**实际不可达的分支**：`WriteBucket` 里 `dynamic_cast<UringFile*>` 后的 `write_aligned` 路径（`:2457-2519`）在当前代码中永远进不去，因为写模式返回的是 `PosixFile`。读代码时不要据此认为 bucket 写用了 O_DIRECT。反过来，offset-allocator 后端的数据文件在 `use_uring` 时是 **O_DIRECT 读写都开**（`:3813-3841`、`4593-4638`），这与前两者不同。

4. **io_uring 是“每线程一个 ring”，不是每文件也不是全局**。`SharedUringRing::instance()` 是 `thread_local`（`uring_file.cpp:56-59`），注释解释了旧全局 ring 的锁竞争 >1ms、而 `fdget()` 只有 ~50ns，所以故意不用 `IOSQE_FIXED_FILE`（`:37-50`）。缓冲区注册却是进程级 `g_buf`（`:31-35`），每个线程 ring 在首次 I/O 时懒注册（`:75-100`）。注册前还会 `madvise(MADV_NOHUGEPAGE)`，避免 THP 下 `FOLL_LONGTERM` pin 失败（`:985-996`）。

5. **`batch_read` 的对齐语义容易被误读**。`use_direct_io_` 时 `UringFile::batch_read` 会校验 `buf`、`len`、`off` 三者都按 4 KiB 对齐，任一不满足直接返回 `FILE_INVALID_BUFFER`（`uring_file.cpp:774-786`）；因此调用方传入的 `dest_slice.ptr` **必须** 4 KiB 对齐——`FileStorage::AllocateBatch` 在 `use_uring` 下正是为此多分配 `2*4096` 并把指针向上取整（`file_storage.cpp:1119-1155`）。`BucketStorageBackend::BatchLoad` 做的是另一件事：把**文件偏移**向下对齐、长度向上取整，从对齐的偏移读到缓冲区起始处，再把 `slice.ptr` 往后挪 `offset_in_buffer`（`storage_backend.cpp:1808-1865`）。所以读完后 `Slice.ptr` 和调用前不是同一个值，`FileStorage::LoadBatch` 专门回填了 `pointers`（`file_storage.cpp:253-262`）。

6. **两阶段驱逐：删除权在 master**。bucket 与 offset-allocator 都遵循“先摘索引 → 通知 master → master 接受后才释放物理空间”。bucket 在 master 拒绝时 `RestorePreparedEviction` 原样放回（`storage_backend.cpp:3002-3045`）；offset-allocator 在 `NotifyAndCommitPreparedEviction` 里回滚（`:4801-4834`）。注意 offset-allocator 的驱逐只有 master 接受后才写 tombstone，否则恢复会误删仍然有效的 key（`:4745-4751` 注释）。

7. **`total_size_` 不是物理磁盘用量**。bucket 的 `total_size_` 是逻辑计数：一个 256 MiB 的 bucket 文件只有在**桶内所有对象**都被驱逐后才删除，所以按对象驱逐时它会显著低于真实占盘。为此 `max_physical_bytes` + `ActualDiskBytesUsedLocked()` 用 `stat.st_blocks * 512` 实测（`storage_backend.cpp:2762-2810`），口径与 `du`/kubelet emptyDir 一致；扫描失败时**fail-closed**（报告已达上限，不缓存），避免静默放开上限。结果按 `disk_scan_cache_ms` 缓存，`FinalizeEviction` 后失效（`:3165-3170`）。

8. **DFS 模式的 `FileStorage::Init` 提前返回**（`file_storage.cpp:142-147`）。因此 DFS 下不会 mount LOCAL_DISK 段、不跑心跳、不 `ScanMeta`。DFS 的 `BatchLoad/IsExist/ScanMeta` 都是 `NOT_SUPPORTED`（`distributed_storage_backend.cpp:815-873`），`IsEnableOffloading()` 返回 `UsesObjectStorage()`，对 DFS 即 false（`:859-861`）。**DFS 的读必须带 descriptor 走 `BatchRead`**，不能按 key 读。

9. **对象存储仍是 `LOCAL_DISK` 副本**。master 只知道“某个 real client 拥有它”，读请求经该 owner 转发（`docs/source/deployment/object-storage-offload.md` 的 Topology 一节）。`MOONCAKE_OFFLOAD_FILE_STORAGE_PATH` 仍是必填的本地目录，但不做缓存。对象存储后端也不参与驱逐（`distributed_storage_backend.cpp:372-376`），容量回收要靠上层/外部生命周期策略。

10. **`ImmutableBucketAllocator` 不做任何持久化元数据**。它的正确性前提是“进程重启后根目录为空”，所以 `Init` 见到任何 bucket 文件就拒绝（`immutable_bucket_allocator.cpp:96-109`）。系统整体也要求 DFS 与 `enable_oplog` 互斥（`master_service.cpp:503-508`）。把它当成“可恢复的持久化层”会错得很离谱。

11. **`ShardAllocator` 的延迟释放是给读者留的安全窗口**。`Free` 只把 handle 放进 `pending_free`，30 秒后才真正还给 allocator（`shard_allocator.cpp:398-401`、`588-611`）；代码没有写明理由，推测是为“读请求手里还握着旧 descriptor”留窗口。`EffectiveUsage` 会把 `pending_free` 里的字节算成“即将可用”（`:613-635`），因此水位判断用的是偏低的逻辑用量：与只看物理 free 相比，驱逐会更晚、更少触发；但这些字节在 `deferred_free_duration` 到期前并不能真正分配，极端情况下会出现“水位没到、分配已失败”。调小 `deferred_free_duration` 会提高空间利用率，但可能让仍在飞的 RDMA 读读到已被复用的区域。

12. **offset-allocator 的持久化是“尽力而为的缓存恢复”**。元数据 checkpoint 与数据写分离，恢复时 `seq >= checkpoint.insert_seq` 一律丢弃（`storage_backend.cpp:4258-4272`）；CRC 是每条可选（`enable_record_crc`），DMA 写入的记录可以不带 CRC；旧版本元数据直接拒绝。`kTransientError` 与 `kCorrupt` 的区分是设计精髓：资源类错误**绝不能**触发清盘（`:3760-3782`）。

13. **`BucketStorageBackend` 的 `BatchLoad` 其实没有用 `file_cache_`**。`GetOrOpenFile` 定义了但无调用点（全仓 grep 只有 `storage_backend.cpp:3488`），`BatchLoad` 走 `OpenFile`（`:1786`）。`file_cache_` 只在 `ClearFileCache`/驱逐/清理时被 erase。把它当成“热桶 fd 缓存”会误判性能特征。

14. **`OffsetAllocatorStorageBackend` 有单写者前提**。`BatchOffload` 顶部的注释明确说：`total_size_/total_keys_` 的读-改-写不是原子的，当前由 `FileStorage` 的单心跳线程保证；若将来引入并发 offload，`EvictToMakeRoom` 里的 `DCHECK_GE` 必须重新评估（`storage_backend.cpp:4844-4854`）。不要在这个后端外面加并发调用。

15. **`StorageBackendAdaptor`（file-per-key）的读写都经过 `struct_pb`**。写路径把 `KVEntry{key,value}` 序列化后整文件写（`storage_backend.cpp:1246-1254`）；读路径先按目标大小构造一个“等长占位串”再读回来（`:1318-1335`）。这意味着它比 bucket 多一次完整内存拷贝，且 `BatchLoad` 是逐 key 串行的，没有批量提交。它适合小规模/可读性优先，不适合百万对象。

16. **bucket 的 `IsExist` 不只看索引**。它还会 `fs::exists` 检查对应 `.bucket` 文件（`storage_backend.cpp:2171-2192`），因为写入是“先文件后索引”，索引命中但文件被外部抹掉时必须回答 false。这正是 `Client::Put` 用来自愈 dangling `LOCAL_DISK` 副本的探测（`real_client.cpp:1504-1518`）。

17. **pinned restore arena 与 io_uring 互斥**。`FileStorage` 构造时若 `use_uring` 打开会打印 “Pinned SSD restore is disabled with io_uring” 并跳过 arena 分配（`file_storage.cpp:56-77`）。依赖 `HasPinnedRestoreArena()` 的同进程 GPU 恢复路径会因此退化到普通路径（`real_client.cpp:8876-8923`），这是配置组合上的隐式约束。

18. **NVMe KV 的 `ScanMeta` 是空实现**（`nvme_kv/backend.cpp:1516-1525`）。也就是说重启后它**不会**向 master 重注册已有对象；`total_size_/total_keys_` 也从 0 开始（`Init` 里显式 store 0，`:473-474`）。这是与 bucket/file-per-key 的重要差异，做容量规划时不能假设“盘上已有数据会被恢复”。

19. **NVMe KV 的 16 字节 key 是哈希，不是路径**。冲突处理靠“最多 64 个 slot + 盘上存 logical key 校验 identity”（`key_codec.h:12-31`、`key_conflict_policy.cpp:23-30`）。因此 `IsExist` 需要逐个 slot `Retrieve` 探测（`backend.cpp:1487-1501`），比本地索引后端贵得多。

20. **优雅下线是锁协议，不是简单置位**。`DrainLocalDiskSegment` 在 `offloading_mutex_` 下先置 `draining_` 再 `UnmountLocalDiskSegment`（`file_storage.cpp:829-866`）。`Heartbeat` 在入口和拿到锁后各检查一次（`:674-676`、`709-711`），因为一个已通过入口检查的 tick 可能正阻塞在这把锁上；等它拿到锁时若不复查，就会因 `SEGMENT_NOT_FOUND` 重新 mount 已摘除的段。后台 `ReRegisterOffloadedObjects` 也在每个批次检查 `draining_` 并主动中止（`:1240-1244`）。

21. **bucket 的 owner lock 与 pid salt 是两层防御**。`Init` 用 `flock(LOCK_EX|LOCK_NB)` 保证一个目录一个活 client（`storage_backend.cpp:1914-1938`）；`BucketIdGenerator` 再把 `getpid() & 4095` 混进起始 id，缩小“同秒启动”的碰撞窗口（`:1460-1470`）。`OpenFile` 写模式对不存在的路径用 `O_EXCL`，对已存在路径才 `O_TRUNC`（`:3452-3461`）。三者缺一不可，不能只依赖 salt。

22. **此处未完全确认**：master 侧 DFS 驱逐的完整接受/回滚分支（`master_service.cpp:8410-8710`）我只精读到“稳定锁序 + 逐条校验 + 接受后提交”这一层；其中 `RunShardDfsEviction` 对 `accepted` 向量的构造（`:8597-8708`）没有逐行核对。若要做 DFS 驱逐相关的修改，请单独精读这段。

---

## 精读清单

建议按下面顺序读，每一步都带着一个具体问题：

1. **先读设计文档**：`docs/source/design/store/ssd-offload.md`（架构图 + 数据流 + 三种本地后端对比），再看 `docs/source/deployment/ssd/ssd-offload.md` 的环境变量表。目标：建立“心跳驱动、master 点名、client 落盘、读经 owner 转发”的全局图。
2. **`mooncake-store/include/storage_backend.h:262-318`**：把 `StorageBackendInterface` 六个纯虚方法的语义抄一遍，特别是 `complete_handler` 与 `EvictionHandler` 的失败语义。然后对照 `file_storage.cpp:342-367` 和 `579-626` 看它们各自的实现。
3. **`file_storage.cpp:662-812`（Heartbeat）**：画出 STEP 1/2/3 的顺序，标注 `offloading_mutex_` 的持有范围、`SEGMENT_NOT_FOUND` 的重挂载与异步 rescan。再读 `829-866`（Drain），理解为什么需要两处 `draining_` 检查。
4. **`file_storage.cpp:307-577`（OffloadObjects）**：重点看分桶、D2H staging、错误分级、`failed_tasks` 哨兵。问自己：一个 key 从 master 队列被取出，到最终 `NotifyOffloadSuccess` 或 NACK，所有分支是否都覆盖了？
5. **`storage_backend.cpp:1514-1684`（Bucket BatchOffload）**：按“BuildBucket → PrepareEviction → notify → Commit → Finalize → WriteBucket → 提交索引 → complete_handler”逐步读，每步记下失败路径。再读 `2812-3000`（PrepareEviction）与 `3080-3176`（FinalizeEviction），确认 `pending_*` 计数与 in-flight 读的配合。
6. **`storage_backend.cpp:1703-1891`（Bucket BatchLoad）**：先读非 uring 路径，再读 uring 的 `batch_read` 对齐计算。重点确认 `BucketReadGuard` 的生命周期覆盖整个 I/O 阶段。
7. **`storage_backend.cpp:1906-2169`（Bucket Init）**：列出所有“删掉这个 bucket”的条件，理解孤儿 `.bucket` 清理与 owner lock。
8. **`storage_backend.cpp:3642-3913` + `4499-4678` + `4161-4493`（OffsetAllocator）**：先看 `RecoveryResult` 四态，再看 `RebuildShardMapsFromAllocator` 的 seq 守卫与 CRC 校验，最后看 `RestoreAndRepairFifoIndex` 的三阶段。这是持久化设计的精华。
9. **`storage_backend.cpp:4838-5258`（OffsetAllocator BatchOffload）**：按 (A) 水位驱逐 → (B) 通知 master → (C) allocate → (D) fallback 驱逐 → (E) 写记录 → (F) 更新索引 + FIFO → 持久化屏障 的顺序读。注意 `eviction_mutex_` 先于 shard 锁的顺序约束。
10. **`uring_file.cpp:24-450`**：先读 `SharedUringRing` 的设计注释，再读 `ensure_buf_registered`、`batch_read`、`submit_rw/collect`。最后读 `979-1055` 的全局注册。目标：说清楚“每线程 ring + 进程级 buffer + 懒注册 + 不用 FIXED_FILE”这四个决策。
11. **`posix_file.cpp` 全文（149 行）**：很短，重点看析构里的 unlink 与 `vector_write` 的“必须一次写满”语义。
12. **`distributed_storage_backend.cpp:174-214`（Init）+ `359-444`（对象存储 BatchOffload）+ `633-813`（DFS BatchWrite/BatchRead）+ `517-631`（批量执行）**：理解两种模式的装配与“控制面分配 descriptor、数据面直写 DFS”的分工。对照 `master_service.cpp:5300-5317`（Allocate）与 `5725-5743`（bucket MarkCommitted）。
13. **`immutable_bucket_allocator.cpp:69-123`（拒绝非空根）+ `186-216`（只追加）+ `415-561`（整桶驱逐）**：理解双状态机与“运行时态不落盘”的取舍。再读 `docs/source/design/store/immutable-dfs-bucket-allocator.md`。
14. **`shard_allocator.cpp:39-160`（Init/恢复容量）+ `255-300`（Allocate）+ `433-577`（Prepare/Resolve 驱逐）+ `588-635`（延迟释放）**：理解 shard 布局、线性探测、LRU 与 pending_free。
15. **`nvme_kv/backend.cpp:479-973`（BatchOffload 三级流水线）+ `975-1165`（OffloadOne 冲突槽）+ `1167-1485`（BatchLoad）**：对照 `key_conflict_policy.cpp:32-143` 与 `key_codec.h`。重点：store-if-not-exists、manifest 可见性、slot 冲突。
16. **`local_ssd/manager.cpp:13-115`（mailbox）+ `117-197`（容量/生命周期）**：站在 master 视角理解 offload/promotion 的调度队列与 HA 快照。
17. **最后读 `storage_backend.cpp:5578-5661`（CreateStorageBackend）与 `file_storage_config.cpp:47-144`**：把环境变量到具体实现的映射表补齐，作为部署时的查表依据。

读完后应该能回答四个问题：(a) 一个 key 从内存到盘、再回到另一个进程的内存，中间经过哪些进程/线程/锁？(b) 盘上文件被删除的唯一合法触发链是什么？(c) 崩溃在“写数据后、写元数据前”和“写元数据后、通知 master 前”分别由谁兜底？(d) 五种后端各自牺牲了什么换取了什么？
