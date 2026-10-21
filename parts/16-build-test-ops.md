# 16 构建、测试与运维

> 本章对应仓库 `D:/xiaojun/work/Mooncake`（main 分支）。所有 `路径:行号` 均相对于仓库根目录，可直接跳转核对。
> 阅读目标：能独立回答"某个 `-DUSE_*` 打开后到底发生了什么""这个 PR 为什么只跑了一半 job""`/metrics` 的数字是累计、速率还是瞬时值"。

---

## 定位与职责

Mooncake 横跨多种硬件后端（NVIDIA CUDA、AMD HIP、MUSA、MACA、Ascend、Hygon DTK、CoreX、SUPA、MLU、EFA/CXI/SHCA、CXL、RISC-V）与多个组件（Transfer Engine、Store、Conductor、EP/PG、P2P Store、Rust-native Store），构建/测试/运维层承担四件事：

1. **把 40+ 个编译开关收敛成可组合、可校验的选项空间。**
   顶层 `CMakeLists.txt:19-45` 定义组件开关，`mooncake-common/common.cmake:79-133` 定义硬件/传输开关，两者通过 `add_compile_definitions` 注入源码。互斥关系（两个 Go c-shared 后端不能共存）在 configure 阶段就 `FATAL_ERROR`（`CMakeLists.txt:79-95`），而不是等到链接或运行期。

2. **把"能编译"变成"能在 CI 复现、能发 wheel"。**
   CI 跑的与发布完全同一条链路：`_build-wheel.yaml` → `scripts/build_wheel.sh` → auditwheel（`ci.yml:35-36` 注释明说 Ubuntu job 是"消费侧测试，不是第二套构建环境"）。因此 wheel 脚本里的 variant 表就是发布配方的单一来源。

3. **把测试分成成本与触发条件不同的四层**：
   C++ gtest/ctest、wheel 安装后的 Python 集成、CI 冒烟套件、外部 GPU 集群 T-one E2E；另有两类元测试（CI 脚本自身的单测、发布门）。

4. **把运行时状态暴露出去**：master 进程内嵌 `coro_http` 管理服务（外部无 exporter），默认 `0.0.0.0:9003`，提供 `/metrics`、`/health`、`/role`、`/ha_status` 等；`monitoring/` 提供 Prometheus+Grafana 消费栈。

边界是单向的：构建层依赖所有模块的目录结构与产物名，运行时模块不依赖构建层。唯一反向渗透的是指标代码（`MasterMetricManager`、`MasterAdminServer` 位于 `mooncake-store/src`，属于 Store 模块），本章只从"运维接口"角度讲它们。

---

## 目录结构速览

| 路径 | 职责 |
| --- | --- |
| `CMakeLists.txt` | 顶层入口：组件 `WITH_*`、互斥校验、条件调用依赖 provider、按组件 `add_subdirectory` |
| `mooncake-common/common.cmake` | 全局编译基线（C++20、警告、debug 符号、ASan、sccache）、全部 `USE_*`、YLT 引入 |
| `mooncake-common/limit_jobs.cmake` | 基于内存的编译/链接 job pool 限流；只对 Ninja 生效 |
| `mooncake-common/FindYLT.cmake` | `yalantinglibs` FetchContent 固定 commit + SHA256；`GH_MIRROR` 拼接 |
| `mooncake-common/FindGTest.cmake` | googletest 1.17.0 FetchContent，强制 `BUILD_GMOCK=OFF`、`INSTALL_GTEST=OFF` |
| `cmake/Dependencies.cmake` | zstd/xxhash/liburing/libzmq/hiredis → `Mooncake::*` IMPORTED 目标 |
| `pyproject.toml`（根） | scikit-build-core 轨道（版本 `0.3.12.post1`） |
| `mooncake-wheel/pyproject.toml` | legacy setuptools 轨道（版本 `0.3.13`），CI/发布实际使用 |
| `scripts/build_wheel.sh` | legacy wheel 打包主流程（变体 sed、.so 收集、auditwheel、EP/PG 注入） |
| `scripts/test_installation.sh` | 干净 venv 安装 wheel + import 结构 + 入口点验证 |
| `scripts/run_tests.sh` | wheel 级集成测试编排（metadata server 双模式、SSD/promotion/CXL 场景） |
| `scripts/ci/` | 冒烟套件与发布门（`services.sh`、`run_*_smoke.sh`、`testpypi_wheel_gate.py`） |
| `scripts/ci/tests/` | CI 脚本自身的单元测试（"测测试"） |
| `scripts/code_format.sh` | clang-format 封装：`--all`、`--check`、`--staged`、`--changed-lines` |
| `.github/workflows/ci.yml` | 主 CI：路径门控 + 17 个 job + `ci-gate` 汇总 |
| `.github/workflows/_build-wheel.yaml` | 可复用 wheel 构建矩阵（variant × arch × python） |
| `.github/workflows/nightly.yml` | 夜间：版本戳、多 variant、TestPyPI（twine，非 24 目标门）、覆盖率/ASan、镜像、gate |
| `.github/workflows/pre-release.yaml` | 预发布：tag 归一化、wheel 矩阵、24 目标发布门、TestPyPI 上传状态校验与消费侧验证 |
| `.github/actions/run-ctest/` | 统一 ctest 环境（端口预留、metadata env、JUnit） |
| `.pre-commit-config.yaml` | ruff、codespell、cmake-format、按 staged 行的 clang-format、Store-RS cargo |
| `docker/master.Dockerfile` | 运行时 master 镜像（pip 装发布 wheel、非 root、tini） |
| `docker/mooncake.Dockerfile` | 从源码构建 wheel 的镜像 |
| `monitoring/` | Prometheus 2.54.0 + Grafana 11.0.0 与 Grafana provisioning |
| `benchmarks/` | `storage_benchmark_v1`（Python 基准框架）、`xypd_benchmarks`（vLLM PD 场景） |
| `docs/source/getting_started/{build,observability}.md` | 构建选项与指标语义的权威文档 |

---

## 核心数据结构

### 1. `MetricsBootstrapConfig` —— 管理端口的默认值单一来源

```cpp
// mooncake-store/include/config/metrics_bootstrap_config.h:9-17
struct MetricsBootstrapConfig {
    static constexpr bool kDefaultEnabled = true;
    static constexpr uint32_t kDefaultPort = 9003;
    static constexpr std::string_view kDefaultHost = "0.0.0.0";
    bool enabled = kDefaultEnabled;
    uint32_t port = kDefaultPort;
    std::string host{kDefaultHost};
};
```

多处引用同一符号（至少：gflags 定义 `master.cpp:135-143`、`MasterConfig` 成员 `master_config.h:96/226`、配置文件解析、测试 `mooncake-store/tests/master_service_config_test.cpp:52-104` 校验 0/65535 边界与非法值）。如果 flag 默认值、配置文件默认值、文档里的 9003 各写一份很容易漂移。注意 `port` 是 `uint32_t`，最终要转 `uint16_t` 传给 socket，所以解析层必须做范围校验。

### 2. 配置解析：文件 → 命令行覆盖的顺序契约

`ResolveMetricsBootstrapConfig`（`mooncake-store/src/config/metrics_bootstrap_config_loader.cpp:10-41`）先读文件，再用命令行覆盖，最后校验：

```cpp
// metrics_bootstrap_config_loader.cpp:15-35（节选）
if (file_config != nullptr) { ... GetBool/GetUInt32/GetString ... }
if (command_line.enabled.has_value()) result.enabled = *command_line.enabled;
if (command_line.port.has_value())    result.port = *command_line.port;
if (command_line.host.has_value())    result.host = *command_line.host;
if (result.port > std::numeric_limits<uint16_t>::max()) throw std::invalid_argument(...);
```

关键在覆盖侧：`master.cpp:474-491` 用 `google::GetCommandLineFlagInfo(..., &info)`，**只有 `!info.is_default` 才把 flag 塞进 overrides**。也就是说没显式写 `--metrics_port` 时，flag 默认值不会压掉配置文件。直接用 `FLAGS_x` 是这类项目常见的错法。

### 3. `MasterAdminServer` —— 内嵌 HTTP 服务与它的两把互斥锁

