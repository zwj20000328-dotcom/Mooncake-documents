# Mooncake + KVClient L3 接入（Kvcs）— PR1 实施步骤

版本：v1.0　日期：2026-11-16

对应设计文档：`Mooncake + KVClient L3 接入（Kvcs）- PR 实施设计.md`（PR1 章节）

代码基线：`Mooncake` main `7a34af83`（设计文档基线 `46c5eca8` 已漂移 42 个提交）

---

## 0. 前提与决策

### 0.1 PR1 边界

PR1 只让 `fs_adapter_type=kvclient` 在运行时可达：

- 环境变量能解析；
- 工厂能创建适配器；
- `FileStorage` / `RealClient` 能把后端注入为 DFS；
- SDK 缺失时明确失败；
- 适配器只落地控制面：`Init`、`waitForCompletion`、错误映射、`Delete` / `DeleteBatch`；
- `Put` / `PutV` / `PutBatch` / `Get` / `GetBatch` / `Exists` / `ListKeys` 返回 `NOT_SUPPORTED`，PR2 替换。

PR1 不做：Master 门控与描述符铸造（PR2）、Client value-first 删除与 miss 自愈（PR3）、SDK 相关单元测试（PR3）。

### 0.2 实现前必须固定的决策

| 项 | 默认值 | 说明 |
| --- | --- | --- |
| SDK API 版本 | `tmp_pr1641/ucm-pr` 变体 | 需要 `BinaryKey`、`CacheKey::FromBytes`、`keyResolution`、`IoOptions`；本机 `DramPool/unified-cache-management` 是旧 API，不能作为目标 |
| `hash == 0` 替代值 | `5381` | 与 v3 和 SDK `key_resolver` 的 seed 一致；不要用 `1` |
| key 字节序 | 显式小端 | 与 SDK `key_resolver.cc:51-55` 一致；不要用主机字节序 `memcpy` |
| `Delete` 归属 | PR1 实现 | `Delete` = `DeleteBatch` 单条包装；不是 `NOT_SUPPORTED` |
| 配置层测试 | 随 PR1 | 不依赖 SDK；SDK 相关测试留 PR3 |
| 超时硬门禁 | 保留 | `cfg.timeoutMs >= wait_timeout_ms`；部署侧必须把 SDK 的 `waitTimeoutMs` / `timeoutMs` 配到 >= 30000 |

### 0.3 分支

```bash
git switch main
git switch -c feat/kvclient-pr1
```

---

## 1. 实施顺序总览

| 步骤 | 内容 | 主要文件 | 验证点 |
| --- | --- | --- | --- |
| 1 | 环境变量声明 | `mooncake-common/include/environment_variables.h` | 头文件可编译 |
| 2 | 配置单元 | `include/storage/distributed/kvclient/kvclient_config.h` | `Validate()` 行为 |
| 3 | key 编码 | 并入步骤 5 的 `MakeCacheKey` | 编码结果稳定 |
| 4 | 适配器头 | 同目录 `kvclient_object_storage_adapter.h` | 类可实例化 |
| 5 | 适配器实现 | `src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp` | Init/Delete 行为 |
| 6 | 配置、工厂、注入 | 7 个既有文件 | 配置可达、后端注入 |
| 7 | CMake | `src/CMakeLists.txt` | SDK 检测 + 缺失构建 |
| 8 | 测试 | 配置层单测 | 无 SDK 可跑 |
| 9 | 构建验证 | `cmake` | 全量回归 |
| 10 | PR 提交 | PR 描述 | 边界写清 |

---

## 2. 步骤 1：环境变量声明

文件：`mooncake-common/include/environment_variables.h`

在 `KvcsEnvironmentVariables` 旁新增：

```cpp
struct KvClientEnvironmentVariables {
    MC_DEFINE_ENV_VAR(std::string, MOONCAKE_KVCLIENT_CONFIG);
    MC_DEFINE_ENV_VAR(uint64_t, MOONCAKE_KVCLIENT_MAX_OBJECT_BYTES);
    MC_DEFINE_ENV_VAR(uint64_t, MOONCAKE_KVCLIENT_WAIT_TIMEOUT_MS);
};
```

