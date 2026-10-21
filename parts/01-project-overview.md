# 01 项目全貌与模块地图

> 本章是 Mooncake 代码导读的第一章，目标是在进入任何子目录之前，先建立一张**经源码核实**的全局地图：Mooncake 是什么、由哪几个可独立使用的层组成、这些层如何组合、构建系统如何把它们裁剪成一个 pip 包、以及一次典型的 Transfer Engine 批量传输和一次典型的 Store Put/Get 分别在哪些类之间流动。
>
> 所有行号以仓库根 `D:/xiaojun/work/Mooncake` 为基准，路径为仓库相对路径。凡本章给出的"某函数在某行"，都已在当前 checkout 上核对过；不确定的地方会显式写"此处未完全确认"。

---

## 定位与职责

### Mooncake 是什么

Mooncake 是 Moonshot AI 为其 LLM 服务 **Kimi** 打造、随后开源的推理/训练基础设施（`README.md:31-32`），论文获 **FAST'25 最佳论文**（`README.md:84`）。它的核心主张写在 README 副标题里：**"A KVCache-centric Disaggregated Architecture for LLM Serving"**。落到工程上，它做两件事：

1. **把 prefill 与 decode 拆到不同集群**（PD 分离，以及更进一步的 EPD 分离）；
2. **把 GPU 集群里闲置的 CPU、DRAM、SSD 汇聚成一个全局 KVCache 池**，用"多换存储、少换计算"（论文标题 *Trading More Storage for Less Computation*，`README.md:319`）提升有效吞吐并满足 SLO。

官方量化结论：真实负载下 Kimi 因此多处理 **75%** 的请求（`README.md:32`）；模拟场景下长上下文吞吐最高提升 **525%**（`docs/source/index.md:38`）。

### 在生态中的位置

官方架构文档把 Mooncake 明确定位为推理引擎**之下**的基础设施层（`docs/source/design/architecture.md:5`）：它不替代 vLLM / SGLang 的模型执行与调度，只补三块能力——**跨节点高效数据搬运**（Transfer Engine）、**分布式 KVCache 与对象管理**（Mooncake Store）、**弹性分布式执行**（PG + EP）。

`README.md:220-247` 的生态对照表列了 **26 个**上游项目（SGLang、vLLM、TokenSpeed、TensorRT-LLM、LMDeploy、vLLM-Ascend、FastDeploy、xLLM、Chitu、LMCache、FlexKV、NIXL、Dynamo、llm-d、RBG、srt-slurm、NeMo RL、TransferQueue、verl、checkpoint-engine、Miles、TorchSpec、Speculators、vLLM-Omni、LightX2V、slime），每行都标注它用的是 **TE**、**Store** 还是 **EP/PG**。这张表是反查"某上游项目到底依赖 Mooncake 哪一层"的最快索引，读任何集成代码前先查它。

### 四条设计原则（贯穿全仓的不变量）

`docs/source/design/architecture.md:11-14` 给出四条原则，建议把它们当作校验自己理解是否正确的标尺：

1. **高效可靠地搬数据**：RDMA 零拷贝、多网卡带宽聚合、拓扑感知选路、多协议与自动 failover。
2. **数据与控制分离**：大张量直接在拥有源/目的 buffer 的进程之间点对点传输；控制服务只交换**位置、租约、成员关系、缓存索引**元数据。这是最容易讲错的一条，后文会反复用它校验 Store 的行为。
3. **存储与计算都是弹性池**：Store segment 可独立于推理 worker 加入/退出；PG 保留稳定 rank 槽位、只改活跃成员而不重新编号。
4. **只组合工作负载需要的层**：可以只用 TE，也可以叠加 Store、PG/EP。

### 五层组件栈

代码层面是五个可独立使用的层，外加若干旁路组件：

```
┌────────────────────────────────────────────────────────────────────┐
│ 推理引擎 / 训练框架：vLLM、SGLang、TensorRT-LLM、LMDeploy、verl、…   │
├────────────────────────────────────────────────────────────────────┤
│ mooncake-wheel/：vLLM Connector、Store 服务、pg.py（PG JIT 适配）    │
│ python/mooncake/：Python API、cli、async_store、ep、buffer_pool      │
│ mooncake-integration/：pybind11 绑定（_store / engine / _conductor） │
├───────────────┬───────────────────────┬────────────────────────────┤
│ Mooncake Store│ Mooncake PG + EP      │ Conductor │ Reshard        │
│ Master+Client │ torch.distributed 后端│ 前缀索引  │ 放置规划       │
├───────────────┴───────────────────────┴────────────┴────────────────┤
│ Transfer Engine（经典 TE / TENT 双实现）                            │
├────────────────────────────────────────────────────────────────────┤
│ 元数据插件：etcd / Redis / HTTP / P2PHANDSHAKE；Transport 插件 22+19│
└────────────────────────────────────────────────────────────────────┘
```

注意分层不是简单的"上下"，而是"**可组合**"：

- Store 的数据面**复用 TE**：Store 的 `TransferSubmitter` 持有 `TransferEngine`，甚至在 TENT 模式下直接拿 `tent::TransferEngine*` 原生指针（成员声明 `mooncake-store/include/transfer_task.h:670`，构造时由 `engine.getTentEngine().get()` 初始化，`mooncake-store/src/transfer_task.cpp:1048`）。
- PG 的数据面同样走 TE，PG 自己的 Coordinator/Agent 只是控制面。
- Conductor **不依赖 TE**：它靠 libzmq + msgpack 订阅 vLLM/SGLang 的 KV 事件流（事件解码在 `mooncake-conductor/src/kvevent/event_handler.cpp` 与 `src/zmq/`；`src/main.cpp:27-58` 只负责装配 `EventManager` 并起 HTTP/RPC 端口）。
- Reshard 是纯 Python 规划库，规划阶段"地址无关"，运行时绑定才落到 TE。

### 代表性场景 → 组件映射

`docs/source/design/architecture.md:52-62` 的官方映射表：

| 场景 | 用到的组件 |
| --- | --- |
| PD 分离 / EPD 分离 | Transfer Engine |
| 跨实例分层 KVCache 复用 | Store + TE；可选 Conductor 做缓存感知路由 |
| 草稿模型训练/推理分离 | TE + Store |
| 弹性 MoE 推理 | PG + EP + TE |
| 模型权重缓存 | Reshard + TE + Store |

### 三种部署形态（以 Store 为例）

`docs/source/design/store/mooncake-store.md:40-43` 明确了 Store Client 的三种用法，这是理解部署拓扑的关键：

1. **Embedded**：Client 与推理进程同进程，`global_segment_size > 0` 时还贡献内存；
2. **dummy-real**：每个推理 rank 一个无资源的 `DummyClient`，每个推理实例一个持有资源的 `RealClient`，dummy 通过 RPC + 共享内存把请求转给 real（TP=8 时是 8 dummy + 1 real）；
3. **Standalone Store service**：独立进程（`mc_store_rest_server`，实现于 `mooncake-wheel/mooncake/mooncake_store_service.py:68` 的 `MooncakeStoreService`），内嵌客户端贡献全局内存/SSD 池；此时嵌入式客户端可以 `global_segment_size = 0`，只贡献网络能力。

### 快速上手路径

pip 安装（`README.md:263-272`）有多个变体：默认 `mooncake-transfer-engine`、CUDA 13 用 `-cuda13`，另有 `-non-cuda` / `-npu` / `-musa` / `-efa` / `-rocm`。安装后最小可跑路径见 `docs/source/getting_started/quick-start.md`：启动 `mooncake_master`，再用 `mooncake.store.MooncakeDistributedStore.setup(...)` 连上。

---

## 目录结构速览

### 顶层目录（已逐项核对存在性）

| 目录/文件 | 职责 | 默认构建 |
| --- | --- | --- |
| `mooncake-transfer-engine/` | C++ 传输引擎：经典 TE（`src/`）、TENT（`tent/`）、transport 插件、拓扑、元数据插件、Rust 绑定（`rust/`） | `WITH_TE=ON`（`CMakeLists.txt:19`） |
| `mooncake-store/` | Store 控制面 Master + 数据面 Client、分配器、HA、存储分层、对象生命周期；`rust/` 是 C++ Store 的 Rust 绑定（走 `include/store_c.h` C ABI） | `WITH_STORE=ON`（`CMakeLists.txt:20`），Rust 绑定 `WITH_STORE_RUST=ON`（`CMakeLists.txt:26`） |
| `mooncake-store-rs/` | **另一套完整的 Rust 原生 Store**（14 个 crate 的 Cargo workspace，`mooncake-store-rs/Cargo.toml:2-17`），与 C++ Store 并存而非移植 | `WITH_STORE_RS=OFF`（`CMakeLists.txt:29-32`） |
| `mooncake-pg/` | `torch.distributed` 后端 + 进程组控制面（Coordinator/Agent）、集合通信、P2P；`torch/` 是**运行时 JIT 编译**的适配源码，不在编译期链接 | 随 `WITH_EP=ON`（`CMakeLists.txt:177-178`），默认 OFF（`CMakeLists.txt:37`） |
| `mooncake-ep/` | DeepEP 风格 MoE dispatch/combine 运行时与设备 kernel | 同上，由 `WITH_EP` 控制 |
| `mooncake-conductor/` | 全局前缀缓存索引：订阅 KV 事件流，向路由器提供 HTTP/RPC 查询 | `WITH_CONDUCTOR=OFF`（`CMakeLists.txt:22`） |
| `mooncake-reshard/` | **纯 Python** 并行放置规划/绑定库（无 CMakeLists、无 C++/Rust），经 `python/CMakeLists.txt:39-45` 的 `install(DIRECTORY)` 进 wheel | 随 Python 包安装 |
| `mooncake-p2p-store/` | Go 写的点对点分发服务；官方高性能版已另立仓库为 MoonshotAI/checkpoint-engine（`README.md:73`） | `WITH_P2P_STORE=OFF`（`CMakeLists.txt:23`） |
| `mooncake-integration/` | pybind11 绑定：`store/store_py.cpp`（`_store`）、`transfer_engine/transfer_engine_py.cpp`（`engine`）、`conductor/conductor_py.cpp`（`_conductor`） | 总是 `add_subdirectory`（`CMakeLists.txt:269`） |
| `python/mooncake/` | Python 包权威源：`store/__init__.py`（cpp/rs 后端选择）、`cli*.py`、`async_store.py`、`ep.py`、`buffer_pool.py` 等 | `SKBUILD` 时（`CMakeLists.txt:7-9`） |
| `mooncake-wheel/` | 发布打包目录：第二份 `pyproject.toml`（setuptools）+ 连接器与服务源码（`mooncake_connector_v1.py`、`mooncake_store_service.py`、`pg.py`） | `scripts/build_wheel.sh` 使用 |
| `mooncake-common/` | 共享构建配置（`common.cmake` 51 个 `option`）、`FindYLT.cmake`、Go 写的 `etcd/` 与 `k8s-lease/` c-shared 库、asio | 总是构建（`CMakeLists.txt:128`） |
| `mooncake-rl/` | 只有一个演示文件 `examples/rl_samples.py`，**不是构建模块**，顶层 CMakeLists 没有它 | 否 |
| `benchmarks/` | `storage_benchmark_v1/`、`xypd_benchmarks/` | 否 |
| `FAST25-release/` | 论文 PDF + 三份 jsonl trace + 历史 arxiv trace，**无代码** | 否 |
| `docs/`、`monitoring/`、`docker/`、`scripts/`、`cmake/`、`extern/`、`image/` | Sphinx 文档、Prometheus/Grafana 看板、Dockerfile、构建/CI 脚本、CMake 依赖模块、pybind11 submodule、README 图片 | — |
| `.claude/skills/`、`.claude-plugin/` | 3 个内置 skill（mooncake-api / mooncake-ci-local / mooncake-troubleshoot）与插件市场清单（`README.md:281-305`） | — |

