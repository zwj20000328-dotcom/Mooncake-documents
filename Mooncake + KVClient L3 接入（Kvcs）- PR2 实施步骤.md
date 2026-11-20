# Mooncake + KVClient L3 接入（Kvcs）— PR2 实施步骤

版本：v1.3　日期：2026-11-19

对应设计文档：`Mooncake + KVClient L3 接入（Kvcs）- PR 实施设计.md`（PR2 章节）

代码基线：`Mooncake` 分支 `Mooncake_l3_pr2`，HEAD `f3c8ffd5`（含 PR1 最新调整）；SDK 基线 `tmp_pr1641/ucm-pr`

---

## 0. 前提与决策

### 0.1 PR2 边界

PR2 在 PR1 的控制面之上补齐数据面与 Master 门控：

- 适配器：`PutBatch` / `GetBatch` 批读写（读在 `waitForCompletion` 内同步完成）；`Exists` 查询。
- buffer 注册：适配器的私有成员函数 `RegisterBuffers` 按请求 `RegisterRegions`，调用返回前统一用 `UnregisterBuffers` 注销；不修改请求结构或上层接口。
- 写路径：零拷贝，使用 `RegisterBuffers` 返回的 handle 提交 `StoreAsync`；不做 staging、不补齐。
- 读路径：`GetBatch` 内注册目标 buffer 后提交 `LoadAsync`，`waitForCompletion` 等到终态后逐项返回 N；返回前注销 handle。
- 超时、参数校验与设备错误全部由 KvClient SDK 负责；适配器不设超时、不校验请求、不保留额外资源。
- 读接口不变：`BatchRead`/`BatchGet`/`ReadDfsReplica` 保持同步，不新增完成句柄。
- Master：`use_kvclient_`、`enable_dfs_`、kvclient 早退分支、descriptor 铸造、NOF 校验、驱逐白名单。
- 测试：PR2 自带适配器、注册/注销与读结果单测。

PR2 不做：

- Client 侧 value-first 删除、`ReconcileMissingKvReplica` 改名与放宽、`AttachHostId`（PR3）。
- `ListKeys`：保持 `NOT_SUPPORTED`；`Exists` 在 PR2 实现（`QueryAsync`）。
- `CheckHealth` / `GetRange` / `PutOnce` / `PutVOnce`：保持基类默认。
- 适配器侧任何超时或参数策略；这些属于 KvClient SDK。

### 0.2 已确认决策（2026-11-19）

| 决策项 | 结论 | 落地要求 |
| --- | --- | --- |
| 写路径 | 零拷贝；成功与否由 KvClient 判定 | 适配器不 staging、不补齐、不校验；只保留空 key、N==0、空指针、同批重复 key 守卫 |
| buffer 注册 | 适配器内部按请求注册，调用返回前注销 | `RegisterBuffers` / `UnregisterBuffers` 是适配器私有成员函数；不改 `ObjectPutRequest`、`ObjectStorageAdapter` 与 `DistributedStorageBackend` |
| 读路径 | 适配器内部注册目标 buffer 做零拷贝读 | `GetBatch` 内 `LoadAsync` + `waitForCompletion`；返回即逐项 N，随后注销 |
| 等待 | 每次提交调用一次 `Wait` | `waitForCompletion(task_id, count)`；返回什么就映射什么，不设适配器超时状态 |
| 超时 | 由 KvClient SDK 负责 | `Wait(task_id, 0, result)`；适配器不重试、不保留资源 |
| `Exists` | PR2 实现：`QueryAsync` + `Wait` | 不注册 buffer；按 entry 级 `ENTRY_KEY_EXIST` / `ENTRY_KEY_NOT_EXIST` 返回 |
| 配置项 | `KvClientAdapterConfig` 只有 `config_path` | 适配器不校验任何数值配置 |
| kvclient + oplog | 启动期 `LOG(FATAL)` 拒绝 | 在 `InitDfsAllocatorFromEnvironment` 的 kvclient 分支内拒绝；恢复门保持 KVCS-only |
| 测试归属 | PR2 自带 | 新增 `kvclient_object_storage_adapter_test.cpp`，仅 SDK 存在时编译 |

### 0.3 与旧文档的差异（实施前必读）

| # | 旧文档写法 | 当前做法 |
| --- | --- | --- |
| 1 | 给请求结构加字段 | 不改请求结构；适配器内部注册 |
| 2 | 适配器内做请求校验/拷贝/资源保留 | 全部删除；请求校验与超时由 KvClient SDK 负责 |
| 3 | `aligned_size = AlignUp(...)` | `object_size = aligned_size = value_length` |
| 4 | Master 预检请求参数 | 不预检；成功/失败由 SDK 判定 |
| 5 | 同步 `GetBatch` | 保持同步；内部 `LoadAsync` + `waitForCompletion`，不引入完成句柄 |
| 6 | `Exists` 保持 `NOT_SUPPORTED` | PR2 实现：`QueryAsync` + entry 级 code |
| 7 | `Completion` 重新声明为命名空间级类型 | 已是类内私有类型，不再声明 |
| 8 | `PutBatch` 无初始化守卫 | 入口检查 `!initialized_ \|\| !kv_client_`，逐项 `DFS_SERVICE_UNAVAILABLE` |
| 9 | 非终态逐项映射 | 不再区分；SDK 返回什么就映射什么 |
| 10 | 适配器每批注册 buffer | 保留每批注册，改为适配器私有成员 `RegisterBuffers` / `UnregisterBuffers`；不改上层 |

### 0.4 分支

```bash
git switch Mooncake_l3_pr2   # 已从 Mooncake_l3_pr1 创建
```

PR3 分支 `Mooncake_l3_pr3` 已基于 PR2 创建，PR2 合入后再开始。

---

## 1. 实施顺序总览

