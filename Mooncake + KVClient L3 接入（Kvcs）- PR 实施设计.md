# Mooncake + KVClient L3 接入（Kvcs）— 三个 PR 的实施设计

版本：v1.2　更新日期：2026-11-16
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
| `include/storage/distributed/kvclient/kvclient_backend.h` | 新增，厂商无关工厂声明 |
| `include/storage/distributed/kvclient/kvclient_object_storage_adapter.h` | 新增，类定义（`kKvClientAdapterName`、注入构造、全部纯虚 override；数据方法占位；含 `Completion` 与 `waitForCompletion`，删除路径依赖它） |
| `src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp` | 新增，控制面实现（`Init`、`Delete`、`DeleteBatch`、私有 `MakeCacheKey` 编码）+ 数据方法 NOT_SUPPORTED 占位 + 工厂实现 |
| `include/config/distributed_storage_config.h`、`src/config/distributed_storage_config.cpp` | `UsesKvClient()`、`Validate()` 分支、`UsesObjectStorage()` 列表 |
| `src/config/file_storage_config.cpp` | `Validate()` 跳过本地目录校验 |
| `include/storage/distributed/distributed_storage_backend.h` | 新增 `UsesKvClient()` |
| `src/storage_backend.cpp` | 工厂新增 `UsesKvClient()` 分支 |
| `src/file_storage.cpp` | `enable_dfs` 加入 `UsesKvClient()` |
| `src/real_client.cpp` | 新增 `enable_kvclient`；构造条件与 `kDistributed` 强制均加入 |
| `src/CMakeLists.txt` | pkg-config、条件源、define/link（具体落点见 1.6.6） |
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
// kvclient_backend.h（厂商无关）
tl::expected<std::unique_ptr<ObjectStorageAdapter>, ErrorCode>
CreateKvClientObjectStorageAdapter();

// storage_backend.cpp（工厂分支；SDK 缺失时不引用任何 SDK 符号）
// 只允许包含厂商无关的 kvclient_backend.h；不得包含适配器头或 key codec。
#if defined(MOONCAKE_HAVE_KVCLIENT_SDK)
#include "storage/distributed/kvclient/kvclient_backend.h"
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
find_package(PkgConfig QUIET)
if(PkgConfig_FOUND)
  # GLOBAL：tests/ 与 src/ 是兄弟目录，非 GLOBAL 的 imported target 不可见
  pkg_check_modules(KVCLIENT QUIET IMPORTED_TARGET GLOBAL "kvclient>=0.1.0")
endif()
if(TARGET PkgConfig::KVCLIENT)
  set(MOONCAKE_KVCLIENT_SDK_FOUND ON)
  message(STATUS "KVClient SDK found: kvclient backend enabled")
else()
  message(STATUS "KVClient SDK not found: kvclient backend disabled")
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
                        PRIVATE PkgConfig::KVCLIENT)
endif()

# ④ 最终库：OBJECT 库的 PRIVATE 链接不传播，mooncake_store 需要再链一次
if(MOONCAKE_KVCLIENT_SDK_FOUND)
  target_link_libraries(mooncake_store PRIVATE PkgConfig::KVCLIENT)
endif()
```

> `mooncake_store_master` 不链接 KVClient SDK。适配器源不得放进 `MOONCAKE_STORE_SHARED_SOURCES`（该对象库会经 `$<TARGET_OBJECTS>` 进入 master）。`storage_backend.cpp` 只允许在 `MOONCAKE_HAVE_KVCLIENT_SDK` 内包含厂商无关的 `kvclient_backend.h`，不得包含 `kvclient_object_storage_adapter.h`。SDK 存在时适配器 TU 才参与编译；SDK 缺失时工厂的 `#else` 分支返回 `NOT_SUPPORTED`，不引用任何 SDK 符号。

### 1.7 验收

| 项 | 预期 |
| --- | --- |
| SDK 缺失构建 | 工厂分支返回 `NOT_SUPPORTED`，适配器 TU 不参与编译，其余后端不受影响 |
| 配置非法（含 `MOONCAKE_KVCLIENT_*` 数值无法解析） | 工厂返回 `INVALID_PARAMS`，不启动 SDK |
| `timeoutMs < wait_timeout_ms` | `Init` 失败 `INVALID_PARAMS` |
| 读写方法 | `Put` / `PutV` / `PutBatch` / `Get` / `GetBatch` 返回 `NOT_SUPPORTED`（PR2 替换）；`Delete` / `DeleteBatch` 已实现 |

---

## PR2：适配器数据面 + Master

### 2.1 目标

实现直传/暂存两种模式的批读写、按批注册与释放、超时保留；并把 Master 侧的描述符铸造与门控接上，使 KVClient 后端可端到端读写。

### 2.2 改动文件