```cpp
// mooncake-store/include/master_admin_service.h:121-141
uint16_t http_port_;
std::string http_host_;
bool enable_metric_reporting_ = false;
coro_http::coro_http_server http_server_;     // 4 个工作线程
std::thread metric_report_thread_;
std::atomic<bool> metric_report_running_{false};
std::binary_semaphore metric_report_stop_sem_{0};
std::atomic<bool> started_{false};
std::shared_ptr<std::atomic<bool>> dfs_expansion_running_;  // 跨请求存活
mutable std::mutex storage_metrics_refresh_mutex_;
mutable std::mutex state_mutex_;
ha::MasterRuntimeState state_{kStarting};
std::optional<ha::MasterView> leader_view_;
std::shared_ptr<WrappedMasterService> service_;
bool service_available_ = false;
```

设计要点：

- **`http_server_` 在构造函数里以 4 线程构造**（`master_admin_service.cpp:264-270`），`Start()` 只做 `RegisterHandler()` + `async_start()` + 可选日志线程。所以绑定失败在 `Start()` 返回 false 时暴露。
- **`RuntimeSnapshot`**（`master_admin_service.h:46-51`）把 `state / leader_view / service / service_available` 打包，`SnapshotState()` 在 `state_mutex_` 下整体拷贝（`:380-388`）。HTTP handler 只拿快照，不长期持锁，避免慢请求阻塞 HA 状态切换。
- **两把锁的获取顺序固定**：`SetServiceDelegate`（`:350-365`）与 `SetServiceAvailable`（`:367-378`）都是"先 `storage_metrics_refresh_mutex_`，再 `state_mutex_`"；`RefreshStorageMetrics`（`:560-567`）同序。头文件注释（`:133-135`）解释原因：防止"采样到旧 leader 的 storage 快照"在 service 下线之后才发布。反序会死锁或发布陈旧数据。
- **`dfs_expansion_running_` 是 `shared_ptr<atomic<bool>>`**：异步 handler 可能比 request 活得久，shared_ptr 保证客户端断开、handler 仍持引用时标志不悬空；注释说明"同一时刻只允许一个 expansion pending"。

### 4. `MasterMetricManager` 与 `basic_hybrid_counter`

`MasterMetricManager` 是进程级单例（`mooncake-store/include/master_metric_manager.h:21-29`，拷贝/移动全部 delete），指标成员是 `ylt::metric` 的 counter/gauge/histogram。

`basic_hybrid_counter`（`mooncake-store/include/hybrid_metric.h:11-219`）是 Mooncake 对 ylt 的扩展：

- 存储是 `dynamic_metric_impl<std::atomic<value_type>, N>`，即"标签元组 → 原子值"映射；`inc()` 用 `try_emplace` 后 `detail::inc_impl`（`:24-26`），读取用 `memory_order::relaxed`（`:33-39`、`:182`）。Mooncake 这一层不再额外加锁；标签表 `try_emplace/find` 内部是否加锁取决于 ylt 的 `dynamic_metric_impl`（由 FetchContent 拉取，不在本仓库），不能据此断言热点路径完全无锁。
- `serialize()`（`:120-132`）先 `Base::copy()` 拿到标签快照再遍历，因此 Mooncake 侧渲染时不直接操作共享标签表（`copy()` 内部是否加锁同样由 ylt 决定）。
- `basic_hybrid_gauge` 只把 `MetricType` 换成 Gauge（`:221-234`），注释解释：否则 Prometheus 会被告知一个可能下降的序列是 monotonic counter。
- 静态标签在构造期预格式化成 `static_labels_str_`（`:170-176`），序列化直接 append，省掉每行重复拼接。

**一个易误解的枚举**：`CacheHitStat`（`master_metric_manager.h:72-90`）通过别名改名而不改值：

```cpp
MEMORY_CURRENT_CACHED_OBJECTS = MEMORY_TOTAL,
MEMORY_HITS_PER_CURRENT_CACHED_OBJECT = MEMORY_HIT_RATE,
```

原因是 RPC/API 的枚举整数值必须保持兼容。更关键的是语义：`calculate_cache_stats()`（`master_metric_manager.cpp:2233-2295`）里 `MEMORY_HIT_RATE = 累计命中数 / 当前缓存对象数`；头文件注释（`master_metric_manager.h:85-86`）说明该值可能超过 1.0，`.cpp:2247-2249` 的注释进一步说明它不是请求/token 级的命中率。

### 5. `SummaryCounters` 与 summary 快照

周期日志的"每秒速率"不是直接读计数器，而是与上次快照求差：

- `SummaryCounters` 保存所有请求/批量/逐出字段的**累计值**；
- `summary_snapshot_`（结构体声明于 `master_metric_manager.h:563-567`，读写于 `master_metric_manager.cpp:2564-2582`）保存 `{initialized, timestamp, counters}`，由 `summary_snapshot_mutex_` 保护；
- `get_summary_string(update)`：只有 `true` 才推进快照。`get_summary_string_and_update_snapshot()` 由周期日志线程调用；`get_summary_string()`（HTTP `/metrics/summary`）不推进。

推论：**`/metrics/summary` 的速率窗口是"上次周期日志刷新至今"**；若 `enable_metric_reporting=false`，快照从未初始化，`has_previous_summary=false`，所有速率显示 0（`:2584-2596`）。逐出类字段用 delta 语义，`delta()` 把负差钳成 0（`:2584-2590`），计数器重置/回绕不会出现负速率。

### 6. `limit_jobs` 的内存模型

| 变量 | 默认 | 含义 |
| --- | --- | --- |
| `MAX_COMPILER_MEMORY_MB` | 1500 | 单个编译 job 峰值内存估算（`limit_jobs.cmake:14-15`） |
| `MAX_LINKER_MEMORY_MB` | 4000 | 单个链接 job 峰值内存估算（`:16-17`） |

计算（`:33-58`）：`可用物理内存 / 每 job 估算`，clamp 到 `[1, nproc]`；GitHub 托管 runner 再统一封顶 3（`:60-72`）。最终通过 Ninja `JOB_POOLS` 生效（`:87-95`）。对 Make 只打印建议（`:96-101`）——这是"wheel 构建必须 `-G Ninja`"的根因。

### 7. CI 服务注册表（`services.sh`）

```bash
# scripts/ci/services.sh:5-7
declare -A CI_SERVICE_PIDS=()    # name -> pid
declare -A CI_SERVICE_LOGS=()    # name -> log path
declare -a CI_SERVICE_ORDER=()   # 启动顺序
```

没有类、没有 systemd，但提供了完整契约：前台启动（PID 归套件所有）、端口就绪等待、失败注解、逆序清理。细节见"流程四"。

### 8. 发布门矩阵（`testpypi_wheel_gate.py`）

```python
# scripts/ci/testpypi_wheel_gate.py:28-35
CORE_PACKAGES = ("mooncake-transfer-engine",
                 "mooncake-transfer-engine-cuda13",
                 "mooncake-transfer-engine-non-cuda")
PYTHON_TAGS = ("cp310", "cp311", "cp312", "cp313")
ARCHITECTURES = ("x86_64", "aarch64")
```

`expected_targets()`（`:54-61`）生成 3×4×2 = 24 个目标；`_target_from_wheel()`（`:77-101`）要求每个文件名**恰好**解析出一个期望目标，版本必须等于 tag 推出的预发布版本；`validate_wheel_set()`（`:104-125`）再检查缺失/多余/重复。这个门由 `pre-release.yaml` 调用（`normalize-version` 归一化 tag、`validate-upload-state` 上传前校验、`wait-upload-state` 上传后逐个比对哈希），并有一套跑在 `ci-script-tests` 里的单测（`mooncake-wheel/tests/test_testpypi_wheel_gate.py`）。

---

## 关键流程剖析

### 流程一：从裸机到 wheel —— 完整构建链

```
dependencies.sh
  └─ detect_os → apt/yum/dnf 安装包列表
  └─ git submodule update --init --recursive
  └─ install_go
cmake -G Ninja .. -D<variants>
  ├─ CMakeLists.txt: include(FindJsonCpp/FindGLOG/common.cmake)
  ├─ option(WITH_*) / option(USE_*) + 互斥校验
  ├─ include(cmake/Dependencies.cmake) → mooncake_provide_*()
  ├─ common.cmake 内部：include(limit_jobs) / include(FindGTest)（BUILD_UNIT_TESTS 时）/ include(FindYLT)
  ├─ add_subdirectory(mooncake-common) → 公共库、etcd / k8s-lease wrapper
  └─ add_subdirectory(mooncake-transfer-engine / mooncake-store / ...)
cmake --build . ; cmake --install .
scripts/build_wheel.sh
  ├─ 拷贝 .so / 二进制到 mooncake-wheel/mooncake/
  ├─ 暂存 python/mooncake/*.py、reshard、store facade
  ├─ sed 改写 mooncake-wheel/pyproject.toml（按 variant）
  ├─ python -m build --wheel
  ├─ auditwheel repair --exclude ...
  └─ 注入 EP/PG device .so → wheel pack
```

