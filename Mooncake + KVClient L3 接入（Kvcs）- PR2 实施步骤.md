# Mooncake + KVClient L3 接入（Kvcs）— PR2 实施步骤

版本：v1.1　日期：2026-11-19

对应设计文档：`Mooncake + KVClient L3 接入（Kvcs）- PR 实施设计.md`（PR2 章节）

代码基线：`Mooncake` 分支 `Mooncake_l3_pr2`，HEAD `f3c8ffd5`（含 PR1 最新调整）；SDK 基线 `tmp_pr1641/ucm-pr`

---

## 0. 前提与决策

### 0.1 PR2 边界

PR2 在 PR1 的控制面之上补齐数据面与 Master 门控：

- 适配器：`PutBatch` / `GetBatchAsync` 批读写；`Put` / `PutV` / `Get` 单条包装；按批注册与释放；读完成句柄。
- 写路径：零拷贝，调用方保证 512B 对齐与连续；适配器**不校验**对齐/连续性，不做 staging、不补齐；成功与否由 KvClient 判定。
- 读路径：注册调用方 buffer 做零拷贝读；同步 `GetBatch` 改为 `GetBatchAsync` + `ObjectReadCompletion`，完成前 buffer 与注册必须有效，超时未终态进入隔离。
- 读完成句柄的传导：`ObjectStorageAdapter` → `DistributedStorageBackend::BatchReadAsync` → `Client`/`RealClient` 的 buffer 所有者。
- Master：`use_kvclient_`、`enable_dfs_`、kvclient 早退分支、descriptor 铸造、NOF 校验、驱逐白名单。
- 测试：PR2 自带适配器与完成句柄单测。

PR2 不做：

- Client 侧 value-first 删除、`ReconcileMissingKvReplica` 改名与放宽、`AttachHostId`（PR3）。
- `ListKeys`：保持 `NOT_SUPPORTED`；`Exists` 在 PR2 实现（`QueryAsync`）。
- `CheckHealth` / `GetRange` / `PutOnce` / `PutVOnce`：保持基类默认。
- `MOONCAKE_KVCLIENT_MAX_OBJECT_BYTES` / `MOONCAKE_KVCLIENT_WAIT_TIMEOUT_MS` 相关校验（PR1 已删除）。
- 引入 `asuSharedProvider=1` 等额外部署前置条件。

### 0.2 已确认决策（2026-11-19）

| 决策项 | 结论 | 落地要求 |
| --- | --- | --- |
| 写路径 | 512B 对齐、零拷贝；成功与否由 KvClient 判定 | 适配器**不校验**对齐/连续性；只保留空 key、N==0、空指针守卫；不 staging、不补齐 |
| 读注册 | 注册调用方 buffer，零拷贝读 | 读契约改为 `GetBatchAsync` + `ObjectReadCompletion`；完成前 buffer/注册有效 |
| 读超时 | 完成句柄进入 `kIndeterminate` | handle 保留、不注销；buffer 由调用方隔离到 adapter `Shutdown` |
| `Exists` | PR2 实现：`QueryAsync` + `Wait` | 不注册 buffer；按 entry 级 `ENTRY_KEY_EXIST` / `ENTRY_KEY_NOT_EXIST` 返回 |
| 配置项 | PR1 已删除两个数值 env；`KvClientAdapterConfig` 只有 `config_path` | PR2 不校验 `max_object_bytes` / `wait_timeout_ms` |
| 等待超时 | `Wait(task_id, 0, result)` 使用 SDK 的 `defaultWaitTimeoutMs` | 适配器不设第二套超时 |
| kvclient + oplog | 启动期 `LOG(FATAL)` 拒绝 | 在 `InitDfsAllocatorFromEnvironment` 的 kvclient 分支内拒绝；恢复门保持 KVCS-only |
| 测试归属 | PR2 自带 | 新增 `kvclient_object_storage_adapter_test.cpp`，仅 SDK 存在时编译 |

### 0.3 与旧文档的差异（实施前必读）

