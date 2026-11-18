# Mooncake + KVClient L3 接入（Kvcs）— PR2 实施步骤

版本：v1.0　日期：2026-11-18

对应设计文档：`Mooncake + KVClient L3 接入（Kvcs）- PR 实施设计.md`（v1.3，PR2 章节）

代码基线：`Mooncake` 分支 `Mooncake_l3_pr2`，HEAD `21d0c268`；SDK 基线 `tmp_pr1641/ucm-pr`（含 `BinaryKey` / `CacheKey::FromBytes` / `keyResolution` / `IoOptions`）

---

## 0. 前提与决策

### 0.1 PR2 边界

PR2 在 PR1 的控制面之上补齐数据面与 Master 门控：

- 适配器：`PutBatch` / `GetBatch` 批读写；`Put` / `PutV` / `Get` 单条包装；直传/暂存；按批注册与释放；超时 orphan 保留。
- 接口层：四个请求结构追加 `capacity`，后端透传。
- Master：`use_kvclient_`、`enable_dfs_`、kvclient 早退分支、descriptor 铸造、长度/NOF 校验、驱逐白名单。
- 测试：PR2 自带适配器单测。

PR2 不做：

- Client 侧 value-first 删除、`ReconcileMissingKvReplica` 改名与放宽、`AttachHostId`（PR3）。
- `Exists` / `ListKeys`：保持 `NOT_SUPPORTED`（与 KVCS 一致；SDK 没有 listing 契约）。
- `CheckHealth` / `GetRange` / `PutOnce` / `PutVOnce`：保持基类默认。
- 上游 `client_service.cpp` / `file_storage.cpp` 的 capacity 传递（第一阶段不启用直传）。

### 0.2 已确认决策（2026-11-18）

| 决策项 | 结论 | 落地要求 |
| --- | --- | --- |
| capacity 策略 | 所有现有调用点一律传 0；直传只在适配器单测中构造 | 不改 `client_service.cpp` / `file_storage.cpp`；`BatchRead` 多 slice 暂存时 capacity 必须为 0 |
| SDK 覆盖语义 | `StoreAsync` 覆盖写 | `PutBatch` 不预删；测试断言调用序列中没有 `DeleteAsync` |
| kvclient + oplog | 启动期 `LOG(FATAL)` 拒绝 | 在 `InitDfsAllocatorFromEnvironment` 的 kvclient 分支内拒绝；恢复门保持 KVCS-only |
| 测试归属 | PR2 自带 | 新增 `kvclient_object_storage_adapter_test.cpp`，仅 SDK 存在时编译 |

### 0.3 实施前必须修正的旧文档问题

| # | 旧文档写法 | 正确做法 |
| --- | --- | --- |
| 1 | §2.6.1 重新声明命名空间级 `Completion` | `Completion` 已是类内私有嵌套类型，PR2 不再声明 |
| 2 | §2.3 写 `uint64 handle`、`vector<Orphan> orphans_` | `kv::MRHandle handle`；`vector<pair<shared_ptr<char[]>, kv::MRHandle>> orphans_` |
| 3 | 只有 `PutBatch` 伪代码 | 必须补 `GetBatch`、`RegisterBuffers`、`ReleaseBuffers`、`RetainOrphans`、`AllocateStaging` |
| 4 | `AllocateStaging` 只说"512B 对齐" | 必须用 aligned new；普通 `new char[]` 不保证 512 |
| 5 | `PutBatch` 无初始化守卫 | 入口检查 `!initialized_ \|\| !kv_client_`，逐项 `DFS_SERVICE_UNAVAILABLE` |
| 6 | 非终态逐项 `ToErrorCode(entryStatus)` | 非终态时全批统一 `RPC_TIMEOUT`，禁止出现 `OBJECT_NOT_FOUND` |
| 7 | DFS 可用性门返回 `INVALID_PARAMS` | 现有实现返回 `DFS_SERVICE_UNAVAILABLE`，保持 |
| 8 | Master 初始化只写三行赋值 | 必须新增 kvclient 早退分支，放在 filesystem allocator 初始化之前 |
| 9 | 未提恢复门 | `3434` / `3726` / `4014` 保持 KVCS-only，不并入 `use_kvclient_` |
| 10 | `aligned_size = AlignUp(...)` 与 KVCS 共用分支 | 拆子分支；KVCS 的 `aligned_size = value_length` 不变 |

### 0.4 分支

```bash
git switch Mooncake_l3_pr2   # 已从 Mooncake_l3_pr1 创建
```

PR3 分支 `Mooncake_l3_pr3` 已基于 PR2 创建，PR2 合入后再开始。

---

## 1. 实施顺序总览

| 步骤 | 内容 | 主要文件 | 验证点 |
| --- | --- | --- | --- |
| 1 | 接口层 capacity | `object_storage_adapter.h`、`distributed_storage_backend.h/.cpp` | 既有聚合初始化仍编译 |
| 2 | 适配器头 | `kvclient_object_storage_adapter.h` | 类可实例化 |
| 3 | helper | `kvclient_object_storage_adapter.cpp` | 对齐、溢出、iov 校验 |
| 4 | 注册 | 同上 | 去重、整批失败、handle 回填 |
| 5 | `PutBatch` | 同上 | 提交/等待/映射/释放 |
| 6 | `GetBatch` | 同上 | 暂存回拷、返回 N |
| 7 | 单条包装与析构 | 同上 | 守卫、orphan 清理顺序 |
| 8 | Master 门控 | `master_service.h/.cpp` | 启动、descriptor、白名单 |
| 9 | 测试 | `tests/kvclient_object_storage_adapter_test.cpp`、`tests/CMakeLists.txt` | 10 个用例 |
| 10 | 构建验证 | CMake / ctest | SDK 缺失与存在两种构建 |
| 11 | PR 提交 | PR 模板 | 边界写清 |

