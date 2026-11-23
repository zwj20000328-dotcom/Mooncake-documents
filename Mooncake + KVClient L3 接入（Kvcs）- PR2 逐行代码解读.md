# Mooncake + KVClient L3 接入（Kvcs）— PR2 逐行代码解读

版本：v1.0　日期：2026-11-20

对应文档：`Mooncake + KVClient L3 接入（Kvcs）- PR 实施设计.md`（PR2 章节）、`... - PR2 实施步骤.md`、`... - PR1 逐行代码解读.md`

---

## 0. 阅读约定

### 0.1 基线与范围

| 项 | 值 |
| --- | --- |
| 分支 | `Mooncake_l3_pr2` |
| HEAD | `a2841229`（含 PR1 全部提交） |
| PR2 状态 | **未提交**，全部改动在工作区 |
| 目标 SDK | `tmp_pr1641/ucm-pr`（分支 `feat/kvclient-typed-keys-namespaces`） |
| 编译环境 | WSL2 Ubuntu，g++ 15.2，`-std=c++20 -fsyntax-only` |

### 0.2 行号规则（与 PR1 解读文档的差异）

- 链接格式 `../Mooncake/<仓库相对路径>#L<行号>`，点击直接跳转。
- **本文档的所有行号都指向当前工作区**，因为 PR2 尚未提交，没有 commit 可以锚定。
- 这与 `PR1 逐行代码解读.md` 的规则**不同**：那份文档把两个适配器文件的链接锚定到 PR1 提交版 `95f88152`（工作区行号会随编辑漂移，所以它刻意不指向工作区）。
- 后果：`kvclient_object_storage_adapter.h` / `.cpp` 这两个文件在两份文档里指向**不同版本**。看 PR1 的原始形态用 `git show 95f88152:<path>`；看 PR2 的完整实现用本文档链接（需工作区未被再次修改）。
- 本文档的行号在编写时用脚本逐条回读校验：206 个链接全部命中、0 个越界。若后续再编辑这两个文件，行号会漂移，需重跑校验。

### 0.3 改动总览

| # | 文件 | 行数变化 | 内容 |
| --- | --- | --- | --- |
| 1 | [`mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h`](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h) | +9 / −0 | 三个私有成员声明 + `<cstdint>` |
| 2 | [`mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp`](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp) | +328 / −33 | 数据面全部实现 |
| 3 | [`mooncake-store/include/master_service.h`](../Mooncake/mooncake-store/include/master_service.h) | +1 / −0 | `use_kvclient_` |
| 4 | [`mooncake-store/src/master_service.cpp`](../Mooncake/mooncake-store/src/master_service.cpp) | +46 / −9 | Master 门控 |
| 5 | [`mooncake-store/tests/CMakeLists.txt`](../Mooncake/mooncake-store/tests/CMakeLists.txt) | +9 / −0 | 测试目标 |
| 6 | [`mooncake-store/tests/kvclient_object_storage_adapter_test.cpp`](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp) | +625（新增文件） | 22 个用例 |

合计 5 个修改文件 + 1 个新增文件，+390 / −42（不含新测试文件）。

**未改动**（与 PR2 文档一致）：`object_storage_adapter.h`、`ObjectPutRequest` / `ObjectGetRequest`、`distributed_storage_backend.*`、`client_service.*`、`real_client.*`。buffer 注册完全收敛在适配器内部，上层零改动。

**单独提交到 PR1 的一处**：`cfg.useBatchIoForSingleKey = true`，提交 `8af2fc10`，已推送到 `origin/PR1`。它在 PR1 上不产生行为变化（只影响 `LoadAsync`/`StoreAsync`，而 PR1 只落地 `DeleteAsync`），属于 Init 强制 SDK 配置的既有职责。该改动同样保留在 PR2 工作区中，所以本文档 §3.2 也会讲到它。

---

## 1. 验证结果

### 1.1 编译

本机（Windows）无 C++ 工具链，实际编译在 WSL2 Ubuntu 上用 g++ 15.2 完成。README（`~/kv/README.md`）指出 **CI 从不编译这个文件**——`.github/` 下 59 个文件无 `KVCLIENT` 字样，CI 不设置 `MOONCAKE_KVCLIENT_SDK_ROOT`，所以适配器 TU 在 CI 里永远不参与编译，绿灯不代表能编过。因此本地验证是必要的。

| TU | 结果 |
| --- | --- |
| `kvclient_object_storage_adapter.cpp` | exit 0 |
| `master_service.cpp` | exit 0（仅 `allocator.h:209` 既有 deprecation warning，与本改动无关） |
| `kvclient_object_storage_adapter_test.cpp` | exit 0 |

### 1.2 测试

链接 `adapter.o + test.o + environ.o + types.o` + SDK stub 库 + gtest/glog/xxhash 后实际运行：

```
[==========] 23 tests from 1 test suite ran. (0 ms total)
[  PASSED  ] 23 tests.
```

其中 `UnregisterFailureIsLoggedNotFatal` 实际触发了告警日志，证明注销失败路径被执行到：

```
W kvclient_object_storage_adapter.cpp:229] KVClient buffer unregister failed, handles=1, error=injected
```

编译过程中被捕获并修正的真实错误：`DeleteBatch` 接受 `std::span<const std::string>`，不能直接传初始化列表。

---

## 2. 适配器头文件

