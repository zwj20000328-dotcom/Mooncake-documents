# Mooncake + KVClient L3 接入（Kvcs）— 三个 PR 的实施设计

版本：v1.3　更新日期：2026-11-18
对应设计文档：`Mooncake + KVClient L3 接入（Kvcs）- v3.md` v4.6
代码核对基线：`Mooncake` main `46c5eca8`

本文把落地工作拆成三个 PR，每个 PR 给出类图、流程图、时序图与关键伪代码。三个 PR 按顺序合入，合并后仓库始终可构建。

| PR | 模块 | 标题 | 依赖 | 预算 |
| --- | --- | --- | --- | --- |
| 1 | M1 配置构建 + 适配器控制面 | `[Store] Add kvclient backend wiring and adapter control plane` | 无 | 495 |
| 2 | M2 适配器数据面 + Master | `[Store] Implement kvclient batch data path and master gates` | PR1 | 490 |
| 3 | M3 Client 集成 + 测试 | `[Store] Adopt value-first delete for KVClient and add adapter tests` | PR2 | 460 |

---

## PR1：配置构建 + 适配器控制面

### 1.1 目标

让 `fs_adapter_type=kvclient` 在运行时可达：配置能解析、工厂能创建适配器、FileStorage/RealClient 能注入 DFS 后端、SDK 缺失失败。适配器只落地控制面（`Init`、等待、错误映射、删除），读写方法以 `NOT_SUPPORTED` 占位，PR2 替换。

### 1.2 改动文件

| 文件 | 改动 |
| --- | --- |
| `mooncake-common/include/environment_variables.h` | 新增 `KvClientEnvironmentVariables`（三个 env 变量声明） |
| `include/storage/distributed/kvclient/kvclient_config.h` | 新增，header-only：`kKvClientMaxValueBytes`、`KvClientAdapterConfig`、`FromEnvironment()`、`Validate()` |
| `include/storage/distributed/kvclient/kvclient_adapter_factory.h` | 新增，厂商无关工厂声明 |
| `include/storage/distributed/kvclient/kvclient_object_storage_adapter.h` | 新增，类定义（`kKvClientAdapterName`、注入构造、全部纯虚 override；数据方法占位；含 `Completion` 与 `waitForCompletion`，删除路径依赖它） |
| `src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp` | 新增，控制面实现（`Init`、`Delete`、`DeleteBatch`、私有 `MakeCacheKey` 编码）+ 数据方法 NOT_SUPPORTED 占位 + 工厂实现 |
| `include/config/distributed_storage_config.h`、`src/config/distributed_storage_config.cpp` | `UsesKvClient()`、`Validate()` 分支、`UsesObjectStorage()` 列表 |
| `src/config/file_storage_config.cpp` | `Validate()` 跳过本地目录校验 |
| `include/storage/distributed/distributed_storage_backend.h` | 新增 `UsesKvClient()` |
| `src/storage_backend.cpp` | 工厂新增 `UsesKvClient()` 分支 |
| `src/file_storage.cpp` | `enable_dfs` 加入 `UsesKvClient()` |
| `src/real_client.cpp` | 新增 `enable_kvclient`；构造条件与 `kDistributed` 强制均加入 |
| `src/CMakeLists.txt` | pkg-config / `find_path`+`find_library` 检测、条件源、define/link（具体落点见 1.6.6） |
| `tests/distributed_storage_config_test.cpp` | 新增 kvclient 解析、校验、allocator 与 SDK 缺失工厂用例 |
| `tests/CMakeLists.txt` | SDK 缺失时给配置测试定义 `MOONCAKE_KVCLIENT_TEST_NO_SDK=1` |

### 1.3 类图

```mermaid
classDiagram
    direction TB

    class KvClientAdapterConfig {
        +string config_path
        +uint64 max_object_bytes
        +uint64 wait_timeout_ms
        +FromEnvironment()$
        +Validate() bool
    }

    class Completion {
        +bool terminal
        +vector~Status~ entryStatus
    }

    class KvClientObjectStorageAdapter {
        -KvClientAdapterConfig config_
        -unique_ptr~KvClient~ kv_client_
        -bool initialized_
        +KvClientObjectStorageAdapter(config) 构造
        +KvClientObjectStorageAdapter(config, client) 测试注入构造
        +KvClientObjectStorageAdapter() 析构
        +Init() expected
        +GetName() const char*
        +Exists(key) expected
        +ListKeys() expected
        +Delete(key) expected
        +DeleteBatch(keys) results
        -MakeCacheKey(key) BinaryKey
        -waitForCompletion(task_id, count) Completion
        +Put(key, data) expected
        +PutV(key, iov, iovcnt) expected
        +PutBatch(requests) results
        +Get(key, buf, len) expected
        +GetBatch(requests) results
        -ToErrorCode(status) ErrorCode$
    }

    KvClientObjectStorageAdapter ..> Completion : 返回

    class SdkKvClient["kv::KvClient"] {
        +Init(cfg)
        +StoreAsync(...)
        +LoadAsync(...)
        +DeleteAsync(...)
        +QueryAsync(...)
        +Wait(task_id, timeout, result)
        +RegisterRegions(...)
        +UnregisterRegions(...)
        +Shutdown()
    }

    class Factory["CreateKvClientObjectStorageAdapter()"] {
        +returns expected~ObjectStorageAdapter~
    }

    KvClientObjectStorageAdapter *-- SdkKvClient : 持有
    KvClientObjectStorageAdapter ..> KvClientAdapterConfig : 读取
    Factory ..> KvClientObjectStorageAdapter : 创建（SDK 存在）
```

> 数据面只有 batch 一条路径：`Put` / `PutV` / `Get` 都只是把单条请求包装成 `PutBatch` / `GetBatch` 调用，不单独实现 I/O；`Delete` 同样只是 `DeleteBatch` 的单条包装。PR1 实现 `Delete` 与 `DeleteBatch`（控制面删除），`Put` / `PutV` / `PutBatch` / `Get` / `GetBatch` 以 `NOT_SUPPORTED` 占位，PR2 用单行包装替换。`GetName()` 的真实签名是 `const char* GetName() const override`，不是 `char*`；适配器头必须 override `object_storage_adapter.h` 的全部纯虚函数（`Put` / `PutV` / `Get` / `Exists` / `Delete` / `ListKeys` / `Init` / `GetName`），并声明析构 `~KvClientObjectStorageAdapter() override`，否则类保持抽象、`make_unique` 无法编译。
>
> **PR2 实现注意**：`Completion` 是 PR1 已存在的类内私有嵌套类型，图中仅示意，PR2 **不要**重新声明；`BufferPlan` 也必须是类内私有类型（`RegisterBuffers` 的签名依赖它）。`orphans_` 的真实类型是 `std::vector<std::pair<std::shared_ptr<char[]>, kv::MRHandle>>`，拆解后只清内存、不重复注销 handle。

### 1.4 流程图

```mermaid
flowchart TB
    ENV["环境变量：MOONCAKE_KVCLIENT_*"] --> CFG["KvClientAdapterConfig::FromEnvironment()"]
    CFG --> VAL{"Validate()"}
    VAL -->|失败| ERR1["工厂返回 INVALID_PARAMS"]
    VAL -->|通过| FOUND{"CMake: MOONCAKE_KVCLIENT_SDK_FOUND?"}
    FOUND -->|否| STUB["storage_backend.cpp 的 #else 分支<br/>直接返回 NOT_SUPPORTED（不编适配器 TU）"]
    FOUND -->|是| CREATE["make_unique<KvClientObjectStorageAdapter>(cfg)"]
    CREATE --> BACKEND["DistributedStorageBackend(cfg, dcfg, nullptr, adapter)"]
    BACKEND --> INJECT["FileStorage 构造：SetDfsStorageBackend(backend)<br/>enable_dfs = !UsesObjectStorage() || UsesKvcs() || UsesKvClient()"]
    INJECT --> INIT["FileStorage::Init → Backend::Init → Adapter::Init"]
    INIT --> OK["后端可用；读写方法暂 NOT_SUPPORTED"]
```

### 1.5 时序图