| 步骤 | 内容 | 主要文件 | 验证点 |
| --- | --- | --- | --- |
| 1 | 写路径零拷贝 | `kvclient_object_storage_adapter.cpp` | 不拷贝、不补齐、不校验 |
| 2 | `RegisterBuffers` / `UnregisterBuffers` | 适配器 `.h` / `.cpp` | 按请求注册、返回前注销 |
| 3 | helper | 同上 `.cpp` | 求和、错误填充 |
| 4 | 适配器头 | `kvclient_object_storage_adapter.h` | PR2 不新增声明 |
| 5 | `PutBatch` | 同上 `.cpp` | 零拷贝提交/等待/映射 |
| 6 | `GetBatch`（`LoadAsync` + `waitForCompletion`） | 同上 `.cpp` | 逐项返回 N |
| 7 | 单条包装、`Exists`、析构 | 适配器 `.cpp` | `Put`/`PutV` 走 batch；`Exists` 查询；析构只 `Shutdown` |
| 8 | Master 门控 | `master_service.h/.cpp` | 启动、descriptor、白名单 |
| 9 | 测试 | `tests/kvclient_object_storage_adapter_test.cpp`、`tests/CMakeLists.txt` | 用例见 §10.3 |
| 10 | 构建验证 | CMake / ctest | SDK 缺失与存在两种构建 |
| 11 | PR 提交 | PR 模板 | 边界写清 |

---

## 2. 步骤 1：写路径零拷贝

### 2.1 模型

KVClient 写路径在适配器内部为每个请求注册 buffer，用返回的 handle 构造 `KVBuffer` 并交给 `StoreAsync`，不做任何数据拷贝或补齐。参数合法性、超时、设备侧可写性全部由 KvClient 判定；失败时按 SDK 状态码映射为 Mooncake `ErrorCode`。

因此：

- 适配器只保留空 key、`N == 0`、空指针、同批重复 key 守卫。
- 不分配 staging、不 `memcpy`、不 `memset`、不补齐。
- `ObjectPutRequest` 不新增字段；注册与注销由 `RegisterBuffers` / `UnregisterBuffers` 在 `PutBatch` 内完成。
- 适配器不设超时、不重试、不保留跨调用资源；这些由 KvClient SDK 负责。

---

## 3. 步骤 2：适配器头文件

文件：`mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h`

### 3.1 include

现有 include 只有 `<memory>/<string>/<string_view>/<vector>`，需要补 `<cstdint>`（新增成员签名里的 `uint64_t`）。

### 3.2 私有成员与声明

```cpp
   private:
    struct Completion {
        bool terminal = false;
        std::vector<kv::Status> entryStatus;
    };

    kv::BinaryKey MakeCacheKey(std::string_view logical_key) const;
    // 调用一次 SDK Wait；OK/PARTIAL_FAILED 的 entryStatus 有效，
    // 其他状态把 task 级状态填满每个 entry。超时由 SDK 判定。
    Completion waitForCompletion(kv::TaskId task_id, size_t entry_count);
    static ErrorCode ToErrorCode(const kv::Status& status);
    // 注册单块 host 内存；返回的 handle 在交给 UnregisterBuffers 前保持有效。
    tl::expected<uint64_t, ErrorCode> RegisterBuffers(void* ptr, size_t size);
    // 释放 handle；SDK 无法重试的失败只记日志。
    void UnregisterBuffers(const std::vector<uint64_t>& handles);

    KvClientAdapterConfig config_;
    std::unique_ptr<kv::KvClient> kv_client_;
    bool initialized_ = false;
```

约束：

- `Completion`、`waitForCompletion`、`MakeCacheKey`、`ToErrorCode`、构造函数与 `config_/kv_client_/initialized_` 在 PR1 已存在，**不要重复声明**。
- PR2 只新增 `RegisterBuffers` / `UnregisterBuffers` 两个私有成员声明；它们是成员而非匿名命名空间函数，因此直接用 `kv_client_`，调用点不需要传 client。
- 该头文件只有自己的 `.cpp` 一个包含者，改动不会触发其他 TU 重编。
- 适配器没有 `BufferPlan`，也不新增 `RegisterBuffer` / `UnregisterBuffer` / `RequiresBufferHandle` 这三个公共虚函数。
- `GetBatch` 已在 base override（PR1 返回 `NOT_SUPPORTED`），PR2 改为真实实现；不新增 `GetBatchAsync` / `ObjectReadCompletion`。
- `Exists` 已在类中 override，PR2 用 `QueryAsync` 实现。

---

## 4. 步骤 3：文件内 helper

文件：`mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp`

新增 include：`<limits>`、`<optional>`、`<set>`。

不提供独立的求和/填充 helper：分段求和直接内联在 `PutBatch` 的逐请求循环里，错误填充也在各调用点就地处理。原 `Errors<T>`（`.cpp:21-28`）在数据面替换后成为死代码，一并删除。

---

## 5. 步骤 4：`RegisterBuffers` / `UnregisterBuffers`

### 5.1 两个私有成员函数

注册与注销各一个私有成员函数，声明在适配器头文件（§3.2），实现放在 `.cpp` 中其他私有成员之后、`Delete` 之前：

```cpp
tl::expected<uint64_t, ErrorCode>
KvClientObjectStorageAdapter::RegisterBuffers(void* ptr, size_t size) {
    if (ptr == nullptr || size == 0) {
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    std::vector<kv::MemoryRegion> regions{
        {kv::MemoryType::HOST, reinterpret_cast<uint64_t>(ptr), size, -1, -1}};
    std::vector<kv::RegisteredMemory> registered;
    const auto status = kv_client_->RegisterRegions(regions, registered);
    if (!status.ok() || registered.size() != 1) {
        return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);
    }
    return registered[0].handle;
}

void KvClientObjectStorageAdapter::UnregisterBuffers(
    const std::vector<uint64_t>& handles) {
    // 注销失败只记日志：SDK 保留该注册，且已无 handle 可重试。
    for (uint64_t handle : handles) {
        const auto status = kv_client_->UnregisterRegions({handle});
        if (!status.ok()) {
            LOG(WARNING) << "KVClient buffer unregister failed, handle="
                         << handle << ", error=" << status.message;
        }
    }
}
```

