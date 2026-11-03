# Mooncake DFS 当前实现：写、读、查、删、ScanMeta 的类图与时序图

> 代码基线：Mooncake `main` @ `522b716e`。图中 `文件:行号` 为当前 HEAD 核对结果。

---

## 1. 写

### 1.1 类图

```mermaid
classDiagram
    class RealClient {
        +put_internal(key, value, config)
    }
    class Client {
        +Put(key, slices, config)
        +WriteDfsReplicas(keys, slice_lists, descriptors)
        +BatchPutEnd(object_metas, replica_type)
        +BatchPutRevoke(keys, replica_type)
    }
    class MasterClient {
        +PutStart(key, slice_lengths, config)
        +PutEnd(key, replica_type)
        +PutRevoke(key, replica_type)
    }
    class MasterService {
        +PutStart(client_id, key, slice_lengths, config)
        +PutEnd(client_id, key, replica_type)
        +PutRevoke(client_id, key, tenant_id, replica_type)
        -AllocateReplicas(key, value_length, config, ...)
        -AllocateAndInsertMetadata(...)
        -InsertMetadata(...)
    }
    class DfsAllocatorInterface {
        <<interface>>
        +Allocate(key, size) DistributedFSDescriptor
        +Free(key, descriptor)
    }
    class ShardAllocator {
        +Allocate(key, size)
        +Free(key, descriptor)
    }
    class ImmutableBucketAllocator {
        +Allocate(key, size)
        +Free(key, descriptor)
    }
    class DistributedFSDescriptor {
        +file_path string
        +offset uint64
        +object_size uint64
        +aligned_size uint64
        +shard_idx int
    }
    class DistributedStorageBackend {
        +BatchWrite(requests) results
        -GetOrOpenShard(descriptor) ShardFile
        -ValidateAndOpenShardFile(path) int
        -OpenBucket(descriptor) int
        -ExecuteSingleIo(op, write)
        -ExecuteShardBatchIo(ops, results, write)
        -ExecuteBucketBatchIo(ops, results, write)
        -SubmitBatchIo(io_requests, io_ops, results, write, complete_short_io)
    }
    class ShardFile {
        +path string
        +fd int
        +mutex
    }
    class DfsIoOp {
        +index size_t
        +key string
        +descriptor DistributedFSDescriptor
        +iovs vector~iovec~
        +size uint64
    }
    class FileSystemAdapter {
        <<interface>>
        +GetFileSize(path) size_t
        +OpenFile(path) int
        +OpenExistingFile(path) int
        +CloseFile(fd)
        +WriteAt(fd, iov, iovcnt, offset) size_t
        +BatchWriteAt(requests) results
        +SupportsBatchIo() bool
        +SupportsFdReopen() bool
    }
    class PosixFsAdapter {
        +WriteAt(fd, iov, iovcnt, offset)
    }
    class Hf3fsAdapter {
        +WriteAt(fd, iov, iovcnt, offset)
        +SupportsBatchIo() true
    }

    RealClient --> Client : Put
    Client --> MasterClient : PutStart / PutEnd / PutRevoke
    MasterClient --> MasterService : RPC
    MasterService --> DfsAllocatorInterface : Allocate / Free
    DfsAllocatorInterface <|-- ShardAllocator
    DfsAllocatorInterface <|-- ImmutableBucketAllocator
    ShardAllocator ..> DistributedFSDescriptor : 生成
    ImmutableBucketAllocator ..> DistributedFSDescriptor : 生成
    Client --> DistributedStorageBackend : WriteDfsReplicas
    DistributedStorageBackend *-- ShardFile
    DistributedStorageBackend *-- DfsIoOp
    DistributedStorageBackend --> FileSystemAdapter : WriteAt / BatchWriteAt
    FileSystemAdapter <|-- PosixFsAdapter
    FileSystemAdapter <|-- Hf3fsAdapter
```

### 1.2 时序图