**对粗读地图的两处校正**：

- `docs/source/design/architecture.md:64-78` 的 Repository map 实际列了 **11 个**目录，不是粗读摘要说的 9 个；但它确实漏掉 `mooncake-store-rs/`、`mooncake-rl/`、`benchmarks/`、`monitoring/`。
- `mooncake-common/common.cmake` 里以 `^option(` 开头的行是 **51 条**，加上顶层 `CMakeLists.txt` 的 **20 条**；"60+ 开关"是量级说法，不要当作精确数字引用。

### 辅助目录的准确职责（易被忽略但各有陷阱）

- **`monitoring/`**：Prometheus + Grafana 监控 `mooncake_master`，docker-compose 一键起。它监控的是 Store 控制面指标，不是 TE 数据面。
- **`docs/`**：Sphinx + MyST 站点。改它之前必须先读 `docs/AGENTS.md`（根 `AGENTS.md` 的强制要求）。
- **`extern/`**：只有 `pybind11` 一个 submodule（`.gitmodules` 全文 4 行）。任何"依赖是不是 submodule"的问题先查这里：yalantinglibs、liburing、zstd 等全部走 FetchContent 或系统包。
- **`cmake/Dependencies.cmake`**：`mooncake_provide_zstd/xxhash/liburing/hiredis/libzmq` 等函数式依赖提供者，顶层按开关组合调用（`CMakeLists.txt:108-123`）。这段条件逻辑本身就在讲"哪个模块需要哪个库"：zstd/xxhash 只服务 Store（对象压缩与校验）、liburing 服务 Store 或 TENT（io_uring 异步文件 I/O）、libzmq 服务 Conductor 或 Store 的 KV events publisher。

### 关键顶层文件

| 文件 | 一句话职责 |
| --- | --- |
| `README.md`（374 行） | 项目门面：定位、四大 Show Case、SGLang/vLLM 集成细节、26 项生态对照表、硬件矩阵、pip 安装、引用 |
| `docs/source/design/architecture.md`（78 行） | 官方架构总览，性价比最高的入口文档 |
| `docs/source/index.md`（241 行） | Sphinx 站点首页与全站导航树 |
| `CMakeLists.txt`（285 行） | 构建入口、模块开关真值表、`add_subdirectory` 顺序、互斥校验 |
| `pyproject.toml`（根，119 行） | scikit-build-core 打包配置，包名 `mooncake-transfer-engine`，版本 `0.3.12.post1`（`:15`） |
| `mooncake-wheel/pyproject.toml`（67 行） | setuptools 打包配置（发布实际使用），版本 `0.3.13`（`:26`） |
| `mooncake-common/common.cmake`（814 行） | 硬件/协议/元数据后端 option 的集中定义与互斥校验 |
| `mooncake-common/FindYLT.cmake` | FetchContent 拉取 yalantinglibs 的唯一来源 |
| `dependencies.sh` | 一键装系统依赖，apt/dnf/yum 分支 |
| `scripts/build_wheel.sh` | 发布 wheel 的真实组装流程 |
| `.gitmodules` | 唯一 submodule：`extern/pybind11` |

### 构建开关真值表

顶层 `CMakeLists.txt:19-40` 定义了主要模块开关：

| 开关 | 默认 | 含义 | 位置 |
| --- | --- | --- | --- |
| `WITH_TE` | ON | 构建 Transfer Engine | `CMakeLists.txt:19` |
| `WITH_STORE` | ON | 构建 Store 库与示例 | `CMakeLists.txt:20` |
| `WITH_STORE_GO` | OFF | Store 的 Go 绑定（调用 `mooncake-store/go/build.sh`） | `CMakeLists.txt:21`, `271-280` |
| `WITH_CONDUCTOR` | OFF | Conductor 服务 | `CMakeLists.txt:22` |
| `WITH_P2P_STORE` | OFF | Go 版 P2P Store | `CMakeLists.txt:23` |
| `WITH_RUST_EXAMPLE` | OFF | TE 的 Rust 接口与示例 | `CMakeLists.txt:24-25` |
| `WITH_STORE_RUST` | **ON** | 给 **C++ Store** 生成 Rust 绑定 | `CMakeLists.txt:26` |
| `WITH_CONDUCTOR_RUST` | OFF | Conductor 的 Rust 绑定 | `CMakeLists.txt:27-28` |
| `WITH_STORE_RS` | **OFF** | **Rust 原生 Store 实现**（另一套实现） | `CMakeLists.txt:29-32` |
| `WITH_STORE_C_SHARED` | OFF | 自包含 `libmooncake_store.so`，只暴露 `store_c.h` C ABI | `CMakeLists.txt:33-36` |
| `WITH_EP` | OFF | EP + PG | `CMakeLists.txt:37` |
| `USE_NOF` | OFF | NoF SSD 池 | `CMakeLists.txt:38` |
| `ENABLE_KV_EVENTS` | OFF | Master 的 ZMQ KV 事件发布器（需 libzmq） | `CMakeLists.txt:77-78` |

几个**仅有读顶层 CMake 才能发现**的硬约束：

- `STORE_USE_K8S_LEASE` 与 `STORE_USE_ETCD` 不能同开，二者都会在同一进程内构建 Go c-shared 库（`CMakeLists.txt:81-93`，两条 `FATAL_ERROR`）。
- `WITH_STORE_RS=ON` 隐含要求 `WITH_TE=ON` 与 `USE_TENT=ON`，否则 `mooncake-store-rs/CMakeLists.txt:7-12` 直接 FATAL。
- `USE_TPU=ON` / `USE_XPU=ON` 强制要求 `USE_TENT=ON`（`mooncake-common/common.cmake:316-322`、`331-336`），因为 TPU/XPU 支持只存在于 TENT。
- `WITH_EP=ON` 且非 `EP_USE_IDE` 时，`libmooncake_pg.so` 与 EP 设备库被 stage 到 `ep_pg_staging`，而 Torch adapter 的源码被打包进 `_pg_jit` 目录等待运行时 JIT（`CMakeLists.txt:210-245`）。

### Transport 插件数量

- 经典 TE：`mooncake-transfer-engine/src/transport/` 下 **22 个子目录**（`ascend_transport` / `barex_transport` / `cxi_transport` / `cxl_transport` / `device` / `efa_transport` / `flagcx_transport` / `hip_transport` / `hylink_transport` / `intranode_nvlink_transport` / `kunpeng_transport` / `maca_transport` / `musa_transport` / `nccl_transport` / `nvlink_transport` / `nvmeof_transport` / `rdma_transport` / `rdma_twosided` / `rpc_communicator` / `shm_transport` / `sunrise_link` / `tcp_transport`），另有 `transport.cpp`。注意多数目录名带 `_transport` 后缀，`device` / `rdma_twosided` / `rpc_communicator` / `sunrise_link` 不带。
- TENT：`mooncake-transfer-engine/tent/src/transport/` 下 **19 个子目录**（多出 bufio、fabric、gds、hp_tcp、io_uring、memfabric、mlu、mnnvl、mpcomm、tpu、ub、xpu，少掉经典侧若干）。

两边**不是一一对应**，README 里的协议清单是合集。统计或讲"支持多少种协议"时必须说明是哪一侧。

### 文档地图

`docs/source/index.md:148-241` 的 toctree 分成七块：Getting Started、Deployment、Performance、Design、API Reference、Troubleshooting、Community（后两者常合看）。其中 Design 区（`index.md:188-204`）的目录顺序基本就是推荐的阅读顺序：

`architecture` → `transfer-engine/index` → `tent/overview` → `store/mooncake-store` → `store/store-rs/index` → `mooncake-backend-pg` → `mooncake-ep` → `conductor/index` → `mooncake-reshard/index` → `p2p-store` → `hicache-design`。

---

## 核心数据结构

本节按"从下到上"的顺序讲：先 TE 的请求/批次模型，再 Store 的控制面与服务端模型，最后 Python 侧的包装层。

### 1. `TransferEngine`：门面类，内部是两套引擎

`mooncake-transfer-engine/include/transfer_engine.h:89` 的 `class TransferEngine` 是经典 TE 对外的主要 C++ 门面（此外还有 TENT 自己的类与 C API）。最容易被误解的是它**不是**一个实现，而是双实现的门面：

```cpp
std::shared_ptr<TransferEngineImpl> impl_;                 // 经典实现
std::shared_ptr<mooncake::tent::TransferEngine> impl_tent_;// TENT 实现
std::unique_ptr<Transport> tent_compat_transport_;         // TENT 下的兼容占位
std::shared_ptr<ShutdownToken> shutdown_token_;
std::vector<std::string> tent_device_filter_;
std::string metadata_conn_string_;
bool use_tent_{false};
```

见 `transfer_engine.h:353-364`。设计后果：

- 构造函数在 `USE_TENT` 编译宏下读取环境变量 `MC_USE_TENT` / `MC_USE_TEV1` 决定 `use_tent_`（`mooncake-transfer-engine/src/transfer_engine.cpp:534-542`、`571-582`），**只创建** `impl_` 或 **只记录** `tent_device_filter_`。
- 真正的 `impl_tent_` 在 `init()` 里才创建（`transfer_engine.cpp:680-707`）。
- 每个公开方法开头都是 `if (use_tent_) { ... } else { impl_->... }` 的分流。
- `getTentEngine()`（`transfer_engine.h:145-147`）是给 Store / TransferSubmitter 走的"后门"：直接拿原生 TENT 引擎，绕开兼容层。
- `isUsingTent()` 让调用方（如 Store 的 `InitTransferEngine`）在运行时判断是否需要手动安装 transport（`client_service.cpp:829-831`）。

### 2. `tent::TransferEngine`：声明式请求 + 运行时选路

`mooncake-transfer-engine/tent/include/tent/transfer_engine.h:256` 起是 TENT 的独立命名空间类。与经典 API 的差异在于：

