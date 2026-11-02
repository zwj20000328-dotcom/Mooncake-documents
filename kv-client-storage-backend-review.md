# KvClientStorageBackend 方案

## 1. 背景与边界

### 1.1 链路

```mermaid
flowchart LR
    FS["FileStorage<br/>BatchOffload / BatchLoad / IsExist / BatchRemoveKeys"]
    BE["KvClientStorageBackend<br/>BuildKvBuffers + FindPersistentHandle"]
    KC["kv::KvClient"]
    ASU["ASU"]

    FS -->|"① BatchOffload"| BE
    BE -->|"① BatchStoreAsync + Wait"| KC
    KC -->|"① 写"| ASU

    FS -->|"② BatchLoad"| BE
    BE -->|"② BatchLoadAsync + Wait"| KC
    KC -->|"② 读"| ASU

    FS -->|"③ IsExist"| BE
    BE -->|"③ QueryAsync + Wait"| KC
    KC -->|"③ 查"| ASU

    FS -.->|"④ BatchRemoveKeys（方案 A，未实现）"| BE
    BE -.->|"④ DeleteAsync + Wait"| KC
    KC -.->|"④ 删"| ASU
```

默认是“写完即下沉”：`PutEnd` 成功后 Master 把对象放进 owner 的 offload 队列，下一次心跳由 `FileStorage` 调 `BatchOffload` 写入 ASU；内存副本在 `NotifyOffloadSuccess` 前不可淘汰。本后端不感知触发时机。

### 1.2 本版范围

| 在范围内 | 说明 |
| --- | --- |
| 写 `BatchOffload` | `BuildKvBuffers` 编码进 tensor 区间，`BatchStoreAsync` + `Wait`，只上报成功 key |
| 读 `BatchLoad` | `BuildKvBuffers` + `BatchLoadAsync` + `Wait`，校验记录头，支持前缀读 |
| 查 `IsExist` | `QueryAsync` + `Wait`，同时看 `exists` 与 `entryStatus` |
| 删 | 逻辑删除；物理删除 |
| `Init` / `IsEnableOffloading` | 注册 `KVCacheRegistration`；`IsEnableOffloading` 在 `kReady` 后返回 true，未就绪返回错误；不查 ASU、不读计数 |
| `ScanMeta` | **不实现枚举**，返回 OK 空扫描；本文不讨论恢复 |

### 1.3 与现有后端的关系

| 后端 | 删除能力 | 恢复能力 |
| --- | --- | --- |
| `StorageBackendAdaptor` | `RemoveAll` | `ScanMeta` 目录扫描 |
| `BucketStorageBackend` | `RemoveAll` / 水位淘汰 | 内存索引 + 持久化元数据 |
| `NvmeKvStorageBackend` | 无 | 无（`ScanMeta` 空实现） |
| `KvClientStorageBackend` | 见 3.4 节 | 无（`ScanMeta` 空实现） |

---

## 2. 总体设计

### 2.1 模块与依赖

```mermaid
flowchart TB
    FS["FileStorage"] --> BE["KvClientStorageBackend"]
    FAC["CreateStorageBackend"] --> BE
    BE --> CFG["KvClientBackendConfig"]
    BE --> H1["私有 helper：KeyCodec / RecordCodec / BuildKvBuffers / FindPersistentHandle"]
    BE -->|"RegisterKVCaches<br/>Store/Load/Query/Delete<br/>Wait"| KC["kv::KvClient"]
    KC --> VIEW["ASU View Service"]
    KC --> ASU["ASU nodes"]
```

- UCM 头只在 `backend.cpp` 包含；`backend.h` 只前置声明 `namespace kv { class KvClient; }`。
- 不新增端口层/适配器层；key 编码、记录编解码、注册区域内的 tensor 区间分配都是 `backend.cpp` 的私有 helper。
- `backend.h` 不暴露 `kv::` 类型，避免 UCM 头进入 Mooncake 公共头。

### 2.2 线程与锁

| 线程 | 入口 | 行为 |
| --- | --- | --- |
| Heartbeat | `BatchOffload` | 串行写 |
| RPC worker（多） | `BatchLoad` | 并发读 |
| dangling probe | `IsExist` | 低频并发查 |
| SDK worker | `KvClientImpl` 内部 | 提交/等待，不回调 Mooncake |

锁纪律：