文件：[`kvclient_object_storage_adapter.h`](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h)

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L3](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h#L3) | `#include <cstdint>` | 新增私有成员签名里的 `uint64_t`。严格说它经由 `kv_types.h` 传递可用，但显式声明直接依赖更稳妥，不会因 SDK 头将来调整而断链。按字母序插在 `<memory>` 前 |
| [L57-60](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h#L57-L60) | 注释：Store 与 Load 共用 SDK 的 entry 解码器 | 说明为什么 `ToErrorCode` 需要 `is_write`：同一个码在读侧是「对象不存在」、在写侧是「写入失败」。注释同时写明**故意不给默认值**的理由——新写调用点漏传会编译失败，而不是静默拿到读语义 |
| [L61](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h#L61) | `static ErrorCode ToErrorCode(const kv::Status& status, bool is_write);` | 合并后的唯一映射入口。`static` 不变（不访问实例状态）。显式 `bool` 无默认值，与仓库既有 `MapNvmeKvStatus(uint32_t, bool is_write)` 约定一致；全部调用点里只有 `PutBatch` 的终态分支传 `true` |
| [L62-63](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h#L62-L63) | 注释：handle 有效期到传给 `UnregisterBuffers` 为止 | 这是**契约**而非描述：调用方必须自己收着 handle，否则注册会泄漏到 `Shutdown()`。写成注释是因为从签名看不出来 |
| [L64](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h#L64) | `tl::expected<uint64_t, ErrorCode> RegisterBuffers(void* ptr, size_t size);` | 注册单块 host 内存。用 `expected` 而非「`uint64_t` + 哨兵值」，因为要区分两类失败：参数非法（`INVALID_PARAMS`）与 SDK 注册失败（`DFS_SERVICE_UNAVAILABLE`），调用方需按项映射。非 `const`：调用 SDK 会推进其内部状态 |
| [L65](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h#L65) | 注释：SDK 无法重试的失败只记日志 | 说明为什么返回 `void` |
| [L66](../Mooncake/mooncake-store/include/storage/distributed/kvclient/kvclient_object_storage_adapter.h#L66) | `void UnregisterBuffers(const std::vector<uint64_t>& handles);` | 批量注销。`const&` 避免拷贝 handle 列表 |

三个成员都放在 `private:` 段、紧跟既有的 `ToErrorCode`。这是与早期方案的关键区别：原先的 `RegisterBuffer` / `UnregisterBuffer` / `RequiresBufferHandle` 是**基类虚函数**，会进入 `ObjectStorageAdapter` 接口、波及 KVCS/OSS/S3；现在是 kvclient 私有实现细节，上层完全看不到。

该头文件只有自己的 `.cpp` 一个包含者（`storage_backend.cpp` 明确禁止包含它），所以改动不触发任何其他 TU 重编。

---

## 3. 适配器实现

文件：[`kvclient_object_storage_adapter.cpp`](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp)

### 3.1 include

| 行 | 代码 | 唯一使用者 |
| --- | --- | --- |
| [L6](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L6) | `#include <limits>` | `std::numeric_limits<size_t>::max()`（[L346](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L346) 累加回绕检查） |
| [L8](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L8) | `#include <optional>` | `entry.mrKey = std::nullopt`（[L379](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L379)、[L480](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L480)） |
| [L9](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L9) | `#include <set>` | `std::set<kv::BinaryKey> seen`（[L315](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L315)） |

匿名命名空间现在只剩 [L22](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L22) 的 `kKvClientHashSeed`。原 `Errors<T>` 已删除——它服务两个 `NOT_SUPPORTED` 占位，数据面实现后成为死代码；`PutBatch`/`GetBatch` 的结果数组必须按下标一一对应（部分项失败、部分项提交），只能就地填充，不能整体替换。

### 3.2 `Init` 强制批量协议（[L57-62](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L57-L62)）

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L57](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L57) | `cfg.keyResolution = {};` | PR1 已有，强制 GENERIC 解析 |
| [L58-61](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L58-L61) | 注释 | 说明原因：非批量 LOAD/STORE 路径只返回 CQE 状态、没有 per-entry result buffer，数据面无法区分「对象不存在」与「一般失败」 |
| [L62](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L62) | `cfg.useBatchIoForSingleKey = true;` | 与 `keyResolution` 同类的「Init 强制配置」。SDK 里 `LoadAsync`/`StoreAsync` 用它决定单条请求走 `BATCH_LOAD`/`BATCH_STORE` 还是 `LOAD`/`STORE`（`kv_client_impl.cc:259,270`）。默认值虽是 `true`，但配置文件 `client.useBatchIoForSingleKey=false` 可覆盖，届时单条读的 miss 会变成 `CQE_CHECK_RESULT_BUFFER` → `INTERNAL_ERROR` |

注意注释末尾的 "This only affects LoadAsync/StoreAsync"（PR1 提交版本）：`DeleteAsync`/`QueryAsync` 不读这个开关，所以该改动在 PR1 上无行为变化。

### 3.3 `ToErrorCode` 补充与读写分流（[L74-132](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L74-L132)）

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L74-75](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L74-L75) | 签名新增 `bool is_write` | 读写方向由调用方表态。**刻意不给默认值**：带默认值会让新增写调用点漏传时静默拿到读语义——编译通过、行为错；无默认值则编译器强制每处表态。与仓库既有 `MapNvmeKvStatus(uint32_t, bool is_write)`（`executor_util.h:62`）同款约定 |
| [L87-90](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L87-L90) | 注释 | 记录关键事实：QUERY 用 `ENTRY_KEY_NOT_EXIST` 报告 miss；LOAD 与 BATCH_STORE 共用同一个解码器，用 `ENTRY_KEY_NOT_FOUND` / `ENTRY_DATA_NOT_EXIST` |
| [L91-93](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L91-L93) | `CQE_KEY_NOT_EXIST` / `ENTRY_KEY_NOT_EXIST` → `OBJECT_NOT_FOUND` | QUERY 专用，写侧不会出现，因此不参与读写分流，直接固定映射 |
| [L94-97](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L94-L97) | `ENTRY_KEY_NOT_FOUND` / `ENTRY_DATA_NOT_EXIST` → `is_write ? FILE_WRITE_FAIL : OBJECT_NOT_FOUND` | **修 PR1 的缺口 + 读写分流**。缺 `ENTRY_DATA_NOT_EXIST` 会让读 miss 落到 `default` → `INTERNAL_ERROR`，验收要求的 `OBJECT_NOT_FOUND` 不成立、PR3 的 miss 自愈也不触发；而写侧这两个码的含义是「写入失败」，映射成 `OBJECT_NOT_FOUND` 会污染调用方分类与写错误指标。`FILE_WRITE_FAIL`（`types.h:420`）比 `INTERNAL_ERROR` 更准确 |
| [L110-114](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L110-L114) | 注释 + `ENTRY_RETRY_ADVISED` → `DFS_SERVICE_UNAVAILABLE` | SDK 标注「可重试」的存储侧状况。原先落到 `default` → `INTERNAL_ERROR`，与单条 LOAD 的行为不一致（单条 LOAD 下同一 raw `0x01` 经 `EntryStatusCode(LOAD, raw)` 变成 `IO_ERROR` → `DFS_SERVICE_UNAVAILABLE`）。现在两条路径一致 |
| [L116-119](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L116-L119) | 注释 + `ENTRY_NO_RETRY_ADVISED` → `INTERNAL_ERROR` | 显式声明而非静默落入 `default`。语义上「明确不可重试」不应被当成瞬时服务不可用，故保持 `INTERNAL_ERROR` |

**哪个调用点传 `true`**——只有一处：

| 位置 | `is_write` | 理由 |
| --- | --- | --- |
| `PutBatch` 终态逐项映射（[L407-416](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L407-L416)） | **`true`** | 唯一能收到 `BATCH_STORE` entry 状态的分支 |
| `PutBatch` 提交被拒（[L387-391](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L387-L391)） | `false` | `StoreAsync` 的提交状态只可能是 `NOT_INITIALIZED` / `INVALID_ARGUMENT` / resolver 错误 / `INTERNAL_ERROR` / `RESOURCE_BUSY`，不含 `ENTRY_*` |
| `PutBatch` 非终态 fan-out（[L400-406](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L400-L406)） | `false` | `waitForCompletion` 非终态只填 `TIMEOUT` / `IN_PROGRESS` / `CANCELED` 或合成的 `INTERNAL_ERROR`，同样不含 `ENTRY_*` |
| `DeleteBatch` / `GetBatch` / `Exists` 全部调用点 | `false` | 读/删/查路径 |

后两行传 `true` 也能编译、结果相同，但那是**空转**——把写侧映射用在了不可能出现写侧 entry 状态的地方，掩盖了「哪些分支才真正需要分流」这个事实。

评审核实过：本仓库写路径（`BatchWrite`/`WriteDfsReplicas`/`RecordFailure`）不特判 `OBJECT_NOT_FOUND`，所以这是**错误分类失真**而非控制流 bug；但仍应修，因为它污染日志与指标。

#### 3.3.1 为什么只有一处传 `true`：`waitForCompletion` 的分支

PR1 已有，PR2 未改动，但它是理解上面那张「哪个调用点传 `true`」表的关键：

| 分支 | 填入的 entry 状态 |
| --- | --- |
| `OK` / `PARTIAL_FAILED` | `terminal = true`，保留 SDK 的逐项结果——**唯一可能含 `ENTRY_*` 的情况** |
| `TASK_NOT_FOUND` | `terminal = false`，填合成的 `INTERNAL_ERROR` |
| `TIMEOUT` / `IN_PROGRESS` / `CANCELED` | `terminal = false`，填 task 级状态 |
| 其他 task 级错误 | `terminal = true`，填 task 级状态 |

所以 `completion.entryStatus[0]` 在非终态分支里**永远是 task 级码**，不可能带 `ENTRY_*`。

### 3.4 `RegisterBuffers`（[L205-218](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L205-L218)）

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L205-206](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L205-L206) | 成员定义签名 | 私有成员，直接用 `kv_client_`，调用点不传 client |
| [L207-209](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L207-L209) | `ptr == nullptr \|\| size == 0` → `INVALID_PARAMS` | 与 SDK provider 的前置校验一致（`aicpu_trans_provider.cc:1765` 对 `addr==0\|\|size==0` 报 `INVALID_ARGUMENT`），提前拒绝避免无谓的锁竞争 |
| [L210-211](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L210-L211) | `{HOST, (uint64_t)ptr, size, -1, -1}` | 聚合初始化 `MemoryRegion`：HOST 类型、指针转整数地址、长度、`deviceId`/`numaNode` 都是 `-1` 表示不限定。**这是零拷贝的关键——注册的就是调用方内存本身** |
| [L212](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L212) | `std::vector<kv::RegisteredMemory> registered;` | SDK 出参，承载 `region`+`handle`+`tokenId` |
| [L213](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L213) | `kv_client_->RegisterRegions(regions, registered)` | 触发 provider 的 MR 注册。host 内存在 UB 协议下走 `AcquireHostMapping`，按对齐基址映射并引用计数 |
| [L214-216](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L214-L216) | `!status.ok() \|\| registered.size() != 1` → `DFS_SERVICE_UNAVAILABLE` | 两个条件合并：SDK 拒绝，或返回条数异常。1 进 1 出才能唯一确定 handle |
| [L217](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L217) | `return registered[0].handle;` | 返回 opaque token；适配器从不解引用它 |

### 3.5 `UnregisterBuffers`（[L220-232](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L220-L232)）

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L222](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L222) | `if (handles.empty()) return;` | 全部请求被守卫拒绝时跳过 SDK 调用。也避免 `catch` 路径在无 handle 时做空操作 |
| [L223-226](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L223-L226) | 注释 | 说明两个决定：(a) 单次批量调用，因为 SDK 自己会跳过 `kInvalidMRHandle` 与未知 handle，批量可避免每 handle 取一次 `memoryMu_`；(b) 失败只记日志，SDK 会保留注册且已无 handle 可重试 |
| [L227](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L227) | `kv_client_->UnregisterRegions(handles)` | **一次传整个 vector**。SDK 内部 `memoryMu_` 只取一次、`providerMemoryStates_` 只遍历一次。SDK 自己的 `kv_client_runner.cc:205-221` 也是这么用的 |
| [L228-231](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L228-L231) | 失败 `LOG(WARNING)` 带 handle 数与 `status.message` | 泄漏必须可见。**代价**：粒度从「哪个 handle」变成「几个 handle、整批失败」，这是批量化换来的 |

改为批量前，110 个 handle（SDK 默认 `batchLoadIoNum`）会产生 110 次 `memoryMu_` 加锁 + 110 次 AICPU 设备上下文构建。

### 3.6 单条入口包装：`Put` / `PutV` / `Get`（[L287-298](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L287-L298)、[L427-432](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L427-L432)）

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L287-288](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L287-L288) | `Put(const std::string& logical_key, std::span<const char> data)` | 参数从匿名改为具名——占位实现不需要，真实现需要 |
| [L289](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L289) | `iovec iov{const_cast<char*>(data.data()), data.size()};` | 把 `span` 包成单个 iovec。`const_cast` 是基类只读接口与 `iovec.iov_base` 非 const 的桥接——KVCS 同一写法（`kvcs_object_storage_adapter.cpp:95`），适配器不写这块内存 |
| [L290](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L290) | `return PutV(logical_key, &iov, 1);` | 转调。`iov` 是栈对象，但 `PutV` 同步返回，生命周期足够 |
| [L293-295](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L293-L295) | `PutBatch({{logical_key, iov, iovcnt}})` | 单元素 batch。注册、`StoreAsync`、`Wait`、注销全在 batch 路径内完成，不单独实现 I/O |
| [L296-297](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L296-L297) | `results.empty() ? INVALID_PARAMS : std::move(results.front())` | `PutBatch` 契约保证一进一出，`empty()` 分支是防御性检查；`std::move` 避免拷贝 |
| [L427-429](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L427-L429) | `Get(..., void* buf, size_t len)` → `GetBatch({{logical_key, buf, len}})` | **读侧对称包装**：`Get` 此前是 `NOT_SUPPORTED` 占位，现在与 `PutV` 同构——把单条请求包成单元素 `GetBatch`，注册、`LoadAsync`、`Wait`、注销全在 batch 路径内完成。与 KVCS 的 `Get` 写法一致（`kvcs_object_storage_adapter.cpp:93-97`） |
| [L430-431](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L430-L431) | `results.empty() ? INVALID_PARAMS : std::move(results.front())` | 同上：`GetBatch` 保证一进一出，成功时返回请求长度 N |

调用链与 KVCS 完全一致：写侧 `Put` → `PutV` → `PutBatch`，读侧 `Get` → `GetBatch`。四个单条入口都不单独实现 I/O，也不做 buffer 注册——全部由 batch 路径承担。

### 3.7 `PutBatch`（[L300-425](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L300-L425)）

#### 入口与 try 结构

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L303](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L303) | `results(requests.size())` | 批量契约：一请求一结果、下标对齐。默认构造的 `expected` 是**成功态**，所以每条路径都必须显式赋值，漏填会静默报成功 |
| [L304-309](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L304-L309) | Init 守卫就地填 `DFS_SERVICE_UNAVAILABLE` | 在 `try` **之外**：此时还没注册任何 handle，无需清理；同时让 `kv_client_` 非空成为后续所有调用的前置条件。原 `FillError` 已内联 |
| [L311-312](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L311-L312) | `handles` 声明 + `reserve` | **必须在 `try` 外**，否则 `catch` 看不到它。`reserve` 保证后续 `push_back` 不因扩容抛异常——登记动作本身没有异常窗口 |
| [L313-314](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L313-L314) | 注释 + `try {` | try 覆盖注册 → 提交 → 等待 → 映射的完整区间 |
| [L315-316](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L315-L316) | `std::set<kv::BinaryKey> seen;` | 同批重复 key 判定；只对写侧有意义（同批写同一 key 语义含糊） |
| [L317-320](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L317-L320) | `entries` / `indices` + `reserve` | 待提交项与原始下标并行；下标用于把结果映射回请求位置 |

#### 逐请求循环

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L321](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L321) | `for (size_t i = 0; ...)` | 用 `i` 作结果下标 |
| [L322-326](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L322-L326) | 空 key / 空 iov / `iovcnt<=0` → `INVALID_PARAMS` | 结构性守卫。**不检查 `iov[0].iov_base`**——段级校验交给下面的求和循环，覆盖所有段而非只首段 |
| [L327](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L327) | `MakeCacheKey(...)` | XXH64(seed=5381) + 显式小端，得到 8 字节 wire key |
| [L328-331](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L328-L331) | `seen.find(...) != end` → `INVALID_PARAMS` | **只查不插**：重复项被拒但不占用 key |
| [L332-337](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L332-L337) | 注释 | 说明三件事：空段/回绕会让注册与提交区间都错；SDK 每 entry 只读一段连续区间，所以必须校验物理连续；用整数做地址运算，因为比较不同分配来的指针是 UB |
| [L338-339](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L338-L339) | `base = reinterpret_cast<uintptr_t>(iov[0].iov_base)` | 首段基址转整数，供连续性比较 |
| [L340-341](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L340-L341) | `total = 0; valid = true;` | 累加器兼「已消费偏移」；标志位替代 `std::optional`，内联后不需要那层间接 |
| [L342-343](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L342-L343) | `for (int j ...)` + `const auto& segment` | `int` 与 `iovcnt` 类型一致，避免有符号/无符号比较告警；引用取段避免拷贝 `iovec` |
| [L344-349](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L344-L349) | 空指针 / 零长度 / `size_t` 回绕 → `valid=false; break;` | 回绕检查必须在 `+=` **之前**。任一段非法即中止，不必扫完剩余段 |
| [L350-354](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L350-L354) | `segment.iov_base != base + total` → `valid=false; break;` | **连续性校验**（blocker 修复）。`total` 此时正好是该段的期望起始偏移。不满足说明切片来自不同分配，若照旧注册 `[base, base+total)`，SDK 会把 buf0 之后的无关堆内存当成对象尾部写入 L3，而 `PutBatch` 报成功、master 记下正确总长，后续读回损坏数据 |
| [L355](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L355) | `total += segment.iov_len;` | 累加，同时推进期望偏移 |
| [L357-360](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L357-L360) | `!valid \|\| total == 0` → `INVALID_PARAMS` | 不注册、不提交、不占用 key |
| [L361](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L361) | `RegisterBuffers(iov[0].iov_base, total)` | 注册区间 = 首段基址 + 总长度。连续性已校验，所以这个区间确实覆盖全部 payload |
| [L362-365](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L362-L365) | 注册失败 → 该项错误，`continue` | 逐项隔离；该项未进 `handles`，不会被注销 |
| [L366-368](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L366-L368) | 注释 + **先** `handles.push_back(*handle)` | 顺序关键：登记必须在任何可能抛异常的操作之前，否则 `catch` 无法释放它 |
| [L369](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L369) | 后 `seen.insert(binary_key)` | 只有完全受理的请求才认领 key，避免无效项挡住后续同 key 的合法请求 |
| [L371-372](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L371-L372) | `entry.key = CacheKey::FromBytes(binary_key)` | 传二进制 key，不走 SDK 的 string 解析路径 |
| [L373-376](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L373-L376) | `region = {HOST, iov[0].base, total, -1, -1}` | 零拷贝：直接指向调用方内存，与注册区间严格一致 |
| [L377](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L377) | `entry.buffer.handle = *handle` | SDK 提交时据此查 `registeredRegions_` 回填 `mrKey`；没有它 SQE 打包会返回 `BUFFER_NOT_REGISTERED` |
| [L378](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L378) | `entry.offset = 0` | 整对象写，无对象内偏移 |
| [L379](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L379) | `entry.mrKey = std::nullopt` | mrKey 是 client 私有元数据，由 SDK 从 handle 解析（`kv_client_impl.cc:467-475`） |
| [L380-381](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L380-L381) | `entries.push_back(std::move(entry))` + `indices.push_back(i)` | `handles`/`entries`/`indices` 三者严格一一对应 |