### 5.2 生命周期

- 两个函数都用成员 `kv_client_`，调用点不传 client；非空由 `PutBatch` / `GetBatch` 的入口守卫保证。
- `PutBatch` / `GetBatch` 用局部 `std::vector<uint64_t> handles` 收集 `RegisterBuffers` 的返回值，函数返回前（正常或提前）统一调用 `UnregisterBuffers`。
- 注销发生在 `Wait` 返回之后；SDK 不会在任务完成时自动释放注册（`registeredRegions_` 只在 `UnregisterRegions` 中删除，`Shutdown` 仅批量兜底）。
- 注销失败只记录 `LOG(WARNING)`。
- `Wait` 返回非终态（超时等）时仍会注销，属于已知风险，见 §15。
- 两个函数不做 RAII；`PutBatch` 用 `try { ... } catch (...) { UnregisterBuffers(handles); throw; }` 包住注册与提交，异常展开时同样会注销已注册的 handle。

约束：

- 不改 `ObjectPutRequest` / `ObjectGetRequest`、`ObjectStorageAdapter`、`DistributedStorageBackend`。
- 注册失败按项隔离：该项返回错误、不写入 `seen`，不影响同批其他请求。
- 并发重复注册同一 `(addr, size)` 由上层保证 buffer 不重叠使用，或由 SDK 的引用计数处理；PR2 不引入适配器侧引用计数。

---

## 6. 步骤 5：`Put` / `PutV` / `PutBatch`

单条入口按 KVCS 的结构包装，不单独实现 I/O：

```cpp
// Put → PutV → PutBatch。
tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::Put(
    const std::string& logical_key, std::span<const char> data) {
    iovec iov{const_cast<char*>(data.data()), data.size()};
    return PutV(logical_key, &iov, 1);
}

tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::PutV(
    const std::string& logical_key, const iovec* iov, int iovcnt) {
    auto results = PutBatch({{logical_key, iov, iovcnt}});
    return results.empty() ? tl::make_unexpected(ErrorCode::INVALID_PARAMS)
                           : std::move(results.front());
}
```

`PutBatch` 是唯一的写 I/O 实现：

```cpp
std::vector<tl::expected<void, ErrorCode>>
KvClientObjectStorageAdapter::PutBatch(
    const std::vector<ObjectPutRequest>& requests) {
    std::vector<tl::expected<void, ErrorCode>> results(requests.size());

    if (!initialized_ || !kv_client_) {
        for (auto& result : results) {
            result = tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);
        }
        return results;
    }

    std::vector<uint64_t> handles;
    handles.reserve(requests.size());
    // 注册过的 handle 在批次异常展开时也必须释放。
    try {
        std::set<kv::BinaryKey> seen;   // 仅写侧：同批重复 key
        std::vector<kv::KVBuffer> entries;
        std::vector<size_t> indices;
        entries.reserve(requests.size());
        indices.reserve(requests.size());
        for (size_t i = 0; i < requests.size(); ++i) {
            if (requests[i].logical_key.empty() || requests[i].iov == nullptr ||
                requests[i].iovcnt <= 0) {
                results[i] = tl::make_unexpected(ErrorCode::INVALID_PARAMS);
                continue;
            }
            const auto binary_key = MakeCacheKey(requests[i].logical_key);
            if (seen.find(binary_key) != seen.end()) {
                results[i] = tl::make_unexpected(ErrorCode::INVALID_PARAMS);
                continue;
            }
            // 逐段求和：空段或回绕都会导致注册/提交区间错误。
            size_t total = 0;
            bool valid = true;
            for (int j = 0; j < requests[i].iovcnt; ++j) {
                const auto& segment = requests[i].iov[j];
                if (segment.iov_base == nullptr || segment.iov_len == 0 ||
                    segment.iov_len >
                        std::numeric_limits<size_t>::max() - total) {
                    valid = false;
                    break;
                }
                total += segment.iov_len;
            }
            if (!valid || total == 0) {
                results[i] = tl::make_unexpected(ErrorCode::INVALID_PARAMS);
                continue;
            }
            auto handle = RegisterBuffers(requests[i].iov[0].iov_base, total);
            if (!handle) {
                results[i] = tl::make_unexpected(handle.error());
                continue;
            }
            // 先登记 handle，后续分配抛异常时 catch 仍能释放它；
            // 只有完全受理的请求才认领 key，避免无效项挡住后续合法请求。
            handles.push_back(*handle);
            seen.insert(binary_key);

            kv::KVBuffer entry;
            entry.key = kv::CacheKey::FromBytes(binary_key);
            entry.buffer.region = {
                kv::MemoryType::HOST,
                reinterpret_cast<uint64_t>(requests[i].iov[0].iov_base),
                total, -1, -1};
            entry.buffer.handle = *handle;
            entry.offset = 0;
            entry.mrKey = std::nullopt;      // SDK 按 handle 回填
            entries.push_back(std::move(entry));
            indices.push_back(i);
        }

        if (!entries.empty()) {
            kv::TaskId task_id = kv::kInvalidTaskId;
            const auto submit = kv_client_->StoreAsync(entries, task_id);
            if (!submit.ok()) {
                for (size_t index : indices) {
                    results[index] = tl::make_unexpected(ToErrorCode(submit));
                }
            } else {
                const auto completion = waitForCompletion(task_id, entries.size());
                if (completion.entryStatus.size() != entries.size()) {
                    for (size_t index : indices) {
                        results[index] =
                            tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
                    }
                } else if (!completion.terminal) {
                    // Task 级错误：entryStatus 已被填成 task 状态，统一映射。
                    const auto error = ToErrorCode(completion.entryStatus[0]);
                    for (size_t index : indices) {
                        results[index] = tl::make_unexpected(error);
                    }
                } else {
                    for (size_t j = 0; j < entries.size(); ++j) {
                        const auto& status = completion.entryStatus[j];
                        results[indices[j]] =
                            status.ok() ? tl::expected<void, ErrorCode>{}
                                        : tl::make_unexpected(ToErrorCode(status));
                    }
                }
            }
        }
    } catch (...) {
        UnregisterBuffers(handles);
        throw;
    }
    UnregisterBuffers(handles);
    return results;
}
```