- Segment 用 **handle 显式导出/导入**：`exportLocalSegment(std::string& shared_handle)` / `importRemoteSegment(SegmentID& handle, const std::string& shared_handle)`，而不是只按名字 `openSegment`。
- 内存管理成对出现：`allocateLocalMemory` / `freeLocalMemory`、`registerLocalMemory` / `unregisterLocalMemory`，且接受 `Permission` 与 `MemoryOptions`。
- 批量接口叫 `allocateBatch` / `freeBatch`（经典叫 `allocateBatchID` / `freeBatchID`）。
- 请求语义是"要搬什么"，不指定"怎么搬"：运行时按拓扑、链路质量、故障状态选 transport 与路径（`docs/source/design/tent/overview.md:25-47`）。

这就是为什么 Store 在 TENT 模式下**不应该**再手动 `installTransport`：TENT 内部自己管。

### 3. `Transport`：所有插件的最小公共契约

`mooncake-transfer-engine/include/transport/transport.h:44` 的 `class Transport` 是全部 transport 插件的抽象基类。四个必须理解的类型：

**`TransferRequest`（`transport.h:60-84`）**

```cpp
struct TransferRequest {
    enum OpCode { READ, WRITE };
    OpCode opcode;
    void *source;            // 本地 buffer 指针
    SegmentID target_id;     // 远端 segment 的整数句柄
    uint64_t target_offset;  // 远端 segment 内的偏移，而非指针
    size_t length;
    int transport_hint = 0;  // TENT 专用：钉住某个 transport
    int intent_type = 0;     // 业务意图（前台 Get / 后台预取 / 迁移）
    uint64_t task_group_id = kNoTaskGroup;
    int priority = PRIO_MEDIUM; // TENT 选路优先级，经典 TE 忽略
    int nic_hint = -1;          // 经典 RDMA 的 RNIC 索引提示
};
```

关键设计：远端用 `target_id + target_offset` 表达，**不是指针**。这意味着请求可以安全地跨进程/跨机序列化与批处理，也让 TE 能在提交时按 segment 选 transport。`transport_hint` / `priority` 是 TENT 专用（注释明确写了 `TENT only`），经典 transport 读不到也无需读。`intent_type` 的取值集中定义在 `transfer_engine.h:41-46`（`kUnspecified` / `kForegroundGet` / `kBackgroundPrefetch` / `kMigration`）。

**`Slice`（`transport.h:168-403`）**：真正的最小传输单元。`Slice` 必须在堆上分配，因为它在 `markSuccess()` / `markFailed()` 时会自删除或回收到线程本地缓存（`transport.h:166-167` 注释）。其中：

- `union` 字段按 transport 分成 `rdma` / `ub` / `local` / `nccl` / `tcp` / `nvmeof` / `cxl` / `hccl` / `ascend_direct` / `ubshmem` / `flagcx` 等分支（`transport.h:203-262`）。
- `markSuccess()`（`transport.h:273-289`）先 `CompletionCallbackPin pin(task)`，再累加 `transferred_bytes` / `success_slice_count`，最后对 `completion_state` 做一次 RMW 并调用 `finishIfComplete`。
- `completion_state` 把"已完成 slice 数"与"提交已封口"打包进同一个 64 位字：bit 63 是 seal 位，低 63 位是计数（`transport.h:265-269`）。注释解释了原因——避免两个原子变量构成 Dekker 对，保证最后一次 CQ 与封口共享同一修改序。
- `CompletionCallbackPin`（`transport.h:379-402`）是生命周期保护：完成回调执行期间给 `BatchDesc.active_completion_callbacks` 加一，`freeBatchID()` 必须等它归零才能删对象，否则会出现"waiter 看到完成 → 立刻 freeBatchID → 回调还在解引用 task"的 UAF。

**`TransferTask`（`transport.h:453-496`）**：一个 task 对应一个 request，内部 `slice_list` 持有该 request 切出来的所有 slice。析构时逐个归还到 `ThreadLocalSliceCache`（`transport.h:492-495`）。

**`BatchDesc`（`transport.h:498-530`）**：一次批量提交的进度容器。字段 `batch_size` / `task_list` / `is_finished` / `finished_task_count` / `has_failure` / `active_completion_callbacks`。其中 `counter_eligible`（`transport.h:517`）只在**所有** task 的 transport 都声明 `supportsBatchCompletionCounter()`（`transport.h:572`）时为真，是 `getBatchTransferStatus` 免遍历快路径的准入条件；快路径还额外要求 `slice_timeout <= 0` 且本批尚无失败（`multi_transport.cpp:522-524`）。

这里有一个必须讲清楚、否则会读错的实现技巧：

```cpp
static inline BatchDesc &toBatchDesc(BatchID id) {
    return *reinterpret_cast<BatchDesc *>(id);
}
```

`transport.h:151-164` 的注释说明：`BatchID` 是一个**装着 BatchDesc 指针值的 64 位整数**，转换刻意跳过任何 map 查找以降低热路径开销；代价是调用方必须保证 BatchDesc 在该句柄使用期间始终存活。

### 4. `MasterService`：只有元数据，没有数据

`mooncake-store/include/master_service.h:120` 的 `class MasterService` 是 Store 控制面契约的单一来源。三个必须记住的事实：

1. **它不转发任何对象字节**。设计文档写得很直白（`docs/source/design/store/mooncake-store.md:56`）："The Master Service does not take over any data flow, only providing corresponding metadata information."
2. **它按 key 分片**：`kNumShards = 1024`（`master_service.h:1098`），每个 shard 是一个 `MetadataShard`：

```cpp
struct MetadataShard {
    mutable SharedMutex mutex;
    std::unordered_map<TenantId, TenantState, TenantIdHash> tenants GUARDED_BY(mutex);
    long disk_object_count GUARDED_BY(mutex) = 0;
    route::Generation next_generation GUARDED_BY(mutex) = 1;
};
std::array<MetadataShard, kNumShards> metadata_shards_;
```

见 `master_service.h:1122-1135`。对象身份按 `(tenant, key)` 归并：非默认 tenant 用 `hash_combine(hash(tenant), key)`，默认 tenant 退化为 `hash(key)`（`master_service.h:1337-1345`）；同一个 key 永远落同一个 shard，所以 `generation` 单调且不会重复。`TenantState` 里再放 `ObjectMap`（`unordered_map<string, ObjectRecord>`），`ObjectRecord` = `generation + ObjectMetadata + route::ObjectState`（`master_service.h:1102-1113`）。

3. **锁顺序是硬约定**。头文件注释（`master_service.h:101-118`）写明 7 级顺序：`client_mutex_` → tenant policy mutex → `master_state_mutex_` → shard mutex → tenant recompute mutex → quota table / `segment_mutex_` → soft-pin index mutex。改任何并发代码前先读这段，否则很容易构造出 ABBA 死锁。

核心 API 与其返回类型（全部用 `tl::expected<T, ErrorCode>`，错误即值）：

| API | 位置 | 语义 |
| --- | --- | --- |
| `MountSegment(const Segment&, const UUID&)` | `master_service.h:180` | Client 注册一段可分配内存；幂等 |
| `GetReplicaList(key, tenant)` | `master_service.h:406` | 查副本并**授予读租约** |
| `BatchGetReplicaList(keys, tenant)` | `master_service.h:422` | 批量版；`Client::BatchQuery` 与批量写路径走它 |
| `BatchGetReplicaListForAdmin` | `master_service.h:431` | **不授租约、不触发 promotion/命中率统计**的只读版 |
| `PutStart(client_id, key, ...)` | `master_service.h:442` | 分配空间、创建 PROCESSING 副本 |
| `PutEnd(client_id, meta, ...)` | `master_service.h:453` | 标记写入完成 |
| `Ping(client_id)` | `master_service.h:712` | 心跳 + view version |

`PutStart` 的实现（`mooncake-store/src/master_service.cpp:5589-5661`）值得单独看：它先 `PrepareWriteStart` 做前置规划，再在 `admit` lambda 里取 `client_mutex_` 与 `master_state_mutex_` 的共享锁，做租户配额扣减、stale 清理，最后 `AllocateAndInsertMetadata`（`master_service.cpp:5404` 起）调用 `AllocateReplicas` 真正向各 segment 的 allocator 要空间。

`GetReplicaList` 的实现（`master_service.cpp:4777-4833`）在返回前做了三件事：挑可读副本、记录 mem/file cache 命中指标、`metadata.GrantReadLease(default_kv_lease_ttl_)`。这个 grant 就是后文"租约"流程的源头。默认值在 `mooncake-store/include/types.h:93-94`（`DEFAULT_DEFAULT_KV_LEASE_TTL = 10000` ms），软 pin 默认 30 分钟在 `types.h:95-96`。

### 5. `Client`：既是客户端，也是存储节点

这是 Store 最容易被讲混的地方。**代码里有两个层次的"客户端"**：

**(a) 内部核心类 `mooncake::Client`**（`mooncake-store/include/client_service.h:77`）

- 持有 `MasterClient master_client_`（RPC 控制面）、`std::shared_ptr<TransferEngine> transfer_engine_`、`TransferSubmitter`、`mounted_segments_`、`hot_cache_`。
- `static Create(...)`（`client_service.h:102-109`，实现在 `client_service.cpp:1046`）负责：连 Master → 读 storage config（决定是否启用本地磁盘后端）→ `EnterHaRuntimeMode()` → 初始化或复用 TE → `InitTransferSubmitter()` → `InitLocalHotCache()`。
- `MountSegment`（`client_service.cpp:3856` / `3914`）把自己的一段内存注册进 TE 并告知 Master；`UnmountSegmentImpl`（`client_service.cpp:3866`）顺序相反：先 `master_client_.UnmountSegment`，再 `transfer_engine_->unregisterLocalMemory`。
- `Get` / `Put` / `BatchGet` / `BatchPut` / `Query` 是数据面入口。

**(b) Python 可见的 `PyClient` 抽象**（`mooncake-store/include/pyclient.h:232`）

- 纯虚接口：`setup_real` / `setup_dummy` / `initAll` / `put` / `get_into` / `batch_query` / `register_buffer` / `remove` / `health_check` 等（`pyclient.h:313-557`）。
- 两个实现：`RealClient : public PyClient`（`mooncake-store/include/real_client.h:83`）与 `DummyClient : public PyClient`（`mooncake-store/include/dummy_client.h:21`）。
- Python 侧真正 `py::class_` 暴露的是 `MooncakeStorePyWrapper`（`mooncake-integration/store/store_py.cpp:533`），它内部持有 `std::shared_ptr<PyClient> store_` 和 `std::shared_ptr<RealClient> real_client_`，并用 `use_dummy_client_` 区分形态。`setup(...)` 绑定在 `store_py.cpp:2485`，`setup_dummy` 在 `store_py.cpp:2529` 左右。

完整的 Python 类层次：

```
Python: mooncake.store.MooncakeDistributedStore
  └─ pybind11: MooncakeStorePyWrapper        (store_py.cpp:533)
       └─ 抽象基类 PyClient                   (pyclient.h:232)
            ├─ RealClient : PyClient          (real_client.h:83)  ── 拥有资源
            └─ DummyClient : PyClient         (dummy_client.h:21) ── 转发给 RealClient
                 └─ RealClient 经 PyClient 基类的 client_ 持有 Client；DummyClient 的 client_ 为空，靠 RpcClientPool 转发
```

