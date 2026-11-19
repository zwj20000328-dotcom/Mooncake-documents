# Mooncake + KVClient L3 接入（Kvcs）— PR2 实施步骤

版本：v1.2　日期：2026-11-19

对应设计文档：`Mooncake + KVClient L3 接入（Kvcs）- PR 实施设计.md`（PR2 章节）

代码基线：`Mooncake` 分支 `Mooncake_l3_pr2`，HEAD `f3c8ffd5`（含 PR1 最新调整）；SDK 基线 `tmp_pr1641/ucm-pr`

---

## 0. 前提与决策

### 0.1 PR2 边界

PR2 在 PR1 的控制面之上补齐数据面与 Master 门控：

- 适配器：`PutBatch` / `GetBatchAsync` 批读写；`Put` / `PutV` / `Get` 单条包装；`Exists` 查询；读完成句柄。
- buffer 注册：由 `DistributedStorageBackend` 在提交前调用适配器 `RegisterBuffer`，请求携带 opaque `buffer_handle`；适配器不注册、不维护注册表。
- 写路径：零拷贝，使用调用方预注册的 handle 提交 `StoreAsync`；不做 staging、不补齐。
- 读路径：使用调用方预注册的 handle 提交 `LoadAsync`；同步 `GetBatch` 改为 `GetBatchAsync` + `ObjectReadCompletion`（`kPending` / `kDone`），完成前 buffer/handle 有效。
- 超时、参数校验与设备错误全部由 KvClient SDK 负责；适配器不设超时、不校验请求、不保留额外资源。
- 读完成句柄的传导：`ObjectStorageAdapter` → `DistributedStorageBackend::BatchReadAsync` → `Client`/`RealClient` 的 buffer 所有者。
- Master：`use_kvclient_`、`enable_dfs_`、kvclient 早退分支、descriptor 铸造、NOF 校验、驱逐白名单。
- 测试：PR2 自带适配器、handle 透传与完成句柄单测。

PR2 不做：

- Client 侧 value-first 删除、`ReconcileMissingKvReplica` 改名与放宽、`AttachHostId`（PR3）。
- `ListKeys`：保持 `NOT_SUPPORTED`；`Exists` 在 PR2 实现（`QueryAsync`）。
- `CheckHealth` / `GetRange` / `PutOnce` / `PutVOnce`：保持基类默认。
- 适配器侧任何超时或参数策略；这些属于 KvClient SDK。

### 0.2 已确认决策（2026-11-19）

| 决策项 | 结论 | 落地要求 |
| --- | --- | --- |
| 写路径 | 零拷贝；成功与否由 KvClient 判定 | 适配器不 staging、不补齐、不校验；只保留空 key、N==0、空指针、同批重复 key、handle 守卫 |
| buffer 注册 | 调用方预注册，请求携带 `buffer_handle` | `RegisterBuffer` / `UnregisterBuffer` 由 `DistributedStorageBackend` 调用；适配器不注册、不维护注册表 |
| 读路径 | 使用 handle 零拷贝读 | `GetBatchAsync` + `ObjectReadCompletion`；完成前 buffer/handle 有效 |
| 完成状态 | 只有 `kPending` / `kDone` | `WaitFor` 调用一次 SDK `Wait`；返回什么就映射什么，不设适配器超时状态 |
| 超时 | 由 KvClient SDK 负责 | `Wait(task_id, 0, result)`；适配器不重试、不保留资源 |
| `Exists` | PR2 实现：`QueryAsync` + `Wait` | 不注册 buffer；按 entry 级 `ENTRY_KEY_EXIST` / `ENTRY_KEY_NOT_EXIST` 返回 |
| 配置项 | `KvClientAdapterConfig` 只有 `config_path` | 适配器不校验任何数值配置 |
| kvclient + oplog | 启动期 `LOG(FATAL)` 拒绝 | 在 `InitDfsAllocatorFromEnvironment` 的 kvclient 分支内拒绝；恢复门保持 KVCS-only |
| 测试归属 | PR2 自带 | 新增 `kvclient_object_storage_adapter_test.cpp`，仅 SDK 存在时编译 |

