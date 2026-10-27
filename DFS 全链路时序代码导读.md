# DFS 全链路时序代码导读

> **对应图**：`FileStorage-offload-实现解读.md` 的 6.7.6「DFS 全链路时序」。
> **代码基线**：`main` @ `67df8c41`，所有位置写成 `路径:起止行`。
> **代码约定**：正文代码块是**源码原文**，只做了两件事——在行尾/行首加了 `//` 中文注释，以及用 `// ...（略 …）` 跳过与本步无关的整段。除这两类改动外没有改写任何代码，你可以直接按行号对照仓库。

## 0. 怎么读这份导读

每一节的结构固定：

```
位置（路径:起止行）
  └─ 代码（带行内注释）
  └─ 这一段在做什么     ← 读完应该记住的判断
  └─ 重点语法 / 惯用法   ← 这段代码用到的 C++ 写法与坑
  └─ 与 6.7.6 的差异     ← 图上看不到、但代码里存在的东西
```

建议顺着 1 → 2 → 3 读一遍，再回看第 4 节（差异汇总）与第 5 节（语法速查）。如果只想搞清「DFS 到底走不走 Transfer Engine」，直接看 2.1 与 2.3。

## 1. 图上每一步落到哪个函数

| 图上消息 | 真实入口 | 位置 | 本文 |
| --- | --- | --- | --- |
| `App->>C: put(key, value)` | `RealClient::put_internal` → `Client::Put` | `client_service.cpp:1959` | 2.1 |
| `C->>M: PutStart(dfs_replica_num=1)` | `MasterClient::PutStart` → `MasterService::PutStart` | `master_service.cpp:5589` | 2.2 |
| `M->>A: Allocate(key, size)` | `DfsAllocatorInterface::Allocate`（master 进程内） | `master_service.cpp:5306` | 2.2 |
| `A-->>M: DistributedFSDescriptor{...}` | `ShardAllocator::Allocate` / `ImmutableBucketAllocator::Allocate` | `shard_allocator.cpp:255` / `immutable_bucket_allocator.cpp:218` | 2.2 |
| `M-->>C: replicas + DFS(PROCESSING)` | `InsertMetadata` + `get_descriptor()` | `master_service.cpp:5343` | 2.2 |
| `C->>C: 先写 MEMORY / NoF` | `Client::Put` 的 `TransferWrite` 循环 | `client_service.cpp:2033` | 2.1 |
| `C->>BE: WriteDfsReplicas → BatchWrite` | `Client::WriteDfsReplicas` → `DistributedStorageBackend::BatchWrite` | `client_service.cpp:2884` / `distributed_storage_backend.cpp:634` | 2.3 / 2.4 |
| `BE->>FS: GetOrOpenShard + BatchWriteAt` | `GetOrOpenShard` / `OpenBucket` → `ExecuteSingleIo` / `SubmitBatchIo` → `FileSystemAdapter` | `distributed_storage_backend.cpp:216` / `454` | 2.5 / 2.6 / 2.7 |
| `C->>M: PutEnd(ALL)` | `MasterService::PutEnd` | `master_service.cpp:5663` | 2.8 |
| `M->>M: DFS 副本 → COMPLETE` | `mark_complete()` + `UpdateAccess()`（bucket 先 `MarkCommitted`） | `master_service.cpp:5725-5763` | 2.8 |
| `C-->>App: 返回（WriteAt 已完成，但不是 fsync）` | `DetermineFinalizeDecision` → `PutEnd` / `PutRevoke` | `client_service.cpp:2081` | 2.1 |
| `App->>R: get(key)` | `RealClient::get_buffer_internal` | `real_client.cpp:3730` | 3.1 |
| `R->>M: Query / BatchQuery` | `Client::Query` → `MasterService::GetReplicaList` | `client_service.cpp:1240` / `master_service.cpp:4777` | 3.1 |
| `M-->>R: replicas（含 DFS descriptor）` | `GetReadableReplicaDescriptors` + `UpdateAccess` | `master_service.cpp:4795` | 3.1 |
| `R->>R: SelectBestReplica` | `SelectBestReplica`（**RealClient 层**）+ `FilterQueryResult` | `replica_selection.h:122` / `real_client.cpp:493` | 3.1 |
| `R->>BE: ReadDfsReplica → BatchRead` | `Client::ReadDfsReplica` / `Client::BatchGet` 攒批 | `client_service.cpp:5046` / `1744` | 3.2 |
| `BE->>FS: BatchReadAt 直接写入调用方 slices` | `DistributedStorageBackend::BatchRead` → adapter | `distributed_storage_backend.cpp:722` | 3.2 |
| `R->>R: VerifyObjectChecksum` | `Client::VerifyObjectChecksum`（默认关闭） | `client_service.cpp:1349` | 3.3 |

## 2. 写入路径：`put` → `PutEnd`

### 2.0 写入路径的总览

一段话版本：**master 只给位置，客户端自己写文件**。`PutStart` 一次性返回 memory / noF / DFS 三类描述符；客户端先铺 DISK（若存在，异步），再同步写 memory / noF，全部成功后才把 DFS 的 `DfsWriteRequest` 批量交给 backend；backend 按 descriptor 打开文件、做位置写；最后 `PutEnd(ALL)` 让 master 把 DFS 副本从 `PROCESSING` 推到 `COMPLETE`。任何一步失败都会整体 `PutRevoke(ALL)`。

### 2.1 应用入口：Client::Put（图上第 1–6 步）

这一段对应图上 `App->>C: put` 到 `C->>C: 先写 MEMORY / NoF`，外加随后决定「要不要写 DFS」的门禁。

#### 2.1.1 Client::Put 的签名、checksum 与 PutStart 前的准备

`mooncake-store/src/client_service.cpp:1959-1985`

```cpp
tl::expected<void, ErrorCode> Client::Put(const ObjectKey& key,  // 成功无返回值，失败带回 ErrorCode
                                          std::vector<Slice>& slices,  // 非 const：为透传给 TransferWrite(vector<Slice>&)
                                          const ReplicateConfig& config) {  // 只读；函数内 AttachHostId 会另拷一份
    std::optional<uint64_t> object_checksum;  // 空 = 不计算校验和（默认关闭）
    if (object_checksum_enabled_) {  // 构造期从 MOONCAKE_STORE_CHECKSUM 快照，运行期不变
        auto checksum_result = ComputeObjectChecksumForSlices(
            key, slices, CalculateSliceSize(slices));  // CRC 扫全部 slice；设备指针内部走 D2H
        if (!checksum_result) {
            return tl::unexpected(checksum_result.error());  // 算不出校验和就整个失败，不留半截对象
        }
        object_checksum = *checksum_result;
    }

    // Prepare slice lengths
    std::vector<size_t> slice_lengths;  // master 只按长度分配，拿不到数据指针
    for (size_t i = 0; i < slices.size(); ++i) {
        slice_lengths.emplace_back(slices[i].size);  // 顺序必须与 slices 一一对应
    }

    ReplicateConfig client_cfg = AttachHostId(config);  // 拷贝后补 host_id，供 master 做节点亲和/同机分配
    if (protocol_ == "cxl") {
        client_cfg.preferred_segment = local_hostname_;  // CXL 下强制落到本机 segment
    }

    if (hot_cache_) {
        hot_cache_->RemoveHotKey(key);  // 写前失效热缓存，避免整个写窗口内命中旧映射
    }
```

**这一段在做什么**

- Put 不拷贝数据：RealClient 已把 value gather 进注册过的 ClientBuffer 并 split_into_slices，这里只拿到一批 view。
- object_checksum 默认关闭，只有打开 MOONCAKE_STORE_CHECKSUM 才算；开关是构造期 magic static，运行中不可变。
- slices 非 const 是被 TransferWrite/TransferSubmitter 的签名传染的，Put 自己不修改 slice 内容。
- master 收到的只有 slice_lengths，它据此决定分配大小，并生成 DFS descriptor 的 object_size/aligned_size。
- AttachHostId 与 cxl 的 preferred_segment 只影响 master 的分配决策，与随后的数据面路径无关。
- RemoveHotKey 发生在 PutStart 之前，覆盖的失效窗口是整个写过程而不是写完之后。

**重点语法 / 惯用法**

- tl::expected<void, ErrorCode>：expected 的 void 特化，成功直接返回 {}，失败用 tl::unexpected 包装错误码。
- std::optional<uint64_t>：空值本身就是“未启用”语义，避免用 0 当哨兵。
- emplace_back：原地构造元素；这里把 Slice::size（uint64）窄化存进 size_t 容器。
- 函数内 static const bool：C++11 magic static，首次调用初始化且线程安全，是“一次性读环境变量”的常见写法。
- protocol_ == "cxl"：std::string 的 operator== 是值比较，不是指针比较。
- Slice 结构：{void* ptr; size_t size} 的 POD，多 slice 表示同一对象的离散内存段。

**与 6.7.6 的差异**：图把 put(key,value) 画成一步，省略了 RealClient 里的 allocate + gather_maybe_device_to_host + split_into_slices；也没有表现 checksum 分支（默认关闭）。

#### 2.1.2 PutStart、副本记账，以及 DISK 副本必须先发起的异步写

`mooncake-store/src/client_service.cpp:1988-2031`

```cpp
    auto start_result = master_client_.PutStart(key, slice_lengths, client_cfg);  // 一次 RPC 拿回全部副本描述符（含 DFS）
    if (!start_result &&
        start_result.error() == ErrorCode::OBJECT_ALREADY_EXISTS &&
        healDanglingLocalDiskReplica(key) == DiskReplicaHealResult::kEvicted) {  // 陈旧 LOCAL_DISK 元数据已被清掉
        start_result = master_client_.PutStart(key, slice_lengths, client_cfg);  // 只有真的清掉了才重试一次
    }
    if (!start_result) {
        ErrorCode err = start_result.error();
        if (err == ErrorCode::OBJECT_ALREADY_EXISTS) {
            VLOG(1) << "object_already_exists key=" << key;
            return {};  // 已存在按成功处理：幂等，但这次没有写任何数据（含 DFS）
        }
        if (err == ErrorCode::NO_AVAILABLE_HANDLE) {
            LOG(WARNING) << "Failed to start put operation for key=" << key
                         << PUT_NO_SPACE_HELPER_STR;  // 典型是空间不足，附排错提示
        } else {
            LOG(ERROR) << "Failed to start put operation for key=" << key
                       << ": " << toString(err);
        }
        return tl::unexpected(err);
    }

    ReplicaTransferSummary transfer_summary;
    for (const auto& replica : start_result.value()) {  // 按类型记账：memory / nof / dfs
        transfer_summary.RecordAllocatedReplica(replica);  // DISK、LOCAL_DISK 不进这本账
    }

    // Record Put transfer latency (all replicas)  // 从 PutStart 返回到此刻，不含 checksum 与分配耗时
    auto t0_put = std::chrono::steady_clock::now();

    // We must deal with disk replica first, then the disk putrevoke/putend can  // 顺序要求：DISK 必须先发起
    // be called surely
    if (storage_backend_) {  // 没配本地 storage backend 就整段跳过
        for (auto it = start_result.value().rbegin();  // 倒序：master 把 DISK/DFS 追加在列表尾部
             it != start_result.value().rend(); ++it) {
            const auto& replica = *it;
            if (replica.is_disk_replica()) {  // 是 DISK（master 侧落盘），不是 LOCAL_DISK
                // Store to local file if storage backend is available
                auto disk_descriptor = replica.get_disk_descriptor();  // 取 master 分配好的 file_path/object_size
                PutToLocalFile(key, slices, disk_descriptor);  // 内部同步 D2H，然后入队异步 StoreObject+PutEnd(DISK)
                break;  // Only one disk replica is needed  // 多个 DISK 也只写一个
            }
        }
    }
```

**这一段在做什么**

- PutStart 是唯一一次“取描述符”的调用：DFS 的 file_path/offset/shard_idx 全在返回值里，后面不再问 master。
- OBJECT_ALREADY_EXISTS 直接 return {} 意味着这次 Put 什么数据都没写，DFS 副本自然也不会被创建。
- healDanglingLocalDiskReplica 是 issue #3709 的兜底：只有确认磁盘文件真的没了才 evict，然后重试一次 PutStart。
- RecordAllocatedReplica 只统计 memory/nof/dfs 三类；DISK 结果不进入 transfer_summary，所以它对 DFS 门禁没有任何影响。
- PutToLocalFile 是“同步 staging + 异步落盘”：D2H 拷贝在调用线程做完（保证 GPU 缓冲仍有效），StoreObject 和 PutEnd(DISK) 丢给 2 线程的 write_thread_pool_。
- 倒序扫描 + break 的作用是“只取列表末尾那一个 DISK 副本”，而不是把所有 DISK 都写一遍。

**重点语法 / 惯用法**

- auto + start_result.error()：tl::expected 的 error() 不消耗对象，可在条件与日志里反复取。
- if 链中调用带副作用的函数：healDanglingLocalDiskReplica 受 && 短路保护，只有前两个条件都成立才执行。
- rbegin()/rend() 反向迭代器：++it 实际向列表头部移动，*it 拿到元素引用。
- return {};：tl::expected<void, E> 的成功写法，等价于构造一个无值成功对象。
- start_result.value() 返回 vector 的引用，range-for 期间生命周期由 start_result 持有。
- VLOG(1)：verbosity 低于 1 时参数不会求值，适合放在可能高频的幂等分支。

**与 6.7.6 的差异**：图里没有 PutStart 的 OBJECT_ALREADY_EXISTS / heal 重试两条分支；也没有画 DISK 副本这一段。DISK 只在 master 配了 --root_fs_dir（use_disk_replica_）时才存在，而且它的落盘是异步的，图上的 “C->>C: 先写 MEMORY / NoF” 之前其实还有一次同步 D2H staging。

#### 2.1.3 MEMORY/NOF 的 Transfer Engine 写，以及 DFS 门禁的组装

`mooncake-store/src/client_service.cpp:2033-2060`

```cpp
    for (const auto& replica : start_result.value()) {  // 同步副本串行写，顺序即 master 返回顺序
        if (replica.is_memory_replica() || replica.is_nof_replica()) {  // 只有这两类走 Transfer Engine
            // Transfer data using allocated handles from all replicas
            const auto replica_type = replica.is_memory_replica()
                                          ? ReplicaType::MEMORY  // 先定枚举，失败记账要分类型
                                          : ReplicaType::NOF_SSD;
            ErrorCode transfer_err = TransferWrite(replica, slices);  // 阻塞到 TE 写完；这里就是 slices 非 const 的原因
            if (transfer_err != ErrorCode::OK) {
                transfer_summary.RecordFailure(replica_type, transfer_err);  // 记录首个错误
                continue;  // 不提前退出：finalize 需要完整账本
            }
            transfer_summary.RecordSuccess(replica_type);
        }
    }

    if (transfer_summary.allocated_dfs_replicas > 0 &&  // 分配过才谈得上写
        NonDfsTransfersSucceeded(transfer_summary)) {  // 硬门禁：memory/nof 必须全成功且零失败
        std::vector<std::string> dfs_keys;  // 三组平行数组，长度必须一致
        std::vector<const std::vector<Slice>*> dfs_slices;  // 存指针，不拷贝 slice 数组
        std::vector<DistributedFSDescriptor> dfs_descriptors;  // master 分配好的 path/offset/shard_idx
        for (const auto& replica : start_result.value()) {
            if (!replica.is_dfs_replica()) {
                continue;
            }
            dfs_keys.push_back(key);
            dfs_slices.push_back(&slices);  // DFS 写的就是调用方这块内存（设备指针在 backend 内 staging）
            dfs_descriptors.push_back(replica.get_dfs_descriptor());
        }
```

**这一段在做什么**

- MEMORY/NOF 通过 TransferWrite 走 TE 同步写完；DFS 的数据面完全不经过这里，两类 I/O 只是顺序相邻。
- 写失败只 RecordFailure + continue，循环继续尝试其余副本，保证 finalize 拿到完整的成功/失败计数。
- NonDfsTransfersSucceeded 要求 successful_memory==allocated_memory、successful_nof==allocated_nof 且两类 failed 都为 0。
- 门禁完全不看 DISK 结果，因为 DISK 不在 transfer_summary 里；DFS 是否写只由 memory/nof 决定。
- dfs_keys/dfs_slices/dfs_descriptors 是 WriteDfsReplicas 批量接口的三元组形状，靠下标对齐。
- dfs_slices 放的是 &slices，说明 GPU 指针的 D2H staging 在 WriteDfsReplicas 内部完成，Put 层不做拷贝。

**重点语法 / 惯用法**

- 三目运算符初始化 const auto：一旦推导出具体枚举类型，后面的 RecordFailure/RecordSuccess 就统一用同一变量。
- range-for 遍历 start_result.value()：拿到的是 master 返回 vector 的 const 引用，不产生副本。
- vector<const vector<Slice>*>：指针容器实现零拷贝，代价是调用方必须保证被指对象仍然存活。
- 布尔短路 &&：allocated_dfs_replicas > 0 为假时，NonDfsTransfersSucceeded 不会被调用。
- continue 而非 break：C++ 里这是“跳过本条、继续处理其余副本”的标准写法。

**与 6.7.6 的差异**：图上 “C->>BE: WriteDfsReplicas → BatchWrite” 的位置与代码一致（在 MEMORY/NoF 之后），但图没有画出这个前置门禁：非 DFS 副本只要有一个失败，整条 DFS 写入就不会发生。

#### 2.1.4 WriteDfsReplicas 的结果记账与被跳过的 DFS 写

`mooncake-store/src/client_service.cpp:2061-2079`

```cpp
        for (auto dfs_result :
             WriteDfsReplicas(dfs_keys, dfs_slices, dfs_descriptors)) {  // 同步调用，返回每个 key 的 ErrorCode
            if (dfs_result == ErrorCode::OK) {
                transfer_summary.RecordSuccess(ReplicaType::DFS);  // 与 allocated_dfs_replicas 配对，供 finalize 比较
            } else {
                transfer_summary.RecordFailure(ReplicaType::DFS, dfs_result);  // 首个错误会被记进 first_error
            }
        }
    } else if (transfer_summary.allocated_dfs_replicas > 0) {  // 分配了 DFS，但门禁没过
        // A DFS replica was allocated but is abandoned because a peer replica
        // failed first. Without this counter the write simply disappears: no  // 没有这个计数就彻底静默
        // error is recorded against DFS and no I/O is attempted. Counted per
        // key, matching SubmitDfsWrites.
        VLOG(1) << "Skipping DFS write, action=skip, key=" << key
                << ", reason=non_dfs_transfer_failed";  // 唯一可观测痕迹
        if (auto* dfs_metric = GetDfsMetricPtr()) {  // metrics_ 未启用时返回 nullptr
            dfs_metric->RecordSkippedWrites(1);  // 单 key 计 1；批量路径聚合成一次
        }
    }
```

**这一段在做什么**

- WriteDfsReplicas 是同步的：返回时 BatchWrite 已经结束，每个 key 对应一个 ErrorCode。
- 这一支对结果逐条记账，成功数会被 AllAllocatedTransfersSucceeded 用来和 allocated_dfs_replicas 对比。
- 跳过分支完全不产生 I/O，也不报错，只打一条 VLOG 并给指标 +1——这是排查“数据凭空消失”的唯一线索。
- 跳过时 allocated_dfs_replicas 仍大于 0，finalize 会因 successful_dfs_transfers 不匹配而判定整体失败并 PutRevoke。
- 单 key 版每条记一次 RecordSkippedWrites(1)，与批量版 SubmitDfsWrites 的聚合计数刻意保持一致。
- 从 DFS_SERVICE_UNAVAILABLE 到 per-key 失败都由 WriteDfsReplicas 内部返回，Put 只负责记账。

**重点语法 / 惯用法**

- range-for 直接遍历函数返回值：产生一个临时 vector<ErrorCode>，其生命周期延续到循环结束。
- if/else 覆盖全部取值：每个 dfs_result 必定落到成功或失败之一，不会漏账。
- else if 与前面的 if 互斥：保证“要么写、要么跳过”，不存在两者都执行。
- if (auto* p = f())：C++17 的 init-statement，p 的作用域被限制在这个 if 语句内。
- 结构化日志字段 action= / reason= / key=：便于 grep 和与指标对齐。

**与 6.7.6 的差异**：图上没有“DFS 被跳过”这条路径，也没有 PutRevoke 分支；看起来 PutEnd(ALL) 总会执行，实际门禁失败时 finalize 会选择 PutRevoke(ALL)。

#### 2.1.5 延迟指标、DetermineFinalizeDecision 与 PutEnd/PutRevoke 收尾

`mooncake-store/src/client_service.cpp:2081-2115`

```cpp
    auto us_put = std::chrono::duration_cast<std::chrono::microseconds>(
                      std::chrono::steady_clock::now() - t0_put)
                      .count();  // 微秒整数，覆盖所有副本的传输耗时
    if (metrics_) {  // 指标未开启时 metrics_ 为空
        metrics_->transfer_metric.put_latency_us.observe(us_put);
    }

    const auto finalize_decision =
        DetermineFinalizeDecision(config, transfer_summary);  // 纯函数：只看配置 + 计数，不做 I/O

    if (finalize_decision.end_type.has_value()) {  // 全成功时是 ReplicaType::ALL
        auto end_result = master_client_.PutEnd(
            ObjectMeta{key, object_checksum}, *finalize_decision.end_type);  // checksum 为空就是 nullopt
        if (!end_result) {
            ErrorCode err = end_result.error();
            LOG(ERROR) << "Failed to end put operation: " << err;
            return tl::unexpected(err);  // 直接返回，不会补一次 PutRevoke
        }
    }

    if (finalize_decision.revoke_type.has_value()) {  // flexible dual-replica 下可能与 end 同时出现
        auto revoke_result =
            master_client_.PutRevoke(key, *finalize_decision.revoke_type);
        if (!revoke_result) {
            LOG(ERROR) << "Failed to revoke put operation";
            return tl::unexpected(revoke_result.error());
        }
    }

    if (!finalize_decision.success) {  // end 成功但整体仍失败（end+revoke 组合外的情况）
        return tl::unexpected(finalize_decision.error);
    }

    return {};  // void expected 的成功返回
}
```

**这一段在做什么**

- DetermineFinalizeDecision 是纯函数：非 flexible 模式下，分配满足且所有副本成功 → PutEnd(ALL)，否则 → PutRevoke(ALL)。
- PutEnd(ALL) 在 master 侧只匹配 memory/nof/dfs 三类，DISK 由 PutToLocalFile 的异步线程自己 PutEnd(DISK) 收尾。
- ObjectMeta{key, object_checksum} 在 checksum 关闭时是 nullopt，master 因此不会做校验和比对。
- PutEnd 失败立即返回，没有补偿性 PutRevoke，master 侧对象会停在 PROCESSING 直到超时回收。
- 函数成功返回只代表 DFS 的 pwritev/USRBIO 已完成，没有 fsync；DISK 副本此刻大概率还在后台队列里。
- PutRevoke 分支服务于 flexible dual-replica（memory/nof 二选一成功），与 DFS 无关。

**重点语法 / 惯用法**

- std::chrono::duration_cast<std::chrono::microseconds>(...).count()：显式指定精度再取整数计数。
- std::optional 的 has_value() + operator*：先判空再解引用，不能用 operator* 直接探测。
- ObjectMeta{key, object_checksum}：聚合初始化，第二个成员是 std::optional<uint64_t>，nullopt 合法。
- return tl::unexpected(err)：早返回模式，函数末尾才是唯一的成功出口。
- const auto finalize_decision：用一个具名的纯值对象承载 end_type/revoke_type/success/error 四个决策。

**与 6.7.6 的差异**：图上的 “M->>M: DFS 副本 → COMPLETE” 确实是 PutEnd(ALL) 的副作用，这点正确；但图没有体现 end_type 可能是 MEMORY/NOF_SSD 子集（flexible dual-replica），也没有 PutRevoke 分支，更没有 DISK 由后台线程单独 PutEnd。另外图注 “WriteAt 已完成，但不是 fsync” 对 DFS 成立，对 DISK 则连 StoreObject 都可能还没跑完。

### 2.2 master 分配：PutStart → Allocate → descriptor（第 2–4 步）

图上 `C->>M: PutStart` 到 `M-->>C: replicas + DFS(PROCESSING)` 的全部实现。真实顺序比图细：`PrepareWriteStart` 校验 → 租户配额 → `AllocateAndInsertMetadata` → `InsertMetadata` 发布。

#### 2.2.1 PrepareWriteStart：DFS 相关的四类拒绝

`mooncake-store/src/master_service.cpp:5442-5474`

```cpp
auto MasterService::PrepareWriteStart(                      // PutStart/Upsert 共用的前置校验，只校验不写元数据
    const UUID& client_id, const std::string& key, const TenantId& tenant_id,
    uint64_t slice_length, const ReplicateConfig& config,
    std::string_view action) -> tl::expected<WriteStartPlan, ErrorCode> {  // 成功给计划，失败给错误码
    if ((config.replica_num == 0 && config.nof_replica_num == 0 &&
         config.dfs_replica_num == 0) ||                     // 拒绝①：三类副本全为 0，无事可做
        key.empty() || slice_length == 0) {                   // 空 key / 零长度同样判非法
        LOG(ERROR) << "key=" << key << ", replica_num=" << config.replica_num
                   << ", nof_replica_num=" << config.nof_replica_num
                   << ", dfs_replica_num=" << config.dfs_replica_num
                   << ", slice_length=" << slice_length
                   << ", key_size=" << key.size() << ", error=invalid_params";
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);  // 构造“失败”的 expected，不抛异常
    }
    if (config.dfs_replica_num > 1) {                         // 拒绝②：DFS 只允许 0 或 1 份
        LOG(ERROR) << "key=" << key
                   << ", dfs_replica_num=" << config.dfs_replica_num
                   << ", error=dfs_replica_num_must_be_0_or_1";
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    if (config.dfs_replica_num > 0 && !tenant_id.IsDefault()) {  // 拒绝③：DFS 暂只支持默认租户
        LOG(ERROR) << "key=" << key << ", tenant_id=" << tenant_id
                   << ", error=dfs_currently_requires_default_tenant";
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    if (config.dfs_replica_num > 0 &&
        (!enable_dfs_ || dfs_allocator_ == nullptr ||        // 拒绝④：总开关关 / 分配器未建 / 未初始化
         !dfs_allocator_->IsInitialized())) {
        LOG(ERROR) << "key=" << key
                   << ", dfs_replica_num=" << config.dfs_replica_num
                   << ", error=dfs_service_unavailable";
        return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);  // 服务不可用≠参数错误
    }
```

**这一段在做什么**

- 所有 DFS 相关的合法性判断都在这里完成，函数只读 config、不碰元数据，所以它在 PutStart 拿锁之前就能失败返回。
- 四类拒绝依次是：三类副本数全 0 / key 空 / 长度 0；dfs_replica_num > 1；非 default tenant；DFS 未启用或 allocator 未初始化。
- 注意拒绝①只要求“三类不同时为 0”，所以 replica_num=0、nof_replica_num=0、dfs_replica_num=1 的纯 DFS 请求能过这一关，DFS 不算必须依附其他副本。
- 非 default tenant 被拒是因为 Allocate 只接收裸 key，分配器里没有租户维度，不同租户的同名 key 会落到同一把 bucket/shard。
- allocator 未就绪返回 DFS_SERVICE_UNAVAILABLE 而非 INVALID_PARAMS，客户端据此区分“我传错了”和“服务端暂时不可用”。
- 函数结尾才组装 WriteStartPlan（C++20 指定初始化器），失败路径全部提前 return，没有清理负担。

**重点语法 / 惯用法**

- 尾置返回类型：`auto f(...) -> tl::expected<T, E>`，便于在定义处看到返回类型。
- tl::expected：`tl::make_unexpected(E)` 显式构造失败值；调用方用 `if (!x)` 或 `x.has_value()` 判断。
- std::string_view action：只读的调用方名字（"put_start"/"upsert"），比 const std::string& 更轻，调用点可传字面量。
- enum class ErrorCode：作用域枚举，日志里靠 operator<< 或 static_cast<int> 输出，不会隐式转 int。

**与 6.7.6 的差异**：时序图从 `App->>C: put` 直接跳到 `C->>M: PutStart` → `M->>A: Allocate`，没有画 master 侧的这层前置校验；图上也没有体现 dfs_replica_num 只能是 0/1、必须 default tenant、enable_dfs_/allocator 三重就绪这三个约束。画错/遗漏的调用方是 M 自己（内部 PrepareWriteStart），不是 A。

#### 2.2.2 AllocateAndInsertMetadata：配额 → AllocateReplicas → InsertMetadata

`mooncake-store/src/master_service.cpp:5404-5440`

```cpp
auto MasterService::AllocateAndInsertMetadata(               // PutStart 核心三步：扣配额→分配→入库
    MetadataAccessorRW& accessor, const UUID& client_id, uint64_t value_length,
    const ReplicateConfig& config, const std::string& writer_host_id,
    const std::string& group_id,
    const std::chrono::system_clock::time_point& now,
    const ResolvedSoftPinRequest& soft_pin_request,
    std::optional<std::chrono::system_clock::time_point>
        committed_soft_pin_timeout,
    bool* dfs_allocation_failed)                             // 出参：DFS 是否因容量耗尽失败
    -> tl::expected<std::vector<Replica::Descriptor>, ErrorCode> {
    assert(!accessor.IsPublished());                         // 只允许在未发布的新 key 上调用
    const std::string& key = accessor.GetKey();

    const uint64_t pending_quota_charge =
        RequestedMemoryQuotaCharge(value_length, config);    // 按尺寸估算要预扣的租户配额
    auto quota_result =
        ChargeTenantQuota(accessor.QuotaAccount(), pending_quota_charge);
    if (!quota_result) {
        return tl::make_unexpected(quota_result.error());    // 配额不够：连分配都不做
    }

    auto allocation_result = AllocateReplicas(               // 真正去拿 MEMORY/NoF/DISK/DFS 句柄
        key, value_length, config, writer_host_id, dfs_allocation_failed);
    if (!allocation_result) {
        ReleaseTenantQuota(accessor.QuotaAccount(), pending_quota_charge);  // 失败必须退回预扣
        return tl::make_unexpected(allocation_result.error());
    }

    auto insert_result = InsertMetadata(                     // 副本移交元数据并生成返回列表
        accessor, client_id, value_length, config, group_id, now,
        soft_pin_request, std::move(allocation_result.value()),
        pending_quota_charge, std::move(committed_soft_pin_timeout));
    if (!insert_result) {
        ReleaseTenantQuota(accessor.QuotaAccount(), pending_quota_charge);
    }
    return insert_result;
}
```