| # | 旧文档写法 | 当前做法 |
| --- | --- | --- |
| 1 | 给四个请求结构加 `capacity` | 不加：写路径零拷贝、读路径单 buffer，现有字段已足够 |
| 2 | 写侧直传 + 暂存两种模式、`wire_len` 补齐 | 写路径只有零拷贝一种；不补齐、不暂存 |
| 3 | `AllocateStaging` / `CopyIovTo` / 写侧 `memset` | 删除；读路径注册调用方 buffer，不引入 bounce buffer |
| 4 | `max_object_bytes` / `wait_timeout_ms` 校验 | 已从 PR1 删除；PR2 不再校验 |
| 5 | `Init` 校验 `cfg.timeoutMs >= wait_timeout_ms` | 已删除；`Wait(..., 0, ...)` 使用 SDK 默认值 |
| 6 | `aligned_size = AlignUp(value_length, 512)` | 写默认 512 对齐，kvclient 与 KVCS 相同：`object_size = aligned_size = value_length` |
| 7 | Master 的 16,776,704 长度门 | 删除；由 KvClient 判定成功/失败 |
| 8 | `Completion` 重新声明为命名空间级类型 | 已是类内私有类型，不再声明 |
| 9 | `vector<Orphan> orphans_` | `vector<kv::MRHandle> orphans_`（零拷贝下不保留内存，只保留 handle） |
| 10 | `PutBatch` 无初始化守卫 | 入口检查 `!initialized_ \|\| !kv_client_`，逐项 `DFS_SERVICE_UNAVAILABLE` |
| 11 | 非终态逐项 `ToErrorCode(entryStatus)` | 写：全批 `RPC_TIMEOUT`；读：完成句柄进入 `kIndeterminate` |
| 12 | 同步 `GetBatch` | 改为 `GetBatchAsync` + `ObjectReadCompletion`，完成前 buffer/注册有效 |
| 13 | 校验 512/4KB 对齐与 iov 连续性 | **不校验**：调用方默认严格符合；适配器只保留空 key/N==0/空指针守卫 |
| 14 | 要求 `asuSharedProvider=1` | 不再作为 PR2 或部署前提 |
| 15 | 读路径用 arena/bounce buffer | 不需要：注册调用方 buffer 做零拷贝读 |
| 16 | `Exists` 保持 `NOT_SUPPORTED` | PR2 实现：`QueryAsync` + `Wait`，按 entry 级 `ENTRY_KEY_EXIST` / `ENTRY_KEY_NOT_EXIST` 返回 true/false |

### 0.4 分支

```bash
git switch Mooncake_l3_pr2   # 已从 Mooncake_l3_pr1 创建
```

PR3 分支 `Mooncake_l3_pr3` 已基于 PR2 创建，PR2 合入后再开始。

---

## 1. 实施顺序总览

| 步骤 | 内容 | 主要文件 | 验证点 |
| --- | --- | --- | --- |
| 1 | 写路径零拷贝判定 | `kvclient_object_storage_adapter.cpp` | 不拷贝、不补齐、不校验对齐/连续性 |
| 2 | 适配器头 | `kvclient_object_storage_adapter.h` | `BufferPlan`、`ObjectReadCompletion`、注册/释放/孤儿 |
| 3 | helper | 同上 `.cpp` | 求和溢出、错误填充 |
| 4 | 注册（写/读） | 同上 | 去重、整批失败、handle 回填 |
| 5 | `PutBatch` | 同上 | 零拷贝提交/等待/映射/释放 |
| 6 | `GetBatchAsync` + 完成句柄 | 同上 | `kPending`/`kDone`/`kIndeterminate` |
| 7 | 后端/Client 传导 + 单条包装、`Exists`、析构 | `distributed_storage_backend.*`、`client_service.*`、`real_client.*`、适配器 | 句柄到 buffer 所有者；`Exists` 查询；隔离到 Shutdown |
| 8 | Master 门控 | `master_service.h/.cpp` | 启动、descriptor、白名单 |
| 9 | 测试 | `tests/kvclient_object_storage_adapter_test.cpp`、`tests/CMakeLists.txt` | 16 个用例 |
| 10 | 构建验证 | CMake / ctest | SDK 缺失与存在两种构建 |
| 11 | PR 提交 | PR 模板 | 边界写清 |

---

## 2. 步骤 1：写路径零拷贝

### 2.1 模型