### 0.3 与旧文档的差异（实施前必读）

| # | 旧文档写法 | 当前做法 |
| --- | --- | --- |
| 1 | 给请求结构加字段 | 增加 opaque `buffer_handle`；默认 0 |
| 2 | 适配器内做请求校验/拷贝/资源保留 | 全部删除；请求校验与超时由 KvClient SDK 负责 |
| 3 | `aligned_size = AlignUp(...)` | `object_size = aligned_size = value_length` |
| 4 | Master 预检请求参数 | 不预检；成功/失败由 SDK 判定 |
| 5 | 同步 `GetBatch` | 改为 `GetBatchAsync` + `ObjectReadCompletion`（`kPending` / `kDone`） |
| 6 | `Exists` 保持 `NOT_SUPPORTED` | PR2 实现：`QueryAsync` + entry 级 code |
| 7 | `Completion` 重新声明为命名空间级类型 | 已是类内私有类型，不再声明 |
| 8 | `PutBatch` 无初始化守卫 | 入口检查 `!initialized_ \|\| !kv_client_`，逐项 `DFS_SERVICE_UNAVAILABLE` |
| 9 | 非终态逐项映射 | 不再区分；SDK 返回什么就映射什么 |
| 10 | 适配器每批注册 buffer | 改为调用方预注册 + `buffer_handle` 透传；adapter 不注册 |

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
| 2 | 适配器头 | `kvclient_object_storage_adapter.h` | `ObjectReadCompletion`、`Exists`、`RegisterBuffer`/`UnregisterBuffer` |
| 3 | helper | 同上 `.cpp` | 求和、错误填充 |
| 4 | 外部 handle 与后端注册 | `object_storage_adapter.h`、`distributed_storage_backend.*` | handle 透传、注册/注销 |
| 5 | `PutBatch` | 同上 | 零拷贝提交/等待/映射 |
| 6 | `GetBatchAsync` + 完成句柄 | 同上 | `kPending` / `kDone` |
| 7 | 后端/Client 传导 + 单条包装、`Exists`、析构 | `distributed_storage_backend.*`、`client_service.*`、`real_client.*`、适配器 | 句柄到 buffer 所有者；`Exists` 查询 |
| 8 | Master 门控 | `master_service.h/.cpp` | 启动、descriptor、白名单 |
| 9 | 测试 | `tests/kvclient_object_storage_adapter_test.cpp`、`tests/CMakeLists.txt` | 用例见 §10.3 |
| 10 | 构建验证 | CMake / ctest | SDK 缺失与存在两种构建 |
| 11 | PR 提交 | PR 模板 | 边界写清 |

---

## 2. 步骤 1：写路径零拷贝

### 2.1 模型

KVClient 写路径使用调用方预注册的 `buffer_handle` 构造 `KVBuffer` 并交给 `StoreAsync`，不做任何数据拷贝或补齐。参数合法性、超时、设备侧可写性全部由 KvClient 判定；失败时按 SDK 状态码映射为 Mooncake `ErrorCode`。

因此：

- 适配器只保留空 key、`N == 0`、空指针、同批重复 key、`buffer_handle == 0` 守卫。
- 不分配 staging、不 `memcpy`、不 `memset`、不补齐。
- `ObjectPutRequest` 增加 `uint64_t buffer_handle`；适配器不注册、不注销。
- 适配器不设超时、不重试、不保留资源；这些由 KvClient SDK 负责。

---

## 3. 步骤 2：适配器头文件

文件：`mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h`

### 3.1 include

现有 include 只有 `<memory>/<string>/<string_view>/<vector>`，需要补：

```cpp
#include <cstdint>
#include <optional>
#include <span>
#include <utility>
```

### 3.2 私有成员与声明

