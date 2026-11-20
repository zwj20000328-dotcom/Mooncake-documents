# Mooncake + KVClient L3 接入（Kvcs）— PR1 逐行代码解读

版本：v1.0　日期：2026-11-20

对应文档：`Mooncake + KVClient L3 接入（Kvcs）- PR 实施设计.md`（v1.5）、`... - PR1 实施步骤.md`（v1.1）

---

## 0. 阅读约定

### 0.1 基线与范围

| 项 | 值 |
| --- | --- |
| 分支 | `Mooncake_l3_pr2`（含 PR1 全部提交） |
| 合并基线 | `e43ee9c4`（`git merge-base upstream/main HEAD`） |
| PR1 提交 | `93ec9401`、`5c4162e4`、`757edd7c`、`86fe246e`、`f3c8ffd5`、`95f88152` |
| 改动规模 | 15 个文件，+625 / −11 |
| 目标 SDK | `tmp_pr1641/ucm-pr` 变体（分支 `feat/kvclient-typed-keys-namespaces`，HEAD `f3e601f`） |

### 0.2 行号规则

- 链接格式 `../Mooncake/<仓库相对路径>#L<行号>`，点击直接跳转。
- **其余 13 个文件**工作区与 PR1 提交版**完全一致**（已逐文件 `git diff --quiet HEAD` 验证），链接在任何情况下都能命中。
- **两个适配器文件是例外**，因为工作区正含未提交且持续变动的 PR2 改动：
  - `kvclient_object_storage_adapter.h`（新增 `ToWriteErrorCode`、`RegisterBuffers`、`UnregisterBuffers` 声明）
  - `kvclient_object_storage_adapter.cpp`（`Put`/`PutV`/`PutBatch`/`Get`/`GetBatch`/`Exists` 已被 PR2 实现替换，新增 buffer 注册逻辑）

  工作区行号随每次编辑漂移，指向它无法稳定跳转。因此：

  | 元素 | 编号基准 |
  | --- | --- |
  | 代码块右侧标注的主行号 `// Lnn` | 编写时的工作区（仅作阅读参照） |
  | 代码块右侧的 **（HEAD:nn）** | **PR1 提交版 `95f88152`** ← **链接指向这里** |
  | 正文 `**Lnn**` 与链接文字 | 与所在代码块的 `// Lnn` 一致 |
  | 标注 **【PR2，不属本 PR】** 的区域 | 工作区（无 HEAD 对应，链接仅作定位参考） |

  换言之：点击这两个文件的链接会跳到 **PR1 提交版**的行号，也就是每行 **（HEAD:nn）** 标注的那个数字。要看跳转后的实际内容：
  ```bash
  git worktree add ../Mooncake-pr1 95f88152   # 推荐：另开工作树，不影响当前 PR2 开发
  # 或只看单文件： git show 95f88152:<path>
  ```

### 0.3 验证状态

| 验证项 | 结果 |
| --- | --- |
| CMake configure（SDK 缺失） | 通过，正确报告 `KVClient SDK not found: adapter disabled` |
| CMake configure（SDK 存在） | 通过，报告 `KVClient SDK found: include=..., library=...` |
| 适配器 TU 实际编译 | **通过**，生成 `kvclient_object_storage_adapter.cpp.o`（1.9 MB），零错误零警告 |
| `mooncake_store_client_objects` | exit 0，`error:` 计数 0 |
| `mooncake_store` 静态库 | exit 0 |
| `distributed_storage_config_test` | **18/18 通过** |

编译环境：WSL Ubuntu，GCC 15，CMake 4.2.3 + Ninja，`Release`。

---

## 1. 改动总览