KVClient 写路径默认由调用方提供 **512B 对齐、连续** 的缓冲区；适配器把该缓冲区直接注册并交给 `StoreAsync`，不做任何数据拷贝或补齐。对齐、长度上限、设备侧可写性全部由 KvClient 判定；失败时按 SDK 状态码映射为 Mooncake `ErrorCode`。

因此：

- 不存在 `wire_len`：交给 SDK 的长度就是 N。
- 不存在 staging：不分配、不 `memcpy`、不 `memset`。
- 不校验对齐与连续性：适配器只拒绝空 key、N==0、空指针、求和溢出；512B 对齐与 iov 连续由调用方契约保证，失败由 SDK 返回并映射为 `INVALID_PARAMS`。
- 不引入 `capacity` 字段，接口层零改动。

### 2.2 与调用方 buffer 生命周期的契约

零拷贝意味着 `StoreAsync` 在途期间设备可能仍读取调用方内存。`Wait` 超时后 SDK 已消费 task、无法再确认完成，因此：

- 非终态返回时，适配器保留 handle、返回 `RPC_TIMEOUT`，**无法替调用方保护 buffer**。
- 调用方（`client_service`）在收到超时后不得立即复用或释放该 buffer；该约束需要写入 PR 描述，并在 PR3 的 Client 集成中落实。
- 若无法接受该约束，只能在写路径恢复"先拷贝到适配器自有 buffer"，那就不再是零拷贝；PR2 按用户确认的零拷贝方案执行。

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
    class ReadCompletion;   // 定义在 .cpp，实现 ObjectReadCompletion

    struct Completion {
        bool terminal = false;
        std::vector<kv::Status> entryStatus;
    };

    struct BufferPlan {
        uintptr_t addr = 0;   // 交给 SDK 的起始地址（当前为调用方 buffer）
        size_t n = 0;         // 交给 SDK 的长度，必须是 N
        kv::MRHandle handle = kv::kInvalidMRHandle;
    };

    kv::BinaryKey MakeCacheKey(std::string_view logical_key) const;
    Completion waitForCompletion(kv::TaskId task_id, size_t entry_count,
                                 uint64_t timeout_ms = 0);   // 0 = SDK 默认
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
    // 零拷贝写不保留内存，只保留无法注销的 handle。
    std::vector<kv::MRHandle> orphans_;
```

约束：

- `Completion`、`waitForCompletion`、`MakeCacheKey`、`ToErrorCode`、构造函数与 `config_/kv_client_/initialized_` 在 PR1 已存在，**不要重复声明**。
- 公共接口新增：
  `std::unique_ptr<ObjectReadCompletion> GetBatchAsync(const std::vector<ObjectGetRequest>&) override;`
  `GetBatch` 保留为兼容包装（内部调用 `GetBatchAsync` + `WaitFor`）。
- `BufferPlan` 不包含 staging / `wire_len` / `staged`。
- `RegisterBatchImpl` 的模板定义只放在 `.cpp`，仅在该 TU 实例化。
- `BufferPlan` 只保存调用方地址、长度与 handle；读/写都注册调用方 buffer，不引入 bounce buffer。

---

## 4. 步骤 3：文件内 helper

文件：`mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp`

新增 include：`<algorithm>`、`<limits>`、`<map>`、`<optional>`、`<set>`、`<span>`、`<utility>`。

匿名命名空间：

```cpp
struct BufferView {
    iovec single{};          // 读路径单段 view 落在自身
    const iovec* iov = nullptr;
    int iovcnt = 0;
    size_t n = 0;
};

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

// 连续性与 512B 对齐由调用方契约保证，适配器不做校验。

template <typename T>
void FillError(std::vector<tl::expected<T, ErrorCode>>& results,
               ErrorCode error) {
    for (auto& result : results) result = tl::make_unexpected(error);
}
```

约束：

- 不提供 `AlignUp` / `CopyIovTo` / `AllocateStaging`：写/读都注册调用方 buffer。
- 现有 `Errors<T>`（`.cpp:21-28`）在替换占位后成为死代码，改成 `FillError` 或删除。
- 不校验对齐，不需要 `kv::kAlignmentBytes`；对齐由调用方契约保证。

---

## 5. 步骤 4：注册（写路径）

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
        // 连续性与 512B 对齐由调用方契约保证；适配器不校验。
        BufferPlan plan;
        plan.addr = reinterpret_cast<uintptr_t>(view.iov[0].iov_base);
        plan.n = view.n;

        const auto region_key = std::make_pair(plan.addr, plan.n);
        auto iter = region_index.find(region_key);
        size_t region;
        if (iter == region_index.end()) {
            region = regions.size();
            regions.push_back({kv::MemoryType::HOST, plan.addr, plan.n, -1, -1});
            region_index.emplace(region_key, region);
        } else {
            region = iter->second;
        }
        plan_to_region[i] = region;
        plans[i] = plan;
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
        });
}
```