```mermaid
sequenceDiagram
    autonumber
    participant RC as RealClient
    participant FS as FileStorage
    participant F as CreateStorageBackend
    participant AD as KvClientObjectStorageAdapter
    participant BE as DistributedStorageBackend
    participant SDK as kv::KvClient
    participant C as Client

    RC->>FS: 构造（UsesKvClient 强制 kDistributed）
    FS->>F: CreateStorageBackend(config)
    F->>F: DistributedStorageConfig::Validate()
    F->>AD: CreateKvClientObjectStorageAdapter()
    AD->>AD: KvClientAdapterConfig::FromEnvironment().Validate()
    F->>BE: DistributedStorageBackend(cfg, dcfg, nullptr, adapter)
    F-->>FS: backend
    FS->>C: SetDfsStorageBackend(backend)
    RC->>FS: Init()
    FS->>BE: Init()
    BE->>AD: Init()
    AD->>AD: kv::CreateKvClient()
    AD->>AD: KvClientConfig::LoadFromFile(path, cfg)
    AD->>AD: cfg.keyResolution = {}（强制 GENERIC）
    AD->>AD: 校验 cfg.timeoutMs >= wait_timeout_ms
    AD->>SDK: Init(cfg)
    SDK-->>AD: Status
    AD-->>BE: OK
    BE-->>FS: OK
```

### 1.6 伪代码

#### 1.6.1 配置

```cpp
// mooncake-common/include/environment_variables.h
struct KvClientEnvironmentVariables {
    MC_DEFINE_ENV_VAR(std::string, MOONCAKE_KVCLIENT_CONFIG);
    MC_DEFINE_ENV_VAR(uint64_t, MOONCAKE_KVCLIENT_MAX_OBJECT_BYTES);
    MC_DEFINE_ENV_VAR(uint64_t, MOONCAKE_KVCLIENT_WAIT_TIMEOUT_MS);
};

// kvclient_config.h（header-only，不包含 SDK 头）
inline constexpr uint64_t kKvClientMaxValueBytes = 16'776'704;
// 与 SDK 的 kAlignmentBytes 同值；本地声明，避免配置单元依赖 SDK 头。
inline constexpr size_t kKvClientAlignment = 512;

struct KvClientAdapterConfig {
    std::string config_path;
    uint64_t max_object_bytes = kKvClientMaxValueBytes;
    uint64_t wait_timeout_ms = 30'000;

    // 数值变量必须严格解析：Environ::ReadOr 对 "abc"/"-1" 只打 stderr
    // 警告并回退默认值，会让非法配置静默通过校验。
    static tl::expected<uint64_t, ErrorCode> ReadUint(
        const EnvironmentVariable<uint64_t>& variable, uint64_t default_value) {
        const auto raw = Environ::Process().Get(variable.name);
        if (!raw.has_value()) return default_value;          // 未设置：用默认值
        const auto parsed = Environ::Process().GetTyped(variable);
        if (!parsed.has_value()) {
            LOG(ERROR) << "Invalid value for env " << variable.name << ": '"
                       << *raw << "'";
            return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
        }
        return *parsed;
    }

    static tl::expected<KvClientAdapterConfig, ErrorCode> FromEnvironment() {
        KvClientAdapterConfig cfg;
        cfg.config_path = Environ::ReadOr(
            KvClientEnvironmentVariables::MOONCAKE_KVCLIENT_CONFIG,
            std::string{});
        auto max_object_bytes = ReadUint(
            KvClientEnvironmentVariables::MOONCAKE_KVCLIENT_MAX_OBJECT_BYTES,
            kKvClientMaxValueBytes);
        if (!max_object_bytes)
            return tl::make_unexpected(max_object_bytes.error());
        cfg.max_object_bytes = *max_object_bytes;
        auto wait_timeout_ms = ReadUint(
            KvClientEnvironmentVariables::MOONCAKE_KVCLIENT_WAIT_TIMEOUT_MS,
            uint64_t{30'000});
        if (!wait_timeout_ms)
            return tl::make_unexpected(wait_timeout_ms.error());
        cfg.wait_timeout_ms = *wait_timeout_ms;
        if (!cfg.Validate()) return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
        return cfg;
    }

    bool Validate() const {
        return !config_path.empty() &&
               max_object_bytes > 0 &&
               max_object_bytes % kKvClientAlignment == 0 &&
               max_object_bytes <= kKvClientMaxValueBytes &&
               wait_timeout_ms > 0;
    }
};
```

> `wait_timeout_ms` 默认 30 s，而 SDK 的 `timeoutMs` / `defaultWaitTimeoutMs` 默认只有 100 ms；`Init` 的硬校验要求 `cfg.timeoutMs >= wait_timeout_ms`。因此部署侧必须在 SDK client 配置里把 `waitTimeoutMs` / `timeoutMs` 调到至少 `MOONCAKE_KVCLIENT_WAIT_TIMEOUT_MS`（默认 30000），否则 `Init` 返回 `INVALID_PARAMS`。`Init` 失败日志必须同时打印 `cfg.timeoutMs` 与 `wait_timeout_ms`，便于定位。

#### 1.6.2 key 编码（适配器私有方法）

编码逻辑只存在于 `KvClientObjectStorageAdapter::MakeCacheKey`，不单独建头文件：

```cpp
// kvclient_object_storage_adapter.cpp（匿名命名空间）
inline constexpr uint64_t kKvClientHashSeed = 5381;   // 与 SDK key_resolver 一致

kv::BinaryKey KvClientObjectStorageAdapter::MakeCacheKey(
    std::string_view logical_key) const {
    uint64_t hash =
        XXH64(logical_key.data(), logical_key.size(), kKvClientHashSeed);
    if (hash == 0) hash = kKvClientHashSeed;   // 全零映射为非零常量
    kv::BinaryKey key{};
    // 显式小端逐字节写入，与 SDK key_resolver 的打包方式一致；
    // 不用 memcpy（主机字节序），否则大小端客户端会得到不同的 CacheKey。
    for (size_t i = 0; i < sizeof(hash); ++i)
        key[i] = static_cast<std::byte>(hash >> (8 * i));
    return key;
}
```

> 三个 wire 值必须同时固定：seed `5381`、`hash == 0` 的替代值 `5381`（不是 `1`）、以及小端字节序。SDK 的 `key_resolver` 对 string key 使用 `XXH64(..., 5381)` 后逐字节取低 8 位，见 `kv_semantics/src/client/key_resolver.cc:51-55`；Mooncake 只传 `CacheKey::FromBytes`，不走 SDK 的 string 路径。

#### 1.6.3 工厂与 SDK 缺失分支

```cpp
// kvclient_adapter_factory.h（厂商无关）
tl::expected<std::unique_ptr<ObjectStorageAdapter>, ErrorCode>
CreateKvClientObjectStorageAdapter();

// storage_backend.cpp（工厂分支；SDK 缺失时不引用任何 SDK 符号）
// 只允许包含厂商无关的 kvclient_adapter_factory.h；不得包含适配器头或 key codec。
#if defined(MOONCAKE_HAVE_KVCLIENT_SDK)
#include "storage/distributed/kvclient/kvclient_adapter_factory.h"
#endif
...
// 接在既有 UsesKvcs() 分支之后；object_storage_adapter 变量在分支前已声明。
} else if (distributed_config.UsesKvClient()) {
#ifdef MOONCAKE_HAVE_KVCLIENT_SDK
    auto adapter = CreateKvClientObjectStorageAdapter();
    if (!adapter) return tl::make_unexpected(adapter.error());
    object_storage_adapter = std::move(*adapter);
#else
    LOG(ERROR) << "KVClient SDK not built in; kvclient backend unavailable";
    return tl::make_unexpected(ErrorCode::NOT_SUPPORTED);
#endif
}

// 与既有分支共用收尾：工厂返回的是 ObjectStorageAdapter，
// 必须包装进 DistributedStorageBackend；不能直接 return 工厂结果。
return std::make_shared<DistributedStorageBackend>(
    config, distributed_config, std::move(fs_adapter),
    std::move(object_storage_adapter));

// kvclient_object_storage_adapter.cpp（SDK 存在时才参与编译）
tl::expected<std::unique_ptr<ObjectStorageAdapter>, ErrorCode>
CreateKvClientObjectStorageAdapter() {
    auto config = KvClientAdapterConfig::FromEnvironment();
    if (!config) return tl::make_unexpected(config.error());
    return std::make_unique<KvClientObjectStorageAdapter>(*config);
}
```

#### 1.6.4 控制面

适配器头必须声明名称常量、注入构造、析构，并 override 全部纯虚函数：

```cpp
inline constexpr char kKvClientAdapterName[] = "kvclient";
```

`Init` 的关键是**把 client 创建到局部变量，成功后才写入成员**。SDK `Init` 非幂等（第二次调用返回 `RESOURCE_BUSY`），因此每次重试都必须重新执行配置加载与校验，任何失败路径都不能留下"非空但未初始化"的 `kv_client_`。