---

## 2. 步骤 1：接口层 capacity

### 2.1 四个结构末尾追加

`mooncake-store/include/storage/distributed/object_storage_adapter.h`：

```cpp
struct ObjectPutRequest {
    std::string logical_key;
    const iovec* iov = nullptr;
    int iovcnt = 0;
    size_t capacity = 0;   // 新增；调用方缓冲区容量，未知时为 0
};

struct ObjectGetRequest {
    std::string logical_key;
    void* buffer = nullptr;
    size_t size = 0;
    size_t capacity = 0;   // 新增
};
```

`mooncake-store/include/storage/distributed/distributed_storage_backend.h`：

```cpp
struct DfsWriteRequest {
    std::string key;
    DistributedFSDescriptor descriptor;
    std::vector<Slice> slices;
    size_t capacity = 0;   // 新增，必须放末尾
};

struct DfsReadRequest {
    std::string key;
    DistributedFSDescriptor descriptor;
    std::vector<Slice> slices;
    uint64_t value_offset = 0;
    bool range_read = false;
    size_t capacity = 0;   // 新增，必须放 range_read 之后
};
```

约束：

- **必须追加在末尾**。现有代码全部是位置初始化，例如 `client_service.cpp:5484-5488` 的 `{keys[i], descriptor, {windows[i]}, value_offsets[i], true}`；插到中间会静默错位。
- 四个结构都没有 `YLT_REFL` / struct_pack 反射，不涉及 RPC 兼容。
- 建议给 `ObjectStorageAdapter::GetBatch` 的注释补一句：返回值必须是逻辑长度 N，不是 `wire_len`。

### 2.2 两个透传点

`mooncake-store/src/storage/distributed/distributed_storage_backend.cpp`：

写路径对象分支（约 `:686-687`）：

```cpp
object_requests.push_back({request.key, iovs.data(),
                           static_cast<int>(iovs.size()),
                           request.capacity});
```

读路径对象分支（约 `:809-818`）：

```cpp
const bool multi_slice = request.slices.size() > 1;
void* buffer = multi_slice ? staging_buffers.back().data()
                           : request.slices.front().ptr;
// 多 slice 时 buffer 是后端自有的 object_size 暂存，容量就是 object_size，
// 不能透传 slices 总和，否则适配器可能按 wire_len 写穿。
const size_t buffer_capacity = multi_slice ? 0 : request.capacity;
object_requests.push_back({request.key, buffer,
                           static_cast<size_t>(desc.object_size),
                           buffer_capacity});
```

`BatchOffload`（`:433-434`）与 `BatchLoad`（`:1011`）保持默认 0，不改。

约束：

- 写路径的 `DfsWriteRequest.capacity` 上游永远是 0（PR2 不改 `client_service.cpp`），因为 `Slice` 没有容量字段、且写路径强制 `sum(slices) == desc.object_size`。
- 读路径单 slice 时上游也是 0；多 slice 时后端已经暂存过一次，适配器再暂存一层是预期行为。
- 不要用 `desc.aligned_size` 当 capacity：它描述远端 wire 布局，不是调用方缓冲区大小；写侧会因此越界清零。

---

## 3. 步骤 2：适配器头文件

文件：`mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h`

### 3.1 include

现有 include 只有 `<memory>/<string>/<string_view>/<vector>`，需要补：

```cpp
#include <cstdint>
#include <mutex>
#include <optional>
#include <span>
#include <utility>
```

### 3.2 私有成员与声明

```cpp
   private:
    struct Completion {
        bool terminal = false;
        std::vector<kv::Status> entryStatus;
    };

    struct BufferPlan {
        uintptr_t addr = 0;
        size_t n = 0;
        size_t wire_len = 0;
        bool staged = false;
        std::shared_ptr<char[]> staging;
        kv::MRHandle handle = kv::kInvalidMRHandle;
    };

    kv::BinaryKey MakeCacheKey(std::string_view logical_key) const;
    Completion waitForCompletion(kv::TaskId task_id, size_t entry_count);
    static ErrorCode ToErrorCode(const kv::Status& status);

    template <typename RequestT, typename Extract>
    tl::expected<void, ErrorCode> RegisterBatchImpl(
        std::span<const RequestT> requests,
        std::vector<std::optional<BufferPlan>>& plans,
        std::span<ErrorCode> errors, bool is_store, Extract&& extract);

    tl::expected<void, ErrorCode> RegisterBuffers(
        std::span<const ObjectPutRequest> requests,
        std::vector<std::optional<BufferPlan>>& plans,
        std::span<ErrorCode> errors);
    tl::expected<void, ErrorCode> RegisterBuffers(
        std::span<const ObjectGetRequest> requests,
        std::vector<std::optional<BufferPlan>>& plans,
        std::span<ErrorCode> errors);

    void ReleaseBuffers(std::span<const std::optional<BufferPlan>> plans);
    void RetainOrphans(std::span<std::optional<BufferPlan>> plans);

    KvClientAdapterConfig config_;
    std::unique_ptr<kv::KvClient> kv_client_;
    bool initialized_ = false;

    std::mutex orphan_mu_;
    // 只保存 staging 与 handle；handle 由 SDK Shutdown 统一释放。
    std::vector<std::pair<std::shared_ptr<char[]>, kv::MRHandle>> orphans_;
```