1. `write_mutex_` 只串行化写路径与 tensor 区间分配。
2. `regions_mutex_` 只保护注册区域的空闲/隔离 tensor 区间集合，**绝不覆盖 `Wait`**。
3. 任何 ASU `Wait` 期间不持锁；`complete_handler` 回调期间不持锁。
4. 无索引锁——读长度以 Master 的 `slice.size` 为准，不依赖本地索引。

---

## 3. 功能实现

entry 构造沿用 asuStore 命名：

- `BlockId`（16B）由 `key_namespace + '\0' + scoped_key` 的 128-bit 哈希派生；
- `MakeAsuKey(BlockId)` 生成 8B `kv::CacheKey`；
- 一个 Mooncake 对象 = 一个 `Shard` + 一个 tensor：`tensorSizes[0] = align_up(24 + Σ slice.size, 512)`，`offset = 0`；
- 注册内存 = `KVCacheRegistration`，Init 时 `RegisterKVCaches` 注册一次，返回 `RegisteredPersistentRegion{region, handle}`；
- `BuildKvBuffers(TaskDesc, tensorSizes)` 生成 `vector<kv::KVBuffer>`；`buffer.handle` 由 `FindPersistentHandle(region)` 解析；`mrKey` 留空，由 SDK 提交时写入。

---

### 3.1 写：`BatchOffload`

#### 3.1.1 类图

```mermaid
classDiagram
    class FileStorage {
        +BatchOffload(batch, complete_handler, eviction_handler)
    }
    class KvClientStorageBackend {
        +BatchOffload(...)
        -BuildTaskDesc(batch) TaskDesc
        -AllocateTensor(size) addr
        -BuildKvBuffers(task, tensorSizes) Entries
        -FindPersistentHandle(region) MRHandle
    }
    class TaskDesc {
        +shards
        +prerequisiteHandle
    }
    class Shard {
        +owner BlockId
        +index
        +addrs
    }
    class KVCacheRegistration {
        +addr
        +size
    }
    class RegisteredPersistentRegion {
        +region MemoryRegion
        +handle MRHandle
    }
    class KvClient {
        +BatchStoreAsync(entries, taskId)
        +Wait(taskId, timeout, result)
    }
    FileStorage --> KvClientStorageBackend : BatchOffload
    KvClientStorageBackend ..> TaskDesc : builds
    TaskDesc *-- Shard
    KvClientStorageBackend ..> KVCacheRegistration : Init 注册
    KvClientStorageBackend ..> RegisteredPersistentRegion : FindPersistentHandle
    KvClientStorageBackend --> KvClient : BatchStoreAsync + Wait
```

#### 3.1.2 时序图

```mermaid
sequenceDiagram
    autonumber
    participant MS as Master
    participant FS as FileStorage
    participant BE as Backend
    participant KC as kv::KvClient
    participant TR as Transport
    participant ASU as ASU

    FS->>MS: OffloadObjectHeartbeat
    MS-->>FS: OffloadTaskItem[]
    FS->>BE: BatchOffload(batch_object)
    loop 每个对象
        BE->>BE: block = MakeBlockId(scoped_key)
        BE->>BE: tensor_size = align_up(24 + Σslice.size, 512)
        BE->>BE: tensor_addr = AllocateTensor(tensor_size)
        BE->>BE: EncodeRecord(tensor_addr, slices)
        BE->>BE: task.push_back(Shard{block, 0, {tensor_addr}})
    end
    BE->>BE: entries = BuildKvBuffers(task, tensorSizes)
    BE->>BE: FindPersistentHandle(region) 取 buffer.handle；offset=0；mrKey 留空

    BE->>KC: BatchStoreAsync(entries, taskId)
    loop 每个 entry
        KC->>KC: mrKey.reset()；mrKey = registeredRegions[handle].tokenId  // 写 mrKey
    end
    KC-->>BE: taskId
    KC->>TR: dispatch
    TR->>TR: ValidateSqeMrKeys：mrKey 必须存在，否则 BUFFER_NOT_REGISTERED
    TR->>TR: PackSqe 读取 *mrKey → SQE.mr_key  // 读 mrKey
    TR->>ASU: Store(key, addr, length, mr_key)

    BE->>KC: Wait(taskId, timeout)
    KC-->>BE: TaskResult{status, entryStatus}
    alt status == OK
        BE->>BE: 只发布 entryStatus == OK 的 key
    else PARTIAL_FAILED / TIMEOUT
        BE->>BE: 本块不发布；TIMEOUT 的 tensor 区间隔离
    end
    BE->>BE: Release 或 Quarantine tensor 区间
    BE->>FS: complete_handler(published_keys, metadatas)
    FS->>MS: NotifyOffloadSuccess(tasks, metadatas)
```