| 文件 | 改动 |
| --- | --- |
| `include/storage/distributed/object_storage_adapter.h` | `ObjectPutRequest` / `ObjectGetRequest` 末尾追加 `size_t capacity = 0` |
| `include/storage/distributed/distributed_storage_backend.h` | `DfsWriteRequest` / `DfsReadRequest` 末尾追加 `size_t capacity = 0` |
| `src/storage/distributed/distributed_storage_backend.cpp` | 透传 capacity |
| `include/storage/distributed/kvclient/kvclient_object_storage_adapter.h` | `BufferPlan`、`RegisterBuffers`/`ReleaseBuffers`/`RetainOrphans`、`orphans_` 成员（`Completion` 已在 PR1 声明） |
| `src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp` | 数据面实现；析构中清理 `orphans_` |
| `include/master_service.h` / `src/master_service.cpp` | `use_kvclient_`、门控、descriptor、白名单 |

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
        +uint64 handle
    }

    class Completion {
        +bool terminal
        +vector~Status~ entryStatus
    }

    class KvClientObjectStorageAdapter {
        -KvClientAdapterConfig config_
        -unique_ptr~KvClient~ kv_client_
        -mutex orphan_mu_
        -vector~Orphan~ orphans_
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
    C -->|否| D{"capacity>=wire_len<br/>且 addr%512==0<br/>且 iov 连续?"}
    D -->|是| DIR["直传：写侧原地清零尾部"]
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
    BE->>AD: PutBatch(ObjectPutRequest{key, iov, iovcnt, capacity})
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
    size_t capacity = 0;
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

        const bool direct = v.capacity >= wire_len &&
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
    bool terminal = !submit.ok();

    if (submit.ok()) {
        const auto completion = waitForCompletion(task_id, entries.size());
        terminal = completion.terminal;
        if (completion.entryStatus.size() != entries.size()) {
            FillError(results, ErrorCode::INTERNAL_ERROR);
        } else {
            for (size_t j = 0; j < entries.size(); ++j) {
                const size_t i = indices[j];
                const auto& st = completion.entryStatus[j];
                if (!st.ok()) results[i] = tl::make_unexpected(ToErrorCode(st));
                else if (!terminal) results[i] = tl::make_unexpected(ErrorCode::RPC_TIMEOUT);
                else results[i] = {};
            }
        }
    } else {
        FillError(results, ToErrorCode(submit));
    }

    if (terminal) ReleaseBuffers(plans);
    else {
        RetainOrphans(plans);
        LOG(ERROR) << "action=kvclient_batch_release_skipped, task_id=" << task_id;
    }
    return results;
}
```

#### 2.6.3 单条入口（batch 包装）

```cpp
// 适配器只有 batch 数据路径；Delete 已在 PR1 落地，
// PR2 只需为以下三个入口补 batch 包装。
tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::Put(
    const std::string& key, std::span<const char> data) {
    iovec iov{const_cast<char*>(data.data()), data.size()};
    auto results = PutBatch({ObjectPutRequest{key, &iov, 1, 0}});
    return results.empty() ? tl::make_unexpected(ErrorCode::INVALID_PARAMS)
                           : results.front();
}

tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::PutV(
    const std::string& key, const iovec* iov, int iovcnt) {
    auto results = PutBatch({ObjectPutRequest{key, iov, iovcnt, 0}});
    return results.empty() ? tl::make_unexpected(ErrorCode::INVALID_PARAMS)
                           : results.front();
}

tl::expected<size_t, ErrorCode> KvClientObjectStorageAdapter::Get(
    const std::string& key, void* buf, size_t len) {
    auto results = GetBatch({ObjectGetRequest{key, buf, len, 0}});
    return results.empty() ? tl::make_unexpected(ErrorCode::INVALID_PARAMS)
                           : results.front();
}
```

#### 2.6.4 Master 门控

```cpp
// 初始化（MasterService 构造 / 配置注入）
use_kvcs_ = (distributed_config.fs_adapter_type == "kvcs-lowlevel");
use_kvclient_ = (distributed_config.fs_adapter_type == "kvclient");
enable_dfs_ = DfsEnablementConfig::FromEnvironment().enabled ||
              distributed_config.UsesKvcs() ||
              distributed_config.UsesKvClient();

// DFS 可用性门（master_service.cpp:5273-5275）：保留 !enable_dfs_ 前缀
if (config.dfs_replica_num > 0 &&
    (!enable_dfs_ ||
     (!use_kvcs_ && !use_kvclient_ &&
      (!dfs_allocator_ || !dfs_allocator_->IsInitialized())))) {
    return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
}

// PrepareWriteStart：长度上限 + 三副本拒绝
if (config.dfs_replica_num > 0 && use_kvclient_) {
    if (slice_length > 16'776'704) return INVALID_PARAMS;
    if (config.nof_replica_num > 0) return INVALID_PARAMS;   // 三副本不支持
}

// descriptor 铸造
if (use_kvclient_) {
    DistributedFSDescriptor d;
    d.object_size = value_length;
    d.aligned_size = AlignUp(value_length, 512);
    d.shard_idx = -1;
    replicas.emplace_back(std::move(d), ReplicaStatus::PROCESSING);
}

// 白名单 1：replica_type 校验
const bool supports_dfs_condition =
    replica_type == ReplicaType::DFS &&
    expected_replica_id != nullptr &&
    (use_kvcs_ || use_kvclient_);

// 白名单 2：target_pred 里同样要放宽（:6481）
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

### 2.7 验收

| 场景 | 预期 |
| --- | --- |
| N=513 直传 | 尾部补零；提交成功；`Get` 返回 513 |
| N=513 暂存（capacity=0） | 调用方 buffer 不被写；读回 513 字节正确 |
| 多段 iov / 未对齐 | 自动走暂存，不报 `INVALID_PARAMS` |
| 同批重复 key | 只失败重复条目 |
| 注册失败 | 整批 `DFS_SERVICE_UNAVAILABLE`，不提交 SDK |
| `Wait` 超时 | 逐项 `RPC_TIMEOUT`；handle+staging 保留；不误报成功 |
| 读 miss | `OBJECT_NOT_FOUND` 触发 miss 清理；超时不触发 |

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
| `tests/CMakeLists.txt` | 新增测试目标（仅 SDK 存在时），并 `target_link_libraries(... PkgConfig::KVCLIENT)` |

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