```cpp
tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::Init() {
    if (initialized_) return {};
    if (!config_.Validate()) return tl::make_unexpected(ErrorCode::INVALID_PARAMS);

    // 注入构造已提供 client（测试缝）：只校验适配器配置，跳过 SDK 级初始化。
    if (kv_client_) {
        initialized_ = true;
        return {};
    }

    auto client = kv::CreateKvClient();      // 先建到局部变量，成功后才赋值
    if (!client) return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);

    kv::KvClientConfig cfg;
    if (!kv::KvClientConfig::LoadFromFile(config_.config_path, cfg).ok())
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    cfg.keyResolution = {};                  // 强制 GENERIC
    if (cfg.timeoutMs < config_.wait_timeout_ms) {
        LOG(ERROR) << "KVClient config timeoutMs=" << cfg.timeoutMs
                   << " is shorter than wait_timeout_ms="
                   << config_.wait_timeout_ms;
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    // Init 非幂等；局部 client 只在成功后提交，重试会从新的 client 开始。
    if (!client->Init(cfg).ok())
        return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);

    kv_client_ = std::move(client);          // 仅在此处提交，失败不会污染成员
    initialized_ = true;
    LOG(INFO) << "KVClient adapter initialized, max_object_bytes="
              << config_.max_object_bytes;
    return {};
}

KvClientObjectStorageAdapter::~KvClientObjectStorageAdapter() {
    // 没有独立的 Shutdown()：先停止受理，再由 SDK 排空 pending task。
    if (kv_client_) kv_client_->Shutdown();
}

ErrorCode KvClientObjectStorageAdapter::ToErrorCode(const kv::Status& s) {
    using kv::StatusCode;
    switch (s.code) {
        case StatusCode::OK: return ErrorCode::OK;
        case StatusCode::TIMEOUT:
        case StatusCode::CQE_IO_TIMEOUT: return ErrorCode::RPC_TIMEOUT;
        case StatusCode::IN_PROGRESS:
        case StatusCode::CANCELED: return ErrorCode::INTERNAL_ERROR;

        // 对象 miss：只有 entry 级 key-not-found 才是 OBJECT_NOT_FOUND。
        case StatusCode::CQE_KEY_NOT_EXIST:
        case StatusCode::ENTRY_KEY_NOT_FOUND:
        case StatusCode::ENTRY_KEY_NOT_EXIST: return ErrorCode::OBJECT_NOT_FOUND;

        // NOT_FOUND 是路由/配置/视图错误，不是对象 miss；
        // 映射为 OBJECT_NOT_FOUND 会让 PR3 误删元数据、PR2 误触发 miss 清理。
        case StatusCode::NOT_FOUND:
        case StatusCode::NOT_INITIALIZED:
        case StatusCode::CONNECTION_ERROR:
        case StatusCode::IO_ERROR:
        case StatusCode::UNSUPPORTED:
        // Delete 的逐项失败码；ENTRY_KEY_NOT_EXIST 是 QUERY 专用。
        case StatusCode::ENTRY_DELETE_FAILED:
            return ErrorCode::DFS_SERVICE_UNAVAILABLE;

        case StatusCode::INVALID_ARGUMENT:
        case StatusCode::BUFFER_NOT_REGISTERED:
        case StatusCode::BUFFER_NOT_SUPPORTED: return ErrorCode::INVALID_PARAMS;
        case StatusCode::CQE_OUT_OF_CREATE_SIZE:
        case StatusCode::RESOURCE_BUSY:
        case StatusCode::CQE_RESOURCE_BUSY: return ErrorCode::NO_AVAILABLE_HANDLE;
        default: return ErrorCode::INTERNAL_ERROR;
    }
}

KvClientObjectStorageAdapter::Completion
KvClientObjectStorageAdapter::waitForCompletion(kv::TaskId task_id,
                                                size_t entry_count) {
    kv::TaskResult result;
    const auto wait_status =
        kv_client_->Wait(task_id, config_.wait_timeout_ms, result);
    const auto code = wait_status.ok() ? result.status.code : wait_status.code;
    const kv::Status& task_status = wait_status.ok() ? result.status : wait_status;

    Completion c;
    switch (code) {
        case kv::StatusCode::TASK_NOT_FOUND:
            // Wait 在超时后也会移除 task；未知 task 不能证明底层 I/O 已结束。
            // 保守地按非终态处理，让调用方保留 handle/staging。
            c.terminal = false;
            c.entryStatus.assign(
                entry_count,
                kv::Status::Error(kv::StatusCode::INTERNAL_ERROR,
                                  "task not found"));
            return c;
        case kv::StatusCode::TIMEOUT:
        case kv::StatusCode::IN_PROGRESS:
        case kv::StatusCode::CANCELED:
            // SDK 在返回超时前可能已填充 entryStatus，但 task 未终态；
            // 逐项结果不可信，统一用 task 级状态填充并保留资源。
            c.terminal = false;
            c.entryStatus.assign(entry_count, task_status);
            return c;
        default:
            // task 已结束：OK / PARTIAL_FAILED，以及 INVALID_ARGUMENT /
            // NOT_INITIALIZED / CONNECTION_ERROR / IO_ERROR / RESOURCE_BUSY
            // 等 task 级最终错误，都算终态。
            c.terminal = true;
            break;
    }

    if (result.entryStatus.size() != entry_count) {
        c.entryStatus.assign(
            entry_count,
            kv::Status::Error(kv::StatusCode::INTERNAL_ERROR,
                              "entry count mismatch"));
        return c;
    }
    c.entryStatus = std::move(result.entryStatus);
    return c;
}
```

#### 1.6.5 删除（PR1 已落地的唯一数据方法）

```cpp
ObjectStorageIoResults KvClientObjectStorageAdapter::DeleteBatch(
    std::span<const std::string> keys) {
    // Init 前或 Init 失败后直接调用：逐项失败，不解引用空 client。
    if (!initialized_ || !kv_client_) {
        return ObjectStorageIoResults(
            keys.size(),
            tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE));
    }
    ObjectStorageIoResults results(keys.size());
    if (keys.empty()) return results;

    std::vector<kv::CacheKey> cache_keys;
    std::vector<size_t> indices;
    for (size_t i = 0; i < keys.size(); ++i) {
        if (keys[i].empty()) {
            results[i] = tl::make_unexpected(ErrorCode::INVALID_PARAMS);
            continue;
        }
        cache_keys.push_back(kv::CacheKey::FromBytes(MakeCacheKey(keys[i])));
        indices.push_back(i);
    }
    if (cache_keys.empty()) return results;

    kv::TaskId task_id = kv::kInvalidTaskId;
    const auto submit = kv_client_->DeleteAsync(cache_keys, task_id);
    if (!submit.ok()) {
        for (size_t i : indices) results[i] = tl::make_unexpected(ToErrorCode(submit));
        return results;
    }
    const auto completion = waitForCompletion(task_id, cache_keys.size());
    for (size_t j = 0; j < indices.size(); ++j) {
        results[indices[j]] = completion.entryStatus[j].ok()
                                  ? tl::expected<void, ErrorCode>{}
                                  : tl::make_unexpected(
                                        ToErrorCode(completion.entryStatus[j]));
    }
    return results;
}

// Delete 只是 DeleteBatch 的单条包装；PR1 即落地，不属于 NOT_SUPPORTED 占位。
tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::Delete(
    const std::string& key) {
    std::array<std::string, 1> keys{key};
    auto results = DeleteBatch(keys);
    return results.empty()
               ? tl::make_unexpected(ErrorCode::INVALID_PARAMS)
               : std::move(results.front());
}
```

#### 1.6.6 构建（CMake）

四个落点必须分开，不能把 KVCS 的写法整体照搬：