#### 3.1.3 实现

```cpp
// Init：注册区域
KVCacheRegistration reg{kv_cache_addr, kv_cache_size};
RegisterKVCaches(&reg, 1);   // -> RegisteredPersistentRegion{region, handle}

// BatchOffload
tl::expected<int64_t, ErrorCode> BatchOffload(batch, complete_handler, ...) {
    if (!ready) return INTERNAL_ERROR;

    Detail::TaskDesc task;
    std::vector<std::size_t> tensorSizes;
    std::vector<std::string> keys;
    for (auto& [scoped_key, slices] : batch) {
        if (over_quota(scoped_key, slices)) continue;   // 配额在这里逐 key 跳过

        const auto block = MakeBlockId(scoped_key);
        const auto tensor_size = AlignUp(kRecordHeaderSize + SumSliceSize(slices),
                                         kv::kAlignmentBytes);
        auto tensor_addr = AllocateTensor(tensor_size);  // 注册区域内的连续区间
        EncodeRecord(tensor_addr, slices);               // header + payload + zero padding
        task.push_back(Detail::Shard{block, 0, {tensor_addr}});
        tensorSizes.push_back(tensor_size);
        keys.push_back(scoped_key);
    }

    auto entries = BuildKvBuffers(task, tensorSizes);   // key/handle/offset 在这里填
    if (entries.empty()) return int64_t{0};

    kv::TaskId taskId{};
    if (!client_->BatchStoreAsync(entries, taskId).ok()) {
        ReleaseTensors(task);                           // 没有 task，立即释放
        return int64_t{0};
    }

    kv::TaskResult result;
    auto st = client_->Wait(taskId, wait_timeout_ms, result);   // 只 Wait 一次
    if (!st.ok() || result.status.code == TIMEOUT ||
        result.status.code == TASK_NOT_FOUND) {
        QuarantineTensors(task);                        // 底层可能仍在飞
        return Map(st);
    }
    if (result.status.code == PARTIAL_FAILED) {
        ReleaseTensors(task);                           // 保守：本块不发布
        return int64_t{0};
    }

    std::vector<std::string> published;
    for (std::size_t i = 0; i < result.entryStatus.size(); ++i) {
        if (result.entryStatus[i].ok()) published.push_back(keys[i]);
    }
    ReleaseTensors(task);

    std::vector<StorageObjectMetadata> metadatas;
    BuildMetadatas(published, metadatas);
    complete_handler(published, metadatas);
    return published.size();
}
```
---

### 3.2 读：`BatchLoad`

#### 3.2.1 类图

```mermaid
classDiagram
    class FileStorage {
        +BatchLoad(slices)
    }
    class KvClientStorageBackend {
        +BatchLoad(slices)
        -BuildTaskDesc(slices) TaskDesc
        -BuildKvBuffers(task, tensorSizes) Entries
        -FindPersistentHandle(region) MRHandle
        -DecodeRecord(buffer) Record
    }
    class TaskDesc
    class Shard
    class KvClient {
        +BatchLoadAsync(entries, taskId)
        +Wait(taskId, timeout, result)
    }
    FileStorage --> KvClientStorageBackend : BatchLoad
    KvClientStorageBackend ..> TaskDesc : builds
    TaskDesc *-- Shard
    KvClientStorageBackend --> KvClient : BatchLoadAsync + Wait
```

#### 3.2.2 时序图