约束：

- `std::span` 不能从 `const std::vector<T>&` 隐式推导成模板实参；调用处显式写 `std::span<const ObjectPutRequest>(requests)`。
- 读路径同样注册调用方 buffer；`GetBatchAsync` 只负责提交与返回完成句柄。
- 写侧同批重复 key 用 `MakeCacheKey` 判重；哈希碰撞属于已接受风险。

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
                               plans[i]->n, -1, -1};
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
        // 任务未入队；只覆盖参与项，保留空 key / 重复 key 的 INVALID_PARAMS。
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
                   << task_id << ", entries=" << entries.size()
                   << ", note=caller buffers must stay valid";
    }
    return results;
}
```

约束：

- 不校验 512 对齐与长度上限；由 `StoreAsync` 返回的 SDK 状态映射。
- 已确认 `StoreAsync` 覆盖写，不预删。
- 提交失败与非终态只覆盖参与项，空 key / 重复 key 的错误保留。
- 非终态必须 `RetainOrphans` 保留 handle；零拷贝下调用方 buffer 的生命周期由调用方保证（见 §2.2）。

---

## 7. 步骤 6：`GetBatchAsync` + 完成句柄

### 7.1 接口契约

`object_storage_adapter.h` 新增：

```cpp
enum class ObjectReadState { kPending, kDone, kIndeterminate };