#### 提交、等待与出口

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L384](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L384) | `if (!entries.empty())` | 用条件块**替代** `if (entries.empty()) return results;` 提前返回——单一出口才能保证注销恰好一次 |
| [L385-386](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L385-L386) | `task_id = kInvalidTaskId` + `StoreAsync` | 异步提交。`StoreAsync` 是覆盖写（upsert），所以不做 `DeleteAsync` 预删 |
| [L387-391](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L387-L391) | 提交被拒 → 仅参与项 `ToErrorCode(submit, false)` | 守卫已填的错误保持不变；不 return，继续走到结尾注销。提交状态含 `NOT_INITIALIZED`/`INVALID_ARGUMENT`/resolver 错误/`INTERNAL_ERROR`/`RESOURCE_BUSY`，**不含 `ENTRY_*`**，所以传 `false` |
| [L392-394](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L392-L394) | `waitForCompletion(task_id, entries.size())` | 同步等待，超时由 SDK 决定（内部 `Wait(task_id, 0, result)`） |
| [L395-399](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L395-L399) | entry 数不匹配 → `INTERNAL_ERROR` | 防御 SDK 结果条数异常 |
| [L400-406](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L400-L406) | 非终态 → `ToErrorCode(entryStatus[0], false)` 统一映射 | 超时/取消时逐项结果不可信；`waitForCompletion` 已把 task 状态填满每个 entry。非终态只含 `TIMEOUT`/`IN_PROGRESS`/`CANCELED`/合成 `INTERNAL_ERROR`，**不含 `ENTRY_*`**，故传 `false` |
| [L407-416](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L407-L416) | 终态 → 逐项 `ok()` / `ToErrorCode(status, true)` | 仅 `OK`/`PARTIAL_FAILED` 走到这里，STORE 成功 entry 是 `Status::OK()`。**这是唯一能收到 `BATCH_STORE` entry 状态的分支，所以传 `true`**——写侧的 miss 码要映射成 `FILE_WRITE_FAIL` |
| [L419-422](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L419-L422) | `catch (...) { UnregisterBuffers(handles); throw; }` | 异常展开时释放已注册 handle，再原样抛出（不吞异常、不改类型） |
| [L423-424](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L423-L424) | `UnregisterBuffers(handles); return results;` | 正常路径注销，发生在 `Wait` 返回之后，满足 SDK「in-flight task 引用的 handle 必须保持注册」。与 `catch` 互斥（catch 里 `throw` 了），不会重复注销 |