```mermaid
sequenceDiagram
    autonumber
    participant FS as FileStorage
    participant BE as Backend
    participant KC as kv::KvClient
    participant TR as Transport
    participant ASU as ASU

    FS->>BE: BatchLoad(map<scoped_key, Slice>)
    loop 每个对象
        BE->>BE: block = MakeBlockId(scoped_key)
        BE->>BE: tensor_size = align_up(24 + slice.size, 512)
        BE->>BE: tensor_addr = AllocateTensor(tensor_size)
        BE->>BE: task.push_back(Shard{block, 0, {tensor_addr}})
    end
    BE->>BE: entries = BuildKvBuffers(task, tensorSizes)
    BE->>BE: FindPersistentHandle(region) 取 buffer.handle；offset=0；mrKey 留空

    BE->>KC: BatchLoadAsync(entries, taskId)
    loop 每个 entry
        KC->>KC: mrKey.reset()；mrKey = registeredRegions[handle].tokenId  // 写 mrKey
    end
    KC-->>BE: taskId
    KC->>TR: dispatch
    TR->>TR: ValidateSqeMrKeys；PackSqe 读取 *mrKey → SQE.mr_key  // 读 mrKey
    TR->>ASU: Load(key, addr, length, mr_key)

    BE->>KC: Wait(taskId, timeout)
    KC-->>BE: TaskResult{status, entryStatus}
    alt status == OK
        BE->>BE: 解析 RecordHeader；头 CRC；要求 slice.size <= payload_size
        alt slice.size == payload_size
            BE->>BE: payload CRC；拷贝 payload 到 slice.ptr
        else slice.size < payload_size
            BE->>BE: 只拷贝前缀 slice.size；payload CRC 不可校验
        end
    else PARTIAL_FAILED / TIMEOUT
        BE->>BE: 本块不返回数据；TIMEOUT 的 tensor 区间隔离
    end
    BE->>BE: Release 或 Quarantine tensor 区间
    BE-->>FS: OK / first_error
```

#### 3.2.3 实现

```cpp
tl::expected<void, ErrorCode> BatchLoad(std::unordered_map<std::string, Slice>& slices) {
    Detail::TaskDesc task;
    std::vector<std::size_t> tensorSizes;
    std::vector<Slice*> sliceList;
    std::vector<std::uintptr_t> tensorAddrs;
    for (auto& [scoped_key, slice] : slices) {
        if (slice.ptr == nullptr || slice.size == 0) return INVALID_PARAMS;

        const auto block = MakeBlockId(scoped_key);
        const auto tensor_size = AlignUp(kRecordHeaderSize + slice.size,
                                         kv::kAlignmentBytes);
        auto tensor_addr = AllocateTensor(tensor_size);  // 读入目标区间
        task.push_back(Detail::Shard{block, 0, {tensor_addr}});
        tensorSizes.push_back(tensor_size);
        sliceList.push_back(&slice);
        tensorAddrs.push_back(tensor_addr);
    }

    auto entries = BuildKvBuffers(task, tensorSizes);
    kv::TaskId taskId{};
    auto submit = client_->BatchLoadAsync(entries, taskId);
    if (!submit.ok()) {
        ReleaseTensors(task);
        return Map(submit);
    }

    kv::TaskResult result;
    auto st = client_->Wait(taskId, wait_timeout_ms, result);   // 只 Wait 一次
    if (!st.ok() || result.status.code == TIMEOUT ||
        result.status.code == TASK_NOT_FOUND) {
        QuarantineTensors(task);
        return Map(st);
    }
    if (result.status.code == PARTIAL_FAILED) {
        ReleaseTensors(task);                           // 本块不返回数据
        return Map(result.status);
    }

    std::optional<ErrorCode> first_error;
    for (std::size_t i = 0; i < entries.size(); ++i) {
        if (!result.entryStatus[i].ok()) { first_error = Map(result.entryStatus[i]); continue; }
        auto record = DecodeRecord(tensorAddrs[i]);     // magic/version/flags/头 CRC
        if (!record) { first_error = record.error(); continue; }
        if (sliceList[i]->size > record->payload_size) { first_error = INVALID_PARAMS; continue; }
        if (sliceList[i]->size == record->payload_size && !VerifyPayloadCrc(record)) {
            first_error = CHECKSUM_MISMATCH; continue;
        }
        std::memcpy(sliceList[i]->ptr, record->payload, sliceList[i]->size);
    }
    ReleaseTensors(task);
    return first_error ? tl::make_unexpected(*first_error) : tl::expected<void, ErrorCode>{};
}
```
---

### 3.3 查：`IsExist`

#### 3.3.1 类图

```mermaid
classDiagram
    class FileStorage {
        +Exists(scoped_key)
    }
    class KvClientStorageBackend {
        +IsExist(scoped_key)
        -MakeBlockId(scoped_key) BlockId
        -MakeAsuKey(block) CacheKey
    }
    class KvClient {
        +QueryAsync(keys, taskId)
        +Wait(taskId, timeout, result)
    }
    FileStorage --> KvClientStorageBackend : IsExist
    KvClientStorageBackend --> KvClient : QueryAsync + Wait
```