**这一段在做什么**

- 图上的 `M->>A: Allocate` 在外层还裹着一层租户配额预扣；DFS 本身不占内存配额，但请求只要带 MEMORY 副本就会先扣。
- 两条失败路径都必须 ReleaseTenantQuota，否则租户额度会被永久占用——读这段时盯住两个 if 里的回滚。
- dfs_allocation_failed 只做透传，这里不解释它的语义；判断与重试逻辑在 AdmitWithDfsRecovery 里。
- `std::move(allocation_result.value())` 把 vector<Replica> 整体移入 InsertMetadata，分配结果此后不再可用。
- InsertMetadata 成功时返回的是 vector<Replica::Descriptor>，也就是 PutStart 响应里的 replicas 字段。

**重点语法 / 惯用法**

- `auto x = f(); if (!x)` 是 tl::expected 的惯用判空写法，等价于 `!x.has_value()`。
- `allocation_result.value()` 返回引用，配合 std::move 实现移动而非拷贝；对 expected 用 `*x` 效果相同。
- assert(!accessor.IsPublished())：只在 debug 生效的契约检查，release 下不产生代码。
- 布尔指针出参（bool*）用于把“DFS 容量耗尽”这一细粒度信号穿出两层调用，调用方需自行判空。

**与 6.7.6 的差异**：时序图没有画租户配额预扣（ChargeTenantQuota）及其失败回滚，也没有画 `dfs_allocation_failed` 这条旁路信号；图上 `M->>A: Allocate(key,size)` 实际发生在本函数调用的 AllocateReplicas 内部，而不是 PutStart 直接发出。

#### 2.2.3 AllocateReplicas 入口与前置门槛：凑不齐副本就根本不会调用 DFS 分配

`mooncake-store/src/master_service.cpp:5140-5153, 5253-5276`

```cpp
auto MasterService::AllocateReplicas(const std::string& key,   // 所有副本的唯一分配入口
                                     uint64_t value_length,
                                     const ReplicateConfig& config,
                                     const std::string& writer_host_id,
                                     bool* dfs_allocation_failed)  // 可为空指针，仅 DFS 分支写它
    -> tl::expected<std::vector<Replica>, ErrorCode> {
    if (dfs_allocation_failed != nullptr) {
        *dfs_allocation_failed = false;                      // 进函数先清零，避免上轮重试残留
    }
    std::vector<Replica> replicas;                           // 按 MEMORY→NoF→DISK→DFS 顺序累加
    const auto write_mode = DetermineReplicaWriteMode(config);
    size_t allocated_memory_replicas = 0;
    size_t allocated_nof_replicas = 0;
    bool has_enough_memory_segments = false;

    // ...（略 5155-5252：MEMORY 与 NoF 各走一次 allocation_strategy_->Allocate；
    //      失败时通常直接返回 NO_AVAILABLE_HANDLE，只有 FLEXIBLE_DUAL_REPLICA 允许降级继续）

    if (!HasExpectedReplicaAllocation(config, allocated_memory_replicas,
                                      allocated_nof_replicas)) {  // 目标副本数没凑齐就整单失败
        if ((config.replica_num > 0 &&
             allocated_memory_replicas != config.replica_num) ||
            (config.nof_replica_num > 0 &&
             allocated_nof_replicas != config.nof_replica_num)) {
            MasterMetricManager::instance().inc_put_start_alloc_failures();
            if (config.replica_num > 0 &&
                allocated_memory_replicas != config.replica_num &&
                has_enough_memory_segments) {
                need_mem_eviction_ = true;                   // 段够却没分到 ⇒ 值得触发淘汰
            }
            if (config.nof_replica_num > 0 &&
                allocated_nof_replicas != config.nof_replica_num) {
                need_nof_eviction_ = true;
            }
        }
        VLOG(1) << "Failed to satisfy replica allocation requirement for key="
                << key << ", requested_memory_replicas=" << config.replica_num
                << ", allocated_memory_replicas=" << allocated_memory_replicas
                << ", requested_nof_replicas=" << config.nof_replica_num
                << ", allocated_nof_replicas=" << allocated_nof_replicas;
        return tl::make_unexpected(ErrorCode::NO_AVAILABLE_HANDLE);  // 提前返回 ⇒ DFS 不会被分配
    }
```

**这一段在做什么**

- 入口先无条件把 *dfs_allocation_failed 置 false：AdmitWithDfsRecovery 会跨多次 admit 复用同一个变量，不清零会让上一次的 true 误触发 bucket 淘汰。
- HasExpectedReplicaAllocation 的语义：若 nof=0 且 dfs=0，只要求 memory>0；FLEXIBLE_DUAL_REPLICA 下 memory+nof>0 即可；否则要求 memory/nof 数量精确等于请求值。
- 这道门槛在 DFS 之前，所以 MEMORY/NoF 分配失败时整单返回 NO_AVAILABLE_HANDLE，DFS descriptor 一个都不会生成——不会出现“只有 DFS 副本的半成品对象”。
- need_mem_eviction_ / need_nof_eviction_ 是写给后台淘汰器的提示位，不是立即淘汰；区分“段本来就少”和“段够但都满了”。
- 注意纯 DFS 请求（replica_num=0, nof=0, dfs=1）会走到最后一个 return 且 0==0，直接通过这道门槛。

**重点语法 / 惯用法**

- `bool* dfs_allocation_failed` 允许传 nullptr，函数内先判空再解引用；这是 C++ 里常见的“可选输出参数”写法。
- `!HasExpectedReplicaAllocation(...)` 承接的是普通 bool，不是 expected；两层“失败”语义（业务判定 vs 错误码）在这里混用，读时别混淆。
- size_t 计数器 + 传值给判定函数，避免判定函数直接看 config 猜结果。
- MasterMetricManager::instance() 单例 + inc_* 计数器，用于暴露 put_start 分配失败率。

**与 6.7.6 的差异**：时序图上 `M->>A: Allocate(key,size)` 只画了一次成功调用；代码里它前面还有 MEMORY/NoF 分配与数量校验，校验不过就直接返回、DFS 分配不会被调用。图上也没有 bucket/内存淘汰提示位（need_mem_eviction_ 等）。

#### 2.2.4 AllocateReplicas 的 DISK 与 DFS 段：Allocate 调用与 PROCESSING 入列

`mooncake-store/src/master_service.cpp:5293-5319`

```cpp
    if (use_disk_replica_) {                                 // 只有配了 root_fs_dir 才有 DISK 副本
        std::string file_path =
            FileUtil::ResolvePathFromKey(key, root_fs_dir_, cluster_id_);
        replicas.emplace_back(file_path, value_length,        // 就地构造 Replica(DiskReplicaData)
                              ReplicaStatus::PROCESSING);     // 状态先给 PROCESSING，客户端异步落盘
    }

    if (config.dfs_replica_num > 0) {                        // 图上的 "M->>A: Allocate(key,size)"
        if (!dfs_allocator_ || !dfs_allocator_->IsInitialized()) {  // 前置已验过，这里防运行期被摘掉
            LOG(ERROR) << "Failed to allocate DFS replica for key=" << key
                       << ", error=dfs_allocator_not_initialized";
            return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);
        }
        auto alloc = dfs_allocator_->Allocate(key, value_length);  // expected<DistributedFSDescriptor, ErrorCode>
        if (!alloc) {                                        // expected 的 operator! 即 !has_value()
            LOG(ERROR) << "Failed to allocate DFS replica for key=" << key
                       << ", error=" << alloc.error();
            if (dfs_allocation_failed != nullptr) {
                *dfs_allocation_failed =
                    alloc.error() == ErrorCode::NO_AVAILABLE_HANDLE;  // 仅容量满才允许触发淘汰
            }
            return tl::make_unexpected(alloc.error());       // 原样上抛，不统一成 NO_AVAILABLE_HANDLE
        }
        replicas.emplace_back(std::move(*alloc), ReplicaStatus::PROCESSING);  // 描述符+PROCESSING 组成 DFS 副本
    }

    return replicas;                                         // 此时 DFS 副本已带 path/offset/shard_idx
}
```

**这一段在做什么**

- DFS 分配在 DISK 之后、函数末尾：它是“最后一块拼图”，前面任何一步失败都不会走到这里。
- Allocate(key, value_length) 只接收 key 和大小，返回值是 tl::expected<DistributedFSDescriptor, ErrorCode>，descriptor 里带 file_path / offset / object_size / aligned_size / shard_idx。
- 第二个 IsInitialized 检查是防御性的：PrepareWriteStart 到真正分配之间，分配器可能被热替换或关闭。
- dfs_allocation_failed 只在 `error() == NO_AVAILABLE_HANDLE`（容量耗尽）时为 true；IO 错误、参数错误等不会被当成“可淘汰恢复”，避免误删无关 DFS 数据。
- 失败时上抛的是 allocator 自己的错误码（可能是 NO_AVAILABLE_HANDLE、DFS_SERVICE_UNAVAILABLE 等），而不是统一改写，客户端据此区分容量与故障。
- `replicas.emplace_back(std::move(*alloc), ReplicaStatus::PROCESSING)` 直接命中 Replica(DistributedFSDescriptor, ReplicaStatus) 重载，分配副本 id 并进入 PROCESSING——对应图上的 DFS(PROCESSING)。

**重点语法 / 惯用法**

- tl::expected 的 `operator*` 返回内部值引用，`std::move(*alloc)` 是移动出 descriptor 的惯用法（此处 expected 之后不再使用）。
- emplace_back 原位构造，避免先造 Replica 再移动；参数个数/类型决定重载解析到 DFS 那个构造函数。
- std::variant 在 Replica 内部以 DfsReplicaData 形式存放 descriptor，构造后类型即确定。
- enum class ReplicaStatus 必须写全限定名 `ReplicaStatus::PROCESSING`，不会隐式转 int。

**与 6.7.6 的差异**：时序图没有 DISK 副本这条分支（它只在 master 配了 --root_fs_dir 时出现，且写入由客户端异步完成）；图也没有画出 bucket/shard 两种分配器差异（bucket 模式下 shard_idx 字段其实装的是 bucket id）。图上的 `A-->>M: DistributedFSDescriptor{path,offset,size,shard_idx}` 与代码一致，但代码里 descriptor 还多一个 aligned_size 字段。

#### 2.2.5 InsertMetadata 前半：把 Replica 转成 Descriptor 列表

`mooncake-store/src/master_service.cpp:5322-5342`

```cpp
auto MasterService::InsertMetadata(
    MetadataAccessorRW& accessor, const UUID& client_id, uint64_t value_length,
    const ReplicateConfig& config, const std::string& group_id,
    const std::chrono::system_clock::time_point& now,
    const ResolvedSoftPinRequest& soft_pin_request,
    std::vector<Replica>&& replicas, uint64_t pending_quota_charge,  // 副本按值移入，此后归元数据所有
    std::optional<std::chrono::system_clock::time_point>
        committed_soft_pin_timeout)
    -> tl::expected<std::vector<Replica::Descriptor>, ErrorCode> {  // 返回给客户端的副本描述列表
    assert(!accessor.IsPublished());
    const std::string& key = accessor.GetKey();
    const TenantId& tenant_id = accessor.GetTenantId();
    const auto deadline_to_index = committed_soft_pin_timeout;

    std::vector<Replica::Descriptor> replica_list;            // 就是 PutStart 响应体里的 replicas
    std::unordered_set<ReplicaID> eligible_replica_ids;       // 供 session tag / soft pin 使用
    replica_list.reserve(replicas.size());                    // 先按数量预留，避免循环中扩容
    eligible_replica_ids.reserve(replicas.size());
    int i = 0;
    VLOG(1) << "PutStart, create replicas: client_id=" << client_id
            << ", key=" << key << ", value_length=" << value_length;
```

**这一段在做什么**

- replicas 以右值引用进入，说明调用方（AllocateAndInsertMetadata）已经放弃所有权，函数内可以随意移动。
- replica_list 与 eligible_replica_ids 分别服务两个目的：前者回给客户端，后者用于 session tag 和 soft pin 的副本集合。
- 两处 reserve 是纯性能考量，副本数最多几百，但在热路径上值得省掉几次 realloc。
- `assert(!accessor.IsPublished())` 声明了前置条件：本函数只负责“新建在途对象”，已发布的 key 必须走别的路径。
- i 是日志用的 1 基计数，后面只有 VLOG 会用到，不影响逻辑。

**重点语法 / 惯用法**

- `std::vector<Replica>&&` 右值引用形参：签名即契约，表示“我拿所有权”。
- `Replica::Descriptor` 是嵌套类型，返回值写成 vector 时要比 vector<Replica> 更注意它是按值拷贝的轻量结构。
- `int i = 0;` 配合后面 `++i` 做前置自增。
- `std::optional<std::chrono::system_clock::time_point>` 用 `if (deadline_to_index)` 判有值，后文再 `*` 取值。

**与 6.7.6 的差异**：无。时序图里 `M-->>C: replicas + DFS(PROCESSING)` 正是本函数返回的 replica_list，status 由 get_descriptor() 从 Replica 带出。

#### 2.2.6 InsertMetadata 循环：get_descriptor() 如何把 PROCESSING 状态带出去

`mooncake-store/src/master_service.cpp:5343-5366`

```cpp
    for (const auto& replica : replicas) {
        const auto desc = replica.get_descriptor();           // 按类型拷出 descriptor，status 一并带出
        replica_list.emplace_back(desc);                      // 追加到返回列表（此刻仍是 PROCESSING）
        eligible_replica_ids.insert(replica.id());            // 记录副本 id，后面绑 session/soft pin

        if (replica.is_memory_replica()) {                    // 以下分支只为打日志，按类型取细节
            const auto& mem_desc = desc.get_memory_descriptor();
            VLOG(1) << "Replica #" << ++i << ": buffer_address="  // ++i：前置自增，编号从 1 开始
                    << mem_desc.buffer_descriptor.buffer_address_
                    << ", transport_endpoint="
                    << mem_desc.buffer_descriptor.transport_endpoint_;
        } else if (replica.is_nof_replica()) {
            const auto& nof_desc = desc.get_nof_descriptor();
            VLOG(1) << "Replica #" << ++i << ": buffer_address="
                    << nof_desc.buffer_descriptor.buffer_address_
                    << ", transport_endpoint="
                    << nof_desc.buffer_descriptor.transport_endpoint_;
        } else if (replica.is_dfs_replica()) {                // DFS 副本：客户端靠这三个字段定位数据
            const auto& dfs_desc = desc.get_dfs_descriptor();
            VLOG(1) << "Replica #" << ++i << ": dfs_file=" << dfs_desc.file_path
                    << ", offset=" << dfs_desc.offset
                    << ", shard_idx=" << dfs_desc.shard_idx;  // bucket 模式下它是 bucket id
        }
    }
```

**这一段在做什么**

- get_descriptor() 只做两件事：拷贝 id_ 与 status_，再把 std::variant 里的具体 descriptor 拷出来；所以返回列表里的 DFS 副本状态恒为 PROCESSING。
- 状态是“快照”而非引用：客户端拿到的是 PutStart 那一刻的 PROCESSING，之后 master 把副本改成 COMPLETE 不会回传。
- 只有 DFS 分支一定会命中——MEMORY/NoF/DISK 副本按各自类型分别处理，DFS 用 file_path + offset + shard_idx 三项定位。
- 这里对每种副本都拷贝了一份 descriptor（desc），DFS 的拷贝只是 string + 几个整型，代价可忽略。
- eligible_replica_ids 收集全部副本 id（含 DFS），后续 session tag / soft pin 都以这个集合为单位。

**重点语法 / 惯用法**

- `const auto desc = replica.get_descriptor();` 按值接收，因为函数返回 Descriptor 而非引用。
- Descriptor 内部是 std::variant，`get_dfs_descriptor()` 用 std::get 取 DfsReplicaData，类型不符会抛 std::bad_variant_access（因此前面必须先判 is_dfs_replica）。
- `replica.is_dfs_replica()` 与静态版本 `Replica::fn_is_dfs_replica` 成对存在，后者用于 std::find_if 之类的谓词。
- `VLOG(1)` 只在 --v=1 以上输出，热路径日志用 INFO 级别的 LOG 会明显拖慢。

**与 6.7.6 的差异**：无。需要补的背景是：图上的 DFS(PROCESSING) 在客户端 PutEnd 之后才会被 master 改成 COMPLETE（PutEnd 不在本节范围），图与代码在这一点一致。

#### 2.2.7 InsertMetadata 后半：Publish、配额转正失败时的 DFS 回收、置为写入中

`mooncake-store/src/master_service.cpp:5368-5401`

```cpp
    accessor.Publish(client_id, now, value_length, std::move(replicas),  // 副本所有权移交元数据
                     std::move(committed_soft_pin_timeout),
                     config.with_hard_pin, config.data_type, group_id,
                     tenant_id, key);
    auto& metadata = accessor.Get();
    if (enable_multi_tenants_) {                              // 多租户下把“预扣”转成“实扣”
        auto adopt_result = metadata.quota_ledger.AdoptPendingCharge(
            accessor.QuotaAccount(), pending_quota_charge);
        if (!adopt_result) {
            LogTenantQuotaLedgerError(adopt_result, "adopt_pending", tenant_id,
                                      key);
            auto failed_replicas = metadata.PopReplicas();    // 失败：先把副本从元数据要回来
            FreeDfsReplicas(key, failed_replicas);            // DFS 空间不是句柄，必须显式归还
            accessor.tenant_state_->metadata.erase(accessor.it_);  // 再抹掉刚建的元数据
            accessor.it_ = accessor.tenant_state_->metadata.end();
            return tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
        }
    }
    metadata.BeginSessionTag(config.SessionId(), eligible_replica_ids);  // 副本 id 绑到会话 tag
    metadata.BeginSoftPinAction(soft_pin_request,
                                std::move(eligible_replica_ids));  // 此后集合不可再用
    if (deadline_to_index) {
        soft_pin_deadline_index_.Upsert(tenant_id.MakeScopedKey(key),
                                        GetMetadataShardIndex(metadata),
                                        *deadline_to_index);  // 登记 soft pin 到期索引
    }
    // Wire grouped objects to the group's shared lease; ungrouped objects keep
    // the per-object lease created at construction.
    if (!group_id.empty()) {
        metadata.lease_ = RegisterGroupMember(tenant_id, key, group_id);
    }
    accessor.GetState().is_processing = true;                 // key 进入“写入中”，Get 会跳过或等待

    return replica_list;                                      // 按值返回，走移动
```

**这一段在做什么**

- Publish 把 replicas 移进元数据并记录 client_id / put_start_time，此时副本状态仍是 PROCESSING——正式“可读”要等 PutEnd。
- 配额转正失败是这里唯一的回滚点，且必须调用 FreeDfsReplicas：MEMORY/NoF 有 refcount 和句柄回收，DFS 的偏移量没有句柄可依赖，只能显式 Free。
- 回滚顺序很讲究：PopReplicas → FreeDfsReplicas → erase 元数据 → 返回 INTERNAL_ERROR，避免出现“元数据已删但 DFS 空间泄漏”。
- BeginSessionTag / BeginSoftPinAction 都消费 eligible_replica_ids，后者用 std::move 之后集合失效，不能再读。
- 最后把 is_processing 置 true，这是 Get 侧判断“对象正在写、不能读”的依据；PutEnd 会把它改回 false。

**重点语法 / 惯用法**

- `accessor.Get()` 返回元数据引用，之后的 Publish/PopReplicas 都作用在同一个对象上。
- `std::move(eligible_replica_ids)` 再次体现“移动即让渡”，配合形参 `std::unordered_set<ReplicaID>` 或 const& 由被调方决定。
- `accessor.it_` 是 shard 内 map 的迭代器，erase 后必须重置为 end() 才算失效安全。
- `return replica_list;` 触发 NRVO/移动，不会深拷贝整个 vector。

**与 6.7.6 的差异**：时序图没有画 Publish 之后的租户配额转正及其回滚，也没有画 session tag / soft pin / group lease 这些元数据副作用；这些都不会改变返回给客户端的 descriptor 内容。

#### 2.2.8 AdmitWithDfsRecovery：DFS 容量耗尽时的重试闸门

`mooncake-store/src/master_service.cpp:5562-5587`

```cpp
auto MasterService::AdmitWithDfsRecovery(
    const ReplicateConfig& config,
    const std::function<tl::expected<std::vector<Replica::Descriptor>,
                                     ErrorCode>(bool* dfs_allocation_failed)>&
        admit) -> tl::expected<std::vector<Replica::Descriptor>, ErrorCode> {
    // Set by AllocateReplicas only when DFS allocation itself failed with
    // capacity exhaustion; gates the forced bucket eviction retry below so
    // memory/NoF exhaustion cannot delete unrelated DFS data.
    bool dfs_allocation_failed = false;                       // 局部变量跨多次 admit 复用
    auto result = admit(&dfs_allocation_failed);              // 第一次尝试
    // Eviction scans metadata shards, so recover only after admit releases
    // the key. Serialize recovery and re-admit before deciding whether this
    // request still needs to evict one bucket.
    if (!result && dfs_allocation_failed && config.dfs_replica_num > 0 &&
        bucket_allocator_ != nullptr) {                       // 四个条件全满足才谈“淘汰恢复”
        std::lock_guard recovery_lock(dfs_bucket_recovery_mutex_);  // CTAD，锁到函数结束
        dfs_allocation_failed = false;
        result = admit(&dfs_allocation_failed);               // 第二次：也许别人已腾出 bucket
        if (!result && dfs_allocation_failed &&
            TryRecoverDfsSpaceAfterAllocationFailure()) {     // 真删掉了东西才值得再试
            dfs_allocation_failed = false;
            result = admit(&dfs_allocation_failed);           // 第三次：用刚回收的空间
        }
    }
    return result;
}
```

**这一段在做什么**

- 重试的四个条件：admit 失败、失败原因是 DFS 容量耗尽（dfs_allocation_failed）、请求确实要 DFS 副本、且处于 bucket 模式（bucket_allocator_ 非空）。
- 把“DFS 容量满”和“内存/NoF 分配失败”严格区分开：后者不会进入这条恢复路径，避免为了写不进 DFS 而删掉别人的 DFS 数据。
- 流程最多三次 admit：直接试 → 拿锁后再试（别人可能已释放）→ 强制回收一个 bucket 后再试。
- dfs_bucket_recovery_mutex_ 把恢复串行化，防止并发 Put 同时扫描元数据 shard、把多个 bucket 一起删掉。
- 注释明确说明为什么必须在 admit 返回（释放 key 锁）之后才做回收：淘汰要扫描元数据 shard，持锁扫描会互锁。

**重点语法 / 惯用法**

- std::function<...> 作为形参把 lambda（PutStart 里的 admit）擦除成可多次调用的对象；每次调用都要传 bool*。
- `std::lock_guard recovery_lock(mutex)` 是 C++17 类模板实参推导（CTAD），不用写 std::lock_guard<std::mutex>。
- `!result` 直接作用于 tl::expected，等价于 `!result.has_value()`；`result.error()` 在这里没被读取。
- 递归式的 `result = admit(...)` 复用同一个 result 变量，最终只返回最后一次结果。

**与 6.7.6 的差异**：时序图完全没有画这层重试：`C->>M: PutStart` 到 `M-->>C: replicas` 之间在 bucket 模式下可能是三次 admit（两次普通重试 + 一次强制回收 bucket），且回收时会同步删掉其他 key 的 DFS 数据。

### 2.3 写 DFS：Client::WriteDfsReplicas（第 7 步）

图上 `C->>BE: WriteDfsReplicas → BatchWrite(descriptor, slices)` 的 Client 半边。

#### 2.3.1 入口契约：三向量等长校验 + 后端缺失的整批失败

`mooncake-store/src/client_service.cpp:2884-2904`

```cpp
std::vector<ErrorCode> Client::WriteDfsReplicas(              // 逐 key 返回结果，与 keys 等长；不抛异常
    const std::vector<std::string>& keys,                     // 本批要写 DFS 的 key
    const std::vector<const std::vector<Slice>*>& slice_lists, // 借用调用方 slices，不拷贝不拥有
    const std::vector<DistributedFSDescriptor>& descriptors) { // master Allocate 下发的落点
    if (keys.size() != slice_lists.size() ||                  // 三向量必须等长，否则下标错位
        keys.size() != descriptors.size()) {                  // 少校验一个就会越界读
        return std::vector<ErrorCode>(keys.size(), ErrorCode::INVALID_PARAMS);
    }                                                         // 整批标错，不做部分成功
    if (keys.empty()) {                                       // 空批直接返回，不碰后端
        return {};
    }
    if (!dfs_storage_backend_) {                              // DFS 未配置或初始化失败
        LOG(ERROR) << "DFS backend is unavailable for synchronous write";
        if (auto* dfs_metric = GetDfsMetricPtr()) {           // C++17 if-init，作用域限本 if
            dfs_metric->RecordWriteErrors(                    // 按 key 数计错，不是按批
                toString(ErrorCode::DFS_SERVICE_UNAVAILABLE),
                static_cast<int64_t>(keys.size()));           // size_t 显式转 int64
        }
        return std::vector<ErrorCode>(keys.size(),            // 每个 key 都判失败
                                      ErrorCode::DFS_SERVICE_UNAVAILABLE);
    }
```

**这一段在做什么**

- 返回值是按 keys 下标对齐的 ErrorCode 数组，调用方（Put / Upsert / SubmitDfsWrites）靠它逐 key 记 transfer_summary。
- 三个入参向量必须一一对应：调用方是「按 DFS 副本循环」push 的，所以同一 key 理论上可以出现多次；当前 master 限制 `dfs_replica_num ≤ 1`，实际每个 key 只出现一次。
- dfs_storage_backend_ 为空属于配置问题，直接整批失败并落 DFS_SERVICE_UNAVAILABLE 指标，不会退化成写内存副本。
- 空 keys 直接返回空 vector，chunk 到 BatchPut 的单 key 路径时不会有额外开销。
- 函数是同步的：返回时 BatchWrite 已经结束，但那只代表数据交到了文件系统/VFS，不是 fsync 级别的持久化。

**重点语法 / 惯用法**

- std::vector<ErrorCode>(n, value)：值填充构造，一次分配 n 个相同初值，比 push_back 循环少一次扩容。
- if (auto* p = f(); cond)：C++17 if-init，p 只在 if 语句（含 else）内可见，避免污染外层命名空间。
- static_cast<int64_t>(keys.size())：size_t 是无符号，直接传给 int64 参数会触发窄化告警，显式转换表意。
- toString(ErrorCode)：把枚举拼成指标 label 字符串，故意不把 types.h 拉进 metric 头文件。

**与 6.7.6 的差异**：6.7.6 图上箭头写作 `C->>BE: WriteDfsReplicas → BatchWrite`，把 Client 内部函数与后端调用压成了一根箭头；实际 WriteDfsReplicas 是 Client 成员，从它到 BE 只有 BatchWrite 一次调用。图上也完全没有“dfs_storage_backend_ 为空 → 整批 DFS_SERVICE_UNAVAILABLE”这条失败分支。

#### 2.3.2 请求容器、旁路字节统计与 staging 计时起点

`mooncake-store/src/client_service.cpp:2906-2933`

```cpp
    auto* dfs_metric = GetDfsMetricPtr();                     // 可为空：指标没开就全程判空
    std::vector<ErrorCode> results(keys.size(), ErrorCode::OK); // 默认 OK，失败处覆盖
    std::vector<DfsWriteRequest> requests;                    // 真正送进 BatchWrite 的请求
    std::vector<size_t> request_indices;                      // 请求序 → 原始 key 序反查表
    // Bytes per request, so the success accounting after BatchWrite does not
    // have to walk the slice lists a second time.
    std::vector<uint64_t> request_bytes;                      // 旁路缓存每 key 字节数
    std::vector<PinnedBufferPool::Buffer> staging_buffers;    // 持有全部 D2H 暂存块
    requests.reserve(keys.size());                            // 一次预留，循环内不再扩容
    request_indices.reserve(keys.size());
    request_bytes.reserve(keys.size());

    // Staging is the device-to-host copy below; it is timed separately from the
    // BatchWrite call so a slow put can be attributed to the right side of the
    // boundary between them.
    const auto staging_start = std::chrono::steady_clock::now(); // 单调时钟，不受改时影响
    bool staging_performed = false;                           // 纯 host 指针时不上报 staging
    auto runtime_accelerator =
        device::GetAcceleratorRegistry().RuntimeAccelerators(); // 无加速器则全按 host 处理
    for (size_t i = 0; i < keys.size(); ++i) {                // 串行预处理，顺序同 keys
        if (slice_lists[i] == nullptr) {                      // 空指针是调用方编程错误
            results[i] = ErrorCode::INVALID_PARAMS;           // 只废这个 key，其余照常
            if (dfs_metric) {
                dfs_metric->RecordWriteErrors(
                    toString(ErrorCode::INVALID_PARAMS));     // count 默认 1
            }
            continue;                                         // 不进 requests，故无下标
        }
```

**这一段在做什么**

- results 先整体填 OK，仅对失败的 key 覆盖，所以“没进 requests”的 key 一定是被显式标错过的，不会静默成功。
- request_indices 是回填的关键：requests 只包含预处理成功的 key，其下标序与 keys 不同，绝不能直接用 i 回填。
- request_bytes 是给成功统计省一次遍历的缓存，不参与 BatchWrite 本身。
- staging_buffers 声明在函数作用域而不是循环内，它决定了暂存块的释放时机（见最后一个 block）。
- staging_performed 只是“本轮是否发生过 D2H”的开关，纯 host 写入不应污染 staging 直方图。
- reserve 三次：因为 requests 的地址被后面 BatchWrite 以 const 引用引用，提前预留能避免扩容时搬移（虽非必需，但让容量与内容确定）。

**重点语法 / 惯用法**

- std::chrono::steady_clock：单调时钟，语义上比 system_clock 更适合测耗时，不会被 NTP 调整影响。
- device::GetAcceleratorRegistry().RuntimeAccelerators()：返回按值持有的代理对象，不是裸指针，退出作用域自动解绑。
- std::vector<PinnedBufferPool::Buffer>：Buffer 拷贝构造被 delete、只提供 noexcept 移动，所以容器只能靠 std::move 搬运。
- LOG(ERROR) << ...：glog 流式宏，作用域结束时析构才真正落盘。

**与 6.7.6 的差异**：图上没有“staging 与 write 分开计时”这件事，也看不出 request_bytes / request_indices 这类旁路结构与逐 key 参数校验；图把一次 BatchWrite 画成 Put 的直接子步骤，而代码里它前面还有一整轮预处理。

#### 2.3.3 host_slices 初始化与设备指针判定：为什么只拷设备指针