约束：

- 不校验请求参数；由 `StoreAsync` 返回的 SDK 状态映射。
- 已确认 `StoreAsync` 覆盖写，不预删。
- 提交失败与 Task 级错误只覆盖参与项，空 key / 重复 key / 注册失败的错误保留。
- 每个参与项由 `RegisterBuffers` 注册；`Wait` 返回后、函数返回前统一用 `UnregisterBuffers` 注销。
- 单一出口：正常路径与提前返回都经过结尾的 `UnregisterBuffers`；异常路径由 `catch` 注销后 rethrow，两者不会重复注销。
- `handles` 在 `try` 外声明，`catch` 才可见；`handles.reserve` 保证 `push_back` 不抛，登记动作本身无异常窗口。
- `Put` / `PutV` 只是单条包装，最终都汇聚到 `PutBatch`。

---

## 7. 步骤 6：`GetBatch`（`LoadAsync` + `waitForCompletion`）

### 7.1 实现

`GetBatch` 已在 base override（PR1 返回 `NOT_SUPPORTED`），PR2 改成与 `PutBatch` 同构的真实实现：

```cpp
std::vector<tl::expected<size_t, ErrorCode>>
KvClientObjectStorageAdapter::GetBatch(
    const std::vector<ObjectGetRequest>& requests) {
    std::vector<tl::expected<size_t, ErrorCode>> results(requests.size());
    if (!initialized_ || !kv_client_) {
        for (auto& result : results) {
            result = tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);
        }
        return results;
    }

    std::vector<uint64_t> handles;
    handles.reserve(requests.size());
    // 注册过的 handle 在批次异常展开时也必须释放。
    try {
        std::vector<kv::KVBuffer> entries;
        std::vector<size_t> indices;
        entries.reserve(requests.size());
        indices.reserve(requests.size());
        for (size_t i = 0; i < requests.size(); ++i) {
            if (requests[i].logical_key.empty() ||
                requests[i].buffer == nullptr || requests[i].size == 0) {
                results[i] = tl::make_unexpected(ErrorCode::INVALID_PARAMS);
                continue;
            }
            // 整段目标 buffer 都注册；SDK 直接把对象写进这块内存。
            auto handle =
                RegisterBuffers(requests[i].buffer, requests[i].size);
            if (!handle) {
                results[i] = tl::make_unexpected(handle.error());
                continue;
            }
            // 先登记 handle，后续分配抛异常时 catch 仍能释放它。
            handles.push_back(*handle);

            kv::KVBuffer entry;
            entry.key =
                kv::CacheKey::FromBytes(MakeCacheKey(requests[i].logical_key));
            entry.buffer.region = {
                kv::MemoryType::HOST,
                reinterpret_cast<uint64_t>(requests[i].buffer),
                requests[i].size, -1, -1};
            entry.buffer.handle = *handle;
            entry.offset = 0;
            entry.mrKey = std::nullopt;   // SDK 按 handle 回填
            entries.push_back(std::move(entry));
            indices.push_back(i);
        }

        if (!entries.empty()) {
            kv::TaskId task_id = kv::kInvalidTaskId;
            const auto submit = kv_client_->LoadAsync(entries, task_id);
            if (!submit.ok()) {
                for (size_t i : indices)
                    results[i] = tl::make_unexpected(ToErrorCode(submit));
            } else {
                const auto completion =
                    waitForCompletion(task_id, entries.size());
                if (completion.entryStatus.size() != entries.size()) {
                    for (size_t i : indices)
                        results[i] =
                            tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
                } else if (!completion.terminal) {
                    // Task 级错误：entryStatus 已被填成 task 状态，统一映射。
                    const auto error = ToErrorCode(completion.entryStatus[0]);
                    for (size_t i : indices)
                        results[i] = tl::make_unexpected(error);
                } else {
                    for (size_t j = 0; j < entries.size(); ++j) {
                        const auto& st = completion.entryStatus[j];
                        const size_t index = indices[j];
                        // 返回请求长度；后端用它比对以发现短读。
                        results[index] =
                            st.ok()
                                ? tl::expected<size_t, ErrorCode>{
                                      requests[index].size}
                                : tl::make_unexpected(ToErrorCode(st));
                    }
                }
            }
        }
    } catch (...) {
        UnregisterBuffers(handles);
        throw;
    }
    UnregisterBuffers(handles);
    return results;
}
```

### 7.2 约束

- 成功项返回 `request.size`（N），不是 SDK 传输长度；`DistributedStorageBackend` 会再比对 `request.size == N`，不等则判 `FILE_READ_FAIL`。
- `Wait` 在 `GetBatch` 调用内完成；返回前用 `UnregisterBuffers` 注销 handle，buffer 随后可复用。
- 多 slice 由后端 staging 承接，只在本调用内有效；`GetBatch` 返回后散回。
- `entryStatus` 仅在 `OK` / `PARTIAL_FAILED` 时逐项可信；非终态统一映射 task 级状态。
- 注册失败按项隔离；适配器不做 payload 拷贝。
- 读路径**没有同批去重**：同一 key 读两次是合法的（各自独立的目标 buffer），因此不像 `PutBatch` 那样维护 `seen`。
- 单一出口与 `PutBatch` 相同：`handles` 在 `try` 外声明，正常路径与 `catch` 各注销一次，互不重复。
- **读 miss 的状态码与写/查询不同**：LOAD 用 `ENTRY_KEY_NOT_FOUND` / `ENTRY_DATA_NOT_EXIST` 报告未命中，QUERY 用 `ENTRY_KEY_NOT_EXIST`。PR1 的 `ToErrorCode` 只列了前两者中的 `ENTRY_KEY_NOT_FOUND`，本步骤补上 `ENTRY_DATA_NOT_EXIST`，否则 miss 会落到 `default` 变成 `INTERNAL_ERROR`，验收要求的 `OBJECT_NOT_FOUND` 不成立，PR3 的 miss 自愈也不会触发。这是对 PR1 已落地代码的一处修正，需在 PR 描述中说明。