#### 3.3.2 时序图

```mermaid
sequenceDiagram
    autonumber
    participant FS as FileStorage
    participant BE as Backend
    participant KC as kv::KvClient
    participant ASU as ASU

    FS->>BE: IsExist(scoped_key)
    BE->>BE: block = MakeBlockId(scoped_key)
    BE->>BE: key = MakeAsuKey(block)
    BE->>KC: QueryAsync({key}, taskId)
    KC-->>BE: taskId
    BE->>KC: Wait(taskId, query_timeout_ms)
    KC-->>BE: TaskResult{exists, entryStatus}
    alt exists == 1 && entryStatus == ENTRY_KEY_EXIST
        BE-->>FS: expected(true)
    else exists == 0 && entryStatus == ENTRY_KEY_NOT_EXIST
        BE-->>FS: expected(false)
    else IO_ERROR / CONNECTION_ERROR / TIMEOUT / 其他
        BE-->>FS: unexpected(mapped_error)
    end
```

#### 3.3.3 实现

```cpp
tl::expected<bool, ErrorCode> IsExist(const std::string& scoped_key) {
    if (!ready) return INTERNAL_ERROR;

    const auto block = MakeBlockId(scoped_key);
    const auto key = MakeAsuKey(block);

    kv::TaskId taskId{};
    auto submit = client_->QueryAsync({key}, taskId);
    if (!submit.ok()) return Map(submit);

    kv::TaskResult result;
    auto st = client_->Wait(taskId, query_timeout_ms, result);   // 只 Wait 一次
    if (!st.ok()) return Map(st);
    if (!result.queryResult || result.queryResult->exists.size() != 1) {
        return INTERNAL_ERROR;
    }

    const bool exists = result.queryResult->exists[0] != 0;
    const auto entry = result.entryStatus[0];
    if (exists && entry.code == ENTRY_KEY_EXIST) return true;
    if (!exists && entry.code == ENTRY_KEY_NOT_EXIST) return false;
    return tl::make_unexpected(Map(entry));   // 分片失败不能当“不存在”
}
```
---

### 3.4 删

#### 3.4.1 类图

```mermaid
classDiagram
    class Client {
        +Remove(key, force)
        +BatchRemove(keys, force)
    }
    class Master {
        +Remove(key, force)
        +BatchRemove(keys, force)
        -EraseMetadata()
        -EnqueueDelete(owner, scoped_keys)
    }
    class LocalSsdTaskMailbox {
        +EnqueueDelete(scoped_keys)
        +TakeDeleteTasks()
    }
    class FileStorage {
        +BatchRemoveKeys(scoped_keys)
    }
    class KvClientStorageBackend {
        +BatchRemoveKeys(scoped_keys)
        -MakeBlockId(scoped_key) BlockId
        -MakeAsuKey(block) CacheKey
    }
    class KvClient {
        +DeleteAsync(keys, taskId)
        +Wait(taskId, timeout, result)
    }
    Client --> Master : Remove / BatchRemove
    Master --> LocalSsdTaskMailbox : EnqueueDelete
    FileStorage ..> LocalSsdTaskMailbox : TakeDeleteTasks
    FileStorage --> KvClientStorageBackend : BatchRemoveKeys
    KvClientStorageBackend --> KvClient : DeleteAsync + Wait
```

#### 3.4.2 时序图

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant C as Client
    participant MS as Master
    participant MB as LocalSsdTaskMailbox
    participant FS as FileStorage
    participant BE as Backend
    participant KC as kv::KvClient
    participant TR as Transport
    participant ASU as ASU

    U->>C: Remove(key) / BatchRemove(keys)
    C->>MS: Remove / BatchRemove RPC
    MS->>MS: 捕获 LOCAL_DISK owner（在 EraseMetadata 之前）
    MS->>MS: EraseMetadata（删 Master 元数据）
    MS->>MB: EnqueueDelete(owner, scoped_keys)
    MS-->>C: per-key result

    FS->>MB: owner 心跳 TakeDeleteTasks()
    MB-->>FS: delete tasks
    FS->>BE: BatchRemoveKeys(scoped_keys)
    BE->>BE: scoped_key → BlockId → MakeAsuKey；按 254 切块

    loop 每个 chunk
        BE->>KC: DeleteAsync(keys, taskId)
        KC-->>BE: taskId
        KC->>TR: dispatch
        TR->>ASU: Delete(key)
        BE->>KC: Wait(taskId, timeout)
        KC-->>BE: entryStatus
        BE->>BE: per-entry：OK 或 CQE_KEY_NOT_EXIST → 成功；否则失败
    end
    BE-->>FS: per-key result
    Note over KC,TR: Delete 不需要 mrKey / 注册内存