约束：

- 必须插在文件末尾 `#undef MC_DEFINE_ENV_VAR` 之前，否则宏已失效、编译失败。
- 变量名沿用 `MOONCAKE_KVCS_*` / `MOONCAKE_DFS_*` 的命名风格。
- 不要把这些变量注册到 `default_config.h`；该文件是 YAML/JSON 配置读取器，与 env 无关。

---

## 3. 步骤 2：配置单元

新增：`mooncake-store/include/storage/distributed/kvclient/kvclient_config.h`

要求：

- header-only，不包含任何 SDK 头。
- 常量：
  - `kKvClientMaxValueBytes = 16'776'704`
  - `kKvClientAlignment = 512`（本地声明，避免配置单元依赖 `kv::kAlignmentBytes`）
- `KvClientAdapterConfig`：
  - `std::string config_path`
  - `uint64_t max_object_bytes = kKvClientMaxValueBytes`
  - `uint64_t wait_timeout_ms = 30'000`
- `FromEnvironment()`：
  - `config_path` 用 `Environ::ReadOr`，显式空值交由 `Validate()` 拒绝；
  - 两个数值变量必须**严格解析**：未设置用默认值，设置但无法解析返回 `INVALID_PARAMS`；
  - 不要直接用 `Environ::ReadOr` 读数值，它会对 `abc` / `-1` 静默回退默认值。
- `Validate()`：

```cpp
return !config_path.empty() &&
       max_object_bytes > 0 &&
       max_object_bytes % kKvClientAlignment == 0 &&
       max_object_bytes <= kKvClientMaxValueBytes &&
       wait_timeout_ms > 0;
```

严格解析示意：

```cpp
static tl::expected<uint64_t, ErrorCode> ReadUint(
    const EnvironmentVariable<uint64_t>& variable, uint64_t default_value) {
    const auto raw = Environ::Process().Get(variable.name);
    if (!raw.has_value()) return default_value;
    const auto parsed = Environ::Process().GetTyped(variable);
    if (!parsed.has_value()) {
        LOG(ERROR) << "Invalid value for env " << variable.name << ": '"
                   << *raw << "'";
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    return *parsed;
}
```

依赖：`<ylt/util/tl/expected.hpp>`、`environ.h`、`glog/logging.h`。`environ.h` 是 mooncake-common 的内部头，仅限 in-tree 使用。

---

## 4. 步骤 3：key 编码（并入步骤 5）

不新增头文件；编码逻辑直接放在 `KvClientObjectStorageAdapter::MakeCacheKey`（步骤 5 的 `.cpp`）中。

```cpp
// kvclient_object_storage_adapter.cpp（匿名命名空间）
inline constexpr uint64_t kKvClientHashSeed = 5381;   // 与 SDK key_resolver 一致

kv::BinaryKey KvClientObjectStorageAdapter::MakeCacheKey(
    std::string_view logical_key) const {
    uint64_t hash =
        XXH64(logical_key.data(), logical_key.size(), kKvClientHashSeed);
    if (hash == 0) hash = kKvClientHashSeed;
    kv::BinaryKey key{};
    for (size_t i = 0; i < sizeof(hash); ++i)
        key[i] = static_cast<std::byte>(hash >> (8 * i));
    return key;
}
```

约束：

- 三个 wire 值必须同时固定：seed `5381`、零值替代 `5381`、小端字节序。
- 不要用 `std::memcpy`，否则大小端客户端会得到不同的 `CacheKey`。
- `MakeCacheKey` 是唯一编码实现；PR3 测试通过注入 fake client 截获 `DeleteAsync` 的 `CacheKey` 来断言。
- `xxhash` 已由 `Mooncake::xxhash` 提供；`mooncake_store_client_objects` 已链接。
- 编码依赖 `kv_types.h` 与 `<xxhash.h>`，因此只在 SDK 存在时编译的适配器 TU 中使用。

---

## 5. 步骤 4：适配器头

新增：`mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h`

必须声明：