---

## 8. 步骤 7：单条包装、`Exists`、析构

### 8.1 单条入口

单条写入口已在步骤 5 实现为 `Put` → `PutV` → `PutBatch` 的包装，这里不再重复。读路径的 `Get` 随 `GetBatch` 在步骤 6 实现，当前保持 `NOT_SUPPORTED`：

```cpp
tl::expected<size_t, ErrorCode> KvClientObjectStorageAdapter::Get(
    const std::string&, void*, size_t) {
    return tl::make_unexpected(ErrorCode::NOT_SUPPORTED);
}
```

### 8.2 析构

```cpp
KvClientObjectStorageAdapter::~KvClientObjectStorageAdapter() {
    // 停止受理并让 SDK 排空在途任务。
    if (kv_client_) kv_client_->Shutdown();
}
```

> 调用方保证 adapter 析构前没有并发 I/O；同步读写都在调用内完成，不存在外部完成句柄。

### 8.3 `Exists`

`Exists` 走 SDK 查询接口，不注册 buffer：

```cpp
tl::expected<bool, ErrorCode> KvClientObjectStorageAdapter::Exists(
    const std::string& logical_key) {
    if (!initialized_ || !kv_client_) {
        return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);
    }
    if (logical_key.empty()) {
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }

    std::vector<kv::CacheKey> keys{
        kv::CacheKey::FromBytes(MakeCacheKey(logical_key))};
    kv::TaskId task_id = kv::kInvalidTaskId;
    const auto submit = kv_client_->QueryAsync(keys, task_id);
    if (!submit.ok()) return tl::make_unexpected(ToErrorCode(submit));

    // key 不存在会让 task 级变成 PARTIAL_FAILED，但它仍携带可信的逐项结果，
    // 因此不能当失败处理。复用 waitForCompletion 而不是裸调 Wait。
    const auto completion = waitForCompletion(task_id, keys.size());
    if (!completion.terminal) {
        return tl::make_unexpected(ToErrorCode(completion.entryStatus[0]));
    }
    // 两种结果都由 Status::Error 产生，ok() 无法区分命中与未命中，必须按 code 分支。
    switch (completion.entryStatus[0].code) {
        case kv::StatusCode::ENTRY_KEY_EXIST: return true;
        case kv::StatusCode::ENTRY_KEY_NOT_EXIST: return false;
        default:
            return tl::make_unexpected(ToErrorCode(completion.entryStatus[0]));
    }
}
```

约束：

- **不能用 `entryStatus[0].ok()`**：`ENTRY_KEY_EXIST` 与 `ENTRY_KEY_NOT_EXIST` 都由 `Status::Error(...)` 产生（SDK `trans_task_utils.cc:142-143`），两种结果都不是 `OK`，必须按 code 分支。
- **不能用 `Wait` 的返回值当成功判据**：`WaitContext` 直接 `return result.status`，而 key 不存在时 `Finalize` 把 `finalStatus` 置为 `PARTIAL_FAILED`（`task_manager.cc:254-256`）。若写成 `if (!wait.ok()) return error`，每次 miss 都会返回错误，`false` 分支永远不可达。
- 复用 `waitForCompletion`：它已把 `OK` / `PARTIAL_FAILED` 判为终态（entryStatus 可信），非终态则把 task 状态填满每个 entry；与 `DeleteBatch` / `PutBatch` 行为一致。
- 不再做 `queryResult->exists.size()` 一致性校验：SDK 的 `CompleteTransportTask` 已校验条数并把不匹配转成 `INTERNAL_ERROR` 填进 entryStatus（`task_manager.cc:182-186`），落到本函数的 `default` 分支即映射为 `INTERNAL_ERROR`。
- 查询不注册 buffer，没有额外资源需要保留；`Exists` 不需要 try/catch。
- 空 key 本地拒绝，不发起 SDK 查询。

### 8.4 保持 `NOT_SUPPORTED`

- `ListKeys` 保持原占位（SDK 无 key 枚举接口）。
- `CheckHealth`、`GetRange`、`PutOnce`、`PutVOnce` 不开 override。

---

## 9. 步骤 8：Master 门控

文件：`mooncake-store/include/master_service.h`、`mooncake-store/src/master_service.cpp`

### 9.1 成员

`master_service.h:2033` 旁新增：

```cpp
bool enable_dfs_{false};
bool use_kvcs_{false};
bool use_kvclient_{false};   // 新增
```

### 9.2 `enable_dfs_`

`master_service.cpp:860-861`：

```cpp
enable_dfs_ = DfsEnablementConfig::FromEnvironment().enabled ||
              dfs_config.UsesKvcs() || dfs_config.UsesKvClient();
```

### 9.3 kvclient 早退分支（关键）

位置：`InitDfsAllocatorFromEnvironment` 内，KVCS 分支（`:886-921`）之后、filesystem allocator 与 recovery 检查（`:923`）之前。

```cpp
if (dfs_config.UsesKvClient()) {
    if (enable_oplog_) {
        LOG(FATAL) << "kvclient backend does not support snapshot/oplog "
                      "recovery; disable enable_oplog/snapshot";
    }
    use_kvclient_ = true;
    // 对象存储：不创建 EFC target 与水位淘汰。
    LOG(INFO) << "KVClient distributed object storage initialized, config={"
              << dfs_config.FormatStr() << "}";
    return;
}
```