约束：

- `Completion`、`waitForCompletion`、`MakeCacheKey`、`ToErrorCode`、构造函数与 `config_/kv_client_/initialized_` 在 PR1 已存在，**不要重复声明**。
- `BufferPlan` 必须是类内私有嵌套类型，因为 `RegisterBuffers` 的签名要用它。
- `RegisterBatchImpl` 的模板定义只放在 `.cpp`，仅在该 TU 实例化。
- `orphans_` 的类型必须与 `RetainOrphans` 一致；不要引入未定义的 `Orphan`。

---

## 4. 步骤 3：文件内 helper

文件：`mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp`

新增 include：`<algorithm>`、`<cstring>`、`<map>`、`<new>`、`<optional>`、`<set>`、`<span>`、`<utility>`。

匿名命名空间：

```cpp
struct BufferView {
    iovec single{};          // 读路径单段 view 落在自身
    const iovec* iov = nullptr;
    int iovcnt = 0;
    size_t n = 0;
    size_t capacity = 0;
};

size_t AlignUp(size_t n, size_t alignment) {
    return (n + alignment - 1) & ~(alignment - 1);
}

// 逐段校验并求和：任一段为空指针/零长度、或累加溢出，返回 nullopt。
std::optional<size_t> SumIov(const iovec* iov, int iovcnt) {
    if (iov == nullptr || iovcnt <= 0) return std::nullopt;
    size_t total = 0;
    for (int i = 0; i < iovcnt; ++i) {
        if (iov[i].iov_base == nullptr || iov[i].iov_len == 0) {
            return std::nullopt;
        }
        if (iov[i].iov_len > std::numeric_limits<size_t>::max() - total) {
            return std::nullopt;
        }
        total += iov[i].iov_len;
    }
    return total;
}

bool IsContiguous(const iovec* iov, int iovcnt) {
    for (int i = 0; i + 1 < iovcnt; ++i) {
        const auto end = reinterpret_cast<uintptr_t>(iov[i].iov_base) +
                         iov[i].iov_len;
        if (end != reinterpret_cast<uintptr_t>(iov[i + 1].iov_base)) {
            return false;
        }
    }
    return true;
}

void CopyIovTo(char* dst, const iovec* iov, int iovcnt) {
    size_t offset = 0;
    for (int i = 0; i < iovcnt; ++i) {
        std::memcpy(dst + offset, iov[i].iov_base, iov[i].iov_len);
        offset += iov[i].iov_len;
    }
}

// 512B 对齐分配；普通 new char[] 只保证 max_align_t。
std::shared_ptr<char[]> AllocateStaging(size_t wire_len) {
    char* ptr = new (std::align_val_t(kv::kAlignmentBytes)) char[wire_len];
    return std::shared_ptr<char[]>(ptr, [](char* p) {
        ::operator delete[](p, std::align_val_t(kv::kAlignmentBytes));
    });
}

template <typename T>
void FillError(std::vector<tl::expected<T, ErrorCode>>& results,
               ErrorCode error) {
    for (auto& result : results) result = tl::make_unexpected(error);
}
```

约束：

- `AlignUp` 之前必须先排除 `n > SIZE_MAX - (alignment - 1)` 的溢出；实际由 `SumIov` 的溢出检查 + `wire_len` 上限检查兜住，但 `make_plan` 里应显式判断。
- 现有 `Errors<T>`（`.cpp:21-28`）在替换占位后成为死代码，改成上面的 `FillError` 或直接删除。
- `kv::kAlignmentBytes` 是 `uint32_t`，与 `size_t` 运算安全。

---

## 5. 步骤 4：注册

### 5.1 `RegisterBatchImpl`