```cpp
inline constexpr char kKvClientAdapterName[] = "kvclient";

class KvClientObjectStorageAdapter final : public ObjectStorageAdapter {
   public:
    explicit KvClientObjectStorageAdapter(KvClientAdapterConfig config = {});
    KvClientObjectStorageAdapter(KvClientAdapterConfig config,
                                 std::unique_ptr<kv::KvClient> client);
    ~KvClientObjectStorageAdapter() override;

    // 全部纯虚函数 override
    tl::expected<void, ErrorCode> Put(...) override;
    tl::expected<void, ErrorCode> PutV(...) override;
    tl::expected<size_t, ErrorCode> Get(...) override;
    tl::expected<bool, ErrorCode> Exists(...) override;
    tl::expected<void, ErrorCode> Delete(...) override;
    tl::expected<std::vector<KeyInfo>, ErrorCode> ListKeys() override;
    tl::expected<void, ErrorCode> Init() override;
    const char* GetName() const override { return kKvClientAdapterName; }

    std::vector<tl::expected<void, ErrorCode>> PutBatch(...) override;
    std::vector<tl::expected<size_t, ErrorCode>> GetBatch(...) override;
    ObjectStorageIoResults DeleteBatch(...) override;

   private:
    struct Completion {
        bool terminal = false;
        std::vector<kv::Status> entryStatus;
    };
    // MakeCacheKey / waitForCompletion / ToErrorCode /
    // config_ / kv_client_ / initialized_
};
```

约束：

- `GetName` 的真实签名是 `const char* GetName() const override`，不是 `char*`；否则类保持抽象。
- 必须 override `object_storage_adapter.h` 的全部纯虚函数，否则 `make_unique` 无法编译。
- 注入构造是 PR3 fake SDK 测试的接缝，PR1 就要声明。
- PR1 不引入 `BufferPlan`、`RegisterBuffers`、`ReleaseBuffers`、`RetainOrphans`、`orphans_`；这些属于 PR2。

---

## 6. 步骤 5：适配器实现

新增：`mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp`

### 6.1 Init 顺序

```cpp
tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::Init() {
    if (initialized_) return {};
    if (!config_.Validate()) return tl::make_unexpected(ErrorCode::INVALID_PARAMS);

    // 注入 client：测试缝，只校验适配器配置
    if (kv_client_) {
        initialized_ = true;
        return {};
    }

    auto client = kv::CreateKvClient();      // 局部变量，成功后才赋值
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
    if (!client->Init(cfg).ok())
        return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE);

    kv_client_ = std::move(client);
    initialized_ = true;
    LOG(INFO) << "KVClient adapter initialized, max_object_bytes="
              << config_.max_object_bytes;
    return {};
}
```

关键点：

- SDK `Init` 非幂等（第二次返回 `RESOURCE_BUSY`），所以 client 必须创建到局部变量，成功后才能写入成员。
- 每次重试都重新执行 `LoadFromFile`、`keyResolution` 复位、超时校验、SDK `Init`。
- 失败路径不要留下"非空但未初始化"的 `kv_client_`。
- Init 失败日志必须同时打印 `cfg.timeoutMs` 与 `wait_timeout_ms`。

### 6.2 析构

```cpp
KvClientObjectStorageAdapter::~KvClientObjectStorageAdapter() {
    if (kv_client_) kv_client_->Shutdown();
}
```

### 6.3 错误映射

| SDK `StatusCode` | `ErrorCode` |
| --- | --- |
| `OK` | `OK` |
| `TIMEOUT`、`CQE_IO_TIMEOUT` | `RPC_TIMEOUT` |
| `IN_PROGRESS`、`CANCELED` | `INTERNAL_ERROR` |
| `CQE_KEY_NOT_EXIST`、`ENTRY_KEY_NOT_FOUND`、`ENTRY_KEY_NOT_EXIST` | `OBJECT_NOT_FOUND` |
| `NOT_FOUND`（路由/配置/视图错误） | `DFS_SERVICE_UNAVAILABLE` |
| `NOT_INITIALIZED`、`CONNECTION_ERROR`、`IO_ERROR`、`UNSUPPORTED`、`ENTRY_DELETE_FAILED` | `DFS_SERVICE_UNAVAILABLE` |
| `INVALID_ARGUMENT`、`BUFFER_NOT_REGISTERED`、`BUFFER_NOT_SUPPORTED` | `INVALID_PARAMS` |
| `CQE_OUT_OF_CREATE_SIZE`、`RESOURCE_BUSY`、`CQE_RESOURCE_BUSY` | `NO_AVAILABLE_HANDLE` |
| 其他 | `INTERNAL_ERROR` |