### 3.8 `GetBatch`（[L434-532](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L434-L532)）

与 `PutBatch` 严格同构，三处差异下面标出。

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L437](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L437) | `results(requests.size())` | 元素类型 `size_t`（回报长度），非 `void` |
| [L438-443](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L438-L443) | Init 守卫 | 同 `PutBatch`，在 `try` 外 |
| [L445-446](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L445-L446) | `handles` + `reserve` | 必须在 `try` 外 |
| [L447-448](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L447-L448) | 注释 + `try {` | 与写路径同一异常契约 |
| [L449-452](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L449-L452) | `entries` / `indices` | **差异 1：没有 `std::set seen`**——读不去重，同一 key 读两次合法（各自独立目标 buffer） |
| [L454-458](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L454-L458) | 空 key / 空 buffer / `size==0` → `INVALID_PARAMS` | **差异 2：没有求和与连续性循环**——读请求本就是单段连续 buffer（`void* buffer` + `size_t size`），不像写那样要合并 iov |
| [L459-462](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L459-L462) | 注释 + `RegisterBuffers(buffer, size)` | 注册整段目标内存；SDK 会把对象**直接写进这块内存**，这就是零拷贝读 |
| [L463-466](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L463-L466) | 注册失败 → 该项错误，`continue` | 逐项隔离 |
| [L467-469](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L467-L469) | 注释 + `handles.push_back` | **差异 3：没有 `seen.insert`**（无去重）；先记 handle 的时序要求与写路径相同 |
| [L471-473](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L471-L473) | `entry.key = CacheKey::FromBytes(...)` | 与写路径同一编码 |
| [L474-477](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L474-L477) | `region = {HOST, buffer, size, -1, -1}` | 目标就是调用方 buffer，零拷贝读的关键 |
| [L478-480](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L478-L480) | `handle` / `offset=0` / `mrKey=nullopt` | 与写路径对称，mrKey 由 SDK 从 handle 解析 |
| [L481-482](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L481-L482) | `entries.push_back` + `indices.push_back(i)` | 同上 |
| [L485-487](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L485-L487) | `if (!entries.empty())` + `LoadAsync` | 单一出口。SDK 内部按 `useBatchIoForSingleKey`（已强制 true）选 `BATCH_LOAD` |
| [L488-492](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L488-L492) | 提交被拒 → 参与项 `ToErrorCode(submit, false)` | 读侧本就传 `false`（读语义）；提交状态不含 `ENTRY_*`，与写路径两处 `false` 同理 |
| [L493-495](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L493-L495) | `waitForCompletion(task_id, entries.size())` | 同上 |
| [L496-500](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L496-L500) | entry 数不匹配 → `INTERNAL_ERROR` | 同上 |
| [L501-507](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L501-L507) | 非终态 → 统一映射 `ToErrorCode(entryStatus[0], false)` | 同上 |
| [L508-523](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L508-L523) | 终态逐项：`ok()` → `requests[index].size`，否则 `ToErrorCode(status, false)` | `index = indices[j]` 是原始请求下标，故 `requests[index]` 取的是正确请求。LOAD 成功 entry 是 `Status::OK()`，`ok()` 有效。读 miss 经 `ToErrorCode(..., false)` 映射为 `OBJECT_NOT_FOUND`——**这里传 `false` 才对**：LOAD 的 miss 是「对象不存在」，不是「写失败」 |
| [L512-515](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L512-L515) | 注释 | 如实说明 SDK 不暴露传输字节数，本层无法检测截断，交由上层 checksum。原写法声称「后端比对以发现短读」是假的——后端 `*object_results[i] != object_requests[i].size`（`distributed_storage_backend.cpp:1078`）对我返回的值永远为假 |
| [L526-529](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L526-L529) | `catch (...) { UnregisterBuffers; throw; }` | 同 `PutBatch` |
| [L530-531](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L530-L531) | `UnregisterBuffers(handles); return results;` | 同 `PutBatch`，注销发生在 `Wait` 返回之后 |