**1) 依赖安装。** `dependencies.sh:94-96` 强制 root；`detect_os` 从 `/etc/os-release` 读 `ID/VERSION_ID`（`:65-92`）。按三大族分支：

- Ubuntu/Debian（`:165-216`）：`libibverbs-dev`、glog/jsoncpp/unwind/numa/boost/grpc/protobuf/curl/hiredis/uring/jemalloc/msgpack/zmq/zstd/asio/xxhash、`patchelf`。
- RHEL 系（`:218-255`）与 Kylin（`:257-309`）用 yum/dnf。

三个易错点都写在脚本里：

- **SHCA 冲突**：检测到 `shca-tools` 时删掉 `libibverbs-dev`（Ubuntu，`:199-206`）或 `rdma-core-devel`（RHEL，`:248-252`），因为 SHCA 自带 ibverbs 头/库；`libboost-all-dev` 会拉 OpenMPI/libfabric 进而依赖发行版 ibverbs，所以 Ubuntu 上还把它降级为 `libboost-dev`。
- **msgpack 过渡包**：Ubuntu 26.04 起 `libmsgpack-dev` 变成 C-only 过渡包，需要补 `libmsgpack-cxx-dev`（`:208-213`），用 `apt-cache show` 探测。
- **Kylin 外部 EPKL 仓库**：从 `nkvers` 解析版本与 release 拼 `KYLIN_EPKL_URL`，用 `dnf --repofrompath` 临时启用（`:286-308`）；解析失败直接报错，不猜。

随后初始化子模块（`:316-335`；**全项目只有 pybind11 一个 submodule**），安装 Go 1.26.9（`:351-415`，三镜像依次回退；只有回退到国内镜像时才设置 `GOPROXY`，`:424-435`）。`--with-spdk` 会 clone SPDK `v23.01.1`、`./configure --with-rdma`、`make install`，并把 DPDK 静态库拷到 `/usr/local/lib`（`:438-503`）。

**2) 配置阶段的选项装配。** 顶层先 include 三个模块（`CMakeLists.txt:11-13`），定义组件开关，再 include `cmake/Dependencies.cmake`（`:108`）拿到 `mooncake_provide_*`，然后**按条件调用**：

```cmake
# CMakeLists.txt:109-123
if(WITH_STORE) mooncake_provide_zstd(); mooncake_provide_xxhash() endif()
if(WITH_CONDUCTOR OR (WITH_STORE AND ENABLE_KV_EVENTS)) mooncake_provide_libzmq() endif()
if(WITH_STORE OR (WITH_TE AND USE_TENT)) mooncake_provide_liburing() endif()
if((WITH_TE AND USE_REDIS) OR (WITH_STORE AND STORE_USE_REDIS)) mooncake_provide_hiredis(REQUIRED)
elseif(WITH_TE AND USE_TENT) mooncake_provide_hiredis() endif()
```

`Dependencies.cmake:1-2` 的注释点明所有权模型：**由构建入口调用 provider，每个 provider 最多调用一次；消费者目录只 link `Mooncake::*`**。这避免同一 IMPORTED 目标被多个子目录重复 `add_library` 报错，也避免"某子目录偷偷 find_library"的隐式依赖。

`mooncake_provide_hiredis` 支持 `REQUIRED`（`:56-74`）：Redis 元数据服务是硬依赖，TENT 场景是软依赖（找不到就不定义目标，消费者自行判断）。

**3) 编译基线与硬件开关。** `common.cmake` 前 130 行决定"所有目标怎么编译"：

- `C++20` / `C99` / `CUDA C++20`（`:1-3`）。
- 默认 `-Wall -Wextra -Wno-unused-parameter -fPIC`；`ENABLE_DEBUG_SYMBOLS=ON`（默认）追加 `-g`（`:5-14`）。
- GCC 追加 `-fcoroutines`（`:16-17`）；Clang 追加 `-Werror=thread-safety`（`:28-30`）。
- **未指定 `CMAKE_BUILD_TYPE` 时默认 `RelWithDebInfo`**（`:41-44`）。
- `ENABLE_ASAN` 同时加 `-fsanitize=leak` 与 `-fsanitize=address`（`:32-39`）。
- `ENABLE_SCCACHE` 找不到 sccache 直接 `FATAL_ERROR`（`:56-67`），然后把 `CMAKE_{C,CXX}_COMPILER_LAUNCHER` 指向它。
- `USE_TCP` 默认 ON（`:105`），`USE_HTTP` 默认 ON（`:185`），`WITH_METRICS` 默认 ON（`:188`）。
- `USE_TPU` / `USE_XPU` 强制要求 `USE_TENT=ON`（`:315-355`）：否则 configure 会"成功但一行代码都不编"。

**4) 产物与打包。** `scripts/build_wheel.sh` 第一步是把 CMake 产物搬进 Python 包目录（`:33-148`）：`engine.*.so`、`_store.*.so`、`mooncake_master`/`mooncake_client`、`libasio.so`、`libmooncake_store.so`、`libtransfer_engine.so`、etcd/k8s-lease wrapper、`nvlink_allocator.so`、`ascend_transport*.so` 等。可选产物（`_store.*.so`、`libmooncake_store.so`、etcd/k8s-lease wrapper、`nvlink_allocator.so`、`ascend_transport*.so`）都用 `[ -f ... ]` / `compgen -G` 判存在，缺失就打印 "likely ... disabled"；`engine.*.so`、`libasio.so` 等核心产物是无条件 `cp`，缺了会直接让脚本失败。

接着暂存迁移中的 Python 模块（`:190-241`），并注册 EXIT trap 清理（`:217-232`、`:252-258`）：把 `python/mooncake/*.py` 拷进 `mooncake-wheel/mooncake/`，构建后删除，保证源树是唯一权威副本。

变体处理是最"脆"的部分：`BUILD_VARIANTS`（`:260`）里八个环境变量最多只能设一个（`:262-270`）；然后按变体 `sed` 改包名/描述/keywords/classifiers（`:282-365`）。这就是 `mooncake-wheel/pyproject.toml:6-12` 那条醒目注释的由来：**`keywords` 与 `classifiers` 必须是单行数组**，否则 `sed 's/^keywords = \[\(.*\)\]$/.../'` 静默匹配不到——包名那一行仍会被改（变体之间不会同名），但 keywords/classifiers 的变体标注会全部丢失，发布元数据与预期不符。

平台标签由 glibc 探测决定（`:411-463`）：`getconf GNU_LIBC_VERSION` → 失败退回 `ldd --version` → 再失败用 `2_17`，拼成 `manylinux_${ver}_${arch}`。随后 `auditwheel repair` 带一长串 `--exclude`（`:487-581`）：既排除系统库（libc/libstdc++/libnuma/libibverbs…），也排除 GPU 运行时（libcuda/libnccl/libamdhip64…）与不能碰的 fatbin 载体（`libmooncake_ep_device.so`、`libmooncake_pg_device.so`、`libmpcomm.so`）。

EP/PG device 库在 auditwheel **之后**注入（`:583-614`）：`wheel unpack` → 拷 `.so` → 给 host 扩展补 `-rpath '$ORIGIN'` → `wheel pack`。注释说得很清楚：patchelf 会破坏 CUDA fatbin，运行期表现为 `cudaErrorInvalidKernelImage`。

### 流程二：一次 PR 的 CI 之旅 —— 路径门控与 gate 汇总