```mermaid
sequenceDiagram
    autonumber
    participant RC as RealClient
    participant C as Client
    participant MC as MasterClient
    participant MS as MasterService
    participant DA as DfsAllocator
    participant BE as DistributedStorageBackend
    participant AD as FileSystemAdapter
    participant DFS as DFS(POSIX/3FS)

    RC->>C: Put(key, slices, config)
    C->>C: ComputeObjectChecksumForSlices（可选）
    C->>MC: PutStart(key, slice_lengths, cfg)
    MC->>MS: PutStart
    MS->>MS: AllocateAndInsertMetadata
    MS->>MS: AllocateReplicas
    MS->>DA: Allocate(key, value_length)
    DA-->>MS: DistributedFSDescriptor{file_path, offset, object_size, aligned_size, shard_idx}
    MS-->>MC: replicas
    MC-->>C: replicas（DFS 状态 PROCESSING）

    C->>C: 先写 MEMORY / NoF（TransferWrite）
    Note over C: DFS 写只在非 DFS 副本全部成功后执行（NonDfsTransfersSucceeded）
    C->>C: WriteDfsReplicas；device slice 先 D2H 到 PinnedBufferPool
    C->>BE: BatchWrite(DfsWriteRequest[])
    BE->>BE: IsDfsDescriptorRangeValid / IsBucketDescriptorRangeValid
    BE->>BE: 汇总 iov；Σiov 必须等于 object_size

    alt SupportsBatchIo() == true
        BE->>BE: ExecuteShardBatchIo / ExecuteBucketBatchIo
        loop 每个 request
            alt shard
                BE->>BE: GetOrOpenShard(desc)
                Note over BE,AD: 首次/重开：GetFileSize → canonical → OpenFile
            else bucket
                BE->>BE: OpenBucket(desc)
                Note over BE,AD: GetFileSize → canonical → OpenExistingFile
            end
        end
        BE->>AD: BatchWriteAt(FdIoRequest[])
        AD->>DFS: pwritev / USRBIO write
        opt 短写
            alt bucket batch
                BE->>BE: TransferAll 续写（短写循环）
                BE->>AD: WriteAt(fd, iov, iovcnt, offset)
            else shard batch
                BE->>BE: 短写按 FILE_WRITE_FAIL
            end
        end
        opt bucket batch
            BE->>AD: CloseFile(fd)
            Note over BE: close 失败会把该桶的请求改为失败
        end
    else SupportsBatchIo() == false
        loop 每个 request
            alt shard
                BE->>BE: GetOrOpenShard(desc)
                BE->>AD: WriteAt(fd, iov, iovcnt, offset)
                AD->>DFS: pwritev
                opt 失败且 SupportsFdReopen()
                    BE->>AD: CloseFile → 重新 Open → 重试一次
                end
            else bucket
                BE->>BE: OpenBucket(desc)
                BE->>BE: TransferAll(...)（短写循环）
                BE->>AD: WriteAt(fd, iov, iovcnt, offset)
                AD->>DFS: pwritev
                BE->>AD: CloseFile(fd)
            end
        end
    end
    BE-->>C: 每个 request 的 expected<void>

    alt 所有已分配副本成功
        C->>MC: PutEnd(ALL)
        MC->>MS: PutEnd
        MS->>MS: DFS replica PROCESSING → COMPLETE
        MS-->>MC: OK
        MC-->>C: OK
    else 任一副本失败
        C->>MC: PutRevoke(ALL)
        MC->>MS: PutRevoke
        MS->>DA: Free(key, descriptor)
        MS-->>MC: OK
        MC-->>C: OK
    end
```

---

## 2. 读

### 2.1 类图