不做这一步的后果：

- `enable_oplog_=true` 时落到 `:923-928` 的 `throw`，Master 起不来；
- `enable_oplog_=false` 时落到 `:930-951` 初始化 `ShardAllocator`，对 object storage 必然失败，`enable_dfs_` 被重置为 false，所有 DFS 写返回 `DFS_SERVICE_UNAVAILABLE`。

### 9.4 descriptor 铸造

`master_service.cpp:4995-5003`，`AllocateReplicas` 内。kvclient 与 KVCS 的 descriptor 形状一致：

```cpp
if (config.dfs_replica_num > 0) {
    if (use_kvcs_ || use_kvclient_) {
        DistributedFSDescriptor descriptor;
        descriptor.object_size = value_length;
        descriptor.aligned_size = value_length;
        descriptor.shard_idx = -1;
        replicas.emplace_back(std::move(descriptor),
                              ReplicaStatus::PROCESSING);
    } else if (!dfs_allocator_ || !dfs_allocator_->IsInitialized()) {
        LOG(ERROR) << "Failed to allocate DFS replica for key=" << key
                   << ", error=dfs_allocator_not_initialized";
        return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);
    } else {
        // 原 filesystem allocator 分支保持不变
    }
}
```

不需要新增 `kvclient_config.h` / `CheckedAlignUp` include。

### 9.5 DFS 可用性门

`master_service.cpp:5154-5156`：

```cpp
if (config.dfs_replica_num > 0 &&
    (!enable_dfs_ ||
     (!use_kvcs_ && !use_kvclient_ &&
      (!dfs_allocator_ || !dfs_allocator_->IsInitialized())))) {
    // 保持现有返回码，不要改成 INVALID_PARAMS。
    return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);
}
```

### 9.6 NOF 拒绝

在 `PrepareWriteStart`（`:5129` 起）内，放在 `dfs_replica_num` / default tenant 校验之后：

```cpp
if (config.dfs_replica_num > 0 && use_kvclient_) {
    if (config.nof_replica_num > 0) {
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
}
```

约束：

- 适配器与 Master 都不预检请求参数；参数合法性由 SDK 判定并返回错误。
- NOF 拒绝必须无条件，不能只依赖 `#ifndef USE_NOF` 分支。
- `PrepareWriteStart` 覆盖 `PutStart`、`UpsertStart`、`BatchPutStart`。

### 9.7 驱逐白名单

`master_service.cpp:6335-6337`：

```cpp
const bool supports_dfs_condition =
    replica_type == ReplicaType::DFS &&
    expected_replica_id != nullptr &&
    (use_kvcs_ || use_kvclient_);
```

`master_service.cpp:6353-6355`：

```cpp
} else if (replica_type == ReplicaType::DFS) {
    return (use_kvcs_ || use_kvclient_) &&
           expected_replica_id != nullptr &&
           r.is_dfs_replica() && r.id() == *expected_replica_id;
}
```

注意：`EvictDiskReplica` / `BatchEvictDiskReplica` 传 `nullptr`，DFS 仍会被拒；PR2 不改变该行为。

### 9.8 恢复门保持 KVCS-only

`master_service.cpp:3434`、`:3726`、`:4014` **不改**，并加一行注释说明原因：

```cpp
// kvclient does not support metadata recovery; enabling it with oplog is
// rejected at startup, so these gates stay KVCS-only.
```

`GetStorageUsageSnapshot`（`:1046-1061`）对 kvclient 自然返回空 DFS usage；不需要新增分支。

---

## 10. 步骤 9：测试

### 10.1 测试目标

`mooncake-store/tests/CMakeLists.txt`，放在 `kvcs_object_storage_adapter_test` 之后：

```cmake
if(MOONCAKE_KVCLIENT_SDK_FOUND)
  add_store_test(kvclient_object_storage_adapter_test
                 kvclient_object_storage_adapter_test.cpp)
  target_compile_definitions(kvclient_object_storage_adapter_test
                             PRIVATE MOONCAKE_KVCLIENT_TEST_SDK=1)
  target_link_libraries(kvclient_object_storage_adapter_test
                        PRIVATE Mooncake::kvclient Mooncake::xxhash)
endif()
```

`add_store_test` 已链接 `mooncake_store`（其中包含适配器 TU）；`PkgConfig::KVCLIENT` 用于 SDK 头路径，`Mooncake::xxhash` 用于断言 key 编码。

### 10.2 fake client

在测试文件中实现 `kv::KvClient` 的测试替身，记录调用并注入结果：

```cpp
class FakeKvClient final : public kv::KvClient {
   public:
    kv::Status Init(const kv::KvClientConfig&) override { return {}; }
    kv::Status Shutdown() override { return {}; }
    kv::Status QueryAsync(const std::vector<kv::CacheKey>&, kv::TaskId&,
                          const kv::IoOptions&) override;
    kv::Status LoadAsync(const std::vector<kv::KVBuffer>& entries,
                         kv::TaskId& task_id,
                         const kv::IoOptions&) override;
    kv::Status StoreAsync(const std::vector<kv::KVBuffer>& entries,
                          kv::TaskId& task_id,
                          const kv::IoOptions&) override;
    kv::Status DeleteAsync(const std::vector<kv::CacheKey>&, kv::TaskId&,
                           const kv::IoOptions&) override;
    bool Check(kv::TaskId) override { return true; }
    kv::Status Wait(kv::TaskId, std::uint64_t, kv::TaskResult&) override;
    kv::Status RegisterRegions(
        const std::vector<kv::MemoryRegion>& regions,
        std::vector<kv::RegisteredMemory>& out) override;
    kv::Status UnregisterRegions(
        const std::vector<kv::MRHandle>&) override;

    // 记录：registered_regions_、store_entries_、unregistered_handles_、
    // load_entries_、delete_calls_，以及可配置的 submit_status_ /
    // wait_status_ / register_status_ / entry_status_。
};
```