```cpp
   private:
    class ReadCompletion;   // 定义在 .cpp，实现 ObjectReadCompletion

    struct Completion {
        bool terminal = false;
        std::vector<kv::Status> entryStatus;
    };

    kv::BinaryKey MakeCacheKey(std::string_view logical_key) const;
    // 调用一次 SDK Wait；OK/PARTIAL_FAILED 的 entryStatus 有效，
    // 其他状态把 task 级状态填满每个 entry。超时由 SDK 判定。
    Completion waitForCompletion(kv::TaskId task_id, size_t entry_count);
    static ErrorCode ToErrorCode(const kv::Status& status);

    KvClientAdapterConfig config_;
    std::unique_ptr<kv::KvClient> kv_client_;
    bool initialized_ = false;
```

约束：

- `Completion`、`waitForCompletion`、`MakeCacheKey`、`ToErrorCode`、构造函数与 `config_/kv_client_/initialized_` 在 PR1 已存在，**不要重复声明**。
- 公共接口新增：
  `std::unique_ptr<ObjectReadCompletion> GetBatchAsync(const std::vector<ObjectGetRequest>&) override;`
  `GetBatch` 保留为兼容包装（内部调用 `GetBatchAsync` + `WaitFor`）。
  `tl::expected<uint64_t, ErrorCode> RegisterBuffer(void*, size_t) override;`
  `void UnregisterBuffer(uint64_t) override;`
  `bool RequiresBufferHandle() const override;`
- `Exists` 已在类中 override，PR2 用 `QueryAsync` 实现。
- 适配器没有 `BufferPlan`、`RegisterBuffers`、`ReleaseBuffers`；请求 buffer 由后端预注册。

---

## 4. 步骤 3：文件内 helper

文件：`mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp`

新增 include：`<limits>`、`<optional>`、`<set>`、`<span>`、`<utility>`。

匿名命名空间：

```cpp
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

template <typename T>
void FillError(std::vector<tl::expected<T, ErrorCode>>& results,
               ErrorCode error) {
    for (auto& result : results) result = tl::make_unexpected(error);
}
```

约束：

- 不提供额外的 buffer 分配或拷贝 helper：适配器只使用调用方注册的 handle。
- 现有 `Errors<T>`（`.cpp:21-28`）在替换占位后成为死代码，改成 `FillError` 或删除。

---

## 5. 步骤 4：外部 handle 与后端注册

### 5.1 适配器侧：`RegisterBuffer` / `UnregisterBuffer`

适配器不维护注册表，只提供两个显式接口给后端：

```cpp
tl::expected<uint64_t, ErrorCode>
KvClientObjectStorageAdapter::RegisterBuffer(void* ptr, size_t size) {
    if (!initialized_ || !kv_client_) {
        return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);
    }
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

void KvClientObjectStorageAdapter::UnregisterBuffer(uint64_t handle) {
    if (handle == kv::kInvalidMRHandle || !kv_client_) return;
    kv_client_->UnregisterRegions({handle});
}

bool KvClientObjectStorageAdapter::RequiresBufferHandle() const { return true; }
```

`ObjectStorageAdapter` 的默认实现：

```cpp
virtual tl::expected<uint64_t, ErrorCode> RegisterBuffer(void*, size_t) {
    return tl::make_unexpected(ErrorCode::NOT_SUPPORTED);
}
virtual void UnregisterBuffer(uint64_t) {}
virtual bool RequiresBufferHandle() const { return false; }
```

请求结构末尾增加：

```cpp
uint64_t buffer_handle = 0;   // opaque adapter token; 0 = unregistered
```

### 5.2 后端侧：提交前注册、完成后注销

`DistributedStorageBackend` 负责 handle 生命周期：

- `RequiresObjectBufferHandle()` 委托给 `object_storage_adapter_->RequiresBufferHandle()`。
- 写：`BatchWrite` 对每个请求计算总长度，调用 `RegisterBuffer(iov[0].base, total)` 得到 handle，填入 `ObjectPutRequest`；`PutBatch` 返回后注销。
- 读：`BatchRead` / `BatchReadAsync` 对目标 buffer（调用方 buffer 或后端 staging）调用 `RegisterBuffer(buffer, N)`，填入 `ObjectGetRequest`；同步读在 `GetBatch` 返回后注销，异步读在完成句柄 `kDone` 后注销。
- 异步读的完成包装同时持有 handle 与多 slice staging，确保 `kDone` 之前不提前释放。