```mermaid
classDiagram
    class RealClient {
        +get_buffer_internal(key, allocator)
        +batch_get_buffer_internal(keys, allocator)
    }
    class ReplicaSelection {
        +SelectBestReplica(replicas, local_endpoints)
        +FilterQueryResult(qr, replica)
    }
    class Client {
        +Query(key) QueryResult
        +BatchQuery(keys) results
        +Get(key, query_result, slices)
        +BatchGet(keys, query_results, slices)
        +ReadDfsReplica(key, replica_descriptor, slices)
    }
    class MasterClient {
        +GetReplicaList(key)
        +BatchGetReplicaList(keys, tenant_id)
    }
    class MasterService {
        +GetReplicaList(key, tenant_id)
        -ReadReplicaList(tenant_id, key, metadata, follow_ups)
    }
    class Replica {
        +is_dfs_replica() bool
        +get_dfs_descriptor() DistributedFSDescriptor
    }
    class DistributedFSDescriptor {
        +file_path string
        +offset uint64
        +object_size uint64
        +aligned_size uint64
        +shard_idx int
    }
    class DistributedStorageBackend {
        +BatchRead(requests) results
        -GetOrOpenShard(descriptor) ShardFile
        -OpenBucket(descriptor) int
        -ExecuteSingleIo(op, write)
        -ExecuteShardBatchIo(ops, results, write)
        -ExecuteBucketBatchIo(ops, results, write)
        -SubmitBatchIo(io_requests, io_ops, results, write, complete_short_io)
    }
    class ShardFile {
        +path string
        +fd int
        +mutex
    }
    class DfsIoOp {
        +index size_t
        +key string
        +descriptor DistributedFSDescriptor
        +iovs vector~iovec~
        +size uint64
    }
    class FileSystemAdapter {
        <<interface>>
        +GetFileSize(path) size_t
        +OpenFile(path) int
        +OpenExistingFile(path) int
        +CloseFile(fd)
        +ReadAt(fd, iov, iovcnt, offset) size_t
        +BatchReadAt(requests) results
        +SupportsBatchIo() bool
        +SupportsFdReopen() bool
    }
    class PosixFsAdapter {
        +ReadAt(fd, iov, iovcnt, offset)
    }
    class Hf3fsAdapter {
        +ReadAt(fd, iov, iovcnt, offset)
        +SupportsBatchIo() true
    }

    RealClient --> ReplicaSelection : SelectBestReplica
    RealClient --> Client : Get / BatchGet
    Client --> MasterClient : GetReplicaList / BatchGetReplicaList
    MasterClient --> MasterService : RPC
    MasterService ..> Replica : 读取
    Replica *-- DistributedFSDescriptor
    Client --> DistributedStorageBackend : ReadDfsReplica
    DistributedStorageBackend *-- ShardFile
    DistributedStorageBackend *-- DfsIoOp
    DistributedStorageBackend --> FileSystemAdapter : ReadAt / BatchReadAt
    FileSystemAdapter <|-- PosixFsAdapter
    FileSystemAdapter <|-- Hf3fsAdapter
```

### 2.2 时序图