注意：

- `NOT_FOUND` 不是对象 miss；不能映射为 `OBJECT_NOT_FOUND`，否则 PR3 会把删除失败当幂等成功，PR2 会把读路由失败当对象缺失。
- `ENTRY_KEY_NOT_EXIST` 是 QUERY 专用；DELETE 的逐项失败码是 `ENTRY_DELETE_FAILED` / `IO_ERROR`。

### 6.4 waitForCompletion

```cpp
const auto wait_status =
    kv_client_->Wait(task_id, config_.wait_timeout_ms, result);
const auto code = wait_status.ok() ? result.status.code : wait_status.code;
const kv::Status& task_status = wait_status.ok() ? result.status : wait_status;

Completion c;
switch (code) {
    case kv::StatusCode::TASK_NOT_FOUND:
        // Wait 超时后也会移除 task；未知 task 不能证明 I/O 结束
        c.terminal = false;
        c.entryStatus.assign(entry_count, kv::Status::Error(
            kv::StatusCode::INTERNAL_ERROR, "task not found"));
        return c;
    case kv::StatusCode::TIMEOUT:
    case kv::StatusCode::IN_PROGRESS:
    case kv::StatusCode::CANCELED:
        // SDK 可能已填充 entryStatus，但 task 未终态；逐项结果不可信
        c.terminal = false;
        c.entryStatus.assign(entry_count, task_status);
        return c;
    default:
        // task 已结束：OK / PARTIAL_FAILED 以及任意 task 级最终错误
        c.terminal = true;
        break;
}
// entry 数不匹配 -> 逐项 INTERNAL_ERROR；否则搬运 result.entryStatus
```

### 6.5 删除

`DeleteBatch`：

1. 入口检查 `!initialized_ || !kv_client_`，逐项返回 `DFS_SERVICE_UNAVAILABLE`；
2. 空 key 单项 `INVALID_PARAMS`；
3. `MakeCacheKey` → `CacheKey::FromBytes`；
4. `DeleteAsync` → `waitForCompletion`；
5. 按下标回填逐项结果。

`Delete`：

```cpp
std::array<std::string, 1> keys{key};
auto results = DeleteBatch(keys);
return results.empty()
           ? tl::make_unexpected(ErrorCode::INVALID_PARAMS)
           : std::move(results.front());
```

### 6.6 占位方法

`Put`、`PutV`、`PutBatch`、`Get`、`GetBatch`、`Exists`、`ListKeys` 返回按请求数填充的 `NOT_SUPPORTED`。

不要照抄 PR2 的四字段构造：

```cpp
// PR1 的 ObjectPutRequest 只有三个字段；capacity 是 PR2 才加
ObjectPutRequest{key, &iov, 1, 0}   // 错误：PR1 不编译
```

### 6.7 工厂

```cpp
tl::expected<std::unique_ptr<ObjectStorageAdapter>, ErrorCode>
CreateKvClientObjectStorageAdapter() {
    auto config = KvClientAdapterConfig::FromEnvironment();
    if (!config) return tl::make_unexpected(config.error());
    return std::make_unique<KvClientObjectStorageAdapter>(*config);
}
```

工厂返回 `unique_ptr<ObjectStorageAdapter>`，调用方必须包装进 `DistributedStorageBackend`，不能直接作为 `CreateStorageBackend` 的返回值。

---

## 7. 步骤 6：配置、工厂与注入

按依赖顺序修改：

### 7.1 `mooncake-store/include/config/distributed_storage_config.h`

```cpp
bool UsesObjectStorage() const {
    return fs_adapter_type == "oss" || fs_adapter_type == "s3" ||
           fs_adapter_type == "kvcs-lowlevel" ||
           fs_adapter_type == "kvclient";
}
bool UsesKvcs() const { return fs_adapter_type == "kvcs-lowlevel"; }
bool UsesKvClient() const { return fs_adapter_type == "kvclient"; }
```