```cpp
template <typename RequestT, typename Extract>
tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::RegisterBatchImpl(
    std::span<const RequestT> requests,
    std::vector<std::optional<BufferPlan>>& plans,
    std::span<ErrorCode> errors, bool is_store, Extract&& extract) {
    plans.assign(requests.size(), std::nullopt);

    std::set<kv::BinaryKey> seen;          // 仅写侧：同批重复 key
    std::vector<kv::MemoryRegion> regions;
    std::map<std::pair<uintptr_t, size_t>, size_t> region_index;
    std::vector<size_t> plan_to_region(requests.size(), SIZE_MAX);

    for (size_t i = 0; i < requests.size(); ++i) {
        if (requests[i].logical_key.empty()) {
            errors[i] = ErrorCode::INVALID_PARAMS;
            continue;
        }
        const auto binary_key = MakeCacheKey(requests[i].logical_key);
        if (is_store && !seen.insert(binary_key).second) {
            errors[i] = ErrorCode::INVALID_PARAMS;   // 第一条受理，后续失败
            continue;
        }

        BufferView view;
        extract(requests[i], view);
        if (view.iov == nullptr || view.iovcnt <= 0 || view.n == 0 ||
            view.iov[0].iov_base == nullptr) {
            errors[i] = ErrorCode::INVALID_PARAMS;
            continue;
        }
        if (view.n > std::numeric_limits<size_t>::max() -
                         (kv::kAlignmentBytes - 1)) {
            errors[i] = ErrorCode::INVALID_PARAMS;
            continue;
        }

        BufferPlan plan;
        plan.n = view.n;
        plan.wire_len = AlignUp(view.n, kv::kAlignmentBytes);
        if (plan.wire_len > config_.max_object_bytes) {
            errors[i] = ErrorCode::INVALID_PARAMS;
            continue;
        }
        plan.addr = reinterpret_cast<uintptr_t>(view.iov[0].iov_base);

        const bool direct =
            view.capacity > 0 && view.capacity >= plan.wire_len &&
            plan.addr % kv::kAlignmentBytes == 0 &&
            IsContiguous(view.iov, view.iovcnt);
        if (!direct) {
            plan.staging = AllocateStaging(plan.wire_len);
            if (!plan.staging) {
                errors[i] = ErrorCode::NO_AVAILABLE_HANDLE;
                continue;
            }
            plan.addr = reinterpret_cast<uintptr_t>(plan.staging.get());
            plan.staged = true;
            if (is_store) CopyIovTo(plan.staging.get(), view.iov, view.iovcnt);
        }
        if (is_store && plan.wire_len > view.n) {
            std::memset(reinterpret_cast<char*>(plan.addr) + view.n, 0,
                        plan.wire_len - view.n);
        }

        const auto region_key = std::make_pair(plan.addr, plan.wire_len);
        auto iter = region_index.find(region_key);
        size_t region;
        if (iter == region_index.end()) {
            region = regions.size();
            regions.push_back({kv::MemoryType::HOST, plan.addr,
                               plan.wire_len, -1, -1});
            region_index.emplace(region_key, region);
        } else {
            region = iter->second;
        }
        plan_to_region[i] = region;
        plans[i] = std::move(plan);
    }

    if (regions.empty()) return {};

    std::vector<kv::RegisteredMemory> registered;
    const auto status = kv_client_->RegisterRegions(regions, registered);
    if (!status.ok() || registered.size() != regions.size()) {
        LOG(ERROR) << "action=kvclient_register_regions_failed, regions="
                   << regions.size() << ", registered=" << registered.size()
                   << ", error=" << status.code;
        // SDK 在失败时自行回滚已注册部分；适配器不再重复注销。
        return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);
    }
    for (size_t i = 0; i < plans.size(); ++i) {
        if (plans[i]) plans[i]->handle = registered[plan_to_region[i]].handle;
    }
    return {};
}
```

### 5.2 两个 `RegisterBuffers` 重载

```cpp
tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::RegisterBuffers(
    std::span<const ObjectPutRequest> requests,
    std::vector<std::optional<BufferPlan>>& plans,
    std::span<ErrorCode> errors) {
    return RegisterBatchImpl(
        requests, plans, errors, /*is_store=*/true,
        [](const ObjectPutRequest& request, BufferView& view) {
            view.iov = request.iov;
            view.iovcnt = request.iovcnt;
            view.capacity = request.capacity;
            view.n = SumIov(request.iov, request.iovcnt).value_or(0);
        });
}

tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::RegisterBuffers(
    std::span<const ObjectGetRequest> requests,
    std::vector<std::optional<BufferPlan>>& plans,
    std::span<ErrorCode> errors) {
    return RegisterBatchImpl(
        requests, plans, errors, /*is_store=*/false,
        [](const ObjectGetRequest& request, BufferView& view) {
            view.single = iovec{request.buffer, request.size};
            view.iov = &view.single;
            view.iovcnt = 1;
            view.n = request.size;
            view.capacity = request.capacity;
        });
}
```

约束：

- `std::span` 不能从 `const std::vector<T>&` 隐式推导成模板实参；在 `PutBatch`/`GetBatch` 里调用时显式传 `std::span<const ObjectPutRequest>(requests)`，或让 `RegisterBuffers` 内部构造 span。
- 读路径只有单段 view；staged 与非 staged 的差别只在 `AllocateStaging` 与回拷。
- 写侧同批重复 key 用 `MakeCacheKey` 判重；哈希碰撞属于已接受风险，不用于正确性保证以外的场景。

---

## 6. 步骤 5：`PutBatch`