| # | 文件 | 类型 | PR1 作用 |
| --- | --- | --- | --- |
| 1 | [environment_variables.h](../Mooncake/mooncake-common/include/environment_variables.h#L176) | 修改 | 声明 `MOONCAKE_KVCLIENT_CONFIG` |
| 2 | [kvclient_config.h](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_config.h) | **新增** | SDK-free 配置单元 |
| 3 | [kvclient_adapter_factory.h](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_adapter_factory.h) | **新增** | SDK-free 工厂声明 |
| 4 | [kvclient_object_storage_adapter.h](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h) | **新增** | 适配器类声明 |
| 5 | [kvclient_object_storage_adapter.cpp](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp) | **新增** | 适配器实现（控制面） |
| 6 | [distributed_storage_config.h](../Mooncake/mooncake-store/include/config/distributed_storage_config.h#L27) | 修改 | 识别 `kvclient` 后端类型 |
| 7 | [distributed_storage_config.cpp](../Mooncake/mooncake-store/src/config/distributed_storage_config.cpp#L78) | 修改 | 校验分支：强制单租户、跳过文件系统校验 |
| 8 | [distributed_storage_backend.h](../Mooncake/mooncake-store/include/storage/distributed/distributed_storage_backend.h#L78) | 修改 | 后端级 `UsesKvClient()` 谓词 |
| 9 | [file_storage_config.cpp](../Mooncake/mooncake-store/src/config/file_storage_config.cpp#L238) | 修改 | 跳过本地目录校验 |
| 10 | [file_storage.cpp](../Mooncake/mooncake-store/src/file_storage.cpp#L162) | 修改 | 启用 DFS 路径 |
| 11 | [real_client.cpp](../Mooncake/mooncake-store/src/real_client.cpp#L1618) | 修改 | 客户端启动接线 |
| 12 | [storage_backend.cpp](../Mooncake/mooncake-store/src/storage_backend.cpp#L6522) | 修改 | 创建适配器 |
| 13 | [src/CMakeLists.txt](../Mooncake/mooncake-store/src/CMakeLists.txt#L223) | 修改 | SDK 检测 + 编译接线 |
| 14 | [tests/CMakeLists.txt](../Mooncake/mooncake-store/tests/CMakeLists.txt#L402) | 修改 | 测试宏门控 |
| 15 | [distributed_storage_config_test.cpp](../Mooncake/mooncake-store/tests/distributed_storage_config_test.cpp#L194) | 修改 | 5 个新测试 |

依赖顺序（阅读顺序）：**1 → 2 → 3 → 4 → 5**（适配器自身），**6 → 7 → 8**（配置层），**9 → 10 → 11 → 12**（接线），**13 → 14 → 15**（构建与测试）。

---

## 2. 环境变量声明

文件：[mooncake-common/include/environment_variables.h](../Mooncake/mooncake-common/include/environment_variables.h#L176)

```cpp
struct KvClientEnvironmentVariables {                                  // L176
    MC_DEFINE_ENV_VAR(std::string, MOONCAKE_KVCLIENT_CONFIG);           // L177
};                                                                     // L178
```

- **L176** — 新增结构体，位置紧跟既有的 `KvcsEnvironmentVariables`（[L165-L174](../Mooncake/mooncake-common/include/environment_variables.h#L165)），让同类后端的声明聚在一起，便于对照维护。
- **L177** — `MC_DEFINE_ENV_VAR` 定义在 [L10-L11](../Mooncake/mooncake-common/include/environment_variables.h#L10)，展开为：
  ```cpp
  inline static constexpr EnvironmentVariable<std::string> MOONCAKE_KVCLIENT_CONFIG { "MOONCAKE_KVCLIENT_CONFIG" };
  ```
  宏把**类型**与**名字字符串**一起绑进一个编译期常量，杜绝手写字符串拼错；`inline` 允许在头文件中定义而不违反 ODR（每个 TU 共享同一实体）。
- **L178** — 无需在任何注册表登记。已 grep 确认：`KvcsEnvironmentVariables` 同样只被使用点引用（`kvcs_capi_driver.cpp:41`），`OssAdapterEnvironmentVariables` 同理（`oss_adapter_config.cpp:22`）。这些结构体纯粹是命名空间式分组。

**为何只有一个变量**：`MOONCAKE_KVCLIENT_MAX_OBJECT_BYTES` 与 `MOONCAKE_KVCLIENT_WAIT_TIMEOUT_MS` 已按 @ayaka836 的要求删除——超时归 SDK，大小与对齐由 KvClient 自行判定。

---

## 3. SDK-free 配置单元

文件：[mooncake-store/include/storage/distributed/kvclient/kvclient_config.h](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_config.h)（新增，31 行）

### 3.1 include 区

```cpp
#pragma once                                                          // L1
                                                                      // L2
#include <string>                                                     // L3
                                                                      // L4
#include <ylt/util/tl/expected.hpp>                                   // L5
                                                                      // L6
#include "environment_variables.h"                                    // L7
#include "environ.h"                                                  // L8
#include "types.h"                                                    // L9
```

- **L1** — `#pragma once`，与仓库其他头文件一致。
- **L3** — 只需 `std::string`（`config_path` 的类型）。已删掉 `<cstddef>` / `<cstdint>` / `<glog/logging.h>`，它们只服务于被移除的对齐常量与 `ReadUint`。
- **L5** — `tl::expected`，`FromEnvironment()` 的返回类型。
- **L7** — 引入第 2 节的 `KvClientEnvironmentVariables`。
- **L8** — `Environ::ReadOr` 的实现头。
- **L9** — `ErrorCode`。

> **本文件最重要的约束**（写在 [L13-L14](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_config.h#L13) 的注释里）：此头**必须能在没有 SDK 的环境下编译**。因为 `storage_backend.cpp` 在 SDK 缺失时也要编译（它含 `#else` 分支），而它经由工厂头间接依赖本文件。所以这里不 include 任何 `kv_*.h`。

### 3.2 结构体

```cpp
namespace mooncake {                                                  // L11
                                                                      // L12
// SDK-free configuration unit: this header must stay includable without the
// KVClient SDK.                                                       // L13-L14
struct KvClientAdapterConfig {                                        // L15
    std::string config_path;                                          // L16
```

- **L13-L14** — 注释固化上述约束，防止后人往这里加 SDK 依赖。
- **L15-L16** — 唯一字段：SDK 配置文件路径。

### 3.3 `FromEnvironment()`

```cpp
    static tl::expected<KvClientAdapterConfig, ErrorCode> FromEnvironment() {  // L18
        KvClientAdapterConfig cfg;                                     // L19
        cfg.config_path = Environ::ReadOr(                             // L20
            KvClientEnvironmentVariables::MOONCAKE_KVCLIENT_CONFIG,     // L21
            std::string{});                                            // L22
        if (!cfg.Validate())                                           // L23
            return tl::make_unexpected(ErrorCode::INVALID_PARAMS);     // L24
        return cfg;                                                    // L25
    }                                                                  // L26
```

- **L18** — 工厂入口。返回 `expected` 而非抛异常，与仓库惯例一致（对照 `DistributedStorageConfig::FromEnvironment`）。
- **L19** — 值初始化，`config_path` 为空串。
- **L20-L22** — `ReadOr(变量, 默认值)`：环境变量未设置时回落到 `std::string{}`。因为类型是 `string`，不存在"数值解析失败静默用默认值"的坑——这正是被删掉的 `ReadUint` 要解决的问题，对 string 而言不需要该防护。
- **L23-L24** — 空路径即非法，早失败，错误码 `INVALID_PARAMS`。
- **L25** — 按值返回，NRVO / 移动语义消除拷贝。

### 3.4 `Validate()`

```cpp
    bool Validate() const { return !config_path.empty(); }             // L28
};                                                                    // L29
                                                                      // L30
}  // namespace mooncake                                              // L31
```

- **L28** — 唯一校验规则。@ayaka836 要求移除 max-object-size 与 512B 对齐校验后，原来的四项合取式收缩成一项；`kKvClientMaxValueBytes`、`kKvClientAlignment` 两个常量随之删除。
- **为何保留 `Validate()` 而不内联到 `FromEnvironment()`** — `Init()`（[adapter .cpp:42](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L42)）也要调它，用于测试注入路径的配置校验。

---

## 4. SDK-free 工厂声明

文件：[mooncake-store/include/storage/distributed/kvclient/kvclient_adapter_factory.h](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_adapter_factory.h)（新增，16 行）

```cpp
#pragma once                                                          // L1
                                                                      // L2
#include <memory>                                                     // L3
                                                                      // L4
#include <ylt/util/tl/expected.hpp>                                   // L5
                                                                      // L6
#include "storage/distributed/object_storage_adapter.h"               // L7
                                                                      // L8
namespace mooncake {                                                  // L9
                                                                      // L10
// Vendor-neutral factory declaration: this header is SDK-free and is the only
// kvclient header storage_backend.cpp may include.                    // L11-L12
tl::expected<std::unique_ptr<ObjectStorageAdapter>, ErrorCode>         // L13
CreateKvClientObjectStorageAdapter();                                  // L14
                                                                      // L15
}  // namespace mooncake                                              // L16
```

- **L3** — `std::unique_ptr`。
- **L5** — `tl::expected`。
- **L7** — 基类 `ObjectStorageAdapter` 的**完整定义**。声明返回 `unique_ptr<ObjectStorageAdapter>` 需要基类可见（`unique_ptr` 的默认删除器要求完整类型）。
- **L11-L12** — 注释是这个文件**存在的全部理由**：`storage_backend.cpp` 在 SDK 缺失时也必须编译，所以它只能 include 这个不含 `kv_*.h` 的头。若直接 include 适配器头，SDK-less 构建会因找不到 `kv_client.h` 而失败。
- **L13-L14** — 返回**基类**指针而非 `KvClientObjectStorageAdapter*`，这样调用方无需知道派生类、也无需 SDK 头。`expected` 包裹以传播配置错误。

**文件名的由来**：原名 `kvclient_backend.h`，提交 `757edd7c` 改为现名。旧名与 `DistributedStorageBackend` 概念冲突——它不是 backend，而是 adapter 的工厂。

---

## 5. 适配器类声明

文件：[mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h)（新增，PR1 为 63 行）

### 5.1 include 区

```cpp
#pragma once                                                          // L1
                                                                      // L2
#include <cstdint>                                                    // L3
#include <memory>                                                     // L4
#include <string>                                                     // L5
#include <string_view>                                                // L6
#include <vector>                                                     // L7
                                                                      // L8
#include "kv_client.h"                                                // L9
#include "kv_types.h"                                                 // L10
#include "storage/distributed/kvclient/kvclient_config.h"              // L11
#include "storage/distributed/object_storage_adapter.h"                // L12
```

- **L3** — `uint64_t`。**【PR2 新增】** PR1 只需 `<memory>` / `<string>` / `<string_view>` / `<vector>`；`<cstdint>` 是 PR2 的 `RegisterBuffers` 返回 `uint64_t` 才引入的（PR1 提交版为 HEAD:3-7，无 `<cstdint>`）。
- **L4** — `std::unique_ptr<kv::KvClient>`。
- **L5** — `std::string`（接口参数）。
- **L6** — `string_view`（`MakeCacheKey` 参数）。
- **L7** — `std::vector`（`Completion::entryStatus`）。
- **L9-L10** — SDK 头。**这两行决定了本文件只能被 SDK-enabled 的 TU include**，故 L17-L18 有显式警告注释。`kv_client.h` 提供 `kv::KvClient` / `CreateKvClient`；`kv_types.h` 提供 `BinaryKey` / `CacheKey` / `StatusCode` / `Status` / `TaskId` / `TaskResult` / `KvClientConfig`。
- **L11** — 第 3 节的配置结构体。
- **L12** — 基类。

### 5.2 适配器名与类头

```cpp
namespace mooncake {                                                  // L14
                                                                      // L15
inline constexpr char kKvClientAdapterName[] = "kvclient";            // L16（HEAD:15）
                                                                      // L17
// This header includes the KVClient SDK, so only SDK-enabled TUs may include
// it. Keep it out of storage_backend.cpp and every SDK-less unit.    // L18-L19（HEAD:17-18）
class KvClientObjectStorageAdapter final : public ObjectStorageAdapter {  // L20（HEAD:19）
   public:                                                            // L21（HEAD:20）
```

- **L16** — `GetName()` 的返回值。`inline constexpr` 允许头文件定义且每个 TU 共享同一地址。字符串 `"kvclient"` 必须与 [distributed_storage_config.h:33](../Mooncake/mooncake-store/include/config/distributed_storage_config.h#L33) 的比较值**逐字一致**——已 grep 核对两处一致。
- **L18-L19** — 防止有人误在 SDK-less TU 里 include 本头。
- **L20** — `final`：不打算被继承，允许编译器去虚化调用。

### 5.3 构造函数与析构函数

```cpp
    explicit KvClientObjectStorageAdapter(KvClientAdapterConfig config = {});  // L22（HEAD:21）
    // Test seam: inject an SDK client instead of creating one in Init().      // L23（HEAD:22）
    KvClientObjectStorageAdapter(KvClientAdapterConfig config,                 // L24（HEAD:23）
                                 std::unique_ptr<kv::KvClient> client);        // L25（HEAD:24）
    ~KvClientObjectStorageAdapter() override;                                  // L26（HEAD:25）
```

- **L22** — 生产构造。`explicit` 防止隐式转换；默认实参 `{}` 让它可零参构造（测试便利）。
- **L23-L25** — 测试注入构造函数：预置 `kv_client_`，使 `Init()` 走 [adapter .cpp:46-49](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L46) 的短路分支，从而无需真实 SDK 服务即可测试适配器逻辑。
- **L26** — 析构必须在 `.cpp` 里定义（要调 `Shutdown()`），所以这里只声明。基类析构是 virtual（[object_storage_adapter.h:46](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L46)），经基类指针删除时会正确调到本析构。

### 5.4 接口覆写

```cpp
    tl::expected<void, ErrorCode> Put(const std::string& logical_key,          // L28（HEAD:27）
                                      std::span<const char> data) override;    // L29（HEAD:28）
    tl::expected<void, ErrorCode> PutV(const std::string& logical_key,         // L30（HEAD:29）
                                       const iovec* iov, int iovcnt) override; // L31（HEAD:30）
    std::vector<tl::expected<void, ErrorCode>> PutBatch(                       // L32（HEAD:31）
        const std::vector<ObjectPutRequest>& requests) override;               // L33（HEAD:32）
    tl::expected<size_t, ErrorCode> Get(const std::string& logical_key,        // L34（HEAD:33）
                                        void* buf, size_t len) override;       // L35（HEAD:34）
    std::vector<tl::expected<size_t, ErrorCode>> GetBatch(                     // L36（HEAD:35）
        const std::vector<ObjectGetRequest>& requests) override;               // L37（HEAD:36）
    tl::expected<bool, ErrorCode> Exists(                                      // L38（HEAD:37）
        const std::string& logical_key) override;                              // L39（HEAD:38）
    tl::expected<void, ErrorCode> Delete(                                      // L40（HEAD:39）
        const std::string& logical_key) override;                              // L41（HEAD:40）
    ObjectStorageIoResults DeleteBatch(                                        // L42（HEAD:41）
        std::span<const std::string> logical_keys) override;                   // L43（HEAD:42）
    tl::expected<std::vector<KeyInfo>, ErrorCode> ListKeys() override;          // L44（HEAD:43）
                                                                               // L45
    tl::expected<void, ErrorCode> Init() override;                             // L46（HEAD:45）
    const char* GetName() const override { return kKvClientAdapterName; }      // L47（HEAD:46）
```

**L28-L44 是基类 [object_storage_adapter.h:48-111](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L48) 的全部纯虚函数。** 漏掉任何一个，类就成为抽象类，`make_unique` 直接编译失败——这是 PR1 开发过程中实际踩到并修复的缺陷。逐一对照基类签名：

| 覆写 | 基类声明 | 签名一致性 |
| --- | --- | --- |
| `Put(string, span<const char>)` | [L48-L49](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L48) | ✓ |
| `PutV(string, const iovec*, int)` | [L53-L55](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L53) | ✓ |
| `PutBatch(vector<ObjectPutRequest>)` | [L73-L74](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L73) | ✓ |
| `Get(string, void*, size_t) → size_t` | [L57-L58](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L57) | ✓ |
| `GetBatch(vector<ObjectGetRequest>)` | [L84-L85](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L84) | ✓ |
| `Exists(string) → bool` | [L95-L96](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L95) | ✓ |
| `Delete(string)` | [L98-L99](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L98) | ✓ |
| `DeleteBatch(span<const string>) → ObjectStorageIoResults` | [L101-L102](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L101) | ✓ |
| `ListKeys() → vector<KeyInfo>` | [L111](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L111) | ✓ |
| `Init()` | [L113](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L113) | ✓ |
| `GetName() const → const char*` | [L123](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L123) | ✓ |

**有意不覆写的虚函数**（基类都带默认实现，这正是基类注释所说的 "source compatible" 设计）：

- `GetRange`（[L63-L66](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L63)）→ 默认 `NOT_SUPPORTED`
- `CheckHealth`（[L120-L122](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L120)）→ 默认 `NOT_SUPPORTED`
- `SupportsSinglePutAttempt` / `SupportsSinglePutVAttempt`（[L130-L131](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L130)）→ 默认 `false`
- `PutOnce` / `PutVOnce`（[L133-L148](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L133)）→ 默认因 `Supports*` 为 false 而返回 `NOT_SUPPORTED`

- **L47** — `GetName()` 内联返回常量。`const char*` + `const override` 与基类 [L123](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L123) 完全一致。调用点在 [distributed_storage_backend.cpp:261](../Mooncake/mooncake-store/src/storage/distributed/distributed_storage_backend.cpp#L261) 和 [:268](../Mooncake/mooncake-store/src/storage/distributed/distributed_storage_backend.cpp#L268) 的日志里。

### 5.5 私有部分

```cpp
   private:                                                            // L49（HEAD:48）
    struct Completion {                                                // L50（HEAD:49）
        bool terminal = false;                                         // L51（HEAD:50）
        std::vector<kv::Status> entryStatus;                           // L52（HEAD:51）
    };                                                                 // L53（HEAD:52）
                                                                       // L54
    kv::BinaryKey MakeCacheKey(std::string_view logical_key) const;     // L55（HEAD:54）
    Completion waitForCompletion(kv::TaskId task_id, size_t entry_count);  // L56（HEAD:55）
    static ErrorCode ToErrorCode(const kv::Status& status);            // L57（HEAD:56）
    // Store entries share the SDK's entry decoder with loads, so the codes
    // that mean "object missing" on a read mean "store failed" on a write.  // L58-L59
    static ErrorCode ToWriteErrorCode(const kv::Status& status);        // L60
    // Registers one host range with the SDK; the returned handle stays valid
    // until it is passed to UnregisterBuffers.                        // L61-L62
    tl::expected<uint64_t, ErrorCode> RegisterBuffers(void* ptr, size_t size);  // L63
    // Releases the handles, logging any failure the SDK cannot retry.  // L64
    void UnregisterBuffers(const std::vector<uint64_t>& handles);       // L65
                                                                       // L66
    KvClientAdapterConfig config_;                                     // L67（HEAD:58）
    std::unique_ptr<kv::KvClient> kv_client_;                          // L68（HEAD:59）
    bool initialized_ = false;                                         // L69（HEAD:60）
};                                                                     // L70（HEAD:61）
```

- **L50-L53** — `waitForCompletion` 的返回载体。私有嵌套结构，不外泄到接口。
  - **L51** `terminal`：任务是否已终态。**PR1 中只写不读**（详见 §18.3），为 PR2 的 buffer / handle 生命周期管理预留。声明处初始化，两个构造函数都不会留下未初始化值。
  - **L52** `entryStatus`：每条 entry 的 SDK 状态，长度必须等于提交条目数——`waitForCompletion` 用 `assign` 强制保证这一点。
- **L55** — 逻辑键 → 8 字节 SDK 键。`const` 因不修改实例状态。返回 `kv::BinaryKey`（SDK `kv_types.h:43` = `std::array<std::byte, 8>`）。
- **L56** — **非 const**：调用 SDK 的 `Wait`，而 `Wait` 在 SDK 侧是非 const 方法（`kv_client.h:60`，它会在内部 `Remove` 任务）。
- **L57** — `static`：纯映射函数，不依赖实例状态。
- **L58-L65** — **【PR2，不属本 PR】** `ToWriteErrorCode`、`RegisterBuffers`、`UnregisterBuffers` 三个声明由未提交的 PR2 改动引入。
- **L67-L69** — 三个数据成员：
  - `config_` 按值持有配置副本，不依赖调用方生命周期。
  - `kv_client_` 用 `unique_ptr` 独占，析构时自动释放。
  - `initialized_` 声明处初始化。
  - 成员声明顺序与两个构造函数的初始化列表顺序一致，无 `-Wreorder` 警告。

---

## 6. 适配器实现 — 头部与内部工具

文件：[mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp)（PR1 为 290 行）

### 6.1 include 区

```cpp
#include "storage/distributed/kvclient/kvclient_object_storage_adapter.h"  // L1
                                                                      // L2
#include <array>                                                      // L3
#include <cstddef>                                                    // L4
#include <cstdint>                                                    // L5
#include <limits>                                                     // L6
#include <memory>                                                     // L7
#include <optional>                                                   // L8
#include <set>                                                        // L9
#include <string_view>                                                // L10
#include <utility>                                                    // L11
#include <vector>                                                     // L12
                                                                      // L13
#include <glog/logging.h>                                             // L14
#include <xxhash.h>                                                   // L15
                                                                      // L16
#include "storage/distributed/kvclient/kvclient_adapter_factory.h"     // L17（HEAD:14）
```

- **L1** — 自身头置于最前，确保它能独立编译（自包含性检查）。
- **L3** — `std::array<std::string, 1>`（`Delete` 里用）。
- **L4** — `std::byte`（`MakeCacheKey` 的字节打包）。
- **L5** — `uint64_t`。
- **L6-L9** — **【PR2 新增】** `<limits>` / `<optional>` / `<set>` 由 PR2 的 `PutBatch` 引入。PR1 只需 `<array>` / `<cstddef>` / `<cstdint>` / `<memory>` / `<string_view>` / `<utility>` / `<vector>`（HEAD:3-9）。
- **L11** — `std::move`。
- **L12** — `std::vector`。
- **L14** — `LOG(INFO)` / `LOG(ERROR)` / `LOG(WARNING)`。
- **L15** — `XXH64`。
- **L17** — 工厂头。**工厂函数的定义必须与声明同时可见**，否则返回类型不完整。

### 6.2 匿名命名空间

```cpp
namespace mooncake {                                                  // L19（HEAD:16）
namespace {                                                           // L20（HEAD:17）
                                                                      // L21
inline constexpr uint64_t kKvClientHashSeed = 5381;                    // L22（HEAD:19）
                                                                      // L23
}  // namespace                                                        // L24（HEAD:30）
```

- **L22** — 哈希种子。**5381 不是随意选的**：SDK `key_resolver.h:31` 定义 `#define HASH_SEED_VALUE (5381)`，`key_resolver.cc:51` 用它调 `XXH64`。适配器必须用同一个种子，否则算出的 key 与 SDK 内部解析的 key 不一致 → **PR2 写入的对象 PR1 删不掉**。已逐字节比对确认一致（详见 §17.1）。
- **L20 / L24** — 匿名命名空间给内部链接，避免与其他 TU 的同名符号冲突。

**PR1 在此处还有一个 helper（HEAD:21-28），工作区已被 PR2 移除：**

```cpp
template <typename T>                                                 // HEAD:21
std::vector<tl::expected<T, ErrorCode>> Errors(size_t count, ErrorCode error) {  // HEAD:22
    std::vector<tl::expected<T, ErrorCode>> results;                   // HEAD:23
    results.reserve(count);                                           // HEAD:24
    for (size_t i = 0; i < count; ++i)                                // HEAD:25
        results.emplace_back(tl::make_unexpected(error));              // HEAD:26
    return results;                                                   // HEAD:27
}                                                                     // HEAD:28
```

- 模板是因为 `PutBatch` 要 `vector<expected<void,...>>` 而 `GetBatch` 要 `vector<expected<size_t,...>>`。
- 作用是**批量填充正确数量的错误**：即使操作不支持，也必须满足基类 [L70-L72](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L70) 的契约"Return exactly one result per request, in the same order"，否则调用方会因数量不符判 `INTERNAL_ERROR`。
- `reserve` + `emplace_back` 避免反复扩容与拷贝。

---

## 7. 适配器实现 — 构造与析构

```cpp
KvClientObjectStorageAdapter::KvClientObjectStorageAdapter(            // L26（HEAD:32）
    KvClientAdapterConfig config)                                      // L27（HEAD:33）
    : config_(std::move(config)) {}                                    // L28（HEAD:34）
                                                                      // L29
KvClientObjectStorageAdapter::KvClientObjectStorageAdapter(            // L30（HEAD:36）
    KvClientAdapterConfig config, std::unique_ptr<kv::KvClient> client) // L31（HEAD:37）
    : config_(std::move(config)), kv_client_(std::move(client)) {}      // L32（HEAD:38）
                                                                      // L33
KvClientObjectStorageAdapter::~KvClientObjectStorageAdapter() {         // L34（HEAD:40）
    // No separate Shutdown() method: stop accepting work and let the SDK drain
    // pending tasks.                                                  // L35-L36（HEAD:41-42）
    if (kv_client_) kv_client_->Shutdown();                            // L37（HEAD:43）
}                                                                     // L38（HEAD:44）
```

- **L28** — 生产构造。`std::move` 避免拷贝 `config_path` 字符串。函数体为空：所有真实工作推迟到 `Init()`，因为构造函数无法返回错误。
- **L32** — 测试注入构造。预置 `kv_client_` 让 `Init()` 短路。初始化列表顺序（`config_` 然后 `kv_client_`）与成员声明顺序（[L67-L68](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h#L58-L59)）一致。
- **L35-L36** — 注释解释为何没有独立的 `Shutdown()`：基类 `ObjectStorageAdapter` 没有定义它，适配器无法在接口层暴露生命周期钩子，只能在析构里收尾。
- **L37** — 空指针检查**必要**：`Init()` 失败时 `kv_client_` 保持 null（见 §8），析构仍会执行。SDK 的 `Shutdown()`（`kv_client_impl.cc:186+`）会取 `defaultWaitTimeoutMs` 并 drain 在途任务，所以在析构里调用是安全的。

---

## 8. 适配器实现 — `Init()`

[L40-L68](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L46-L74)（HEAD:46-74）

```cpp
tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::Init() {    // L40（HEAD:46）
    if (initialized_) return {};                                       // L41（HEAD:47）
    if (!config_.Validate())                                           // L42（HEAD:48）
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);         // L43（HEAD:49）
                                                                      // L44
    // Injected client (test seam): only the adapter config is validated.  // L45（HEAD:51）
    if (kv_client_) {                                                  // L46（HEAD:52）
        initialized_ = true;                                           // L47（HEAD:53）
        return {};                                                     // L48（HEAD:54）
    }                                                                  // L49（HEAD:55）
                                                                      // L50
    auto client = kv::CreateKvClient();                                // L51（HEAD:57）
    if (!client)                                                       // L52（HEAD:58）
        return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE); // L53（HEAD:59）
                                                                      // L54
    kv::KvClientConfig cfg;                                            // L55（HEAD:61）
    if (!kv::KvClientConfig::LoadFromFile(config_.config_path, cfg).ok())  // L56（HEAD:62）
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);         // L57（HEAD:63）
    cfg.keyResolution = {};  // Force GENERIC.                         // L58（HEAD:64）
    // Init is not idempotent; the local client is only committed to the member
    // after this succeeds, so a retry starts from a fresh client.      // L59-L60（HEAD:65-66）
    if (!client->Init(cfg).ok())                                       // L61（HEAD:67）
        return tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE); // L62（HEAD:68）
                                                                      // L63
    kv_client_ = std::move(client);                                    // L64（HEAD:70）
    initialized_ = true;                                               // L65（HEAD:71）
    LOG(INFO) << "KVClient adapter initialized";                       // L66（HEAD:72）
    return {};                                                         // L67（HEAD:73）
}                                                                     // L68（HEAD:74）
```

- **L41** — 幂等保护：已初始化直接成功返回。`{}` 构造 `expected<void, ErrorCode>` 的成功值。调用方 [distributed_storage_backend.cpp:255](../Mooncake/mooncake-store/src/storage/distributed/distributed_storage_backend.cpp#L255) 会检查返回值。
- **L42-L43** — 先校验配置再创建任何资源。此时 `kv_client_` 仍为 null、`initialized_` 仍为 false → **重试安全**。
- **L45-L49** — 测试注入路径：客户端已存在，跳过 `CreateKvClient` / `LoadFromFile` / `Init`。注意仍要求 `config_.Validate()` 通过（L42），避免测试绕过配置检查。
- **L51** — SDK 工厂函数，`kv_client.h:72` 声明 `std::unique_ptr<KvClient> CreateKvClient();`。
- **L52-L53** — 创建失败 → `DFS_SERVICE_UNAVAILABLE`（服务不可用，而非参数错误）。
- **L51 用局部变量 `client` 而非直接写 `kv_client_`，是本函数最关键的设计。** 这是修复过的缺陷：原先直接赋值给成员，导致后续步骤失败时 `kv_client_` 已被污染，而 SDK 的 `Init` **不是幂等的**（`kv_client_impl.cc:177` 无条件设 `initialized_ = true`），重试会撞上"半初始化的客户端"从而假成功。
- **L55** — 默认构造 `KvClientConfig`。SDK `kv_types.h:248-265` 的默认值：`maxInflightTasks=1024`、`defaultWaitTimeoutMs=100`、`timeoutMs=100`、`sharedProviderMode=INDEPENDENT`。
- **L56-L57** — 从文件加载，SDK `kv_types.h:250` 声明 `static Status LoadFromFile(const std::string&, KvClientConfig&)`。这是出参式 API，失败时 `cfg` 可能部分填充，所以直接返回、不继续使用。配置文件错误 → `INVALID_PARAMS`。
- **L58** — 值初始化 `KeyResolutionConfig`（SDK `kv_types.h:252`）。其默认成员是 `InferenceEngine::GENERIC`、`layerCount=0`、`layerSlotBytes=0`。**为何必须显式复位**：
  - `LoadFromFile` 会从配置文件解析 `keyResolution.engine`（SDK `config_utils.cc:189-193` 支持 `VLLM_ASCEND` / `SGLANG` / `GENERIC`）。
  - 若配置文件设成 `VLLM_ASCEND`，SDK 会走 `ResolveVllmLayerKey`（`key_resolver.cc:66-99`），把 `@chunk_hash@layer` 后缀解析成分层偏移——那会破坏 Mooncake 的逻辑键语义。
  - 复位保证适配器**自己算的 `BinaryKey` 与 SDK 解析结果一致**：GENERIC 路径下 `TryGetBinaryKey()` 命中，`key_resolver.cc:41-44` 直接原样返回输入。
  - SDK `kv_client_impl.cc:123-124` 还会校验：非 `VLLM_ASCEND` 时 `layerCount` / `layerSlotBytes` 必须为 0，否则 `Init` 失败。`= {}` 同时满足这一点。
- **L59-L60** — 注释记录非幂等约束与应对方式。
- **L61-L62** — SDK `Init`（`kv_client.h:39`）失败 → 服务不可用。此时 `client` 是**局部变量**，函数返回即析构，成员状态完全未变，所以调用方重试会从 `CreateKvClient` 重新开始。
- **L64-L65** — 两步提交：先转移所有权，再置标志。顺序保证不会留下"已初始化但无客户端"的中间态。
- **L66** — 日志不再打印 `max_object_bytes`（该字段已按 @ayaka836 的要求删除）。

---

## 9. 适配器实现 — `ToErrorCode()`

[L70-L112](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L76-L118)（HEAD:76-118）

```cpp
ErrorCode KvClientObjectStorageAdapter::ToErrorCode(const kv::Status& status) {  // L70（HEAD:76）
    using kv::StatusCode;                                            // L71（HEAD:77）
    switch (status.code) {                                           // L72（HEAD:78）
        case StatusCode::OK:                                         // L73（HEAD:79）
            return ErrorCode::OK;                                    // L74（HEAD:80）
        case StatusCode::TIMEOUT:                                    // L75（HEAD:81）
        case StatusCode::CQE_IO_TIMEOUT:                             // L76（HEAD:82）
            return ErrorCode::RPC_TIMEOUT;                           // L77（HEAD:83）
        case StatusCode::IN_PROGRESS:                                // L78（HEAD:84）
        case StatusCode::CANCELED:                                   // L79（HEAD:85）
            return ErrorCode::INTERNAL_ERROR;                        // L80（HEAD:86）
```

- **L71** — `using` 缩短后续每个 case 的书写。
- **L73-L74** — 成功直通。
- **L75-L77** — 两种超时：任务级 `TIMEOUT` 和 CQE 级 `CQE_IO_TIMEOUT`（SDK `kv_types.h`，值 `0x10000|0x716`）。都映射到 `RPC_TIMEOUT`，让上层能区分"超时"与"服务挂了"。
- **L78-L80** — 非终态却走到了错误映射，说明调用方在非终态下读取了结果——这是适配器的内部逻辑错误，不是外部服务问题，故 `INTERNAL_ERROR`。实际上 `waitForCompletion` 在非终态分支直接返回，不会调到这里，属**防御性映射**。

```cpp
        // Object misses: only entry-level key-not-found is a miss.   // L82（HEAD:88）
        case StatusCode::CQE_KEY_NOT_EXIST:                          // L83（HEAD:89）
        case StatusCode::ENTRY_KEY_NOT_FOUND:                        // L84（HEAD:90）
        case StatusCode::ENTRY_KEY_NOT_EXIST:                        // L85（HEAD:91）
            return ErrorCode::OBJECT_NOT_FOUND;                      // L86（HEAD:92）
```

- **L83-L86** — 三种"键不存在"，来源各不同（已在 SDK 源码逐一核对）：

| 状态码 | 值 | SDK 产生位置 | 适用 op |
| --- | --- | --- | --- |
| `CQE_KEY_NOT_EXIST` | `0x10000\|0x701` | `trans_task_utils.cc:121-122` | **所有**（含 DELETE） |
| `ENTRY_KEY_NOT_FOUND` | `0x0100\|0x03` | `trans_task_utils.cc:67-68` | LOAD / STORE |
| `ENTRY_KEY_NOT_EXIST` | `0x0300\|0x00` | `trans_task_utils.cc:89-90` | 仅 QUERY |

都映射为 `OBJECT_NOT_FOUND`，让上层能识别"对象确实不在"。

```cpp
        // NOT_FOUND is a routing/config/view error, not a missing object.
        // Mapping it to OBJECT_NOT_FOUND would let PR3 delete metadata and
        // PR2 evict live metadata on a routing failure.              // L88-L90（HEAD:94-96）
        case StatusCode::NOT_FOUND:                                  // L91（HEAD:97）
        case StatusCode::NOT_INITIALIZED:                            // L92（HEAD:98）
        case StatusCode::CONNECTION_ERROR:                           // L93（HEAD:99）
        case StatusCode::IO_ERROR:                                   // L94（HEAD:100）
        case StatusCode::UNSUPPORTED:                                // L95（HEAD:101）
        // Delete entry failures use ENTRY_DELETE_FAILED; ENTRY_KEY_NOT_EXIST
        // is produced only for QUERY.                               // L96-L97（HEAD:102-103）
        case StatusCode::ENTRY_DELETE_FAILED:                        // L98（HEAD:104）
            return ErrorCode::DFS_SERVICE_UNAVAILABLE;               // L99（HEAD:105）
```

- **L88-L90** — **本函数最重要的语义决策。** SDK 的任务级 `NOT_FOUND` 表示路由 / 视图 / 配置找不到目标，**不是**"对象不存在"。若误映射为 `OBJECT_NOT_FOUND`，上层会认为对象真的没了，从而：PR2 驱逐仍然有效的元数据、PR3 删除 Master 元数据 —— 造成**数据丢失**。所以映射到 `DFS_SERVICE_UNAVAILABLE`（可重试的服务错误）。
- **L92-L95** — 其余服务级错误同样归为"服务不可用"，让上层重试而非认定数据丢失。
- **L96-L97** — 注释说明 DELETE 路径用 `ENTRY_DELETE_FAILED`（SDK `trans_task_utils.cc:79-80` 确认：`AsuOpType::DELETE` 分支只产生它），而 `ENTRY_KEY_NOT_EXIST` 只来自 QUERY（`trans_task_utils.cc:89-90`）——所以把 `ENTRY_DELETE_FAILED` 归为服务错误、把 `ENTRY_KEY_NOT_EXIST` 归为对象缺失，两者不冲突。
- **L98** — `ENTRY_DELETE_FAILED`（`0x0200|0x01`）：删除动作本身失败（不是键不存在）→ 服务不可用。

```cpp
        case StatusCode::INVALID_ARGUMENT:                           // L101（HEAD:107）
        case StatusCode::BUFFER_NOT_REGISTERED:                      // L102（HEAD:108）
        case StatusCode::BUFFER_NOT_SUPPORTED:                       // L103（HEAD:109）
            return ErrorCode::INVALID_PARAMS;                        // L104（HEAD:110）
        case StatusCode::CQE_OUT_OF_CREATE_SIZE:                     // L105（HEAD:111）
        case StatusCode::RESOURCE_BUSY:                              // L106（HEAD:112）
        case StatusCode::CQE_RESOURCE_BUSY:                          // L107（HEAD:113）
            return ErrorCode::NO_AVAILABLE_HANDLE;                   // L108（HEAD:114）
        default:                                                     // L109（HEAD:115）
            return ErrorCode::INTERNAL_ERROR;                        // L110（HEAD:116）
    }                                                                // L111（HEAD:117）
}                                                                    // L112（HEAD:118）
```

- **L101-L104** — 调用方错误（参数非法、buffer 未注册 / 不支持）→ `INVALID_PARAMS`，不应重试。`BUFFER_*` 两项在 PR1 的 Delete 路径不会出现（Delete 不需要注册 buffer），是给 PR2 的 Put / Get 预留的正确映射。
- **L105-L108** — 资源耗尽类：超出创建大小上限、资源忙。映射到 `NO_AVAILABLE_HANDLE`，上层可退避重试。
- **L109-L110** — 兜底。**覆盖性核对**：SDK `StatusCode` 共 24 个枚举值，此 switch 显式覆盖 19 个，其余落入 default → `INTERNAL_ERROR`：

  `PARTIAL_FAILED`、`TASK_NOT_FOUND`、`INTERNAL_ERROR`、`ENTRY_RETRY_ADVISED`、`ENTRY_NO_RETRY_ADVISED`、`ENTRY_DATA_NOT_EXIST`、`ENTRY_KEY_EXIST`、`CQE_INVALID_COMMAND_OPCODE`、`CQE_INVALID_FIELD_IN_COMMAND`、`CQE_INTERNAL_ERROR`、`CQE_WRITE_FAULT`、`CQE_UNRECOVERED_READ_ERROR`、`CQE_KEY_ALREADY_EXISTED`、`CQE_CHECK_RESULT_BUFFER`

  已逐一核对：**没有把重要错误码误落 default 的情况**。`PARTIAL_FAILED` / `TASK_NOT_FOUND` 由 `waitForCompletion` 在调 `ToErrorCode` 之前就分流掉了。

---

## 10. 适配器实现 — `waitForCompletion()`

[L114-L170](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L120-L176)（HEAD:120-176）

这是 @ayaka836 两条规则（超时归 SDK、`entryStatus` 读取限制）的核心实现。

### 10.1 等待与双层状态解包

```cpp
KvClientObjectStorageAdapter::Completion                              // L114（HEAD:120）
KvClientObjectStorageAdapter::waitForCompletion(kv::TaskId task_id,    // L115（HEAD:121）
                                                size_t entry_count) {  // L116（HEAD:122）
    kv::TaskResult result;                                            // L117（HEAD:123）
    // 0 lets the SDK use its configured defaultWaitTimeoutMs; the adapter does
    // not impose a second timeout. The SDK reports TIMEOUT via the result.  // L118-L119（HEAD:124-125）
    const auto wait_status = kv_client_->Wait(task_id, 0, result);     // L120（HEAD:126）
    const auto code = wait_status.ok() ? result.status.code : wait_status.code;  // L121（HEAD:127）
    const kv::Status& task_status =                                   // L122（HEAD:128）
        wait_status.ok() ? result.status : wait_status;               // L123（HEAD:129）
```

- **L117** — SDK `kv_types.h:137-141`：`{Status status; vector<Status> entryStatus; optional<QueryResult> queryResult;}`。默认构造时 `status.code` 为 `OK`。
- **L118-L119** — 注释记录超时归属决策。
- **L120** — `timeoutMs=0` 的含义已在 SDK 源码验证：
  - `kv_client_impl.cc:294`：`const auto waitMs = timeoutMs == 0 ? config_.defaultWaitTimeoutMs : timeoutMs;`
  - `kv_client.h:59` 注释：*"Waits for the result and consumes the task even on timeout; **zero uses the default timeout**."*

  **超时策略完全归 SDK 配置文件所有**，适配器不再有 `MOONCAKE_KVCLIENT_WAIT_TIMEOUT_MS`，也不做 `timeoutMs < wait_timeout_ms` 的交叉校验（两者均已删除）。

  SDK `Wait` 的另一个关键行为：`task_manager.cc:113-116` 在 `WaitContext` 之后**无条件** `Remove(taskId)`，即使超时。这是下面 `TASK_NOT_FOUND` 分支的依据。

- **L121** — **双层状态的正确取值**：`Wait` 的返回值是**等待层**状态（如 `TASK_NOT_FOUND`、等待本身失败），`result.status` 是**任务层**状态（如 `PARTIAL_FAILED`、`TIMEOUT`）。等待层成功时才看任务层。
  - 验证：SDK `kv_client_impl.cc:304` 对 `TASK_NOT_FOUND` 会 `return status;`（提前返回，不走视图刷新），所以这种情况 `wait_status.ok()` 为假，`code` 取 `wait_status.code` = `TASK_NOT_FOUND` ✓
- **L122-L123** — 绑定到实际生效的那个 `Status`，供后续 `assign` 复用。用 const 引用避免拷贝 `message` 字符串。

### 10.2 终态白名单

```cpp
    Completion completion;                                            // L125（HEAD:131）
    switch (code) {                                                   // L126（HEAD:132）
        case kv::StatusCode::OK:                                      // L127（HEAD:133）
        case kv::StatusCode::PARTIAL_FAILED:                          // L128（HEAD:134）
            // Only these task statuses carry meaningful per-entry results.  // L129（HEAD:135）
            completion.terminal = true;                               // L130（HEAD:136）
            break;                                                    // L131（HEAD:137）
```

- **L125** — `terminal` 默认 false（[adapter .h:51](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h#L50)），`entryStatus` 默认空。
- **L127-L131** — **@ayaka836 规则的白名单**：只有这两个任务级状态携带可信的逐条结果。`break` 而非 `return`，落到 L161 的条目数校验。
- `PARTIAL_FAILED` 表示部分条目失败——此时 `entryStatus` 才是权威的逐条结果来源。

### 10.3 非终态分支

```cpp
        case kv::StatusCode::TASK_NOT_FOUND:                          // L132（HEAD:138）
            // Wait removes the task even after a timeout, so an unknown task
            // does not prove that its I/O has finished. Keep it non-terminal
            // so callers retain handles and staging buffers.          // L133-L135（HEAD:139-141）
            completion.terminal = false;                              // L136（HEAD:142）
            completion.entryStatus.assign(                            // L137（HEAD:143）
                entry_count,                                          // L138（HEAD:144）
                kv::Status::Error(kv::StatusCode::INTERNAL_ERROR,      // L139（HEAD:145）
                                  "task not found"));                  // L140（HEAD:146）
            return completion;                                        // L141（HEAD:147）
```

- **L133-L135** — 注释的依据是 SDK `task_manager.cc:113-116`：`Wait` 无论成功或超时都 `Remove(taskId)`。所以"任务查不到"有两种可能：(i) 从未提交成功；(ii) 之前某次等待超时后被移除了，但**底层 IO 可能仍在进行**。
- **L136** — 因此判为**非终态**：调用方必须保留 buffer / handle（PR2 场景），不能假定内存已不再被 SDK 使用。这是防止 use-after-free 的关键判断。
- **L137-L140** — 仍填满 `entry_count` 个条目，保证 `DeleteBatch` 的索引映射不会越界。用 `INTERNAL_ERROR` 而非 `OBJECT_NOT_FOUND`——因为不知道对象是否被删掉了。
- **L141** — 直接返回，不读 `result.entryStatus`（此时它必然为空或不可信）。

```cpp
        case kv::StatusCode::TIMEOUT:                                 // L142（HEAD:148）
        case kv::StatusCode::IN_PROGRESS:                             // L143（HEAD:149）
        case kv::StatusCode::CANCELED:                                // L144（HEAD:150）
            // The SDK may have populated entryStatus before returning the
            // timeout, but the task is not terminal; per-entry results are not
            // trustworthy. Fill with the task-level status and retain
            // resources.                                             // L145-L148（HEAD:151-154）
            completion.terminal = false;                              // L149（HEAD:155）
            completion.entryStatus.assign(entry_count, task_status);   // L150（HEAD:156）
            return completion;                                        // L151（HEAD:157）
```

- **L142-L144** — 三种非终态。`TIMEOUT` 来自 SDK 等待超时；`IN_PROGRESS` 任务仍在跑；`CANCELED` 被取消但资源可能未回收。
- **L145-L148** — 注释点明陷阱：SDK 可能在返回超时**之前**已部分填充 `entryStatus`，但那些值不代表最终结果（IO 可能稍后成功）。所以**不读** `result.entryStatus`。
- **L149** — 非终态 → 调用方保留资源。
- **L150** — 用任务级状态填满所有条目，让每条都报同一个错误（`ToErrorCode(TIMEOUT)` → `RPC_TIMEOUT`），语义清晰且索引安全。

### 10.4 终态错误分支

```cpp
        default:                                                      // L152（HEAD:158）
            // A final task-level error. entryStatus may still hold submit-time
            // OK values, so propagate the overall error to every entry instead
            // of reading it.                                         // L153-L155（HEAD:159-161）
            completion.terminal = true;                               // L156（HEAD:162）
            completion.entryStatus.assign(entry_count, task_status);   // L157（HEAD:163）
            return completion;                                        // L158（HEAD:164）
    }                                                                 // L159（HEAD:165）
```

- **L152-L158** — **@ayaka836 规则的另一半，也是提交 `95f88152` 的核心修改。** 走到 default 的是任务级**终态错误**（`NOT_FOUND`、`CONNECTION_ERROR`、`IO_ERROR`、`NOT_INITIALIZED`、`INVALID_ARGUMENT` 等）。
- **L153-L155** — 注释解释了为什么不能读 `entryStatus`：SDK 在提交时会把 `entryStatus` 预填为 `OK`（`trans_task_utils.cc:145-148` 的 `std::fill(entryStatus.begin(), entryStatus.end(), ... Status::OK())`）。若任务级失败后仍读它，会把**整体失败误判为逐条成功** → 假成功。这正是修改前的缺陷。
- **L156** — 终态：IO 已结束，资源可释放。
- **L157** — 整体错误传播到每个已提交条目。

### 10.5 条目数校验与结果移交

```cpp
    if (result.entryStatus.size() != entry_count) {                   // L161（HEAD:167）
        completion.entryStatus.assign(                                // L162（HEAD:168）
            entry_count,                                              // L163（HEAD:169）
            kv::Status::Error(kv::StatusCode::INTERNAL_ERROR,          // L164（HEAD:170）
                              "entry count mismatch"));                // L165（HEAD:171）
        return completion;                                            // L166（HEAD:172）
    }                                                                 // L167（HEAD:173）
    completion.entryStatus = std::move(result.entryStatus);            // L168（HEAD:174）
    return completion;                                                // L169（HEAD:175）
}                                                                     // L170（HEAD:176）
```

- **L161-L166** — 只有 `OK` / `PARTIAL_FAILED` 会走到这里。SDK 返回的条目数与提交数不符 → 契约违背，用 `INTERNAL_ERROR` 填满，避免 L260 的 `entryStatus[j]` 越界。
- **L161** — 注意此分支**未显式设置 `terminal`**，沿用 L130 设的 `true`（因为只有 `OK` / `PARTIAL_FAILED` 能 `break` 到这里）。逻辑正确，但依赖控制流——这是 `terminal` 死字段的连带问题（§18.3）。
- **L168** — `std::move` 避免拷贝整个 `vector<Status>`（每个 `Status` 含一个 `std::string message`）。热路径上的实质优化。

---

## 11. 适配器实现 — `MakeCacheKey()`

[L172-L185](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L178-L191)（HEAD:178-191）

```cpp
kv::BinaryKey KvClientObjectStorageAdapter::MakeCacheKey(              // L172（HEAD:178）
    std::string_view logical_key) const {                              // L173（HEAD:179）
    uint64_t hash =                                                   // L174（HEAD:180）
        XXH64(logical_key.data(), logical_key.size(), kKvClientHashSeed);  // L175（HEAD:181）
    // The protocol rejects an all-zero key; map it to the seed to keep the
    // wire key non-zero without a second constant.                    // L176-L177（HEAD:182-183）
    if (hash == 0) hash = kKvClientHashSeed;                           // L178（HEAD:184）
    kv::BinaryKey key{};                                              // L179（HEAD:185）
    // Pack explicitly little-endian like the SDK's key_resolver; a memcpy of
    // the host-order uint64_t would differ on big-endian clients.      // L180-L181（HEAD:186-187）
    for (size_t i = 0; i < sizeof(hash); ++i)                          // L182（HEAD:188）
        key[i] = static_cast<std::byte>(hash >> (8 * i));              // L183（HEAD:189）
    return key;                                                       // L184（HEAD:190）
}                                                                     // L185（HEAD:191）
```

- **L173** — `string_view` 而非 `const std::string&`：允许传入任意连续字节（Mooncake 的逻辑键可含任意字节，见基类 [L39-L42](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L39) 的契约注释），且 `DeleteBatch` 的 `keys[i]` 可隐式转换。
- **L174-L175** — `XXH64`，种子 5381。**与 SDK 逐字节一致**：`key_resolver.cc:51`
  ```cpp
  auto hash = XXH64(identity.data(), identity.size(), HASH_SEED_VALUE);
  ```
  这个一致性是 PR1 Delete 能命中 PR2 写入对象的前提。
- **L176-L178** — 全零 key 守卫。依据是 SDK `kv_protocol.cc:116-120` 的 `IsCacheKeyAllZero`：
  ```cpp
  static bool IsCacheKeyAllZero(const CacheKey& key) {
      return std::all_of(CacheKeyView(key).begin(), CacheKeyView(key).end(),
                         [](char value) { return value == '\0'; });
  }
  ```
  在 `kv_protocol.cc` 的 **:219（Store）、:362（Retrieve）、:526（Batch）、:719（Batch）** 四处拒绝全零键，返回 `INVALID_ARGUMENT "key is all zeros"`。`XXH64` 有 2⁻⁶⁴ 概率返回 0；不加守卫，那个逻辑键将永远无法写入 / 删除。

  **映射到 5381 而非引入第二个常量**，避免多一个需要维护的魔数——注释明确说明了这个取舍。代价是 `hash==0` 的键会与"哈希值恰为 5381"的键碰撞，但后者概率同为 2⁻⁶⁴，实际影响可忽略。
- **L179** — `{}` 值初始化 8 字节数组。`kv::BinaryKey` = `std::array<std::byte, 8>`（SDK `kv_types.h:42-43`）。
- **L180-L183** — 与 SDK `key_resolver.cc:52-55` 对齐：
  ```cpp
  for (auto& byte : result.key) { byte = static_cast<std::byte>(hash & 0xff); hash >>= 8; }
  ```
  SDK 用"取低字节后右移"，适配器用"右移后取低字节"，两者产生**相同的小端字节序**。
- **为何不用 `memcpy(&key, &hash, 8)`** — `memcpy` 保留主机字节序，在大端机器上会与 SDK 的小端打包不一致，导致跨端点键不匹配。显式移位保证与平台无关。
- **`static_cast<std::byte>` 必需** — `std::byte` 不接受隐式整数转换。

**历史**：原先这是独立的 `EncodeBinaryKey` 类 / 函数，按用户要求折叠进 `MakeCacheKey`，codec 头文件已删除。

---

## 12. 适配器实现 — `Delete()`

[L215-L222](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L193-L200)（HEAD:193-200）

```cpp
tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::Delete(     // L215（HEAD:193）
    const std::string& key) {                                          // L216（HEAD:194）
    std::array<std::string, 1> keys{key};                              // L217（HEAD:195）
    auto results = DeleteBatch(keys);                                  // L218（HEAD:196）
    return results.empty()                                             // L219（HEAD:197）
               ? tl::make_unexpected(ErrorCode::INVALID_PARAMS)        // L220（HEAD:198）
               : std::move(results.front());                           // L221（HEAD:199）
}                                                                     // L222（HEAD:200）
```

- **L217** — 栈上 1 元素数组。`std::span<const std::string>` 能从 `std::array<std::string, 1>` 隐式构造（span 有接受 contiguous range 的构造函数），**零堆分配**。
- **L218** — **单键删除复用批量路径**，避免两套逻辑不一致。这是 @ayaka836 指出的缺陷：原先 `Delete` 与 `DeleteBatch` 行为不同。
- **L219-L221** — `empty()` 在 `keys.size()==1` 时恒为假（`DeleteBatch` 两条返回路径都产出 1 个元素），所以是**防御性死分支**。保留它与姊妹适配器 `KvcsObjectStorageAdapter::Delete`（[kvcs_object_storage_adapter.cpp:221-227](../Mooncake/mooncake-store/src/storage/distributed/kvcs/kvcs_object_storage_adapter.cpp#L221)）写法一致。
- **L221** — `std::move(results.front())` 避免拷贝 `expected`。

---

## 13. 适配器实现 — `DeleteBatch()`

[L224-L266](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L202-L244)（HEAD:202-244）

### 13.1 前置守卫

```cpp
ObjectStorageIoResults KvClientObjectStorageAdapter::DeleteBatch(       // L224（HEAD:202）
    std::span<const std::string> keys) {                               // L225（HEAD:203）
    if (!initialized_ || !kv_client_) {                                // L226（HEAD:204）
        return ObjectStorageIoResults(                                 // L227（HEAD:205）
            keys.size(),                                               // L228（HEAD:206）
            tl::make_unexpected(ErrorCode::DFS_SERVICE_UNAVAILABLE));   // L229（HEAD:207）
    }                                                                  // L230（HEAD:208）
    ObjectStorageIoResults results(keys.size());                       // L231（HEAD:209）
    if (keys.empty()) return results;                                  // L232（HEAD:210）
```

- **L226** — **双重守卫**：既查标志也查指针。修复的是空指针解引用缺陷——`Init()` 失败时 `initialized_` 为 false 且 `kv_client_` 为 null；若只查 `initialized_`，注入构造（`kv_client_` 已设但未 `Init`）的路径会有歧义。两者都查最稳妥。
- **L227-L229** — `ObjectStorageIoResults` 是 `std::vector<tl::expected<void, ErrorCode>>` 的别名（[object_storage_adapter.h:33](../Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h#L33)）。这个 `(size_t, value)` 构造函数是 `std::vector` 的 fill 构造，产出 `keys.size()` 个相同错误。
  - **返回正确数量至关重要**：调用方 [distributed_storage_backend.cpp:1132](../Mooncake/mooncake-store/src/storage/distributed/distributed_storage_backend.cpp#L1132) 直接透传结果，而 [client_service.cpp:3951-3955](../Mooncake/mooncake-store/src/client_service.cpp#L3951) 会检查 `deleted.size() != removed_keys.size()`，数量不符就把整批判为 `INTERNAL_ERROR`。
- **L231** — 值初始化 → 每个元素是 `expected<void, ErrorCode>{}`，即**成功**。所以后续必须为每个下标显式赋值，否则会漏报失败。
- **L232** — 空输入早返回，避免向 SDK 提交空批次。

### 13.2 键编码与索引压缩

```cpp
    std::vector<kv::CacheKey> cache_keys;                              // L234（HEAD:212）
    std::vector<size_t> indices;                                       // L235（HEAD:213）
    cache_keys.reserve(keys.size());                                   // L236（HEAD:214）
    indices.reserve(keys.size());                                      // L237（HEAD:215）
    for (size_t i = 0; i < keys.size(); ++i) {                         // L238（HEAD:216）
        if (keys[i].empty()) {                                         // L239（HEAD:217）
            results[i] = tl::make_unexpected(ErrorCode::INVALID_PARAMS); // L240（HEAD:218）
            continue;                                                  // L241（HEAD:219）
        }                                                              // L242（HEAD:220）
        cache_keys.push_back(kv::CacheKey::FromBytes(MakeCacheKey(keys[i])));  // L243（HEAD:221）
        indices.push_back(i);                                          // L244（HEAD:222）
    }                                                                  // L245（HEAD:223）
    if (cache_keys.empty()) return results;                            // L246（HEAD:224）
```

- **L234** — 提交给 SDK 的键数组。SDK `DeleteAsync` 签名（`kv_client.h:55-56`）：
  ```cpp
  virtual Status DeleteAsync(const std::vector<CacheKey>& keys, TaskId& taskId,
                             const IoOptions& options = {}) = 0;
  ```
- **L235** — **压缩索引映射**：`cache_keys[j]` 对应 `keys[indices[j]]`。需要它是因为空键会被跳过，导致两个数组长度不等。
- **L236-L237** — 预留容量，避免循环中反复扩容（最坏情况无空键时正好用满）。
- **L239-L241** — 空逻辑键就地报 `INVALID_PARAMS`，不提交给 SDK。与 kvcs 的 `EncodeKey`（[kvcs_object_storage_adapter.cpp:59-60](../Mooncake/mooncake-store/src/storage/distributed/kvcs/kvcs_object_storage_adapter.cpp#L59)）对空键的处理一致。
- **L243** — `MakeCacheKey` 算出 8 字节键，`CacheKey::FromBytes` 包装成 SDK 的 `CacheKey`（`kv_types.h:49`）：
  ```cpp
  static CacheKey FromBytes(BinaryKey key) { return CacheKey(key); }
  ```
  **必须用这个工厂**：`CacheKey` 的构造函数是 **private** 的（`kv_types.h:59-60`），内部是 `std::variant<BinaryKey, std::string>`（`kv_types.h:62`），所以 `FromBytes` 是构造二进制键的唯一合法途径。
- **L246** — 全是空键时早返回，此时 `results` 已全部填为 `INVALID_PARAMS` ✓。

### 13.3 提交与提交失败处理

```cpp
    kv::TaskId task_id = kv::kInvalidTaskId;                           // L248（HEAD:226）
    const auto submit = kv_client_->DeleteAsync(cache_keys, task_id);   // L249（HEAD:227）
    if (!submit.ok()) {                                                // L250（HEAD:228）
        // Submission failed before the SDK accepted the task; propagate the
        // overall error to every submitted entry.                     // L251-L252（HEAD:229-230）
        for (size_t i : indices)                                       // L253（HEAD:231）
            results[i] = tl::make_unexpected(ToErrorCode(submit));     // L254（HEAD:232）
        return results;                                                // L255（HEAD:233）
    }                                                                  // L256（HEAD:234）
```

- **L248** — `kInvalidTaskId = 0`（SDK `kv_types.h:68`）。`DeleteAsync` 通过引用出参写入真实 task id；预置无效值让"SDK 未写入"可被检测。
- **L249** — 异步提交，立即返回。
- **L250-L255** — **提交失败的处理**（@ayaka836 规则的补充）：任务从未被 SDK 接受，所以不存在逐条结果。
  - 遍历 **`indices`**（只覆盖**已提交**的条目），统一填入提交错误。
  - 这个区分很关键：若遍历 `0..keys.size()`，会把空键的 `INVALID_PARAMS` 覆盖成提交错误，**丢失更精确的诊断**。

### 13.4 等待与结果反映射

```cpp
    const auto completion = waitForCompletion(task_id, cache_keys.size());  // L258（HEAD:236）
    for (size_t j = 0; j < indices.size(); ++j) {                      // L259（HEAD:237）
        results[indices[j]] = completion.entryStatus[j].ok()           // L260（HEAD:238）
                                  ? tl::expected<void, ErrorCode>{}    // L261（HEAD:239）
                                  : tl::make_unexpected(               // L262（HEAD:240）
                                        ToErrorCode(completion.entryStatus[j]));  // L263（HEAD:241）
    }                                                                  // L264（HEAD:242）
    return results;                                                    // L265（HEAD:243）
}                                                                     // L266（HEAD:244）
```

- **L258** — `entry_count` 传 **`cache_keys.size()`**（已提交数，非 `keys.size()`）。这与 `waitForCompletion` 内部所有 `assign(entry_count, ...)` 一致，保证 `entryStatus.size() == indices.size()`。
- **L259-L263** — **索引反映射**：`entryStatus[j]` → `results[indices[j]]`。因 L161 / L137 / L150 / L157 都用 `entry_count` 填充，`entryStatus[j]` 对所有 `j < indices.size()` 必然有效，**无越界**。
- **L260-L261** — 成功的条目写 `expected<void, ErrorCode>{}`（成功值）。
- **L262-L263** — 失败的经 `ToErrorCode` 转换。
- **L265** — 返回时 `results` 的**每个下标都已被显式赋值**：空键 → L240；提交失败 → L254；正常 → L260 / L263。**没有依赖 L231 的默认成功值**，这是正确性的关键。

---

## 14. 适配器实现 — `NOT_SUPPORTED` 桩与工厂

### 14.1 PR1 的桩实现

**PR1 提交版（HEAD:246-281）**中，`Put` / `PutV` / `PutBatch` / `Get` / `GetBatch` / `Exists` / `ListKeys` 全部返回 `NOT_SUPPORTED`：

```cpp
tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::Put(        // HEAD:246
    const std::string&, std::span<const char>) {                       // HEAD:247
    return tl::make_unexpected(ErrorCode::NOT_SUPPORTED);              // HEAD:248
}                                                                     // HEAD:249
                                                                      // HEAD:250
tl::expected<void, ErrorCode> KvClientObjectStorageAdapter::PutV(       // HEAD:251
    const std::string&, const iovec*, int) {                           // HEAD:252
    return tl::make_unexpected(ErrorCode::NOT_SUPPORTED);              // HEAD:253
}                                                                     // HEAD:254
                                                                      // HEAD:255
std::vector<tl::expected<void, ErrorCode>>                             // HEAD:256
KvClientObjectStorageAdapter::PutBatch(                                // HEAD:257
    const std::vector<ObjectPutRequest>& requests) {                   // HEAD:258
    return Errors<void>(requests.size(), ErrorCode::NOT_SUPPORTED);     // HEAD:259
}                                                                     // HEAD:260
                                                                      // HEAD:261
tl::expected<size_t, ErrorCode> KvClientObjectStorageAdapter::Get(      // HEAD:262
    const std::string&, void*, size_t) {                               // HEAD:263
    return tl::make_unexpected(ErrorCode::NOT_SUPPORTED);              // HEAD:264
}                                                                     // HEAD:265
                                                                      // HEAD:266
std::vector<tl::expected<size_t, ErrorCode>>                           // HEAD:267
KvClientObjectStorageAdapter::GetBatch(                                // HEAD:268
    const std::vector<ObjectGetRequest>& requests) {                   // HEAD:269
    return Errors<size_t>(requests.size(), ErrorCode::NOT_SUPPORTED);   // HEAD:270
}                                                                     // HEAD:271
                                                                      // HEAD:272
tl::expected<bool, ErrorCode> KvClientObjectStorageAdapter::Exists(     // HEAD:273
    const std::string&) {                                              // HEAD:274
    return tl::make_unexpected(ErrorCode::NOT_SUPPORTED);              // HEAD:275
}                                                                     // HEAD:276
                                                                      // HEAD:277
tl::expected<std::vector<KeyInfo>, ErrorCode>                          // HEAD:278
KvClientObjectStorageAdapter::ListKeys() {                             // HEAD:279
    return tl::make_unexpected(ErrorCode::NOT_SUPPORTED);              // HEAD:280
}                                                                     // HEAD:281
```

- 形参**匿名**（只留类型）：明确表示不使用，同时避免 `-Wunused-parameter`。
- `PutBatch` / `GetBatch` 用 §6.2 的 `Errors<T>` helper 填充**正确数量**的错误。
- 这些桩让 PR1 的适配器满足基类契约却不做数据面工作，PR2 逐个替换。

### 14.2 工作区当前状态

**【PR2，不属本 PR】** 工作区的以下区域已被未提交的 PR2 改动替换，不再是 `NOT_SUPPORTED` 桩：

| 函数 | 工作区行号 | 状态 |
| --- | --- | --- |
| `Put` | [L268-L272](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L268-L272) | PR2 改为转发到 `PutV` |
| `PutV` | [L274-L279](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L274-L279) | PR2 改为转发到 `PutBatch` |
| `PutBatch` | [L281-L392](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L281-L392) | PR2 完整实现（含 buffer 注册） |
| `GetBatch` | [L399-L407](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L399-L407) | PR2 展开填充（仍是 `NOT_SUPPORTED`，但不再用 helper） |
| `RegisterBuffers` | [L187-L200](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L187-L200) | PR2 新增 |
| `UnregisterBuffers` | [L202-L213](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L202-L213) | PR2 新增 |

**仍保持 PR1 原样**的桩：

```cpp
tl::expected<size_t, ErrorCode> KvClientObjectStorageAdapter::Get(      // L394
    const std::string&, void*, size_t) {                               // L395
    return tl::make_unexpected(ErrorCode::NOT_SUPPORTED);              // L396
}                                                                     // L397
                                                                      // L398
...
tl::expected<bool, ErrorCode> KvClientObjectStorageAdapter::Exists(     // L409
    const std::string&) {                                              // L410
    return tl::make_unexpected(ErrorCode::NOT_SUPPORTED);              // L411
}                                                                     // L412
                                                                      // L413
tl::expected<std::vector<KeyInfo>, ErrorCode>                          // L414
KvClientObjectStorageAdapter::ListKeys() {                             // L415
    return tl::make_unexpected(ErrorCode::NOT_SUPPORTED);              // L416
}                                                                     // L417
```

### 14.3 工厂函数

[L419-L424](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L283-L288)（HEAD:283-288）

```cpp
tl::expected<std::unique_ptr<ObjectStorageAdapter>, ErrorCode>          // L419（HEAD:283）
CreateKvClientObjectStorageAdapter() {                                  // L420（HEAD:284）
    auto config = KvClientAdapterConfig::FromEnvironment();             // L421（HEAD:285）
    if (!config) return tl::make_unexpected(config.error());            // L422（HEAD:286）
    return std::make_unique<KvClientObjectStorageAdapter>(*config);     // L423（HEAD:287）
}                                                                     // L424（HEAD:288）
                                                                      // L425
}  // namespace mooncake                                              // L426（HEAD:290）
```

- **L419** — 返回类型与 [kvclient_adapter_factory.h:13-14](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_adapter_factory.h#L13) 的声明**逐字一致**。返回**基类** `unique_ptr` 让调用方（`storage_backend.cpp`）无需 SDK 头。这是修过的缺陷：原先返回派生类指针，导致 `storage_backend.cpp` 必须 include SDK 头。
- **L421** — 从环境读配置。定义放在 `.cpp` 而非头文件，所以只有 SDK-enabled TU 会实例化它。
- **L422** — 传播配置错误（如 `MOONCAKE_KVCLIENT_CONFIG` 未设 → `INVALID_PARAMS`），不吞掉。
- **L423** — `*config` 解引用 `expected` 取值，`make_unique` 构造。只调**生产构造函数**——`Init()` 由 [distributed_storage_backend.cpp:255](../Mooncake/mooncake-store/src/storage/distributed/distributed_storage_backend.cpp#L255) 稍后调用。

---

## 15. 配置层

### 15.1 后端类型识别

文件：[mooncake-store/include/config/distributed_storage_config.h](../Mooncake/mooncake-store/include/config/distributed_storage_config.h#L27)

```cpp
    bool UsesObjectStorage() const {                                   // L27
        return fs_adapter_type == "oss" || fs_adapter_type == "s3" ||  // L28
               fs_adapter_type == "kvcs-lowlevel" ||                   // L29
               fs_adapter_type == "kvclient";                          // L30
    }                                                                  // L31
    bool UsesKvcs() const { return fs_adapter_type == "kvcs-lowlevel"; }  // L32
    bool UsesKvClient() const { return fs_adapter_type == "kvclient"; }   // L33
```

- **L30** — 把 kvclient 加入"对象存储"家族。**这一处改动自动让 kvclient 继承所有 `UsesObjectStorage()` 的既有行为**：跳过文件系统分配器、走 object adapter 的读写路径、被 `ValidateForAllocator` 拒绝。
- **L32-L33** — 两个平行谓词，供需要区分具体厂商的地方使用。`kvclient` 与 `kvcs-lowlevel` 是**互斥**的两个字符串，所以两个谓词不会同时为真。
- 相关默认值：`fsdir = "/mnt/3fs/mooncake"`（[L10](../Mooncake/mooncake-store/include/config/distributed_storage_config.h#L10)）、`fs_adapter_type = "hf3fs"`（[L11](../Mooncake/mooncake-store/include/config/distributed_storage_config.h#L11)）、`single_tenant = true`（[L19](../Mooncake/mooncake-store/include/config/distributed_storage_config.h#L19)）。

### 15.2 校验分支

文件：[mooncake-store/src/config/distributed_storage_config.cpp](../Mooncake/mooncake-store/src/config/distributed_storage_config.cpp#L78)

```cpp
    if (UsesKvClient()) {                                              // L78
        if (!single_tenant) {                                          // L79
            // Client::Put/BatchPut -> WriteDfsReplicas -> DfsWriteRequest ->
            // ObjectPutRequest passes the raw user key; only the offload/read
            // RPC paths use TenantId::MakeScopedKey. The SDK's IoOptions.kvns
            // is not consumed in this version, so two tenants sharing a key
            // would collide in the KV namespace.                       // L80-L84
            LOG(ERROR) << "DistributedStorageConfig: KVClient requires "
                          "single_tenant=true";                        // L85-L86
            return false;                                              // L87
        }                                                              // L88
        // KVClient has no capacity API, so eviction_check_interval is not
        // required.                                                   // L89-L90
        return true;                                                   // L91
    }                                                                  // L92
```

- **L78** — 分支位置在 `UsesKvcs()`（[L64-L77](../Mooncake/mooncake-store/src/config/distributed_storage_config.cpp#L64)）**之后**、`fsdir` 检查（[L93](../Mooncake/mooncake-store/src/config/distributed_storage_config.cpp#L93)+）**之前**。**顺序重要**：必须先于文件系统路径检查返回，否则 kvclient 会被 `fsdir` 为空 / 非绝对路径的校验误拒。测试 `KvClientSkipsFilesystemRootChecks` 覆盖了这一点。
- **L79-L87** — **单租户强制**。注释给出的是实地核查过的调用链证据：
  - 同步写路径 `Client::Put/BatchPut` → `WriteDfsReplicas` → `DfsWriteRequest` → `ObjectPutRequest` 传的是**原始用户键**，未经租户前缀化
  - 只有 offload / read 的 RPC 路径用 `TenantId::MakeScopedKey`（在 [file_storage.cpp:1016](../Mooncake/mooncake-store/src/file_storage.cpp#L1016) 看到 `TenantId::ParseScopedKey` 的反向操作，印证这条路径确实带前缀）
  - SDK 的 `IoOptions.kvns`（`kv_types.h:120-122`）本可提供命名空间隔离，但当前版本**未被消费**（已 grep 确认适配器没有设置 `IoOptions`）
  - 结论：多租户下两个租户的同名键会在 KV 命名空间**直接碰撞** → 必须禁止
- **L85-L86** — 错误信息指明具体违反的约束，便于运维定位。
- **L89-L91** — 提前返回 `true`，跳过 `fsdir` / `shard_count` / `shard_capacity` / `alignment` / `eviction_check_interval` 等文件系统专属校验。理由：kvclient 无容量查询 API，所以驱逐检查间隔无意义。
- `single_tenant` 默认 `true`，所以默认配置下 kvclient 直接可用，只有显式设成 false 才报错。

**`ValidateForAllocator()` 无需改动**（已有实现）：

```cpp
bool DistributedStorageConfig::ValidateForAllocator() const {
    if (!Validate()) return false;
    if (UsesObjectStorage()) {
        LOG(ERROR) << "DistributedStorageConfig: DFS allocator requires a "
                      "filesystem adapter";
        return false;
    }
    return ValidateEvictionSettings(*this);
}
```

因为 `UsesObjectStorage()` 已含 kvclient（L30），allocator 校验自动拒绝它。测试 `KvClientIsNotAValidAllocatorBackend` 锁定这一行为。

### 15.3 后端级谓词

文件：[mooncake-store/include/storage/distributed/distributed_storage_backend.h](../Mooncake/mooncake-store/include/storage/distributed/distributed_storage_backend.h#L78)

```cpp
    bool UsesKvClient() const {                                        // L78
        return UsesObjectStorage() && distributed_config_.UsesKvClient();  // L79
    }                                                                  // L80
```

- **L79** — 双重条件：既要是对象存储，又要具体是 kvclient。与紧邻的 `UsesKvcs()`（[L74-L76](../Mooncake/mooncake-store/include/storage/distributed/distributed_storage_backend.h#L74)）结构完全对称。
- 外层 `UsesObjectStorage()` 看似冗余（config 的 `UsesKvClient()` 已蕴含它），但它保证谓词在 backend 层的语义自洽：只有真的走了 object adapter 路径才为真。

---

## 16. 接线层

### 16.1 跳过本地目录校验

文件：[mooncake-store/src/config/file_storage_config.cpp](../Mooncake/mooncake-store/src/config/file_storage_config.cpp#L238)

```cpp
bool FileStorageConfig::Validate() const {                              // L232
    // storage_filepath may be a comma-separated multi-disk list. ValidatePath
    // stats a single path and would reject the raw list string (stat of
    // "/d0,/d1" is ENOENT), so validate each root on its own. Remote object
    // backends resolve their paths through the distributed backend, so keep
    // skipping the local-directory checks there.                       // L233-L237
    bool uses_remote_object_storage = false;                            // L238
    if (storage_backend_type == StorageBackendType::kDistributed) {      // L239
        const auto distributed_config =                                 // L240
            DistributedStorageConfig::FromEnvironment();                // L241
        uses_remote_object_storage = distributed_config.UsesKvcs() ||    // L242
                                     distributed_config.UsesKvClient();  // L243
    }                                                                   // L244
    if (!uses_remote_object_storage) {                                  // L245
```

- **L235-L237** — 注释相应地从"Distributed storage on kvcs resolves its paths..."改写为"**Remote object backends** resolve their paths..."，因为现在覆盖两个后端。
- **L238** — 初值 false：非分布式后端一律走本地目录校验。
- **L239-L241** — 只在分布式后端下才解析分布式配置，避免无谓的环境变量读取。
- **L242-L243** — 把 kvclient 加入"远程对象存储"集合。**原先是单表达式** `const bool uses_kvcs = ... && ...UsesKvcs();`，现在拆成可变局部变量 + if 块，因为要**只解析一次** `FromEnvironment()` 再检查两个谓词（原先的写法若直接写成 `UsesKvcs() || UsesKvClient()` 会解析两次）。
- **L245** — 远程对象存储的路径由分布式后端自己解析，所以跳过 `storage_filepath` 的本地目录存在性检查。

### 16.2 启用 DFS 路径

文件：[mooncake-store/src/file_storage.cpp](../Mooncake/mooncake-store/src/file_storage.cpp#L162)

```cpp
        config_.enable_dfs = !distributed_backend->UsesObjectStorage() ||  // L162
                             distributed_backend->UsesKvcs() ||           // L163
                             distributed_backend->UsesKvClient();         // L164
        if (config_.enable_dfs && client_) {                             // L165
            client_->SetDfsStorageBackend(distributed_backend);           // L166
        }                                                                // L167
```

- **L162** — 非对象存储（即真正的文件系统 DFS）→ 启用。
- **L163-L164** — 对象存储中，kvcs 和 kvclient 需要启用 DFS 路径（它们通过 `ObjectStorageAdapter` 走 DFS 语义），而 oss / s3 不需要。
- **L165-L166** — 启用后把后端注入 client。**这三行是 kvclient 能被 client 侧看到的关键**：没有它，`dfs_storage_backend_` 为 null，PR2 / PR3 的所有数据面调用都无从发起。

### 16.3 客户端启动接线

文件：[mooncake-store/src/real_client.cpp](../Mooncake/mooncake-store/src/real_client.cpp#L1618)

```cpp
    const auto distributed_config = DistributedStorageConfig::FromEnvironment();  // L1616
    const bool enable_kvcs = distributed_config.UsesKvcs();               // L1617
    const bool enable_kvclient = distributed_config.UsesKvClient();        // L1618
    if (enable_ssd_offload || enable_kvcs || enable_kvclient) {            // L1619
        auto file_storage_config = FileStorageConfig::FromEnvironment();   // L1620
        if (enable_kvcs || enable_kvclient) {                              // L1621
            file_storage_config.storage_backend_type =                     // L1622
                StorageBackendType::kDistributed;                          // L1623
        }                                                                  // L1624
```

- **L1618** — 与 L1617 的 `enable_kvcs` 完全平行，同样从已解析的 `distributed_config` 取值（**不重复解析**环境变量）。
- **L1619** — 加入 `||` 链：kvclient 也需要构造 `FileStorage`（它承载 DFS 后端）。
- **L1621-L1623** — **强制后端类型为分布式**。这一步不可省：`FileStorageConfig::FromEnvironment()` 读出的默认类型不是 `kDistributed`，若不强制，`CreateStorageBackend` 会走本地文件后端，kvclient 适配器根本不会被创建。
- 与 §16.1 的 [file_storage_config.cpp:239](../Mooncake/mooncake-store/src/config/file_storage_config.cpp#L239) 形成闭环：那里也检查 `storage_backend_type == kDistributed` 才跳过目录校验，所以两处必须一致地设置 / 读取这个字段。

### 16.4 创建适配器

文件：[mooncake-store/src/storage_backend.cpp](../Mooncake/mooncake-store/src/storage_backend.cpp#L77)

```cpp
#ifdef MOONCAKE_HAVE_KVCLIENT_SDK                                        // L77
#include "storage/distributed/kvclient/kvclient_adapter_factory.h"        // L78
#endif                                                                   // L79
```

- **L77-L79** — 条件 include。只 include **SDK-free 的工厂头**（§4），不含适配器头——这是那个文件存在的全部理由。
- 宏由 [src/CMakeLists.txt:763-764](../Mooncake/mooncake-store/src/CMakeLists.txt#L763) 在 `mooncake_store_client_objects` 上定义，而 `storage_backend.cpp` 正在该 target 的源列表里（[CMakeLists.txt:119](../Mooncake/mooncake-store/src/CMakeLists.txt#L119)），所以宏必然可见。

```cpp
            } else if (distributed_config.UsesKvcs()) {                   // L6518
                object_storage_adapter =                                 // L6519
                    std::make_unique<KvcsObjectStorageAdapter>(           // L6520
                        distributed_config.object_storage_config_path);   // L6521
            } else if (distributed_config.UsesKvClient()) {               // L6522
#ifdef MOONCAKE_HAVE_KVCLIENT_SDK                                        // L6523
                auto adapter = CreateKvClientObjectStorageAdapter();       // L6524
                if (!adapter)                                            // L6525
                    return tl::make_unexpected(adapter.error());          // L6526
                object_storage_adapter = std::move(*adapter);             // L6527
#else                                                                    // L6528
                LOG(ERROR) << "KVClient SDK not built in; kvclient backend "
                              "unavailable";                             // L6529-L6530
                return tl::make_unexpected(ErrorCode::NOT_SUPPORTED);      // L6531
#endif                                                                   // L6532
            } else {                                                     // L6533
                return tl::make_unexpected(ErrorCode::INVALID_PARAMS);     // L6534
            }                                                            // L6535
```

- **L6522** — 分支紧跟 `UsesKvcs()` 分支（L6518-L6521）之后、`else` 兜底之前。顺序无关紧要（谓词互斥），但放在一起便于对照。
- **L6524** — 调工厂，返回 `expected<unique_ptr<ObjectStorageAdapter>>`。
- **L6525-L6526** — 传播配置错误（如 `MOONCAKE_KVCLIENT_CONFIG` 未设 → `INVALID_PARAMS`）。
- **L6527** — `std::move(*adapter)` 解引用 `expected` 并移动 `unique_ptr` 到 `object_storage_adapter`（类型是 `std::unique_ptr<ObjectStorageAdapter>`）。**类型精确匹配**是修过的缺陷——工厂返回基类指针，赋值目标也是基类指针，无需转换。
- **L6528-L6531** — SDK 缺失时的**显式失败**：记日志 + 返回 `NOT_SUPPORTED`。这比静默降级好得多——用户配了 `fs_adapter_type=kvclient` 却拿到一个"看起来正常但什么都做不了"的后端，是最难排查的故障。测试 `KvClientFactoryReturnsNotSupportedWithoutSdk` 覆盖了这条路径。
- 之后 `object_storage_adapter` 会被包进 `DistributedStorageBackend`（L6536+，既有代码），所以 PR1 不需要改包装逻辑。

**注意与 kvcs 分支的差异**：kvcs 用 `make_unique` 直接构造（L6520），kvclient 必须走工厂函数——因为 `storage_backend.cpp` 不能 include 适配器头（含 SDK 依赖）。

---

## 17. 构建系统

### 17.1 SDK 检测

文件：[mooncake-store/src/CMakeLists.txt](../Mooncake/mooncake-store/src/CMakeLists.txt#L223)

#### 变量与选项

```cmake
# KVClient is optional, but the SDK does not publish kvclient.pc. Detect it
# from pkg-config when a future package provides one, otherwise from an
# explicit SDK root or include/library hints. Set
# MOONCAKE_KVCLIENT_REQUIRED=ON to turn a missing SDK into a configure error
# instead of silently skipping the adapter and hiding compile errors.     # L223-L227
set(MOONCAKE_KVCLIENT_SDK_FOUND OFF)                                     # L228
# The SDK currently installs libraries but not headers; point this at the SDK
# source tree, or set MOONCAKE_KVCLIENT_INCLUDE_DIR explicitly.           # L229-L230
set(MOONCAKE_KVCLIENT_SDK_ROOT "" CACHE PATH                              # L231
    "KVClient SDK source tree or install prefix")                         # L232
option(MOONCAKE_KVCLIENT_REQUIRED                                         # L233
       "Fail configuration when the KVClient SDK is not found" OFF)       # L234
```

- **L223-L227** — 注释交代整块代码的动机：**SDK 不提供 `.pc` 文件**（已用 `find` 确认两个快照都没有），所以原先纯 pkg-config 的检测必然失败 → 适配器被静默跳过 → 编译错误被掩盖。这正是 @ayaka836 指出的问题，也是提交 `f3c8ffd5` 的全部内容。
- **L228** — 先置 OFF，后面各分支按需置 ON。
- **L231-L232** — `CACHE PATH` 让用户可用 `-DMOONCAKE_KVCLIENT_SDK_ROOT=...` 指定，并在 CMake GUI / ccmake 里可见。默认空串。
- **L233-L234** — `option` 创建布尔 cache 变量，默认 OFF（保持可选依赖的向后兼容）。
- **L229-L230 的注释有一处不准确**：它建议指向"SDK source tree"，但源码树的 out-of-source 构建产物路径不在 `PATH_SUFFIXES` 里，照做会 FATAL（详见 §19.2）。

#### pkg-config 路径

```cmake
find_package(PkgConfig QUIET)                                            # L236
if(PkgConfig_FOUND)                                                      # L237
  pkg_check_modules(KVCLIENT QUIET IMPORTED_TARGET GLOBAL "kvclient")     # L238
endif()                                                                  # L239
                                                                         # L240
if(TARGET PkgConfig::KVCLIENT)                                            # L241
  add_library(Mooncake::kvclient INTERFACE IMPORTED GLOBAL)               # L242
  set_target_properties(Mooncake::kvclient PROPERTIES                     # L243
                        INTERFACE_LINK_LIBRARIES PkgConfig::KVCLIENT)     # L244
  set(MOONCAKE_KVCLIENT_SDK_FOUND ON)                                     # L245
  message(STATUS "KVClient SDK found via pkg-config: kvclient backend enabled")  # L246
```

- **L236-L239** — 保留 pkg-config 路径作为**未来兼容**：若某天 SDK 开始发布 `kvclient.pc`，无需改 CMake。`QUIET` 避免找不到时刷警告。
- **L238** — `IMPORTED_TARGET` 生成 `PkgConfig::KVCLIENT`；`GLOBAL` 让该 target 在所有目录可见（`mooncake-store/tests/` 也需要）。
- **L241** — 用 `if(TARGET ...)` 而非检查变量，更健壮（`pkg_check_modules` 失败时不生成 target）。
- **L242-L244** — 包一层 `Mooncake::kvclient` INTERFACE target。**这层间接的价值**：下游代码只依赖统一名字 `Mooncake::kvclient`，不关心它背后是 pkg-config 还是手工 find，两条检测路径对消费者完全透明。`GLOBAL` 同样为了跨目录可见。
- **L245-L246** — 置位并报告。

#### 手工探测路径

```cmake
else()                                                                   # L247
  set(_kvclient_hints)                                                   # L248
  if(MOONCAKE_KVCLIENT_SDK_ROOT)                                          # L249
    list(APPEND _kvclient_hints "${MOONCAKE_KVCLIENT_SDK_ROOT}")          # L250
  endif()                                                                # L251
  if(DEFINED ENV{MOONCAKE_KVCLIENT_SDK_ROOT})                             # L252
    list(APPEND _kvclient_hints "$ENV{MOONCAKE_KVCLIENT_SDK_ROOT}")       # L253
  endif()                                                                # L254
```

- **L248** — 下划线前缀表示内部变量（CMake 惯例）。空列表初始化。
- **L249-L251** — cache 变量形式的 root 作为 hint。
- **L252-L254** — 环境变量形式的 root 也作为 hint。**但 L290 的 fail-loud 闸门只检查 L249 的 cache 变量**，这就是 §19.4 那条不一致的来源。

```cmake
  find_path(MOONCAKE_KVCLIENT_INCLUDE_DIR                                 # L255
            NAMES kv_client.h                                            # L256
            HINTS ${_kvclient_hints}                                     # L257
            PATH_SUFFIXES include kv_semantics/include include/kv_semantics)  # L258
  find_library(MOONCAKE_KVCLIENT_LIBRARY                                  # L259
               NAMES kv_client                                           # L260
               HINTS ${_kvclient_hints}                                  # L261
               PATH_SUFFIXES lib lib64 kv_semantics/src ucm/store/asu src
                             build/src build)                            # L262-L263
  find_library(MOONCAKE_KVCLIENT_TRANSPORT_LIBRARY                        # L264
               NAMES kv_transport                                        # L265
               HINTS ${_kvclient_hints}                                  # L266
               PATH_SUFFIXES lib lib64 kv_semantics/src ucm/store/asu src
                             build/src build)                            # L267-L268
```

- **L256** — 探测头文件名 `kv_client.h`。**仅按文件名匹配，不校验 API 变体** → §19.1 里"指向旧 API SDK 会 configure 绿灯、编译红灯"的根源。
- **L258** — 三个后缀覆盖：标准 install 前缀（`include`）、SDK 源码树布局（`kv_semantics/include`，已确认两个快照都是这个布局）、反向布局（`include/kv_semantics`）。
- **L260 / L265** — 库名不带 `lib` 前缀和 `.so` 后缀，`find_library` 会自动补全平台约定。
- **L262-L263 / L267-L268** — 库搜索后缀。`kv_semantics/src` 和 `ucm/store/asu` 精确对应 SDK 的 install 规则（`kv_semantics/src/CMakeLists.txt:141-142`：`LIBRARY DESTINATION ${INSTALL_REL_PATH}` 和 `ucm/store/asu`）。`build/src` 对应单独构建 `kv_semantics` 子项目的布局。**缺 `build/kv_semantics/src`**（顶层构建的产物路径）→ §19.2。
- `kv_transport` 单独查找是因为 SDK 把传输层拆成独立共享库（`kv_semantics/src/CMakeLists.txt:20`），`kv_client` 运行时依赖它。

```cmake
  if(MOONCAKE_KVCLIENT_INCLUDE_DIR AND MOONCAKE_KVCLIENT_LIBRARY)          # L269
    add_library(Mooncake::kvclient UNKNOWN IMPORTED GLOBAL)                # L270
    set(_kvclient_link_libraries "${MOONCAKE_KVCLIENT_LIBRARY}")           # L271
    if(MOONCAKE_KVCLIENT_TRANSPORT_LIBRARY)                                # L272
      list(APPEND _kvclient_link_libraries                                 # L273
           "${MOONCAKE_KVCLIENT_TRANSPORT_LIBRARY}")                       # L274
    endif()                                                                # L275
    set_target_properties(                                                 # L276
      Mooncake::kvclient                                                   # L277
      PROPERTIES IMPORTED_LOCATION "${MOONCAKE_KVCLIENT_LIBRARY}"          # L278
                 INTERFACE_INCLUDE_DIRECTORIES                             # L279
                 "${MOONCAKE_KVCLIENT_INCLUDE_DIR}"                        # L280
                 INTERFACE_LINK_LIBRARIES "${_kvclient_link_libraries}")    # L281
    set(MOONCAKE_KVCLIENT_SDK_FOUND ON)                                    # L282
    message(                                                               # L283
      STATUS "KVClient SDK found: include=${MOONCAKE_KVCLIENT_INCLUDE_DIR}" # L284
             ", library=${MOONCAKE_KVCLIENT_LIBRARY}")                      # L285
  endif()                                                                  # L286
endif()                                                                    # L287
```

- **L269** — **头和库都必须找到**才算 SDK 可用。只找到头（本机常态：源码树未构建）→ 保持 OFF。这个合取是正确的：缺库时链接必然失败，不如在 configure 阶段就判定不可用。
- **L270** — `UNKNOWN IMPORTED`：库类型未知（实际是 SHARED），CMake 会在链接时自行判断。`GLOBAL` 跨目录可见。
- **L271-L275** — 组装链接列表，`kv_transport` 可选（若 SDK 未来把传输层静态链进 `kv_client`，缺失也不致命）。
- **L278** — `IMPORTED_LOCATION` 指向实际 `.so`。
- **L279-L280** — `INTERFACE_INCLUDE_DIRECTORIES` 让所有链接 `Mooncake::kvclient` 的 target 自动获得该 include 路径——所以适配器 TU 不需要手工 `-I`。
- **L281** — 把两个库都挂上。
- **L284-L285** — 报告实际解析到的路径，便于排查"找到了错误的 SDK"。

#### fail-loud 闸门

```cmake
if(NOT MOONCAKE_KVCLIENT_SDK_FOUND)                                        # L289
  if(MOONCAKE_KVCLIENT_REQUIRED OR MOONCAKE_KVCLIENT_SDK_ROOT)              # L290
    message(                                                               # L291
      FATAL_ERROR                                                          # L292
        "KVClient SDK not found (MOONCAKE_KVCLIENT_REQUIRED=ON or "         # L293
        "MOONCAKE_KVCLIENT_SDK_ROOT was set). Set MOONCAKE_KVCLIENT_SDK_ROOT "  # L294
        "to a source tree or install prefix that contains kv_client.h and "  # L295
        "libkv_client, or set MOONCAKE_KVCLIENT_INCLUDE_DIR and "            # L296
        "MOONCAKE_KVCLIENT_LIBRARY.")                                       # L297
  endif()                                                                  # L298
  message(                                                                 # L299
    STATUS                                                                 # L300
      "KVClient SDK not found: adapter disabled. Set "                     # L301
      "MOONCAKE_KVCLIENT_SDK_ROOT or pass -DMOONCAKE_KVCLIENT_REQUIRED=ON " # L302
      "to make this fatal.")                                                # L303
endif()                                                                    # L304
set(MOONCAKE_KVCLIENT_SDK_FOUND                                            # L305
    ${MOONCAKE_KVCLIENT_SDK_FOUND}                                          # L306
    PARENT_SCOPE)                                                          # L307
```

- **L290** — **fail-loud 闸门**：显式要求（`REQUIRED=ON`）或显式指定了 root（说明用户**打算**用 kvclient）却找不到 → 中止 configure。**这是消除"静默跳过"的核心机制。** 缺陷：只测 cache 变量，不测 `DEFINED ENV{...}`（§19.4）。
- **L293-L297** — 错误信息给出三条可操作的修复路径。
- **L299-L303** — 未显式要求时降级为 STATUS，保持"可选依赖"语义。文案也告诉用户如何升级为致命。
- **L305-L307** — `PARENT_SCOPE` 把结果传给 `mooncake-store/` 目录作用域。因为 `add_subdirectory(src)` 在 `add_subdirectory(tests)` **之前**（[mooncake-store/CMakeLists.txt:63](../Mooncake/mooncake-store/CMakeLists.txt#L63) vs [:80](../Mooncake/mooncake-store/CMakeLists.txt#L80)），所以 [tests/CMakeLists.txt:402](../Mooncake/mooncake-store/tests/CMakeLists.txt#L402) 能读到正确值。三条路径（pkg-config 找到 / find 找到 / 未找到）都会执行到这里，**变量总是被设置** ✓

### 17.2 编译接线

#### 源文件门控

```cmake
# The adapter TU includes the KVClient SDK; only compile it when the SDK is
# present. Keep it out of MOONCAKE_STORE_SHARED_SOURCES: that object library
# also feeds mooncake_store_master, which must stay vendor-free.           # L730-L732
if(MOONCAKE_KVCLIENT_SDK_FOUND)                                             # L733
  list(APPEND MOONCAKE_STORE_CLIENT_SOURCES                                 # L734
       storage/distributed/kvclient/kvclient_object_storage_adapter.cpp)     # L735
endif()                                                                     # L736
```

- **L730-L732** — 注释解释了两个决策：为何门控（TU 含 SDK 头），以及为何放进 `CLIENT_SOURCES` 而非 `SHARED_SOURCES`（后者会喂给 `mooncake_store_master`，master 必须保持厂商中立）。
- **L734-L735** — 追加到 client 源列表，该列表在 [L739-L740](../Mooncake/mooncake-store/src/CMakeLists.txt#L739) 被 `mooncake_store_client_objects` 使用。与 `storage_backend.cpp`（[L119](../Mooncake/mooncake-store/src/CMakeLists.txt#L119)）**同一个 target** —— 这是宏定义能生效的前提。

#### 宏与链接

```cmake
if(MOONCAKE_KVCLIENT_SDK_FOUND)                                             # L760
  # storage_backend.cpp and the adapter TU are both compiled into this object
  # library, so the define must live here rather than on shared objects.     # L761-L762
  target_compile_definitions(mooncake_store_client_objects                   # L763
                             PRIVATE MOONCAKE_HAVE_KVCLIENT_SDK=1)           # L764
  target_link_libraries(mooncake_store_client_objects                       # L765
                        PRIVATE Mooncake::kvclient)                          # L766
endif()                                                                     # L767
```

- **L761-L762** — 注释记录了修过的缺陷：宏原先定义在 shared objects 上，但 `storage_backend.cpp` 不在那里编译，导致 `#ifdef` 分支永不生效。
- **L763-L764** — 定义宏。`PRIVATE` 足够——只有本 target 的源文件需要它。
- **L765-L766** — 链接 SDK。通过 `Mooncake::kvclient` 自动带来 include 目录（L279-L280 设的 `INTERFACE_INCLUDE_DIRECTORIES`）和两个库。
- **放在 L760 而非紧跟 L733** — 因为 `target_*` 命令必须在 `add_library`（L739）之后。

#### 最终库重链接

```cmake
if(MOONCAKE_KVCLIENT_SDK_FOUND)                                             # L881
  target_link_libraries(mooncake_store PRIVATE Mooncake::kvclient)           # L883
endif()
```

- **L881-L883** — 最终的 `mooncake_store` 库重新链接 SDK。object library 的 `PRIVATE` 链接不会自动传播到最终库，所以需要显式再挂一次，否则符号未定义。

#### 宏覆盖完整性核查

已验证 `MOONCAKE_HAVE_KVCLIENT_SDK` 覆盖所有需要它的 TU：

| 检查项 | 结果 |
| --- | --- |
| `storage_backend.cpp` 出现在几个源列表 | **仅 1 处**（[L119](../Mooncake/mooncake-store/src/CMakeLists.txt#L119)，`MOONCAKE_STORE_CLIENT_SOURCES`） |
| 该列表喂给几个 target | **仅 1 个**（[L739-L740](../Mooncake/mooncake-store/src/CMakeLists.txt#L739) `mooncake_store_client_objects`） |
| 宏定义在哪个 target | 同一个（[L763-L764](../Mooncake/mooncake-store/src/CMakeLists.txt#L763)） |
| 适配器源文件在哪个 target | 同一个（[L734-L735](../Mooncake/mooncake-store/src/CMakeLists.txt#L734)） |
| master 是否含适配器 | **否**（保持 vendor-free） |

→ **无 TU 漏掉宏**，`#ifdef` 分支必然生效。

---

## 18. 测试

### 18.1 测试宏门控

文件：[mooncake-store/tests/CMakeLists.txt](../Mooncake/mooncake-store/tests/CMakeLists.txt#L402)

```cmake
add_store_test(distributed_storage_config_test                            # L400
               distributed_storage_config_test.cpp)                        # L401
if(NOT MOONCAKE_KVCLIENT_SDK_FOUND)                                        # L402
  target_compile_definitions(distributed_storage_config_test                # L403
                             PRIVATE MOONCAKE_KVCLIENT_TEST_NO_SDK=1)       # L404
endif()                                                                    # L405
```

- **L402** — **取反**：只在 SDK **缺失**时定义宏。因为要测的正是"SDK 缺失时的降级行为"。
- **L403-L404** — 定义 `MOONCAKE_KVCLIENT_TEST_NO_SDK=1`，与测试文件的 `#ifdef`（[L207](../Mooncake/mooncake-store/tests/distributed_storage_config_test.cpp#L207)）**逐字匹配**。
- **L402 能读到 `MOONCAKE_KVCLIENT_SDK_FOUND`** — 依赖 [src/CMakeLists.txt:305-307](../Mooncake/mooncake-store/src/CMakeLists.txt#L305) 的 `PARENT_SCOPE` 和 `add_subdirectory` 顺序（src 在 tests 之前）。

### 18.2 测试用例

文件：[mooncake-store/tests/distributed_storage_config_test.cpp](../Mooncake/mooncake-store/tests/distributed_storage_config_test.cpp#L13)

#### 新增 include

```cpp
#include "config/distributed_storage_config.h"                              // L12
#include "storage_backend.h"                                                // L13
```

- **L13** — 为 L215 的 `CreateStorageBackend` 调用所需。

#### 默认配置断言

```cpp
    EXPECT_TRUE(config.object_storage_config_path.empty());                  // L81
    EXPECT_FALSE(config.UsesKvcs());                                        // L82
    EXPECT_FALSE(config.UsesKvClient());                                    // L83
    EXPECT_FALSE(config.UsesObjectStorage());                               // L84
}                                                                           // L85
```

- **L82-L84** — 在 `ExpectDefaultConfig` 里断言默认配置（`fs_adapter_type = "hf3fs"`）**不**触发任何对象存储谓词。**L84 尤其重要**：它锁定了 `UsesObjectStorage()` 不会因新增 kvclient 而意外对 hf3fs 返回 true。

#### 用例 1：后端类型识别

```cpp
TEST_F(DistributedStorageConfigTest,                                        // L194
       ExplicitKvClientAdapterSelectsObjectStorage) {                        // L195
    env.fs_adapter.Set("kvclient");                                         // L196
                                                                            // L197
    const auto config = DistributedStorageConfig::FromEnvironment();         // L198
                                                                            // L199
    EXPECT_EQ(config.fs_adapter_type, "kvclient");                           // L200
    EXPECT_TRUE(config.UsesObjectStorage());                                 // L201
    EXPECT_TRUE(config.UsesKvClient());                                      // L202
    EXPECT_FALSE(config.UsesKvcs());                                         // L203
    EXPECT_TRUE(config.Validate());                                          // L204
}                                                                           // L205
```

- **L196** — `env` 是 fixture 的环境变量守卫（RAII，测试结束自动恢复），避免污染其他测试。
- **L200-L203** — 验证字符串解析与两个谓词的**互斥性**。
- **L204** — `Validate()` 在默认 `single_tenant=true`、`fsdir` 为默认值的情况下返回 true —— 这正是 §15.2 提前返回分支的效果。

#### 用例 2：SDK 缺失降级

```cpp
#ifdef MOONCAKE_KVCLIENT_TEST_NO_SDK                                        // L207
TEST_F(DistributedStorageConfigTest,                                        // L208
       KvClientFactoryReturnsNotSupportedWithoutSdk) {                       // L209
    env.fs_adapter.Set("kvclient");                                         // L210
                                                                            // L211
    FileStorageConfig file_config;                                          // L212
    file_config.storage_backend_type = StorageBackendType::kDistributed;      // L213
                                                                            // L214
    const auto backend = CreateStorageBackend(file_config);                  // L215
                                                                            // L216
    ASSERT_FALSE(backend);                                                  // L217
    EXPECT_EQ(backend.error(), ErrorCode::NOT_SUPPORTED);                     // L218
}                                                                           // L219
#endif                                                                      // L220
```

- **L207 / L220** — `#ifdef` 与 §18.1 的宏精确对应。**只在 SDK 缺失的构建里编译**——SDK 存在时这条路径不存在，测试也就无意义。
- **L213** — 必须设 `kDistributed`，否则 `CreateStorageBackend` 不会走分布式分支。
- **L217** — 用 `ASSERT_`（而非 `EXPECT_`）：`backend` 为假是 L218 访问 `.error()` 的前提，失败就必须中止本测试，否则会崩。
- **L218** — 锁定 [storage_backend.cpp:6531](../Mooncake/mooncake-store/src/storage_backend.cpp#L6531) 的 `#else` 分支返回 `NOT_SUPPORTED`。
- **这个测试的价值**：它让"SDK 缺失"这条**平时被跳过**的路径在 CI 里得到覆盖。

#### 用例 3：多租户拒绝

```cpp
TEST(DistributedStorageConfigValidationTest, KvClientRejectsMultiTenant) {    // L309
    DistributedStorageConfig config;                                        // L310
    config.fs_adapter_type = "kvclient";                                     // L311
    config.single_tenant = false;                                            // L312
    EXPECT_FALSE(config.Validate());                                         // L313
}                                                                           // L314
```

- **L309** — 用 `TEST` 而非 `TEST_F`：直接构造 config，不经环境变量，所以不需要 fixture 的环境守卫。
- **L312-L313** — 锁定 §15.2 的单租户强制。

#### 用例 4：跳过文件系统校验

```cpp
TEST(DistributedStorageConfigValidationTest,                                 // L316
     KvClientSkipsFilesystemRootChecks) {                                     // L317
    DistributedStorageConfig config;                                         // L318
    config.fs_adapter_type = "kvclient";                                      // L319
                                                                             // L320
    config.fsdir.clear();                                                    // L321
    EXPECT_TRUE(config.Validate());                                           // L322
                                                                             // L323
    config.fsdir = "relative/path";                                          // L324
    EXPECT_TRUE(config.Validate());                                           // L325
}                                                                            // L326
```

- **L321-L322** — `fsdir` 为空仍通过，证明 kvclient 分支在 `fsdir` 检查**之前**返回（§15.2 的 L78 位置正确）。
- **L324-L325** — 相对路径也通过。对 hf3fs 这两种情况都会被 [L93-L105](../Mooncake/mooncake-store/src/config/distributed_storage_config.cpp#L93) 拒绝（构建日志里的 `fsdir is empty` / `fsdir must be an absolute path` 正是那两行的输出）。
- 一个测试覆盖两种非法输入，因为它们验证的是同一个"提前返回"行为。

#### 用例 5：分配器后端拒绝

```cpp
TEST(DistributedStorageConfigValidationTest,                                 // L328
     KvClientIsNotAValidAllocatorBackend) {                                   // L329
    DistributedStorageConfig config;                                         // L330
    config.fs_adapter_type = "kvclient";                                      // L331
                                                                             // L332
    EXPECT_TRUE(config.Validate());                                           // L333
    EXPECT_FALSE(config.ValidateForAllocator());                               // L334
}                                                                            // L335
```

- **L333** — 先确认普通校验通过，排除"因为别的原因失败"的干扰。
- **L334** — `ValidateForAllocator()` 必须拒绝。这锁定了 §15.1 把 kvclient 加入 `UsesObjectStorage()` 的连带效果：kvclient 不能被当作文件系统分配器后端。构建日志里的 `DFS allocator requires a filesystem adapter` 正是这条断言触发的输出。

---

## 19. 交叉验证证据

### 19.1 key 编码一致性（最关键的正确性前提）

| 项 | 适配器 | SDK | 一致 |
| --- | --- | --- | --- |
| 哈希算法 | `XXH64` | `XXH64`（`key_resolver.cc:51`） | ✓ |
| 种子 | `5381`（[.cpp:22](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L19)） | `HASH_SEED_VALUE (5381)`（`key_resolver.h:31`） | ✓ |
| 字节序 | 显式小端（[.cpp:182-183](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L188)） | `hash & 0xff; hash >>= 8`（`key_resolver.cc:52-55`） | ✓ |
| 键长度 | 8 字节 | `kCacheKeySizeBytes = 8`（`kv_types.h:42`） | ✓ |

**意义**：GENERIC 路径下 SDK 对 `BinaryKey` 输入**原样返回**（`key_resolver.cc:41-44` 的 `TryGetBinaryKey()` 命中即 `result.key = *bytes; return Status::OK();`）。所以适配器自算的键 == SDK 解析的键 → **PR2 写入的对象，PR1 的 Delete 能命中**。

### 19.2 `Wait(taskId, 0, ...)` 语义

| 证据 | 内容 |
| --- | --- |
| `kv_client.h:59` | *"Waits for the result and consumes the task even on timeout; **zero uses the default timeout**."* |
| `kv_client_impl.cc:294` | `const auto waitMs = timeoutMs == 0 ? config_.defaultWaitTimeoutMs : timeoutMs;` |
| `kv_types.h:259` | `std::uint64_t defaultWaitTimeoutMs{100};` |
| `config_utils.cc:183-186` | 配置文件的 `waitTimeoutMs` / `wait_timeout_ms` 键写入 `defaultWaitTimeoutMs` |

→ 适配器传 0 完全正确，超时策略归 SDK 配置文件所有。

### 19.3 `TASK_NOT_FOUND` 判为非终态的依据

`task_manager.cc:108-116`：

```cpp
Status ClientTaskManager::Wait(TaskId taskId, std::uint64_t waitTimeoutMs, TaskResult& result)
{
    auto task = Get(taskId);
    if (!task) { return Status::Error(StatusCode::TASK_NOT_FOUND, "task not found"); }

    auto status = WaitContext(task, waitTimeoutMs, result);
    (void)Remove(taskId);          // ← 无条件移除，即使超时
    return status;
}
```

→ "任务查不到"不能证明 IO 已完成，必须保留 buffer / handle。适配器 [.cpp:132-141](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L138) 的判断正确。

### 19.4 全零 key 被协议拒绝

`kv_protocol.cc:116-120` 定义 `IsCacheKeyAllZero`，在 **:219（Store）、:362（Retrieve）、:526（Batch）、:719（Batch）** 四处拒绝，返回 `INVALID_ARGUMENT "key is all zeros"`。

→ 适配器 [.cpp:178](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L184) 的 `hash == 0 → 5381` 守卫**必要**。

### 19.5 DELETE 的状态码集合

`trans_task_utils.cc:77-84`：

```cpp
if (opType == AsuOpType::DELETE) {
    switch (EntryStatusCode(opType, rawResult)) {
        case StatusCode::ENTRY_DELETE_FAILED:
            return Status::Error(StatusCode::ENTRY_DELETE_FAILED, "delete failed");
        default:
            return Status::Error(StatusCode::IO_ERROR, "unknown ASU delete result is " + ...);
    }
}
```

`ENTRY_KEY_NOT_EXIST` 仅在 `AsuOpType::QUERY` 分支产生（`trans_task_utils.cc:87-96`）。

→ 适配器把 `ENTRY_DELETE_FAILED` 归为服务错误、`ENTRY_KEY_NOT_EXIST` 归为对象缺失，两者不冲突；[.cpp:96-97](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L102) 的注释准确。

### 19.6 `entryStatus` 预填为 OK 的证据

`trans_task_utils.cc:145-148`：

```cpp
if (status.ok()) {
    std::fill(entryStatus.begin(), entryStatus.end(), isQuery ? keyExist : Status::OK());
    return;
}
```

→ 任务级失败时 `entryStatus` 可能仍持有提交期的 `OK`，所以 [.cpp:152-158](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L158) 的 default 分支**必须**用整体错误覆盖，不能读它。

---

## 20. 审核结论与待决事项

### 20.1 结论

PR1 的核心逻辑**正确**：key 编码与 SDK 逐字节一致、超时归属清晰、`entryStatus` 读取规则严格、失败路径不污染成员状态、索引映射无越界、CMake 宏覆盖完整、测试锁定了每个决策。已实测编译通过、18/18 测试通过、PR1 新增文件零警告。

### 20.2 待决事项（3 项）

#### 【高】SDK 依赖未合并分支，且 CMake 不校验 API 变体

本机存在两个 API 不兼容的 SDK 快照：

| 快照 | 分支 | `CacheKey` 形态 | `BinaryKey` / `FromBytes` / `keyResolution` |
| --- | --- | --- | --- |
| `D:\xiaojun\work\DramPool\unified-cache-management` | `z00984767-0917`（含 develop 合并） | `std::array<std::byte,8>` 裸别名 | **全部不存在** |
| `D:\xiaojun\work\tmp_pr1641\ucm-pr` | `feat/kvclient-typed-keys-namespaces`（PR #1641，未合入 develop） | `class`，私有构造 + `std::variant` | 全部存在 |

PR1 依赖的三个符号（[adapter .h:55](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h#L54) 的 `kv::BinaryKey`、[.cpp:243](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L221) 的 `CacheKey::FromBytes`、[.cpp:58](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L64) 的 `cfg.keyResolution`）**只在 #1641 分支存在**。

设计文档（`... - PR1 实施步骤.md` §0.2 第 30 行）已明确钉住该变体，所以代码本身没写错。真实风险是：[CMakeLists.txt:256](../Mooncake/mooncake-store/src/CMakeLists.txt#L256) 的 `find_path` 只按**文件名**匹配，不探测符号。指向 `DramPool` 时 configure 会打印 `KVClient SDK found`，随后编译失败——**configure 绿灯、编译红灯**，报错点离原因很远。设计文档第 706 行风险表自己写了"固定 `tmp_pr1641/ucm-pr` 变体，并在 CMake 检测处记录版本"，**这项没有落地**。

**待决**：PR 描述里写明依赖 SDK PR #1641 及提交号，还是加一个 `check_cxx_source_compiles` 符号探测守卫？（两个快照都无版本宏、无 `.pc`，符号探测是唯一可行的守卫。）

#### 【中】`PATH_SUFFIXES` 缺 `build/kv_semantics/src`

[CMakeLists.txt:262-263](../Mooncake/mooncake-store/src/CMakeLists.txt#L262) 与 [:267-268](../Mooncake/mooncake-store/src/CMakeLists.txt#L267) 的库搜索后缀为 `lib lib64 kv_semantics/src ucm/store/asu src build/src build`。

但 SDK 顶层构建（`BUILD_UCM_ASU=ON`，默认 OFF，见 SDK `CMakeLists.txt:13,72-73`）会把 `libkv_client.so` 输出到 `<sdk>/build/kv_semantics/src`（`kv_semantics/src/CMakeLists.txt:120` 无 `LIBRARY_OUTPUT_DIRECTORY` 覆盖）。而 [CMakeLists.txt:229-230](../Mooncake/mooncake-store/src/CMakeLists.txt#L229) 的注释恰恰建议"point this at the SDK source tree"。

结果：`find_path` 命中头文件，`find_library` 失败，`SDK_ROOT` 非空 → [L290-L297](../Mooncake/mooncake-store/src/CMakeLists.txt#L290) FATAL_ERROR，**即使 SDK 已正常构建**。指向 `<sdk>/build` 也不行——SDK 的 install 规则（`kv_semantics/src/CMakeLists.txt:141-142`）只装库不装头文件。

**修复**：两处 `PATH_SUFFIXES` 各加 `build/kv_semantics/src`。

#### 【低】两项小缺陷

1. **`Completion::terminal` 是只写不读的死字段** — [adapter .h:51](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h#L50) 声明，[.cpp:130](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L136) / [:136](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L142) / [:149](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L155) / [:156](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L162) 四处赋值，但 `DeleteBatch`（[L258-L265](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L236-L243)）只读 `entryStatus`。全仓库 grep `.terminal` / `->terminal` 在适配器外无任何命中。它承载的是"非终态需保留 buffer / handle"语义，但 PR1 没有需要保留的资源（Delete 不注册 buffer）。属为 PR2 预留的死代码。

2. **env var 形式的 `SDK_ROOT` 不触发 fail-loud** — [L252-L253](../Mooncake/mooncake-store/src/CMakeLists.txt#L252) 把 `ENV{MOONCAKE_KVCLIENT_SDK_ROOT}` 当作搜索 hint，但 [L290](../Mooncake/mooncake-store/src/CMakeLists.txt#L290) 的闸门只测 cache 变量。CMake 不会把环境变量灌进同名 cache 变量，所以 `export MOONCAKE_KVCLIENT_SDK_ROOT=<源码树>` 时只走 [L299-L303](../Mooncake/mooncake-store/src/CMakeLists.txt#L299) 的 STATUS 跳过分支——这正是 `f3c8ffd5` 声称要消除的"静默跳过"场景。**修复**：条件里加 `OR DEFINED ENV{MOONCAKE_KVCLIENT_SDK_ROOT}`。

另有一处**不算缺陷**：`Delete` 的 `results.empty()`（[.cpp:219](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L197)）在 `keys.size()==1` 时恒为假，是防御性死分支，但与姊妹适配器写法一致，保留无妨。

### 20.3 需确认的分期问题

`BatchDeleteObjects` 的**唯一**生产调用点 [client_service.cpp:3947-3950](../Mooncake/mooncake-store/src/client_service.cpp#L3947) 被 `UsesKvcs()` 门控：

```cpp
if (dfs_storage_backend_ && dfs_storage_backend_->UsesKvcs() && !removed_keys.empty()) {
    const auto deleted = dfs_storage_backend_->BatchDeleteObjects(removed_keys);
```

`Client::BatchRemove` 对 kvclient 走不到这里，所以 **PR1 的 Delete / DeleteBatch 目前没有生产调用路径**。在 PR1 阶段无害（Put 还是 `NOT_SUPPORTED`，没有对象被写入，不存在泄漏），但 PR2 落地写入后**必须**把这个门控扩成 `UsesKvcs() || UsesKvClient()`，否则对象只写不删。

同类门控清单（全部待 PR2 / PR3 处理）：

| 位置 | 用途 | 是否需扩到 kvclient |
| --- | --- | --- |
| [client_service.cpp:601](../Mooncake/mooncake-store/src/client_service.cpp#L601) | `AttachHostId` 强制 `dfs_replica_num=1` | PR3（记忆中标注为 PR3 范围） |
| [client_service.cpp:3864](../Mooncake/mooncake-store/src/client_service.cpp#L3864) | `Remove` 转 `BatchRemove` | PR3（value-first 删除） |
| [client_service.cpp:3891](../Mooncake/mooncake-store/src/client_service.cpp#L3891) | `RemoveByRegex` 返回 `NOT_SUPPORTED` | 待定 |
| [client_service.cpp:3912](../Mooncake/mooncake-store/src/client_service.cpp#L3912) | `RemoveAll` 返回 `NOT_SUPPORTED` | 待定 |
| [client_service.cpp:3947](../Mooncake/mooncake-store/src/client_service.cpp#L3947) | `BatchRemove` 调 `BatchDeleteObjects` | **PR2 必须** |
| [client_service.cpp:5491](../Mooncake/mooncake-store/src/client_service.cpp#L5491) | `ReconcileMissingKvcsReplica` | PR3（miss 自愈） |
| [master_service.cpp:832-833](../Mooncake/mooncake-store/src/master_service.cpp#L832) | master 侧 `enable_dfs_` | **PR2 必须**（否则 master 不启用 DFS） |

---

## 附录：提交历史

| 提交 | 说明 |
| --- | --- |
| `93ec9401` | [Store] Add kvclient backend wiring and adapter control plane |
| `5c4162e4` | [Store] Document why kvclient requires single-tenant mode |
| `757edd7c` | [Store] Rename the kvclient factory header |
| `86fe246e` | [Store] Drop the unpublished kvclient version floor |
| `f3c8ffd5` | [Store] Detect the kvclient SDK without pkg-config |
| `95f88152` | [Store] Let the kvclient SDK own timeouts and results |

评审修改对应的三条 @ayaka836 意见：

1. **超时归 SDK** → `95f88152`：删除 `MOONCAKE_KVCLIENT_WAIT_TIMEOUT_MS` 与 `timeoutMs < wait_timeout_ms` 交叉校验，改为 `Wait(task_id, 0, result)`。
2. **`entryStatus` 读取限制** → `95f88152`：只在 `OK` / `PARTIAL_FAILED` 时读取，其他终态统一传播整体错误。
3. **SDK 无 `.pc` 文件** → `f3c8ffd5`：pkg-config 之外增加 `SDK_ROOT` / include / library 探测、`Mooncake::kvclient` imported target、`MOONCAKE_KVCLIENT_REQUIRED` fail-fast。

另有两条通过对话确认的修改：删除 `MOONCAKE_KVCLIENT_MAX_OBJECT_BYTES` 与 512B 对齐校验（`95f88152`）；`EncodeBinaryKey` 折叠进 `MakeCacheKey`、删除 codec 头。