### 7.2 `mooncake-store/src/config/distributed_storage_config.cpp`

在 KVCS 分支旁新增，放在 `fsdir` 检查之前：

```cpp
if (UsesKvClient()) {
    if (!single_tenant) {
        // Client::Put/BatchPut -> WriteDfsReplicas -> DfsWriteRequest ->
        // ObjectPutRequest 传的是 raw user key；只有 offload/read RPC 路径
        // 用 TenantId::MakeScopedKey。SDK 的 IoOptions.kvns 当前未被消费，
        // 多租户会出现跨租户 key 碰撞。
        LOG(ERROR) << "DistributedStorageConfig: KVClient requires "
                      "single_tenant=true";
        return false;
    }
    // KVClient 没有 capacity API，不要求 eviction_check_interval
    return true;
}
```

### 7.3 `mooncake-store/src/config/file_storage_config.cpp`

原谓词只判断 KVCS；现在先解析一次分布式配置，再判断两种远程对象后端：

```cpp
bool uses_remote_object_storage = false;
if (storage_backend_type == StorageBackendType::kDistributed) {
    const auto distributed_config =
        DistributedStorageConfig::FromEnvironment();
    uses_remote_object_storage = distributed_config.UsesKvcs() ||
                                 distributed_config.UsesKvClient();
}
if (!uses_remote_object_storage) {
```

不要放宽到 `UsesObjectStorage()`，否则会改变 oss/s3 的本地目录校验行为。

### 7.4 `mooncake-store/include/storage/distributed/distributed_storage_backend.h`

```cpp
bool UsesKvClient() const {
    return UsesObjectStorage() && distributed_config_.UsesKvClient();
}
```

### 7.5 `mooncake-store/src/storage_backend.cpp`

- 只在 `MOONCAKE_HAVE_KVCLIENT_SDK` 内包含厂商无关的 `kvclient_adapter_factory.h`；
- 不要包含 `kvclient_object_storage_adapter.h`；
- 在 `UsesKvcs()` 分支后新增：

```cpp
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
```

- 与既有分支共用收尾：

```cpp
return std::make_shared<DistributedStorageBackend>(
    config, distributed_config, std::move(fs_adapter),
    std::move(object_storage_adapter));
```

### 7.6 `mooncake-store/src/file_storage.cpp`

```cpp
config_.enable_dfs = !distributed_backend->UsesObjectStorage() ||
                     distributed_backend->UsesKvcs() ||
                     distributed_backend->UsesKvClient();
```

### 7.7 `mooncake-store/src/real_client.cpp`

```cpp
const bool enable_kvcs = distributed_config.UsesKvcs();
const bool enable_kvclient = distributed_config.UsesKvClient();
if (enable_ssd_offload || enable_kvcs || enable_kvclient) {
    auto file_storage_config = FileStorageConfig::FromEnvironment();
    if (enable_kvcs || enable_kvclient) {
        file_storage_config.storage_backend_type =
            StorageBackendType::kDistributed;
    }
    // ...
}
```

注意：只改 `if (enable_ssd_offload || enable_kvcs)` 而漏掉 `if (enable_kvcs)` 的强制 `kDistributed`，会静默掉回 bucket 后端。

---

## 8. 步骤 7：CMake

文件：`mooncake-store/src/CMakeLists.txt`

### 8.1 检测块（pkg-config 或 find_path/find_library）

```cmake
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
```

说明：SDK 不生成 `kvclient.pc`，pkg-config 仅作未来兼容；主路径是 `MOONCAKE_KVCLIENT_SDK_ROOT`（源码树或安装前缀）或显式 `MOONCAKE_KVCLIENT_INCLUDE_DIR` / `MOONCAKE_KVCLIENT_LIBRARY`。SDK 源码树可作为 root：头文件在 `kv_semantics/include`，库在 `build/src` 或安装目录 `kv_semantics/src`。`MOONCAKE_KVCLIENT_REQUIRED=ON` 或显式设置 `MOONCAKE_KVCLIENT_SDK_ROOT` 时未找到 SDK 会让 configure 直接失败。