```cpp
std::vector<tl::expected<void, ErrorCode>>
KvClientObjectStorageAdapter::PutBatch(
    const std::vector<ObjectPutRequest>& requests) {
    std::vector<tl::expected<void, ErrorCode>> results(requests.size());

    if (!initialized_ || !kv_client_) {
        FillError(results, ErrorCode::DFS_SERVICE_UNAVAILABLE);
        return results;
    }

    std::vector<std::optional<BufferPlan>> plans;
    std::vector<ErrorCode> errors(requests.size(), ErrorCode::INVALID_PARAMS);
    auto registered = RegisterBuffers(
        std::span<const ObjectPutRequest>(requests), plans, errors);
    if (!registered) {
        FillError(results, ErrorCode::DFS_SERVICE_UNAVAILABLE);
        return results;
    }
    for (size_t i = 0; i < plans.size(); ++i) {
        if (!plans[i]) results[i] = tl::make_unexpected(errors[i]);
    }

    std::vector<kv::KVBuffer> entries;
    std::vector<size_t> indices;
    for (size_t i = 0; i < plans.size(); ++i) {
        if (!plans[i]) continue;
        kv::KVBuffer entry;
        entry.key = kv::CacheKey::FromBytes(
            MakeCacheKey(requests[i].logical_key));
        entry.buffer.region = {kv::MemoryType::HOST, plans[i]->addr,
                               plans[i]->wire_len, -1, -1};
        entry.buffer.handle = plans[i]->handle;
        entry.offset = 0;
        entry.mrKey = std::nullopt;      // SDK 按 handle 覆盖
        entries.push_back(std::move(entry));
        indices.push_back(i);
    }
    if (entries.empty()) return results;

    kv::TaskId task_id = kv::kInvalidTaskId;
    const auto submit = kv_client_->StoreAsync(entries, task_id);
    if (!submit.ok()) {
        for (size_t index : indices) {
            results[index] = tl::make_unexpected(ToErrorCode(submit));
        }
        // 任务未入队；注册已发生，必须释放。
        ReleaseBuffers(plans);
        return results;
    }

    const auto completion = waitForCompletion(task_id, entries.size());
    if (completion.entryStatus.size() != entries.size()) {
        for (size_t index : indices) {
            results[index] =
                tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
        }
    } else if (!completion.terminal) {
        // 逐项结果不可信；统一超时，绝不映射成 miss。
        for (size_t index : indices) {
            results[index] = tl::make_unexpected(ErrorCode::RPC_TIMEOUT);
        }
    } else {
        for (size_t j = 0; j < entries.size(); ++j) {
            const auto& status = completion.entryStatus[j];
            results[indices[j]] =
                status.ok() ? tl::expected<void, ErrorCode>{}
                            : tl::make_unexpected(ToErrorCode(status));
        }
    }

    if (completion.terminal) {
        ReleaseBuffers(plans);
    } else {
        RetainOrphans(plans);
        LOG(ERROR) << "action=kvclient_batch_release_skipped, task_id="
                   << task_id << ", entries=" << entries.size();
    }
    return results;
}
```

约束：

- 已确认 `StoreAsync` 覆盖写，因此**不要**加 `DeleteAsync` 预删。
- 提交失败时只覆盖参与批的 `indices`，不要 `FillError` 全量，否则空 key / 重复 key 的 `INVALID_PARAMS` 会被覆盖。
- `entryStatus.size() != entries.size()` 与 `!terminal` 两个分支同样只覆盖参与项。
- `ReleaseBuffers` 只在"任务已终态"或"任务未入队"时调用；非终态必须保留 handle 与 staging。
- 写侧 direct 会原地清零调用方 `[N, wire_len)`；PR2 生产路径 capacity=0 不触发，单测构造 direct 时需显式保证容量。

---

## 7. 步骤 6：`GetBatch`

```cpp
std::vector<tl::expected<size_t, ErrorCode>>
KvClientObjectStorageAdapter::GetBatch(
    const std::vector<ObjectGetRequest>& requests) {
    std::vector<tl::expected<size_t, ErrorCode>> results(requests.size());

    if (!initialized_ || !kv_client_) {
        FillError(results, ErrorCode::DFS_SERVICE_UNAVAILABLE);
        return results;
    }

    std::vector<std::optional<BufferPlan>> plans;
    std::vector<ErrorCode> errors(requests.size(), ErrorCode::INVALID_PARAMS);
    auto registered = RegisterBuffers(
        std::span<const ObjectGetRequest>(requests), plans, errors);
    if (!registered) {
        FillError(results, ErrorCode::DFS_SERVICE_UNAVAILABLE);
        return results;
    }
    for (size_t i = 0; i < plans.size(); ++i) {
        if (!plans[i]) results[i] = tl::make_unexpected(errors[i]);
    }

    std::vector<kv::KVBuffer> entries;
    std::vector<size_t> indices;
    for (size_t i = 0; i < plans.size(); ++i) {
        if (!plans[i]) continue;
        kv::KVBuffer entry;
        entry.key = kv::CacheKey::FromBytes(
            MakeCacheKey(requests[i].logical_key));
        entry.buffer.region = {kv::MemoryType::HOST, plans[i]->addr,
                               plans[i]->wire_len, -1, -1};
        entry.buffer.handle = plans[i]->handle;
        entry.offset = 0;
        entry.mrKey = std::nullopt;
        entries.push_back(std::move(entry));
        indices.push_back(i);
    }
    if (entries.empty()) return results;

    kv::TaskId task_id = kv::kInvalidTaskId;
    const auto submit = kv_client_->LoadAsync(entries, task_id);
    if (!submit.ok()) {
        for (size_t index : indices) {
            results[index] = tl::make_unexpected(ToErrorCode(submit));
        }
        ReleaseBuffers(plans);
        return results;
    }

    const auto completion = waitForCompletion(task_id, entries.size());
    if (completion.entryStatus.size() != entries.size()) {
        for (size_t index : indices) {
            results[index] = tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
        }
    } else if (!completion.terminal) {
        for (size_t index : indices) {
            results[index] = tl::make_unexpected(ErrorCode::RPC_TIMEOUT);
        }
    } else {
        for (size_t j = 0; j < entries.size(); ++j) {
            const size_t index = indices[j];
            const auto& status = completion.entryStatus[j];
            if (!status.ok()) {
                results[index] = tl::make_unexpected(ToErrorCode(status));
                continue;
            }
            const auto& plan = *plans[index];
            if (plan.staged) {
                std::memcpy(requests[index].buffer, plan.staging.get(),
                            plan.n);
            }
            results[index] = plan.n;   // 必须是 N，不是 wire_len
        }
    }

    if (completion.terminal) {
        ReleaseBuffers(plans);
    } else {
        RetainOrphans(plans);
        LOG(ERROR) << "action=kvclient_batch_release_skipped, task_id="
                   << task_id << ", entries=" << entries.size();
    }
    return results;
}
```