```
pull_request(opened|synchronize|reopened)
  └─ job check-paths（dorny/paths-filter）→ outputs: src / tent / docs
  ├─ spell-check / clang-format / python-lint / reshard-type-check /
  │  ci-script-tests / tone-script-tests        （无条件）
  ├─ build-wheel            if src  → _build-wheel.yaml（py3.10/3.12）
  │    └─ test-wheel-ubuntu（ubuntu-22.04+py3.12 / 24.04+py3.10）
  ├─ unit-tests             if src  → cmake + ctest
  ├─ build-flags            if src  → TE-only 构建
  ├─ build-wheel-cu13       if src
  ├─ build-wheel-cu13-arm64 if src
  ├─ build-wheel-efa        if src  → ci_efa.yml
  ├─ build-wheel-rocm       if src  → ci_rocm.yml
  ├─ docs-check             if docs
  ├─ tent-ci                if tent → 5 个矩阵腿
  └─ ci-gate（汇总）
```

**1) 触发与并发。** `ci.yml:3-17` 的 `pull_request.types` 故意不含 `labeled`，注释（`:12-15`）解释：自动打标签器会加 `run-ci`；若监听 `labeled`，任何新标签（包括 `run-e2e-ci`）都会重触发整个 workflow 并取消进行中的 run。同 SHA 重触发走 `ci-on-label.yml`（`pull_request_target`，只用 GitHub API，不 checkout PR 代码）。

并发组是 `workflow + (PR number | ref)`，但 `cancel-in-progress` 只在非 push 时为真（`:25-27`）。注释（`:22-24`）：main 上的完整构建比两次合并的间隔还长，每次 push 都取消会让 main 永远没有"完成"结果；新 push 仍替换 pending 的那个。

**2) check-paths 的三组过滤器**（`:534-581`）：

```yaml
src:  mooncake-*/**, extern/**, cmake/**, CMakeLists.txt, dependencies.sh,
      scripts/**, .github/actions/**, .github/workflows/**
tent: mooncake-transfer-engine/**, mooncake-store/src/transfer_task.cpp,
      mooncake-store/tests/transfer_task_test.cpp, mooncake-store/tests/CMakeLists.txt,
      mooncake-common/**, extern/**, cmake/**, CMakeLists.txt, dependencies.sh,
      scripts/ci/run_transfer_engine_rust_smoke.sh, .github/actions/**,
      .github/workflows/ci.yml
docs: docs/**, .github/workflows/ci.yml
```

三点注意：`src` 的 `mooncake-*/**` 覆盖所有模块源码；`tent` 是 `src` 的子集但包含 `mooncake-common/**`（YLT/limit_jobs 变化影响 TENT）；`workflow_dispatch` 时三个 output 被强制为 `true`（`:539-542`），手动跑就是全量。

**3) gate 的语义。** `ci-gate`（`:802-872`）不是简单的 `needs.*.result == success`：

- `if: ${{ (github.event_name != 'push' && always()) || !cancelled() }}`（`:807`）。push 被取消时 gate 不产生结果而非报失败（注释 `:804-806`：让 main 上已开始的构建跑完）；PR 上必须 fail，因为"skipped 的 gate"会满足 required check。
- 三段校验：**无条件 job** 必须 `success`（`:830-843`，含 `check-paths` 自己）；**source job** 期望值是 `src == 'true' ? 'success' : 'skipped'`，逐个比较（`:845-860`）；**TENT / docs** 同理（`:862-872`）。
- 关键不变量：**失败的 selector 不会把 skipped 变成绿色**。若 `check-paths` 挂了，`needs.check-paths.outputs.src` 是空串，`EXPECTED` 变成 `skipped`，但第一段已因 `check-paths` 非 success 而失败。

**4) 产物如何流到测试。** `build-wheel` 调用 `_build-wheel.yaml`，artifact 名如 `mooncake-wheel-ci-py312`；`test-wheel-ubuntu` 用 `actions/download-artifact` 下到 `mooncake-wheel/dist`（`:68-80`），再跑 `scripts/test_installation.sh`。

**5) `unit-tests` 的 configure 行**（`:221-233`）：

```
cmake -S . -B build -G Ninja
  -DUSE_HTTP=ON -DUSE_CXL=ON -DUSE_UB=ON -DUSE_ETCD=ON -DUSE_CUDA=ON
  -DUSE_MNNVL=ON -DWITH_P2P_STORE=ON -DWITH_CONDUCTOR=ON
  -DSTORE_USE_ETCD=ON -DCMAKE_BUILD_TYPE=Release
  -DBUILD_UNIT_TESTS=ON -DENABLE_SCCACHE=ON
```

显式 `Release` 覆盖 `RelWithDebInfo` 默认值；`cmake --build build --parallel 128`（`:241`）与 `limit_jobs` 不冲突——实际并行度由 Ninja job pool 二次约束，`--parallel` 只是上限。

**6) `build-flags`**（`:275-347`）在 `mooncake-transfer-engine` 子目录单独 configure，只验证 TE 能独立构建；把 `CMAKE_{C,CXX}_FLAGS_RELWITHDEBINFO` 显式设成 `-O2 -DNDEBUG`（`:283`、`:338-339`），并关闭 `USE_ETCD/USE_CUDA/USE_MNNVL/USE_UB`。这是"组件解耦"的回归测试。

### 流程三：一次 `/metrics` 抓取从头到尾发生了什么

```
Prometheus GET /metrics
  └─ coro_http_server 工作线程
       └─ MasterAdminServer::HandleMetrics            (master_admin_service.cpp:580-584)
            ├─ resp.add_header("Content-Type","text/plain; version=0.0.4")
            └─ BuildMetricsText()                     (:390-400)
                 ├─ RefreshStorageMetrics()           (:560-567)
                 │    ├─ lock(storage_metrics_refresh_mutex_)
                 │    ├─ SnapshotState() → lock(state_mutex_)
                 │    ├─ service->GetStorageUsageSnapshot() 或空快照
                 │    └─ MasterMetricManager::project_storage_usage(...)
                 ├─ MasterMetricManager::serialize_metrics()
                 ├─ HAMetricManager::serialize_metrics()
                 └─ BuildTenantQuotaMetricsText()      (:402-474)
```

**1) 服务怎么起来的。** 非 HA：`master.cpp:1723-1733` 构造 `MasterAdminServer(port, enabled, host)`，`Start()` 失败直接 `return 1`（不会"降级无指标运行"）；然后 `SetRuntimeState(kServing)`、`SetServiceDelegate(...)`、`SetServiceAvailable(true)`。HA：`master_service_supervisor.cpp:1042-1053` 用同一配置构造，交给 `RunSupervisorLoop`，随角色变化调用 `SetRuntimeState` / `ActivateServingState` / `DeactivateServingState`（`:325-418`）。嵌入模式（`embedded_master.cpp:70-80`）固定 `enable_metric_reporting=false`、`host=127.0.0.1`，用于进程内 E2E。

**2) 路由表。** `RegisterHandler()`（`master_admin_service.cpp:1367-1489`）注册 20+ 条，运维核心：

| 路由 | 方法 | 内容 |
| --- | --- | --- |
| `/metrics` | GET | Prometheus 文本（`BuildMetricsText`） |
| `/metrics/summary` | GET | 人类可读摘要（与周期日志同源） |
| `/health` | GET | JSON：`status/role/ha_state/service_ready/leader_address/view_version` |
| `/version` | GET | JSON：`version`（RPC 兼容）与 `display_version`（含 git hash） |
| `/role` | GET | 纯文本 `leader`/`standby` |
| `/ha_status` | GET | 纯文本 HA runtime state |
| `/kv_events/status` | GET | KV events 发布器状态 |
| `/api/v1/drain_jobs*` | POST/GET | drain 作业创建/查询/取消 |
| `/api/v1/tenant_quotas` | GET/PUT/DEL | 租户配额 |
| `/api/v1/remove_all` | POST | 批量删除 |

`HandleMetrics` 只设置 `Content-Type: text/plain; version=0.0.4` 和内容（`:580-584`），没有分页/gzip/缓存——每次抓取重新渲染全量。

**3) 存储指标为什么要"投影"。** `project_storage_usage` 的声明注释（`master_metric_manager.h:64-70`）写得很重：

> This is an observability projection. Business code must use the domain snapshots directly and must not read the resulting gauges.

即 `mem_allocated_size_` 这类 gauge 是"从权威 allocator 快照刷出的镜像"，不是业务状态源。`RefreshStorageMetrics` 先取 service 快照再投影，service 不可用时投影空快照，保证 leader 切换后旧数字不残留。

**4) 序列化的两个细节。**