class ObjectReadCompletion {
   public:
    virtual ~ObjectReadCompletion() = default;
    virtual ObjectReadState State() const = 0;
    // 首次调用执行一次 SDK Wait；之后返回固定状态，不会再次消费 task。
    virtual ObjectReadState WaitFor(uint64_t timeout_ms) = 0;
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
| `kPending` | adapter 持有注册；buffer 归调用方但不得修改 | 等待 `WaitFor`，或把句柄上送 |
| `kDone` | adapter 已注销 handle；buffer 归还调用方 | 读 `Results()`，可复用 buffer |
| `kIndeterminate` | adapter 保留 handle；buffer 由调用方隔离 | 不得复用/释放 buffer，直到 adapter `Shutdown` |

- `kIndeterminate` 不会再变为 `kDone`：SDK `Wait` 已消费 task，且没有 cancel/query 接口；唯一安全释放点是 `~KvClientObjectStorageAdapter`。
- `WaitFor` 的第一次调用执行 `waitForCompletion(task_id, count, timeout_ms)`（`0` 表示 SDK 默认超时）；非终态立即转 `kIndeterminate` 并 `RetainOrphans`。
- 默认实现（KVCS/OSS/S3）返回 `kDone`，同步路径无行为变化。

### 7.2 适配器实现

```cpp
class KvClientObjectStorageAdapter::ReadCompletion final
    : public ObjectReadCompletion {
   public:
    ReadCompletion(KvClientObjectStorageAdapter* adapter, kv::TaskId task_id,
                   std::vector<std::optional<BufferPlan>> plans,
                   std::vector<size_t> indices,
                   std::vector<tl::expected<size_t, ErrorCode>> results)
        : adapter_(adapter), task_id_(task_id), plans_(std::move(plans)),
          indices_(std::move(indices)), results_(std::move(results)) {}

    ~ReadCompletion() override {
        // 调用方在 kPending 时丢弃句柄：SDK 任务仍可能引用 buffer，
        // 只能保留 handle。kIndeterminate 已在 WaitFor 中保留过。
        if (state_ == ObjectReadState::kPending) {
            adapter_->RetainOrphans(plans_);
        }
    }

    ObjectReadState State() const override { return state_; }

    ObjectReadState WaitFor(uint64_t timeout_ms) override {
        if (state_ != ObjectReadState::kPending) return state_;

        const auto completion = adapter_->waitForCompletion(
            task_id_, indices_.size(), timeout_ms);
        if (!completion.terminal) {
            state_ = ObjectReadState::kIndeterminate;
            adapter_->RetainOrphans(plans_);
            for (size_t i : indices_) {
                results_[i] = tl::make_unexpected(ErrorCode::RPC_TIMEOUT);
            }
            return state_;
        }
        if (completion.entryStatus.size() != indices_.size()) {
            for (size_t i : indices_) {
                results_[i] = tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
            }
        } else {
            for (size_t j = 0; j < indices_.size(); ++j) {
                const size_t i = indices_[j];
                results_[i] = completion.entryStatus[j].ok()
                                  ? tl::expected<size_t, ErrorCode>{plans_[i]->n}
                                  : tl::make_unexpected(
                                        ToErrorCode(completion.entryStatus[j]));
            }
        }
        adapter_->ReleaseBuffers(plans_);
        state_ = ObjectReadState::kDone;
        return state_;
    }

    std::vector<tl::expected<size_t, ErrorCode>> Results() override {
        return results_;
    }

   private:
    KvClientObjectStorageAdapter* adapter_;
    kv::TaskId task_id_;
    std::vector<std::optional<BufferPlan>> plans_;
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

    if (!initialized_ || !kv_client_) {
        FillError(results, ErrorCode::DFS_SERVICE_UNAVAILABLE);
        return MakeReadyCompletion(std::move(results));
    }

    std::vector<std::optional<BufferPlan>> plans;
    std::vector<ErrorCode> errors(requests.size(), ErrorCode::INVALID_PARAMS);
    auto registered = RegisterBuffers(
        std::span<const ObjectGetRequest>(requests), plans, errors);
    if (!registered) {
        FillError(results, ErrorCode::DFS_SERVICE_UNAVAILABLE);
        return MakeReadyCompletion(std::move(results));
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
                               plans[i]->n, -1, -1};
        entry.buffer.handle = plans[i]->handle;
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
        ReleaseBuffers(plans);
        return MakeReadyCompletion(std::move(results));
    }

    return std::make_unique<ReadCompletion>(
        this, task_id, std::move(plans), std::move(indices),
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
    completion->WaitFor(0);
    if (completion->State() != ObjectReadState::kDone) {
        // 同步契约无法把 buffer 隔离承诺传给调用方；kvclient 的主路径是
        // GetBatchAsync。这里返回超时，句柄析构时由 RetainOrphans 保留 handle。
        std::vector<tl::expected<size_t, ErrorCode>> results(requests.size(),
            tl::make_unexpected(ErrorCode::RPC_TIMEOUT));
        return results;
    }
    return completion->Results();
}
```

### 7.3 后端与 Client 传导

```cpp
struct DfsReadAsyncResult {
    // 已完成条目
    std::vector<tl::expected<void, ErrorCode>> results;
    // 非空表示还有未完成条目；持有 buffer/注册/多 slice staging 的所有权
    std::shared_ptr<ObjectReadCompletion> pending;
};

DfsReadAsyncResult DistributedStorageBackend::BatchReadAsync(
    const std::vector<DfsReadRequest>& requests);
```

- 多 slice 请求的 `staging_buffers` 由 `pending` 持有；完成时由 completion 的回调把 N 字节散回各 slice。
- `Client::BatchGet` / `ReadDfsReplica` / `RealClient` 增加完成句柄出口（异步变体或 out 参数）：
  - `kDone` → 散回并返回；
  - `kIndeterminate` → 返回 `RPC_TIMEOUT`，句柄交给 buffer 所有者；在 `Shutdown` 之前不得释放/复用该批 slices。
- 同步 `BatchRead` 仅作为兼容入口：内部调用 `BatchReadAsync`，`kIndeterminate` 时把 pending 放入后端的隔离表（到 Shutdown 清理），并向调用方返回 `RPC_TIMEOUT`。KVClient 的主路径必须使用 `BatchReadAsync`。

### 7.4 生命周期

- `GetBatchAsync` 返回后，buffer 与注册归完成句柄管理；调用方不得 `resize`/释放/复用。
- `kDone` 时 adapter 已注销 handle，`Results()` 有效。
- `kIndeterminate` 时 handle 保留、buffer 隔离；析构顺序为 `Shutdown()` → 清隔离表，避免在设备仍使用时注销 MR。
- 该契约要求 `Client`/`RealClient` 把句柄传到真正拥有 slices 的层；这是 PR2 的主要接口变更。
- 完成句柄不得长于 adapter：句柄持有 adapter 指针，`Client` 必须在 adapter 析构前消费或丢弃；若需要跨层长期持有，改为共享状态（`shared_ptr`）而不是裸指针。

---

## 8. 步骤 7：单条包装、释放、孤儿、析构

### 8.1 单条入口

```cpp
tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::Put(
    const std::string& key, std::span<const char> data) {
    iovec iov{const_cast<char*>(data.data()), data.size()};
    auto results = PutBatch({{key, &iov, 1}});
    return results.empty()
               ? tl::make_unexpected(ErrorCode::INVALID_PARAMS)
               : std::move(results.front());
}

tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::PutV(
    const std::string& key, const iovec* iov, int iovcnt) {
    auto results = PutBatch({{key, iov, iovcnt}});
    return results.empty()
               ? tl::make_unexpected(ErrorCode::INVALID_PARAMS)
               : std::move(results.front());
}

tl::expected<size_t, ErrorCode> KvClientObjectStorageAdapter::Get(
    const std::string& key, void* buf, size_t len) {
    auto completion = GetBatchAsync({{key, buf, len}});
    completion->WaitFor(0);
    if (completion->State() != ObjectReadState::kDone) {
        return tl::make_unexpected(ErrorCode::RPC_TIMEOUT);
    }
    auto results = completion->Results();
    return results.empty()
               ? tl::make_unexpected(ErrorCode::INVALID_PARAMS)
               : std::move(results.front());
}
```

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
        orphans_.push_back(plan->handle);
    }
    LOG(ERROR) << "action=kvclient_orphans_retained, total=" << orphans_.size();
}
```

约束：

- 零拷贝下只保留 handle；调用方 buffer 的生命周期由调用方负责。
- 零拷贝读写都不保留内存，只保留无法注销的 handle；超时进入隔离。

### 8.4 析构

```cpp
KvClientObjectStorageAdapter::~KvClientObjectStorageAdapter() {
    // 先停止受理并让 SDK 排空在途任务。
    if (kv_client_) kv_client_->Shutdown();

    std::lock_guard<std::mutex> lock(orphan_mu_);
    if (!orphans_.empty()) {
        LOG(ERROR) << "action=kvclient_orphans_released_at_shutdown, count="
                   << orphans_.size();
    }
    orphans_.clear();
}
```

> 若还有存活的完成句柄，它们不得在 adapter 析构后使用；`ReadCompletion` 的析构只负责在 `kPending` 时保留 handle，不负责延长 adapter 生命周期。

### 8.5 `Exists`

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
- `Wait` 超时 → `RPC_TIMEOUT`；查询不注册 buffer，没有需要保留的 handle/staging。
- 空 key 本地拒绝，不发起 SDK 查询。

### 8.6 保持 `NOT_SUPPORTED`

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

`master_service.cpp:4995-5003`，`AllocateReplicas` 内。写路径默认 512B 对齐，kvclient 与 KVCS 的 descriptor 形状一致：

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

- **不新增长度上限校验**：`MOONCAKE_KVCLIENT_MAX_OBJECT_BYTES` 已删除，超过 SDK 协议上限时由 `StoreAsync` 拒绝并映射为 `INVALID_PARAMS`。
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
| `WriteIsZeroCopy` | N=1024、地址 512 对齐：`RegisterRegions` 收到的 `addr` 等于调用方 buffer，`size==1024`；`StoreAsync` 的 region 同样指向调用方 buffer |
| `UnalignedWriteDelegatedToSdk` | N=513：适配器不拒绝、不补齐；fake `StoreAsync` 返回 `INVALID_ARGUMENT` 时结果映射为 `INVALID_PARAMS` |
| `ContinuityNotValidated` | 两段不相邻 iov：适配器不拒绝；只按首地址与长度之和注册一个 region（调用方契约保证连续） |
| `RejectsEmptyKeyZeroLengthAndDuplicate` | 空 key / `size==0` 逐项 `INVALID_PARAMS`；同批重复 key 第一条成功、后续 `INVALID_PARAMS` |
| `RegisterFailureFailsWholeBatch` | `RegisterRegions` 返回 `PARTIAL_FAILED`：全批 `DFS_SERVICE_UNAVAILABLE`；`StoreAsync` 未调用 |
| `TimeoutKeepsHandles` | 写 `Wait` 返回 `TIMEOUT`：全批 `RPC_TIMEOUT`，不出现 `OBJECT_NOT_FOUND`；`UnregisterRegions` 未调用；析构不崩溃 |
| `ReadDone` | `GetBatchAsync` → `WaitFor` 返回 `kDone`；`Results()` 为 N；`UnregisterRegions` 已调用 |
| `ReadSubmitFailure` | `LoadAsync` 提交失败：返回 `kDone` 句柄，逐项错误；调用方 buffer 未被写 |
| `ReadIndeterminate` | `Wait` 返回 `TIMEOUT`：状态为 `kIndeterminate`；句柄保留、`UnregisterRegions` 未调用；`Results()` 为 `RPC_TIMEOUT` |
| `ReadCompletionDropped` | 丢弃 `kPending` 句柄：`RetainOrphans` 保留 handle，不注销、不崩溃 |
| `ExistsReturnsEntryCode` | `QueryAsync` 的 entry 为 `ENTRY_KEY_EXIST` / `ENTRY_KEY_NOT_EXIST` → `true` / `false`；不使用 `ok()` |
| `ExistsErrors` | 空 key → `INVALID_PARAMS`（零 SDK 调用）；提交失败/超时/entry 错误 → 对应 `ErrorCode` |
| `OverwriteDoesNotPreDelete` | 连续两次 `Put` 同一 key：调用序列中没有 `DeleteAsync`；两次都 `StoreAsync` |
| `UninitializedBatchFailsSafely` | 未 `Init` 时 `PutBatch`/`GetBatchAsync` 返回 `DFS_SERVICE_UNAVAILABLE`，SDK 方法零调用 |
| `SharedBufferRegisteredOnce` | 两个请求指向同一 `(addr, N)`：`RegisterRegions` 只收到 1 个 region；终态 `UnregisterRegions` 只收到 1 个 handle |

### 10.4 测试环境约束

- 测试只用 fake client，不需要真实设备或 view service。
- 注入构造 `KvClientObjectStorageAdapter(config, std::make_unique<FakeKvClient>())` 后调用 `Init()`：注入路径只校验配置、不调用 SDK `Init`。
- 零拷贝用例的 buffer 用普通 `std::vector<char>` 即可；适配器不校验对齐，对齐由调用方契约与 fake SDK 决定。
- 完成句柄用例通过 fake `Wait` 注入 `OK` / `TIMEOUT`，并断言 `UnregisterRegions` 的调用序列；`kPending` 丢弃用例检查 `RetainOrphans` 后析构不崩溃。

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
# 写路径调用方保证 512B 对齐与连续；读路径注册调用方 buffer
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
  - 写路径零拷贝；调用方保证 512B 对齐与连续，适配器不校验；不 staging、不补齐，成功由 KvClient 判定；
  - 写路径超时返回后调用方 buffer 不得立即复用；
  - 读路径注册调用方 buffer，`GetBatchAsync` + `ObjectReadCompletion`；完成前 buffer/注册不得释放，`kIndeterminate` 隔离到 `Shutdown`；
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
| 写路径 | 零拷贝提交；`RegisterRegions` 指向调用方 buffer；不校验对齐/连续性 |
| 写失败判定 | 未对齐/超上限由 SDK 拒绝，映射为 `INVALID_PARAMS`，适配器不补齐 |
| 读路径 | `GetBatchAsync` 返回完成句柄；`kDone` 返回 N；`kIndeterminate` 隔离 buffer/handle |
| `Exists` | `QueryAsync` 的 `ENTRY_KEY_EXIST` / `ENTRY_KEY_NOT_EXIST` → `true` / `false`；超时/错误正确映射；`ListKeys` 仍 `NOT_SUPPORTED` |
| 注册 | `(addr, N)` 去重；失败整批 `DFS_SERVICE_UNAVAILABLE` |
| 超时 | 写：全批 `RPC_TIMEOUT`；读：`kIndeterminate`；handle 保留、不注销 |
| 覆盖写 | 不产生 `DeleteAsync` |
| 守卫 | Init 前调用逐项 `DFS_SERVICE_UNAVAILABLE` |
| 配置 | 不校验已删除的两个 env |
| Master 启动 | kvclient 早退分支生效；oplog 时 FATAL |
| descriptor | `object_size == aligned_size == N`；`shard_idx=-1` |
| Master 门控 | NOF 拒绝、DFS 可用性门、驱逐白名单生效 |
| 恢复门 | 保持 KVCS-only |
| 测试 | 16 个用例全部通过；SDK 缺失构建不受影响 |
| 回归 | 既有 KVCS / OSS / S3 / DFS 测试全绿 |

---

## 14. 实现约束速查

1. **写路径**：零拷贝；不 staging、不补齐、不校验对齐/连续性；成功由 KvClient 判定。
2. **读路径**：注册调用方 buffer；`GetBatchAsync` 返回完成句柄；只有 `kDone` 才能复用 buffer。
3. **注册**：按 `(addr, N)` 去重；输出条数必须等于输入条数。
4. **释放**：handle 去重后一次 `UnregisterRegions`；只在 `kDone` 或提交失败时调用。
5. **非终态**：写全批 `RPC_TIMEOUT`；读 `kIndeterminate`，绝不返回 `OBJECT_NOT_FOUND`。
6. **buffer 生命周期**：`kPending` / `kIndeterminate` 期间调用方不得复用；句柄传导到 buffer 所有者。
7. **覆盖写**：不预删；不引入 `DeleteAsync` + `StoreAsync`。
8. **守卫**：`PutBatch` / `GetBatchAsync` 入口检查 `initialized_` / `kv_client_`。
9. **析构**：先 `Shutdown()`，再清 `orphans_`/隔离表。
10. **Master 初始化**：kvclient 早退分支必须在 filesystem allocator 之前。
11. **descriptor**：与 KVCS 相同；`aligned_size = value_length`。
12. **错误码**：DFS 可用性门保持 `DFS_SERVICE_UNAVAILABLE`。
13. **恢复门**：不改，保持 KVCS-only。
14. **配置**：只保留 `config_path`；不校验 max object bytes / wait timeout。
15. **完成句柄**：`ObjectStorageAdapter` → `BatchReadAsync` → `Client`/`RealClient` 全链路传递，不得在中间层丢弃。
16. **`Exists`**：`QueryAsync` + `Wait(0)`，按 entry 级 `ENTRY_KEY_EXIST` / `ENTRY_KEY_NOT_EXIST` 返回；不能用 `ok()`；不注册 buffer。

---

## 15. 已知风险与待确认

| 风险 | 处理 |
| --- | --- |
| 零拷贝写超时后设备可能仍在读取调用方 buffer | 适配器无法保护；PR 描述明确调用方不得立即复用；PR3 的 Client 集成需落实 |
| 读 `kIndeterminate` 会永久占用 handle 与调用方 buffer | SDK 无 cancel/query；隔离到 `Shutdown`，需要监控与容量规划 |
| 完成句柄必须传导到 buffer 所有者 | `BatchReadAsync`、`Client`/`RealClient` 任一层丢弃句柄都会导致提前释放；PR 评审重点 |
| 完成句柄持有 adapter 裸指针 | 句柄不得长于 adapter；跨层长期持有需改为共享状态，否则 UAF |
| 并发 batch 注册同一 4KB 页可能触发 provider host-mapping 冲突 | PR2 不做引用计数；压测验证 |
| orphan handle 无界增长 | PR2 只有 ERROR 日志；指标与上限留后续 |
| XXH64 非单射 | 已接受；日志记录 `cache_key` |
| 未对齐/超上限对象由 SDK 拒绝 | Master 不再拦截；Client 侧会触发 PutRevoke，属预期 |
| `CheckHealth` 未实现 | `MOONCAKE_DISTRIBUTED_HEALTH_CHECK=1` 时 Init 失败；部署必须关闭 |
| 端到端写依赖调用方显式 `dfs_replica_num=1` | PR3 的 `AttachHostId` 才自动补 |
| `EvictDiskReplica` / `BatchEvictDiskReplica` 的 DFS 分支不可用 | PR2 明确不改；只验证 `EvictDiskReplicaIfCurrent` |