约束：

- 适配器只在 `RegisterBuffer` / `UnregisterBuffer` 内调用 SDK 的 `RegisterRegions` / `UnregisterRegions`；`PutBatch` / `GetBatchAsync` 不注册。
- `buffer_handle == 0` 的参与项返回 `INVALID_PARAMS`，不提交该项。
- handle 模式要求请求的 iov 覆盖一段连续 host 内存（取首地址 + 总长度注册）；后端遇到不连续 iov 时返回 `INVALID_PARAMS`。
- 并发重复注册同一 `(addr, size)` 由后端的上层保证不重叠，或由 SDK 的注册语义处理；PR2 不引入适配器侧引用计数。

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

    std::set<kv::BinaryKey> seen;   // 仅写侧：同批重复 key
    std::vector<kv::KVBuffer> entries;
    std::vector<size_t> indices;
    for (size_t i = 0; i < requests.size(); ++i) {
        if (requests[i].logical_key.empty() || requests[i].iov == nullptr ||
            requests[i].iovcnt <= 0 ||
            requests[i].iov[0].iov_base == nullptr) {
            results[i] = tl::make_unexpected(ErrorCode::INVALID_PARAMS);
            continue;
        }
        const auto binary_key = MakeCacheKey(requests[i].logical_key);
        if (!seen.insert(binary_key).second) {
            results[i] = tl::make_unexpected(ErrorCode::INVALID_PARAMS);
            continue;
        }
        const auto total = SumIov(requests[i].iov, requests[i].iovcnt);
        if (!total || *total == 0 || requests[i].buffer_handle == 0) {
            results[i] = tl::make_unexpected(ErrorCode::INVALID_PARAMS);
            continue;
        }
        kv::KVBuffer entry;
        entry.key = kv::CacheKey::FromBytes(binary_key);
        entry.buffer.region = {
            kv::MemoryType::HOST,
            reinterpret_cast<uint64_t>(requests[i].iov[0].iov_base),
            *total, -1, -1};
        entry.buffer.handle = requests[i].buffer_handle;
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
        return results;
    }

    const auto completion = waitForCompletion(task_id, entries.size());
    if (completion.entryStatus.size() != entries.size()) {
        for (size_t index : indices) {
            results[index] = tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
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
    return results;
}
```

约束：

- 不校验请求参数；由 `StoreAsync` 返回的 SDK 状态映射。
- 已确认 `StoreAsync` 覆盖写，不预删。
- 提交失败与 Task 级错误只覆盖参与项，空 key / 重复 key / 缺 handle 的错误保留。
- 适配器不注册、不注销 handle；`Wait` 后由后端注销。

---

## 7. 步骤 6：`GetBatchAsync` + 完成句柄

### 7.1 接口契约

`object_storage_adapter.h` 新增：

```cpp
enum class ObjectReadState { kPending, kDone };

class ObjectReadCompletion {
   public:
    virtual ~ObjectReadCompletion() = default;
    virtual ObjectReadState State() const = 0;
    // 调用一次 SDK Wait；超时由 SDK 判定。之后返回固定状态。
    virtual ObjectReadState WaitFor() = 0;
    // 仅 kDone 有效：按请求顺序返回 N。
    virtual std::vector<tl::expected<size_t, ErrorCode>> Results() = 0;
};

class ObjectStorageAdapter {
    // 默认实现：同步 GetBatch 后返回 kDone，保持 KVCS/OSS/S3 行为不变。
    virtual std::unique_ptr<ObjectReadCompletion> GetBatchAsync(
        const std::vector<ObjectGetRequest>& requests);
};
```

规则：

| 状态 | buffer / 注册归属 | 调用方动作 |
| --- | --- | --- |
| `kPending` | adapter 持有注册；buffer 归调用方但不得修改或释放 | 调用 `WaitFor()`，或把句柄上送 |
| `kDone` | adapter 已注销 handle；buffer 归还调用方 | 读 `Results()`，可复用 buffer |

- `WaitFor()` 调用一次 `waitForCompletion(task_id, count)`（内部 `Wait(task_id, 0)`）；SDK 返回什么就映射什么，适配器不设超时状态。
- 完成句柄必须被 `WaitFor()` 消费；如果在 `kPending` 时丢弃，handle 无人注销，由 SDK `Shutdown` 兜底。
- 默认实现（KVCS/OSS/S3）返回 `kDone`，同步路径无行为变化。

### 7.2 适配器实现

```cpp
class KvClientObjectStorageAdapter::ReadCompletion final
    : public ObjectReadCompletion {
   public:
    ReadCompletion(KvClientObjectStorageAdapter* adapter, kv::TaskId task_id,
                   std::vector<size_t> sizes,
                   std::vector<size_t> indices,
                   std::vector<tl::expected<size_t, ErrorCode>> results)
        : adapter_(adapter), task_id_(task_id), sizes_(std::move(sizes)),
          indices_(std::move(indices)), results_(std::move(results)) {}

    ObjectReadState State() const override { return state_; }

    ObjectReadState WaitFor() override {
        if (state_ != ObjectReadState::kPending) return state_;

        const auto completion = adapter_->waitForCompletion(
            task_id_, indices_.size());
        if (completion.entryStatus.size() != indices_.size()) {
            for (size_t i : indices_) {
                results_[i] = tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
            }
        } else if (!completion.terminal) {
            // Task 级错误：entryStatus 已被填成 task 状态，统一映射。
            const auto error = ToErrorCode(completion.entryStatus[0]);
            for (size_t i : indices_) {
                results_[i] = tl::make_unexpected(error);
            }
        } else {
            for (size_t j = 0; j < indices_.size(); ++j) {
                const size_t i = indices_[j];
                results_[i] = completion.entryStatus[j].ok()
                                  ? tl::expected<size_t, ErrorCode>{sizes_[i]}
                                  : tl::make_unexpected(
                                        ToErrorCode(completion.entryStatus[j]));
            }
        }
        state_ = ObjectReadState::kDone;
        return state_;
    }

    std::vector<tl::expected<size_t, ErrorCode>> Results() override {
        return results_;
    }

   private:
    KvClientObjectStorageAdapter* adapter_;
    kv::TaskId task_id_;
    std::vector<size_t> sizes_;
    std::vector<size_t> indices_;
    std::vector<tl::expected<size_t, ErrorCode>> results_;
    ObjectReadState state_ = ObjectReadState::kPending;
};
```

`GetBatchAsync`：

```cpp
std::unique_ptr<ObjectReadCompletion>
KvClientObjectStorageAdapter::GetBatchAsync(
    const std::vector<ObjectGetRequest>& requests) {
    // 未完成前 Results() 不应被读取；这里预填 INTERNAL_ERROR，
    // 避免默认构造的 expected<size_t> 被误读成“成功 0 字节”。
    std::vector<tl::expected<size_t, ErrorCode>> results(
        requests.size(), tl::make_unexpected(ErrorCode::INTERNAL_ERROR));
    std::vector<size_t> sizes(requests.size(), 0);

    if (!initialized_ || !kv_client_) {
        FillError(results, ErrorCode::DFS_SERVICE_UNAVAILABLE);
        return MakeReadyCompletion(std::move(results));
    }

    std::vector<kv::KVBuffer> entries;
    std::vector<size_t> indices;
    for (size_t i = 0; i < requests.size(); ++i) {
        if (requests[i].logical_key.empty() || requests[i].buffer == nullptr ||
            requests[i].size == 0 || requests[i].buffer_handle == 0) {
            results[i] = tl::make_unexpected(ErrorCode::INVALID_PARAMS);
            continue;
        }
        sizes[i] = requests[i].size;
        kv::KVBuffer entry;
        entry.key = kv::CacheKey::FromBytes(
            MakeCacheKey(requests[i].logical_key));
        entry.buffer.region = {
            kv::MemoryType::HOST,
            reinterpret_cast<uint64_t>(requests[i].buffer),
            requests[i].size, -1, -1};
        entry.buffer.handle = requests[i].buffer_handle;
        entry.offset = 0;
        entry.mrKey = std::nullopt;
        entries.push_back(std::move(entry));
        indices.push_back(i);
    }
    if (entries.empty()) return MakeReadyCompletion(std::move(results));

    kv::TaskId task_id = kv::kInvalidTaskId;
    const auto submit = kv_client_->LoadAsync(entries, task_id);
    if (!submit.ok()) {
        for (size_t index : indices) {
            results[index] = tl::make_unexpected(ToErrorCode(submit));
        }
        return MakeReadyCompletion(std::move(results));
    }

    return std::make_unique<ReadCompletion>(
        this, task_id, std::move(sizes), std::move(indices),
        std::move(results));
}
```

`MakeReadyCompletion(results)` 返回一个 `kDone` 的轻量完成句柄；`Results()` 直接返回已填好的向量。实现时可以是一个只持有 `std::vector<tl::expected<size_t, ErrorCode>>` 的私有 `ReadyCompletion` 小类。

兼容用的同步 `GetBatch`：

```cpp
std::vector<tl::expected<size_t, ErrorCode>>
KvClientObjectStorageAdapter::GetBatch(
    const std::vector<ObjectGetRequest>& requests) {
    auto completion = GetBatchAsync(requests);
    completion->WaitFor();
    return completion->Results();
}
```

### 7.3 后端与 Client 传导

```cpp
struct DfsReadAsyncResult {
    // 已完成条目
    std::vector<tl::expected<void, ErrorCode>> results;
    // 非空表示还有未完成条目；持有 handle、buffer 与多 slice staging 的所有权
    std::shared_ptr<ObjectReadCompletion> pending;
};

DfsReadAsyncResult DistributedStorageBackend::BatchReadAsync(
    const std::vector<DfsReadRequest>& requests);
```

- 多 slice 请求的 `staging_buffers` 由 `pending` 持有；完成时由 completion 的回调把 N 字节散回各 slice。
- `Client::BatchGet` / `ReadDfsReplica` / `RealClient` 增加完成句柄出口（异步变体或 out 参数）：`kDone` 后散回并返回。
- 同步 `BatchRead` 仅作为兼容入口：内部调用 `BatchReadAsync` + `WaitFor()`；KVClient 的主路径使用 `BatchReadAsync`。

### 7.4 生命周期

- `GetBatchAsync` 返回后，buffer 与 handle 由后端的完成包装持有；调用方不得 `resize`/释放/复用。
- `kDone` 时后端注销 handle，`Results()` 有效，buffer 归还调用方。
- 完成句柄必须被 `WaitFor()` 消费；若在 `kPending` 丢弃，由后端的完成包装在 `Shutdown` 时兜底注销。
- 该契约要求 `Client`/`RealClient` 把句柄传到真正拥有 slices 的层；这是 PR2 的主要接口变更。
- 完成句柄不得长于 adapter：句柄持有 adapter 指针，`Client` 必须在 adapter 析构前消费或丢弃；若需要跨层长期持有，改为共享状态（`shared_ptr`）而不是裸指针。

---

## 8. 步骤 7：单条包装、`Exists`、析构

### 8.1 单条入口

kvclient 的 handle 只能通过 batch 请求携带；`Put` / `PutV` / `Get` 没有 handle 参数，因此保持 `NOT_SUPPORTED`，所有 kvclient I/O 走 `PutBatch` / `GetBatch` / `GetBatchAsync`。

```cpp
tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::Put(
    const std::string&, std::span<const char>) {
    return tl::make_unexpected(ErrorCode::NOT_SUPPORTED);
}

tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::PutV(
    const std::string&, const iovec*, int) {
    return tl::make_unexpected(ErrorCode::NOT_SUPPORTED);
}

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

> 若还有存活的完成句柄，它们不得在 adapter 析构后使用；句柄不负责延长 adapter 生命周期。

### 8.3 `Exists`

`Exists` 走 SDK 查询接口，不注册 buffer，也不涉及完成句柄：

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

    kv::TaskResult result;
    const auto wait = kv_client_->Wait(task_id, 0, result);
    if (!wait.ok()) return tl::make_unexpected(ToErrorCode(wait));
    if (result.status.code != kv::StatusCode::OK &&
        result.status.code != kv::StatusCode::PARTIAL_FAILED) {
        return tl::make_unexpected(ToErrorCode(result.status));
    }
    if (result.entryStatus.size() != 1 ||
        (result.queryResult &&
         result.queryResult->exists.size() != 1)) {
        return tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
    }
    switch (result.entryStatus[0].code) {
        case kv::StatusCode::ENTRY_KEY_EXIST: return true;
        case kv::StatusCode::ENTRY_KEY_NOT_EXIST: return false;
        default:
            return tl::make_unexpected(ToErrorCode(result.entryStatus[0]));
    }
}
```

约束：

- **不能用 `entryStatus[0].ok()`**：查询成功的 entry 是 `ENTRY_KEY_EXIST`（非 `OK`），必须按 code 分支。
- `queryResult->exists` 只做一致性校验；布尔值以 entryStatus 的明确 code 为准。
- `Wait` 返回的失败按 SDK 状态映射；查询不注册 buffer，没有额外资源需要保留。
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
| `HandleIsPassedThrough` | `RegisterBuffer` 返回 fake SDK 的 handle；`PutBatch` / `GetBatchAsync` 构造的 `KVBuffer.buffer.handle` 等于请求中的 `buffer_handle`；适配器内部不调用 `RegisterRegions` |
| `RejectsEmptyKeyZeroLengthMissingHandleAndDuplicate` | 空 key / `size==0` / `buffer_handle==0` 逐项 `INVALID_PARAMS`；同批重复 key 第一条成功、后续 `INVALID_PARAMS` |
| `RegisterBufferFailure` | `RegisterBuffer` 在 SDK 返回 `PARTIAL_FAILED` 时返回 `DFS_SERVICE_UNAVAILABLE`；`PutBatch` / `GetBatchAsync` 未被调用 |
| `WriteTaskErrorMapsAll` | fake `Wait` 返回 task 级错误：参与项统一映射该错误 |
| `ReadDone` | `GetBatchAsync` → `WaitFor` 返回 `kDone`；`Results()` 为 N |
| `ReadSubmitFailure` | `LoadAsync` 提交失败：返回 `kDone` 句柄，逐项错误；调用方 buffer 未被写 |
| `ReadTaskErrorMapsAll` | fake `Wait` 返回 task 级错误：`Results()` 全部映射该错误 |
| `ExistsReturnsEntryCode` | `QueryAsync` 的 entry 为 `ENTRY_KEY_EXIST` / `ENTRY_KEY_NOT_EXIST` → `true` / `false`；不使用 `ok()` |
| `ExistsErrors` | 空 key → `INVALID_PARAMS`（零 SDK 调用）；提交失败/entry 错误 → 对应 `ErrorCode` |
| `OverwriteDoesNotPreDelete` | 两次 `PutBatch` 同一 key：调用序列中没有 `DeleteAsync`；两次都 `StoreAsync` |
| `UninitializedBatchFailsSafely` | 未 `Init` 时 `PutBatch`/`GetBatchAsync` 返回 `DFS_SERVICE_UNAVAILABLE`，SDK 方法零调用 |
| `UnregisterBufferCallsSdk` | `UnregisterBuffer(handle)` 只调用一次 `UnregisterRegions({handle})`；`kInvalidMRHandle` 被忽略 |

### 10.4 测试环境约束

- 测试只用 fake client，不需要真实设备或 view service。
- 注入构造 `KvClientObjectStorageAdapter(config, std::make_unique<FakeKvClient>())` 后调用 `Init()`：注入路径只校验配置、不调用 SDK `Init`。
- 零拷贝用例的 buffer 用普通 `std::vector<char>` 即可；参数是否可接受由 fake SDK 决定。
- 完成句柄用例通过 fake `Wait` 注入 `OK` / task 级错误；断言适配器在 `PutBatch` / `GetBatchAsync` 内**不**调用 `UnregisterRegions`。注册/注销单独通过 `RegisterBuffer` / `UnregisterBuffer` 用例覆盖。

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
# 写路径零拷贝；读路径注册调用方 buffer
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
  - 读路径注册调用方 buffer，`GetBatchAsync` + `ObjectReadCompletion`（`kPending` / `kDone`）；完成前 buffer/注册不得释放；
  - 超时、参数校验与设备错误由 KvClient SDK 负责；
  - 完成句柄已从 adapter 经 `BatchReadAsync` 传导到 `Client`/`RealClient`；
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
| 写路径 | 零拷贝提交；请求携带后端预注册的 `buffer_handle`，适配器不注册 |
| 写失败判定 | SDK 返回的错误按 `ToErrorCode` 映射；适配器不做请求校验 |
| 读路径 | 请求携带 `buffer_handle`；`GetBatchAsync` 返回完成句柄；`WaitFor` 后 `kDone`，返回 N |
| `Exists` | `QueryAsync` 的 `ENTRY_KEY_EXIST` / `ENTRY_KEY_NOT_EXIST` → `true` / `false`；错误正确映射；`ListKeys` 仍 `NOT_SUPPORTED` |
| buffer handle | 后端在提交前 `RegisterBuffer`、完成后 `UnregisterBuffer`；适配器只在 `buffer_handle == 0` 时拒绝该项 |
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
2. **读路径**：使用请求中的 `buffer_handle`；`GetBatchAsync` 返回完成句柄（`kPending` / `kDone`）；只有 `kDone` 才能复用 buffer。
3. **buffer handle**：调用方（后端）预注册并透传；适配器只在 `RegisterBuffer` / `UnregisterBuffer` 内调用 SDK 注册接口。
4. **解除注册**：后端在 `PutBatch` 返回或读完成 `kDone` 后调用 `UnregisterBuffer`；适配器不注销。
5. **任务级错误**：由 SDK 返回，统一映射到参与项；适配器不设超时状态。
6. **buffer 生命周期**：`kPending` 期间调用方不得复用；句柄传导到 buffer 所有者。
7. **覆盖写**：不预删；不引入 `DeleteAsync` + `StoreAsync`。
8. **守卫**：`PutBatch` / `GetBatchAsync` 入口检查 `initialized_` / `kv_client_`，并拒绝 `buffer_handle == 0`。
9. **析构**：只调用 SDK `Shutdown()`。
10. **Master 初始化**：kvclient 早退分支必须在 filesystem allocator 之前。
11. **descriptor**：与 KVCS 相同；`aligned_size = value_length`。
12. **错误码**：DFS 可用性门保持 `DFS_SERVICE_UNAVAILABLE`。
13. **恢复门**：不改，保持 KVCS-only。
14. **配置**：只保留 `config_path`；不校验数值配置。
15. **完成句柄**：`ObjectStorageAdapter` → `BatchReadAsync` → `Client`/`RealClient` 全链路传递，不得在中间层丢弃。
16. **`Exists`**：`QueryAsync` + `Wait`，按 entry 级 `ENTRY_KEY_EXIST` / `ENTRY_KEY_NOT_EXIST` 返回；不能用 `ok()`；不注册 buffer。

---

## 15. 已知风险与待确认

| 风险 | 处理 |
| --- | --- |
| 完成句柄必须传导到 buffer 所有者 | `BatchReadAsync`、`Client`/`RealClient` 任一层丢弃句柄都会导致提前释放；PR 评审重点 |
| 完成句柄持有 adapter 裸指针 | 句柄不得长于 adapter；跨层长期持有需改为共享状态，否则 UAF |
| XXH64 非单射 | 已接受；日志记录 `cache_key` |
| `CheckHealth` 未实现 | `MOONCAKE_DISTRIBUTED_HEALTH_CHECK=1` 时 Init 失败；部署必须关闭 |
| 端到端写依赖调用方显式 `dfs_replica_num=1` | PR3 的 `AttachHostId` 才自动补 |
| `EvictDiskReplica` / `BatchEvictDiskReplica` 的 DFS 分支不可用 | PR2 明确不改；只验证 `EvictDiskReplicaIfCurrent` |