- `serialize_metrics()`（`master_metric_manager.cpp:2014-2031`）开头注释：**值为 0 且未变化的指标不会出现在输出里**；注释还称"全为 0 时返回空串"，但构造函数把 `build_info_` 置为 1 并在 `:2228` 无条件序列化，所以实际输出至少含 `mooncake_build_info` 一行，"完全空串"在当前代码里不会发生。对单个指标而言，"指标消失"通常意味着值确实是 0，而不是注册丢了。构造函数（`:549`）调用 `update_metrics_for_zero_output()`，其中对 gauge 调 `update(0)`、对 counter 调 `inc(0)`（例如 `:696-699`），把每个序列标记为"已变更"一次，保证首轮序列化就能看到它们。
- `basic_hybrid_counter::serialize()` 先 `Base::serialize_head()` 输出 `# HELP`/`# TYPE`，再由 `serialize_map()` 输出样本行（`hybrid_metric.h:120-132`、`:178-196`）；标签顺序"静态在前、动态在后"（`:205-214`）。

**5) 周期日志线程。** `Start()` 在 `enable_metric_reporting_` 为真时创建线程（`master_admin_service.cpp:289-322`）：

```cpp
while (metric_report_running_.load()) {
    RefreshStorageMetrics();
    const auto snapshot = SnapshotState();
    LOG(INFO) << "Master Admin Metrics: role=..., state=..., service_ready=..., "
                 "master={" << ...get_summary_string_and_update_snapshot() << "}, "
                 "ha={" << HAMetricManager::instance().get_summary_string() << "}";
    if (metric_report_stop_sem_.try_acquire_for(std::chrono::seconds(kMetricReportIntervalSeconds)))
        break;
}
```

用 `binary_semaphore::try_acquire_for` 而非 `sleep_for`，让 `Stop()` 能立刻唤醒退出；`Stop()`（`:328-337`）先置 `metric_report_running_=false`（必要时 `http_server_.stop()`），若日志线程可 join 则 `release()` 信号量再 `join()`。`kMetricReportIntervalSeconds = 10`（`:35`）是编译期常量，文档说的 "configurable via kMetricReportIntervalSeconds" 实际是"改源码重编"，不是运行期参数。

**6) `enable_metric_reporting=false` 关了什么。** `RegisterHandler()` 与 `async_start()` 在判断 `enable_metric_reporting_` **之前**（`:279-289`），所以 HTTP 端点照常可用，只是没有 10 秒周期日志。`observability.md:171` 的描述准确；`monitoring/README.md:50-68` 说"必须启用 metric reporting 才能被监控"是措辞不精确。以 observability.md 为准。

**7) `/metrics/summary` 与周期日志的差异。** `BuildMetricsSummaryText()`（`:476-490`）调用不推进快照的 `get_summary_string()`，日志线程调用 `get_summary_string_and_update_snapshot()`。并发抓取 summary 不会打乱日志的速率窗口——刻意的分工。

### 流程四：测试分层与本地复现

**四层结构：**

| 层 | 位置 | 入口 | 触发 |
| --- | --- | --- | --- |
| L1 C++ 单测 | 各模块 `tests/`（`mooncake-store/tests/CMakeLists.txt` 下 190+ 个 `add_store_test/add_ha_test`） | `ctest`（`.github/actions/run-ctest/run.sh`） | `ci.yml:unit-tests`、nightly |
| L2 wheel Python 集成 | `mooncake-wheel/tests`（31 个文件）、`python/tests/{unit,integration,services,store,ssd,ep,packaging}` | `scripts/test_installation.sh`、`scripts/run_tests.sh` | `ci.yml:test-wheel-ubuntu`（只跑 `test_installation.sh`）、`nightly`（跑 `run_tests.sh`） |
| L3 CI 冒烟套件 | `scripts/ci/run_*_smoke.sh`、`scripts/run_standalone_store_e2e.sh`（进程内 master E2E） | 直接执行 | `test-wheel-ubuntu`、`nightly` |
| L4 外部 GPU/NPU E2E | `scripts/tone_tests`（T-One 套件）、`scripts/e2e`（手工/release 与 ROCm 校验） | T-One 作业（tone-cli） | `e2e-ci.yml`（`run-e2e-ci` 标签，复用 `ci_ascend.yml`/`ci_rocm.yml`/`integration-test.yml`） |

**L1：ctest 的统一环境。** `add_store_test`（`mooncake-store/tests/CMakeLists.txt:3-19`）链 `mooncake_store + master_service_test_peer + transfer_engine + cachelib_memory_allocator + ${ETCD_WRAPPER_LIB} + glog::glog + ibverbs + gtest + gtest_main + pthread`；`add_ha_test`（`:21-38`）额外加 `gflags::gflags`。这意味着**每个 store 测试进程都链接了完整的 store 库**（而不是只链 mock），所以 ctest 吃内存，也真能测到线程/生命周期问题。

`.github/actions/run-ctest/run.sh` 做的事少但都必需：

```bash
# 预留 50052，避免 client RPC 端口被临时端口抢走
sysctl -w "net.ipv4.ip_local_reserved_ports=${reserved_ports:+$reserved_ports,}50052"
args=(--parallel "$(nproc)" --output-on-failure)   # + 可选 --label-exclude / --output-junit
cd build
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH:-}:/usr/local/lib
MC_METADATA_SERVER=http://127.0.0.1:8080/metadata \
DEFAULT_KV_LEASE_TTL=500 ctest "${args[@]}"
```

`ci.yml:258-263` 传 `label-exclude: '^nvlink_gpu$'`，因为 GitHub runner 没有 NVLink 硬件。

**L2：`scripts/run_tests.sh` 的编排契约。**

- 两种 metadata server 模式：`RUN_TESTS_METADATA_SERVER_MODE=managed`（自己起 `mooncake_http_metadata_server --port 8080` 并注册 EXIT trap）或 `external`（只等端口就绪）。等待逻辑 50×0.1s，且每轮检查进程是否已退出（`:24-37`），避免"进程崩了还傻等 5 秒"。
- 停止逻辑先 `SIGTERM`、50×0.1s 等端口释放、超时 `SIGKILL`，最后再等一轮（`:60-100`）。
- **防串扰检查**（`:129-131`）：

  ```bash
  which mooncake_master | grep -q '/usr/local/bin/mooncake_master' && \
    { echo "ERROR: mooncake_master found in /usr/local/bin, not installed by python"; exit 1; }
  ```

  保证测的是 wheel 安装的入口，而不是本地 `cmake --install` 的残留。混装过源码的开发者最常在这里翻车。

- 主流程：TE 测试（target 进程 + initiator 进程）→ master（`--default_kv_lease_ttl=500`）→ 分布式对象存储测试 → `mooncake_client` + 多 dummy client → 扩展场景。
- 三个开关：`TEST_SSD_OFFLOAD_IN_EVICT`（`:163-179`）、`TEST_PROMOTION_ON_HIT`（`:181-222`）、`TEST_CXL`（`:224-243`）。

**promotion 场景的参数注释值得逐行读**（`:185-215`）：master 侧 `--enable_offload=true --offload_on_evict=true --promotion_on_hit=true --promotion_admission_threshold=1 --promotion_max_per_heartbeat=16`；客户端侧把 `MOONCAKE_OFFLOAD_HEARTBEAT_INTERVAL_SECONDS=2`、`MOONCAKE_OFFLOAD_BUCKET_KEYS_LIMIT=10`、`MOONCAKE_OFFLOAD_BUCKET_SIZE_LIMIT_BYTES=10485760` 调小，否则 ~64MB 测试负载会一直躺在 bucket backend 的 ungrouped pool 里不落盘，测试永远等不到 promotion。这类"测试参数即系统语义"的注释比测试代码本身更有信息量。

**L3：冒烟套件的公共契约。** `scripts/ci/services.sh` 只有 80 行，但每个套件都必须遵守：

1. `set -e -o pipefail` + `cd` 到仓库根（`run_store_api_smoke.sh:3-5`）；
2. `source scripts/ci/services.sh` 并 `trap 'ci_cleanup_services "$?"' EXIT`（`:7-9`）；
3. 服务必须**前台**启动（`"$@" >"$log" 2>&1 &`，`services.sh:12`），PID 由套件持有；
4. `ci_wait_service name port...` 最多等 5 秒，每次都检查进程存活（`:32-55`）；
5. 失败时把日志尾 20 行转义成 GitHub annotation（`:18-30`）：

   ```bash
   detail=${detail//%/%25}; detail=${detail//$'\r'/%0D}; detail=${detail//$'\n'/%0A}
   printf '::error title=%s startup::%s\n' "$name" "$detail"
   ```

   不转义换行的话 annotation 只显示第一行，排障信息全丢。