```mermaid
sequenceDiagram
    autonumber
    participant RC as RealClient
    participant RS as ReplicaSelection
    participant C as Client
    participant MC as MasterClient
    participant MS as MasterService
    participant BE as DistributedStorageBackend
    participant AD as FileSystemAdapter
    participant DFS as DFS(POSIX/3FS)

    RC->>C: Query(key) / BatchQuery(keys)
    C->>MC: GetReplicaList / BatchGetReplicaList
    MC->>MS: GetReplicaList / BatchGetReplicaList
    MS->>MS: ReadReplicaList → GetReadableReplicaDescriptors
    opt 存在 DFS replica
        MS->>MS: dfs_allocator_->UpdateAccess(key, desc)
    end
    MS-->>MC: replicas
    MC-->>C: replicas（含 DistributedFSDescriptor）
    C-->>RC: QueryResult

    RC->>RS: SelectBestReplica(replicas, local_endpoints)
    RS-->>RC: best replica
    RC->>RS: FilterQueryResult(qr, replica)
    RS-->>RC: 只保留该副本的 QueryResult

    RC->>C: Get(key, filtered_qr, slices)
    C->>C: replica.is_dfs_replica()
    C->>C: ReadDfsReplica(key, desc, slices)
    C->>BE: BatchRead(DfsReadRequest[])
    BE->>BE: range check；按 object_size 裁剪 slices

    alt SupportsBatchIo() == true
        BE->>BE: ExecuteShardBatchIo / ExecuteBucketBatchIo
        loop 每个 request
            alt shard
                BE->>BE: GetOrOpenShard(desc)
                Note over BE,AD: 首次/重开：GetFileSize → canonical → OpenFile
            else bucket
                BE->>BE: OpenBucket(desc)
                Note over BE,AD: GetFileSize → canonical → OpenExistingFile
            end
        end
        BE->>AD: BatchReadAt(FdIoRequest[])
        AD->>DFS: preadv / USRBIO read
        opt 短读
            alt bucket batch
                BE->>BE: TransferAll 续读（短读循环）
                BE->>AD: ReadAt(fd, iov, iovcnt, offset)
            else shard batch
                BE->>BE: 短读按 FILE_READ_FAIL
            end
        end
        opt bucket batch
            BE->>AD: CloseFile(fd)
            Note over BE: close 失败会把该桶的请求改为失败
        end
    else SupportsBatchIo() == false
        loop 每个 request
            alt shard
                BE->>BE: GetOrOpenShard(desc)
                BE->>AD: ReadAt(fd, iov, iovcnt, offset)
                AD->>DFS: preadv
                opt 失败且 SupportsFdReopen()
                    BE->>AD: CloseFile → 重新 Open → 重试一次
                end
            else bucket
                BE->>BE: OpenBucket(desc)
                BE->>BE: TransferAll(...)（短读循环）
                BE->>AD: ReadAt(fd, iov, iovcnt, offset)
                AD->>DFS: preadv
                BE->>AD: CloseFile(fd)
            end
        end
    end
    BE-->>C: 每个 request 的 expected<void>
    C-->>RC: OK / 错误
    RC->>RC: VerifyObjectChecksum（MOONCAKE_STORE_CHECKSUM 开启时）
```

---

## 3. 查

### 3.1 类图

```mermaid
classDiagram
    class RealClient {
        +isExist(key) bool
        +batchIsExist(keys) results
    }
    class Client {
        +IsExist(key) bool
        +BatchIsExist(keys) results
        +Query(key) QueryResult
        +BatchQuery(keys) results
    }
    class MasterClient {
        +ExistKey(key) bool
        +BatchExistKey(keys) results
        +GetReplicaList(key)
        +BatchGetReplicaList(keys, tenant_id)
    }
    class MasterService {
        +ExistKey(key, tenant_id)
        +BatchExistKey(keys, tenant_id)
        -ExistKeyImpl(key, tenant_id, grant_lease)
        -ReadExistence(metadata, grant_lease) bool
        -HasReadableReplica(metadata) bool
        +GetReplicaList(key, tenant_id)
        -ReadReplicaList(tenant_id, key, metadata, follow_ups)
        -GetReadableReplicaDescriptors(metadata) replicas
    }
    class MetadataAccessorRO {
        +Exists() bool
        +Get() ObjectMetadata
    }
    class ObjectMetadata {
        +GetAllReplicas() replicas
        +IsValid() bool
    }
    class Replica {
        +is_dfs_replica() bool
        +get_dfs_descriptor() DistributedFSDescriptor
    }
    class DfsAllocatorInterface {
        <<interface>>
        +UpdateAccess(key, descriptor)
    }
    class DistributedStorageBackend {
        +IsExist(key) bool
        +IsEnableOffloading() bool
    }

    RealClient --> Client
    Client --> MasterClient
    MasterClient --> MasterService : RPC
    MasterService --> MetadataAccessorRO
    MetadataAccessorRO --> ObjectMetadata
    ObjectMetadata *-- Replica
    MasterService ..> DfsAllocatorInterface : DFS UpdateAccess
    Client ..> DistributedStorageBackend : DFS 模式不调用 IsExist
```

### 3.2 时序图