```cmake
# ① 检测块：与 src/CMakeLists.txt 的 KVCS 检测块并列
set(MOONCAKE_KVCLIENT_SDK_FOUND OFF)
set(MOONCAKE_KVCLIENT_SDK_ROOT "" CACHE PATH
    "KVClient SDK source tree or install prefix")
option(MOONCAKE_KVCLIENT_REQUIRED
       "Fail configuration when the KVClient SDK is not found" OFF)

find_package(PkgConfig QUIET)
if(PkgConfig_FOUND)
  pkg_check_modules(KVCLIENT QUIET IMPORTED_TARGET GLOBAL "kvclient")
endif()

if(TARGET PkgConfig::KVCLIENT)
  add_library(Mooncake::kvclient INTERFACE IMPORTED GLOBAL)
  set_target_properties(Mooncake::kvclient PROPERTIES
                        INTERFACE_LINK_LIBRARIES PkgConfig::KVCLIENT)
  set(MOONCAKE_KVCLIENT_SDK_FOUND ON)
  message(STATUS "KVClient SDK found via pkg-config: kvclient backend enabled")
else()
  set(_kvclient_hints)
  if(MOONCAKE_KVCLIENT_SDK_ROOT)
    list(APPEND _kvclient_hints "${MOONCAKE_KVCLIENT_SDK_ROOT}")
  endif()
  if(DEFINED ENV{MOONCAKE_KVCLIENT_SDK_ROOT})
    list(APPEND _kvclient_hints "$ENV{MOONCAKE_KVCLIENT_SDK_ROOT}")
  endif()
  find_path(MOONCAKE_KVCLIENT_INCLUDE_DIR
            NAMES kv_client.h
            HINTS ${_kvclient_hints}
            PATH_SUFFIXES include kv_semantics/include include/kv_semantics)
  find_library(MOONCAKE_KVCLIENT_LIBRARY
               NAMES kv_client
               HINTS ${_kvclient_hints}
               PATH_SUFFIXES lib lib64 kv_semantics/src ucm/store/asu src
                             build/src build)
  find_library(MOONCAKE_KVCLIENT_TRANSPORT_LIBRARY
               NAMES kv_transport
               HINTS ${_kvclient_hints}
               PATH_SUFFIXES lib lib64 kv_semantics/src ucm/store/asu src
                             build/src build)
  if(MOONCAKE_KVCLIENT_INCLUDE_DIR AND MOONCAKE_KVCLIENT_LIBRARY)
    add_library(Mooncake::kvclient UNKNOWN IMPORTED GLOBAL)
    set(_kvclient_link_libraries "${MOONCAKE_KVCLIENT_LIBRARY}")
    if(MOONCAKE_KVCLIENT_TRANSPORT_LIBRARY)
      list(APPEND _kvclient_link_libraries
           "${MOONCAKE_KVCLIENT_TRANSPORT_LIBRARY}")
    endif()
    set_target_properties(
      Mooncake::kvclient
      PROPERTIES IMPORTED_LOCATION "${MOONCAKE_KVCLIENT_LIBRARY}"
                 INTERFACE_INCLUDE_DIRECTORIES
                 "${MOONCAKE_KVCLIENT_INCLUDE_DIR}"
                 INTERFACE_LINK_LIBRARIES "${_kvclient_link_libraries}")
    set(MOONCAKE_KVCLIENT_SDK_FOUND ON)
    message(STATUS "KVClient SDK found: include=${MOONCAKE_KVCLIENT_INCLUDE_DIR}"
                   ", library=${MOONCAKE_KVCLIENT_LIBRARY}")
  endif()
endif()

if(NOT MOONCAKE_KVCLIENT_SDK_FOUND)
  if(MOONCAKE_KVCLIENT_REQUIRED OR MOONCAKE_KVCLIENT_SDK_ROOT)
    message(
      FATAL_ERROR
        "KVClient SDK not found (MOONCAKE_KVCLIENT_REQUIRED=ON or "
        "MOONCAKE_KVCLIENT_SDK_ROOT was set). Set MOONCAKE_KVCLIENT_SDK_ROOT "
        "to a source tree or install prefix that contains kv_client.h and "
        "libkv_client, or set MOONCAKE_KVCLIENT_INCLUDE_DIR and "
        "MOONCAKE_KVCLIENT_LIBRARY.")
  endif()
  message(
    STATUS
      "KVClient SDK not found: adapter disabled. Set "
      "MOONCAKE_KVCLIENT_SDK_ROOT or pass -DMOONCAKE_KVCLIENT_REQUIRED=ON "
      "to make this fatal.")
endif()
set(MOONCAKE_KVCLIENT_SDK_FOUND
    ${MOONCAKE_KVCLIENT_SDK_FOUND}
    PARENT_SCOPE)

# ② 条件源：必须在 add_library(mooncake_store_client_objects ...) 之前
if(MOONCAKE_KVCLIENT_SDK_FOUND)
  list(APPEND MOONCAKE_STORE_CLIENT_SOURCES
       storage/distributed/kvclient/kvclient_object_storage_adapter.cpp)
endif()

# ③ define/link：storage_backend.cpp 与适配器 TU 都在 client objects 中。
#    挂到 mooncake_store_shared_objects 会让 storage_backend.cpp 永远走 #else。
if(MOONCAKE_KVCLIENT_SDK_FOUND)
  target_compile_definitions(mooncake_store_client_objects
                             PRIVATE MOONCAKE_HAVE_KVCLIENT_SDK=1)
  target_link_libraries(mooncake_store_client_objects
                        PRIVATE Mooncake::kvclient)
endif()

# ④ 最终库：OBJECT 库的 PRIVATE 链接不传播，mooncake_store 需要再链一次
if(MOONCAKE_KVCLIENT_SDK_FOUND)
  target_link_libraries(mooncake_store PRIVATE Mooncake::kvclient)
endif()
```

> - SDK 只安装 `libkv_client.so` / `libkv_transport.so`，**不安装头文件、不生成 `kvclient.pc`**。因此检测顺序是：pkg-config（未来兼容）→ `MOONCAKE_KVCLIENT_SDK_ROOT` / `MOONCAKE_KVCLIENT_INCLUDE_DIR` + `MOONCAKE_KVCLIENT_LIBRARY` 的 `find_path`/`find_library`。SDK 源码树可直接作为 root：头文件在 `kv_semantics/include`，库在 `build/src` 或安装目录 `kv_semantics/src`。`MOONCAKE_KVCLIENT_REQUIRED=ON` 或显式设置 `MOONCAKE_KVCLIENT_SDK_ROOT` 时，未找到 SDK 直接 `FATAL_ERROR`，避免"构建成功但适配器从未编译"掩盖编译错误。
> - `mooncake_store_master` 不链接 KVClient SDK。适配器源不得放进 `MOONCAKE_STORE_SHARED_SOURCES`（该对象库会经 `$<TARGET_OBJECTS>` 进入 master）。`storage_backend.cpp` 只允许在 `MOONCAKE_HAVE_KVCLIENT_SDK` 内包含厂商无关的 `kvclient_adapter_factory.h`，不得包含 `kvclient_object_storage_adapter.h`。检测到 SDK 时适配器 TU 才参与编译；SDK 缺失时工厂的 `#else` 分支返回 `NOT_SUPPORTED`，不引用任何 SDK 符号。

### 1.7 验收

| 项 | 预期 |
| --- | --- |
| SDK 缺失构建 | 工厂分支返回 `NOT_SUPPORTED`，适配器 TU 不参与编译，其余后端不受影响 |
| SDK 存在但未检测到且 `MOONCAKE_KVCLIENT_REQUIRED=ON` 或设置了 `SDK_ROOT` | configure 阶段 `FATAL_ERROR`，不会生成"未编译适配器"的构建 |
| 指定 `MOONCAKE_KVCLIENT_SDK_ROOT` 或 include/library | 适配器 TU 参与编译，`MOONCAKE_HAVE_KVCLIENT_SDK` 生效 |
| 配置非法（含 `MOONCAKE_KVCLIENT_*` 数值无法解析） | 工厂返回 `INVALID_PARAMS`，不启动 SDK |
| `timeoutMs < wait_timeout_ms` | `Init` 失败 `INVALID_PARAMS` |
| 读写方法 | `Put` / `PutV` / `PutBatch` / `Get` / `GetBatch` 返回 `NOT_SUPPORTED`（PR2 替换）；`Delete` / `DeleteBatch` 已实现 |

---

## PR2：适配器数据面 + Master

### 2.1 目标

实现直传/暂存两种模式的批读写、按批注册与释放、超时保留；并把 Master 侧的描述符铸造与门控接上，使 KVClient 后端可端到端读写。

#### 2.1.1 已确认决策（2026-11-18）

| 决策项 | 结论 | 对实施的影响 |
| --- | --- | --- |
| 直传判定 | 不新增 capacity 字段；直传条件为 N 已 512 对齐（`wire_len == N`）且地址对齐、iov 连续 | 接口层零改动；未对齐对象一律暂存；现有 `iov`/`iovcnt`、`buffer`/`size` 已足够表达安全性 |
| SDK 覆盖语义 | `StoreAsync` 对已存在 key 为覆盖写（upsert） | `PutBatch` 不做预删，不需要 `DeleteAsync` + `StoreAsync` 组合；重复写不会返回 `CQE_KEY_ALREADY_EXISTED` |
| kvclient + oplog | `InitDfsAllocatorFromEnvironment` 在 `enable_oplog_` 为真时 `LOG(FATAL)` 拒绝启动 | kvclient 不支持快照/oplog 恢复；恢复门保持 KVCS-only，不并入 `use_kvclient_` |
| PR2 测试归属 | PR2 自带适配器单测（原文档放在 PR3） | `tests/kvclient_object_storage_adapter_test.cpp` 与 `tests/CMakeLists.txt` 纳入 PR2 改动面 |