`RegisterRegions` 返回按输入顺序、一一对应的 handle；`Wait` 默认返回 `OK` 且 `entryStatus` 全 OK。

### 10.3 用例清单

| 用例 | 断言 |
| --- | --- |
| `MakeCacheKeyIsStableAndLittleEndian` | 通过 `DeleteBatch` 捕获 key；与 `XXH64(seed=5381)` 逐字节小端结果一致；同一 logical key 两次调用相同 |
| `RegisterBuffersRegistersAndReleases` | `PutBatch` 对每个参与项调用一次 `RegisterRegions`；返回前对每个 handle 调用一次 `UnregisterRegions`（fake 记录调用次数与顺序） |
| `RejectsEmptyKeyZeroLengthAndDuplicate` | 空 key / `size==0` / 空指针逐项 `INVALID_PARAMS`；同批重复 key 第一条成功、后续 `INVALID_PARAMS` |
| `RegisterFailureIsolatesEntry` | fake `RegisterRegions` 返回 `PARTIAL_FAILED`：该项 `DFS_SERVICE_UNAVAILABLE`，不提交且不占用 key，其余项正常 `StoreAsync` |
| `SingleEntryUsesBatchPath` | `Put` 转成单 iovec 调 `PutV`；`PutV` 走单元素 `PutBatch`：fake 观察到一次 `RegisterRegions` / `StoreAsync` / `UnregisterRegions`；空 `span` → `INVALID_PARAMS` |
| `WriteTaskErrorMapsAll` | fake `Wait` 返回 task 级错误：参与项统一映射该错误 |
| `ReadSucceeds` | `GetBatch` 提交 `LoadAsync` 并 `Wait`；成功项返回 N |
| `ReadSubmitFailure` | `LoadAsync` 提交失败：逐项错误；调用方 buffer 未被写 |
| `ReadTaskErrorMapsAll` | fake `Wait` 返回 task 级错误：逐项全部映射该错误 |
| `ExistsReturnsEntryCode` | `QueryAsync` 的 entry 为 `ENTRY_KEY_EXIST` / `ENTRY_KEY_NOT_EXIST` → `true` / `false`；不使用 `ok()` |
| `ExistsErrors` | 空 key → `INVALID_PARAMS`（零 SDK 调用）；提交失败/entry 错误 → 对应 `ErrorCode` |
| `OverwriteDoesNotPreDelete` | 两次 `PutBatch` 同一 key：调用序列中没有 `DeleteAsync`；两次都 `StoreAsync` |
| `UninitializedBatchFailsSafely` | 未 `Init` 时 `PutBatch`/`GetBatch` 返回 `DFS_SERVICE_UNAVAILABLE`，SDK 方法零调用 |
| `UnregisterFailureIsLogged` | fake `UnregisterRegions` 返回错误：`PutBatch` 仍返回逐项结果，适配器只记录告警、不重试 |

### 10.4 测试环境约束

- 测试只用 fake client，不需要真实设备或 view service。
- 注入构造 `KvClientObjectStorageAdapter(config, std::make_unique<FakeKvClient>())` 后调用 `Init()`：注入路径只校验配置、不调用 SDK `Init`。
- 零拷贝用例的 buffer 用普通 `std::vector<char>` 即可；参数是否可接受由 fake SDK 决定。
- 读结果用例通过 fake `Wait` 注入 `OK` / task 级错误；断言 `GetBatch` 返回前对每个已注册 handle 调用一次 `UnregisterRegions`。
- `RegisterBuffers` / `UnregisterBuffers` 是私有成员，测试不直接调用，而是通过 `PutBatch` / `GetBatch` 驱动，并检查 fake client 记录的 `RegisterRegions` / `UnregisterRegions` 调用次数与顺序。

---

## 11. 步骤 10：构建与验证

### 11.1 SDK 存在构建

```bash
PKG_CONFIG_PATH=/path/to/kvclient/lib/pkgconfig \
cmake -S . -B build-sdk -DWITH_STORE=ON -DBUILD_UNIT_TESTS=ON
cmake --build build-sdk -j"$(nproc)"
ctest --test-dir build-sdk --output-on-failure \
      -R "kvclient_object_storage_adapter_test|distributed_storage_config_test"
```

### 11.2 SDK 缺失构建

```bash
cmake -S . -B build -DWITH_STORE=ON -DBUILD_UNIT_TESTS=ON
cmake --build build -j"$(nproc)"
ctest --test-dir build --output-on-failure
```

必须确认：

- 适配器 TU 与 kvclient 测试均不参与编译；
- `mooncake_store_master` 不链接 KVCLIENT；
- 既有后端与 DFS 测试全绿。

### 11.3 启动验收

| 场景 | 预期 |
| --- | --- |
| `fs_adapter_type=kvclient` + `enable_oplog=false` | Master 正常启动，日志出现 `KVClient distributed object storage initialized` |
| `fs_adapter_type=kvclient` + `enable_oplog=true` | Master 启动期 `LOG(FATAL)`，不进入运行态 |
| `dfs_replica_num=1` 写 | 铸造 `object_size=N` / `aligned_size=N` / `shard_idx=-1` 的 DFS descriptor |
| `dfs_replica_num=1` 且 `nof_replica_num>0` | `INVALID_PARAMS` |
| `EvictDiskReplicaIfCurrent(ReplicaType::DFS)` | 白名单放行；`EvictDiskReplica` 仍返回 `INVALID_PARAMS` |

### 11.4 端到端联调说明

PR2 不改 `AttachHostId`，因此调用方必须显式传 `dfs_replica_num=1` 才会铸造 DFS descriptor；否则只写内存副本。端到端手工验证时：

```bash
MOONCAKE_DFS_FS_ADAPTER=kvclient \
MOONCAKE_KVCLIENT_CONFIG=/path/to/kvclient.conf \
# 写/读都零拷贝；buffer 由后端预注册、调用返回后注销
```