```mermaid
sequenceDiagram
    autonumber
    participant RC as RealClient
    participant C as Client
    participant MC as MasterClient
    participant MS as MasterService
    participant DA as DfsAllocator
    participant BE as DistributedStorageBackend

    alt 存在性：IsExist / BatchIsExist
        RC->>C: IsExist(key)
        C->>MC: ExistKey(key)
        MC->>MS: ExistKey
        MS->>MS: ExistKeyImpl → MetadataAccessorRO.Exists()
        alt 对象不存在
            MS-->>MC: false
            MC-->>C: false
        else 对象存在
            MS->>MS: ReadExistence → HasReadableReplica
            MS-->>MC: true / false
            MC-->>C: true / false
        end
        C-->>RC: expected<bool>
    else 元数据查询：Query / BatchQuery
        RC->>C: Query(key) / BatchQuery(keys)
        C->>MC: GetReplicaList / BatchGetReplicaList
        MC->>MS: GetReplicaList
        MS->>MS: ReadReplicaList → GetReadableReplicaDescriptors
        opt 存在 DFS replica
            MS->>DA: UpdateAccess(key, desc)
        end
        MS-->>MC: replicas
        MC-->>C: replicas（含 DFS descriptor）
        C-->>RC: QueryResult
    end

    opt 后端探针（DFS 模式不在这条链路上）
        C->>BE: IsExist(key)
        BE-->>C: unexpected(NOT_SUPPORTED)
    end
```

---

## 4. 删

### 4.1 类图

```mermaid
classDiagram
    class RealClient {
        +remove(key, force)
        +removeAll(force)
    }
    class Client {
        +Remove(key, force)
        +BatchRemove(keys, force)
        +RemoveAll(force)
    }
    class MasterClient {
        +Remove(key, force)
        +BatchRemove(keys, force)
        +RemoveAll(force)
    }
    class MasterService {
        +Remove(key, tenant_id, force)
        -RemoveObject(key, tenant_id, force, allow_managed_weight)
        +RemoveAll(force)
        -RemoveTenantObjects(tenant_id, force, now, matches_key)
        -EraseMetadata(ns, hold, erase_mode, previous_media_hint)
        -FreeDfsReplicas(key, replicas)
        -RunBucketDfsEvictionInternal(force_one)
    }
    class MetadataAccessorRW {
        +Erase(erase_mode, previous_media_hint)
    }
    class ObjectMetadata {
        +GetAllReplicas() replicas
    }
    class Replica {
        +is_dfs_replica() bool
        +get_dfs_descriptor() DistributedFSDescriptor
    }
    class DfsAllocatorInterface {
        <<interface>>
        +Free(key, descriptor)
        +PrepareEviction() PendingEviction
        +DeleteEvictedBucket(bucket)
    }
    class ShardAllocator {
        +Free(key, descriptor)
        -QueuePendingFree(shard, handle, bytes, when)
    }
    class ImmutableBucketAllocator {
        +Free(key, descriptor)
        +PrepareEviction() PendingEviction
        +DeleteEvictedBucket(bucket)
        -TombstoneLocked(key, bucket, entry)
    }
    class FileSystemAdapter {
        <<interface>>
        +DeleteFile(path)
    }

    RealClient --> Client
    Client --> MasterClient
    MasterClient --> MasterService : RPC
    MasterService --> MetadataAccessorRW : RemoveObject
    MetadataAccessorRW --> ObjectMetadata
    ObjectMetadata *-- Replica
    MasterService --> DfsAllocatorInterface : FreeDfsReplicas
    DfsAllocatorInterface <|-- ShardAllocator
    DfsAllocatorInterface <|-- ImmutableBucketAllocator
    ImmutableBucketAllocator --> FileSystemAdapter : DeleteEvictedBucket
```

### 4.2 时序图