### 8.2 条件源（必须在 `add_library(mooncake_store_client_objects ...)` 之前）

```cmake
if(MOONCAKE_KVCLIENT_SDK_FOUND)
  list(APPEND MOONCAKE_STORE_CLIENT_SOURCES
       storage/distributed/kvclient/kvclient_object_storage_adapter.cpp)
endif()
```

### 8.3 define/link（挂 client objects，不是 shared objects）

```cmake
if(MOONCAKE_KVCLIENT_SDK_FOUND)
  # storage_backend.cpp and the adapter TU are both compiled into this object
  # library, so the define must live here rather than on shared objects.
  target_compile_definitions(mooncake_store_client_objects
                             PRIVATE MOONCAKE_HAVE_KVCLIENT_SDK=1)
  target_link_libraries(mooncake_store_client_objects
                        PRIVATE Mooncake::kvclient)
endif()
```

### 8.4 最终库再链一次

```cmake
if(MOONCAKE_KVCLIENT_SDK_FOUND)
  # OBJECT-library PRIVATE links do not propagate to the final link.
  target_link_libraries(mooncake_store PRIVATE Mooncake::kvclient)
endif()
```

约束：

- `mooncake_store_master` 不链接 KVClient SDK。
- 适配器源不得放入 `MOONCAKE_STORE_SHARED_SOURCES`（该对象库会进入 master）。
- SDK 缺失时适配器 TU 不参与编译，工厂 `#else` 分支返回 `NOT_SUPPORTED`，不引用任何 SDK 符号。
- SDK 存在但检测不到时，`MOONCAKE_KVCLIENT_REQUIRED=ON` 或显式 `MOONCAKE_KVCLIENT_SDK_ROOT` 会让 configure 直接失败，不会静默跳过适配器。
- SDK 不生成 `kvclient.pc`：用 `MOONCAKE_KVCLIENT_SDK_ROOT` 指向源码树/安装前缀，或直接设置 `MOONCAKE_KVCLIENT_INCLUDE_DIR` + `MOONCAKE_KVCLIENT_LIBRARY`。

---

## 9. 步骤 8：测试

### 9.1 PR1 必带（不依赖 SDK）

在 `mooncake-store/tests/distributed_storage_config_test.cpp` 增加：

| 用例 | 预期 |
| --- | --- |
| `fs_adapter_type = "kvclient"` | `Validate()` 通过；`UsesObjectStorage()` 与 `UsesKvClient()` 为真 |
| `kvclient` + `single_tenant=false` | `Validate()` 失败 |
| `kvclient` + 空/相对 `fsdir` | 不被拒绝（走 kvclient 提前返回分支） |
| `kvclient` + `ValidateForAllocator()` | 返回 false（kvclient 是对象存储，不是文件系统 allocator 后端） |
| 默认配置 | `UsesKvClient()` 为假，既有断言不变 |

工厂行为测试只在 SDK 缺失构建中编译：`tests/CMakeLists.txt` 在 `NOT MOONCAKE_KVCLIENT_SDK_FOUND` 时给 `distributed_storage_config_test` 定义 `MOONCAKE_KVCLIENT_TEST_NO_SDK=1`；该用例设置 `MOONCAKE_DFS_FS_ADAPTER=kvclient`，断言 `CreateStorageBackend` 返回 `NOT_SUPPORTED`。

### 9.2 留 PR3（需 SDK）

- `kvclient_object_storage_adapter_test.cpp`：Init 门禁（timeout）、`MakeCacheKey` 稳定性/小端、`DeleteBatch` 守卫、错误映射。
- 测试目标只在 SDK 存在时创建；若测试直接计算 `XXH64` 作断言，需额外链接 `Mooncake::xxhash`。

---

## 10. 步骤 9：构建验证

### 10.1 SDK 缺失构建（PR1 主门槛）

```bash
cmake -S . -B build -DWITH_STORE=ON -DBUILD_UNIT_TESTS=ON
cmake --build build -j"$(nproc)"
ctest --test-dir build --output-on-failure
```

必须确认：