### 11.5 提交前格式检查

```bash
pre-commit run --files <本次改动文件>
```

不要使用 `pre-commit run --all-files`。

---

## 12. 步骤 11：PR 提交

- 标题：`[Store] Implement kvclient batch data path and master gates`
- PR 描述必须写清：
  - 写路径零拷贝；适配器不做请求校验、不拷贝、不 staging，成功由 KvClient 判定；
  - 读路径由适配器内部注册目标 buffer，`GetBatch` 内 `LoadAsync` + `waitForCompletion`；返回前注销，buffer 随后可复用；
  - 超时、参数校验与设备错误由 KvClient SDK 负责；
  - `Exists` 已实现（`QueryAsync` + entry 级 code）；`ListKeys` 仍为 `NOT_SUPPORTED`；
  - kvclient 不支持 snapshot/oplog，`enable_oplog=true` 时启动期拒绝；
  - 恢复门保持 KVCS-only；
  - 端到端写需要调用方显式 `dfs_replica_num=1`（`AttachHostId` 在 PR3）。
- 使用 `.github/pull_request_template.md`，填写模块、类型、测试、检查清单、AI 协助披露。
- 人工逐行审阅所有改动。

---

## 13. 验收清单

| 项 | 通过标准 |
| --- | --- |
| 写路径 | 零拷贝提交；适配器内按请求 `RegisterBuffers` 并只用返回的 handle |
| 写失败判定 | SDK 返回的错误按 `ToErrorCode` 映射；适配器不做请求校验 |
| 读路径 | `GetBatch` 内注册目标 buffer → `LoadAsync` + `waitForCompletion`，成功项返回 N |
| `Exists` | `QueryAsync` 的 `ENTRY_KEY_EXIST` / `ENTRY_KEY_NOT_EXIST` → `true` / `false`；错误正确映射；`ListKeys` 仍 `NOT_SUPPORTED` |
| buffer handle | `RegisterBuffers` 在 batch 内注册、返回前用 `UnregisterBuffers` 逐个注销；上层接口不变 |
| 覆盖写 | 不产生 `DeleteAsync` |
| 守卫 | Init 前调用逐项 `DFS_SERVICE_UNAVAILABLE` |
| 配置 | 只有 `config_path`，不校验数值配置 |
| Master 启动 | kvclient 早退分支生效；oplog 时 FATAL |
| descriptor | `object_size == aligned_size == N`；`shard_idx=-1` |
| Master 门控 | NOF 拒绝、DFS 可用性门、驱逐白名单生效 |
| 恢复门 | 保持 KVCS-only |
| 测试 | 13 个用例全部通过；SDK 缺失构建不受影响 |
| 回归 | 既有 KVCS / OSS / S3 / DFS 测试全绿 |

---

## 14. 实现约束速查

1. **写路径**：零拷贝；不 staging、不补齐、不校验；成功由 KvClient 判定。
2. **读路径**：`GetBatch` 内由 `RegisterBuffers` 注册目标 buffer；`LoadAsync` + `waitForCompletion`；返回前注销，随后即可复用 buffer。
3. **buffer 注册**：适配器内部 `RegisterBuffers` 调用 SDK 注册接口；不改请求结构或上层。
4. **解除注册**：`UnregisterBuffers` 在 `PutBatch` / `GetBatch` 返回前逐个 `UnregisterRegions`；SDK 不会在任务完成后自动释放；注销失败只告警。
5. **任务级错误**：由 SDK 返回，统一映射到参与项；适配器不设超时状态。
6. **buffer 生命周期**：调用进行期间不得复用；`PutBatch` / `GetBatch` 返回前注销 handle。
7. **覆盖写**：不预删；不引入 `DeleteAsync` + `StoreAsync`。
8. **守卫**：`PutBatch` / `GetBatch` 入口检查 `initialized_` / `kv_client_`；注册失败按项返回错误。
9. **析构**：只调用 SDK `Shutdown()`。
10. **Master 初始化**：kvclient 早退分支必须在 filesystem allocator 之前。
11. **descriptor**：与 KVCS 相同；`aligned_size = value_length`。
12. **错误码**：DFS 可用性门保持 `DFS_SERVICE_UNAVAILABLE`。
13. **恢复门**：不改，保持 KVCS-only。
14. **配置**：只保留 `config_path`；不校验数值配置。
15. **`Exists`**：`QueryAsync` + `Wait`，按 entry 级 `ENTRY_KEY_EXIST` / `ENTRY_KEY_NOT_EXIST` 返回；不能用 `ok()`；不注册 buffer。

---

## 15. 已知风险与待确认

| 风险 | 处理 |
| --- | --- |
| 调用返回前 buffer 不得复用 | `UnregisterBuffers` 在 `PutBatch` / `GetBatch` 返回前注销 handle；调用方在返回后才可复用 buffer；PR 评审重点 |
| 超时后提前注销 | `Wait` 返回非终态时 SDK 不保证 I/O 已结束，但 `UnregisterBuffers` 仍会注销；SDK 未提供可等待的完成信号，按 §2.1.1 决策接受 |
| 异常路径 | `PutBatch` 用 try/catch 包住注册与提交，catch 中先 `UnregisterBuffers` 再 rethrow；不依赖 RAII |
| XXH64 非单射 | 已接受；日志记录 `cache_key` |
| `CheckHealth` 未实现 | `MOONCAKE_DISTRIBUTED_HEALTH_CHECK=1` 时 Init 失败；部署必须关闭 |
| 端到端写依赖调用方显式 `dfs_replica_num=1` | PR3 的 `AttachHostId` 才自动补 |
| `EvictDiskReplica` / `BatchEvictDiskReplica` 的 DFS 分支不可用 | PR2 明确不改；只验证 `EvictDiskReplicaIfCurrent` |