设计文档里说的"Client 类"指 (a)；Python 用户看到的 `MooncakeDistributedStore` 是 (b) 的包装。两者不是同一层，改名或查日志时务必区分。

值得注意：`DummyClient::setup_real` 与 `initAll` 在头文件里就是**内联返回 -1 的桩**（`dummy_client.h:28-53`），而 `RealClient` 反过来实现 `setup_real`。也就是说"两种 setup 只能选一种"不是文档约定而是类型层面的硬约束。

**`QueryResult`（`client_service.h:43-72`）** 是读路径的核心载体：

```cpp
const std::vector<Replica::Descriptor> replicas;
const std::chrono::steady_clock::time_point lease_timeout;
const std::optional<uint64_t> object_checksum;
const std::optional<std::string> resolved_tenant_id;
bool IsLeaseExpired() const;
```

注意 `replicas` 与 `lease_timeout` 都是 `const`：一旦构造就不可变，天然适合跨线程传递快照。

### 6. Store 公共类型底座

`mooncake-store/include/types.h` 是 master 与 client 之间契约的单一来源：

| 类型 | 位置 | 说明 |
| --- | --- | --- |
| `ObjectKey` | `types.h:200` | 直接 `using ObjectKey = std::string` |
| `UUID` | `types.h:215` | `std::pair<uint64_t, uint64_t>` |
| `ObjectDataType` | `types.h:159` | KVCACHE / TENSOR / WEIGHT / SAMPLE / ACTIVATION / GRADIENT / OPTIMIZER_STATE / METADATA / GENERAL / UNKNOWN |
| `ErrorCode` | `types.h:303` | 全链路错误码，`tl::expected` 的 error 类型 |
| `Segment` | `types.h:457` | 一段注册内存的元数据（base/size/name/te_endpoint/protocol/host_id 等） |
| Store `Slice` | `types.h:445` | `{void* ptr; size_t size;}`，**与 TE 的 `Transport::Slice` 同名不同物** |

`DEFAULT_GLOBAL_SEGMENT_SIZE` 是 16 MB（`types.h:244`），这也解释了为什么很多示例里的 `global_segment_size` 默认值都是 16MB。

`mooncake-store/include/replica.h:108-135` 的 `ReplicateConfig` 是每次写的策略包：

```cpp
size_t replica_num{1};
size_t nof_replica_num{0};
size_t dfs_replica_num{0};
SoftPinAction soft_pin_action{SoftPinAction::PRESERVE};
std::optional<uint64_t> soft_pin_ttl_ms{};
bool with_hard_pin{false};
std::vector<std::string> preferred_segments{};
std::string preferred_segment{};         // Deprecated: 单段兼容字段
std::vector<std::string> preferred_nof_segments{};
bool prefer_alloc_in_same_node{false};
ObjectDataType data_type{ObjectDataType::UNKNOWN};
std::string host_id{};
std::optional<std::vector<std::string>> group_ids{};
struct_pack::compatible<std::vector<std::string>, 1> session_ids;
```

三个字段族对应三种"保护等级"：普通副本数（`replica_num`）、软 pin（可过期、可被驱逐）、硬 pin（永不驱逐，只能显式删除）。`group_ids` 则是**生命周期分组**，不是路由键——路由永远是 `hash(tenant, key)`（`replica.h:126-130` 注释与 `docs/source/design/store/mooncake-store.md:186` 一致）。

`Replica` 本身（`replica.h:301` 起）是 move-only，内部 `std::variant` 存 `MemoryReplicaData` / `NoFReplicaData` / `DiskReplicaData` / `LocalDiskReplicaData` / `DfsReplicaData`；状态机 `UNDEFINED → INITIALIZED → PROCESSING → COMPLETE → REMOVED/FAILED`（`replica.h:56-63`）。

### 7. Store-RS：并存的第二实现

`mooncake-store-rs/` 不是移植，而是重写。它的核心不变量写在 `docs/source/design/store/store-rs/architecture.md` 的 **Hot-Path Invariant**：

> 稳态请求热路径不得依赖后端往返；Redis / etcd / admin RPC 必须待在每请求读写延迟路径之外。

默认路由模式是 `EmbeddedWrh`（embedded weighted rendezvous hashing）：客户端从活跃租约里挑候选，按 key 选 top-k 权威，最高者为 CAS primary、其余为 mirrored。这正是它强制要求 `USE_TENT=ON` 的根因——数据面完全交给 TENT。

---

## 关键流程剖析

### 流程 A：经典 TE 初始化 + 一次批量传输

**入口**：`TransferEngine::init(metadata_conn, local_server_name, ip, port)`（`transfer_engine.cpp:128` 或带协议的 `138`）。

**classic 分支调用链：**

```
TransferEngine::init
  └─ TransferEngineImpl::init                       transfer_engine_impl.cpp:108
       ├─ parseHostNameWithPort                     :145
       ├─ 选择 RPC 绑定方式（legacy/P2P 或 new RPC） :151-211
       ├─ metadata_ = make_shared<TransferMetadata> :223
       ├─ multi_transports_ = make_shared<MultiTransport> :227-231
       ├─ metadata_->addRpcMetaEntry(name, desc)    :233
       └─ transport 安装决策树
            ├─ MC_FORCE_TCP → 只装 tcp                :240-255
            ├─ MC_FORCE_SHM（非 multi-protocol）→ 只装 shm :262-274
            ├─ Ascend / UBSHMEM / CXL / … 各分支
            └─ auto discovery 下的 RDMA/TCP 安装
```

**分步说明：**

1. **先定 RPC 端点，再建引擎**。`init` 的第一件事不是碰网卡，而是确定本进程在元数据服务里叫什么、RPC 监听在哪。`P2PHANDSHAKE` 模式会现找一个可用 TCP 端口并重写 `local_server_name_`（`transfer_engine_impl.cpp:170-186`），因为去中心化模式下没有外部服务帮忙分配端口。
2. **先建元数据，再建 MultiTransport**。`TransferMetadata` 负责 Segment 描述与 RPC 元数据的发布/缓存；`MultiTransport` 构造时拿到同一个 metadata 指针，因此所有 transport 插件共享同一个元数据视图（`transfer_engine_impl.cpp:223-231`）。
3. **RPC 入口先注册**。`addRpcMetaEntry` 在装 transport 之前完成，保证别的节点来握手时本进程已经可寻址。
4. **transport 安装是决策树，不是列表**。环境变量优先级最高（`MC_FORCE_TCP` / `MC_FORCE_SHM`），其次是编译期硬件分支，最后才是 auto discovery。读这段时要意识到：**同一个二进制在不同机器上装出来的 transport 集合可能不同**。

**一次批量传输：**

```
engine.allocateBatchID(n)                  → BatchDesc（n 个 TransferTask）
engine.registerLocalMemory(addr, len, loc) → 本地 buffer 进入 segment
engine.openSegment(remote_name)            → 远端 SegmentID
engine.submitTransfer(batch_id, requests)  → 按 target_id 分组，逐组选 transport
engine.getTransferStatus(batch_id, i, st)  → 轮询每个 task
engine.freeBatchID(batch_id)               → 可能返回 BatchCleanupDeferred
```

- `registerLocalMemory` 是本地行为：把地址范围与 `location`（`cpu:0` / `cuda:0` / `cpu:1` 之类）登记进 TE，供拓扑选路与远端 rkey 生成使用。
- `submitTransfer` 的核心是按 `target_id` 把 `TransferRequest` 分派给对应 Segment 的 transport；每个 request 会按 64KB 粒度切成多个 `Slice`（`docs/source/design/transfer-engine/index.md` 的 Topology 节明确写了 64KB 切分），每个 slice 可能走不同 NIC。
- 完成语义由 `TransferStatus`（`transport.h:96-99`）表达：`WAITING / PENDING / INVALID / CANCELED / COMPLETED / TIMEOUT / FAILED` + `transferred_bytes`。
- `freeBatchID` 的返回可能是 `BatchCleanupDeferred`（`transfer_engine.h:269-273` 的注释），表示还有 in-flight transport work 持有 buffer，调用方不能立刻复用这块内存。

**并发/生命周期要点：**

- `BatchDesc` 用指针值塞进 `BatchID`，所以**绝不能跨进程序列化 BatchID**。
- 完成回调与 `freeBatchID` 的竞争由 `active_completion_callbacks` 的 pin 机制解决（`transport.h:373-402`）。
- `BatchDesc` 在 TENT 兼容模式下不会被创建：见流程 B。

### 流程 B：TENT 分叉，以及 Store 如何绕过兼容层

**编译期开关**：`mooncake-transfer-engine/src/CMakeLists.txt:208-211` 在 `USE_TENT` 时 `add_compile_definitions(transfer_engine PUBLIC USE_TENT)` 并链接 `tent_link_group`。该宏把 `transfer_engine.cpp` 劈成两个完整实现（`#ifndef USE_TENT` 在 `transfer_engine.cpp:38`，`#else` 在 `transfer_engine.cpp:386`）。

**运行时开关**：`MC_USE_TENT` 或 `MC_USE_TEV1` 只要不被判为 false，就置 `use_tent_ = true`（`transfer_engine.cpp:28-33` 的 `isTransferEngineEnvEnabled` + `534-542`）。

**TENT 路径的 init：**

```
TransferEngine::init(..., protocol)
  ├─ buildTentConfig(metadata_conn, local_name)      transfer_engine.cpp:642-660
  │    ├─ local_segment_name
  │    ├─ metadata_type = "p2p" | etcd | redis | http
  │    ├─ metadata_servers
  │    └─ topology/rdma_whitelist ← tent_device_filter_
  ├─ protocol == "tcp" → ConfigHelper::forceTcp     :682-688
  ├─ Ascend 相关 flag 透传                            :689-702
  └─ impl_tent_ = make_shared<tent::TransferEngine>(config) :703
       └─ !available() → ERR_CONTEXT
```

**兼容层真相**：TENT 模式下 `installTransport()` 不再安装任何东西，而是惰性创建一个 `TentCompatibilityTransport` 占位对象（每个 `TransferEngine` 一份，受 `tent_compat_transport_mutex_` 保护；`transfer_engine.cpp:726-743`），这个类所有方法都是 `NotImplemented` 或 `ERR_NOT_IMPLEMENTED`（`transfer_engine.cpp:447-492`）。它存在的主要理由是让"先 install 再 submit"的旧调用序列不崩。`uninstallTransport` 直接返回 0（`:745-750`）。

`openSegment` 也被重写：TENT 下调用 `impl_tent_->openSegment(handle, name)`，失败返回 `ERR_INVALID_ARGUMENT`（`:773-782`）。`CheckSegmentStatus` 更微妙——它主动 `probePeerAliveByID`，因为"总是返回 OK"会让上层永久缓存死 peer 的句柄（`:784-798`，注释引用了 #3995）。