读路径的连续 buffer 假设**成立**：后端 `BatchReadOnce` 已把多 slice 读先 staging 成一段连续 `vector<char>` 再调 `GetBatch`（`distributed_storage_backend.cpp:1057-1064`）。

### 3.9 `Exists`（[L534-569](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L534-L569)）

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L534-535](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L534-L535) | `Exists(const std::string& logical_key)` | 参数从匿名改为具名 |
| [L536-538](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L536-L538) | Init 守卫 | 与其他数据方法一致 |
| [L539-542](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L539-L542) | 注释 + 空 key 本地拒绝 → `INVALID_PARAMS` | 不发起 SDK 查询。注意 `MakeCacheKey` 对空串会算出合法哈希（seed=5381），**不会**被全零检查拦住，所以必须在此显式拒绝 |
| [L544-545](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L544-L545) | `keys{CacheKey::FromBytes(MakeCacheKey(...))}` | 与写/删同一编码 |
| [L546-549](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L546-L549) | `QueryAsync` + 提交失败 `ToErrorCode(submit, false)` | 查询走 `AsuOpType::QUERY`，**不注册 buffer**（`SubmitAsync` 的 keys 重载不涉及 `registeredRegions_`）。无写语义，显式传 `false` |
| [L551-552](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L551-L552) | 注释 | 记录关键陷阱：key 不存在会让 task 级变成 `PARTIAL_FAILED`，但它仍携带可信的逐项结果，不能当失败处理 |
| [L553](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L553) | `waitForCompletion(task_id, keys.size())` | **替代裸 `Wait`**。`WaitContext` 直接 `return result.status`，而 miss 时 `Finalize` 把 `finalStatus` 置为 `PARTIAL_FAILED`（`task_manager.cc:254-256`）；若写成 `if (!wait.ok()) return error`，每次 miss 都会返回 `INTERNAL_ERROR`，`false` 分支永远不可达 |
| [L554-557](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L554-L557) | `!terminal` → `ToErrorCode(..., false)` | 覆盖超时/取消/`TASK_NOT_FOUND`。此时 `entryStatus` 已被 helper 填满 task 状态，`[0]` 一定有效 |
| [L558-559](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L558-L559) | 注释 | 说明为什么必须按 code 分支：`ENTRY_KEY_EXIST` / `ENTRY_KEY_NOT_EXIST` 都由 `Status::Error` 产生（`trans_task_utils.cc:142-143`），`ok()` 无法区分命中与未命中 |
| [L560-568](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L560-L568) | `switch`：`ENTRY_KEY_EXIST` → `true`；`ENTRY_KEY_NOT_EXIST` → `false`；`default` → `ToErrorCode(..., false)` | 命中/未命中直接返回布尔；其他 entry 级错误按读语义映射 |

**删掉的一处冗余**：文档原要求校验 `queryResult->exists.size() != 1`。SDK 的 `CompleteTransportTask` 已做这件事并把不匹配转成 `INTERNAL_ERROR` 写进 `entryStatus`（`task_manager.cc:182-186`），会自然落到 `default` 分支。重复检查是死代码。