约束：

- 返回值必须是 `request.size`（N）。Backend 在 `distributed_storage_backend.cpp:838` 与 `:1019-1026` 按 N 校验，返回 `wire_len` 会让所有未对齐对象报 `FILE_READ_FAIL`。
- staged 时只回拷 N 字节；staging 的 `[N, wire_len)` 是补齐区，不含有效数据。
- 存储侧对象长度是 `wire_len`；客户端读到 `wire_len` 后只暴露前 N 字节，长度校验由 Backend 的第二层比对完成。

---

## 8. 步骤 7：单条包装、释放、孤儿、析构

### 8.1 单条入口

```cpp
tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::Put(
    const std::string& key, std::span<const char> data) {
    iovec iov{const_cast<char*>(data.data()), data.size()};
    auto results = PutBatch({{key, &iov, 1, 0}});
    return results.empty()
               ? tl::make_unexpected(ErrorCode::INVALID_PARAMS)
               : std::move(results.front());
}

tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::PutV(
    const std::string& key, const iovec* iov, int iovcnt) {
    auto results = PutBatch({{key, iov, iovcnt, 0}});
    return results.empty()
               ? tl::make_unexpected(ErrorCode::INVALID_PARAMS)
               : std::move(results.front());
}

tl::expected<size_t, ErrorCode> KvClientObjectStorageAdapter::Get(
    const std::string& key, void* buf, size_t len) {
    auto results = GetBatch({{key, buf, len, 0}});
    return results.empty()
               ? tl::make_unexpected(ErrorCode::INVALID_PARAMS)
               : std::move(results.front());
}
```

注意：空 `data` / `len==0` 会因 `n==0` 返回 `INVALID_PARAMS`，与 KVCS 一致。

### 8.2 `ReleaseBuffers`

```cpp
void KvClientObjectStorageAdapter::ReleaseBuffers(
    std::span<const std::optional<BufferPlan>> plans) {
    std::vector<kv::MRHandle> handles;
    handles.reserve(plans.size());
    for (const auto& plan : plans) {
        if (plan && plan->handle != kv::kInvalidMRHandle) {
            handles.push_back(plan->handle);
        }
    }
    std::sort(handles.begin(), handles.end());
    handles.erase(std::unique(handles.begin(), handles.end()), handles.end());
    if (handles.empty()) return;

    const auto status = kv_client_->UnregisterRegions(handles);
    if (!status.ok()) {
        LOG(ERROR) << "action=kvclient_batch_unregister_failed, handles="
                   << handles.size() << ", error=" << status.code;
    }
}
```

### 8.3 `RetainOrphans`

```cpp
void KvClientObjectStorageAdapter::RetainOrphans(
    std::span<std::optional<BufferPlan>> plans) {
    std::lock_guard<std::mutex> lock(orphan_mu_);
    for (auto& plan : plans) {
        if (!plan || plan->handle == kv::kInvalidMRHandle) continue;
        orphans_.emplace_back(std::move(plan->staging), plan->handle);
    }
    LOG(ERROR) << "action=kvclient_orphans_retained, total=" << orphans_.size();
}
```

约束：

- 不去重：同一 handle 可能被多个 plan 共享；析构时不逐个注销、只清内存，重复项无害。
- 无界增长风险已知：每次超时都可能新增；PR2 先记录 ERROR 日志，指标与上限留后续 PR。
- `orphans_` 只在 `RetainOrphans` 与析构中访问，`orphan_mu_` 保护这两处。

### 8.4 析构

```cpp
KvClientObjectStorageAdapter::~KvClientObjectStorageAdapter() {
    // 先停止受理并让 SDK 排空在途任务，再释放 staging。
    if (kv_client_) kv_client_->Shutdown();

    std::lock_guard<std::mutex> lock(orphan_mu_);
    if (!orphans_.empty()) {
        LOG(ERROR) << "action=kvclient_orphans_released_at_shutdown, count="
                   << orphans_.size();
    }
    orphans_.clear();
}
```

顺序不能反：先清 `orphans_` 再 `Shutdown()` 会让迟到写命中已释放的 staging。

### 8.5 保持 `NOT_SUPPORTED`

- `Exists`、`ListKeys` 保持原占位；不要用 `Get` 探测存在性。
- `CheckHealth`、`GetRange`、`PutOnce`、`PutVOnce` 不开 override。

---

## 9. 步骤 8：Master 门控

文件：`mooncake-store/include/master_service.h`、`mooncake-store/src/master_service.cpp`

### 9.1 成员

`master_service.h:2034` 旁新增：

```cpp
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
    // 对象存储：不创建 EFC target / capacity provider / 水位淘汰。
    LOG(INFO) << "KVClient distributed object storage initialized, config={"
              << dfs_config.FormatStr() << "}";
    return;
}
```

不做这一步的后果：

- `enable_oplog_=true` 时落到 `:923-928` 的 `throw`，Master 起不来；
- `enable_oplog_=false` 时落到 `:930-951` 初始化 `ShardAllocator`，对 object storage 必然失败，`enable_dfs_` 被重置为 false，所有 DFS 写返回 `DFS_SERVICE_UNAVAILABLE`。

### 9.4 descriptor 铸造

`master_service.cpp:4995-5003`，`AllocateReplicas` 内：