**Store 的特殊处理**：Store 的 `Client::InitTransferEngine` 会用 `transfer_engine_->isUsingTent()` 判断，TENT 下跳过手动 transport 安装（`client_service.cpp:829-1010`）。而真正的数据面，`TransferSubmitter` 在构造时就把 `engine.getTentEngine().get()` 存成 `tent_engine_`（`transfer_task.cpp:1048`；成员声明见 `transfer_task.h:670`），提交时优先走 `tent_engine_->submitTransfer(batch_id, tent_requests)`（`transfer_task.cpp:1784`），只有 TENT 不可用才退回经典 `engine_.allocateBatchID` + `submitTransfer`（`transfer_task.cpp:1794`）。这就是"经典 TE 是一层皮"的具体体现。

**易踩点**：`installTransport` 在 TENT 下返回非空指针，但它是**假的**。任何依赖 `Transport*` 去调用 `getTransferStatus` 或 `registerLocalMemory` 的代码在 TENT 下都会拿到 `NotImplemented`。判断"我到底在哪个引擎上"最可靠的方法是 `isUsingTent()`（`getTentEngine()` 返回非空也可作为等价判据），不是 `installTransport` 是否返回 null。错误码也要经过 `tentToClassicError`（`transfer_engine.cpp:413-445`）翻译，无法精确映射的统一落 `ERR_CONTEXT`。

### 流程 C：一次 Store Put（写路径全链路）

**入口**：Python `store.put(key, data, config)` → `MooncakeStorePyWrapper` → `PyClient::put`（`pyclient.h:335`）→ `RealClient::put`（`real_client.h:118`）→ `Client::Put`。

**调用链（以 `Client::Put` 为主干）：**

```
Client::Put(key, slices, config)                         client_service.cpp:1959
  ├─ ComputeObjectChecksumForSlices（可选）               :1962-1970
  ├─ AttachHostId(config)（local_first 策略需要）         :1978
  ├─ hot_cache_->RemoveHotKey(key)                        :1983-1985
  ├─ master_client_.PutStart(key, slice_lengths, cfg)     :1988
  │    └─ [RPC] MasterService::PutStart                   master_service.cpp:5589
  │         ├─ PrepareWriteStart
  │         ├─ ChargeTenantQuota
  │         ├─ CleanupStaleBeforeWrite
  │         └─ AllocateAndInsertMetadata → AllocateReplicas
  ├─ 磁盘副本先写（PutToLocalFile）                        :2020-2031
  ├─ 对 MEMORY / NOF_SSD 副本逐个 TransferWrite            :2033-2046
  │    └─ TransferData → transfer_submitter_->submit       :4698-4729
  ├─ DFS 副本写（要求非 DFS 传输已成功）                   :2048-2079
  ├─ DetermineFinalizeDecision                            :2088
  └─ master_client_.PutEnd(...) 或 PutRevoke(...)          :2091-2108
```

**分步说明：**

1. **先算校验和（可选）**。`object_checksum_enabled_` 打开时，写入前对 slices 算一次对象级 checksum，随 `PutEnd` 的 `ObjectMeta` 传给 Master；读路径用它做端到端校验。
2. **PutStart 是"预留"，不是"写入"**。Master 收到请求后按 `AllocationStrategy` 从各 segment 的 allocator 分配 offset，返回一组 `Replica::Descriptor`，副本状态是 `PROCESSING`。此时数据一个字节都还没动。
3. **Master 不搬数据**。这一步之后，Client 拿到的是"目标 segment + offset"的地址簿；真正的写由 Client 通过 TE 直接打到目标 Client 的注册内存。这是设计原则 2 的直接落地。
4. **副本写入顺序有讲究**。磁盘副本先写（`:2018-2031`，注释："We must deal with disk replica first, then the disk putrevoke/putend can be called surely"），因为磁盘路径可能失败，先失败可以先 revoke；内存/NoF 副本随后提交；DFS 副本最后写，且只有非 DFS 传输都成功才写（`:2048-2050` 的 `NonDfsTransfersSucceeded`）。
5. **PutEnd 才让对象可读**。Master 的 `PutEnd`（`master_service.cpp:5663`）会校验 `client_id == metadata.client_id`（`:5676-5680`，否则 `ILLEGAL_CLIENT`），再把对应类型的副本标成 `COMPLETE`。
6. **失败路径是 PutRevoke，不是删除**。`DetermineFinalizeDecision` 根据 `transfer_summary` 决定 end 哪些副本类型、revoke 哪些类型。这保证了部分成功不会被误判为整体成功。

**并发/生命周期要点：**

- `Client::Put` 内部的 `TransferWrite` 是**同步等待**的（`TransferData` 里 `future->get()`，`client_service.cpp:4728`），但底层 TE 是异步的；批量接口 `BatchPut` 才会把多个 key 的提交聚合成一次 scatter（`BatchTransferWriteRanges`，`client_service.cpp:4882-4974`，注释："One scatter submit covers every replica of every entry"）。
- `hot_cache_->RemoveHotKey(key)` 在写前执行：写入会改变对象内容，本地热缓存必须失效。
- `PutStart` 可能因 `OBJECT_ALREADY_EXISTS` 失败，此时会尝试 `healDanglingLocalDiskReplica` 后再试一次（`:1989-1993`）；再遇 `OBJECT_ALREADY_EXISTS` 则幂等地当作成功返回（`:1996-1999`）。

### 流程 D：一次 Store Get（读路径 + 租约）

**调用链：**

```
Client::Get(key, slices)                                 client_service.cpp:1168
  ├─ Query(key)                                          :1240
  │    ├─ master_client_.GetReplicaList(key)
  │    └─ lease_timeout = start_time + lease_ttl_ms      :1250
  ├─ Client::Get(key, query_result, slices)              :1382
  │    ├─ FindFirstCompleteReplica                       :1387
  │    ├─ hot_cache_ && memory replica → RedirectToHotCache :1397-1399
  │    ├─ DFS 副本 → ReadDfsReplica；否则 TransferRead     :1402-1406
  │    ├─ hot_cache_->ReleaseHotKey                      :1409-1411
  │    ├─ VerifyObjectChecksum                           :1425-1435
  │    ├─ ShouldAdmitToHotCache → ProcessSlicesAsync     :1440-1442
  │    └─ query_result.IsLeaseExpired() → LEASE_EXPIRED  :1444-1448
  └─ return
```

**分步说明：**

1. **Query 同时完成"拿地址"和"拿租约"**。Master 侧 `GetReplicaList` 在返回前 `metadata.GrantReadLease(default_kv_lease_ttl_)`（`master_service.cpp:4832-4833`）。默认 TTL 是 10000 ms：`mooncake-store/src/master.cpp:42-44` 有 `static_assert(DEFAULT_DEFAULT_KV_LEASE_TTL == 10000)`，flag 字符串在 `master.cpp:52`，gflags 定义在 `master.cpp:144`。
2. **租约时间戳在客户端算，但基准是"发起查询的时刻"**。`Client::Query` 记录 `start_time`，然后 `start_time + lease_ttl_ms`（`client_service.cpp:1242-1251`）。这个细节很重要：它把一次 RPC 的网络往返时间也算进了租约窗口，因此客户端判断过期时是保守的，不会比 Master 更宽松。
3. **副本选择只挑 COMPLETE**。`FindFirstCompleteReplica` 会跳过 `PROCESSING` / `FAILED` / `REMOVED`。如果没有可用副本且所有副本都是 `REMOVED`，Master 返回 `OBJECT_NOT_FOUND`；否则返回 `REPLICA_IS_NOT_READY`（`master_service.cpp:4807-4815`）。
4. **本地热缓存是一层旁路**。命中时 `RedirectToHotCache` 把 replica descriptor 改成本地地址，`TransferRead` 退化成 memcpy；读完必须 `ReleaseHotKey`（`:1409-1411`），否则缓存块泄漏。
5. **校验和失败直接失败（有前提）**。只有写入时启用了 `object_checksum_enabled_`、且 Master 返回的对象带 checksum 时才校验；两个条件任一不满足就直接放行（`client_service.cpp:1349-1358`）。`VerifyObjectChecksum`（`:1425`）对整对象校验，不匹配返回 `CHECKSUM_MISMATCH`。这是"启用校验后 Get 返回正确完整数据"的最后一道闸。
6. **传输完成后还要再查一次租约**。`:1444-1448`：如果传输过程中租约过期，返回 `LEASE_EXPIRED` 且**不返回数据**。这是为了防止"Master 已经允许删除、Reader 还在读旧内存"的撕裂。
7. **频率准入**。`ShouldAdmitToHotCache` 用 count-min sketch 判断是否值得把对象提升到本地热缓存，避免一次性扫描把缓存冲掉（`:1437-1442` 注释）。

**为什么租约必须存在**：`docs/source/design/store/mooncake-store.md:360-368` 说明，租约保护对象不被 `Remove` / `RemoveAll` / 驱逐；`Remove` 对已租约对象会失败，`RemoveAll` 只删无租约对象；租约过期后 `Get` 失败而不是返回可能损坏的数据。

### 流程 E：`mooncake_master` 启动

**入口**：`mooncake-store/src/master.cpp:1411` 的 `main`。

**分步说明：**

1. **gflags + glog**。`gflags::ParseCommandLineFlags`（`:1414`）→ 若指定 `--log_dir`，把 glog 的 INFO 目标重定向成单一日志文件并关闭高等级文件（`:1421-1434`），避免同一行日志写多份。
2. **配置分层**：`DefaultConfig`（配置文件）→ `LoadConfigFromCmdline`（命令行覆盖）→ 派生配置（metrics、client liveness、RPC 地址）（`:1460-1495`）。
3. **启动前校验**：HA backend 连接串缺失直接 FATAL（`:1490-1498`）；`enable_oplog` 要求 `enable_ha=true` 且后端支持 OpLog（`:1497-1506`）；allocator 必须是 `cachelib` 或 `offset`（`:1518-1524`）。
4. **可选内嵌 HTTP 元数据服务**。`enable_http_metadata_server=true` 时启动进程内 `HttpMetadataServer`（`:1651-1664`，函数定义在 `:1385` 左右），并 sleep 1 秒等待就绪。默认是 false，也就是说 **Store 的元数据服务默认要单独部署**（与 `docs/source/design/store/mooncake-store.md:30` 一致）。
5. **两条主干**：
   - **HA**：先装 shutdown watchdog 与信号处理器，再构造 `MasterServiceSupervisor` 并 `Start()`（`:1674-1702`）。注释明确：必须在任何租约可能被获取之前武装好，否则 SIGTERM 会杀掉正在任期内的 leader。
   - **非 HA**：构造 `coro_rpc::coro_rpc_server`（`:1706-1710`）→ `WrappedMasterService`（`:1719-1722`）→ `MasterAdminServer`（`:1723-1733`）→ `RpcWorkerPool`（`:1737-1738`）→ `RegisterRpcService`（`:1739-1740`）。
6. **优雅退出**：信号只置 `shutdown_requested`（`:1742-1747`），主线程每 100ms 轮询，`server.stop()` 后 join（`:1756-1761`）。注释说明 `worker_pool` 声明在 service 之后，保证先 drain 再销毁 service（`:1735-1736`）。