### 2.2 改动文件

| 文件 | 改动 |
| --- | --- |
| `include/storage/distributed/object_storage_adapter.h` / `include/storage/distributed/distributed_storage_backend.h` / `src/storage/distributed/distributed_storage_backend.cpp` | **不改**：直传判定只用现有 `iov`/`iovcnt`、`buffer`/`size` 与 `wire_len == N` |
| `include/storage/distributed/kvclient/kvclient_object_storage_adapter.h` | `BufferPlan`、`RegisterBuffers`/`ReleaseBuffers`/`RetainOrphans`、`orphans_` 成员（`Completion` 已在 PR1 声明） |
| `src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp` | 数据面实现；析构中清理 `orphans_` |
| `include/master_service.h` / `src/master_service.cpp` | `use_kvclient_`、门控、descriptor、白名单；`InitDfsAllocatorFromEnvironment` 新增 kvclient 早退分支（`enable_oplog_` 拒绝） |
| `tests/kvclient_object_storage_adapter_test.cpp` | 新增，适配器数据面单测（SDK 存在时编译） |
| `tests/CMakeLists.txt` | SDK 存在时新增测试目标并 `target_link_libraries(... Mooncake::kvclient)` |

> **直传判定**：`Slice` 只有 `{ptr, size}`，现有接口没有缓冲区容量信息，因此直传条件收紧为 `wire_len == N`（N 已 512 对齐）+ `addr % 512 == 0` + iov 连续；不满足即暂存。未对齐对象即使底层分配有富余也不直传，这是有意的安全取舍。`ObjectPutRequest` / `ObjectGetRequest` / `DfsWriteRequest` / `DfsReadRequest` **不新增字段**。
>
> **测试归属**：PR2 自带 §2.7 的适配器级用例；PR3 再补 Client 集成、删除顺序与 miss 自愈用例。

### 2.3 类图

```mermaid
classDiagram
    direction TB

    class BufferPlan {
        +uintptr_t addr
        +size_t n
        +size_t wire_len
        +bool staged
        +shared_ptr~char~ staging
        +MRHandle handle
    }

    class Completion {
        +bool terminal
        +vector~Status~ entryStatus
    }

    class KvClientObjectStorageAdapter {
        -KvClientAdapterConfig config_
        -unique_ptr~KvClient~ kv_client_
        -mutex orphan_mu_
        -vector~pair~ orphans_
        +PutBatch(requests) results
        +GetBatch(requests) results
        -RegisterBatchImpl(requests, plans, errors, is_store, extract)
        -RegisterBuffers(put_requests, plans, errors)
        -RegisterBuffers(get_requests, plans, errors)
        -ReleaseBuffers(plans)
        -RetainOrphans(plans)
        -waitForCompletion(task_id, count) Completion
    }

    class MasterService {
        -bool use_kvclient_
        +PrepareWriteStart(...)
        +GetReplicaList(...)
        +EvictDiskReplicaIfCurrent(...)
    }

    KvClientObjectStorageAdapter ..> BufferPlan : 产生
    KvClientObjectStorageAdapter ..> Completion : 返回
    MasterService ..> DistributedFSDescriptor : 铸造
```

### 2.4 流程图（写/读共用骨架）

```mermaid
flowchart TB
    A["PutBatch / GetBatch(requests)"] --> B["RegisterBuffers：逐条校验"]
    B --> C{"空 key / n==0 / wire_len>上限?"}
    C -->|是| E["errors[i]=INVALID_PARAMS<br/>plans[i]=空"]
    C -->|否| D{"wire_len==N（N 已 512 对齐）<br/>且 addr%512==0<br/>且 iov 连续?"}
    D -->|是| DIR["直传：无需补齐"]
    D -->|否| STG["暂存：分配 wire_len/512 对齐 buffer<br/>写侧 memcpy + 清零；读侧留空"]
    DIR --> REG["按 (addr,wire_len) 去重<br/>一次 RegisterRegions<br/>handle 回填 plan"]
    STG --> REG
    REG --> SUB["一次 StoreAsync / LoadAsync"]
    SUB --> SUBOK{"提交被受理?"}
    SUBOK -->|否| SUBFAIL["逐项 ToErrorCode(submit)<br/>ReleaseBuffers（未触达设备）"]
    SUBOK -->|是| WAIT["waitForCompletion(task_id, count)"]
    WAIT --> TERM{"terminal?"}
    TERM -->|否| KEEP["返回 RPC_TIMEOUT/INTERNAL_ERROR<br/>RetainOrphans：handle+staging 留到 Shutdown"]
    TERM -->|是| MAP["逐项映射 entryStatus<br/>读且 staged：拷回 N 字节<br/>返回 N（不是 wire_len）"]
    MAP --> REL["ReleaseBuffers：注销 handle<br/>staging 随 plan 释放"]
    KEEP --> END["返回逐项结果"]
    REL --> END
    SUBFAIL --> END
```

### 2.5 时序图（写路径，读路径对称）

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant BE as DistributedStorageBackend
    participant AD as KvClientObjectStorageAdapter
    participant SDK as kv::KvClient
    participant S as 远端存储

    C->>BE: BatchWrite(key, desc, slices)
    BE->>BE: 校验 total_size == desc.object_size<br/>每 slice 转 iovec
    BE->>AD: PutBatch(ObjectPutRequest{key, iov, iovcnt})
    AD->>AD: RegisterBuffers：校验 + 直传/暂存 + 去重注册
    alt 注册失败
        AD-->>BE: 逐项 DFS_SERVICE_UNAVAILABLE
    else 注册成功
        AD->>SDK: StoreAsync(entries, taskId)
        SDK-->>AD: 提交结果
        SDK->>S: worker 线程异步派发 / 子批写入
        alt 提交被拒
            AD->>AD: 逐项 ToErrorCode(submit)
            AD->>AD: ReleaseBuffers（未触达设备，handle 可注销）
            AD-->>BE: 逐项错误
        else 已受理
            AD->>SDK: Wait(taskId, timeout, result)
            alt 终态
                SDK-->>AD: entryStatus[]
                AD->>AD: 逐项映射（写侧无回拷）
                AD->>AD: ReleaseBuffers（去重注销，staging 随 plan 释放）
            else 非终态
                AD->>AD: 逐项 RPC_TIMEOUT/INTERNAL_ERROR<br/>RetainOrphans（handle+staging 留到 Shutdown）
            end
            AD-->>BE: 逐项 expected
        end
    end
    BE-->>C: results[index]
```

### 2.6 伪代码

#### 2.6.1 BufferPlan 与注册

```cpp
// 文件内 helper（kvclient_object_storage_adapter.cpp）
struct BufferView {
    iovec single{};          // 读路径的单段 view 落在自身
    const iovec* iov = nullptr;
    int iovcnt = 0;
    size_t n = 0;
};

inline size_t AlignUp(size_t n, size_t a) { return (n + a - 1) & ~(a - 1); }

size_t SumIov(const iovec* iov, int iovcnt);

bool IsContiguous(const iovec* iov, int iovcnt) {
    for (int j = 0; j + 1 < iovcnt; ++j) {
        const auto end = reinterpret_cast<uintptr_t>(iov[j].iov_base) +
                         iov[j].iov_len;
        if (end != reinterpret_cast<uintptr_t>(iov[j + 1].iov_base)) return false;
    }
    return true;
}

void CopyIovTo(char* dst, const iovec* iov, int iovcnt);

std::shared_ptr<char[]> AllocateStaging(size_t wire_len);  // 512B 对齐，失败返回 null

template <typename T>
void FillError(std::vector<tl::expected<T, ErrorCode>>& results, ErrorCode e) {
    for (auto& r : results) r = tl::make_unexpected(e);
}
```

```cpp
struct BufferPlan {
    uintptr_t addr = 0;
    size_t n = 0;                 // 逻辑长度
    size_t wire_len = 0;          // AlignUp(n, 512)
    bool staged = false;
    std::shared_ptr<char[]> staging;
    kv::MRHandle handle = kv::kInvalidMRHandle;
};