```cpp
if (config.dfs_replica_num > 0) {
    if (use_kvcs_) {
        // 不变：KVCS 的 aligned_size 仍是 value_length
        DistributedFSDescriptor descriptor;
        descriptor.object_size = value_length;
        descriptor.aligned_size = value_length;
        descriptor.shard_idx = -1;
        replicas.emplace_back(std::move(descriptor),
                              ReplicaStatus::PROCESSING);
    } else if (use_kvclient_) {
        const auto aligned =
            CheckedAlignUp(value_length, kKvClientAlignment);
        if (!aligned) return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
        DistributedFSDescriptor descriptor;
        descriptor.object_size = value_length;
        descriptor.aligned_size = *aligned;
        descriptor.shard_idx = -1;
        replicas.emplace_back(std::move(descriptor),
                              ReplicaStatus::PROCESSING);
    } else {
        // 原 filesystem allocator 分支（判空、分配、失败返回）保持不变
    }
}
```

需要新增 include：

```cpp
#include "storage/distributed/bucket_entry_layout.h"   // CheckedAlignUp
#include "storage/distributed/kvclient/kvclient_config.h"  // 常量
```

`kvclient_config.h` 是 SDK-free 头，Master 可以包含。

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

### 9.6 长度上限与 NOF 拒绝

在 `PrepareWriteStart`（`:5129` 起）内，放在 `dfs_replica_num` / default tenant 校验之后：

```cpp
if (config.dfs_replica_num > 0 && use_kvclient_) {
    // 16,776,704 已 512 对齐，因此 N <= 上限等价于 AlignUp(N,512) <= 上限。
    if (slice_length > kKvClientMaxValueBytes) {
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    if (config.nof_replica_num > 0) {
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
}
```

约束：

- 上限用 `kKvClientMaxValueBytes`，不要写字面量；Master 看不到 Client 进程的 `MOONCAKE_KVCLIENT_MAX_OBJECT_BYTES`，所以必须用协议上限。
- NOF 拒绝必须**无条件**，不能只依赖 `#ifndef USE_NOF` 分支；`-DUSE_NOF=ON` 构建下同样成立。
- `PrepareWriteStart` 覆盖 `PutStart`、`UpsertStart`、`BatchPutStart`，不需要在 Upsert 单独加门。

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
                        PRIVATE PkgConfig::KVCLIENT Mooncake::xxhash)
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

`RegisterRegions` 返回按输入顺序、一一对应的 handle；`Wait` 默认返回 `OK` 且 `entryStatus` 全 OK。`LoadAsync` 在返回前把测试模式写进每个 entry 的 region（模拟设备读取）。

### 10.3 用例清单

| 用例 | 断言 |
| --- | --- |
| `MakeCacheKeyIsStableAndLittleEndian` | 通过 `DeleteBatch` 捕获 key；与 `XXH64(seed=5381)` 逐字节小端结果一致；同一 logical key 两次调用相同 |
| `RejectsEmptyKeyZeroLengthAndDuplicate` | 空 key / `size==0` 逐项 `INVALID_PARAMS`；同批重复 key 第一条成功、后续 `INVALID_PARAMS`；`StoreAsync` 条目数等于有效项数 |
| `DirectPathPadsTail` | 512 对齐、1024 容量、513 长度的请求：注册区 `size==1024`、`addr==caller buffer`；`[513,1024)` 全 0；`StoreAsync` 只发一次 |
| `StagedPathDoesNotTouchCallerBuffer` | `capacity=0`：注册地址 512 对齐且不等于 caller buffer；caller buffer 内容不变；读回前 N 字节正确 |
| `RegisterFailureFailsWholeBatch` | `RegisterRegions` 返回 `PARTIAL_FAILED`：全批 `DFS_SERVICE_UNAVAILABLE`；`StoreAsync` 未调用 |
| `TimeoutKeepsHandlesAndStaging` | `Wait` 返回 `TIMEOUT`：全批 `RPC_TIMEOUT`，不出现 `OBJECT_NOT_FOUND`；`UnregisterRegions` 未调用；析构不崩溃 |
| `ReadReturnsLogicalLength` | 513 字节对象：`GetBatch` 返回 513；`ObjectGetRequest.size` 为 513、capacity=0 时走 staged 并回拷 513 |
| `OverwriteDoesNotPreDelete` | 连续两次 `Put` 同一 key：调用序列中没有 `DeleteAsync`；两次都 `StoreAsync` |
| `UninitializedBatchFailsSafely` | 未 `Init` 时 `PutBatch`/`GetBatch` 逐项 `DFS_SERVICE_UNAVAILABLE`，SDK 方法零调用 |
| `SharedBufferRegisteredOnce` | 两个请求指向同一 `(addr, wire_len)`：`RegisterRegions` 只收到 1 个 region；终态 `UnregisterRegions` 只收到 1 个 handle |

### 10.4 测试环境约束

- 测试只用 fake client，不需要真实设备或 view service。
- 注入构造 `KvClientObjectStorageAdapter(config, std::make_unique<FakeKvClient>())` 后调用 `Init()`：注入路径只校验配置、不调用 SDK `Init`。
- 所有 direct 用例必须显式构造 512 对齐的缓冲区（例如 aligned 分配），普通 `std::vector` 不保证对齐。

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
| `dfs_replica_num=1` 写 | 铸造 `object_size=N` / `aligned_size=AlignUp(N,512)` / `shard_idx=-1` 的 DFS descriptor |
| `dfs_replica_num=1` 且 `N > 16,776,704` | `INVALID_PARAMS` |
| `dfs_replica_num=1` 且 `nof_replica_num>0` | `INVALID_PARAMS` |
| `EvictDiskReplicaIfCurrent(ReplicaType::DFS)` | 白名单放行；`EvictDiskReplica` 仍返回 `INVALID_PARAMS` |