6. 清理必须**逆序**（`:67-80`）：先停依赖别人的服务，最后停 metadata server；非零退出时把所有服务日志完整 `cat` 出来。

代表套件 `run_store_api_smoke.sh:11-34`：起 master（`--port 50051 --enable_http_metadata_server=true`，等 50051+8080）→ upsert/weight snapshot/async store/copy_move/safetensor/drain_http → 再起 dummy client（等 50052）跑 `tensor_from` 与 external host 路径。

**L3.5：CI 脚本自身的测试。** `scripts/ci/tests/` 用 `unittest` + `mkfifo` 精确控制时序测 `services.sh`（如 `test_services.py:28-50` 断言失败时返回码保留、所有子进程被杀、日志被打印）；`test_run_ctest.py` stub 掉 `nproc`/`sysctl` 断言端口预留参数拼接。`ci.yml:379-400` 的 `ci-script-tests` 跑的就是这些。这是"让 CI 脚本可回归"的关键投资。

**本地复现的真实入口（事实核对）：**

> **`scripts/run_ci_test.sh` 在当前仓库中不存在**：磁盘上没有该文件，`git log --all -- scripts/run_ci_test.sh` 无历史。但 `.claude/skills/mooncake-ci-local/SKILL.md`、README 技能表、`.github/pull_request_template.md` 仍在引用它。不要照文档去跑这个脚本。

可用顺序（与 CI job 一一对应）：

```
# 1) 静态检查（对应 spell-check / clang-format / python-lint）
typos
./scripts/code_format.sh --check --changed-lines --base origin/main
#     注意：真实路径是 scripts/ci/check_changed_python_format.py，
#     且它只收位置参数（BASE_REF + 变更的 .py 文件列表），没有 --base：
python scripts/ci/check_changed_python_format.py origin/main <changed.py>...

# 2) 源码构建 + ctest（对应 unit-tests）
sudo bash dependencies.sh -y
cmake -S . -B build -G Ninja -DUSE_HTTP=ON -DUSE_CUDA=OFF -DBUILD_UNIT_TESTS=ON
cmake --build build --parallel
ctest --test-dir build --output-on-failure
#    需要 MC_METADATA_SERVER=http://127.0.0.1:8080/metadata 时，
#    先起 mooncake_http_metadata_server（wheel 入口点）或 example/http-metadata-server-python/bootstrap_server.py

# 3) wheel + 安装验证（对应 build-wheel + test-wheel-ubuntu）
#    build_wheel.sh 不读 VERSION 环境变量；版本取自 mooncake-wheel/pyproject.toml（当前 0.3.13）。
#    要模拟 CI 的 0.0.0.dev0，先按 _build-wheel.yaml:126-130 的 sed 改 pyproject。
./scripts/build_wheel.sh
bash scripts/test_installation.sh

# 4) 冒烟（对应 test-wheel-ubuntu 的 smoke step）
./scripts/run_standalone_store_e2e.sh
./scripts/ci/run_store_api_smoke.sh
./scripts/ci/run_rpc_smoke.sh
./scripts/ci/run_ssd_offload_smoke.sh
#    更完整的 wheel 级集成套件（对应 nightly）：
bash scripts/run_tests.sh
```

`test_installation.sh`（`:10-85`）本身是一条完整链路：建 `test_env` venv → **先断言 `import mooncake.engine` 失败**（防止测到系统里已装的包）→ 装 wheel → 断言 import 成功 → 跑 `test_import_structure.py`、`test_fast_copy.py`、HTTP metadata server 测试、config 测试、reshard/ssd pytest → 逐个验证 `mooncake_master`/`mooncake_client`/`transfer_engine_bench`/`mooncake_http_metadata_server`/`transfer_engine_topology_dump` 入口点。

**L4：外部 E2E。** `e2e-ci.yml` 用 `pull_request_target` + `types: [labeled]`，只有 `run-e2e-ci` 标签或手动 dispatch 才跑；用 `checkout_ref`（`integration-test.yml` 用 `pr_sha`）显式指定 PR head SHA，避免 `pull_request_target` 默认检出 base 分支的陷阱。`ci_ascend.yml`、`ci_rocm.yml`、`integration-test.yml` 都被它复用。

---

## 与其他模块的边界

**构建层 → 模块（被依赖）：**

| 模块 | 耦合方式 |
| --- | --- |
| Transfer Engine | `WITH_TE`、`USE_TENT`、全部 `USE_<硬件>` 编译定义；`build-flags` job 单独验证 TE-only 构建 |
| Store | `WITH_STORE`、`STORE_USE_ETCD/REDIS/K8S_LEASE`、`STORE_USE_JEMALLOC`、`ENABLE_KV_EVENTS`；产物名 `mooncake_master`/`mooncake_client`/`libmooncake_store.so` 被 `build_wheel.sh` 硬编码拷贝 |
| EP/PG | `WITH_EP` 决定是否 `add_subdirectory(mooncake-pg)`；`EP_PG_STAGING_DIR` 是打包注入约定（`CMakeLists.txt:210-265`） |
| Conductor | `WITH_CONDUCTOR`，且 `ENABLE_KV_EVENTS` 时额外拉 `libzmq`（`CMakeLists.txt:113-115`） |
| Rust 绑定 / Store-RS | `WITH_STORE_RUST`（默认 ON）、`WITH_STORE_RS`（默认 OFF，要求 `WITH_TE`+`USE_TENT`+cargo）；pre-commit 的 cargo hooks |
| Python 包 | 根 `pyproject.toml`（scikit-build-core）与 `mooncake-wheel/pyproject.toml`（setuptools）双轨 |

**运维层 → 其他模块（依赖）：**

- 指标语义直接读 Store 的 `MasterMetricManager` / `HAMetricManager` / `WrappedMasterService`（`GetStorageUsageSnapshot`、`ListTenantQuotaSnapshots`、`GetTenantQuotaAllocatableCapacityBytes`）；这是运维端点与 Store 之间**唯一**的接口契约。
- `MasterAdminServer` 依赖 `ha::MasterRuntimeState` / `ha::MasterView`（`master_admin_service.h:16`）渲染角色与 leader，但只读快照，不参与选主。
- `docker/master.Dockerfile` 依赖 PyPI 上的 wheel 包名与版本，所以发布流程是镜像的前置条件。
- `monitoring/` 只依赖 `/metrics` 文本格式（Prometheus exposition 0.0.4），不依赖任何 Mooncake 代码。

**数据流：** 构建期 CMake cache → `add_compile_definitions` → 源码 `#ifdef` → 不同 `.so`/二进制；打包期 CMake 构建树 → `build_wheel.sh` → `mooncake-wheel/mooncake/` → auditwheel → wheel → PyPI → `pip install` → 镜像；运行期 Store 内部计数器（原子）→ `serialize_metrics()` → HTTP 文本 → Prometheus → Grafana。

---

## 设计取舍与易踩坑点

### 1. 依赖为什么做成 `Mooncake::*` provider 函数

同一依赖会被多个组件同时需要（liburing 被 Store 与 TENT 同时需要），而 IMPORTED target 不能重复 `add_library`。`Dependencies.cmake:1-2` 把"调用一次"的责任上移到构建入口，消费者只认目标名。代价是：**新增组件时必须回顶层 `CMakeLists.txt` 加条件调用**，否则子目录拿不到目标，链接期才报错。

### 2. `common.cmake` 的 Release 标志陷阱

```cmake
# common.cmake:16-23
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fcoroutines")
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -fno-tree-slp-vectorize")
endif()
set(CMAKE_C_FLAGS_RELEASE "-O3")
set(CMAKE_CXX_FLAGS_RELEASE "-O3")          # ← 覆盖，不是追加
```

