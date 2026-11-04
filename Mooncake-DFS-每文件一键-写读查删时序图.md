# Mooncake DFS「每文件一键」变体：写、读、查、删时序图

> 变体前提：一个 shard 文件 = 一个 8B key，`file_key = XXH64(namespace, normalized(file_path))`；
> 读写用 `Store/Retrieve(key, buffer, offset, length)` 做文件内 offset 寻址。
> 代码基线：Mooncake `main` @ `522b716e`；ASU SDK `kv_semantics` 的 `kv_client.h` / `kv_types.h`。

---

## 1. 写

```mermaid
sequenceDiagram
    autonumber
    participant C as Client(Put)
    participant MS as MasterService
    participant DA as DfsAllocator
    participant BE as DistributedStorageBackend
    participant AD as AsuKvFsAdapter
    participant SL as SlotPool
    participant KC as kv::KvClient
    participant ASU as ASU

    C->>MS: PutStart(key, slice_lengths, cfg)
    MS->>DA: Allocate(key, value_length)
    DA-->>MS: descriptor{file_path, offset, object_size, aligned_size}
    MS-->>C: replicas（DFS PROCESSING）

    C->>C: 先写 MEMORY / NoF
    C->>BE: WriteDfsReplicas → BatchWrite(requests)
    BE->>AD: BatchWriteAt(requests)
    AD->>AD: file_key = H(ns, normalized(file_path))
    AD->>AD: 按 max_io_bytes 切段<br/>同一 file_key，offset += done
    AD->>SL: Acquire(n)
    AD->>AD: gather iov → slot，尾部补零到 512B
    AD->>KC: BatchStoreAsync(entries)
    Note over KC: 所有 entry.key 相同<br/>entry.offset = descriptor.offset + done<br/>entry.buffer.handle = arena handle
    KC->>ASU: Store(file_key, slot, aligned_len, file_offset)
    KC-->>AD: taskId
    AD->>KC: Wait(taskId)
    KC-->>AD: TaskResult{status, entryStatus}

    alt OK / PARTIAL_FAILED
        AD->>SL: Release(slots)
    else TIMEOUT / TASK_NOT_FOUND
        AD->>SL: Poison(slots)
    end
    AD-->>BE: 每个 request 的 expected
    BE-->>C: 每个 request 的 expected

    alt 全部成功
        C->>MS: PutEnd(ALL)
        MS->>MS: DFS replica PROCESSING → COMPLETE
    else 任一失败
        C->>MS: PutRevoke(ALL)
        MS->>DA: Free(key, descriptor)
    end
```

约束：

- 单次 I/O ≤ 16,776,704B（24-bit、512B 对齐）；
- `descriptor.offset + done` 必须 < 2^32，所以 `shard_capacity ≤ 4 GiB`；
- 同一 key 的多个 entry 是否被设备接受、是否支持任意 offset 写/自动扩展，取决于 H1/H1b 真机验证。

## 2. 读

```mermaid
sequenceDiagram
    autonumber
    participant RC as RealClient
    participant C as Client
    participant MS as MasterService
    participant DA as DfsAllocator
    participant BE as DistributedStorageBackend
    participant AD as AsuKvFsAdapter
    participant SL as SlotPool
    participant KC as kv::KvClient
    participant ASU as ASU

    RC->>C: Query(key)
    C->>MS: GetReplicaList
    MS->>DA: UpdateAccess(key, desc)
    MS-->>C: replicas（含 DFS descriptor）
    C-->>RC: QueryResult

    RC->>RC: SelectBestReplica / FilterQueryResult
    RC->>C: Get(key, filtered_qr, slices)
    C->>C: ReadDfsReplica
    C->>BE: BatchRead(requests)
    BE->>AD: BatchReadAt(requests)
    AD->>AD: file_key = H(ns, normalized(file_path))
    AD->>AD: 按 max_io_bytes 切段<br/>同一 file_key，offset += done
    AD->>SL: Acquire(n)
    AD->>KC: BatchLoadAsync(entries)
    KC->>ASU: Retrieve(file_key, slot, aligned_len, file_offset)
    KC-->>AD: TaskResult
    AD->>AD: 每段只把真实字节 Scatter 回 iov
    AD-->>BE: 每个 request 的 expected
    BE-->>C: 每个 request 的 expected
    C-->>RC: OK / 错误
    RC->>RC: VerifyObjectChecksum（可选）
```