```

#### 3.4.3 实现

```cpp
// Master::BatchRemove（删除按 key 已接入的形态）
//   owners = CollectLocalDiskOwners(metadata);   // 必须在 EraseMetadata 之前
//   EraseMetadata();
//   for (owner : owners) EnqueueDelete(owner, scoped_keys);

// owner 心跳拉取删除任务，调用 backend：
tl::expected<...> BatchRemoveKeys(scoped_keys) {
    for (const auto& scoped_key : scoped_keys) {
        const auto block = MakeBlockId(scoped_key);
        const auto key = MakeAsuKey(block);       // 与写/读/查同一套 key 编码
        keys.push_back(key);
    }

    for (chunk : Chunk(keys, 254)) {              // deleteIoNum 默认 254
        kv::TaskId taskId{};
        if (!client_->DeleteAsync(chunk, taskId).ok()) {
            // 提交失败：没有 task，本块 key 全部记失败
            continue;
        }
        kv::TaskResult result;
        client_->Wait(taskId, wait_timeout_ms, result);   // 每块只 Wait 一次

        for (i : chunk) {
            if (result.entryStatus[i].ok()) success;
            else if (whole_cqe == CQE_KEY_NOT_EXIST) success;  // 幂等
            else failed;   // 记日志，交给后续重试
        }
    }
}
```

#### 3.4.4 注意事项

**现状：按 key 删除到不了后端**

| 用户操作 | 实际动作 | 到 `StorageBackendInterface`？ | 证据 |
| --- | --- | --- | --- |
| `Client::Remove(key)` | 删 Master 元数据 + hot cache；物理删除调用被注释 | **否** | `client_service.cpp:3795-3812`，`3801-3803` 为注释 |
| `Client::BatchRemove(keys)` | Master 逐 key 删元数据 | **否** | `client_service.cpp:3848-3870`；`master_service.cpp:7437-7530` |
| `Client::RemoveByRegex` | Master 扫描删元数据 | **否** | `client_service.cpp:3814-3831` |
| `Client::RemoveAll` | Master 删元数据；客户端物理清空当前不可达 | **否（当前 HEAD）** | `file_storage.cpp:912-921` 零调用者；`RequestRemoveAll` 也无调用者 |
| `EvictAboveDiskWatermark` | 后端选 key，通知 Master 后物理删 | 是，但**不是用户按 key** | `storage_backend.h:323-328` |
| KvClient | `DeleteAsync(vector<CacheKey>)` | 是（若被调用） | `kv_client.h:50` |

- `Client::Remove` 里被注释的 `storage_backend_` 是客户端本地文件后端的 `std::shared_ptr<StorageBackend>`（`client_service.h:1178`），不是 `FileStorage` 持有的 `StorageBackendInterface`（`file_storage.h:291`）。即使取消注释也不会到达 offload 后端。
- `RemoveAll` 链路两端都是死的：`RequestRemoveAll()`、`FileStorage::RemoveAll()`、`Client::PollRemoveAll()` 均无调用者；RPC 还在，但标志位永远不会被置起。
- `KvClient` 没有 `RemoveAll`/drop/scan；只能按 8 字节 key 调 `DeleteAsync`。

**接口能否承载按 key 删除**

`StorageBackendInterface` 能到达后端的只有 `BatchOffload` / `BatchLoad` / `IsExist` / `GetObjectDataSize` / `RemoveAll` / `EvictAboveDiskWatermark`；没有任何方法能携带“要删除的 key 集合”。因此：

- **要按 key 物理删除，必须扩展后端契约并补 Master→owner 路由**；只加接口方法不接路由，方法永远没有调用者。
- **不改接口只能做逻辑删除**：`Remove` 后 ASU value 仍在，靠同 key 覆盖写 / ASU GC / 命名空间轮转回收。

| 方案 | 做法 | 按 key 物理删除 | 容量回收 | 改动面 | 结论 |
| --- | --- | --- | --- | --- | --- |
| A. 扩展基类接口 + 路由 | `StorageBackendInterface` 加 `BatchRemoveKeys`（默认 `NOT_SUPPORTED`）；Master 删除前捕获 owner、投递按 key 任务 | 是 | 立即 | 约 500–900 行 / 10 文件 | 需要容量回收时唯一正确方案 |
| B. 旁路能力接口 + 同样路由 | 不动基类，新增 `KeyDeleteCapable`，`FileStorage` `dynamic_cast` | 是 | 立即 | 同 A（路由成本不变） | 不想动公共基类时的备选 |
---

## 4. SFMEA 分析与测试设计

### 4.1 失效模式与影响

> 严重度：S1 数据损坏/误删；S2 功能不可用/永久停；S3 性能/配额退化。

| ID | 功能 | 失效模式 | 影响 | 严重度 | 检测/缓解 | 测试 |
| --- | --- | --- | --- | --- | --- | --- |
| FM-INIT-01 | Init | `KvClient::Init` / `RegisterKVCaches` 失败或返回空结果 | 节点不参与 offload | S2 | 失败回滚 `kUninit`；空结果当失败 | TC-INIT-01 |
| FM-ENTRY-01 | BuildKvBuffers | `FindPersistentHandle` 找不到 / handle 非注册返回值 | `BUFFER_NOT_REGISTERED` | S2 | 边界检查；只用注册区域 handle | TC-ENTRY-02 |
| FM-ENTRY-02 | Key | `MakeAsuKey` 全零或 64-bit 碰撞 | 写错对象/覆盖先写 | S1 | 全零重映射；碰撞作为已知风险接受 | TC-ENTRY-03 |
| FM-ENTRY-03 | BuildKvBuffers | `tensorSizes[0]` 超 24-bit 或未 512B 对齐 | SDK `INVALID_ARGUMENT` | S2 | 提交前校验；超限 key 跳过 | TC-ENTRY-04 |
| FM-WRITE-01 | 写 | `PARTIAL_FAILED` 下把 OK entry 当成功 | 假成功，Master 指向未发送数据 | S1 | 本块整体不发布 | TC-STORE-01 |
| FM-WRITE-02 | 写 | `Wait` 超时/`TASK_NOT_FOUND` 后复用 tensor | 数据串写 / use-after-free | S1 | tensor 区间隔离，不复用 | TC-STORE-01 |
| FM-WRITE-03 | 写/配额 | 返回 `KEYS_ULTRA_LIMIT` 或硬错误 | offload 永久关闭 / 心跳中断 | S2 | 逐 key 跳过，返回 `expected(count)` | TC-STORE-02 |
| FM-WRITE-04 | 写 | `complete_handler` 失败 | 数据在 ASU、Master 未登记 | S2 | 返回错误、记日志；重试 Put 覆盖 | TC-STORE-03 |
| FM-READ-01 | 读 | 前缀读被拒绝 / `slice.size > payload_size` | ranged read 失败或超读 | S2 | 允许 `slice.size <= payload_size` | TC-LOAD-02 |
| FM-READ-02 | 读 | 头 / payload CRC 损坏 | 返回错误数据 | S1 | 头 CRC + payload CRC | TC-LOAD-03 |
| FM-QUERY-01 | 查 | `IO_ERROR`/超时返回 false | probe 误删 Master 元数据 | S1 | 一律 `unexpected`，绝不返回 false | TC-QUERY-01 |
| FM-DEL-01 | 删 | V1 逻辑删除不释放 ASU | 容量残留 | S2（接受） | 明确对外口径；覆盖/GC/namespace 轮转 | TC-DEL-03 |
| FM-DEL-02 | 删 | owner 路由错误或 `DeleteAsync` 失败 | 漏删/删错/残留 | S1/S3 | Master→owner；逐 key 日志；幂等重试 | TC-DEL-01/02 |
| FM-CONC-01 | 并发 | `Wait` 持锁 / tensor 未完成就复用 | 心跳阻塞 / 数据损坏 | S1/S2 | `Wait` 不持锁；超时隔离 | TC-CONC-01 |

### 4.2 测试用例与验收点

| 编号 | 场景 | 注入/前置 | 验收点 |
| --- | --- | --- | --- |
| TC-INIT-01 | Init 注册失败 / 空结果 | 注入注册失败或空 vector | 回滚 `kUninit`、不进入 `kReady`；FileStorage Init 失败 |
| TC-ENTRY-01 | Store entry 构造 | 正常注册区域 | `mrKey` 非空且等于 `tokenId`；`region.size` 512B 对齐；handle 来自 `FindPersistentHandle` |
| TC-ENTRY-02 | 伪造 / 未注册 handle | 伪造 `buffer.handle` | `BUFFER_NOT_REGISTERED`；不发布；tensor 释放 |
| TC-ENTRY-03 | Key 编码稳定性 | 同一 `scoped_key` 写/读/查 | 得到同一非零 `MakeAsuKey(BlockId)`；不同 namespace 不同 |
| TC-ENTRY-04 | 超 24-bit / 未对齐 | `payload_size` 超限 | 该 key 跳过、不提交；其他 key 正常 |
| TC-STORE-01 | OK / `PARTIAL_FAILED` / 超时 | 注入部分失败/延迟 | OK 只发布成功 entry；PARTIAL 不发布；超时 tensor 隔离 |
| TC-STORE-02 | 超配额 | 计数超 limit | 超限 key 跳过；返回 `expected(count)`；不返回 `KEYS_ULTRA_LIMIT` |
| TC-STORE-03 | `complete_handler` 失败 | handler 返回错误 | 数据仍可读；返回错误；不回滚 |
| TC-LOAD-01 | 全长读 + CRC | 写入后按全长读 | 字节一致；head/payload CRC 通过 |
| TC-LOAD-02 | 前缀读 | `slice.size < payload_size` | 返回前缀；不越界 |
| TC-LOAD-03 | 数据损坏 | 篡改头 / payload | `INVALID_READ` / `CHECKSUM_MISMATCH`；不拷贝 |
| TC-QUERY-01 | 查询错误 | `exists=0` + `IO_ERROR` / 超时 | 返回 `unexpected`；绝不返回 false |
| TC-DEL-01 | 按 key 删除 | 多 client，非 owner 调用 | master 通知 owner；owner 删除；254 分块；`CQE_KEY_NOT_EXIST` 成功 |
| TC-DEL-02 | 删除失败 / 重复删除 | 注入 `ENTRY_DELETE_FAILED` | 失败有日志；重复删除幂等；计数只扣成功 key |
| TC-DEL-03 | V1 逻辑删除（若选 B） | `Client::Remove` 后检查 ASU | ASU value 仍在；行为与文档一致；再次 Put 覆盖 |
| TC-CONC-01 | 多读并发 + 写 | 多线程读写 | TSAN/ASAN 干净；tensor 区间全部归还 |

---

## 5. 风险与待确认

| # | 风险 | 影响 | 处置 |
| --- | --- | --- | --- |
| R1 | 删除方案未拍板 | V1 范围可能膨胀到跨层协议改造 | 评审先定 B 或 A |
| R2 | 前缀读的 payload CRC 无法覆盖完整 value | 完整性拦截面收窄 | V1 接受并文档化；或用本地长度索引做全长读（多 I/O） |
| R3 | 默认 16 MiB 超过 24-bit 传输上限 | 满尺寸写全部失败 | 默认改 2 MiB，硬上限 `16,776,680` |
| R4 | 64-bit `MakeAsuKey(BlockId)` 哈希碰撞 | 后写覆盖先写，读错对象 | 默认假设不发生；P1 可加碰撞检测/版本化 key |
| R5 | `Wait` 超时后 tensor 区间永久隔离 | 反复超时会耗尽注册区域 | 监控隔离区间数量；P1 研究取消/回收 |
| R6 | 子区间 MR 寻址未被真机验证 | 可能只能每个 tensor 区间单独注册 | 真机确认；不支持则退化注册 |
| R7 | `ScanMeta` 不实现 | Master 丢元数据后无法重建账本 | 已明确不在本版；文档化 |
| R8 | HEAD 漂移 / `RemoveAll` 死代码 | 行号与可达性变化 | 评审前重新核对；不要引用旧行号 |
| R9 | ASU 同 key 覆盖写语义未验证 | 若 Store 对已存在 key 返回 `CQE_KEY_ALREADY_EXISTED`，Put 覆盖会失败，B 方案的“覆盖回收”也不成立 | 真机验证；必要时先 `DeleteAsync` 再 `StoreAsync`（代价是一次额外删除） |