struct Completion {
    bool terminal = false;
    std::vector<kv::Status> entryStatus;
};
```

```cpp
template <typename RequestT, typename Extract>
tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::RegisterBatchImpl(
    std::span<const RequestT> requests,
    std::vector<std::optional<BufferPlan>>& plans,
    std::span<ErrorCode> errors, bool is_store, Extract&& extract) {
    plans.assign(requests.size(), std::nullopt);

    auto make_plan = [&](const BufferView& v,
                         ErrorCode& error) -> std::optional<BufferPlan> {
        if (v.iov == nullptr || v.iovcnt <= 0 || v.n == 0 ||
            v.iov[0].iov_base == nullptr) {
            error = ErrorCode::INVALID_PARAMS;
            return std::nullopt;
        }
        const size_t wire_len = AlignUp(v.n, kv::kAlignmentBytes);
        if (wire_len > config_.max_object_bytes) {
            error = ErrorCode::INVALID_PARAMS;
            return std::nullopt;
        }

        BufferPlan plan;
        plan.n = v.n;
        plan.wire_len = wire_len;
        plan.addr = reinterpret_cast<uintptr_t>(v.iov[0].iov_base);

        // N 已 512 对齐时 wire_len == N，直传不会越过调用方声明的范围。
        const bool direct = wire_len == v.n &&
                            plan.addr % kv::kAlignmentBytes == 0 &&
                            IsContiguous(v.iov, v.iovcnt);
        if (!direct) {
            plan.staging = AllocateStaging(wire_len);   // 512B 对齐，失败返回 null
            if (!plan.staging) {
                error = ErrorCode::NO_AVAILABLE_HANDLE;
                return std::nullopt;
            }
            plan.addr = reinterpret_cast<uintptr_t>(plan.staging.get());
            plan.staged = true;
            if (is_store) CopyIovTo(plan.staging.get(), v.iov, v.iovcnt);
        }
        if (is_store && wire_len > v.n) {
            // 只有暂存路径会出现补齐；直传已保证 wire_len == n。
            std::memset(reinterpret_cast<char*>(plan.addr) + v.n, 0,
                        wire_len - v.n);
        }
        return plan;
    };

    std::set<kv::BinaryKey> seen;   // 仅写侧
    std::vector<kv::MemoryRegion> regions;
    std::vector<size_t> plan_to_region(requests.size(), SIZE_MAX);

    for (size_t i = 0; i < requests.size(); ++i) {
        if (requests[i].logical_key.empty()) {
            errors[i] = ErrorCode::INVALID_PARAMS;
            continue;
        }
        if (is_store && !seen.insert(MakeCacheKey(requests[i].logical_key)).second) {
            errors[i] = ErrorCode::INVALID_PARAMS;   // 同批重复 key
            continue;
        }
        BufferView view;
        extract(requests[i], view);
        auto plan = make_plan(view, errors[i]);
        if (!plan) continue;

        size_t region = regions.size();
        for (size_t j = 0; j < regions.size(); ++j) {
            if (regions[j].addr == plan->addr &&
                regions[j].size == plan->wire_len) {
                region = j;
                break;
            }
        }
        if (region == regions.size()) {
            regions.push_back({kv::MemoryType::HOST, plan->addr,
                               plan->wire_len, -1, -1});
        }
        plans[i] = std::move(*plan);
        plan_to_region[i] = region;
    }

    if (regions.empty()) return {};

    std::vector<kv::RegisteredMemory> out;
    if (!kv_client_->RegisterRegions(regions, out).ok() ||
        out.size() != regions.size()) {
        return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);
    }
    for (size_t i = 0; i < plans.size(); ++i) {
        if (plans[i]) plans[i]->handle = out[plan_to_region[i]].handle;
    }
    return {};
}
```

#### 2.6.2 批提交

```cpp
std::vector<tl::expected<void, ErrorCode>> KvClientObjectStorageAdapter::PutBatch(
    const std::vector<ObjectPutRequest>& requests) {
    std::vector<tl::expected<void, ErrorCode>> results(requests.size());
    if (!initialized_ || !kv_client_) {
        FillError(results, ErrorCode::DFS_SERVICE_UNAVAILABLE);
        return results;
    }

    std::vector<std::optional<BufferPlan>> plans;
    std::vector<ErrorCode> errors(requests.size(), ErrorCode::INVALID_PARAMS);

    if (!RegisterBuffers(requests, plans, errors)) {
        FillError(results, ErrorCode::DFS_SERVICE_UNAVAILABLE);
        return results;
    }
    for (size_t i = 0; i < plans.size(); ++i)
        if (!plans[i]) results[i] = tl::make_unexpected(errors[i]);

    std::vector<kv::KVBuffer> entries;
    std::vector<size_t> indices;
    for (size_t i = 0; i < plans.size(); ++i) {
        if (!plans[i]) continue;
        kv::KVBuffer e;
        e.key = kv::CacheKey::FromBytes(MakeCacheKey(requests[i].logical_key));
        e.buffer.region = {kv::MemoryType::HOST, plans[i]->addr,
                           plans[i]->wire_len, -1, -1};
        e.buffer.handle = plans[i]->handle;
        e.offset = 0;
        e.mrKey = std::nullopt;
        entries.push_back(std::move(e));
        indices.push_back(i);
    }
    if (entries.empty()) return results;

    kv::TaskId task_id = kv::kInvalidTaskId;
    const auto submit = kv_client_->StoreAsync(entries, task_id);
    if (!submit.ok()) {
        // 任务未入队；只覆盖参与项，保留空 key / 重复 key 的 INVALID_PARAMS。
        for (size_t i : indices)
            results[i] = tl::make_unexpected(ToErrorCode(submit));
        ReleaseBuffers(plans);
        return results;
    }

    const auto completion = waitForCompletion(task_id, entries.size());
    if (completion.entryStatus.size() != entries.size()) {
        for (size_t i : indices)
            results[i] = tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
    } else if (!completion.terminal) {
        // 未终态：逐项结果不可信，统一超时；绝不返回 OBJECT_NOT_FOUND。
        for (size_t i : indices)
            results[i] = tl::make_unexpected(ErrorCode::RPC_TIMEOUT);
    } else {
        for (size_t j = 0; j < entries.size(); ++j) {
            const auto& st = completion.entryStatus[j];
            results[indices[j]] = st.ok()
                                      ? tl::expected<void, ErrorCode>{}
                                      : tl::make_unexpected(ToErrorCode(st));
        }
    }

    if (completion.terminal) ReleaseBuffers(plans);
    else {
        RetainOrphans(plans);
        LOG(ERROR) << "action=kvclient_batch_release_skipped, task_id=" << task_id;
    }
    return results;
}
```

> **实现约束（文档旧伪代码缺失，必须补上）**：
>
> 1. 入口先检查 `!initialized_ || !kv_client_`，逐项返回 `DFS_SERVICE_UNAVAILABLE`，与 PR1 的 `DeleteBatch` 一致。
> 2. `StoreAsync` 已确认是覆盖写，因此不预删；不要为 upsert 引入 `DeleteAsync` + `StoreAsync` 组合。
> 3. 非终态（`terminal == false`）时**所有条目统一返回 `RPC_TIMEOUT`**，不得逐项调用 `ToErrorCode(entryStatus)`——未完成的 entry 可能是默认 `OK` 或中间的 `OBJECT_NOT_FOUND`，后者会让 PR3 误删元数据。注意旧伪代码 `!st.ok() → ToErrorCode(st)` 的写法必须改掉。
> 4. 注册失败时如果 SDK 已回滚部分注册，直接整批 `DFS_SERVICE_UNAVAILABLE`；`PutBatch` 不要再次 `UnregisterRegions`。

#### 2.6.3 单条入口（batch 包装）

```cpp
// 适配器只有 batch 数据路径；Delete 已在 PR1 落地，
// PR2 只需为以下三个入口补 batch 包装。
tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::Put(
    const std::string& key, std::span<const char> data) {
    iovec iov{const_cast<char*>(data.data()), data.size()};
    auto results = PutBatch({ObjectPutRequest{key, &iov, 1}});
    return results.empty() ? tl::make_unexpected(ErrorCode::INVALID_PARAMS)
                           : results.front();
}

tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::PutV(
    const std::string& key, const iovec* iov, int iovcnt) {
    auto results = PutBatch({ObjectPutRequest{key, iov, iovcnt}});
    return results.empty() ? tl::make_unexpected(ErrorCode::INVALID_PARAMS)
                           : results.front();
}

tl::expected<size_t, ErrorCode> KvClientObjectStorageAdapter::Get(
    const std::string& key, void* buf, size_t len) {
    auto results = GetBatch({ObjectGetRequest{key, buf, len}});
    return results.empty() ? tl::make_unexpected(ErrorCode::INVALID_PARAMS)
                           : results.front();
}
```

#### 2.6.4 Master 门控

```cpp
// 初始化落点：MasterService::InitDfsAllocatorFromEnvironment(const MasterServiceConfig&)
// （实际 master_service.cpp:855-951；文档旧行号 :906-911 已漂移）
const auto dfs_config = DistributedStorageConfig::FromEnvironment();
enable_dfs_ = DfsEnablementConfig::FromEnvironment().enabled ||
              dfs_config.UsesKvcs() || dfs_config.UsesKvClient();