`mooncake-store/src/client_service.cpp:2935-2949`

```cpp
        std::vector<Slice> host_slices;                       // 最终交给后端的 slice 列表
        host_slices.reserve(slice_lists[i]->size());          // 数量与用户 slices 一致
        uint64_t key_bytes = 0;                               // 只累计真正提交的字节
        bool staging_succeeded = true;                        // 任一 slice 失败即整 key 失败
        for (const auto& slice : *slice_lists[i]) {           // 逐 slice 判断要不要 D2H
            device::PointerInfo info{};                       // 值初始化，device_id 默认 -1
            auto* device = slice.ptr == nullptr
                               ? nullptr
                               : runtime_accelerator.FindDeviceForPointer(
                                     slice.ptr, &info);       // 非空 = 设备指针，回填卡号
            if (device == nullptr) {                          // host 指针（或空指针）
                key_bytes += slice.size;
                host_slices.push_back(slice);                 // 零拷贝：直接沿用原地址
                continue;
            }
```

**这一段在做什么**

- 这一段是“为什么只有设备指针才拷贝”的核心：host_slices 对 host slice 是原样复制同一个 Slice（指针+长度），数据零拷贝直达文件 I/O。
- FindDeviceForPointer 返回非空即代表该地址落在某块加速器设备内存里，同时把 device_id 写进 info；返回空则说明是 host 或未注册内存。
- slice.ptr == nullptr 时不会去查设备，避免把空指针喂给设备查询接口。
- key_bytes 累加的是每个 slice 的 size，与是否拷贝无关，保证成功指标里的字节数与用户提交量一致。
- staging_succeeded 是 per-key 的标志，break 之后不会再为这个 key 拼出半截 requests。

**重点语法 / 惯用法**

- device::PointerInfo info{}：花括号值初始化，把所有成员初始化到 0/-1，避免未初始化 device_id 被 SetContext 误用。
- 条件表达式返回指针：两个分支都是指针类型，编译器不会插入隐式转换，nullptr 分支类型推导为 const AcceleratorDevice*。
- host_slices.push_back(slice)：Slice 是平凡可拷贝的 POD（ptr+size），按值塞进去等于“记下同一块内存”。

**与 6.7.6 的差异**：图上把 `C->>BE: WriteDfsReplicas` 画成一次透明转发，没有体现“只有设备指针才拷贝、host 指针原样透传”这个分支；也看不出 DFS 读端只接受 host 虚拟地址这一约束在这里被写入端“兼容”。

#### 2.3.4 D2H staging：Acquire 暂存、Copy、失败回滚与请求构造

`mooncake-store/src/client_service.cpp:2951-2985`

```cpp
            staging_performed = true;                         // 出现过设备指针才记 staging
            device->SetContext(info.device_id);               // 多卡机器先切到该 slice 的卡
            auto buffer = pinned_buffer_pool_->Acquire(slice.size); // 从池里借，非每次新建
            if (!device->Copy(buffer.data, slice.ptr, slice.size,
                              device::CopyDirection::kDeviceToHost)) { // 同步 D2H
                LOG(ERROR) << "DFS D2H staging failed, action=stage, key="
                           << keys[i] << ", size=" << slice.size
                           << ", device_id=" << info.device_id;
                pinned_buffer_pool_->Release(std::move(buffer)); // 失败块立刻归还
                results[i] = ErrorCode::TRANSFER_FAIL;
                staging_succeeded = false;
                break;                                        // 放弃本 key，先跳过余下 slice
            }
            key_bytes += slice.size;
            host_slices.emplace_back(Slice{buffer.data, slice.size}); // 指向暂存区
            staging_buffers.push_back(std::move(buffer));     // 所有权上交函数作用域
        }
        if (!staging_succeeded) {
            // Counted as a DFS write failure: the key was destined for DFS and
            // never got there, even though the backend was never reached.
            if (dfs_metric) {
                dfs_metric->RecordWriteErrors(
                    toString(ErrorCode::TRANSFER_FAIL));     // 后端没被调用也要记错
            }
            continue;                                         // 本 key 不进 requests
        }

        requests.push_back(
            DfsWriteRequest{keys[i], descriptors[i], std::move(host_slices)}); // 聚合初始化
        request_indices.push_back(i);                         // 记住回填位置
        request_bytes.push_back(key_bytes);                   // 供成功统计复用
    }
    if (dfs_metric && staging_performed) {
        dfs_metric->ObserveWriteStaging(elapsed_us_since(staging_start)); // 批次级 staging
    }
```

**这一段在做什么**

- Acquire 是从池里拿一块容量 ≥ size 的旧块，池空才真的分配 pinned 内存；缓冲区可跨 key/跨调用复用，这是高频小写入不被打爆的关键。
- SetContext 必须在 Copy 之前：多卡机器上不切卡，D2H 会拷错卡或直接失败。
- host_slices 里存的是 buffer.data 裸指针，所以 Buffer 必须以 std::move 搬到 staging_buffers，绝不能在这里析构或 Release。
- D2H 失败时只 Release 当前失败的那块，同一 key 之前已成功拷好的块仍留在 staging_buffers 里，最后统一归还——不会泄漏，但也不会立刻回收。
- staging 耗时用 staging_performed 做闸门：整批都是 host 指针时不上报，直方图里不会出现一堆 0。
- 请求对象是放在循环末尾才 push 的，所以中途失败/continue 的 key 根本不会出现在 requests 里，与 request_indices 天然对齐。

**重点语法 / 惯用法**

- PinnedBufferPool::Buffer 的拷贝构造/赋值被 delete，只有 noexcept 移动构造；push_back(std::move(buffer)) 之后 buffer 处于“已搬空”状态（data 被置 nullptr）。
- emplace_back(Slice{...})：先构造临时 Slice 再移动进容器，效果等同 push_back，写成 emplace 只是省一次显式类型名重复。
- DfsWriteRequest{key, desc, std::move(v)}：聚合初始化 + 移动，把 host_slices 的 vector 缓冲直接过继给请求。
- device::CopyDirection::kDeviceToHost：枚举限定名，防止与 H2D 方向混用。
- elapsed_us_since(...)：工具函数返回 uint64_t 微秒数，内部做 duration_cast，避免每个调用点重写一行。

**与 6.7.6 的差异**：6.7.6 图上根本没有 D2H staging 这一步：`C->>C: 先写 MEMORY / NoF` 之后就直接 `C->>BE: WriteDfsReplicas`。对 GPU 张量，代码在这里还有一次逐 slice 的 pinned host 拷贝，图上应补一条 C 内部的自环。另外图没画“D2H 失败 → 该 key 记 TRANSFER_FAIL 且不进 requests”的分支。

#### 2.3.5 一次 BatchWrite、返回条数校验与逐 key 结果回填

`mooncake-store/src/client_service.cpp:2987-3015`

```cpp
    const auto write_start = std::chrono::steady_clock::now(); // 后端计时起点，不含 staging
    auto write_results = dfs_storage_backend_->BatchWrite(requests); // 一次批写，同步返回
    const auto write_latency_us = elapsed_us_since(write_start);
    int64_t success_keys = 0;
    uint64_t success_bytes = 0;
    if (write_results.size() != requests.size()) {            // 后端契约：逐条返回
        LOG(ERROR) << "DFS BatchWrite response size mismatch: expected "
                   << requests.size() << ", got " << write_results.size();
        for (size_t index : request_indices) {                // 无法定位到单条，整批降级
            results[index] = ErrorCode::INTERNAL_ERROR;
        }
        if (dfs_metric && !request_indices.empty()) {
            dfs_metric->RecordWriteErrors(
                toString(ErrorCode::INTERNAL_ERROR),
                static_cast<int64_t>(request_indices.size())); // 按条数累计
        }
    } else {
        for (size_t i = 0; i < write_results.size(); ++i) {   // i 是请求序，不是 key 序
            results[request_indices[i]] =                     // 用反查表还原原始下标
                write_results[i] ? ErrorCode::OK : write_results[i].error();
            if (write_results[i]) {
                ++success_keys;
                success_bytes += request_bytes[i];            // 用缓存值，不再遍历 slices
            } else if (dfs_metric) {
                dfs_metric->RecordWriteErrors(
                    toString(write_results[i].error()));      // 每次失败落一个 label
            }
        }
    }
```

**这一段在做什么**

- write_start 在 BatchWrite 之前才取，staging 耗时不会算进 dfs_write_latency：慢 put 能被拆成“卡在 D2H”还是“卡在文件系统”。
- 返回值是 vector<tl::expected<void, ErrorCode>>，长度必须等于请求数；不等说明后端违约，只能整批判 INTERNAL_ERROR，因为无法知道哪条对应哪条。
- 成功计数的下标是 request 序 i，回填 results 用的是 request_indices[i]，两者不能混用。
- 失败时错误码来自后端（INVALID_PARAMS / FILE_WRITE_FAIL / DFS_SERVICE_UNAVAILABLE 等），逐条按 label 记进 dfs_write_errors。
- BatchWrite 本身是同步的：函数返回时数据已经 pwritev/提交给 3FS，但既不 fsync 也不改 master 元数据——DFS 副本状态还停在 PROCESSING，等 PutEnd 才置 COMPLETE。

**重点语法 / 惯用法**

- tl::expected<void, ErrorCode> 的布尔转换：write_results[i] 为真表示无错误，取 .error() 前必须先判真（否则是 UB）。
- for (size_t index : request_indices)：范围 for 遍历 size_t 容器，避免手写下标。
- auto write_results = f(...)：返回 vector<expected> 的按值复制（NRVO/移动），不要写 auto&& 绑到临时。
- dfs_metric && !request_indices.empty()：短路求值，避免空批时给计数器 inc(0)。

**与 6.7.6 的差异**：图上 `C->>BE: WriteDfsReplicas → BatchWrite(descriptor,slices)` 暗示“一个 Put 一次 BatchWrite”；实际单 key 的 Put 走的是 WriteDfsReplicas(dfs_keys=1)，而 BatchPut 走 SubmitDfsWrites 把多个 op 的 key（以及同一 key 的多个 DFS 副本）攒成一次 BatchWrite。图上也没有“返回条数不等 → 整批 INTERNAL_ERROR”的契约校验。

#### 2.3.6 指标口径、缓冲释放时机与函数收尾

`mooncake-store/src/client_service.cpp:3016-3033`

```cpp
    // Latency is per batch, so it is only observed when the batch delivered
    // something; a fully failed batch would otherwise pollute the histogram
    // with error-path timings.
    if (dfs_metric && success_keys > 0) {                     // 全失败不记延迟
        dfs_metric->ObserveWrite(success_keys,
                                 static_cast<int64_t>(success_bytes),
                                 write_latency_us);           // ops/bytes 只算成功的
    }
    VLOG(1) << "DFS batch write finished, action=write, requested="
            << requests.size() << ", succeeded=" << success_keys
            << ", bytes=" << success_bytes
            << ", latency_us=" << write_latency_us;

    for (auto& buffer : staging_buffers) {                    // 手工 RAII：此处才归还
        pinned_buffer_pool_->Release(std::move(buffer));      // BatchWrite 已返回，可安全回收
    }
    return results;
}
```

**这一段在做什么**

- ObserveWrite 只在 success_keys > 0 时调用：直方图描述的是“成功交付的批”的延迟，全失败批次会污染分位数。
- dfs_write_ops / dfs_write_bytes 只累计成功条数和字节，失败量走 dfs_write_errors，两条线不重叠。
- staging_buffers 的释放是本函数最后一步——这就是“为什么不能释放太早”：host_slices 里的裸指针在整个 BatchWrite 期间被后端引用，提前释放就是 use-after-free。
- 这里用的是手工 for 循环释放而不是智能指针，是因为 Buffer 需要回收到 PinnedBufferPool 而不是直接 free；如果中途抛异常（本函数实际不抛），这块内存会随 Buffer 析构被释放，不会泄漏。
- 返回的 results 已覆盖所有 key：成功、参数错、D2H 失败、后端失败、条数不匹配，五种来源共同写入同一数组。

**重点语法 / 惯用法**

- VLOG(1)：glog 的 verbose 级别日志，默认不输出，需 --v=1 才可见，用于不打扰生产的排障。
- for (auto& buffer : staging_buffers) + std::move(buffer)：取引用再搬走其内容，容器本身在函数末尾析构，不会重复释放。
- Release(Buffer buf) 按值收参：调用方必须显式 std::move，否则编译失败（拷贝被 delete），这是一道防误用的类型护栏。

**与 6.7.6 的差异**：图上没有指标语义这一层，也没有画出“staging 缓冲一直活到 BatchWrite 返回之后才释放”的生命周期；`C-->>App: 返回（WriteAt 已完成，但不是 fsync）` 与代码一致（Put 在 WriteDfsReplicas 返回后才 UpsertEnd），这点无需修正。

### 2.4 数据面编排：DistributedStorageBackend::BatchWrite（第 7–8 步）

图把 `WriteDfsReplicas → BatchWrite` 画成一跳；这里看清 BatchWrite 内部如何校验、归一化并选择执行器。

#### 2.4.1 BatchWrite 的入口：results 预分配与两次整批短路

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:633-648`

```cpp
std::vector<tl::expected<void, ErrorCode>>
DistributedStorageBackend::BatchWrite(
    const std::vector<DfsWriteRequest>& requests) {
    // 先按请求数分配；expected<void,E> 默认构造即"成功"，等于先假定全成功
    std::vector<tl::expected<void, ErrorCode>> results(requests.size());

    if (UsesObjectStorage()) {          // oss/s3 模式没有 descriptor 语义
        results.assign(requests.size(),
                       tl::make_unexpected(ErrorCode::NOT_SUPPORTED));
        return results;                 // 整批失败，但长度与 requests 对齐
    }
    if (!initialized_) {                // Init() 未完成：adapter/fd 都不可用
        LOG(ERROR) << "DistributedStorageBackend is not initialized";
        results.assign(requests.size(),
                       tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE));
        return results;                 // 用可重试错误码，区别于 INVALID_PARAMS
    }
```

**这一段在做什么**

- results 一次性按请求数构造，默认值就是 success，因此后续只需覆盖失败位置。
- 真正的整批短路只有两处：UsesObjectStorage 与 !initialized_；第三个提前返回是循环后的 `if (!batch_io) return results;`，属于正常出口而非错误短路。
- 两种短路的错误码刻意不同：NOT_SUPPORTED 表示部署不支持，DFS_SERVICE_UNAVAILABLE 表示暂时不可用、可重试。
- 截断返回必须用 assign 铺满整批而不是返回空 vector，否则调用方 WriteDfsReplicas 的 size 校验会把它升级成 INTERNAL_ERROR。
- 此函数全程不抛异常，所有错误都编码在 returned vector 里，逐 key 结算。

**重点语法 / 惯用法**

- tl::expected<void, ErrorCode>：value 类型是 void，默认构造即代表"无错误"。
- vector::assign(n, value)：把 vector 重置成 n 份同一错误值，比 clear+push 更直接。
- tl::make_unexpected(e)：构造 expected 的 error 分支，不能把 ErrorCode 直接赋给 expected。
- 按值返回 vector 依赖 NRVO/移动语义，不会深拷贝 results。

**与 6.7.6 的差异**：图上 C->>BE 直接进 BatchWrite(descriptor,slices)。代码里 BE 可能是对象存储 adapter（oss/s3）或尚未 Init，这两种情况整批直接失败、根本不碰 FS；图上没有这两个前置分支。另外 DfsWriteRequest 实际是 {key, descriptor, slices} 三元组，key 只服务于日志与指标。

#### 2.4.2 两个正交开关：bucket_mode 与 batch_io，以及逐请求的 descriptor 校验

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:650-677`

```cpp
    // 布局开关：bucket 用临时 fd 且补齐短写；shard 复用缓存 fd 且短写即失败
    // Storage layout: bucket I/O uses temporary fds and completes short writes;
    // shard I/O reuses cached fds and fails short writes.
    const bool bucket_mode = UsesBucketAllocator();
    // 执行开关：3FS 这类 adapter 先只做校验，循环外统一提交；POSIX 循环内逐条写
    // Execution mode: adapters that opt in (currently 3FS) collect validated
    // writes for submission after the loop. Other adapters (including POSIX)
    // execute each write in the loop, preserving per-request bucket handles
    // and shard I/O locking.
    const bool batch_io = fs_adapter_->SupportsBatchIo();
    std::vector<DfsIoOp> ops;                  // 仅批量路径使用的待提交队列
    if (batch_io) ops.reserve(requests.size());  // 逐条路径永远为空，不必预留
    for (size_t i = 0; i < requests.size(); ++i) {
        const auto& request = requests[i];      // 引用：下面 op 也引用它的成员
        const auto& desc = request.descriptor;
        const bool range_valid =                // 两套布局用不同容量口径
            bucket_mode
                ? IsBucketDescriptorRangeValid(desc, distributed_config_)
                : IsDfsDescriptorRangeValid(desc, distributed_config_);
        if (!range_valid) {
            LOG(ERROR) << "Invalid DFS descriptor range for key " << request.key
                       << ", offset=" << desc.offset
                       << ", object_size=" << desc.object_size
                       << ", aligned_size=" << desc.aligned_size
                       << ", capacity="
                       << (bucket_mode ? distributed_config_.bucket_capacity
                                       : distributed_config_.shard_capacity);
            results[i] = tl::make_unexpected(ErrorCode::INVALID_PARAMS);
            continue;                           // 单条失败不拖累同批其他 key
        }
```

**这一段在做什么**

- bucket_mode 决定"用什么 fd、短写怎么处理"，batch_io 决定"循环内写还是循环外统一提交"，两者正交、可组合出四种路径。
- SupportsBatchIo() 是 FileSystemAdapter 的虚函数，基类默认返回 false；只有 3FS adapter 重写为 true，因此同一份 BatchWrite 在 POSIX 与 3FS 上走不同执行流。
- desc 是 request.descriptor 的 const 引用，稍后 DfsIoOp 还会再存一份引用，因此 requests 与 request 必须活得比 op 更久。
- 校验函数与日志里的 capacity 用的是同一个三元表达式，保证"按哪套容量拒绝"和"按哪套容量打印"一致。
- 校验失败只写 results[i] 后 continue，批量写不是事务，同批其他 key 照常落盘。
- 整个循环没有任何文件系统调用，纯内存判断，因此在拿锁/打开 fd 之前就能剔除坏请求。

**重点语法 / 惯用法**

- const auto& request = requests[i]：避免拷贝 DfsWriteRequest（内含 std::string 与 vector<Slice>）。
- 条件运算符用于初始化 const bool range_valid，两个分支都是函数调用，靠共同的 bool 返回类型收敛。
- UsesBucketAllocator() 是 const 成员函数，内部只比较 allocator_type == "bucket"，无锁无状态。
- if (batch_io) ops.reserve(...)：条件预留，避免非批量路径分配一块永远用不上的内存。

**与 6.7.6 的差异**：图把 `BE->>FS: GetOrOpenShard + BatchWriteAt` 画成唯一一条 FS 路径。实际是 shard/bucket × 批量/逐条 的四种组合（3FS 走批量，POSIX 走逐条 ExecuteSingleIo），GetOrOpenShard 只在 shard 模式出现。图上也完全没有 descriptor 几何校验这一步。

#### 2.4.3 slice 校验、DfsIoOp 构造与两种执行分派

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:679-720`

```cpp
        // 下推给 readv/writev 的 iovcnt 是 int，超过会截断成负数或小值
        if (request.slices.size() >
            static_cast<size_t>(std::numeric_limits<int>::max())) {
            results[i] = tl::make_unexpected(ErrorCode::INVALID_PARAMS);
            continue;
        }
        std::vector<iovec> iovs;                // Slice → iovec 的本地副本
        iovs.reserve(request.slices.size());
        uint64_t total_size = 0;
        bool invalid = false;
        for (const auto& slice : request.slices) {
            // 空指针 + 0 长度合法；空指针 + 非零长度一定是调用方传错
            if ((!slice.ptr && slice.size > 0) ||
                slice.size >
                    std::numeric_limits<uint64_t>::max() - total_size) {
                invalid = true;                 // 空指针或溢出，跳出后统一报错
                break;
            }
            total_size += slice.size;
            iovs.push_back({slice.ptr, slice.size});
        }
        // 必须严格相等：多一个字节也说明 slice 列表与 descriptor 不匹配
        if (invalid || total_size != desc.object_size) {
            LOG(WARNING) << "Invalid DFS write request for key " << request.key
                         << ", expected=" << desc.object_size
                         << ", actual=" << total_size;
            results[i] = tl::make_unexpected(ErrorCode::INVALID_PARAMS);
            continue;
        }
        DfsIoOp op{i, request.key, desc, std::move(iovs), total_size};  // 后两项为引用
        if (batch_io) {
            ops.push_back(std::move(op));       // 连 iovs 一起搬进 ops，op 被移空
        } else {
            results[i] = ExecuteSingleIo(op, true);  // POSIX 等：立刻写完并回填
        }
    }

    if (!batch_io) return results;              // 逐条路径已全部回填，提前返回
    if (bucket_mode) {
        ExecuteBucketBatchIo(ops, results, true);
    } else {
        ExecuteShardBatchIo(ops, results, true);
    }
    return results;                             // 结果按 op.index 回填，长度不变
```

**这一段在做什么**

- slice 个数上限取 INT_MAX，因为 FdIoRequest.iovcnt 与 readv/writev 的 iovcnt 都是 int。
- 空 slice（ptr==nullptr 且 size==0）放行并计入 iovs，但 ptr==nullptr 且 size>0 直接判非法。
- total_size 用 uint64 累加，并在加法前用 max()-total_size 做余量判断；最后必须严格等于 desc.object_size，多写一个字节也算错。
- DfsIoOp 里 key 与 descriptor 是 const 引用，绑定到 requests[i] 的成员；iovs 被 move 进 op，避免二次拷贝 iovec 数组。
- 批量路径只攒 op、循环结束后按 bucket_mode 分派到 ExecuteBucketBatchIo / ExecuteShardBatchIo；逐条路径当场调 ExecuteSingleIo 并把结果写回 results[i]。
- 两条路径都通过 results[op.index] 回填，index 是 requests 的下标而不是 ops 的下标，因此返回 vector 与入参严格对齐。

**重点语法 / 惯用法**

- 聚合初始化 iovs.push_back({slice.ptr, slice.size})：iovec 是 POD 聚合体，可直接花括号构造。
- 无符号溢出不会 UB 而是回绕，所以必须显式做 max()-total_size 预判，否则 total_size 会绕过相等性检查。
- std::move(op) 转移 vector<iovec> 的所有权；DfsIoOp 因含引用成员而不可拷贝赋值，只能移动/构造。
- if (!batch_io) return results; 属于 early-return 惯用法，使后段代码不必再判一次开关。
- static_cast<size_t>(std::numeric_limits<int>::max())：把 int 上限转成 size_t 再比，规避有符号/无符号比较告警。

**与 6.7.6 的差异**：图上 `BatchWrite(descriptor,slices)` 之后直接是 `GetOrOpenShard + BatchWriteAt`。实际中间还有：slice 数量上限、sum(slices) 必须严格等于 object_size、以及按 SupportsBatchIo() 决定是否攒批。bucket 模式落盘用的是 OpenBucket（每批最多 kMaxOpenBucketsPerBatch=64 个）而不是 GetOrOpenShard。

#### 2.4.4 DfsIoOp 的引用语义与逐条执行路径 ExecuteSingleIo

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:446-479`

```cpp
struct DistributedStorageBackend::DfsIoOp {
    size_t index;  // Position in the caller's request batch.
    const std::string& key;                    // 引用：指向 requests[i].key
    const DistributedFSDescriptor& descriptor; // 引用：指向 requests[i].descriptor
    std::vector<iovec> iovs;                   // 拥有 iovec 数组本体
    uint64_t size;                             // 期望字节数，用于短写判定
};

tl::expected<void, ErrorCode> DistributedStorageBackend::ExecuteSingleIo(
    DfsIoOp& op, bool write) {
    if (UsesBucketAllocator()) {               // bucket：一次性 fd，必须 close
        auto fd = OpenBucket(op.descriptor);
        if (!fd) return tl::make_unexpected(fd.error());
        auto result = TransferAll(*fs_adapter_, *fd, std::move(op.iovs),
                                  static_cast<int64_t>(op.descriptor.offset),
                                  op.size, 0, write);   // 从已传 0 字节开始补
        auto close_result = fs_adapter_->CloseFile(*fd);
        if (!result) return result;            // 写失败优先于 close 失败上报
        return close_result;
    }

    auto shard = GetOrOpenShard(op.descriptor);  // shard：走缓存 fd
    if (!shard) {
        LOG(ERROR) << "Failed to open DFS shard for key " << op.key << ": "
                   << shard.error();
        return tl::make_unexpected(shard.error());
    }
    std::lock_guard lock((*shard)->mutex);     // 同一 shard 的 I/O 在此串行化
    const auto count = static_cast<int>(op.iovs.size());
    const auto offset = static_cast<int64_t>(op.descriptor.offset);
    auto result =
        write
            ? fs_adapter_->WriteAt((*shard)->fd, op.iovs.data(), count, offset)
            : fs_adapter_->ReadAt((*shard)->fd, op.iovs.data(), count, offset);
```

**这一段在做什么**

- DfsIoOp 的两个引用成员是整套设计的核心：批量时不复制 key/descriptor，代价是 op 生命周期受限于 requests。
- index 记录的是调用方 requests 的下标，而不是 ops 里的下标，因此结果能直接回填到对齐的位置。
- 逐条路径按布局分裂：bucket 用临时 fd + TransferAll 续写短写；shard 用缓存 fd 并持 shard->mutex 串行化。
- shard 分支不补短写——*result != op.size 直接返回 FILE_WRITE_FAIL，缓存 fd 下的短写被视为真实故障。
- 只有 SupportsFdReopen() 的 adapter（POSIX）才会在失败后重开一次 shard 重试（紧随其后的代码），批量路径完全不重试。
- bucket 分支里写失败优先于 close 失败返回，避免掩盖真正的 I/O 错误。

**重点语法 / 惯用法**

- struct DistributedStorageBackend::DfsIoOp 是嵌套类型的类外定义：头文件只前置声明 `struct DfsIoOp;`，定义藏在 .cpp 里。
- const std::string& / const DistributedFSDescriptor& 作为成员：使类型不可默认构造、不可拷贝赋值，只能聚合初始化。
- std::lock_guard lock((*shard)->mutex)：CTAD 推导模板参数，额外括号避免 most vexing parse。
- std::move(op.iovs) 传值给 TransferAll：补写需要就地修改 iov_base/iov_len，所以必须转移所有权。
- 聚合初始化 {i, request.key, desc, std::move(iovs), total_size} 把引用与所有权一次绑好。

**与 6.7.6 的差异**：图上没有"非批量 adapter 逐条写、持 shard mutex、失败后可选重开 fd"这一层，也没有 index 回填机制；图把 BatchWrite 当成一次原子 FS 调用。

#### 2.4.5 descriptor 几何校验：IsDfsDescriptorRangeValid 与 bucket 版本

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:58-93`

```cpp
bool IsDfsDescriptorRangeValid(const DistributedFSDescriptor& desc,
                               const DistributedStorageConfig& config) {
    // 纯算术守卫，不碰文件系统；alignment 为 0 会让下面 % 变成除零
    if (config.alignment == 0 || desc.object_size == 0 ||
        // 对齐后反而比对象还小，说明 master 少算了 aligned_size
        desc.aligned_size < desc.object_size ||
        // offset 与 aligned_size 都必须落在对齐边界（3FS USRBIO 的硬要求）
        desc.offset % config.alignment != 0 ||
        desc.aligned_size % config.alignment != 0) {
        return false;                    // 不做修正，直接判定 descriptor 非法
    }
    // 先比 offset 再比差值，避免 offset+aligned_size 在 uint64 上回绕
    if (desc.offset > config.shard_capacity ||
        desc.aligned_size > config.shard_capacity - desc.offset) {
        return false;                    // 会写穿 shard 文件尾部
    }

    // pwrite/pread 的 offset 是 off_t（有符号 64 位），上限要单独再卡一次
    constexpr uint64_t kMaxFileOffset =
        static_cast<uint64_t>(std::numeric_limits<int64_t>::max());
    return desc.offset <= kMaxFileOffset &&
           desc.aligned_size <= kMaxFileOffset - desc.offset;  // 差值式比较
}

