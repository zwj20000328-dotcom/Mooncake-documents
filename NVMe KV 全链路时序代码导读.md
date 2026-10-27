# NVMe KV 全链路时序代码导读

> **对应图**：`FileStorage-offload-实现解读.md` 的 6.7.3「NVMe KV 全链路时序」（24 步版本）。
> **代码基线**：`main` @ `67df8c41`，所有位置写成 `路径:起止行`。
> **代码约定**：正文代码块是**源码原文**，只做了两件事——在行尾/行首加 `//` 中文注释，以及用 `// ...（略 …）` 跳过与本步无关的整段。除这两类改动外没有改写任何代码，你可以直接按行号对照仓库。

## 0. 怎么读这份导读

每一节的结构固定：

```
位置（路径:起止行）
  └─ 代码（带行内注释）
  └─ 这一段在做什么     ← 读完应该记住的判断
  └─ 重点语法 / 惯用法   ← 这段代码用到的 C++ 写法与坑
  └─ 与 6.7.3 的差异     ← 图上看不到、但代码里存在的东西
```

建议顺着 2 → 3 → 4 读一遍（写入前半 → owner 落盘 → 读路径），再回看第 5 节（命令层）与第 6 节（差异汇总）。如果只想回答某一个问题：

| 想回答的问题 | 直接看 |
| --- | --- |
| 为什么 `PutStart` 只回 MEMORY / NoF，不回 `LOCAL_DISK` | 2.3 |
| `Put` 已经返回了，为什么盘上还没有数据 | 2.2 / 2.4 |
| 同一份数据为什么会被写进多个 client 的邮箱，但只钉住一个副本 | 2.5 |
| `BatchQuerySegmentSlices` 到底是不是零拷贝 | 3.2 / 3.3 |
| 大对象为什么"chunk 先写、root 后写"，失败时怎么清 | 3.5 / 3.6 |
| 落盘成功了，`LOCAL_DISK` 为什么有时不新增副本 | 3.7 |
| 读路径什么时候是设备直读、什么时候要 memcpy | 4.5 / 5.1 |
| 谁是 owner、TE 地址从哪来、buffer 什么时候释放 | 4.2 / 4.6 |

## 1. 图上每一步落到哪个函数

| 图上步骤 | 真实入口 | 位置 | 本文 |
| --- | --- | --- | --- |
| 1 `App->>W: put` | `RealClient::put` → `put_internal` | `real_client.cpp:2543` / `2483` | 2.1 |
| 2 `W->>M: PutStart` | `Client::Put` → `MasterService::PutStart` | `client_service.cpp:1988` / `master_service.cpp:5589` | 2.2 / 2.3 |
| 3 `M-->>W: MEMORY / NoF_SSD` | `AllocateReplicas` + `InsertMetadata` | `master_service.cpp:5140` / `5317` | 2.3 |
| 4 `W->>M: PutEnd(ALL)` | `Client::Put` 收尾 → `MasterService::PutEnd` | `client_service.cpp:2088` / `master_service.cpp:5663` | 2.2 / 2.4 |
| 5 `M->>M: 入队 + inc_refcnt` | `PutEnd` 的 `VisitReplicas` → `PushOffloadingQueue` | `master_service.cpp:5785` / `9161` | 2.4 / 2.5 |
| 6 `W-->>App: Put 返回` | `DetermineFinalizeDecision` → `PutEnd` 返回 | `client_service.cpp:2088-2114` | 2.2 |
| 7 `FS->>M: OffloadObjectHeartbeat` | `FileStorage::Heartbeat` → `Client::OffloadObjectHeartbeat` | `file_storage.cpp:662` / `client_service.cpp:4200` | 3.1 |
| 8 `M-->>FS: OffloadTaskItem[]` | `MasterService::OffloadObjectHeartbeat` | `master_service.cpp:8887` | 3.1 |
| 9 `FS->>M: BatchQuery` | `BatchQuerySegmentSlices` → `Client::BatchQuery` | `file_storage.cpp:1047` / `client_service.cpp:1259` | 3.2 |
| 10 `M-->>FS: replicas[]` | `MasterService::BatchGetReplicaList` | `master_service.cpp:4905` | 3.2 |
| 11 `FS->>FS: 取本机地址` | `Client::IsReplicaOnLocalMemory` | `client_service.cpp:5652` | 3.2 |
| 12 `FS->>BE: BatchOffload` | `FileStorage::OffloadObjects` → `NvmeKvStorageBackend::BatchOffload` | `file_storage.cpp:501` / `backend.cpp:479` | 3.3 / 3.4 |
| 13 `BE->>Dev: StoreBatch` | `store_chunks` / `OffloadOne` → executor `StoreBatch` | `backend.cpp:598` / `976` | 3.5 / 3.6 / 5.2 |
| 14 `FS->>M: NotifyOffloadSuccess` | `complete_handler` → `Client::NotifyOffloadSuccess` | `file_storage.cpp:342` / `client_service.cpp:4261` | 3.3 / 3.7 |
| 15 `M->>M: dec_refcnt / 登记` | `MasterService::NotifyOffloadSuccess` | `master_service.cpp:8973` | 3.7 |
| 16 `R->>M: BatchGetReplicaList` | `RealClient` 读入口 → `Client::BatchQuery` | `real_client.cpp:4110` / `client_service.cpp:1259` | 4.1 |
| 17 `M-->>R: replicas + lease_ttl_ms` | `GetReadableReplicaDescriptors` + `GrantReadLease` | `master_service.cpp:4969` / `5006` | 4.1 |
| 18 `R->>FS: batch_get_offload_object` | `restore_offload_objects` → `ClientRequester::batch_get_offload_object` → `RealClient::batch_get_offload_object` | `real_client.cpp:8893` / `9067` / `8694` | 4.2 |
| 19 `FS->>BE: BatchLoad` | `BatchGet` → `LoadBatch` → `AllocateBatch` → `backend->BatchLoad` | `file_storage.cpp:267` / `231` / `1092` / `backend.cpp:1167` | 4.3 |
| 20 `BE->>Dev: root + chunk` | `ResolveRoot` + `read_group` | `backend.cpp:1198` / `1336` | 4.4 / 4.5 |
| 21 `Dev-->>FS: 直写 / memcpy` | `RetrieveIntoBatch` / `RetrieveBufferBatch` | `backend.cpp:1360-1376` | 4.5 |
| 22 `FS-->>R: {batch_id, pointers[], ...}` | `BatchGetOffloadObjectResponse` | `real_client.cpp:8718` | 4.6 |
| 23 `R->>FS: READ` | `TransferSubmitter::submit_batch_get_offload_object` | `transfer_task.cpp:1614` | 4.6 |
| 24 `R->>FS: release_offload_buffer` | `OffloadRestoreLease::~OffloadRestoreLease` → `FileStorage::ReleaseBuffer` | `real_client.cpp:600` / `file_storage.cpp:1202` | 4.6 |

两条贯穿全文的约定：

- **"W / R" 都可能是同一个进程**。图上把写入方 `W` 和请求方 `R` 画成两个 RealClient，但代码里它们只是两次 `put` / `get` 调用；owner `FS` 也是一个 RealClient 进程内的 `FileStorage`。
- **`storage_key` 与 `user_key` 不同**。`FileStorage` 交给后端的是 `TenantId::MakeScopedKey()` 之后的字符串（`tenant:key`），物理 key 由这个字符串派生；master RPC 用的是 `(tenant_id, user_key)` 二元组。读代码时不要混用。

## 2. 写入前半：`put` → master 入队（第 1–6 步）

### 2.1 RealClient::put → Client::Put：数据先落到注册 buffer（第 1 步）

mooncake-store/src/real_client.cpp:2483-2520

```cpp
tl::expected<void, ErrorCode> RealClient::put_internal(   // 真正的 put 实现：成功返回 void，失败带 ErrorCode
    const std::string &key, std::span<const char> value,  // value 是 std::span：非拥有视图，既可能是 CPU 指针也可能是 GPU 指针
    const ReplicateConfig &config,                        // 副本策略，原样透传给 Client::Put
    const std::shared_ptr<ClientBufferAllocator> &client_buffer_allocator) {  // 用哪个注册 buffer 池：成员池，或某个 shm context 的池
    if (config.prefer_alloc_in_same_node) {               // 这个开关在当前客户端 put 路径上没有实现
        LOG(ERROR) << "prefer_alloc_in_same_node is not supported.";
        return tl::unexpected(ErrorCode::INVALID_PARAMS);  // 直接拒绝，而不是悄悄忽略这个配置
    }
    if (!client_) {                                        // Client 由 RealClient 持有，未 connect 成功时为空
        LOG(ERROR) << "Client is not initialized";
        return tl::unexpected(ErrorCode::INVALID_PARAMS);
    }
    if (!client_buffer_allocator) {                        // 没有注册 buffer 池就没法让 TE 写，提前失败
        LOG(ERROR) << "Client buffer allocator is not provided";
        return tl::unexpected(ErrorCode::INVALID_PARAMS);
    }
    auto alloc_result = client_buffer_allocator->allocate(value.size_bytes());  // 从池里切一块 size 字节，返回 std::optional<BufferHandle>
    if (!alloc_result) {                                   // optional 的 operator!：空即分配失败（池耗尽或超过单块上限）
        LOG(ERROR) << "Failed to allocate buffer for put operation, key: "
                   << key << ", value size: " << value.size();  // 上面用 size_bytes()（字节），这里用 size()（元素数），char 下相等
        return tl::unexpected(ErrorCode::INVALID_PARAMS);  // 坑：池耗尽的错误码是 INVALID_PARAMS，Python 侧看起来像"参数不对"
    }
    auto &buffer_handle = *alloc_result;                   // 绑定到 optional 内部对象；BufferHandle 是 move-only，这里只取引用不拷贝
    auto scatter_result = gather_maybe_device_to_host(     // 名字叫 scatter，实际是 gather（源 → host）；历史命名，别按 scatter 理解
        buffer_handle.ptr(), value.data(), value.size_bytes(), "put:" + key);  // dst=注册 buffer，src=用户指针，context 拼 key 便于定位报错
    if (!scatter_result) {
        return tl::unexpected(scatter_result.error());     // D2H/拷贝失败已在 gather 内部记日志，这里只透传 TRANSFER_FAIL
    }

    std::vector<Slice> slices = split_into_slices(buffer_handle);  // 按 kMaxSliceSize 切成若干 {ptr,size}；切片要匹配 master 的 slice_lengths 与单次 TE 请求上限

    auto put_result = client_->Put(key, slices, config);   // 交给 Client::Put：PutStart → TE 写 → PutEnd，全程同步阻塞
    if (!put_result) {
        return tl::unexpected(put_result.error());         // 错误码原样上抛，这一层不做重试
    }

    return {};                                             // tl::expected 的成功空值：等价于 tl::expected<void,ErrorCode>()
}                                                          // buffer_handle 在此析构 → RAII 把这块注册内存还给池子
```

**这一段在做什么**
- 依次校验三个前置条件：`prefer_alloc_in_same_node` 不支持、`client_` 已初始化、allocator 非空；任一不满足都在分配内存之前返回。
- 从预先注册好的 `ClientBufferAllocator` 里 `allocate(value.size_bytes())`，拿到一块 RAII 管理的 `BufferHandle`。
- 把用户指针指向的内容搬进这块注册 buffer：`gather_maybe_device_to_host`，CPU 源退化为 memcpy，GPU 源走 D2H。
- 用 `split_into_slices` 把连续 buffer 切成 `kMaxSliceSize` 粒度的 `Slice` 列表，得到 `{ptr, size}` 数组。
- 调 `Client::Put(key, slices, config)`，后续 PutStart / TE 写 / PutEnd 全部在这一个调用里完成。
- 函数返回时 `BufferHandle` 析构，注册内存自动归还池子。

**重点语法 / 惯用法**
- `std::span<const char>`：C++20 非拥有连续视图，既能接 `std::string` 也能接裸指针加长度；`size_bytes()` 是字节数，`size()` 是元素个数，源码两处混用（char 下数值相同）。
- `tl::expected<void, ErrorCode>`：Rust `Result` 风格，`!x` 判失败，`tl::unexpected(e)` 造失败，`return {}` 造成功。
- `std::optional<BufferHandle>` 配合 `auto &buffer_handle = *alloc_result;`：`allocate` 用 optional 而不是 expected，失败没有细分错误码，只能给一个笼统的 `INVALID_PARAMS`。
- `BufferHandle` 是 move-only 的 RAII 句柄（拷贝构造/赋值均 `= delete`），析构里调 `release_fn_` 或让 `OffsetAllocationHandle` 释放；这里的释放靠作用域结束。
- `gather_maybe_device_to_host` 内部用 `device::GetAcceleratorRegistry().RuntimeAccelerators().CopyToHost`，纯 CPU 指针时它是一次 memcpy，所以这条路径永远有一次拷贝。
- 变量名 `scatter_result` 与函数名 `gather_...` 不一致，是历史命名残留，读代码时以函数名为准。

**与 6.7.3 的差异**
- 图第 1 步只有一条 `App->>W: put`，实际有两层：`RealClient::put`（计时 + 返回码转换）和 `RealClient::put_internal`（真正逻辑）。
- 图上没画"用户数据先拷进注册 buffer"。这一步是 TE 能做 RDMA 的前提：只有注册过的内存才能被网卡直接读写，用户指针没有注册。
- 因此"零拷贝"只对本来就是注册 buffer 的调用成立；从 Python `put` 进来的普通对象一定有一次拷贝，GPU tensor 还多一次 D2H。
- 图上第 1 步隐含"数据已经在 W 手里"；代码里 `allocate` 失败会在接触 master 之前返回，第 2 步根本不会发生。
- 图上第 6 步的返回值是 `Put` 的 `expected`，而 `put` 又把它压成 `int64_t`（0 成功 / 负错误码），Python 看到的是整数而不是异常。
- 图上 `W` 是抽象的写入方；`put_internal` 还被 `put_dummy_helper` 用另一个 allocator 复用，所以同一段逻辑也服务 shm dummy client。

mooncake-store/src/real_client.cpp:2543-2556

```cpp
int RealClient::put(const std::string &key, std::span<const char> value,  // Python/C++ 边界的入口：返回 int64 错误码，不是 expected
                    const ReplicateConfig &config) {
    auto result = execute_timed_operation<tl::expected<void, ErrorCode>>(  // 模板参数显式给返回类型；后三个 lambda 分别是主体 / 成功判定 / 观测
        [&]() {                                                            // 主体：按引用捕获全部外部变量
            return put_internal(key, value, config, client_buffer_allocator_);  // 注意用的是成员池；put_dummy_helper 走的是别的池
        },
        [](const auto &ret) { return ret.has_value(); },                   // 成功判定：expected 有值才算成功，失败不打点
        [&](uint64_t latency_us, const auto &) {                           // 观测：只有成功判定为真时才会被调用
            client_->ObserveTransferOperation(TransferOperationKind::kWrite,  // 把这次传输归类为写
                                              "put", value.size_bytes(),      // 操作名 + 字节数进计数器/直方图
                                              latency_us);
        });
    return to_py_ret(result);                                              // expected<void,ErrorCode> → 0 或负错误码
}
```

**这一段在做什么**
- 这是 `RealClient::put` 的薄包装：把真正的 put 塞进 `execute_timed_operation`，顺带计时与上报指标。
- 主体 lambda 调 `put_internal(key, value, config, client_buffer_allocator_)`，用的是 RealClient 自己的成员 buffer 池。
- 成功判定 lambda 只检查 `ret.has_value()`；失败时观测 lambda 不会被调用。
- 观测 lambda 调 `ObserveTransferOperation(kWrite, "put", 字节数, 微秒)`，把本次写计入指标。
- 最后 `to_py_ret(result)` 把 `expected<void, ErrorCode>` 转成 `int64_t` 返回给绑定层。

**重点语法 / 惯用法**
- `execute_timed_operation<Result>(op, success_fn, observe_fn)` 是泛型工具模板：`Result` 必须显式给出，三个可调用对象用转发引用接，内部 `std::forward` 逐个调用。
- 计时用 `std::chrono::steady_clock`（单调时钟，不受系统时间调整影响），`elapsed_us_since` 里做 `duration_cast<microseconds>`。
- 第一个 lambda 是 `[&]() { ... }`，`value` 是 span，捕获的是视图本身而不是底层数据，开销很小。
- 第三个 lambda 的第二个参数写成匿名的 `const auto &`：只关心延迟，不关心返回值内容。
- `to_py_ret` 用 `requires is_supported_return_type_v<T>` 约束，只接受 `void` 或整型；`void` 特化直接返回 0。
- `ObserveTransferOperation` 是 `Client` 成员，内部先判 `metrics_` 是否为空，所以这里不需要自己判空。

**与 6.7.3 的差异**
- 图上第 1–6 步是"一次 put"；代码里这次调用被统计了两次延迟：这里统计整个 put（含两次 RPC），`Client::Put` 内部的 `put_latency_us` 只统计 TE 写段。
- 图上箭头从 `App` 直连 `W`；代码里返回的是整数错误码，Python 侧靠 `to_py_ret` 的约定解读，失败不抛异常。
- 图上没有 hot cache、checksum、heal 这些旁路，它们都在 `Client::Put` 里，不在这层包装里。
- 图上第 6 步的"Put 返回"意味着数据可读；但 `OBJECT_ALREADY_EXISTS` 也会让这里返回 0（见 2.2 的短路），此时并没有真的写入。
- 图上把 `W` 与 `FS` 画成不同进程；`put_internal` 在 owner 自己身上同样会被调用，`W`/`FS` 只是角色区分而非进程区分。

### 2.2 `Client::Put`：PutStart、TE 写、PutEnd（第 2–4、6 步）

`PutStart` 只回答"往哪里写"，`TransferWrite` 才真正搬数据，最后 `DetermineFinalizeDecision` 决定 `PutEnd` 还是 `PutRevoke`。

mooncake-store/src/client_service.cpp:1959-2115

```cpp
tl::expected<void, ErrorCode> Client::Put(const ObjectKey& key,          // 成功即"数据已写进至少一个副本，且已 PutEnd 完成"
                                          std::vector<Slice>& slices,    // 非 const 引用：TE submitter 的签名就是非 const，slice 可能被按需调整
                                          const ReplicateConfig& config) {  // 副本策略：replica_num / nof_replica_num / dfs_replica_num ...
    std::optional<uint64_t> object_checksum;                              // 有值才在 PutEnd 时塞进 ObjectMeta；校验和默认关闭，保持 nullopt
    if (object_checksum_enabled_) {                                       // 构造期定下的只读开关
        auto checksum_result = ComputeObjectChecksumForSlices(            // 按整个对象算校验和，覆盖所有 slice
            key, slices, CalculateSliceSize(slices));                     // CalculateSliceSize 只是把各 slice.size 累加
        if (!checksum_result) {
            return tl::unexpected(checksum_result.error());               // 算校验和失败就不发 RPC，直接返回
        }
        object_checksum = *checksum_result;                               // 解引用 expected 取值
    }

    // Prepare slice lengths
    std::vector<size_t> slice_lengths;                                    // master 只要长度，不要指针：指针留在客户端
    for (size_t i = 0; i < slices.size(); ++i) {
        slice_lengths.emplace_back(slices[i].size);                       // emplace_back 原地构造 size_t，省一次临时对象
    }

    ReplicateConfig client_cfg = AttachHostId(config);                    // 拷贝一份再改，不动调用方的 config
    if (protocol_ == "cxl") {                                             // CXL 共享内存协议要指定本机 segment
        client_cfg.preferred_segment = local_hostname_;                   // 注意是 preferred_segment（单数、已废弃字段），不是 preferred_segments
    }

    if (hot_cache_) {                                                     // hot cache 是可选的，未启用时为 null
        hot_cache_->RemoveHotKey(key);                                    // 写之前先失效旧缓存：否则本地读可能命中过期值
    }

    // Start put operation
    auto start_result = master_client_.PutStart(key, slice_lengths, client_cfg);  // 图上第 2 步：同步 RPC，返回副本描述符列表
    if (!start_result &&
        start_result.error() == ErrorCode::OBJECT_ALREADY_EXISTS &&       // 只在"已存在"这一种错误上尝试修复
        healDanglingLocalDiskReplica(key) == DiskReplicaHealResult::kEvicted) {  // 且必须证明旧 LOCAL_DISK 文件确实没了、且已 evict
        start_result = master_client_.PutStart(key, slice_lengths, client_cfg);  // 修复成功才重试一次，不循环
    }
    if (!start_result) {
        ErrorCode err = start_result.error();                             // 先取出来，下面要比较多次
        if (err == ErrorCode::OBJECT_ALREADY_EXISTS) {
            VLOG(1) << "object_already_exists key=" << key;
            return {};                                                    // 短路成成功：图上第 3、4、5 步在这里都不会发生
        }
        if (err == ErrorCode::NO_AVAILABLE_HANDLE) {
            LOG(WARNING) << "Failed to start put operation for key=" << key
                         << PUT_NO_SPACE_HELPER_STR;                      // 空间不足是预期内错误，WARNING + 排查提示串
        } else {
            LOG(ERROR) << "Failed to start put operation for key=" << key
                       << ": " << toString(err);                          // 其他错误按 ERROR 记，toString 把枚举转可读名
        }
        return tl::unexpected(err);                                       // 原始错误码上抛，不做映射
    }

    ReplicaTransferSummary transfer_summary;                              // 纯本地计数器：分配了几类副本、成功/失败几个
    for (const auto& replica : start_result.value()) {
        transfer_summary.RecordAllocatedReplica(replica);                 // 只统计 MEMORY/NOF/DFS；DISK 与 LOCAL_DISK 不计数
    }

    // Record Put transfer latency (all replicas)
    auto t0_put = std::chrono::steady_clock::now();                       // 计时起点在 PutStart 之后：put_latency_us 不含这次 RPC

    // We must deal with disk replica first, then the disk putrevoke/putend can
    // be called surely
    if (storage_backend_) {                                                // 只有配了 legacy 文件后端（--root_fs_dir）才会有 DISK 副本
        for (auto it = start_result.value().rbegin();                      // 逆序遍历副本列表
             it != start_result.value().rend(); ++it) {
            const auto& replica = *it;                                     // 解引用迭代器，拿 Descriptor 引用，不拷贝
            if (replica.is_disk_replica()) {                               // 只认 legacy DISK；LOCAL_DISK 不会出现在 PutStart 的返回里
                // Store to local file if storage backend is available
                auto disk_descriptor = replica.get_disk_descriptor();      // 取出 file_path 等磁盘描述
                PutToLocalFile(key, slices, disk_descriptor);              // 同步写本地文件；返回 void，内部失败会自己 PutRevoke(DISK)
                // 只需要一个磁盘副本：命中第一个就跳出，不再看后面的
                break;  // Only one disk replica is needed
            }
        }
    }

    for (const auto& replica : start_result.value()) {                     // 正序遍历，逐副本做 TE 写
        if (replica.is_memory_replica() || replica.is_nof_replica()) {     // 只处理"当下能直接寻址写"的两类副本
            // Transfer data using allocated handles from all replicas
            const auto replica_type = replica.is_memory_replica()          // 三元表达式，只为给统计归类
                                          ? ReplicaType::MEMORY
                                          : ReplicaType::NOF_SSD;
            ErrorCode transfer_err = TransferWrite(replica, slices);       // 同步写：submit + future->get() 等完成
            if (transfer_err != ErrorCode::OK) {
                transfer_summary.RecordFailure(replica_type, transfer_err);  // 记失败，并保留第一个错误码
                continue;                                                  // 关键：不中断循环，其他副本继续写，最后统一裁决
            }
            transfer_summary.RecordSuccess(replica_type);                  // 成功只加计数
        }
    }

    if (transfer_summary.allocated_dfs_replicas > 0 &&
        NonDfsTransfersSucceeded(transfer_summary)) {                      // 前提：所有 MEMORY/NOF 副本都写成功，才愿意写 DFS
        std::vector<std::string> dfs_keys;
        std::vector<const std::vector<Slice>*> dfs_slices;                 // 存指针而不是拷贝整个 slices 向量
        std::vector<DistributedFSDescriptor> dfs_descriptors;
        for (const auto& replica : start_result.value()) {
            if (!replica.is_dfs_replica()) {
                continue;                                                  // 只挑 DFS 副本
            }
            dfs_keys.push_back(key);                                       // 每个 DFS 副本对应一个 key 条目
            dfs_slices.push_back(&slices);                                 // 同一份 slices，取地址复用
            dfs_descriptors.push_back(replica.get_dfs_descriptor());
        }
        for (auto dfs_result :                                            // 批量写，返回每项 ErrorCode 的 vector
             WriteDfsReplicas(dfs_keys, dfs_slices, dfs_descriptors)) {
            if (dfs_result == ErrorCode::OK) {
                transfer_summary.RecordSuccess(ReplicaType::DFS);
            } else {
                transfer_summary.RecordFailure(ReplicaType::DFS, dfs_result);
            }
        }
    } else if (transfer_summary.allocated_dfs_replicas > 0) {               // 分配了 DFS 但被放弃（因为有别的副本先失败）
        // A DFS replica was allocated but is abandoned because a peer replica
        // failed first. Without this counter the write simply disappears: no
        // error is recorded against DFS and no I/O is attempted. Counted per
        // key, matching SubmitDfsWrites.
        VLOG(1) << "Skipping DFS write, action=skip, key=" << key
                << ", reason=non_dfs_transfer_failed";                     // 只在 VLOG(1) 里留痕，默认日志看不到
        if (auto* dfs_metric = GetDfsMetricPtr()) {                        // C++17 if-init：指针作用域被限制在 if 内，顺带判空
            dfs_metric->RecordSkippedWrites(1);                            // 计入 skipped 指标，否则这次"消失的写"无法观测
        }
    }

    auto us_put = std::chrono::duration_cast<std::chrono::microseconds>(   // 计算耗时并 cast 到微秒
                      std::chrono::steady_clock::now() - t0_put)
                      .count();
    if (metrics_) {                                                        // 指标对象是可选注入的
        metrics_->transfer_metric.put_latency_us.observe(us_put);          // 直方图只观察，不做聚合
    }

    const auto finalize_decision =                                         // 纯本地决策，不发任何 RPC
        DetermineFinalizeDecision(config, transfer_summary);

    if (finalize_decision.end_type.has_value()) {                           // ALL / MEMORY / NOF_SSD 之一；nullopt 表示一个副本都没写成
        auto end_result = master_client_.PutEnd(                           // 图上第 4 步：把副本从 PROCESSING 转成 COMPLETE
            ObjectMeta{key, object_checksum}, *finalize_decision.end_type);  // 聚合初始化 + optional 解引用
        if (!end_result) {
            ErrorCode err = end_result.error();
            LOG(ERROR) << "Failed to end put operation: " << err;
            return tl::unexpected(err);                                    // 注意：这里直接上抛，不会再补一次 PutRevoke
        }
    }

    if (finalize_decision.revoke_type.has_value()) {                       // 例如 MEMORY 成功、NOF_SSD 失败时 revoke_type=NOF_SSD
        auto revoke_result =
            master_client_.PutRevoke(key, *finalize_decision.revoke_type);
        if (!revoke_result) {
            LOG(ERROR) << "Failed to revoke put operation";
            return tl::unexpected(revoke_result.error());                  // revoke 失败也上抛；此时另一类副本可能已经 COMPLETE
        }
    }

    if (!finalize_decision.success) {
        return tl::unexpected(finalize_decision.error);                    // 结束动作做完才报错：副本已按裁决清理或保留
    }

    return {};
}
```

**这一段在做什么**
- 可选地算对象级 checksum，并把 `slices` 的**长度**抽成 `slice_lengths`（指针留在本地，master 不需要）。
- 拷贝 config 注入 `host_id`（`AttachHostId`）与 CXL 的 `preferred_segment`，并在写之前让本机 hot cache 的该 key 失效。
- 调 `PutStart`（图第 2 步）：成功拿回 `std::vector<Replica::Descriptor>`，每个 descriptor 带着 TE 能直接写的 buffer 地址。
- `OBJECT_ALREADY_EXISTS` 先尝试 `healDanglingLocalDiskReplica` 修复悬空 `LOCAL_DISK`，修复成功才重试一次；仍存在则**当成成功返回**。
- 用 `ReplicaTransferSummary` 统计分配/成功/失败计数，然后按副本类型分别写：legacy DISK → `PutToLocalFile`，MEMORY/NOF_SSD → `TransferWrite`，DFS → `WriteDfsReplicas`。
- 计时后由 `DetermineFinalizeDecision` 算出 `(end_type, revoke_type, success, error)`，再按需调 `PutEnd` / `PutRevoke`，最后按 `success` 决定返回。

**重点语法 / 惯用法**
- `tl::expected` 的固定三件套：`if (!x) return tl::unexpected(x.error());`、`*x` 解引用、`x.value()` 取值、`return {}` 表示成功。
- `std::optional` 的 `has_value()` / `*opt`：`object_checksum` 与 `finalize_decision.end_type` 都用它表达"这次不需要带 / 不需要调"。
- `FinalizeDecision` 是聚合体，在 `DetermineFinalizeDecision` 里用指定初始化器 `.end_type = ...` 构造，返回后按字段消费。
- `ReplicaType` 是 `enum class`（`MEMORY/DISK/LOCAL_DISK/NOF_SSD/ALL/DFS`）；`ALL` 的语义是"所有同步副本一起 finalize"，不是"所有类型都写"。
- `auto* dfs_metric = GetDfsMetricPtr()` 用的是 C++17 if-with-initializer，指针只在 if 语句内可见。
- `start_result.value()` 在循环里被反复调用（包括 `rbegin()/rend()`），`expected::value()` 每次都会检查状态；读的时候可直接当成"取 vector 引用"。
- `NonDfsTransfersSucceeded` 在没有任何 MEMORY/NOF 副本时返回 true（0==0），所以纯 DFS 的 put 不会被这个条件挡住。

**与 6.7.3 的差异**
- 图第 4 步写的是 `PutEnd(ALL)`，代码里结束动作是一个二元组：`end_type` 和 `revoke_type` 可能分别出现，`ALL` 只是其中一种取值。
- 图第 2→3 步默认一定发生；代码里 `OBJECT_ALREADY_EXISTS` 会在 `PutStart` 之后立刻 `return {}`，**第 3、4、5 步被短路**，且返回上层是成功。
- 图里没画 legacy DISK 与 DFS 两条写路径，也没画"必须先处理 disk replica"的顺序约束；这两条都属于同一次 `Put`。
- 图第 6 步在代码里不是 `PutEnd` 返回就结束，而是 `finalize_decision.success` 说了算；`PutEnd` 成功但 `success=false` 时仍然返回错误。
- 图里没有 checksum、hot cache 失效、`healDanglingLocalDiskReplica` 重试这些旁路，而它们都会改变"这次 put 到底有没有真的写进去"。
- 图第 5 步的入队发生在 `PutEnd` 内部的 master 侧（`PushOffloadingQueue` + `inc_refcnt`），客户端完全看不到；`Put` 只是等 `PutEnd` 的 OK。

### 2.3 MasterService::PutStart 与 AllocateReplicas（第 3 步）

时序第 2→3 步在代码里是五层套娃：`PutStart` → `AdmitWithDfsRecovery` → `admit` 闭包 → `AllocateAndInsertMetadata` → `AllocateReplicas` + `InsertMetadata`。读之前先记住一句：**这一层只决定"数据往哪块可写缓冲区里放"，`LOCAL_DISK` 在这里结构上不可能出现**——它要等到 3.x 的 `NotifyOffloadSuccess` 才被登记出来。

#### 2.3.1 入口：PutStart 只做参数归一与前置校验

`mooncake-store/src/master_service.cpp:5589-5599`

```cpp
auto MasterService::PutStart(const UUID& client_id, const std::string& key,   // W 的 client_id：会被写进 ObjectMetadata.client_id，PutEnd 时校验
                             const TenantId& tenant_id,                      // RPC 层已做过 ResolveTenantIdForWrite，这里已是最终租户
                             const uint64_t slice_length,                    // 单个 slice 的字节数（不是对象总长），决定要分配多大的 buffer
                             const ReplicateConfig& config)                  // 副本策略：replica_num / nof_replica_num / dfs_replica_num
    -> tl::expected<std::vector<Replica::Descriptor>, ErrorCode> {           // 注意返回的是 Descriptor 列表：跨 RPC 只能传可序列化描述，不能传 Replica 本体
    const auto object_id = MakeObjectIdentityForRequest(key, tenant_id);     // 归一成 {tenant, user_key}；后面所有 shard 查找都用它，避免两个参数到处传
    auto plan = PrepareWriteStart(client_id, key, object_id.tenant_id,       // 前置校验 + 归一化（5442-5526）：非法配置在这里就被挡掉
                                  slice_length, config, "put_start");        // 最后的 "put_start" 只是日志里的 action 名，UpsertStart 复用同一函数
    if (!plan) {                                                             // 校验失败：INVALID_PARAMS / DFS_SERVICE_UNAVAILABLE / …
        return tl::make_unexpected(plan.error());                            // 原样上抛，不包装、不改码
    }
```

**这一段在做什么**
- `PutStart` 是写路径第一个 master RPC（时序第 2 步），函数体只做两件事：把 `(user_key, tenant_id)` 归一成 `ObjectIdentity`，再调 `PrepareWriteStart` 拿到 `WriteStartPlan`。
- `PrepareWriteStart` 已经挡掉了：全零副本配置、`dfs_replica_num > 1`、DFS 带非默认租户、DFS 未初始化、`prefer_alloc_in_same_node` + NoF、非 `USE_NOF` 构建下的 NoF 请求、CACHELIB 下超过 `kMaxSliceSize` 的 slice。
- 它还顺手算出三样后续要用的东西：`soft_pin_request`（软钉）、`writer_host_id`（`LOCAL_FIRST` 或同节点偏好且 `replica_num == 1` 时才有值）、`group_id`（分组淘汰用）。
- 返回类型是 `vector<Replica::Descriptor>` 而不是 `vector<Replica>`——客户端只拿到地址，buffer 所有权留在 master。

**重点语法 / 惯用法**
- `auto f(...) -> tl::expected<T, E>`：尾置返回类型配合 `tl::expected` 这个"值或错误码"的 sum type，是这套代码的统一错误模型。
- `tl::make_unexpected(e)` 构造错误态；调用方统一写 `if (!x) return tl::make_unexpected(x.error());` 做错误透传。
- `const auto object_id = ...` 是值语义的小结构体，函数内部靠它定位 shard，避免把 `key` 和 `tenant` 两个参数反复配对。
- 空 tag 类型（`WriteStartPlan` 用指定初始化器构造）让"可选字段"在类型层面就能看出来，例如 `plan->writer_host_id` 可能为空串。

**与 6.7.3 的差异**
- 图上第 2 步只画了一次 `PutStart`；实际客户端还可能走 `BatchPutStart`（rpc_service.cpp:502-608），逐个 key 调同一个 `PutStart`，返回 `OBJECT_ALREADY_EXISTS` 的 key 会被当成成功。
- 图上没有"前置校验"这一层：非法 `ReplicateConfig` 在**分配之前**就返回 `INVALID_PARAMS`，不会出现"分配完再回滚"。
- 图上没有 `writer_host_id` 这个维度。它只影响 `preferred_segments` 的排序（本机优先），并不保证副本落在本机。

#### 2.3.2 admit 闭包：在 key 的 shard 写锁里完成"清理旧对象 → 分配 → 发布"

`mooncake-store/src/master_service.cpp:5601-5651`

```cpp
    auto admit = [&](bool* dfs_allocation_failed)                            // 把"清理+分配+发布"打包成闭包，AdmitWithDfsRecovery 失败后可原样重试
        -> tl::expected<std::vector<Replica::Descriptor>, ErrorCode> {       // 显式写出返回类型，方便被 std::function 接住
        auto now = std::chrono::system_clock::now();                         // 整个 admit 用同一个时间戳：既做 put_start_time，也做 stale 判定基准
        std::shared_lock<std::shared_mutex> client_lock(client_mutex_);      // GetRetainingClientIdsLocked 要求调用方持 client_mutex_（master_service.h:1060）
        std::shared_lock<std::shared_mutex> shared_lock(master_state_mutex_); // 全局读锁贯穿整个 admit：所有 metadata 访问都在它下面发生
        auto retaining_clients = GetRetainingClientIdsLocked();              // 哪些 client 还"留着"数据（离线但未过期），stale 清理时不能动它们的副本
        client_lock.unlock();                                                // 拿到列表就立刻放掉 client_mutex_，避免与需要反向锁序的路径互等
        // The accessor holds the key across the stale cleanup and the
        // create, so no other writer of the key lands in between.
        MetadataAccessorRW accessor(this, object_id,                         // 拿该 key 所在 shard 的**写锁**（当前实现里持 key 即持整个 shard 写锁）
                                    MetadataAccessorRW::CreateKey{});        // CreateKey tag：key 不存在也占位，保证后面能 Publish
        if (auto admitted = ChargeTenantQuota(accessor.QuotaAccount(), 0);   // 以 0 额度做一次准入检查（租户存在/未超限），真正的额度在 2.3.4 预扣
            !admitted) {
            return tl::make_unexpected(admitted.error());                    // 租户层面的拒绝在这里就返回，不进入分配
        }
        if (auto cleaned = CleanupStaleBeforeWrite("PutStart(stale cleanup)",// 清理历史遗留的 PROCESSING 副本；oplog 模式下这一步还可能直接回 OBJECT_ALREADY_EXISTS
                                                   accessor, retaining_clients);
            !cleaned) {
            return tl::make_unexpected(cleaned.error());                     // 清理失败（含 oplog 持久化失败）直接上抛
        }
        if (accessor.IsPublished()) {                                        // key 下已经有对象，说明是重写/覆盖场景
            auto& metadata = accessor.Get();
            if (metadata.HasReplica(&Replica::fn_is_completed) ||            // 已有 COMPLETE 副本 = 对象可读，语义上就是"已存在"
                metadata.put_start_time + put_start_discard_timeout_sec_ >=  // 或者上一次 PutStart 还在丢弃宽限期内——也不能抢
                    now) {
                return tl::make_unexpected(ErrorCode::OBJECT_ALREADY_EXISTS); // 客户端把该错误当成功（数据已在），第 3 步不会发生
            }
            if (enable_oplog_ && ordered_oplog_writer_) {                    // HA 模式：删除动作必须先落 oplog，从库才能重放出同样的结果
                auto err = PersistRemoveForHA("PutStart(stale cleanup REMOVE)",
                                              object_id.tenant_id, key);
                if (!err) {
                    return tl::make_unexpected(err.error());                 // 持久化失败 → 整个 PutStart 失败，避免主从分叉
                }
            }
            auto replicas = PopReplicasWithCacheTotalAccounting(             // 把上一轮没写完的 PROCESSING 副本整体摘走（连带扣减缓存总量）
                metadata, &Replica::fn_is_processing);
            if (!replicas.empty()) {
                FreeDfsReplicas(key, replicas);                              // DFS 副本有额外的空间账，必须先显式释放
                std::lock_guard lock(discarded_replicas_mutex_);             // 延迟释放队列有自己的锁，锁序在 shard 锁之后
                discarded_replicas_.emplace_back(
                    std::move(replicas),
                    metadata.put_start_time + put_start_release_timeout_sec_); // 真正归还 buffer 要等这个 TTL：可能还有读者握着旧地址
            }
            accessor.Erase();                                                // 抹掉旧对象；注意 key 仍然被 accessor 占着，紧接着可以重新 Publish
        }

        return AllocateAndInsertMetadata(                                    // 真正的分配 + 发布，见 2.3.4
            accessor, client_id, slice_length, config, plan->writer_host_id, // writer_host_id 为空表示不做本机优先
            plan->group_id, now, plan->soft_pin_request, std::nullopt,       // std::nullopt：Put 路径不吃 committed soft-pin deadline
            dfs_allocation_failed);
    };
```

**这一段在做什么**
- 把"持锁 → 清理同名旧对象 → 分配并发布新 metadata"打包成一个可重试的闭包，交给 `AdmitWithDfsRecovery` 按需再执行。
- 加锁顺序是 `client_mutex_` → `master_state_mutex_`，拿到 `retaining_clients` 后立刻释放 `client_mutex_`，只保留全局读锁贯穿全程。
- `MetadataAccessorRW(..., CreateKey{})` 占住 key：清理与创建之间不会有别的写者插入，这是"擦除 + 重建"原子性的来源。
- 已有对象时按两条规则拒绝：有 `COMPLETE` 副本，或 `put_start_time + put_start_discard_timeout_sec_ >= now`（前一次写还在宽限期）。两者都不满足才拆掉重建。
- 拆旧对象时把 `PROCESSING` 副本移进 `discarded_replicas_`，用一个更长的 `put_start_release_timeout_sec_` 延迟归还，防止读者手里还留着旧地址。

**重点语法 / 惯用法**
- `if (auto admitted = f(); !admitted)` 是 C++17 的 init-statement + 条件，变量作用域被限制在 if 内，读起来比先声明再判断更紧凑。
- `MetadataAccessorRW::CreateKey{}` 是**空 tag struct**，纯粹用来在重载之间选构造函数（"只持现有对象" vs "无论如何占住 key"）。
- `[&]` 捕获一切外部变量；返回类型写在下一行，是因为它太长，也是为了让 `AdmitWithDfsRecovery` 用 `std::function` 接收时一眼看清签名。
- `discarded_replicas_.emplace_back(std::move(replicas), deadline)`：`Replica` 是 move-only（持有 `AllocatedBuffer`），只能移动不能拷贝。
- `metadata.HasReplica(&Replica::fn_is_completed)` 传的是**静态成员函数指针**，作为谓词；`fn_is_processing` 同理，这是这套代码统一的"按状态筛选副本"手法。

**与 6.7.3 的差异**
- 图上第 3 步之前 master 看起来是无状态的；实际 `PutStart` 可能**先删掉一个已存在但未完成的对象**，还要走 oplog 持久化，并把它挂进延迟释放队列——图里完全没有这段。
- `OBJECT_ALREADY_EXISTS` 在这一层产生，不是分配失败。客户端把它当成功返回，所以这类 key 根本走不到第 3 步的回包。
- 图上是一次分配；代码里 `AdmitWithDfsRecovery` 最多会让这个闭包跑三次（见 2.3.3），每次都重新清理、重新分配。
- 整个闭包跑在 shard 写锁里（`MetadataAccessorRW` 的注释明确：当前没有 per-key 锁，持 key 即持 shard 写锁），所以分配器快照与 eviction 标志也是在锁内完成的。

#### 2.3.3 收尾：AdmitWithDfsRecovery 与配额拒绝指标

`mooncake-store/src/master_service.cpp:5653-5661`

```cpp
    // Tenant over-quota remains a background-eviction concern. Bucket-count
    // exhaustion is different: see AdmitWithDfsRecovery.
    auto result = AdmitWithDfsRecovery(config, admit);                       // 仅当 DFS 容量不足时才会重试，内存/NoF 不足直接失败
    if (!result && result.error() == ErrorCode::TENANT_QUOTA_EXCEEDED) {     // 先判 !result 再取 error()：expected 有值时调 error() 是未定义行为
        MasterMetricManager::instance().inc_tenant_quota_reject(             // 单独打点，便于和"容量不足"区分
            object_id.tenant_id.value(), "quota_exceeded");
    }
    return result;                                                           // 其余错误不加工，直接上抛
}                                                                            // PutStart 结束；回包即第 3 步的 descriptor 列表
```

**这一段在做什么**
- `AdmitWithDfsRecovery`（5562-5587）的恢复条件很窄：只有 `dfs_allocation_failed == true`（DFS 自己因 `NO_AVAILABLE_HANDLE` 失败）且配置了 DFS 才重试。
- 重试分两级：先加 `dfs_bucket_recovery_mutex_` 重跑一次 `admit`；仍失败则 `TryRecoverDfsSpaceAfterAllocationFailure()` 回收一个 bucket 再跑第三次。
- 重试前会先释放 `admit` 里的 shard 锁，因为回收要扫描 metadata shard，不能持锁递归加锁。
- 只有 `TENANT_QUOTA_EXCEEDED` 会额外打点；其他错误由 `execute_rpc` 层面统一计失败数。

**重点语法 / 惯用法**
- `std::function<tl::expected<std::vector<Replica::Descriptor>, ErrorCode>(bool*)>` 作为形参，把闭包类型擦除掉，才能在同一函数里反复调用。
- `std::lock_guard recovery_lock(dfs_bucket_recovery_mutex_);` 用 CTAD（类模板实参推导）省掉 `<std::mutex>`。
- 失败分支里把 `dfs_allocation_failed` 重新置 `false` 再调 `admit`，否则第二次失败会被误判成"还是 DFS 的锅"。
- 传指针而不是引用的原因：闭包签名要能被 `std::function` 统一，且要表达"可选输出参数"（调用方可以传 `nullptr`）。

**与 6.7.3 的差异**
- 图上第 3 步是一条直线；这里最坏会执行三次完整分配，还会顺带触发一次 DFS bucket 回收（有真实的数据删除副作用），耗时和影响面都远大于图示。
- 图上没有 `TENANT_QUOTA_EXCEEDED` 这个失败面；开多租户后它是 `PutStart` 的合法返回值之一。
- 图上没有"重试"箭头：三次 `admit` 之间 master 会释放并重新获取 shard 写锁，所以中间的 metadata 状态对外可见（其他请求可以插进来）。

#### 2.3.4 AllocateAndInsertMetadata：先扣配额、再分配、再发布

`mooncake-store/src/master_service.cpp:5404-5440`

```cpp
auto MasterService::AllocateAndInsertMetadata(
    MetadataAccessorRW& accessor, const UUID& client_id, uint64_t value_length, // accessor 已持有 key 的写锁，这里只是借用
    const ReplicateConfig& config, const std::string& writer_host_id,
    const std::string& group_id,
    const std::chrono::system_clock::time_point& now,
    const ResolvedSoftPinRequest& soft_pin_request,
    std::optional<std::chrono::system_clock::time_point>
        committed_soft_pin_timeout,
    bool* dfs_allocation_failed)
    -> tl::expected<std::vector<Replica::Descriptor>, ErrorCode> {
    assert(!accessor.IsPublished());                                         // 前置条件：key 下必须为空，否则 Publish 会 abort
    const std::string& key = accessor.GetKey();

    const uint64_t pending_quota_charge =
        RequestedMemoryQuotaCharge(value_length, config);                    // 按 slice 长度 × 请求的 MEMORY 副本数算出预扣额度
    auto quota_result =
        ChargeTenantQuota(accessor.QuotaAccount(), pending_quota_charge);    // 先在账上占住，避免并发写把租户额度打穿
    if (!quota_result) {
        return tl::make_unexpected(quota_result.error());                    // TENANT_QUOTA_EXCEEDED 从这里出去
    }

    auto allocation_result = AllocateReplicas(
        key, value_length, config, writer_host_id, dfs_allocation_failed);   // 见 2.3.5-2.3.9；失败原因可能是 NO_AVAILABLE_HANDLE / DFS_*
    if (!allocation_result) {
        ReleaseTenantQuota(accessor.QuotaAccount(), pending_quota_charge);   // 分配失败必须退额度，否则额度会随失败请求泄漏
        return tl::make_unexpected(allocation_result.error());
    }

    auto insert_result = InsertMetadata(
        accessor, client_id, value_length, config, group_id, now,
        soft_pin_request, std::move(allocation_result.value()),              // vector<Replica> 的所有权在此转移，之后 allocation_result 不可再用
        pending_quota_charge, std::move(committed_soft_pin_timeout));
    if (!insert_result) {
        ReleaseTenantQuota(accessor.QuotaAccount(), pending_quota_charge);   // 发布失败同样退额度（InsertMetadata 内部已把 metadata 拆掉）
    }
    return insert_result;                                                    // 成功时返回的就是第 3 步 RPC 的 payload
}
```

**这一段在做什么**
- 三个动作严格有序：预扣租户内存配额 → 分配副本 → 发布 metadata，任一步失败都把配额退回。
- 配额按 `value_length × replica_num` 预扣，扣的是 **MEMORY** 副本的账；NoF/DFS 不在这个账本里。
- 发布成功后配额不在本函数结算，而是把 `pending_quota_charge` 交给 `InsertMetadata` 里的 `AdoptPendingCharge` 落账。
- 所以"配额占用"与"对象是否存在"在这里是同生共死的：要么都成立，要么都回滚。

**重点语法 / 惯用法**
- `std::move(allocation_result.value())`：`tl::expected::value()` 返回引用，`std::move` 之后才真正转移 `vector<Replica>`。
- `std::optional<time_point>` 用 `std::move` 传，因为 `InsertMetadata` 也按值收（`Publish` 里再 `std::move` 进构造函数）。
- 失败路径全部 `return tl::make_unexpected(...)`，成功路径直接 `return insert_result;`——`expected` 可以直接作为返回值，不用再包一层。
- `assert(!accessor.IsPublished())` 是内部契约检查，不是运行时错误处理；违反它是编程错误。

**与 6.7.3 的差异**
- 图上第 3 步只有"成功/失败"；代码里还有第三种状态：**分配成功但发布失败**（配额已扣又退回），以及 **发布成功但配额落账失败**（见 2.3.11 的回滚）。
- 图上没有租户配额这一层，单租户部署时它确实是空操作，但多租户下它会先于容量成为拒绝原因。
- 图上 `PutStart` 只回一次包；代码里"分配"与"发布"是两次可能各自失败的子步骤，只有两次都成功才有第 3 步的 descriptor 列表。

#### 2.3.5 AllocateReplicas 头部：write_mode 与三个计数器

`mooncake-store/src/master_service.cpp:5140-5153`

```cpp
auto MasterService::AllocateReplicas(const std::string& key,                // 只用于日志与本机优先的 hash 输入，不参与寻址
                                     uint64_t value_length,                 // 要分配的字节数，直接喂给分配器
                                     const ReplicateConfig& config,
                                     const std::string& writer_host_id,     // 空 = 不做本机优先；非空 = 把该 host 的 segment 排到前面
                                     bool* dfs_allocation_failed)
    -> tl::expected<std::vector<Replica>, ErrorCode> {                      // 这里返回的是 Replica 本体（含 buffer 与 id），不是 Descriptor
    if (dfs_allocation_failed != nullptr) {                                 // 允许调用方传 nullptr（不关心 DFS 失败原因）
        *dfs_allocation_failed = false;                                     // 先清零：只有 DFS 因容量失败时才会被置 true
    }
    std::vector<Replica> replicas;                                          // MEMORY / NOF / DISK / DFS 四种副本最终都追加进这一个 vector
    const auto write_mode = DetermineReplicaWriteMode(config);              // SINGLE / FLEXIBLE_DUAL / RELIABLE_MULTI，决定"少给副本算不算失败"
    size_t allocated_memory_replicas = 0;                                   // MEMORY 实到数量
    size_t allocated_nof_replicas = 0;                                      // NoF 实到数量（仅 USE_NOF 构建会变）
    bool has_enough_memory_segments = false;                                // segment 数量本来就够？只用来决定要不要置 need_mem_eviction_
```

**这一段在做什么**
- `dfs_allocation_failed` 是"可选输出参数"：传 `nullptr` 就完全不碰它，传指针才在 DFS 容量不足时置位。
- `write_mode` 由配置推导（replica.h:226-237）：`dfs_replica_num == 0 && replica_num == 1 && nof_replica_num == 1` 是 `FLEXIBLE_DUAL_REPLICA`；任一数量 > 1 或配了 DFS 是 `RELIABLE_MULTI_REPLICA`；其余是 `SINGLE_REPLICA`。
- 三个计数器的作用各不相同：前两个判"是否满足请求"，第三个判"失败该不该触发内存淘汰"。
- 默认 `ReplicateConfig{1,0,0}`（replica.h:108-111）走的就是 `SINGLE_REPLICA` + 1 个 MEMORY。

**重点语法 / 惯用法**
- `DetermineReplicaWriteMode` 定义在头文件里且是 `inline`，master 与 client 共用同一份判定，避免两边对"少给副本"的理解漂移。
- 指针参数 + 先置默认值，是 C++ 里模拟"可选输出参数"的常见写法（不能用引用，因为要允许不传）。
- 计数器与 `bool` 都给了初值，后面所有分支只做 `+=` / `= true`，没有"未初始化就读"的风险。

**与 6.7.3 的差异**
- 图上第 3 步写成 "MEMORY/NoF_SSD" 的二选一；代码里 MEMORY 与 NoF 是两个**先后独立**的分支，可以同时存在，`write_mode` 才决定它们是否必须都成功。
- 图上没有"实到数量可能少于请求数量"这个维度，而这正是 2.3.8 那段检查存在的理由。
- 图上没有 `need_mem_eviction_` 这类"给后台线程的暗示"；它是本函数唯一的副作用输出，不进入任何返回值。

#### 2.3.6 MEMORY 分支：唯一由客户端直接 RDMA 写入的 buffer

`mooncake-store/src/master_service.cpp:5155-5217`

```cpp
    if (config.replica_num > 0) {                                           // 只有显式要 MEMORY 副本时才分配
        std::vector<std::string> preferred_segments;                        // 候选 segment 顺序表，越靠前越优先
        auto append_preferred_segment = [&preferred_segments](              // 局部 lambda：去重后追加
                                            const std::string& segment_name) {
            if (!segment_name.empty() &&
                std::find(preferred_segments.begin(), preferred_segments.end(),
                          segment_name) == preferred_segments.end()) {      // 已存在就不重复加：重复会扭曲采样权重
                preferred_segments.push_back(segment_name);
            }
        };
        if (!config.preferred_segment.empty()) {                            // 旧字段优先（向后兼容）
            append_preferred_segment(config.preferred_segment);
        } else {
            for (const auto& preferred_segment : config.preferred_segments) { // 新字段是列表，可以有多个偏好
                append_preferred_segment(preferred_segment);
            }
        }
        AllocatorManager allocator_snapshot;                                // 声明在锁外：锁释放后还要用它做真正的分配
        {
            ScopedAllocatorAccess allocator_access =
                segment_manager_.getAllocatorAccess();                      // RAII：持 segment_mutex_ 读锁，离开作用域自动释放
            has_enough_memory_segments = allocator_access.getAllocatorManager()
                                             .getServingNames()
                                             .size() >= config.replica_num; // 段数够却分不到，说明是"段满了"，值得触发淘汰
            if (!writer_host_id.empty()) {
                auto host_ordered_segments =
                    allocator_access.GetHostOrderedSegments(writer_host_id,
                                                            key);           // 按 writer 所在 host 给 segment 排序（同 host 优先）
                for (const auto& segment_name : host_ordered_segments) {
                    append_preferred_segment(segment_name);
                }
                if (!host_ordered_segments.empty()) {
                    VLOG(1) << "key=" << key
                            << ", writer_host_id=" << writer_host_id
                            << ", local_first_preferred_segments="
                            << host_ordered_segments.size();                // 只是日志：本机优先不保证命中
                }
            }
            allocator_snapshot = allocator_access.SnapshotAllocatorManager(); // 拷贝一份 AllocatorManager，把"读指针"变成"持有快照"
        }

        auto allocation_result = allocation_strategy_->Allocate(
            allocator_snapshot, value_length, config.replica_num,           // 请求数量；实际可能少于它（best-effort）
            preferred_segments, std::set<std::string>(), ReplicaType::MEMORY); // 空 excluded_segments + 明确要 MEMORY 类型

        if (!allocation_result.has_value()) {                                // 一个都没分到
            VLOG(1) << "Failed to allocate replicas for key=" << key
                    << ", error: " << allocation_result.error();
            if (allocation_result.error() == ErrorCode::INVALID_PARAMS) {    // 参数问题：重试也没用
                return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
            }
            if (write_mode != ReplicaWriteMode::FLEXIBLE_DUAL_REPLICA) {     // 双副本"灵活"模式下，内存侧失败不算致命，可以只靠 NoF
                MasterMetricManager::instance().inc_put_start_alloc_failures();
                if (has_enough_memory_segments) {
                    need_mem_eviction_ = true;                              // 段够但满了 → 叫醒后台淘汰，下次可能就成功
                }
                return tl::make_unexpected(ErrorCode::NO_AVAILABLE_HANDLE);
            }
        } else {
            allocated_memory_replicas = allocation_result->size();           // 记录实到数量（可能 < replica_num）
            replicas = std::move(allocation_result.value());                 // 第一个来源，直接整体赋值，不用逐条 push_back
        }
    }
```

**这一段在做什么**
- 先把 `preferred_segment`（旧）或 `preferred_segments`（新）拼成去重的候选表，旧字段优先级更高。
- 在 `ScopedAllocatorAccess` 作用域内做两件事：判断 segment 数够不够（只影响淘汰标志），以及把 `writer_host_id` 对应的本机 segment 排到候选前面。
- `SnapshotAllocatorManager()` 把分配器视图拷出来，锁释放后才调用 `Allocate`，把持锁时间压到最短。
- `Allocate` 的语义是 best-effort：返回 1..replica_num 个副本，只有"一个都没有"才算失败（allocation_strategy.h:238-246）。
- 失败时按 `write_mode` 分流：`FLEXIBLE_DUAL_REPLICA` 允许内存侧全灭交给 NoF；其余模式记指标、必要时置 `need_mem_eviction_`，返回 `NO_AVAILABLE_HANDLE`。

**重点语法 / 惯用法**
- `ScopedAllocatorAccess allocator_access = segment_manager_.getAllocatorAccess();` 是 RAII 守卫，析构时放 `segment_mutex_`；因此 `allocator_snapshot` 必须在块外声明，不能只存在于块内。
- `allocator_access.GetAllocatorManager().getServingNames().size()` 链式调用返回临时 vector，只取 `.size()`，避免额外拷贝整个名字表。
- `allocation_result->size()` 与 `*.value()`：`tl::expected` 的 `operator->` 直接透到底层 `vector`。
- `std::set<std::string>()` 传空集合而不是 `{}`，因为该参数是按值传的 `std::set`，`{}` 会有歧义风险。
- `replicas = std::move(allocation_result.value())` 而不是 `emplace_back` 循环：MEMORY 是第一个来源，此时 `replicas` 必为空。

**与 6.7.3 的差异**
- 图上第 3 步只回 "MEMORY/NoF_SSD（PROCESSING）"，看不出"数量可能不足"；在 `RELIABLE_MULTI_REPLICA` 下 allocation 只要 ≥1 就算过检查，缺的部分要到 transfer 阶段才被撤销（`DetermineFinalizeDecision`）。
- 图上没有"本机优先"：`writer_host_id` 只调整候选顺序，不保证落在本机，也不改变返回的副本种类。
- 图上的 `PROCESSING` 在这里就已经定了——`Allocate` 产出的 `Replica` 构造时就带 `PROCESSING`，不是后面改的。
- 图上没有"段数够但满了 → 置 `need_mem_eviction_`"这条反馈链。

#### 2.3.7 NoF_SSD 分支：同一套分配器，换一个 segment 管理器

`mooncake-store/src/master_service.cpp:5219-5251`

```cpp
#ifdef USE_NOF                                                              // 编译期开关：不开 NoF 的构建里这段根本不存在
    if (config.nof_replica_num > 0 &&
        nof_segment_manager_.getMountedSegmentCount() > 0) {                // 没挂载 NoF segment 时静默跳过，不报错
        ScopedAllocatorAccess allocator_access =
            nof_segment_manager_.getAllocatorAccess();                      // 与 MEMORY 不同：不取快照，直接持有访问对象
        const auto& allocator_manager = allocator_access.getAllocatorManager(); // 引用语义，所以 Allocate 必须在同一作用域内调用

        std::vector<std::string> preferred_segments =
            config.preferred_nof_segments;                                  // NoF 有自己的偏好字段，不与 MEMORY 共用

        auto allocation_result = allocation_strategy_->Allocate(
            allocator_manager, value_length, config.nof_replica_num,
            preferred_segments, std::set<std::string>(), ReplicaType::NOF_SSD); // 唯一区别是 ReplicaType，决定产出 NoFDescriptor

        if (!allocation_result.has_value()) {
            VLOG(1) << "Failed to allocate nof replicas for key=" << key
                    << ", error: " << allocation_result.error();
            if (allocation_result.error() == ErrorCode::INVALID_PARAMS) {
                return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
            }
            if (write_mode != ReplicaWriteMode::FLEXIBLE_DUAL_REPLICA) {     // 与 MEMORY 对称：灵活双副本模式下这一侧可以先不管
                MasterMetricManager::instance().inc_put_start_alloc_failures();
                need_nof_eviction_ = true;                                  // 置的是 NoF 的淘汰标志，不是 mem 的
                return tl::make_unexpected(ErrorCode::NO_AVAILABLE_HANDLE);
            }
        } else {
            allocated_nof_replicas = allocation_result->size();
            for (auto& replica : allocation_result.value()) {
                replicas.push_back(std::move(replica));                      // 追加而不是覆盖：MEMORY 副本已经在 vector 里了
            }
        }
    }
#endif
```

**这一段在做什么**
- 整段被 `#ifdef USE_NOF` 包住；`PrepareWriteStart` 在非 NoF 构建下会提前把 `nof_replica_num > 0` 判成 `INVALID_PARAMS`（5483-5490）。
- 第二道门是 `getMountedSegmentCount() > 0`：没有挂载 NoF segment 时静默跳过，`allocated_nof_replicas` 保持 0，由后面的满足性检查决定后果。
- 分配器从 `segment_manager_` 换成 `nof_segment_manager_`，`ReplicaType::NOF_SSD` 让 `Allocate` 产出 `NoFDescriptor`。
- 结果用 `push_back` **追加**到同一个 `replicas`，因此一个对象可以同时拥有 MEMORY 与 NoF 副本，顺序上 MEMORY 在前。
- 失败处理与 MEMORY 对称，只是置 `need_nof_eviction_`。

**重点语法 / 惯用法**
- `#ifdef` 直接切在函数体中间：宏未定义时 `config.nof_replica_num` 字段仍存在、仍可传，只是永远不会被消费。
- 这里与 2.3.6 的写法差异是刻意的：MEMORY 用 `SnapshotAllocatorManager()`（因为后面要放锁），NoF 直接引用（锁一直持到 `Allocate` 返回）。
- `for (auto& replica : allocation_result.value()) replicas.push_back(std::move(replica));` 逐元素移动，因为目标是"追加"到已有 vector。
- `allocated_nof_replicas = allocation_result->size();` 在循环前取，避免循环内 move 后误读。

**与 6.7.3 的差异**
- 图上 "MEMORY/NoF_SSD" 读起来像二选一；代码里 NoF 是**附加**的，`FLEXIBLE_DUAL_REPLICA` 下两类副本会同时出现。
- 图上没有"未挂载 NoF 就静默跳过"这个分支：跳过之后若 MEMORY 也失败，最终错误是 `NO_AVAILABLE_HANDLE`，而不是某种 NoF 专属错误。
- 图上 NoF 与 MEMORY 看起来同一个流程；实际两者用两个 segment 管理器、两套偏好字段、两个淘汰标志。

#### 2.3.8 配额满足性检查：best-effort 与 reliable 的分界

`mooncake-store/src/master_service.cpp:5253-5291`

```cpp
    if (!HasExpectedReplicaAllocation(config, allocated_memory_replicas,
                                      allocated_nof_replicas)) {           // 唯一判定点：见 master_service.cpp:110-122
        if ((config.replica_num > 0 &&
             allocated_memory_replicas != config.replica_num) ||            // 这两组条件只为"精确诊断"服务
            (config.nof_replica_num > 0 &&
             allocated_nof_replicas != config.nof_replica_num)) {
            MasterMetricManager::instance().inc_put_start_alloc_failures();
            if (config.replica_num > 0 &&
                allocated_memory_replicas != config.replica_num &&
                has_enough_memory_segments) {                               // 段够却分不到 = 段满，值得淘汰
                need_mem_eviction_ = true;
            }
            if (config.nof_replica_num > 0 &&
                allocated_nof_replicas != config.nof_replica_num) {
                need_nof_eviction_ = true;                                  // NoF 不管段数，直接置位
            }
        }
        VLOG(1) << "Failed to satisfy replica allocation requirement for key="
                << key << ", requested_memory_replicas=" << config.replica_num
                << ", allocated_memory_replicas=" << allocated_memory_replicas
                << ", requested_nof_replicas=" << config.nof_replica_num
                << ", allocated_nof_replicas=" << allocated_nof_replicas;
        return tl::make_unexpected(ErrorCode::NO_AVAILABLE_HANDLE);        // 注意：返回码统一是 NO_AVAILABLE_HANDLE
    }

    // Best-effort / flexible modes may pass the check above with fewer
    // replicas than requested (see HasExpectedReplicaAllocation). Surface
    // the degradation so callers and operators can detect the reduced
    // redundancy instead of failing silently.
    if (allocated_memory_replicas < config.replica_num ||
        allocated_nof_replicas < config.nof_replica_num) {                  // 能走到这里说明是"允许少给"的模式
        MasterMetricManager::instance().inc_put_start_partial_allocations(); // 降级要可见，否则运维看不出冗余已经减少
        LOG(WARNING) << "key=" << key << ", action=put_start_partial_allocation"
                     << ", requested_memory_replicas=" << config.replica_num
                     << ", allocated_memory_replicas="
                     << allocated_memory_replicas
                     << ", requested_nof_replicas=" << config.nof_replica_num
                     << ", allocated_nof_replicas=" << allocated_nof_replicas;
    }
```

**这一段在做什么**
- `HasExpectedReplicaAllocation`（master_service.cpp:110-122）是唯一判定点：无 NoF/DFS 时只要 `allocated_memory_replicas > 0`；`FLEXIBLE_DUAL_REPLICA` 只要两类之和 > 0；其余必须**精确等于**请求数。
- 不满足时先按"哪一侧短了"分别打点并置对应淘汰标志，再统一返回 `NO_AVAILABLE_HANDLE`。
- 满足但少于请求数时**不算失败**，只打 `inc_put_start_partial_allocations` 和一条 WARNING。
- 这段是"分配成功"与"冗余达标"两个概念的分离点：`PutStart` 可以成功，但副本数比配置少。

**重点语法 / 惯用法**
- `HasExpectedReplicaAllocation(config, mem, nof)` 是文件内匿名命名空间的自由函数，不挂在类上——它被 master 与 client 两侧各自实现一份等价语义（client_service.cpp:322-336）。
- 返回值只有 `bool`，错误码统一，所以调用方无法从错误码区分"内存不足"还是"NoF 不足"，诊断只能看日志与指标。
- `LOG(WARNING)` 与 `VLOG(1)` 的分工：前者是运维可见的降级信号，后者是调试细节。
- 注释块（5278-5281）解释了"为什么允许通过"：与其静默失败，不如降级并留痕。

**与 6.7.3 的差异**
- 图上第 3 步默认"请求 1 个就给 1 个"；这一层明确允许"少给但仍成功"，图的语义只在 `RELIABLE_MULTI_REPLICA` 下严格成立。
- 图上没有 `need_mem_eviction_` / `need_nof_eviction_` 这种"给后台淘汰线程的暗示"，也没有"部分分配"指标。
- 图上第 3 步失败只有一种原因；代码里 `NO_AVAILABLE_HANDLE` 同时覆盖"一个没分到"和"没分够"两种情形（后者仅限严格模式）。

#### 2.3.9 追加写目标：legacy DISK 与 DFS

`mooncake-store/src/master_service.cpp:5293-5319`

```cpp
    if (use_disk_replica_) {                                                // 只有配置了 --root_fs_dir 时才为 true（master_service.cpp:462-463）
        std::string file_path =
            FileUtil::ResolvePathFromKey(key, root_fs_dir_, cluster_id_);   // 由 key 推导出主控本机文件路径，客户端可直接按路径写
        replicas.emplace_back(file_path, value_length,
                              ReplicaStatus::PROCESSING);                   // 选 Replica(string, uint64_t, status) 重载 → DiskDescriptor
    }

    if (config.dfs_replica_num > 0) {                                       // DFS 是显式请求的，不是全局开关
        if (!dfs_allocator_ || !dfs_allocator_->IsInitialized()) {
            LOG(ERROR) << "Failed to allocate DFS replica for key=" << key
                       << ", error=dfs_allocator_not_initialized";
            return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE); // 此时前面已分配的 memory 副本随 replicas 析构自动归还
        }
        auto alloc = dfs_allocator_->Allocate(key, value_length);           // 返回 DistributedFSDescriptor（path/offset/shard）
        if (!alloc) {
            LOG(ERROR) << "Failed to allocate DFS replica for key=" << key
                       << ", error=" << alloc.error();
            if (dfs_allocation_failed != nullptr) {
                *dfs_allocation_failed =
                    alloc.error() == ErrorCode::NO_AVAILABLE_HANDLE;        // 只有容量不足才置位，其它错误不触发上层重试
            }
            return tl::make_unexpected(alloc.error());
        }
        replicas.emplace_back(std::move(*alloc), ReplicaStatus::PROCESSING); // 移动 descriptor 进副本，同样是先占位后完成
    }

    return replicas;                                                        // 到此为止：MEMORY / NoF / DISK / DFS，没有 LOCAL_DISK
```

**这一段在做什么**
- `use_disk_replica_` 是全局开关（`root_fs_dir_` 非空即开），开启后每个 key 都会**额外**获得一个 legacy DISK 副本，路径由 key 推导。
- DFS 是每请求开关（`dfs_replica_num`），分配失败时把 `dfs_allocation_failed` 置为 `NO_AVAILABLE_HANDLE`，供 `AdmitWithDfsRecovery` 决定是否回收 bucket 重试。
- 两种副本都用 `ReplicaStatus::PROCESSING` 构造：先占位，等客户端写完 `PutEnd` 才转 `COMPLETE`。
- 这里返回的 `replicas` 就是全部可写目标，**整段没有 `LOCAL_DISK` 分支**——原因见 2.3.12。

**重点语法 / 惯用法**
- `replicas.emplace_back(file_path, value_length, ReplicaStatus::PROCESSING)` 靠参数类型选 `Replica(std::string, uint64_t, ReplicaStatus)` 重载（replica.h:316-318）。
- `replicas.emplace_back(std::move(*alloc), ReplicaStatus::PROCESSING)` 移动 `DistributedFSDescriptor`，避免拷贝路径字符串。
- `if (!alloc)` 用的是 `tl::expected` 的布尔语义，比 `has_value()` 更简洁；但取 `.error()` 前必须先确认失败。
- `alloc.error() == ErrorCode::NO_AVAILABLE_HANDLE` 是唯一的"可恢复"信号，其余错误（如 `DFS_*`）直接上抛。

**与 6.7.3 的差异**
- 图上第 3 步只回 MEMORY / NoF_SSD；代码在同一次响应里还可能带上 legacy DISK 与 DFS 的 descriptor。这四者的共同点才是关键：**客户端此刻都能直接写**。
- 图上第 3 步只有"成功"一条路径；这里 DFS 分配失败会让整个 `PutStart` 失败，而此前已分配的 memory 副本只是随局部变量析构归还，不经过 `discarded_replicas_`。
- 图上没有"legacy DISK 与 DFS 互不排斥"这一层：两者可以同时出现在同一个 `replicas` 里。

#### 2.3.10 InsertMetadata：Replica → Descriptor，并把所有权交给 master

`mooncake-store/src/master_service.cpp:5322-5366`

```cpp
auto MasterService::InsertMetadata(
    MetadataAccessorRW& accessor, const UUID& client_id, uint64_t value_length,
    const ReplicateConfig& config, const std::string& group_id,
    const std::chrono::system_clock::time_point& now,
    const ResolvedSoftPinRequest& soft_pin_request,
    std::vector<Replica>&& replicas, uint64_t pending_quota_charge,          // 右值引用：明确表达"所有权转进 metadata"
    std::optional<std::chrono::system_clock::time_point>
        committed_soft_pin_timeout)
    -> tl::expected<std::vector<Replica::Descriptor>, ErrorCode> {
    assert(!accessor.IsPublished());                                        // key 必须为空：Publish 内部还有同样的断言（object_route.h:267-277）
    const std::string& key = accessor.GetKey();
    const TenantId& tenant_id = accessor.GetTenantId();
    const auto deadline_to_index = committed_soft_pin_timeout;              // 拷一份，后面 Publish 会把原值 move 走

    std::vector<Replica::Descriptor> replica_list;                          // 这就是第 3 步回给客户端的 payload
    std::unordered_set<ReplicaID> eligible_replica_ids;                     // 哪些副本 id 参与本次 session 记账
    replica_list.reserve(replicas.size());                                  // 预留容量：避免返回前再扩容
    eligible_replica_ids.reserve(replicas.size());
    int i = 0;                                                              // 只被下面省略的 VLOG 序号使用
    VLOG(1) << "PutStart, create replicas: client_id=" << client_id
            << ", key=" << key << ", value_length=" << value_length;
    for (const auto& replica : replicas) {
        const auto desc = replica.get_descriptor();                         // 只拷贝可序列化字段（id/status/buffer 描述），不碰 buffer 所有权
        replica_list.emplace_back(desc);                                    // desc 后面还要读（日志），所以传左值拷贝而不是 move
        eligible_replica_ids.insert(replica.id());                          // ReplicaID 是进程内自增 id，只在 master 生命周期内有效

        // ...（略：按 MEMORY / NoF / DFS 三种类型分别 VLOG 打印 buffer_address、transport_endpoint、dfs_file，只影响日志）
    }
```

**这一段在做什么**
- 断言 key 为空（`Publish` 要求），然后逐条把 `Replica` 转成 `Descriptor` 放进 `replica_list`。
- `Descriptor` 的结构是 `{id, variant<Memory,NoF,Disk,LocalDisk,DFS>, status}`（replica.h:674-680），所以回包里既有地址信息，也有 `PROCESSING` 状态和副本 id。
- 同时把每个副本 id 记进 `eligible_replica_ids`，供后面的 session 记账决定"哪个副本算这次写的功劳"。
- 返回的 `replica_list` 直接作为 RPC 回包内容；而 `replicas` 本体（含 `AllocatedBuffer` 与 refcnt）在 `Publish` 时转进 `ObjectMetadata`，master 侧继续持有。

**重点语法 / 惯用法**
- `replica.get_descriptor()` 是 `const` 成员函数（replica.h:385），内部按变体类型逐个填充；对"buffer 已被回收"的副本会打印错误并给出全 0 地址（replica.h:877-884），不会抛异常。
- `replica_list.emplace_back(desc)` 而不是 `std::move(desc)`：`desc` 是 `const auto`，且后面日志还要用。
- `ReplicaID` / `UUID` 都是强类型别名，`unordered_set<ReplicaID>` 与 `unordered_set<UUID>` 不会混用。
- `std::vector<Replica>&&` + `std::move(...)` 的双重表达，是这套代码标注"所有权转移"的固定写法。

**与 6.7.3 的差异**
- 图上第 3 步只画"master 回包"；代码里回包的 descriptor 与 master 内部的 `Replica` 是**两份**数据：一份给客户端寻址，一份给 master 记 refcnt（时序第 5 步 `inc_refcnt` 作用在后者上）。
- 图上 `PROCESSING` 只是一个标注；代码里它是 `Descriptor.status` 字段，客户端据此知道"这批地址现在只能写、不能读"。
- 图上没有 `eligible_replica_ids`：session 记账要等 `PutEnd` 时才知道哪个副本真正写成功。

#### 2.3.11 Publish 之后：配额落账、session、soft-pin、is_processing

`mooncake-store/src/master_service.cpp:5368-5401`

```cpp
    accessor.Publish(client_id, now, value_length, std::move(replicas),      // 原地构造 ObjectMetadata：now 成为 put_start_time，replicas 所有权在此转移
                     std::move(committed_soft_pin_timeout),
                     config.with_hard_pin, config.data_type, group_id,       // 参数顺序必须与 object_metadata.h:70-78 完全一致
                     tenant_id, key);
    auto& metadata = accessor.Get();                                        // Publish 之后才能取 metadata
    if (enable_multi_tenants_) {
        auto adopt_result = metadata.quota_ledger.AdoptPendingCharge(       // 把预扣额度正式落账到对象名下
            accessor.QuotaAccount(), pending_quota_charge);
        if (!adopt_result) {
            LogTenantQuotaLedgerError(adopt_result, "adopt_pending", tenant_id,
                                      key);
            auto failed_replicas = metadata.PopReplicas();                  // 先把副本取回：ObjectMetadata 析构不会帮你注销 DFS 副本
            FreeDfsReplicas(key, failed_replicas);
            accessor.tenant_state_->metadata.erase(accessor.it_);           // 直接 erase，不走 accessor.Erase() 的完整配额/指标撤销路径
            accessor.it_ = accessor.tenant_state_->metadata.end();          // 手工把 accessor 置回"未发布"，否则析构会二次处理
            return tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
        }
    }
    metadata.BeginSessionTag(config.SessionId(), eligible_replica_ids);     // 记录本次写会话及可归功的副本集合
    metadata.BeginSoftPinAction(soft_pin_request,
                                std::move(eligible_replica_ids));           // 之后 eligible_replica_ids 不可再用
    if (deadline_to_index) {                                                // 只有传了 committed deadline 才进这个索引
        soft_pin_deadline_index_.Upsert(tenant_id.MakeScopedKey(key),       // 索引 key 用 tenant:key 拼接，与 user_key 不同
                                        GetMetadataShardIndex(metadata),
                                        *deadline_to_index);
    }
    // Wire grouped objects to the group's shared lease; ungrouped objects keep
    // the per-object lease created at construction.
    if (!group_id.empty()) {
        metadata.lease_ = RegisterGroupMember(tenant_id, key, group_id);    // 分组对象改用组的共享 lease，覆盖构造时创建的默认 lease
    }
    accessor.GetState().is_processing = true;                               // 对象级"正在写"标记：与副本级 PROCESSING 是两套状态

    return replica_list;                                                    // 回到 admit 闭包 → PutStart 返回给 W
```

**这一段在做什么**
- `accessor.Publish(...)` 用构造参数原地构造 `ObjectMetadata`：`now` 成为 `put_start_time`（后续 stale 判定与 TTL 回收都用它），`replicas` 的所有权在这一刻转移。
- 多租户下把预扣额度正式落账；失败则把刚发布的对象拆掉（取回副本、释放 DFS、直接 erase）并返回 `INTERNAL_ERROR`——这是 `Publish` 之后唯一允许失败的点。
- `BeginSessionTag` / `BeginSoftPinAction` 记录这次写的会话与软钉；`soft_pin_deadline_index_` 只在有 deadline 时插入。
- `group_id` 非空时把对象挂到组的共享 lease 上。
- 最后置对象级 `is_processing = true`：与副本级 `PROCESSING` 是两个独立状态，后台 reaper 依据它判断"这个对象写了一半"。

**重点语法 / 惯用法**
- `Publish` 是可变参模板（master_service.h:1838-1851），把参数完美转发给 `ObjectMetadata` 构造函数；参数顺序与 `object_metadata.h:70-78` 不一致时编译器不一定报错（类型相近），所以要用命名参数对照着读。
- `metadata.PopReplicas()` 把副本从 metadata 里摘出来再释放，是"对象析构不会处理外部资源（DFS 空间）"的补丁式写法。
- 直接操作 `accessor.tenant_state_->metadata.erase(accessor.it_)` 属于打破封装的最小回滚：绕过了 `accessor.Erase()` 的完整撤销路径。
- `tenant_id.MakeScopedKey(key)` 与 RPC 用的 `(tenant_id, user_key)` 二元组是两种 key 表示，别混。
- `metadata.lease_ = RegisterGroupMember(...)` 是直接改成员：lease 是对象生命周期的一部分，不是可选项。

**与 6.7.3 的差异**
- 图上第 3 步到第 4 步之间没有"对象级 `is_processing`"这个概念；它决定后台 reaper 是否把这个 key 当"写了一半"。
- 图上没有配额落账失败后的回滚路径（拆 metadata、放 DFS、退额度）。
- 图上没有 session / soft-pin / group lease 这些副作用；它们不影响第 3 步的回包内容，但会影响后续淘汰与 refcnt 行为。

#### 2.3.12 为什么 LOCAL_DISK 不在这张表里

先给结论，再用代码对照：

- **它不是"写目标"，是"写结果"。** `AllocateReplicas` 的四个分支只产出"客户端此刻能直接寻址并写入"的介质：MEMORY（segment 里的 buffer 地址）、NoF_SSD（同样形状的 buffer 地址）、legacy DISK（主控本机文件路径）、DFS（分布式文件路径 + offset）。`LOCAL_DISK` 的 descriptor 里根本没有可写地址，客户端拿着它无从下手。
- **它的登记点在时序第 15 步 `NotifyOffloadSuccess`。** owner 的 `FileStorage` 把数据真正落盘之后，master 才用 `Replica(client_id, metadata.data_size, metadata.transport_endpoint, ReplicaStatus::COMPLETE, record)`（master_service.cpp:9023-9025）造出一个 `COMPLETE` 的 `LOCAL_DISK` 副本。
- **`PROCESSING` 的三重作用。** (a) `IsReplicaReadable` 要求 `is_completed()`（master_service.cpp:4716-4719），读者看不到未写完的副本；(b) `PutStart` 的 stale 清理只摘 `fn_is_processing` 的副本（5635-5636）；(c) 后台 `DiscardExpiredProcessingReplicas` 以 `put_start_time + put_start_release_timeout_sec_` 为 TTL 回收（10353-10362）。
- **descriptor 的带出路径。** `AllocateReplicas` 返回 `vector<Replica>`（含 buffer 与 id）→ `InsertMetadata` 逐条 `get_descriptor()` 拷成 `vector<Descriptor>` → 作为 `PutStart` 返回值经 RPC 回到客户端；`Replica` 本体留在 metadata 里继续记 refcnt。

`mooncake-store/include/replica.h:294-299`

```cpp
struct LocalDiskDescriptor {                                                // 注意：没有 buffer 地址，没有协议字段
    UUID client_id;                                                         // owner 的 client id：读者据此找到"数据在谁手里的盘上"
    uint64_t object_size = 0;                                               // 对象原始大小，用于校验与预分配 client buffer
    std::string transport_endpoint;                                         // owner 的 offload **RPC** 地址（不是 Transfer Engine 地址）
    YLT_REFL(LocalDiskDescriptor, client_id, object_size, transport_endpoint); // 序列化反射：字段顺序即协议顺序
};
```

**这一段在做什么**
- 展示 `LocalDiskDescriptor` 的全部字段：只有 owner 的 `client_id`、对象大小、owner 的 offload RPC endpoint，没有任何可写地址。
- 这三个字段足以让读者定位 owner，再通过 `batch_get_offload_object` 让 owner 现场把数据取进 client buffer（时序第 18-22 步）。
- 与 `MemoryDescriptor` / `NoFDescriptor`（只有 `buffer_descriptor`，内含 `buffer_address_`）对照，就能看出为什么 `LOCAL_DISK` 无法出现在 `PutStart` 的返回里。
- master 侧要等到第 15 步才用这三个字段加一个 `COMPLETE` 状态构造副本，所以第 3 步的 descriptor 集合天然不包含它。

**重点语法 / 惯用法**
- `YLT_REFL(...)` 是序列化反射宏，列出参与序列化的字段；RPC 与 HA oplog 都依赖它，因此**字段顺序即协议顺序**，改字段要同步考虑兼容性。
- `UUID client_id;` 靠 `UUID` 自身的默认构造，`object_size = 0` 显式给默认值——宏不关心默认值，只关心字段名与顺序。
- `std::string transport_endpoint;` 在 `Descriptor` 变体里以值存储，序列化时按长度前缀编码；空串是合法值（表示 owner 没有可用的 RPC 地址）。
- 整个结构体没有方法、没有虚函数，是纯 POD-ish 的传输结构，便于放进 `std::variant` 并整体拷贝。

**与 6.7.3 的差异**
- 图上第 3 步写 "MEMORY/NoF_SSD（PROCESSING）"，读者容易以为 `LOCAL_DISK` 只是"另一种副本类型、只是这一步没选"；实际是**结构上不可能**：descriptor 里没有可写地址，master 也没有为它预留任何空间。
- 图上第 15 步的"登记"用的正是这张表里的三个字段；第 3 步的 descriptor 与第 15 步的 descriptor 不是同一个对象，也不是同一个来源。
- 图上第 15 步的 `LOCAL_DISK` 副本一出生就是 `COMPLETE`（9023-9025），这与第 3 步"一出生就是 `PROCESSING`"正好相反——这也是两者无法在同一函数里产生的原因。

### 2.4 MasterService::PutEnd：置 COMPLETE 并触发入队（第 4–5 步）

`PutEnd` 是 Put 的收尾 RPC。它先做身份与幂等校验，再把本次 target 的 PROCESSING 副本置为 COMPLETE；只有全部收尾完成后，第 5785 行起的入队块才会把 completed 的 MEMORY 副本投进 owner 的 offload 邮箱。入队失败不会向上传播——这是"Put 成功但数据可能永远不落 NVMe"的直接原因。

#### 2.4.1 函数入口、目标副本谓词与重试幂等

mooncake-store/src/master_service.cpp:5663-5723

```cpp
auto MasterService::PutEnd(const UUID& client_id, const ObjectMeta& object_meta, // 发起 End 的客户端 id + PutStart 时回给它的元信息（含 key）
                           const TenantId& tenant_id, ReplicaType replica_type) // tenant 随请求带上；replica_type 决定"这次 End 收尾哪几类副本"
    -> tl::expected<void, ErrorCode> { // tl::expected：成功返回 void，失败携带 ErrorCode，不抛异常
    const auto& key = object_meta.key; // 只读引用，不拷贝；key 是后续日志和状态查找的锚点
    std::shared_lock<std::shared_mutex> shared_lock(master_state_mutex_); // 全局状态锁取共享模式；真正的写互斥交给下面的分片访问器
    const auto object_id = MakeObjectIdentityForRequest(key, tenant_id); // (tenant, user_key) → 内部 ObjectIdentity，内部会做 tenant 归一化
    MetadataAccessorRW accessor(this, object_id); // RAII：构造时锁住该对象所在的 metadata 分片，析构自动释放；寿命覆盖整个函数
    if (!accessor.Exists()) { // 对象已被并发删除或淘汰
        LOG(ERROR) << "key=" << key << ", error=object_not_found"; // 记录日志，便于定位非法或迟到的 End
        return tl::make_unexpected(ErrorCode::OBJECT_NOT_FOUND); // 早返回；此时还没有改动任何状态
    }

    auto& metadata = accessor.Get(); // 取该对象的 ObjectMetadata 引用，后续所有副本操作都在分片锁保护下进行
    if (client_id != metadata.client_id) { // 只有 PutStart 的客户端才能 End，防止 A 写、B 收尾
        LOG(ERROR) << "Illegal client " << client_id << " to PutEnd key " << key
                   << ", was PutStart-ed by " << metadata.client_id;
        return tl::make_unexpected(ErrorCode::ILLEGAL_CLIENT); // 直接失败，不做任何清理
    }

    auto is_target_replica = [replica_type](const Replica& replica) { // 值捕获 replica_type：谓词无副作用、可复制，供循环与 VisitReplicas 复用
        if (replica_type == ReplicaType::ALL) { // 客户端 Put 默认发 ALL（对应时序图第 4 步的 PutEnd(ALL)）
            return (replica.is_memory_replica() && // ALL 覆盖三类：MEMORY
                    !replica.has_invalid_mem_handle()) || // handle 已失效的副本不算目标，避免把失效副本写成 COMPLETE
                   (replica.is_nof_replica() && // NoF（Nvme-oF SSD）副本
                    !replica.has_invalid_nof_handle()) ||
                   replica.is_dfs_replica(); // DFS 副本；注意 LOCAL_DISK 不在这里——它是 offload 的"产物"，不是 End 的目标
        }
        if (replica_type == ReplicaType::MEMORY) { // 只想收尾 MEMORY 副本
            return replica.is_memory_replica() &&
                   !replica.has_invalid_mem_handle();
        }
        if (replica_type == ReplicaType::NOF_SSD) { // 只想收尾 NoF 副本
            return replica.is_nof_replica() &&
                   !replica.has_invalid_nof_handle();
        }
        return replica.type() == replica_type; // 兜底：精确类型匹配（例如单独对 DFS 发 End）
    }; // 谓词里没有任何 LOCAL_DISK 分支，这与时序图第 15 步"无 LOCAL_DISK 才登记"互为呼应

    // A successful End removes the processing marker. Treat a retry as a
    // no-op only when every replica targeted by that End is already COMPLETE.
    // In particular, a promotion-owned PROCESSING replica keeps this check
    // from accepting a MEMORY/ALL End and is never modified here.
    if (!accessor.InProcessing()) { // processing 标记已被上一次成功的 End 摘掉 → 进入"是否为重试"的判定
        bool has_target_replica = false; // 本次 target 类型下是否至少存在一个副本
        bool all_target_replicas_complete = true; // 这些 target 副本是否已经全部 COMPLETE
        for (const auto& replica : metadata.GetAllReplicas()) { // 全量只读遍历，GetAllReplicas 返回 const 容器
            if (!is_target_replica(replica)) {
                continue; // 非本次目标的副本（例如 ALL End 遇到的 LOCAL_DISK）直接跳过
            }
            has_target_replica = true;
            if (!replica.is_completed()) { // 只要有一个目标副本没完成，就不能当作幂等重试
                all_target_replicas_complete = false;
                break; // 提前退出，后续副本无需再看
            }
        }
        if (has_target_replica && all_target_replicas_complete) { // 典型场景：客户端超时后重发 PutEnd
            return {}; // 幂等成功：不重复 mark_complete，也不重复入队
        }
        LOG(ERROR) << "key=" << key << ", error=no_primary_write_in_progress"; // 例如 promotion 持有的 PROCESSING 副本会让这里失败
        return tl::make_unexpected(ErrorCode::INVALID_WRITE); // 没有可收尾的写在途，拒绝
    }
```

**这一段在做什么**

- 校验对象存在：不存在直接 `OBJECT_NOT_FOUND`，不做任何副作用。
- 校验调用者身份：`client_id` 必须等于 `metadata.client_id`，否则 `ILLEGAL_CLIENT`。
- 定义 `is_target_replica`：把请求里的 `replica_type` 翻译成"哪些副本归这次 End 收尾"，`ALL` 覆盖 MEMORY / NoF / DFS 三类。
- 处理重复 End：`processing` 标记已摘且目标副本全部 COMPLETE 时直接返回成功，保证幂等。
- 拒绝"没有在途写"的 End：目标副本未完成且不在 processing 中，返回 `INVALID_WRITE`。

**重点语法 / 惯用法**

- `tl::expected<void, ErrorCode>` + `tl::make_unexpected(...)`：用返回值表达错误，调用方不必 try/catch。
- `std::shared_lock<std::shared_mutex>` 与 `MetadataAccessorRW` 双层锁：全局共享锁保持结构稳定，访问器负责按 key 分片写互斥。
- lambda 以值捕获 `replica_type`、以 `const Replica&` 作参数：谓词不修改状态，可安全传给 `VisitReplicas` 和普通循环。
- `has_invalid_mem_handle()` / `has_invalid_nof_handle()`：句柄失效的副本不能作为收尾目标，避免把已失效副本置为 COMPLETE。
- `return {};` 表示"空的 expected"，即成功返回 `void`，与错误路径的 `tl::make_unexpected` 形成对照。

**与 6.7.3 的差异**

- 时序图第 4 步只画了 `W->>M: PutEnd(ALL)`，看不出 End 可能被客户端重发并以幂等方式短路；重试路径不会再次入队。
- 图中第 5 步说"每个 completed MEMORY 副本"，但 End 实际还收尾 NoF 与 DFS，只是它们不会进入 offload 队列。
- 图中完全没有 LOCAL_DISK 副本，代码用谓词显式把它排除，与第 14–15 步"由 NotifyOffloadSuccess 登记 LOCAL_DISK"一致。
- 图中第 4 步紧接着就是第 5 步入队，代码在两者之间还隔着锁、身份、幂等三道检查。
- 图中未体现 promotion 场景：promotion 持有的 PROCESSING 副本会让 MEMORY/ALL 的 End 在这里被拒绝。

#### 2.4.2 置 COMPLETE：第 5 步真正的前置条件

mooncake-store/src/master_service.cpp:5745-5783

```cpp
// ...（略：5725-5743，bucket_allocator_ 的 DFS MarkCommitted 提交块，与本步的 MEMORY 收尾/入队无关）
    const bool had_completed_replica =
        metadata.HasReplica(&Replica::fn_is_completed); // 记录 End 之前是否已有 COMPLETE 副本：决定软 pin 是否属"首次完成"
    bool completed_pending_replica = false; // 本次被收尾的副本里，是否有被 pending soft pin 持有的
    metadata.VisitReplicas(
        [&is_target_replica](const Replica& replica) {
            return replica.is_processing() && is_target_replica(replica); // 谓词只选"仍在 PROCESSING 且属于本次 target"的副本
        },
        [this, &key, &metadata, &completed_pending_replica](Replica& replica) { // 修改器拿 Replica&，可原地改状态
            if (replica.is_processing() &&
                metadata.PendingSoftPinOwnsReplica(replica.id())) { // 该副本被挂起的 soft pin 持有
                completed_pending_replica = true; // 记下来，稍后决定是否提交 soft pin
            }
            metadata.CommitSessionTag(replica.id()); // 把 PutStart 时打的会话标记落定，供一致性/恢复逻辑识别
            replica.mark_complete(); // 关键一行：PROCESSING → COMPLETE；这是入队谓词 is_completed() 能选中它的前提
            if (replica.is_dfs_replica() && dfs_allocator_) { // DFS 副本额外更新分配器的访问记录
                const auto& desc = replica.get_dfs_descriptor();
                dfs_allocator_->UpdateAccess(key, desc);
            }
        });

    if (!had_completed_replica && completed_pending_replica &&
        metadata.HasReplica(&Replica::fn_is_completed)) { // 仅"首次出现 COMPLETE 副本"时才结算 pending soft pin
        const auto soft_pin_result =
            metadata.CommitPendingSoftPin(std::chrono::system_clock::now());
        ApplySoftPinEvaluation(metadata, soft_pin_result); // 依据 soft pin 结果调整淘汰优先级等策略
    }

    if (object_meta.object_checksum.has_value() || // 客户端给了 checksum，或本次 End 覆盖 MEMORY/NoF/ALL 时都要落 checksum
        replica_type == ReplicaType::ALL ||
        replica_type == ReplicaType::MEMORY ||
        replica_type == ReplicaType::NOF_SSD) {
        metadata.MutablePart(0).SetChecksum(object_meta.object_checksum); // checksum 存在 metadata 的第 0 个分片里
    }

    auto settle_result =
        SettlePrimaryWriteQuotaIfReady(accessor.QuotaAccount(), metadata); // primary write 的配额结算：把预扣转成实扣
    if (!settle_result) {
        return tl::make_unexpected(settle_result.error()); // 结算失败则整个 PutEnd 失败，后面的入队根本不会执行
    }
```

**这一段在做什么**

- 把本次 target 的 PROCESSING 副本逐个 `mark_complete()`——它是第 5 步"completed MEMORY 副本"这一说法的来源。
- 提交 session tag、维护 pending soft pin，并在"首次完成"时结算 soft pin。
- 条件满足时写入对象的 checksum。
- 结算 primary write 的配额，失败则整个 PutEnd 返回错误。
- 注意：这一段只改元数据，不入队；入队发生在下一块。

**重点语法 / 惯用法**

- `VisitReplicas(predicate, mutator)` 两参重载：谓词收 `const Replica&`，修改器收 `Replica&`，比手写 for 循环更安全。
- 捕获列表逐项列引用（`&key`、`&metadata`、`&completed_pending_replica`），明确谁会改状态，也避免 `shared_ptr` 拷贝。
- `metadata.HasReplica(&Replica::fn_is_completed)` 用成员函数指针作谓词：`fn_is_*` 是库提供的谓词工厂。
- `mark_complete()` 对非 PROCESSING 状态只打 WARNING 返回，不抛错，所以调用点不需要额外判断。
- `settle_result` 是 `expected`，`if (!settle_result)` 后原样透传 `error()`，保持错误码语义。

**与 6.7.3 的差异**

- 图中第 4 步到第 5 步之间只有一条箭头，代码里真正的转折点是 `mark_complete()`；没有它，入队谓词一个副本都选不中。
- 图中第 5 步写"每个 completed MEMORY 副本 PushOffloadingQueue"，但在代码里是先统一置 COMPLETE，再由下一块的 `VisitReplicas` 入队，两个动作分属不同代码块。
- 图中未出现 checksum 与配额结算；而配额结算失败会让 PutEnd 提前返回，`OffloadingTask` 不会被创建。
- 图中未出现 soft pin：首次完成的对象会在这里提交 pending soft pin，进而影响后续淘汰与 offload 时机的判断。
- 图中"completed 副本"只提 MEMORY，代码这里其实也把 NoF/DFS 置为 COMPLETE，只是入队时按类型过滤掉。

#### 2.4.3 入队块：三个门与"首个成功源副本"的 refcnt

mooncake-store/src/master_service.cpp:5785-5811

```cpp
    if (replica_type != ReplicaType::DFS && enable_offload_ && // 门 1：DFS 的 End 不触发 offload（数据已在 DFS 后端）；门 2：master 侧总开关
        !offload_on_evict_ && !metadata.HasReplica([](const Replica& replica) { // 门 3：offload_on_evict_ 模式下入队改由淘汰路径负责，这里必须让位
            return replica.is_dfs_replica() && replica.is_processing(); // 附加条件：仍有 DFS 副本在写，源数据可能还在变，先不 offload
        })) {
        auto& state = accessor.GetState(); // 对象级运行时状态（offloading_task / replication_task 等），与 metadata 同一分片锁保护
        // One marker covers every mirror pushed below, so the mirrors are
        // collected before the marker is recorded.
        std::optional<ReplicaID> source_id; // 记录"第一个成功入队"的副本 id，用来做 refcnt pin 与后续 dec
        std::vector<UUID> mirror_clients; // 收集所有成功入队的 owner client：一个 key 可能在多个 client 的邮箱里各有一份待办
        metadata.VisitReplicas(
            [](const Replica& replica) {
                return replica.is_completed() && replica.is_memory_replica(); // 只遍历已 COMPLETE 的 MEMORY 副本；NoF/DFS/LOCAL_DISK 全部被过滤
            },
            [this, &object_id, &source_id, &mirror_clients](Replica& replica) { // 引用捕获 object_id 与两个出参，遍历中就地累积
                auto result =
                    PushOffloadingQueue(object_id, replica, &mirror_clients); // 对该副本的每个 segment owner 投递任务；返回 expected 表示成败
                if (result && !source_id.has_value()) { // result 可隐式转 bool 表示成功；只有"第一次成功"才进入
                    replica.inc_refcnt(); // 原子 +1：refcnt > 0 的副本不可被淘汰，把源数据钉在内存里直到 offload 完成
                    source_id = replica.id(); // 记住被 pin 的是哪一个副本；NotifyOffloadSuccess / TTL 过期 / Cancel 都只 dec 它
                }
            });
        if (source_id.has_value() && !state.offloading_task) { // 有源且此前没有 marker 才写入；已有 marker 时新镜像不会被登记（边界情况）
            state.offloading_task =
                OffloadingTask{*source_id, std::chrono::system_clock::now(), // marker 记录源副本 id 与起始时间（TTL 从这里起算）
                               std::move(mirror_clients)}; // 移动而非拷贝，把镜像 client 列表交给对象状态保管
        }
    }
```

**这一段在做什么**

- 三个门 + 一个附加条件共同决定是否入队：`replica_type != DFS`、`enable_offload_`、`!offload_on_evict_`，且没有 PROCESSING 的 DFS 副本。
- 门 1 的依据：DFS 的 Put 数据已经落在 DFS 后端，不需要再从内存 offload 到本地盘。
- 门 3 的依据：`offload_on_evict_=true` 时入队发生在淘汰路径（`try_evict_or_offload`），若 PutEnd 也入队，同一对象会被两个入口各排一次。
- 遍历所有 completed MEMORY 副本，逐个 `PushOffloadingQueue`；只有第一个成功者 `inc_refcnt()` 并记为 `source_id`。
- 所有成功入队的 client 累积进 `mirror_clients`，随 `source_id` 一起写进对象级 `offloading_task`。
- 入队失败只让 `result` 为假，函数继续往下走，最终 `PutEnd` 仍返回成功。

**重点语法 / 惯用法**

- 跨行 `&&` 条件链，最后一项是立即调用的 lambda（`HasReplica([]{...})`），用一条表达式表达"不存在 processing 的 DFS 副本"。
- `if (result && ...)`：`tl::expected` 可隐式转换为 bool，成功为 true，代码据此区分成功/失败而不用 `.has_value()`。
- `std::optional<ReplicaID>` 表达"源副本尚未选定"，`!source_id.has_value()` 即"还没选过"。
- `std::move(mirror_clients)` 把 vector 移进 `OffloadingTask`，避免整表拷贝。
- 捕获列表按引用逐项列出，避免 `this` 之外的多余拷贝；`PushOffloadingQueue` 通过指针出参回填镜像。

**与 6.7.3 的差异**

- 图中第 5 步一句"每个 completed MEMORY 副本 PushOffloadingQueue；首个成功源副本 inc_refcnt"压缩掉了三个门和镜像收集，读图会以为 PutEnd 无条件入队。
- 图中没有 `offload_on_evict_` 这一互斥模式：该模式开启时 PutEnd 一行都不入队，任务由淘汰路径产生。
- 图中第 6 步紧接着就是 `Put 返回`；代码里入队失败没有任何返回值传播，这正是"Put 成功但可能永不落盘"的根因。
- 图中未画"仍有 PROCESSING DFS 副本"的抑制条件，代码用它避免在源数据可能变化时开写。
- 图中"每个副本"实际只指 MEMORY；NoF 在此被 `is_memory_replica()` 过滤，DFS 连门都进不来。
- 图中没有对象级 marker（`offloading_task`）；它正是第 14 步 `NotifyOffloadSuccess` 用来识别"这是 master 准入的 offload"的凭据。
- 为什么只给第一个成功副本 `inc_refcnt`：`dec_refcnt` 只针对 `offloading_task->source_id` 发生（NotifyOffloadSuccess、TTL 过期、Cancel、删除元数据各一处），若每个镜像都 inc 就会有 N-1 个永远没有配对 dec 的 pin，副本永久不可淘汰，即 #2997 那类泄漏。

#### 2.4.4 收尾：摘 processing、租约、事件与 oplog

mooncake-store/src/master_service.cpp:5813-5834

```cpp
    // If the object is completed, remove it from the processing set.
    if (metadata.AllReplicas(&Replica::fn_is_completed) && // 必须"所有"副本都 COMPLETE 才算整体完成
        accessor.InProcessing()) { // 半完成状态（如 NoF 未完成）会继续留在 processing 集合里
        accessor.EraseFromProcessing(); // 从 processing 集合摘除，后续请求不再把它当作在途写
    }

    SyncCacheTotalAccounting(metadata); // 同步 master 侧的内存占用统计
    // TODO: add inc_nof_cache_nums() (ranhaojia)
    metadata.GrantReadLease(std::chrono::milliseconds::zero()); // 零时长租约：这里只是把租约清成不延长，真正的读租约在 BatchGetReplicaList 时授予
    PublishKvStored(key, metadata, object_id.tenant_id); // 向 KV 事件流发布 stored：外部可见性在这一刻建立

    if (enable_oplog_ && ordered_oplog_writer_) { // oplog 双开关：未启用或未注入 writer 时整段跳过
        std::string payload = SerializeMetadataForOpLog(metadata); // 把元数据序列化成 oplog 负载
        auto result = AppendOpLogVisibleBeforeDurable(
            OpType::PUT_END, object_id.tenant_id.value(), key, payload); // 先写可见、后写持久；返回 expected
        if (!result) {
            LOG(WARNING) << "PutEnd: OpLog queue failed for key=" << key
                         << ", err=" << static_cast<int>(result.error()); // 只告警：oplog 失败不影响 Put 的成功语义
        }
    }
    return {};
}
```

**这一段在做什么**

- 所有副本 COMPLETE 后把对象从 processing 集合摘掉；否则对象会永远显示为"在途写"。
- 同步内存统计、清空读租约、发布 `KV stored` 事件。
- 可选写 oplog；失败只打 WARNING。
- 返回成功——注意此时 offload 只是"已排入某 client 的邮箱"，数据尚未落 NVMe。

**重点语法 / 惯用法**

- `metadata.AllReplicas(&Replica::fn_is_completed)`：成员函数指针作谓词，表达"全部满足"。
- `std::chrono::milliseconds::zero()` 作为"不授予/不延长"的哨兵值，语义靠调用点约定。
- `enable_oplog_ && ordered_oplog_writer_`：配置开关 + 依赖注入指针的双重判断。
- `static_cast<int>(result.error())`：把强类型 `ErrorCode` 转成整数写日志。
- `AppendOpLogVisibleBeforeDurable` 的命名直接说明了它的顺序契约：先可见、后持久。

**与 6.7.3 的差异**

- 图中第 17 步才出现 `lease_ttl_ms`，代码在这里先把租约置零；真正的读租约是读路径 `BatchGetReplicaList` 授予的，与 offload buffer 的 `gc_ttl_ms` 是两套机制。
- 图中没有 processing 集合的处理；缺了这一步，对象会被后续请求一直判为在途写。
- 图中没有 oplog 与 `PublishKvStored` 两个副作用，它们不影响时序图上的数据流，但影响故障恢复与外部可见性。
- 图中第 6 步 `Put 返回` 前似乎"已排好队"，代码显示第 5785 块的入队可能整体失败且不报错；此时仍会执行到 `return {}`。
- 图中把"尚未落盘"归因于第 12–13 步的 StoreBatch，代码说明 PutEnd 阶段连入队都只是"尽力而为"。

### 2.5 PushOffloadingQueue：从副本找到 owner 与邮箱（第 5 步）

这个函数是"把 key 投进某个 owner client 的 offload 邮箱"的唯一入口。它同时被 PutEnd（第 5 步，非淘汰模式）和淘汰路径 `try_evict_or_offload`（`offload_on_evict_` 模式）调用；函数只负责解析 owner、校验可读性、投递任务，不负责 pin 副本——`inc_refcnt` 由调用方做。

#### 2.5.1 前置校验：segment、可读 descriptor 与 liveness

mooncake-store/src/master_service.cpp:9161-9185

```cpp
tl::expected<void, ErrorCode> MasterService::PushOffloadingQueue(
    const ObjectIdentity& object_id, Replica& replica, // object_id 提供 tenant + user_key；replica 是待 offload 的源副本
    std::vector<UUID>* mirror_clients) { // 可选出参：成功入队的 owner client 逐个追加；调用方可以传 nullptr
    const auto& segment_names = replica.get_segment_names(); // 返回 vector<optional<string>> 的临时量，const 引用绑定延长其生命周期（无额外拷贝）
    // No source segment names means there is no usable source segment to
    // offload from. Returning a silent {} here caused the caller to record an
    // OffloadingTask + inc_refcnt for work that was never enqueued, leaking the
    // refcount until TTL expiry (issue #2997). Return an explicit
    // UNABLE_OFFLOADING so callers treat this as a no-op rather than a success.
    if (segment_names.empty() || // 没有任何 segment 名：无从判断 owner
        std::none_of(segment_names.begin(), segment_names.end(),
                     [](const auto& name) { return name.has_value(); })) { // 全部为 nullopt：等价于没有可用 segment
        return tl::make_unexpected(ErrorCode::UNABLE_OFFLOADING); // 显式失败；绝不能返回成功让调用方记 task + inc_refcnt
    }
    Replica::Descriptor source_descriptor; // 下面用于取内存 buffer 的 size
    if (!TryGetReadableReplicaDescriptor(replica, source_descriptor) || // 副本必须"可读"（COMPLETE + available）且 descriptor 可用
        !source_descriptor.is_memory_replica()) { // offload 只从内存读，非 MEMORY 直接拒绝
        return tl::make_unexpected(ErrorCode::UNAVAILABLE_IN_CURRENT_STATUS); // 瞬时状态不满足：调用方当作 no-op，不重试也不记任务
    }
    bool any_enqueued = false; // 是否至少成功入队一次；用于最后的防御性兜底
    std::string client_id_str; // 仅用于最后的 metric 打标签；循环里会被覆盖成"最后一个成功 client"
    const auto liveness = replica.getClientLiveness(); // 副本绑定的客户端存活句柄（共享所有权）；持有它保证 owner 记录不被并发摘除
    if (!liveness) { // 副本尚未绑定或绑定已失效
        return tl::make_unexpected(ErrorCode::UNAVAILABLE_IN_CURRENT_STATUS);
    }
```

**这一段在做什么**

- 契约：`object_id` 提供 tenant/key，`replica` 是源，`mirror_clients` 是可选回填列表。
- 第一道门：`segment_names` 为空或全 `nullopt` 时返回 `UNABLE_OFFLOADING`，否则调用方会记下任务并 `inc_refcnt`，而实际什么都没入队（#2997 的 refcount 泄漏）。
- 第二道门：必须能取到可读 descriptor，且它是 MEMORY 副本。
- 第三道门：必须能拿到副本绑定的 client liveness 句柄。
- 三道门都只返回 `expected` 错误，不改任何状态。

**重点语法 / 惯用法**

- `std::vector<std::optional<std::string>>`：`nullopt` 表示"这一份数据没有归属 segment"。
- `std::none_of(begin, end, lambda)`：判断"没有任何一个有值"，比手写循环更直观。
- `const auto&` 绑定函数返回的临时 `vector`：生命周期延长到引用作用域结束，避免一次拷贝。
- `TryGetReadableReplicaDescriptor` 内部是 `IsReplicaReadable(replica) && getDescriptorIfAvailable(...)`，两段短路。
- `tl::make_unexpected(ErrorCode::UNABLE_OFFLOADING)` 与 `UNAVAILABLE_IN_CURRENT_STATUS` 的区别：前者表示"这份源无法 offload"，后者表示"此刻状态不满足"。

**与 6.7.3 的差异**

- 图中第 5 步只有一次"PushOffloadingQueue"的语义动作，看不出它可能在这些前置条件上直接失败。
- 图中未体现"源副本没有 segment"的失败分支，而这正是曾经导致 refcnt 泄漏的坑。
- 图中第 5 步的成败对 Put 没有可见影响，代码用 `UNABLE_OFFLOADING` 把失败降级为 no-op。
- 图中"第一个本机内存副本"的"本机"由 segment owner 决定；这里只校验副本可读且是 MEMORY，owner 解析放在下一段。

#### 2.5.2 逐 segment 解析 owner、TOCTOU 复核与投递

mooncake-store/src/master_service.cpp:9186-9246

```cpp
    for (const auto& segment_name_it : segment_names) { // 逐 segment 处理；对 MEMORY 副本 get_segment_names() 只返回 0/1 个元素，循环通常只跑一次
        if (!segment_name_it.has_value()) {
            continue; // 空 segment 名跳过，不影响其它 segment 入队
        }
        std::optional<UUID> client_id;
        {
            auto allocator_access = segment_manager_.getAllocatorAccess(); // 取分配器访问器；内层花括号刻意缩短其生命周期（可能持锁）
            client_id =
                allocator_access.GetOwnerClientId(segment_name_it.value()); // segment → 当前 owner client
        }
        if (!client_id) {
            return tl::make_unexpected(ErrorCode::SEGMENT_NOT_FOUND); // 找不到 owner：segment 可能已卸载
        }
        // Callers hold the source metadata shard. This serving observation is
        // the admission point; terminal cleanup must acquire the same shard
        // before it can remove the source or unregister the destination.
        if (!liveness->IsServing()) { // 准入点：client 已进入退出流程就不再投递，避免给正在下线的 owner 排队
            return tl::make_unexpected(
                ErrorCode::UNAVAILABLE_IN_CURRENT_STATUS);
        }
        auto allocator_access = segment_manager_.getAllocatorAccess();
        const auto current_owner =
            allocator_access.GetOwnerClientId(segment_name_it.value()); // 第二次读取 owner
        if (!current_owner || *current_owner != *client_id || // TOCTOU 复核：两次 owner 必须一致
            !TryGetReadableReplicaDescriptor(replica, source_descriptor)) { // 并重新确认副本此刻仍可读（descriptor 可能已失效）
            return tl::make_unexpected(
                ErrorCode::UNAVAILABLE_IN_CURRENT_STATUS); // 期间发生了 remount/迁移/失效，本次放弃
        }
        const int64_t size =
            source_descriptor.get_memory_descriptor().buffer_descriptor.size_; // 从内存 descriptor 取对象字节数，随任务下发给 worker
        auto err = local_ssd_manager_.EnqueueOffload(
            *client_id,
            OffloadTaskItem{.tenant_id = object_id.tenant_id.value(), // 指定初始化器（C++20）：构造投递项
                            .key = object_id.user_key,
                            .size = size},
            offloading_queue_limit_); // 队列上限，用于限流；超出返回 KEYS_ULTRA_LIMIT
        if (err == ErrorCode::SEGMENT_NOT_FOUND) { // 目标 client 没有可用的本地 SSD segment
            return tl::make_unexpected(ErrorCode::UNABLE_OFFLOADING); // 归一化成"这份源不可 offload"，让调用方当 no-op
        }
        if (err != ErrorCode::OK) { // 其它错误：队列满、参数错误等
            if (err == ErrorCode::KEYS_ULTRA_LIMIT) { // 队列达到上限
                MasterMetricManager::instance().inc_offload_enqueue_rejected(
                    UuidToString(*client_id)); // 按 client 打点：被拒绝一次
            }
            return tl::make_unexpected(err); // 原样返回，调用方据此放弃本次入队
        }
        if (mirror_clients != nullptr) { // 可选出参：调用方不关心镜像时可以传 nullptr
            mirror_clients->push_back(*client_id); // 记录这个 owner 的邮箱里已经有本 key 的待办
        }
        client_id_str = UuidToString(*client_id); // 覆盖式记录，最后 metric 只带最后一个成功 client
        any_enqueued = true; // 至少有一个 segment 投递成功
    }
    // Every segment name was nullopt (or EnqueueOffload found no usable
    // segment), so nothing was enqueued. Same invariant as the empty case:
    // never report success when no task was actually submitted.
    if (!any_enqueued) { // 防御性兜底：前面已拦掉"全 nullopt"，但 EnqueueOffload 也可能没真正接单
        return tl::make_unexpected(ErrorCode::UNABLE_OFFLOADING); // 绝不"没入队却报成功"
    }
    MasterMetricManager::instance().inc_offload_enqueued(client_id_str); // 成功计数（只按最后一个 client 打标签）
    return {};
}
```

**这一段在做什么**

- 逐 segment 解析 owner client，拿不到就 `SEGMENT_NOT_FOUND`。
- 用 `liveness->IsServing()` 做准入，随后二次读取 owner 并与第一次比对，同时重新取 descriptor——典型的 TOCTOU 复核。
- 从内存 descriptor 取 `size`，构造 `OffloadTaskItem` 投递到该 client 的队列，并受 `offloading_queue_limit_` 限流。
- 错误映射：`SEGMENT_NOT_FOUND` → `UNABLE_OFFLOADING`；`KEYS_ULTRA_LIMIT` 记 `inc_offload_enqueue_rejected`；其它错误原样返回。
- 成功的 client 追加进 `mirror_clients`，并置 `any_enqueued`；最后按 client 打点 `inc_offload_enqueued`。

**重点语法 / 惯用法**

- 内层 `{ auto allocator_access = ...; }` 花括号：显式缩短可能持锁的访问器生命周期，避免跨调用持锁。
- `std::optional<UUID>` + `if (!client_id)` / `*client_id`：optional 的布尔转换与解引用惯用法。
- C++20 指定初始化器 `OffloadTaskItem{.tenant_id = ..., .key = ..., .size = ...}`：字段名显式，避免位置错配。
- 该函数返回的是 `ErrorCode` 而不是 `expected`；`err == ErrorCode::SEGMENT_NOT_FOUND`、`err != ErrorCode::OK` 是它的判断方式。
- `mirror_clients != nullptr` 的可选出参模式：调用方不关心镜像时无需构造容器。
- `any_enqueued` 是"绝不假成功"的不变量守卫，与开头 `segment_names.empty()` 的检查对称。

**与 6.7.3 的差异**

- 图中第 7 步心跳之后 master 才回 `OffloadTaskItem[]`；真正生产队列项的正是这里的 `EnqueueOffload`，图把生产者省略了。
- 图中第 11 步"取第一个本机内存副本组成 Slice"发生在 FS 侧；master 侧在这里负责按 segment owner 把任务分发到不同 client 的邮箱。
- 图中没有队列满/限流分支；代码里 `KEYS_ULTRA_LIMIT` 会打点后放弃，Put 的成功语义不受影响。
- 图中没有 mirror 概念；`PutEnd` 对每个 completed MEMORY 副本各调用一次本函数，副本分布在不同 client 上，`mirror_clients` 因此会累积多个 client，取消时要逐个撤（`CancelQueuedOffloadTask`）。
- 图中未体现 liveness 与 owner 二次校验；这是与 terminal cleanup（卸载 segment、删除元数据）竞争时的关键防线。
- 图中第 5 步只画了"入队成功"的路径；代码里任何一次失败都直接 `return`，不会"部分成功后继续尝试剩余 segment"——第一个失败就整体放弃。
- 入队失败不影响 Put，是因为调用方 `PutEnd` 丢弃了 `result`，且 `PutEnd` 在第 5811 行之后没有任何依赖入队结果的返回分支；`source_id` 未被赋值时连 `offloading_task` 都不会创建。

## 3. owner 侧：心跳 → 落盘 → 登记（第 7–15 步）

进入第 7 步，主角从写者切到**持有 LOCAL_DISK segment 的 owner 进程**（`FileStorage` 所在的 storage client）。这一节的代码回答三个问题：任务怎么被取到；被写去哪里；写完怎么让 master 知道。

### 3.1 FileStorage::Heartbeat 与 Master 侧的取任务（第 7–8 步）

第 7–8 步不是一个"offload 线程"，而是 `FileStorage` 里一个周期线程调用 `Heartbeat()`，由它发出 `OffloadObjectHeartbeat` RPC。先看这个线程长什么样。

mooncake-store/src/file_storage.cpp:213-223

```cpp
    heartbeat_running_.store(true);                              // 启动开关置位；析构时置 false，循环自然退出
    heartbeat_thread_ = std::thread([this]() {                   // 独立后台线程，与 RPC 处理线程解耦
        LOG(INFO) << "Starting periodic task with interval: "
                  << config_.heartbeat_interval_seconds
                  << "s, running is: " << heartbeat_running_.load();
        while (heartbeat_running_.load()) {                      // 轮询式循环：没有条件变量，无法被立即唤醒
            Heartbeat();                                         // 一次 tick 的全部工作；返回值被丢弃 → 单次失败不会终止线程
            std::this_thread::sleep_for(                          // 固定间隔睡眠，不扣除 Heartbeat 自身耗时
                std::chrono::seconds(config_.heartbeat_interval_seconds));
        }
    });
```

**这一段在做什么**

- 在 `Init()` 末尾启动一条名为"periodic task"的后台线程，无限循环调用 `Heartbeat()`。
- 每个 tick 之间固定睡 `config_.heartbeat_interval_seconds` 秒；真实周期 = 睡眠 + 一次 tick 的耗时。
- `heartbeat_running_` 是 `std::atomic<bool>`，作为线程的生命开关。
- 注意 `Heartbeat()` 的 `tl::expected` 返回值在这里被**整个丢弃**：错误只在该函数内部打日志。
- 这一行 `while` 也是第 7 步"周期性发起"的唯一来源，图上的箭头只画了一次。

**重点语法 / 惯用法**

- `std::thread(lambda)` + `[this]` 捕获：成员函数里起线程的常见写法，生命周期由类负责（析构时 `join`）。
- `std::atomic<bool>::load()/store()` 无锁读写，避免为开关再加一把 mutex。
- `std::chrono::seconds(...)` 显式时长类型，配合 `sleep_for` 防止"裸整数秒"误用。
- `while (flag.load())` 而不是 `while (flag)`：写明了这是原子读，也在读代码时提示并发。
- 没有 `try/catch`：说明 `Heartbeat` 内部约定不抛异常，错误一律走 `tl::expected`。

**与 6.7.3 的差异**

- 图上第 7 步只出现一次；实际是按 `heartbeat_interval_seconds` **无限重复**的周期动作。
- 图上 7→8 紧邻；实际两次心跳之间还夹着 `PollRemoveAll`、promotion、磁盘水位淘汰等工作（见下文 786-811 块）。
- 图没有"失败不致命"的语义：这里 tick 返回错误只是本 tick 结束，线程继续下一轮。
- 图隐含 owner 一直在服务；实际 `draining_` 置位后 tick 直接返回，第 7 步根本不会发出（见下一个块）。

继续之前，需要先确认第 7 步那个 `enable_offloading_` 参数从哪来——它不是每次现算的。

mooncake-store/src/file_storage.cpp:142-171

```cpp
    if (config_.enable_dfs) {                                    // DFS 模式：完全不走本地盘 offload 路径
        client_buffer_gc_running_.store(true);
        client_buffer_gc_thread_ =
            std::thread(&FileStorage::ClientBufferGCThreadFunc, this);
        return {};                                               // 提前返回 → 下面既不读 enable_offloading_ 也不启动心跳线程
    }
    auto enable_offloading_result = IsEnableOffloading();        // 问存储后端"你是否支持 offload"
    if (enable_offloading_result.has_value()) {
        LOG(INFO) << "IsEnableOffloading result: "
                  << (enable_offloading_result.value() ? "true" : "false");
    } else {
        LOG(INFO) << "IsEnableOffloading result: error: "
                  << enable_offloading_result.error();
    }
    if (!enable_offloading_result) {                             // 查询失败按致命错误处理，Init 失败
        LOG(ERROR) << "Failed to get enable persist result, error : "
                   << enable_offloading_result.error();
        return tl::make_unexpected(enable_offloading_result.error());
    }
    {
        MutexLocker locker(&offloading_mutex_);                  // 与 Heartbeat/Drain 共用同一把锁
        enable_offloading_ = enable_offloading_result.value();   // 一次性求值并缓存（latch），此后由心跳原样带给 master
        auto mount_file_storage_result =
            client_->MountLocalDiskSegment(enable_offloading_);  // 真正的"挂载"发生在 Init，而不是第 7 步
        if (!mount_file_storage_result) {
            LOG(ERROR) << "Failed to mount file storage: "
                       << mount_file_storage_result.error();
            return mount_file_storage_result;
        }
    }
```

**这一段在做什么**

- DFS 模式直接走人：不查询 offload 能力、不挂载 LOCAL_DISK、也不启动心跳线程（所以 `enable_offloading_` 未初始化也不会被读到）。
- 否则调用 `IsEnableOffloading()`（其实现 649-660 只是转发 `storage_backend_->IsEnableOffloading()`），把结果当成硬错误处理。
- 在 `offloading_mutex_` 保护下把结果写进成员 `enable_offloading_`，并在同一次锁内完成 `MountLocalDiskSegment`。
- 从此 `enable_offloading_` 是一个"缓存值"：心跳不再重算，只是把它作为参数传出去。
- 运行期唯一会改它的地方是 `OffloadObjects` 遇到 `KEYS_ULTRA_LIMIT`（file_storage.cpp:532-536）。

**重点语法 / 惯用法**

- `tl::expected<bool, ErrorCode>` 同时表达"值/错误"；`has_value()` 与 `if (!expected)` 是两种等价判法。
- 三元 `x.value() ? "true" : "false"` 直接喂给流，避免 `<< bool` 打出 0/1。
- `MutexLocker locker(&offloading_mutex_)` 是项目自定义的 RAII 锁，作用域结束自动解锁。
- 成员声明为 `bool GUARDED_BY(offloading_mutex_) enable_offloading_;`（file_storage.h:243），`GUARDED_BY` 是 clang thread-safety 注解，编译期检查"只在持锁时访问"。
- `return tl::make_unexpected(code);` 与 `return {};` 分别是失败/成功的两个惯用返回形状。

**与 6.7.3 的差异**

- 图上第 7 步的参数写成一个普通实参；实际它是 Init 时求值一次的**缓存**，不是心跳时实时查询的。
- 图上没有"挂载"这一步：LOCAL_DISK segment 在 Init 就挂好了，第 7 步只是取任务。
- 图上第 8 步的 master 一定认识这个 client；实际若 master 重启丢过 segment，会先返回 `SEGMENT_NOT_FOUND`（见 717-777 块）。
- 图没有画出 `enable_offloading_` 运行期会被改成 `false`（磁盘 key 数超限时），此时第 7 步传 false、master 回空数组，后续再无新数据落盘。
- 图还默认 DFS 与 NVMe KV 是同一条路径；代码里 DFS 在 142-147 处就分流了。

下面进入 `Heartbeat` 本体。

mooncake-store/src/file_storage.cpp:662-716

```cpp
tl::expected<void, ErrorCode> FileStorage::Heartbeat() {         // 用 expected 报错，而不是异常或返回码
    if (client_ == nullptr) {                                    // 防御：client_ 可能尚未注入/已失效
        LOG(ERROR) << "client is nullptr";
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);   // 构造失败分支
    }

    // A drain deregistered the disk tier on purpose. Skipping the whole tick
    // matters for the SEGMENT_NOT_FOUND branch below, which would re-mount the
    // segment and re-register every object; there is also no point taking new
    // offload work for a store that is leaving. The client TTL is renewed by
    // Ping on the storage heartbeat thread, not here, so the client stays
    // alive for its memory segments.
    // ↑ 原文注释：drain 是主动下线，跳过整个 tick 是为了防止下面的 SEGMENT_NOT_FOUND 分支把刚注销的 segment 又挂回来
    if (draining_.load()) {                                      // 快路径：无锁原子读
        return {};                                               // 空 expected = 成功；本 tick 什么都不做
    }

    // Join previous rescan if completed
    // ↑ 原文注释：上一次后台 rescan 已完成就回收 future
    if (rescan_future_.valid() && rescan_future_.wait_for(std::chrono::seconds(
                                      0)) == std::future_status::ready) {  // wait_for(0s) == 非阻塞探测
        rescan_future_ = std::future<void>();                    // 重置为无效 future，释放共享状态
    }

    // Retry metadata resync if previous attempt failed.
    // ↑ 原文注释：上一轮 rescan 失败过就重试
    if (metadata_resync_pending_.load() && !rescan_future_.valid()) {  // 有两个条件：需要重试 且 当前没有在跑的
        LOG(INFO) << "Retrying background metadata rescan";
        rescan_future_ = std::async(std::launch::async, [this]() {     // 起后台任务，不阻塞本次心跳
            auto result = ReRegisterOffloadedObjects();          // 扫盘 + NotifyOffloadSuccess 重新登记
            if (!result) {
                LOG(ERROR) << "Background metadata rescan retry "
                           << "failed: " << result.error();      // 失败：保留 pending 标志，下个 tick 再来
            } else {
                metadata_resync_pending_.store(false);           // 成功才清标志；这个写发生在异步线程里
            }
        });
    }

    std::vector<OffloadTaskItem>                                 // 第 8 步的承载容器：tenant_id/key/size
        offloading_objects;  // Objects selected for offloading   // 原注释：被选中要落盘的对象

    // === STEP 1: Send heartbeat and get offloading decisions ===
    // ↑ 原文注释：第一步，发心跳、拿"该落哪些对象"的决策
    {
        MutexLocker locker(&offloading_mutex_);                  // 与 Drain 互斥；锁一直持有到本块结束（777 行）
        // Re-checked under the lock: the entry check above is only a fast
        // path. A drain (which latches and deregisters under this same
        // mutex) may have completed while this tick was parked on it; going
        // on would hit SEGMENT_NOT_FOUND below and re-mount the segment the
        // drain just deregistered.
        // ↑ 原文注释：入口那次 draining_ 检查只是快路径，这里必须在锁内再查一次（双检）
        if (draining_.load()) {                                  // 持锁复查：drain 可能刚在本 tick 等锁期间完成
            return {};                                           // 直接放弃本 tick，绝不重挂 segment
        }
        auto fetch_offload_tasks = [&]() -> tl::expected<void, ErrorCode> {  // 局部 lambda：把"发心跳"封装一次，供重试复用
            return client_->OffloadObjectHeartbeat(enable_offloading_,       // 第 7 步：带上缓存的开关
                                                   offloading_objects);      // 第 8 步：结果写进出参（out-param）
        };                                                       // 返回 expected，错误码透传
        auto heartbeat_result = fetch_offload_tasks();           // 第一次发心跳（正常情况下只发这一次）
```

**这一段在做什么**

- 空 `client_` 直接判参错误；这是唯一一个"调用方写错了"的错误码。
- `draining_` 快路径：已经开始 drain 就整 tick 跳过——既不发心跳，也不去碰下面的重挂分支。
- 处理上一轮后台 `rescan`：已就绪就回收 future；若上次失败（`metadata_resync_pending_`）则在本 tick 重新拉起一个。
- 进入 STEP 1：拿 `offloading_mutex_`，**再查一次** `draining_`（双检），然后定义 `fetch_offload_tasks` lambda 并调用它。
- 第 7 步的 RPC 真正发生在 `client_->OffloadObjectHeartbeat(...)` 这一行；第 8 步的数据落进 `offloading_objects`。
- 锁在本块结束处（同一作用域）才释放，因此"发 RPC + 后续重挂 + 重试"整体与 drain 串行。

**重点语法 / 惯用法**

- `tl::expected<void, ErrorCode>`：`void` 特化表示"只关心成败"；成功写 `return {};`。
- 双检锁（double-checked under lock）：无锁快路径 + 持锁复检，配合 `std::atomic<bool>`。这里两次检查**都是原子的**，不需要内存屏障写法。
- `auto fetch_offload_tasks = [&]() -> tl::expected<void, ErrorCode> { ... };`：局部 lambda 带显式尾置返回类型，捕获引用以便写 `offloading_objects`。
- 出参 `std::vector<OffloadTaskItem>&` 而不是返回 vector：与 `tl::expected` 只能有一个返回值的约束配合。
- `std::future::valid()` + `wait_for(0s) == std::future_status::ready` 是"非阻塞 join"惯用法。
- `std::async(std::launch::async, ...)` 显式要求新线程，避免实现选择延迟执行。
- `metadata_resync_pending_` 是"失败后重试"的粘滞标志，只有成功才被清掉。

**与 6.7.3 的差异**

- 图上第 7 步是一次干净的服务调用；实际它被"检查 drain → 回收 rescan → 重试 rescan → 再查 drain"四段前置逻辑包着。
- 图上没有 `draining_`：drain 之后第 7 步不再发生，图里 owner 永远在线。
- 图上第 7 步的箭头只画一根；代码里这个 lambda 在 `SEGMENT_NOT_FOUND` 恢复后还会被**再调用一次**（743 行）。
- 图上第 8 步的返回值只包含任务；实际"没任务"与"segment 丢了"是两种不同结果，后者走 717 起的恢复分支。
- 图上没有 rescan 这条后台线：它才是 master 重启后把磁盘上已有对象重新登记回去的机制。

mooncake-store/src/file_storage.cpp:717-777

```cpp
        if (!heartbeat_result) {                                 // RPC 层面的失败（网络/错误码）
            ErrorCode err = heartbeat_result.error();            // 先取出错误码，后面按值分支
            if (err == ErrorCode::SEGMENT_NOT_FOUND) {           // master 不认识这个 client 的 LOCAL_DISK segment
                // Master lost our LOCAL_DISK segment (likely restarted).
                // Re-register the segment, retry the heartbeat, and
                // trigger async ScanMeta to re-register object metadata.
                // ↑ 原文注释：master 重启丢了 segment → 重挂、重试心跳、异步重扫元数据
                LOG(WARNING) << "OffloadObjectHeartbeat returned "
                             << "SEGMENT_NOT_FOUND, attempting to "
                             << "re-register local disk segment and "
                             << "re-register object metadata";
                auto remount_result =
                    client_->MountLocalDiskSegment(enable_offloading_);  // 用同一个开关原样重挂，语义与 Init 一致
                if (remount_result) {                            // 重挂成功
                    // Report configured SSD capacity so the Master can
                    // restore file_total_capacity_ (the denominator in
                    // "SSD Storage: X / Y").  This was lost on restart;
                    // re-reporting it here avoids the 0 B display.
                    // ↑ 原文注释：master 重启丢了容量，这里补报，否则面板显示 0 B
                    if (config_.total_size_limit > 0) {          // 0 表示"未配置"，不上报
                        auto cap_result = client_->ReportSsdCapacity(
                            config_.total_size_limit);           // 再发一次 RPC：上报 SSD 总容量
                        if (!cap_result) {
                            LOG(WARNING)                         // 只告警：容量显示错不影响落盘正确性
                                << "ReportSsdCapacity failed during "
                                << "heartbeat recovery: " << cap_result.error();
                        }
                    }
                    heartbeat_result = fetch_offload_tasks();    // 重挂后重试第 7 步；此时 master 应能认出我们
                    if (!heartbeat_result) {                     // 重试仍失败 → 本轮放弃
                        LOG(ERROR) << "Heartbeat failed after re-registration: "
                                   << heartbeat_result.error();
                        return heartbeat_result;                 // 把同一个 expected 上抛
                    }
                    // Master lost all object metadata on restart.
                    // Trigger async ScanMeta to re-register them,
                    // same as what Init() does on startup.
                    // ↑ 原文注释：master 重启后对象元数据也丢了，异步扫盘重新登记
                    if (!rescan_future_.valid()) {               // 同一时刻只允许一个 rescan
                        LOG(INFO) << "Triggering background metadata rescan "
                                  << "after LOCAL_DISK segment re-registration";
                        metadata_resync_pending_.store(true);     // 先置"待重试"，成功后才由异步线程清掉
                        rescan_future_ =
                            std::async(std::launch::async, [this]() {   // 后台线程执行，本 tick 不等它
                                auto result = ReRegisterOffloadedObjects();
                                if (!result) {
                                    LOG(ERROR) << "Background metadata rescan "
                                               << "failed: " << result.error();
                                } else {
                                    metadata_resync_pending_.store(false);
                                }
                            });
                    }
                } else {                                         // 重挂失败：segment 真的回不来了
                    LOG(ERROR) << "Failed to re-register local disk segment: "
                               << remount_result.error();
                    return tl::make_unexpected(remount_result.error());  // 用重挂的错误码，而不是 SEGMENT_NOT_FOUND
                }
            } else {                                             // 其它错误：不尝试自愈
                LOG(ERROR) << "Failed to send heartbeat with error: " << err;
                return heartbeat_result;                         // 原样上抛，本 tick 结束
            }
        }
    }                                                            // 这里才释放 offloading_mutex_
```

**这一段在做什么**

- 第 7 步失败的分类处理：只有 `SEGMENT_NOT_FOUND` 才尝试自愈，其它错误立刻上抛。
- 自愈第一步：`MountLocalDiskSegment(enable_offloading_)` 把 LOCAL_DISK segment 重新挂到 master。
- 自愈第二步：若配置了 `total_size_limit`，补报 SSD 容量（这只影响展示，失败仅告警）。
- 自愈第三步：重试第 7 步，拿到真正的任务列表。
- 自愈第四步：拉起后台 `ReRegisterOffloadedObjects()`（ScanMeta + `NotifyOffloadSuccess`），把磁盘上已有对象登记回 master。
- 重试仍失败，或重挂失败，或遇到其它错误码：本 tick 直接结束。

**重点语法 / 惯用法**

- 复用局部 lambda 完成重试（`fetch_offload_tasks()` 第二次调用），避免复制 RPC 参数。
- `if (remount_result)` / `if (!heartbeat_result)` 直接对 `tl::expected` 取布尔，语义是"有没有值"。
- `return heartbeat_result;` 直接把同名类型的 expected 上抛，不需要重新包装。
- `return tl::make_unexpected(remount_result.error());` 是"换一个错误来源"的写法，注意这里换了错误码来源。
- `if (!rescan_future_.valid())` 做单飞（single-flight）：已有 rescan 在跑就不重复起。
- `LOG(WARNING)` 与 `LOG(ERROR)` 的区分：可恢复/仅影响展示的用 WARNING，影响功能才用 ERROR。
- 整段处于 `MutexLocker locker(&offloading_mutex_)` 作用域内，锁的粒度覆盖了"检查 + 重挂 + 重试"。

**与 6.7.3 的差异**

- 图上完全没有这条恢复路径；第 8 步被画成"必定返回任务数组"。
- 图上 owner 与 master 的心跳是幂等的；实际这条路径会在一次 tick 里额外发出最多三个 RPC（重挂、报容量、重试心跳）。
- 图上没画 rescan：它才是 master 重启后第 16 步还能查到 LOCAL_DISK 副本的前提。
- 图上没有 `enable_offloading_` 被复用为"重挂参数"的细节：重挂时传的仍是 Init 时的缓存值，不是重新探测的结果。
- 图上把错误当异常终止；这里是"能自愈就继续本 tick，不能就结束本 tick"，线程存活不受影响。

mooncake-store/src/file_storage.cpp:786-811

```cpp
    // === STEP 3: Persist offloaded objects (trigger actual data migration) ===
    // ↑ 原文注释：第三步，真正触发数据迁移
    if (!offloading_objects.empty()) {                          // 没取到任务就跳过，避免空 BatchOffload
        auto offload_result = OffloadObjects(offloading_objects);  // 第 12 步：内部按 bucket 调 BatchOffload
        if (!offload_result) {
            LOG(ERROR) << "Failed to persist objects with error: "
                       << offload_result.error();
            return offload_result;                              // 落盘失败 → 本轮 tick 到此为止
        }
    }

    VLOG(1) << "Completed heartbeat with offloaded objects count: "
            << offloading_objects.size();

    // Drive any pending L2->L1 promotion work for this client. Failures
    // inside ProcessPromotionTasks are logged per-key and do not propagate;
    // promotion is best-effort and must never break offload.
    // ↑ 原文注释：promotion 是尽力而为，绝不打断 offload
    (void)ProcessPromotionTasks();                              // 显式丢弃返回值（nodiscard/expected）

    // Proactive disk watermarks keep LOCAL_DISK usage below the configured
    // low watermark even when no new write arrives to trigger reactive
    // eviction.
    // ↑ 原文注释：主动水位淘汰，防止磁盘涨过低水位
    auto disk_eviction_result = RunDiskWatermarkEviction();
    if (!disk_eviction_result) {
        LOG(WARNING) << "Disk watermark eviction failed: "      // 只告警，不 return：淘汰失败不影响心跳语义
                     << disk_eviction_result.error();
    }
    return {};                                                  // 成功结束本 tick
```

**这一段在做什么**

- 只有本 tick 从 master 取到了任务（`offloading_objects` 非空）才调用 `OffloadObjects`，即第 12 步的入口。
- 落盘失败就直接返回：本 tick 结束，不再执行后面的 promotion 与淘汰。
- 落盘成功后，tick 还有两件"顺带"的事：驱动 L2→L1 promotion、执行磁盘水位淘汰。
- promotion 的返回值被显式丢弃；淘汰失败只打 WARNING。
- 最后统一 `return {}`，表示这一轮心跳成功。

**重点语法 / 惯用法**

- `(void)expr;` 显式表达"我知道它有返回值，故意不用"，压制 `[[nodiscard]]`/警告。
- `return offload_result;` 与函数返回类型一致时直接上抛 expected。
- `if (!offloading_objects.empty())` 空容器短路，既是性能也是正确性（空 map 会被 backend 判 `INVALID_KEY`）。
- `VLOG(1)` 是 glog 的 verbose 日志，只在 `--v=1` 时输出。
- 尾部任务用 "log and continue" 而不是 "return error"：区分**关键路径**与**尽力而为路径**。

**与 6.7.3 的差异**

- 图上第 12 步是一条独立箭头；实际它是 `Heartbeat()` 内部同步调用 `OffloadObjects` 的结果，慢 bucket 会拖住整个心跳线程。
- 图上第 14 步 `NotifyOffloadSuccess` 画在第 13 步之后独立出现；实际它在 `OffloadObjects` 里的 per-bucket completion 回调（file_storage.cpp:342-367）中执行，**按 bucket 多次调用**，并不在这个函数体里。
- 图上 tick 只包含 7/8/12/14；实际末尾还有 promotion 与磁盘水位淘汰，图上没有。
- 图上假设落盘成功；这里失败会让本 tick 提前结束，后续 bucket 这一轮不处理（未处理 key 由 NACK/TTL 兜底）。
- 图上没有"没任务也照样心跳"的情况：`offloading_objects` 为空时这一步整段跳过，但 tick 仍算成功。

现在看 master 侧第 8 步的全部逻辑。

mooncake-store/src/master_service.cpp:8887-8929

```cpp
auto MasterService::OffloadObjectHeartbeat(const UUID& client_id,
                                           bool enable_offloading)
    -> tl::expected<std::vector<OffloadTaskItem>, ErrorCode> {  // 尾置返回类型：先写参数再写返回，便于阅读长类型
    const auto record = FindClientRecord(client_id);            // 查 client 存活记录，拿 shared_ptr（可能为空）
    auto serving_guard =
        record ? record->TryAcquireServingGuard() : std::nullopt;  // 三元 + nullopt：record 为空时短路，guard 为 optional
    if (!serving_guard) {                                       // 找不到 client，或它正处于 drain/下线状态
        return tl::make_unexpected(ErrorCode::UNAVAILABLE_IN_CURRENT_STATUS);  // 统一成一个"现在不能服务"
    }
    std::shared_lock<std::shared_mutex> shared_lock(master_state_mutex_);  // 读锁：元数据表只读访问
    auto pending = local_ssd_manager_.SetOffloadingAndTakePending(
        client_id, enable_offloading);                          // 关键：写入开关 + 取走并清空该 client 的待落盘队列
    if (!pending) {                                             // client 的 LOCAL_DISK segment 已不存在
        LOG(ERROR) << "Local disk segment not found with client id = "
                   << client_id;
        return tl::make_unexpected(pending.error());          // 通常是 SEGMENT_NOT_FOUND，触发客户端重挂
    }
    if (enable_offloading) {                                    // 开启 offload：任务就是返回值（第 8 步）
        return std::move(*pending);                             // 从 optional 里移动 vector，避免整份拷贝
    }
    // Offloading is disabled: clear the pending queue to prevent unbounded
    // growth that would trigger KEYS_ULTRA_LIMIT in PushOffloadingQueue. We
    // must also clean up the corresponding offloading tasks and decrement
    // source replica refcounts to avoid resource leaks and blocked writes
    // (OBJECT_HAS_REPLICATION_TASK).
    // ↑ 原文注释：关闭时也要清队列，否则会涨到 KEYS_ULTRA_LIMIT；同时要把第 5 步 inc 的 refcnt 还回去
    for (auto& task : *pending) {                               // 逐个任务做"撤销"
        const auto object_id =
            MakeObjectIdentity(task.key, TenantId(task.tenant_id));  // 对象身份 = tenant + key，不能只用 key
        MetadataAccessorRW accessor(this, object_id);           // RW accessor：拿该对象的写权限（分片锁）
        if (accessor.Exists()) {                                // 对象可能已被淘汰/删除
            auto& offloading_task = accessor.GetState().offloading_task;  // 对象上的"正在进行中的 offload"状态
            if (offloading_task) {                              // optional 转 bool
                auto source =
                    accessor.Get().GetReplicaByID(offloading_task->source_id);  // 按 id 找回第 5 步选中的源副本
                if (source) {
                    source->dec_refcnt();                       // 撤销 inc_refcnt，解除对源副本的钉住
                }
                offloading_task.reset();                        // 清掉任务状态，writes 不再被 OBJECT_HAS_REPLICATION_TASK 阻塞
            }
        }
    }
    return std::vector<OffloadTaskItem>{};                     // 关闭态统一返回空数组，客户端不会落盘
}
```

**这一段在做什么**

- 先做 client 级门禁：记录不存在、或拿不到 serving guard（drain 中/已下线），一律 `UNAVAILABLE_IN_CURRENT_STATUS`。
- 取 `master_state_mutex_` 读锁后，调用 mailbox 的 `SetOffloadingAndTakePending`：**写入本次开关**、**取走全部待办并清空**。
- client 的 LOCAL_DISK segment 不在（例如已卸载）→ 返回错误（常见是 `SEGMENT_NOT_FOUND`），触发客户端的重挂分支。
- 开关为 true：任务原样返回，这就是第 8 步的 `OffloadTaskItem[]`。
- 开关为 false：任务被"就地撤销"——找回对象的 `offloading_task`、对 source 副本 `dec_refcnt()`、再 `reset()` 状态。
- 最后返回空数组，客户端因此不会执行第 12 步。

**重点语法 / 惯用法**

- 尾置返回类型 `auto f(...) -> tl::expected<...>`，与项目里其它 RPC 保持一致。
- `record ? record->TryAcquireServingGuard() : std::nullopt`：三元表达式两边都归一到 optional，`if (!serving_guard)` 一处判空。
- serving guard 是 RAII 且**自带一把 `transition_mutex_`**（client_liveness.h:82-89）：只有状态为 `ACTIVE` 才返回 guard，guard 活着期间锁不释放，等于把该 client 的状态机钉在 ACTIVE，直到本次 RPC 结束。
- `std::shared_lock<std::shared_mutex>` 读锁，配合后面的 `MetadataAccessorRW`（写访问器）分两级保护。
- `std::move(*pending)`：从 `tl::expected<T,...>` 解引用出的 vector 直接移动。
- `MakeObjectIdentity(task.key, TenantId(task.tenant_id))` 构造 tenant-scoped 身份，避免不同租户同名 key 命中同一对象。
- `auto& offloading_task = ...; if (offloading_task)` 用 optional 的布尔语义判"有没有在途任务"。

**与 6.7.3 的差异**

- 图上第 8 步只是"返回数组"；实际 disable 时 master 会**先清空队列并归还 refcnt**，返回空数组——第 15 步的 `dec_refcnt` 在这条路径上提前发生了。
- 图上没有 client 门禁：drain 中或已下线的 client 拿不到任务，会收到 `UNAVAILABLE_IN_CURRENT_STATUS`。
- 图上没有"取走即清空"的后果：任务从此只在 owner 内存里，master 不会重发；owner 崩溃只能等 `put_start_release_timeout_sec_` 的 TTL 清理。
- 图上第 5 步入队像是必然成功；实际入队受 mailbox 的 `enable_offloading_`（上一次心跳写入的值）与队列上限约束，可能静默失败（local_ssd/manager.cpp:13-27）。
- 图上 `enable_offloading` 只是"要不要给任务"的开关；在 master 端它还是邮箱的**准入状态**。

最后补齐"取走即清空"的实现细节。

mooncake-store/src/local_ssd/manager.cpp:29-40

```cpp
std::vector<OffloadTaskItem> LocalSsdTaskMailbox::SetOffloadingAndTakePending(
    bool enabled) {
    MutexLocker lock(&mutex_);                                  // 与 EnqueueOffload/RemoveOffload 同一把锁
    enable_offloading_ = enabled;                               // 把本次心跳的开关"落地"，影响之后的入队准入
    std::vector<OffloadTaskItem> tasks;                         // 返回值：本批任务
    tasks.reserve(pending_offloads_.size());                    // 预分配，避免 push 时反复扩容
    for (auto& [_, task] : pending_offloads_) {                 // 结构化绑定；_ 是 tenant-scoped key，这里用不到
        tasks.push_back(std::move(task));                       // 移动而非拷贝（task 里有 std::string）
    }
    pending_offloads_.clear();                                  // 取走即清空：master 侧不再持有这一批任务
    return tasks;                                               // 按值返回，依赖移动/RVO
}
```

**这一段在做什么**

- 持锁把 `enabled` 写入成员 `enable_offloading_`，这是后续 `EnqueueOffload` 的准入判断依据。
- 预留结果容量（等于当前队列长度）。
- 遍历 `pending_offloads_`（一个以 tenant-scoped key 为键的 map），把每个任务移动进结果 vector。
- `clear()` 清空邮箱，实现"取走即清空"。
- 返回这一批任务。

**重点语法 / 惯用法**

- `MutexLocker lock(&mutex_)`：整个"读 + 清 + 写开关"是原子的，避免与入队交错。
- 结构化绑定 `for (auto& [_, task] : pending_offloads_)`，`_` 表示"这个绑定故意不用"。
- `std::move(task)` + `push_back` 的移动语义；`emplace_back` 亦可，但这里的 task 已是左值引用。
- `reserve` 后再 `push_back` 是 C++ 容器的标准优化套路。
- 返回值按值 + 移动，编译器通常直接构造在调用方位置（RVO）。

**与 6.7.3 的差异**

- 图上第 8 步看不到 master 端的状态变化；真实语义是**一次性移交**，同一批 key 不会在下次心跳再出现。
- 图上没有 `enable_offloading_` 在 master 端落地的位置；就是这里的 `enable_offloading_ = enabled;`。
- 图上第 5 步的入队与这里的取走，共用同一把 `mutex_`：入队/取走/禁用是串行化的，图上体现不出这层互斥。
- 图没有"任务丢失"窗口：从 mailbox 清空到 owner 成功调用 `NotifyOffloadSuccess` 之间，只有 owner 内存持有这批任务。

### 3.2 BatchQuerySegmentSlices：先问 master，再取本机地址（第 9–11 步）

这段代码最容易读错的地方是把它当成"本地查表"。它不是：第一步就是一次 master RPC。

先看真实的调用点，它决定了参数形状与失败语义。

mooncake-store/src/file_storage.cpp:391-413

```cpp
        for (const auto& [tenant_id, storage_keys] : storage_keys_by_tenant) {  // 按 tenant 分组：RPC 的租户维度
            std::vector<std::string> user_keys;
            user_keys.reserve(storage_keys.size());
            for (const auto& storage_key : storage_keys) {
                user_keys.push_back(task_by_storage_key.at(storage_key).key);  // storage_key(tenant:key) → 原始 user key
            }
            std::unordered_map<std::string, std::vector<Slice>>
                user_batch_object;                              // 本组 key 的查址结果
            [[maybe_unused]] auto query_result = BatchQuerySegmentSlices(
                user_keys, tenant_id, user_batch_object);       // 第 9–11 步入口；返回值有意忽略
            // BatchQuerySegmentSlices is now best-effort: it always returns
            // OK. Keys present in user_batch_object go to batch_object; the
            // rest are reported as failed.
            // ↑ 原文注释：该函数永远返回 OK；进了 map 的才算查到，没进的按失败处理
            for (const auto& storage_key : storage_keys) {
                const auto& task = task_by_storage_key.at(storage_key);
                auto it = user_batch_object.find(task.key);     // 用 user key 查结果
                if (it != user_batch_object.end()) {
                    batch_object.emplace(storage_key, std::move(it->second));  // 命中：整组 slice 移动进落盘批次
                } else {
                    failed_tasks.push_back(task);               // 未命中：记入失败，稍后 NACK(data_size<0) 回 master
                }
            }
        }
```

**这一段在做什么**

- 落盘前按 `tenant_id` 分组，一个 tenant 调一次 `BatchQuerySegmentSlices`（master 的查询接口是带 tenant 的）。
- 把本地用于落盘的 `storage_key`（`tenant:key` 形式）还原成 master 认识的 user key。
- 调用查址函数，结果写入 `user_batch_object`。
- 返回值被 `[[maybe_unused]]` 显式忽略，因为注释已说明它永远返回 OK——失败靠"key 不在 map 里"表达。
- 未查到的 key 进入 `failed_tasks`，后续用 `data_size=-1` 的 NACK 让 master 清账（见 file_storage.cpp:369-377 的注释）。

**重点语法 / 惯用法**

- `[[maybe_unused]]` 标注"有意不用返回值"，比 `(void)` 更能表达契约。
- `std::unordered_map<std::string, std::vector<Slice>>`：一个对象对应一组 slice 的结果容器。
- `task_by_storage_key.at(key)` 用 `at` 而非 `[]`：越界会抛异常，避免静默插入空元素。
- `auto it = m.find(k); if (it != m.end())` 是 C++ 里"查一次、用一次"的标准写法（比 `count` + `[]` 少一次查找）。
- `std::move(it->second)` 把整组 slice 移出 map，避免深拷贝。

**与 6.7.3 的差异**

- 图上第 9–11 步像是每个 key 一次交互；实际是**按 tenant 批处理**，且 key 形态在调用前被翻译过一次。
- 图上没有失败语义：查不到不是错误，而是"回 NACK 让 master 撤销任务、还 refcnt"。
- 图上第 11 步说"取第一个本机内存副本"；调用方只关心有没有取到，不区分"不在本机"与"对象不存在"。
- 图上省略了 tenant 维度：master 的 `BatchGetReplicaList` 必须带 `tenant_id`，否则查到的是别的租户。

mooncake-store/src/file_storage.cpp:1047-1073

```cpp
tl::expected<void, ErrorCode> FileStorage::BatchQuerySegmentSlices(
    const std::vector<std::string>& keys, const std::string& tenant_id,  // user key 列表 + 租户
    std::unordered_map<std::string, std::vector<Slice>>& batched_slices) {  // 出参：key → slices
    auto batched_query_results = client_->BatchQuery(keys, tenant_id);  // 第 9 步：这是一次 master RPC，不是本地查表
    if (batched_query_results.empty()) {                        // keys 为空（或上游没给数据）时直接成功返回
        return {};                                              // 空结果不算错误
    }
    for (size_t i = 0; i < batched_query_results.size(); ++i) { // 按位置对齐：结果[i] 对应 keys[i]
        if (batched_query_results[i]) {                         // expected 为真 = 这个 key 查到了
            for (const auto& descriptor :
                 batched_query_results[i].value().replicas) {   // 10 步：master 给的全量副本（含远端、含非内存）
                if (client_->IsReplicaOnLocalMemory(descriptor)) {  // 11 步：过滤出"本机内存副本"
                    const auto& memory_descriptor =
                        descriptor.get_memory_descriptor();     // 取 MemoryDescriptor（类型已由上一行保证）
                    std::vector<Slice> slices;                  // 一个对象一组 slice
                    void* slice_ptr = reinterpret_cast<void*>(
                        memory_descriptor.buffer_descriptor.buffer_address_);  // uintptr_t → 本进程指针
                    slices.emplace_back(Slice{
                        slice_ptr, memory_descriptor.buffer_descriptor.size_});  // 只造一个 Slice，不分片
                    batched_slices.insert({keys[i], std::move(slices)});  // 用 user key 作为 map 的键
                    break;                                      // 只取第一个本机副本，命中即停
                }
            }
        }
    }
    return {};                                                  // best-effort：无论查到多少，都返回成功
}
```

**这一段在做什么**

- 第一行就发出 master RPC：`client_->BatchQuery(keys, tenant_id)`，即第 9 步。
- 结果为空直接成功返回；为空通常意味着调用方没给 key。
- 按索引遍历结果向量：结果与输入 `keys` 等长且按位置对齐。
- 对每个成功的 key，遍历 master 返回的**全部**副本，用 `IsReplicaOnLocalMemory` 逐个过滤。
- 第一个命中的本机内存副本：取出 `buffer_address_` 与 `size_`，组装**恰好一个** `Slice` 放进结果 map。
- `break` 表示"只取第一个"，命中后不再看后面的副本。
- 一个都没命中的 key 不会出现在 map 里，由调用方算作失败；函数本身仍返回成功。

**重点语法 / 惯用法**

- `tl::expected<void, ErrorCode>` 的"恒成功"用法：签名保留错误通道，实现上不使用。
- `const auto& descriptor`：`Replica::Descriptor` 是 variant 包装，按引用遍历避免复制。
- `descriptor.get_memory_descriptor()` 内部是 `std::get_if<MemoryDescriptor>`，**类型不符会抛 `std::runtime_error`**；安全性完全依赖前一行 `IsReplicaOnLocalMemory` 已判过 `is_memory_replica()`。
- `reinterpret_cast<void*>(uintptr_t)`：`buffer_address_` 声明为 `uintptr_t`（见 allocator.h:143-150），需要显式转回指针。
- `slices.emplace_back(Slice{ptr, size})`：聚合初始化 + 原地构造，`Slice` 是简单结构体。
- `batched_slices.insert({keys[i], std::move(slices)})`：插入 pair；**若 key 已存在则静默丢弃**新值（这里依赖 key 唯一）。

**与 6.7.3 的差异**

- 图上第 9、10、11 步是三个独立动作；代码里它们全在这一个函数内，master 只负责给全量副本，**不告诉客户端哪个是本机**。
- 图上第 11 步像是一定能取到；实际可能一个副本都不匹配，此时该 key 静默缺席而不是报错。
- 图上第 11 步产出"Slice 列表"；实现里每个对象只产生 **1 个 Slice**，没有 `split_into_slices` 那样的分片（对比迁移路径 client_service.cpp:4387-4392）。
- 图上没有 best-effort 契约：这里永远返回 `{}`，连 `BatchQuery` 的逐 key 错误都被吞掉。
- "零拷贝"只对 host 内存成立：本函数把 master 给的地址直接当本进程可解引用指针；若源是 GPU 指针，会在 3.3 的 `OffloadObjects`（file_storage.cpp:418-453）里先做 D2H 拷进 pinned buffer，那时已经不是零拷贝。

mooncake-store/src/client_service.cpp:1259-1292

```cpp
std::vector<tl::expected<QueryResult, ErrorCode>> Client::BatchQuery(
    const std::vector<std::string>& object_keys, const std::string& tenant_id) {
    std::chrono::steady_clock::time_point start_time =
        std::chrono::steady_clock::now();                       // 单调时钟：后面用"起点 + ttl"算租约截止
    auto response = master_client_.BatchGetReplicaList(object_keys, tenant_id);  // 唯一一次 RPC，就是第 9 步的网络动作

    // Check if we got the expected number of responses
    // ↑ 原文注释：校验返回条数
    if (response.size() != object_keys.size()) {                // 条数不一致说明协议层出错（部分返回）
        LOG(ERROR) << "BatchQuery response size mismatch. Expected: "
                   << object_keys.size() << ", Got: " << response.size();
        // Return vector of RPC_FAIL errors
        // ↑ 原文注释：整批标成 RPC_FAIL
        std::vector<tl::expected<QueryResult, ErrorCode>> results;
        results.reserve(object_keys.size());
        for (size_t i = 0; i < object_keys.size(); ++i) {
            results.emplace_back(tl::unexpected(ErrorCode::RPC_FAIL));  // 逐 key 填失败，保持"等长对齐"契约
        }
        return results;
    }
    std::vector<tl::expected<QueryResult, ErrorCode>> results;
    results.reserve(response.size());                           // 预分配，避免扩容
    for (size_t i = 0; i < response.size(); ++i) {
        if (response[i]) {                                      // 该 key 查询成功
            results.emplace_back(
                tl::in_place, std::move(response[i].value().replicas),  // 移动副本列表，不拷贝
                start_time +
                    std::chrono::milliseconds(response[i].value().lease_ttl_ms),  // 相对 ttl → 绝对截止时间
                response[i].value().object_checksum,            // 校验和原样透传
                response[i].value().resolved_tenant_id);        // master 解析出的租户 id
        } else {
            results.emplace_back(tl::unexpected(response[i].error()));  // 逐 key 错误码透传（如 OBJECT_NOT_FOUND）
        }
    }
    return results;
}
```

**这一段在做什么**

- 记录 `start_time`，用于把回包里的相对 `lease_ttl_ms` 折算成绝对截止时间。
- 发出唯一一次 RPC：`master_client_.BatchGetReplicaList(object_keys, tenant_id)`——这正是时序图第 9 步，也是读者侧第 16 步用的同一个 RPC。
- 条数不一致时，把所有 key 都标成 `RPC_FAIL`，维持"结果与输入等长对齐"的契约。
- 逐条转换：成功项用 `tl::in_place` 原地构造 `QueryResult`（replicas / lease deadline / checksum / resolved_tenant_id）；失败项原样透传错误码。
- 返回等长的 `expected` 向量，交给上游 `BatchQuerySegmentSlices` 按位置对齐使用。

**重点语法 / 惯用法**

- `std::chrono::steady_clock` 是单调时钟，适合算超时；`system_clock` 会被系统时间调整影响。
- `start_time + std::chrono::milliseconds(ttl)` 得到 `time_point`，把"相对时长"转成"绝对截止"。
- `tl::in_place` 是 expected 的原地构造标签：直接在存储区里构造 `QueryResult`，省一次临时对象。
- `std::move(...replicas)` 移动 vector；副本描述符里含字符串，拷贝代价不低。
- `results.emplace_back(tl::unexpected(code))` 构造失败值，与 `emplace_back(tl::in_place, args...)` 形成对称。
- 整个函数返回 `std::vector<tl::expected<...>>`：批量接口用"逐项 expected"表达部分失败。

**与 6.7.3 的差异**

- 图上第 9 步只画一次；实际按 tenant 分组可能调用多次（调用点 file_storage.cpp:391）。
- 图第 10 步只画 `replicas[]`；回包里还有 `lease_ttl_ms`、`object_checksum`、`resolved_tenant_id`，本路径只消费 `replicas`。
- 第 9 步和第 16 步是**同一个 RPC**（`BatchGetReplicaList`），图上画成了方向相反的两个动作。
- 图上没有逐 key 失败：实际可能是"部分 key 成功、部分 `OBJECT_NOT_FOUND`"。
- 租约从发请求**之前**开始计时，所以 owner 可用的窗口比 `lease_ttl_ms` 略短（扣掉 RTT）。
- 这条路径虽然算出了 lease deadline，但 `BatchQuerySegmentSlices` 只用 `replicas`，**没有把这个租约用在 offload 数据上**——offload buffer 的保护是回包里的 `gc_ttl_ms`（第 22 步），不是这里的 `lease_ttl_ms`。

mooncake-store/src/client_service.cpp:5652-5662

```cpp
bool Client::IsReplicaOnLocalMemory(const Replica::Descriptor& replica) {
    if (!replica.is_memory_replica()) {                         // 非 MEMORY 副本直接排除（LOCAL_DISK/NoF/DFS 都不行）
        return false;
    }
    const auto replica_transfer_endpoint =
        replica.get_memory_descriptor().buffer_descriptor.transport_endpoint_;  // 注意：这里是按值拷贝 string
    if (metadata_connstring_ == P2PHANDSHAKE) {                 // P2P 握手模式：endpoint 是 TE 的 IP:Port
        return replica_transfer_endpoint == GetTransportEndpoint();  // 与本进程 TE 的本地地址比较
    }
    return local_hostname_ == replica_transfer_endpoint;        // 其它模式：与注册 segment 时用的逻辑主机名比较
}
```

**这一段在做什么**

- 第一道过滤：必须是 MEMORY 副本；`LOCAL_DISK`、`NoF`、`DFS` 一律返回 false。
- 取出 `buffer_descriptor.transport_endpoint_`，这是发布该副本的客户端在 master 上登记的位置标识。
- P2P 模式下与 `GetTransportEndpoint()`（内部是 `transfer_engine_->getLocalIpAndPort()`，见 client_service.h:710-712）比较。
- 其它模式与 `local_hostname_` 比较——与热缓存路径发布 descriptor 时的 endpoint 选择完全对称（client_service.cpp:1948-1956）。
- 相等即认为"这份内存在本机"，于是调用方可以直接解引用 `buffer_address_`。

**重点语法 / 惯用法**

- `std::holds_alternative<MemoryReplicaData>(data_)`（在 `is_memory_replica` 内）是 variant 的类型判别惯用法。
- `const auto replica_transfer_endpoint = ...` 未加 `&`，会拷贝一份 `std::string`；语义安全但有额外分配。
- 成员函数**不是 const**：因为它调用非 const 的 `GetTransportEndpoint()`（要触达 transfer engine）。
- `metadata_connstring_ == P2PHANDSHAKE`：字符串内容比较，运行时判断一次连接模式。
- 两次 return 形成"模式分派 + 同一个判据"，比复制一段比较逻辑更清晰。

**与 6.7.3 的差异**

- 图上的"本机内存副本"像是一个身份属性；实现只是 **endpoint 字符串相等**，属于"同机"判据，不是"同地址空间"的证明。
- 随后直接解引用 `buffer_address_` 的前提是：这段内存确实在本进程可寻址（本客户端的 buffer 段 / 已映射的共享内存）。判据本身不校验这一点。
- 图上没有体现 NoF 与 LOCAL_DISK 被排除：它们即使就在本机，也不会成为 offload 源。
- 图第 11 步说"取第一个"；实现里"第一个"的顺序完全取决于 master 返回副本的排列，而该函数只做过滤、不排序，所以顺序不影响正确性，只影响是否命中。
- 图没画 P2P / 非 P2P 两种 endpoint 语义：前者是 `IP:Port`，后者是主机名，粒度不同。

### 3.3 OffloadObjects：D2H 暂存、complete_handler 与 NACK（第 12、14 步）

本小节顺着 `FileStorage::OffloadObjects` 的源码顺序读：先把 (tenant, key) 规范成 scoped key，再分桶、查源副本、必要时做 D2H 暂存，最后交给 `BatchOffload` 并在完成回调里上报 master；没上报成功的 key 全部走 NACK。

#### 3.3.1 入口与 storage_key：tenant 作用域化

mooncake-store/src/file_storage.cpp:307-321

```cpp
tl::expected<void, ErrorCode> FileStorage::OffloadObjects(  // 函数定义：成功=void，失败=ErrorCode（expected 风格，不抛异常）
    const std::vector<OffloadTaskItem>& offloading_objects) {  // 入参是第 8 步 heartbeat 领到的任务数组，每项含 tenant_id / key / size
    if (offloading_objects.empty()) {  // 本 tick 没有要落盘的对象
        return {};  // 直接返回成功；expected<void> 的 {} 就是“有值且值为 void”
    }  // 空批次分支结束
    std::unordered_map<std::string, int64_t> storage_object_sizes;  // storage_key -> 对象字节数，只给分桶估算桶大小用
    std::unordered_map<std::string, OffloadTaskItem> task_by_storage_key;  // storage_key -> 原始任务，回调阶段靠它还原 tenant_id 和用户 key
    storage_object_sizes.reserve(offloading_objects.size());  // 预分配哈希桶，避免 rehash 带来的额外拷贝
    task_by_storage_key.reserve(offloading_objects.size());  // 同上
    for (const auto& task : offloading_objects) {  // 逐条任务做 key 规范化
        const auto storage_key =  // auto 推导为 std::string（MakeScopedKey 按值返回）
            TenantId(task.tenant_id).MakeScopedKey(task.key);  // 关键：拼成 tenant + '\0' + 用户 key，后端与 master 都用这个全局唯一的 key
        storage_object_sizes.emplace(storage_key, task.size);  // emplace：同 key 重复时保留先插入的那条，不覆盖（顺带完成本批去重）
        task_by_storage_key.emplace(storage_key, task);  // 存整条任务副本（含 tenant_id 原串），后面用 .at(storage_key) O(1) 回查
    }  // 建表结束
```

**这一段在做什么**
- 函数入口：入参就是第 8 步 `OffloadObjectHeartbeat` 返回、由 master 从待办队列里摘下来的任务。
- 空批次直接成功返回，避免后面所有哈希表/循环白跑。
- 建立两张以 `storage_key` 为键的索引表：尺寸表给分桶用，任务表给完成回调反查用。
- 用 `TenantId(task.tenant_id).MakeScopedKey(task.key)` 把“租户 + 用户 key”合成后端/master 统一寻址的 scoped key。
- `reserve` 精确到任务条数；`emplace` 顺带做同批去重（同 key 保留第一条）。

**重点语法 / 惯用法**
- `tl::expected<void, ErrorCode>`：成功是 `return {}`，失败是 `tl::make_unexpected(code)`，全程不使用异常。
- `MakeScopedKey` 的分隔符是 `'\0'`（tenant_id.h:76），所以 `storage_key` 内含空字符，不能当 C 字符串交给 `printf`/`strlen`，否则会被截断。
- `TenantId(task.tenant_id)` 是临时值，其生命周期到整条语句结束；`MakeScopedKey` 返回独立的 `std::string`，没有悬垂引用。
- `emplace` 与 `operator[]` 的区别：前者遇到重复 key 不会覆盖已有值，正好是“本批去重”想要的行为。
- 任务表存的是 `OffloadTaskItem` 的**副本**，即使外部 vector 在后续被释放也不影响回调取数。

**与 6.7.3 的差异**
- 图里 key 从头到尾画的是用户 key；实际在进入后端前统一换成 tenant scoped key，master 侧再按 `task.tenant_id` 还原。
- 图第 12 步直接把“任务”接到 `BatchOffload`；实际中间先建了两张 `storage_key` 索引表，回调阶段依赖 `task_by_storage_key`。
- 图没有体现同一 key 在单批内重复出现的去重语义（`emplace` 保留首条）。
- 图没有体现 `size` 的唯一用途（参与分桶估算），也没说明它是对象 payload 字节数而非 key 长度。

#### 3.3.2 分桶：NVMe KV 走“单个伪桶”分支

mooncake-store/src/file_storage.cpp:323-340

```cpp
    std::vector<std::vector<std::string>> buckets_keys;  // 桶列表；外层每项 = 一个桶的 storage_key 列表，顺序即执行顺序
    if (auto bucket_backend =  // 条件声明：转换成功则指针非空
            std::dynamic_pointer_cast<BucketStorageBackend>(storage_backend_)) {  // 运行时探测：只有经典 bucket 后端才有“分桶”能力
        auto allocate_res = bucket_backend->AllocateOffloadingBuckets(  // 按 bucket_keys_limit / bucket_size_limit 打包
            storage_object_sizes, buckets_keys);  // 入：storage_key->size；出：buckets_keys（不够一桶的 key 可能被留在后端 ungrouped 池）
        if (!allocate_res) {  // 分组过程本身报错
            LOG(ERROR) << "AllocateOffloadingBuckets failed with error: "  // 记日志
                       << allocate_res.error();  // 打具体错误码
            return allocate_res;  // 直接整轮返回错误：本批 key 走不到 NACK flush，只能等 master 的 TTL reaper 回收（坑）
        }  // 失败分支结束
    } else {  // 非 BucketStorageBackend（NVMe KV、OffsetAllocator、DFS 等）：没有桶概念
        std::vector<std::string> keys;  // 本轮所有 key 合成一个“伪桶”
        keys.reserve(storage_object_sizes.size());  // 预留容量
        for (const auto& it : storage_object_sizes) {  // 遍历 map，只取 first（key）
            keys.emplace_back(it.first);  // 顺序是 unordered_map 的迭代顺序，不稳定但无妨
        }  // 收集结束
        buckets_keys.emplace_back(std::move(keys));  // 整个 vector 移动进列表，避免逐元素拷贝
    }  // 分桶结束
```

**这一段在做什么**
- 声明 `buckets_keys`：外层是“桶”，内层是该桶要一次性写盘的 `storage_key` 列表。
- 只有 `BucketStorageBackend` 才调用 `AllocateOffloadingBuckets` 做真实分桶（按 key 数、按总字节数、跳过已存在 key 与超大对象）。
- 分桶失败时直接返回错误，整个 `OffloadObjects` 结束。
- 其他后端（**NVMe KV 就走这一支**）把本轮所有 key 塞进唯一一个 vector，形成单个伪桶。
- 因此对 NVMe KV 而言，后面的桶循环只迭代一次，`BatchOffload` 也只调一次。

**重点语法 / 惯用法**
- `if (auto p = std::dynamic_pointer_cast<T>(base))` 是条件声明（不是 C++17 的 if-init），失败时 `p == nullptr`，天然可当布尔用。
- `dynamic_pointer_cast` 依赖 RTTI，用来做“后端是否具备某种能力”的运行时探测，避免给接口加虚函数。
- `emplace_back(std::move(keys))` 用移动构造，O(1) 而不是 O(n) 拷贝。
- 非 bucket 分支遍历 `unordered_map` 只读 `first`，顺序不确定，因此后续上报顺序也不确定。
- `allocate_res` 与函数返回类型同为 `tl::expected<void, ErrorCode>`，所以能直接 `return allocate_res;` 透传。

**与 6.7.3 的差异**
- 图第 12 步只画了一次 `BatchOffload`；实际是“按桶循环调用”：NVMe KV 一轮一次，bucket 后端每桶一次。
- 图没有 bucket 后端的跳过规则（key 已落盘、对象超过 `bucket_size_limit`、未满桶 carryover），这些 key 不会出现在任何桶里，后面 3.3.8 才补 NACK。
- 图没有“分桶失败直接返回”的分支，也就看不出这条路径上任务只能等 TTL reaper。
- 图第 12 步已写 `host slices`，但此刻 `batch_object` 里还是原始（可能是设备）指针，D2H 在下一个循环才发生。

#### 3.3.3 complete_handler：先改 transport_endpoint，再 Notify

mooncake-store/src/file_storage.cpp:342-367

```cpp
    auto complete_handler =  // 后端落盘成功后的“上报 master”闭包，稍后按值拷进 BatchOffload
        [this, &task_by_storage_key](  // 捕获 this（用 client_ / local_rpc_addr_）与任务表引用（闭包不逃逸，后端同步调用）
            const std::vector<std::string>& keys,  // 后端告知：这些 storage_key 已经写完
            std::vector<StorageObjectMetadata>& metadatas) -> ErrorCode {  // 与 keys 一一对应；非 const 引用是为了原地补字段
        VLOG(1) << "Success to store objects, keys count: " << keys.size();  // 只在 -v=1 以上输出，避免热路径刷日志
        for (auto& metadata : metadatas) {  // 逐条元数据
            metadata.transport_endpoint = local_rpc_addr_;  // 关键：后端 BuildBucket 留空，这里补上本机 offload RPC 地址，读端靠它找 owner
        }  // 补 endpoint 结束
        std::vector<OffloadTaskItem> tasks;  // 准备与 keys/metadatas 等长的任务数组
        tasks.reserve(keys.size());  // 预留
        for (const auto& key : keys) {  // 用 scoped key 反查原始任务
            auto it = task_by_storage_key.find(key);  // 用 find 而不是 at：要显式区分“找不到”这一异常
            if (it == task_by_storage_key.end()) {  // 理论上不可能，key 都来自这张表
                LOG(ERROR) << "Offload task not found for storage key";  // 记错误
                return ErrorCode::INVALID_KEY;  // 返回错误 → 后端把整桶当失败（bucket 后端还会回滚已提交的本地索引）
            }  // 找不到分支结束
            tasks.push_back(it->second);  // 拷贝任务（tenant_id + 用户 key），master 需要它来定位对象
        }  // 反查结束
        auto result = client_->NotifyOffloadSuccess(tasks, metadatas);  // 第 14 步的 RPC：用 tasks 重载，让 master 按 tenant 寻址
        if (!result) {  // RPC 失败或 master 校验失败
            LOG(ERROR) << "[OFFLOAD] NotifyOffloadSuccess failed with error: "  // 记日志
                       << result.error() << " keys count: " << keys.size();  // 带错误码与 key 数
            return result.error();  // 把错误透传给后端，触发后端的回滚/失败处理
        }  // 失败分支结束
        return ErrorCode::OK;  // 只有 OK 才会被外层 bucket_complete_handler 记进 reported_keys
    };  // 闭包定义结束；这是赋值语句，结尾分号不能少
```

**这一段在做什么**
- 定义“落盘成功 → 通知 master”的闭包，稍后交给 `BatchOffload` 在写完数据后回调。
- **先**把每条 `StorageObjectMetadata::transport_endpoint` 覆盖为 `local_rpc_addr_`，**再**发 RPC；后端构造元数据时这个字段是空的。
- 按回调给的 `keys`（scoped key）从任务表反查 `OffloadTaskItem`，拼出与 metadatas 等长的 tasks。
- 调用 `client_->NotifyOffloadSuccess(tasks, metadatas)`，让 master 登记 `LOCAL_DISK` 副本。
- Notify 失败就把错误码透传，后端据此判定整桶失败（bucket 后端会回滚刚提交的本地索引并删文件）。

**重点语法 / 惯用法**
- lambda 被赋给 `auto`，随后作为 `std::function<ErrorCode(const std::vector<std::string>&, std::vector<StorageObjectMetadata>&)>` 传入后端；返回类型显式写 `-> ErrorCode`。
- 引用捕获 `&task_by_storage_key` 安全的前提是闭包**不逃逸**：两个后端都在 `BatchOffload` 返回前同步调用它。
- `metadatas` 是非 const 引用，闭包原地改字段，调用者（后端）能看到修改后的值。
- `client_->NotifyOffloadSuccess` 有两个重载（keys 版与 tasks 版），这里必须用 tasks 版，因为 master 要 `tenant_id`。
- `return result.error();` 在 `expected` 失败分支上取值，类型正好是 `ErrorCode`，与闭包返回类型一致。

**与 6.7.3 的差异**
- 图第 14 步的“回调内上报”方向是对的，但没有画“先把 `transport_endpoint` 改成本机 RPC 地址”这一前置动作。
- 图第 22 步回给读端的 `transfer_engine_addr` 与本字段不是一回事：这里是 owner 的 **offload RPC** 地址，TE 地址在读取回包里。
- 回调的 `keys` 只是“后端选出的、确实写完且未跳过”的那部分，不保证等于整桶 key。
- 图没有 Notify 失败后的回滚语义（bucket 后端 `RollbackCommittedBucket`、其他后端返回 unexpected）。

#### 3.3.4 逐桶组装 batch_object：tenant 分组 + BatchQuerySegmentSlices

mooncake-store/src/file_storage.cpp:369-416

```cpp
    // Collect keys drained from master queue but not actually offloaded.  // 原文注释：收集“已从 master 队列取出、但没真正落盘”的 key
    // Report them back with data_size=-1 sentinel so the master can clean up  // 原文注释：用 data_size=-1 哨兵回传
    // orphaned offloading tasks and release source replica refcounts.  // 原文注释：让 master 清掉孤儿任务、减源副本引用计数
    std::vector<OffloadTaskItem> failed_tasks;  // 本轮所有失败/未上报任务，最后统一 NACK
    std::unordered_set<std::string> all_bucket_keys;  // 本轮真正进入过某个桶的 key 集合，用于最后扫出“漏网”key
    // Set when a whole-cycle error aborts the bucket loop early. We still fall  // 原文注释：整轮中止时记录错误码
    // through to the NACK flush below before returning it, so no drained key is  // 原文注释：返回前仍会执行 NACK flush
    // left waiting on the TTL reaper.  // 原文注释：避免 key 只能等 TTL 回收
    std::optional<ErrorCode> abort_error;  // 无值=没发生硬错误；有值=本轮提前结束并把这个错误码上抛

    for (const auto& keys : buckets_keys) {  // 逐桶处理；NVMe KV 只有一轮（伪桶）
        for (const auto& k : keys) all_bucket_keys.insert(k);  // 先登记：这些 key 至少进了桶，不算“漏网”
        std::unordered_map<std::string, std::vector<Slice>> batch_object;  // storage_key -> 源切片（此刻可能还是 GPU 指针）
        std::unordered_map<std::string, std::vector<std::string>>  // tenant -> 本桶内属于该 tenant 的 storage_key
            storage_keys_by_tenant;  // 续行：变量名
        for (const auto& storage_key : keys) {  // 按 tenant 分组：BatchQuery 接口要求带 tenant_id
            const auto it = task_by_storage_key.find(storage_key);  // 找任务
            if (it != task_by_storage_key.end()) {  // 找得到才分组（理论上都在）
                storage_keys_by_tenant[it->second.tenant_id].push_back(  // operator[] 默认构造 vector 再 push，天然完成分组
                    storage_key);  // 存 scoped key；查副本时还要换回用户 key
            }  // if 结束
        }  // 分组结束
        for (const auto& [tenant_id, storage_keys] : storage_keys_by_tenant) {  // 结构化绑定遍历；不同 tenant 互不影响
            std::vector<std::string> user_keys;  // BatchQuery 只认用户 key，不认 scoped key
            user_keys.reserve(storage_keys.size());  // 预留
            for (const auto& storage_key : storage_keys) {  // 把 scoped key 还原成用户 key
                user_keys.push_back(task_by_storage_key.at(storage_key).key);  // .at：确定存在，且不像 [] 那样可能插入
            }  // 还原结束
            std::unordered_map<std::string, std::vector<Slice>>  // 查询结果：用户 key -> slices
                user_batch_object;  // 续行：变量名
            [[maybe_unused]] auto query_result = BatchQuerySegmentSlices(  // 第 9/11 步的实际落点：查副本列表并挑本机内存副本
                user_keys, tenant_id, user_batch_object);  // 恒返回成功（best-effort），所以返回值标 maybe_unused
            // BatchQuerySegmentSlices is now best-effort: it always returns  // 原文注释：现在是 best-effort，恒返回 OK
            // OK. Keys present in user_batch_object go to batch_object; the  // 原文注释：查到的进 batch_object
            // rest are reported as failed.  // 原文注释：查不到的记为失败
            for (const auto& storage_key : storage_keys) {  // 按 storage_key 回填，保证 batch_object 的键是 scoped key
                const auto& task = task_by_storage_key.at(storage_key);  // 取任务（要用它的用户 key 去查结果）
                auto it = user_batch_object.find(task.key);  // 用用户 key 命中查询结果
                if (it != user_batch_object.end()) {  // 找到了本机内存副本
                    batch_object.emplace(storage_key, std::move(it->second));  // move 走 slices，避免拷贝 Slice 数组
                } else {  // 没有本机内存副本（已被淘汰 / 不在本机 / 查询失败）
                    failed_tasks.push_back(task);  // 本轮不落盘，先记账，最后 NACK
                }  // if 结束
            }  // 回填结束
        }  // tenant 循环结束
        if (batch_object.empty()) {  // 整桶一个可读源都没有
            continue;  // 跳过该桶：不 D2H、不 BatchOffload
        }  // if 结束
```

**这一段在做什么**
- 准备三个跨桶累积的状态：`failed_tasks`（待 NACK）、`all_bucket_keys`（进过桶的 key）、`abort_error`（硬错误码）。
- 进入桶循环，先把本桶所有 key 登记到 `all_bucket_keys`。
- 按 `tenant_id` 把本桶的 scoped key 分组，因为 `BatchQuery` 必须带 tenant。
- 每个 tenant 单独调 `BatchQuerySegmentSlices`：传用户 key，返回“用户 key → 本机内存切片”。
- 再把结果按 scoped key 回填进 `batch_object`；查不到的 key 直接进 `failed_tasks`（对象级失败，不中止本轮）。
- 整桶都没有可读源时 `continue`，不触发后续任何动作。

**重点语法 / 惯用法**
- `for (const auto& [tenant_id, storage_keys] : ...)` 是 C++17 结构化绑定，注意 `tenant_id` 的类型是 `std::string`。
- `[[maybe_unused]]` 明确表达“返回值故意不看”，同时避免 `-Wunused-variable`。
- `task_by_storage_key.at(storage_key)` 在 const 上下文里安全：不会像 `operator[]` 那样插入默认值。
- `batch_object.emplace(storage_key, std::move(it->second))` 转移整个 `vector<Slice>`，只搬指针数组，不搬数据。
- `storage_keys_by_tenant[it->second.tenant_id]` 借 `operator[]` 的默认构造实现“分组桶”，一行搞定。
- `BatchQuerySegmentSlices` 的签名仍是 `tl::expected<void, ErrorCode>`，但实现恒返回 `{}`，best-effort 由注释约定。

**与 6.7.3 的差异**
- 图第 9 步把 `BatchQuery` 画成本轮一次；实际是“每桶 × 每 tenant”各一次，且传的是用户 key 而非 scoped key。
- 图第 11 步“取第一个本机内存副本组成 Slice”发生在 `BatchQuerySegmentSlices` 内部（`IsReplicaOnLocalMemory` + `memory_descriptor.buffer_descriptor`），这里只负责调用。
- 图没有画 best-effort：源副本查不到的 key 只是进 `failed_tasks`，不会让本轮报错。
- 图第 10 步的 `replicas[]` 没画副本类型过滤，实际只认 `IsReplicaOnLocalMemory` 的那一个，并且一个对象只取一个内存 buffer。

#### 3.3.5 D2H 暂存：GPU 指针先拷到 pinned host，CPU 指针零拷贝

mooncake-store/src/file_storage.cpp:418-468

```cpp
        // D2H staging: replace device slices with host memory slices  // 原文注释：D2H 暂存，把设备切片换成主机切片
        // so that storage_backend (ConcatSlicesToString / BuildBucket /  // 原文注释：让后端这几处永远拿到 host 指针
        // WriteBucket) always receives host pointers.  // 原文注释：续
        std::unordered_map<std::string, std::vector<Slice>> host_batch_object;  // 与 batch_object 同构，但值全部指向 host 内存
        std::vector<PinnedBufferPool::Buffer> staging_bufs;  // 持有所有暂存缓冲，生命周期必须覆盖整个 BatchOffload
        auto runtime_accelerator =  // 设备运行时注册表（GPU 后端）
            device::GetAcceleratorRegistry().RuntimeAccelerators();  // 取出全部加速器，用它的 FindDeviceForPointer 判断指针归属

        for (auto& [obj_key, slices] : batch_object) {  // 结构化绑定 + 引用：obj_key 是 scoped key，slices 之后可被 move
            std::vector<Slice> host_slices;  // 该对象最终的 host 切片
            bool obj_success = true;  // 对象级成功标志：任一 slice 失败就整体放弃该对象
            for (const auto& slice : slices) {  // 逐切片判断是否需要拷贝
                device::PointerInfo info{};  // 值初始化：device_id 等字段清零，供 FindDeviceForPointer 回填
                auto* device =  // 非空表示这是设备（GPU）指针
                    runtime_accelerator.FindDeviceForPointer(slice.ptr, &info);  // 只做地址区间匹配，不访问数据
                if (device) {  // GPU 指针：必须 D2H 拷贝
                    device->SetContext(info.device_id);  // 切到该指针所属设备，很多后端要求当前 context 正确
                    auto buf = pinned_buffer_pool_->Acquire(slice.size);  // 取一块 >= size 的 pinned host 缓冲（池满/无 pinned 时回退 pageable）
                    if (!device->Copy(buf.data, slice.ptr, slice.size,  // 同步 D2H 拷贝
                                      device::CopyDirection::kDeviceToHost)) {  // 方向枚举：设备 → 主机
                        LOG(ERROR) << "D2H staging failed for key: " << obj_key;  // 拷贝失败：记录 key
                        pinned_buffer_pool_->Release(std::move(buf));  // 立刻还池，避免泄漏
                        obj_success = false;  // 标记对象失败
                        failed_tasks.push_back(task_by_storage_key.at(obj_key));  // 对象级 NACK 记账（整对象放弃，不落盘）
                        break;  // 放弃该对象剩余切片
                    }  // 拷贝失败分支结束
                    host_slices.emplace_back(Slice{buf.data, slice.size});  // 记录 host 指针；buf.data 仍有效，因为 buf 马上移进 staging_bufs
                    staging_bufs.push_back(std::move(buf));  // 所有权转移：Buffer 是 move-only，move 后 buf 变为空
                } else {  // 普通主机内存指针
                    host_slices.push_back(slice);  // 原样透传 = CPU 侧零拷贝
                }  // if 结束
            }  // slice 循环结束
            if (obj_success) {  // 所有切片都就绪才进入待落盘 map
                host_batch_object[obj_key] = std::move(host_slices);  // move 进 map；operator[] 默认构造后移动赋值
            }  // if 结束
        }  // 对象循环结束

        // If every object in this bucket failed D2H staging, host_batch_object  // 原文注释：若整桶对象都 D2H 失败
        // is empty (those keys are already in failed_tasks). Skip BatchOffload,  // 原文注释：host_batch_object 为空（key 已在 failed_tasks）
        // which rejects an empty map as INVALID_KEY and would otherwise trip  // 原文注释：BatchOffload 会以 INVALID_KEY 拒绝空 map
        // the whole-cycle abort below for a bucket that has nothing left to  // 原文注释：否则会误触发整轮中止
        // persist. staging_bufs can still be non-empty here (an object whose  // 原文注释：此时 staging_bufs 可能非空（对象前半成功、后半失败）
        // first slices copied fine but a later one failed), so hand those  // 原文注释：这些缓冲要还池
        // buffers back before continuing: the release loop after BatchOffload  // 原文注释：BatchOffload 之后的释放循环在此路径不可达
        // is unreachable on this path.  // 原文注释：续
        if (host_batch_object.empty()) {  // 整桶没有可落盘对象
            for (auto& buf : staging_bufs) {  // 手工释放
                pinned_buffer_pool_->Release(std::move(buf));  // 还池；缓冲仍存活到此处，指针没有悬空
            }  // 释放结束
            continue;  // 处理下一个桶
        }  // if 结束
```

**这一段在做什么**
- 新建 `host_batch_object`：与 `batch_object` 同构，但值全部指向 host 内存，保证后端（`ConcatSlicesToString`/`BuildBucket`/`WriteBucket`）只看到主机指针。
- `staging_bufs` 持有所有 pinned 缓冲，是这些 host 指针的生命周期保证。
- 用加速器注册表的 `FindDeviceForPointer` 判断每个切片指针是否属于 GPU。
- GPU 切片：切设备 context → 取 pinned 缓冲 → `Copy(kDeviceToHost)` → 记录 host 指针。
- CPU 切片：原样放入，不做任何拷贝（这一支才是“零拷贝”）。
- 对象级失败只 NACK 该对象；整桶失败则先把 `staging_bufs` 还池再 `continue`，避免 `BatchOffload` 收到空 map 报 `INVALID_KEY`。

**重点语法 / 惯用法**
- `PinnedBufferPool::Buffer` 拷贝构造/赋值被 `= delete`，只能 `std::move`；`Release(Buffer)` 也是按值收，配合 `std::move` 转移所有权。
- `Acquire` 优先复用容量足够的池缓冲，池里没有就 `AllocatePinned`，pinned 也失败才退回 pageable 内存（pinned_buffer_pool.h:77-129）。
- `FindDeviceForPointer(ptr, &info)` 返回 `nullptr` 表示“不是设备指针”，此时无需 D2H，这是“CPU 零拷贝”的判定依据。
- `device::PointerInfo info{}` 用 `{}` 做值初始化，避免未初始化字段参与判断。
- `obj_success` + `break` 组合出对象级 all-or-nothing：失败对象不会进入 `host_batch_object`。
- 失败时先 `Release` 刚取的这块 buf 再 `break`，而同一对象先前已成功的切片缓冲仍留在 `staging_bufs`，靠后面统一释放。

**与 6.7.3 的差异**
- 图第 12 步写 `BatchOffload(host slices)`，仿佛 host 指针天然存在；实际 GPU 源要先 D2H，只有 CPU 源才是零拷贝。
- 图把 offload 描述成零拷贝，这个说法只在“副本就在本机 CPU 内存”时成立。
- 图没有 D2H 失败路径：失败是对象级 NACK，同桶其他对象照常落盘。
- 图没有 pinned 缓冲池，也没有“缓冲必须活到 `BatchOffload` 返回之后才释放”这条生命周期约束。

#### 3.3.6 bucket_complete_handler 与 BatchOffload：reported_keys 记账

mooncake-store/src/file_storage.cpp:470-520

```cpp
        auto offload_start = std::chrono::steady_clock::now();  // 本桶计时起点，覆盖整个 BatchOffload
        std::unordered_set<std::string> reported_keys;  // 本桶内“已成功 Notify 给 master”的 key；每个桶重新建
        reported_keys.reserve(host_batch_object.size());  // 预留
        auto bucket_complete_handler =  // 在 complete_handler 外包一层“记账 + 指标”
            [this, offload_start, complete_handler, &reported_keys](  // 时间点/闭包按值捕获；reported_keys 按引用（桶作用域内同步调用，安全）
                const std::vector<std::string>& keys,  // 后端本次真正写完的 key，可能只是桶内一部分
                std::vector<StorageObjectMetadata>& metadatas) -> ErrorCode {  // 与 keys 等长
            auto res = complete_handler(keys, metadatas);  // 先补 endpoint 再 Notify master（见 3.3.3）
            if (res == ErrorCode::OK) {  // 只有上报成功才记账
                reported_keys.insert(keys.begin(), keys.end());  // 批量插入；没上报的 key 之后会被 NACK
            }  // if 结束
            if (res == ErrorCode::OK && ssd_metric_) {  // 成功且指标对象存在才统计
                auto elapsed_us =  // 本桶耗时（微秒）
                    std::chrono::duration_cast<std::chrono::microseconds>(  // 时钟换算
                        std::chrono::steady_clock::now() - offload_start)  // 单调时钟，不受系统时间跳变影响
                        .count();  // 取整数微秒
                int64_t total_bytes = 0;  // 本桶总字节数
                for (const auto& metadata : metadatas) {  // 累加每条 metadata 的 data_size
                    total_bytes += metadata.data_size;  // 后端填的是对象 payload 大小（不含 key）
                }  // 累加结束
                ssd_metric_->ssd_write_ops.inc(keys.size());  // 写操作计数
                ssd_metric_->ssd_write_bytes.inc(total_bytes);  // 写字节计数
                ssd_metric_->ssd_write_latency_us.observe(elapsed_us);  // 写时延直方图
                ssd_metric_->ssd_write_latency_summary.observe(elapsed_us);  // 写时延摘要（分位数）
                ssd_metric_->ssd_total_ops.inc(keys.size());  // 总操作数（读写共用口径）
                ssd_metric_->ssd_total_bytes.inc(total_bytes);  // 总字节数
                ssd_metric_->ssd_total_latency_us.observe(elapsed_us);  // 总时延
                ssd_metric_->ssd_total_latency_summary.observe(elapsed_us);  // 总时延摘要
            }  // 指标结束
            return res;  // 原样返回；失败会让后端回滚并非零返回
        };  // 闭包定义结束（赋值语句，别漏分号）
        auto offload_res = storage_backend_->BatchOffload(  // 第 12/13 步：真正落盘（NVMe KV 内先 chunk、后 root）
            host_batch_object, bucket_complete_handler,  // 数据 + 完成回调（同步调用）
            [this](const std::vector<std::string>& evicted_keys) {  // 写盘空间不足时的淘汰回调
                return NotifyEvictedDiskReplicas(evicted_keys);  // 把被淘汰的 key 通知 master 删除对应 LOCAL_DISK 副本
            });  // 调用结束

        // Release staging buffers back to pool.  // 原文注释：D2H 缓冲还池
        for (auto& buf : staging_bufs) {  // 到这一步后端已同步写完，host 指针不再被使用
            pinned_buffer_pool_->Release(std::move(buf));  // Buffer move-only，只能 move 进 Release
        }  // 释放结束
        if (offload_res) {  // BatchOffload 整体成功 ≠ 每个 key 都上报了
            // A successful batch may skip duplicates or individual writes.  // 原文注释：一批成功也可能跳过重复 key 或单条写失败
            // Only the completion callback's keys were reported to Master;  // 原文注释：只有回调里的 key 报给了 master
            // NACK the rest to release their tasks and source replica refs.  // 原文注释：其余 NACK，释放任务与源副本引用
            for (const auto& [key, _] : host_batch_object) {  // 遍历本桶全部 key；_ 只是占位名
                if (reported_keys.find(key) == reported_keys.end()) {  // 没进过 reported_keys
                    failed_tasks.push_back(task_by_storage_key.at(key));  // 记 NACK（例如并发 offload 已提交该 key，被后端跳过）
                }  // if 结束
            }  // 遍历结束
        }  // if 结束
```

**这一段在做什么**
- 每个桶记录起始时间，并新建本桶的 `reported_keys` 集合。
- `bucket_complete_handler` 包住 `complete_handler`：先上报，成功才把 key 记进 `reported_keys`，顺带打 SSD 写指标。
- 调用 `storage_backend_->BatchOffload(host_batch_object, 回调, 淘汰回调)`，后者在磁盘空间不足时把淘汰 key 同步通知 master。
- `BatchOffload` 返回后立刻释放所有 `staging_bufs`，因为后端是同步 API，返回时数据已拷走。
- 成功分支里补 NACK：`host_batch_object` 中没出现在 `reported_keys` 的 key 都要回传失败。

**重点语法 / 惯用法**
- lambda 按值捕获 `offload_start`（时间点）和 `complete_handler`（`std::function` 拷贝），按引用捕获 `reported_keys`；引用安全依赖“回调在桶作用域内同步执行”。
- `reported_keys.insert(keys.begin(), keys.end())` 是区间插入，比循环 `insert` 少多次哈希。
- `ssd_metric_` 是裸指针成员，可能为 `nullptr`，所以用 `&&` 短路先判空。
- `std::chrono::steady_clock` + `duration_cast<std::chrono::microseconds>` 是仓库统一计时写法。
- `if (offload_res)` 利用 `tl::expected` 的显式布尔转换判断成功。
- `for (const auto& [key, _] : host_batch_object)` 中 `_` 只是普通标识符，不是特殊语法。

**与 6.7.3 的差异**
- 图第 14 步只画一个“按 bucket 上报”的箭头；实际每个桶一次回调，NVMe KV 整轮（单个伪桶）只有一次。
- 图没有 `reported_keys` 这层过滤，读者容易以为“`BatchOffload` 返回成功 = 每个 key 都已通知 master”。
- 图没有 SSD 写指标（ops/bytes/latency）。
- 图没有 `eviction_handler`：写盘空间不足时会在 `BatchOffload` 内部同步淘汰并通知 master，画面上并不存在这一环。

#### 3.3.7 失败分类与 KEYS_ULTRA_LIMIT：软错误继续，硬错误中止

先看分类器本体（它只认两个错误码）：

mooncake-store/src/file_storage.cpp:302-305

```cpp
bool FileStorage::IsPerBucketSoftOffloadError(ErrorCode error) {  // 判定：该错误是否只影响当前桶
    return error == ErrorCode::INVALID_READ ||  // 源数据读不到/暂存失败 → 本桶这轮放弃即可
           error == ErrorCode::OBJECT_ALREADY_EXISTS;  // key 已在落盘或正被并发 offload → 本桶整体被原子拒绝，也只需放弃本桶
}  // 其余错误（KEYS_ULTRA_LIMIT、INTERNAL_ERROR、INVALID_PARAMS…）一律视为整轮硬错误
```

**这一段在做什么**
- 用白名单方式定义“软错误”：只有 `INVALID_READ` 和 `OBJECT_ALREADY_EXISTS`。
- 其他所有错误都归为硬错误，包括 `KEYS_ULTRA_LIMIT`。
- 这个函数是 `static` 成员，不依赖实例状态，便于单测直接调用。

**重点语法 / 惯用法**
- 返回 `bool` 的纯函数，`const`/无副作用，适合放进 `if (!Is...)` 做分支条件。
- `||` 短路求值，两个枚举比较，无隐藏成本。
- 头文件注释解释了为什么 `OBJECT_ALREADY_EXISTS` 必须是软错误（bucket 后端 `PrepareEviction` 的重复守卫会原子拒绝整桶，若当致命错误会泄漏任务并让 master/SSD 元数据不一致，issue #2827）。

**与 6.7.3 的差异**
- 图中第 12/14 步只有一条“成功”路径，没有软/硬错误之分。
- 图没有体现“同一个 `BatchOffload` 调用可能整桶失败，但整轮仍继续”。
- 图没有体现错误码会反过来决定“是否继续消费后续桶”。

接着看调用点：

mooncake-store/src/file_storage.cpp:521-548

```cpp
        if (!offload_res) {  // BatchOffload 失败（对本桶而言是整桶失败）
            LOG(ERROR) << "Failed to store objects with error: "  // 记错误
                       << offload_res.error();  // 具体错误码
            // This bucket did not persist, so report its keys back to the  // 原文注释：这个桶没落盘，key 全部回传
            // master as failed regardless of whether we continue or abort.  // 原文注释：无论继续还是中止都要回传
            // Doing it here (rather than only on the soft path) keeps their  // 原文注释：放在这里而不是只在软错误路径
            // offloading tasks and source-replica refcounts from leaking until  // 原文注释：避免任务和引用计数泄漏
            // the put_start_release_timeout_sec_ TTL reaper fires.  // 原文注释：直到 TTL reaper 触发
            for (const auto& [key, _] : host_batch_object) {  // 整桶每个 key
                failed_tasks.push_back(task_by_storage_key.at(key));  // 全部记为失败（后面统一 NACK）
            }  // 遍历结束
            if (offload_res.error() == ErrorCode::KEYS_ULTRA_LIMIT) {  // 磁盘 key 数/容量已到上限（后端也可能已自行禁用）
                // Disk is over the key-count limit: stop offloading entirely.  // 原文注释：超过 key 数上限，彻底停止 offload
                MutexLocker locker(&offloading_mutex_);  // 与 heartbeat 读 enable_offloading_ 用同一把锁（GUARDED_BY 标注）
                enable_offloading_ = false;  // 闩锁：本进程此后不再接 offload 任务；Init 之外没有地方再置回 true，只能重启恢复
            }  // if 结束
            if (!IsPerBucketSoftOffloadError(offload_res.error())) {  // 不是软错误 → 整轮硬错误
                // Whole-cycle error (KEYS_ULTRA_LIMIT or any hard failure):  // 原文注释：整轮错误（KEYS_ULTRA_LIMIT 或任意硬失败）
                // stop processing further buckets, but fall through to the NACK  // 原文注释：停止处理后续桶
                // flush below so every drained key is released. Unvisited  // 原文注释：但仍落到下面的 NACK flush
                // buckets are not yet in all_bucket_keys, so the sweep NACKs  // 原文注释：未访问的桶不在 all_bucket_keys 里
                // them too; this bucket's keys were just pushed above.  // 原文注释：所以最后扫描会一并 NACK 它们
                abort_error = offload_res.error();  // 记住错误码，循环结束后上抛
                break;  // 跳出桶循环；用 break 而不是 return，保证后面的 NACK 一定执行
            }  // if 结束
            // Soft per-bucket error: keep processing the remaining buckets.  // 原文注释：软错误只影响本桶，继续下一个桶
        }  // 失败分支结束
    }  // 桶循环结束
```

**这一段在做什么**
- `BatchOffload` 失败时先把整个桶的 key 记进 `failed_tasks`，无论后面是继续还是中止。
- 若错误码是 `KEYS_ULTRA_LIMIT`，加 `offloading_mutex_` 把 `enable_offloading_` 置 `false`，彻底停止 offload。
- 再按软/硬错误分类：软错误（`INVALID_READ`、`OBJECT_ALREADY_EXISTS`）继续下一个桶。
- 硬错误把错误码存进 `abort_error` 后 `break`，但仍会执行函数末尾的 NACK flush。
- 失败 key 全部记账，避免 offloading task 与源副本 refcount 泄漏到 `put_start_release_timeout_sec_` 的 TTL reaper。

**重点语法 / 惯用法**
- `MutexLocker locker(&offloading_mutex_)` 是 RAII 锁，作用域结束自动解锁；成员声明上有 `GUARDED_BY(offloading_mutex_)` 注解供静态分析。
- `enable_offloading_` 的生命周期很关键：只在 `Init()` 里赋值、在这里被置 `false`，之后没有恢复路径。
- `std::optional<ErrorCode> abort_error` 用 `has_value()`/`if (abort_error)` 判断，支持“没有错误”这个第三态。
- `break` 只跳出桶循环而不是 `return`，这是“先记账、后返回”的刻意写法。
- `offload_res.error()` 只能在失败分支调用（`expected` 的错误访问），与 `*offload_res` 的适用分支相反。

**与 6.7.3 的差异**
- 图第 15 步只画了成功路径（`dec_refcnt`、无 `LOCAL_DISK` 才登记、已有则更新），没画失败 key 的 NACK（`data_size=-1`）。
- 图没有 `KEYS_ULTRA_LIMIT` 的全局副作用：下一次 heartbeat 会带 `enable_offloading=false`，master 据此清空 pending 队列并对每条任务 `dec_refcnt`、清 `offloading_task`（master_service.cpp `OffloadObjectHeartbeat`）。
- 图没有软/硬错误区分，也看不出硬错误中止后仍会执行 NACK。
- 图把“`BatchOffload` 失败”当成一个原子事件；实际上可能部分 key 已经上报成功（`reported_keys` 非空），失败分支仍把整桶再 NACK 一次。master 侧此时 `offloading_task` 已被成功路径清空，NACK 变成幂等空操作。

#### 3.3.8 未进桶的 key 与批量 NACK flush

mooncake-store/src/file_storage.cpp:550-577

```cpp
    // Keys skipped by GroupOffloadingKeysByBucket don't appear in any bucket,  // 原文注释：被 GroupOffloadingKeysByBucket 跳过的 key 不属于任何桶
    // so they never reach BatchOffload or complete_handler.  // 原文注释：因此永远走不到 BatchOffload / complete_handler
    for (const auto& [storage_key, task] : task_by_storage_key) {  // 全量扫一遍任务表
        if (all_bucket_keys.find(storage_key) == all_bucket_keys.end()) {  // 从未进入任何桶（已存在 / 超大对象 / 留在 ungrouped 池）
            failed_tasks.push_back(task);  // 补记 NACK
        }  // if 结束
    }  // 扫描结束

    if (!failed_tasks.empty()) {  // 有失败/未上报任务才发 RPC
        std::vector<StorageObjectMetadata> failed_metadatas;  // 与 failed_tasks 等长的元数据数组
        failed_metadatas.reserve(failed_tasks.size());  // 预留
        for (size_t i = 0; i < failed_tasks.size(); ++i) {  // 只需要个数，i 本身不用（保留原写法）
            failed_metadatas.push_back(StorageObjectMetadata{-1, 0, 0, -1, ""});  // 聚合初始化：bucket_id=-1, offset=0, key_size=0, data_size=-1（NACK 哨兵）, endpoint=""
        }  // 填充结束
        auto result =  // NACK 也复用 NotifyOffloadSuccess，靠 data_size<0 区分
            client_->NotifyOffloadSuccess(failed_tasks, failed_metadatas);  // 图里没有的“第二次上报”：一次批量 NACK
        if (!result) {  // NACK 失败
            LOG(WARNING) << "[OFFLOAD] NotifyOffloadSuccess for failed tasks "  // 只告警不返回错误
                            "returned error: "  // 相邻字符串字面量自动拼接
                         << result.error() << " count: " << failed_tasks.size();  // 错误码 + 条数
        }  // if 结束
    }  // if 结束

    if (abort_error) {  // 有硬错误：把错误码向上抛
        return tl::make_unexpected(*abort_error);  // 解引用 optional 取 ErrorCode，包成 unexpected
    }  // if 结束
    return {};  // expected<void> 的成功态：本轮“尽力而为”结束
}  // OffloadObjects 结束
```

**这一段在做什么**
- 扫一遍 `task_by_storage_key`，找出从未进入任何桶的 key（分桶阶段跳过的），补进 `failed_tasks`。
- 有失败任务时，构造与任务等长的 `StorageObjectMetadata`，全部填 `data_size = -1` 作为 NACK 哨兵。
- 复用 `NotifyOffloadSuccess(tasks, metadatas)` 一次批量回传，让 master 清任务、减 refcount。
- NACK RPC 失败只打 `WARNING`：尽力而为，剩下的靠 master 的 TTL reaper。
- 最后按 `abort_error` 决定返回失败还是成功；返回失败只表示“本轮被硬错误中止”。

**重点语法 / 惯用法**
- `StorageObjectMetadata` 用聚合初始化，字段顺序是 `bucket_id, offset, key_size, data_size, transport_endpoint`（types.h:539-547），写错顺序不会报错但语义全错。
- `data_size = -1` 是跨进程协议约定：master 侧用 `metadata.data_size < 0` 判定 NACK（master_service.cpp 的 `NotifyOffloadSuccess`），此时只 `dec_refcnt` + 清 `offloading_task`，不 `AddReplica`。
- `for (size_t i = 0; i < failed_tasks.size(); ++i)` 只为取次数，`i` 未使用，保留原写法。
- `tl::make_unexpected(*abort_error)`：从 `optional` 解引用取值再包成 `expected` 的错误态。
- `return {}` 是 `expected<void, ErrorCode>` 的成功返回。
- `LOG(WARNING) << "..." "..."` 相邻字符串字面量在编译期拼接，跨行书写无需 `+`。

**与 6.7.3 的差异**
- 图第 14 步之后没有“第二次 `NotifyOffloadSuccess`（批量 NACK）”这条边。
- 图没有“key 从未进入任何桶”这条支路（分桶阶段跳过的 key 只能在这里被释放）。
- 图第 15 步只描述成功路径；NACK 路径在 master 侧只做 `dec_refcnt` 和清 `offloading_task`，**不**登记副本。
- 图第 6 步 `Put` 早已返回给 App，本节的失败与返回值都不影响 `Put` 的结果；`OffloadObjects` 的返回值只用于 heartbeat 侧记录本 tick 是否被硬错误中止。

### 3.4 BatchOffload 的准入与 prepare（第 12–13 步）

mooncake-store/src/nvme_kv/backend.cpp:479-542

```cpp
tl::expected<int64_t, ErrorCode> NvmeKvStorageBackend::BatchOffload(   // 返回值：成功落盘的 key 个数；失败返回 ErrorCode
    const std::unordered_map<std::string, std::vector<Slice>> &batch_object,   // 入参：storage key → 该对象的全部 Slice（本机内存指针+长度）
    std::function<ErrorCode(const std::vector<std::string> &keys,   // 回调参数 1：写成功的 key 列表
                            std::vector<StorageObjectMetadata> &metadatas)>   // 回调参数 2：与 keys 同序的元数据（bucket/offset/size）
        complete_handler,   // FileStorage 注入的回调，内部转调 master 的 NotifyOffloadSuccess（图第 14 步）
    NvmeKvStorageBackend::EvictionHandler eviction_handler) {   // 淘汰回调；NVMe KV 后端不做容量淘汰
    (void)eviction_handler;   // 显式标记未使用，避免 -Wunused-parameter（接口签名不能改）
    if (!initialized_.load(std::memory_order_acquire)) {   // acquire 读：与 Init() 里的 release 写配对，保证 connector_ 已可见
        return tl::make_unexpected(ErrorCode::INTERNAL_ERROR);   // 未 Init 就 offload，属于调用方错误
    }
    if (batch_object.empty()) {   // 空批次直接报错，而不是返回 0
        return tl::make_unexpected(ErrorCode::INVALID_KEY);   // 注意错误码是 INVALID_KEY，不是 INVALID_PARAMS
    }

    struct OffloadPlan {   // 单个对象的落盘计划：只存指针+长度，不拷贝 payload
        std::string key;   // 逻辑 key（tenant:key），最后要回传给 complete_handler
        const std::vector<Slice> *slices = nullptr;   // 指向 batch_object 内部 vector；生命周期由入参 const& 保证
        size_t payload_size = 0;   // 所有 Slice 长度之和
    };
    std::vector<OffloadPlan> plans;   // 过滤后的待写对象列表
    plans.reserve(batch_object.size());   // 预留上限，避免扩容搬移（slices 指针指向的是 map，不受影响）
    int64_t batch_total_size = 0;   // 本批次总字节数，用于配额预检查
    for (const auto &[key, slices] : batch_object) {   // C++17 结构化绑定；unordered_map 迭代顺序不稳定 → plans 顺序不稳定
        if (slices.empty()) {   // 该 key 没有任何本机副本地址
            continue;   // 跳过而不是报错：可能副本刚好被淘汰
        }
        size_t payload_size = 0;   // 本对象累计长度
        for (const auto &slice : slices) {   // 逐个 Slice 累加
            if (slice.size >
                std::numeric_limits<size_t>::max() - payload_size) {   // 先判减法溢出，再加，避免 size_t 回绕
                return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
            }
            payload_size += slice.size;   // 此时加法一定安全
        }
        if (payload_size >
                static_cast<size_t>(std::numeric_limits<int64_t>::max()) ||   // 单对象不能超过 int64 上限
            batch_total_size > std::numeric_limits<int64_t>::max() -
                                   static_cast<int64_t>(payload_size)) {   // 累计值同样用减法判溢出
            return tl::make_unexpected(ErrorCode::INVALID_PARAMS);   // 一个对象超限，整批拒绝
        }
        batch_total_size += static_cast<int64_t>(payload_size);   // 累加到批次总量
        plans.push_back(OffloadPlan{key, &slices, payload_size});   // key 拷贝一份；&slices 是 map 元素的地址
    }

    if (plans.empty()) {   // 所有 key 都因 slices 为空被跳过
        return 0;   // 无事可做 → 返回 0，不是错误
    }

    if (total_keys_.load(std::memory_order_relaxed) +
            static_cast<int64_t>(plans.size()) >
        file_storage_config_.total_keys_limit) {   // key 配额预检查（只读不预留，存在并发竞态）
        return tl::make_unexpected(ErrorCode::KEYS_ULTRA_LIMIT);   // 配额满 → 整批拒绝
    }
    if (total_size_.load(std::memory_order_relaxed) + batch_total_size >
        file_storage_config_.total_size_limit) {   // 字节配额同样预检查
        return tl::make_unexpected(ErrorCode::KEYS_ULTRA_LIMIT);   // 复用同一个错误码（语义上略混）
    }
    auto enable_offloading_res = IsEnableOffloading();   // 总开关：未初始化 / 未接设备 / 配额用尽 都会返回 false 或错误
    if (!enable_offloading_res) {   // 查询本身失败（例如 INTERNAL_ERROR）
        return tl::make_unexpected(enable_offloading_res.error());   // 原样冒泡
    }
    if (!enable_offloading_res.value()) {   // 明确返回 false：本机不该再接收 offload
        return tl::make_unexpected(ErrorCode::KEYS_ULTRA_LIMIT);
    }
```

**这一段在做什么**
- 把 `unordered_map<key, vector<Slice>>` 归一化成 `vector<OffloadPlan>`：每个对象一条，记录逻辑 key、Slice 地址、总长度。
- 做三组准入：参数合法性（空批次、空 slices、溢出）、配额（key 数 / 字节数）、总开关 `IsEnableOffloading()`。
- 任何一组不通过都**整批拒绝**，不会只挑一部分对象写；`plans.empty()` 是唯一返回 0 的成功短路。
- 这一步不碰设备，`BatchOffload` 在这里还只是"算账"；真正的 I/O 在 `store_chunks()`（3.5）。

**重点语法 / 惯用法**
- `tl::expected<T, E>`：正常路径 `return 值`，异常路径 `return tl::make_unexpected(code)`，没有异常也没有 `std::pair`。
- 结构化绑定 `for (const auto &[key, slices] : ...)`，`slices` 绑定到 map 里的 `vector`，因此后面 `&slices` 取到的是 **map 元素的地址**（不是临时对象）。
- 溢出检查统一用"减法判"（`max - x`），避免先加后判时 `size_t` / `int64_t` 回绕；两处都写了注释可以对照看。
- `std::memory_order_relaxed` 用于计数器：只要求原子、不要求顺序，代价最低，但不同线程看到的数值可能有延迟。
- 配额是"先读后写"的乐观检查：这里只 `load`，真正的 `fetch_add` 在函数末尾，因此并发下可能略微超限。

**与 6.7.3 的差异**
- 图上第 12 步直接画 `BatchOffload(host slices)`，看不出这里有一整套准入（配额、总开关、溢出）；被拒绝时 `FileStorage` 看到的是错误码而不是部分的成功。
- 图上"host slices"是复数，但代码里每个 key 对应 `vector<Slice>`：一个对象可能由多段本机内存拼成，长度在此处才合并计算。
- `slices.empty()` 的 key 被**静默跳过**，图上没有这个分支；这类 key 会从返回值和回调里一并消失。
- 配额错误用的都是 `KEYS_ULTRA_LIMIT`，即使超的是字节限额；按错误码排查时不要以为只有 key 数超了。

mooncake-store/src/nvme_kv/backend.cpp:544-595

```cpp
    std::vector<tl::expected<OffloadResult, ErrorCode>> results;   // 每个 plan 一个结果槽
    results.reserve(plans.size());
    for (size_t i = 0; i < plans.size(); ++i) {
        results.emplace_back(OffloadResult{});   // 默认值 {stored=false, inserted=false}，即"还没写"
    }

    struct PreparedOffload {   // 每个 plan 的写入过程状态（跨 prepare / chunk / root 三个阶段复用）
        std::string payload_storage;   // 仅当对象由多个 Slice 组成时才用到：拼接后的连续缓冲
        std::string_view payload;   // 单 Slice 时直接指向调用方内存；多 Slice 时指向 payload_storage
        std::optional<NvmeKvKeyConflictPolicy::WritePlan> write_plan;   // prepare 成功前为 nullopt
        std::vector<PhysicalKey> written_chunk_keys;   // 本次已写成功的 chunk key，失败时按此清单回滚（只删自己写的）
        bool root_written = false;   // root 是否已写成功；root 最后写，等价于"是否已提交"
        bool needs_fallback = false;   // 需要交给 OffloadOne 逐槽位重试
    };
    std::vector<PreparedOffload> prepared(plans.size());   // 定长构造，之后绝不 resize（见"重点语法"）

    const auto prepare_one = [&](size_t index) {   // 单对象准备：拼 payload + 生成写入计划
        const auto &plan = plans[index];
        auto &item = prepared[index];
        item.payload = BuildPayloadView(*plan.slices, plan.payload_size,
                                        item.payload_storage);   // 单 Slice → 零拷贝 view；多 Slice → memcpy 进 storage
        auto write_plan = NvmeKvKeyConflictPolicy::BuildWritePlan(
            NvmeKvObjectIdentity{.logical_key = plan.key}, item.payload, 0,   // 关键：批量路径固定只试 slot 0
            std::max(connector_->GetCapabilities().effective_max_value_size,
                     kMinMaxValueSize));   // 设备上限可能小到装不下 header，用 kMinMaxValueSize 兜底
        if (!write_plan) {   // 例如 payload 超过 uint32、manifest 装不进单值
            results[index] = tl::make_unexpected(write_plan.error());   // 直接落到该槽位，后续不再被覆盖
            return;   // 注意：不设 needs_fallback，这类错误换槽位也没用
        }
        item.write_plan.emplace(std::move(write_plan.value()));   // optional::emplace，避免 WritePlan 再拷贝一次
    };

    struct StoreContext {   // chunk 请求的"我是谁"：哪个 plan 的第几个 chunk
        size_t plan_index = 0;
        size_t chunk_index = 0;
    };
    const auto apply_chunk_results =
        [&prepared](std::vector<NvmeKvCommandExecutor::StoreRequest> &requests,
                    const std::vector<StoreContext> &contexts) {   // requests 与 contexts 必须同序同长
            for (size_t request_index = 0; request_index < requests.size();
                 ++request_index) {
                const auto &context = contexts[request_index];
                auto &item = prepared[context.plan_index];
                if (requests[request_index].result) {   // 该 chunk 写成功
                    item.written_chunk_keys.push_back(
                        item.write_plan->chunk_values[context.chunk_index]
                            .first);   // 记下物理 key（.first），失败时按它删
                } else {   // 任一 chunk 失败
                    item.needs_fallback = true;   // 批量路径不在这里报错，统一降级给 OffloadOne
                }
            }
        };
```

**这一段在做什么**
- 给每个对象分配一个"过程状态" `PreparedOffload`，它跨越 chunk 阶段和 root 阶段，记录已写 chunk、root 是否已写、是否要回落。
- `prepare_one` 做两件事：把 `vector<Slice>` 变成一个连续的 `string_view`（单 Slice 零拷贝），再调 `BuildWritePlan(..., slot=0, ...)` 算出 root key / chunk key / 内联与否。
- `apply_chunk_results` 是批量写之后的"对账"：成功的 chunk 记进 `written_chunk_keys`，失败的只置 `needs_fallback`，不产生错误码。
- prepare 阶段的**结构性错误**（payload 过大）写进 `results[index]`，这类对象不会进入回落，直接判失败。

**重点语法 / 惯用法**
- `std::string_view` 不持有内存：`item.payload` 在多 Slice 时指向同一结构体里的 `payload_storage`，两者生命周期绑定，谁先析构谁出问题。
- `std::vector<PreparedOffload> prepared(plans.size())` 是**定长**的。一旦这个 vector 扩容或搬移，`payload_storage` 里的字符会换地址，`payload` 和 `chunk_values` 里的 string_view 立刻悬空——所以后面只有下标访问，没有 `push_back`。
- `std::optional<T>::emplace(std::move(...))`：在 optional 内部原地构造，比 `item.write_plan = std::move(x)` 少一次移动赋值。
- `tl::expected` 可以整体赋值为 `tl::make_unexpected(code)`，用于把某一条 lane 的结果扭成错误，而不抛出。
- `apply_chunk_results` 用**平行数组** `requests`/`contexts` 传参，靠下标一一对应；这是因为 `StoreBatch` 按原下标回填 `result`，不能重排。

**与 6.7.3 的差异**
- 图上第 13 步只有一次 `StoreBatch`，看不到"prepare"这个纯 CPU 阶段；实际上它可能并行执行（`prepare_concurrency_`），且在写之前就把所有 key 都算好了。
- 图上没有"slot"概念：批量路径写死了 `slot=0`，其余 63 个槽位只在回落路径出现（见 3.6）。
- 图上 chunk 与 root 是同一个动作的两半，代码里它们是两套独立的数据结构（`chunk_values` vs `root_blob`），失败后的处置也不同。
- `written_chunk_keys` 是回滚清单：图上的失败路径没有画出"只删本次写的 chunk"这一约束，这正是幂等性的来源。

mooncake-store/src/nvme_kv/key_conflict_policy.cpp:36-49,69-95

```cpp
    if (payload.size() >
        static_cast<size_t>(std::numeric_limits<uint32_t>::max())) {   // blob 头里的 payload_size 是 uint32
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }

    WritePlan plan;
    plan.identity = identity;   // 逻辑 key 原样带进 plan
    plan.root_key = EncodeNvmeKvPhysicalKey(identity, slot);   // 16B 物理 key = hash(role=root, slot, logical_key)

    const auto root_identity_metadata = SerializeNvmeKvStoredIdentity(
        identity, BuildNvmeKvStoredIdentityMetadata(plan.root_key, slot));   // 把 resolved key + slot 写进对象自身，供读路径校验归属
    const uint32_t inline_payload_limit = PayloadLimitForIdentityMetadata(
        max_value_size, static_cast<uint32_t>(root_identity_metadata.size()));   // 扣除 header 和 identity 后还能放多少 payload
    plan.store_inline = payload.size() <= inline_payload_limit;   // 小对象直接内联进 root，不产生任何 chunk
    // ...（略：store_inline 为真时，把 header + identity + payload 打包成单个 root_blob 并直接 return）
    const uint32_t chunk_payload_limit = max_value_size;   // chunk 的值是裸 payload（无 header），可以吃满设备上限
    if (chunk_payload_limit == 0) {
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    const size_t chunk_count =
        (payload.size() + chunk_payload_limit - 1) / chunk_payload_limit;   // 向上取整
    if (chunk_count == 0 ||
        chunk_count >
            static_cast<size_t>(std::numeric_limits<uint32_t>::max())) {   // manifest 里的 chunk_count 也是 uint32
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }

    plan.manifest_records.reserve(chunk_count);
    plan.chunk_values.reserve(chunk_count);
    for (size_t chunk_index = 0; chunk_index < chunk_count; ++chunk_index) {
        const size_t offset = chunk_index * chunk_payload_limit;
        const size_t chunk_size = std::min(
            static_cast<size_t>(chunk_payload_limit), payload.size() - offset);
        std::string_view chunk_payload(payload.data() + offset, chunk_size);   // 子视图，仍然零拷贝
        const auto chunk_key = EncodeNvmeKvChunkPhysicalKey(
            identity, static_cast<uint32_t>(chunk_index), slot);   // chunk key 同时掺入 chunk_index 和 slot
        const uint32_t chunk_checksum =
            ComputeNvmeKvPayloadChecksum(chunk_payload);   // 每个 chunk 单独校验和，写进 manifest
        plan.chunk_values.emplace_back(chunk_key, chunk_payload);   // value 是 string_view → 写请求不需要拷贝数据
        plan.manifest_records.push_back(NvmeKvManifestChunkRecord{
            chunk_key, static_cast<uint32_t>(chunk_size), chunk_checksum});
    }
```

**这一段在做什么**
- `BuildWritePlan` 把"一个逻辑对象 + 一个 slot"确定性地映射成一组物理 key：1 个 root key + N 个 chunk key。
- 判断走内联还是分块：payload 装得进单个 value 就直接内联；否则切成 `max_value_size` 大小的 chunk，root 里放 manifest。
- 同时产出 `chunk_values`（写请求用的 key/value 视图）和 `manifest_records`（读路径用来定位 chunk 的清单）。
- 所有物理 key 都由 `(logical_key, slot)` 派生，slot 变了整套 key 就全变——这是 3.6 逐槽位重试能成立的前提。

**重点语法 / 惯用法**
- `EncodeNvmeKvPhysicalKey(identity, slot)` 返回 `std::array<uint8_t, 16>`，是**内容哈希**而不是路径；同一对象每次算出来都一样，所以写入天然是"按内容寻址"。
- `plan.chunk_values` 的元素类型是 `std::pair<PhysicalKey, std::string_view>`，value 指向调用方传入的 `payload`，`WritePlan` 本身不拥有数据。
- `std::string_view chunk_payload(payload.data() + offset, chunk_size)`：切片只移动指针+长度，不复制。
- 三处 `uint32_t` 上限检查（payload 总长、chunk_count、单值大小）对应 blob 头里三个 uint32 字段，缺一个就会在序列化时截断。

**与 6.7.3 的差异**
- 图上第 13 步只画了 `StoreBatch`，看不到"内联对象根本不会产生 chunk"这个分支：小对象一次 `Store` 就结束。
- 图上 root 里"存 manifest"是一句话，代码里 manifest 是有长度上限的：`manifest_payload + header + identity > max_value_size` 时直接判 `INVALID_PARAMS`。
- 图上没说 slot；这里 `slot` 作为参数贯穿 root key 与 chunk key，`slot=0` 只是批量路径传的实参。
- chunk 的 value 是**裸 payload**（不带 header），所以 chunk 上限是 `max_value_size`，而 root 的上限要先扣掉 header 和 identity 元数据。

### 3.5 BatchOffload 的两条写流水线（第 13 步）

mooncake-store/src/nvme_kv/backend.cpp:597-657

```cpp
    bool roots_pipelined = false;   // 标记 root 是否已由流水线写掉；后面 862 的串行 root 阶段据此整体跳过
    const auto store_chunks = [&]() {   // 整个 chunk 写入阶段封装成一个 lambda，内部分"串行简化版"和"流水线版"
        if (io_workers_ == nullptr || submit_workers_ == nullptr ||
            root_submit_workers_ == nullptr || batch_submit_concurrency_ <= 1) {   // 任一线程池没建（并发=1）→ 退化
            RunParallelIo(plans.size(), prepare_one, prepare_concurrency_);   // 先并行准备所有 plan（限流 prepare_concurrency_）
            std::vector<NvmeKvCommandExecutor::StoreRequest> chunk_requests;   // 所有 plan 的 chunk 请求摊平到一个数组
            std::vector<StoreContext> chunk_contexts;   // 与 chunk_requests 同序的上下文
            for (size_t plan_index = 0; plan_index < plans.size();
                 ++plan_index) {
                auto &item = prepared[plan_index];
                if (!item.write_plan || item.write_plan->store_inline) {   // prepare 失败 或 内联对象
                    continue;   // 没有 chunk 要写（后者只写 root）
                }
                for (size_t chunk_index = 0;
                     chunk_index < item.write_plan->chunk_values.size();
                     ++chunk_index) {
                    const auto &[chunk_key, chunk_value] =
                        item.write_plan->chunk_values[chunk_index];
                    NvmeKvCommandExecutor::StoreRequest request;
                    request.key = chunk_key;   // 16B 物理 key
                    request.value = chunk_value;   // string_view，指向 prepared 里的 payload，不拷贝
                    chunk_requests.push_back(request);
                    chunk_contexts.push_back(
                        StoreContext{plan_index, chunk_index});   // 记下归属，回填结果时用
                }
            }
            StoreBatchParallel(chunk_requests);   // 整批发出去（内部再按 batch_submit_concurrency_ 分片）
            apply_chunk_results(chunk_requests, chunk_contexts);   // 回填 written_chunk_keys / needs_fallback
            return;   // 串行路径到此结束，root 交给 868 起的串行阶段
        }

        const size_t lane_count =
            std::min(batch_submit_concurrency_, plans.size());   // 车道数不超过对象数，避免空转线程
        const size_t root_lane_count =
            std::min(root_submit_concurrency_, plans.size());   // root 流水线独立车道数
        roots_pipelined = true;   // 关键状态位：root 将由本流水线写，后面的串行 root 阶段必须跳过
        std::vector<std::unique_ptr<IndexQueue>> queues;   // 每车道一条索引队列；unique_ptr 因为 IndexQueue 含 mutex，不可移动
        queues.reserve(lane_count);
        for (size_t lane = 0; lane < lane_count; ++lane) {
            queues.push_back(std::make_unique<IndexQueue>());
        }

        const auto close_queues = [&]() {   // 统一"停止收单"：Close 后 Push 返回 false、Pop 排空后返回 false
            for (auto &queue : queues) {
                queue->Close();
            }
        };
        IndexQueue root_queue;   // root 流水线只有一条队列（plan 粒度，不按键分片）
        std::mutex consumer_error_mutex;   // 多个消费者线程可能同时进 catch
        std::exception_ptr consumer_error;   // 只保留第一个异常，其余丢弃
        const auto set_consumer_error = [&]() {
            std::lock_guard<std::mutex> lock(consumer_error_mutex);
            if (consumer_error == nullptr) {
                consumer_error = std::current_exception();   // 必须在 catch 块内调用，否则取到 nullptr
            }
            close_queues();   // 关队列 → 其它消费者 Pop 返回 false 退出，避免死等
            root_queue.Close();
        };
        std::latch consumers_done(static_cast<std::ptrdiff_t>(lane_count));   // C++20 latch：计数归零即放行，一次性
        std::latch root_consumers_done(
            static_cast<std::ptrdiff_t>(root_lane_count));
```

**这一段在做什么**
- 先按"线程池是否建好 + `batch_submit_concurrency_ > 1`"分流：不满足就走串行简化路径，一次性把所有 chunk 摊平后 `StoreBatchParallel`。
- 流水线路径建立 `lane_count` 条 chunk 队列和 1 条 root 队列：chunk 队列按 `plan_index % lane_count` 分片，root 队列收集"chunk 全写完"的对象。
- 置位 `roots_pipelined = true`，让后面的串行 root 阶段不要重复写 root。
- 同步原语全是 `std::latch`（等所有消费者退出）+ `std::exception_ptr`（跨线程传异常）+ `IndexQueue::Close()`（可等的关闭信号）。

**重点语法 / 惯用法**
- `std::latch` 是 C++20 的一次性倒计时门闩：构造时给计数，每个消费者 `count_down()`，主线程 `wait()`；不能重复使用。
- `std::current_exception()` 只能在 `catch` 块内取到有效值，所以 `set_consumer_error` 被设计成"在 catch 里调用"，它内部不 `throw`。
- `IndexQueue` 用 `condition_variable::wait(lock, pred)`：谓词是 `closed_ || !queue_.empty()`，所以 `Close()` 之后不会卡在空队列上。
- `std::vector<std::unique_ptr<IndexQueue>>`：`IndexQueue` 内嵌 `std::mutex`，不可拷贝也不可移动，只能存指针。
- `set_consumer_error` 先加锁再关闭队列：关闭动作本身线程安全，但"只记录第一个异常"必须与它一起做，避免首个异常被后到的覆盖。

**与 6.7.3 的差异**
- 图上第 13 步是"一条 `StoreBatch`"，代码里可能是 1 次（串行路径）或"每条 lane 多次"（`flush_chunks` 按 `queue_depth` 分批）。
- 图上没有 `lane` 概念；chunk 的分片依据是 `plan_index % lane_count`，同一对象的 chunk 一定落在同一条 lane。
- 图上"chunk 先、root 后"暗示了全局顺序，实际流水线中 root 写可以与**其它对象**的 chunk 写交错进行（只保证同一对象内 chunk 全成功后才推 root）。
- 图上没有异常路径；这里任何消费者抛异常都会 `Close` 所有队列并让主线程 `rethrow`。

mooncake-store/src/nvme_kv/backend.cpp:659-721

```cpp
        size_t enqueued_root_consumers = 0;   // 已成功投递的 root 消费者数，异常时用来补计数
        try {
            for (; enqueued_root_consumers < root_lane_count;
                 ++enqueued_root_consumers) {
                root_submit_workers_->enqueue([&]() {   // 引用捕获：主线程会 wait 到 latch 归零，栈帧仍然有效
                    try {
                        auto connector = connector_;   // 拷贝 shared_ptr，避免中途被替换
                        if (connector == nullptr) {
                            throw std::runtime_error(
                                "NVMe KV connector is unavailable");   // 用异常统一走下面的 catch
                        }
                        const size_t queue_depth = std::max<size_t>(
                            1, connector->GetCapabilities().queue_depth);   // 设备队列深度，0 也当 1 用
                        std::vector<NvmeKvCommandExecutor::StoreRequest>
                            requests;   // 攒批缓冲
                        std::vector<size_t> contexts;   // 与 requests 同序的 plan_index
                        requests.reserve(queue_depth);
                        contexts.reserve(queue_depth);
                        const auto flush = [&]() {   // 达到 queue_depth 或收尾时发一批
                            if (requests.empty()) {
                                return;   // 空批不发
                            }
                            connector->StoreBatch(requests);   // 同步接口：返回时每个 request.result 已填好
                            for (size_t index = 0; index < requests.size();
                                 ++index) {
                                auto &item = prepared[contexts[index]];
                                if (requests[index].result) {
                                    item.root_written = true;   // root 写成功 = 该对象已提交
                                } else {
                                    item.needs_fallback = true;   // root 失败 → 整对象回落重试
                                }
                            }
                            requests.clear();   // clear 不释放容量，下一轮继续复用
                            contexts.clear();
                        };

                        size_t plan_index = 0;
                        while (root_queue.Pop(plan_index)) {   // 阻塞等；队列关闭且排空时返回 false
                            auto &item = prepared[plan_index];
                            NvmeKvCommandExecutor::StoreRequest request;
                            request.key = item.write_plan->root_key;
                            request.value = item.write_plan->root_blob;   // root_blob 归 PreparedOffload 所有
                            requests.push_back(request);
                            contexts.push_back(plan_index);
                            if (requests.size() == queue_depth) {   // 攒满一个设备队列就发
                                flush();
                            }
                        }
                        flush();   // 排空最后不足一批的请求
                    } catch (...) {
                        set_consumer_error();   // 记录首个异常并关闭所有队列，让同伴退出
                    }
                    root_consumers_done.count_down();   // 无论成败都要倒数，否则主线程永久阻塞
                });
            }
        } catch (...) {   // 线程池 enqueue 失败（例如已停机）
            close_queues();
            root_queue.Close();
            root_consumers_done.count_down(static_cast<std::ptrdiff_t>(
                root_lane_count - enqueued_root_consumers));   // 补上没投递成功的计数
            root_consumers_done.wait();   // 等已投递的消费者退出，避免它们引用已析构的栈变量
            throw;   // 原样上抛
        }
```

**这一段在做什么**
- 启动 root 消费者：每个消费者从 `root_queue` 里 `Pop` 出 `plan_index`，把该对象的 `root_blob` 攒成一批，攒满 `queue_depth` 就 `StoreBatch`。
- 每批返回后立刻对账：成功的置 `root_written = true`，失败的置 `needs_fallback = true`。
- 异常处理分两层：消费者内部异常 → `set_consumer_error()` 关掉所有队列；`enqueue` 本身失败 → 自己补计数、等已投递者退出再 rethrow。
- 注意 root 消费者是**先于** chunk 消费者启动的：它空转等队列，保证后面 chunk 消费者推 root 时一定有人接。

**重点语法 / 惯用法**
- `while (root_queue.Pop(plan_index))` 把"队列关闭"编码成 `false`，循环自然结束，不需要额外的停止标志。
- lambda 用 `[&]` 捕获全部局部变量（含 `prepared`、`root_queue`）；安全性依赖主线程在函数返回前 `wait()` 所有 latch。
- `count_down(n)` 是 latch 的批量倒数，用于 enqueue 失败时补齐未投递的计数，避免主线程死等。
- `requests.clear()` 保留 `capacity`，配合 `reserve(queue_depth)` 让每批都复用同一块内存。
- `throw std::runtime_error(...)` 把"connector 为空"这种非预期状态转成异常，统一走 catch 分支，省掉一堆 return 路径。

**与 6.7.3 的差异**
- 图上 root 是"每个对象一次写"，代码里是**按设备队列深度攒批**：多个对象的 root 可能共用一次 `StoreBatch`。
- 图上没有 root 消费者的启动顺序；实际先启 root、后启 chunk，是为了避免 chunk 消费者推入时无人消费而阻塞。
- 图上没有"root 写失败"的分支：失败只会置 `needs_fallback`，不会让整批失败。
- enqueue 失败时主线程会 `wait()` 已投递的消费者，这是防止悬垂引用的关键，图上完全看不到。

mooncake-store/src/nvme_kv/backend.cpp:723-795

```cpp
        size_t enqueued_consumers = 0;   // 已投递的 chunk 消费者数
        try {
            for (; enqueued_consumers < lane_count; ++enqueued_consumers) {
                submit_workers_->enqueue([&, lane = enqueued_consumers]() {   // init-capture：每个线程固定绑定一条 lane
                    try {
                        auto connector = connector_;
                        if (connector == nullptr) {
                            throw std::runtime_error(
                                "NVMe KV connector is unavailable");
                        }
                        const size_t queue_depth = std::max<size_t>(
                            1, connector->GetCapabilities().queue_depth);
                        std::vector<NvmeKvCommandExecutor::StoreRequest>
                            chunk_requests;
                        std::vector<StoreContext> chunk_contexts;
                        chunk_requests.reserve(queue_depth);
                        chunk_contexts.reserve(queue_depth);
                        const auto flush_chunks = [&]() {   // chunk 攒批发送 + 判定是否可以推 root
                            if (chunk_requests.empty()) {
                                return;
                            }
                            connector->StoreBatch(chunk_requests);
                            apply_chunk_results(chunk_requests, chunk_contexts);   // 先对账，再判 root
                            for (const auto &context : chunk_contexts) {
                                auto &item = prepared[context.plan_index];
                                if (context.chunk_index + 1 ==
                                        item.write_plan->chunk_values.size() &&   // 是该对象的最后一个 chunk
                                    !item.needs_fallback) {   // 且此前没有失败
                                    if (!root_queue.Push(context.plan_index)) {   // 推给 root 流水线
                                        item.needs_fallback = true;   // 队列已关（同伴出错）→ 降级
                                    }
                                }
                            }
                            chunk_requests.clear();
                            chunk_contexts.clear();
                        };

                        size_t plan_index = 0;
                        while (queues[lane]->Pop(plan_index)) {   // 阻塞取本 lane 的下一个对象
                            auto &item = prepared[plan_index];
                            for (size_t chunk_index = 0;
                                 chunk_index <
                                 item.write_plan->chunk_values.size();
                                 ++chunk_index) {
                                const auto &[chunk_key, chunk_value] =
                                    item.write_plan->chunk_values[chunk_index];
                                NvmeKvCommandExecutor::StoreRequest request;
                                request.key = chunk_key;
                                request.value = chunk_value;   // 仍是 string_view，零拷贝
                                chunk_requests.push_back(request);
                                chunk_contexts.push_back(
                                    StoreContext{plan_index, chunk_index});
                                if (chunk_requests.size() == queue_depth) {   // 攒满就发
                                    flush_chunks();
                                }
                            }
                        }
                        flush_chunks();   // 排空剩余
                    } catch (...) {
                        set_consumer_error();
                    }
                    consumers_done.count_down();
                });
            }
        } catch (...) {
            close_queues();
            root_queue.Close();
            consumers_done.count_down(
                static_cast<std::ptrdiff_t>(lane_count - enqueued_consumers));
            consumers_done.wait();
            root_consumers_done.wait();   // root 消费者也必须等，否则它们可能引用已析构的栈
            throw;
        }
```

**这一段在做什么**
- 每条 lane 一个消费者线程，从自己的 `IndexQueue` 取对象，按顺序把该对象的全部 chunk 摊进攒批缓冲，攒满 `queue_depth` 就发。
- 每批发完立刻 `apply_chunk_results` 对账；随后扫描本批上下文，若某条记录是该对象**最后一个 chunk** 且该对象没有失败，就把 `plan_index` 推进 `root_queue`。
- 这就是"chunk 先、root 后"的落地点：`root_queue.Push` 只发生在最后一个 chunk 的写结果返回之后。
- 收尾同样是关队列 + latch；异常时还要多等一个 `root_consumers_done`，因为 root 线程也可能在跑。

**重点语法 / 惯用法**
- init-capture `[&, lane = enqueued_consumers]()`：按值捕获循环变量，避免所有线程共享同一个 `lane`（经典坑）。
- `StoreBatch` 是同步的（头文件注释写明"实现必须在返回前访问完所有 request 持有的 buffer"），所以 `flush_chunks` 内可以安全地读 `result` 并清空缓冲。
- `apply_chunk_results` 在 `flush_chunks` 内部被调用，和串行路径共用同一份对账逻辑，保证两条路径语义一致。
- `root_queue.Push` 返回 `bool`：`Close` 之后返回 `false`，这里把它翻译成 `needs_fallback`，而不是抛异常。

**与 6.7.3 的差异**
- 图上第 13 步之后紧跟第 14 步（Notify），代码里 root 入队是**在 chunk 结果回调里**发生的（`flush_chunks` 内），不是独立一步。
- 图上"chunk 先、root 后"是对象内部顺序，实际一个 `flush_chunks` 批次里可能混着多个对象的尾部 chunk，root 推入也因此交错。
- 判"最后一个 chunk"用的是 `context.chunk_index + 1 == chunk_values.size()`，依赖 chunk 按 index 递增入队；图上没有这个隐含前提。
- 队列被提前关闭（同伴出错）时，本来能成功的对象也会被置 `needs_fallback`，这是图上看不到的"连带降级"。

mooncake-store/src/nvme_kv/backend.cpp:797-830

```cpp
        std::exception_ptr producer_error;   // producer 阶段的异常先存起来，等消费者收尾后再抛
        try {
            RunParallelIo(
                plans.size(),
                [&](size_t index) {   // 每个对象一个任务
                    prepare_one(index);   // 准备 payload + write_plan（可能并行执行）
                    auto &item = prepared[index];
                    if (!item.write_plan) {   // prepare 失败：results 已经填好错误
                        return;
                    }
                    const bool queued =
                        item.write_plan->store_inline
                            ? root_queue.Push(index)   // 内联对象没有 chunk，直接进 root 流水线
                            : queues[index % lane_count]->Push(index);   // 否则按取模分到某条 lane
                    if (!queued) {   // 队列已关（通常是别的线程出错）
                        results[index] =
                            tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
                    }
                },
                prepare_concurrency_);   // 限制同时 prepare 的对象数
        } catch (...) {
            producer_error = std::current_exception();   // 先接住，不能让消费者线程悬在队列上
        }
        close_queues();   // chunk 侧停止收单
        consumers_done.wait();   // 等所有 chunk 消费者排空并退出
        root_queue.Close();   // chunk 全退出后才关 root 队列 → 保证不会有迟到的 Push
        root_consumers_done.wait();   // 等 root 消费者排空
        if (producer_error != nullptr) {
            std::rethrow_exception(producer_error);   // producer 异常优先抛出
        }
        if (consumer_error != nullptr) {
            std::rethrow_exception(consumer_error);   // 其次抛消费者异常
        }
    };   // store_chunks 结束：两种路径都保证返回时所有消费者线程已 join
```

**这一段在做什么**
- producer 并行遍历所有 plan：`prepare_one` 后把对象推进对应的队列（内联对象进 `root_queue`，大对象进 `queues[index % lane_count]`）。
- 关闭顺序被严格设计成"先 chunk 后 root"：只有所有 chunk 消费者退出，才可能没有新的 `Push` 进 `root_queue`，此时再 `Close` 才安全。
- 两处异常分开保存、按固定顺序抛出：producer 先于 consumer，避免把"队列被关"的次生异常当成根因。
- `Push` 失败的对象直接判 `INTERNAL_ERROR`，不会被误认为成功。

**重点语法 / 惯用法**
- `RunParallelIo(item_count, task, max_inflight)`：内部用 `std::atomic<size_t>` 分发下标 + `std::latch` 等待，`max_inflight` 同时也是线程数上限。
- 异常被存进 `std::exception_ptr` 而不是立即 `throw`：在"有后台消费者还在跑"的阶段抛出会让它们访问已析构的栈对象。
- `close_queues(); consumers_done.wait(); root_queue.Close(); root_consumers_done.wait();` 是固定的四步收尾，顺序本身就是正确性的一部分。
- `queues[index % lane_count]` 保证同一 `index` 永远落到同一条 lane，从而保证同一对象的 chunk 顺序。

**与 6.7.3 的差异**
- 图上没有 producer/consumer 结构；它看起来像"直接对每个对象写"，实际是"并行 prepare → 入队 → 多消费者写"。
- 内联对象在图上和普通对象一样走 `StoreBatch` 的 chunk 阶段，代码里它**跳过 chunk 队列**，直接进 root 队列。
- 入队失败会被记为 `INTERNAL_ERROR`（硬失败），而写失败只是 `needs_fallback`（可重试），两种失败语义不同，图上无法区分。
- 收尾顺序（chunk 关 → 等 → root 关 → 等）是为了避免悬垂引用，图上的第 13 步终点没有画这个 join。

mooncake-store/src/nvme_kv/backend.cpp:858-928

```cpp
    ErrorCode worker_error = ErrorCode::OK;   // 整段写入阶段的总错误（目前只会被置成 INTERNAL_ERROR）
    try {
        store_chunks();   // 执行上面两条流水线；异常在这里抛出

        for (size_t plan_index = 0; plan_index < plans.size(); ++plan_index) {
            if (prepared[plan_index].needs_fallback) {   // chunk 阶段失败的对象
                cleanup_new_writes(plan_index, false);   // 删掉它已写成功的 chunk；root 一定没写，delete_root=false
            }
        }

        std::vector<NvmeKvCommandExecutor::StoreRequest> root_requests;   // 串行路径的 root 请求
        std::vector<size_t> root_contexts;   // 与 root_requests 同序的 plan_index
        for (size_t plan_index = 0; plan_index < plans.size(); ++plan_index) {
            auto &item = prepared[plan_index];
            if (!item.write_plan || item.needs_fallback || roots_pipelined) {   // 三选一：没计划 / 已失败 / root 已由流水线写
                continue;
            }
            NvmeKvCommandExecutor::StoreRequest request;
            request.key = item.write_plan->root_key;
            request.value = item.write_plan->root_blob;
            root_requests.push_back(request);
            root_contexts.push_back(plan_index);
        }
        StoreBatchParallel(root_requests);   // 串行路径下，所有 root 一次性批量写（chunk 已全部完成）
        for (size_t request_index = 0; request_index < root_requests.size();
             ++request_index) {
            const size_t plan_index = root_contexts[request_index];
            auto &item = prepared[plan_index];
            if (!root_requests[request_index].result) {   // root 写失败
                item.needs_fallback = true;
                cleanup_new_writes(plan_index, false);   // 删掉该对象的 chunk，交给 OffloadOne 重来
                continue;
            }
            item.root_written = true;   // root 成功 = 对象已提交
        }

        for (size_t plan_index = 0; plan_index < plans.size(); ++plan_index) {
            auto &item = prepared[plan_index];
            if (!item.write_plan || item.needs_fallback || !item.root_written) {   // 只有"root 已写"才算成功
                continue;
            }
            item.root_written = false;   // 先清标记：下面 worker_error 清理时不能误删已提交的数据
            item.written_chunk_keys.clear();   // 同上，chunk 清单也清空
            results[plan_index] =
                OffloadResult{.stored = true, .inserted = true};   // 指定初始化器（C++20）
            if (!item.write_plan->store_inline) {   // 内联对象没有 manifest 可缓存
                CacheManifestAfterWrite(plans[plan_index].key,
                                        plans[plan_index].payload_size, 0,
                                        item.write_plan->manifest_records);   // 缓存 manifest，读路径可跳过 64 槽探测
            }
        }

        RunParallelIo(plans.size(), [&](size_t index) {   // 剩下的失败对象逐槽位重试
            if (!prepared[index].needs_fallback) {
                return;
            }
            results[index] = OffloadOne(plans[index].key, *plans[index].slices,
                                        plans[index].payload_size);   // 注意这里返回的是 expected，直接赋给槽位
        });
    } catch (const std::exception &error) {
        LOG(ERROR) << "NVMe KV batch offload worker failed: " << error.what();
        worker_error = ErrorCode::INTERNAL_ERROR;   // 任何未消化异常都升级为整批 INTERNAL_ERROR
    } catch (...) {
        LOG(ERROR) << "NVMe KV batch offload worker failed";
        worker_error = ErrorCode::INTERNAL_ERROR;
    }
    if (worker_error != ErrorCode::OK) {
        for (size_t index = 0; index < prepared.size(); ++index) {
            cleanup_new_writes(index, prepared[index].root_written);   // 只有 root 已写的才需要连 root 一起删
        }
    }
```

**这一段在做什么**
- `store_chunks()` 跑完两条流水线后，这里做三件收尾：清理失败对象的 chunk、补写串行路径的 root、把成功对象标记成 `stored/inserted`。
- 串行 root 阶段在 `roots_pipelined == true` 时整体被跳过（`continue` 的第三个条件），避免重复写。
- 成功提交时**先清空回滚清单**（`root_written=false`、`written_chunk_keys.clear()`），这样后面 catch 里的清理不会把已提交的数据删掉。
- 最后对 `needs_fallback` 的对象并行调用 `OffloadOne`，由它去试 slot 1..63；异常则整批降级成 `INTERNAL_ERROR` 并回滚。

**重点语法 / 惯用法**
- 指定初始化器 `OffloadResult{.stored = true, .inserted = true}`：C++20 特性，字段顺序必须与声明一致。
- 三个 `continue` 条件把"该不该写 root"编码成一次判断：`!write_plan || needs_fallback || roots_pipelined`。
- `results[index] = OffloadOne(...)`：`OffloadOne` 返回 `tl::expected<OffloadResult, ErrorCode>`，与槽位类型相同，可直接赋值（含错误分支）。
- `cleanup_new_writes(index, prepared[index].root_written)`：第二个参数是"要不要连 root 一起删"，只有真正写过 root 的对象才为 true。

**与 6.7.3 的差异**
- 图上第 13 步之后就直接是 Notify（第 14 步），看不到这条**第二条 root 通道**：串行路径的 root 是在 chunk 全写完后统一批量写的。
- `roots_pipelined` 决定了 root 写到底发生在流水线里还是这里，同一个函数有两套控制流，图上只画了一套。
- 回落不是"重试同一个 slot"，而是 `OffloadOne` 从 slot 0 开始重新扫 64 个槽位（见 3.6）。
- 清理的触发点有两处：逐对象失败时清一次，整批异常时再按 `root_written` 清一次；后者依赖前者已把清单清空，否则会重复删。

### 3.6 OffloadOne：槽位回落（第 13 步）

mooncake-store/src/nvme_kv/backend.cpp:975-1010

```cpp
tl::expected<NvmeKvStorageBackend::OffloadResult, ErrorCode>
NvmeKvStorageBackend::OffloadOne(const std::string &key,
                                 const std::vector<Slice> &slices,
                                 size_t payload_size) {   // 单对象、逐槽位的"慢路径"
    if (payload_size >
        static_cast<size_t>(std::numeric_limits<uint32_t>::max())) {   // 与批量的 int64 检查不同，这里卡的是 uint32
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    std::string payload_storage;   // 多 Slice 时的拼接缓冲
    const std::string_view payload =
        BuildPayloadView(slices, payload_size, payload_storage);   // 单 Slice 零拷贝；多 Slice 拷一次

    auto connector = connector_;   // 拷贝 shared_ptr，保证整个函数期间设备可用
    if (connector == nullptr) {
        return tl::make_unexpected(ErrorCode::OBJECT_NOT_FOUND);   // 用 NOT_FOUND 表示"设备没了"，上层看到的是"没存上"
    }
    const uint32_t effective_max_value_size =
        std::max(connector->GetCapabilities().effective_max_value_size,
                 kMinMaxValueSize);   // 设备上限太小会让 header 都放不下，兜底成 header+1

    for (uint32_t slot = 0; slot < kNvmeKvMaxPhysicalKeySlots; ++slot) {   // 64 个槽位依次尝试
        auto write_plan_res = NvmeKvKeyConflictPolicy::BuildWritePlan(
            NvmeKvObjectIdentity{.logical_key = key}, payload, slot,
            effective_max_value_size);   // 每个 slot 重新生成一整套 root/chunk key
        if (!write_plan_res) {   // 例如 manifest 装不进单值
            return tl::make_unexpected(write_plan_res.error());   // 结构性错误，换槽位也没用 → 直接返回
        }
        auto &write_plan = write_plan_res.value();
        const bool store_inline = write_plan.store_inline;   // 内联对象没有 chunk
        const auto &physical_key = write_plan.root_key;   // root 物理 key（本槽位）
        const auto &root_blob = write_plan.root_blob;   // root 的实际字节（内联时就是整份数据）
        const auto &chunk_store_values = write_plan.chunk_values;   // (chunk_key, string_view) 列表

        std::vector<PhysicalKey> written_chunk_keys;   // 本槽位已写成功的 chunk
        bool root_written = false;   // root 是否已写（root 最后写）
        const auto cleanup_new_writes = [&]() {   // 本槽位的回滚：只删"本次写成功的"
```

**这一段在做什么**
- 重新算一遍 payload 视图（与批量路径重复，但这条路径是独立的单对象入口），并把设备上限取成"至少能放下 header"。
- 进入 `slot = 0..63` 的循环，每个槽位都重新 `BuildWritePlan`：root key 和 chunk key 都会因为 slot 不同而完全改变。
- 为每个槽位准备独立的状态：`written_chunk_keys` 和 `root_written`，互不影响。
- `BuildWritePlan` 失败是结构性错误（payload/manifest 太大），直接返回，不做槽位轮换。

**重点语法 / 惯用法**
- `kNvmeKvMaxPhysicalKeySlots` 定义在 `include/nvme_kv/key_codec.h:18`，值是 `64`；它是一个"**并发写者上限**"而不是容量单位。
- `auto &write_plan = write_plan_res.value();` 取的是 expected 内部对象的引用，后续只读，不 move。
- `const auto &physical_key = write_plan.root_key;` 引用绑定到 plan 内部，`write_plan` 活到循环末尾，引用一直有效。
- 三个 `const auto &` 别名只是可读性糖，不复制 `root_blob`（可能是几百字节的 manifest）。

**与 6.7.3 的差异**
- 图上第 13 步只有一次 `StoreBatch`；这里每次回落都要重新规划一整套 key，且可能重复最多 64 次。
- 图上没有"槽位"这个维度；`slot` 同时掺进 root key 与 chunk key 的哈希，所以不同槽位之间完全没有覆盖关系。
- payload 校验用 `uint32_t` 上限而不是 `int64_t`，因为 blob 头的 `payload_size` 字段是 32 位。
- `connector_ == nullptr` 返回 `OBJECT_NOT_FOUND` 而不是 `INTERNAL_ERROR`，这是刻意的：上层只想区分"存上了/没存上"。

mooncake-store/src/nvme_kv/backend.cpp:1010-1054

```cpp
        const auto cleanup_new_writes = [&]() {   // 只回滚本槽位、本次写入的数据
            if (root_written) {   // root 写了才需要删 root
                auto delete_res = connector->Delete(physical_key);
                if (!delete_res &&
                    delete_res.error() != ErrorCode::OBJECT_NOT_FOUND) {   // 已经没了不算错误（幂等删除）
                    LOG(WARNING)
                        << "NVMe KV failed to clean orphan root for " << key
                        << " key=" << NvmeKvPhysicalKeyToHex(physical_key)
                        << " error=" << toString(delete_res.error());
                }
                root_written = false;   // 置回 false，二次调用不会重复删
            }
            if (written_chunk_keys.empty()) {
                return;   // 没有 chunk（内联对象）→ 直接结束
            }
            for (const auto &chunk_key : written_chunk_keys) {
                auto delete_res = connector->Delete(chunk_key);
                if (!delete_res &&
                    delete_res.error() != ErrorCode::OBJECT_NOT_FOUND) {
                    LOG(WARNING)
                        << "NVMe KV failed to clean orphan chunk for " << key
                        << " key=" << NvmeKvPhysicalKeyToHex(chunk_key)
                        << " error=" << toString(delete_res.error());
                }
            }
            written_chunk_keys.clear();   // 清空清单：避免重复删除
        };
        const auto resolve_existing_root =
            [&]() -> tl::expected<OffloadResult, ErrorCode> {   // "已有 root"到底是不是同一份数据
            auto decision_res = NvmeKvKeyConflictPolicy::ResolveExistingObject(
                *connector, physical_key, root_blob);   // 读出旧值并与本次 root_blob 逐字节比较
            if (!decision_res) {
                return tl::make_unexpected(decision_res.error());   // 读失败（不是"不存在"）
            }
            if (decision_res.value() ==
                NvmeKvKeyConflictPolicy::ExistingObjectDecision::kNotFound) {
                return OffloadResult{};   // 竞态：刚还在，现在没了 → 既不 stored 也不报错
            }
            if (decision_res.value() ==
                NvmeKvKeyConflictPolicy::ExistingObjectDecision::
                    kDifferentObject) {
                return OffloadResult{};   // 同槽位上已经是别的对象 → 不认领
            }
            return OffloadResult{.stored = true, .inserted = false};   // kSameObject：幂等命中，数据已经是这份
        };
```

**这一段在做什么**
- `cleanup_new_writes` 是本槽位的回滚器，边界非常明确：**只删 `root_written` 为真时的 root，只删 `written_chunk_keys` 里记录的 chunk**。
- 删除失败只在"不是 `OBJECT_NOT_FOUND`"时打 WARNING，因为并发下别人可能已经删了，这属于正常竞态。
- `resolve_existing_root` 把"root 已存在"翻译成三种语义：不存在（竞态）、不同对象（换槽）、相同对象（幂等成功）。
- 只有 `kSameObject` 会返回 `stored = true, inserted = false`，即"数据已在盘上，但不是我插入的"。

**重点语法 / 惯用法**
- 内部 lambda `[&]` 捕获 `root_written`、`written_chunk_keys`、`physical_key`、`connector`，通过引用直接修改外层状态。
- 显式返回类型 `-> tl::expected<OffloadResult, ErrorCode>`：lambda 有多个 `return` 且类型不同，必须写出来。
- `Delete` 的 `OBJECT_NOT_FOUND` 被当成成功：删除接口本身不做"存在性"前置检查，幂等语义由调用方兜。
- `NvmeKvPhysicalKeyToHex` 把 16 字节 key 打成 32 个十六进制字符，日志里能直接和 `EncodeNvmeKvPhysicalKey` 的派生结果对上。

**与 6.7.3 的差异**
- 图上没有任何回滚步骤；这里有两类清理：写 chunk 失败时清已写 chunk、root 冲突时清本次 chunk。
- 图上"已存在"是一种状态，代码里被拆成三种（`kNotFound` / `kSameObject` / `kDifferentObject`），只有最后一种是"换槽"的理由。
- `kNotFound` 也返回 `stored = false` 而不是错误：读到了"别人刚好删掉"的瞬间，不算失败。
- 清理永远不删**别人**写的数据：`written_chunk_keys` 只记录本次成功的 chunk，冲突的那个 chunk key 从不进清单。

mooncake-store/src/nvme_kv/key_conflict_policy.cpp:125-143

```cpp
tl::expected<NvmeKvKeyConflictPolicy::ExistingObjectDecision, ErrorCode>
NvmeKvKeyConflictPolicy::ResolveExistingObject(NvmeKvConnector& connector,
                                               const PhysicalKey& physical_key,
                                               std::string_view expected_blob) {
    auto existing_value_res = connector.Retrieve(physical_key);   // 整块读回（root 或 chunk）
    if (!existing_value_res) {
        if (existing_value_res.error() == ErrorCode::OBJECT_NOT_FOUND) {
            return ExistingObjectDecision::kNotFound;   // 不存在是正常判定，不是错误
        }
        return tl::make_unexpected(existing_value_res.error());   // 真正的读失败才上抛
    }
    const auto& existing = existing_value_res.value();
    if (existing.size() == expected_blob.size() &&
        std::memcmp(existing.data(), expected_blob.data(),
                    expected_blob.size()) == 0) {   // 长度相同 + 逐字节相同
        return ExistingObjectDecision::kSameObject;   // 幂等的依据：内容完全一致
    }
    return ExistingObjectDecision::kDifferentObject;   // 长度或内容不同 → 槽位被占
}
```

**这一段在做什么**
- 把 `store-if-not-exists` 失败后的"已存在"二值化：读回旧值，和本次要写的 blob 做**全字节比较**。
- 比较的是整个 blob（header + identity 元数据 + payload），不是只比 payload；因此"同 payload、不同 slot"也会被判为不同对象。
- `OBJECT_NOT_FOUND` 被降级成 `kNotFound` 而不是错误，覆盖"读之前被删"的竞态。
- 只有返回 `kSameObject` 时调用方才敢认定"数据已经在盘上"，从而跳过一次真实写入。

**重点语法 / 惯用法**
- `std::memcmp(a, b, n) == 0` 配合 `size()` 先行比较：先比长度可以避免读到越界。
- `std::string_view expected_blob` 是只读视图，比较期间不拷贝；调用方保证 `root_blob` / `chunk_blob` 还活着。
- 枚举 `ExistingObjectDecision` 有三态，把"不存在/相同/不同"从 `ErrorCode` 里分离出来，避免用错误码表达正常分支。
- 返回类型仍是 `tl::expected`：正常的三态判定和"设备读失败"被清楚区分开。

**与 6.7.3 的差异**
- 图上完全没有"读回来比一次"的动作；这是 NVMe KV 实现幂等的核心，代价是冲突路径上多一次 `Retrieve`。
- 图上"已存在"就结束，代码里"已存在但内容不同"必须继续找下一个槽位，否则会误把别人的对象当成自己的。
- 这个函数对 root 和 chunk 复用同一套逻辑（`OffloadOne` 里两处调用），图上没有区分两种粒度的冲突。
- 比较包含 header 里的 checksum 和 identity 元数据，因此"同 payload 但落在不同 slot"一定判不同——这正是槽位隔离的保证。

mooncake-store/src/nvme_kv/backend.cpp:1056-1128

```cpp
        bool write_success = true;   // 本槽位写入是否成功
        bool slot_collision = false;   // 是否是"槽位被别的对象占用"这类可换槽的失败
        ErrorCode write_error = ErrorCode::OK;   // 失败原因
        if (store_inline) {   // 小对象：整个对象就是 root
            auto store_res = connector->Store(physical_key, root_blob);   // 单条 store-if-not-exists
            if (!store_res) {   // 已存在会被映射成 OBJECT_ALREADY_EXISTS
                write_success = false;
                write_error = store_res.error();
            } else {
                root_written = true;   // 一步到位，对象已提交
            }
        } else {
            const auto handle_chunk_store_error =
                [&](const PhysicalKey &chunk_key, std::string_view chunk_blob,
                    ErrorCode error) -> bool {   // 返回 true 表示"这个错误可以忽略，继续写下一个 chunk"
                if (error == ErrorCode::OBJECT_ALREADY_EXISTS) {   // chunk key 已被占用
                    auto existing_chunk_res =
                        NvmeKvKeyConflictPolicy::ResolveExistingObject(
                            *connector, chunk_key, chunk_blob);   // 读回来比一次
                    if (existing_chunk_res &&
                        existing_chunk_res.value() ==
                            NvmeKvKeyConflictPolicy::ExistingObjectDecision::
                                kSameObject) {
                        return true;   // 内容一致 → 上次已经写成功过，跳过即可（幂等）
                    }
                    if (!existing_chunk_res) {
                        write_error = existing_chunk_res.error();   // 读失败：保留真实错误
                    } else {
                        write_error = ErrorCode::OBJECT_ALREADY_EXISTS;   // 内容不同：槽位冲突
                    }
                    slot_collision = true;   // 标记为"可换槽"
                } else {
                    write_error = error;   // 其它设备错误：不可换槽
                }
                write_success = false;
                cleanup_new_writes();   // 立刻回滚本槽位已写的 chunk
                return false;
            };
            std::vector<NvmeKvCommandExecutor::StoreRequest> chunk_requests;
            chunk_requests.reserve(chunk_store_values.size());
            for (const auto &[chunk_key, chunk_blob] : chunk_store_values) {
                NvmeKvCommandExecutor::StoreRequest request;
                request.key = chunk_key;
                request.value = chunk_blob;   // string_view，指向 payload
                chunk_requests.push_back(request);
            }
            connector->StoreBatch(chunk_requests);   // 一次把该对象所有 chunk 发出去（同步返回）

            for (size_t chunk_index = 0;
                 chunk_index < chunk_store_values.size(); ++chunk_index) {
                const auto &[chunk_key, chunk_blob] =
                    chunk_store_values[chunk_index];
                const auto &chunk_res = chunk_requests[chunk_index].result;
                if (!chunk_res) {   // 这个 chunk 没写成功
                    if (!handle_chunk_store_error(chunk_key, chunk_blob,
                                                  chunk_res.error())) {
                        break;   // 不可恢复或槽位冲突 → 停止后续 chunk
                    }
                    continue;   // 幂等命中 → 不算失败，继续下一个
                }
                written_chunk_keys.push_back(chunk_key);   // 记进回滚清单
            }
            if (write_success) {   // 所有 chunk 都成功（或被幂等跳过）后才写 root
                auto store_res = connector->Store(physical_key, root_blob);   // root 最后写 = 提交点
                if (!store_res) {
                    write_success = false;
                    write_error = store_res.error();
                    cleanup_new_writes();   // root 没写成，chunk 变成孤儿 → 删掉
                } else {
                    root_written = true;
                }
            }
        }
```

**这一段在做什么**
- 分两条写路径：内联对象一条 `Store` 就完成；大对象先 `StoreBatch` 写全部 chunk，全部成功后再 `Store` root。
- chunk 写失败时走 `handle_chunk_store_error`：只有"已存在且内容相同"可以忽略，其余都会终止本槽位并回滚已写 chunk。
- root 写在所有 chunk 成功之后，是对象的**提交点**：root 存在即代表 chunk 齐全。
- 失败时立刻 `cleanup_new_writes()`，保证不会留下只有一半 chunk 的孤儿数据。

**重点语法 / 惯用法**
- `handle_chunk_store_error` 返回 `bool` 并显式写 `-> bool`：既表达"能否继续"，又通过引用捕获修改 `write_success`/`write_error`，是"函数内多层状态"的常见写法。
- `StoreBatch` 的 `result` 是按请求下标回填的，所以下面用 `chunk_index` 同时索引 `chunk_store_values` 和 `chunk_requests`，隐含"两者同序"的约定。
- `StoreRequest.value` 是 `std::string_view`，`request.value = chunk_blob` 不复制 payload；前提是 `payload`/`payload_storage` 活到 `StoreBatch` 返回。
- `break` 与 `continue` 的语义被严格区分：冲突/硬错误 `break`（放弃本槽位），幂等命中 `continue`（继续写后面的 chunk）。

**与 6.7.3 的差异**
- 图上"chunk 先、root 后"是一次 `StoreBatch` 加一次写；代码里 chunk 是**一次 StoreBatch 发全部**，root 是单独一次 `Store`。
- 图上没有"chunk 已存在但内容相同"的分支：这种情况下代码继续往下写，最终 root 也会幂等命中。
- 图上失败就直接报错；这里 chunk 冲突只置 `slot_collision`，真正的决策在下面的 `1130` 起。
- 内联对象没有 chunk 阶段，图上把它和数据对象画成同一条路径。

mooncake-store/src/nvme_kv/backend.cpp:1130-1165

```cpp
        if (!write_success && write_error == ErrorCode::OBJECT_ALREADY_EXISTS) {   // 失败的根因是"已存在"
            auto existing_res = resolve_existing_root();   // 读回 root，判断是不是同一份数据
            if (existing_res && existing_res.value().stored) {   // kSameObject：幂等命中
                if (!store_inline) {
                    CacheManifestAfterWrite(key, payload_size, slot,
                                            write_plan.manifest_records);   // 补缓存 manifest，读路径可直接用
                }
                return existing_res;   // 返回 {stored=true, inserted=false}
            }
            if (!existing_res) {
                write_error = existing_res.error();   // 读失败：用真实错误覆盖 ALREADY_EXISTS
            } else {
                slot_collision = true;   // kNotFound 或 kDifferentObject → 当作槽位冲突
                cleanup_new_writes();   // 删掉我们刚写的 chunk（root 没写，不会误删别人的）
            }
        }

        if (!write_success) {
            if (!slot_collision && !store_inline) {   // 非冲突类失败（设备错误等）
                cleanup_new_writes();   // 再兜一次清理；清单已空时是空操作
            }
            written_chunk_keys.clear();   // 无论如何都清空，防止后续误删
            if (slot_collision) {
                continue;   // 换下一个 slot 重试
            }
            return tl::make_unexpected(write_error);   // 真错误：直接冒泡，不换槽
        }

        if (!store_inline) {
            CacheManifestAfterWrite(key, payload_size, slot,
                                    write_plan.manifest_records);   // 成功后缓存 manifest
        }
        return OffloadResult{.stored = true, .inserted = true};   // 真正由本次插入
    }
    return OffloadResult{};   // 64 个槽位全部冲突 → 静默放弃（stored=false），不是错误
}   // 循环外的出口：调用方拿到的 expected 始终"有值"，失败信息全在 OffloadResult 字段里
```

**这一段在做什么**
- `OBJECT_ALREADY_EXISTS` 的统一收口：读回 root 判三态。同一份数据 → 幂等成功；不同/不存在 → 当作槽位冲突，清掉自己的 chunk 后换槽。
- `kNotFound` 被当作冲突（`slot_collision = true`）而不是错误，因为"读之前被删"意味着这个槽位现在空着，可以再试。
- 非冲突类失败（真正的设备错误）不换槽，直接 `return tl::make_unexpected(write_error)` 冒泡。
- 循环走完 64 个槽位仍全部冲突时返回 `OffloadResult{}`（`stored=false`），上层把它当作"没存上"，不报错。

**重点语法 / 惯用法**
- `if (existing_res && existing_res.value().stored)`：先判 expected 有值，再判业务字段，避免对 unexpected 调 `value()`（UB）。
- `return existing_res;` 直接返回一个 `tl::expected`，利用隐式拷贝，不需要 `return existing_res.value()`。
- `cleanup_new_writes()` 被设计成**可重复调用**：`root_written=false`、`written_chunk_keys.clear()` 保证第二次是空操作。
- 循环外的 `return OffloadResult{}` 是"穷尽后放弃"的写法，用默认值表达"没插入"，与错误码区分开。

**与 6.7.3 的差异**
- 图上没有"64 个槽位全满"的结局；代码里它是**静默丢弃**（返回 `stored=false`，无错误码），除了日志之外很难察觉。
- 图上"写入成功"是一件事；这里"成功"分两种：`inserted=true`（本次插入）与 `inserted=false`（幂等命中），只影响配额统计。
- 幂等命中时也会 `CacheManifestAfterWrite`（非内联），因为盘上的 manifest 与本地的完全一致。
- 冲突清理只删本次写的 chunk，从不删 root——root 是别人写的，删了就破坏别人的对象。

### 3.7 NotifyOffloadSuccess：登记或更新 LOCAL_DISK（第 14–15 步）

`mooncake-store/src/master_service.cpp:8973-9159`

```cpp
auto MasterService::NotifyOffloadSuccess(
    const UUID& client_id, const std::vector<OffloadTaskItem>& tasks,
    const std::vector<StorageObjectMetadata>& metadatas)
    -> tl::expected<void, ErrorCode> {              // 尾置返回类型：成功是 void，失败是 ErrorCode（用返回值表达错误，不抛异常）
    if (tasks.size() != metadatas.size()) {         // tasks[i] 是 master 当初下发的任务（tenant/key/size）
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);  // metadatas[i] 是后端回填的落盘结果，两者必须一一对应
    }
    const auto record = FindClientRecord(client_id);  // 取进程内的 client liveness 记录（shared_ptr，可能为 nullptr）
    auto retaining_guard =
        record ? record->TryAcquireRetainingGuard() : std::nullopt;  // 三目两侧都是 optional<RetainingGuard>；拿"保留权"锁
    if (!retaining_guard) {                          // record 为空（未知 client）或该 client 已判 OFFLINE
        return tl::make_unexpected(ErrorCode::UNAVAILABLE_IN_CURRENT_STATUS);  // guard 在整个批次期间持锁，防止中途被判离线
    }
    // Set when an entry's replica registration is refused because the client
    // no longer has a LOCAL_DISK segment entry (see the per-entry checks
    // below). NACK cleanups still run for the rest of the batch; the caller
    // gets SEGMENT_NOT_FOUND so a rescan stops re-registering.
    bool refused_unmounted = false;                  // 批级"累计"标志：任一条被拒就置位，最后统一返回 SEGMENT_NOT_FOUND

    for (size_t i = 0; i < tasks.size(); ++i) {      // 单条失败不中断整批（NACK 清理要继续跑），只有硬错误才 return
        const auto& task = tasks[i];                 // 任务侧：tenant_id / key / size（size 在此函数里未使用）
        const auto& metadata = metadatas[i];         // 结果侧：bucket_id / offset / key_size / data_size / transport_endpoint
        const TenantId task_tenant = enable_multi_tenants_
                                         ? TenantId(task.tenant_id)   // 多租户打开：用任务里带的 tenant 原样还原
                                         : TenantId::Default();       // 关闭：一律收敛到 "default"，与写路径口径一致
        const auto request_object_id =
            MakeObjectIdentityForRequest(task.key, task_tenant);  // 得到 {归一化 tenant, user_key} 的复合身份

        // NACK sentinel: offload failed on worker. Clean up the
        // offloading_task + dec_refcnt but skip AddReplica.
        if (metadata.data_size < 0) {                // 哨兵：客户端用 data_size = -1 表示"这批 key 我取走了但没写成"
            std::shared_lock<std::shared_mutex> shared_lock(
                master_state_mutex_);                // 全局状态锁的共享模式：只读元数据表结构，禁止 mount/unmount 并发改表
            MetadataAccessorRW accessor(this, request_object_id);  // 内部再取该 key 所属 shard 的独占锁（要改 metadata）
            if (accessor.Exists()) {                 // 对象可能已被淘汰/删除，那就什么都不用做
                auto& offloading_task = accessor.GetState().offloading_task;  // 运行时状态与 metadata 同锁，随对象生灭
                if (offloading_task) {               // 只有确实还挂着 offload 任务才需要善后
                    auto source = accessor.Get().GetReplicaByID(
                        offloading_task->source_id); // 找回当初被 inc_refcnt 钉住的源副本（通常是 MEMORY）
                    if (source != nullptr) {         // 源副本可能已经随对象一起消失
                        source->dec_refcnt();        // 释放钉住，让它重新可被淘汰（对应第 5 步的 inc_refcnt）
                    }
                    offloading_task.reset();         // 清掉任务标记：该 key 不再"有 offload 在飞"
                    MasterMetricManager::instance().inc_offload_failed(
                        UuidToString(client_id));    // 失败计数按 client 维度上报
                }
            }
            continue;                                // 关键：NACK 只做清理，绝不登记副本
        }

        Replica replica(client_id, metadata.data_size,   // 走到这里说明后端确实写成功了，构造 LOCAL_DISK 副本
                        metadata.transport_endpoint, ReplicaStatus::COMPLETE,  // 直接 COMPLETE：数据已落盘，无需 PROCESSING 中间态
                        record);                     // 绑定 liveness record：之后 is_available() 取决于该 client 是否在服务
        bool handled_existing_object = false;        // 本条的"对象已存在"分支是否已处理完（决定要不要走 AddReplica 回退）
        bool added_new_local_disk_replica = false;   // 本次是否真的"新增"了副本（只有新增才计 used_bytes）
        {
            std::shared_lock<std::shared_mutex> shared_lock(
                master_state_mutex_);                // 块作用域控制持锁时长：出块即放，避免整批长期占读锁
            // A disk replica may only be registered for a client whose
            // LOCAL_DISK segment entry still exists. Checked inside this
            // shared-lock section, before the shard lock, so the check and
            // the write below cannot straddle UnmountLocalDiskSegment's
            // removal (which holds master_state_mutex_ exclusively): either the
            // replica lands first and its sweep erases it, or the check here
            // sees the segment gone and refuses. Without this, a
            // registration racing a deregistration -- an in-flight rescan
            // batch, or an offload completion from a heartbeat that was past
            // its own drain check -- could land after the sweep and leave
            // the master advertising a departed owner. Scoped to
            // enable_offload_, where the segment registry exists and a
            // deregistration can race; with the subsystem off both the mount
            // and unmount RPCs refuse, so there is nothing to check against
            // and no race to close.
            const bool segment_mounted =
                !enable_offload_ || HasMountedLocalDiskSegment(client_id);  // 准入判定：offload 关闭时恒真；开启时查 registry 里是否还有该 client 的 segment entry
            MetadataAccessorRW accessor(this, request_object_id);  // 再取 shard 独占锁；注意"检查 segment"与"写副本"同处一个 shared_lock 区间
            if (accessor.Exists()) {                 // 对象还存在（可能是 PutEnd 之后、也可能是重启后的 rescan 重新发现）
                auto& obj_metadata = accessor.Get();  // 可写引用的对象元数据
                const auto previous_kv_media = KvMediaSnapshot(obj_metadata);  // 变更前快照介质集合；kv 事件未开启时返回空 vector，零成本
                auto& offloading_task = accessor.GetState().offloading_task;  // 本 key 在飞的 offload 任务标记
                if (offloading_task &&
                    replica.type() != ReplicaType::LOCAL_DISK) {  // 防御性断言：有任务时只允许登记 LOCAL_DISK
                    LOG(ERROR) << "Invalid replica type: " << replica.type()
                               << ". Expected ReplicaType::LOCAL_DISK.";
                    return tl::make_unexpected(ErrorCode::INVALID_PARAMS);  // 直接中止整批（RAII 保证两把锁都会释放）
                }

                // Existing orphan objects can only bypass tenant registration
                // for a master-admitted offload completion. Without this task
                // marker, fall through to the regular registration check.
                if (offloading_task) {               // 有任务标记 = 这是一次"master 认可的"offload 完成
                    auto source =
                        obj_metadata.GetReplicaByID(offloading_task->source_id);  // 找到被钉住的源副本
                    if (source != nullptr) {
                        source->dec_refcnt();        // 成功路径同样要 dec_refcnt：源 MEMORY 副本从此刻起可被淘汰
                    }
                    offloading_task.reset();         // 任务标记无论登记成功与否都清掉（避免 TTL 回收器重复处理）

                    if (!segment_mounted) {          // segment 已被 UnmountLocalDiskSegment 摘掉
                        // The offload bookkeeping above still ran; only the
                        // registration is refused.
                        refused_unmounted = true;    // 只拒绝"登记"，不拒绝上面已做的记账
                    } else if (!obj_metadata.HasReplica(
                                   &Replica::fn_is_local_disk_replica)) {  // 该对象当前没有任何 LOCAL_DISK 副本 = 首次登记
                        std::vector<Replica> replicas;
                        replicas.emplace_back(std::move(replica));  // move 进临时 vector（Replica 不可拷贝，只能移动）
                        obj_metadata.AddReplicas(std::move(replicas));  // 真正挂到对象上，成为可读副本
                        accessor.OnDiskReplicaAdded(obj_metadata);  // 若这是首个 completed LOCAL_DISK，则 shard.disk_object_count++
                        SyncCacheTotalAccounting(obj_metadata);  // 同步 file_cache_nums 等总量指标（幂等，只在 0→1 / 1→0 时增减）
                        MasterMetricManager::instance().inc_offload_completed(
                            UuidToString(client_id));  // 成功计数：只在"新增"路径加，更新路径不加
                        added_new_local_disk_replica = true;  // 标记新增，后面据此调整 used_bytes
                    } else {                         // 已有 LOCAL_DISK 副本：不新增，只把 endpoint/大小刷新成最新
                        obj_metadata.VisitReplicas(
                            [client_id](const Replica& rep) {  // pred：只看 LOCAL_DISK 且属于同一个 client 的那个副本
                                return rep.type() == ReplicaType::LOCAL_DISK &&
                                       rep.get_descriptor()
                                               .get_local_disk_descriptor()
                                               .client_id == client_id;
                            },
                            [&replica](Replica& rep) {  // visit：原地改写（VisitReplicas 的非 const 重载允许修改副本）
                                rep.get_descriptor()
                                    .get_local_disk_descriptor()
                                    .transport_endpoint =
                                    replica.get_descriptor()
                                        .get_local_disk_descriptor()
                                        .transport_endpoint;  // 刷新 owner 的 offload RPC 地址（进程重启后可能变）
                                rep.get_descriptor()
                                    .get_local_disk_descriptor()
                                    .object_size =
                                    replica.get_descriptor()
                                        .get_local_disk_descriptor()
                                        .object_size;  // 刷新对象大小（重扫时以最新数据为准）
                            });
                    }
                    handled_existing_object = true;  // 对象已存在且已处理完，下面不再走 AddReplica 回退
                    SyncKvObjectState(request_object_id.user_key, obj_metadata,
                                      request_object_id.tenant_id,
                                      previous_kv_media);  // 发布 kv 状态变更（介质集合可能从 memory 变成 memory+local_disk）
                }
            }
        }

        if (!handled_existing_object) {              // 对象不存在、或存在但没有 offloading_task（例如重启后 rescan 发现的孤儿对象）
            auto normalized_tenant_result =
                ResolveTenantIdForWrite(request_object_id.tenant_id);  // 写路径专用的 tenant 归一化（未注册租户会在这里报错）
            if (!normalized_tenant_result) {
                return tl::make_unexpected(normalized_tenant_result.error());  // tenant 不合法 = 硬错误，整批中止
            }
            const ObjectIdentity object_id{
                std::move(normalized_tenant_result.value()),
                request_object_id.user_key};     // 用归一化后的 tenant 重新拼对象身份
            auto res = AddReplicaForRetainedClient(
                client_id, object_id.user_key, object_id.tenant_id, replica);  // 复用的登记入口：必要时会 CreateKey 新建 metadata
            if (!res) {                              // 回退路径也会做同样的 segment_mounted 检查
                if (res.error() == ErrorCode::OBJECT_NOT_FOUND) {
                    continue;                        // 对象在两步之间被删掉了：跳过本条，不视为错误
                }
                if (res.error() == ErrorCode::SEGMENT_NOT_FOUND) {
                    // AddReplica's own mounted-segment check refused it (the
                    // segment entry vanished under a deregistration). Finish
                    // the batch so remaining NACK cleanups still run.
                    refused_unmounted = true;        // 同样只记标志，继续处理剩余条目
                    continue;
                }
                LOG(ERROR) << "Failed to add replica: error=" << res.error()
                           << ", client_id=" << client_id
                           << ", tenant_id=" << object_id.tenant_id.value()
                           << ", key=" << object_id.user_key;
                return tl::make_unexpected(res.error());  // 其余错误（如 INVALID_PARAMS）视为不可恢复，整批失败
            }
            added_new_local_disk_replica = res.value();  // bool 返回值：true=新建副本，false=只是刷新了已有副本
        }
        if (metadata.data_size > 0 && added_new_local_disk_replica) {  // 只有"新增"才计费；更新路径重复计费会把 used_bytes 越加越大
            local_ssd_manager_.AdjustUsedBytes(client_id, metadata.data_size);  // 累加该 client 的 SSD 已用字节（relaxed 原子加）
        }
    }                                                // 循环体结束：每条 key 走完"清理 / 新增 / 更新"三选一

    if (refused_unmounted) {                         // 只要有一条因 segment 消失被拒，整批返回该错误
        LOG(WARNING) << "client_id=" << client_id
                     << ", action=notify_offload_success_refused"
                     << ", error=no_local_disk_segment";
        return tl::make_unexpected(ErrorCode::SEGMENT_NOT_FOUND);  // 客户端据此停止 rescan，避免对着已卸载的 segment 反复重登记
    }
    return {};                                       // 成功：空的 expected<void>
}
```

**这一段在做什么**

- 这是 master 侧 offload 的收尾入口：客户端在 `BatchOffload` 的 completion 回调里，把"这批 key 已经写进本地 SSD"的事实回填给 master（`file_storage.cpp:342-367`）。
- 对每条 key 先做三分支判定：`data_size < 0` 的 NACK 只清理任务；对象已存在且有 `offloading_task` 时做"新增或更新"；其余情况回退到通用登记入口 `AddReplicaForRetainedClient`。
- 成功路径的核心动作是 `dec_refcnt()`（松开第 5 步对源 MEMORY 副本的钉住）、清掉 `offloading_task`、把 `ReplicaStatus::COMPLETE` 的 LOCAL_DISK 副本挂到对象上。
- "已有 LOCAL_DISK 则只更新"体现在 `HasReplica(fn_is_local_disk_replica)` 的第二分支：用 `VisitReplicas` 原地改写 `transport_endpoint` 与 `object_size`，不新增副本、不加成功计数。
- 记账被严格限定在"新增"：`AdjustUsedBytes` 由 `added_new_local_disk_replica && data_size > 0` 双重保护，避免更新路径重复累加。
- 整个函数是"尽力而为 + 批级错误":单条失败多数只 `continue`，只有 `INVALID_PARAMS`、tenant 非法、`AddReplica` 的未知错误才中止整批。

**重点语法 / 惯用法**

- `tl::expected<void, ErrorCode>` + `tl::make_unexpected(...)`：用返回值表达错误，函数体里没有任何 `throw`；`return {};` 是成功返回的惯用写法。
- `record ? record->TryAcquireRetainingGuard() : std::nullopt`：三目两个分支都推导成 `std::optional<RetainingGuard>`，RAII guard 在函数返回前一直持有 `transition_mutex_`，把 client 状态机钉在非 OFFLINE。
- 双层锁有固定顺序：先 `master_state_mutex_`（`std::shared_lock`），再 `MetadataAccessorRW` 内部的 shard 独占锁；`UnmountLocalDiskSegment` 走 `master_state_mutex_` 独占，因此二者要么整体在前、要么整体在后。
- 显式写类型 `std::shared_lock<std::shared_mutex> shared_lock(master_state_mutex_);` 并用 `{}` 块限定作用域，是这份代码控制持锁时长的常见手法。
- `VisitReplicas(pred, visit)` 的 `visit` 参数是 `Replica&`（非 const 重载），因此可以原地改写副本的 descriptor——这正是"只更新"分支的实现基础。
- `Replica` 的拷贝构造被 `= delete`，所以处处是 `std::move(replica)` 与 `emplace_back`，不存在隐式拷贝。
- `data_size < 0` 是"带外哨兵"：同一个 `int64_t` 字段既表示长度又表示失败，省掉一次协议字段变更；客户端侧对应 `StorageObjectMetadata{-1, 0, 0, -1, ""}`。

**与 6.7.3 的差异**

- 时序图第 14 步画成"一次 `NotifyOffloadSuccess`"，实际是**按 bucket 多次调用**，每次 `tasks` 与 `metadatas` 一一对应；失败批次还会单独再发一次全 `-1` 的 NACK 调用（`file_storage.cpp:558-571`）。
- 第 15 步写的是"`dec_refcnt`；无 LOCAL_DISK 才登记，已有则更新"，但代码里 `dec_refcnt` **只在 `offloading_task` 存在时**执行；对象存在却没有任务标记（典型是 master 重启后的 rescan）会走 `AddReplicaForRetainedClient` 回退，甚至可能新建 metadata。
- 图里没有画失败路径：NACK 分支在 `accessor.Exists()` 为假时什么都不做（对象已随淘汰消失），不是"一定 dec_refcnt"。
- "登记"隐含"新增"，而 `AdjustUsedBytes` 只在新增且 `data_size > 0` 时执行；更新路径零计费，零长对象也不计费。
- 图里看不到 `segment_mounted` 准入：它和写副本同处一个 `master_state_mutex_` 共享锁区间，用来堵住与 `UnmountLocalDiskSegment` 的竞态；被拒后整批返回 `SEGMENT_NOT_FOUND`，客户端据此停止重登记。
- 一个对象只有一个 LOCAL_DISK 槽位：更新分支只匹配 `client_id` 相同的副本，若已有 LOCAL_DISK 属于别的 client，本次既不新增也不改写，但仍会把 `handled_existing_object` 置真。
- 第 14 步回填的 `transport_endpoint` 由客户端在回调里统一改成自己的 `local_rpc_addr_`，时序图要到第 22 步 `transfer_engine_addr` 才用得上这个地址。

## 4. 读路径：查询 → 代读 → TE 拉取 → 释放（第 16–24 步）

### 4.1 BatchGetReplicaList 与 SelectBestReplica（第 16–17 步）

`mooncake-store/include/replica.h:294-299`

```cpp
struct LocalDiskDescriptor {
    UUID client_id;                 // LOCAL_DISK 副本的归属者：offload 数据存在这个 client 的本地 SSD 上
    uint64_t object_size = 0;       // 对象字节数；读路径用它校验长度、也用于落盘后的大小刷新
    std::string transport_endpoint; // owner 暴露的 offload RPC 地址（不是 TE 地址），读路径据此发起 batch_get_offload_object
    YLT_REFL(LocalDiskDescriptor, client_id, object_size, transport_endpoint);  // 注册反射字段：序列化顺序即声明顺序
};
```

**这一段在做什么**

- 定义 LOCAL_DISK 副本在线路上的全部信息——只有三个字段，没有可写 buffer 地址，也没有文件路径。
- `client_id` 让读端知道"该向谁要数据"，同时也是 master 侧更新副本时匹配的键（3.7 的更新 lambda 就按它过滤）。
- `object_size` 让读端在真正拉取前就能知道期望长度，用于分配 buffer 和校验。
- `transport_endpoint` 是 owner 的 offload RPC 地址，不是 Transfer Engine 地址；TE 地址要等第 22 步 `BatchGetOffloadObjectResponse::transfer_engine_addr`。
- `YLT_REFL` 把三个字段注册给序列化框架，字段顺序即线格式顺序。

**重点语法 / 惯用法**

- 纯聚合体（aggregate）加默认成员初始化 `= 0`，没有构造函数，方便聚合初始化和 `std::get_if` 取用。
- `UUID` 是强类型别名，不是裸 `std::string`，避免和 key、segment name 混用。
- `YLT_REFL(类型, 字段...)` 是 struct_pack/ylt 的反射宏：一次声明同时供 RPC 编解码与 JSON 输出使用，字段增删必须两端同步。
- 三个字段都参与序列化，没有 `compatible<>` 包装，说明这是较新的字段集合，老版本 master 不会返回该变体。

**与 6.7.3 的差异**

- 时序图第 17 步只写"全部可读副本 + `lease_ttl_ms`"，看不出 LOCAL_DISK 副本长什么样；实际它就是这三个字段，读端拿不到任何直接可读的内存地址。
- 图里 `transport_endpoint` 一词在第 12 步（写）与第 18 步（读）都出现，含义一致：owner 的 offload **RPC** 地址；真正的数据面地址来自第 22 步的 `transfer_engine_addr`。
- 图里没有体现"LOCAL_DISK 没有文件路径"这一点，因此会让人误以为读端可以直接打开文件；实际必须经 owner 代读。

`mooncake-store/src/master_service.cpp:4695-4719`

```cpp
bool MasterService::TryGetReadableReplicaDescriptor(
    const Replica& replica, Replica::Descriptor& descriptor) const {
    return IsReplicaReadable(replica) &&                  // 先判"可读"：COMPLETE + 存储仍可用 + 不在等 remount
           replica.getDescriptorIfAvailable(descriptor);  // 再取描述符；内部会二次检查 availability，防止取的过程中存储消失
}

std::vector<Replica::Descriptor> MasterService::GetReadableReplicaDescriptors(
    const ObjectMetadata& metadata) const {
    std::vector<Replica::Descriptor> descriptors;
    descriptors.reserve(metadata.CountReplicas());        // 预分配上限=全部副本数，避免 push_back 反复扩容
    metadata.VisitReplicas(
        [](const Replica&) { return true; },              // pred 全选：类型过滤不在这里做，交给客户端的 SelectBestReplica
        [this, &descriptors](const Replica& replica) {    // visit 只收集"此刻确实可读"的副本
            Replica::Descriptor descriptor;
            if (TryGetReadableReplicaDescriptor(replica, descriptor)) {
                descriptors.push_back(std::move(descriptor));  // 拷贝出快照：返回值不持有任何锁，也不引用 metadata
            }
        });
    return descriptors;                                   // 空 vector 表示"一个可读副本都没有"
}

bool MasterService::IsReplicaReadable(const Replica& replica) const {
    return replica.is_completed() && replica.is_available() &&
           !IsAwaitingRemount(replica);                   // 三个条件必须同时满足，缺一不可
}
```

**这一段在做什么**

- 把对象元数据里的副本集合"翻译"成一份可读描述符快照，过滤掉所有此刻不能服务的副本。
- `is_completed()` 过滤掉 PROCESSING（写了一半）和 REMOVED（已摘除）的副本。
- `is_available()` 对 LOCAL_DISK 而言是"owner 的 liveness record 存在且 `IsServing()`"，即 owner 进程还活着。
- `IsAwaitingRemount` 在 standby 恢复后、owner 重新 mount 之前的窗口里，按 `transport_endpoint` 排除尚未重新挂载的副本。
- 返回值是纯值语义的 `vector<Descriptor>`，调用方拿到的是一份与 master 内部状态脱钩的快照。

**重点语法 / 惯用法**

- `VisitReplicas` 的 const 重载接受 `std::function<void(const Replica&)>`，因此这里可以传 lambda 并 `push_back(std::move(...))`。
- `descriptor` 在循环内构造、按值移动进 vector，避免了对外部对象的引用悬垂。
- `IsReplicaReadable` 是 `const` 成员函数，只读元数据，不触锁——因为调用方（`BatchGetReplicaList`）已持有 shard 读锁。
- `getDescriptorIfAvailable` 内部做"取前检查 + 取后复检"，是典型的 TOCTOU 防护写法。
- 空的 `std::vector` 被当作"没有可读副本"的哨兵，调用方用 `replica_list.empty()` 判断，而不是额外的 bool。

**与 6.7.3 的差异**

- 时序图第 17 步的"全部可读副本"其实是**过滤后**的集合：PROCESSING、owner 离线的 LOCAL_DISK、等待 remount 的端点都会被剔除。
- 图里没有体现 `IsAwaitingRemount` 这一层：standby 切换后短时间内同一个 key 可能返回空列表或 `REPLICA_IS_NOT_READY`，而不是"副本还在"。
- 图里 LOCAL_DISK 与 MEMORY 副本并列返回，但二者可读性判据完全不同：MEMORY 看 buffer/segment，LOCAL_DISK 看 owner 心跳。

`mooncake-store/src/master_service.cpp:4904-5061`

```cpp
std::vector<tl::expected<GetReplicaListResponse, ErrorCode>>
MasterService::BatchGetReplicaList(const std::vector<std::string>& keys,
                                   const TenantId& tenant_id) {
    using GetResult = tl::expected<GetReplicaListResponse, ErrorCode>;  // 每个 key 独立成败：批量接口不做整体失败

    assert(tenant_id.IsValid());                     // Debug 断言：tenant 已在 RPC 层校验过

    std::vector<GetResult> results(
        keys.size(), tl::make_unexpected(ErrorCode::OBJECT_NOT_FOUND));  // 预填"未找到"：任何未被扫到的 key 都保持这个默认值
    if (keys.empty()) {
        return results;                              // 空请求直接返回空 vector
    }

    const TenantId& normalized_tenant = ResolveRequestTenantId(tenant_id);  // 多租户关闭时返回 Default() 静态单例的 const 引用（不拷贝）
    constexpr size_t kInvalidKeyIndex = std::numeric_limits<size_t>::max();  // 用最大值充当"链表尾"哨兵
    std::array<size_t, kNumShards> key_list_heads;   // 每个 shard 一条"索引链表"的头
    key_list_heads.fill(kInvalidKeyIndex);           // 初始化为空链
    std::vector<size_t> next_key_indexes(keys.size(), kInvalidKeyIndex);  // 索引链表：每个 key 指向同 shard 的下一个 key
    for (size_t i = keys.size(); i > 0; --i) {       // 倒序插入链表头，使正序遍历时保持调用方给的原始顺序
        const size_t original_idx = i - 1;           // 还原原始下标（results 的下标必须与入参 keys 对齐）
        const size_t shard_idx =
            getShardIndex(normalized_tenant, keys[original_idx]);  // tenant+key 共同决定 shard，保证同对象总在同一分片
        next_key_indexes[original_idx] = key_list_heads[shard_idx];  // 新节点接在当前链表头之后
        key_list_heads[shard_idx] = original_idx;    // 新节点成为新的链表头
    }

    const size_t start_shard = randomIndex(kNumShards);  // 随机起始分片：打散多客户端同时扫描时的锁竞争热点
    for (size_t scanned = 0; scanned < kNumShards; ++scanned) {  // 保证每个 shard 恰好访问一次
        const size_t shard_idx =
            (start_shard + kNumShards - scanned) % kNumShards;  // 环形遍历：从随机点出发绕一圈
        if (key_list_heads[shard_idx] == kInvalidKeyIndex) {
            continue;                                // 该 shard 没有本次请求的 key，跳过（不取锁）
        }

        std::vector<ObjectIdentity> promotion_candidates;  // 需要加入"提升回 MEMORY"队列的 key（锁外再入队）
        std::vector<ObjectIdentity> dynamic_replication_candidates;  // 需要动态补副本的 key（同理）
        std::unordered_set<std::string> dynamic_replication_seen;  // 批内去重：同 key 只记一次访问
        std::shared_lock<std::shared_mutex> shared_lock(master_state_mutex_);  // 每个 shard 一轮：读锁 + shard 锁的嵌套顺序固定
        {
            MetadataShardAccessorRO shard(this, shard_idx);  // shard 读锁；与 master_state_mutex_ 共享锁配合，允许并发读
            const auto tenant_it = shard->tenants.find(normalized_tenant);  // 该租户在本 shard 的元数据表
            for (size_t original_idx = key_list_heads[shard_idx];
                 original_idx != kInvalidKeyIndex;
                 original_idx = next_key_indexes[original_idx]) {  // 沿索引链表遍历本 shard 的所有 key
                const std::string& key = keys[original_idx];  // 原始 key（未加租户前缀）
                MasterMetricManager::instance().inc_total_get_nums();  // 请求计数：无论命中与否都算一次 get

                if (tenant_it == shard->tenants.end()) {  // 该租户在此 shard 完全没有数据
                    VLOG(1) << "key=" << key << ", info=object_not_found";
                    results[original_idx] =
                        tl::make_unexpected(ErrorCode::OBJECT_NOT_FOUND);
                    continue;                        // 保持默认错误值
                }

                const auto& tenant_state = tenant_it->second;  // 该租户的元数据状态
                const auto metadata_it = tenant_state.metadata.find(key);  // 按 user_key 找对象
                if (metadata_it == tenant_state.metadata.end() ||
                    !metadata_it->second.metadata.IsValid()) {  // 不存在，或元数据已标记失效（删除/重建中）
                    VLOG(1) << "key=" << key << ", info=object_not_found";
                    results[original_idx] =
                        tl::make_unexpected(ErrorCode::OBJECT_NOT_FOUND);
                    continue;
                }

                const auto& metadata = metadata_it->second.metadata;  // 对象元数据（副本集合 + 租约）
                auto replica_list = GetReadableReplicaDescriptors(metadata);  // 可读副本快照：MEMORY / NoF / LOCAL_DISK / DFS / DISK 混排

                if (dfs_allocator_) {                // 仅当配置了 DFS 分配器时才做访问统计
                    for (const auto& replica : replica_list) {
                        if (replica.is_dfs_replica()) {
                            const auto& desc = replica.get_dfs_descriptor();
                            dfs_allocator_->UpdateAccess(key, desc);  // 记录 DFS 访问热度，供后续分配决策
                        }
                    }
                }

                if (replica_list.empty()) {          // 一个可读副本都没有：区分"已删除"与"还没就绪"
                    if (metadata.AllReplicas([](const Replica& replica) {
                            return replica.status() == ReplicaStatus::REMOVED;  // 全部副本都是 REMOVED = 对象已删除
                        })) {
                        results[original_idx] =
                            tl::make_unexpected(ErrorCode::OBJECT_NOT_FOUND);
                        continue;
                    }
                    LOG(WARNING)
                        << "key=" << key << ", error=replica_not_ready";  // 还有 PROCESSING 副本 = 写未完成，让调用方稍后重试
                    results[original_idx] =
                        tl::make_unexpected(ErrorCode::REPLICA_IS_NOT_READY);
                    continue;
                }

                if (replica_list[0].is_memory_replica()) {  // 命中统计只看 list[0]，是近似口径（选副本由客户端全表扫描决定）
                    MasterMetricManager::instance().inc_mem_cache_hit_nums();
                    MasterMetricManager::instance().inc_mem_cache_hit_bytes(
                        static_cast<int64_t>(metadata.TotalLength()));  // byte 维度：按对象总长度计
                } else if (replica_list[0].is_local_disk_replica() ||
                           replica_list[0].is_disk_replica()) {  // LOCAL_DISK 与 legacy DISK 都算 file cache 命中
                    MasterMetricManager::instance().inc_file_cache_hit_nums();
                    MasterMetricManager::instance().inc_file_cache_hit_bytes(
                        static_cast<int64_t>(metadata.TotalLength()));
                }
                MasterMetricManager::instance().inc_valid_get_nums();  // 有效 get（找到了对象）
                metadata.GrantReadLease(
                    std::chrono::milliseconds(default_kv_lease_ttl_));  // 关键：把对象级读租约续到 now + ttl（单调取最大）

                if (promotion_on_hit_) {             // 命中 LOCAL_DISK 时触发"回迁 MEMORY"的候选收集
                    const bool any_memory =
                        std::any_of(replica_list.begin(), replica_list.end(),
                                    [](const auto& descriptor) {
                                        return descriptor.is_memory_replica();
                                    });          // 列表里是否还有 MEMORY 副本
                    const bool any_local_disk = std::any_of(
                        replica_list.begin(), replica_list.end(),
                        [](const auto& descriptor) {
                            return descriptor.is_local_disk_replica();
                        });                      // 列表里是否有 LOCAL_DISK 副本
                    if (!any_memory && any_local_disk) {  // 只有"只剩 LOCAL_DISK"的冷对象才值得回迁热缓存
                        promotion_candidates.push_back(
                            MakeObjectIdentity(key, normalized_tenant));  // 记下身份，锁外再入队
                    }
                }
                if (dynamic_replication_.Enabled()) {  // 动态副本：MEMORY 副本数不足时补一个
                    const size_t memory_replicas =
                        std::count_if(replica_list.begin(), replica_list.end(),
                                      [](const auto& descriptor) {
                                          return descriptor.is_memory_replica();
                                      });     // 只数 MEMORY，LOCAL_DISK 不计入
                    if (dynamic_replication_.WantsAnotherMemoryReplica(
                            memory_replicas)) {   // 控制器判断是否低于目标副本数
                        auto object_id =
                            MakeObjectIdentity(key, normalized_tenant);
                        if (dynamic_replication_seen
                                .insert(object_id.tenant_id.MakeScopedKey(
                                    object_id.user_key))
                                .second) {       // set::insert 的 .second 表示"首次插入"，即批内去重
                            dynamic_replication_candidates.push_back(
                                std::move(object_id));
                        }
                    }
                }

                results[original_idx] = GetReplicaListResponse(
                    std::move(replica_list), default_kv_lease_ttl_,
                    metadata.GetPart(0).checksum());  // 回包：副本快照 + 租约毫秒数 + 可选校验和
            }
        }

        for (const auto& object_id : promotion_candidates) {  // 已释放 shard 锁 / master_state_mutex_ 才入队
            TryPushPromotionQueue(object_id);    // 尽力而为：失败不影响本次读
        }
        for (const auto& object_id : dynamic_replication_candidates) {
            dynamic_replication_.RecordAccess(object_id.tenant_id,
                                              object_id.user_key);  // 记录访问，供后续决策
        }
    }

    return results;                              // 长度与入参 keys 严格一致，顺序也一致
}
```

**这一段在做什么**

- 批量查询的 master 实现：把一批 key 按 shard 分组，逐 shard 加锁读取，为每个 key 独立返回一份 `GetReplicaListResponse` 或错误码。
- 分组用"索引链表"而不是 `vector<vector<key>>`：`key_list_heads` + `next_key_indexes` 把同 shard 的 key 串起来，避免为每个 shard 分配容器。
- 对每个 key：取可读副本快照 → 全空时区分 `OBJECT_NOT_FOUND` 与 `REPLICA_IS_NOT_READY` → 记命中指标 → **续对象级读租约** → 收集 promotion / 动态副本候选 → 组装回包。
- 命中指标按 `replica_list[0]` 归类为 mem cache 或 file cache；这只是一个近似口径，真正的选择在客户端。
- 候选入队被刻意挪到锁外（shard 块结束之后），避免在持锁期间做可能失败的队列操作。
- 返回 vector 的下标与入参 `keys` 一一对应，调用方（`Client::BatchQuery`）按位置合并结果。

**重点语法 / 惯用法**

- `std::vector<GetResult> results(keys.size(), default_value)`：批量接口的"先填默认错误、再逐条覆盖"惯用法，保证长度恒定。
- `constexpr size_t kInvalidKeyIndex = std::numeric_limits<size_t>::max();` 作为链表尾哨兵，省掉额外的 `std::optional`。
- 倒序插入链表头 + 正序遍历：既完成分桶，又保持原始顺序，是 O(n) 的无额外分配写法。
- `std::shared_lock<std::shared_mutex> shared_lock(master_state_mutex_);` 与 `MetadataShardAccessorRO` 的嵌套顺序固定为"全局锁 → 分片锁"。
- `randomIndex(kNumShards)` 随机起点：纯粹为了错开多客户端的锁竞争，不影响正确性。
- lambda 捕获 `[this, &descriptors]` / `[client_id]` / `[&replica]` 的差异体现所有权意图：只读引用 vs 按值捕获小对象。
- `MakeScopedKey` + `unordered_set::insert(...).second` 是"批内去重"的经典写法。

**与 6.7.3 的差异**

- 时序图第 16 步是"R → M: BatchGetReplicaList"，第 17 步是"全部可读副本 + `lease_ttl_ms`"；实际返回的是**每个 key 一个 response 对象**，`lease_ttl_ms` 是对象级的，不是整批一个。
- `lease_ttl_ms` 取的是 master 配置的 `default_kv_lease_ttl_`，与 offload buffer 的 `gc_ttl_ms`（第 22 步，来自 `client_buffer_gc_ttl_ms`）是两个完全不同的租约，图里容易混为一谈。
- 读租约只保护 MEMORY 副本不被淘汰：它对 LOCAL_DISK 无效，LOCAL_DISK 副本的生命周期由 owner 心跳、`used_bytes` 与卸载流程决定。
- 图里第 17 步看起来"总是有副本"，实际还有 `REPLICA_IS_NOT_READY` 与 `OBJECT_NOT_FOUND` 两种失败；前者是写未完成，后者是对象已删或不存在。
- 图里没画 promotion：当 `promotion_on_hit_` 开启且对象只剩 LOCAL_DISK 时，这次读会顺带把对象推进回迁队列——正是本时序图描述的"冷读"场景。
- 图里"批量"是逻辑上的；实现上按 shard 串行加锁，且 key 越多持锁轮次越多，没有跨 shard 的并行。
- 命中指标按 `replica_list[0]` 归类，而客户端实际用 `SelectBestReplica` 全表扫描；两者口径不同，指标只能当趋势看。

`mooncake-store/include/object_metadata.h:312-318`

```cpp
// Grant a read lease at now() + ttl. Grouped objects extend the shared
// group TTL; a zero ttl is a no-op on a live lease (PutEnd cannot
// expire a live group).
    void GrantReadLease(std::chrono::milliseconds ttl) const {  // const 方法却修改状态：内部用 shared_ptr + 自旋锁保护
        SpinLocker locker(&lock);                // 对象级自旋锁：临界区只有一次原子写，比 mutex 更轻
        lease_->GrantReadLease(ttl);             // 委托给 Lease；同组对象共享同一个 Lease，一次续租整组生效
    }
```

**这一段在做什么**

- 把对象（或对象组）的读租约续到 `now + ttl`，告诉 master"刚刚有人在读，暂时别淘汰"。
- `lease_` 是 `shared_ptr<Lease>`：同一组的多个 key 指向同一个 Lease，任意一次读都会延长整组的 TTL。
- 方法声明为 `const`，因为调用方持有的是 `const ObjectMetadata&`（在 `MetadataShardAccessorRO` 下），逻辑上"续租"不算修改对象内容。
- 自旋锁只保护指针与 deadline 的读取，真正的 deadline 更新由 `Lease` 内部的原子 CAS 完成。

**重点语法 / 惯用法**

- `mutable` 风格的 const 语义：用 `shared_ptr` 指向可变状态，使 `const` 方法也能修改被指向的数据。
- `SpinLocker` 是仓库自定义的 RAII 包装，构造即加锁、析构即解锁；临界区必须足够短。
- 委托式设计：`ObjectMetadata` 只管"找对 Lease"，时间计算与并发控制在 `Lease` 里做，职责分离。
- `std::chrono::milliseconds` 显式类型，避免裸整数带来的单位歧义。

**与 6.7.3 的差异**

- 时序图把第 17 步的 `lease_ttl_ms` 画成回包里的一个数字，看不出它是"在这次查询里顺手续租"的副作用；续租发生在返回之前。
- 图里"读租约"三个字容易被读成保护 offload 数据；实际上它只与 MEMORY 副本的淘汰判定相关。
- 租约是**单调取最大**的（见下一块），所以频繁读只会不断推后 deadline，不会因为并发读而缩短。

`mooncake-store/include/lease.h:14-30`

```cpp
class Lease {
   public:
    // Extend the deadline (monotonic max) to now + ttl. The read path
    // advertises the full ttl to clients, so the server must honor now + ttl
    // rather than keeping an earlier existing deadline (which would let the
    // server evict before the client-held lease expires).
    void GrantReadLease(std::chrono::milliseconds ttl) {
        const auto ttl_ns =
            std::chrono::duration_cast<std::chrono::nanoseconds>(ttl).count();  // 统一换成纳秒整数，内部只存 int64
        const int64_t now = NowNs();             // 单调时钟的当前时刻（纳秒）
        const int64_t new_deadline = now + ttl_ns;  // 目标 deadline
        int64_t cur = deadline_ns_.load(std::memory_order_relaxed);  // 读当前值；relaxed 足够，因为 CAS 本身保证原子性
        while (cur < new_deadline &&
               !deadline_ns_.compare_exchange_weak(cur, new_deadline,
                                                   std::memory_order_relaxed)) {
        }                                        // CAS 循环：只在"新 deadline 更大"时写入，失败时 cur 被刷新为最新值
    }
```

**这一段在做什么**

- 用无锁 CAS 实现"单调取最大"的 deadline 更新：并发续租时最终结果是所有请求里最晚的那个截止时间。
- 为什么必须取最大：客户端拿到的是完整 TTL 并据此计算本地截止时间；如果服务端保留更早的 deadline，就会在客户端租约到期前提前淘汰。
- `compare_exchange_weak` 放在 `while` 条件里，失败时由硬件写回 `cur`，循环自然重试，无需显式分支。
- `memory_order_relaxed` 是安全的：deadline 只是一个单调递增的时间戳，不需要与其他内存操作建立顺序关系。

**重点语法 / 惯用法**

- `std::atomic<int64_t>` + CAS 循环，是这份代码里最典型的 lock-free 更新模式。
- `compare_exchange_weak` 允许伪失败，因此必须写在循环里；`strong` 版本才可用于单次尝试。
- `duration_cast<nanoseconds>(ttl).count()` 把强类型时长降为整数存储，跨平台比 `system_clock` 精度更可控。
- 空循环体 `{}` 加条件表达式完成"尝试 + 重试"，是 CAS 惯用写法。

**与 6.7.3 的差异**

- 图里第 17 步的 `lease_ttl_ms` 看起来像"剩余时间"，实际回包给的是**完整 TTL**，由客户端自己加上本地起始时间。
- 续租是"取最大"而不是"覆盖"：同一 key 并发多次查询不会互相缩短租约，图中串行箭头体现不出这一点。
- 这个租约与第 22 步的 `gc_ttl_ms` 完全独立：后者是 offload buffer 的读者侧截止时间，超时返回 `OBJECT_HAS_LEASE`。

`mooncake-store/include/rpc_types.h:77-98`

```cpp
struct GetReplicaListResponse {
    std::vector<Replica::Descriptor> replicas;  // 可读副本快照；LOCAL_DISK 只含 client_id/object_size/transport_endpoint
    uint64_t lease_ttl_ms;                      // 对象级读租约时长（毫秒），客户端自行换算成绝对截止时间
    std::optional<uint64_t> object_checksum;    // 可选对象校验和，用于读后校验
    // The tenant the master resolved the query to: the requesting tenant when
    // tenant isolation is on, "default" otherwise. Readers forward it in
    // VerifyDiskReplicaRequest so the replica owner probes and evicts under
    // the exact scope the write path used. Empty when the master predates
    // this field.
    struct_pack::compatible<std::string, 1> resolved_tenant_id;  // 向后兼容字段：老 master 不回传，新客户端能安全解析

    GetReplicaListResponse() : lease_ttl_ms(0) {}  // 默认构造：租约为 0，表示"没有租约"
    GetReplicaListResponse(
        std::vector<Replica::Descriptor>&& replicas_param,
        uint64_t lease_ttl_ms_param,
        std::optional<uint64_t> object_checksum_param = std::nullopt)
        : replicas(std::move(replicas_param)),  // 右值引用入参 + move：避免整份副本列表拷贝
          lease_ttl_ms(lease_ttl_ms_param),
          object_checksum(object_checksum_param) {}
};
YLT_REFL(GetReplicaListResponse, replicas, lease_ttl_ms, object_checksum,
         resolved_tenant_id);                   // 反射注册：字段顺序即线格式顺序，增删需考虑兼容
```

**这一段在做什么**

- 定义第 17 步的回包结构：副本列表 + 对象级租约 + 可选校验和 + 归一化后的 tenant。
- `lease_ttl_ms` 是一个时长（TTL），不是绝对时间点；客户端用它加上请求起始时间得到本地截止时间。
- `resolved_tenant_id` 会被读端在后续 `VerifyDiskReplicaRequest` 里回传给 owner，保证探测与淘汰使用与写入时相同的租户作用域。
- 构造函数只接受右值副本列表，从签名上就杜绝了整表拷贝。

**重点语法 / 惯用法**

- `struct_pack::compatible<T, 1>` 是版本兼容字段包装：老版本对端没有该字段时反序列化不报错，`YLT_REFL` 也照常注册。
- `std::optional<uint64_t>` 表示"可能没有校验和"，比用 0 当哨兵更明确。
- 右值引用参数 + `std::move` 初始化成员，是 RPC 大对象传递的标准做法。
- 默认构造把 `lease_ttl_ms` 显式初始化为 0，避免未初始化值导致客户端立刻判定租约过期。
- `YLT_REFL` 一行注册全部字段，序列化与 JSON 输出共用同一份声明。

**与 6.7.3 的差异**

- 图里第 17 步只写"全部可读副本 + `lease_ttl_ms`"，漏掉了 `object_checksum` 与 `resolved_tenant_id` 两个字段；后者是后续 owner 侧探测/淘汰的作用域凭证。
- `lease_ttl_ms` 是**每个对象**一个（每个 key 一份 response），不是整个 batch 一个；虽然取值通常相同（都来自 `default_kv_lease_ttl_`）。
- 图里把租约画在"读路径"上，像是在保护数据；实际它只影响 MEMORY 副本何时可被淘汰。

`mooncake-store/src/client_service.cpp:1277-1292`

```cpp
    std::vector<tl::expected<QueryResult, ErrorCode>> results;
    results.reserve(response.size());            // 预留容量，结果数与回包一致
    for (size_t i = 0; i < response.size(); ++i) {
        if (response[i]) {                       // 该 key 查询成功
            results.emplace_back(
                tl::in_place, std::move(response[i].value().replicas),
                start_time +
                    std::chrono::milliseconds(response[i].value().lease_ttl_ms),
                response[i].value().object_checksum,
                response[i].value().resolved_tenant_id);  // 把 TTL 换算成绝对截止时间：start_time + ttl
        } else {
            results.emplace_back(tl::unexpected(response[i].error()));  // 失败原样透传错误码
        }
    }
    return results;
}
```

**这一段在做什么**

- 客户端把 master 回包的"相对 TTL"换算成本地 `steady_clock` 的绝对截止时间，存进 `QueryResult`。
- `start_time` 是**发请求之前**取的时间点，因此本地截止时间比"收到回包时刻 + TTL"更早一个 RTT——这是保守取值，宁可早到期也不晚。
- 每个 key 独立处理成功/失败，与 master 侧"每 key 一个 expected"的约定一一对应。
- 得到的 `QueryResult` 会随副本列表一起被后续 `Get` 流程用于判断"这份副本信息还能不能用"。

**重点语法 / 惯用法**

- `tl::expected` 的 `tl::in_place` 构造：直接在原位构造成功值，避免先造临时对象再移动。
- `start_time + std::chrono::milliseconds(ttl)`：强类型时间点 + 强类型时长，编译期就能发现单位错误。
- `emplace_back(tl::unexpected(...))` 构造失败分支，与 `expected` 的工厂函数配套。
- `results.reserve(response.size())` 在已知数量时消除扩容。

**与 6.7.3 的差异**

- 图里第 17 步到第 18 步之间没有画出"TTL → 绝对时间"的换算；这个换算发生在客户端，且以**请求发起时刻**为基准。
- 图里看不出这个截止时间只对副本列表的"新鲜度"负责，与第 22 步的 `gc_ttl_ms` 是两套独立计时。
- 若某个 key 查询失败，客户端仍然返回等长的结果数组，失败位置保留错误码——图里的单条箭头体现不出批量的逐项成败。

`mooncake-store/include/replica_selection.h:122-175`

```cpp
inline const Replica::Descriptor *SelectBestReplica(
    const std::vector<Replica::Descriptor> &replicas,
    const std::unordered_set<std::string> &local_endpoints) {  // local_endpoints：本进程已知的"本地"传输端点集合
    const Replica::Descriptor *local_memory = nullptr;  // 本机 MEMORY：最高优先级
    const Replica::Descriptor *local_nof = nullptr;     // 本机 NoF_SSD：次高
    const Replica::Descriptor *first_memory = nullptr;  // 远端 MEMORY：再次
    const Replica::Descriptor *first_nof = nullptr;     // 远端 NoF_SSD：再次
    for (const auto &r : replicas) {                    // 第一轮：只看内存类副本（MEMORY / NoF）
        if (r.status != ReplicaStatus::COMPLETE) continue;  // PROCESSING/REMOVED 一律跳过
        if (r.is_memory_replica()) {
            if (local_endpoints.count(
                    r.get_memory_descriptor()
                        .buffer_descriptor.transport_endpoint_)) {  // 端点在本机集合里 = 本地副本
                if (!local_memory) local_memory = &r;   // 只记第一个，后续同级的忽略
            } else {
                if (!first_memory) first_memory = &r;   // 远端 MEMORY 记第一个（后面可能被评分替换）
            }
        } else if (r.is_nof_replica()) {
            if (local_endpoints.count(
                    r.get_nof_descriptor()
                        .buffer_descriptor.transport_endpoint_)) {  // NoF 同样用 transport_endpoint_ 判本地
                if (!local_nof) local_nof = &r;
            } else {
                if (!first_nof) first_nof = &r;
            }
        }
    }
    // Local MEMORY always beats local NOF regardless of list order.
    if (local_memory) return local_memory;              // 本地 MEMORY 直接胜出，不再看后面的类型
    if (local_nof) return local_nof;                    // 没有本地 MEMORY 才轮到本地 NoF
    // No local replica. Among remote MEMORY replicas, optionally pick the
    // best-scoring one instead of the first encountered (issue #2516).
    if (first_memory && RemoteReplicaScoringEnabled()) {  // 评分开关默认关闭，关闭时行为与历史版本逐字节一致
        if (const auto *scored =
                PickBestRemoteMemory(replicas, local_endpoints)) {  // C++17 if-init：把返回的裸指针限定在 if 作用域
            return scored;                              // 选出评分最低（最优）的远端 MEMORY
        }
    }
    if (first_memory) return first_memory;              // 未开启评分：退回"第一个远端 MEMORY"
    if (first_nof) return first_nof;                    // 再退到远端 NoF

    const Replica::Descriptor *best = nullptr;          // 第二轮：磁盘类副本的优先级 fight
    for (const auto &r : replicas) {
        if (r.status != ReplicaStatus::COMPLETE) continue;  // 同样只考虑 COMPLETE
        if (r.is_local_disk_replica()) {
            best = &r;  // LOCAL_DISK always overrides DFS and DISK  // 无条件覆盖：LOCAL_DISK 优先于 DFS 与 legacy DISK
        } else if (r.is_dfs_replica()) {
            if (!best || !best->is_local_disk_replica()) best = &r;  // 只有当前不是 LOCAL_DISK 时才让 DFS 顶上
        } else if (r.is_disk_replica() && !best) {
            best = &r;                                  // legacy DISK 优先级最低：只在没有任何候选时才取
        }
    }
    return best;                                        // 全都不满足时返回 nullptr（调用方需判空）
}
```

**这一段在做什么**

- 给定 master 返回的副本列表与本进程的本地端点集合，选出"这一读该从哪份副本取"。
- 优先级顺序严格是：本地 MEMORY > 本地 NoF_SSD > （可选的评分最优）远端 MEMORY > 远端 NoF_SSD > LOCAL_DISK > DFS > legacy DISK。
- 第一轮扫描内存类副本，用四个指针分别记住"本地/远端 × MEMORY/NoF"的第一个候选，因此不依赖 master 的返回顺序。
- 只有在完全没有内存类副本时，第二轮才会考虑磁盘类副本；此时 LOCAL_DISK 会无条件压过 DFS 与 DISK。
- 返回的是指向入参 vector 内部的裸指针，调用方必须保证 `replicas` 在使用期间存活。

**重点语法 / 惯用法**

- `const Replica::Descriptor *` 裸指针 + 返回 `nullptr` 表示"没有可用副本"：零拷贝，但把生命周期责任交给调用方。
- `local_endpoints.count(endpoint)`：`unordered_set` 的判存在惯用法，O(1) 且不需要 `find` 的结果。
- `if (const auto *scored = ...; ...)` 是 C++17 的 if-init 语句，把变量作用域限制在 if 内。
- 两级指针的"先记第一个、再按优先级裁决"模式，避免了排序整个列表的 O(n log n) 开销。
- `RemoteReplicaScoringEnabled()` 内部用函数内静态变量缓存环境变量，只读一次；注入式 scorer 则实时检查，方便测试。
- 函数是 `inline` 的自由函数（命名空间作用域），没有状态，便于各调用点内联。

**与 6.7.3 的差异**

- 时序图第 17–18 步之间看不出"选择"这一步的存在：图里直接从"全部可读副本"跳到"向 FS 发起代读"，实际客户端必须先做本地/远端与类型排序。
- 图里只有一条读路径，实际 LOCAL_DISK 是**最后才被考虑**的磁盘类选项：只要还有任何 MEMORY/NoF 副本（哪怕远端），都不会走 offload 代读。
- LOCAL_DISK 在磁盘类内部优先级最高（压过 DFS 与 legacy DISK），但它排在所有内存类副本之后——这正是 6.7.3 描述的"冷读"才会触发的路径。
- 图里没有体现评分开关：`MC_STORE_REPLICA_SCORING=1` 或注入 scorer 时，远端 MEMORY 的选择会从"第一个"变成"评分最优"，默认关闭时行为与旧版一致。
- 图里若画了"多个副本"，容易以为顺序由 master 决定；实际 master 的返回顺序被完全忽略（除评分平局时保留返回序）。

### 4.2 restore_offload_objects 与 offload RPC（第 18 步）

第 18 步是读者把"我要读这些 key"告诉 owner 的那一跳。它看起来只是一次 RPC，但函数里实际有两条路径：跨进程走 `client_requester_->batch_get_offload_object`，同进程且目标 buffer 在 accelerator 内存上时走 `file_storage_->BatchGetLocal` 的 local_batch 捷径。两条路径返回同一个 `BatchGetOffloadObjectResponse` 结构，后面的读取代码因此不需要分叉。

#### 4.2.1 pinned 捷径的判定

mooncake-store/src/real_client.cpp:8873-8891

```cpp
bool RealClient::can_use_pinned_restore_arena(
    const std::string &target_rpc_service_addr,  // 这次读要发给谁：LOCAL_DISK descriptor 里的 transport_endpoint
    const std::unordered_map<std::string, std::vector<Slice>> &objects) const {  // 读者自己的目标 buffer（key -> 切片列表）
    if (!file_storage_ || target_rpc_service_addr != local_rpc_addr ||  // 没有本地存储或目标不是自己 → 只能走 RPC
        !file_storage_->HasPinnedRestoreArena()) {  // 没配 pinned arena → 走 RPC；HasPinnedRestoreArena() 只判指针非空
        return false;  // 早退：任一前提不成立就放弃捷径
    }
    auto accelerators = device::GetAcceleratorRegistry().RuntimeAccelerators();  // 拿到当前所有 accelerator 的拷贝（按值，避免持有注册表内部引用）
    bool has_data = false;  // 至少要有一块非空 slice，否则空读没有走捷径的意义
    for (const auto &object : objects) {  // 逐个对象
        for (const auto &slice : object.second) {  // 逐个目标切片
            if (slice.size && !accelerators.FindDeviceForPointer(slice.ptr)) {  // 有数据但落了普通 CPU 内存 → 不能用 accelerator pinned arena
                return false;  // 只要有一块不在设备上就整体退回 RPC，不做部分捷径
            }
            has_data |= slice.size != 0;  // 注意是 |= 不是 =：任意一块非空即置位，且不会把已置位的 true 抹掉
        }
    }
    return has_data;  // 空读（所有 slice.size==0）直接拒绝捷径
}
```

**这一段在做什么**

- 判断"这次 offload 读能不能在自己进程内完成并且复用 accelerator pinned 内存"。
- 三重前提：`file_storage_` 存在、目标 RPC 地址等于本地 `local_rpc_addr`、pinned restore arena 已配置。
- 再逐个 `Slice` 检查：只要有数据并且指针不在 accelerator 上，整批退回 RPC。
- 最后要求至少有一块非空切片，纯空读不占捷径。
- 这个函数被 `restore_offload_objects` 和单 key 读路径 `can_use_pinned_restore_arena(endpoint, objects)` 共用。

**重点语法 / 惯用法**

- 整函数 `const` 成员函数，不修改任何状态，纯判定。
- `objects` 是 `unordered_map<string, vector<Slice>>`：一个 key 可以有多块切片（GPU 多段 buffer），所以内层还要再遍历 `vector<Slice>`。
- `has_data |= slice.size != 0;` 用位或赋值累计布尔值，等价于 `has_data = has_data || ...`，但不会短路后面的循环体。
- `auto accelerators = ...RuntimeAccelerators();` 取值拷贝而不是 `auto&`，注册表在并发注册设备时不会因引用悬空出问题。
- 提前 `return false` 的守卫写法，让主逻辑保持在一层缩进内。

**与 6.7.3 的差异**

- 图里第 18 步只有一条 `R->>FS: batch_get_offload_object`，没有体现"目标就是自己"时可以完全不发 RPC。
- 图未画出 pinned arena 这个可选组件，它是部署期配置，缺省时上面这个函数第一行就返回 false。
- 图把 restore 当成纯网络动作，实际这里还做了 accelerator 指针归属检查（决定后面第 23 步是同进程 memcpy 还是 RDMA READ）。
- 图里读者只有一个"R"，实际 `local_rpc_addr` 与 `descriptor.transport_endpoint` 的相等比较才是区分"自己"和"远端 owner"的唯一依据。

#### 4.2.2 入口：参数校验、租户作用域与 local_batch 决策

mooncake-store/src/real_client.cpp:8893-8914

```cpp
tl::expected<std::unique_ptr<RealClient::OffloadRestoreLease>, ErrorCode>  // 返回 RAII 租约：析构时自动 release_offload_buffer
RealClient::restore_offload_objects(
    const std::string &target_rpc_service_addr,  // owner 的 offload RPC 服务地址；不是 Transfer Engine 地址
    const std::vector<std::string> &keys,  // 读者视角的 key，未带租户前缀
    const std::vector<int64_t> &restore_sizes,  // 每个 key 的“存储侧总长度”，不是本次要读的字节数
    const std::unordered_map<std::string, std::vector<Slice>> &objects,  // key -> 读者侧目标 buffer 切片
    bool allow_pinned) {  // 调用方是否允许走同进程 pinned 捷径（声明处默认 true）
    if (keys.size() != restore_sizes.size()) {  // 两个数组后面按下标并行遍历，长度不等会越界
        return tl::unexpected(ErrorCode::INVALID_PARAMS);  // tl::expected 的错误构造：不抛异常，直接当返回值
    }
    offload_rpc_read_count_.fetch_add(1, std::memory_order_relaxed);  // 纯统计计数；relaxed 足够，不参与同步
    const auto start_time = std::chrono::steady_clock::now();  // 单调时钟起点，后面和 gc_ttl_ms 比较判断租约是否过期
    const bool local_batch =  // 只有“允许 + 全部条件成立”才走同进程捷径
        allow_pinned &&  // 调用方开关（单 key 读路径在某些分支会关掉）
        can_use_pinned_restore_arena(target_rpc_service_addr, objects);  // 见上一小节：地址相同 + arena 存在 + 指针在设备上

    const TenantId tenant_id(client_->tenant_id());  // 从 RealClient 取本连接的租户；空串会被 TenantId 构造函数替换成默认租户
    std::vector<std::string> storage_keys;  // 加租户前缀后的 key 列表，发给 owner 和后端用的都是这一份
    storage_keys.reserve(keys.size());  // 预分配，避免循环里多次扩容导致迭代器/指针失效
    for (const auto &key : keys) {  // 按原顺序逐个变换，保证下标与 keys、restore_sizes 对齐
        storage_keys.emplace_back(tenant_id.MakeScopedKey(key));  // tenant_id + '\0' + key（分隔符是 '\0'，不是冒号）
    }
```

**这一段在做什么**

- 校验 `keys` 与 `restore_sizes` 一一对应，长度不等直接 `INVALID_PARAMS`。
- 打点统计 `offload_rpc_read_count_`，并记录单调时钟起点，供后续判断 `gc_ttl_ms` 是否超时。
- 决定 `local_batch`：调用方允许 **且** `can_use_pinned_restore_arena` 成立。
- 把用户 key 批量加租户前缀，得到后端真正使用的 `storage_keys`。
- 注意 `restore_sizes` 是"存储侧完整长度"，不是本次要 copy 的长度；局部读时由调用方传入 `read_range->restore_size`。

**重点语法 / 惯用法**

- 返回 `tl::expected<std::unique_ptr<...>, ErrorCode>`：C++20 `std::expected` 的 backport，错误码走类型系统而不是异常。
- `tl::unexpected(...)` 构造错误分支，调用方用 `if (!x)` 判断后用 `x.error()` 取码。
- `MakeScopedKey` 内部是 `value_ + kScopedKeySeparator + local_key`，其中 `kScopedKeySeparator = '\0'`（tenant_id.h:76），因此前缀不会与用户 key 里可能出现的字符冲突。
- `reserve` + `emplace_back`：避免 `push_back` 的临时对象与多次重分配。
- `std::memory_order_relaxed` 表示这里只要原子性、不要顺序保证。

**与 6.7.3 的差异**

- 图里写 `batch_get_offload_object(tenant:key[], restore_sizes[])`，实际拼接分隔符是 `'\0'` 而不是 `':'`。
- 图没有画出 `local_batch` 这个分叉，它是"读者恰好就是 owner 且 buffer 在 accelerator 上"时的整段捷径。
- 图里 `restore_sizes` 像是读取长度，实际是对象在存储侧的完整 payload 大小，局部读由 `OffloadReadRange` 另外表达。
- 图未体现读计数打点与 `steady_clock` 起点的采集，这两者是第 22 步 `gc_ttl_ms` 超时判定的前置条件。

#### 4.2.3 两条获取路径：RPC 与 BatchGetLocal

mooncake-store/src/real_client.cpp:8916-8936

```cpp
    std::optional<FileStorage::LocalBatchResult> local_owner;  // 只在 local_batch 分支里被 emplace；持有分配出的 buffer，生命周期跟着本函数/租约
    auto response =  // 先声明，后面用立即调用 lambda 一次性求出两条路径中某一条的结果
        [&]() -> tl::expected<BatchGetOffloadObjectResponse, ErrorCode> {  // IIFE：局部 lambda 立刻执行，返回统一类型
        if (!local_batch) {  // 跨进程常规路径
            return client_requester_->batch_get_offload_object(
                target_rpc_service_addr, storage_keys, restore_sizes);  // 真正发出第 18 步的 RPC
        }
        auto result = file_storage_->BatchGetLocal(storage_keys, restore_sizes);  // 同进程：直接读自己后端并分配本地 buffer
        if (!result) {
            return tl::unexpected(result.error());  // 失败时把 FileStorage 的 ErrorCode 原样透传
        }
        local_owner.emplace(std::move(result.value()));  // 把 batch 的所有权移进 local_owner，防止 buffer 提前释放
        return BatchGetOffloadObjectResponse(0,  // batch_id=0：本地路径不存在远端可释放的 batch 号
                                             std::move(local_owner->pointers),  // 目标指针数组（bytes 地址）交给调用方
                                             client_->GetSegmentEndpoint(), 0);  // TE 地址填自己的 segment；gc_ttl_ms=0 表示无远端租约
    }();  // 注意这一对括号：lambda 被立即调用，response 拿到的就是结果
    if (!response) {
        LOG(ERROR) << "Batch get offload object failed with error: "
                   << response.error();  // ErrorCode 有 operator<<，直接打进日志
        return tl::unexpected(response.error());  // 失败时不构造租约，本地 buffer 由 local_owner 析构自动回收
    }
```

**这一段在做什么**

- 用立即调用的 lambda 把"RPC 还是本地"两条分支收敛成一个 `tl::expected<BatchGetOffloadObjectResponse, ErrorCode>`。
- 非常规路径下直接 `batch_get_offload_object`，参数是租户作用域 key 和存储侧长度。
- 本地路径调用 `FileStorage::BatchGetLocal`，把返回的 batch 存进 `local_owner` 保活。
- 本地路径手工拼一个响应：`batch_id=0`、`pointers` 来自本地分配、`transfer_engine_addr` 填自己的 segment、`gc_ttl_ms=0`。
- 任何失败都原样上抛错误码，并保证本地 buffer 随 `local_owner` 析构释放。

**重点语法 / 惯用法**

- `[&]() -> tl::expected<...> { ... }()`：IIFE（立即调用函数表达式），用返回值初始化 `auto response`，避免写一个只调用一次的自由函数。
- `std::optional<T>::emplace` 原地构造，`std::move(result.value())` 把 `LocalBatchResult` 里的 `shared_ptr<void> owner` 转移进来。
- lambda 捕获 `[&]` 引用 `local_owner`，所以 `emplace` 对外部变量生效；这也是 IIFE 能"带出"状态的原因。
- 返回类型显式写成 `tl::expected<...>`，让两条 `return` 分支的隐式转换有明确目标类型。
- `BatchGetOffloadObjectResponse` 是 `YLT_REFL` 注册的 RPC 结构（字段顺序 `batch_id, pointers, transfer_engine_addr, gc_ttl_ms`）。

**与 6.7.3 的差异**

- 图第 18 步把"读者调用"和"owner 返回"画成一次往返，实际同进程时这一步完全没有网络。
- 图里回包一定有 `batch_id`，本地路径固定填 0，因此第 24 步的 `release_offload_buffer` 不能用 batch_id 释放任何远端资源。
- 图没有体现 `transfer_engine_addr` 在本地路径被填成本机 segment endpoint，这是第 23 步"同进程 memcpy"判断的依据之一。
- 图里 `gc_ttl_ms` 像是总存在，本地路径填 0，靠 `local_batch_` 短路避免误判超时。

#### 4.2.4 构造租约与回包自检

mooncake-store/src/real_client.cpp:8938-8947

```cpp
    auto lease = std::make_unique<OffloadRestoreLease>(  // RAII：离开作用域自动 release（本地路径除外）
        this, target_rpc_service_addr, local_batch, std::move(local_owner),  // local_owner 交给租约持有，保证 buffer 活到租约死亡
        std::move(*response), start_time);  // 回包整体 move，避免拷贝 pointers 数组
    if (lease->response().pointers.size() != keys.size()) {  // owner 少给/多给指针都会让下游按下标写入错位
        LOG(ERROR) << "Pointer count mismatch from owner: expected="
                   << keys.size()
                   << ", got=" << lease->response().pointers.size();
        return tl::unexpected(ErrorCode::INVALID_PARAMS);  // lease 在此处析构 → 自动触发一次远端 release，不会泄漏
    }
    return lease;  // 成功：调用方用 lease->response() 取 pointers / transfer_engine_addr / gc_ttl_ms
```

**这一段在做什么**

- 把两条路径的结果统一包成 `OffloadRestoreLease`，同时把 `local_owner` 的所有权移交给它。
- 校验 `pointers.size() == keys.size()`，保证后面按 key 下标取指针不会越界或错位。
- 失败时提前 return，`lease` 是智能指针，析构即执行 `release_offload_buffer`，不会把 owner 侧 buffer 挂住。
- 成功时把租约交给调用方，由调用方决定何时结束（读取完成后析构）。

**重点语法 / 惯用法**

- `std::make_unique<T>(...)` 直接构造，比 `unique_ptr(new T)` 少一次失败路径考虑。
- `std::move(*response)` 把 `tl::expected` 里的值移出；此前已确认 `response` 有值。
- 用 `lease->response()` 而不是保存 `response` 引用：租约是唯一持有者，语义更清楚。
- 返回 `std::unique_ptr` 的 `tl::expected` 本身不可拷贝，函数天然是"一次性所有权转移"。
- 校验写成"预期 vs 实得"两个数字都打日志，便于在跨版本 owner 上定位字段不兼容。

**与 6.7.3 的差异**

- 图里第 18 步的返回没有"计数校验"这一步，实际这是防 owner 版本不一致/部分成功的关键闸门。
- 图把 batch 的生命周期简化成"读完后释放"，实际由租约对象析构驱动，失败分支也会释放。
- 图未表现 `local_owner` 与回包 `pointers` 的所有权关系：本地路径下指针指向的内存必须由租约保活，否则会 use-after-free。

#### 4.2.5 ClientRequester：把调用送出去

mooncake-store/src/real_client.cpp:9067-9080

```cpp
tl::expected<BatchGetOffloadObjectResponse, ErrorCode>
ClientRequester::batch_get_offload_object(const std::string &client_addr,  // 目标 owner 的 offload RPC 地址
                                          const std::vector<std::string> &keys,  // 已带 tenant 前缀
                                          const std::vector<int64_t> &sizes) {  // 存储侧完整长度
    auto result =  // 同步风格的返回，内部由 coro_rpc 的协程客户端完成
        invoke_rpc<&RealClient::batch_get_offload_object,  // 模板参数：被调用的服务端成员函数指针
                   BatchGetOffloadObjectResponse>(client_addr, keys, sizes);  // 第二模板参数：期望的回包类型
    if (!result) {
        LOG(ERROR)
            << "Failed to invoke batch_get_offload_object, client_addr = "
            << client_addr << ", error is: " << result.error();  // 连不上/超时在这里统一记日志
    }
    return result;  // 错误原样上抛，由 restore_offload_objects 处理
}
```

**这一段在做什么**

- 这是真实发出 RPC 的地方，`client_addr` 就是 `descriptor.transport_endpoint`。
- `invoke_rpc` 以成员函数指针 + 回包类型为模板参数，把参数打包、连接池取连接、发送、反序列化一条龙做掉。
- 失败只记日志并原样返回错误，不做重试或降级。
- 函数本身不区分 local_batch，那条路径在调用方就已经分叉掉了。

**重点语法 / 惯用法**

- `invoke_rpc<&RealClient::batch_get_offload_object, Response>(addr, args...)`：用非类型模板参数固化"调哪个远程函数、回什么类型"，避免手写字符串路由。
- 连接来自 `ClientRequester` 的 `client_pools_`（`coro_io::client_pool`），池化复用连接、带重连与超时配置。
- 返回值统一 `tl::expected`，与上游风格一致。
- 日志用 `operator<<` 拼接，不做字符串格式化，减少分配。

**与 6.7.3 的差异**

- 图第 18 步的箭头 `R->>FS` 暗示一次网络往返，实际连接是池化复用的，冷启动才有 TCP/RDMA 建连成本。
- 图未表现 `client_addr` 就是 LOCAL_DISK descriptor 的 `transport_endpoint`——它是 owner 的 offload RPC 端口，与 Transfer Engine 端口是两个东西。
- 图未画失败路径：RPC 失败时不会降级为其他副本，这一跳直接把错误抛给用户读请求。

#### 4.2.6 endpoint 从哪来：LOCAL_DISK descriptor

mooncake-store/src/real_client.cpp:4781-4786

```cpp
    if (replica.is_local_disk_replica()) {  // 读者选中的是 LOCAL_DISK 副本
        const auto &endpoint =
            replica.get_local_disk_descriptor().transport_endpoint;  // owner 在 NotifyOffloadSuccess 时登记的 offload RPC 地址
        void *dst = static_cast<char *>(buffer) + dst_offset;  // 用户 buffer 加偏移，作为写入终点
        std::unordered_map<std::string, std::vector<Slice>> objects{
            {key, {{dst, size}}}};  // 单 key 单切片：这就是后面传给 restore_offload_objects 的 objects
```

**这一段在做什么**

- 说明 `restore_offload_objects` 的第一个参数 `target_rpc_service_addr` 的唯一来源：`LocalDiskDescriptor::transport_endpoint`。
- 说明第 2 个参数 `keys` 由 map 的 key 收集而来，第 4 个参数 `objects` 就是这里的 map。
- 单 key 读路径下每个对象只有一个切片，`dst` 由用户 buffer 加 `dst_offset` 得到。
- 这里还没有任何网络动作，只是把"发给谁、写到哪"准备好。

**重点语法 / 惯用法**

- `replica.is_local_disk_replica()` 是副本类型的判定入口，与 `get_local_disk_descriptor()` 配对使用，取错 descriptor 会读到未初始化字段。
- `const auto &endpoint = ...transport_endpoint;` 绑定到 descriptor 内部字符串，避免拷贝。
- `std::unordered_map<std::string, std::vector<Slice>> objects{{key, {{dst, size}}}};` 三层花括号分别是 map 元素、vector 元素、`Slice` 聚合初始化。
- `static_cast<char *>(buffer) + dst_offset`：`void*` 不能直接做算术，必须先转成字符指针。

**与 6.7.3 的差异**

- 图第 16-18 步暗示读者拿到副本就知道去哪读，实际地址藏在 `LocalDiskDescriptor::transport_endpoint` 里。
- 这个地址是 owner 的 offload RPC 端口，与第 22 步回包里的 `transfer_engine_addr` 不是同一个值，两者分别用于"请求"和"搬运"。
- 图未画出 `objects` 的形状：它是 key 到切片列表的映射，一个 key 可以有多块目标切片。

#### 4.2.7 租约析构：远端 batch 的释放

`OffloadRestoreLease` 的完整定义（构造、删除的拷贝/移动、`LeaseExpired` 与析构里的 `release_offload_buffer`）集中在第 24 步，见 **4.6.1**。这里只需要记住一条：`restore_offload_objects` 返回的是 `unique_ptr<OffloadRestoreLease>`，**从这一刻起，无论后续 READ 成功还是提前 return，析构都会发一次释放**（本地 `local_batch` 路径除外）。

### 4.3 BatchGet / LoadBatch / AllocateBatch（第 19 步）

第 19 步在图上是一条 `FS->>BE: BatchLoad`，但真正的顺序是：先 `AllocateBatch` 在 `client_buffer_allocator_`（或 pinned arena）里切出目标 buffer，再 `BatchLoad` 把数据写进去，最后按 `offset_in_buffer` 修正 `slice.ptr` 并回写 `pointers`。`BatchGet` 还额外把整个 batch 登记进 `client_buffer_allocated_batches_`，交给 GC 线程按 `lease_timeout` 回收。

#### 4.3.1 LoadBatch：选 arena、分配、回读、修正指针

mooncake-store/src/file_storage.cpp:230-265

```cpp
tl::expected<std::shared_ptr<FileStorage::AllocatedBatch>, ErrorCode>
FileStorage::LoadBatch(const std::vector<std::string>& keys,  // 已带 tenant 前缀的存储侧 key
                       const std::vector<int64_t>& sizes, bool prefer_pinned) {  // sizes 是每个对象的完整 payload 长度
    const bool use_pinned = prefer_pinned && pinned_restore_arena_allocator_;  // 只有显式偏好且 arena 存在才用它；
                                                                              // 注意 shared_ptr 在这里隐式转 bool
    auto& allocator = use_pinned ? *pinned_restore_arena_allocator_  // 用 pinned arena（可为空时上面已排除）
                                 : *client_buffer_allocator_;  // 否则用常规 client buffer（进程启动时创建）
    auto allocate_res = AllocateBatch(keys, sizes, allocator);  // 先分配：返回 batch_id、slices、pointers、handles
    if (!allocate_res && use_pinned &&
        allocate_res.error() == ErrorCode::BUFFER_OVERFLOW) {  // pinned arena 是固定容量，打满是一种正常情况
        VLOG(1) << "Pinned SSD restore arena exhausted; using default arena";  // 记录降级，便于容量规划
        allocate_res = AllocateBatch(keys, sizes, *client_buffer_allocator_);  // 换常规 arena 重试一次（batch_id 会换新）
    }
    if (!allocate_res) {
        LOG(ERROR) << "Failed to allocate batch objects";
        return tl::make_unexpected(allocate_res.error());  // 两个 arena 都不行才真正失败
    }
    auto allocated_batch = std::move(allocate_res.value());  // 从 expected 里取出 shared_ptr
    auto result = BatchLoad(allocated_batch->slices);  // 关键：slices 是“值传递的引用参数”，后端会就地改 slice.ptr
    if (!result) {
        LOG(ERROR) << "Batch load object failed,err_code = " << result.error();
        return tl::make_unexpected(result.error());  // 失败时 allocated_batch 析构 → handles 归还 arena
    }

    // After BatchLoad, slice.ptr may have been adjusted by offset_in_buffer
    // (for O_DIRECT aligned reads). Update pointers to reflect actual data
    // positions.
    for (size_t i = 0; i < keys.size(); ++i) {  // 按 keys 原始顺序遍历，保证 pointers 与调用方下标一致
        auto it = allocated_batch->slices.find(keys[i]);  // slices 是 map，需要按 key 找回对应切片
        if (it != allocated_batch->slices.end()) {
            allocated_batch->pointers[i] =
                reinterpret_cast<uintptr_t>(it->second.ptr);  // 用后端修正后的 ptr 覆盖分配时的原始地址
        }
    }

    return allocated_batch;  // 交回给 BatchGet / BatchGetLocal
}
```

**这一段在做什么**

- 选择 arena：`prefer_pinned` 且 pinned arena 存在时优先用它，否则用 `client_buffer_allocator_`。
- 调用 `AllocateBatch` 真正切 buffer；pinned arena 打满（`BUFFER_OVERFLOW`）时自动降级到常规 arena 重试一次。
- 把 `allocated_batch->slices` 交给后端 `BatchLoad`，后端可能就地改写 `slice.ptr`。
- 因为 O_DIRECT 读会把指针向前对齐，所以必须按 key 把修正后的 `ptr` 写回 `pointers`。
- 任何失败都靠 `shared_ptr` 析构把 `BufferHandle` 归还，无需手工释放。

**重点语法 / 惯用法**

- `const bool use_pinned = prefer_pinned && pinned_restore_arena_allocator_;` 依赖 `shared_ptr` 的 `explicit operator bool`（此处上下文转换为 bool）。
- `auto& allocator = cond ? *a : *b;` 三目返回引用：两个分支都是左值且类型相同（`ClientBufferAllocator&`），不会产生拷贝。
- `tl::expected` 的 `allocate_res.value()` 用 `std::move` 取出，避免 `shared_ptr` 引用计数抖动。
- `BatchLoad` 形参是 `std::unordered_map<std::string, Slice>&`，调用处传的是 batch 内部 map 的引用，所以后端改的就是这批 slice。
- `reinterpret_cast<uintptr_t>` 把 `void*` 转成字节地址整数，便于跨 RPC 传输（`pointers` 是 `vector<uint64_t>`）。

**与 6.7.3 的差异**

- 图第 19 步只有 `BatchLoad`，实际它前面还有一次完整的分配，后面还有一次指针修正。
- 图没有 pinned arena 与降级重试这条分支；降级会换一个新的 `batch_id`。
- 图把 buffer 分配画在"第 21 步直写 ClientBuffer"里，实际目标 buffer 在 `BatchLoad` 之前就已经分配好了，第 21 步只是往里写。
- 图未体现 `slice.ptr` 会被后端改写、必须回写 `pointers` 这一点，这正是 O_DIRECT 的隐形坑。

#### 4.3.2 BatchGet：登记 batch 供 GC 回收

mooncake-store/src/file_storage.cpp:267-287

```cpp
tl::expected<FileStorage::BatchGetResult, ErrorCode> FileStorage::BatchGet(
    const std::vector<std::string>& keys, const std::vector<int64_t>& sizes) {
    auto start_time = std::chrono::steady_clock::now();  // 计时起点，只用于 VLOG 观测
    auto load_result = LoadBatch(keys, sizes, false);  // 注意第三个参数 false：BatchGet 永远不用 pinned arena
    if (!load_result) return tl::make_unexpected(load_result.error());  // 分配或读取失败直接上抛

    auto allocated_batch = std::move(load_result.value());  // 取出 batch
    uint64_t batch_id = allocated_batch->batch_id;  // 先留住 id，因为下一步会把 batch move 进 map
    BatchGetResult batch_result{batch_id, allocated_batch->pointers};  // 回给调用方的是 id + 指针数组（拷贝一份 vector）

    MutexLocker locker(&client_buffer_mutex_);  // 与 GC 线程、AllocateBatch 的回收扫描互斥
    client_buffer_allocated_batches_.emplace(batch_id,
                                             std::move(allocated_batch));  // map 接管所有权，buffer 活到被 GC 或显式释放
    auto end_time = std::chrono::steady_clock::now();
    auto elapsed_time = std::chrono::duration_cast<std::chrono::microseconds>(
                            end_time - start_time)
                            .count();  // 统计包含分配 + 读盘的总耗时
    VLOG(1) << "Time taken for FileStorage::BatchGet: " << elapsed_time
            << "us, key size: " << keys.size() << ", batch_id: " << batch_id;
    return batch_result;  // 返回的是值拷贝，不是 batch 本身
}
```

**这一段在做什么**

- 调用 `LoadBatch(keys, sizes, false)`，明确不走 pinned 捷径。
- 先保存 `batch_id`，因为紧接着 batch 会被 move 走。
- 在 `client_buffer_mutex_` 保护下把 batch 放进 `client_buffer_allocated_batches_`。
- 返回 `BatchGetResult{batch_id, pointers}`，调用方拿指针但不持有 buffer。
- 记录含分配耗时的整体耗时日志。

**重点语法 / 惯用法**

- `BatchGetResult batch_result{batch_id, allocated_batch->pointers};` 聚合初始化，`pointers` 是拷贝（`vector<uint64_t>`），生命周期独立于 batch。
- `MutexLocker` 是项目封装的 `std::lock_guard` 别名，构造即加锁、析构即解锁。
- `unordered_map::emplace(batch_id, std::move(...))` 键用值、值用 move，避免 `shared_ptr` 计数变化。
- GC 侧按 `lease_timeout`（`AllocateBatch` 里用 `client_buffer_gc_ttl_ms` 算出的绝对时间点）扫描删除。

**与 6.7.3 的差异**

- 图里第 22 步回包带 `batch_id`，但没说明 reader 侧的 `batch_id` 是本地分配的编号，与 owner 侧的 batch 号是两套独立的计数器。
- 图未体现 `client_buffer_allocated_batches_` 这张表：reader 进程内的 buffer 靠它 + GC 线程兜底回收，而不是只靠应用及时释放。
- 图未体现 `BatchGet` 与 `BatchGetLocal` 的差异：前者登记进全局表等 GC，后者靠返回值里的 `shared_ptr owner` 精确释放。
- 图里读取耗时像是只有 IO，实际这里统计的是"分配 + 读盘"。

#### 4.3.3 BatchGetLocal：pinned 路径的返回值语义

mooncake-store/src/file_storage.cpp:289-300

```cpp
tl::expected<FileStorage::LocalBatchResult, ErrorCode>
FileStorage::BatchGetLocal(const std::vector<std::string>& keys,
                           const std::vector<int64_t>& sizes) {
    auto load_result = LoadBatch(keys, sizes, true);  // 第三个参数 true：允许 pinned arena
    if (!load_result) return tl::make_unexpected(load_result.error());

    auto batch = std::move(load_result.value());
    LocalBatchResult result;
    result.pointers = std::move(batch->pointers);  // 指针数组移出去给调用方（RPC 回包字段）
    result.owner = std::move(batch);  // shared_ptr<AllocatedBatch> → shared_ptr<void>，所有权随之转移，
                                     // 只要 owner 活着，handles 就不会归还、buffer 就不会被复用
    return result;
}
```

**这一段在做什么**

- 与 `BatchGet` 唯一的区别是 `prefer_pinned = true`，其余流程完全一致。
- 不登记 `client_buffer_allocated_batches_`，而是把 batch 的所有权直接塞进 `LocalBatchResult::owner`。
- 返回 `pointers` 与 `owner` 分离：前者给调用方读数据，后者保证生命周期。
- 这也解释了 `restore_offload_objects` 里为什么要 `local_owner.emplace(std::move(result.value()))` 并把 `local_owner` 交给租约。

**重点语法 / 惯用法**

- `LocalBatchResult::owner` 是 `std::shared_ptr<void>`，可以隐式接收任意 `shared_ptr<T>`（类型擦除式保活）。
- `owner` 是 private，只有 `FileStorage` 能写，外部只能读 `pointers`，强制"所有权不外卖"。
- `std::move` 两次：一次从 `expected` 取值，一次把 `pointers`、`batch` 分别移出。

**与 6.7.3 的差异**

- 图完全没有这条路径：它只在"读者就是 owner 且目标 buffer 在 accelerator 上"时出现，是一种部署相关的优化。
- 图第 24 步的 `release_offload_buffer` 在本地路径不存在，释放方式是 `LocalBatchResult::owner` 的引用计数归零。
- 图未表现 pinned arena 容量打满会退化成普通 arena，此时 `owner` 指向的是普通 client buffer。

#### 4.3.4 AllocateBatch 前半：batch_id、尺寸校验与总时长

mooncake-store/src/file_storage.cpp:1092-1126

```cpp
tl::expected<std::shared_ptr<FileStorage::AllocatedBatch>, ErrorCode>
FileStorage::AllocateBatch(const std::vector<std::string>& keys,
                           const std::vector<int64_t>& sizes,
                           ClientBufferAllocator& allocator) {  // 由调用方决定用哪个 arena（常规 / pinned）
    if (keys.size() != sizes.size()) {  // 再次防御：LoadBatch 的两个参数来自不同来源
        LOG(ERROR) << "Mismatched keys and sizes count: keys=" << keys.size()
                   << ", sizes=" << sizes.size();
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    auto result = std::make_shared<AllocatedBatch>();  // shared_ptr：BatchGet 存表、BatchGetLocal 交给 owner
    result->batch_id = next_batch_id_.fetch_add(1, std::memory_order_relaxed);  // 从 1 开始全局递增，0 留给本地/未知
    std::chrono::steady_clock::time_point now =
        std::chrono::steady_clock::now();  // 同一批共用同一个起点，保证所有 slice 的租约同时到期
    auto lease_timeout =
        now + std::chrono::milliseconds(config_.client_buffer_gc_ttl_ms);  // 绝对到期时刻；GC 线程按它比较
    static constexpr size_t kDirectIOAlignment = 4096;  // O_DIRECT/uring 的块大小，常见页大小

    u_int64_t total_size = 0;  // 累计逻辑字节数（不含对齐冗余），用于统计与限额观测
    bool gc_triggered = false;  // 本次分配是否已经触发过一次 GC，防止在循环里反复全表扫描
    for (size_t i = 0; i < keys.size(); ++i) {  // 逐 key 分配
        if (sizes[i] < 0 || sizes[i] > config_.local_buffer_size) {  // 负数或超过单 buffer 上限都拒绝
            LOG(ERROR) << "Invalid size for key " << keys[i] << ": " << sizes[i]
                       << " (limit: " << config_.local_buffer_size << ")";
            return tl::make_unexpected(ErrorCode::INVALID_PARAMS);  // 早退：已分配的 handles 随 result 析构归还
        }

        size_t data_size = static_cast<size_t>(sizes[i]);  // 校验过非负，可安全转换
        size_t alloc_size = std::max<size_t>(data_size, 1);  // 0 字节对象也要占 1 字节，避免空分配返回空指针
        if (config_.use_uring) {  // 注意开关是 use_uring，不是 use_o_direct：对齐逻辑挂在 uring 后端上
            // O_DIRECT reads need room to align the pointer and to round an
            // unaligned file range out to full pages.
            alloc_size = align_up(data_size, kDirectIOAlignment) +  // 先向上取整到 4096 的整数倍
                         2 * kDirectIOAlignment;  // 再加一页给指针对齐、一页给文件范围向上取整
        }
```

**这一段在做什么**

- 再校验一次 `keys`/`sizes` 长度，并给每个对象做尺寸上下限检查。
- 分配 `batch_id`：全局原子自增，从 1 起，0 保留给"无远端 batch"的本地路径。
- 用同一时刻算一个绝对 `lease_timeout`，整批共享，GC 按这个时间点回收。
- 每个对象至少分配 1 字节；`use_uring` 时分配 `align_up(data_size,4096) + 2*4096`。
- `gc_triggered` 保证单个 batch 的循环里最多触发一次回收扫描。

**重点语法 / 惯用法**

- `next_batch_id_.fetch_add(1, std::memory_order_relaxed)`：`std::atomic<uint64_t>` 的自增，relaxed 只保证唯一性。
- `u_int64_t` 是 POSIX 暴露的非标准拼写（等价 `uint64_t`），沿用自既有代码。
- `static constexpr size_t kDirectIOAlignment = 4096;` 函数内静态常量，编译期求值且不占对象空间。
- `align_up(size, alignment) = (size + alignment - 1) & ~(alignment - 1)`（storage_backend.h:1069），要求 alignment 是 2 的幂。
- `std::max<size_t>(data_size, 1)` 显式指定模板参数，避免 `int`/`size_t` 推导冲突。

**与 6.7.3 的差异**

- 图里没有"分配"这一步，读者容易以为 `BatchLoad` 自己找 buffer；实际 buffer 由 `AllocateBatch` 精确切出来并登记进 `pointers`。
- 图未体现 `use_uring` 会额外放大分配量（最多多 8 KiB/对象），这直接影响池容量规划。
- 图未体现 `local_buffer_size` 是单个对象的硬上限，超限直接 `INVALID_PARAMS`。
- 图未体现 `lease_timeout` 是整批共享的绝对时刻，而不是每个对象各自计时。

#### 4.3.5 AllocateBatch 后半：分配、GC 兜底与指针对齐

mooncake-store/src/file_storage.cpp:1127-1167

```cpp
        auto alloc_result = allocator.allocate(alloc_size);  // 从 arena 切一块；返回 BufferHandle（RAII 归还）
        if (!alloc_result && !gc_triggered &&  // 第一次失败才尝试回收，避免每轮都全表扫
            &allocator == client_buffer_allocator_.get()) {  // 只对常规 arena 做 GC；pinned arena 是固定容量，扫表也没用
            gc_triggered = true;
            {
                MutexLocker locker(&client_buffer_mutex_);  // 与 BatchGet 登记、GC 线程互斥
                auto gc_now = std::chrono::steady_clock::now();
                for (auto it = client_buffer_allocated_batches_.begin();
                     it != client_buffer_allocated_batches_.end();) {
                    if (gc_now >= it->second->lease_timeout) {  // 租约到期的 batch 才能回收
                        it = client_buffer_allocated_batches_.erase(it);  // erase 返回下一个迭代器，所以不用 ++it
                    } else {
                        ++it;
                    }
                }
            }
            alloc_result = allocator.allocate(alloc_size);  // 回收后再试一次
        }
        if (!alloc_result) {
            return tl::make_unexpected(ErrorCode::BUFFER_OVERFLOW);  // 仍失败 → 调用方决定是否换 arena（pinned 降级在这里发生）
        }

        void* data_ptr = alloc_result->ptr();  // BufferHandle 的起始地址
        if (config_.use_uring) {
            data_ptr =
                reinterpret_cast<void*>((reinterpret_cast<uintptr_t>(data_ptr) +
                                         kDirectIOAlignment - 1) &  // +4095 再掩码：向上对齐到 4096
                                        ~(kDirectIOAlignment - 1));
        }

        total_size += data_size;  // 统计的是逻辑大小
        result->slices.emplace(keys[i], Slice{data_ptr, data_size});  // 切片长度是 data_size，不是 alloc_size：多余部分只是对齐冗余
        // pointers will be adjusted after BatchLoad (offset_in_buffer
        // correction)
        result->pointers.emplace_back(reinterpret_cast<uintptr_t>(data_ptr));  // 先用对齐后的地址占位，BatchLoad 后再按 key 覆盖
        result->handles.emplace_back(std::move(alloc_result.value()));  // 保住 handle：只要它在，这块内存就不会被复用
        result->lease_timeout = lease_timeout;  // 同一批反复赋值同一个值，放在循环里只是写法顺带
    }
    result->total_size = total_size;
    return result;  // 失败路径上 result 析构 → handles 全部归还，不会泄漏
}
```

**这一段在做什么**

- 从 arena 切内存；常规 arena 首次失败时，在锁内扫表回收过期 batch 再重试一次。
- 仍失败返回 `BUFFER_OVERFLOW`，由 `LoadBatch` 决定是否从 pinned arena 降级到常规 arena。
- `use_uring` 时把指针向上对齐到 4096，这就是后面 `offset_in_buffer` 修正的由来。
- 把 `Slice{data_ptr, data_size}` 放进 `slices`，把地址放进 `pointers`，把 `BufferHandle` 放进 `handles`。
- 记录批内共享的 `lease_timeout` 与累计 `total_size`。

**重点语法 / 惯用法**

- `allocator.allocate()` 返回 `std::optional<BufferHandle>` 类似物，用 `if (!alloc_result)` 判断；`alloc_result->ptr()` 取地址，`std::move(alloc_result.value())` 移进 `handles`。
- 迭代器删除惯用法：`it = map.erase(it);` 而不是 `erase(it); ++it;`（后者在 `unordered_map` 上是未定义行为）。
- `(p + 4095) & ~4095` 是经典向上对齐；要求 `p` 的地址空间足够大，这也是 `alloc_size` 要多留一页的原因。
- `slices` 是 `unordered_map<string, Slice>`，`emplace` 遇到重复 key 会静默丢弃第二个，而 `pointers`/`handles` 仍会增长——批内出现重复 key 会造成下标错位。
- `handles` 里的 `BufferHandle` 是 move-only，整个 batch 因此不可拷贝、只可移动。

**与 6.7.3 的差异**

- 图里"直写 ClientBuffer"像是把数据写到已知地址，实际地址是先对齐再写，逻辑起点与分配起点可能相差最多 4095 字节。
- 图未体现 GC 兜底：reader 侧内存不足时会先回收过期 batch，再决定失败或降级。
- 图未体现 `slices` 的 key 是租户作用域 key，而调用方拿到的 `pointers` 按下标与用户 key 对齐，两者靠顺序而不是 key 关联。
- 图未显示 `handles` 与 `pointers` 的分工：前者管生命周期，后者管数据地址。

### 4.4 BatchLoad 的 prepare：槽位探测与 manifest 展开（第 20 步）

第 20 步"root 1 次 + manifest 的 N 个 chunk"在代码里分成两个阶段：先是 prepare 阶段把每个 key 的 root 读出来、解析成 `CachedManifest`（这一步才是那个"root 1 次"），再是后面的 chunk 读取阶段按 manifest 逐块拉取并 `memcpy` 到目标 buffer。本小节覆盖 prepare 阶段。

#### 4.4.1 槽位上限常量：一个 key 最多 64 个物理槽

mooncake-store/include/nvme_kv/key_codec.h:12-18

```cpp
using NvmeKvPhysicalKey = std::array<uint8_t, 16>;  // 真正下发给 NVMe KV 的物理键：固定 16 字节

struct NvmeKvObjectIdentity {
    std::string logical_key;  // 逻辑 key（含租户前缀），物理键由它派生
};

constexpr uint32_t kNvmeKvMaxPhysicalKeySlots = 64;  // 一个逻辑 key 最多占 64 个物理槽；读时只能线性试
```

**这一段在做什么**

- 定义物理键类型：固定 16 字节的 `std::array<uint8_t, 16>`，直接作为 NVMe KV 的键。
- 定义逻辑身份 `NvmeKvObjectIdentity`，目前只有一个 `logical_key` 字段。
- 定义槽位数上限 `kNvmeKvMaxPhysicalKeySlots = 64`，写入最多占 64 个槽，读取最多试 64 次。
- 这三个声明是后端"没有目录、只能按 hash+slot 派生键再逐个探测"这一设计的前提。

**重点语法 / 惯用法**

- `using X = std::array<...>` 是类型别名声明，比 `typedef` 更易读。
- `std::array<uint8_t, 16>` 是聚合类型，长度编译期固定，可 memcpy、可当哈希键。
- `constexpr uint32_t` 编译期常量，循环上界与 `EncodeNvmeKvPhysicalKey` 的 `slot` 取值域共用同一个值，避免两处写死。
- 头文件里的 `struct` 只声明字段、不带方法，保持纯数据、便于跨模块共享。

**与 6.7.3 的差异**

- 图第 20 步只写"root 1 次"，没有任何"槽位"概念；实际读之前要先知道最多可能有 64 个候选键。
- 图未体现物理键是 16 字节定长，而逻辑 key 是变长字符串，两者之间靠 `EncodeNvmeKvPhysicalKey` 转换。
- 这个 64 不是"分片数"，而是**同一逻辑 key 的冲突槽位上限**，图里完全没有对应的位置。

#### 4.4.2 ResolveRoot：没有目录时的 64 槽线性探测

mooncake-store/src/nvme_kv/backend.cpp:141-184

```cpp
struct ResolvedRoot {
    NvmeKvPhysicalKey physical_key{};  // 命中的物理键（16 字节），后续 chunk 键由同一 identity 派生
    uint32_t slot = 0;  // 命中的槽号，稍后要与 stored_identity.resolved_slot 交叉校验
    std::string object_blob;  // root 的完整 blob（header + identity metadata + payload），在内存里解析
};

tl::expected<std::optional<ResolvedRoot>, ErrorCode> ResolveRoot(
    NvmeKvConnector &connector, const std::string &logical_key) {
    const NvmeKvObjectIdentity identity{.logical_key = logical_key};  // 指定初始化器（C++20），其余字段默认
    for (uint32_t slot = 0; slot < kNvmeKvMaxPhysicalKeySlots; ++slot) {  // 64 槽线性探测：后端没有目录，只能逐个试
        const auto physical_key = EncodeNvmeKvPhysicalKey(identity, slot);  // 物理键 = f(hash(identity), slot)
        auto object_res = connector.Retrieve(physical_key);  // 一次 NVMe KV 读：第 20 步里“root 1 次”的那一次
        if (!object_res) {
            if (object_res.error() == ErrorCode::OBJECT_NOT_FOUND) {
                continue;  // 空槽不是错误，继续试下一个槽
            }
            return tl::make_unexpected(object_res.error());  // 真实 IO 错误立刻上抛，不能当空槽吞掉
        }

        NvmeKvObjectHeader header{};
        std::string_view identity_metadata;  // string_view 指向 object_res 内部，不拷贝
        std::string_view payload;
        if (!ParseNvmeKvObjectBlob(object_res.value(), header,
                                   identity_metadata, payload)) {  // 解析失败说明这条物理键上的数据不是本格式
            return tl::make_unexpected(ErrorCode::FILE_READ_FAIL);
        }

        NvmeKvStoredIdentityView stored_identity{};
        if (!ParseNvmeKvStoredIdentity(identity_metadata, stored_identity)) {
            return tl::make_unexpected(ErrorCode::FILE_READ_FAIL);
        }
        if (stored_identity.logical_key != logical_key) {
            continue;  // 槽被别的 key 占了（哈希冲突），换下一个槽
        }
        if (stored_identity.resolved_slot != slot ||  // 写入时记录的槽号必须与本次探测槽一致，防串槽/防迁移残留
            !NvmeKvKeyConflictPolicy::ValidateResolvedRootPlacement(
                identity, stored_identity, physical_key)) {  // 再校验物理键确实是该 identity 的合法落点
            return tl::make_unexpected(ErrorCode::FILE_READ_FAIL);
        }
        return std::optional<ResolvedRoot>(
            ResolvedRoot{physical_key, slot, std::move(object_res.value())});  // 命中：把 blob move 出来，避免再拷一份
    }
    return std::nullopt;  // 64 槽全空 → 对象不存在（用 optional 表达“没有”，与 IO 错误区分开）
}
```

**这一段在做什么**

- 逻辑 key 派生 16 字节物理键，`slot` 是派生参数，因此一个 key 最多有 64 个候选物理键。
- 逐个槽 `Retrieve`，`OBJECT_NOT_FOUND` 视为空槽继续，其他错误立即上抛。
- 解析 root blob，校验里面记录的 `logical_key` 与探测 key 一致，否则视为哈希冲突继续探测。
- 再校验 `resolved_slot == slot` 且物理键落点合法，任一不满足均判定数据损坏。
- 命中则 move 出 blob，返回 `optional<ResolvedRoot>`；64 槽全空返回 `nullopt`。

**重点语法 / 惯用法**

- `tl::expected<std::optional<ResolvedRoot>, ErrorCode>`：三层语义——错误码、未找到（nullopt）、找到。
- `std::string_view` 切分二进制 blob 的各个区段，零拷贝；但它的生命周期依附于 `object_blob`，源码注释也强调要实现方保证。
- 指定初始化器 `NvmeKvObjectIdentity{.logical_key = logical_key}` 是 C++20 特性，字段顺序必须与声明一致。
- `for (uint32_t slot = 0; slot < 64; ++slot)` 里的 `continue` 是"空槽/冲突"，`return` 是"致命错误"，两种分支严格区分。
- `std::move(object_res.value())` 把串移进 `ResolvedRoot`，避免多一次整块拷贝。

**与 6.7.3 的差异**

- 图第 20 步写"root 1 次"，最坏情况其实是 64 次 `Retrieve`（全部槽为空或冲突）；"1 次"是命中第一个槽时的常态。
- 图把 root 读画进"manifest 的 N 个 chunk"同一批 IO，实际 root 探测在 prepare 阶段串行于单个 plan，chunk 读取在后面独立批量提交。
- 图未体现"空槽"与"IO 错误"的严格区分，以及哈希冲突时继续探测的控制流。
- 图未体现槽号交叉校验，这是防止旧版本残留/迁移后串读的关键一致性检查。

#### 4.4.3 BatchLoad 入口：ReadPlan 与 manifest 缓存查询

mooncake-store/src/nvme_kv/backend.cpp:1167-1192

```cpp
tl::expected<void, ErrorCode> NvmeKvStorageBackend::BatchLoad(
    std::unordered_map<std::string, Slice> &batched_slices) {  // 引用传入：后端会就地改 slice.ptr（对齐修正）
    if (!initialized_.load(std::memory_order_acquire)) {  // acquire 与初始化路径的 release 配对
        return tl::make_unexpected(ErrorCode::INTERNAL_ERROR);  // 未初始化直接拒绝
    }

    const auto connector = connector_;  // 拷贝一份 shared_ptr，避免后续用到半析构的成员
    if (connector == nullptr) {
        return tl::make_unexpected(ErrorCode::OBJECT_NOT_FOUND);  // 没有设备连接 → 当作对象不可用
    }

    struct ReadPlan {
        std::string key;  // 逻辑 key
        Slice dest_slice;  // 目标 buffer 与长度（来自 AllocateBatch）
        std::shared_ptr<const CachedManifest> cached_manifest;  // 缓存命中则为非空，可跳过整个 prepare
    };
    std::vector<ReadPlan> plans;
    plans.reserve(batched_slices.size());  // 预分配，plan 与后续 prepared_reads 下标一一对应

    for (const auto &[key, dest_slice] : batched_slices) {  // 结构化绑定遍历 map
        if (dest_slice.size > std::numeric_limits<uint32_t>::max()) {  // 元数据里长度字段是 uint32
            return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
        }
        plans.push_back(ReadPlan{key, dest_slice,
                                 FindCachedManifest(key, dest_slice.size)});  // 按 key 查 manifest 缓存；长度也算匹配条件
    }
```

**这一段在做什么**

- 用 acquire 语义检查初始化标志，再拷一份 `connector_` 的 `shared_ptr` 保活。
- 为每个 key 建一个 `ReadPlan`，携带 key、目标 `Slice`、以及可能命中的 `CachedManifest`。
- 长度超过 `uint32_t` 上限直接拒绝，因为 manifest/header 里的长度字段是 32 位。
- 缓存查询把 `payload_size` 作为匹配条件之一，长度变了就视为未命中。

**重点语法 / 惯用法**

- `const auto connector = connector_;` 拷贝 `shared_ptr`（不是引用），让连接在整个 `BatchLoad` 期间保持存活。
- `for (const auto &[key, dest_slice] : batched_slices)` 结构化绑定；注意 `key`/`dest_slice` 是引用，`ReadPlan` 里存的是拷贝。
- `std::numeric_limits<uint32_t>::max()` 表达"字段宽度上限"，比魔法数字清楚。
- `plans` 与后面的 `prepared_reads` 都用同一下标，靠 `reserve` 保证不会因扩容导致引用失效。

**与 6.7.3 的差异**

- 图第 20 步直接从 `BatchLoad` 到设备读，实际先做了一轮"每个 key 是否有 manifest 缓存"的判定，命中就不必再读 root。
- 图未体现 `payload_size` 参与缓存键匹配，长度变化会让缓存自然失效。
- 图未体现 `connector_` 是 `shared_ptr`：析构与读取可以并发，靠引用计数保活。

#### 4.4.4 prepare_one 前半：root 探测与身份校验

mooncake-store/src/nvme_kv/backend.cpp:1194-1231

```cpp
    struct PreparedRead {
        std::shared_ptr<const CachedManifest> manifest;  // prepare 的产物：要么来自缓存，要么来自本次解析
    };
    std::vector<PreparedRead> prepared_reads(plans.size());  // 与 plans 同下标；默认构造即空 manifest
    const auto prepare_one =
        [&](size_t plan_index) -> tl::expected<void, ErrorCode> {  // 单个 plan 的准备工作，可被并行调度
        const auto &plan = plans[plan_index];
        auto resolved_res = ResolveRoot(*connector, plan.key);  // 这一步才真正读 root（64 槽探测）
        if (!resolved_res) {
            return tl::make_unexpected(resolved_res.error());  // 探测过程中的 IO/格式错误
        }
        if (!resolved_res.value()) {
            return tl::make_unexpected(ErrorCode::OBJECT_NOT_FOUND);  // optional 为空：64 槽都没找到
        }
        const auto &resolved = *resolved_res.value();
        const std::string &object_blob = resolved.object_blob;  // 注意是引用：blob 活得和 resolved 一样久
        NvmeKvObjectHeader header{};
        std::string_view identity_metadata_view;
        std::string_view payload_view;
        if (!ParseNvmeKvObjectBlob(object_blob, header, identity_metadata_view,
                                   payload_view)) {
            return tl::make_unexpected(ErrorCode::FILE_READ_FAIL);
        }

        NvmeKvStoredIdentityView stored_identity_view{};
        if (!ParseNvmeKvStoredIdentity(identity_metadata_view,
                                       stored_identity_view) ||
            stored_identity_view.logical_key.empty() ||  // 空 key 说明元数据坏了
            stored_identity_view.logical_key != plan.key) {  // 双重确认：ResolveRoot 已查过，这里防 blob 被换
            return tl::make_unexpected(ErrorCode::FILE_READ_FAIL);
        }
        NvmeKvObjectIdentity identity{.logical_key =
                                          stored_identity_view.logical_key};  // 用元数据里的 key 重建 identity
        if (!NvmeKvKeyConflictPolicy::ValidateResolvedRootPlacement(
                identity, stored_identity_view, resolved.physical_key) ||  // 物理键落点合法性
            stored_identity_view.resolved_slot != resolved.slot) {  // 槽号一致性
            return tl::make_unexpected(ErrorCode::FILE_READ_FAIL);
        }
```

**这一段在做什么**

- `PreparedRead` 只存 manifest，是 prepare 阶段唯一的产物。
- `prepare_one` 是单个 plan 的完整准备流程，可被并行调度。
- 先 `ResolveRoot`，把"optional 为空"翻译成 `OBJECT_NOT_FOUND`，把错误码原样上抛。
- 解析 root blob 得到 header、identity 元数据、payload 三个视图。
- 用元数据里的 key 重建 `identity`，再做物理键落点与槽号一致性校验（与 `ResolveRoot` 内部的检查互为双保险）。

**重点语法 / 惯用法**

- `const auto prepare_one = [&](size_t i) -> tl::expected<void, ErrorCode> { ... };`：带显式返回类型的捕获式 lambda，便于在并行框架里当任务提交。
- `if (!resolved_res.value())` 用 `optional` 的布尔转换判断"未找到"，与 `!resolved_res`（错误）区分开。
- `stored_identity_view.logical_key != plan.key` 用 `string_view` 与 `std::string` 比较，无需构造临时串。
- 指定初始化器 `NvmeKvObjectIdentity{.logical_key = ...}` 跨行书写，`=` 后换行只是格式。
- 引用 `const std::string &object_blob = resolved.object_blob;` 保证 `string_view` 的底层存储不被提前释放。

**与 6.7.3 的差异**

- 图把 root 读当成"设备读"的一部分，实际它在 prepare 里完成并且会做大量元数据校验，失败会在 chunk 读之前就返回。
- 图未体现同一份数据被 `ResolveRoot` 和 `prepare_one` 校验了两遍（前者查冲突，后者防 blob 被替换）。
- 图未体现 prepare 可以被并发调度（`prepare_concurrency_`），读放大与并发度相关。

#### 4.4.5 inline 分支：一次 memcpy 结束

mooncake-store/src/nvme_kv/backend.cpp:1233-1259

```cpp
        const auto object_type =
            static_cast<NvmeKvObjectType>(header.object_type);  // 元数据里存的是整数，强转成枚举
        const uint32_t expected_identity_size =
            ComputeNvmeKvStoredIdentityMetadataSize(identity);  // 由 identity 反推应该有多长的元数据
        if (object_type == NvmeKvObjectType::kInline) {  // 小对象：payload 直接内联在 root 里，没有 chunk
            if (!ValidateNvmeKvHeader(
                    header, identity, expected_identity_size,  // 元数据长度/identity 一致性
                    static_cast<uint32_t>(plan.dest_slice.size),  // 期望的 payload 长度 = 目标切片长度
                    NvmeKvObjectType::kInline) ||  // 类型也必须一致
                header.payload_checksum !=
                    ComputeNvmeKvPayloadChecksum(payload_view) ||  // 内容校验：算一遍整块 payload 的校验和
                payload_view.size() != plan.dest_slice.size) {  // 长度兜底：校验和相同但长度不同也要拒绝
                return tl::make_unexpected(ErrorCode::FILE_READ_FAIL);
            }
            std::memcpy(plan.dest_slice.ptr, payload_view.data(),
                        payload_view.size());  // 直接把 root 里的 payload 拷进目标 buffer，本次读到此结束
            return {};  // 空 expected 表示成功；没有 chunk 阶段
        }

        if (object_type != NvmeKvObjectType::kManifest ||  // 走到这里只允许 manifest 类型
            !ValidateNvmeKvHeader(header, identity, expected_identity_size,
                                  static_cast<uint32_t>(payload_view.size()),  // manifest 的 payload 是元数据，长度随 chunk 数变
                                  NvmeKvObjectType::kManifest) ||
            header.payload_checksum !=
                ComputeNvmeKvPayloadChecksum(payload_view)) {  // manifest 本体也要过校验和
            return tl::make_unexpected(ErrorCode::FILE_READ_FAIL);
        }
```

**这一段在做什么**

- 从 header 取出对象类型，并算出 identity 元数据应有的长度。
- inline 对象：校验 header 各字段、payload 校验和、payload 长度，全部通过后 `memcpy` 到目标切片并直接返回。
- 非 inline 时只接受 `kManifest` 类型，同样校验 header 与整块 payload 的校验和。
- inline 分支是整个 `BatchLoad` 里唯一"读到就结束"的路径，不产生任何 chunk 读。

**重点语法 / 惯用法**

- `static_cast<NvmeKvObjectType>(header.object_type)` 把线上整数转枚举；非法值不会自动报错，靠后面的 `!=` 判断兜住。
- 多重条件用 `||` 串联，短路求值意味着前面失败就不再算校验和（校验和是 CPU 热点，这个顺序是对的）。
- `std::memcpy(dst, src, n)` 要求两块内存不重叠；这里 dst 是用户 buffer、src 是刚读出的 root blob，天然不重叠。
- `return {};` 构造一个"成功但无值"的 `tl::expected<void, ErrorCode>`。
- `payload_view.data()` 来自 `std::string` 底层存储，长度由 `size()` 给出，不做 `strlen` 语义。

**与 6.7.3 的差异**

- 图第 20 步写"root 1 次 + manifest 的 N 个 chunk"，inline 对象实际是"root 1 次、chunk 0 次"，`memcpy` 就地完成。
- 图未体现 inline 与 manifest 是同一个 root 键下的两种 header 类型，而非两种键。
- 图未体现校验和与长度都要过，任一不符都返回 `FILE_READ_FAIL`。
- 图把设备直写与 memcpy 并列在第 21 步，inline 情况下这一步实际是 CPU memcpy，不经过设备直写路径。

#### 4.4.6 manifest 解析与 CacheManifest

mooncake-store/src/nvme_kv/backend.cpp:1261-1277

```cpp
        NvmeKvManifestMetadata metadata{};
        std::vector<NvmeKvManifestChunkRecord> chunk_records;  // manifest 里每个 chunk 的物理键/长度/校验和
        if (!ParseNvmeKvManifest(payload_view, metadata, chunk_records) ||  // 反序列化 manifest
            metadata.logical_payload_size != plan.dest_slice.size ||  // 声明的逻辑总长必须等于目标切片长度
            metadata.chunk_count != chunk_records.size()) {  // 声明条数与实际解析条数必须一致
            return tl::make_unexpected(ErrorCode::FILE_READ_FAIL);
        }

        auto cached_manifest =
            CacheManifest(plan.key, plan.dest_slice.size,
                          stored_identity_view.resolved_slot, chunk_records);  // 校验 chunk 偏移并写入进程内缓存
        if (cached_manifest == nullptr) {  // 校验失败（如 chunk 重叠/越界）返回空
            return tl::make_unexpected(ErrorCode::FILE_READ_FAIL);
        }
        prepared_reads[plan_index].manifest = std::move(cached_manifest);  // 交给后续 chunk 读取阶段
        return {};
    };  // prepare_one lambda 结束；上面每个 return 对应一个 plan 的准备结果
```

**这一段在做什么**

- 反序列化 manifest，得到元数据与 chunk 记录数组。
- 做两项一致性检查：声明的逻辑总长等于目标切片长度、声明的 chunk 数等于实际解析出的条数。
- 调 `CacheManifest`：它会校验 chunk 偏移合法性，并把 manifest 放进进程内缓存。
- 把缓存返回的 `shared_ptr<const CachedManifest>` 存进 `prepared_reads[plan_index]`，供 chunk 阶段使用。

**重点语法 / 惯用法**

- `std::vector<NvmeKvManifestChunkRecord>` 作为 out 参数按引用传入，解析器直接往里 `push_back`。
- `CacheManifest` 返回 `shared_ptr<const CachedManifest>`：const 表示缓存内容不可被调用方修改，多个 reader 共享同一份。
- 返回 `nullptr` 表达"校验失败"，与 `tl::expected` 的错误路径混用（因为缓存写入不是本函数的主错误源）。
- `prepared_reads` 是预先按 `plans.size()` 构造的 vector，多线程写不同下标是安全的（前提是各线程下标不重叠，`RunParallelIo` 保证）。

**与 6.7.3 的差异**

- 图没有 manifest 缓存这一层，实际同一对象被反复读时只需再读一次 root（甚至一次都不读，见下一小节）。
- 图把"解析 manifest"放在 chunk 读之后，实际解析在 prepare 阶段先做完，chunk 读只是照着清单取数据。
- 图未体现 `logical_payload_size`、`chunk_count` 两处一致性检查，这是防半写/防截断的关键。

#### 4.4.7 FindCachedManifest：按 key 与长度查 manifest 缓存

mooncake-store/src/nvme_kv/backend.cpp:354-364

```cpp
std::shared_ptr<const NvmeKvStorageBackend::CachedManifest>
NvmeKvStorageBackend::FindCachedManifest(const std::string &key,
                                         size_t payload_size) const {
    std::shared_lock<std::shared_mutex> lock(manifest_cache_mutex_);  // 读锁：允许多个 reader 并发查
    const auto it = manifest_cache_.find(key);
    if (it == manifest_cache_.end() ||
        it->second->payload_size != payload_size) {  // key 命中但长度不同 → 视为未命中（对象被重写）
        return nullptr;
    }
    return it->second;  // 返回 shared_ptr 拷贝，读锁释放后对象仍存活
}
```

**这一段在做什么**

- 在进程内 `manifest_cache_` 里按逻辑 key 查已解析的 manifest。
- 命中条件有两个：key 存在 **且** 缓存的 `payload_size` 与本次目标长度相等。
- 返回 `shared_ptr<const CachedManifest>`，引用计数保证读锁释放后对象仍然有效。
- 未命中返回 `nullptr`，调用方据此决定是否需要 `ResolveRoot`。

**重点语法 / 惯用法**

- `std::shared_lock<std::shared_mutex>` 是读锁（共享锁），与 `CacheManifest` 里的 `std::unique_lock` 构成读写分离。
- 函数是 `const` 成员函数却锁了 `mutable std::shared_mutex manifest_cache_mutex_`，这是缓存类成员的常规写法。
- 返回 `shared_ptr` 的拷贝而不是引用，避免调用方持有引用时缓存被并发替换。
- `it->second->payload_size` 说明缓存条目本身记录长度，长度是缓存有效性的一部分。

**与 6.7.3 的差异**

- 图里完全没有 manifest 缓存层，第 20 步被画成每次都读 root。
- 图未体现 `payload_size` 参与匹配：对象被重写成不同长度时会自然失效，不需要显式删除。
- 这张缓存是**进程内**的，多 reader 进程之间不共享；图里"同一对象被多个 R 读"的场景每个 R 各有一份缓存。

#### 4.4.8 缓存命中跳过整段 prepare

mooncake-store/src/nvme_kv/backend.cpp:1297-1317

```cpp
    std::vector<tl::expected<void, ErrorCode>> prepare_results(plans.size());
    std::vector<size_t> prepare_misses;
    prepare_misses.reserve(plans.size());  // 只收集未命中缓存的 plan
    for (size_t plan_index = 0; plan_index < plans.size(); ++plan_index) {
        if (plans[plan_index].cached_manifest == nullptr) {  // 缓存未命中 → 稍后真正去读 root
            prepare_misses.push_back(plan_index);
            continue;  // 跳过下面两行：不写 manifest、不标记结果
        }
        prepared_reads[plan_index].manifest = plans[plan_index].cached_manifest;  // 命中：直接复用缓存的 manifest
        prepare_results[plan_index] = {};  // 预置“成功”，避免后面统一遍历时把默认构造的 expected 当失败
    }
    try {
        RunParallelIo(prepare_misses.size(), [&](size_t index) {
            const size_t plan_index = prepare_misses[index];  // 两层下标的映射：任务序号 → plans 下标
            prepare_results[plan_index] = prepare_one(plan_index);  // 只对未命中的 plan 读 root
        });
        for (const auto &result : prepare_results) {  // 统一检查：命中项已在上面置为成功
            if (!result) {
                return tl::make_unexpected(result.error());
            }
        }
        // ...（略：try 块后续的 chunk 读取、直写与收尾，见第 20 步后半）
```

**这一段在做什么**

- 主循环只做分流：未命中的 plan 收进 `prepare_misses`，命中的直接写 `prepared_reads` 并预置结果成功。
- 真正读 root 的只有 `prepare_misses` 里的下标，命中项被整段跳过（不探测、不解析、不校验）。
- `RunParallelIo` 按 `prepare_concurrency_` 并行执行 `prepare_one`，结果写回原 `plan_index`。
- 最后统一遍历 `prepare_results`，任一失败即整体返回，不做部分成功。
- `try` 块内后续还有 chunk 读取与收尾，与本步无关，故略。

**重点语法 / 惯用法**

- `prepare_results[plan_index] = {};` 用默认构造的 `tl::expected<void, ErrorCode>` 表示"成功且无值"。
- `RunParallelIo(n, task, max_inflight)` 内部是 `atomic<size_t> next_index` + `std::latch`：工作线程抢下标，主线程等 `count_down` 到零。
- 下标间接层 `prepare_misses[index] → plan_index` 让并行框架只看到紧凑任务号，结果仍写回原下标（各线程写不同下标，无需同步）。
- `try` 包住并行段，线程池任务抛出的异常会被捕获并转成错误，不穿透到调用方。
- `std::vector<size_t> prepare_misses` 配 `reserve(plans.size())`，最坏情况（全部未命中）也不扩容。

**与 6.7.3 的差异**

- 图第 20 步固定"root 1 次"，缓存全命中时这一次也省掉，该 key 的 prepare 阶段是零 IO。
- 图未体现 prepare 的并发度由 `prepare_concurrency_` 控制，且与 `io_parallelism_` 取较小值。
- 图未体现"部分命中、部分未命中"的混合批次是常态，两者在同一次调用里走不同路径。
- 图把第 20 步画成单一顺序流程，实际 prepare 与后续 chunk 读取是两个阶段，中间还隔着一次全量结果检查。

### 4.5 BatchLoad 的读与校验流水线（第 20–21 步）

第 20 步在代码里其实分成两件事：**先把每个对象的 root 读出来并解析 manifest**（`prepare_one`，见 4.4），**再按 manifest 把 N 个 chunk 读进用户 buffer**（本节）。`BatchLoad` 的入参是 `std::unordered_map<std::string, Slice> &`，所以"用户 buffer"就是调用方（`AllocateBatch` 分配的 client buffer）给的 `Slice`，读路径的终点从第一行起就已经确定，不需要再拷一次。

#### 4.5.1 设备能否直接写进用户 buffer 的判据

mooncake-store/src/nvme_kv/backend.cpp:69-76

```cpp
bool CanRetrieveDirectlyInto(const char *data, uint32_t size) {  // 判定"这块用户内存能否当设备 DMA 的目标"
    if (data == nullptr || size == 0) {  // 空指针/零长度没有任何可读内容
        return false;  // 直接否掉，调用方会自动落到 buffer 路径
    }
    const uint32_t alignment = NvmeKvTransferAlignmentBytes();  // 对齐粒度；默认 4096，可由环境变量覆盖
    return alignment != 0 && size % alignment == 0 &&  // 条件一：长度必须对齐；alignment==0 视为不可直读，防除零
           reinterpret_cast<uintptr_t>(data) % alignment == 0;  // 条件二：起始地址也必须对齐，两个条件缺一不可
}
```

**这一段在做什么**

- 这是全节唯一的"分流开关"：返回 `true` 走 `RetrieveIntoBatch`（设备直写目标内存），返回 `false` 走 `RetrieveBufferBatch`（先读进后端缓冲，再 `memcpy`）。
- 对齐粒度来自 `NvmeKvTransferAlignmentBytes()`，默认 `kDefaultNvmeKvTransferAlignmentBytes = 4096`（见 `executor_util.h:23`），不是硬编码。
- `alignment != 0` 是防御性判断：若环境变量把它配成 0，这里返回 `false` 而不是在做 `%` 时崩溃。
- 两个条件分别对应 DMA 的"长度对齐"和"地址对齐"。缺任一条，NVMe 命令的 `data_len`/`addr` 都会破坏对齐要求。

**重点语法 / 惯用法**

- `reinterpret_cast<uintptr_t>(data) % alignment`：把指针当整数取模，是判断地址对齐的标准写法；`uintptr_t` 保证宽度足够放下指针。
- 函数是**纯函数**、无日志、无异常，便于在热路径每 chunk 调用一次。
- 判据只看"能不能"，不关心"该不该"；是不是尾块并没有被显式判断，而是由 `chunk_offset + payload_size` 是否仍然对齐自然体现出来。

**与 6.7.3 的差异**

- 图上第 21 步画成 `Dev-->>FS: 直写 ClientBuffer / buffer+memcpy` 一个节点，实际是**逐 chunk 独立判定**：同一个对象里可能前 N-1 个 chunk 直读、最后一个走 buffer。
- 判据是"地址 + 长度"双对齐，不是"对象是否对齐"。即使对象起点 4096 对齐，只要某个 chunk 的 `chunk_offsets[i]` 不是 4096 的倍数，这个 chunk 也走 buffer。
- 这里没有检查 `size` 是否超过设备最大传输值；上限由 `Capabilities::effective_max_value_size` 在别处约束，本节不重复。

#### 4.5.2 尾块的落地：`load_chunk_blob` 的"先校验再拷贝"

mooncake-store/src/nvme_kv/backend.cpp:1279-1295

```cpp
    const auto load_chunk_blob =
        [&](size_t plan_index, size_t chunk_index,
            std::string_view chunk_blob) -> tl::expected<void, ErrorCode> {  // 返回 void 的 expected：只表达成败
        const auto &plan = plans[plan_index];  // 该对象的目标 Slice（用户 buffer）
        const auto &prepared = prepared_reads[plan_index];  // 该对象已解析好的 manifest
        const auto &chunk_record =
            prepared.manifest->chunk_records[chunk_index];  // 这个 chunk 的元数据：物理 key、长度、校验和
        if (chunk_blob.size() != chunk_record.payload_size ||  // 长度不符说明设备返回的不是我们要的那块
            chunk_record.payload_checksum !=
                ComputeNvmeKvPayloadChecksum(chunk_blob)) {  // 校验和不符说明数据损坏/串块
            return tl::make_unexpected(ErrorCode::FILE_READ_FAIL);  // 统一映射成读失败，不区分两种原因
        }
        std::memcpy(static_cast<char *>(plan.dest_slice.ptr) +  // 目标 = 用户 buffer 起点
                        prepared.manifest->chunk_offsets[chunk_index],  //     + 该 chunk 在对象内的偏移
                    chunk_blob.data(), chunk_blob.size());  // 源 = 设备写进后端缓冲的那段字节
        return {};  // expected<void> 的成功写法：返回空花括号
    };
```

**这一段在做什么**

- 这是 buffer 路径的收尾：把后端临时缓冲里的 chunk 字节校验后 `memcpy` 到用户 `Slice` 的对应偏移。
- 目标地址用 `plan.dest_slice.ptr + chunk_offsets[chunk_index]` 计算，`chunk_offsets` 由 `ValidateChunkRecords` 按 `payload_size` 累加得出，因此 chunk 在对象内是**连续紧密排列**的。
- 校验是"长度 + 校验和"两道，且**在拷贝之前**完成，避免把坏数据写进用户 buffer 后再回滚。
- 这里返回 `tl::expected<void, ErrorCode>`，成功时 `return {};`，失败时只带 `ErrorCode`，不带上下文。

**重点语法 / 惯用法**

- `std::string_view` 只借用设备缓冲的字节，不做拷贝；真正的一次拷贝发生在 `memcpy`。
- `static_cast<char *>(plan.dest_slice.ptr)`：`Slice::ptr` 是 `void *`，指针算术前必须转成字符指针，否则偏移量按 `void` 算不合法。
- 每次调用都重新取 `prepared.manifest->chunk_records[...]`，没有缓存裸指针，避免 manifest 被替换后悬空。
- 两个 `||` 条件短路：长度不符时不再算校验和，省一次全块哈希。

**与 6.7.3 的差异**

- 图上第 21 步只写"buffer+memcpy"，看不出这里**顺带做了完整性校验**；直读路径的校验被推迟到 4.5.5，两条路径的校验时机并不对称。
- `memcpy` 的目标可能落在 GPU 内存（走 pinned restore arena 时），此处不区分，由调用方保证 `dest_slice.ptr` 在 CPU 侧可写。
- 校验和只覆盖 chunk 的 payload，不覆盖物理 key 或偏移，防的是"数据坏了"，不防"读错块"——读错块由物理 key 的唯一性保证。

#### 4.5.3 组内规划：把一个组拆成直读批次和缓冲批次

mooncake-store/src/nvme_kv/backend.cpp:1319-1378

> 说明：`plans` / `prepared_reads` 的构造（含 inline 对象的就地拷贝）见 4.4；这里从 chunk 级读计划开始。

```cpp
        std::vector<tl::expected<void, ErrorCode>> chunk_results(plans.size());  // 每个对象一个结果槽；默认构造即"成功"
        const size_t read_plan_batch_size = ReadPlanBatchSize();  // 每组处理多少个对象，来自环境变量配置
        const size_t read_group_count =
            (plans.size() + read_plan_batch_size - 1) / read_plan_batch_size;  // 向上取整算组数，避免漏掉最后一组
        struct ChunkContext {  // 把"批次里的第 i 个请求"映射回"哪个对象的哪个 chunk"
            size_t plan_index = 0;
            size_t chunk_index = 0;
        };
        struct ReadBatch {  // 一个组内的两条通道，直读与缓冲各带一份上下文
            std::vector<NvmeKvCommandExecutor::RetrieveIntoRequest>
                direct_requests;  // 设备直写目标内存的请求
            std::vector<ChunkContext> direct_contexts;  // 与 direct_requests 下标一一对应
            std::vector<NvmeKvCommandExecutor::RetrieveBufferRequest>
                buffer_requests;  // 读进后端缓冲的请求
            std::vector<ChunkContext> buffer_contexts;  // 与 buffer_requests 下标一一对应
        };

        const auto read_group = [&](size_t group_index) {  // 每个组一个任务：提交 IO 并把结果落进 chunk_results
            const size_t plan_begin = group_index * read_plan_batch_size;  // 组内第一个对象下标
            const size_t plan_end =
                std::min(plans.size(), plan_begin + read_plan_batch_size);  // 最后一组可能不满，用 min 夹住
            ReadBatch batch;  // 每条通道的请求在组内攒齐后一次性提交

            for (size_t plan_index = plan_begin; plan_index < plan_end;
                 ++plan_index) {  // 先把组内所有 chunk 分类，暂不发 IO
                const auto &prepared = prepared_reads[plan_index];  // 该对象已解析的 manifest
                const auto chunk_count =
                    !prepared.manifest
                        ? 0
                        : prepared.manifest->chunk_records.size();  // inline 对象没有 manifest → 计 0 个 chunk
                if (chunk_count == 0) {  // inline 对象在 prepare 阶段已经 memcpy 完，这里无事可做
                    chunk_results[plan_index] = {};  // 显式标成功，防止默认值语义变化
                    continue;
                }
                for (size_t chunk_index = 0; chunk_index < chunk_count;
                     ++chunk_index) {  // 逐 chunk 决定走直读还是缓冲
                    const auto &chunk_record =
                        prepared.manifest->chunk_records[chunk_index];  // chunk 的物理 key / 长度 / 校验和
                    char *dest =
                        static_cast<char *>(plans[plan_index].dest_slice.ptr) +
                        prepared.manifest->chunk_offsets[chunk_index];  // 该 chunk 在用户 buffer 里的落点
                    if (CanRetrieveDirectlyInto(dest,
                                                chunk_record.payload_size)) {  // 地址和长度都对齐才能直读
                        NvmeKvCommandExecutor::RetrieveIntoRequest request;  // 直读请求：给设备一个真地址
                        request.key = chunk_record.physical_key;  // 16 字节物理 key，决定读哪一块
                        request.data = dest;  // 设备 DMA 目标
                        request.size = chunk_record.payload_size;  // 精确长度，不含对齐填充
                        batch.direct_contexts.push_back(
                            ChunkContext{plan_index, chunk_index});  // 先记上下文，再塞请求，保证下标对齐
                        batch.direct_requests.push_back(std::move(request));  // request 只在批内使用，移动即可
                    } else {  // 尾块或非对齐落点都走这里
                        NvmeKvCommandExecutor::RetrieveBufferRequest request;  // 缓冲请求：只给 key 和长度提示
                        request.key = chunk_record.physical_key;  // 同上
                        request.size_hint = chunk_record.payload_size;  // hint 只用于决定读多少字节，不是硬约束
                        batch.buffer_contexts.push_back(
                            ChunkContext{plan_index, chunk_index});  // 同样保持下标一一对应
                        batch.buffer_requests.push_back(std::move(request));  // 攒批，稍后一次提交
                    }
                }
            }
```

**这一段在做什么**

- 把"每个对象 N 个 chunk"重新组织成"每个组一次批量提交"，分组大小由 `ReadPlanBatchSize()` 决定。
- 关键判断是 `CanRetrieveDirectlyInto(dest, payload_size)`：**落点地址**是 `dest_slice.ptr + chunk_offset`，所以即使对象起点对齐，只要偏移不对齐，该 chunk 也会落到 buffer 通道。
- inline 对象（`manifest == nullptr`）在这里被识别为"0 个 chunk"，直接标成功——因为 `prepare_one` 在解析 root 时已经把它 `memcpy` 到用户 buffer 了。
- 两组请求各自维护 `contexts`，靠**下标**把异步返回的结果映射回 `(plan_index, chunk_index)`；请求结构体里不放序号，避免结构体膨胀。

**重点语法 / 惯用法**

- `(n + b - 1) / b` 是 size_t 域上的向上取整惯用法；`b` 为 0 会除零，这里由配置层保证非 0。
- `chunk_count` 的三目表达式返回 `size_t`：`0` 与 `.size()` 混用时的公共类型推导，避免 `int/size_t` 比较告警。
- 先 `push_back(context)` 再 `push_back(request)` 的顺序是**隐式契约**：任何一处插入失败或条件分支变化都会让两数组错位。
- `std::move(request)` 后局部对象处于有效但未指定状态，循环里没有复用，因此安全。
- `ReadBatch batch;` 每组新建、组结束即析构，天然无跨组状态残留。

**与 6.7.3 的差异**

- 图上第 20 步是"root 1 次 + manifest 的 N 个 chunk"，看起来是一次大提交；实际是**分组 + 组内双通道**：先 root（4.4），再每组一次 `RetrieveIntoBatch`、一次 `RetrieveBufferBatch`。
- 同一个组的 chunk 数可能远大于 `read_plan_batch_size`（分组按对象数，不按 chunk 数），所以单次批量的命令数取决于对象有多少 chunk。
- 图上没有体现"inline 对象不产生任何设备命令"这一分支，它在这一段被 `chunk_count == 0` 吃掉。

#### 4.5.4 两条通道的提交与结果回填

mooncake-store/src/nvme_kv/backend.cpp:1380-1431

```cpp
            if (!batch.direct_requests.empty()) {  // 有对齐的 chunk 才提交直读批次
                connector->RetrieveIntoBatch(batch.direct_requests);  // 同步调用：返回时每个 request.result 已填好
                for (size_t i = 0; i < batch.direct_requests.size(); ++i) {  // 按请求下标遍历，依赖 contexts 同序
                    const auto &context = batch.direct_contexts[i];  // 还原这个请求属于哪个对象
                    const auto &direct_res = batch.direct_requests[i].result;  // 单条命令的执行结果
                    if (!direct_res) {  // 传输/状态错误
                        chunk_results[context.plan_index] =
                            tl::make_unexpected(direct_res.error());  // 整个对象判失败，而不是只丢这个 chunk
                        continue;  // 继续看其它请求，只为把错误尽量记录完整
                    }
                    const auto &chunk_record =
                        prepared_reads[context.plan_index]
                            .manifest->chunk_records[context.chunk_index];  // 取期望长度做交叉校验
                    if (direct_res.value() != chunk_record.payload_size) {  // 设备报告写进去的字节数必须精确相等
                        chunk_results[context.plan_index] =
                            tl::make_unexpected(ErrorCode::FILE_READ_FAIL);  // 少写或多写都算坏数据
                        continue;
                    }
                }
            }

            if (!batch.buffer_requests.empty()) {  // 有尾块/非对齐 chunk 才提交缓冲批次
                connector->RetrieveBufferBatch(batch.buffer_requests);  // 同样同步；后端负责分配临时缓冲
                for (size_t i = 0; i < batch.buffer_requests.size(); ++i) {  // 逐条回填
                    const auto &context = batch.buffer_contexts[i];  // 请求 → 对象的映射
                    if (!chunk_results[context.plan_index]) {  // 该对象已有直读错误，没必要再拷
                        continue;  // 直接跳过，省一次 memcpy
                    }
                    const auto &chunk_res = batch.buffer_requests[i].result;  // 缓冲读结果
                    if (!chunk_res) {  // 读失败
                        chunk_results[context.plan_index] =
                            tl::make_unexpected(chunk_res.error());  // 传播原始错误码（如 OBJECT_NOT_FOUND）
                        continue;
                    }
                    const auto &chunk_buffer = chunk_res.value();  // {owner, data, size} 三元组，owner 管生命周期
                    auto loaded = load_chunk_blob(
                        context.plan_index, context.chunk_index,
                        std::string_view(chunk_buffer.data, chunk_buffer.size));  // 校验 + memcpy 一气呵成
                    if (!loaded) {  // 长度或校验和不符
                        chunk_results[context.plan_index] = loaded;  // expected 直接赋值，保留 FILE_READ_FAIL
                    }
                }
            }

            for (size_t plan_index = plan_begin; plan_index < plan_end;
                 ++plan_index) {  // 组内收尾：把"成功"归一化成 {}
                if (!chunk_results[plan_index]) {  // 已有错误的对象保持错误
                    continue;
                }
                chunk_results[plan_index] = {};  // 幂等归一化，防止默认构造语义被改后出错
            }
        };  // read_group lambda 到此结束（组内对象全部处理完）
```

**这一段在做什么**

- 两条通道**先后**提交（先直读、后缓冲），每条通道内部都是一次批量调用，完成后立即回填 `chunk_results`。
- 直读通道只做"结果存在 + 字节数正确"两项检查，**校验和留到 4.5.5**，因为此时数据已经在用户 buffer 里，再算一遍哈希不影响 IO 并发。
- 缓冲通道复用了 4.5.2 的 `load_chunk_blob`，所以校验和与拷贝都在这里完成，`chunk_results` 里不会留下"半成品"。
- 失败粒度是**对象级**：任一 chunk 失败，`chunk_results[plan_index]` 被置错，后续同对象的 chunk 不再拷贝，最终整个 `BatchLoad` 返回错误（all-or-nothing）。

**重点语法 / 惯用法**

- `RetrieveIntoBatch` / `RetrieveBufferBatch` 是 `void` 返回、通过 `request.result` 出错的"批量 API"风格；调用方必须自己遍历结果，漏检就等于静默失败。
- `chunk_results[...] = loaded;` 把 `tl::expected<void,E>` 从一个 expected 赋给另一个，等价于整块结果搬运，省去先判 `has_value` 再取 `error()`。
- `chunk_buffer` 是 `RetrievedBuffer`，`owner` 是 `shared_ptr<void>`——缓冲在 `chunk_buffer` 析构前一直有效，`memcpy` 期间不会悬空。
- `std::string_view(chunk_buffer.data, chunk_buffer.size)` 不接管所有权，只是给 `load_chunk_blob` 一个只读窗口。
- `if (!chunk_results[context.plan_index]) continue;` 利用 `tl::expected` 的 `operator bool` 表示"有值"，读起来像判空指针。

**与 6.7.3 的差异**

- 图上第 21 步是一个从上到下的箭头，实际是**两波 IO**：直读批先完成，缓冲批后完成；两波之间没有重叠。
- 图上写"直写 ClientBuffer / buffer+memcpy"，容易误解成"二选一"；这里一个对象可以同时出现在两条通道里。
- 直读成功只保证"字节数对"，**不保证内容对**——内容校验在下一段的 `verify_read_group`，因此图上第 21 步结束时数据其实还没被验证。
- `direct_res.value() != chunk_record.payload_size` 这个检查在图上完全没有对应节点，却是**在算校验和之前**发现"设备少写/截断"最直接的手段（即使漏掉，后续 checksum 通常也会失败，但定位会难得多）。

#### 4.5.5 直读结果的延迟校验与 IO/校验流水线

mooncake-store/src/nvme_kv/backend.cpp:1433-1485

```cpp
        const auto verify_read_group = [&](size_t group_index) {  // 与 read_group 同分组，只处理直读 chunk
            const size_t plan_begin = group_index * read_plan_batch_size;  // 与读阶段完全相同的分组边界
            const size_t plan_end =
                std::min(plans.size(), plan_begin + read_plan_batch_size);  // 同样夹住末组
            for (size_t plan_index = plan_begin; plan_index < plan_end;
                 ++plan_index) {
                if (!chunk_results[plan_index]) {  // 读阶段已失败，跳过，避免在坏数据上算哈希
                    continue;
                }
                const auto &manifest = prepared_reads[plan_index].manifest;  // inline 对象这里是 nullptr
                if (manifest == nullptr) {  // 没有 manifest = inline 或无需 chunk 读
                    continue;  // prepare 阶段已校验过 inline 的 checksum
                }
                for (size_t chunk_index = 0;
                     chunk_index < manifest->chunk_records.size();
                     ++chunk_index) {  // 逐 chunk 复算校验和
                    const auto &chunk_record =
                        manifest->chunk_records[chunk_index];
                    const char *dest = static_cast<const char *>(
                                           plans[plan_index].dest_slice.ptr) +
                                       manifest->chunk_offsets[chunk_index];  // 用 const 指针读，表达"只校验不改"
                    if (!CanRetrieveDirectlyInto(dest,
                                                 chunk_record.payload_size)) {  // 非直读 chunk 走 load_chunk_blob，checksum 已在拷贝前验过
                        continue;  // 不重复计算，避免同一块哈希两次
                    }
                    if (chunk_record.payload_checksum !=
                        ComputeNvmeKvPayloadChecksum(std::string_view(
                            dest, chunk_record.payload_size))) {  // 对用户 buffer 里的字节复算
                        chunk_results[plan_index] =
                            tl::make_unexpected(ErrorCode::FILE_READ_FAIL);  // 一个 chunk 坏，整个对象失败
                        break;  // 没必要再验同对象的其它 chunk
                    }
                }
            }
        };

        RunPipelinedIo(read_group_count, read_group, verify_read_group);  // IO 与校验分两池：读第 N+1 组时可校验第 N 组
        for (size_t plan_index = 0; plan_index < plans.size(); ++plan_index) {  // 汇总所有对象的结果
            if (!chunk_results[plan_index]) {  // 只要有一个对象失败
                const auto error = chunk_results[plan_index].error();  // 取第一个遇到的错误码
                return tl::make_unexpected(error);  // 整个 BatchLoad 失败：all-or-nothing
            }
        }
    } catch (const std::exception &error) {  // 线程池/分配等同步异常也不能越过 API 边界
        LOG(ERROR) << "NVMe KV batch load worker failed: " << error.what();
        return tl::make_unexpected(ErrorCode::INTERNAL_ERROR);  // 统一转成 INTERNAL_ERROR，不泄漏实现异常
    } catch (...) {  // 捕获所有非 std 异常，避免 noexcept 边界终止进程
        LOG(ERROR) << "NVMe KV batch load worker failed";
        return tl::make_unexpected(ErrorCode::INTERNAL_ERROR);  // 同上
    }

    return {};  // 所有对象的 chunk_results 都成功，整批成功
}
```

**这一段在做什么**

- `verify_read_group` 只校验**直读 chunk**：非直读 chunk 在 `load_chunk_blob` 里已经"校验后拷贝"，再验一遍是重复开销。
- 校验对象是**用户 buffer 里的最终字节**（`dest_slice.ptr + offset`），所以能发现 DMA 落点错位、截断、脏数据。
- `RunPipelinedIo(group_count, read_group, verify_read_group)` 把"读"和"校验"拆到两个线程池：`io_task` 在 `submit_workers_`，`completion_task` 在 `io_workers_`，于是第 N 组校验与第 N+1 组提交可以重叠。
- 收尾循环把 `chunk_results` 里的第一个错误上抛，`BatchLoad` 整体失败——**all-or-nothing**：不会出现"部分对象成功、部分失败"的半成品返回。

**重点语法 / 惯用法**

- `const char *dest = static_cast<const char *>(...)`：校验路径只读，用 `const` 把契约写进类型里。
- `std::string_view(dest, size)` 把裸内存包装成可哈希视图，`ComputeNvmeKvPayloadChecksum` 内部不必再判长度。
- `RunPipelinedIo` 的第三个参数是**完成回调**，语义是"`io_task(i)` 返回后调 `completion_task(i)`"，不是回调式异步；异常会被捕获进 `item_exceptions` 并在最后重抛。
- 两段 `catch` 分别接 `std::exception` 和 `...`，是跨线程池边界时保命的标准写法（线程函数不能抛出去）。
- `chunk_results` 默认构造是**成功**（`tl::expected<void,E>` 默认有值），所以只有显式赋值才会变成失败；这点决定了"漏填 = 成功"，也是 1424-1430 那段归一化存在的原因。

**与 6.7.3 的差异**

- 图上第 21 步到第 22 步之间看不出校验存在；实际校验是**并行于下一组读**的独立阶段，是这条链路里最容易被忽略的一步。
- "all-or-nothing" 意味着一批里任何一个对象坏掉，**其它已经读好的对象也一起失败**，调用方看到的不是部分成功——图上第 22 步的回包粒度是整批。
- 若 `BatchLoad` 失败，`AllocateBatch` 返回的 `shared_ptr<AllocatedBatch>` 会在 `BatchGet` 的错误路径上析构，整批 client buffer 随引用计数归零释放（见 4.3.1 / 4.3.2），不会留下"读了一半的 batch"。
- `prepare_one` 阶段的失败（root 解析/identifier 校验）在这一段之前就 `return` 了，**不会进入读 chunk 阶段**；图上把 root 和 chunk 画成先后两步，但失败语义是同一个函数级返回。

### 4.6 回包、TE READ 与 release（第 22–24 步）

#### 4.6.1 租约对象：`OffloadRestoreLease` 的 RAII 释放

mooncake-store/src/real_client.cpp:586-636

```cpp
class RealClient::OffloadRestoreLease {  // 一次 offload 读的"租约 + 析构即释放"封装
   public:
    OffloadRestoreLease(
        RealClient *owner, std::string endpoint, bool local_batch,
        std::optional<FileStorage::LocalBatchResult> local_owner,
        BatchGetOffloadObjectResponse response,
        std::chrono::steady_clock::time_point start_time)  // 构造即接管：owner/batch_id/起始时刻全部拷进来
        : owner_(owner),
          endpoint_(std::move(endpoint)),  // owner 的 RPC 地址，释放时要原样发回去
          local_batch_(local_batch),  // true = 走本机 pinned arena，不需要 RPC 释放
          local_owner_(std::move(local_owner)),  // 本机 batch 的所有权，靠它析构时归还 buffer
          response_(std::move(response)),  // 回包：batch_id / pointers / TE 地址 / gc_ttl
          start_time_(start_time) {}  // 租约计时起点，用于 LeaseExpired()

    ~OffloadRestoreLease() {
        if (owner_ == nullptr || local_batch_ ||
            owner_->client_requester_ == nullptr) {  // 三种情况不该发 RPC：已失效/本机批次/无 RPC 通道
            return;  // 本机批次由 local_owner_ 的析构负责归还
        }
        owner_->client_requester_->release_offload_buffer(endpoint_,
                                                          response_.batch_id);  // 第 24 步：通知 owner 可以回收这块 buffer
    }

    OffloadRestoreLease(const OffloadRestoreLease &) = delete;  // 禁止拷贝，否则 batch_id 会被释放两次
    OffloadRestoreLease &operator=(const OffloadRestoreLease &) = delete;  // 同上
    OffloadRestoreLease(OffloadRestoreLease &&) = delete;  // 禁止移动：一旦移动，源对象的析构仍会释放一次
    OffloadRestoreLease &operator=(OffloadRestoreLease &&) = delete;  // 同上，只能通过 unique_ptr 传递

    bool local_batch() const { return local_batch_; }  // 调用方据此选择 memcpy 还是 TE READ
    BatchGetOffloadObjectResponse &response() { return response_; }  // 非 const 版本允许调用方改写 pointers[0]（read_range 偏移）
    const BatchGetOffloadObjectResponse &response() const { return response_; }  // 只读版本

    uint64_t ElapsedMs() const {  // 从拿到回包到现在的毫秒数
        return static_cast<uint64_t>(
            std::chrono::duration_cast<std::chrono::milliseconds>(
                std::chrono::steady_clock::now() - start_time_)
                .count());  // steady_clock 不受系统时间跳变影响
    }

    bool LeaseExpired() const {  // 读者侧的截止判断
        return !local_batch_ && ElapsedMs() >= response_.gc_ttl_ms;  // 本机批次没有 gc 语义，恒不过期
    }

   private:
    RealClient *owner_;  // 不拥有 RealClient，生命周期由调用栈保证
    std::string endpoint_;  // 释放 RPC 的目标
    bool local_batch_;  // 是否本机 pinned arena
    std::optional<FileStorage::LocalBatchResult> local_owner_;  // 本机批次的真正所有者
    BatchGetOffloadObjectResponse response_;  // 回包副本（不是引用）
    std::chrono::steady_clock::time_point start_time_;  // 起始时刻
};
```

**这一段在做什么**

- 把"拿到 buffer → 用 → 还回去"变成一个对象的生命周期：析构函数里发 `release_offload_buffer`，调用方任何路径（成功、失败、异常）都不会漏释放。
- 释放走的是 **owner 的 RPC**（`client_requester_->release_offload_buffer(endpoint_, batch_id)`），不是本地调用，因为 buffer 在 owner 进程里。
- `local_batch_` 为真时**不发 RPC**：数据在本机 pinned arena，`local_owner_` 是 `LocalBatchResult`，析构即归还，无需远端配合。
- `LeaseExpired()` 用 `steady_clock` 计算已耗时，与回包里的 `gc_ttl_ms` 比较；这是**读者自己**判断"我是不是用得太久"，不是向 owner 续租。

**重点语法 / 惯用法**

- 四个特殊成员函数**全部 delete**：不可拷贝也不可移动，因此只能以 `std::unique_ptr<OffloadRestoreLease>` 形式返回（见 `restore_offload_objects` 的返回类型），所有权唯一、释放恰好一次。
- 析构函数**不抛异常**：`release_offload_buffer` 返回 `bool`，这里直接丢弃——释放失败不能影响调用方的控制流。
- 成员声明顺序决定析构顺序：`response_` 在 `local_owner_` 之后声明，析构时先销毁 `response_` 再销毁 `local_owner_`；释放用的是 `response_.batch_id`，但那是析构函数体内、成员销毁之前，所以安全。
- `std::move(endpoint)` / `std::move(response)` 之后形参不再使用，符合"构造即接管"的写法。
- `std::optional<LocalBatchResult>` 让"本机批次"与"远端批次"共用一个字段，避免多态或空指针。

**与 6.7.3 的差异**

- 图上第 24 步是一条 `R->>FS: release_offload_buffer（RAII）` 的箭头，实际**只有远端批次才发这条 RPC**；本机 pinned 批次在这里静默归还。
- 图上把释放画在读完成之后，代码里释放点是**租约对象析构**，可能晚于 `get` 返回（对象被调用方持有到作用域结束），也可能因异常提前触发。
- 图上没有画 "gc_ttl 过期" 的分支：过期时数据其实已经拷到用户 buffer，但调用方会拿到 `OBJECT_HAS_LEASE`（见 4.6.3），租约对象仍会正常析构并释放。
- `batch_id = 0` 用于本机批次（`restore_offload_objects` 里构造的响应），因此即使误发 RPC，也不会释放别人的 buffer。

#### 4.6.2 owner 端的第 22 步：组包回包

mooncake-store/src/real_client.cpp:8694-8722

```cpp
async_simple::coro::Lazy<tl::expected<BatchGetOffloadObjectResponse, ErrorCode>>
RealClient::batch_get_offload_object(const std::vector<std::string> &keys,
                                     const std::vector<int64_t> &sizes) {  // 该函数跑在 owner 进程，由 RPC 框架调用
    // Run SSD I/O on a dedicated thread pool so the coro_rpc IO thread is
    // free to handle ping and other RPCs. Same heap-owned state pattern as
    // batch_get_into_dummy_helper: the lambda captures only a raw pointer.
    struct CallState {  // 协程里的"堆上状态"：跨越 co_await 的变量不能放栈上
        std::vector<std::string> keys;  // 从 RPC 请求拷一份，保证 post 期间有效
        std::vector<int64_t> sizes;  // 同上
        std::shared_ptr<FileStorage> file_storage;  // 延长 FileStorage 生命周期，避免 post 执行时已被销毁
    };
    auto state = std::make_unique<CallState>();  // 堆分配，所有权暂时在本协程
    state->keys = keys;  // 拷贝而不是引用：形参在 co_await 期间可能失效
    state->sizes = sizes;
    state->file_storage = file_storage_;  // 拷贝 shared_ptr，引用计数 +1
    auto *s = state.get();  // 只把裸指针交给 lambda
    auto try_result = co_await coro_io::post(
        [s]() { return s->file_storage->BatchGet(s->keys, s->sizes); });  // 把阻塞的 SSD 读甩到专用线程池
    auto result = try_result.value();  // 外层 expected 表示 post 是否成功；这里取出 BatchGet 的内层结果
    if (!result) {  // 内层失败：存储层没读到
        LOG(ERROR) << "Batch get offload object failed,err_code = "
                   << result.error();
        co_return tl::make_unexpected(result.error());  // 原样上报错误码
    }
    co_return BatchGetOffloadObjectResponse(
        result.value().batch_id, std::move(result.value().pointers),  // batch_id 用于后续释放；pointers 是 owner 进程内的虚拟地址
        client_->GetSegmentEndpoint(),  // TE 段名：读方用它 openSegment，才能按地址 READ
        file_storage_->config_.client_buffer_gc_ttl_ms);  // 读者侧截止时间（毫秒），不是 owner 侧的锁
}
```

**这一段在做什么**

- 这是第 22 步的服务端：owner 收到 `(tenant:key[], restore_sizes[])` 后，调用本机 `FileStorage::BatchGet` 分配并填充 client buffer，再把 `{batch_id, pointers[], transfer_engine_addr, gc_ttl_ms}` 打包回给读者。
- `BatchGet` 是**阻塞的 SSD 读**，所以整段用 `coro_io::post` 挪到专用线程池，`coro_rpc` 的 IO 线程不被占用（注释已写明动机）。
- `transfer_engine_addr` 来自 `client_->GetSegmentEndpoint()`：P2PHANDSHAKE 模式返回实际 RPC 端点（IP:Port），其它模式返回注册用的 `local_hostname_`；它标识 **owner 的段**，读方拿到它才能 `openSegment` 并访问 `pointers[]` 指向的已注册内存。
- `gc_ttl_ms` 来自 `file_storage_->config_.client_buffer_gc_ttl_ms`，会同时写进 `AllocatedBatch::lease_timeout`（见 `file_storage.cpp:1105-1106`），是**同一个值的两种用法**。

**重点语法 / 惯用法**

- `async_simple::coro::Lazy<T>` 是惰性协程：函数体在 `co_await`/`co_return` 之前不会执行，RPC 框架负责启动。
- `CallState` + `unique_ptr` + 裸指针捕获：协程挂起会销毁栈帧，lambda 若按引用捕获会悬空；把状态放堆上、lambda 只拿裸指针，是这里唯一安全的写法。
- `state->file_storage = file_storage_` 拷贝 `shared_ptr`，把 `FileStorage` 的生命周期钉到 `state` 析构为止。
- `try_result.value()` 只在 post 失败时抛 `tl::bad_expected_access`；正常路径下 `try_result` 有值，内层才是真正的业务结果。**这是个坑**：外层错误没有被 `if` 处理。
- `std::move(result.value().pointers)` 把 vector 移出，避免回包序列化前再拷一份大数组；`result` 之后不再使用。

**与 6.7.3 的差异**

- 图上第 22 步写 `{batch_id, pointers[], transfer_engine_addr, gc_ttl_ms}`，但没有说 `batch_id` 与 `pointers` 都是**owner 进程视角**的东西：`pointers` 在读者进程里没有任何意义，只能当 TE 段内偏移用。
- `transfer_engine_addr` 不是"读方的 TE 地址"，而是 **owner 的段名**；图上第 23 步的 `R->>FS: one-sided RDMA READ` 正是以它为目标段。
- 图上第 18 步到第 22 步之间还有一层：若读方与 owner 同进程且 `allow_pinned`，这一步不会走 RPC，而是 `FileStorage::BatchGetLocal` 直接返回本机地址（见 4.2.2 的 `local_batch` 判定）。
- `gc_ttl_ms` 是**读者侧**的截止时间，owner 到点就回收，不等待读者确认；这是"乐观租约"，不是锁。

#### 4.6.3 读者端的第 22–23 步：取回包、判过期、发起 READ

mooncake-store/src/real_client.cpp:8984-9018

> 说明：这是 `batch_get_into_offload_object_internal` 的后半段；`restore_offload_objects` 的完整实现（含本机 pinned 分支与 `local_batch` 判定）见 4.2。

```cpp
    auto restored =
        restore_offload_objects(target_rpc_service_addr, keys, sizes, objects);  // 第 22 步：拿回包并包成租约对象
    if (!restored) {
        return tl::unexpected(restored.error());  // 取回包失败（RPC 错误 / BatchGet 错误 / 指针数不匹配）
    }
    auto &lease = **restored;  // 解引用 unique_ptr → OffloadRestoreLease&
    if (read_range) {  // 只读对象的一段（range get）时，调整目标地址
        if (lease.response().pointers[0] >
            std::numeric_limits<uint64_t>::max() - read_range->source_offset) {  // 防 uint64 上溢
            return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
        }
        lease.response().pointers[0] += read_range->source_offset;  // 把 owner buffer 的起始地址往后挪
    }
    auto result = client_->BatchGetOffloadObject(
        lease.response().transfer_engine_addr, keys, lease.response().pointers,  // 目标段 = owner 的 TE 地址；目标偏移 = pointers
        objects,
        lease.local_batch() ? OffloadBufferAccess::kLocalAddress
                            : OffloadBufferAccess::kTransferEngine);  // 本机批次可直接 memcpy，否则必须走 TE
    const auto elapsed_time = lease.ElapsedMs();  // 从拿到回包到 READ 完成的总耗时
    LOG(INFO) << "Time taken for batch_get_into_offload_object_internal: "
              << elapsed_time
              << "ms, with target_rpc_service_addr: " << target_rpc_service_addr
              << ", key size: " << objects.size()
              << ", batch_id: " << lease.response().batch_id
              << ", gc ttl: " << lease.response().gc_ttl_ms << "ms.";

    if (!result) {  // READ 本身失败（开段失败/传输失败/超时）
        LOG(ERROR) << "Batch get into offload object failed with error: "
                   << result.error();
        return result;  // 透传错误码
    }
    if (lease.LeaseExpired()) {  // READ 成功了，但已经超过 owner 的 gc 截止时间
        return tl::make_unexpected(ErrorCode::OBJECT_HAS_LEASE);  // 数据可能已被回收复用，视为失败
    }
    return {};
```

**这一段在做什么**

- `restore_offload_objects` 拿到回包后立刻构造 `OffloadRestoreLease`，之后的任何返回路径都会触发释放（RAII）。
- `read_range` 分支只改 `pointers[0]`：把 owner buffer 的起始地址加上 `source_offset`，后面按原来的 slice 长度读，就实现了"只读对象的一段"。
- `client_->BatchGetOffloadObject(...)` 是第 23 步的真正发起者：把 `(transfer_engine_addr, pointers, objects, buffer_access)` 交给 `TransferSubmitter`。
- `buffer_access` 二选一：`kLocalAddress` 表示 owner 与本进程同进程、`pointers` 就是本进程虚拟地址，可直接 `memcpy`；`kTransferEngine` 表示必须开段走 RDMA/`TCP` 的 READ。
- READ 完成后才检查 `LeaseExpired()`：即使传输成功，只要超时也返回 `OBJECT_HAS_LEASE`，因为 owner 的 GC 可能已经回收并复用这块 buffer。

**重点语法 / 惯用法**

- `**restored`：`restored` 是 `tl::expected<std::unique_ptr<Lease>, E>`，一次 `*` 取 unique_ptr，再一次 `*` 取对象引用。
- `std::numeric_limits<uint64_t>::max() - read_range->source_offset`：先减后比的**防溢出加法**模板，比 `a + b < a` 更清楚。
- `return result;` 直接把 `tl::expected<void,E>` 透传，类型一致时无需再包一层。
- `lease.response()` 非 const 重载在这里被用到（要改 `pointers[0]`），const 版本留给只读场景。
- 日志里同时打 `batch_id` 与 `gc_ttl`，是为了事后能把"超时"归因到具体批次，而不是只看到 `OBJECT_HAS_LEASE`。

**与 6.7.3 的差异**

- 图上第 23 步只有一条 READ 箭头，实际有**两条实现**：同进程 `memcpy`（`kLocalAddress`）与 TE READ（`kTransferEngine`），分支点在 `lease.local_batch()`。
- 图上 `GC ttl` 只出现在第 22 步的回包里；代码里它真正的用途在 READ **之后**的 `LeaseExpired()`，决定这次读算不算成功。
- `read_range` 的地址偏移是图上完全没有的：range get 复用的是同一份 owner buffer，只是把起点挪了 `source_offset`。
- 过期检查**不阻止**已经发生的 READ（数据已经拷进用户 buffer），只是把结果判为失败；用户 buffer 里的内容是否可信，取决于 owner 是否恰好复用了那块内存。

#### 4.6.4 READ 的请求构造：source / target 的真实语义

mooncake-store/src/transfer_task.cpp:1614-1694

```cpp
std::optional<TransferFuture>
TransferSubmitter::submit_batch_get_offload_object(
    const std::string& transfer_engine_addr,  // owner 的 TE 段名（来自第 22 步回包）
    const std::vector<std::string>& keys, const std::vector<uint64_t>& pointers,
    const std::unordered_map<std::string, std::vector<Slice>>& batched_slices,  // 每个 key 在本地的目标 slice 列表
    OffloadBufferAccess buffer_access, TransferIntent intent) {
    if (keys.size() != pointers.size()) {  // keys 与 owner buffer 地址必须一一对应
        LOG(ERROR) << "Mismatched offload transfer argument counts";
        return std::nullopt;  // 上层映射成 TRANSFER_FAIL
    }

    const bool use_local_memcpy =
        buffer_access == OffloadBufferAccess::kLocalAddress;  // 同进程 → 直接 memcpy，不开段
    if (use_local_memcpy && !canUseLocalMemcpy(transfer_engine_addr)) {  // 双重确认：endpoint 必须等于本进程段名
        LOG(ERROR) << "Offload source is not locally addressable: "
                   << transfer_engine_addr;  // 同机不同进程也不行（虚拟地址空间不同）
        return std::nullopt;  // 宁可失败，也不做会 segfault 的 memcpy
    }

    std::vector<TransferRequest> requests;  // 走 TE 的请求
    std::vector<MemcpyOperation> operations;  // 走本机拷贝的操作
    constexpr uint64_t kMaxAddress = std::numeric_limits<uint64_t>::max();  // 用于手工上溢检查
    SegmentHandle seg = 0;  // TE 段句柄，整批只开一次
    if (!use_local_memcpy) {
        // Open once: all keys share the same transfer endpoint.
#ifdef USE_TENT
        if (tent_engine_) {  // 编译期启用 TENT 且运行时有 TENT 引擎
            tent::SegmentID seg_id;
            auto status =
                tent_engine_->openSegment(seg_id, transfer_engine_addr);  // TENT 的段 id 类型与经典 TE 不同
            if (!status.ok()) {
                LOG(ERROR) << "Failed to open segment " << transfer_engine_addr;
                return std::nullopt;
            }
            seg = seg_id;
        } else
#endif
        {
            seg = engine_.openSegment(transfer_engine_addr);  // 经典 TE：按段名拿到 SegmentHandle
            if (seg == static_cast<uint64_t>(ERR_INVALID_ARGUMENT)) {  // 约定用这个值表示失败
                LOG(ERROR) << "Failed to open segment " << transfer_engine_addr;
                return std::nullopt;
            }
        }
    }

    for (size_t i = 0; i < keys.size(); ++i) {  // 每个 key 一段 owner buffer
        const auto& key = keys[i];
        auto it = batched_slices.find(key);  // 本地目标 slice 列表
        if (it == batched_slices.end()) {
            LOG(ERROR) << "Key not found in batched_slices: " << key;
            return std::nullopt;
        }
        uint64_t offset = 0;  // 在 owner buffer 内的偏移，也是本地多 slice 的累加偏移
        for (const auto& slice : it->second) {  // 一个 key 可能被拆成多个本地 slice
            if (slice.size == 0) continue;  // 空 slice 不产生请求
            if (!slice.ptr || pointers[i] > kMaxAddress - offset ||
                slice.size > kMaxAddress - pointers[i] - offset) {  // 两段上溢检查
                LOG(ERROR) << "Invalid offload transfer range for key: " << key;
                return std::nullopt;
            }
            if (use_local_memcpy) {
                operations.emplace_back(
                    slice.ptr,  // 目的一：本地目标地址
                    reinterpret_cast<const void*>(pointers[i] + offset),  // 源：owner buffer 里的地址（同进程有效）
                    slice.size);  // 拷贝长度
            } else {
                requests.emplace_back(TransferRequest{
                    .opcode = TransferRequest::READ,  // READ = 从 target 拉到 source
                    .source = static_cast<char*>(slice.ptr),  // 本地目标地址
                    .target_id = seg,  // owner 的段
                    .target_offset = pointers[i] + offset,  // owner buffer 内的偏移
                    .length = slice.size,  // 本 slice 长度
                });
            }
            offset += slice.size;  // 下一个 slice 接着 owner buffer 往后读
        }
    }
    return use_local_memcpy ? submitMemcpyOperations(std::move(operations))
                            : submitTransfer(requests, intent);  // 两条路都返回 TransferFuture
}
```

**这一段在做什么**

- 把"每个 key 的每个本地 slice"翻译成一条传输原语：同进程是 `MemcpyOperation`，跨进程是 `TransferRequest{READ}`。
- **READ 的字段语义与直觉相反**：`source` 是**本地**目标缓冲，`target_id/target_offset` 才是**远端**源（owner 的段与 buffer 偏移）——即"READ from target into source"。
- `openSegment` 整批只做一次：所有 key 的 buffer 都在 owner 同一个 TE 段里，段句柄可以复用。
- `pointers[i] + offset` 是 owner buffer 内的地址；在 TE 语义里它作为 `target_offset` 使用，因为 owner buffer 已注册进它的段，虚拟地址即段内偏移。
- 同进程分支必须先过 `canUseLocalMemcpy`：它要求 `memcpy_enabled_` 且 endpoint **完全等于**本进程段名（`isSameProcessEndpoint`），同机不同进程会被拒绝。

**重点语法 / 惯用法**

- `std::optional<TransferFuture>` 表示"提交成功但结果为异步"：`nullopt` 是**提交阶段**失败，传输失败在 `future->get()` 里返回。
- 指定初始化器 `.opcode = ...` 只写关心的字段，其余字段用结构体默认值（`advise_retry_cnt`、`task_group_id`、`priority` 等）。
- `#ifdef USE_TENT` 内嵌 `else`，让 TENT 与经典 TE 共享同一段控制流；`seg` 用 `SegmentHandle` 统一承载两种 id。
- 上溢检查分两步：先确认 `pointers[i] + offset` 不溢出，再确认加上 `slice.size` 不溢出，避免中间量回绕。
- `slice.ptr` 为空直接判非法；空 slice 则**跳过**而不是报错——两种"空"语义不同。

**与 6.7.3 的差异**

- 图上第 23 步写"one-sided RDMA READ（同进程 memcpy）"，把两种实现塞进一个括号；代码里是**两个完全不同的 API**（`submitMemcpyOperations` vs `submitTransfer`），且同进程分支连段都不开。
- 图上 `pointers[]` 看起来像"本地地址"；在 TE 分支里它被当作 owner 段内的 `target_offset`，只有 memcpy 分支才是真地址。
- 一个 key 对应**多条** READ：本地 slice 列表有 N 个 slice 就产生 N 条请求，各自 `target_offset` 递增。
- 同进程判定是 `endpoint == 本进程段名`，不是"同一台机器"——注释里专门解释了为什么不能用 IP/主机名判断（会 segfault）。

#### 4.6.5 第 24 步的落点：`ReleaseBuffer` 的幂等性

mooncake-store/src/file_storage.cpp:1202-1214

```cpp
bool FileStorage::ReleaseBuffer(uint64_t batch_id) {  // owner 侧：读者说"用完了，可以回收"
    MutexLocker locker(&client_buffer_mutex_);  // 与 AllocateBatch / GC 线程共用一把锁
    auto it = client_buffer_allocated_batches_.find(batch_id);  // batch_id → AllocatedBatch（含 BufferHandle 与 lease_timeout）
    if (it != client_buffer_allocated_batches_.end()) {  // 正常路径：批次还在
        VLOG(1) << "Releasing buffer for batch_id: " << batch_id
                << " (transfer completed)";
        client_buffer_allocated_batches_.erase(it);  // erase 即析构 → BufferHandle 归还 allocator（RAII）
        return true;  // 告诉调用方"确实释放了"
    }
    VLOG(1) << "batch_id " << batch_id
            << " not found (may have been GC'd already)";  // 已被 GC 线程按 gc_ttl 回收
    return false;  // 幂等：重复释放或过期释放都不算错误
}
```

**这一段在做什么**

- 这是第 24 步在 owner 侧的真正动作：从 `client_buffer_allocated_batches_` 里删掉 `batch_id` 对应的 `AllocatedBatch`，`erase` 触发析构，`BufferHandle` 把内存还给 `ClientBufferAllocator`。
- 删除是**幂等**的：找不到就返回 `false` 并打日志，不报错——因为 GC 线程可能已经按 `lease_timeout` 回收过。
- `lease_timeout` 在 `AllocateBatch` 时设为 `now + client_buffer_gc_ttl_ms`（`file_storage.cpp:1105-1106`），GC 线程周期性扫描并 `erase` 过期批次（`ClientBufferGCThreadFunc`，`file_storage.cpp:1169-1190`）。
- 因此"RAII 释放"与"GC 兜底"是**两条独立路径**，谁先到都行；只要读者在 `gc_ttl` 内完成 READ，数据就还在。

**重点语法 / 惯用法**

- `MutexLocker locker(&client_buffer_mutex_)` 是项目内的 RAII 锁别名，作用域结束自动解锁。
- `erase(it)` 返回下一个迭代器（此处未用）；`AllocatedBatch` 的析构由 `unordered_map::erase` 自动完成，不需要手动 free。
- 返回 `bool` 而不是 `ErrorCode`：调用方（`RealClient::release_offload_buffer`）也只把它当"是否找到"用。
- 日志用 `VLOG(1)` 而非 `LOG(INFO)`：释放是高频事件，避免刷爆日志。
- "找不到"不是异常而是**预期状态**，注释直接写出"may have been GC'd already"，把契约写进代码。

**与 6.7.3 的差异**

- 图上第 24 步是读者 → owner 的单向箭头，看起来是"通知"；代码里它是**真正的释放动作**，owner 删掉记录后这块内存立刻可被下一次 `AllocateBatch` 复用。
- 图上没有画 GC：若读者始终不释放（进程崩溃、租约对象泄漏），`gc_ttl` 到期后 GC 线程会代为回收，这也是 `OBJECT_HAS_LEASE` 存在的原因。
- 释放 RPC 失败（返回 `false`）不影响正确性，只影响内存回收时延；下一次分配可能触发同步 GC（`AllocateBatch` 里 `gc_triggered` 分支）。
- 第 24 步可能发生在第 23 步完成很久之后（租约对象存活到作用域结束），图上箭头紧挨着 READ 会让人误以为释放是同步收尾。

## 5. 命令层：16 字节 key 如何变成一条 NVMe 命令

### 5.1 物理 key 打包与命令构造

后端拿到的是 `PhysicalKey = std::array<uint8_t, 16>`，而 NVMe KV 命令没有"key 缓冲区"这种字段——16 字节必须被塞进命令的 4 个 CDW（Command DWord）。这一节只讲"怎么塞"和"塞进去之后哪些位被占用"。

#### 5.1.1 三个构造器共用同一套编码

mooncake-store/include/nvme_kv/executor_util.h:66-122

```cpp
inline uint32_t BuildNvmeKvKeyLengthField(size_t key_length) {  // key 长度字段只有 1 字节
    return static_cast<uint32_t>(key_length) & 0xFFu;  // 超过 255 会被静默截断；物理 key 恒为 16，所以这里安全
}

template <typename Command>  // 同一套编码要喂给两种命令结构：nvme_passthru_cmd（ioctl）与 nvme_uring_cmd（uring）
void EncodeNvmeKvKey(const NvmeKvCommandExecutor::PhysicalKey &key,
                     Command &cmd) {  // 把 16 字节 key 摊到 cdw2/3/14/15
    const auto fields = PackNvmeKvPhysicalKey(key);  // 真正做字节切分的地方（见 5.1.2）
    cmd.cdw2 = fields.cdw2;  // key 字节 0..3
    cmd.cdw3 = fields.cdw3;  // key 字节 4..7
    cmd.cdw14 = fields.cdw14;  // key 字节 8..10 + CSI（bit 31:24）
    cmd.cdw15 = fields.cdw15;  // key 字节 12..15
}

template <typename Command>
void BuildNvmeKvStoreCommand(Command &cmd, uint32_t nsid,
                             const NvmeKvCommandExecutor::PhysicalKey &key,
                             const void *data, uint32_t transfer_bytes) {  // Store = 把 host 内存写进设备
    cmd = {};  // 关键：值初始化，把所有没显式赋值的 CDW 清零，避免脏位
    cmd.opcode = kNvmeKvStoreOpcode;  // 0x01，KV 命令集的 Store
    cmd.nsid = nsid;  // 目标 namespace
    cmd.addr = reinterpret_cast<uint64_t>(data);  // host 数据地址，必须按传输粒度对齐
    cmd.data_len = transfer_bytes;  // host 侧数据长度（已按 4096 round-up 后的值）
    cmd.cdw10 = transfer_bytes;  // CDW10 同样放传输长度，两个字段语义不同但这里同值
    cmd.cdw11 = (kNvmeKvStoreIfNotExistsOption << 8) |
                BuildNvmeKvKeyLengthField(key.size());  // [15:8] 放 Store 选项（2 = Store If Not Exists），[7:0] 放 key 长度
    cmd.cdw12 = ComputeNvmeKvValueBlockCountMinusOne(transfer_bytes);  // 按 512B 块计数并减一（见 5.1.2）
    cmd.timeout_ms = kNvmeKvCommandTimeoutMs;  // 30000ms，命令级超时
    EncodeNvmeKvKey(key, cmd);  // 最后再填 key，覆写 cdw2/3/14/15
}

template <typename Command>
void BuildNvmeKvRetrieveCommand(Command &cmd, uint32_t nsid,
                                const NvmeKvCommandExecutor::PhysicalKey &key,
                                void *data, uint32_t transfer_bytes) {  // Retrieve = 把设备数据读回 host
    cmd = {};  // 同样清零
    cmd.opcode = kNvmeKvRetrieveOpcode;  // 0x02
    cmd.nsid = nsid;
    cmd.addr = reinterpret_cast<uint64_t>(data);  // 目标地址（直读时就是用户 buffer）
    cmd.data_len = transfer_bytes;
    cmd.cdw10 = transfer_bytes;
    cmd.cdw11 = BuildNvmeKvKeyLengthField(key.size());  // 读没有 Store 选项，[15:8] 保持 0
    cmd.cdw12 = ComputeNvmeKvValueBlockCountMinusOne(transfer_bytes);  // 期望读回的块数
    cmd.timeout_ms = kNvmeKvCommandTimeoutMs;
    EncodeNvmeKvKey(key, cmd);
}

template <typename Command>
void BuildNvmeKvDeleteCommand(Command &cmd, uint32_t nsid,
                              const NvmeKvCommandExecutor::PhysicalKey &key) {  // Delete 只带 key，没有数据面
    cmd = {};
    cmd.opcode = kNvmeKvDeleteOpcode;  // 0x10
    cmd.nsid = nsid;
    cmd.cdw11 = BuildNvmeKvKeyLengthField(key.size());  // 只有 key 长度
    cmd.timeout_ms = kNvmeKvCommandTimeoutMs;
    EncodeNvmeKvKey(key, cmd);
}
```

**这一段在做什么**

- 三个构造器把"命令集语义"集中在一处：opcode（`0x01`/`0x02`/`0x10`）、`nsid`、`addr`/`data_len`、`cdw10`/`cdw11`/`cdw12`、`timeout_ms`，最后统一调 `EncodeNvmeKvKey`。
- 16 字节 key 被拆进 `cdw2`、`cdw3`、`cdw14`、`cdw15` 四个 DWord，每个 DWord 承载 4 字节。
- `cdw11` 的低 8 位是 **key 长度**（`key.size()` = 16），高 8 位在 Store 里放选项；`kNvmeKvStoreIfNotExistsOption = 0x2` 左移 8 位后落在 **bit 9**（`0x200`），语义是"key 已存在则返回 `KEY_EXISTS` 而不是覆盖"。
- `cdw12` 是"值占多少个 512B 块，减一"，0 表示 1 块；`cdw10` 与 `data_len` 都是传输字节数。
- `cmd = {}` 是这段代码里最容易忽略的一行：它保证未使用的 CDW（如 `cdw13`）为 0，否则复用栈上结构体会带进脏位。

**重点语法 / 惯用法**

- `template <typename Command>` 让 `nvme_passthru_cmd` 与 `nvme_uring_cmd` 共享构造逻辑；两者前若干字段同名同义，模板只要求"有这些成员"。
- `cmd = {}` 对聚合/平凡结构体是值初始化（全零），比 `memset` 类型安全。
- `& 0xFFu` 是显式截断：把 `size_t` 收窄到 1 字节字段，编译器不会报 `-Wconversion`。
- `(option << 8) | length` 用位或拼两个子字段，依赖两者位域不重叠（`0x2 << 8 = 0x200`，`length <= 0xFF`）。
- `timeout_ms` 是宿主侧字段，不属于 NVMe 命令本身；uring/ioctl 两套执行器各自把它翻译成超时机制。

**与 6.7.3 的差异**

- 图上第 13/20 步的 `StoreBatch`/`Retrieve` 只画到"命令"这一层，看不出 key 是被**拆成 4 个 CDW** 的；也没有"CSI 占用一个字节"这一信息。
- `cdw11` 的选项字段位于 `[15:8]`，值 `2` 实际是 **bit 9**，不是 bit 8；按"第 8 位"理解会写错命令。
- Store 的 `transfer_bytes` 是**已对齐**的长度（`ResolveNvmeKvStoreSubmissionBytes` 先 round-up），所以 `cdw12` 反映的是对齐后的块数，不是逻辑 payload 长度。
- Delete 命令没有 `addr`/`data_len`/`cdw10`/`cdw12`，图上没有 Delete 分支，但后端清理 chunk 时会用到它。

#### 5.1.2 字节切分与块计数

mooncake-store/src/nvme_kv/executor_util.cpp:131-151

```cpp
uint32_t ComputeNvmeKvValueBlockCountMinusOne(uint32_t bytes) {  // 把字节数换算成"512B 块数 - 1"
    if (bytes == 0) {  // 零长度没有块
        return 0;  // 返回 0，语义上是"1 块"，调用方不会用 0 长度发命令
    }
    const uint32_t rounded_bytes = RoundUpToNvmeKvTransferBytes(bytes);  // 先按传输对齐（默认 4096）向上取整
    const uint32_t block_unit = NvmeKvValueBlockUnitBytes();  // 值块单位，默认 512（kDefaultNvmeKvValueBlockUnitBytes）
    const uint64_t block_count =
        (static_cast<uint64_t>(rounded_bytes) + block_unit - 1) / block_unit;  // 再按 512 向上取整，用 uint64 防溢出
    return block_count == 0 ? 0 : block_count - 1;  // 字段是"减一"编码，0 表示 1 块
}

NvmeKvPackedKeyFields PackNvmeKvPhysicalKey(
    const NvmeKvCommandExecutor::PhysicalKey &key) {  // 16 字节 → 4 个 CDW
    NvmeKvPackedKeyFields fields;
    fields.cdw2 = ReadLe32(key.data());  // 字节 0..3，按小端解释
    fields.cdw3 = ReadLe32(key.data() + 4);  // 字节 4..7
    fields.cdw14 = (static_cast<uint32_t>(kNvmeKvCommandSetIdentifier) << 24) |
                   (ReadLe32(key.data() + 8) & 0x00FFFFFFu);  // 高 8 位放 CSI=0x01，只剩字节 8..10 有效
    fields.cdw15 = ReadLe32(key.data() + 12);  // 字节 12..15
    return fields;
}
```

**这一段在做什么**

- 块计数是**两级取整**：先把字节数按传输对齐（4096）round-up，再除以值块单位（512），最后 `-1` 编码。
- `PackNvmeKvPhysicalKey` 是"16 字节 → 4 个 CDW"的唯一实现：`cdw2` 取字节 0..3、`cdw3` 取 4..7、`cdw14` 取 8..10、`cdw15` 取 12..15。
- `cdw14` 的高 8 位被 **CSI（Command Set Identifier = 0x01）** 占用，因此 key 的第 11 个字节（下标 11）被 CSI 覆盖——**有效 key 只有 15 字节**。
- 所有读取都走 `ReadLe32`，显式按小端拼接，不依赖主机字节序（x86 上是恒等，但语义明确）。

**重点语法 / 惯用法**

- `(a + b - 1) / b` 两次向上取整，中间用 `uint64_t` 承载，避免 `rounded_bytes + block_unit - 1` 在 `uint32_t` 上回绕。
- `& 0x00FFFFFFu` 只保留低 24 位，是"遮掉 CSI 位"的写法；CSI 用 `<< 24` 占住高 8 位。
- 返回结构体 `NvmeKvPackedKeyFields` 而不是写 `std::array<uint32_t,4>`，让 `cdw2/cdw3/cdw14/cdw15` 的字段名自解释。
- `ReadLe32` 是文件内匿名命名空间里的手写小端读取，不用 `reinterpret_cast<uint32_t*>`，规避未对齐访问与严格别名问题。
- `block_count == 0 ? 0 : block_count - 1` 是防御式的"减一"，对 `bytes == 0` 提前返回后其实不会走到。

**与 6.7.3 的差异**

- 物理 key 是 **XXH64 派生出的 16 字节哈希**（`DerivePhysicalKey`，`key_codec.cpp:42-49`），不是可读字符串；因此"第 11 字节被 CSI 覆盖"不会造成语义损失，但 key 空间实际是 120 位而非 128 位。
- 块计数用的是**对齐后**的长度：逻辑 payload 只有 1 字节，`cdw12` 也会是 `4096/512-1 = 7`（8 个块），即最小传输单位是 4096B。
- 图上第 13 步的 `StoreBatch` 看起来"一次写一个对象"，实际每个 chunk 都是一条独立命令，块计数按各自 chunk 的 `payload_size` 分别计算。
- 没有 CRC/端到端校验字段：这里的 `payload_checksum` 是文件格式层的软件校验和（4.5 节），不进入 NVMe 命令。

### 5.2 io_uring 的提交-收割

`executor_io_uring.cpp` 里真正的 IO 引擎是文件内的 `SharedNvmeUringRing`（`thread_local`，每个线程一个 ring）。`SubmitBatch` 是它的核心：**窗口化提交 + 收割时按 token 认领 CQE**。下面先把 97–278 行按"准备/收割/等待/主循环"拆开读，最后补上 ring 重建与 SQE 构造这两个辅助函数（311–360 行）。

#### 5.2.1 一次提交的状态与即时收割

mooncake-store/src/nvme_kv/executor_io_uring.cpp:97-185

```cpp
    tl::expected<void, ErrorCode> SubmitBatch(
        int fd, std::vector<BatchCommand>& commands) {  // 一批命令，commands[i] 自带 cmd/上下文/结果槽
        if (commands.empty()) {
            return {};  // 空批次直接成功，避免动 ring
        }
        if (!initialized_) {  // ring 初始化失败（设备不支持 SQE128 等）
            return tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
        }
        if (commands.size() >= UINT32_MAX) {  // user_data 低 32 位要放 index+1，超了会回绕
            return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
        }

        const uint32_t completion_token = NextCompletionToken();  // 本批次唯一 token，用来识别"上一批遗留的 CQE"
        size_t next_command = 0;  // 下一条要准备的命令下标
        size_t completed_count = 0;  // 已完成条数，主循环的退出条件
        size_t inflight_ops = 0;  // 已提交未收割的条数，窗口控制的核心变量
        ErrorCode batch_error = ErrorCode::OK;  // 批级错误（stale CQE 等），最后统一上抛

        const auto drain_ready = [&]() -> tl::expected<size_t, ErrorCode> {  // 非阻塞收割：把 CQ 里现成的都吃掉
            size_t drained = 0;  // 本次消费了几个 CQE
            while (inflight_ops != 0) {  // 没有在途命令就不用看 CQ
                io_uring_cqe* cqe = nullptr;
                const int peek_ret = io_uring_peek_cqe(&ring_, &cqe);  // 只 peek 不阻塞
                if (peek_ret == -EAGAIN) {  // CQ 空了
                    break;  // 交给主循环继续提交或 wait
                }
                if (peek_ret < 0 || cqe == nullptr) {  // ring 本身出错
                    return tl::make_unexpected(
                        MapNvmeKvTransportError(peek_ret < 0 ? -peek_ret : EIO,
                                                commands.front().is_write));  // 用首条命令的读写属性挑错误码
                }
                const uint64_t user_data = cqe->user_data;  // 提交时写进去的 token|index
                const int res = cqe->res;  // CQE 的 res：>0 状态码、<0 -errno、0 成功
                const uint64_t cqe_extra_result =
                    cqe32_enabled_ ? cqe->big_cqe[0] : static_cast<uint64_t>(0);  // CQE32 才有 big_cqe，装 nvme 命令的 result
                io_uring_cqe_seen(&ring_, cqe);  // 先拷贝完字段再归还 CQE，之后 cqe 指针不可再用

                const auto complete_unknown_inflight = [&]() -> bool {  // stale CQE 时"顶掉"一条在途命令，保持计数一致
                    for (auto& command : commands) {  // 找一条已提交未完成的
                        if (!command.submitted || command.completed) {
                            continue;
                        }
                        command.mapped_error = ErrorCode::INTERNAL_ERROR;  // 无法确认它的真实结果，只能判失败
                        command.completed = true;
                        --inflight_ops;  // 保证窗口计数不会永远卡住
                        ++completed_count;
                        ++drained;
                        return true;  // 顶掉一条就够，CQE 与命令是一对一
                    }
                    return false;  // 没有任何在途命令，说明账目已经乱了
                };

                size_t command_index = 0;
                if (!DecodeUserData(user_data, completion_token,
                                    command_index) ||  // token 不匹配 = 上一批/上一次 ResetRing 之前的 CQE
                    command_index >= commands.size() ||  // 越界 = 内存被复用或 token 碰撞
                    !commands[command_index].submitted ||  // 没提交过却有完成
                    commands[command_index].completed) {  // 重复完成
                    LOG(ERROR) << "[NvmeKvIoUringExecutor] invalid or stale "
                                  "io_uring completion token";
                    batch_error = ErrorCode::INTERNAL_ERROR;  // 记下批级错误，先把在途的收干净再返回
                    if (!complete_unknown_inflight()) {  // 账目已经对不上，无法继续
                        return tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
                    }
                    continue;  // 丢弃这个 CQE，继续收割
                }
                auto& command = commands[command_index];  // 认领到具体命令
                if (res >= 0 && cqe32_enabled_ &&
                    cqe_extra_result <= UINT32_MAX) {  // CQE32 里 32 位 result 才可信
                    command.observed_result =
                        static_cast<uint32_t>(cqe_extra_result);  // 设备报告的实际结果（如读到的字节数）
                    command.has_observed_result = true;  // 标记"这个值有效"
                }
                if (res > 0) {  // 正数 = NVMe 状态码
                    command.mapped_error = MapNvmeKvStatus(
                        static_cast<uint32_t>(res), command.is_write);  // 0x87→OBJECT_NOT_FOUND、0x89→OBJECT_ALREADY_EXISTS 等
                } else if (res < 0) {  // 负数 = -errno
                    command.mapped_error =
                        MapNvmeKvTransportError(-res, command.is_write);  // 传输层错误（EIO/ENOMEM/...）
                } else {  // 0 = 成功
                    command.mapped_error = ErrorCode::OK;
                }
                command.completed = true;  // 标记完成，防止重复认领
                --inflight_ops;  // 窗口让出一个位置
                ++completed_count;  // 推进总进度
                ++drained;  // 本轮回填计数
            }
            return drained;  // 0 表示"这一轮没收到任何完成"
        };
```

**这一段在做什么**

- `SubmitBatch` 是同步 API：函数返回时，`commands[i].mapped_error` 必须全部就绪。因此它内部既提交又收割，直到 `completed_count == commands.size()`。
- `drain_ready` 用 `io_uring_peek_cqe` **非阻塞**地取走所有已就绪 CQE；`-EAGAIN` 表示 CQ 空，直接 `break` 回去继续提交。
- `res` 的三态映射是这套代码的核心约定：`>0` 是 NVMe 状态码（走 `MapNvmeKvStatus`）、`<0` 是 `-errno`（走 `MapNvmeKvTransportError`）、`==0` 成功。
- `cqe32_enabled_` 时从 `cqe->big_cqe[0]` 取 32 位 result（NVMe 命令的 `result` 字段，如实际传输字节数），`res` 只承载状态码。
- 遇到无法认领的 CQE（token 不匹配、重复完成、越界），调用 `complete_unknown_inflight()` 主动判一条在途命令失败：若这个 CQE 本应属于某条在途命令（完成丢失/串批），不顶掉一条就会少一个 `completed_count`，主循环永远等不到 `commands.size()`。

**重点语法 / 惯用法**

- `io_uring_cqe_seen` 必须在读完 `user_data`/`res`/`big_cqe` 之后调用：`seen` 会推进 CQ head，之后该 `cqe` 可能被内核复用。
- lambda 捕获 `[&]` 直接改外层的 `inflight_ops` / `completed_count` / `drained`，`drain_ready` 与 `complete_unknown_inflight` 是嵌套 lambda，共享同一组计数器。
- `tl::make_unexpected(...)` 在返回类型为 `tl::expected<size_t, ErrorCode>` 的 lambda 里构造错误值，调用方用 `!result` 判断。
- `static_cast<uint64_t>(0)` 而不是裸 `0`：三目两分支类型要一致，否则 `int` 与 `uint64_t` 的公共类型推导会让人意外。
- 用 `size_t` 做下标、`uint32_t` 做 token，靠 `DecodeUserData` 的显式截断衔接，避免有符号/无符号比较。

**与 6.7.3 的差异**

- 图上第 13/20 步只有"`StoreBatch`/`Retrieve` 一次批量"，看不出**收割是按 CQE 逐条、顺序不确定**的：`command_index` 由 `user_data` 决定，完成顺序与提交顺序无关。
- 图上没有"stale CQE"这个概念；代码里专门用 per-batch token + `submitted/completed` 双标志来防上一批的迟到 CQE 串到这一批。
- `observed_result` 是图上没有的输出：直读路径靠它和 `payload_size` 比对（4.5.4），少了这一步就无法发现"设备少写"。
- `big_cqe` 的存在意味着 ring 初始化用了 `IORING_SETUP_CQE32`；若设备/内核不支持，退化为只有 `res`，此时 `has_observed_result` 恒为 false。

#### 5.2.2 等待完成与窗口化提交

mooncake-store/src/nvme_kv/executor_io_uring.cpp:187-278

```cpp
        const auto wait_and_drain =
            [&](unsigned wait_min) -> tl::expected<size_t, ErrorCode> {  // 阻塞到至少 wait_min 个完成，再顺手收割
            io_uring_cqe* first_cqe = nullptr;
            int wait_ret = 0;
            do {
                wait_ret = io_uring_wait_cqe_nr(&ring_, &first_cqe, wait_min);  // wait_min=1：至少等 1 个
            } while (wait_ret == -EINTR);  // 被信号打断就重试，不能把 EINTR 当错误
            if (wait_ret < 0 || first_cqe == nullptr) {
                return tl::make_unexpected(MapNvmeKvTransportError(
                    wait_ret < 0 ? -wait_ret : EIO, commands.front().is_write));
            }
            // wait_cqe_nr leaves first_cqe in the CQ; drain_ready consumes it
            // together with any other completions that are already available.
            return drain_ready();  // 关键：first_cqe 没有被 seen，交给 drain_ready 统一消费
        };

        while (completed_count < commands.size()) {  // 直到整批完成
            const size_t prepared_start = next_command;  // 本窗口第一条新命令的下标
            size_t prepared_ops = 0;  // 本窗口准备了多少 SQE
            while (next_command < commands.size() &&
                   inflight_ops + prepared_ops < queue_depth_) {  // 窗口上限 = queue_depth
                auto& command = commands[next_command];
                io_uring_sqe* sqe = io_uring_get_sqe(&ring_);  // 从 SQ 取一个空位
                if (sqe == nullptr) {  // SQ 满了（内核队列深度限制）
                    break;
                }
                PrepareSqe(sqe, fd, command.cmd,
                           EncodeUserData(completion_token, next_command));  // SQE 里写入 token|index
                ++next_command;
                ++prepared_ops;
            }
            if (prepared_ops == 0 && inflight_ops == 0) {  // 既准备不了新的、又没有在途的 → 无法推进
                return tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
            }

            ErrorCode submit_error = ErrorCode::OK;
            size_t submitted_prepared = 0;  // 本窗口已成功提交的 SQE 数
            while (submitted_prepared < prepared_ops) {  // 处理"一次 submit 没吃完全部 SQE"的情况
                int submit_ret = 0;
                do {
                    submit_ret = submitted_prepared == 0
                                     ? io_uring_submit_and_wait(&ring_, 1)  // 首次提交顺带等 1 个完成，避免空转
                                     : io_uring_submit(&ring_);  // 后续只提交
                } while (submit_ret == -EINTR);  // 同样重试 EINTR
                const size_t remaining = prepared_ops - submitted_prepared;  // 还剩几条没提交
                if (submit_ret <= 0 ||
                    static_cast<size_t>(submit_ret) > remaining) {  // 0/负数或返回值超预期都算错
                    submit_error = MapNvmeKvTransportError(
                        submit_ret < 0 ? -submit_ret : EIO,
                        commands.front().is_write);
                    break;  // 跳出提交循环，进入清理
                }
                const size_t accepted = static_cast<size_t>(submit_ret);  // 内核真正接受的条数
                for (size_t index = 0; index < accepted; ++index) {  // 只把被接受的标成 submitted
                    auto& command =
                        commands[prepared_start + submitted_prepared + index];
                    command.submitted = true;  // 与 user_data 的 index 对应
                    ++inflight_ops;  // 窗口占用增加
                }
                submitted_prepared += accepted;  // 继续提交剩下的
            }

            if (submit_error != ErrorCode::OK) {  // 提交失败
                while (inflight_ops != 0) {  // 必须把在途的全部收完，否则 ring 状态不可预测
                    auto drain_result = wait_and_drain(1);
                    if (!drain_result || drain_result.value() == 0) {  // 等不到完成 = ring 已坏
                        const ErrorCode drain_error =
                            drain_result ? ErrorCode::INTERNAL_ERROR
                                         : drain_result.error();
                        ResetRing();  // 重建 ring
                        return tl::make_unexpected(drain_error);
                    }
                }
                ResetRing();  // 清空残留的 SQ/CQ
                return tl::make_unexpected(submit_error);  // 上报原始提交错误
            }

            auto drain_result = wait_and_drain(1);  // 正常路径：至少等 1 个完成并收割全部就绪的
            if (!drain_result || drain_result.value() == 0) {  // 等到却一条都没收割到
                const ErrorCode drain_error = drain_result
                                                  ? ErrorCode::INTERNAL_ERROR
                                                  : drain_result.error();
                ResetRing();
                return tl::make_unexpected(drain_error);
            }
        }
        if (batch_error != ErrorCode::OK) {  // 过程中出现过 stale CQE
            ResetRing();  // ring 状态已不可信，重建
            return tl::make_unexpected(batch_error);  // 整批判失败（即使其它命令成功了）
        }
        return {};
    }
```

**这一段在做什么**

- 主循环是"**准备一个窗口 → 提交 → 等一个完成 → 收割所有就绪**"的重复，窗口大小同时受 `queue_depth_` 和 SQ 空位限制。
- 首批用 `io_uring_submit_and_wait(&ring_, 1)`：提交后阻塞到至少 1 个完成，保证下一轮 `drain_ready` 有东西可收，不会忙等。
- `io_uring_submit` 可能只接受部分 SQE，所以内层 `while` 持续提交剩余部分，并且只把**被接受**的命令标 `submitted`、计入 `inflight_ops`。
- 任何一处失败都先 `wait_and_drain(1)` 把在途命令收干净，再 `ResetRing()`；这是"ring 级故障必须整体重建"的写法。
- 结束时若 `batch_error != OK`（出现过 stale/非法 CQE），即使所有命令都完成了也返回错误——因为无法证明每条命令的结果归属正确。

**重点语法 / 惯用法**

- `io_uring_wait_cqe_nr(&ring_, &first_cqe, wait_min)` 返回时 `first_cqe` **仍在 CQ 中**，必须由 `io_uring_cqe_seen` 消费；这里刻意不 seen，交给 `drain_ready` 统一处理（注释已强调）。
- `EINTR` 用 `do { ... } while (ret == -EINTR)` 重试，是系统调用封装的标准姿势；`EAGAIN` 在 `peek` 里表示"暂时没有"，两者语义不同。
- `submit_ret <= 0 || static_cast<size_t>(submit_ret) > remaining` 同时防"没提交"与"内核返回值异常"，后者是防御式断言。
- 内层失败时用 `break` 跳出而不是直接 `return`，是为了先走统一的 `inflight_ops` 清理逻辑。
- `ResetRing()` 被放在所有错误路径上，成功路径不调用；ring 是 `thread_local`，重建后下一批从干净状态开始。

**与 6.7.3 的差异**

- 图上把一次 `StoreBatch` 画成"后端 → 设备"一条线，实际是**多个窗口**：队列深度为 256 时，超过 256 条的命令要分多轮提交与收割。
- 图上没有超时概念；代码里单条命令的 `timeout_ms = 30000`（5.1.1）由命令结构体携带，`io_uring` 自身在这里没有设置 `IORING_OP_LINK_TIMEOUT`。
- "提交失败要先把在途的收完"是图上完全看不到的收尾逻辑，也是这套代码里最容易写漏、写漏就会让 ring 永久卡死的部分。
- `batch_error` 让**整批**失败，即使只有一条命令的 CQE 认领失败；这与 4.5 的 all-or-nothing 是同一设计取向。

#### 5.2.3 Ring 重建、SQE 构造与 token 编解码

mooncake-store/src/nvme_kv/executor_io_uring.cpp:311-360

```cpp
    void ResetRing() {  // 把 ring 整体推倒重建
        if (initialized_) {
            io_uring_queue_exit(&ring_);  // 释放 SQ/CQ 映射与内核资源，未收割的 CQE 一并丢弃
        }
        ring_ = {};  // 句柄清零，避免 queue_exit 被重复调用
        initialized_ = false;
        cqe32_enabled_ = false;
        InitializeRing();  // 重新 mmap、重新探测 SQE128/CQE32
    }
    // ...（略：320-322，QueueDepth() 辅助函数，与提交-收割无关）
    static void PrepareSqe(io_uring_sqe* sqe, int fd,
                           const struct nvme_uring_cmd& cmd,
                           uint64_t user_data) {  // 把一条 NVMe 命令塞进 SQE
        static_assert(sizeof(cmd) <= 80,
                      "nvme_uring_cmd must fit in SQE128 cmd area");  // 编译期守住 SQE128 的 80 字节 cmd 区
        constexpr size_t kSqe128Bytes = sizeof(io_uring_sqe) * 2;  // SQE128 = 两个普通 SQE 的宽度
        std::memset(sqe, 0, kSqe128Bytes);  // 清空 128 字节，防止上一批残留位
        sqe->opcode = IORING_OP_URING_CMD;  // 走 uring_cmd 透传，而不是 read/write
        sqe->fd = fd;  // 目标 NVMe 字符设备
        sqe->cmd_op = NVME_URING_CMD_IO;  // NVMe 的 uring_cmd 子命令：普通 IO
        sqe->len = sizeof(struct nvme_uring_cmd);  // cmd 区有效长度
        sqe->user_data = user_data;  // token|index，收割时用它找回命令
        std::memcpy(sqe->cmd, &cmd, sizeof(cmd));  // 命令本体放进 80 字节 cmd 区
    }

    uint32_t NextCompletionToken() {  // 每批一个单调递增 token
        ++completion_token_;
        if (completion_token_ == 0) {  // 跳过 0，0 留给 DecodeUserData 判非法
            ++completion_token_;
        }
        return completion_token_;
    }

    static uint64_t EncodeUserData(uint32_t token, size_t command_index) {
        return (static_cast<uint64_t>(token) << 32) |  // 高 32 位：批次 token
               (static_cast<uint32_t>(command_index) + 1u);  // 低 32 位：index+1，0 表示非法
    }

    static bool DecodeUserData(uint64_t user_data, uint32_t expected_token,
                               size_t& command_index) {
        const uint32_t token = static_cast<uint32_t>(user_data >> 32);  // 取出批次 token
        const uint32_t encoded_index = static_cast<uint32_t>(user_data);  // 取出 index+1
        if (token != expected_token || encoded_index == 0) {  // token 不符 = 别的批次；0 = 非法编码
            return false;
        }
        command_index = static_cast<size_t>(encoded_index - 1u);  // 还原成 0 基下标
        return true;
    }
```

**这一段在做什么**

- `ResetRing` 是错误路径的唯一出口：`queue_exit` 释放内核资源、清零句柄、重新初始化；它保证下一批不会看到上一批残留的 SQ/CQ。
- `PrepareSqe` 负责"SQE 级别的胶水"：`IORING_OP_URING_CMD` + `NVME_URING_CMD_IO`，命令本体 `memcpy` 进 SQE 的 `cmd` 区。
- `static_assert(sizeof(cmd) <= 80)` 是编译期护栏：`nvme_uring_cmd` 有 72 字节，普通 SQE 的 `cmd` 区只有 64 字节，必须开 `IORING_SETUP_SQE128`。
- `EncodeUserData` / `DecodeUserData` 把"批次 token + 命令下标"压进 64 位 `user_data`，高 32 位防串批、低 32 位多存 1 以便用 0 表示非法。

**重点语法 / 惯用法**

- `static_assert` 放在函数体内，只在该模板/函数被实例化时检查，是"接口契约"最轻量的表达方式。
- `std::memset(sqe, 0, kSqe128Bytes)` 必须清满 128 字节：SQE 槽位会被复用，残留位会让 `cmd_op`、`personality`、`flags` 等字段带进来历不明的值。
- `completion_token_ == 0` 时再 `++` 一次，是"跳过保留值"的常见写法（0 被 `DecodeUserData` 当作非法）。
- `+1u` / `-1u` 成对出现，把 0 变成哨兵值；代价是 `commands.size()` 必须 `< UINT32_MAX`（5.2.1 已检查）。
- token 是 `uint32_t`，回绕后理论上可能与旧值相同；实践中依赖"旧 CQE 不会存活 2^32 批"这一前提。

**与 6.7.3 的差异**

- 图上第 13/20 步没有"ring 重建"这一环；只要出现过 stale CQE、提交失败或等待失败，代码都会重建 ring，代价是重新 `mmap` 与重新探测 CQE32。
- 图上没有 token 概念；`user_data` 不是简单的命令下标，而是"批次号 + 下标 + 1"，这是防 CQE 串批的关键。
- `ResetRing` 会丢弃**未收割的 CQE**，因此调用它之前必须先把 `inflight_ops` 收到 0（5.2.2 的清理循环），否则那些命令的结果永远丢失。
- SQE128/CQE32 是硬件/内核能力：不支持时 `InitializeRing` 退化重试（`InitializeRing` 里的 `params.flags = IORING_SETUP_SQE128` 分支），这决定了 `big_cqe` 是否可用。

### 5.3 ioctl 的单命令提交

io_uring 执行器是批量窗口，`ioctl` 执行器则是"一命令一系统调用"的对照实现。它的公共接口相同（`Store`/`Retrieve`/`Delete`），差别全在这个 `Submit` 里。

mooncake-store/src/nvme_kv/executor_ioctl.cpp:179-205

```cpp
    tl::expected<uint32_t, ErrorCode> Submit(nvme_passthru_cmd &cmd,
                                             bool is_write,
                                             const char *op_name) const {  // 单条命令的提交入口
        errno = 0;  // 先清零：ioctl 成功时不会写 errno，不清零会读到上一次的旧值
        const int ret = ::ioctl(fd_, NVME_IOCTL_IO_CMD, &cmd);  // 同步 ioctl，返回时命令已完成
        const int err = errno;  // 立刻保存 errno，后续 LOG/映射都不能保证它不被改写
        if (ret < 0) {  // -1：系统调用失败，具体原因在 errno
            const ErrorCode mapped = MapNvmeKvTransportError(err, is_write);  // EINVAL/ENOMEM/ENODEV → 传输层错误码
            LOG(ERROR) << "[NvmeKvIoctlExecutor] ioctl failed"
                       << " op=" << op_name << " device=" << device_path_
                       << " errno=" << err << " strerror=" << strerror(err)
                       << " mapped_error=" << toString(mapped);
            return tl::make_unexpected(mapped);
        }
        if (ret > 0) {  // 正数：NVMe 完成状态码（命令失败，但设备正常响应）
            const ErrorCode mapped =
                MapNvmeKvStatus(static_cast<uint32_t>(ret), is_write);  // 0x87→OBJECT_NOT_FOUND 等
            if (!IsNvmeKvControlFlowError(mapped)) {  // key 不存在/key 已存在属于正常控制流，不该刷 ERROR
                LOG(ERROR) << "[NvmeKvIoctlExecutor] ioctl returned NVMe status"
                           << " op=" << op_name << " device=" << device_path_
                           << " status=" << ret
                           << " mapped_error=" << toString(mapped);
            }
            return tl::make_unexpected(mapped);
        }
        return static_cast<uint32_t>(cmd.result);  // 0：成功；把命令的 result 字段（如实际字节数）返回给调用方
    }
```

**这一段在做什么**

- `NVME_IOCTL_IO_CMD` 是同步语义：`ioctl` 返回时命令已经完成，没有队列、没有 CQE、没有 `user_data`。
- 返回值的三态与 io_uring 的 `cqe->res` 完全一致：`<0` → `errno` 传输错误，`>0` → NVMe 状态码，`==0` → 成功。
- 成功时把 `cmd.result` 作为 `uint32_t` 返回，这对应 io_uring 路径里的 `big_cqe[0]`（`observed_result`）。
- `IsNvmeKvControlFlowError(mapped)` 用来抑制"key 不存在/已存在"这类**预期内**错误的日志，避免正常业务把日志刷满。

**重点语法 / 惯用法**

- `errno = 0` + 立即 `const int err = errno`：C 风格系统调用封装的固定套路，任何中间函数调用都可能覆盖 `errno`。
- `::ioctl` 用全局作用域解析，明确调用 POSIX 接口而不是成员/重载。
- `strerror(err)` 只用于日志，不能拿去和错误码比较；比较一律用 `errno` 常量。
- 返回 `tl::expected<uint32_t, ErrorCode>`：成功值携带"实际字节数"，错误值只带 `ErrorCode`，与 io_uring 路径的 `observed_result` 对齐。
- `MapNvmeKvStatus` / `MapNvmeKvTransportError` 两个映射函数与 io_uring 执行器共用（`executor_util.cpp`），保证两条路径对同一状态码给出同一个 `ErrorCode`。

**与 6.7.3 的差异**

- 图上第 13/20 步的批量语义在 ioctl 路径上**不存在**：每条命令一次 `ioctl`，队列深度只能靠上层多线程并发来利用。
- io_uring 路径里 `res > 0` 的状态码来自 CQE，这里来自 `ioctl` 的**返回值本身**；两者数值语义相同，所以 `MapNvmeKvStatus` 可以复用。
- 这里没有 stale CQE、没有 token、没有 `ResetRing`：同步调用没有"迟到完成"的问题，但也没有任何批量摊销。
- `cmd.result` 在 ioctl 路径是直接可读字段；io_uring 路径要开 `CQE32` 才能从 `big_cqe[0]` 拿到同一个值，否则 `has_observed_result` 为 false。

## 6. 与 6.7.3 的差异汇总

这一节只列"图上看不出来、但读代码会踩到"的点。更细的 14 条（含 6.7.2 类图问题）在 `FileStorage-offload-实现解读.md` 的 6.7.4.6。

1. **第 9–10 步是一次 master RPC**。`BatchQuerySegmentSlices` 不是纯本地动作：它先 `Client::BatchQuery` → `BatchGetReplicaList` 拿副本列表，再在本地挑第一个 `IsReplicaOnLocalMemory()` 的地址。找不到本机内存副本的 key 直接进失败集合，是 best-effort。
2. **"零拷贝"有三处前提**：offload 源只对 host 内存零拷贝（GPU 先 D2H）；写出时只有"满块 + 源地址 4 KiB 对齐"能直接提交；读回时只有"满块 + 目标地址 4 KiB 对齐"能设备直读。尾块永远走 buffer + memcpy。
3. **第 14 步发生在第 12 步内部**。`NotifyOffloadSuccess` 由 `BatchOffload` 的 `complete_handler` 同步触发，并且按 bucket 逐次上报；失败 key 用同一个 RPC、`data_size = -1` 作 NACK。
4. **第 15 步不是无条件新增副本**。已有 `LOCAL_DISK` 副本时只更新同 client 的 `transport_endpoint / object_size`；LOCAL_DISK segment 已卸载时拒绝登记（返回 `SEGMENT_NOT_FOUND`），但 `dec_refcnt` 已经执行。
5. **第 5 步入队是尽力而为**。`PutEnd` 静默忽略入队失败，`Put` 仍成功；入队门是 `replica_type != DFS && enable_offload_ && !offload_on_evict_`；`offload_on_evict_` 为真时改在淘汰路径入队。
6. **第 6 步的"尚未落盘"只对 NVMe KV 成立**。若本次同时写了 `NoF_SSD` 副本，那份在 `PutEnd` 之前就已经落到 Nvme-oF SSD。
7. **两个批量接口的失败语义相反**：`BatchOffload` 是"部分成功 + 回调上报"，返回值仍可能带第一个错误；`BatchLoad` 是 all-or-nothing，一个 key 失败整批失败。
8. **第 20 步的 root 读可能是 1 次或 2 次**。`ResolveRoot` 用 `size_hint = 0`，首读 4096 字节；root blob ≤ 4096 时一次读完（默认参数下覆盖到 payload 约 42 MiB），更大则设备报 `0x85`、走最大 buffer 重试。
9. **第 17 步的 `lease_ttl_ms` 不保护 offload 数据**。它是对象级读租约，只防 MEMORY 副本被淘汰；offload buffer 的租约是第 22 步的 `gc_ttl_ms`，而且只是读者侧截止时间（超时后读完返回 `OBJECT_HAS_LEASE`）。
10. **第 23 步只在跨进程时是 RDMA READ**。读者就是 owner 时走 `kLocalAddress`，退化为本地 memcpy。跨进程时 `TransferRequest` 的 `source` 字段是**读者本地落点**，`target_id / target_offset` 才指向 owner segment。
11. **第 24 步不是显式调用**。释放由 `OffloadRestoreLease` 析构触发，调用点不检查返回值；读者异常退出时靠 `client_buffer_gc_ttl_ms` 到期的 GC 线程兜底。
12. **重启不恢复目录**。`NvmeKvStorageBackend::ScanMeta` 是空实现，`total_keys_ / total_size_` 也在 `Init` 里清零：Master 侧 `LOCAL_DISK` 记录不会重建，配额从 0 起算。

## 7. C++ 语法与惯用法速查

只列本文代码里真实出现、且容易看错的写法。

| 写法 | 典型位置 | 说明 / 坑 |
| --- | --- | --- |
| `tl::expected<T, ErrorCode>` | 2.1–4.6 | 值或错误二选一。`if (!x)` 判错，`x.value()` / `*x` 取值；对 `void` 特化成功返回 `{}`。 |
| `tl::make_unexpected(e)` / `tl::unexpected(e)` | 2.1 / 4.1 | 前者直接构造一个失败的 `expected`（返回值），后者只构造包装、常作为 `emplace_back` 的实参。两者不能混用。 |
| `tl::in_place` | 4.1 `BatchQuery` | 让 `expected<QueryResult>` 原地构造，避免先构造再 move。 |
| `std::optional<T>` | 2.4 / 3.3 | 空值本身有业务语义（`object_checksum` 未启用、`abort_error` 未发生），不要用 0/空串当哨兵。 |
| designated initializers | 2.5 / 3.1 / 3.3 | `OffloadTaskItem{.tenant_id = ..., .key = ..., .size = ...}` 是 C++20 聚合初始化；字段顺序必须与声明一致，漏字段会零初始化。 |
| 结构化绑定 | 3.3 / 3.5 / 4.4 | `for (const auto &[key, slices] : batch_object)`；`auto [it, inserted] = cache.try_emplace(...)`。绑定的是引用/副本，注意 `auto &` 与 `auto` 的区别。 |
| `std::string_view` | 3.4 / 5.1 | 不拥有内存。`StoreRequest::value` 指向源 buffer，所以 `StoreBatch` 必须同步完成（见 3.5 的注释）。 |
| `std::span<const char>` | 2.1 | 对连续内存的非拥有视图；`value.data()` 可能是设备指针，所以先 `gather_maybe_device_to_host`。 |
| `std::latch` + `count_down()/wait()` | 3.5 | 一次性闩锁，用来等固定数量的 worker；异常路径也要 `count_down` 剩余数量，否则永久阻塞。 |
| `std::atomic<size_t>` + `fetch_add` | 3.5 | 多个 worker 无锁领取下标；`memory_order_relaxed` 足够，因为只做计数分发。 |
| `thread_local std::unique_ptr<Ring>` | 5.2 | 每个线程一个 io_uring 环；`thread_local` 的析构在退出线程时执行，避免跨线程共享 SQ/CQ。 |
| `std::shared_mutex` + `shared_lock` / `unique_lock` | 3.4（manifest cache） | 读多写少：查缓存用共享锁，写缓存用独占锁。 |
| `try_emplace` | 3.4 / 4.4 | 键不存在才构造，避免无谓的值构造；返回 `{iterator, inserted}`。 |
| `std::function` 回调 | 3.3 / 3.4 | `complete_handler` / `eviction_handler` 按值捕获时要保证生命周期覆盖调用；NVMe KV 直接丢弃 `eviction_handler`。 |
| `posix_memalign` + `memset` | 5.1 | DMA buffer 必须 4 KiB 对齐并按对齐长度补零；`std::malloc` 不保证。 |
| `reinterpret_cast<uintptr_t>` | 4.5 / 5.1 | 做地址对齐判断；`ptr % 4096 == 0` 只对整数地址有意义。 |
| `unique_ptr::release()` + `shared_ptr(ptr, deleter)` | 5.2 | 把对齐 buffer 的所有权交给 `RetrievedBuffer::owner`，deleter 用 `free`；`release()` 之后原 `unique_ptr` 不再负责释放。 |
| RAII 析构释放 | 4.6 | `OffloadRestoreLease` 析构时发 `release_offload_buffer`；不要手动调，也不要 move 这个对象（它删除了拷贝/移动）。 |
| `#ifdef USE_NOF` / `#ifdef MOONCAKE_HAVE_NVME_URING_CMD` | 2.3 / 5.2 | 编译期裁剪：NoF 副本与 io_uring executor 都可能根本不存在，运行时配了也不会生效。 |
| `std::numeric_limits<size_t>::max()` | 3.4 | 手写溢出检查（`slice.size > max - payload_size`），因为 `size_t` 回绕是未定义语义的常见来源。 |
| `[[nodiscard]]` | `FileStorage::HasPinnedRestoreArena` 等 | 返回值被忽略通常是 bug；编译期警告。 |
| `GUARDED_BY(mutex)` | `FileStorage` 成员 | Clang thread-safety 注解，编译期检查"读这个字段前是否持锁"。 |
| `LOG(INFO)` / `VLOG(1)` / `LOG(WARNING)` | 全文 | glog；`VLOG(1)` 只在 `--v=1` 时输出，热路径用它避免日志开销。 |
| `constexpr` 常量 | 5.1 | 队列深度、对齐、opcode 等编译期常量；与环境变量读出的运行期值分开。 |
| `enum class` + `static_cast<uint32_t>` | 5.1 | 强类型枚举，落进 NVMe 命令字段前要显式转换。 |