**与 TE 的边界**：`mooncake_master` 本身**不初始化 TransferEngine**。它只做元数据。`enable_metadata_cleanup_on_timeout` 走的是 HTTP DELETE 到元数据服务，而不是通过 TE（`:1537-1569`）。

**独立数据面进程**：`mooncake_client` 的入口在 `mooncake-store/src/real_client_main.cpp:114`，`RegisterClientRpcService`（`:38-111`，调用点 `:161`）注册了 `put_dummy_helper` / `put_batch_dummy_helper` 等一整组 dummy↔real RPC handler。这条 RPC 面就是 dummy-real 部署形态的完整契约。

### 流程 F：Python 包的组装与 Store 后端选择

这条流程解释了"为什么 wheel 里既有 C++ 又有 Rust，而 import 时只看到一个"。

1. **构建期**：`scripts/build_wheel.sh` 把 `engine.so`（`:39`）、`_store.so`（`:63-65`）、`mooncake_master` / `mooncake_client`（`:68-70`）、`libmooncake_pg.so`、`libasio.so`、`_pg_jit/` 等复制进 `mooncake-wheel/mooncake/`，然后 `cd mooncake-wheel` 用 setuptools 打包（`:244`）。
2. **命名空间包**：`python/CMakeLists.txt:18-24` 用 `install(DIRECTORY ../mooncake-wheel/mooncake/ ... PATTERN "*.py")` 把连接器与服务装进 `mooncake/`，注释自称这是"Phase 1 迁移中间态"，Phase 2 会逐子系统消失。三处 `__init__.py` 都用 `pkgutil.extend_path` 合并命名空间（`python/mooncake/__init__.py:5-7`、`mooncake-wheel/mooncake/__init__.py:6-7`）。
3. **运行时选择**：`python/mooncake/store/__init__.py:95` 读 `MOONCAKE_STORE_BACKEND`：
   - `"cpp"`（默认）→ `importlib.import_module("mooncake._store")`（`:98`）；
   - `"rs"` → `importlib.import_module("mooncake.store.rs.store")`（`:109`）；
   - 其他值 → `ValueError`（`:119-121`）。
4. **导出白名单**：`_CPP_EXPORTS`（`:9-59`）与 `_RS_EXPORTS`（`:71-94`）两份 frozenset 精确刻画两套实现的 API 面差异；两者之间还有一份只给 cpp 后端用的 `_CPP_INTERNAL_EXPORTS`（`:61-68`，`_serialize_tensor` 等内部符号）。`__getattr__`（`:126-138`）只放行白名单；若 rs 后端访问 C++ 独有符号，会抛出带解释的 `AttributeError`。这是判断"某功能在哪个后端可用"的最快依据。

---

## 与其他模块的边界

### 依赖方向（谁用谁）

```
vLLM / SGLang / TensorRT-LLM / LMDeploy / ...   （下游集成方，加载 Mooncake）
        │ KV Connector / HiCache L3 后端 / EP 通信后端
mooncake-wheel（连接器、服务） + python/mooncake（API 门面）
        │ pybind11
mooncake-integration（_store / engine / conductor 绑定）
        │
┌───────┼──────────────┬───────────────┬────────────────┐
Store(C++)        Store-RS(Rust)      PG + EP         Conductor
│                    │                 │                │（不依赖 TE！
└─────都依赖──── Transfer Engine ◄──────┘                │  靠 zmq+msgpack
                     │                                  │  订阅 KV 事件流）
              transport 插件 ×22(经典)/×19(TENT)
                     │
        元数据服务：etcd / Redis / HTTP / P2PHANDSHAKE(去中心化)
```

### TE ↔ Store

| 维度 | TE | Store |
| --- | --- | --- |
| 元数据内容 | Segment、buffer、RPC 端点、rkey | 对象目录、副本状态、租约、租户配额 |
| 元数据后端 | etcd / Redis / HTTP / `P2PHANDSHAKE` | `mooncake_master`（Store 自己的 RPC 服务） |
| 是否搬数据 | 是（唯一的数据面） | Master 不搬；Client 搬 |
| 接口 | `submitTransfer` / `submitScatter` / `submitTransferWithNotify` | `PutStart/PutEnd/GetReplicaList/...` |

**最容易混的一点**：`mooncake_master` 可以内嵌一个 HTTP 元数据服务（`--enable_http_metadata_server=true`），但它默认关闭；而这个服务是给 **TE** 用的 peer discovery 元数据，不是 Store 的对象目录。两套元数据是**两个不同的东西**，只是恰好可以由同一个进程提供。`P2PHANDSHAKE` 字面量宏定义在 `transfer_metadata.h:56`。

Store 对 TE 的依赖是编译期链接 + 运行期调用：`mooncake-store/src/CMakeLists.txt` 链接 `transfer_engine`；`Client::Create` 里 `std::make_shared<TransferEngine>()`（`client_service.cpp:1145`）；`TransferSubmitter` 再封装成策略层（memcpy / TE / scatter / TENT native）。

### Store ↔ Python / 集成层

- C++ 契约在 `mooncake-store/include/pyclient.h`（`PyClient` 纯虚）。
- 绑定在 `mooncake-integration/store/store_py.cpp`，模块名 `_store`（`store_py.cpp:2137`）。
- Python 门面在 `python/mooncake/store/__init__.py`。
- **注意**：`_store` 是扩展模块，`mooncake.store` 是 Python 包；前者是后者的后端之一，不是替代关系。

### PG / EP ↔ TE 与 Store

- PG 是 `torch.distributed` 后端，数据面走 TE；控制面是 Coordinator（global rank 0）+ 每进程 Agent。
- EP 是 DeepEP 风格 dispatch/combine，通常从 PG 进程组 bootstrap，复用 PG 的 active-rank 信息（`docs/source/design/architecture.md:40-42`）。
- PG 的 Torch adapter **不在编译期链接**：`mooncake-wheel/mooncake/pg.py:80` 从 `__file__` 同级的 `_pg_jit` 取源码，`:133` 生成扩展名，`:192` 在 MUSA 下先装 torchada，然后 `torch.utils.cpp_extension.load` 现场编译。顶层 CMake 的 `mooncake_pg_jit_sources_stage` 目标（`CMakeLists.txt:217-245`）就是为这条路径准备源码包。
- `EP_USE_IDE=ON` 只用于 clangd 索引，CMake 里明确 `DO NOT USE IN PRODUCTION`（`CMakeLists.txt:181-193`）。

### Conductor

- 默认不构建（`WITH_CONDUCTOR=OFF`）。
- 不依赖 TE；依赖 libzmq + msgpack 订阅 vLLM/SGLang 事件（`CMakeLists.txt:113-115` 提供 libzmq）。
- 进程入口 `mooncake-conductor/src/main.cpp:27`：`ParseConfig` → `EventManager` → `StartHTTPServer`（默认 13333）→ `StartRPCServer`（默认 13334）。
- 索引实现的主类叫 `PrefixCacheTable`（`mooncake-conductor/include/conductor/prefixindex/prefix_indexer.h:98`），文件名是 `prefix_indexer`，别被文件名误导。

### Reshard

- 纯 Python，无 CMake。`mooncake-reshard/README.md` 定义了 `contracts` / `transfer_engine` / `weight` / `kv_cache` 四个职责面。
- 规划阶段**地址无关**：`WeightPlacementManifest` 只有逻辑张量切分与 N 维片段，运行时绑定才把区域 attest 到不可变 buffer 快照上。
- wheel 安装路径由 `python/CMakeLists.txt:39-45` 决定。

### P2P Store

- Go 实现，`WITH_P2P_STORE=OFF`。
- 官方高性能版已独立为 checkpoint-engine（`README.md:73`）。本仓库内它主要是历史遗留与示例，不要当作生产路径。

### Rust 的三个不同位置

| 位置 | 是什么 | 开关 |
| --- | --- | --- |
| `mooncake-store/rust/` | C++ Store 的 Rust 绑定，走 `store_c.h` C ABI | `WITH_STORE_RUST=ON`（默认） |
| `mooncake-store-rs/` | 独立 Rust 原生 Store，14 个 crate | `WITH_STORE_RS=OFF`（默认） |
| `mooncake-transfer-engine/rust/`、`mooncake-conductor/rust/` | TE / Conductor 的 Rust 接口 | `WITH_RUST_EXAMPLE` / `WITH_CONDUCTOR_RUST` |

### 对外接口契约在哪

| 边界 | 契约位置 |
| --- | --- |
| C++ TE 公开 API | `transfer_engine.h:89`（经典门面）+ `tent/include/tent/transfer_engine.h:256`（TENT）+ 同文件 C API |
| Python TE | `PYBIND11_MODULE(engine)`：`mooncake-integration/transfer_engine/transfer_engine_py.cpp:1743` |
| Python Store | `PYBIND11_MODULE(_store)`：`mooncake-integration/store/store_py.cpp:2137`；后端选择门面 `python/mooncake/store/__init__.py:95-121` |
| Master RPC | `WrappedMasterService` 注册进 coro_rpc；方法签名即 `master_service.h` 的 `tl::expected` 系列 |
| dummy↔real RPC | `real_client_main.cpp:38-111` 的 handler 注册表 |
| store-rs 数据面 | 强制经 TENT；C++ Store 的 Rust 绑定走 `store_c.h` C ABI |
| vLLM 集成 | `mooncake-wheel/mooncake/mooncake_connector_v1.py`：`MooncakeConnector`（`:121`，实现 `KVConnectorBase_V1`）、`MooncakeConnectorScheduler`（`:231`）、`MooncakeConnectorWorker`（`:394`） |
| 独立 Store 服务 | `mooncake-wheel/mooncake/mooncake_store_service.py`：`MooncakeStoreService`（`:68`） |

### cpp/rs 双 Store 实现的 API 差异面

`python/mooncake/store/__init__.py` 用两份 frozenset 白名单精确刻画了差异：`_CPP_EXPORTS`（`:9-59`）与 `_RS_EXPORTS`（`:71-94`）。rs 后端**缺** `RangedReadSnapshot`、`SoftPinAction`、任务/查询类、Engram 等；**多出** `init_tracing`、`start_metrics_server`、`ParallelAxis` / `TensorParallelism` / `ReadTarget` 与 `AXIS_DP/TP/EP/PP`、`READ_MODE_*` 等并行读语义符号。`__getattr__`（`:126-138`）在 rs 后端下访问 C++ 专有符号会抛出带明确提示的 `AttributeError`；cpp 后端还额外放行 `_CPP_INTERNAL_EXPORTS`（`:61-68`）。

---

## 设计取舍与易踩坑点

### 1. 为什么 TE 要有两套实现而不是重构一套？

TENT 的动机写在 `docs/source/design/tent/overview.md:8-19`：经典 TE 假设"一个进程绑定一个 transport 后端"，在同构集群没问题，但现代集群里 NVLink / RDMA / 主机内存混用、链路质量动态变化，静态选路会让慢链路拖垮尾延迟。TENT 把选路、切片调度、故障重试搬进运行时。