CMake 对 GCC 的 `CMAKE_CXX_FLAGS_RELEASE` 初始值通常是 `-O3 -DNDEBUG`。第 23 行的 `set()` 是**整体替换**，于是：第 18-19 行刚追加的 `-fno-tree-slp-vectorize` 被丢掉；默认 `-DNDEBUG` 也被丢掉（Release 下 `assert` 仍生效）。随后第 70 行 `add_compile_options(-fno-tree-slp-vectorize)` 又无条件补回该选项，所以最终 Release 是 `-O3 -fno-tree-slp-vectorize`（外加默认 `ENABLE_DEBUG_SYMBOLS` 带来的 `-g`）。要把 `-DNDEBUG` 加回来，应改第 22-23 行而不是第 18 行。没有显式 `-DCMAKE_BUILD_TYPE` 时更隐蔽，因为默认走 `RelWithDebInfo` 的另一组标志。

### 3. 并行度只对 Ninja 生效

`limit_jobs.cmake:87-101` 只在 `CMAKE_GENERATOR MATCHES "Ninja"` 时设 job pool；Make 会无视它，在 4 vCPU/16GB 的 GitHub runner 上四个重编译单元并发直接 OOM kill（exit 137）。`_build-wheel.yaml:171-173` 与注释明确要求 `-G Ninja`。即使 Ninja，`link_pool` 也只能约束主构建；第三方 `torch.utils.cpp_extension` 的嵌套构建不受它管，所以 `_build-wheel.yaml:177-179` 另设 `MAX_JOBS: "2"`。

### 4. wheel 双轨与版本不同步

- 根 `pyproject.toml`：scikit-build-core，版本 `0.3.12.post1`，`SKBUILD` 时顶层执行 `add_subdirectory(python)`（`CMakeLists.txt:7-9`）。
- `mooncake-wheel/pyproject.toml` + `scripts/build_wheel.sh`：setuptools，版本 `0.3.13`，CI/发布实际走这条。

改版本号要同时改两处（或确认发布用 `VERSION` 环境变量覆盖：`_build-wheel.yaml:118-130` 在有 `version-override` 时 `sed` 改 pyproject）。更隐蔽的是 `keywords/classifiers` 单行约束（`mooncake-wheel/pyproject.toml:6-12`），换行会让变体 `sed` 静默失效，变体 metadata 与预期不符。

### 5. `run_ci_test.sh` 的"幽灵引用"

见上文事实核对。技能描述、PR 模板与 `.gitignore` 的 `local_test`（该脚本本应写入 `local_test/run-ci-logs/`）都指向一个不存在的脚本。这类"文档先于实现"的漂移必须标出，否则读者会花时间找一个永远找不到的文件。

### 6. CI 路径门控的两个反直觉点

- **`labeled` 不能加进 `pull_request.types`**：自动标签器会加 `run-ci`，`labeled` 会让每个新标签都重跑并取消进行中的 run；同 SHA 重跑单独走 `ci-on-label.yml`。
- **push 被取消时 `ci-gate` 不产生结果**（`ci.yml:804-807`）。这不是"失败"，不会把 main 的 required check 判红；但若把同样条件抄到 PR 工作流，就会出现"取消 = 通过"的安全漏洞。注释专门解释了 PR 侧为什么用 `always()`。

### 7. `ci-gate` 的期望值推导依赖 selector 输出

`EXPECTED: ${{ needs.check-paths.outputs.src == 'true' && 'success' || 'skipped' }}`。若 `check-paths` 因 infra 故障没产出 output，`src` 为空 → `EXPECTED='skipped'`，此时 source job 恰好 skipped 也可能让第二段通过；但第一段无条件 job 校验含 `needs.check-paths.result == 'success'`，整体仍失败。**两段互补，不能只留一段。**

### 8. 指标的三类语义必须分清

`observability.md:32` 与 `:101-110` 把周期日志字段分成三类：**rate**（两次日志之间的差值 / 秒数）、**delta**（窗口差值：逐出成功/尝试、逐出 key 数/size、alloc fail）、**live**（瞬时值：存储、key 数、client 数、discard staging size）。对应代码：rate 由 `SummaryCounters` 差值与 `elapsed_seconds` 得到（`master_metric_manager.cpp:2564-2601`）；live 直接 `.value()`；`/metrics` 导出的 counter 是**累计值**，rate 要靠 Prometheus `rate()` 自己算。混读（拿 `/metrics` 的 counter 当 QPS）是最常见的误用。

### 9. `CacheHitStat` 的"命中率"可以大于 1

`MEMORY_HIT_RATE` 的分子是**累计**命中数，分母是**当前**缓存对象数（`master_metric_manager.cpp:2247-2257`），只有分母为 0 时才输出 0。长时间运行后该值会远大于 1。枚举别名 `MEMORY_HITS_PER_CURRENT_CACHED_OBJECT`（`master_metric_manager.h:83-90`）才是准确名字，旧名为了 RPC 枚举值兼容保留。告警不要写 `hit_rate > 0.9` 这种阈值。

### 10. 管理端点的可用性 ≠ 服务可用性

`/health` 永远返回 HTTP 200，payload 的 `status` 固定为 `"ok"`（`master_admin_service.cpp:503-516`），真正表达可用性的是 `service_ready`、`role`、`ha_state`。如果负载均衡器只看 HTTP 状态码，会把 standby 当健康后端。同理 `/version` 在服务未就绪时也能返回，因为 `HandleVersion` 不经过 `GetActiveService()`。

### 11. 文档与实现的边界（已核对）

| 文档说法 | 代码事实 |
| --- | --- |
| `observability.md:7` "configurable via `kMetricReportIntervalSeconds`" | 它是 `const uint64_t = 10`（`master_admin_service.cpp:35`），只能改源码重编 |
| `observability.md:12` 日志示例来自 `rpc_service.cpp:269` | 实际输出在 `master_admin_service.cpp:296-315`，文件/行号已过时 |
| `monitoring/README.md:52` "must be started with metric reporting enabled" | 端点与 `enable_metric_reporting` 无关（`master_admin_service.cpp:279-289`） |
| `observability.md:171` "HTTP endpoints remain available regardless" | 与代码一致 |
| `.claude/skills/mooncake-ci-local/SKILL.md` 要求跑 `scripts/run_ci_test.sh` | 文件不存在 |

### 12. Docker 两镜像的"三行差异"约束

`docker/master.Dockerfile:3-5` 顶部注释与 `publish-master-image.yaml:81-94` 的归一化 diff 共同约束：`master.Dockerfile` 与 `master-cuda13.Dockerfile` 只允许在 `FROM cudalibs`、`COPY libcudart`、pip 包名三处不同。往其中一个加运行时依赖（比如补 `libnuma1`）而忘记另一个，CI 会直接失败。

镜像的两个非显然决定：
- 基础镜像是 `python:3.12-slim-trixie`（glibc 2.41），因为 aarch64 wheel 是 `manylinux_2_39`，bookworm（2.36）装不上（`:10-12`）；
- 必须 apt 装 `ibverbs-providers`，因为 `engine.so`/`store.so`/`mooncake_master` 对 `libmlx5.so.1` 有硬 `DT_NEEDED`，而 auditwheel 故意不 vendor RDMA 库（`:18-29`）。缺了它，即使只用 TCP，`import mooncake.engine` 也会失败。
- 运行身份 `65532:65532` + `tini -g --`（`:39-54`），避免 PID 1 不转发信号导致 SIGTERM 无法优雅退出。

### 13. 镜像里的 EP/PG 扩展清理（glob 与产物名不完全对应）

`master.Dockerfile:40-44` 在 `pip install` 之后执行 `rm -f "$PKG"/ep_*.so "$PKG"/pg_*.so`，注释的意图是删掉 master 容器用不到的 EP/PG 扩展（device 库会链 CUDA，启用 NCCL DeviceTransport 时还会链 NCCL；PG 的 Torch 适配器在运行期 JIT，需要 torch）。

但要留意这个 glob 的匹配范围：它只匹配以 `ep_` / `pg_` 开头的文件名。本仓库构建出的 EP host 扩展是 `_ep.cpython-*.so`（`build_wheel.sh:50`），device 库是 `libmooncake_ep_device.so` / `libmooncake_pg_device.so`（`CMakeLists.txt` 里的 target 名），都不匹配该模式；wheel 里还会带 `_pg_jit/*.cpp` 源文件。因此"这行一定删掉了扩展"不能仅凭 glob 断言，需要按实际发布 wheel 的文件清单再确认。

### 14. 冒烟脚本的"5 秒"是硬编码的