if (!enable_dfs_) return;

// ... Validate() / single_tenant 检查不变 ...

// 必须在 filesystem allocator 初始化（:930）之前新增 kvclient 早退分支，
// 否则会落入 oplog 检查（throw）或 ShardAllocator::Init（必然失败）。
if (dfs_config.UsesKvClient()) {
    if (enable_oplog_) {
        // kvclient 不支持元数据恢复；启动期拒绝，避免运行期 restore 静默失败。
        LOG(FATAL) << "kvclient backend does not support snapshot/oplog recovery";
    }
    use_kvclient_ = true;
    // 不创建 EFC target、capacity provider 与水位淘汰；直接返回。
    LOG(INFO) << "KVClient distributed object storage initialized, config={"
              << dfs_config.FormatStr() << "}";
    return;
}

// DFS 可用性门（实际 master_service.cpp:5154-5156）：保留 !enable_dfs_ 前缀
if (config.dfs_replica_num > 0 &&
    (!enable_dfs_ ||
     (!use_kvcs_ && !use_kvclient_ &&
      (!dfs_allocator_ || !dfs_allocator_->IsInitialized())))) {
    // 现有实现返回 DFS_SERVICE_UNAVAILABLE（:5160），不要改成 INVALID_PARAMS。
    return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);
}

// PrepareWriteStart：长度上限 + NOF 拒绝（覆盖 PutStart / UpsertStart / BatchPutStart）
if (config.dfs_replica_num > 0 && use_kvclient_) {
    // 上限本身已 512 对齐，N <= 上限等价于 AlignUp(N,512) <= 上限。
    if (slice_length > kKvClientMaxValueBytes) return INVALID_PARAMS;
    if (config.nof_replica_num > 0) return INVALID_PARAMS;   // NOF 与 kvclient 不共存
}

// descriptor 铸造：kvclient 单独子分支，KVCS 的 aligned_size 保持 value_length
if (config.dfs_replica_num > 0) {
    if (use_kvcs_) {
        // ... 原 KVCS 分支不变（aligned_size = value_length）...
    } else if (use_kvclient_) {
        auto aligned = CheckedAlignUp(value_length, kKvClientAlignment);
        if (!aligned) return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
        DistributedFSDescriptor d;
        d.object_size = value_length;
        d.aligned_size = *aligned;
        d.shard_idx = -1;
        replicas.emplace_back(std::move(d), ReplicaStatus::PROCESSING);
    } else if (!dfs_allocator_ || !dfs_allocator_->IsInitialized()) {
        return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);
    }
}

// 白名单 1：replica_type 校验
const bool supports_dfs_condition =
    replica_type == ReplicaType::DFS &&
    expected_replica_id != nullptr &&
    (use_kvcs_ || use_kvclient_);

// 白名单 2：target_pred 里同样要放宽（实际 :6345-6360）
auto target_pred = [this, replica_type, &client_id, expected_replica_id](
                       const Replica& r) {
    if (replica_type == ReplicaType::DFS) {
        return (use_kvcs_ || use_kvclient_) &&
               expected_replica_id != nullptr &&
               r.is_dfs_replica() && r.id() == *expected_replica_id;
    }
    // DISK / LOCAL_DISK 分支保持原样
    return false;
};
```

> **Master 侧实现约束**：
>
> - `aligned_size` 必须 512 对齐；Master TU 没有现成 `AlignUp`，可用 `bucket_entry_layout.h` 的 `CheckedAlignUp`（返回 `optional`）。不能把 `if (use_kvcs_)` 直接改成 `if (use_kvcs_ || use_kvclient_)`——KVCS 的 `aligned_size = value_length` 要保持不变。
> - 恢复门（`master_service.cpp:3434 / 3726 / 4014`）保持 KVCS-only，**不要**并入 `use_kvclient_`；kvclient 的恢复能力由启动期 `LOG(FATAL)` 拒绝 `enable_oplog_` 兜底。
> - 16MB 上限用 `kKvClientMaxValueBytes`（`kvclient_config.h:18`），不要写字面量。`nof_replica_num > 0` 的拒绝在默认构建已有 `#ifndef USE_NOF` 分支（`:5170-5177`），kvclient 需要**无条件**拒绝，避免 `-DUSE_NOF=ON` 构建下漏拦。
> - `PrepareWriteStart` 同时覆盖 `PutStart`、`UpsertStart` 与 `BatchPutStart`；Upsert 的 descriptor 门必须一起放宽。
> - `EvictDiskReplica` / `BatchEvictDiskReplica` 以 `expected_replica_id = nullptr` 调用共享实现，DFS 永远被拒；PR2 不改变该行为，只有 `EvictDiskReplicaIfCurrent` 可用。

### 2.7 验收

| 场景 | 预期 | 层级 |
| --- | --- | --- |
| N=1024 直传（512 对齐） | 不暂存；提交成功；`Get` 返回 1024 | 适配器单测 |
| N=513 暂存（未对齐） | 写侧补齐到 1024；调用方 buffer 不被写；读回 513 字节正确 | 适配器单测 |
| N=513 但底层分配有富余 | 仍走暂存（不读取未声明的容量），结果正确 | 适配器单测 |
| 多段 iov / 未对齐 | 自动走暂存，不报 `INVALID_PARAMS` | 适配器单测 |
| 同批重复 key | 第一条受理，后续条目 `INVALID_PARAMS` | 适配器单测 |
| 注册失败 | 整批 `DFS_SERVICE_UNAVAILABLE`，不提交 SDK | 适配器单测 |
| `Wait` 超时 | 全批 `RPC_TIMEOUT`（**不得出现 `OBJECT_NOT_FOUND`**）；handle+staging 保留且不注销 | 适配器单测 |
| 已存在 key 覆盖写 | `StoreAsync` 直接覆盖；调用序列中不出现 `DeleteAsync` | 适配器单测（fake client） |
| 空 key / N=0 | 逐项 `INVALID_PARAMS`，不进入注册与提交 | 适配器单测 |
| Init 前调用 `PutBatch` / `GetBatch` | 逐项 `DFS_SERVICE_UNAVAILABLE`，不空指针 | 适配器单测 |
| 读 miss | 适配器返回 `OBJECT_NOT_FOUND`；触发 miss 清理属于 PR3 | 单测 + PR3 集成 |
| kvclient + `enable_oplog` | Master 启动期 `LOG(FATAL)`，拒绝启动 | 启动验收 |
| Master descriptor | `object_size=N`；`aligned_size=AlignUp(N,512)`；`shard_idx=-1` | 集成 |
| 恢复门 | 保持 KVCS-only；kvclient 不做元数据恢复 | 代码审查 |

---

## PR3：Client 集成 + 测试

### 3.1 目标

把删除顺序改为 value-first、放宽 miss 清理与 Remove 门控、改名 `ReconcileMissingKvReplica`，并补齐单元/集成测试。

### 3.2 改动文件

| 文件 | 改动 |
| --- | --- |
| `include/client_service.h` | `ReconcileMissingKvcsReplica` → `ReconcileMissingKvReplica` |
| `src/client_service.cpp` | 改名（声明 + 2 处调用点 `:1626`、`:1936`）；`BatchRemove` 的 KVClient 分支；`Remove`/`RemoveAll`/`RemoveByRegex` 门控；`AttachHostId` |
| `tests/kvclient_object_storage_adapter_test.cpp` | 新增（含 `MakeCacheKey` 稳定性/小端断言） |
| `tests/CMakeLists.txt` | 新增测试目标（仅 SDK 存在时），并 `target_link_libraries(... Mooncake::kvclient)` |

### 3.3 类图

```mermaid
classDiagram
    direction TB

    class Client {
        -shared_ptr~DistributedStorageBackend~ dfs_storage_backend_
        +BatchRemove(keys, force) results
        +Remove(key) expected
        +RemoveAll() expected
        +RemoveByRegex(regex) expected
        -ReconcileMissingKvReplica(key, query, replica)
    }

    class DistributedStorageBackend {
        +UsesKvcs() bool
        +UsesKvClient() bool
        +BatchDeleteObjects(keys) results
    }

    class MasterClient {
        +BatchRemove(keys, force) results
        +EvictDiskReplicaIfCurrent(key, tenant, type, id) expected
    }

    Client o-- DistributedStorageBackend
    Client ..> MasterClient : RPC
```