代价是**同一个类名两个实现**：`mooncake::TransferEngine` 与 `mooncake::tent::TransferEngine`。任何"TE 是怎么做的"的结论都必须先声明在哪个实现里。经典类在 TENT 模式下只是一层兼容皮，`installTransport` 返回的 `TentCompatibilityTransport` 是个所有方法都 `NotImplemented` 的占位（`transfer_engine.cpp:447-492`）。

### 2. `USE_TENT`（编译期）≠ `MC_USE_TENT`（运行期）

- 编译期 `USE_TENT=OFF`（默认，`common.cmake:193`）时，`MC_USE_TENT` 环境变量**完全无效**——因为没有 TENT 代码可走。
- 编译期 `USE_TENT=ON` 时，默认仍走经典实现，除非设置 `MC_USE_TENT` / `MC_USE_TEV1`。
- 判断运行时到底在哪条路：`TransferEngine::isUsingTent()`（`transfer_engine.h:141`）。Store 的 `InitTransferEngine` 就是这么做的（`client_service.cpp:829-831`），并据此跳过手动 transport 安装。

### 3. `WITH_STORE_RUST` vs `WITH_STORE_RS`

这是全仓最易混的一对开关：

| | `WITH_STORE_RUST` | `WITH_STORE_RS` |
| --- | --- | --- |
| 默认 | ON | OFF |
| 产物 | C++ Store 的 Rust **绑定** | **Rust 原生 Store** |
| 目录 | `mooncake-store/rust/` | `mooncake-store-rs/` |
| 依赖 | `WITH_STORE=ON` | `WITH_TE=ON` + `USE_TENT=ON` + cargo |
| Python 选择 | `MOONCAKE_STORE_BACKEND=cpp` | `MOONCAKE_STORE_BACKEND=rs` |

### 4. "Client" 有三个名字

- `mooncake::Client`（`client_service.h:77`）：内部核心逻辑。
- `PyClient`（`pyclient.h:232`）：Python 侧抽象基类，`RealClient` / `DummyClient` 实现它。
- `MooncakeStorePyWrapper`（`store_py.cpp:533`）：pybind 真正暴露的类，Python 里叫 `MooncakeDistributedStore`。

设计文档里的"Client 类"= 第一个；用户文档里的 "client" 可能是第三个。提 issue 或查日志时要带命名空间。

### 5. Master 不是 Redis，也不是 NameNode+DataNode

Master 只存目录与租约，对象字节永远 Client↔Client。它的内存里只有 metadata 和 allocator 的位图/offset 状态，**没有对象内容**。因此：

- Master 的容量瓶颈是元数据规模，不是数据量；
- 单 Master（默认模式）重启会丢失对象目录、租约与副本状态。对象字节虽然还在 Client 的 segment 里，但没有元数据就无法定位，实际等同于不可读。HA 模式才靠 standby 快照 / OpLog 恢复对象与副本状态（`docs/source/design/store/mooncake-store.md:58-66`），且租约等运行时状态仍要重新建立。
- 把 `mooncake_master` 类比成 Redis server 会推出完全错误的故障模型。

### 6. Store 的元数据服务与 TE 的元数据服务是两套

`docs/source/design/store/mooncake-store.md:30` 明确："Note that the `metadata service` required by the `Transfer Engine` (via etcd, Redis, or HTTP, etc.) is not included in the `Master Service` and needs to be deployed separately."

实践中常见混淆：启动 `mooncake_master --enable_http_metadata_server=true` 后以为不用再配 `metadata_server`。实际上这个内嵌 HTTP 服务是给 TE 做 peer discovery 的，且默认关闭；Store Client 的 `setup(metadata_server=...)` 仍然必须指向一个 TE 元数据服务（或 `P2PHANDSHAKE`）。

### 7. HA：选主成功 ≠ 可以服务

`docs/source/design/store/mooncake-store.md:48` 强调：被选中的节点还必须完成 standby 追赶、导出并校验完整 promotion context、恢复该 context（包括合法的空 context）、再次验证 leadership。任一步失败，节点保持**不可用**而不是带着不完整元数据上线。（`architecture.md:33` 只写到 "leader election and standby recovery"，没有展开这些细节。）把 HA 简化成"etcd 选主即可"会严重误判故障窗口。

### 8. 租约的时钟基准容易读错

`Client::Query` 用 `start_time + lease_ttl_ms`（`client_service.cpp:1250`），`start_time` 是**发起 RPC 之前**的时刻。所以客户端算出的过期时间比 Master 的 `GrantReadLease` 时刻更早，是保守的。另一方面，读传输完成后还有一次 `IsLeaseExpired()` 检查（`:1444`），过期则丢弃已读数据返回 `LEASE_EXPIRED`。这两处共同保证"宁可失败，不返回可能被覆盖的数据"。

### 9. `BatchID` 是指针值，不能落盘/跨进程

`Transport::toBatchDesc` 是 `reinterpret_cast`（`transport.h:162-164`）。任何把 `BatchID` 写进日志后想在另一个进程里复用的想法都是错的。同理，`freeBatchID` 返回 `BatchCleanupDeferred` 时，batch 句柄已失效但底层 buffer 仍被 in-flight 工作持有。

### 10. 两个 `Slice`

`types.h:445` 的 Store `Slice`（`ptr+size` 的暂存描述）与 `transport.h:168` 的 TE `Slice`（带完成状态机与 transport union 的传输切片）同名不同物。跨模块读代码时先确认命名空间，否则会把"Store 的分片"和"TE 的 slice"混为一谈。

### 11. 文档与代码的不一致（已实测）

- `README.md:291` 与 `.claude/skills/mooncake-ci-local/SKILL.md:8` 都把 `scripts/run_ci_test.sh` 写成默认 CI 入口，但 `git ls-files scripts/run_ci_test.sh` 返回 0 条——该文件**未被跟踪**，实际存在的是 `scripts/ci/`（`run_store_api_smoke.sh`、`run_rpc_smoke.sh`、`run_store_rs_smoke.sh`、`services.sh` 等）、`scripts/run_tests.sh`、`scripts/run_standalone_store_e2e.sh`。照抄命令会直接报 file not found。
- 版本号两份且不一致：根 `pyproject.toml:15` 是 `0.3.12.post1`（scikit-build-core 路径），`mooncake-wheel/pyproject.toml:26` 是 `0.3.13`（`scripts/build_wheel.sh` 实际使用的发布路径）。`mooncake-store/CMakeLists.txt:1` 还有第三个版本 `2.0.0`（Store 子项目版本），`master.cpp` 打印的是 `MOONCAKE_DISPLAY_VERSION`。
- 根 `pyproject.toml:105-109` 的 `sdist.include` 写了 `extern/yalantinglibs/**`，但 yalantinglibs 实际由 `mooncake-common/FindYLT.cmake` 用 FetchContent 下载；`.gitmodules` 里只有 `extern/pybind11`。这是历史残留路径。

### 12. 构建路径有两条，产物能力不同

- **根 `pyproject.toml`（scikit-build-core）**：`wheel.packages = ["python/mooncake"]`（`:103`）、`install.components = ["python"]`（`:104`），且 `[tool.scikit-build.cmake.define]` 默认 `USE_CUDA=false`、`WITH_EP=false`、`WITH_STORE_RUST=false`、`WITH_STORE_RS=false`（`:111-119`）。也就是说，直接从源码 `pip wheel .` 得到的是**精简 CPU 构建**。
- **`scripts/build_wheel.sh`（setuptools）**：发布用路径，按变体矩阵 sed 替换 `keywords` / `classifiers`（`mooncake-wheel/pyproject.toml:7-11` 的注释明确要求这两个数组必须单行），并决定是否带 CUDA / NPU / EFA / MUSA / ROCm。

把这两条路径搞混，会得到"为什么我本地 pip install 出来的包没有 EP"之类的困惑。

### 13. 统计代码量时先排除 vendored 代码

`mooncake-store/include/cachelib_memory_allocator/` 是 vendored 的 Facebook CacheLib slab 分配器，内部还自带一份 fmt（`core.h` 约 3008 行、`format.h` 约 2830 行）。`mooncake-store/AGENTS.md` 专门写了"不要迁移 cachelib_memory_allocator，除非明确要求"。统计"项目自身代码量"时必须排除，否则会被几万行第三方代码带偏。

### 14. `mooncake-reshard` / `mooncake-rl` / `benchmarks` 不是构建模块

- `mooncake-reshard` 没有 `CMakeLists.txt`，靠 `python/CMakeLists.txt` 的 `install(DIRECTORY)` 进 wheel。
- `mooncake-rl` 只有一个 `examples/rl_samples.py`，顶层 CMake 里没有它；真正的 RL 集成在 verl / Miles / TransferQueue / SGLang。
- `benchmarks/` 是独立基准，不参与默认构建。

### 15. Transport 数量要分侧数

经典 22 个子目录、TENT 19 个，不是一一对应。README 的协议清单是合集；写"支持 N 种协议"时必须说明是哪个引擎、是否包含 `device` / `rpc_communicator` 这类非纯 transport 的实现。

### 16. `global_segment_size=0` / `local_buffer_size=0` 是两种退化形态

- `global_segment_size = 0`：纯客户端，只发请求不贡献内存（`docs/source/design/store/mooncake-store.md:37`）；
- `local_buffer_size = 0`：纯服务端，只贡献内存，**不允许**发起 Get/Put（同上 `:38`）。

这两种配置在部署脚本里很常见，但语义完全相反，读配置时先确认是哪一种。默认 `DEFAULT_GLOBAL_SEGMENT_SIZE` 是 16MB（`types.h:244`）。

### 17. FAST25 trace 的 hash 块大小

`FAST25-release/README.md:55`：`hash_ids` 以 **512 token** 为块，相同 hash 前缀即可复用 KVCache。这直接对应 Conductor 的前缀索引与 SGLang HiCache L3 的匹配语义，是理解"为什么需要全局前缀索引"的数据基础。`FAST25-release/` 里只有三份 jsonl + 论文 PDF + 历史 arxiv trace，**没有任何代码**。

---

## 精读清单

下面是一条经过验证的阅读路径，每一步都给出"读什么"和"该看懂什么"。

### 第 1 步：建立全局印象（约 30 分钟）

1. `docs/source/design/architecture.md`（78 行，全仓性价比最高）——看懂四条原则、五个组件、场景映射、Repository map。
2. `README.md:94-210`——看懂 Show Cases 与 SGLang/vLLM 集成细节；`README.md:211-251` 的生态表留作反查索引，不必逐行读。
3. `docs/source/index.md:148-241`——看懂文档分区，知道后续该去哪里查。

**验收**：能不看笔记说出"TE / Store / PG+EP / Conductor / Reshard 各解决什么问题"以及"哪条原则说 Master 不搬数据"。

### 第 2 步：先读构建系统，再看代码