同一对象的多段读共享同一个 `file_key`，只是 `offset` 不同；读路径不关心文件里其它对象。

## 3. 查

```mermaid
sequenceDiagram
    autonumber
    participant RC as RealClient
    participant C as Client
    participant MS as MasterService
    participant AD as AsuKvFsAdapter
    participant KC as kv::KvClient
    participant ASU as ASU

    alt 对象级 Exist（当前语义，只查 Master 元数据）
        RC->>C: IsExist(key)
        C->>MS: ExistKey
        MS->>MS: ExistKeyImpl → ReadExistence
        MS-->>C: true / false
        C-->>RC: expected<bool>
        Note over MS: 不访问 ASU，也不看文件
    else 文件级 Exist（新增探针，需要 descriptor）
        C->>MS: Query(key)
        MS-->>C: descriptor{file_path, offset, object_size}
        C->>AD: IsExistFile(descriptor)
        AD->>AD: file_key = H(ns, normalized(file_path))
        AD->>KC: QueryAsync([file_key])
        KC->>ASU: Exist(file_key)
        KC-->>AD: exists[0], entryStatus[0]
        alt exists=1 且 ENTRY_KEY_EXIST
            AD-->>C: true
        else exists=0 且 ENTRY_KEY_NOT_EXIST
            AD-->>C: false
        else 分片失败 / IO_ERROR / TIMEOUT
            AD-->>C: unexpected
        end
    end

    Note over AD,ASU: Query 只回答“这个 shard 文件还在不在”<br/>不能回答“某个对象还在不在”
```

注意：Query 的分片失败会让 `exists` 保持 0，所以不能只看 `exists == 0` 判“不存在”，必须同时看任务状态和 `entryStatus`。

## 4. 删

```mermaid
sequenceDiagram
    autonumber
    participant RC as RealClient
    participant C as Client
    participant MS as MasterService
    participant DA as DfsAllocator
    participant AD as AsuKvFsAdapter
    participant KC as kv::KvClient
    participant ASU as ASU

    alt 按对象删除（当前 DFS 语义，不动 ASU 数据）
        RC->>C: Remove(key)
        C->>MS: Remove
        MS->>MS: RemoveObject → CheckRemovable
        MS->>MS: EraseMetadata → FreeDfsReplicas
        MS->>DA: Free(key, descriptor)
        DA->>DA: 从 LRU / offset_to_handle 摘除<br/>QueuePendingFree（默认 30s）
        MS-->>C: OK
        C-->>RC: OK
        Note over MS,ASU: 不调用 DeleteAsync，也不删 ASU 数据
    else 按文件物理删除（只能用于退役 shard / 空文件）
        RC->>AD: DeleteFileKey(file_path)
        AD->>AD: file_key = H(ns, normalized(file_path))
        AD->>KC: DeleteAsync([file_key])
        KC->>ASU: Delete(file_key)
        KC-->>AD: TaskResult{entryStatus}
        AD->>AD: CQE_KEY_NOT_EXIST 视为成功（幂等）
        AD-->>RC: OK
        Note over DA,ASU: 整个 shard 的所有对象一起消失<br/>allocator 里仍存活的 extent 会变成悬空账本
    end
```

文件级删除不能接到按对象的 `Remove` 上：一个 key 覆盖整个 shard，删掉它会让同一 shard 内其它仍然存活的对象一起不可读；同时 `ShardAllocator::offset_to_handle` / LRU 仍然认为这些 extent 有效，必须先把 shard 退役并重置 allocator 状态。这个变体能给出“文件是否还在”的答案，但**按对象删除/Exist 仍然做不到**。