- 适配器 TU 未参与编译；
- `mooncake_store_master` 没有 kvclient 依赖；
- 既有后端测试全绿；
- 设置 `MOONCAKE_DFS_FS_ADAPTER=kvclient` 时工厂返回 `NOT_SUPPORTED`。

### 10.2 SDK 存在构建（打包落地后或指向 SDK 源码树）

```bash
cmake -S . -B build-sdk -DWITH_STORE=ON -DBUILD_UNIT_TESTS=ON \
  -DMOONCAKE_KVCLIENT_SDK_ROOT=/path/to/kv_semantics \
  -DMOONCAKE_KVCLIENT_REQUIRED=ON
cmake --build build-sdk -j"$(nproc)"
```

确认 `MOONCAKE_HAVE_KVCLIENT_SDK` 生效、`mooncake_store_client_objects` 与 `mooncake_store` 都链到 `Mooncake::kvclient`。SDK 头文件在 `kv_semantics/include`，库在 `build/src` 或安装目录 `kv_semantics/src`。

### 10.3 提交前格式检查

```bash
pre-commit run --files <本次改动文件>
```

不要使用 `pre-commit run --all-files`。

---

## 11. 步骤 10：PR 提交

- 标题：`[Store] Add kvclient backend wiring and adapter control plane`
- PR 描述必须写清：
  - Master 未改，端到端读写不可用；
  - `Put/PutV/PutBatch/Get/GetBatch` 返回 `NOT_SUPPORTED`；
  - SDK 缺失时工厂返回 `NOT_SUPPORTED`；
  - `MOONCAKE_ENABLE_DFS=1` + oplog/snapshot 不适用于 kvclient；
  - `waitTimeoutMs >= wait_timeout_ms` 的部署要求。
- 使用 `.github/pull_request_template.md`，填写模块、类型、测试、检查清单、AI 协助披露。
- 人工逐行审阅所有改动。

---

## 12. 验收清单

| 项 | 通过标准 |
| --- | --- |
| 配置可达 | `fs_adapter_type=kvclient` 通过 `DistributedStorageConfig::Validate()` |
| 工厂可达 | 有 SDK（pkg-config 或 SDK_ROOT/include/library 检测到）时创建 adapter；无 SDK 时 `NOT_SUPPORTED`；`REQUIRED=ON` 或设置了 `SDK_ROOT` 时缺失在 configure 失败 |
| CMake 检测 | SDK 无 `kvclient.pc` 时仍可通过 `MOONCAKE_KVCLIENT_SDK_ROOT` / include / library 编译适配器 |
| 注入可达 | `FileStorage::enable_dfs` 为真并调用 `SetDfsStorageBackend` |
| Init 门禁 | 非法配置 / SDK Init 失败 / timeout 不足按预期失败 |
| Init 重试 | 首次失败后重试不会假成功，不会撞 `RESOURCE_BUSY` |
| 删除 | `Delete` / `DeleteBatch` 可用；未 Init 时逐项 `DFS_SERVICE_UNAVAILABLE` |
| 读写 | 五个数据入口返回 `NOT_SUPPORTED` |
| CMake | define/link 在 client objects；最终库再链；master 不链 |
| 回归 | 既有后端与单测全绿；SDK 缺失构建无未定义符号 |

---

## 13. 实现约束速查（不要照抄旧伪代码）

1. **Init**：client 创建到局部变量，成功后才写成员；每次重试重新校验。
2. **工厂**：返回 adapter 后必须包装 `DistributedStorageBackend`，不能直接 `return`。
3. **DeleteBatch**：入口检查 `initialized_` / `kv_client_`。
4. **Delete**：PR1 实现为 `DeleteBatch` 单条包装。
5. **类声明**：`const char* GetName() const override`，override 全部纯虚，声明析构与注入构造。
6. **CMake**：define 挂 `mooncake_store_client_objects`，不是 `mooncake_store_shared_objects`。
7. **waitForCompletion**：`TASK_NOT_FOUND` / `TIMEOUT` / `IN_PROGRESS` / `CANCELED` 非终态；其余 task 最终错误终态。
8. **错误映射**：`NOT_FOUND`、`NOT_INITIALIZED`、`ENTRY_DELETE_FAILED` → `DFS_SERVICE_UNAVAILABLE`；只有 entry 级 key-not-found → `OBJECT_NOT_FOUND`。
9. **配置**：`max_object_bytes > 0`；数值 env 严格解析；`timeoutMs` 不足时打印两个值。
10. **`MakeCacheKey`**：零值替代 `5381`；显式小端；不用 `memcpy`。