1. `CMakeLists.txt:19-40`——开关真值表。
2. `CMakeLists.txt:59-123`——互斥校验与依赖提供（zstd/xxhash/libzmq/liburing/hiredis）。
3. `CMakeLists.txt:128-174`——`add_subdirectory` 顺序（common → TE → Store → Conductor → Rust 绑定 → store-rs）。
4. `CMakeLists.txt:176-267`——EP/PG 的 JIT staging 逻辑。
5. `mooncake-common/common.cmake:182-200`——硬件/协议开关；`:316-350` 的 TENT 强制约束。

**验收**：给定一个 `-D` 组合，能判断会不会触发 FATAL_ERROR，以及最终 wheel 里有哪些模块。

### 第 3 步：TE 门面与双实现

1. `mooncake-transfer-engine/include/transfer_engine.h:89-365`——先只看成员变量（`:353-364`）和几个关键方法：`init` / `installTransport` / `openSegment` / `registerLocalMemory` / `submitTransfer` / `getTransferStatus`。
2. `mooncake-transfer-engine/src/transfer_engine.cpp:534-542` 与 `662-708`——构造与 init 的分叉。
3. `transfer_engine.cpp:726-743` 与 `447-492`——TENT 兼容层。
4. `transfer_engine.cpp:773-798`——`openSegment` / `CheckSegmentStatus` 在 TENT 下的语义修正。

**该看懂**：同一个公开 API 如何路由到两套实现；`MC_USE_TENT` 与 `USE_TENT` 的区别；为什么 `installTransport` 的返回值不能作为"是否成功"的依据。

### 第 4 步：Transport 契约与批次模型

1. `mooncake-transfer-engine/include/transport/transport.h:44-99`——`TransferRequest` / `TransferStatus` / `TransferStatusEnum`。
2. `transport.h:151-164`——`BatchID ↔ BatchDesc` 的指针转换与生命周期契约。
3. `transport.h:168-310`——`Slice` 的 union、`markSuccess` / `markFailed`、seal 位协议。
4. `transport.h:315-402`——`finishIfComplete` 与 `CompletionCallbackPin`。
5. `transport.h:453-530`——`TransferTask` / `BatchDesc` 字段。
6. `transport.h:601-602`——`batch_desc_set_` 与 `RWSpinlock`。

**该看懂**：一次批量传输的完成判定是"seal 位 + 计数相等 + `completion_published` 只发一次"；`freeBatchID` 为什么必须延迟。

### 第 5 步：经典 TE 的初始化实现

1. `mooncake-transfer-engine/src/transfer_engine_impl.cpp:108-234`——RPC 端点选择、metadata、MultiTransport、`addRpcMetaEntry`。
2. `transfer_engine_impl.cpp:240-302`——`MC_FORCE_TCP` / `MC_FORCE_SHM` / Ascend 分支。
3. `mooncake-transfer-engine/src/transfer_engine_impl.cpp:780` 起的 `registerLocalMemory`——两阶段预留与回滚（细节可留给第 02 章）。

**该看懂**：初始化顺序为什么是"先 RPC 元数据、再 transport"；auto discovery 与显式 `installTransport` 的分工。

### 第 6 步：TENT 的声明式模型

1. `mooncake-transfer-engine/tent/include/tent/transfer_engine.h:256-330`——`exportLocalSegment` / `importRemoteSegment` / `allocateLocalMemory` / `allocateBatch`。
2. `docs/source/design/tent/overview.md:21-47`——动态选路、细粒度调度、运行时故障处理三个设计选择。
3. `mooncake-transfer-engine/src/transfer_engine.cpp:642-660`——`buildTentConfig` 如何把经典连接串翻译成 TENT 配置。

**该看懂**：TENT 的 API 是"声明要搬什么"，经典 TE 的 API 是"提交一条请求"。这个差异决定了上层是否需要自己选 transport。

### 第 7 步：Store 控制面契约

1. `mooncake-store/include/master_service.h:101-118`——**锁顺序**，必须逐条记住。
2. `master_service.h:180-235`——`MountSegment` / `ReMountSegment` / `UnmountSegment` 的幂等语义。
3. `master_service.h:394-470`——`GetReplicaList` / `BatchGetReplicaList` / `PutStart` / `PutEnd`。
4. `master_service.h:1098-1135`——1024 分片与 `MetadataShard`。
5. `mooncake-store/src/master_service.cpp:4777-4833`——读路径授租约。
6. `master_service.cpp:5589-5661`——PutStart 的配额、stale 清理、分配。

**该看懂**：为什么所有 API 返回 `tl::expected<T, ErrorCode>`；为什么 `BatchGetReplicaListForAdmin` 要单独存在。

### 第 8 步：Store 数据面（C++）

1. `mooncake-store/include/client_service.h:43-146`——`QueryResult` / `Client` / `Create` / `Get`。
2. `mooncake-store/src/client_service.cpp:809-1010`——`InitTransferEngine` 的 TENT 判断与 transport 安装（TENT 判断在 `:829-831`，安装分支从 `:918` 起）。
3. `client_service.cpp:1046-1166`——`Create` 的完整初始化顺序。
4. `client_service.cpp:1959-2108`——`Put` 全链路。
5. `client_service.cpp:1168-1252` 与 `1382-1456`——`Get` 与租约检查。
6. `client_service.cpp:4698-4729`——`TransferData` 如何落到 `TransferSubmitter`。

**该看懂**：Put 的"预留 → 写副本 → PutEnd/Revoke"三段式；Get 的"Query（含租约）→ 选副本 → 传输 → 校验 → 复查租约"五段式。

### 第 9 步：Store 数据面（Python 侧）

1. `mooncake-store/include/pyclient.h:232-343`——`PyClient` 的方法清单。
2. `mooncake-store/include/real_client.h:83-150` 与 `dummy_client.h:21-70`——两个实现。
3. `mooncake-integration/store/store_py.cpp:533-620`——`MooncakeStorePyWrapper` 与 `init_real_client`。
4. `store_py.cpp:2440-2560`——`setup` / `setup_dummy` 的绑定与参数。
5. `python/mooncake/store/__init__.py:9-138`——cpp/rs 后端选择与白名单。

**该看懂**：为什么 Python 里一个 `MooncakeDistributedStore` 背后可能是 RealClient、DummyClient，甚至 Rust Store。

### 第 10 步：Store-RS 的第二实现

1. `mooncake-store-rs/README.md`——构建前提与 `MOONCAKE_STORE_BACKEND`。
2. `docs/source/design/store/store-rs/architecture.md` 的 **Hot-Path Invariant** 与 **Component Model** 表。
3. `mooncake-store-rs/CMakeLists.txt:7-12`——为什么强制 `WITH_TE=ON` + `USE_TENT=ON`。

**该看懂**：store-rs 不是移植，而是"没有专用 Master、路由靠 WRH、热路径不碰后端"的另一套设计。

### 第 11 步：写路径与读路径的端到端串联

建议在读完第 7-9 步后，回到 `docs/source/design/store/mooncake-store.md:52-195` 与 `:352-415`，把文档里的 PutStart/PutEnd、租约、软/硬 pin、僵尸对象清理与代码对上：

- 租约：`master_service.cpp:4832` + `client_service.cpp:1250/1444`；
- 软 pin：`replica.h:112-115` + `docs/source/design/store/mooncake-store.md:370-389`；
- 硬 pin：`replica.h:116` + `docs/source/design/store/mooncake-store.md:391-401`；
- 僵尸对象：`put_start_discard_timeout` / `put_start_release_timeout`（`docs/...:405-413`）。

**该看懂**：文档里的每个策略字段，都能在 `ReplicateConfig` 或 master flag 里找到落点。

### 第 12 步：把边界画出来

最后读三份"边界文档"，确认自己没有把组件混在一起：

1. `docs/source/design/mooncake-backend-pg.md:180-248`——PG 的 Coordinator/Agent/Collective/P2P。
2. `docs/source/design/mooncake-ep.md:42-60` 与 `:104-118`——EP 与 PG 的关系、fast path 与 fallback。
3. `docs/source/design/conductor/conductor-architecture-design.md:14-92`——Conductor 的 HTTP/RPC 双端口、事件流与查询流。
4. `mooncake-reshard/README.md` 的 Public API Migration 节——规划与绑定的分离。

**验收**：能回答"Conductor 需不需要 TE？"（不需要）、"EP 能不能脱离 PG 单独跑？"（可以，但通常从 PG bootstrap）、"Reshard 会不会搬数据？"（规划阶段不会，执行阶段通过 adapter 调 TE）。

### 第 13 步：打包与运维（可选但强烈建议）

1. `scripts/build_wheel.sh:39-160`——wheel 里到底装了什么。
2. `python/CMakeLists.txt:15-45`——命名空间包的三处来源。
3. `pyproject.toml:89-120` 与 `mooncake-wheel/pyproject.toml:20-58`——两个打包路径的差异。
4. `mooncake-store/src/master.cpp:1411-1769`——master 的启动、HA 与非 HA 分支、优雅退出。

**验收**：能解释"为什么 `pip install mooncake-transfer-engine` 的包里有 `mooncake_master` 可执行文件"以及"为什么从源码 `pip wheel .` 得到的包可能没有 EP"。

---

## 附：一页纸速查

| 问题 | 答案 | 依据 |
| --- | --- | --- |
| Mooncake 的中心抽象？ | KVCache-centric、PD 分离、全局 KVCache 池 | `README.md:101-103` |
| 有几层？ | TE / Store / PG+EP / Conductor / Reshard（+ P2P Store 旁路） | `architecture.md:16-50` |
| Master 搬数据吗？ | 不搬 | `docs/.../mooncake-store.md:56` |
| 数据怎么走？ | Client ↔ Client 直传，走 TE | `architecture.md:12`, `mooncake-store.md:34` |
| 有几个 TransferEngine？ | 两个：经典 + TENT，靠 `USE_TENT`/`MC_USE_TENT` 分叉 | `transfer_engine.h:353-364`, `transfer_engine.cpp:534-542` |
| Store 有几个实现？ | 两套：C++（`WITH_STORE`）与 Rust（`WITH_STORE_RS`） | `CMakeLists.txt:20/29`, `store/__init__.py:95-121` |
| 元数据服务有几套？ | 两套：TE 的 peer discovery 与 Store 的 master | `mooncake-store.md:30` |
| 默认租约多久？ | 10 秒（可配） | `types.h:93-94`, `master.cpp:42-52` |
| 默认分片数？ | 1024 | `master_service.h:1098` |
| 构建开关在哪？ | 顶层 `CMakeLists.txt:19-40` + `common.cmake` 51 个 option | 已核对 |
| 发布 wheel 怎么打？ | `scripts/build_wheel.sh` → `cd mooncake-wheel` → setuptools | `build_wheel.sh:244` |
| 最小可跑路径？ | pip install → `mooncake_master` → `store.setup(...)` | `README.md:263-278`, `quick-start.md` |

读完本章并走完精读清单，你应当能独立回答：某个功能属于哪一层、由哪个 CMake 开关控制、Python 符号从哪个 `.so` 导出、数据在哪些进程之间流动、控制消息走哪条 RPC。之后再进入各子系统专章（Transfer Engine/TENT、Store Master、Store Client、PG/EP、Conductor/Reshard）就是纯纵深问题了。