bool IsBucketDescriptorRangeValid(const DistributedFSDescriptor& desc,
                                  const DistributedStorageConfig& config) {
    // bucket 模式多一个条件：shard_idx 必须非负（编号来自文件名解析）
    if (config.alignment == 0 || desc.shard_idx < 0 || desc.object_size == 0 ||
        desc.aligned_size < desc.object_size ||
        desc.offset % config.alignment != 0 ||
        desc.aligned_size % config.alignment != 0 ||
        // 上界从 shard_capacity 换成 bucket_capacity
        desc.offset > config.bucket_capacity ||
        desc.aligned_size > config.bucket_capacity - desc.offset ||
        // 对象本体也必须落在 bucket 范围内，而不只是对齐后的长度
        desc.object_size > config.bucket_capacity - desc.offset) {
        return false;
    }
    constexpr uint64_t kMaxOffset =
        static_cast<uint64_t>(std::numeric_limits<int64_t>::max());
    return desc.offset <= kMaxOffset &&
           desc.object_size <= kMaxOffset - desc.offset &&
           desc.aligned_size <= kMaxOffset - desc.offset;
```

**这一段在做什么**

- 两个函数都在匿名命名空间内，是纯算术守卫：不 open、不 stat，只回答"这个 descriptor 能不能拿去写"。
- 三个基本不变量：alignment != 0、object_size != 0、aligned_size >= object_size，任一破坏即非法。
- 容量检查一律写成 `x > cap - offset` 而非 `offset + x > cap`，防止 uint64 回绕后"看起来合法"。
- shard 模式以 shard_capacity 为界；bucket 模式以 bucket_capacity 为界，且额外要求 shard_idx >= 0 与 object_size 本身也在 bucket 内。
- 最后一层是 off_t 上界：落地时 offset 会被 static_cast<int64_t>，超过 INT64_MAX 会变成负数。
- 函数只返回 bool，错误码翻译交给调用方（BatchWrite 统一翻成 INVALID_PARAMS）。

**重点语法 / 惯用法**

- 对 0 取模是未定义行为（整数除零），所以 `config.alignment == 0` 必须排在 `%` 之前。
- 无符号减法下溢：offset > cap 时 cap - offset 会变成一个巨大值，所以前一个比较不能省。
- std::numeric_limits<int64_t>::max() 配合 static_cast<uint64_t>：把有符号上界转成无符号常量再参与比较。
- constexpr 局部常量：编译期求值，替代魔数并保留类型。
- 短路求值 `a && b`：后一个条件依赖前一个成立时才安全（差值比较依赖 offset 已被约束）。

**与 6.7.6 的差异**：图中 A-->>M 之后直接进入写入，没有任何"descriptor 几何校验"；实际 BatchWrite 在碰 FS 前先做纯内存校验，失败即 INVALID_PARAMS。图上也没有 bucket/shard 两套容量口径的区分。

### 2.5 文件定位与打开：GetOrOpenShard / OpenBucket（第 8 步）

图上 `BE->>FS: GetOrOpenShard + BatchWriteAt` 的前半段。这一步决定了「descriptor 能不能被信任」。

#### 2.5.1 文件名即契约：ParseShardFileName 定义的命名规则

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:16-35`

```cpp
std::optional<int> ParseShardFileName(std::string_view name) { // ▶ 零拷贝解析文件名
    constexpr std::string_view prefix = "dfs_shard_";   // ▶ 命名前缀写死
    constexpr std::string_view suffix = ".data";        // ▶ 后缀写死
    if (!name.starts_with(prefix) || !name.ends_with(suffix)) { // ▶ C++20 接口
        return std::nullopt;                            // ▶ 不是分片文件
    }
    name.remove_prefix(prefix.size());                  // ▶ 原地切头,不分配
    name.remove_suffix(suffix.size());                  // ▶ 原地切尾
    if (name.size() < 2) return std::nullopt;           // ▶ 至少两位如 00

    int index = 0;
    for (char digit : name) {                           // ▶ 逐字符转数字
        if (digit < '0' || digit > '9' ||
            index > (std::numeric_limits<int>::max() - (digit - '0')) / 10) { // ▶ 先判溢出
            return std::nullopt;                        // ▶ 非法或会溢出
        }
        index = index * 10 + (digit - '0');
    }
    return index;                                       // ▶ 纯数字才收
}
```

**这一段在做什么**

- 这个函数是 GetOrOpenShard 三道校验里的第一道：它把"文件名长什么样"变成了可判定的契约。
- 前缀后缀写死，等价于"只有 master 侧 ShardAllocator 生成的名字才被接受"，客户端无法用任意文件冒充分片。
- 位数下限 2 与 ShardAllocator::FormatShardIdx 的 std::setw(2) 对齐；单数字文件名被当成非法，避免 "dfs_shard_1.data" 和 "dfs_shard_01.data" 两个别名。
- 失败一律返回 nullopt，由调用方映射成 ErrorCode::INVALID_PARAMS，不用异常。
- 整段没有任何堆分配，只移动 string_view 的两个指针。

**重点语法 / 惯用法**

- std::string_view：只读视图，remove_prefix/remove_suffix 改变的是视图的起点和长度，底层字符串不变。
- constexpr std::string_view：编译期常量，函数里用 prefix.size() 是编译期已知量。
- starts_with / ends_with：C++20 新增，比 substr 再比较更省一次构造。
- std::optional<int>：表达"可能没有值"，调用方必须写 if (!index) 再 *index。
- 溢出保护写法 index > (INT_MAX - d) / 10：先除后比，避免 index * 10 + d 本身溢出（有符号溢出是 UB）。

**与 6.7.6 的差异**：6.7.6 把 GetOrOpenShard 画成一个原子步骤，没有体现"文件名本身要被解析成 shard 编号再比对"这一步；图上也没有 DFS 的两种命名（dfs_shard_NN.data 与 bucket_NNNNNN.data）。时序位置与图一致，属于图未展开的细节。

#### 2.5.2 GetOrOpenShard 的快路径：只认原始 file_path

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:216-246`

```cpp
tl::expected<DistributedStorageBackend::ShardFile*, ErrorCode> // ▶ 返回缓存节点
DistributedStorageBackend::GetOrOpenShard(                     // ▶ 每个 op 都先过这里
    const DistributedFSDescriptor& descriptor) {               // ▶ 描述符来自 master
    if (descriptor.shard_idx < 0) {                            // ▶ 负索引无意义
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS); // ▶ 直接判参数错
    }

    ShardFile* shard = nullptr;                                // ▶ 先声明后填
    {
        std::lock_guard cache_lock(shard_files_mutex_);        // ▶ 锁只护映射表
        auto existing = shard_files_.find(descriptor.shard_idx); // ▶ 按编号找
        if (existing != shard_files_.end()) shard = existing->second.get(); // ▶ 命中
    }                                                          // ▶ 出作用域即解锁
    if (shard) {                                               // ▶ 命中才试快路径
        std::lock_guard shard_lock(shard->mutex);              // ▶ 换成分片锁
        // Only the exact path validated when this fd was opened can bypass
        // normalization. Aliases and changed descriptors take the full
        // validation path below. Cache entries live until backend teardown.
        if (shard->fd >= 0 && shard->path == descriptor.file_path) return shard; // ▶ 快路径
    }

    const auto path =
        std::filesystem::path(descriptor.file_path).lexically_normal(); // ▶ 词法规范化
    auto root = std::filesystem::path(root_dir_).lexically_normal(); // ▶ 本地根目录
    if (root.filename().empty()) root = root.parent_path();          // ▶ 去掉尾斜杠
    auto parent = path.parent_path();                                // ▶ 取父目录
    if (parent.empty()) parent = ".";                                // ▶ 相对路径兜底
    const auto index = ParseShardFileName(path.filename().string()); // ▶ 解出编号
    if (!index || *index != descriptor.shard_idx || parent != root) { // ▶ 三重一致性
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);        // ▶ 任一不符即拒
    }
```

**这一段在做什么**

- 快路径的条件是"原始 file_path 与当初打开这条 fd 时保存的规范化路径逐字节相等"；带 ./、..、多余斜杠的别名描述符会落到下面的完整校验。
- map 锁只包住 find，拿到裸指针立即释放；shard_files_ 里放的是 unique_ptr，节点地址稳定，所以裸指针在锁外仍安全。
- 词法检查有三条：父目录必须等于 root_dir_、文件名的数字必须等于 descriptor.shard_idx、shard_idx 不能为负。
- shard_idx 是 master 侧分配器按 std::hash(key) % 分片数 选出来的，还会在分片满时向后探测，扩容后取模结果变化——所以客户端无法由 key 反推路径，只能信描述符并校验。
- lexically_normal() 只做字符串层面的 . 和 .. 消解，不碰文件系统、不解符号链接，因此它只是廉价预筛，真正的安全边界在 ValidateAndOpenShardFile。

**重点语法 / 惯用法**

- std::lock_guard 的 CTAD：C++17 起可省模板参数，std::lock_guard cache_lock(m) 自动推导。
- lexically_normal()：纯词法规范化，不会因为文件不存在而失败（对比 canonical 会访问文件系统）。
- path::filename() 对以分隔符结尾的路径返回空串，所以用 root.filename().empty() 来剥掉尾部分隔符。
- path.filename().string() 返回临时 std::string，作为 string_view 形参的生命周期延续到整个完整表达式结束，函数内使用是安全的。
- tl::expected 用 tl::make_unexpected 构造错误态：错误是返回值的一部分，不抛异常，也不产生歧义。

**与 6.7.6 的差异**：图上 "BE->>FS: GetOrOpenShard + BatchWriteAt" 读起来像每批都去打开一次文件；代码里 fd 按 shard 缓存到 backend 析构，命中快路径时根本不碰文件系统。另外图把 BE 画成独立一跳，实际 dfs_storage_backend_ 是 Client 进程内成员（client_service.cpp:2988 直接 BatchWrite），没有网络往返。

#### 2.5.3 GetOrOpenShard 的慢路径：双检插入与 fd 发布

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:248-279`

```cpp
    if (!shard) {                                               // ▶ 慢路径才建节点
        std::lock_guard cache_lock(shard_files_mutex_);         // ▶ 再拿映射表锁
        auto existing = shard_files_.find(descriptor.shard_idx); // ▶ 二次检查
        if (existing == shard_files_.end()) {
            existing = shard_files_
                           .emplace(descriptor.shard_idx,
                                    std::make_unique<ShardFile>())
                           .first;                          // ▶ 插入空节点 fd=-1
        }
        shard = existing->second.get();                         // ▶ 取稳定裸指针
    }

    // File validation and opening may block. Serialize them only with this
    // shard's initialization and non-batched I/O, leaving other cached shards
    // available.
    std::lock_guard shard_lock(shard->mutex);                   // ▶ 只串行化本分片
    auto file_path = path.string();                             // ▶ 规范化后的路径
    if (shard->fd >= 0) {                                       // ▶ 已被别人打开
        if (shard->path != file_path) {                         // ▶ 但路径不一致
            return tl::make_unexpected(ErrorCode::INVALID_PARAMS); // ▶ 同号不同文件
        }
        return shard;                                           // ▶ 一致就复用
    }

    auto fd = ValidateAndOpenShardFile(file_path);              // ▶ 校验并打开
    if (!fd) return tl::make_unexpected(fd.error());            // ▶ 失败不写缓存
    // Failed attempts keep fd == -1 so a later request can retry
    // initialization.
    shard->path = std::move(file_path);                         // ▶ 先记已验路径
    shard->fd = *fd;                                            // ▶ 后发布 fd
    return shard;
}
```

**这一段在做什么**

- 第一次查表在锁外只拿指针，插入时重新加锁再查一遍（双检），防止两个线程为同一个 shard_idx 各建一个 ShardFile。
- 插入和打开分两把锁：map 锁保护结构，shard->mutex 保护这个分片的校验与 fd，不同分片打开互不阻塞。
- fd 已存在但规范化路径不同，直接 INVALID_PARAMS 而不是覆盖：说明同一 shard_idx 出现了两个文件，属于元数据冲突，覆盖会写错位置。
- 失败时直接 return，不写 shard->path 也不写 fd，fd 保持 -1，下一个请求会重新走这里重试。
- 赋值顺序是先 path 后 fd：保证任何看到 fd >= 0 的线程，同时能看到已校验过的 path。

**重点语法 / 惯用法**

- std::unique_ptr<ShardFile>：ShardFile 内含 std::mutex，天然不可拷贝不可移动，只能堆分配后由容器持有。
- .emplace(k, v).first：unordered_map::emplace 返回 pair<iterator,bool>，这里只取迭代器，忽略"是否新建"。
- get() 取裸指针：锁外使用不会延长生命周期，但容器节点在 backend 析构前不会失效。
- lock_guard 在 return 时析构解锁，所以函数提前 return 也是异常安全的。
- std::move(file_path) 移进成员：省一次字符串拷贝，之后 file_path 不应再被使用。

**与 6.7.6 的差异**：图未画两条失败分支：同一 shard_idx 却换了 file_path → INVALID_PARAMS；打开失败后保留 fd=-1 供下次重试。图也未体现客户端的 root_dir_（来自 config fsdir，构造函数 :157）必须与 master 的 mount_path 前缀一致，否则 parent != root 直接被拒。