`services.sh:36` 与 `run_tests.sh:24` 都是 50×0.1s。在慢速 CI 或首次加载大 `.so` 的机器上，master 可能超过 5 秒才监听端口，表现为"启动失败"但日志里没有错误。排障时先看 annotation 里贴出的日志尾 20 行。

### 15. `test_installation.sh` 会污染当前 shell

脚本 `source test_env/bin/activate` 后在结尾明确不 deactivate（`:90-92`），注释说"让后续脚本复用这个环境"。CI 里是特性（下一步直接 `source test_env/bin/activate`），本地交互式运行是坑（跑完你的 `python` 就变味了）。

### 16. pre-commit 的格式化粒度

`.pre-commit-config.yaml:33-39` 的 `mooncake-code-format` 调用 `./scripts/code_format.sh --staged`，即**只格式化暂存区的变更行**（基于 `git-clang-format`，`code_format.sh:195-280`），不是整文件。CI 侧用 `--changed-lines --check --base <ref>`（`ci.yml:460`）。后果：未改动的旧代码即使不符合 clang-format 也不会报错；反之你只改了 1 行，也别期待 pre-commit 整理整个文件。

三个 Store-RS hook（`store-rust-dlopen-bindings`、`store-rs-rustfmt`、`store-rs-clippy`）在 pre-commit.ci 上被 skip（`.pre-commit-config.yaml:12-16`），因为需要本地 CMake 构建树与 cargo。其中 `store-rust-dlopen-bindings` 的机制是"重新生成 → 有 diff 就算失败"（`:41-48`），改了 `store_c.h` 必须重新生成并提交生成物。

### 17. macOS CI 只验证"能编译 master"

`ci-macos.yml:73-78` 只 `cmake --build build --target mooncake_master` 然后 `--version`，没有 RDMA/NUMA，也不产 wheel。它的价值是防止代码混入 Linux-only 头文件；不要当作功能回归测试。

### 18. 夜间流水线的 gate 与通知

`nightly.yml:532-574` 的 `nightly-gate` 在 `always()` 下汇总，`notify-failure`（`:575-577`）只在 gate 失败时触发。夜间承担 PR 上不跑的重活：覆盖率（`CXXFLAGS=--coverage` + `ENABLE_ASAN=ON` + `CMAKE_BUILD_TYPE=Debug`，`:394-460`）、Rust sanitizer、Go 集成、musa 构建、镜像构建、Ascend 测试、`publish-testpypi`（`:168-218`，受 `skip_publish` 保护）。注意 nightly 的 TestPyPI 发布走的是 `twine upload`，并不跑 24 目标门；真正的"24 目标发布门"在 `pre-release.yaml`（`:42` 归一化版本、`:158` validate-upload-state、`:172` wait-upload-state 校验每个已发布 wheel 的哈希）。

---

## 精读清单

按顺序读，每步给出"应该看懂什么"：

1. **`CMakeLists.txt:1-130`** —— 选项空间骨架。看懂 `WITH_*`（组件）与 `USE_*`（硬件/传输）分属两个文件；`STORE_USE_K8S_LEASE` 与 `STORE_USE_ETCD`/非 legacy `USE_ETCD` 为什么互斥（两个 Go c-shared 不能同进程）；`mooncake_provide_*` 的调用条件。

2. **`common.cmake:1-130`，再 `:241-355`、`:641-758`、`:804-815`** —— 编译基线。看懂默认 `RelWithDebInfo`、`-O3` 覆盖陷阱、ASan 双 sanitizer、sccache 接线、`USE_TPU/USE_XPU` 的 TENT guard、`GH_MIRROR` 与 `FindYLT.cmake` 的固定 commit。

3. **`limit_jobs.cmake` 全文件（101 行）** —— 为什么 CI 必须 Ninja，GitHub runner 为什么封顶 3。

4. **`cmake/Dependencies.cmake` 全文件（74 行）** —— IMPORTED 目标所有权模型，`hiredis(REQUIRED)` 的软/硬依赖区分。

5. **`dependencies.sh:52-120`、`:143-315`、`:316-435`** —— 三大 OS 族差异、SHCA 冲突、msgpack 过渡包、Kylin EPKL、Go 镜像回退与 `GOPROXY` 条件设置。

6. **`scripts/build_wheel.sh:23-178`（拷贝与暂存）、`:252-365`（变体 sed）、`:407-472`（平台标签）、`:487-614`（auditwheel 与注入）** —— 发布配方的单一来源。注意 `keywords/classifiers` 单行约束与 `--exclude libmpcomm.so*` 的理由。

7. **`_build-wheel.yaml:60-116`（variant 表）、`:149-188`（构建）、`:223-291`（冒烟校验）** —— 六个 variant×arch 组合的 cmake_args；non-cuda wheel 的"不得依赖 CUDA 库"检查；EFA 的"不得链接 libfabric"检查。

8. **`ci.yml:534-582`（check-paths）、`:802-872`（ci-gate）** —— 门控与汇总的全部语义。先读注释（`:804-806`、`:828-829`），再读表达式。

9. **`.github/actions/run-ctest/run.sh` + `action.yml`** —— 统一测试环境的三个注入（端口预留、metadata env、JUnit）与 `label-exclude` 用途。

10. **`scripts/ci/services.sh` 全文件（80 行）** —— 冒烟套件公共契约；重点 `ci_wait_service` 的存活检查与 `ci_cleanup_services` 的逆序清理。

11. **`scripts/ci/run_store_api_smoke.sh` 全文件（34 行）** —— 一个真实套件如何串起 master、API 测试、dummy client 与端口等待。

12. **`scripts/run_tests.sh:11-118`（双模式与清理）、`:120-159`（主链路）、`:163-243`（扩展场景）** —— 特别是 `/usr/local/bin` 防串扰检查与 promotion 参数注释。

13. **`scripts/test_installation.sh` 全文件（92 行）** —— wheel 安装验证的完整断言集，以及"先断言 import 失败"的技巧。

14. **`metrics_bootstrap_config.h` + `metrics_bootstrap_config_loader.cpp` + `master.cpp:474-491`** —— 默认值单一来源与"仅非默认 flag 才覆盖配置"的规则。

15. **`master_admin_service.h:121-141` + `master_admin_service.cpp:264-400`** —— 两把锁的获取顺序、周期线程的启动/停止、`RuntimeSnapshot` 的快照语义。

16. **`master_admin_service.cpp:390-474`（渲染）、`:560-591`（投影与 handler）、`:1367-1489`（路由表）** —— `/metrics` 的完整渲染路径与全部端点。

17. **`master_metric_manager.h:64-93` + `master_metric_manager.cpp:2014-2031`、`:2233-2295`、`:2315-2604`** —— 投影语义、0 值省略、别名陷阱、rate/delta 实现。

18. **`hybrid_metric.h:11-39`、`:120-132`、`:221-234`** —— 原子标签计数器与 gauge 的类型区分。

19. **`docs/source/getting_started/observability.md` 全文（325 行）** —— 与第 15-18 步对照读，标出文档与实现的差异（见"易踩坑 11"）。

20. **`monitoring/docker-compose.yml`、`prometheus/prometheus.yml`、`README.md`** —— 消费侧最小栈；注意 5 秒抓取间隔与 `extra_hosts: host.docker.internal:host-gateway`。

21. **`docker/master.Dockerfile` 全文（55 行）、`publish-master-image.yaml:81-110`、`docker/mooncake.Dockerfile:1-45`** —— 两种部署形态（运行时 vs 源码构建）与"三行差异"约束。

22. **`.pre-commit-config.yaml` 全文件 + `scripts/code_format.sh:1-30`、`:195-280`** —— staged/check/changed-lines 三种模式的差异，以及 Store-RS hooks 为什么在 CI 被 skip。

23. **`nightly.yml:35-56`、`:168-218`、`:219-392`、`:394-530`、`:532-574`** —— 版本戳、TestPyPI、集成、覆盖率/ASan、gate 与通知。

24. **`e2e-ci.yml` 全文件** —— `pull_request_target` + `labeled` 的安全模型与 `checkout_ref` 的必要性。

读完这 24 步，应当能回答：给定一个 `-D` 选项，它影响哪些源码/产物；给定一个 PR 改动路径，它触发哪些 job、gate 如何判定；给定一个 `/metrics` 数字，它是累计、速率还是瞬时值，以及是否可信。