### 11.4 端到端联调说明

PR2 不改 `AttachHostId`，因此调用方必须显式传 `dfs_replica_num=1` 才会铸造 DFS descriptor；否则只写内存副本。端到端手工验证时：

```bash
MOONCAKE_DFS_FS_ADAPTER=kvclient \
MOONCAKE_KVCLIENT_CONFIG=/path/to/kvclient.conf \
MOONCAKE_KVCLIENT_WAIT_TIMEOUT_MS=30000 \
# SDK 配置必须包含 client.asuSharedProvider=1
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
  - 生产路径只走暂存；直传仅在适配器单测可达；
  - `StoreAsync` 覆盖写，未引入预删；
  - `Exists` / `ListKeys` 仍为 `NOT_SUPPORTED`；
  - kvclient 不支持 snapshot/oplog，`enable_oplog=true` 时启动期拒绝；
  - 恢复门保持 KVCS-only；
  - 端到端写需要调用方显式 `dfs_replica_num=1`（`AttachHostId` 在 PR3）。
- 使用 `.github/pull_request_template.md`，填写模块、类型、测试、检查清单、AI 协助披露。
- 人工逐行审阅所有改动。

---

## 13. 验收清单

| 项 | 通过标准 |
| --- | --- |
| 接口层 | 四个结构末尾追加 capacity；既有聚合初始化不改仍编译 |
| 写路径 | `PutBatch` 对 513 字节对象补齐到 1024 并成功提交 |
| 读路径 | `GetBatch` 返回 N；Backend 长度校验通过 |
| 暂存 | `capacity=0` 时不触碰调用方写缓冲区 |
| 直传 | 单测中 512 对齐 + 足量容量时走直传并补零 |
| 注册 | `(addr, wire_len)` 去重；失败整批 `DFS_SERVICE_UNAVAILABLE` |
| 超时 | 全批 `RPC_TIMEOUT`；handle/staging 保留；不注销 |
| 覆盖写 | 不产生 `DeleteAsync` |
| 守卫 | Init 前调用逐项 `DFS_SERVICE_UNAVAILABLE` |
| Master 启动 | kvclient 早退分支生效；oplog 时 FATAL |
| descriptor | `aligned_size` 512 对齐；`shard_idx=-1` |
| Master 门控 | 长度上限、NOF 拒绝、DFS 可用性门、驱逐白名单生效 |
| 恢复门 | 保持 KVCS-only |
| 测试 | 10 个用例全部通过；SDK 缺失构建不受影响 |
| 回归 | 既有 KVCS / OSS / S3 / DFS 测试全绿 |

---

## 14. 实现约束速查

1. **capacity**：只加字段与透传，生产路径全 0；多 slice 读固定 0。
2. **staging**：aligned new 512；写侧拷贝并清零尾部，读侧只回拷 N。
3. **注册**：按 `(addr, wire_len)` 去重；输出条数必须等于输入条数。
4. **释放**：handle 去重后一次 `UnregisterRegions`；只在终态或提交失败时调用。
5. **非终态**：全批 `RPC_TIMEOUT`，绝不返回 `OBJECT_NOT_FOUND`。
6. **覆盖写**：不预删；不引入 `DeleteAsync` + `StoreAsync`。
7. **守卫**：`PutBatch` / `GetBatch` 入口检查 `initialized_` / `kv_client_`。
8. **析构**：先 `Shutdown()`，再清 `orphans_`。
9. **Master 初始化**：kvclient 早退分支必须在 filesystem allocator 之前。
10. **descriptor**：kvclient 单独子分支，KVCS 行为不变。
11. **错误码**：DFS 可用性门保持 `DFS_SERVICE_UNAVAILABLE`。
12. **恢复门**：不改，保持 KVCS-only。

---

## 15. 已知风险与待确认

| 风险 | 处理 |
| --- | --- |
| SDK HOST 内存注册要求 `client.asuSharedProvider=1`；默认 `INDEPENDENT` 下 AICPU `BindMemory` 只接受 `MEM_DEVICE` | Init 期建议增加前置校验：`cfg.sharedProviderMode != SHARED` 时记录 ERROR 并拒绝；部署文档写明 |
| 并发 batch 注册同一 4KB 页可能触发 provider 的 host-mapping 冲突 | PR2 不做引用计数；先记录风险，压测验证；必要时后续改为预注册 arena |
| orphan 无界增长 | PR2 只有 ERROR 日志；指标与上限留后续 |
| XXH64 非单射 | 已接受；日志记录 `cache_key` 便于定位碰撞 |
| `max_object_bytes` 配小于 Master 协议上限 | Master 放行、适配器拒绝，写会触发 PutRevoke；部署时保持默认或与 Master 上限一致 |
| `CheckHealth` 未实现 | `MOONCAKE_DISTRIBUTED_HEALTH_CHECK=1` 时 Init 失败；部署必须关闭 |
| 端到端写依赖调用方显式 `dfs_replica_num=1` | PR3 的 `AttachHostId` 才自动补；PR2 联调用例显式传 |
| `EvictDiskReplica` / `BatchEvictDiskReplica` 的 DFS 分支不可用 | PR2 明确不改；只验证 `EvictDiskReplicaIfCurrent` |