---

## 14. 已知风险与待确认

| 风险 | 处理 |
| --- | --- |
| SDK 未打包，无 `kvclient.pc` 且头文件未安装 | 用 `MOONCAKE_KVCLIENT_SDK_ROOT` 指向源码树/安装前缀，或显式 include/library；`REQUIRED=ON` 或显式 `SDK_ROOT` 让缺失在 configure 失败 |
| 两个本地 SDK 快照 API 不一致 | 固定 `tmp_pr1641/ucm-pr` 变体，并在 CMake 检测处记录版本 |
| 默认 `wait_timeout_ms=30000` 与 SDK 默认 100 ms 冲突 | 部署配置必须显式提高 `waitTimeoutMs` / `timeoutMs`；Init 日志打印两个值 |
| `MOONCAKE_DISTRIBUTED_HEALTH_CHECK=1` | PR1 未实现 `CheckHealth`，会 `NOT_SUPPORTED`；文档写明必须关闭 |
| Master 未识别 kvclient | PR1 阶段不要启用 `MOONCAKE_ENABLE_DFS=1`；端到端等 PR2 |
| `DeleteBatch` 在 PR1 生产路径不可达 | `BatchDeleteObjects` 目前只在 `UsesKvcs()` 门内调用；PR1 通过直接调用后端/适配器验收 |
| XXH64 非单射 | 接受碰撞风险；部署限定 key 数量级；PR3 测试名称不要写 "Injective" |

---

## 15. 代码改动量

统计范围：`Mooncake` 工作树相对 `main` 的未提交改动；不含 `Mooncake_documents` 文档。

| 指标 | 数量 |
| --- | ---: |
| 代码文件 | 15（11 修改 + 4 新增） |
| 物理行新增 | +602 |
| 物理行删除 | -11 |

### 15.1 分类

| 类别 | 文件数 | 新增 | 删除 |
| --- | ---: | ---: | ---: |
| 生产代码（修改） | 9 | +95 | -11 |
| 生产代码（新增） | 4 | +444 | 0 |
| 测试代码 | 2 | +63 | 0 |
| **合计** | **15** | **+602** | **-11** |

### 15.2 新增文件

| 文件 | 行数 |
| --- | ---: |
| `mooncake-store/include/storage/distributed/kvclient/kvclient_adapter_factory.h` | 16 |
| `mooncake-store/include/storage/distributed/kvclient/kvclient_config.h` | 78 |
| `mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h` | 63 |
| `mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp` | 287 |
| **合计** | **444** |

### 15.3 修改文件

| 文件 | 新增 | 删除 |
| --- | ---: | ---: |
| `mooncake-common/include/environment_variables.h` | +6 | 0 |
| `mooncake-store/include/config/distributed_storage_config.h` | +3 | -1 |
| `mooncake-store/include/storage/distributed/distributed_storage_backend.h` | +4 | 0 |
| `mooncake-store/src/CMakeLists.txt` | +38 | 0 |
| `mooncake-store/src/config/distributed_storage_config.cpp` | +14 | 0 |
| `mooncake-store/src/config/file_storage_config.cpp` | +11 | -7 |
| `mooncake-store/src/file_storage.cpp` | +2 | -1 |
| `mooncake-store/src/real_client.cpp` | +3 | -2 |
| `mooncake-store/src/storage_backend.cpp` | +14 | 0 |
| `mooncake-store/tests/CMakeLists.txt` | +4 | 0 |
| `mooncake-store/tests/distributed_storage_config_test.cpp` | +59 | 0 |
| **合计** | **+158** | **-11** |

> `kvclient_key_codec.h` 曾创建后合并进 `MakeCacheKey` 并删除，不计入最终行数。