#### 2.5.4 ValidateAndOpenShardFile：先看大小，再解真身，最后才 open

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:281-304`

```cpp
tl::expected<int, ErrorCode>
DistributedStorageBackend::ValidateAndOpenShardFile(
    const std::string& file_path) {
    // The master prepares new shards before publishing their descriptors.
    // OpenFile may create a file, so reject missing/unprepared shards first.
    auto file_size = fs_adapter_->GetFileSize(file_path);   // ▶ 先 stat
    if (!file_size) return tl::make_unexpected(file_size.error()); // ▶ 不存在即失败
    if (*file_size != distributed_config_.shard_capacity) { // ▶ 必须精确相等
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    auto root = std::filesystem::path(root_dir_).lexically_normal(); // ▶ 本地根
    if (root.filename().empty()) root = root.parent_path(); // ▶ 去掉尾斜杠
    const auto path = std::filesystem::path(file_path);     // ▶ 待验路径
    std::error_code ec;                                     // ▶ 复用错误码
    const auto canonical_root = std::filesystem::canonical(root, ec); // ▶ 解真身
    if (ec) return tl::make_unexpected(ErrorCode::FILE_OPEN_FAIL); // ▶ 根不可达
    const auto canonical_path = std::filesystem::canonical(path, ec); // ▶ 解真身
    if (ec) return tl::make_unexpected(ErrorCode::FILE_OPEN_FAIL); // ▶ 路径不可达
    if (canonical_path.parent_path() != canonical_root ||   // ▶ 必须在根下
        canonical_path.filename() != path.filename()) {     // ▶ 不许被改名
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    return fs_adapter_->OpenFile(file_path);                // ▶ 带 O_CREAT
}
```

**这一段在做什么**

- 三步严格有序：GetFileSize 挡掉不存在的文件 → canonical 挡掉越界路径 → OpenFile 才真正创建/打开 fd。顺序颠倒就会在磁盘上留下垃圾文件。
- "精确相等"而不是小于：分片是 master 侧 PreallocateFile 成 shard_capacity 的定长文件（shard_allocator.cpp:224），分配器按整片 [0, capacity) 建 offset 空间；大小不符说明这不是同一份布局，写进去会越界或错位。
- 必须先 stat 的直接原因：两个适配器的 OpenFile 都用 O_RDWR|O_CREAT（posix_fs_adapter.cpp:92、hf3fs_adapter.cpp:100），文件不存在时会被悄悄建成 0 字节，后续所有写都落到"假分片"里。
- canonical 解析符号链接和 ..，把"词法上在 root 下、实际指向别处"的软链挡掉；再比对 filename，防止软链把目标文件改名。
- 本函数只负责校验和拿 fd，不做任何读写；fd 交给 GetOrOpenShard 缓存，之后的 I/O 才由 BatchWriteAt/BatchReadAt 发出。

**重点语法 / 惯用法**

- canonical(p, ec) 重载：使用 error_code 版本不抛异常，失败时置 ec 并返回空 path。
- ec 在两次 canonical 间复用，第二次会覆盖第一次，但每次都在使用前立刻检查，所以安全。
- canonical 要求路径存在，因此它同时起到了"再确认一次文件在"的作用。
- GetFileSize 返回 std::expected<size_t,...>，与 uint64_t 的 shard_capacity 直接比较，无窄化转换。
- 函数返回 tl::expected<int, ErrorCode>：fd 的 -1 不表示错误，错误只走 expected 的 error 分支。

**与 6.7.6 的差异**：图上 GetOrOpenShard 与 BatchWriteAt 连成一步；代码里每个 op 先 stat + canonical + open，再进 BatchWriteAt，中间多两道校验。图也没有"OpenFile 带 O_CREAT，所以必须先用 GetFileSize 挡掉未准备的分片"这层因果。

#### 2.5.5 对照：ParseBucketFileName 与 shard 命名的差别

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:37-56`

```cpp
std::optional<int> ParseBucketFileName(std::string_view name) { // ▶ 桶模式专用
    constexpr std::string_view prefix = "bucket_";      // ▶ 前缀不同
    constexpr std::string_view suffix = ".data";
    if (!name.starts_with(prefix) || !name.ends_with(suffix)) {
        return std::nullopt;                            // ▶ 不是桶文件
    }
    name.remove_prefix(prefix.size());
    name.remove_suffix(suffix.size());
    if (name.empty()) return std::nullopt;              // ▶ 只查空,不限制位数

    int index = 0;
    for (char digit : name) {
        if (digit < '0' || digit > '9' ||
            index > (std::numeric_limits<int>::max() - (digit - '0')) / 10) { // ▶ 同样防溢出
            return std::nullopt;
        }
        index = index * 10 + (digit - '0');
    }
    return index;                                       // ▶ 返回 bucket id
}
```

**这一段在做什么**

- 与 ParseShardFileName 唯一的实质差异是位数检查：分片要求 name.size() >= 2，桶只要求非空。
- 原因是桶名用 std::setw(6) 零填充（immutable_bucket_allocator.cpp:127），桶号超过 6 位时 setw 不会截断，解析端不能假设固定宽度；分片则强制零填充到 2 位。
- 两套前缀刻意分开，避免 shard 描述符和 bucket 描述符互相顶替——这是两种完全不同的分配器布局。
- 同名字段 shard_idx 在桶模式下装的是 bucket id，所以 OpenBucket 里比对的是 bucket_id 而不是分片编号。
- 解析只回答"这串字符是不是合法编号、编号是几"，安全性判断全部留给调用方的路径/大小/canonical 校验。

**重点语法 / 惯用法**

- setw 只保证最小宽度，数字更长时照常输出，因此宽度约束不能写死在解析器里。
- string_view 的 remove_prefix/remove_suffix 仍然零拷贝，与分片版本写法对称，便于对照阅读。
- 两个函数几乎逐行重复：这是刻意保留的两种命名空间，合并反而会掩盖语义差异。
- std::optional<int> 的 *index 在 if 短路里已经保证有值（!index 先判）。

**与 6.7.6 的差异**：6.7.6 完全没有画 bucket 模式（allocator_type=bucket）：同一份 DistributedFSDescriptor 在桶模式下 shard_idx 字段装的是 bucket id，文件名前缀也换成 bucket_。这是图与代码最明显的结构性缺口，而不是时序差异。

#### 2.5.6 OpenBucket：每次重开、必须已存在

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:328-357`

```cpp
tl::expected<int, ErrorCode> DistributedStorageBackend::OpenBucket(
    const DistributedFSDescriptor& descriptor) {
    const auto path =
        std::filesystem::path(descriptor.file_path).lexically_normal(); // ▶ 词法规范化
    auto root = std::filesystem::path(root_dir_).lexically_normal();
    if (root.filename().empty()) root = root.parent_path();
    auto parent = path.parent_path();
    if (parent.empty()) parent = ".";
    const auto bucket_id = ParseBucketFileName(path.filename().string()); // ▶ 解出桶号
    if (!bucket_id || *bucket_id != descriptor.shard_idx || parent != root) { // ▶ 三重一致
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }

    const std::string file_path = path.string();            // ▶ 单独持有一份
    auto file_size = fs_adapter_->GetFileSize(file_path);   // ▶ 先 stat
    if (!file_size) return tl::make_unexpected(file_size.error());
    if (*file_size != distributed_config_.bucket_capacity) { // ▶ 必须整桶大小
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    std::error_code error;
    const auto canonical_root = std::filesystem::canonical(root, error); // ▶ 解真身
    if (error) return tl::make_unexpected(ErrorCode::FILE_OPEN_FAIL);
    const auto canonical_path = std::filesystem::canonical(path, error); // ▶ 解真身
    if (error) return tl::make_unexpected(ErrorCode::FILE_OPEN_FAIL);
    if (canonical_path.parent_path() != canonical_root ||   // ▶ 必须在根下
        canonical_path.filename() != path.filename()) {     // ▶ 不许被改名
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    return fs_adapter_->OpenExistingFile(file_path);        // ▶ 严禁 O_CREAT
}
```

**这一段在做什么**

- bucket 分支不查 shard_files_ 缓存：每次 BatchWrite/BatchRead 都重新校验并打开，用完立刻 close（ExecuteSingleIo:457-464），所以这里没有快路径。
- 大小必须精确等于 bucket_capacity：每个桶是 PreallocateFile 出来的整桶定长文件，offset 空间按整桶算。
- 最后一步必须用 OpenExistingFile 而不是 OpenFile：头文件明确要求实现不得使用 O_CREAT，因为桶描述符可能在整桶驱逐之后才到达。
- 如果用 O_CREAT，会在共享文件系统上凭空复活一个 0 字节桶文件；这个桶不在分配器的 buckets_ 映射里，写进去的数据无人回收，后续同 id 的桶还会与之冲突。
- 校验顺序与分片版一致（文件名 → 大小 → canonical → open），保证两种模式的安全语义对齐。

**重点语法 / 惯用法**

- OpenExistingFile 与 OpenFile 的差异属于接口契约而非参数：posix 版用 O_RDWR|O_CLOEXEC 且对 EINTR 重试；3fs 版还额外把 ENOENT 映射成 FILE_NOT_FOUND。
- const std::string file_path = path.string()：path.string() 返回临时对象，要跨多条语句使用就必须落到命名变量里。
- std::error_code error：与 shard 版的 ec 写法相同，名字不同但语义一致。
- 错误枚举分层：路径不存在 → FILE_NOT_FOUND（来自 GetFileSize），存在但类型/大小不对 → INVALID_PARAMS，I/O 打不开 → FILE_OPEN_FAIL。

**与 6.7.6 的差异**：6.7.6 只画了 shard 一种布局；bucket 模式不缓存 fd，每批 open/close，且 ExecuteBucketBatchIo 每批最多同时开 64 个桶（kMaxOpenBucketsPerBatch）。图上的 "BatchWriteAt 直接写入" 在桶分支要先 OpenBucket、再 TransferAll、最后 CloseFile，比图多两步。

### 2.6 执行器：ExecuteSingleIo / SubmitBatchIo（第 8 步）

图上 `+ BatchWriteAt` 的实际落点。两个能力开关在这里分化出四种执行形状。

#### 2.6.1 匿名命名空间的 TransferAll：短传输续传的唯一出口

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:95-136`

```cpp
// Completes a positional transfer of `total` bytes at `offset` whose first
// `transferred` bytes are already done, retrying short transfers.
tl::expected<void, ErrorCode> TransferAll(FileSystemAdapter& adapter, int fd,
                                          std::vector<iovec> iovs,  // 按值传，函数内可改 iov
                                          int64_t offset, uint64_t total,
                                          uint64_t transferred, bool write) {  // write 决定写还是读
    size_t index = 0;                    // 当前处理到第几个 iovec
    size_t consumed = transferred;       // 已完成字节数，用于跳过前段
    while (true) {
        while (consumed > 0 && index < iovs.size()) {  // 先把“已完成部分”从 iov 头上切掉
            if (consumed >= iovs[index].iov_len) {     // 整个 iovec 都已完成
                consumed -= iovs[index].iov_len;
                ++index;
            } else {                                   // 只完成一部分，改基址和长度
                iovs[index].iov_base =
                    static_cast<char*>(iovs[index].iov_base) + consumed;  // void* 先转 char* 才能加
                iovs[index].iov_len -= consumed;
                consumed = 0;
            }
        }
        if (transferred >= total) return {};  // 全部完成，返回空 expected 表示成功

        while (index < iovs.size() && iovs[index].iov_len == 0) ++index;  // 跳过空 iovec
        if (index == iovs.size() ||          // 没有剩余数据却还没传够，入参自相矛盾
            iovs.size() - index >
                static_cast<size_t>(std::numeric_limits<int>::max())) {  // iovcnt 是 int，防截断
            return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
        }
        const auto count = static_cast<int>(iovs.size() - index);   // 本次提交的 iovec 个数
        const auto position = offset + static_cast<int64_t>(transferred);  // 续传位置=起点+已完成
        auto result =
            write ? adapter.WriteAt(fd, iovs.data() + index, count, position)
                  : adapter.ReadAt(fd, iovs.data() + index, count, position);
        if (!result) return tl::make_unexpected(result.error());  // 硬错误直接上抛
        if (*result == 0 || *result > total - transferred) {  // 0 字节=假成功；超额=适配器异常
            return tl::make_unexpected(write ? ErrorCode::FILE_WRITE_FAIL
                                             : ErrorCode::FILE_READ_FAIL);
        }
        consumed = *result;              // 下一轮把这 *result 字节从 iov 头切掉
        transferred += *result;          // 累计真实进度
    }
}
```

**这一段在做什么**

- consumed 和 transferred 分工不同：前者只用来切掉 iovec 头部已完成的字节，后者才是相对 offset 的真实进度。
- ret==0 必须当成失败：循环靠 transferred 单调递增收敛，返回 0 会让 while(true) 原地空转（读文件到 EOF 时也是 0）。
- ret > total - transferred 视为适配器返回了不可能的值，直接判失败，不信任它写出的进度。
- iovs 按值传入，函数里改 iov_base/iov_len 不会污染调用方的 vector，这是它能安全“切头”的前提。
- 只有 bucket 路径会调用它（ExecuteSingleIo 的 bucket 分支、SubmitBatchIo 的 complete_short_io 分支），shard 路径从不续传。

**重点语法 / 惯用法**

- tl::expected<void, ErrorCode>：成功时无返回值，失败用 tl::make_unexpected(ErrorCode) 承载错误码。
- static_cast<char*>(iov_base) + consumed：void* 不能做指针算术，必须先转成 char*。
- std::numeric_limits<int>::max()：readv/writev 的 iovcnt 是 int，超过会截断，这里提前拦住。
- iovec 是 C 结构体 {void* iov_base; size_t iov_len;}，std::vector<iovec> 连续存储，可传 .data()。
- return {};：值初始化一个“成功的 expected<void>”，比 return tl::expected<void,ErrorCode>{} 简洁。

**与 6.7.6 的差异**：图里没有短传输续传这一步。BatchWriteAt/BatchReadAt 返回少于请求字节时，bucket 路径会调用 TransferAll 从断点续传，shard 路径则直接判失败——两条路径语义相反，图上只画了一次 FS 调用。

#### 2.6.2 ExecuteSingleIo 上：bucket 的 open→TransferAll→close 与 shard 的持锁单次 I/O

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:454-479`

```cpp
tl::expected<void, ErrorCode> DistributedStorageBackend::ExecuteSingleIo(
    DfsIoOp& op, bool write) {
    if (UsesBucketAllocator()) {          // bucket 模式：fd 是临时的，每请求开一次
        auto fd = OpenBucket(op.descriptor);   // 校验路径/容量后打开 bucket 文件
        if (!fd) return tl::make_unexpected(fd.error());
        auto result = TransferAll(*fs_adapter_, *fd, std::move(op.iovs),  // iov 交出所有权
                                  static_cast<int64_t>(op.descriptor.offset),
                                  op.size, 0, write);   // 从 0 开始，短写由 TransferAll 续
        auto close_result = fs_adapter_->CloseFile(*fd);    // 先关 fd，异常路径也不泄漏
        if (!result) return result;      // I/O 失败优先上报，close 结果被丢弃
        return close_result;             // I/O 成功时 close 失败仍要上报
    }

    auto shard = GetOrOpenShard(op.descriptor);  // shard 模式：fd 常驻缓存
    if (!shard) {
        LOG(ERROR) << "Failed to open DFS shard for key " << op.key << ": "
                   << shard.error();
        return tl::make_unexpected(shard.error());
    }
    std::lock_guard lock((*shard)->mutex);  // 锁粒度=单个 shard，覆盖到函数返回
    const auto count = static_cast<int>(op.iovs.size());   // iovcnt 要转 int
    const auto offset = static_cast<int64_t>(op.descriptor.offset);  // 文件内绝对偏移
    auto result =
        write
            ? fs_adapter_->WriteAt((*shard)->fd, op.iovs.data(), count, offset)
            : fs_adapter_->ReadAt((*shard)->fd, op.iovs.data(), count, offset);  // 只调一次，不续传
```

**这一段在做什么**

- 只有 SupportsBatchIo()==false 的适配器（POSIX 等）才会进入本函数；3FS 走 ExecuteBucketBatchIo/ExecuteShardBatchIo。
- bucket 模式 fd 是临时的：每个请求 open 一次、close 一次，因此不需要像 shard 那样加互斥锁。
- bucket 分支先算好 close_result 再判 I/O 结果，保证失败返回路径上 fd 也一定被关闭。
- shard 模式从缓存拿常驻 fd，lock_guard 覆盖整个函数尾部，把同一 shard 上的并发 I/O 串行化——这就是“锁粒度是 shard 而不是全局”的落点。
- op.iovs 被 move 进 TransferAll，调用返回后 op 内部的 vector 已经为空。

**重点语法 / 惯用法**

- auto fd = OpenBucket(...) 返回 tl::expected<int, ErrorCode>，*fd 才是真正的 fd 值。
- std::move(op.iovs) 传入按值形参，避免整份 iovec 数组拷贝。
- static_cast<int64_t>(op.descriptor.offset)：descriptor.offset 是 uint64_t，pread/pwrite 需要 off_t 语义的 int64_t。
- 条件运算符 write ? A : B 要求两分支类型一致，这里都是 tl::expected<size_t, ErrorCode>。

**与 6.7.6 的差异**：图写的是 GetOrOpenShard + BatchWriteAt，但这条路径既没有 BatchWriteAt 也没有 close：它按请求逐个 WriteAt/ReadAt，bucket 还要额外 open/close。BatchWriteAt 只出现在批量适配器路径。

#### 2.6.3 ExecuteSingleIo 下：一次 reopen 重试与短传输硬失败

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:480-515`

```cpp
    const char* op_name = write ? "write" : "read";   // 日志与错误码复用同一分支
    if (!result && fs_adapter_->SupportsFdReopen()) {  // 仅 POSIX 类适配器声明可重开
        // A cached descriptor can go stale (fd invalidated, shard file
        // rotated). Reopen once under the shard lock and retry before
        // surfacing the failure. A short transfer below is treated as genuine
        // corruption and not retried. The lock already serializes shard I/O, so
        // the reopen adds no cost to the success path. Only adapters whose I/O
        // faults are fd-recoverable (POSIX) opt in via SupportsFdReopen.
        LOG(WARNING) << "DFS " << op_name << " failed for key " << op.key
                     << ", error=" << result.error() << ", reopening shard "
                     << op.descriptor.shard_idx;
        auto reopen = ReopenShardLocked(**shard);   // 已持 shard 锁，故函数名带 Locked
        if (reopen) {
            result = write ? fs_adapter_->WriteAt((*shard)->fd, op.iovs.data(),
                                                  count, offset)
                           : fs_adapter_->ReadAt((*shard)->fd, op.iovs.data(),
                                                 count, offset);  // 只重试一次
        } else {
            LOG(ERROR) << "DFS shard " << op.descriptor.shard_idx
                       << " reopen failed for key " << op.key
                       << ", error=" << reopen.error();
        }
    }
    if (!result) {
        LOG(WARNING) << "DFS " << op_name << " failed for key " << op.key
                     << ", error=" << result.error();
        return tl::make_unexpected(result.error());
    }
    if (*result != op.size) {            // shard 路径短传输=硬失败，不续传
        LOG(WARNING) << "DFS short " << op_name << " for key " << op.key
                     << ", expected=" << op.size << ", actual=" << *result;
        return tl::make_unexpected(write ? ErrorCode::FILE_WRITE_FAIL
                                         : ErrorCode::FILE_READ_FAIL);
    }
    return {};
}
```

**这一段在做什么**

- 重试条件很窄：系统调用报错（!result）且适配器声明 SupportsFdReopen()，并且只重试一次，不在循环里反复试。
- 能安全重开的前提正是上面那把 shard 锁：关闭旧 fd 时不可能有其他线程正在通过它读写。
- 短传输（*result != op.size）不重试而是判 FILE_WRITE_FAIL/FILE_READ_FAIL，因为重写同一区间可能写出交错数据，宁可直接暴露问题。
- 3FS 适配器 SupportsFdReopen() 默认 false，压根不走这段；它的失败处理在 SubmitBatchIo 里。
- ReopenShardLocked 失败时会把 fd 置 -1，后续 GetOrOpenShard 会重新初始化，不会永久卡死在坏 fd 上。

**重点语法 / 惯用法**

- **shard：shard 是 tl::expected<ShardFile*, ErrorCode>，第一个 * 取指针，第二个 * 解引用到 ShardFile 对象。
- auto reopen = ...; if (reopen) 依赖 expected 的 explicit operator bool，失败时进 else 分支。
- result.error() 需要先取出或直接拷贝，因为 result 随后可能被第二次 I/O 覆盖。
- (void)fs_adapter_->CloseFile(...)（ReopenShardLocked 内部）：显式丢弃返回值，表明“旧 fd 关不掉也无所谓”。

**与 6.7.6 的差异**：图里完全没有 reopen 重试和短传输判定这两步。它们只存在于 POSIX 单条路径；3FS 批量路径在 SubmitBatchIo 里另有自己的短传输策略。

#### 2.6.4 ExecuteShardBatchIo：批量路径为什么不拿 shard 锁

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:517-539`

```cpp
void DistributedStorageBackend::ExecuteShardBatchIo(
    std::vector<DfsIoOp>& ops,
    std::vector<tl::expected<void, ErrorCode>>& results, bool write) {
    std::vector<FdIoRequest> io_requests;   // 交给适配器的扁平 I/O 描述
    std::vector<DfsIoOp*> io_ops;           // 与 io_requests 下标一一对应
    io_requests.reserve(ops.size());        // 两个 vector 同步 push，预留避免扩容
    io_ops.reserve(ops.size());
    for (auto& op : ops) {
        auto shard = GetOrOpenShard(op.descriptor);   // 这里只解析 fd，不提交 I/O
        if (!shard) {
            LOG(ERROR) << "Failed to open DFS shard for key " << op.key << ": "
                       << shard.error();
            results[op.index] = tl::make_unexpected(shard.error());  // 按原下标回填失败
            continue;
        }
        io_requests.push_back({(*shard)->fd, op.iovs.data(),
                               static_cast<int>(op.iovs.size()),
                               static_cast<int64_t>(op.descriptor.offset)});
        io_ops.push_back(&op);              // 地址稳定：ops 收齐后不再增删
    }
    // Shard I/O fails short transfers.
    SubmitBatchIo(io_requests, io_ops, results, write, false);  // false=短传输判失败
}
```

**这一段在做什么**

- 本函数只做两件事：把 descriptor 解析成 fd、组装 FdIoRequest；真正的 VFS/USRBIO 调用全部留给 SubmitBatchIo。
- io_requests[i] 与 io_ops[i] 是并行数组，结果回填全靠下标对应，任何一边重排都会把错误安到别的 key 上。
- 与 ExecuteSingleIo 最大差别：这里没有 lock_guard shard->mutex，同一 shard 的多条请求被一次性提交、可并发执行。
- 敢不加锁的依据是头文件约定：批量适配器不会触发 ReopenShardLocked，fd 只在 shard 锁下变更，因此可以并发使用。
- 打开失败直接写进 results[op.index] 并从批量数组里剔除，不会让一个坏 key 拖垮整批。

**重点语法 / 惯用法**

- 聚合初始化 {fd, iov, iovcnt, offset} 直接构造 FdIoRequest，字段顺序必须与结构体声明一致。
- std::vector<DfsIoOp*> 只存指针，避免拷贝含 const std::string& 引用成员的 DfsIoOp。
- results 是 vector<tl::expected<void, ErrorCode>>，默认构造即“成功”，只有失败才赋值。
- const auto& 结构化绑定、reserve 与 size_t 下标混用是这批代码的常规写法。

**与 6.7.6 的差异**：图里 BatchWriteAt 是“一步完成”，没体现它只属于批量适配器；更重要的是这条路径刻意不加 shard 锁，与单条路径的锁语义正好相反，图上看不出来。

#### 2.6.5 ExecuteBucketBatchIo 上：按 descriptor 身份分组，一次最多开 64 个 bucket

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:541-576`

```cpp
void DistributedStorageBackend::ExecuteBucketBatchIo(
    std::vector<DfsIoOp>& ops,
    std::vector<tl::expected<void, ErrorCode>>& results, bool write) {
    // Group ops by the full descriptor identity OpenBucket validates, so each
    // bucket is opened once per batch.
    std::map<std::pair<std::string, int>, std::vector<DfsIoOp*>> buckets;  // 有序，便于复现
    for (auto& op : ops) {
        buckets[{op.descriptor.file_path, op.descriptor.shard_idx}].push_back(
            &op);                            // 分组键=OpenBucket 真正校验的两项
    }

    auto next = buckets.begin();
    while (next != buckets.end()) {          // 每轮最多开 64 个 bucket
        std::vector<std::pair<int, const std::vector<DfsIoOp*>*>> opened;  // 待关闭的 fd
        std::vector<FdIoRequest> io_requests;
        std::vector<DfsIoOp*> io_ops;
        for (; next != buckets.end() && opened.size() < kMaxOpenBucketsPerBatch;
             ++next) {                       // 控制并发 fd 数，防止耗尽 fd 上限
            const auto& bucket_ops = next->second;
            auto fd = OpenBucket(bucket_ops.front()->descriptor);  // 同组描述符等价，取首个
            if (!fd) {
                LOG(ERROR) << "Failed to open DFS bucket " << next->first.first
                           << ": " << fd.error();
                for (auto* op : bucket_ops) {
                    results[op->index] = tl::make_unexpected(fd.error());  // 整组失败
                }
                continue;                    // 不占 opened 名额，换下一个 bucket
            }
            opened.emplace_back(*fd, &bucket_ops);
            for (auto* op : bucket_ops) {    // 一个 bucket 内可含多条请求
                io_requests.push_back(
                    {*fd, op->iovs.data(), static_cast<int>(op->iovs.size()),
                     static_cast<int64_t>(op->descriptor.offset)});
                io_ops.push_back(op);
            }
        }

        // Bucket I/O completes short transfers.
        SubmitBatchIo(io_requests, io_ops, results, write, true);  // true=短传输可续传
```

**这一段在做什么**

- 分组键是 OpenBucket 真正校验的 (file_path, shard_idx) 二元组，所以同一文件在一个批次里只会 open 一次。
- kMaxOpenBucketsPerBatch=64（头文件第 95 行）限制同时持有的 fd 数，批量很大时分多轮，避免把进程 fd 吃光。
- open 失败的 bucket 把错误写给该组所有 op 后 continue，不占用 64 个名额，也不影响其他 bucket。
- 每轮提交完就关掉本轮打开的 fd（下一段），fd 不会跨轮累积。
- 用 std::map 而不是 unordered_map，遍历顺序稳定，出问题时批次内容可复现。

**重点语法 / 惯用法**

- std::map<std::pair<std::string,int>, ...> 的 operator[] 需要键可比较，std::pair 自带字典序。
- for 的第 1 和第 3 部分被拆开写，next 在循环体间持续推进，外层 while 负责多轮。
- opened.size() < kMaxOpenBucketsPerBatch：size_t 与 static constexpr size_t 比较，无符号一致。
- bucket_ops.front()->descriptor：组内任取一条即可，因为分组键保证了 file_path 与 shard_idx 相同。

**与 6.7.6 的差异**：6.7.6 图里没有 bucket 这一层，只有一个 BatchWrite/BatchRead；实际 bucket 模式每次都要 open/close 文件，还多出 64 个同时打开 fd 的上限与多轮提交。

#### 2.6.6 ExecuteBucketBatchIo 下：close 失败如何回写到结果

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:577-591`

```cpp
        // A failed close fails the bucket's otherwise successful requests.
        for (const auto& [fd, bucket_ops] : opened) {   // 提交完再统一关闭
            auto close_result = fs_adapter_->CloseFile(fd);
            if (close_result) continue;     // 关闭成功，不动已有结果
            for (auto* op : *bucket_ops) {
                auto& result = results[op->index];
                if (result) result = tl::make_unexpected(close_result.error());
            }                               // 只覆盖原本成功的结果
        }
    }
}
```

**这一段在做什么**

- close 放在整批 SubmitBatchIo 之后统一做，避免关掉正在被适配器使用的 fd。
- 只有原本成功（if (result)）的请求才会被 close 错误覆盖，更早的 I/O 失败原因得以保留。
- 这解释了 bucket 模式的“写成功”其实包含 close 成功；shard 模式没有这一层语义。
- 末尾两个右花括号：第一个结束外层 while（还有剩余 bucket 就继续下一轮），第二个结束函数。

**重点语法 / 惯用法**

- 结构化绑定 for (const auto& [fd, bucket_ops] : opened) 解出 int 与指针，bucket_ops 是指针需再 * 才能遍历。
- auto& result = results[op->index]; 必须取引用，否则改的是副本。
- if (result) 判断 expected 是否持有值（成功态）；失败态才能调 error()。

**与 6.7.6 的差异**：图里完全没有 close 步骤，也没有“close 失败会让整组写失败”的语义；对 bucket 模式来说，FS 交互是 open→BatchWriteAt→close 三步，不是一步。

#### 2.6.7 SubmitBatchIo：一次 BatchWriteAt/BatchReadAt，两种短传输语义

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:593-631`

```cpp
void DistributedStorageBackend::SubmitBatchIo(
    const std::vector<FdIoRequest>& io_requests,
    const std::vector<DfsIoOp*>& io_ops,
    std::vector<tl::expected<void, ErrorCode>>& results, bool write,
    bool complete_short_io) {
    if (io_requests.empty()) return;         // 空批不发任何系统调用
    const char* op_name = write ? "write" : "read";
    auto io_results = write ? fs_adapter_->BatchWriteAt(io_requests)
                            : fs_adapter_->BatchReadAt(io_requests);  // 3FS 走这里
    if (io_results.size() != io_requests.size()) {   // 适配器必须一请求一结果
        LOG(ERROR) << "Filesystem adapter returned an invalid batch " << op_name
                   << " result count";
        for (auto* op : io_ops) {
            results[op->index] = tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
        }
        return;                              // 对应关系不明，整批判失败
    }
    for (size_t i = 0; i < io_results.size(); ++i) {
        auto& op = *io_ops[i];               // 并行数组把结果映射回原请求
        const auto& result = io_results[i];
        if (!result) {
            LOG(WARNING) << "DFS " << op_name << " failed for key " << op.key
                         << ", error=" << result.error();
            results[op.index] = tl::make_unexpected(result.error());
        } else if (*result == op.size) {
            continue;                        // 传满，正常路径
        } else if (complete_short_io && *result != 0 && *result < op.size) {
            auto rest =
                TransferAll(*fs_adapter_, io_requests[i].fd, std::move(op.iovs),
                            io_requests[i].offset, op.size, *result, write);  // 从断点续传
            if (!rest) results[op.index] = tl::make_unexpected(rest.error());
        } else {                             // 短传输但不允许续传，或返回 0
            LOG(WARNING) << "DFS short " << op_name << " for key " << op.key
                         << ", expected=" << op.size << ", actual=" << *result;
            results[op.index] = tl::make_unexpected(
                write ? ErrorCode::FILE_WRITE_FAIL : ErrorCode::FILE_READ_FAIL);
        }
    }
}
```

**这一段在做什么**

- 一次适配器调用承载整批请求，3FS 的 USRBIO 就落在这里；本层不关心底层是 pwritev 还是 USRBIO。
- 返回条数与请求数不符时整批判 INTERNAL_ERROR，不做“部分成功”猜测，因为无法确定哪条结果属于哪个请求。
- complete_short_io=true（bucket）时用 TransferAll 从断点续传；false（shard）时短传输直接判失败——这就是两种语义。
- *result != 0 这个附加条件让“返回 0 字节”永远落到失败分支，否则 TransferAll 会拿着 0 进度空转。
- 错误一律按 io_ops[i]->index 回填，调用方看到的失败位置与它提交请求的顺序一致。

**重点语法 / 惯用法**

- std::span<const FdIoRequest> 形参：io_requests 是 vector，可隐式转换成 span，无需拷贝。
- std::move(op.iovs) 只在续传分支发生；成功分支保持 op.iovs 原样，不能无条件 move。
- explicit operator bool：if (!result) 判断 expected 失败，需先确认非空才能取 error()。
- else if 链最后的 else 同时兜住两种情况：短传输但不允许续传、以及返回 0 字节。

**与 6.7.6 的差异**：图写“BatchReadAt 直接写入调用方 slices”：本层 iovec 的 base 确实就是调用方 slice 指针，但 3FS 适配器内部可能再经一层 bounce buffer，这不在本文件；图上的“零拷贝”只是相对文件 I/O 而言。另外，批量 Get 会把多个 key、多个 descriptor 合进同一次调用，图里画成一次 Get 对应一次 BatchReadAt 也不对。

### 2.7 适配器：PosixFsAdapter / Hf3fsAdapter（第 8 步）

数据的最后一次下沉：`pwritev` 与 3FS USRBIO。

#### 2.7.1 适配器内部的两个公共前置：ValidateIov 与 errno→ErrorCode 映射

`mooncake-store/src/storage/distributed/posix_fs_adapter.cpp:17-32`

```cpp
tl::expected<void, ErrorCode> ValidateIov(const iovec* iov, int iovcnt) {
    if (iovcnt < 0 || (iovcnt > 0 && iov == nullptr)) {  // 计数为负或空数组先挡掉
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    for (int i = 0; i < iovcnt; ++i) {
        if (!iov[i].iov_base && iov[i].iov_len > 0) {  // 有长度却没基址才算非法
            return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
        }
    }
    return {};  // 空 expected 即成功，无值可返回
}

ErrorCode ReadOpenError() {
    return errno == ENOENT ? ErrorCode::FILE_NOT_FOUND  // 只有 ENOENT 才是"不存在"
                           : ErrorCode::FILE_OPEN_FAIL;  // 其余统一归为打开失败
}
```

**这一段在做什么**

- ValidateIov 是 WriteAt/ReadAt 共用的参数闸门，负责把非法 iovec 挡在内核调用之前。
- 注意 iov_base 为 nullptr 本身不非法，只有 iov_len>0 时才算错，这是允许零长度片段的写法。
- ReadOpenError 读取的是调用点刚失败后尚未被覆盖的全局 errno，所以必须在 open 后立刻调用。
- FILE_NOT_FOUND 与 FILE_OPEN_FAIL 的区分会决定上层是否把该对象当作"已知不存在"处理，不要随手合并。

**重点语法 / 惯用法**

- tl::expected<void, E>：无值成功用 return {} 表示，失败用 tl::make_unexpected(...) 构造。
- 匿名 namespace 包裹：这两个符号仅本翻译单元可见，等价于内部链接。
- 三元运算符直接返回 ErrorCode 枚举值，未加 static_cast 是因为同类型。
- errno 是线程局部的全局变量，任何库调用都可能覆盖它，读取窗口要尽量短。

**与 6.7.6 的差异**：无。图上前置校验与错误码映射这两步没有画，但也不影响时序。

#### 2.7.2 PosixFsAdapter::OpenFile / OpenExistingFile：flags 差异与 ENOENT 映射

`mooncake-store/src/storage/distributed/posix_fs_adapter.cpp:91-105`

```cpp
tl::expected<int, ErrorCode> PosixFsAdapter::OpenFile(const std::string& path) {
    int fd = ::open(path.c_str(), O_RDWR | O_CREAT | O_CLOEXEC, 0644);  // 写路径：不存在就建
    if (fd < 0) return tl::make_unexpected(ErrorCode::FILE_OPEN_FAIL);
    return fd;
}

tl::expected<int, ErrorCode> PosixFsAdapter::OpenExistingFile(
    const std::string& path) {
    int fd;
    do {
        fd = ::open(path.c_str(), O_RDWR | O_CLOEXEC);  // 无 O_CREAT：只开已存在文件
    } while (fd < 0 && errno == EINTR);  // open 可能被信号打断，重试
    if (fd < 0) return tl::make_unexpected(ReadOpenError());  // ENOENT→FILE_NOT_FOUND
    return fd;
}
```

**这一段在做什么**

- OpenFile 带 O_CREAT|O_RDWR，用于 shard/bucket 首建以及 PreallocateFile 之前的建文件场景。
- OpenExistingFile 故意不带 O_CREAT：bucket descriptor 可能在整桶淘汰后才到达，此时应该报 FILE_NOT_FOUND 而不是凭空建出一个空文件。
- 只有 OpenExistingFile 做了 EINTR 重试并区分 ENOENT；OpenFile 失败一律 FILE_OPEN_FAIL，因为它理论上总能创建成功。
- O_CLOEXEC 很关键：这些 fd 会长期缓存到 shard 表里，不能被 fork/exec 出去的子进程继承。

**重点语法 / 惯用法**

- ::open 加全局作用域符，避免成员函数或命名空间里的同名解析。
- open 的第三个参数 0644 只在带 O_CREAT 时生效，但两个调用都写了，缺少时是未定义行为。
- fd 用 int 装在 tl::expected<int, ErrorCode> 中，成功值就是裸 fd，不做 RAII 包装（由调用方 CloseFile）。
- do-while(ret<0 && errno==EINTR) 是 POSIX 系统调用的标准重试惯用法。

**与 6.7.6 的差异**：图上没有 open 这一步。6.7.6 写的是 "GetOrOpenShard + BatchWriteAt"，而实际 POSIX 路径的 OpenFile 只在首次建 shard 时调用，读写复用缓存 fd；OpenExistingFile 服务于重开/桶模式，图里完全缺席。

#### 2.7.3 PosixFsAdapter::WriteAt：pwritev + EINTR 重试 + 返回真实短写字节数

`mooncake-store/src/storage/distributed/posix_fs_adapter.cpp:128-145`

```cpp
tl::expected<size_t, ErrorCode> PosixFsAdapter::WriteAt(int fd,
                                                        const iovec* iov,
                                                        int iovcnt,
                                                        int64_t offset) {
    if (fd < 0 || offset < 0) {  // 负 offset 不能静默转成 off_t
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    auto valid = ValidateIov(iov, iovcnt);
    if (!valid) return tl::make_unexpected(valid.error());
    if (iovcnt == 0) return size_t{0};  // 空写直接成功，不敲系统调用

    ssize_t ret;
    do {
        ret = ::pwritev(fd, iov, iovcnt, static_cast<off_t>(offset));  // 定位写，不动 fd 游标
    } while (ret < 0 && errno == EINTR);  // 被信号打断则原样重试
    if (ret < 0) return tl::make_unexpected(ErrorCode::FILE_WRITE_FAIL);
    return static_cast<size_t>(ret);  // 可能小于总长，短写照实上报
}
```

**这一段在做什么**

- offset 为负是 INVALID_PARAMS 而非写失败，因为这是调用方 bug，不是存储层故障。
- pwritev 的原子性只保证单次系统调用内；pwritev 返回小于总长即短写，本函数不循环补写，把补写责任交给上层。
- 返回真实字节数是刻意设计：上层 DistributedStorageBackend 用 *result != op.size 判定短写并报错，所以这里绝不能返回"期望长度"。
- 定位写不改 fd 的文件偏移，这是多个 key 共享同一 shard fd 时安全的前提。

**重点语法 / 惯用法**

- ssize_t 与 size_t 之间必须显式 static_cast，直接隐式转换会产生 -Wsign-conversion 告警。
- static_cast<off_t>(offset) 把协议里的 int64_t 转成平台 off_t，32 位平台上会截断（这是隐含前提）。
- size_t{0} 用花括号初始化，确保返回的是"成功值 0"而不是被 expected 当成失败构造。
- if (!valid) 利用了 tl::expected 的 explicit operator bool，失败时取 valid.error() 传递错误码。

**与 6.7.6 的差异**：图上是 "BE->>FS: GetOrOpenShard + BatchWriteAt" 一步完成。实际上 POSIX 适配器没有覆写 BatchWriteAt，走基类默认实现：逐请求串行调用 WriteAt，即每个 key 一次 pwritev。图里也看不出短写会被判失败。

#### 2.7.4 PosixFsAdapter::ReadAt：preadv 与同样的短读语义

`mooncake-store/src/storage/distributed/posix_fs_adapter.cpp:147-163`

```cpp
tl::expected<size_t, ErrorCode> PosixFsAdapter::ReadAt(int fd, iovec* iov,
                                                       int iovcnt,
                                                       int64_t offset) {
    if (fd < 0 || offset < 0) {
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    auto valid = ValidateIov(iov, iovcnt);
    if (!valid) return tl::make_unexpected(valid.error());
    if (iovcnt == 0) return size_t{0};

    ssize_t ret;
    do {
        ret = ::preadv(fd, iov, iovcnt, static_cast<off_t>(offset));  // 按偏移散读进 slices
    } while (ret < 0 && errno == EINTR);
    if (ret < 0) return tl::make_unexpected(ErrorCode::FILE_READ_FAIL);
    return static_cast<size_t>(ret);  // 返回实际读到字节，可能少于请求
}
```

**这一段在做什么**

- iovec* 在 ReadAt 中是输出参数，preadv 直接把数据写进调用方提供的 slices，这是 POSIX 路径唯一的"真零拷贝"点。
- 与 WriteAt 结构完全对称，唯一差别是错误码用 FILE_READ_FAIL，便于日志区分读写方向。
- 读到文件尾时 preadv 返回 0，本函数把它当作合法的"成功 0 字节"返回，短读判定由上层 ExecuteSingleIo 的 *result != op.size 负责。
- 对同一 shard fd 的并发读由 DistributedStorageBackend 的 shard->mutex 串行化，适配器自身不加锁。

**重点语法 / 惯用法**

- preadv 的第二个参数是非 const 的 iovec*，因为内核要往里写；const 版本只存在于 write 方向。
- 同一个 ValidateIov 同时接受 const iovec* 与 iovec*，因为 iovec* 可隐式转 const iovec*。
- static_cast<size_t>(ret) 在 ret>=0 分支内执行，负值已被上面的 if 拦掉，不会下溢成巨大数值。

**与 6.7.6 的差异**：6.7.6 图中 BatchReadAt 这一步写作 "直接写入调用方 slices"，这只在 POSIX 路径成立；3FS 路径是读进共享 iov 后再 memcpy，见后续 Hf3fsAdapter::ReadAt 一段。

#### 2.7.5 Hf3fsAdapter::OpenFile / OpenExistingFile / CloseFile：open 之后必须 hf3fs_reg_fd

`mooncake-store/src/storage/distributed/hf3fs_adapter.cpp:99-134`

```cpp
tl::expected<int, ErrorCode> Hf3fsAdapter::OpenFile(const std::string& path) {
    int fd = open(path.c_str(), O_RDWR | O_CREAT | O_CLOEXEC, 0644);  // 先拿普通 VFS fd
    if (fd < 0) return tl::make_unexpected(ErrorCode::FILE_OPEN_FAIL);

    if (hf3fs_reg_fd(fd, 0) > 0) {  // 3FS 约定返回 >0 才是失败
        close(fd);  // 注册失败必须自己回收 fd，否则泄漏
        return tl::make_unexpected(ErrorCode::FILE_OPEN_FAIL);
    }
    return fd;
}

tl::expected<int, ErrorCode> Hf3fsAdapter::OpenExistingFile(
    const std::string& path) {
    int fd;
    do {
        fd = open(path.c_str(), O_RDWR | O_CLOEXEC);  // 同样不带 O_CREAT
    } while (fd < 0 && errno == EINTR);
    if (fd < 0) {
        return tl::make_unexpected(errno == ENOENT ? ErrorCode::FILE_NOT_FOUND
                                                   : ErrorCode::FILE_OPEN_FAIL);
    }
    if (hf3fs_reg_fd(fd, 0) > 0) {  // 未注册的 fd 不能走 USRBIO 读写
        close(fd);
        return tl::make_unexpected(ErrorCode::FILE_OPEN_FAIL);
    }
    return fd;
}

tl::expected<void, ErrorCode> Hf3fsAdapter::CloseFile(int fd) {
    if (fd < 0) return tl::make_unexpected(ErrorCode::FILE_INVALID_HANDLE);
    hf3fs_dereg_fd(fd);  // 返回值不检查：dereg 失败不该阻断 close
    if (close(fd) != 0) {
        return tl::make_unexpected(ErrorCode::FILE_INVALID_HANDLE);
    }
    return {};
}
```

**这一段在做什么**

- OpenFile/OpenExistingFile 的 flags 与 POSIX 版逐字相同，唯一新增是 hf3fs_reg_fd(fd,0)，把 fd 登记到 3FS 用户态 I/O 通道。
- hf3fs_reg_fd 用 >0 表示失败（C API 返回错误码约定），写成 !=0 会把成功当失败。
- 注册失败必须先 close(fd) 再返回，否则每次失败都漏一个 fd，长时间运行会打爆 fd 上限。
- CloseFile 先 dereg 再 close，顺序不能反：dereg 需要有效 fd。dereg 的返回值被刻意忽略，避免把"清理失败"升级成对象级错误。
- FileExists 用的是 access()，走的是 FUSE/VFS 命名空间而不是 USRBIO，源码注释明确假设两者命名空间一致。

**重点语法 / 惯用法**

- if (hf3fs_reg_fd(fd, 0) > 0) 直接以 >0 判败，是 C 风格错误码与 tl::expected 混用时的典型适配。
- 错误码三元表达式跨行书写，clang-format 把 ? 与 : 对齐在操作数列。
- hf3fs_dereg_fd(fd); 忽略返回值要加 [[maybe_unused]] 或直接不接收，否则触发 -Wunused-result（此处该函数无该属性）。
- 类内声明 SupportsBatchIo()=true（include/storage/distributed/hf3fs_adapter.h:43），OpenFile 系列则完全没有声明 SupportsFdReopen，即继承基类默认 false。

**与 6.7.6 的差异**：时序图把 "GetOrOpenShard + BatchWriteAt" 画成一步；3FS 实际多出 reg/dereg 两个登记动作，图里未画。这也是 3FS 不开 SupportsFdReopen 的根因：fd 之外还有注册状态与每线程 ior/iov。

#### 2.7.6 Hf3fsAdapter::WriteAt（上）：参数校验 + 取线程 USRBIO 资源 + memcpy 进注册 iov

`mooncake-store/src/storage/distributed/hf3fs_adapter.cpp:157-201`

```cpp
tl::expected<size_t, ErrorCode> Hf3fsAdapter::WriteAt(int fd, const iovec* iov,
                                                      int iovcnt,
                                                      int64_t offset) {
    if (fd < 0 || offset < 0 || iovcnt < 0 || (iovcnt > 0 && iov == nullptr)) {
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    for (int i = 0; i < iovcnt; ++i) {
        if (!iov[i].iov_base && iov[i].iov_len > 0) {
            return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
        }
    }

    auto* resource = resource_manager_->getThreadResource();  // 每线程一份 iov/ior
    if (!resource || !resource->initialized) {  // 线程资源没建起来就别往下走
        return tl::make_unexpected(ErrorCode::FILE_OPEN_FAIL);
    }

    size_t total_length = 0;
    for (int i = 0; i < iovcnt; ++i) total_length += iov[i].iov_len;
    if (total_length == 0) return size_t{0};  // 零长度请求：成功且不发 I/O

    auto& threefs_iov = resource->iov_;  // 注册过的共享内存段
    auto& ior_write = resource->ior_write_;
    size_t total_written = 0;
    size_t remaining = total_length;
    int iov_idx = 0;  // 跨 chunk 保持游标，不被 while 重置
    size_t iov_off = 0;
    off_t current_offset = static_cast<off_t>(offset);

    while (remaining > 0) {
        size_t chunk = std::min(remaining, resource->config_.iov_size);  // 单次最多 32MB
        size_t copied = 0;
        char* dest = reinterpret_cast<char*>(threefs_iov.base);  // 每次都从段首开始填
        while (copied < chunk && iov_idx < iovcnt) {
            if (iov_off >= iov[iov_idx].iov_len) {  // 当前片写完，跳到下一片
                ++iov_idx;
                iov_off = 0;
                continue;
            }
            size_t n = std::min(chunk - copied, iov[iov_idx].iov_len - iov_off);
            memcpy(dest + copied,
                   static_cast<char*>(iov[iov_idx].iov_base) + iov_off, n);
            copied += n;
            iov_off += n;
        }
```

**这一段在做什么**

- 这里没有复用 ValidateIov，校验是内联展开的一份拷贝，行为与 POSIX 版一致。
- getThreadResource() 用线程 id 查表，每个线程一份 hf3fs_iov + 读写各一个 ior；所以适配器不线程安全的问题被下沉成了"每线程一份资源"。
- chunk 上限来自 Hf3fsConfig::iov_size（默认 32MB），超过就分多轮 prep/submit/wait。
- iov_idx / iov_off 定义在 while 之外：跨 chunk 连续推进，否则第二轮会从 slices 头部重拷。
- memcpy 是 3FS 路径的"bounce buffer"成本来源：用户数据必须先落进注册共享内存，USRBIO 才能直接 DMA。
- copied 可能小于 chunk（iov 总量不足时），下一段用 copied==0 兜底防死循环。

**重点语法 / 惯用法**

- auto& threefs_iov = resource->iov_; 绑定引用，避免反复解引用指针。
- reinterpret_cast<char*>(threefs_iov.base) 把 void* 转可做指针运算的 char*。
- static_cast<char*>(iov[i].iov_base) + iov_off：先转 char* 再做字节偏移，void* 算术非法。
- std::min(remaining, resource->config_.iov_size)：两参类型不同（size_t 与 size_t）此处一致，换类型时要显式转换。
- 外层 while 的游标语义类似手工实现的状态机，读的时候要同时盯 remaining/iov_idx/iov_off 三个量。

**与 6.7.6 的差异**：时序图写 "BE->>FS: BatchWriteAt(descriptor,slices)"，暗示数据直接交给文件系统。3FS 实际多一层：先把 slices memcpy 进线程私有注册 iov，再从 iov 发 USRBIO；图上没有这个 bounce buffer，也没有每线程资源准备这一步。

#### 2.7.7 Hf3fsAdapter::WriteAt（下）：prep/submit/wait(1) 与"未传满即失败"

`mooncake-store/src/storage/distributed/hf3fs_adapter.cpp:202-226`

```cpp
        if (copied == 0) break;  // 一个字节都没拷进 iov：避免空 prep

        int ret =
            hf3fs_prep_io(&ior_write, &threefs_iov, false, threefs_iov.base, fd,
                          current_offset, copied, nullptr);  // false = 写方向
        if (ret < 0) break;
        ret = hf3fs_submit_ios(&ior_write);
        if (ret < 0) break;
        struct hf3fs_cqe cqe;
        ret = hf3fs_wait_for_ios(&ior_write, &cqe, 1, 1, nullptr);  // 同步等 1 个完成
        if (ret < 0 || cqe.result < 0) break;  // 真正字节数在 cqe.result

        size_t bytes_written = cqe.result;
        if (bytes_written == 0) break;  // 0 字节当失败，避免 remaining 不减死循环
        total_written += bytes_written;
        current_offset += bytes_written;
        remaining -= bytes_written;
        if (bytes_written < copied) break;  // 短写立即退出，不补写
    }

    if (total_written != total_length) {  // 只有整份写满才算成功
        return tl::make_unexpected(ErrorCode::FILE_WRITE_FAIL);
    }
    return total_written;
}
```

**这一段在做什么**

- prep_io 的第三个参数 false 表示写（读方向为 true），第 4 个参数是数据在 iov 内的起始地址。
- wait_for_ios(&ior, &cqe, 1, 1, nullptr) 一次只等 1 个完成事件，所以这是一个完全同步、无流水线的循环；3FS 的排队优势没在这里发挥。
- 错误判断要看两处：hf3fs_* 返回值和 cqe.result，后者才是真正的传输字节数，为负表示该 I/O 失败。
- 任何 break 都会让 total_written != total_length，最终统一收敛成 FILE_WRITE_FAIL，不区分短写还是硬错误。
- 也就是说 Hf3fsAdapter::WriteAt 是"全有或全无"语义，绝不返回部分成功；这与 POSIX 版返回真实短写字节数形成对比。

**重点语法 / 惯用法**

- struct hf3fs_cqe cqe; 是纯 C 结构体，用 struct 前缀声明（C++ 中可省，但沿用 C API 风格）。
- cqe.result 类型为 ssize_t/int，赋给 size_t bytes_written 前已经由上面 if 保证非负。
- 多分支 break 共用一个出口，靠最后的 total_written != total_length 汇总失败，避免在循环内生成多条错误路径。
- hf3fs_wait_for_ios 末参 nullptr 是超时参数，表示使用 ior 默认行为。

**与 6.7.6 的差异**：图中 execute 阶段的 BatchWriteAt 被当成一次 FS 调用。代码里它被拆成 prep→submit→wait→校验四步同步往返，每 32MB 一轮；图也没体现"短写一律失败"这一语义。

#### 2.7.8 Hf3fsAdapter::ReadAt 的比较：读进共享 iov，再 memcpy 回调用方 slices

`mooncake-store/src/storage/distributed/hf3fs_adapter.cpp:257-294`

```cpp
    while (remaining > 0) {
        size_t chunk = std::min(remaining, resource->config_.iov_size);
        int ret = hf3fs_prep_io(&ior_read, &threefs_iov, true, threefs_iov.base,
                                fd, current_offset, chunk, nullptr);  // true = 读方向
        if (ret < 0) break;
        ret = hf3fs_submit_ios(&ior_read);
        if (ret < 0) break;
        struct hf3fs_cqe cqe;
        ret = hf3fs_wait_for_ios(&ior_read, &cqe, 1, 1, nullptr);
        if (ret < 0 || cqe.result < 0) break;

        size_t bytes_read = cqe.result;
        if (bytes_read == 0) break;  // 0 字节：失败，防止空转

        size_t to_copy = bytes_read;
        char* src = reinterpret_cast<char*>(threefs_iov.base);
        while (to_copy > 0 && iov_idx < iovcnt) {
            if (iov_off >= iov[iov_idx].iov_len) {  // 片写完，推进到下一片
                ++iov_idx;
                iov_off = 0;
                continue;
            }
            size_t n = std::min(to_copy, iov[iov_idx].iov_len - iov_off);
            memcpy(static_cast<char*>(iov[iov_idx].iov_base) + iov_off, src, n);
            src += n;
            to_copy -= n;
            total_read += n;  // 注意：在拷贝循环里累加
            remaining -= n;
            current_offset += n;
            iov_off += n;
        }
        if (bytes_read < chunk) break;  // 短读：交给末尾统一判失败
    }

    if (total_read != total_length) {
        return tl::make_unexpected(ErrorCode::FILE_READ_FAIL);
    }
    return total_read;
}
```

**这一段在做什么**

- 读方向是 write 的镜像：多出"从 iov 拷回 slices"的第二段循环，`to_copy` 控制拷贝量。
- total_read / remaining / current_offset 只在成功拷出的字节上累加，与 bytes_read 解耦，所以拷贝没填满时不会误判进度。
- `if (bytes_read < chunk) break;` 放在拷贝之后：先把已读到的部分交给调用方，再决定失败，保证缓冲不留半截脏数据给上层复用。
- 与写一样是 all-or-nothing：total_read != total_length 即 FILE_READ_FAIL，不返回短读长度。

**重点语法 / 惯用法**

- prep_io 第三个参数 true 表示读，语义与 WriteAt 的 false 相对。
- 两个 while 共用 iov_idx/iov_off，读循环与写循环的游标推进方式完全对称。
- src 与 dest 命名区分"iov 内地址"与"调用方地址"，读时 src 是 iov.base、目标是 slices。
- memcpy 的参数顺序（目的, 源）在写/读两段正好互换，是 review 时最容易看错的地方。

**与 6.7.6 的差异**：图中 "BatchReadAt 直接写入调用方 slices" 对 3FS 不成立：数据先落进每线程注册 iov，再 memcpy 到 slices。图中的"零拷贝"描述只适用于 PosixFsAdapter::ReadAt 的 preadv。

#### 2.7.9 SupportsFdReopen / SupportsBatchIo：POSIX 与 3FS 的能力开关

`mooncake-store/include/storage/distributed/fs_adapter.h:113-137`

```cpp
    virtual tl::expected<size_t, ErrorCode> WriteAt(int /*fd*/,
                                                    const iovec* /*iov*/,
                                                    int /*iovcnt*/,
                                                    int64_t /*offset*/) {
        return tl::make_unexpected(ErrorCode::NOT_SUPPORTED);
    }

    virtual tl::expected<size_t, ErrorCode> ReadAt(int /*fd*/, iovec* /*iov*/,
                                                   int /*iovcnt*/,
                                                   int64_t /*offset*/) {
        return tl::make_unexpected(ErrorCode::NOT_SUPPORTED);
    }

    // Whether a hard WriteAt/ReadAt failure can be recovered by closing and
    // reopening the fd (stale-fd self-healing). Only meaningful for adapters
    // backed by a plain long-lived positional fd (POSIX). Adapters whose I/O
    // failures are unrelated to the file fd (e.g. 3FS USRBIO submit/completion
    // or per-thread ior/iov state) must leave this false so the backend does
    // not perform a pointless close/reopen/retry under the shard lock.
    virtual bool SupportsFdReopen() const { return false; }  // 默认关闭

    // Opt in to backend batching and concurrent positional I/O on cached
    // shard fds. Other adapters retain per-request bucket handles and
    // serialized shard I/O.
    virtual bool SupportsBatchIo() const { return false; }  // 默认逐请求串行
```

**这一段在做什么**

- 基类把 fd 级 I/O 全部默认成 NOT_SUPPORTED，只有 POSIX 与 3FS 覆写；这让对象存储适配器（S3/OSS）天然不支持 shard 模式。
- SupportsFdReopen 只有 posix_fs_adapter.h:37 覆写为 true，Hf3fsAdapter 保持基类 false，这是设计意图而非遗漏。
- 3FS 不开 FdReopen 的理由写在注释里：USRBIO 的失败来自 submit/completion 或每线程 ior/iov 状态，close/reopen fd 修不好，反而在 shard 锁内白跑一次重放。
- POSIX 失败多为 stale fd 或 shard 文件轮转，close+reopen 确实能自愈，所以开启。
- SupportsBatchIo 只有 3FS 覆写为 true（hf3fs_adapter.h:43）；它决定 backend 是"收集后批量提交"还是"在循环里逐条执行"。

**重点语法 / 惯用法**

- virtual ... = 0 与带默认实现的虚函数混用：纯虚强制覆写，非纯虚给可选项，两者都在同一个接口类里。
- 参数写成注释形式 /*fd*/ 消除未使用参数告警，是接口默认实现的常用写法。
- const 成员函数返回 bool，配合后端 bool batch_io = fs_adapter_->SupportsBatchIo(); 做运行期分派。
- 同一族能力开关放在基类注释里说明"什么时候该保持 false"，比分散在子类更利于维护。

**与 6.7.6 的差异**：时序图没有任何"重开 fd"或"批量提交开关"的分支。图中的 GetOrOpenShard→BatchWriteAt 在代码里被 SupportsBatchIo 一分为二：3FS 收集后批量执行，POSIX 在循环里直接 WriteAt。

### 2.8 提交：PutEnd / PutRevoke（第 9–10 步）

图上 `C->>M: PutEnd(ALL)` 与 `M->>M: DFS 副本 → COMPLETE`。代码里比图多一步 bucket 的 `MarkCommitted`。

#### 2.8.1 PutEnd 入口：谁有权提交、对象必须存在

`mooncake-store/src/master_service.cpp:5663-5680`

```cpp
auto MasterService::PutEnd(const UUID& client_id, const ObjectMeta& object_meta,
                           const TenantId& tenant_id, ReplicaType replica_type)
    -> tl::expected<void, ErrorCode> {                 // 尾置返回：失败要带错误码、成功无返回值
    const auto& key = object_meta.key;
    std::shared_lock<std::shared_mutex> shared_lock(master_state_mutex_); // 读多写少，先拿共享锁
    const auto object_id = MakeObjectIdentityForRequest(key, tenant_id);
    MetadataAccessorRW accessor(this, object_id);       // RAII 句柄：定位到该 key 的元数据
    if (!accessor.Exists()) {
        LOG(ERROR) << "key=" << key << ", error=object_not_found";
        return tl::make_unexpected(ErrorCode::OBJECT_NOT_FOUND);
    }

    auto& metadata = accessor.Get();
    if (client_id != metadata.client_id) {              // 只有 PutStart 的那个客户端能提交
        LOG(ERROR) << "Illegal client " << client_id << " to PutEnd key " << key
                   << ", was PutStart-ed by " << metadata.client_id;
        return tl::make_unexpected(ErrorCode::ILLEGAL_CLIENT);
    }
```

**这一段在做什么**

- PutEnd 是 DFS 写入链路里“客户端说数据已经写完”的提交点，时序图上对应 C->>M 那一步。
- 第一道门是 client_id 必须等于 PutStart 时记录的 client_id，防止别的客户端抢先替它提交。
- Exists() 同时检查对象已发布且 metadata 仍有效，避免对已擦除的 key 做提交。
- shared_lock 在函数返回时自动释放，所有提前 return 的分支都不会漏解锁。

**重点语法 / 惯用法**

- 尾置返回类型 `-> tl::expected<void, ErrorCode>`：用 tl::expected 表达“可能失败的无返回值 RPC”，错误用 `tl::make_unexpected` 包装。
- `std::shared_lock<std::shared_mutex>`：RAII 共享锁，构造加锁、析构解锁。
- `MetadataAccessorRW`：把“查表 + 加锁 + 校验”封装成一个对象，`Get()` 返回可写引用。

**与 6.7.6 的差异**：图上只画了 `C->>M: PutEnd(ALL)`，没有画 client_id 不匹配和 OBJECT_NOT_FOUND 两条失败分支；正常路径确实是一次 ALL（end_type 由客户端 DetermineFinalizeDecision 决定）。

#### 2.8.2 is_target_replica：ALL 为什么必须显式列出三类副本

`mooncake-store/src/master_service.cpp:5682-5699`

```cpp
    auto is_target_replica = [replica_type](const Replica& replica) {  // 按值捕获本次 End 的范围
        if (replica_type == ReplicaType::ALL) {                        // ALL 不做类型相等比较
            return (replica.is_memory_replica() &&
                    !replica.has_invalid_mem_handle()) ||              // MEMORY 还要求句柄有效
                   (replica.is_nof_replica() &&
                    !replica.has_invalid_nof_handle()) ||              // NoF 同理
                   replica.is_dfs_replica();                           // DFS 无句柄，看类型即可
        }
        if (replica_type == ReplicaType::MEMORY) {
            return replica.is_memory_replica() &&
                   !replica.has_invalid_mem_handle();
        }
        if (replica_type == ReplicaType::NOF_SSD) {
            return replica.is_nof_replica() &&
                   !replica.has_invalid_nof_handle();
        }
        return replica.type() == replica_type;                         // 其余类型直接比枚举
    };
```

**这一段在做什么**

- ALL 不是“所有副本”，而是“三类可写副本的并集”：MEMORY、NoF、DFS，LOCAL_DISK 被显式排除。
- MEMORY / NoF 额外要求句柄仍然有效，避免把已被回收的远端句柄标记成 COMPLETE。
- DFS 副本只检查类型，因为它的 descriptor 是 path/offset，不存在运行时句柄。
- 这个 lambda 同时被后面的幂等检查和 VisitReplicas 复用，保证两处判断口径完全一致。

**重点语法 / 惯用法**

- 带捕获的 lambda 随后被隐式转成 `std::function<bool(const Replica&)>` 传给 `VisitReplicas`；按值捕获枚举是安全的。
- `&&` 优先级高于 `||`，这里 `a && b || c && d || e` 不需要额外括号，括号只用于可读性。
- `ReplicaType::ALL` 是枚举哨兵值，不是 `Replica::type()` 可能返回的值，所以必须单独分支。

**与 6.7.6 的差异**：图把 `PutEnd(ALL)` 画成一个整体动作，实际 ALL 在代码里是三段显式布尔表达式，且 LOCAL_DISK 不参与；这点图上完全看不出来。

#### 2.8.3 幂等判定：只有全部目标副本 COMPLETE 才算成功

`mooncake-store/src/master_service.cpp:5701-5723`

```cpp
    // A successful End removes the processing marker. Treat a retry as a
    // no-op only when every replica targeted by that End is already COMPLETE.
    // In particular, a promotion-owned PROCESSING replica keeps this check
    // from accepting a MEMORY/ALL End and is never modified here.
    if (!accessor.InProcessing()) {                     // 对象已不在 processing 集合 = 重试/竞态
        bool has_target_replica = false;
        bool all_target_replicas_complete = true;
        for (const auto& replica : metadata.GetAllReplicas()) {
            if (!is_target_replica(replica)) {
                continue;                               // 不归这次 End 管的副本直接跳过
            }
            has_target_replica = true;
            if (!replica.is_completed()) {              // 只要发现一个没完成
                all_target_replicas_complete = false;
                break;
            }
        }
        if (has_target_replica && all_target_replicas_complete) {
            return {};
        }
        LOG(ERROR) << "key=" << key << ", error=no_primary_write_in_progress";
        return tl::make_unexpected(ErrorCode::INVALID_WRITE);
    }
```

**这一段在做什么**

- 只有对象已经退出 processing 集合时才进入这段，意味着这是重复 PutEnd 或与别人发生竞态。
- has_target_replica 是必要的：如果这次 End 一个目标副本都匹配不到，不能算成功。
- 必须全部目标副本都 is_completed() 才返回成功，任何一个还在 PROCESSING 都返回 INVALID_WRITE。
- 注释点出 promotion 持有的 PROCESSING 副本不会被这次 End 顺手接受，避免误判成功。

**重点语法 / 惯用法**

- `metadata.GetAllReplicas()` 返回 `std::span<const Replica>`，是只读视图，遍历不拷贝。
- `return {};` 构造 `tl::expected<void, E>` 的“有值但无内容”成功返回，等价于 `return tl::expected<void,E>{}`。
- `break` 在发现第一个未完成副本后立即跳出，扫描是短路语义。

**与 6.7.6 的差异**：图没有幂等重试分支：真实的重复 `PutEnd(ALL)` 在这里会直接返回成功，时序图上看不到这条路径。

#### 2.8.4 bucket 模式预提交：MarkCommitted 必须在 mark_complete 之前

`mooncake-store/src/master_service.cpp:5725-5743`

```cpp
    if (bucket_allocator_ != nullptr && (replica_type == ReplicaType::ALL ||
                                         replica_type == ReplicaType::DFS)) { // 仅 bucket 模式且有 DFS
        std::vector<DistributedFSDescriptor> pending_descriptors;
        metadata.VisitReplicas(
            [](const Replica& replica) {                    // 谓词：正在写、且是 DFS
                return replica.is_dfs_replica() && replica.is_processing();
            },
            [&pending_descriptors](const Replica& replica) { // 先收集，回调里不能改副本集合
                pending_descriptors.push_back(replica.get_dfs_descriptor());
            });
        for (const auto& descriptor : pending_descriptors) {
            if (!bucket_allocator_->MarkCommitted(key, descriptor)) { // PENDING 桶条目 → COMMITTED
                LOG(ERROR) << "key=" << key
                           << ", error=dfs_runtime_commit_failed, bucket_id="
                           << descriptor.shard_idx;
                return tl::make_unexpected(ErrorCode::FILE_WRITE_FAIL); // 失败即返回，不回滚
            }
        }
    }
```

**这一段在做什么**

- 只有 bucket 模式（bucket_allocator_ 非空）且 End 范围包含 DFS 时才做预提交；shard 模式没有这一步。
- 谓词用 `is_processing()`，所以这段必须跑在 mark_complete 之前，否则一个描述符都收集不到。
- 先收集 descriptor 再调 MarkCommitted，是因为 VisitReplicas 的回调约定不允许增删副本，而 allocator 调用会拿自己的锁。
- MarkCommitted 把桶条目的 PENDING 状态改成 COMMITTED，之后 GC 才不会把这块空间当垃圾回收。
- 失败直接返回 FILE_WRITE_FAIL 且不回滚已提交的条目，对象仍留在 processing，等待客户端 PutRevoke。

**重点语法 / 惯用法**

- 两个 lambda 分别以 `const Replica&` 和 `const Replica&` 传入 `VisitReplicas` 的 const 重载，编译器按 const 性选重载。
- `std::vector<DistributedFSDescriptor>` 按值存 descriptor，因为 allocator 的接口只接受 const 引用，不需要持有副本地址。
- `descriptor.shard_idx` 被当日志里的 bucket_id 打印，字段名与语义不完全对应，读日志时注意。

**与 6.7.6 的差异**：图上 `M->>M: DFS 副本 → COMPLETE` 之前没有任何预提交步骤，漏掉了 bucket 模式的 MarkCommitted，也漏掉了它可能返回 FILE_WRITE_FAIL 的失败出口。

#### 2.8.5 VisitReplicas：把 PROCESSING 副本推进到 COMPLETE，并刷新 DFS 访问

`mooncake-store/src/master_service.cpp:5745-5763`

```cpp
    const bool had_completed_replica =
        metadata.HasReplica(&Replica::fn_is_completed);  // 提交前是否已有完成副本
    bool completed_pending_replica = false;
    metadata.VisitReplicas(
        [&is_target_replica](const Replica& replica) {
            return replica.is_processing() && is_target_replica(replica); // 只碰本次目标且未完成的
        },
        [this, &key, &metadata, &completed_pending_replica](Replica& replica) {
            if (replica.is_processing() &&
                metadata.PendingSoftPinOwnsReplica(replica.id())) {
                completed_pending_replica = true;        // 记下软 pin 是否随本次提交落地
            }
            metadata.CommitSessionTag(replica.id());
            replica.mark_complete();                     // PROCESSING → COMPLETE
            if (replica.is_dfs_replica() && dfs_allocator_) {
                const auto& desc = replica.get_dfs_descriptor();
                dfs_allocator_->UpdateAccess(key, desc); // 刷新分片 LRU，别刚写完就被回收
            }
        });
```

**这一段在做什么**

- 谓词同时要求 `is_processing()` 与 `is_target_replica()`，所以只推进本次 End 负责的副本，不会误动别人正在写的副本。
- mark_complete() 对 MEMORY / NoF / DFS 一视同仁；图上只写“DFS 副本 → COMPLETE”是不完整的。
- DFS 多一步 UpdateAccess，把 descriptor 对应的 bucket/shard 推到 LRU 头部。
- CommitSessionTag 与后面的软 pin 提交配合，决定对象是否仍受 pending 软 pin 保护。
- 整个 PutEnd 里只有这里会把副本状态改成 COMPLETE，幂等重试的安全性依赖 mark_complete 自身的幂等。

**重点语法 / 惯用法**

- `&Replica::fn_is_completed` 是静态成员函数指针，可直接被 `std::function<bool(const Replica&)>` 接住，比写 lambda 更短。
- 访问器 lambda 捕获 `this` 与多个引用，是可写版本 `std::function<void(Replica&)>`。
- `const auto& desc = replica.get_dfs_descriptor();` 绑定到返回的 const 引用，避免复制 descriptor。
- `mark_complete()` 内部只在 PROCESSING 时迁移状态，对 COMPLETE 调用不会回退。

**与 6.7.6 的差异**：图上只有一个 `M->>M: DFS 副本 → COMPLETE` 自环；实际同一趟遍历会把 MEMORY / NoF 的 PROCESSING 副本一起置 COMPLETE，并额外对 DFS 做 UpdateAccess。

#### 2.8.6 checksum 写入、配额结算与 offload 入队门控

`mooncake-store/src/master_service.cpp:5772-5788`

```cpp
    if (object_meta.object_checksum.has_value() ||
        replica_type == ReplicaType::ALL ||
        replica_type == ReplicaType::MEMORY ||
        replica_type == ReplicaType::NOF_SSD) {          // DFS-only 且无 checksum 时不覆盖
        metadata.MutablePart(0).SetChecksum(object_meta.object_checksum);
    }

    auto settle_result =
        SettlePrimaryWriteQuotaIfReady(accessor.QuotaAccount(), metadata); // 主写完成才结算配额
    if (!settle_result) {
        return tl::make_unexpected(settle_result.error());
    }

    if (replica_type != ReplicaType::DFS && enable_offload_ &&
        !offload_on_evict_ && !metadata.HasReplica([](const Replica& replica) {
            return replica.is_dfs_replica() && replica.is_processing(); // 还有 DFS 在写就不入队
        })) {
```

**这一段在做什么**

- checksum 写入条件包含“带值的 checksum”和三种含内存/NoF 的 End；纯 DFS End 且 checksum 为空时保持原值。
- SetChecksum 作用在 MutablePart(0)，即多 part 元数据里的主 part。
- SettlePrimaryWriteQuotaIfReady 只在主写完成时结算租户配额，失败会让整个 PutEnd 返回错误。
- offload 门控三连：本次不是 DFS 的 End、启用了 offload、且不是 on-evict 模式。
- 只要还有 DFS 副本处于 PROCESSING，就不入 offload 队列——说明 DFS 数据还没落共享盘，不能提前把内存副本挪走。

**重点语法 / 惯用法**

- `HasReplica` 接收无捕获 lambda，空捕获列表 `[]` 可转成函数指针，再转 `std::function`。
- `metadata.MutablePart(0)` 返回可写 part，链式调用 `SetChecksum`。
- `if (...) { ... }` 的最后一个条件是带 lambda 的 `HasReplica(...)`，`!` 作用于整个调用结果，括号位置容易读错。

**与 6.7.6 的差异**：图上完全没有 checksum 写入、租户配额结算和 offload 入队这三步，它们都发生在 PutEnd 返回之前。

#### 2.8.7 PutEnd 收尾：什么时候才清掉 processing 标记

`mooncake-store/src/master_service.cpp:5813-5834`

```cpp
    // If the object is completed, remove it from the processing set.
    if (metadata.AllReplicas(&Replica::fn_is_completed) &&
        accessor.InProcessing()) {                       // 所有副本都完成才清标记
        accessor.EraseFromProcessing();
    }

    SyncCacheTotalAccounting(metadata);
    // TODO: add inc_nof_cache_nums() (ranhaojia)
    metadata.GrantReadLease(std::chrono::milliseconds::zero());
    PublishKvStored(key, metadata, object_id.tenant_id);

    if (enable_oplog_ && ordered_oplog_writer_) {
        std::string payload = SerializeMetadataForOpLog(metadata);
        auto result = AppendOpLogVisibleBeforeDurable(
            OpType::PUT_END, object_id.tenant_id.value(), key, payload);
        if (!result) {
            LOG(WARNING) << "PutEnd: OpLog queue failed for key=" << key
                         << ", err=" << static_cast<int>(result.error()); // 只告警不失败
        }
    }
    return {};
```

**这一段在做什么**

- EraseFromProcessing 的条件是“所有副本 COMPLETE”，不是“本次目标副本 COMPLETE”。
- 混合副本场景（例如 LOCAL_DISK 还在异步写）下，对象会继续留在 processing 集合里。
- GrantReadLease 传零时长，表示不额外续租；随后 PublishKvStored 把元数据同步到 KV 视图。
- OpLog 入队失败只打 WARNING 并继续返回成功，说明这条日志链路是尽力而为。
- 这段是 PutEnd 真正对读路径可见的收尾，时序图上通常被压缩成一个箭头。

**重点语法 / 惯用法**

- `&Replica::fn_is_completed` 作为 `AllReplicas` 的谓词参数，是静态成员函数指针的常规用法。
- `std::chrono::milliseconds::zero()` 显式构造零时长，语义由 GrantReadLease 解释。
- `static_cast<int>(result.error())`：枚举无法直接进日志流，必须先转成整数。
- `Enable_oplog_ && ordered_oplog_writer_`：双重开关，oplog 未启用或写入器未就绪时整段跳过。

**与 6.7.6 的差异**：图暗示 PutEnd 返回后对象即可读；实际只有全部副本 COMPLETE 才清 processing，混合副本时对象会继续处于 processing 状态。

#### 2.8.8 PutRevoke：与 PutEnd 对称的 ALL 谓词

`mooncake-store/src/master_service.cpp:5989-6018`

```cpp
    if (!accessor.InProcessing()) {
        LOG(ERROR) << "key=" << key << ", error=no_primary_write_in_progress";
        return tl::make_unexpected(ErrorCode::INVALID_WRITE);
    }

    auto processing_rep =
        metadata.GetFirstReplica([replica_type](const Replica& replica) {
            if (replica_type == ReplicaType::ALL) {       // 注意：谓词是“未在 processing”
                return (replica.is_memory_replica() ||
                        replica.is_nof_replica() || replica.is_dfs_replica()) &&
                       !replica.is_processing();
            }
            return replica.type() == replica_type && !replica.is_processing();
        });
    if (processing_rep != nullptr) {                      // 已有副本落定 → 不允许 Revoke
        LOG(ERROR) << "key=" << key << ", status=" << processing_rep->status()
                   << ", error=invalid_replica_status";
        return tl::make_unexpected(ErrorCode::INVALID_WRITE);
    }

    auto target_pred = [replica_type](const Replica& r) { // 真正要删的副本
        if (!r.is_processing()) {
            return false;                                 // 已 COMPLETE 的一个都不动
        }
        if (replica_type == ReplicaType::ALL) {
            return r.is_memory_replica() || r.is_nof_replica() ||
                   r.is_dfs_replica();                    // 与 PutEnd 同一套“三类并列”
        }
        return r.type() == replica_type;
    };
```

**这一段在做什么**

- PutRevoke 先确认对象仍在 processing，否则说明已经提交过，直接报 INVALID_WRITE。
- 变量名 processing_rep 有误导性：谓词实际找的是 `!is_processing()` 的副本，即“已经有副本落定”。
- 只要存在一个非 processing 的目标类型副本，就拒绝 Revoke，避免回滚掉已经对读方可见的数据。
- target_pred 与 PutEnd 的 is_target_replica 口径对齐：ALL 显式枚举 MEMORY / NoF / DFS。
- 两者差异在于 Revoke 只检查类型和 processing 状态，不检查句柄是否有效。

**重点语法 / 惯用法**

- `GetFirstReplica` 返回裸指针 `Replica*`，用 `!= nullptr` 判存在，而不是 optional。
- lambda 的返回值类型由多个 `return` 推导；所有分支都返回 bool，所以无需显式 `-> bool`。
- `replica_type == ReplicaType::ALL` 作为编译期不可知的运行期分支，两个 lambda 各写一遍，容易漏改其中一处。

**与 6.7.6 的差异**：6.7.6 的时序图完全没有 PutRevoke 分支；实际 PutEnd 失败或 FLEXIBLE_DUAL_REPLICA 下半成功时都会走到这里。

#### 2.8.9 Revoke 的清理：EraseReplicas 与配额退还

`mooncake-store/src/master_service.cpp:6063-6080`

```cpp
    const uint64_t before_charge = CompletedMemoryQuotaCharge(metadata);
    EraseReplicasWithCacheTotalAccounting(metadata, target_pred); // 按谓词批量摘除副本
    metadata.ClearPendingSoftPinIfNoViableReplica();
    metadata.ClearSessionTagIfNoViableReplica();
    const uint64_t after_charge = CompletedMemoryQuotaCharge(metadata);
    if (enable_multi_tenants_ && before_charge > after_charge) {
        auto release_result = metadata.quota_ledger.ReleaseCommitted(
            accessor.QuotaAccount(), before_charge - after_charge); // 退还差额
        if (!release_result) {
            LogTenantQuotaLedgerError(release_result, "release_committed",
                                      object_id.tenant_id, key);
            return tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
        }
    }
    if (!metadata.IsValid()) {
        accessor.Erase();                                 // 副本清空 → 整条元数据删除
        return {};
    }
```

**这一段在做什么**

- EraseReplicasWithCacheTotalAccounting 按 target_pred 批量摘除，同时修正 cache total 计数。
- 配额用“删前快照减删后快照”的差值来释放，避免逐个副本记账出错。
- 软 pin 与 session tag 在“没有可用副本”时才清，说明它们可能还挂着其他副本。
- 元数据失效后调用 accessor.Erase() 删除整个 key，而不是留一条空记录。
- 启用 oplog 时走的是另一条分支（mark_removed + 持久化回调），根本不会执行到这段同步删除。

**重点语法 / 惯用法**

- `before_charge` / `after_charge` 是 `uint64_t` 快照，比较用 `>` 而不是 `!=`，防御计数异常。
- `metadata.quota_ledger.ReleaseCommitted(...)` 返回 expected，失败转成 INTERNAL_ERROR 上抛。
- `return {};` 在这里表示“对象已彻底删除”也是一种成功。

**与 6.7.6 的差异**：图未画 Revoke 的清理路径；这里的同步删除与 6020-6061 的 oplog 分支互斥，图上都没有体现。

## 3. 读取路径：`get` → `bytes`

一段话版本：**RealClient 选副本，Client 执行**。RealClient 拿到 master 返回的全部副本后用 `SelectBestReplica` 挑一个，并用 `FilterQueryResult` 把列表裁成只剩它，再交给 `Client::Get`；Client 看到 DFS 描述符就调 `ReadDfsReplica`（批量则攒成一次 `BatchRead`），由 backend 直接把文件字节读进调用方 slices，最后做（默认关闭的）checksum 校验并返回。

### 3.1 查询与选副本（第 12–15 步）

图上 `App->>R: get` 到 `R->>R: SelectBestReplica`。注意「选副本」和「下发执行」分属 RealClient 与 Client 两层。

#### 3.1.1 Client::Query：读路径唯一的一次 master RPC

`mooncake-store/src/client_service.cpp:1240-1257`

```cpp
tl::expected<QueryResult, ErrorCode> Client::Query(
    const std::string& object_key) {
    std::chrono::steady_clock::time_point start_time =
        std::chrono::steady_clock::now();              // 记 RPC 发起前本地时刻
    auto result = master_client_.GetReplicaList(object_key);  // 全程唯一一次 master 调用
    if (!result) {
        return tl::unexpected(result.error());         // OBJECT_NOT_FOUND 等原样上抛
    }
    return tl::expected<QueryResult, ErrorCode>(
        tl::in_place, std::move(result.value().replicas),  // 副本列表整体搬走
        start_time + std::chrono::milliseconds(result.value().lease_ttl_ms),  // lease 本地算
        result.value().object_checksum, result.value().resolved_tenant_id);
}

std::vector<tl::expected<QueryResult, ErrorCode>> Client::BatchQuery(
    const std::vector<std::string>& object_keys) {
    return BatchQuery(object_keys, master_client_.tenant_id());  // 补 tenant 后走重载
}
```

**这一段在做什么**

- Query 只发一次 RPC，不做副本筛选；筛选在 RealClient 层。
- lease_timeout = 本地时钟 + master 给的 lease_ttl_ms，不是 master 的时间点。
- 副本顺序不保证，所以下游必须 SelectBestReplica 而不能取第一个。
- object_checksum 可能为空，VerifyObjectChecksum 会据此短路。
- BatchQuery 只是补齐 tenant_id 后复用同一逻辑，结果与 key 顺序对齐。

**重点语法 / 惯用法**

- tl::expected：std::expected 的 C++17 回填，错误分支用 tl::unexpected 构造。
- tl::in_place：in-place 构造 tag，省一次中间对象拷贝。
- std::move 把 replicas 从返回值中搬进 QueryResult。

**与 6.7.6 的差异**：图只画 R->>M: Query；代码里 lease 截止点是客户端自己用 start_time + lease_ttl_ms 算的，图上没有这一步。

#### 3.1.2 MasterService::GetReplicaList：读也是一次 LRU 访问

`mooncake-store/src/master_service.cpp:4795-4814`

```cpp
        const auto& metadata = accessor.Get();

        auto replica_list = GetReadableReplicaDescriptors(metadata);  // 先过滤出可读副本
        if (dfs_allocator_) {                       // 只有配了 DFS 分配器才执行
            for (const auto& replica : replica_list) {
                if (replica.is_dfs_replica()) {
                    const auto& desc = replica.get_dfs_descriptor();
                    dfs_allocator_->UpdateAccess(key, desc);   // 读 = 触碰一次 LRU
                }
            }
        }

        if (replica_list.empty()) {
            if (metadata.AllReplicas([](const Replica& replica) {
                    return replica.status() == ReplicaStatus::REMOVED;
                })) {
                return tl::make_unexpected(ErrorCode::OBJECT_NOT_FOUND);  // 全被删了
            }
            LOG(WARNING) << "key=" << key << ", error=replica_not_ready";
            return tl::make_unexpected(ErrorCode::REPLICA_IS_NOT_READY);
```

**这一段在做什么**

- UpdateAccess 只对 DFS 副本调用，说明 DFS 的淘汰热度挂在 shard/bucket allocator 上。
- 读不改状态、不续 lease，只是把访问热度记下来影响后续淘汰。
- 批量路径 BatchGetReplicaList 在 master_service.cpp:4971-4978 有一模一样的循环。
- 返回哪些副本由 GetReadableReplicaDescriptors 决定，这里只做访问记录。
- 全 REMOVED 报 OBJECT_NOT_FOUND，未就绪报 REPLICA_IS_NOT_READY。

**重点语法 / 惯用法**

- 尾置返回类型 `-> tl::expected<...>`。
- shared_lock 是 RAII：构造加共享锁、析构自动释放。
- lambda 作为谓词传给 AllReplicas，捕获列表为空。
- tl::make_unexpected 直接构造错误分支。

**与 6.7.6 的差异**：6.7.6 的 get 时序里 master 只回 replicas；代码里 master 还会对每个 DFS 副本调用 dfs_allocator_->UpdateAccess，这是一次隐式 LRU 访问，图上完全没有。

#### 3.1.3 SelectBestReplica：两遍扫描，DFS 在第二遍

`mooncake-store/include/replica_selection.h:122-175`

```cpp
inline const Replica::Descriptor *SelectBestReplica(
    const std::vector<Replica::Descriptor> &replicas,
    const std::unordered_set<std::string> &local_endpoints) {
    const Replica::Descriptor *local_memory = nullptr;   // 本机内存副本
    const Replica::Descriptor *local_nof = nullptr;      // 本机 NoF 副本
    const Replica::Descriptor *first_memory = nullptr;   // 远端内存副本
    const Replica::Descriptor *first_nof = nullptr;
    for (const auto &r : replicas) {                     // 第一遍：只扫 memory/nof
        if (r.status != ReplicaStatus::COMPLETE) continue;  // 未完成副本跳过
        if (r.is_memory_replica()) {
            if (local_endpoints.count(
                    r.get_memory_descriptor()
                        .buffer_descriptor.transport_endpoint_)) {  // endpoint 属于本机
                if (!local_memory) local_memory = &r;
            } else {
                if (!first_memory) first_memory = &r;
            }
        } else if (r.is_nof_replica()) {
            if (local_endpoints.count(
                    r.get_nof_descriptor()
                        .buffer_descriptor.transport_endpoint_)) {
                if (!local_nof) local_nof = &r;
            } else {
                if (!first_nof) first_nof = &r;
            }
        }
    }
    // Local MEMORY always beats local NOF regardless of list order.
    if (local_memory) return local_memory;   // 优先级 1
    if (local_nof) return local_nof;         // 优先级 2
    // No local replica. Among remote MEMORY replicas, optionally pick the
    // best-scoring one instead of the first encountered (issue #2516).
    if (first_memory && RemoteReplicaScoringEnabled()) {   // 打分开关
        if (const auto *scored =
                PickBestRemoteMemory(replicas, local_endpoints)) {
            return scored;
        }
    }
    if (first_memory) return first_memory;   // 优先级 3：远端内存
    if (first_nof) return first_nof;         // 优先级 4：远端 NoF

    const Replica::Descriptor *best = nullptr;
    for (const auto &r : replicas) {         // 第二遍：只看落盘类副本
        if (r.status != ReplicaStatus::COMPLETE) continue;
        if (r.is_local_disk_replica()) {
            best = &r;  // LOCAL_DISK always overrides DFS and DISK
        } else if (r.is_dfs_replica()) {
            if (!best || !best->is_local_disk_replica()) best = &r;  // 无 LOCAL_DISK 才胜出
        } else if (r.is_disk_replica() && !best) {
            best = &r;   // 最慢，最后才选
        }
    }
    return best;
}
```

**这一段在做什么**

- 第一遍完全不看 DFS，只有 memory/nof 参与。
- DFS 在第二遍出现：LOCAL_DISK > DFS > DISK，与列表顺序无关。
- `!best || !best->is_local_disk_replica()` 让后遇到的 LOCAL_DISK 仍能覆盖已选中的 DFS。
- is_local_disk_replica()（owner 本机盘，走 offload RPC）≠ is_disk_replica()（远端 SSD）。
- 全部扫描后仍无 COMPLETE 副本则返回 nullptr。
- 调用点在 RealClient：real_client.cpp:3767（单 key get_buffer_internal）与 4154（批量）。

**重点语法 / 惯用法**

- 返回裸指针，指向入参 vector 内部元素，调用方不得越生命周期使用。
- `if (const auto *scored = ...)` 是 C++17 if 初始化语句，变量作用域仅限该 if。
- `if (!local_memory)` 用指针判空代替 bool 标记。
- 头文件里的 inline 函数避免多个 TU 重复定义。

**与 6.7.6 的差异**：图把 SelectBestReplica 画成 reader 收到 replicas 后立即执行的抽象步骤；代码里它只在 RealClient（缓冲/分配层）被调用，Client::Get 内部不知道发生过选择，而且 DFS 只有在没有任何 memory/nof/local_disk 副本时才会被选中。

#### 3.1.4 FilterQueryResult：把副本列表裁成一份

`mooncake-store/src/real_client.cpp:490-500`

```cpp
// Build a QueryResult containing only the chosen replica so that
// Client::Get / Client::BatchGet (which internally call
// FindFirstCompleteReplica) cannot pick a different replica type.
inline QueryResult FilterQueryResult(const QueryResult &qr,
                                     const Replica::Descriptor &replica,
                                     bool include_object_checksum = true) {
    return QueryResult(
        {replica}, qr.lease_timeout,     // 只留这一份副本，lease 原样带过去
        include_object_checksum ? qr.object_checksum : std::nullopt,  // 可裁掉校验和
        qr.resolved_tenant_id);          // 租户信息要传给 disk verify
}
```

**这一段在做什么**

- 不裁剪的话 Client::Get 的 FindFirstCompleteReplica 会取"第一个 COMPLETE"，可能不是刚选中的那个。
- 裁剪成单副本等于把选择结果锁死，防止回落成 LOCAL_DISK 或别的类型。
- lease_timeout 必须保留，否则 Client::Get 末尾的 lease 过期检查失效。
- include_object_checksum=false 用于不想再校验一次的路径。
- 调用点在 real_client.cpp:3844（单 key）与各批量分支。

**重点语法 / 惯用法**

- `{replica}` 花括号初始化构造临时 vector<Descriptor>，交给 QueryResult 的 && 形参。
- 三目运算符两分支类型需可转换：optional<uint64_t> 与 nullopt_t。
- 默认实参 true 让多数调用点不必写第三个参数。

**与 6.7.6 的差异**：图里 reader 选完副本直接读；代码里中间多了一次"把 QueryResult 拷贝成单副本"的步骤，图上没有。

#### 3.1.5 RealClient 交给 Client::Get：两层分工的分界线

`mooncake-store/src/real_client.cpp:3828-3852`

```cpp
    // MEMORY / DISK / DFS: use client_->Get. FilterQueryResult ensures
    // Client::Get internal FindFirstCompleteReplica can only see
    // the replica we selected, preventing accidental LOCAL_DISK picks.
    auto runtime_accelerator =
        device::GetAcceleratorRegistry().RuntimeAccelerators();
    if ((replica.is_disk_replica() || replica.is_dfs_replica()) &&
        runtime_accelerator.FindDeviceForPointer(buffer_handle->ptr())) {  // 缓冲区在 GPU 上
        LOG(WARNING) << (replica.is_dfs_replica() ? "DFS" : "DISK")
                     << " replica for key '" << key
                     << "' received a device pointer from the allocator; "
                     << "file I/O cannot write to GPU memory — read will fail. "
                     << "Ensure client_buffer_allocator_ returns host memory.";
    }

    std::vector<Slice> slices;
    allocateSlices(slices, replica, buffer_handle->ptr());  // slice 直接指向 ClientBuffer
    auto filtered_qr = FilterQueryResult(query_result.value(), replica);  // 锁定单副本
    auto get_result = client_->Get(key, filtered_qr, slices);  // 执行层在 Client
    if (!get_result) {
        LOG(ERROR) << "Get failed for key: " << key
                   << " with error: " << toString(get_result.error());
        return nullptr;
    }

    return buffer_handle;
}
```

**这一段在做什么**

- 这就是"选副本在 RealClient、执行在 Client"的分界：选完只传一份副本下去。
- DFS 只接受 host 虚拟地址；设备指针只打 warning 不报错，随后 read 必失败。
- slices 的 ptr 是 ClientBuffer，DFS 后端会把数据直接写进这里，没有 bounce buffer。
- LOCAL_DISK 已在前面分支用 offload RPC 处理，不会走到这里。
- 上游 real_client.cpp:3767 先 SelectBestReplica，失败直接返回 nullptr，不进入 Client。

**重点语法 / 惯用法**

- `buffer_handle->ptr()` 返回可写 void*，交给 allocateSlices 切分。
- 条件里先判副本类型再查设备，短路避免无谓查找。
- filtered_qr 按值持有，生命周期覆盖 client_->Get 调用。

**与 6.7.6 的差异**：图上 reader 直接 R->>BE: ReadDfsReplica；代码里中间隔着 RealClient→Client 的交接、QueryResult 裁剪、slice 切分三步，图上把三层压成了一层。

#### 3.1.6 Client::Get 的分派：DFS 走自己的后端

`mooncake-store/src/client_service.cpp:1382-1406`

```cpp
tl::expected<void, ErrorCode> Client::Get(const std::string& object_key,
                                          const QueryResult& query_result,
                                          std::vector<Slice>& slices) {
    // Find the first complete replica
    Replica::Descriptor replica;
    ErrorCode err = FindFirstCompleteReplica(query_result.replicas, replica);  // 首个 COMPLETE
    if (err != ErrorCode::OK) {
        if (err == ErrorCode::INVALID_REPLICA) {
            LOG(ERROR) << "no_complete_replicas_found key=" << object_key;
        }
        return tl::unexpected(err);
    }

    // Check local hot cache and update replica descriptor if cache hit
    bool cache_used = false;
    if (hot_cache_ && replica.is_memory_replica()) {      // hot cache 只服务内存副本
        cache_used = RedirectToHotCache(object_key, replica);  // 命中则改 descriptor
    }

    auto t0_get = std::chrono::steady_clock::now();
    if (replica.is_dfs_replica()) {                        // DFS 唯一分流点
        err = ReadDfsReplica(object_key, replica, slices);
    } else {
        err = TransferRead(replica, slices);               // 其余走 TE
    }
```

**这一段在做什么**

- 唯一分流点：is_dfs_replica() 为真走 ReadDfsReplica，否则 TransferRead。
- hot cache 只对 memory 副本尝试，DFS 连判断都进不去。
- FindFirstCompleteReplica 取"第一个"而非"最好"，所以上游必须先裁剪成单副本。
- TransferRead 对 DFS 会直接返回 INVALID_REPLICA（client_service.cpp:5037-5040，日志 "DFS reads require object key context"），这层分流就是为绕开它。
- 单 key DFS 读的耗时也计入 transfer_metric.get_latency_us。

**重点语法 / 惯用法**

- Get 有两个重载，单参版本只是先 Query 再调本函数。
- err 变量复用承接两步的结果。
- slices 是非 const 引用，DFS 与 TE 都往里写。

**与 6.7.6 的差异**：图里 reader 收到 replicas 就直接 ReadDfsReplica；实际要先经过 RealClient 的 SelectBestReplica + FilterQueryResult 才进入 Client::Get，而 Client::Get 内部又做一次 FindFirstCompleteReplica。

### 3.2 分发与执行（第 16–17 步）

图上 `R->>BE: ReadDfsReplica → BatchRead` 与 `BE->>FS: BatchReadAt 直接写入调用方 slices`。

#### 3.2.1 ReadDfsReplica：单 key 也复用 BatchRead

`mooncake-store/src/client_service.cpp:5046-5093`

```cpp
ErrorCode Client::ReadDfsReplica(const std::string& key,
                                 const Replica::Descriptor& replica_descriptor,
                                 std::vector<Slice>& slices) {
    if (!replica_descriptor.is_dfs_replica()) {   // 防御：类型不对
        return ErrorCode::INVALID_REPLICA;
    }
    if (!dfs_storage_backend_) {                  // 客户端没初始化 DFS 后端
        LOG(ERROR) << "DFS backend is not initialized";
        if (auto* dfs_metric = GetDfsMetricPtr()) {
            dfs_metric->RecordReadErrors(
                toString(ErrorCode::DFS_SERVICE_UNAVAILABLE));
        }
        return ErrorCode::DFS_SERVICE_UNAVAILABLE;
    }

    auto* dfs_metric = GetDfsMetricPtr();
    const auto& desc = replica_descriptor.get_dfs_descriptor();  // path/offset/size
    std::vector<DfsReadRequest> requests{DfsReadRequest{key, desc, slices}};  // 单元素批量
    const auto read_start = std::chrono::steady_clock::now();
    auto results = dfs_storage_backend_->BatchRead(requests);  // 同步调用，直接写 slices
    const auto read_latency_us = elapsed_us_since(read_start);
    if (results.size() != 1) {                    // 后端必须一一对应返回
        LOG(ERROR) << "DFS BatchRead response size mismatch for key " << key;
        if (dfs_metric) {
            dfs_metric->RecordReadErrors(toString(ErrorCode::INTERNAL_ERROR));
        }
        return ErrorCode::INTERNAL_ERROR;
    }
    if (!results[0]) {                            // 失败：打点后透传错误码
        LOG(ERROR) << "DFS read failed, action=read, key=" << key
                   << ", offset=" << desc.offset
                   << ", size=" << desc.object_size
                   << ", latency_us=" << read_latency_us
                   << ", error=" << results[0].error();
        if (dfs_metric) {
            dfs_metric->RecordReadErrors(toString(results[0].error()));
        }
        return results[0].error();
    }
    if (dfs_metric) {
        dfs_metric->ObserveRead(1, static_cast<int64_t>(desc.object_size),
                                read_latency_us);   // 成功也打点
    }
    VLOG(1) << "DFS read finished, action=read, key=" << key
            << ", offset=" << desc.offset << ", size=" << desc.object_size
            << ", latency_us=" << read_latency_us;
    return ErrorCode::OK;
}
```

**这一段在做什么**

- 单 key 接口内部复用 BatchRead，只传一个元素，避免两套实现。
- 没有本地中转缓冲：slices 直接作为 iovec 交给后端落盘式读取。
- 返回条数必须等于请求数，否则整批 INTERNAL_ERROR。
- 失败 RecordReadErrors、成功 ObserveRead，指标口径对称。
- 全程同步，没有任何 TransferFuture——与 TransferRead 的根本差异。
- slice 容量检查不在这里，交给 BatchRead 的 min() 截断逻辑。

**重点语法 / 惯用法**

- `std::vector<DfsReadRequest> requests{DfsReadRequest{key, desc, slices}};` 聚合初始化，slices 按值拷贝。
- if 初始化语句里的 `auto* dfs_metric` 作用域受限。
- static_cast<int64_t> 避免 uint64_t 到指标类型的窄化告警。

**与 6.7.6 的差异**：图只画 ReadDfsReplica → BatchRead(descriptor,slices)；代码还包含后端未初始化、返回条数校验、失败/成功两套 metric，并且单 key 也走批量接口，图上都没有。

#### 3.2.2 BatchGet 的 DFS 攒批：先提交 TE，再同步批量读

`mooncake-store/src/client_service.cpp:1744-1800`

```cpp
        // Submit transfer operation asynchronously
        std::optional<TransferFuture> future;
        if (replica.is_dfs_replica()) {          // DFS 不进 TE 提交队列
            if (!dfs_storage_backend_) {
                LOG(ERROR) << "DFS backend is not initialized";
                if (auto* dfs_metric = GetDfsMetricPtr()) {
                    dfs_metric->RecordReadErrors(
                        toString(ErrorCode::DFS_SERVICE_UNAVAILABLE));
                }
                results[i] = tl::unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);
                continue;
            }
            const auto& desc = replica.get_dfs_descriptor();
            dfs_read_requests.push_back(
                DfsReadRequest{key, desc, slices_it->second});  // 先攒请求，不读
            dfs_read_indices.push_back(i);                      // 记住批次内下标
            continue;                                          // 跳过 future 提交
// ...（省略 1761-1792：else if (replica.is_nof_replica()) 与 else 分支调用
//      transfer_submitter_->submit(...) 拿 TransferFuture；future 为空时回滚
//      hot cache 并置 TRANSFER_FAIL；成功则 pending_transfers.emplace_back；
//      省略区包含 if/else 的收尾大括号。下一行 } 是 for 循环的收尾）
    }

    if (!dfs_read_requests.empty()) {            // 循环结束后一次性批量读
        auto* dfs_metric = GetDfsMetricPtr();
        const auto dfs_read_start = std::chrono::steady_clock::now();
        auto dfs_results = dfs_storage_backend_->BatchRead(dfs_read_requests);
        const auto dfs_read_latency_us = elapsed_us_since(dfs_read_start);
        int64_t dfs_success_keys = 0;
```

**这一段在做什么**

- 两条流水线：DFS 请求进 dfs_read_requests，内存/NoF 走 transfer_submitter_ 的 future，互不阻塞。
- dfs_read_indices 保存原始下标，因为结果要按 object_keys 顺序填回 results。
- BatchRead 在循环之后同步执行，此时已提交的 TE future 可能仍在飞。
- 批量路径进入前有 `if (!transfer_submitter_)` 前置检查（1672-1680），纯 DFS 批也会被拒——历史耦合，不代表 DFS 依赖 TE。
- 单个 key 的 DFS 失败只影响 results[index]，不打断整批。

**重点语法 / 惯用法**

- std::optional<TransferFuture> 表示"本分支可能不产生 future"。
- 两个平行 vector（requests/indices）靠下标隐式对齐。
- continue 让 DFS 分支跳过后续所有 TE 逻辑。
- DfsReadRequest{key, desc, slices_it->second} 聚合初始化，slice 按值拷贝。

**与 6.7.6 的差异**：图只有单 key 一条线；批量路径是"先提交 TE future、再同步 BatchRead"，两类 I/O 时间上重叠，图上没画，也没画 CheckChecksum 在 DFS 分支内单独调用（1837-1847）。

#### 3.2.3 DistributedStorageBackend::BatchRead 的读契约

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp:776-812`

```cpp
        std::vector<iovec> iovs;                 // 直接指向调用方 slices
        iovs.reserve(request.slices.size());
        size_t remaining = static_cast<size_t>(desc.object_size);  // 还差多少字节
        bool invalid = false;
        for (const auto& slice : request.slices) {
            if (!slice.ptr && slice.size > 0) {  // 空指针但有长度：非法
                invalid = true;
                break;
            }
            if (remaining == 0 || slice.size == 0) {  // 已填满或空 slice：跳过
                continue;
            }
            const size_t read_size = std::min(slice.size, remaining);  // 末段截断
            iovs.push_back({slice.ptr, read_size});
            remaining -= read_size;
        }
        if (invalid || remaining != 0) {         // 容量不足或非法地址
            LOG(WARNING) << "Invalid DFS read request for key " << request.key
                         << ", expected capacity at least=" << desc.object_size;
            results[i] = tl::make_unexpected(ErrorCode::INVALID_PARAMS);
            continue;
        }
        DfsIoOp op{i, request.key, desc, std::move(iovs), desc.object_size};
        if (batch_io) {
            ops.push_back(std::move(op));        // 3FS：攒起来后提交
        } else {
            results[i] = ExecuteSingleIo(op, false);  // POSIX：立即单条读
        }
    }

    if (!batch_io) return results;
    if (bucket_mode) {
        ExecuteBucketBatchIo(ops, results, false);   // 临时 fd，短读补齐
    } else {
        ExecuteShardBatchIo(ops, results, false);    // 缓存 fd，短读报错
    }
    return results;
}
```

**这一段在做什么**

- 读契约：slice 总容量 >= descriptor.object_size；多出来的容量靠 min() 在最后一个 slice 截断，不会多写。
- 剩余字节 remaining != 0 是"容量不足"的唯一判据，直接 INVALID_PARAMS。
- iovec.base 就是调用方 slice.ptr，所以数据落盘即落调用方内存，中间没有 bounce buffer。
- 两条执行路径由 fs_adapter_->SupportsBatchIo() 决定：3FS 走 BatchReadAt，POSIX 逐条 ExecuteSingleIo（722-765 的入口校验未展开）。
- bucket 模式每次开临时 fd 并补齐短读；shard 模式复用缓存 fd 且短读算失败。
- 返回 vector 与 requests 同长同序，按下标对齐。

**重点语法 / 惯用法**

- iovec 聚合初始化 {ptr, len}。
- std::min(slice.size, remaining) 两个 size_t，无符号比较安全。
- std::move(iovs) 把 vector 所有权转移进 op。
- `results[i] = ExecuteSingleIo(op, false)` 直接落单条结果，false 表示读。

**与 6.7.6 的差异**：图写 BE->>FS: BatchReadAt 直接写入调用方 slices 基本正确，但没体现 min() 截断、容量不足判 INVALID_PARAMS、bucket/shard 两套 fd 策略，以及 POSIX 适配器其实是逐条读而非批量。

### 3.3 校验与返回（第 18–19 步）

图上 `R->>R: VerifyObjectChecksum`。这一步默认是空操作。

#### 3.3.1 VerifyObjectChecksum：默认关闭的一步

`mooncake-store/src/client_service.cpp:1349-1372`

```cpp
tl::expected<void, ErrorCode> Client::VerifyObjectChecksum(
    const std::string& object_key, const std::vector<Slice>& slices,
    size_t object_size, std::optional<uint64_t> expected_checksum) {
    if (!object_checksum_enabled_) {         // 默认关闭：直接算通过
        return {};
    }
    if (!expected_checksum.has_value()) {    // master 没带回校验和
        VLOG(1) << "object_checksum_absent key=" << object_key;
        return {};
    }

    auto actual_checksum =
        ComputeObjectChecksumForSlices(object_key, slices, object_size);  // 遍历 slices 重算
    if (!actual_checksum) {
        return tl::unexpected(actual_checksum.error());
    }
    if (*actual_checksum != *expected_checksum) {   // 不一致即数据损坏
        LOG(ERROR) << "object_checksum_mismatch key=" << object_key
                   << " expected=" << *expected_checksum
                   << " actual=" << *actual_checksum;
        return tl::unexpected(ErrorCode::CHECKSUM_MISMATCH);
    }
    return {};
}
```

**这一段在做什么**

- 两个短路：开关关闭或没有期望值，都算校验通过。
- 开关在 Client 构造时由 ClientObjectChecksumConfig::IsEnabledAtFirstUse() 决定（client_service.cpp:453）。
- 校验对象是调用方 slices（DFS 已把数据写进去），不是后端内部缓冲。
- 不一致返回 CHECKSUM_MISMATCH，DFS 路径还会额外 RecordReadErrors。
- 默认关闭意味着 6.7.6 里这一步在多数部署中是空操作。

**重点语法 / 惯用法**

- `return {};` 构造 tl::expected<void, ErrorCode> 的成功分支。
- optional 用 has_value() 判断、`*` 取值。
- ComputeObjectChecksumForSlices 返回 expected<uint64_t>，用 `if (!x)` 判错。

**与 6.7.6 的差异**：图把 VerifyObjectChecksum 画成 get 的固定一步；代码里默认 object_checksum_enabled_=false 直接返回成功，只有部署显式开启且 master 带回 checksum 时才真正计算。

## 4. 与 6.7.6 时序图的差异（精选）

逐段的细节差异写在各小节的末尾。这里只留下会改变理解的 15 条：【改】= 图上画错，【补】= 图缺了关键一步，【补注】= 方向对但需要注明实现层次。

1. **【补注】** `SelectBestReplica` 是「读取方」做的事，方向没错；但它不是 `mooncake::Client` 的成员，而是 `replica_selection.h:122` 的 inline 自由函数，由 **RealClient** 调用（`real_client.cpp:3767`）。`Client::Get/BatchGet` 内部还会再调一次 `FindFirstCompleteReplica`。建议在图注里点明这两层。
2. **【补】** 选完副本后 RealClient 必须用 `FilterQueryResult` 把副本列表裁成只剩一个（`real_client.cpp:493-500`），否则 Client 层会按「列表里第一个 COMPLETE」改选。
3. **【改】** 图上 `M->>A: Allocate` 看起来由 PutStart 直接发出；实际它发生在 `PrepareWriteStart` 校验、租户配额预扣之后，由 `AllocateReplicas` 调用，而且前面还有 MEMORY/NoF 分配与数量校验（`master_service.cpp:5140`、`5253`、`5300`）。
4. **【补】** 描述符字段少画了 `aligned_size`；bucket 模式下 `shard_idx` 装的是 **bucket id** 而非分片序号（`replica.h:264`、`bucket_entry_layout.h:57`）。
5. **【补】** 写 DFS 之前有一道硬门禁：只有 memory/noF 传输全部成功才会写 DFS，否则整段跳过并记 `dfs_writes_skipped`（`client_service.cpp:2048-2084`）。图上没有这条分支。
6. **【补】** DISK 副本：只在 master 配了 `--root_fs_dir` 时存在，且写入是「同步 D2H + 异步落盘、异步 `PutEnd(DISK)`」（`client_service.cpp:2020-2033`、`4647-4692`）。
7. **【补】** GPU 数据在 `WriteDfsReplicas` 里还有一次宿主暂存（pinned pool + `Copy(kDeviceToHost)`），图上没有这一步（`client_service.cpp:2941-2965`）。
8. **【补】** `BE->>FS` 不是一条路径：shard 用 `GetOrOpenShard`（fd 缓存），bucket 用 `OpenBucket`（每次 open/close，每批最多 64 个）；另有 `SupportsBatchIo` 决定逐条执行还是收集后批量提交（`distributed_storage_backend.cpp:652-657`、`541-591`）。
9. **【补】** `BatchWriteAt` 不是一次原子系统调用：基类默认实现是逐请求调 `WriteAt`，POSIX 与 3FS 都没有覆写，所以单个 batch 内部仍是串行的（`fs_adapter.h:144-164`）。
10. **【补】** 3FS 的读并不「直接写入调用方 slices」：先读进每线程注册 iov，再 memcpy 回 slices（`hf3fs_adapter.cpp:257-294`）。这句话只对 POSIX 的 `preadv` 成立。
11. **【补】** 图上的 `M->>M: DFS 副本 → COMPLETE` 之前少了一步：bucket 模式必须先 `MarkCommitted`，失败返回 `FILE_WRITE_FAIL` 且不回滚（`master_service.cpp:5725-5743`）。
12. **【补】** `VerifyObjectChecksum` 默认是空操作（`client_object_checksum_config.h:6`、`client_service.cpp:1352`），只有显式打开且 master 带回 checksum 时才真正校验。
13. **【补】** 读侧只有单 key 一条线：批量走 `BatchGet`，把多个 DFS 请求攒成一次 `BatchRead`，并与已提交的 TE future 重叠执行（`client_service.cpp:1746-1798`）。
14. **【补】** 图里没有失败分支：`Put` 失败会发 `PutRevoke(ALL)`，客户端自己从不 `Free` DFS 区间（`client_service.cpp:2101-2108`；`master_service.cpp:8386`）。
15. **【补】** 「WriteAt 已完成，但不是 fsync」对 DFS 成立；对 DISK 分支则连 `StoreObject` 都可能还没跑完（`client_service.cpp:4647`）。

## 5. 重点语法速查

按小节顺序汇总正文里出现过的 C++ 惯用法与坑（同一条只列一次）。

**2.1**

- tl::expected<void, ErrorCode>：expected 的 void 特化，成功直接返回 {}，失败用 tl::unexpected 包装错误码。
- std::optional<uint64_t>：空值本身就是“未启用”语义，避免用 0 当哨兵。
- emplace_back：原地构造元素；这里把 Slice::size（uint64）窄化存进 size_t 容器。
- 函数内 static const bool：C++11 magic static，首次调用初始化且线程安全，是“一次性读环境变量”的常见写法。
- protocol_ == "cxl"：std::string 的 operator== 是值比较，不是指针比较。
- Slice 结构：{void* ptr; size_t size} 的 POD，多 slice 表示同一对象的离散内存段。
- auto + start_result.error()：tl::expected 的 error() 不消耗对象，可在条件与日志里反复取。
- if 链中调用带副作用的函数：healDanglingLocalDiskReplica 受 && 短路保护，只有前两个条件都成立才执行。
- rbegin()/rend() 反向迭代器：++it 实际向列表头部移动，*it 拿到元素引用。
- return {};：tl::expected<void, E> 的成功写法，等价于构造一个无值成功对象。
- start_result.value() 返回 vector 的引用，range-for 期间生命周期由 start_result 持有。
- VLOG(1)：verbosity 低于 1 时参数不会求值，适合放在可能高频的幂等分支。
- 三目运算符初始化 const auto：一旦推导出具体枚举类型，后面的 RecordFailure/RecordSuccess 就统一用同一变量。
- range-for 遍历 start_result.value()：拿到的是 master 返回 vector 的 const 引用，不产生副本。
- vector<const vector<Slice>*>：指针容器实现零拷贝，代价是调用方必须保证被指对象仍然存活。
- 布尔短路 &&：allocated_dfs_replicas > 0 为假时，NonDfsTransfersSucceeded 不会被调用。
- continue 而非 break：C++ 里这是“跳过本条、继续处理其余副本”的标准写法。
- range-for 直接遍历函数返回值：产生一个临时 vector<ErrorCode>，其生命周期延续到循环结束。
- if/else 覆盖全部取值：每个 dfs_result 必定落到成功或失败之一，不会漏账。
- else if 与前面的 if 互斥：保证“要么写、要么跳过”，不存在两者都执行。
- if (auto* p = f())：C++17 的 init-statement，p 的作用域被限制在这个 if 语句内。
- 结构化日志字段 action= / reason= / key=：便于 grep 和与指标对齐。
- std::chrono::duration_cast<std::chrono::microseconds>(...).count()：显式指定精度再取整数计数。
- std::optional 的 has_value() + operator*：先判空再解引用，不能用 operator* 直接探测。
- ObjectMeta{key, object_checksum}：聚合初始化，第二个成员是 std::optional<uint64_t>，nullopt 合法。
- return tl::unexpected(err)：早返回模式，函数末尾才是唯一的成功出口。
- const auto finalize_decision：用一个具名的纯值对象承载 end_type/revoke_type/success/error 四个决策。

**2.2**

- 尾置返回类型：`auto f(...) -> tl::expected<T, E>`，便于在定义处看到返回类型。
- tl::expected：`tl::make_unexpected(E)` 显式构造失败值；调用方用 `if (!x)` 或 `x.has_value()` 判断。
- std::string_view action：只读的调用方名字（"put_start"/"upsert"），比 const std::string& 更轻，调用点可传字面量。
- enum class ErrorCode：作用域枚举，日志里靠 operator<< 或 static_cast<int> 输出，不会隐式转 int。
- `auto x = f(); if (!x)` 是 tl::expected 的惯用判空写法，等价于 `!x.has_value()`。
- `allocation_result.value()` 返回引用，配合 std::move 实现移动而非拷贝；对 expected 用 `*x` 效果相同。
- assert(!accessor.IsPublished())：只在 debug 生效的契约检查，release 下不产生代码。
- 布尔指针出参（bool*）用于把“DFS 容量耗尽”这一细粒度信号穿出两层调用，调用方需自行判空。
- `bool* dfs_allocation_failed` 允许传 nullptr，函数内先判空再解引用；这是 C++ 里常见的“可选输出参数”写法。
- `!HasExpectedReplicaAllocation(...)` 承接的是普通 bool，不是 expected；两层“失败”语义（业务判定 vs 错误码）在这里混用，读时别混淆。
- size_t 计数器 + 传值给判定函数，避免判定函数直接看 config 猜结果。
- MasterMetricManager::instance() 单例 + inc_* 计数器，用于暴露 put_start 分配失败率。
- tl::expected 的 `operator*` 返回内部值引用，`std::move(*alloc)` 是移动出 descriptor 的惯用法（此处 expected 之后不再使用）。
- emplace_back 原位构造，避免先造 Replica 再移动；参数个数/类型决定重载解析到 DFS 那个构造函数。
- std::variant 在 Replica 内部以 DfsReplicaData 形式存放 descriptor，构造后类型即确定。
- enum class ReplicaStatus 必须写全限定名 `ReplicaStatus::PROCESSING`，不会隐式转 int。
- `std::vector<Replica>&&` 右值引用形参：签名即契约，表示“我拿所有权”。
- `Replica::Descriptor` 是嵌套类型，返回值写成 vector 时要比 vector<Replica> 更注意它是按值拷贝的轻量结构。
- `int i = 0;` 配合后面 `++i` 做前置自增。
- `std::optional<std::chrono::system_clock::time_point>` 用 `if (deadline_to_index)` 判有值，后文再 `*` 取值。
- `const auto desc = replica.get_descriptor();` 按值接收，因为函数返回 Descriptor 而非引用。
- Descriptor 内部是 std::variant，`get_dfs_descriptor()` 用 std::get 取 DfsReplicaData，类型不符会抛 std::bad_variant_access（因此前面必须先判 is_dfs_replica）。
- `replica.is_dfs_replica()` 与静态版本 `Replica::fn_is_dfs_replica` 成对存在，后者用于 std::find_if 之类的谓词。
- `VLOG(1)` 只在 --v=1 以上输出，热路径日志用 INFO 级别的 LOG 会明显拖慢。
- `accessor.Get()` 返回元数据引用，之后的 Publish/PopReplicas 都作用在同一个对象上。
- `std::move(eligible_replica_ids)` 再次体现“移动即让渡”，配合形参 `std::unordered_set<ReplicaID>` 或 const& 由被调方决定。
- `accessor.it_` 是 shard 内 map 的迭代器，erase 后必须重置为 end() 才算失效安全。
- `return replica_list;` 触发 NRVO/移动，不会深拷贝整个 vector。
- std::function<...> 作为形参把 lambda（PutStart 里的 admit）擦除成可多次调用的对象；每次调用都要传 bool*。
- `std::lock_guard recovery_lock(mutex)` 是 C++17 类模板实参推导（CTAD），不用写 std::lock_guard<std::mutex>。
- `!result` 直接作用于 tl::expected，等价于 `!result.has_value()`；`result.error()` 在这里没被读取。
- 递归式的 `result = admit(...)` 复用同一个 result 变量，最终只返回最后一次结果。

**2.3**

- std::vector<ErrorCode>(n, value)：值填充构造，一次分配 n 个相同初值，比 push_back 循环少一次扩容。
- if (auto* p = f(); cond)：C++17 if-init，p 只在 if 语句（含 else）内可见，避免污染外层命名空间。
- static_cast<int64_t>(keys.size())：size_t 是无符号，直接传给 int64 参数会触发窄化告警，显式转换表意。
- toString(ErrorCode)：把枚举拼成指标 label 字符串，故意不把 types.h 拉进 metric 头文件。
- std::chrono::steady_clock：单调时钟，语义上比 system_clock 更适合测耗时，不会被 NTP 调整影响。
- device::GetAcceleratorRegistry().RuntimeAccelerators()：返回按值持有的代理对象，不是裸指针，退出作用域自动解绑。
- std::vector<PinnedBufferPool::Buffer>：Buffer 拷贝构造被 delete、只提供 noexcept 移动，所以容器只能靠 std::move 搬运。
- LOG(ERROR) << ...：glog 流式宏，作用域结束时析构才真正落盘。
- device::PointerInfo info{}：花括号值初始化，把所有成员初始化到 0/-1，避免未初始化 device_id 被 SetContext 误用。
- 条件表达式返回指针：两个分支都是指针类型，编译器不会插入隐式转换，nullptr 分支类型推导为 const AcceleratorDevice*。
- host_slices.push_back(slice)：Slice 是平凡可拷贝的 POD（ptr+size），按值塞进去等于“记下同一块内存”。
- PinnedBufferPool::Buffer 的拷贝构造/赋值被 delete，只有 noexcept 移动构造；push_back(std::move(buffer)) 之后 buffer 处于“已搬空”状态（data 被置 nullptr）。
- emplace_back(Slice{...})：先构造临时 Slice 再移动进容器，效果等同 push_back，写成 emplace 只是省一次显式类型名重复。
- DfsWriteRequest{key, desc, std::move(v)}：聚合初始化 + 移动，把 host_slices 的 vector 缓冲直接过继给请求。
- device::CopyDirection::kDeviceToHost：枚举限定名，防止与 H2D 方向混用。
- elapsed_us_since(...)：工具函数返回 uint64_t 微秒数，内部做 duration_cast，避免每个调用点重写一行。
- tl::expected<void, ErrorCode> 的布尔转换：write_results[i] 为真表示无错误，取 .error() 前必须先判真（否则是 UB）。
- for (size_t index : request_indices)：范围 for 遍历 size_t 容器，避免手写下标。
- auto write_results = f(...)：返回 vector<expected> 的按值复制（NRVO/移动），不要写 auto&& 绑到临时。
- dfs_metric && !request_indices.empty()：短路求值，避免空批时给计数器 inc(0)。
- VLOG(1)：glog 的 verbose 级别日志，默认不输出，需 --v=1 才可见，用于不打扰生产的排障。
- for (auto& buffer : staging_buffers) + std::move(buffer)：取引用再搬走其内容，容器本身在函数末尾析构，不会重复释放。
- Release(Buffer buf) 按值收参：调用方必须显式 std::move，否则编译失败（拷贝被 delete），这是一道防误用的类型护栏。

**2.4**

- tl::expected<void, ErrorCode>：value 类型是 void，默认构造即代表"无错误"。
- vector::assign(n, value)：把 vector 重置成 n 份同一错误值，比 clear+push 更直接。
- tl::make_unexpected(e)：构造 expected 的 error 分支，不能把 ErrorCode 直接赋给 expected。
- 按值返回 vector 依赖 NRVO/移动语义，不会深拷贝 results。
- const auto& request = requests[i]：避免拷贝 DfsWriteRequest（内含 std::string 与 vector<Slice>）。
- 条件运算符用于初始化 const bool range_valid，两个分支都是函数调用，靠共同的 bool 返回类型收敛。
- UsesBucketAllocator() 是 const 成员函数，内部只比较 allocator_type == "bucket"，无锁无状态。
- if (batch_io) ops.reserve(...)：条件预留，避免非批量路径分配一块永远用不上的内存。
- 聚合初始化 iovs.push_back({slice.ptr, slice.size})：iovec 是 POD 聚合体，可直接花括号构造。
- 无符号溢出不会 UB 而是回绕，所以必须显式做 max()-total_size 预判，否则 total_size 会绕过相等性检查。
- std::move(op) 转移 vector<iovec> 的所有权；DfsIoOp 因含引用成员而不可拷贝赋值，只能移动/构造。
- if (!batch_io) return results; 属于 early-return 惯用法，使后段代码不必再判一次开关。
- static_cast<size_t>(std::numeric_limits<int>::max())：把 int 上限转成 size_t 再比，规避有符号/无符号比较告警。
- struct DistributedStorageBackend::DfsIoOp 是嵌套类型的类外定义：头文件只前置声明 `struct DfsIoOp;`，定义藏在 .cpp 里。
- const std::string& / const DistributedFSDescriptor& 作为成员：使类型不可默认构造、不可拷贝赋值，只能聚合初始化。
- std::lock_guard lock((*shard)->mutex)：CTAD 推导模板参数，额外括号避免 most vexing parse。
- std::move(op.iovs) 传值给 TransferAll：补写需要就地修改 iov_base/iov_len，所以必须转移所有权。
- 聚合初始化 {i, request.key, desc, std::move(iovs), total_size} 把引用与所有权一次绑好。
- 对 0 取模是未定义行为（整数除零），所以 `config.alignment == 0` 必须排在 `%` 之前。
- 无符号减法下溢：offset > cap 时 cap - offset 会变成一个巨大值，所以前一个比较不能省。
- std::numeric_limits<int64_t>::max() 配合 static_cast<uint64_t>：把有符号上界转成无符号常量再参与比较。
- constexpr 局部常量：编译期求值，替代魔数并保留类型。
- 短路求值 `a && b`：后一个条件依赖前一个成立时才安全（差值比较依赖 offset 已被约束）。

**2.5**

- std::string_view：只读视图，remove_prefix/remove_suffix 改变的是视图的起点和长度，底层字符串不变。
- constexpr std::string_view：编译期常量，函数里用 prefix.size() 是编译期已知量。
- starts_with / ends_with：C++20 新增，比 substr 再比较更省一次构造。
- std::optional<int>：表达"可能没有值"，调用方必须写 if (!index) 再 *index。
- 溢出保护写法 index > (INT_MAX - d) / 10：先除后比，避免 index * 10 + d 本身溢出（有符号溢出是 UB）。
- std::lock_guard 的 CTAD：C++17 起可省模板参数，std::lock_guard cache_lock(m) 自动推导。
- lexically_normal()：纯词法规范化，不会因为文件不存在而失败（对比 canonical 会访问文件系统）。
- path::filename() 对以分隔符结尾的路径返回空串，所以用 root.filename().empty() 来剥掉尾部分隔符。
- path.filename().string() 返回临时 std::string，作为 string_view 形参的生命周期延续到整个完整表达式结束，函数内使用是安全的。
- tl::expected 用 tl::make_unexpected 构造错误态：错误是返回值的一部分，不抛异常，也不产生歧义。
- std::unique_ptr<ShardFile>：ShardFile 内含 std::mutex，天然不可拷贝不可移动，只能堆分配后由容器持有。
- .emplace(k, v).first：unordered_map::emplace 返回 pair<iterator,bool>，这里只取迭代器，忽略"是否新建"。
- get() 取裸指针：锁外使用不会延长生命周期，但容器节点在 backend 析构前不会失效。
- lock_guard 在 return 时析构解锁，所以函数提前 return 也是异常安全的。
- std::move(file_path) 移进成员：省一次字符串拷贝，之后 file_path 不应再被使用。
- canonical(p, ec) 重载：使用 error_code 版本不抛异常，失败时置 ec 并返回空 path。
- ec 在两次 canonical 间复用，第二次会覆盖第一次，但每次都在使用前立刻检查，所以安全。
- canonical 要求路径存在，因此它同时起到了"再确认一次文件在"的作用。
- GetFileSize 返回 std::expected<size_t,...>，与 uint64_t 的 shard_capacity 直接比较，无窄化转换。
- 函数返回 tl::expected<int, ErrorCode>：fd 的 -1 不表示错误，错误只走 expected 的 error 分支。
- setw 只保证最小宽度，数字更长时照常输出，因此宽度约束不能写死在解析器里。
- string_view 的 remove_prefix/remove_suffix 仍然零拷贝，与分片版本写法对称，便于对照阅读。
- 两个函数几乎逐行重复：这是刻意保留的两种命名空间，合并反而会掩盖语义差异。
- std::optional<int> 的 *index 在 if 短路里已经保证有值（!index 先判）。
- OpenExistingFile 与 OpenFile 的差异属于接口契约而非参数：posix 版用 O_RDWR|O_CLOEXEC 且对 EINTR 重试；3fs 版还额外把 ENOENT 映射成 FILE_NOT_FOUND。
- const std::string file_path = path.string()：path.string() 返回临时对象，要跨多条语句使用就必须落到命名变量里。
- std::error_code error：与 shard 版的 ec 写法相同，名字不同但语义一致。
- 错误枚举分层：路径不存在 → FILE_NOT_FOUND（来自 GetFileSize），存在但类型/大小不对 → INVALID_PARAMS，I/O 打不开 → FILE_OPEN_FAIL。

**2.6**

- tl::expected<void, ErrorCode>：成功时无返回值，失败用 tl::make_unexpected(ErrorCode) 承载错误码。
- static_cast<char*>(iov_base) + consumed：void* 不能做指针算术，必须先转成 char*。
- std::numeric_limits<int>::max()：readv/writev 的 iovcnt 是 int，超过会截断，这里提前拦住。
- iovec 是 C 结构体 {void* iov_base; size_t iov_len;}，std::vector<iovec> 连续存储，可传 .data()。
- return {};：值初始化一个“成功的 expected<void>”，比 return tl::expected<void,ErrorCode>{} 简洁。
- auto fd = OpenBucket(...) 返回 tl::expected<int, ErrorCode>，*fd 才是真正的 fd 值。
- std::move(op.iovs) 传入按值形参，避免整份 iovec 数组拷贝。
- static_cast<int64_t>(op.descriptor.offset)：descriptor.offset 是 uint64_t，pread/pwrite 需要 off_t 语义的 int64_t。
- 条件运算符 write ? A : B 要求两分支类型一致，这里都是 tl::expected<size_t, ErrorCode>。
- **shard：shard 是 tl::expected<ShardFile*, ErrorCode>，第一个 * 取指针，第二个 * 解引用到 ShardFile 对象。
- auto reopen = ...; if (reopen) 依赖 expected 的 explicit operator bool，失败时进 else 分支。
- result.error() 需要先取出或直接拷贝，因为 result 随后可能被第二次 I/O 覆盖。
- (void)fs_adapter_->CloseFile(...)（ReopenShardLocked 内部）：显式丢弃返回值，表明“旧 fd 关不掉也无所谓”。
- 聚合初始化 {fd, iov, iovcnt, offset} 直接构造 FdIoRequest，字段顺序必须与结构体声明一致。
- std::vector<DfsIoOp*> 只存指针，避免拷贝含 const std::string& 引用成员的 DfsIoOp。
- results 是 vector<tl::expected<void, ErrorCode>>，默认构造即“成功”，只有失败才赋值。
- const auto& 结构化绑定、reserve 与 size_t 下标混用是这批代码的常规写法。
- std::map<std::pair<std::string,int>, ...> 的 operator[] 需要键可比较，std::pair 自带字典序。
- for 的第 1 和第 3 部分被拆开写，next 在循环体间持续推进，外层 while 负责多轮。
- opened.size() < kMaxOpenBucketsPerBatch：size_t 与 static constexpr size_t 比较，无符号一致。
- bucket_ops.front()->descriptor：组内任取一条即可，因为分组键保证了 file_path 与 shard_idx 相同。
- 结构化绑定 for (const auto& [fd, bucket_ops] : opened) 解出 int 与指针，bucket_ops 是指针需再 * 才能遍历。
- auto& result = results[op->index]; 必须取引用，否则改的是副本。
- if (result) 判断 expected 是否持有值（成功态）；失败态才能调 error()。
- std::span<const FdIoRequest> 形参：io_requests 是 vector，可隐式转换成 span，无需拷贝。
- std::move(op.iovs) 只在续传分支发生；成功分支保持 op.iovs 原样，不能无条件 move。
- explicit operator bool：if (!result) 判断 expected 失败，需先确认非空才能取 error()。
- else if 链最后的 else 同时兜住两种情况：短传输但不允许续传、以及返回 0 字节。

**2.7**

- tl::expected<void, E>：无值成功用 return {} 表示，失败用 tl::make_unexpected(...) 构造。
- 匿名 namespace 包裹：这两个符号仅本翻译单元可见，等价于内部链接。
- 三元运算符直接返回 ErrorCode 枚举值，未加 static_cast 是因为同类型。
- errno 是线程局部的全局变量，任何库调用都可能覆盖它，读取窗口要尽量短。
- ::open 加全局作用域符，避免成员函数或命名空间里的同名解析。
- open 的第三个参数 0644 只在带 O_CREAT 时生效，但两个调用都写了，缺少时是未定义行为。
- fd 用 int 装在 tl::expected<int, ErrorCode> 中，成功值就是裸 fd，不做 RAII 包装（由调用方 CloseFile）。
- do-while(ret<0 && errno==EINTR) 是 POSIX 系统调用的标准重试惯用法。
- ssize_t 与 size_t 之间必须显式 static_cast，直接隐式转换会产生 -Wsign-conversion 告警。
- static_cast<off_t>(offset) 把协议里的 int64_t 转成平台 off_t，32 位平台上会截断（这是隐含前提）。
- size_t{0} 用花括号初始化，确保返回的是"成功值 0"而不是被 expected 当成失败构造。
- if (!valid) 利用了 tl::expected 的 explicit operator bool，失败时取 valid.error() 传递错误码。
- preadv 的第二个参数是非 const 的 iovec*，因为内核要往里写；const 版本只存在于 write 方向。
- 同一个 ValidateIov 同时接受 const iovec* 与 iovec*，因为 iovec* 可隐式转 const iovec*。
- static_cast<size_t>(ret) 在 ret>=0 分支内执行，负值已被上面的 if 拦掉，不会下溢成巨大数值。
- if (hf3fs_reg_fd(fd, 0) > 0) 直接以 >0 判败，是 C 风格错误码与 tl::expected 混用时的典型适配。
- 错误码三元表达式跨行书写，clang-format 把 ? 与 : 对齐在操作数列。
- hf3fs_dereg_fd(fd); 忽略返回值要加 [[maybe_unused]] 或直接不接收，否则触发 -Wunused-result（此处该函数无该属性）。
- 类内声明 SupportsBatchIo()=true（include/storage/distributed/hf3fs_adapter.h:43），OpenFile 系列则完全没有声明 SupportsFdReopen，即继承基类默认 false。
- auto& threefs_iov = resource->iov_; 绑定引用，避免反复解引用指针。
- reinterpret_cast<char*>(threefs_iov.base) 把 void* 转可做指针运算的 char*。
- static_cast<char*>(iov[i].iov_base) + iov_off：先转 char* 再做字节偏移，void* 算术非法。
- std::min(remaining, resource->config_.iov_size)：两参类型不同（size_t 与 size_t）此处一致，换类型时要显式转换。
- 外层 while 的游标语义类似手工实现的状态机，读的时候要同时盯 remaining/iov_idx/iov_off 三个量。
- struct hf3fs_cqe cqe; 是纯 C 结构体，用 struct 前缀声明（C++ 中可省，但沿用 C API 风格）。
- cqe.result 类型为 ssize_t/int，赋给 size_t bytes_written 前已经由上面 if 保证非负。
- 多分支 break 共用一个出口，靠最后的 total_written != total_length 汇总失败，避免在循环内生成多条错误路径。
- hf3fs_wait_for_ios 末参 nullptr 是超时参数，表示使用 ior 默认行为。
- prep_io 第三个参数 true 表示读，语义与 WriteAt 的 false 相对。
- 两个 while 共用 iov_idx/iov_off，读循环与写循环的游标推进方式完全对称。
- src 与 dest 命名区分"iov 内地址"与"调用方地址"，读时 src 是 iov.base、目标是 slices。
- memcpy 的参数顺序（目的, 源）在写/读两段正好互换，是 review 时最容易看错的地方。
- virtual ... = 0 与带默认实现的虚函数混用：纯虚强制覆写，非纯虚给可选项，两者都在同一个接口类里。
- 参数写成注释形式 /*fd*/ 消除未使用参数告警，是接口默认实现的常用写法。
- const 成员函数返回 bool，配合后端 bool batch_io = fs_adapter_->SupportsBatchIo(); 做运行期分派。
- 同一族能力开关放在基类注释里说明"什么时候该保持 false"，比分散在子类更利于维护。

**2.8**

- 尾置返回类型 `-> tl::expected<void, ErrorCode>`：用 tl::expected 表达“可能失败的无返回值 RPC”，错误用 `tl::make_unexpected` 包装。
- `std::shared_lock<std::shared_mutex>`：RAII 共享锁，构造加锁、析构解锁。
- `MetadataAccessorRW`：把“查表 + 加锁 + 校验”封装成一个对象，`Get()` 返回可写引用。
- 带捕获的 lambda 随后被隐式转成 `std::function<bool(const Replica&)>` 传给 `VisitReplicas`；按值捕获枚举是安全的。
- `&&` 优先级高于 `||`，这里 `a && b || c && d || e` 不需要额外括号，括号只用于可读性。
- `ReplicaType::ALL` 是枚举哨兵值，不是 `Replica::type()` 可能返回的值，所以必须单独分支。
- `metadata.GetAllReplicas()` 返回 `std::span<const Replica>`，是只读视图，遍历不拷贝。
- `return {};` 构造 `tl::expected<void, E>` 的“有值但无内容”成功返回，等价于 `return tl::expected<void,E>{}`。
- `break` 在发现第一个未完成副本后立即跳出，扫描是短路语义。
- 两个 lambda 分别以 `const Replica&` 和 `const Replica&` 传入 `VisitReplicas` 的 const 重载，编译器按 const 性选重载。
- `std::vector<DistributedFSDescriptor>` 按值存 descriptor，因为 allocator 的接口只接受 const 引用，不需要持有副本地址。
- `descriptor.shard_idx` 被当日志里的 bucket_id 打印，字段名与语义不完全对应，读日志时注意。
- `&Replica::fn_is_completed` 是静态成员函数指针，可直接被 `std::function<bool(const Replica&)>` 接住，比写 lambda 更短。
- 访问器 lambda 捕获 `this` 与多个引用，是可写版本 `std::function<void(Replica&)>`。
- `const auto& desc = replica.get_dfs_descriptor();` 绑定到返回的 const 引用，避免复制 descriptor。
- `mark_complete()` 内部只在 PROCESSING 时迁移状态，对 COMPLETE 调用不会回退。
- `HasReplica` 接收无捕获 lambda，空捕获列表 `[]` 可转成函数指针，再转 `std::function`。
- `metadata.MutablePart(0)` 返回可写 part，链式调用 `SetChecksum`。
- `if (...) { ... }` 的最后一个条件是带 lambda 的 `HasReplica(...)`，`!` 作用于整个调用结果，括号位置容易读错。
- `&Replica::fn_is_completed` 作为 `AllReplicas` 的谓词参数，是静态成员函数指针的常规用法。
- `std::chrono::milliseconds::zero()` 显式构造零时长，语义由 GrantReadLease 解释。
- `static_cast<int>(result.error())`：枚举无法直接进日志流，必须先转成整数。
- `Enable_oplog_ && ordered_oplog_writer_`：双重开关，oplog 未启用或写入器未就绪时整段跳过。
- `GetFirstReplica` 返回裸指针 `Replica*`，用 `!= nullptr` 判存在，而不是 optional。
- lambda 的返回值类型由多个 `return` 推导；所有分支都返回 bool，所以无需显式 `-> bool`。
- `replica_type == ReplicaType::ALL` 作为编译期不可知的运行期分支，两个 lambda 各写一遍，容易漏改其中一处。
- `before_charge` / `after_charge` 是 `uint64_t` 快照，比较用 `>` 而不是 `!=`，防御计数异常。
- `metadata.quota_ledger.ReleaseCommitted(...)` 返回 expected，失败转成 INTERNAL_ERROR 上抛。
- `return {};` 在这里表示“对象已彻底删除”也是一种成功。

**3.1**

- tl::expected：std::expected 的 C++17 回填，错误分支用 tl::unexpected 构造。
- tl::in_place：in-place 构造 tag，省一次中间对象拷贝。
- std::move 把 replicas 从返回值中搬进 QueryResult。
- 尾置返回类型 `-> tl::expected<...>`。
- shared_lock 是 RAII：构造加共享锁、析构自动释放。
- lambda 作为谓词传给 AllReplicas，捕获列表为空。
- tl::make_unexpected 直接构造错误分支。
- 返回裸指针，指向入参 vector 内部元素，调用方不得越生命周期使用。
- `if (const auto *scored = ...)` 是 C++17 if 初始化语句，变量作用域仅限该 if。
- `if (!local_memory)` 用指针判空代替 bool 标记。
- 头文件里的 inline 函数避免多个 TU 重复定义。
- `{replica}` 花括号初始化构造临时 vector<Descriptor>，交给 QueryResult 的 && 形参。
- 三目运算符两分支类型需可转换：optional<uint64_t> 与 nullopt_t。
- 默认实参 true 让多数调用点不必写第三个参数。
- `buffer_handle->ptr()` 返回可写 void*，交给 allocateSlices 切分。
- 条件里先判副本类型再查设备，短路避免无谓查找。
- filtered_qr 按值持有，生命周期覆盖 client_->Get 调用。
- Get 有两个重载，单参版本只是先 Query 再调本函数。
- err 变量复用承接两步的结果。
- slices 是非 const 引用，DFS 与 TE 都往里写。

**3.2**

- `std::vector<DfsReadRequest> requests{DfsReadRequest{key, desc, slices}};` 聚合初始化，slices 按值拷贝。
- if 初始化语句里的 `auto* dfs_metric` 作用域受限。
- static_cast<int64_t> 避免 uint64_t 到指标类型的窄化告警。
- std::optional<TransferFuture> 表示"本分支可能不产生 future"。
- 两个平行 vector（requests/indices）靠下标隐式对齐。
- continue 让 DFS 分支跳过后续所有 TE 逻辑。
- DfsReadRequest{key, desc, slices_it->second} 聚合初始化，slice 按值拷贝。
- iovec 聚合初始化 {ptr, len}。
- std::min(slice.size, remaining) 两个 size_t，无符号比较安全。
- std::move(iovs) 把 vector 所有权转移进 op。
- `results[i] = ExecuteSingleIo(op, false)` 直接落单条结果，false 表示读。

**3.3**

- `return {};` 构造 tl::expected<void, ErrorCode> 的成功分支。
- optional 用 has_value() 判断、`*` 取值。
- ComputeObjectChecksumForSlices 返回 expected<uint64_t>，用 `if (!x)` 判错。