```mermaid
sequenceDiagram
    autonumber
    participant RC as RealClient
    participant C as Client
    participant MC as MasterClient
    participant MS as MasterService
    participant RW as MetadataAccessorRW
    participant DA as DfsAllocator
    participant AD as FileSystemAdapter
    participant DFS as DFS(POSIX/3FS)

    alt 按对象删除：Remove / BatchRemove
        RC->>C: Remove(key, force)
        C->>MC: Remove / BatchRemove
        MC->>MS: Remove / BatchRemove
        MS->>MS: RemoveObject → CheckRemovable
        MS->>RW: Erase(EraseMode::kFull)
        RW->>MS: EraseMetadata
        MS->>MS: FreeDfsReplicas
        MS->>DA: Free(key, descriptor)
        alt shard
            DA->>DA: ShardAllocator::Free
            DA->>DA: QueuePendingFree / 从 LRU 摘除
        else bucket
            DA->>DA: ImmutableBucketAllocator::Free → TombstoneLocked
        end
        Note over MS,AD: 按对象删除不调用 FileSystemAdapter::DeleteFile
        MS-->>MC: OK
        MC-->>C: OK
        C-->>RC: OK
    else RemoveAll
        RC->>C: RemoveAll(force)
        C->>MC: RemoveAll
        MC->>MS: RemoveAll
        MS->>MS: RemoveTenantObjects
        loop 每个对象
            MS->>RW: Erase → EraseMetadata → FreeDfsReplicas
            MS->>DA: Free(key, descriptor)
        end
        MS-->>MC: removed_count
        MC-->>C: removed_count
        C-->>RC: removed_count
    end

    opt bucket 整桶淘汰（Master 内部 DFS 淘汰）
        MS->>DA: RunBucketDfsEvictionInternal → PrepareEviction
        DA-->>MS: EvictedBucket
        MS->>DA: DeleteEvictedBucket
        DA->>AD: DeleteFile(bucket_XXXXXX.data)
        AD->>DFS: unlink
    end
```

---

## 5. ScanMeta

### 5.1 类图

```mermaid
classDiagram
    class FileStorage {
        +Init()
        +ReRegisterOffloadedObjects()
    }
    class StorageBackendInterface {
        <<interface>>
        +ScanMeta(handler)
        +ResetScanIterator()
    }
    class DistributedStorageBackend {
        +ScanMeta(handler)
        +ResetScanIterator()
        -UsesObjectStorage() bool
    }
    class ObjectStorageAdapter {
        <<interface>>
        +ListKeys() KeyInfo[]
    }
    class FileSystemAdapter {
        <<interface>>
        +ListFiles(dir) names
    }
    class ScanHandler {
        +operator()(keys, metadatas) ErrorCode
    }

    FileStorage --> StorageBackendInterface : storage_backend_
    StorageBackendInterface <|-- DistributedStorageBackend
    DistributedStorageBackend --> ObjectStorageAdapter : 对象存储模式
    DistributedStorageBackend ..> FileSystemAdapter : DFS 模式不使用
    FileStorage ..> ScanHandler : 传入回调
```

### 5.2 时序图

```mermaid
sequenceDiagram
    autonumber
    participant FS as FileStorage
    participant BE as DistributedStorageBackend
    participant OS as ObjectStorageAdapter
    participant MS as Master

    alt DFS 模式
        Note over FS: Init 在 enable_dfs 时提前返回（file_storage.cpp:142-148），不调用 ScanMeta
        Note over FS: Heartbeat 线程不启动，ReRegisterOffloadedObjects 也不会被触发
        opt 若被外部或未来入口调用
            FS->>BE: ScanMeta(handler)
            BE->>BE: !UsesObjectStorage() → NOT_SUPPORTED
            BE-->>FS: unexpected(NOT_SUPPORTED)
            FS->>FS: enable_dfs + NOT_SUPPORTED → 记录 INFO 并返回成功
        end
    else 对象存储模式（OSS/S3，同一函数的另一分支）
        FS->>BE: ScanMeta(handler)
        BE->>OS: ListKeys()
        OS-->>BE: KeyInfo[]
        loop 每满 scanmeta_iterator_keys_limit
            BE->>FS: handler(keys, metadatas)
            FS->>MS: NotifyOffloadSuccess
        end
        BE-->>FS: OK
    end
```