### 3.4 流程图（BatchRemove）

```mermaid
flowchart TB
    A["Client::BatchRemove(keys, force)"] --> B{"UsesKvClient()?"}
    B -->|否| OLD["沿用现有路径：<br/>先删元数据，KVCS 再删 value"]
    B -->|是| C["value-first：BatchDeleteObjects(keys)"]
    C --> D{"逐项结果"}
    D -->|成功或 OBJECT_NOT_FOUND| E["加入 metadata_keys"]
    D -->|其他错误| F["results[i]=错误<br/>value 与元数据都保留"]
    E --> G["MasterClient::BatchRemove(metadata_keys, force)"]
    G --> H{"逐项元数据删除"}
    H -->|成功| I["results[i]=成功<br/>清 hot cache"]
    H -->|失败（lease 等）| J["results[i]=错误<br/>value 已删、元数据保留，靠 miss 自愈"]
    I --> K["回填原下标，返回"]
    J --> K
    F --> K
```

### 3.5 时序图（value-first 删除）

```mermaid
sequenceDiagram
    autonumber
    participant APP as 应用
    participant C as Client
    participant BE as DistributedStorageBackend
    participant AD as KvClientObjectStorageAdapter
    participant SDK as kv::KvClient
    participant MC as MasterClient
    participant M as MasterService

    APP->>C: BatchRemove(keys, force)
    C->>BE: BatchDeleteObjects(全部 keys)
    BE->>AD: DeleteBatch(keys)
    AD->>SDK: DeleteAsync + Wait
    SDK-->>AD: entryStatus[]
    AD-->>BE: OK / OBJECT_NOT_FOUND / 错误
    BE-->>C: 逐项结果
    C->>C: 成功项 → metadata_keys
    C->>MC: BatchRemove(metadata_keys, force)
    MC->>M: RPC
    M->>M: CheckRemovable（lease/就绪/复制任务）
    alt 允许
        M-->>MC: OK
        MC-->>C: OK
        C->>C: 清 hot cache
    else 拒绝
        M-->>MC: OBJECT_HAS_LEASE 等
        MC-->>C: 错误
        C->>C: 保留元数据，等 miss 自愈
    end
    C-->>APP: 逐 key 结果
```

### 3.6 伪代码

#### 3.6.1 BatchRemove

```cpp
std::vector<tl::expected<void, ErrorCode>> Client::BatchRemove(
    const std::vector<std::string>& keys, bool force) {
    std::vector<tl::expected<void, ErrorCode>> results(keys.size());
    if (hot_cache_) hot_cache_->BumpKeyGenerations(keys);
    const bool kvclient =
        dfs_storage_backend_ && dfs_storage_backend_->UsesKvClient();

    std::vector<std::string> metadata_keys;
    std::vector<size_t> metadata_indices;

    if (kvclient) {
        const auto deleted = dfs_storage_backend_->BatchDeleteObjects(keys);
        if (deleted.size() != keys.size()) {
            for (auto& r : results)
                r = tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
            return results;
        }
        for (size_t i = 0; i < keys.size(); ++i) {
            if (deleted[i] ||
                deleted[i].error() == ErrorCode::OBJECT_NOT_FOUND) {
                metadata_keys.push_back(keys[i]);
                metadata_indices.push_back(i);
            } else {
                results[i] = tl::make_unexpected(deleted[i].error());
            }
        }
    } else {
        metadata_keys = keys;
        metadata_indices.resize(keys.size());
        std::iota(metadata_indices.begin(), metadata_indices.end(), 0);
    }

    if (!metadata_keys.empty()) {
        const auto master = master_client_.BatchRemove(metadata_keys, force);
        for (size_t j = 0; j < metadata_indices.size(); ++j) {
            results[metadata_indices[j]] =
                j < master.size() ? master[j]
                                  : tl::make_unexpected(ErrorCode::RPC_FAIL);
        }
    }

    if (!kvclient && dfs_storage_backend_ &&
        dfs_storage_backend_->UsesKvcs()) {
        std::vector<std::string> removed;
        std::vector<size_t> indices;
        for (size_t i = 0; i < keys.size(); ++i) {
            if (results[i]) { removed.push_back(keys[i]); indices.push_back(i); }
        }
        if (!removed.empty()) {
            const auto vals = dfs_storage_backend_->BatchDeleteObjects(removed);
            for (size_t j = 0; j < indices.size(); ++j) {
                if (j >= vals.size())
                    results[indices[j]] = tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
                else if (!vals[j])
                    results[indices[j]] = vals[j];
            }
        }
    }

    if (hot_cache_) {
        std::vector<std::string> hot;
        for (size_t j = 0; j < metadata_indices.size(); ++j)
            if (results[metadata_indices[j]]) hot.push_back(metadata_keys[j]);
        if (!hot.empty()) hot_cache_->RemoveHotKeys(hot);
    }
    return results;
}
```

#### 3.6.2 miss 清理门控

```cpp
void Client::ReconcileMissingKvReplica(
    const std::string& key, const QueryResult& query,
    const Replica::Descriptor& replica) {
    if (!dfs_storage_backend_ ||
        !(dfs_storage_backend_->UsesKvcs() ||
          dfs_storage_backend_->UsesKvClient()) ||
        !replica.is_dfs_replica() ||
        !query.resolved_tenant_id.has_value()) {
        return;
    }
    auto r = master_client_.EvictDiskReplicaIfCurrent(
        key, *query.resolved_tenant_id, ReplicaType::DFS, replica.id);
    if (!r) {
        LOG(WARNING) << "conditional eviction failed, key=" << key
                     << ", error=" << toString(r.error());
    }
}
```

#### 3.6.3 门控放宽

```cpp
// Remove：单 key
if (!dfs_storage_backend_ ||
    !(dfs_storage_backend_->UsesKvcs() ||
      dfs_storage_backend_->UsesKvClient())) {
    // ...原有路径...
}

// RemoveAll / RemoveByRegex
if (dfs_storage_backend_ &&
    (dfs_storage_backend_->UsesKvcs() ||
     dfs_storage_backend_->UsesKvClient())) {
    return tl::make_unexpected(ErrorCode::NOT_SUPPORTED);   // 不删元数据
}

// AttachHostId：const 入参，改本地副本
ReplicateConfig Client::AttachHostId(const ReplicateConfig& config) const {
    ReplicateConfig client_cfg = config;
    if (dfs_storage_backend_ &&
        (dfs_storage_backend_->UsesKvcs() ||
         dfs_storage_backend_->UsesKvClient())) {
        client_cfg.dfs_replica_num = 1;
    }
    if (!host_id_.empty()) client_cfg.host_id = host_id_;
    return client_cfg;
}
```

#### 3.6.4 测试清单

```cpp
TEST(KvClientAdapter, MakeCacheKeyIsStableAndLittleEndian) { /* T-01 */ }
TEST(KvClientAdapter, RejectEmptyKeyZeroLengthAndDuplicate) { /* T-02 */ }
TEST(KvClientAdapter, DirectAndStagedPaths) { /* T-03 */ }
TEST(KvClientAdapter, RegisterFailureFailsBatch) { /* T-20 */ }
TEST(KvClientAdapter, TimeoutKeepsHandlesAndStaging) { /* T-08 */ }
TEST(KvClientAdapter, BufferNotRegisteredFailsSubBatch) { /* T-07 */ }
TEST(KvClientClient, ValueFirstDeleteOrder) { /* T-17 */ }
TEST(KvClientClient, RemoveAllAndRegexAreNotSupported) { /* T-18 */ }
```

### 3.7 验收

| 场景 | 预期 |
| --- | --- |
| value 删除失败 | 该 key 不删元数据，两者保留可重试 |
| value 删除成功、元数据被 lease 拒绝 | value 已删、元数据保留；后续读 miss 自愈 |
| `RemoveAll` / `RemoveByRegex` | `NOT_SUPPORTED`，元数据完全未变 |
| miss 清理 | 仅 `use_kvcs_ \|\| use_kvclient_` 且 DFS 副本且 tenant 已解析时触发；ReplicaID 不匹配保留 R2 |

---

## 合并顺序与回滚

1. PR1 合并后：`kvclient` 配置可启动，读写返回 `NOT_SUPPORTED`，所有既有后端回归通过。
2. PR2 合并后：KVClient 后端端到端可用；Master 门控生效。
3. PR3 合并后：删除语义与 miss 自愈生效，测试补齐。

任一步回滚不需要数据迁移：配置项未启用时新代码路径不可达；已写入的 L3 对象不依赖 PR3 的 Client 逻辑。