`ListKeys`（[L571-574](../Mooncake/mooncake-store/src/storage/distributed/kvclient/kvclient_object_storage_adapter.cpp#L571-L574)）保持 `NOT_SUPPORTED`——SDK 无 key 枚举接口。

---

## 4. Master 门控

### 4.1 成员（[`master_service.h` L2065](../Mooncake/mooncake-store/include/master_service.h#L2065)）

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L2065](../Mooncake/mooncake-store/include/master_service.h#L2065) | `bool use_kvclient_{false};` | 紧跟 `use_kvcs_`，与既有 DFS 标志同区。默认 `false` 保证未配置 kvclient 时所有新门控都走原路径 |

### 4.2 `enable_dfs_`（[`master_service.cpp` L829-834](../Mooncake/mooncake-store/src/master_service.cpp#L829-L834)）

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L830-832](../Mooncake/mooncake-store/src/master_service.cpp#L830-L832) | 注释改为「KVCS and KVClient」 | 原注释只提 KVCS |
| [L833-834](../Mooncake/mooncake-store/src/master_service.cpp#L833-L834) | `enable_dfs_ = ...enabled \|\| UsesKvcs() \|\| UsesKvClient();` | **整个门控的入口**。不加这一条，kvclient 配置下 `enable_dfs_` 为 false，[L835](../Mooncake/mooncake-store/src/master_service.cpp#L835) 直接 return，后面所有分支都不会执行。重排为两行是因为加了第三个条件后超出 80 列 |

### 4.3 kvclient 早退分支（[L896-911](../Mooncake/mooncake-store/src/master_service.cpp#L896-L911)）— 最关键

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L896](../Mooncake/mooncake-store/src/master_service.cpp#L896) | `if (dfs_config.UsesKvClient()) {` | 位置必须在 KVCS 分支的 `return`（[L893](../Mooncake/mooncake-store/src/master_service.cpp#L893)）之后、filesystem 的 `recovery_enabled` throw（[L913](../Mooncake/mooncake-store/src/master_service.cpp#L913)）之前。`fs_adapter_type` 是单字符串，所以 `UsesKvcs()` 与 `UsesKvClient()` **互斥**（`distributed_storage_config.h:32-33`），放在 KVCS 之后安全 |
| [L897-899](../Mooncake/mooncake-store/src/master_service.cpp#L897-L899) | 注释 | 说明为什么启动期就拒绝：kvclient 无恢复路径，而下面的恢复门保持 KVCS-only，否则运行期 restore 会静默失败 |
| [L900-906](../Mooncake/mooncake-store/src/master_service.cpp#L900-L906) | `if (recovery_enabled) { LOG(ERROR) << ...; throw std::invalid_argument(...); }` | 复用 [L837](../Mooncake/mooncake-store/src/master_service.cpp#L837) 的 `recovery_enabled`（即 `enable_oplog_`）。**用 `throw` 而非文档原写的 `LOG(FATAL)`**：同函数内既有配置校验一律用 `throw std::invalid_argument`（[L843](../Mooncake/mooncake-store/src/master_service.cpp#L843)、[L853](../Mooncake/mooncake-store/src/master_service.cpp#L853)、[L917](../Mooncake/mooncake-store/src/master_service.cpp#L917)），`master.cpp` 也有 `catch (const std::invalid_argument&)` 的惯例（[L1540-1545](../Mooncake/mooncake-store/src/master.cpp#L1540-L1545)）。`LOG(FATAL)` 会 `abort()` 留下 core，与周围风格不一致 |
| [L907](../Mooncake/mooncake-store/src/master_service.cpp#L907) | `use_kvclient_ = true;` | 置位后，descriptor 铸造、可用性门、驱逐白名单三处才会放行 |
| [L908](../Mooncake/mooncake-store/src/master_service.cpp#L908) | 注释 | 对象存储后端：不创建 EFC target、不做水位淘汰（与 KVCS 的差异——KVCS 有容量 API） |
| [L909-911](../Mooncake/mooncake-store/src/master_service.cpp#L909-L911) | `LOG(INFO) << "KVClient distributed object storage initialized..."` + `return;` | 验收项要求日志出现这句话，带 `FormatStr()` 便于排查配置。**必须 return**：否则会继续走 filesystem allocator 初始化 |

不这么做的后果（文档所列，已核实成立）：`enable_oplog=true` 落到 [L914-919](../Mooncake/mooncake-store/src/master_service.cpp#L914-L919) 的 throw；`enable_oplog=false` 落到 [L921-932](../Mooncake/mooncake-store/src/master_service.cpp#L921-L932) 初始化 `ShardAllocator`，对 object storage 必然失败，[L940](../Mooncake/mooncake-store/src/master_service.cpp#L940) 把 `enable_dfs_` 重置为 false，所有 DFS 写返回 `DFS_SERVICE_UNAVAILABLE`。

`<stdexcept>` 已在 [L19](../Mooncake/mooncake-store/src/master_service.cpp#L19)，无需新增 include。

### 4.4 descriptor 铸造（[L4987-4995](../Mooncake/mooncake-store/src/master_service.cpp#L4987-L4995)）

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L4988](../Mooncake/mooncake-store/src/master_service.cpp#L4988) | `if (use_kvcs_ \|\| use_kvclient_)` | **唯一的改动**。kvclient 与 KVCS 的 descriptor 形状完全一致，所以共用分支 |
| [L4989-4992](../Mooncake/mooncake-store/src/master_service.cpp#L4989-L4992) | `object_size = aligned_size = value_length; shard_idx = -1` | 既有代码，未动。**不做 `AlignUp`** |
| [L4995-4998](../Mooncake/mooncake-store/src/master_service.cpp#L4995-L4998) | `else if (!dfs_allocator_ \|\| !IsInitialized())` | 原 filesystem 分支保持不变，kvclient 已在 [L4988](../Mooncake/mooncake-store/src/master_service.cpp#L4988) 被截获 |

无需新增 `kvclient_config.h` / `CheckedAlignUp` include。

### 4.5 DFS 可用性门（[L5146-5154](../Mooncake/mooncake-store/src/master_service.cpp#L5146-L5154)）

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L5147-5150](../Mooncake/mooncake-store/src/master_service.cpp#L5147-L5150) | 加入 `!use_kvclient_ &&` | 原条件要求「非 KVCS 时必须有已初始化的 allocator」；kvclient 同样没有 allocator，必须一并豁免，否则 [L5146](../Mooncake/mooncake-store/src/master_service.cpp#L5146) 恒真、所有写被拒 |
| [L5151-5154](../Mooncake/mooncake-store/src/master_service.cpp#L5151-L5154) | 日志与 `DFS_SERVICE_UNAVAILABLE` | **保持原返回码不变**，按要求不改成 `INVALID_PARAMS` |

### 4.6 NOF 拒绝（[L5155-5164](../Mooncake/mooncake-store/src/master_service.cpp#L5155-L5164)）

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L5155-5157](../Mooncake/mooncake-store/src/master_service.cpp#L5155-L5157) | 注释 | 说明为什么**不能**只依赖 `#ifndef USE_NOF`：那种写法在 `-DUSE_NOF=ON` 构建下会漏拦 |
| [L5158-5159](../Mooncake/mooncake-store/src/master_service.cpp#L5158-L5159) | `if (dfs_replica_num > 0 && use_kvclient_ && nof_replica_num > 0)` | **无条件检查**，位于 `#ifndef USE_NOF` 块（[L5173](../Mooncake/mooncake-store/src/master_service.cpp#L5173)）之前，所以两种构建都生效 |
| [L5160-5162](../Mooncake/mooncake-store/src/master_service.cpp#L5160-L5162) | `LOG(ERROR)` 带 `error=nof_not_supported_with_kvclient` | 沿用该函数既有的 `key=... error=...` 日志风格，便于按标记检索 |
| [L5163](../Mooncake/mooncake-store/src/master_service.cpp#L5163) | `return INVALID_PARAMS` | 与文档一致 |

`PrepareWriteStart` 覆盖 `PutStart` / `UpsertStart` / `BatchPutStart`——已核实这三者都调它，所以一处改动即可覆盖。

### 4.7 驱逐白名单（[L6329-6331](../Mooncake/mooncake-store/src/master_service.cpp#L6329-L6331)、[L6347-6350](../Mooncake/mooncake-store/src/master_service.cpp#L6347-L6350)）

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L6331](../Mooncake/mooncake-store/src/master_service.cpp#L6331) | `(use_kvcs_ \|\| use_kvclient_)` | `supports_dfs_condition`：`replica_type` 校验。不加则 `EvictDiskReplicaIfCurrent(DFS)` 在 [L6333](../Mooncake/mooncake-store/src/master_service.cpp#L6333) 被判 `INVALID_PARAMS` |
| [L6348-6350](../Mooncake/mooncake-store/src/master_service.cpp#L6348-L6350) | `(use_kvcs_ \|\| use_kvclient_) && expected_replica_id != nullptr && ...` | `target_pred` 内的**第二处**白名单，必须与第一处同步放宽，否则前置校验通过但匹配不到副本 |

两处都在 `EvictDiskReplicaImpl` 内。`EvictDiskReplica` / `BatchEvictDiskReplica` 传 `expected_replica_id = nullptr`，所以 DFS 分支仍为假、仍被拒——**PR2 不改变该行为**。

### 4.8 恢复门保持 KVCS-only（三处只加注释）

| 行 | 所属函数 | 说明 |
| --- | --- | --- |
| [L3422-3423](../Mooncake/mooncake-store/src/master_service.cpp#L3422-L3423) | `ConstructLegacyStandbyReplicas` | 注释 + `if (!use_kvcs_ \|\| ...)` 未改 |
| [L3717-3718](../Mooncake/mooncake-store/src/master_service.cpp#L3717-L3718) | `RestoreFromStandbyStateCore` | 注释 + `if (enable_dfs_ && !use_kvcs_)` 未改 |
| [L4007-4008](../Mooncake/mooncake-store/src/master_service.cpp#L4007-L4008) | `RestoreFromStandbyStateCore` | 注释 + `if (!use_kvcs_ \|\| ...)` 未改 |

三处都**不并入 `use_kvclient_`**，注释说明理由：kvclient + oplog 已在启动期 throw 拒绝，所以 kvclient 运行永远不会到达这些路径。若误并入，会让 kvclient 走进它并不支持的 DFS descriptor 恢复逻辑。

`GetStorageUsageSnapshot` **无需改动**：kvclient 既无 `dfs_allocator_` 也无 `dfs_kvcs_capacity_provider_`，两个分支都为假，`snapshot.dfs` 自然保持 `{}`。

---

## 5. 测试构建

文件：[`tests/CMakeLists.txt`](../Mooncake/mooncake-store/tests/CMakeLists.txt)

| 行 | 代码 | 解释 |
| --- | --- | --- |
| [L485-487](../Mooncake/mooncake-store/tests/CMakeLists.txt#L485-L487) | 注释 | 说明为什么测试目标自己也要链 `Mooncake::kvclient`：测试 include SDK 头，而 `mooncake_store` 是 PRIVATE 链接（`src/CMakeLists.txt:883`），include 目录不会传播 |
| [L488](../Mooncake/mooncake-store/tests/CMakeLists.txt#L488) | `if(MOONCAKE_KVCLIENT_SDK_FOUND)` | SDK 缺失时整个目标不创建。该变量经 `PARENT_SCOPE` 从 `src/CMakeLists.txt:305-307` 传出，且 `add_subdirectory(src)` 在 `add_subdirectory(tests)` 之前 |
| [L489-490](../Mooncake/mooncake-store/tests/CMakeLists.txt#L489-L490) | `add_store_test(kvclient_object_storage_adapter_test ...)` | 复用既有函数，它已 PUBLIC 链接 `mooncake_store`（含适配器 TU）、gtest、glog |
| [L491-492](../Mooncake/mooncake-store/tests/CMakeLists.txt#L491-L492) | `target_link_libraries(... PRIVATE Mooncake::kvclient Mooncake::xxhash)` | `Mooncake::kvclient` 提供 SDK 头路径与库；`Mooncake::xxhash` 供断言 key 编码（`XXH64`） |

位置在 `kvcs_object_storage_adapter_test` 之后（[L476-484](../Mooncake/mooncake-store/tests/CMakeLists.txt#L476-L484)），与 KVCS 的 SDK 条件写法并列。

与 KVCS 的差异：KVCS 测试**总是**创建目标，靠 `MOONCAKE_KVCS_TEST_NO_SDK` 宏切换断言；kvclient 测试整体包在 `if(SDK_FOUND)` 内，因为 `FakeKvClient` 继承 `kv::KvClient`，没有 SDK 头就无法编译。

---

## 6. 测试实现

文件：[`tests/kvclient_object_storage_adapter_test.cpp`](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp)（625 行，22 个用例）

### 6.1 基础设施

| 位置 | 内容 | 说明 |
| --- | --- | --- |
| [L19](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L19) | `constexpr uint64_t kHashSeed = 5381` | 与适配器 `kKvClientHashSeed` 同值，用于独立重算期望 key |
| [L21-23](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L21-L23) | `Injected(StatusCode)` | 构造带 message 的错误状态，避免每处重复 `Status::Error(code, "...")` |
| [L28-148](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L28-L148) | `class FakeKvClient final : public kv::KvClient` | 实现 SDK 的全部 9 个纯虚函数 |
| [L153-170](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L153-L170) | `struct Harness` | 持有 fake 指针 + 注入构造的适配器 |
| [L173-175](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L173-L175) | `RequireInitialized(harness)` | 每个用例开头断言 Init 成功 |
| [L177-185](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L177-L185) | `ExpectedKey(logical_key)` | 独立重算 XXH64 + 小端打包，与被测代码不共享实现 |

`FakeKvClient` 的关键设计：

| 行 | 内容 | 说明 |
| --- | --- | --- |
| [L32-35](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L32-L35) | `Shutdown` 计数 | 析构会调它 |
| [L37-67](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L37-L67) | 四个 `*Async` | 各自计数、记录入参（keys / entries），再委托 `StartTask` |
| [L69](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L69) | `Check` 恒 true | 适配器不调用它，只需满足纯虚 |
| [L71-81](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L71-L81) | `Wait` | 填 `result.status = task_status_`；`entryStatus` 用注入值，否则按提交条数填全 OK。**模拟 SDK 行为：adapter 拒绝条数不匹配，所以必须镜像提交数** |
| [L83-94](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L83-L94) | `RegisterRegions` | 按输入顺序返回单调递增 handle（从 1 开始）；`register_status_` 非 OK 时返回错误且**不填 out**——模拟 SDK 的全有或全无 |
| [L96-102](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L96-L102) | `UnregisterRegions` | 计数 + 累积所有 handle，可断言批量调用的内容与次数 |
| [L104-108](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L104-L108) | `TotalSdkCalls()` | 汇总所有调用计数，用于断言「零 SDK 调用」 |
| [L109-118](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L109-L118) | 可注入状态 | `task_status_` / `wait_status_` / `register_status_` / `unregister_status_` / 四个 `*_submit_status_` / `entry_status_` |
| [L120-135](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L120-L135) | 记录字段 | 各调用计数、捕获的 keys/entries/regions/handles |
| [L137-143](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L137-L143) | `StartTask` | 先记录 `pending_entries_`（即使提交失败也要记，因为 adapter 不会调 Wait），再按注入状态决定是否分配 task_id |

`Harness` 用 `std::unique_ptr` 而非直接持有适配器，因为适配器不可拷贝；`initialized` 字段把构造期 `Init()` 结果导出——**构造函数里的 `EXPECT_TRUE` 无法中止测试**，所以改由 `RequireInitialized` 在用例体内 `ASSERT_TRUE`。

### 6.2 用例清单

| # | 用例 | 行 | 断言要点 |
| --- | --- | --- | --- |
| 1 | `MakeCacheKeyIsStableAndLittleEndian` | [L187](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L187) | 经 `DeleteBatch` 捕获 key；同 key 两次编码相同；`[0]` 是 hash 低字节、`[7]` 是高字节（证明显式小端） |
| 2 | `RegisterBuffersRegistersAndReleases` | [L208](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L208) | 2 个请求 → 2 次 `RegisterRegions`（各自 addr/size 正确）、**1 次** `UnregisterRegions` 含 2 个 handle（批量）、handle 1/2 原样进入 `KVBuffer.buffer.handle`、`mrKey` 为空 |
| 3 | `RejectsEmptyKeyZeroLengthAndDuplicate` | [L245](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L245) | 6 个请求：空 key / 重复 key / 零长度 / 空指针 / 无 iov 各自 `INVALID_PARAMS`，仅第一条 `dup` 成功；只有 1 次注册与 1 个 entry |
| 4 | `RejectsNonContiguousSegments` | [L275](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L275) | 两块独立 `vector` 组成的 2-iov 请求 → `INVALID_PARAMS`，**零注册、零提交** |
| 5 | `AcceptsContiguousSegments` | [L291](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L291) | 同一 `vector` 切成 12+20 → 成功，`region.addr` = 首段基址、`region.size` = 32 |
| 6 | `RegisterFailureIsolatesEntry` | [L308](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L308) | `PARTIAL_FAILED` → 两项都 `DFS_SERVICE_UNAVAILABLE`，`store_calls_==0`、`unregister_calls_==0` |
| 7 | `RegisterFailureDoesNotClaimKey` | [L327](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L327) | 同 key 先失败后成功：证明被拒项**不占用** `seen`，第二次能写入 |
| 8 | `SingleEntryUsesBatchPath` | [L344](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L344) | `Put` → 1 次 `StoreAsync`/注册/注销，`region.size` 等于 value 长度；`PutV` 同理；空 `span` → `INVALID_PARAMS` 且 **SDK 调用总数不变** |
| 9 | `WriteTaskErrorMapsAll` | [L368](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L368) | `CONNECTION_ERROR` → 两项都 `DFS_SERVICE_UNAVAILABLE`；仍注销 2 个 handle |
| 10 | `WriteSubmitFailureKeepsGuardErrors` | [L389](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L389) | `RESOURCE_BUSY` → 参与项 `NO_AVAILABLE_HANDLE`，空 key 项**保持** `INVALID_PARAMS`（提交失败只覆盖参与项）；`wait_calls_==0` |
| 11 | `WriteEntryFailureKeepsOtherEntriesAndReleases` | [L407](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L407) | `PARTIAL_FAILED` + `[OK, ENTRY_RETRY_ADVISED]` → 第一项成功、第二项 `DFS_SERVICE_UNAVAILABLE`（可重试 ≠ 内部错误）；两个 handle 都注销 |
| 12 | `ReadSucceeds` | [L427](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L427) | `load_calls_==1`、`store_calls_==0`、`wait_calls_==1`；返回 N=24；`region.size`/`handle` 正确；注册 1 次、注销 1 个 |
| 13 | `ReadSubmitFailureLeavesBufferUntouched` | [L450](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L450) | `CONNECTION_ERROR` → `DFS_SERVICE_UNAVAILABLE`、`wait_calls_==0`、**buffer 内容逐字节未变**、handle 已注销 |
| 14 | `ReadTaskErrorMapsAll` | [L467](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L467) | `CQE_IO_TIMEOUT` → 两项都 `RPC_TIMEOUT`；2 个 handle 注销 |
| 15 | `ReadMissIsObjectNotFound` | [L485](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L485) | `PARTIAL_FAILED` + `ENTRY_KEY_NOT_FOUND` → `OBJECT_NOT_FOUND` |
| 16 | `ReadDataNotExistIsObjectNotFound` | [L502](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L502) | `ENTRY_DATA_NOT_EXIST` → `OBJECT_NOT_FOUND`（**守住 3.3 补的那行**） |
| 17 | `ExistsReturnsEntryCode` | [L517](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L517) | `ENTRY_KEY_EXIST` → `true`（OK task）；`ENTRY_KEY_NOT_EXIST` → `false`（**PARTIAL_FAILED task**，正是文档伪代码会漏掉的组合）；`register_calls_==0` 证明查询不注册 buffer |
| 18 | `ExistsErrors` | [L540](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L540) | 空 key → `INVALID_PARAMS` 且 **SDK 调用总数不变**；提交失败 → `DFS_SERVICE_UNAVAILABLE`；entry 级 `IO_ERROR` → `DFS_SERVICE_UNAVAILABLE` |
| 19 | `SingleEntryReadUsesBatchPath` | [L562](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L562) | `Get` → 1 次 `LoadAsync`、0 次 `StoreAsync`、1 次注册/1 个 handle 注销，`load_entries_[0].buffer.region.size` 等于目标 buffer 长度（证明 `Get` 真的走 `GetBatch`）；空目标指针 → `INVALID_PARAMS` 且 **SDK 调用总数不变** |
| 20 | `OverwriteDoesNotPreDelete` | [L585](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L585) | 同 key 两次 `PutBatch` → `store_calls_==2`、`delete_calls_==0` |
| 21 | `UninitializedBatchFailsSafely` | [L598](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L598) | 未 Init：`PutBatch`/`GetBatch`/`Exists` 都 `DFS_SERVICE_UNAVAILABLE`，`TotalSdkCalls()==0` |
| 22 | `UnregisterFailureIsLoggedNotFatal` | [L622](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L622) | `unregister_status_` 为错误 → 写仍成功、`unregister_calls_==1`（不重试）。运行时确实打出告警日志 |
| 23 | `ListKeysIsUnsupported` | [L638](../Mooncake/mooncake-store/tests/kvclient_object_storage_adapter_test.cpp#L638) | `NOT_SUPPORTED`；`GetName()` 返回 `"kvclient"` |

相比文档 §10.3 的 13 条，实际写了 23 条。新增的 10 条覆盖：连续性拒绝/接受（用例 4、5）、注册失败不占用 key（7）、写提交失败保留守卫错误（10）、写 entry 级 retry 码（11）、读 miss 的两个码（15、16）、`Get` 走批量（19）、`GetName`（23）。

---

## 7. 与设计文档的偏差

实现过程中发现文档有 5 处需要修正，均已改代码并按新写法同步文档：

| # | 文档原写法 | 实际做法 | 原因 |
| --- | --- | --- | --- |
| 1 | `Exists` 用裸 `Wait` + `if (!wait.ok()) return error` | 用 `waitForCompletion` + `!terminal` 判定 | `Wait` 返回 `result.status`，miss 时是 `PARTIAL_FAILED`，原写法会让 `false` 分支永远不可达 |
| 2 | `Exists` 校验 `queryResult->exists.size()` | 删除该校验 | SDK 的 `CompleteTransportTask` 已校验并转成 `INTERNAL_ERROR` 填进 `entryStatus` |
| 3 | 早退分支用 `LOG(FATAL)` | `LOG(ERROR)` + `throw std::invalid_argument` | 同函数既有校验一律用 throw，`master.cpp` 有对应 catch 惯例 |
| 4 | 无连续性校验；`ToErrorCode` 缺 `ENTRY_DATA_NOT_EXIST` / retry 码；写侧映射用一个独立函数 | 补连续性校验（3.7）、补三个映射（3.3）、把写侧 override 折进 `ToErrorCode(status, bool is_write)`（显式 bool、无默认值） | 见 3.7（blocker）与 3.3。独立函数的唯一好处是「强制每个调用点表态」，用无默认值的 bool 参数同样能做到，且少一份重复的 switch |
| 5 | `Get` 保持 `NOT_SUPPORTED` | `Get` → 单元素 `GetBatch`，取首项返回 | `Get` 的调用方期望落到 `GetBatch`；只有走批量协议才能拿到 miss 的 `ENTRY_*` entry 级状态（`useBatchIoForSingleKey` 已强制）。与 `Put` → `PutV` → `PutBatch` 对称 |

---

## 8. 未决风险

| 风险 | 现状 |
| --- | --- |
| **512 字节长度对齐** | SDK 的 `KvBatchStoreProtocol::ValidateRequest`（`kv_protocol.cc:541-552`）要求 `length % 512 == 0` 且 `<= 0xFFFFFF`，Retrieve 侧同样（`kv_protocol.cc:325-366`）。当前 `region.size` 是 `value_length` 原值，普通不等长 value 会被 SDK 拒绝；`>= 2^32` 时 `static_cast<uint32_t>` 会静默截断。方案（descriptor 对齐 / staging 补零 / 守卫）未定 |
| **超时后提前注销** | `Wait` 返回非终态时 SDK 不保证 I/O 已结束，但 `UnregisterBuffers` 仍会执行。SDK 未提供可等待的完成信号（再次 `Wait` 只得 `TASK_NOT_FOUND`），按 §2.1.1 决策接受 |
| **短读无法检测** | `kv::TaskResult` 只有 `status`/`entryStatus`/`queryResult`，无传输长度。成功项一律返回请求长度，后端 `FILE_READ_FAIL` 分支不可达；截断只能靠上层 `VerifyObjectChecksum` 发现。代码注释已如实说明 |
| **注销失败残留** | SDK 在 provider 部分失败时保留 `registeredRegions_` 条目，适配器无 handle 可重试，只记 `LOG(WARNING)`；残留由 `Shutdown()` 兜底 |
| **端到端写依赖显式 `dfs_replica_num=1`** | PR2 不改 `AttachHostId`（PR3），否则只写内存副本 |
| **`EvictDiskReplica` / `BatchEvictDiskReplica` 的 DFS 分支不可用** | 传 `nullptr`，DFS 仍被拒；PR2 明确不改，只验证 `EvictDiskReplicaIfCurrent` |
| **`CheckHealth` 未实现** | `MOONCAKE_DISTRIBUTED_HEALTH_CHECK=1` 时 Init 失败；部署必须关闭 |

---

## 9. 提交状态

| 仓库 | 状态 |
| --- | --- |
| `Mooncake-documents`（`origin/main`） | 设计文档与 PR2 步骤文档的 §2.6.1-§2.6.3、§3-§7 已同步并推送（`fc8c68c`、`e47ccb1`、`23a2413`）。本轮修订——`ToErrorCode(status, is_write)` 合并、`Get` → `GetBatch` 包装、23 条测试、全部行号重算——已推送至 `origin/main` |
| `Mooncake`（`origin/PR1`） | `8af2fc10 [Store] Keep single-key kvclient IO on the batch protocol` 已推送 |
| `Mooncake`（`Mooncake_l3_pr2`） | **未提交**。工作区 5 个改动文件 + 1 个新增测试文件 |
