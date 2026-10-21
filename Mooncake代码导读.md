# Mooncake 代码导读

> 仓库：`D:/xiaojun/work/Mooncake`  
> 分支 `main`，HEAD `67df8c41`  
> 生成日期：2026-10-21  
> 覆盖：16 个子系统，约 25 万行 C/C++、18 万行 Rust，以及 Python/Go 集成层

---

## 这本书怎么读

Mooncake 是 Moonshot AI 为 Kimi 推理服务构建、获 FAST'25 最佳论文的**以 KVCache 为中心的分离式基础设施**。它不训练模型、不调度请求，只做两件事：把数据搬得足够快，把分散在各节点的内存和 SSD 组织成一个可寻址的对象池。

贯穿全书只有一条不变量：

> **控制面只做决策，数据面才搬字节；控制面永不接触数据本身。**

### 四条推荐路线

| 路线 | 适合谁 | 章节顺序 |
| --- | --- | --- |
| A · KVCache 存取全链路 | 想搞懂 put/get 端到端 | 01 → 02 → 06 → 07 → 03 |
| B · 传输层 | 要改 TE / 排查 RDMA | 02 → 03 → 04 → 05 |
| C · Python 接入 | 要扩 API / 打包 | 01 → 13 → 07 → 11 |
| D · 运维与故障排查 | 要部署、定位问题 | 16 → 08 → 09 → 11 → 06 |

### 三个必须先知道的陷阱

1. **`TransferEngine` 是两个类**：经典的 `mooncake::TransferEngine` 只是门面，内部同时持有 `impl_` 与 `impl_tent_`，靠编译宏 `USE_TENT` 加环境变量 `MC_USE_TENT` / `MC_USE_TEV1` 决定路由。
2. **`BatchTransfer` 在代码中不存在**：它只是设计文档里的概念词，落地是 `BatchID` + `TransferRequest` 列表 + `getBatchTransferStatus()`。`BatchID` 字面上就是 `new BatchDesc()` 的指针（`transport.h:162`）。
3. **Master 一个字节都不搬**：它只返回"副本在哪个 Client 的哪个 offset"，真实数据由 Client ↔ Client 经 Transfer Engine 直传；而 Client 本身往往也是存储节点。

---

## 全书目录

- **01 项目全貌与模块地图**
  - 定位与职责
  - 目录结构速览
  - 核心数据结构
  - 关键流程剖析
  - 与其他模块的边界
  - 设计取舍与易踩坑点
  - 精读清单
  - 附：一页纸速查
- **02 Transfer Engine 核心**
  - 定位与职责
  - 目录结构速览
  - 核心数据结构
  - 关键流程剖析
  - 与其他模块的边界
  - 设计取舍与易踩坑点
  - 精读清单
- **03 RDMA 传输实现**
  - 定位与职责
  - 目录结构速览
  - 核心数据结构
  - 关键流程剖析
  - 与其他模块的边界
  - 设计取舍与易踩坑点
  - 精读清单
- **04 其它传输后端与共享内存段**
  - 定位与职责
  - 目录结构速览
  - 核心数据结构
  - 关键流程剖析
  - 与其他模块的边界
  - 设计取舍与易踩坑点
  - 精读清单
- **05 TENT 新一代传输引擎**
  - 定位与职责
  - 目录结构速览
  - 核心数据结构
  - 关键流程剖析
  - 与其他模块的边界
  - 设计取舍与易踩坑点
  - 精读清单
- **06 Store Master 服务与放置策略**
  - 定位与职责
  - 目录结构速览
  - 核心数据结构
  - 关键流程剖析
  - 与其他模块的边界
  - 设计取舍与易踩坑点
  - 精读清单
- **07 Store Client 与数据面**
  - 定位与职责
  - 目录结构速览
  - 核心数据结构
  - 关键流程剖析
  - 与其他模块的边界
  - 设计取舍与易踩坑点
  - 精读清单
- **08 高可用与主备复制**
  - 定位与职责
  - 目录结构速览
  - 核心数据结构
  - 关键流程剖析
  - 与其他模块的边界
  - 设计取舍与易踩坑点
  - 精读清单
- **09 持久化与分层存储后端**
  - 定位与职责
  - 目录结构速览
  - 核心数据结构
  - 关键流程剖析
  - 与其他模块的边界
  - 设计取舍与易踩坑点
  - 精读清单
- **10 内存分配器体系**
  - 定位与职责
  - 目录结构速览
  - 核心数据结构
  - 关键流程剖析
  - 与其他模块的边界
  - 设计取舍与易踩坑点
  - 精读清单
- **11 Store 特性层**
  - 定位与职责
  - 目录结构速览
  - 核心数据结构
  - 关键流程剖析
  - 与其他模块的边界
  - 设计取舍与易踩坑点
  - 精读清单
- **12 Rust / Go 实现**
  - 定位与职责
  - 目录结构速览
  - 核心数据结构
  - 关键流程剖析
  - 与其他模块的边界
  - 设计取舍与易踩坑点
  - 精读清单
- **13 Python 绑定与打包**
  - 定位与职责
  - 目录结构速览
  - 核心数据结构
  - 关键流程剖析
  - 与其他模块的边界
  - 设计取舍与易踩坑点
  - 精读清单
- **14 Expert Parallelism 与 PyTorch Backend**
  - 定位与职责
  - 目录结构速览
  - 核心数据结构
  - 关键流程剖析
  - 与其他模块的边界
  - 设计取舍与易踩坑点
  - 精读清单
- **15 Reshard / Conductor / RL / P2P Store / HiCache**
  - 定位与职责
  - 目录结构速览
  - 核心数据结构
  - 关键流程剖析
  - 与其他模块的边界
  - 设计取舍与易踩坑点
  - 精读清单
- **16 构建、测试与运维**
  - 定位与职责
  - 目录结构速览
  - 核心数据结构
  - 关键流程剖析
  - 与其他模块的边界
  - 设计取舍与易踩坑点
  - 精读清单

---

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

---

# 02 Transfer Engine 核心

> 本章精读 `mooncake-transfer-engine/` 的**引擎核心层**：对外门面 `TransferEngine`、状态持有者 `TransferEngineImpl`、批量传输语义所有者 `MultiTransport`、公共契约 `Transport`，以及四个旁路基础设施——`TransferMetadata`（Segment/RPC 元数据 + 插件）、`Topology`（设备发现与选卡）、`GlobalConfig`（`MC_*` 环境变量）、`memory_location`（buffer 的 location 标签）。`graceful_shutdown.cpp` 与 `show_links.cpp` 作为两个小而关键的辅助件一并讲。
>
> **行号基准**：仓库 commit `67df8c41`，路径相对仓库根 `D:/xiaojun/work/Mooncake`。源码会漂移，引用前请以函数名重新定位。
>
> **先确认编译分支**：
> - `src/transfer_engine.cpp` 一个文件里有两套完整实现：`#ifndef USE_TENT`（`:38-385`，classic 转发层）与 `#else`（`:386-1254`，TENT 兼容 shim），`:1256` 之后是不受该宏约束的共享段。默认构建 `USE_TENT=OFF`（`mooncake-common/common.cmake:193`），下文除特别说明均指 **classic TE**。
> - `ENABLE_MULTI_PROTOCOL` 默认 `OFF`（`common.cmake:194-199`）；`USE_EVENT_DRIVEN_COMPLETION` 默认 `OFF`（`common.cmake:190-191`）。这两个宏决定了多处代码是否参与编译。
>
> **术语校正**：范围提示里的 "BatchTransfer" 在代码中**不存在同名类型**。它是 `docs/source/design/transfer-engine/index.md:4` 的概念名词；落到代码是 `BatchID` + `std::vector<TransferRequest>` + `getTransferStatus`/`getBatchTransferStatus` 这组 API。仓库里没有名为 `BatchTransfer` 的类型；`grep BatchTransfer` 命中的只是 `getBatchTransferStatus`、Store 侧的 `BatchTransferReadRanges` 以及文档/示例里的 API 名。

---

## 定位与职责

Transfer Engine（下称 TE）是 Mooncake 的**零拷贝异步数据搬运底座**。它把"本地一段已注册内存 ↔ 远端 Segment 的一段地址"抽象成一组可批量提交、异步查询完成的请求，屏蔽底层用哪条链路（RDMA / TCP / EFA / CXI / NVLink / SHM / CXL / NVMeoF / Ascend / Kunpeng UB …）。

它**只做搬运**：不含 KV 语义、不管理对象生命周期、不做副本一致性。上层的 `mooncake-store`、`mooncake-integration/transfer_engine`（Python 绑定）、`mooncake-p2p-store`（Go/cgo）、`mooncake-ep`、`mooncake-reshard` 都通过这一层发起跨节点数据移动。

纵向三层：

| 层 | 类 | 职责边界 |
|---|---|---|
| 门面 | `TransferEngine`（`include/transfer_engine.h:89`） | 对外唯一 API 面；classic 模式下几乎是纯转发；TENT 模式下是兼容 shim |
| 引擎 | `TransferEngineImpl`（`include/transfer_engine_impl.h:82`） | 生命周期（`init`/`freeEngine`）、transport 安装决策树、本地内存区域表（两阶段预留）、notify/probe、metrics |
| 路由与批次 | `MultiTransport`（`include/multi_transport.h:33`） | `BatchDesc` 的唯一所有者；按 Segment 协议选 transport；提交/完成状态聚合；延迟清理线程 |

旁路两个基础设施：`TransferMetadata`（Segment/RPC 元数据的发布、缓存、版本仲裁，三种存储插件 + 点对点握手插件）与 `Topology`（location → preferred/avail HCA 的优先级矩阵，负责设备发现与选卡）。

两个核心概念：

- **Segment**：一个进程内一个 `TransferEngine` 实例对应一个本地 Segment（`LOCAL_SEGMENT_ID = 0`），逻辑上覆盖"已注册 buffer 的集合"；远端 Segment 用 `openSegment(name)` 取得一个**本进程内有效的** `SegmentID`。代码里没有 `Segment` 类，实体是 `TransferMetadata::SegmentDesc`（一份 JSON）。
- **批量传输**：`allocateBatchID(n)` 拿到一个批次句柄，往里面 `submitTransfer` 一批 `TransferRequest`，用 `getTransferStatus(batch_id, task_id)` 或 `getBatchTransferStatus(batch_id)` 轮询，最后 `freeBatchID`。一个请求在引擎内部可能被切成多个 Slice，甚至（scatter 路径下）多个请求被合并成一个 TransferTask。

---

## 目录结构速览

| 文件 | 一句话职责 |
|---|---|
| `include/transfer_engine.h` | 门面类 `TransferEngine` 的公开 API 与 `ScatterTransferRange`/`ScatterTransferOperation`/`AutoDiscoverConfig`/`PeerLiveness` 等类型 |
| `src/transfer_engine.cpp` | 门面方法体。`:38/386/1254` 用 `#ifndef USE_TENT` / `#else` / `#endif` 分成 classic 转发层与 TENT shim；`:1256` 之后是共享段（`getSegmentBuffers`、`ScatterTransferOperation::Impl`） |
| `include/transfer_engine_impl.h` | `TransferEngineImpl` 声明。热路径 `submitTransfer`/`getTransferStatus`/`getBatchTransferStatus`/`freeBatchID` 大量内联在这里；文件顶部还有 `shouldInstallAscendTransport` 编译期判定 |
| `src/transfer_engine_impl.cpp` | 引擎主体：`init()` 的 RPC 端口绑定与 transport 安装决策树；`registerLocalMemory` 的两阶段区域预留与逆序回滚；metrics 线程 |
| `include/multi_transport.h` | `MultiTransport` 声明：`transport_map_`、`ScatterSubmission`、延迟清理线程成员、`selectTransport(s)` |
| `src/multi_transport.cpp` | 批量传输语义核心：`allocateBatchID`/`freeBatchID`/`tryFreeBatchID`/`deferredCleanupLoop`、`submitTransfer` 的 task 切分与 seal、状态聚合、`installTransport` 的协议→类工厂 |
| `include/transport/transport.h` | 所有 transport 的公共契约：`TransferRequest`/`TransferStatus`/`Slice`/`TransferTask`/`BatchDesc`/`SliceTaskPtr`/`ThreadLocalSliceCache` 与完成协议 |
| `src/transport/transport.cpp` | `Transport` 基类默认实现：thread-local slice 缓存实例、基类版 `allocateBatchID`/`freeBatchID`、`install()` |
| `include/transfer_metadata.h` | `TransferMetadata` 声明与全部元数据 POD：`SegmentDesc`/`BufferDesc`/`DeviceDesc`/`RpcMetaDesc`/`HandShakeDesc`/`NotifyDesc`，`GetResult` 三态，segment 缓存成员 |
| `src/transfer_metadata.cpp` | 元数据编解码 + 本地缓存 + 版本仲裁：`encodeSegmentDesc`/`decodeSegmentDesc`、`getSegmentDescInternal`、`syncSegmentCache`、`addRpcMetaEntry`/`startHandshakeDaemon` |
| `src/transfer_metadata_dump.cpp` | `SegmentDesc::dump()` 与节流的 `dumpMetadataContent()`，纯日志辅助 |
| `include/transfer_metadata_plugin.h` | `MetadataStoragePlugin` 与 `HandShakePlugin` 两个接口的契约与注释 |
| `src/transfer_metadata_plugin.cpp` | `RedisStoragePlugin`/`HTTPStoragePlugin`/`EtcdStoragePlugin`、`SocketHandShakePlugin`、`MetadataStoragePlugin::Create` 分派、`findLocalIpAddresses`/`findAvailableTcpPort` |
| `include/topology.h` | `TopologyEntry`/`TopologyMatrix`/`Topology`/`ResolvedTopologyEntry`；`discover`/`parse`/`selectDevice` 系列 |
| `src/topology.cpp` | `listInfiniBandDevices`（`MC_TE_FILTERS` 白名单、EFA 排除）、`discoverCpuTopology`、`discoverCudaTopology`、`resolve()`、`selectDevice` |
| `include/config.h` | `GlobalConfig` 全量字段与语义注释；`globalConfig`/`loadGlobalConfig`/`ValidatePortRange` |
| `src/config.cpp` | 约 100 个 `MC_*` 环境变量到 `GlobalConfig` 的解析、校验与 `dumpGlobalConfig`；`std::call_once` 单例 |
| `include/memory_location.h` | location 字符串模型：`"*"`、`"cpu:N"`、`"cuda:N"`、`"segments:<page>:<numa,...>"` |
| `src/memory_location.cpp` | 读 `/proc/self/smaps` 判页大小、按 NUMA/CUDA 给 buffer 打标签、segments 编码的按偏移解析 |
| `src/graceful_shutdown.cpp` | 进程级优雅退出：`ShutdownToken` 注册表、`atexit`、SIGTERM/SIGINT + self-pipe + detached watcher |
| `src/show_links.cpp` | 诊断渲染器：`collectLocalNics` 汇总 RdmaContext 的 GID/NUMA/速率，输出链路体检报告 |
| `src/transfer_engine_c.cpp` + `include/transfer_engine_c.h` | C ABI；注意 `segment_id_t` 是 `int32_t`，与 C++ 的 `uint64_t SegmentID` 存在窄化 |

---

## 核心数据结构

### 3.1 Segment：`SegmentDesc` + `SegmentID`

`include/transfer_metadata.h:120-185`。**代码里没有 `Segment` 类**。一个 Segment 由三样东西共同表达：

1. 一份 `SegmentDesc`（可 JSON 序列化的结构体）；
2. 一个进程内自增分配的 `SegmentID`（`next_segment_id_` 从 1 起，`transfer_metadata.cpp:305`；本地段固定 0，`include/common.h:63`）；
3. 一张 `SegmentID → shared_ptr<SegmentDesc>` + `name → SegmentID` 的缓存表（`transfer_metadata.h:352-355`）。

`SegmentID` **不是全局 ID**，只是本进程 `TransferMetadata` 实例内的一个句柄（映射到本地缓存里的 `SegmentDesc`）。两个进程对同一个远端 segment 可能分到不同 ID；`syncSegmentCache` 失效后重新 `openSegment` 还可能拿到新 ID，旧句柄随之失效（见 §D.4）。

关键字段：

| 字段 | 含义 |
|---|---|
| `name` | 全局唯一标识，通常 `ip:port`；作为元数据 key 的一部分 |
| `protocol` | 路由依据。默认构建是单值 `"rdma"`/`"tcp"`/…；`ENABLE_MULTI_PROTOCOL` 下多个 transport 在 `install()` 里**追加逗号**（`rdma_transport.cpp:804-809`、`tcp_transport.cpp:453-462`），形成 `"rdma,hip"` |
| `metadata_version` | 墙上时钟微秒，严格单调（`transfer_metadata.cpp:71-78`），用于版本仲裁 |
| `devices` | `DeviceDesc{name, lid, gid, eid}`；**下标即 `device_id`**，被 `BufferDesc::lkey/rkey` 按下标索引 |
| `topology` | 发布端整份 `Topology` 优先级矩阵（JSON 字段名 `priority_matrix`）。**远端选卡用的是远端发布的拓扑** |
| `buffers` | `BufferDesc` 列表。**`buffers[i].name` 存的是 location 字符串**（`cpu:0`/`cuda:1`/`*`/`segments:...`），不是名字 |
| `buffer_range_index` | 由 `buffers` 派生的地址有序快照，`rebuildBufferRangeIndex()` 必须在每次改动 `buffers` 后、发布前调用（`transfer_metadata.h:128-132`） |
| `interned_nic_paths` | `vector<const string*>`，指向进程级 intern pool 的 `server@nic` 字符串，避免每 slice 构造 std::string |
| `tcp_data_host`/`tcp_data_port`/`tcp_proto_version` | TCP 数据面端点与协议版本（v2 带确认） |
| `rdma_server_name` | 双网卡（`MC_RDMA_BIND_ADDRESS`）下 RDMA 可达地址；`nicPathServerName()` 优先用它（`transfer_metadata.h:176-178`） |
| `nvmeof_buffers`/`cxl_name`/`cxl_base_addr`/`rank_info` | NVMeoF/CXL/Ascend 专用 |

**SegmentDesc 不是 `TransferEngineImpl` 建的**，而是各 transport 在 `install()` 里调 `metadata_->addLocalSegment(LOCAL_SEGMENT_ID, ...)` 建的。以 RDMA 为例：`RdmaTransport::install`（`rdma_transport.cpp:204`）在 `:237` 调 `allocateLocalSegmentID()`（`:794-821`），后者拷贝已有 desc（若存在）、设 `name`/`rdma_server_name`/`protocol`、把每个 `RdmaContext` 的 `deviceName/lid/gid` 追加进 `devices`、把 `local_topology_` 整份拷进 `desc->topology`，最后 `addLocalSegment`。TCP 同理（`tcp_transport.cpp:453`）。

所以"每个进程只有一个 Segment"更准确的说法是：**每个 `TransferEngine` 实例（每个 `TransferMetadata` 实例）有一个本地 Segment**。同进程里多个 engine（测试、P2P + Store 混跑）各自有独立的 `LOCAL_SEGMENT_ID = 0`。

发布路径：`addLocalSegment` 先 `rebuildInternedNicPaths()`，再在写锁下同时插入 id→desc 与 name→id（`transfer_metadata.cpp:1802-1810`）。buffer 增删走 **copy-on-write**：写锁下 `make_shared<SegmentDesc>(*old)`、改副本、重建派生索引、换回表里（`addLocalMemoryBuffer`，`:1822-1834`）。这样在飞的传输持有旧 `shared_ptr`，看到的永远是一致快照。

### 3.2 `TransferRequest`：一次搬运意图

`include/transport/transport.h:60-84`。

```cpp
struct TransferRequest {
    enum OpCode { READ, WRITE };
    OpCode opcode;
    void *source;              // 本地已注册地址
    SegmentID target_id;       // openSegment 得到的本地缓存 ID
    uint64_t target_offset;    // 远端地址：RAM 段是远端虚拟地址，NVMeoF 段是文件偏移
    size_t length;
    int advise_retry_cnt = 0;  // 建议的重试起点（影响首次选卡是否随机）
    int transport_hint = 0;    // TENT only
    int intent_type = 0;       // TENT only
    uint64_t task_group_id = kNoTaskGroup;  // 仅 scatter 路径用于合并
    int priority = PRIO_MEDIUM; // TENT only
    int nic_hint = -1;         // classic RDMA 专用：本地 RNIC 在 hca_list_ 的下标
};
```

要点：

- `READ` 表示"从 `<target_id, target_offset>` 读到 `source`"，`WRITE` 反之。
- `source` 必须已 `registerLocalMemory`。单个 **slice** 必须完整落在同一注册 buffer（MR）内（transport 按该 buffer 选 RNIC、取 lkey），但**一个 `TransferRequest` 可以跨多个 MR**：RDMA 的 `SliceLengthCalculator`（`rdma_transport.cpp:121-150`）会按本地/远端 MR 边界把请求切成多个 slice，注释明确写着 “capped at MR boundary to avoid multi-MR WRs”。
- `target_offset` 对 RAM Segment 是**远端虚拟地址**，不是相对偏移；多协议路由里 CXL 是唯一例外（`buffer.offset + cxl_base_addr`，见 `multi_transport.cpp:835-838`）。
- `task_group_id` 只在 `submitTransfer(..., task_sizes)` 这条 scatter 内部路径上参与合并；普通 `submitTransfer(batch_id, entries)` 永远忽略它（见 §A.2）。
- `nic_hint` 只影响 RDMA transport 的**首次**提交，重试会丢弃它（`rdma_transport.cpp:1032-1034`）。

### 3.3 `TransferTask`：可完成的工作单元

`include/transport/transport.h:453-496`。

```text
volatile uint64_t slice_count / success_slice_count / failed_slice_count
volatile bool     completion_published
volatile uint64_t completion_state      // bit63=seal, bits0..62=已完成 slice 数
volatile uint64_t transferred_bytes
volatile bool     is_finished
uint64_t          total_bytes
BatchID           batch_id
Transport*        transport_             // 提交时写入，用于委托状态查询
const TransferRequest* request           // 指向调用方 entries[i]（Ascend 异构下是可变指针）
size_t            request_count = 1      // >1 表示合并了相邻同组请求
std::vector<Slice*> slice_list           // 析构时逐个归还 thread-local 池
```

要点：

1. **cacheline 布局是刻意的**（`transport.h:457-460` 注释）：`slice_count/success/failed/completion_published/completion_state/transferred_bytes` 要尽量同处一条 cacheline。`markSuccess` 会 RMW `success_slice_count` 与 `completion_state`；若后者越过 offset 64，就会在两个 CQ worker 之间来回弹两条 line。
2. `completion_state` 默认值是 `kCompletionSealBit`（**已密封**），让测试与遗留路径无需显式 seal；支持计数器完成的 transport 会在提交前 `unseal`、提交后 `seal`（`multi_transport.cpp:302-306`、`:315-319`）。
3. `request` 是裸指针：**调用方必须保证 `entries` 在 batch 完成/释放前不被销毁**。`getScatterRequestStatuses` 会在完成后读 `task.request[i].length`（`multi_transport.cpp:486`）；部分 transport 的后台路径也可能继续读它。
4. `~TransferTask()` 把 `slice_list` 里每个 slice 归还给 `Transport::getSliceCache()`（`transport.h:492-495`）——slice 是池化对象，不是直接 `delete`。

### 3.4 `BatchDesc` 与 `BatchID`

`include/transport/transport.h:498-530`。`BatchDesc` 持有 `task_list`（**按值存放的 `vector<TransferTask>`**）、完成标志与计数：

```cpp
BatchID id;                 // 等于 this 指针
size_t batch_size;          // 容量上限
std::vector<TransferTask> task_list;
std::atomic<bool> has_failure{false};
std::atomic<bool> is_finished{false};
std::atomic<bool> status_cached{false};
std::atomic<uint64_t> finished_transfer_bytes{0};
std::atomic<uint64_t> finished_task_count{0};
bool counter_eligible = false;
std::atomic<uint64_t> active_completion_callbacks{0};
#ifdef USE_EVENT_DRIVEN_COMPLETION
std::mutex completion_mutex;
std::condition_variable completion_cv;
#endif
```

**`BatchID` 就是 `new BatchDesc()` 的指针值**（`transport.h:151-164` 的 `toBatchDesc`）：

```cpp
static inline BatchDesc &toBatchDesc(BatchID id) {
    return *reinterpret_cast<BatchDesc *>(id);
}
```

因此：

- 没有哈希查找、没有引用计数、无法校验野值；传错 `BatchID` 就是 UB。
- `freeBatchID` 本质是 `delete` 那个指针（`multi_transport.cpp:195`）。
- `batch_desc_set_`（`multi_transport.h:136`，类型 `unordered_map<BatchID, shared_ptr<BatchDesc>>`）的**声明是无条件的**，只有 `CONFIG_USE_BATCH_DESC_SET` 分支里的插入/删除代码会使用它；而该宏在仓库里**从未被任何 CMake/Makefile 定义**（grep 只命中 `#ifdef` 与一个测试的 `#ifndef`）。所以默认构建下这张表永远是空的，**任何“用 BatchID 查表”的讲解都是错的**。

`task_list` 按值存放带来一个硬约束：`submitTransfer` 里的 `task_list.reserve(...)`（`multi_transport.cpp:279`）不是可选优化。`emplace_back` 一旦触发扩容，已经交给 transport 的 `TransferTask*` 全部悬空（`submit_tasks` 里存的是裸指针）。reserve 保证整个提交过程中 vector 不搬家。

`BatchID` 的失败返回值也有坑：`allocateBatchID` 的 `if (!batch_desc) return ERR_MEMORY`（`:108-109`）是死代码（默认 `new` 抛 `bad_alloc`），即便命中，返回的 `uint64_t` 也不等于调用方检查的 `INVALID_BATCH_ID = UINT64_MAX`。

### 3.5 `Slice` 与全引擎最微妙的完成协议

`include/transport/transport.h:168-403`。Slice 是**硬件级最小传输单元**（RDMA 下受 `slice_size`/`fragment_limit`/MR 边界切分）。它有一个大 `union`（`:203-262`）让各 transport 塞私有字段（`rdma`/`ub`/`local`/`nccl`/`tcp`/`nvmeof`/`cxl`/`hccl`/`ascend_direct`/`ubshmem`/`flagcx`）。

Slice 必须在堆上分配并池化（注释 `:166-167`）。`slice->task` 由提交线程写、完成线程读，所以是 `SliceTaskPtr`（`std::atomic<TransferTask*>` 的薄包装，`:116-149`）——直接放 `std::atomic` 会破坏 "`slice = Transport::Slice()`" 这种按值重置写法。

#### 打包握手字

```cpp
static constexpr uint64_t kCompletionSealBit  = 1ull << 63;
static constexpr uint64_t kCompletionDoneMask = kCompletionSealBit - 1;
```

`completion_state` 把 **seal 标志（bit 63）** 与 **已完成 slice 计数（低 63 位）** 打包进一个字（`transport.h:264-269`）。动机是消除两个独立原子量上的 **Dekker 竞态**：

- 若 `sealed` 与 `done_count` 分开：完成线程递增 `done_count` 后读 `sealed=false` 决定不发布；同时提交线程写 `sealed=true` 后读 `done_count` 的旧值也决定不发布。两边都放弃，batch 永远 WAITING。
- 打包后，两个 RMW 在同一变量上形成全序，最后执行的那次必然看到对方的效果，`finishIfComplete` 一定会被某一方触发。

#### `markSuccess` / `markFailed` / seal / unseal

`transport.h:273-310`：

```cpp
void markSuccess() {
    CompletionCallbackPin pin(task);         // ① 先钉住 BatchDesc
    status = Slice::SUCCESS;                 // ② transport 内部状态
    TransferTask *t = task.load(acquire);
    __atomic_fetch_add(&t->transferred_bytes, length, RELAXED);
    __atomic_fetch_add(&t->success_slice_count, 1, ACQ_REL);
    const uint64_t state =
        __atomic_add_fetch(&t->completion_state, 1, ACQ_REL);
    finishIfComplete(t, state, false);       // ③ 用本次 RMW 的返回值判定
}
```

- **每个 slice 必须恰好标记一次**；pin 协议假设"单次终态发布"。
- `slice->status` 是 transport 内部状态，**不是终态通道**，所以它的写不参与判定。
- `markFailed` 对称，额外把 `batch_desc.has_failure` 置 true。
- `sealTaskSubmission` **也持 pin**，`__atomic_or_fetch` 置 bit63 后调 `finishIfComplete`。这覆盖"所有 slice 在 seal 之前就已完成"的情形。
- `unsealTaskSubmission` 是 release-store 0。它必须在**任何 slice 被提交之前**执行；否则 `completion_state` 默认带 seal，`markSuccess` 可能在 `slice_count` 还没写完时看到 `done == slice_count`（例如两者都是 0）而过早发布。

#### `finishIfComplete`（`transport.h:315-371`）

判定顺序：

1. `if (!task->batch_id) return;` —— 握手/dummy slice 不挂 BatchDesc。
2. `done = state & kCompletionDoneMask`；acquire 读 `slice_count`；若 `(state & seal) == 0 || done != slice_count` 直接返回。
3. `__atomic_exchange_n(&task->completion_published, true, ACQ_REL)` —— **只有一个线程能赢得交换**，保证终态发布恰好一次。
4. 分叉：
   - **非 EDC（默认）**：只 `batch_desc.finished_task_count.fetch_add(1, acq_rel)`。注释（`:337-340`）明确：`is_finished` 由前台 `getTransferStatus` 在完成遍历时写，CQ 线程**不能**写——因为 WAITING 快路径永不释放 batch，而 COMPLETED 总是先走一遍遍历。
   - **EDC**（`USE_EVENT_DRIVEN_COMPLETION`，默认 OFF）：先 relaxed 写 `task->is_finished`，再 `fetch_add` 计数；代码用 `prev + 1 == batch_desc.batch_size` 判断“最后一个 task”，在 `completion_mutex` 下 release-store `batch_desc.is_finished`，**释放锁之后**再 `notify_all`。注意这里比较的是**容量字段 `batch_size`**，不是 `task_list.size()`：scatter 路径会把它改写成实际 task 数，普通路径若只提交了容量的一部分，计数永远到不了 `batch_size`，等待方可能一直等到超时——使用 EDC 时最好按实际请求数分配 batch。EDC 的等待方在 Store：`mooncake-store/src/transfer_task.cpp:928-960` 直接 `wait_for` `BatchDesc::completion_cv`，并用 `is_finished` 做快路径。

#### `CompletionCallbackPin`（`transport.h:372-402`）

这是最容易被忽略的正确性机制。问题：终态切片计数对等待者可见的时刻**早于**完成回调栈展开结束。等待者此刻若调 `freeBatchID()`，`BatchDesc` 及其按值持有的 `TransferTask` 会被 delete，而 `finishIfComplete` 还在解引用它们 → UAF。

解法：回调进入时 `active_completion_callbacks.fetch_add(1, acq_rel)`，析构时减 1。`MultiTransport::tryFreeBatchID` 除了检查每个 task 终态，还要**显式等待 pin 计数归零**（`multi_transport.cpp:187-191`），否则返回 `Status::BatchBusy`。

### 3.6 `ThreadLocalSliceCache`

`transport.h:405-451`。容量 4096 的环形 free list，`head_`/`tail_` 是单调递增的 `uint64_t`，取模定位。`allocate()` 空则 `new Slice()` 并置 `from_cache=false`；`deallocate()`：

1. 先取出并清空 `cleanup_callback`，再调用它（注释：**必须在回调前清空**，否则缓存复用的 slice 会把上一次的 transport 私有 cleanup 带进下一次使用）；
2. 清 `interned_peer_nic_path`；
3. 池满才真正 `delete`。

实例是 `thread_local static`（`src/transport/transport.cpp:21`）。因此：

- 池本身不做同步，只被一个线程访问；
- slice 在哪个线程被 `deallocate`，就进入哪个线程的池。一个 batch 的 slice 通常由提交线程分配、由释放 batch 的线程归还，**跨线程"迁移"是允许的**；不要假设全局只有一个池。
- `from_cache` 被 RDMA 用来统计本次真正新申请的 slice 数，以 `kSubmitWatermark = max_wr * num_qp_per_ep` 决定是否中途 flush（`rdma_transport.cpp:1069-1071`、`:1161-1166`）。

### 3.7 `Topology`

`include/topology.h:38-146`。

- `TopologyEntry{name, preferred_hca[], avail_hca[]}`：**字符串** HCA 名；`name` 是 location（`cpu:0`、`cuda:1`、`*`）。
- `TopologyMatrix = unordered_map<string /*location*/, TopologyEntry>`。
- `Topology` 持有三份状态：
  - `matrix_`：字符串矩阵，可 JSON 化并随 SegmentDesc 发布；
  - `hca_list_`：去重后的 HCA 名数组，**下标即 `device_id`**；
  - `resolved_matrix_`：`ResolvedTopologyEntry`，名字换成下标 + 两张 name→index 反查表；
  - 另有 `resolved_hca_peer_affinity_by_local_`（本地卡→远端卡亲和）。
- `use_round_robin_` 由构造时的 `MC_PATH_ROUNDROBIN` 决定（`topology.cpp:676-683`）。

`resolve()`（`topology.cpp:886-926`）是字符串→下标的唯一入口，并且做了一件容易漏看的事：**为 `kWildcardLocation`（`"*"`）合成一条条目**，把所有第一次出现的 HCA 都塞进它的 `preferred_hca`。所以注册内存时不指定 location 并不会"没有拓扑"，而是拿到全量网卡。

`selectDevice(storage_type, retry_count)`（`:853-884`）**返回 `hca_list_` 的下标，不是设备名**；`ERR_DEVICE_NOT_FOUND = -6` 与下标共用返回值域。策略：

- `retry_count == 0`：`MC_PATH_ROUNDROBIN` 打开时用 thread_local 计数器 `% 10000`，否则 `SimpleRandom::Get().next()`；优先在 `preferred_hca` 里选，preferred 为空才落到 `avail`。
- `retry_count > 0`：确定性轮转 `(retry_count-1) % (preferred.size()+avail.size())`，前半段 preferred、后半段 avail。这是 failover 路径，每次重试换一张卡且顺序可复现。

另有 `selectDevice(storage_type, hint, retry_count)`（`:804-821`，先按名字精确命中，miss 才回落）和 `selectDeviceByLocalHca`（`:823-851`，`MC_ENABLE_HCA_PEER_AFFINITY` + `MC_NIC_PEER_AFFINITY`）。

`disableDevice`（`:928-977`）在 `matrix_` 与 `resolved_matrix_` 里同时摘掉某个下标，但**刻意不重跑 `resolve()`**：RDMA transport 按已解析的下标存 lkey/rkey 与 context 数组，压缩下标会让这些数组指向错误的 RNIC。

### 3.8 `GlobalConfig`

`include/config.h:42-201`，约 60 个字段（头文件注释说"全量"，不是 100 个字段；`MC_*` 环境变量约 100 个）。`globalConfig()`（`src/config.cpp:932-937`）是 `std::call_once` 单例，首次调用跑 `loadGlobalConfig()`（`:108-843`）。

与本章直接相关的默认值：

| 字段 | 默认 | 作用 |
|---|---|---|
| `slice_size` | 65536 | 单个 Slice 的目标大小 |
| `fragment_limit` | 16384 | RDMA 分片上限 |
| `retry_cnt` | 9 | 单 slice 最大重试次数 |
| `metacache` | true | 关闭（`MC_DISABLE_METACACHE`）后每次选路都走元数据后端 |
| `slice_timeout` | -1（秒） | **默认关闭**；直接决定 `getBatchTransferStatus` 走快路径还是慢路径 |
| `te_metadata_refresh_interval_seconds` | 0 | 默认不起元数据后台刷新线程 |
| `rpc_min_port`/`rpc_max_port` | 15000/17000 | 随机 RPC 端口范围 |
| `handshake_listen_backlog` | 128 | 握手 daemon backlog |
| `max_wr` / `num_qp_per_ep` | 256 / 2 | 决定 RDMA 提交 watermark |
| `use_rdma_twosided` | false | `autoDiscoverTransport()` 选 rdma 还是 rdma_twosided |
| `trace` | false | 打印路由等细节日志 |

`updateGlobalConfig(ibv_device_attr&)`（`:858-873`）在打开设备后把各上限 clamp 到硬件能力。`ValidatePortRange`（`:941-961`）拒绝 well-known（0-1023）与 ephemeral（32768-60999），非法输入回落到默认 `{15000, 17000}`。

注意：`MC_TE_FILTERS`、`MC_PATH_ROUNDROBIN`、`MC_FORCE_TCP`、`MC_FORCE_SHM`、`MC_FORCE_HCA` 等**不在 GlobalConfig 里**，它们是各自模块直接 `getenv` 的。语义也不统一（见 §设计取舍）。

### 3.9 `memory_location`：buffer 的 location 标签

`include/memory_location.h`：

- `kWildcardLocation = "*"`：让 transport 自己探测；
- `"cpu:N"` / `"cuda:N"`：直接指定 NUMA/GPU；
- `kSegmentsLocationPrefix = "segments:"`：编码 `"segments:<page_size>:<numa0>,<numa1>,..."`，把一个 buffer 等分成 N 段、第 i 段绑到 `numa[i]`。按偏移访问时由 `resolveSegmentsLocation` 惰性算出 `cpu:N`，**越界会 clamp 到最后一段**（`memory_location.cpp:203-212`）。

`detectBufferPageSize`（`:27-53`）解析 `/proc/self/smaps` 中目标地址所在 VMA 的 `KernelPageSize`，失败回落 `sysconf(_SC_PAGESIZE)`；这让 HugeTLB buffer 的 MR 切块能对齐到真实页大小。

`getMemoryLocation`（`:85-155`）：

1. GPU 构建下先 `cudaPointerGetAttributes`；`cudaMemoryTypeDevice` → `cuda:N` 直接返回；
2. 否则按页对齐，用 `numa_move_pages(0, n, pages, nullptr, status, 0)` 逐页查 NUMA 节点；
3. 把连续同节点的页合并成一个 `MemoryLocationEntry`；
4. 任何失败都回落为 `"*"`。

一个细节：`detectCudaDevicePresent()` 用函数内 `static` 缓存"本机有没有 CUDA 设备"（`:74-82`），避免无 GPU 的 RDMA-only sidecar 对每个 buffer 都打一条 ERROR。

`segments:` 编码的典型生产者是 Store 的 `real_client.cpp:1317`（`buildSegmentsLocation(page_sz, seg_numa_nodes)`），消费者是 RDMA transport 的 `pickTopologyDevice` 与 `rdma_batch_cache.h:54`。

---

## 关键流程剖析

### 流程 A：一次批量传输的完整生命周期

以 `example/transfer_engine_bench.cpp:421-501` 的 `initiatorWorker` 为骨架：

```text
allocateBatchID(batch_size)                       → BatchID（= BatchDesc*）
构造 vector<TransferRequest>                      → 调用方持有，生命周期覆盖整个批次
submitTransfer(batch_id, requests)                → Status
循环 getTransferStatus(batch_id, task_id, st)     直到 COMPLETED / FAILED
freeBatchID(batch_id)                             → OK 或 BatchCleanupDeferred
```

#### A.1 `allocateBatchID`

`TransferEngine::allocateBatchID`（classic `transfer_engine.cpp:267-269`）→ `TransferEngineImpl::allocateBatchID`（内联 `transfer_engine_impl.h:279-281`）→ `MultiTransport::allocateBatchID`（`multi_transport.cpp:107-120`）：`new BatchDesc()` + `id = (BatchID)batch_desc` + `batch_size` + `task_list.reserve(batch_size)` + `context = NULL`。

#### A.2 `submitTransfer`：选路 → 切分/合并 → unseal → 分派 → seal

`multi_transport.cpp:254-328`，逐段拆。

**① 容量检查（`:257-262`）**

```cpp
if (!task_sizes &&
    batch_desc.task_list.size() + entries.size() > batch_desc.batch_size)
    return Status::TooManyRequests("Exceed the limitation of batch capacity");
```

`!task_sizes` 是关键：**scatter 路径（传了 `task_sizes`）跳过容量检查**，因为它可能把 N 个请求合并成更少的 task；它在 `:311` 反过来把 `batch_size` 重设为实际 task 数。`submitScatter` 用 `allocateBatchID(0)` 起头（`:336`）。

**② 批量选路（`:264-266` → `selectTransports`，`:746-776`）**

对每个 request 调 `selectTransport`，但带批量选路缓存：`reused_transport` / `reused_target` / `reuse_allowed`。缓存正确性前提写在 `multi_transport.h:112-116`：`allows_reuse == true` 意味着"这个 segment 的 transport 只是 `target_id` 的函数"。

`selectTransport`（`:778-898`）的规则：

- `ENABLE_MULTI_PROTOCOL` 打开时，`*allows_reuse = (protocol 里没有逗号)`；混合协议 segment 必须逐地址解析。
- 未编译多协议时无条件 `true`。
- `target_id` 无效时 `false` 并返回 `InvalidArgument`。
- 缓存还要求 `globalConfig().metacache` 为真（`:763`）：关掉元数据缓存时必须保留每次显式刷新的行为。

`ENABLE_MULTI_PROTOCOL` 下，若 `target_segment_desc->protocol` 含逗号（如 `"rdma,hip"`），`selectTransport` 进入**按目标地址打分**的慢路径（`:807-874`）：

1. 遍历 `SegmentDesc::buffers`，找覆盖 `target_offset` 的那个 buffer。CXL 的起始地址是 `buffer.offset + cxl_base_addr`，其余协议用 `buffer.addr`（`:835-838`）。
2. 若 buffer 协议是 `hip`/`musa`/`shm` 且目标主机与本地不同（`isLocalIpcReachableTarget(desc->name, local_server_name_)`），**跳过**——这些是进程/主机内 IPC，跨主机不可达（`:841-845`）。
3. 按固定优先级选最高分：`hylink` 5（`MC_DISABLE_HYLINK` 存在即降为 0）> `hip`/`maca`/`musa`/`shm` 4（`hip`/`maca`/`musa` 可分别被 `MC_DISABLE_HIP`/`MC_DISABLE_MACA`/`MC_DISABLE_MUSA` 降为 0，`shm` 没有关闭开关）> `cxl` 3 > `rdma` 2 > `tcp` 1（`:808-821`）。**注意 `maca` 不在第 2 步的 IPC 门控里。**
4. 选出的协议若未安装，`rdma` 可降级到 `rdma_twosided`（`:859-862`），否则返回 `NotSupportedTransport`。

一个 buffer 同时以 `rdma` 和 `hip` 注册（设备 KV pool 的常见形态）时，这个打分保证了同主机走 IPC、跨主机自动落到 rdma，无需运维设 `MC_DISABLE_HIP`。

**③ `counter_eligible` 计算（`:270-274`）**

```cpp
batch_desc.counter_eligible =
    (task_list.empty() || batch_desc.counter_eligible) &&
    std::all_of(transports.begin(), transports.end(),
                [](Transport* t) { return t->supportsBatchCompletionCounter(); });
```

只有**全部** task 都由"无需前台轮询即可发布完成"的 transport 承担时才为真；全仓库只有 `RdmaTransport` 覆盖返回 `true`（`include/transport/rdma_transport/rdma_transport.h:67`）。一旦某 transport 提交失败，`:321` 清回 false；后续追加 legacy transport 也永久为 false（`task_list.empty()` 只为第一次提交服务）。

**④ 状态复位（`:275-277`）**：`status_cached=false`、`is_finished=false`、`finished_transfer_bytes=0`（relaxed）。**`finished_task_count` 不复位**——它跨同一 batch 的多次提交累加，而快路径比较的是累加的 `task_list.size()`，自洽。另注意 `entries.empty()` 在 `:268` 提前返回，所以空提交不会执行这次复位。

**⑤ task 切分与合并（`:283-310`）**

```cpp
for (size_t i = 0; i < entries.size();) {
    size_t count = 1;
    const auto group_id = entries[i].task_group_id;
    if (task_sizes && group_id != kNoTaskGroup &&
        transports[i]->supportsGroupedScatter()) {
        while (i + count < entries.size() &&
               entries[i + count].task_group_id == group_id &&
               transports[i + count] == transports[i])
            ++count;
    }
    auto& task = task_list.emplace_back();
    task.batch_id = batch_id;
    task.transport_ = transports[i];
    task.request = &entries[i];
    task.request_count = count;
    if (transports[i]->supportsBatchCompletionCounter())
        Transport::Slice::unsealTaskSubmission(&task);
#ifdef USE_EVENT_DRIVEN_COMPLETION
    if (count > 1) Transport::Slice::unsealTaskSubmission(&task);
#endif
    submit_tasks[transports[i]].push_back(&task);
    if (task_sizes) task_sizes->push_back(count);
    i += count;
}
```

**合并只在四个条件同时满足时发生**：传了 `task_sizes`（即 scatter 路径）、`task_group_id != kNoTaskGroup`、该 transport `supportsGroupedScatter()`（只有 `RdmaTransport`（`rdma_transport.h:105`）与 `TcpTransport`（`tcp_transport.h:172`）返回 true）、相邻且 `task_group_id` 相同并路由到同一 transport。合并后 `request_count = N`，于是 **`getTransferStatus` 的 `task_id` 不再等于请求下标**——必须用 `task_sizes` 前缀和换算，`ScatterTransferOperation::Impl::poll`（`transfer_engine.cpp:1773-1845`）正是这么做的。

`unsealTaskSubmission` 在 task 构造完、交给 transport 之前清零 `completion_state`。若不清零，默认的 seal bit 会让空 task（`slice_count == 0`）在构造瞬间就被判为"完成"。

`#ifdef USE_EVENT_DRIVEN_COMPLETION` 下 `count > 1` 也要 unseal。代码没有注释说明原因，合理推断是：合并 task 的多个请求可能由不同 CQ 线程完成，必须等提交线程把全部 slice 发布完再 seal（此处为推断，未完全确认）。

**⑥ 分派与 seal（`:312-327`）**

按 transport 分组，逐个调 `entry.first->submitTransferTask(entry.second)`。**transport 契约（`transport.h:550-552`）要求实现在返回前同步发布所有 slice；返回之后任何 worker 都不得再追加或发布 slice。** 然后对每个仍未 seal 的 task 调 `sealTaskSubmission`（`:315-319`，relaxed 读 seal bit 判断）。

失败处理：`overall_status = status`、`counter_eligible = false`，**但其他 transport 已提交的任务不回滚**——batch 处于"部分提交"状态。调用方拿到的 Status 只反映最后一个失败，不能据此认为整个 batch 干净。被失败的 transport 已经发布的 slice 会自行走向终态（`markFailed`），未发布的空 task 在 seal 后按"0 slice 完成"处理。

#### A.3 transport 侧：RDMA 如何把 task 变成 slice

`RdmaTransport::submitTransferTask`（`rdma_transport.cpp:942-1175`）值得看一眼，它解释了上层协议的若干约束：

- 取本地 SegmentDesc（`LOCAL_SEGMENT_ID`）与按 `request.target_id` 缓存的远端 SegmentDesc（`:1014-1021`）。
- **按 request 选一次本地 RNIC**（`:1028-1055`，`BatchRdmaDeviceCache`），让一个请求的所有 slice 通常走同一张卡；`nic_hint` 只在 `advise_retry_cnt == 0` 时生效。
- `SliceLengthCalculator` 按 `slice_size`/`fragment_limit`/本地与远端 MR 边界算每片长度（`:1057-1065`）。
- 选不到设备时 `fail_task_and_cleanup`：把**当前这片**（可能是非零长度）`markFailed()`，再调用 `fail_unstarted_tasks` 为**后续尚未开始的 task** 各造一个零长度失败 slice（`:973-986`），让“success + failed == slice_count”这个统一判据把 task 推到 FAILED，无需特例分支。
- 每片 `task.slice_list.push_back(slice)` + `__sync_fetch_and_add(&task.slice_count, 1)`（`:1084`、`:1158`）——**`slice_count` 是边构造边递增的**，这正是必须先 unseal 的原因。
- 攒到 `kSubmitWatermark` 中途 flush 一次 `submitPostSend`（`:1161-1166`），最后统一 flush（`:1172-1173`）。

完成侧：`WorkerPool` 为每个 `RdmaContext` 起若干 `transferWorker` 线程 + 1 个 `monitorWorker`。CQ 轮询到 WC 后进 `processCompletions`，终态统一走 `finalize_slice` 调 `slice->markSuccess()`/`markFailed()`。**`markSuccess` 就跑在 CQ worker 线程上**，`CompletionCallbackPin` 保护的正是"CQ 线程还在栈上、前台线程已经 delete BatchDesc"这个窗口。

`slice->ts` 在 post 成功时设为当前纳秒，重试时清 0；`getTransferStatus` 的超时检查读的就是它。

#### A.4 `getTransferStatus`：单 task 查询（含隐藏副作用）

`TransferEngine::getTransferStatus`（classic `transfer_engine.cpp:296-299`）→ `TransferEngineImpl::getTransferStatus`（**内联在 `transfer_engine_impl.h:302-363`**）：

1. 调 `multi_transports_->getTransferStatus(batch_id, task_id, status)`；
2. `WITH_METRICS` 段：终态（COMPLETED/FAILED/CANCELED/TIMEOUT）都会把 `task.start_time` 归零以防重复计数；其中只有 COMPLETED 会累加字节计数器与延迟直方图；
3. `#ifdef USE_ASCEND_DIRECT` 在此**提前 return**（`:348-350`）——Ascend Direct 构建下没有下面的 notify 副作用；
4. **隐藏副作用（`:351-361`）**：若 `status.s == COMPLETED`，顺手调 `getBatchTransferStatus(batch_id, dummy, /*skip_metrics=*/true)`。

第 4 步是理解 `submitTransferWithNotify` 的钥匙：`TransferEngineImpl::submitTransferWithNotify`（`:180-209`）在提交成功后**只是把 `NotifyDesc` 存进 `notifies_to_send_[batch_id]`**（写锁保护），并不发送。真正的 `sendNotifyByID` 发生在 `getBatchTransferStatus`（`:384-398`）：聚合状态 COMPLETED 时，取出并 `erase` 该 batch 的通知，然后在**锁外**发送（网络操作不能持锁）。

结论：**notify 是在调用方轮询状态时才发出去的；不轮询就不会发。** 逐 task 轮询同样能触发（通过第 4 步）。这也是为什么 `notifies_to_send_` 的清理必须挂在 `freeBatchID` 的 `before_delete` 上（`impl.h:283-290`）：BatchID 是地址派生的，必须在描述符被删、地址可能复用之前擦掉侧表。

`MultiTransport::getTransferStatus`（`multi_transport.cpp:393-464`）本体：

- `task_id >= task_count` → `InvalidArgument`；
- `checkSliceTimeout` lambda（`:404-418`）：`slice_timeout <= 0` 直接 false；否则遍历 `slice_list` 看 `ts > 0 && now - ts > slice_timeout * 1e9`；
- **`task.transport_` 非空时委托给该 transport 的 `getTransferStatus`**（`:426-437`）。注释解释这一步不可省：NVLink async transport 就是在这里 `cudaStreamQuery()` 的，没有这次调用 slice 状态永远不会更新。委托返回 WAITING 时再叠加超时检查改判 TIMEOUT；
- `transport_` 为空的遗留路径（`:439-463`）自己算 `success + failed == slice_count`，命中则写 `is_finished`（relaxed）。

`RdmaTransport::getTransferStatus`（`rdma_transport.cpp:1205-1231`）是纯计数器读：acquire 读两个计数、relaxed 读 `transferred_bytes`（注释：与 `markSuccess` 配对，先观察完成再读字节，避免终态携带过期字节数），相等则判 COMPLETED/FAILED 并写 `is_finished`，否则 WAITING。**它不驱动任何硬件进度**——RDMA 进度由 CQ worker 自驱。

#### A.5 `getBatchTransferStatus`：快慢两条路

`multi_transport.cpp:503-565`：

```cpp
status.transferred_bytes = 0;
if (status_cached.load(acquire) || task_count == 0) {        // ① 已缓存/空批次
    status.s = COMPLETED;
    status.transferred_bytes = finished_transfer_bytes.load(relaxed);
    return OK;
}
const uint64_t finished = finished_task_count.load(acquire);
if (counter_eligible && slice_timeout <= 0 &&
    !has_failure.load(acquire) && finished < task_count) {   // ② 计数器快路径
    status.s = WAITING;                                      //    bytes 留 0
    return OK;
}
// ③ 全量遍历每个 task 的 getTransferStatus
```

- **①** `task_count == 0` 也返回 COMPLETED——空批次视为已完成。`status_cached` 只在③判定 COMPLETED 时置位（`:559`），终态之后所有查询 O(1)。
- **②** 条件是"能力"而非"计数器非零"（`:517-519` 注释：混合/遗留 transport 可能需要轮询才能推进）。**WAITING 期间 `transferred_bytes` 报 0 是设计而非 bug**：0 是合法下界，直到 COMPLETED 才由③填真实值；`docs/source/api-reference/cpp/transfer-engine.md:157-163` 明确写了这一点。快路径**不写 `is_finished`**（`:530-531`），与 `finishIfComplete` 非 EDC 分支的约定成对。
- **③** 逐 task 调 `getTransferStatus`：COMPLETED 累加字节并计 `success_count`；FAILED 或 TIMEOUT **立即短路返回 FAILED**（`:546-550`）；查询本身出错也短路返回 FAILED。全部 COMPLETED 才置 `status_cached`/`is_finished`/`finished_transfer_bytes`。

一个代码层面的观察：`:561-563` 的 `else if (status.s == FAILED) { has_failure.store(true); }` 在循环内失败分支已经 `return` 的前提下**不可达**。`has_failure` 的实际写者是 `Slice::finishIfComplete`（`transport.h:322-324`）在 `markFailed` 回调里。这不影响语义，但读代码时不要以为失败态是在这里落盘的。

`TransferEngineImpl::getBatchTransferStatus`（`impl.h:372-400`）在其上叠加 metrics 与 notify 发送。

#### A.6 `freeBatchID`：BatchBusy vs BatchCleanupDeferred

`TransferEngineImpl::freeBatchID`（`impl.h:283-290`）传了一个 `before_delete` lambda：在 `BatchDesc` 被 delete 之前从 `notifies_to_send_` 擦掉该 `batch_id`。

`MultiTransport::freeBatchID`（`:126-152`）：

1. 持 `deferred_cleanup_mutex_`；
2. 若 `batch_id` 已在 `deferred_cleanup_batches_` 里 → 直接返回 `BatchCleanupDeferred`（幂等，防重复入队）；
3. 调 `tryFreeBatchID`；非 `BatchBusy` 的结果原样返回；
4. `BatchBusy` → 把 `{batch_id, before_delete}` 存进 `deferred_cleanup_batches_`，**惰性启动**清理线程（`:140-146`），`notify_one`，打 WARNING，返回 `BatchCleanupDeferred`。

`tryFreeBatchID`（`:154-201`）的三重检查：

- 遍历每个 task：`is_finished` 为真跳过；`slice_count == 0` 直接标 finished 跳过（未提交的空 task）；否则调 `getTransferStatus`，非 COMPLETED/FAILED 就 `BatchBusy`。
- **`active_completion_callbacks != 0` → `BatchBusy`**（`:187-191`）。
- 都通过才 `before_delete()` + `delete &batch_desc`。

`deferredCleanupLoop`（`:203-247`）：100ms 一轮，重试每个 deferred batch；成功则移除；**非 BatchBusy 的永久错误也移除并 `LOG(ERROR)`**（`:223-228`）；BatchBusy 留到下一轮。停机时若还有 busy batch，**只打 WARNING 然后返回，故意泄漏**（`:234-243`），以免阻塞 shutdown 等待 transport 回调。

**语义要点**：`BatchCleanupDeferred` 与 OK 一样使 `batch_id` 失效（`multi_transport.h:58-60` 与 `transfer_engine.h:271-273`）。C API 把它映射成 `BATCH_CLEANUP_DEFERRED = 5`（`transfer_engine_c.h:30`），与 `Status::Code::kBatchCleanupDeferred = 5`（`status.h:39`）对齐，也与 `ERR_BATCH_CLEANUP_DEFERRED = -5`（`error.h:22`）只差符号——**三个常量分散在两个头文件里，改动必须同步**。

注意：deferred 返回后句柄已失效，**不要再次 freeBatchID**。第二次调用会走 `toBatchDesc` 解引用一个可能已被 delete 的指针。

#### A.7 scatter 变体与 `ScatterTransferOperation`

scatter 是"一次把多个不连续片段读/写到多个远端区间"的便捷层，实现分两半：

- **classic**：`TransferEngine::submitScatter`（`transfer_engine.cpp:1893-1904`）构造 `Impl`；`Impl::build`（`:1581-1771`）把每个 `ScatterTransferRange` 展开成 fragment，统一 `task_group_id = 1`，调 `backend_.legacy->submitScatter(requests_, submission)`（`multi_transport.cpp:330-341`）→ `allocateBatchID(0)` + `submitTransfer(..., &task_sizes)`。
- **TENT**：`Impl::build` 直接用 `backend_.tent->allocateBatch` + `submitTransferRequiringPostSubmitCancellation`，并在 `requestAbort` 里真正调 `cancelTransfer`（`:1490-1505`）。classic 的 `requestAbort` 只记录状态，没有取消能力——`#ifdef USE_TENT if (!backend_.tent) return;`。

轮询 `poll`（`:1773-1845`）按 `task_sizes_` 前缀和把 task 状态映射回 fragment；若一个 grouped task 失败（`task_sizes>1`），用 `getScatterRequestStatuses`（`multi_transport.cpp:466-501`）按 slice 顺序恢复每个 fragment 的成败。

几个重要语义：

- `ScatterTransferOperation` 是**单消费者**：`wait()`/`waitFor()` 不能并发调用，也不能从 fragment 回调里调（`transfer_engine.h:217-223`）。
- `waitFor` 超时**不取消传输**；operation 与 buffer 必须保持存活直到后续 wait 完成。
- 析构函数 `~Impl() { wait(); }`（`:1354`）——**析构会阻塞到物理完成**。
- `legacyBatchPhysicallyDrained`（`:1507-1537`）在提交失败后判断 batch 是否真的排空：非 EDC 下，slice 计数齐全并不等于完成回调已收尾，但它需要 `freeBatchID` 成功，所以会主动把 `is_finished` 置 true；EDC 下直接 `return false`，让完成线程自己发布。
- `getScatterRequestStatuses` 对 slice 布局有强假设：按 `task.request[i].length` 依次消费 `slice_list`，要求每个请求的 slice 连续、长度之和恰等于请求长度、且整表被完全消费（`:497-499`）。这正是 `transport.h:564-566` 那条"grouped transports 必须按请求顺序 append slice"契约的兑现点。

### 流程 B：引擎初始化与 transport 安装决策树

`TransferEngineImpl::init`（`transfer_engine_impl.cpp:108-543`）。调用方是 `TransferEngine::init`（classic `transfer_engine.cpp:128-145`；带 `protocol` 参数的重载先 `applyProtocolHint(protocol)`，见 `impl.h:452-458`）。

**① 抬高 fd 上限**（`:115-119` → `setFilesLimit`，`:77-94`）：把 `RLIMIT_NOFILE` 的 soft 提到 hard，失败只 WARNING。

**② 解析 `local_server_name`**（`:131-149`）：Ascend 构建走 `parseHostNameWithPortAscend`（名字格式 `ip:port:npu_x`，`local_server_name_` 只留 `ip:port`）；其余走 `parseHostNameWithPort`，原样保留。

**③ RPC 端口绑定方式三选一**（`:151-221`）：

| 方式 | 触发条件 | 行为 |
|---|---|---|
| `legacy/P2P` | `MC_LEGACY_RPC_PORT_BINDING` 已设 | 用 `local_server_name` 解析出的 host/port，`sockfd = -1`，由 daemon 自己 bind |
| `P2P handshake` | `metadata_conn_string == P2PHANDSHAKE` | `findAvailableTcpPort(desc.sockfd)` 随机挑端口并**保留已 bind 的 fd**（避免 TOCTOU），随后把 `local_server_name_` 改写为实际 `ip:port`（IPv6 用 `maybeWrapIpV6`） |
| `new RPC mapping`（默认） | 其余 | 忽略传入的 `rpc_port`/`ip_or_host_name`；IP 取 `MC_TCP_BIND_ADDRESS` 或 `findLocalIpAddresses()[0]`；端口 `findAvailableTcpPort(desc.sockfd)` 随机 |

`findAvailableTcpPort`（`transfer_metadata_plugin.cpp:1480-1554`）在 `[rpc_min_port, rpc_max_port]` 里最多试 500 次随机端口，每次真的 `socket()` + `bind()`，成功就把 fd 交回；`startDaemon` 会复用这个已绑定 fd 直接 `listen`（`:828-829`）。`findLocalIpAddresses`（`:1434-1479`）用 `getifaddrs`，跳过 `lo` 与非 `IFF_UP|IFF_RUNNING` 接口。

**为什么 P2P 要改写名字、默认模式不改**：默认模式下 segment name 只是元数据 key，真实 RPC 地址由 `rpc_meta/<name>` 条目发布，所以名字里的端口无关紧要；P2P 模式没有元数据服务，对端直接按 segment name 解析 `ip:port` 来握手，因此必须写实际端口。

**④ 建 metadata 与 multi_transports**（`:223-234`）：`make_shared<TransferMetadata>(conn_string)`，再 `make_shared<MultiTransport>(metadata_, local_server_name_)`（Ascend 传带 `:npu_x` 后缀的副本）。`addRpcMetaEntry` 发布本机 RPC 位置；P2P 模式下这一步同时注册 metadata/notify/probe 三个回调并 `startDaemon`（`transfer_metadata.cpp:1883-1902`）。

**⑤ transport 安装决策树**（按短路顺序）：

| 顺序 | 条件 | 动作 |
|---|---|---|
| 1 | `MC_FORCE_TCP` **存在**（不检查值） | 装 `tcp` 后 `return 0`；`USE_TCP` 未编译则 `return -1`（`:240-255`） |
| 2 | `MC_FORCE_SHM` 且 `!ENABLE_MULTI_PROTOCOL` | 装 `shm` 后 `return 0`（`:262-274`） |
| 3 | Ascend 构建且 `shouldInstallAscendTransport(hint)` | 装 `ascend` + 可选 shm 后 `return`（`:276-292`） |
| 4 | `USE_UBSHMEM` | 装 `ubshmem`，**关掉 auto_discover**（`:294-302`） |
| 5 | `USE_CXL && MC_CXL_DEV_PATH` | 装 `cxl`（`:304-314`） |
| 6 | `auto_discover_config_.enabled` | 见下（`:316-539`） |
| 7 | 收尾 | `maybeInstallShmTransport`（多协议构建且 `MC_FORCE_SHM` 时追加，`transfer_engine_impl.cpp:57-74`） |

第 6 步内部先做拓扑发现：默认 `local_topology_->discover(filter_, /*exclude_efa=*/!use_efa)`；若设了 `MC_CUSTOM_TOPO_JSON`，读文件后 `parse`；**只有文件为空/读取失败才回落到 discover，非空但 JSON 非法时 `parse` 的返回码被忽略、不会回落**（`:321-334`）。注意 `discover()` 的返回码在 `init` 里**也没有被检查**：Data Direct 配对失败会返回 `ERR_DEVICE_NOT_FOUND` 并得到空 HCA 列表，随后通常静默落到 TCP。

然后按构建宏互斥选择：

- `USE_EFA` 且 `use_efa`：装 `efa` + 可选 shm 后 **return**（`:343-353`）。注释解释必须放在 per-build 选择之前：NVLink/MNNVL 分支从不装 EFA，否则默认 CUDA wheel（同时开 `USE_EFA` 与 `USE_MNNVL`）会退化成 TCP 或走非 EFA 网卡的 RDMA。
- `USE_UB`：装 `ub`（`:356-363`）。
- `USE_ASCEND_HETEROGENEOUS`：装 `ascend`（`:365-371`）。
- `USE_MACA`：`MC_MACA_HOST_TRANSPORT` 决定装 rdma/tcp 主机 transport，否则装 maca（`:372-407`）。
- `USE_MNNVL || USE_INTRA_NVLINK || USE_MUSA`（`:409-484`）：`MC_INTRANODE_NVLINK` → `nvlink_intra`；否则 GPU P2P 已编译且（`MC_FORCE_MNNVL`/`MC_FORCE_MUSA` 或（**无 HCA** 且未设 `MC_FORCE_HCA`））→ `nvlink`/`musa`；否则（有 HCA 或设了 `MC_FORCE_HCA`）→ `rdma`；否则 → `tcp`。`kGpuP2PCompiled` 是编译期常量（`:426-430`），防止只开 `USE_INTRA_NVLINK` 的构建去请求 MultiTransport 造不出来的协议。
- 默认（`#elif !defined(USE_SUNRISE)`，`:486-522`）：**有 HCA 且未设 `MC_FORCE_TCP`，或设了 `MC_FORCE_HCA`** → `autoDiscoverTransport()`（`impl.h:517-528`：barex / efa / rdma_twosided / rdma）；否则装 `tcp`。Sunrise classic 刻意跳过这条，由 tebench 在专用 setup 后显式安装。
- `USE_HIP`（`:525-538`）：**追加**装 `hip`，可与跨节点 transport 共存，失败只 WARNING。

这段树解释了运维最常见的问题——"**我这台机器明明装了 RDMA 为什么走 TCP**"：多半是 `discover()` 得到空 `hca_list_`（被 `MC_TE_FILTERS` 白名单挡掉、端口非 ACTIVE、`/dev/infiniband/*` 无读写权限、或被 `exclude_efa` 过滤），或者 `MC_FORCE_TCP` 存在。诊断入口是 `showLinks()`（流程 G）。

**⑥ `installTransport` 的协议工厂**（`multi_transport.cpp:567-744`）：一长串 `if/else if` 把 `"rdma"`/`"rdma_twosided"`/`"ub"`/`"barex"`/`"tcp"`/`"nvmeof"`/`"nccl"`/`"ascend"`/`"nvlink_intra"`/`"hip"`/`"hylink"`/`"maca"`/`"musa"`/`"nvlink"`/`"cxl"`/`"shm"`/`"ubshmem"`/`"flagcx"`/`"efa"`/`"cxi"`/`"sunrise_link"` 映射到具体类，每个都被对应 `#ifdef` 包住；未编译进来的协议会走到 `:697-701` 的 "Unsupported transport, please rebuild Mooncake"。两条互斥规则：`nccl` 必须是唯一 transport（`:569-576`）；`rdma` 与 `rdma_twosided` 不能共存（`:578-584`）。构造成功后 `transport->install(local_server_name_, metadata_, topo)`，失败则 `delete` 并返回 nullptr；成功存进 `transport_map_`（`std::map<string, shared_ptr<Transport>>`，MultiTransport 独占所有权）。

`TransferEngineImpl::installTransport`（`transfer_engine_impl.cpp:554-592`）在此之上多做两件事：已装则直接返回并 WARNING；**把所有已注册的本地内存区域补注册到新 transport 上**（`:586-590`）。它还会把 `args[0]` 当作 NIC 优先级矩阵 JSON 传给 `local_topology_->parse()`（`:569-576`）——这是 tebench 在关闭 auto-discover 时手写拓扑的入口（`transfer_engine_bench.cpp:559-564`）。

头文件注释 "Only for testing"（`impl.h:122`）**已经过期**：`mooncake-store/src/client_service.cpp:918-1007` 会按协议装 tcp/cxl/cxi/ub/nvlink_intra，`mooncake-integration/transfer_engine/transfer_engine_py.cpp:445-497` 会装 efa/flagcx/cxi/tcp。

### 流程 C：`registerLocalMemory` 的两阶段预留

`TransferEngineImpl::registerLocalMemory`（`transfer_engine_impl.cpp:780-825`）：

```cpp
if (length == 0) return ERR_INVALID_ARGUMENT;
std::vector<MemoryRegion> regions = {{addr, length, location, remote_accessible}};
if (!tryReserveMemoryRegions(regions)) return ERR_ADDRESS_OVERLAPPED;   // ① 预留
std::vector<Transport*> attempted_transports;
for (auto transport : multi_transports_->listTransports()) {            // ② 逐个注册
    attempted_transports.push_back(transport);
    int ret = transport->registerLocalMemory(addr, length, location,
                                             remote_accessible, update_metadata);
    if (ret < 0) {
        for (auto it = attempted_transports.rbegin(); it != rend(); ++it)  // ③ 逆序回滚
            (*it)->unregisterLocalMemory(addr, update_metadata);
        releaseMemoryRegions(regions);                                  // ④ 释放预留
        return ret;
    }
}
commitMemoryRegions(regions);                                           // ⑤ 提交
```

**为什么要两张表**（`local_memory_regions_` 与 `registering_memory_regions_`，均为 `std::map<uintptr_t, MemoryRegion>`，`impl.h:487-509`）：MR 注册是长耗时操作（大 buffer 可能数秒）。如果只在注册完成后才把区间写进 `local_memory_regions_`，这段窗口里另一个线程注册重叠区间不会被拦住。`tryReserveMemoryRegions`（`:1077-1095`）在**排他锁**下先查重叠再写 `registering_memory_regions_`，于是**重叠检查在注册开始前就生效**。

重叠判定 `hasOverlapLocked`（`:1046-1050`）同时查两张表；`hasOverlapInMapLocked`（`:1052-1075`）利用 `std::map` 有序性只看 `lower_bound(addr)` 与它的前驱两个候选（配合 `common.h` 的 `overlap()`），O(log n)。

`commitMemoryRegions`（`:1097-1105`）把区间从 registering 表搬到 local 表；`releaseMemoryRegions`（`:1107-1114`）只从 registering 表删。三个函数各自取排他锁——**注册期间锁是释放的**，并发注册不重叠区间可以真正并行。

`registerLocalMemoryBatch`（`:963-1028`）同构，差别：先按地址排序、用"相邻两项 `address - previous_address < previous.length`"做 O(n log n) 的批内重叠检查（`:965-990`），再一次性预留全部区间，失败时逆序 `unregisterLocalMemoryBatch(addr_list)` 回滚。

`unregisterLocalMemory`（`:827-841`）是 **best-effort**：遍历所有 transport 都试一遍，只记第一个错误；全部成功才从 `local_memory_regions_` 擦掉。

**transport 侧发布元数据**：以 RDMA 为例，`registerLocalMemoryInternal`（`rdma_transport.cpp:339-664`）把大 buffer 按 `max_mr_size` 切块（HugeTLB 情况下还要向下对齐到页大小），每块逐个 context 注册 MR、收集 `lkey`/`rkey`，构造 `BufferDesc` 并调 `metadata_->addLocalMemoryBuffer(buffer_desc, /*update_metadata=*/false)`（`:632-633`）。**`update_metadata=false` 是关键**：分块 buffer 否则会对元数据服务器发布 N 次；所有块加完后在 `:642-651` 统一 `updateLocalSegmentDesc()` 一次。

`addLocalMemoryBuffer`（`transfer_metadata.cpp:1822-1834`）用 copy-on-write：写锁下拷贝 `SegmentDesc`、`push_back` 新 buffer、`rebuildBufferRangeIndex()`、换回表里。**在飞的传输持有旧 `shared_ptr`，看到的是一致快照**。

`updateLocalSegmentDesc`（`:1762-1800`）先读一次已发布版本，在写锁下拷贝 desc 并 `nextWallTimeMetadataVersion(local, published)` 打新版本号（`:71-78`：取当前墙上微秒，若已发布或本地版本 ≥ 它就 +1，保证严格单调且非 0），换回表，最后**在锁外** `publishSegmentDesc`（`:789-808`）。P2P 模式下 `publishSegmentDesc`/`updateSegmentDesc`/`updateLocalSegmentDesc` 全是 no-op（`:770-772`、`:791-793`、`:1775-1777`）。

`location == "*"` 的解析发生在 transport 内部：`rdma_transport.cpp:611-614` 在**第 0 块注册完成后**（页已 fault in）才调 `getMemoryLocation(addr, length, /*only_first_page=*/true)`；注释说明必须等内存被 pin 住再解析，否则 NUMA 发现会保留 `"*"`，选中不相关 NUMA 上的网卡。

### 流程 D：元数据平面

#### D.1 连接串、模式与 key 规则

`TransferMetadata` 构造函数（`transfer_metadata.cpp:304-341`）：

1. `next_segment_id_ = 1`；
2. `common_key_prefix_ = "mooncake/" + [MC_METADATA_CLUSTER_ID + "/"]`；`rpc_meta_prefix_ = common_key_prefix_ + "rpc_meta/"`（`:310-321`）；
3. **总是**创建 `HandShakePlugin`（`handshake_plugin_ = HandShakePlugin::Create(conn_string)`），它恒返回 `SocketHandShakePlugin`；
4. `conn_string == P2PHANDSHAKE` → `p2p_handshake_mode_ = true`，**`storage_plugin_` 保持 nullptr**，直接返回；
5. 否则 `MetadataStoragePlugin::Create(conn_string)`。

`getFullMetadataKey`（`:417-432`）：普通名字 → `mooncake/[cluster/]ram/<name>`；名字里含 `/` → 直接 `mooncake/[cluster/] + name`（不再插 `ram/`）。RPC 元数据用 `mooncake/[cluster/]rpc_meta/<server_name>`。

`P2PHANDSHAKE` 是完全独立的第三种模式，不是存储插件：此时远端 SegmentDesc 全靠 `exchangeMetadata` 向对端 RPC 换取；`publishSegmentDesc`/`updateSegmentDesc`/`updateLocalSegmentDesc`/`removeRpcMetaEntry`/`rePublishRpcMetaEntry` 全部 no-op；`removeSegmentDesc` 只清本地缓存；`getRpcMetaEntryInternal` 直接从 server_name 解析 ip/port（`:1989-1993`）。

#### D.2 `openSegment` 只是 `getSegmentID`

`TransferEngineImpl::openSegment`（`transfer_engine_impl.cpp:722-745`）：剥掉前导 `/`、调 `metadata_->getSegmentID(name)`、返回结果。**它不建立任何连接**；`closeSegment`（`:762-764`）在 classic TE 里就是 `return 0`。文档示例里 `engine.closeSegment(segment)` 那行给人"句柄需要配对释放"的错觉。

但 `openSegment` **可能触发网络 I/O**：`getSegmentID`（`transfer_metadata.cpp:1740-1760`）先读锁查 `segment_name_to_id_map_`；miss 则**在锁外**调 `getSegmentDesc(name)`（etcd/HTTP/Redis 往返，或 P2P 直接向对端换 JSON），成功后写锁 double-check 再 `next_segment_id_.fetch_add(1)` 分配新 id。失败返回 `-1`（`SegmentID` 是 `uint64_t`，实际是 `UINT64_MAX`），不是具名常量。

`CheckSegmentStatus`（`:747-760`）在 classic TE 里除 Barex 外**恒返回 OK**——已失效的远端 segment 只能靠下一次 `getSegmentDescByID` 返回 nullptr 才暴露。TENT 分支则主动 `probePeerAliveByID`（`transfer_engine.cpp:784-800`）。

#### D.3 两条获取路径

`getSegmentDescInternal`（`:1335-1421`）：

- **P2P 模式**（`:1341-1365`）：从 segment_name 解析 `ip:port`，读锁取本地 SegmentDesc，`encodeSegmentDesc` 成 JSON，调 `handshake_plugin_->exchangeMetadata(ip, port, local_json, peer_json)` 与对端直接互换。任何失败都置 `kUnavailable`（P2P 没有"权威缺失"概念）。对端由 `receivePeerMetadata`（`:1311-1328`）应答，encode 自己的本地 desc。
- **存储模式**（`:1366-1375`）：`storage_plugin_->getWithStatus(getFullMetadataKey(name), peer_json)`，非 `kFound` 就把 `GetResult` 透传并返回 nullptr。

拿到 JSON 后 `decodeSegmentDesc`（`:997-1309`）按 protocol 分支解析，最后 `rebuildBufferRangeIndex()` + `rebuildInternedNicPaths()`（`:1306-1307`）。**对畸形数据 fail-closed**：rdma/efa/cxi 下 `device.name` 或 `gid` 为空直接 nullptr；`rkey.size() != lkey.size()` 或（按 device_id 索引 key 的协议）`rkey.size() != devices.size()` 也 nullptr。注释解释了后果：`selectDevice` 会拿一个 `device_id` 同时索引 `buffers[].rkey` 与 `devices[]`，key 向量比 device 列表长会把下标推出边界。

tcp 兼容性：老描述符可能缺 `tcp_data_host`，`getSegmentDescInternal` 会从遗留 RPC 元数据补一次并冻结进快照（`:1389-1397`）。

#### D.4 缓存与版本仲裁

`getSegmentDescByName`（`:1658-1702`）：`metacache && !force_update` 时读锁直接命中；否则先检查是不是 `LOCAL_SEGMENT_ID`（本地段永不走网络），再锁外 fetch，写锁安装。安装前调 `shouldInstallFetchedSegmentDesc`（`:124-156`）：

- 新旧任一为 null → 装；
- 任一版本为 0（老发布者没写版本）→ 装；
- `new_version < old_version` → **拒绝**（WARNING "Rejecting stale segment metadata"）；
- `new_version == old_version && *old != *new` → **拒绝**（ERROR "Rejecting conflicting segment metadata with unchanged version"）。

`SegmentDesc::operator==`（`:1456-1468`）**排除** `timestamp`、`metadata_version`、`buffer_range_index`、`interned_nic_paths`（前两者会被发布刷新，后两者是派生数据）。

`getSegmentDescByID`（`:1704-1738`）：`metacache` 打开且非 force 时纯读锁查表；否则先从表里取 name（读锁）、锁外 fetch、写锁仲裁安装。这是热路径——`selectTransport` 每次选路都调它；默认 `metacache=true` 时只是一次读锁哈希查找。

`getRpcMetaEntryInternal`（`:1973-2016`）与 segment 缓存不同：它在**持有 `rpc_meta_lock_` 写锁的情况下**调用 `storage_plugin_->get(...)`（`:1983`、`:1996`），也就是**网络 I/O 在锁内**。这能避免同一 server_name 的重复抓取，但元数据服务慢时会阻塞所有 RPC 地址查询。此处未见注释说明这是刻意取舍，标记为"设计意图未完全确认"。

#### D.5 `syncSegmentCache`：为什么区分 kNotFound 与 kUnavailable

`GetResult` 三态（`transfer_metadata.h:50-54`）就是为此设计的：

- `kNotFound` = 后端**权威地**说 key 没了（master 在 peer unmount/过期时删了 key）；
- `kUnavailable` = curl 超时、etcd 抖动、连接断开等瞬态失败。

`MetadataStoragePlugin::getWithStatus` 的**默认实现把任何失败都映射成 `kUnavailable`**（`include/transfer_metadata_plugin.h:38-41`），这是刻意的 fail-closed：无法区分"缺失"与"故障"的插件绝不能报 `kNotFound`，否则 `syncSegmentCache` 会在后端故障时驱逐仍活着的缓存。

`syncSegmentCache`（`transfer_metadata.cpp:1470-1656`）三阶段：

**① 收集（读锁，`:1475-1483`）**：遍历 `segment_id_to_desc_map_`，跳过 `LOCAL_SEGMENT_ID`，按可选 `segment_name` 过滤，把名字收集到 vector。**锁在网络 I/O 之前释放。**

**② 抓取（无锁，`:1503-1521`）**：每个 name 调 `getSegmentDescInternal(name, true, &status)`：

- 成功 → `updates`，`fetched_count++`；
- `kNotFound` → `failed_names`，`failed_count++`，WARNING "segment X is now invalid"；
- `kUnavailable` → 只 `unavailable_count++`，**不进 `failed_names`**（注释：key 可能还在，不得推进失效连击）。

**③ 应用（写锁，`:1523-1629`）**：

- 成功抓取：`segment_failure_counts_[name] = 0`；查 name→id；`shouldInstallFetchedSegmentDesc` 仲裁；`*old != *new` 才算 changed；unchanged 但版本更高也替换（`:1553-1561`）；changed 则替换 + WARNING + **节流 0.5s 的 desc dump**（`:1576-1593`，`static std::atomic<uint64_t>` + CAS，`trace` 时无条件打印）。
- `failed_names`：`++segment_failure_counts_[name]`；再扫计数表，**只有 `>= kStaleSegmentFailureThreshold`（= 2，`:186`）才真正 erase**（同时删 id→desc 与 name→id 两张表），并跳过 `LOCAL_SEGMENT_ID`。

**④ 清 RPC 缓存（另一个写锁，`:1634-1639`）**：只对通过版本仲裁的 `updated_names` 擦 `rpc_meta_map_`。

设计意图（`:1596-1605` 长注释）：元数据服务一次抖动不能清缓存，否则会把仍在飞的传输（调用方还持有旧 `shared_ptr`）搁浅；只有**跨同步周期持续的权威缺失连击**才说明 segment 真的没了。

两个使用注意：

- `syncSegmentCache` **永远返回 0**（`:1655`），抓取失败只体现在日志统计行与 `invalidated_count`。想据此做健康判断的调用方得解析日志或自己数。
- 一旦某个 segment 被判定失效并从缓存 erase，`openSegment` 拿到的旧 ID 就查不到了（`getSegmentDescByID` 返回 nullptr），调用方必须重新 `openSegment`。这也是 Python/Store 侧要在非 OK 状态时重新打开 segment 的原因。

后台刷新：`startMetadataRefreshPollingIfNeeded`（`:363-380`）只在 `metacache` 为真且 `te_metadata_refresh_interval_seconds != 0` 时启线程；循环 `metadataRefreshPollingLoop`（`:390-415`）用可被停机打断的 `cv.wait_for`，并 try/catch 兜住所有异常。默认间隔 0，即**默认不起这个线程**。

#### D.6 三个存储插件与 P2P 握手

`MetadataStoragePlugin::Create`（`transfer_metadata_plugin.cpp:708-760`）按 `parseConnectionString`（`:689-706`，`proto://domain`，无 `://` 时 proto 默认 `"etcd"`）分派：

- `etcd` → `EtcdStoragePlugin`（`#ifdef USE_ETCD`）；
- `redis` → `RedisStoragePlugin`（`#ifdef USE_REDIS`，附带 `MC_REDIS_USERNAME`/`MC_REDIS_PASSWORD`/`MC_REDIS_DB_INDEX`）；
- `http`/`https` → `HTTPStoragePlugin`（`#ifdef USE_HTTP`，收完整 conn_string）；
- 其余 → **`LOG(FATAL)`（直接 abort），不是返回 nullptr**（`:756-759`）。

etcd 还进一步分叉：`USE_ETCD_LEGACY` 用 etcd-cpp-api（C++/gRPC），否则链接 `mooncake-common` 构建的 `libetcd_wrapper.so`（Go/C 桥接）——两个同名 `struct EtcdStoragePlugin` 被 `#ifdef` 互斥编译。

三个插件的 `getWithStatus` 差异（这是 `kNotFound` 权威性的唯一来源）：

| 插件 | 权威缺失 | 其他失败 |
|---|---|---|
| Redis（`:181-214`） | `resp->str == nullptr`（nil reply） | `!resp`、`REDIS_REPLY_ERROR`、JSON 解析失败 → `kUnavailable` |
| HTTP（`:374-416`） | HTTP **404** | curl 错误、非 200 非 404、JSON 解析失败 → `kUnavailable`；GET 超时 3s / 连接 1.5s |
| Etcd legacy（`:537-554`） | RPC `is_ok()` 且 value 为空 | RPC 失败 → `kUnavailable` |
| Etcd wrapper（`:627-652`） | `EtcdGetWrapper` 返回 0 且 `json_data == nullptr` | 非 0 → `kUnavailable` |

`SocketHandShakePlugin`（`:782-1427`）：

- 监听线程 `accept` → 设 5s 收发超时 → `readString` → 按 type 分派到四个回调之一 → `writeString` 回同一 type → `shutdown(SHUT_WR)` → 再 `read` 等对端关闭 → `close`。**每个连接同步串行处理，没有 per-connection 线程池**。
- 四种帧类型定义在 `common.h:65-73`：`Connection=0`/`Metadata=1`/`Notify=2`/`Probe=3`/`Invalid=0xfe`/`OldProtocol=0xff`。`writeString` 写 `uint64 length + uint8 type + payload`；`readString` 校验长度上限 `MC_HANDSHAKE_MAX_LENGTH`（默认 1MB）。
- `doConnect`（`:1118-1217`）用**非阻塞 connect + poll**，超时 `handshake_connect_timeout`（默认 5s）。注释指出：`SO_RCVTIMEO` 不作用于 `connect()`，对不可路由地址的阻塞 connect 会卡满内核 SYN 重试周期（数分钟），而这段代码跑在 RDMA worker 线程上，会顺带阻塞 CQ 轮询。
- 发送端 `send/sendNotify/sendProbe/exchangeMetadata` 都遍历 `getaddrinfo` 结果逐个尝试，遇到 `ERR_MALFORMED_JSON` 立即放弃。

`receivePeerNotify`（`transfer_metadata.cpp:434-456`）在写锁下把 notify 追加进 `notifys`；**队列满时驱逐最老一条**（上限 `MC_HANDSHAKE_MAX_NOTIFY_ENTRIES`，默认 1024），注释指出这是防 RSS 无界增长（#4445），因为 daemon 接受任何 peer 的 notify。`getNotifies`（`:466-473`）一次取走并清空。

### 流程 E：拓扑发现与选卡

`Topology::discover`（`topology.cpp:704-728`）：`listInfiniBandDevices(filter, exclude_efa)` → `discoverCpuTopology` →（GPU 构建）`discoverCudaTopology` →（`USE_UB`）UB 设备 → `resolve()`。

`listInfiniBandDevices`（`:206-323`）的过滤链：

1. `filter`：构造 `TransferEngine(auto_discover, filter)` 传入的设备白名单；
2. `MC_TE_FILTERS`：环境变量白名单（`getIbvDeviceWhitelist`，`:164-193`，逗号分隔 + trim）；
3. `exclude_efa && isEfaDevice(name)`（`:196-203`，读 `/sys/class/infiniband/<dev>/device/driver` 的 realpath basename 是否为 `"efa"`）；
4. `isIbDeviceAvailable`（`:105-143`）：真的 `ibv_open_device` + `ibv_query_port`，要求至少一个端口 `IBV_PORT_ACTIVE` 且 `gid_tbl_len > 0`；`isIbDeviceAccessible`（`:49-76`）还要求 `/dev/infiniband/<dev>` 是字符设备且有读写权限；
5. 取 PCI bus id 与 `numa_node`。

**为什么 RDMA 发现要排除 EFA**：`topology.h:79-81` 与 `transfer_engine_impl.cpp:318-320` 的注释都写了——EFA 是 verbs 设备，但只暴露 SRD，RDMA transport 需要的 RC QP 用不了。配置 EFA transport 时 `use_efa` 为真，`exclude_efa` 传 false。这一条不看注释几乎不可能从代码猜出来。

`discoverCudaTopology`（`:556-673`）两种模式：

- **Data Direct 配对**（`MC_RDMA_DATA_DIRECT` + `MC_RDMA_DATA_DIRECT_PAIRING`）：只把 Data Direct function 与 GPU 在同一 PCI root 的 NIC 放进 `preferred_hca`，`avail_hca` 留空。**fail-closed**：任何 NIC 没有 Data Direct function、或任何 GPU 找不到配对 NIC，都 `*rc = ERR_DEVICE_NOT_FOUND` 并返回空，**不回落到 PCIe 距离**（注释：回落可能选中一张根本无法注册该内存的卡，导致 lkey/rkey 为 0）。
- **PCIe 距离**：先筛与 GPU 同 NUMA 的 NIC（`isSameNumaNode`，`:545-554`；两个 `numa_node` 都非 -1 且相等），非空则只在其中选；一遍扫描找最小 `getPciDistance`（`:515-543`，比较 `/sys/bus/pci/devices/<bus>` 的 realpath，从公共前缀之后数 `/`），最小的进 `preferred_hca`，其余进 `avail_hca`。

`resolve()` 的产物 `hca_list_` 是 transport 的 `device_id` 空间。RDMA 侧真正把"一次传输"接到卡上的是 `pickTopologyDevice`（`rdma_transport.cpp:1402-1436`）：

1. 算 location：默认是 `buffer.name`；若为 `segments:` 编码，则 `parseSegmentsLocation` + `resolveSegmentsLocation(seg_info, buffer.length, offset - buffer.addr)` 按偏移惰性解析出 `cpu:N`；
2. 有 hint 时先 `getDeviceIndex(location, hint)`，miss 再 `getDeviceIndex("*", hint)`；
3. `selectDevice(location, retry_count)`，miss 再 `selectDevice("*", retry_count)`。

`selectDeviceImpl`（`:1438-1490`）负责找**覆盖 `[offset, offset+length)` 的 buffer**：有 `BufferRangeIndex`（buffer 列表无重叠）时先试 `hint_buffer_id`，再 `findCovering` 二分；索引不可用（列表有重叠）时退回原始顺序线性扫描，以保持"首个匹配"语义（`buffer_range_index.h:44-49`）。`retry_count==0` 时允许复用 `hint_device_id`，但 `segments:` buffer 例外——它的 location 随偏移变化，不能复用。

### 流程 F：优雅退出

**这不是 RAII**：需要显式调 `TransferEngine::enableGracefulShutdown()`（classic `transfer_engine.cpp:372-377`）或 C API 的 `enableGracefulShutdown`（`transfer_engine_c.cpp:255-258`），才会注册 `ShutdownToken` 并安装信号处理器。

`installGracefulShutdownHandlers`（`graceful_shutdown.cpp:181-211`）在 `g_install_mutex` 下：

1. 已安装且 pid 匹配 → 直接返回（幂等）；
2. 首次注册 `atexit(atexitCleanup)`；
3. `startSignalWatcherLocked`（`:98-144`）：关旧 pipe、`pipe(g_signal_pipe)`、**先 `pthread_sigmask(SIG_BLOCK, {SIGTERM,SIGINT}, saved_mask)` 再 `std::thread(signalWatcher).detach()`**，记录 `g_handlers_pid = getpid()`；
4. `sigaction` 装 `shutdownSignalHandler` 到 SIGTERM/SIGINT，`sa_mask` 同时阻塞这两个信号（防重入），`sa_flags = 0`（不带 `SA_RESTART`）；
5. **最后**才恢复原掩码。

第 3 步的顺序是硬约束（注释 `:110-116`）：handler 写完管道就永远 `pause()`；如果它跑在 watcher 线程上就没人读管道、清理永不执行，所以 watcher 必须继承屏蔽掩码。第 5 步放最后（`:208-209`）：安装期间到达的信号保持 pending，直到 handler 就位，否则 disposition 还是 `SIG_DFL` 时信号会直接杀死进程。

`shutdownSignalHandler`（`:146-167`）是 async-signal-safe 的：

- **`g_handlers_pid != getpid()` → `_Exit(128+signo)`**：`fork()` 后子进程继承 handler 与 pipe fd 但没有 watcher 线程，`pause()` 会永远阻塞；
- 首个信号写一字节进 pipe；写失败也 `_Exit`；
- 然后 `for(;;) pause();` —— **主线程永远停在这里**。

`signalWatcher`（`:83-96`）在 detached 线程上 `read` 管道（`EINTR` 重试），读到一字节就 `cleanupEngines()` 然后 `_Exit(128+signo)`。`cleanupEngines`（`:72-79`）用 `g_cleanup_started.exchange(true)` 保证只跑一次（atexit 与信号路径共用），对 `collectTokensForCleanup()` move 出的每个 token 调 `shutdown()`。

`TransferEngineShutdownToken`（`transfer_engine.cpp:49-72`，classic 与 TENT 分支各有一份）持裸 `TransferEngine*` + 自己的 mutex；`shutdown()` 在锁内取出并置空指针后**在锁外**调 `engine->freeEngine()`；`detach()` 只置空指针。**移动构造/移动赋值会 detach 旧 token 并对新地址重新注册**（`:96-124`），否则信号处理器会调一个已经被移空的对象。另一个实现 `TransferEngineImplShutdownToken`（`graceful_shutdown.cpp:47-63`）持 `weak_ptr<TransferEngineImpl>`。

`TransferEngineImpl::freeEngine`（`transfer_engine_impl.cpp:545-551`）做的事很少：`metadata_->removeRpcMetaEntry(local_server_name_)` + `metadata_.reset()`。它**不**逐个 unregister 内存、也不显式销毁 transport；这些发生在 `TransferEngineImpl` 析构时（成员 `multi_transports_` 销毁 → `MultiTransport` 析构 → transport 析构，例如 `RdmaTransport::~` 会 `removeSegmentDesc`）。`TransferEngine::freeEngine`（`transfer_engine.cpp:147-155`）额外清 `metadata_conn_string_`、detach token，并且**只在 `impl_.use_count() == 1` 时**才调 `impl_->freeEngine()`——正在进行的 `ScatterTransferOperation` 持有 `shared_ptr<TransferEngineImpl>`，此时 engine 先放手，等 operation 结束时由 `~TransferEngineImpl` 收尾。

### 流程 G：`showLinks` 诊断

`src/show_links.cpp`，**只做本地诊断输出，不涉及任何链路协商**；名字里的 "links" 容易被误读成连接管理。

`collectLocalNics`（`:29-62`）：拿 `impl->getTransport("rdma")`；拿不到时退化用 `getLocalTopology()->getHcaList()` 造只有设备名的 `NicDiagInfo`（`numa_node=-1`、`gid_index=-1`，输出会标注 "topology only; RDMA transport not initialized"）；拿到时 `static_cast<RdmaTransport*>` 后遍历 `getContextList()` 收集 `deviceName/socketId/gid/gidIndex/portNum/activeSpeed/activeWidth`。

`buildShowLinksJson`（`:73-99`）输出 `{local_nics:[{device,numa,gid,gid_index,port,speed_gbps,source}], topology:<priority matrix>}`；`buildShowLinksReadable`（`:101-141`）输出人读文本。速率由 `ibLinkSpeedGbps(active_speed, active_width)` 算出并截断成整数 Gbps。TENT 模式下 `showLinks` 返回 `"{}"` 或占位串（`transfer_engine.cpp:1245-1251`）。

### 流程 H：TENT 兼容层的边界

`USE_TENT=ON` 的构建里，`TransferEngine` 在**运行期**根据 `MC_USE_TENT` / `MC_USE_TEV1` 决定走哪套（`transfer_engine.cpp:534-542`、`:571-582`）：

- 未设 → `impl_` 是 classic `TransferEngineImpl`；
- 设了 → `impl_tent_` 是 `tent::TransferEngine`，classic API 退化为兼容层。

兼容层的几个关键事实：

- `tentToClassicError`（`:413-445`）做 `tent::Status::Code → ERR_*` 映射；两套错误码数值完全不同，`default` 统一落到 `ERR_CONTEXT`。
- `installTransport` 在 TENT 下是 **no-op**，返回一个 `TentCompatibilityTransport` 单例（`:447-530`、`:726-743`）；它的 `allocateBatchID` 返回 `INVALID_BATCH_ID`，`submitTransfer` 返回 `NotImplemented`。因此 classic 调用方只能把返回的 `Transport*` 当作"已安装"的能力标志，不能真的拿它提交。
- `allocateSharedMemory` 返回 nullptr；`showLinks` 返回占位串；`closeSegment` 映射 TENT 错误码；`CheckSegmentStatus` 主动 probe。
- `buildTentConfig`（`:642-660`）把 conn_string 拆成 `metadata_type`/`metadata_servers`，P2P 映射成 `metadata_type=p2p`，设备 filter 映射成 `topology/rdma_whitelist`。
- `ScatterTransferOperation` 的共享实现通过 `Backend{legacy, tent}` 同时支持两套（`:1331-1865`）；TENT 路径支持 post-submit 取消，classic 不支持。

---

## 与其他模块的边界

### 依赖谁

| 依赖 | 用途 | 是否可选 |
|---|---|---|
| `libibverbs` / `rdma-core` | `topology.cpp`、`config.h` 直接 include `infiniband/verbs.h`；RDMA transport 的基础 | 硬依赖（classic 构建） |
| glog / gflags | `LOG`/`VLOG`/`PLOG`；`config.cpp:93` 直接引用 `google::INFO` 作为默认日志级别 | 硬依赖 |
| JsonCpp | SegmentDesc/Topology/握手帧的序列化 | 硬依赖 |
| libnuma | `numa_move_pages`；CMake 显式链接 | 硬依赖（Linux） |
| asio / yalantinglibs | RPC 与 metrics counter/histogram | `WITH_METRICS` 等 |
| `mooncake-common` 的 `Status` | 新 API 的返回类型 | 硬依赖 |
| hiredis | Redis 插件 | `USE_REDIS` |
| libcurl | HTTP 插件 | `USE_HTTP` |
| etcd-cpp-api 或 `libetcd_wrapper.so` | etcd 插件 | `USE_ETCD` |
| CUDA/HIP/MUSA/MACA/NCCL/libfabric/cuFile/Ascend CANN/Kunpeng UB | 各加速器 transport 与拓扑发现 | 各自 `USE_*` 宏 |
| `tent/` | 新一代引擎 | `USE_TENT` |

### 被谁依赖

| 调用方 | 入口 | 用到的接口 |
|---|---|---|
| `mooncake-store` | `src/client_service.cpp:918-1007`、`src/transfer_task.cpp` | `installTransport("tcp"/"cxl"/"cxi"/"ub"/"nvlink_intra")`、`registerLocalMemory`、`submitTransfer`、`getTransferStatus`/`getBatchTransferStatus`、`freeBatchID`、`isTcpOnly` |
| Python 绑定 | `mooncake-integration/transfer_engine/transfer_engine_py.cpp` | `initialize`、`installTransport("efa"/"flagcx"/"cxi"/"tcp")`、`openSegment`、批量传输、scatter |
| Go 绑定 | `mooncake-p2p-store`（cgo） | `transfer_engine_c.h` 全套 C ABI |
| `mooncake-ep` / `mooncake-reshard` | — | C++ API；reshard 侧再包一层有界在途窗口 |
| benchmark | `example/transfer_engine_bench.cpp:421-501` | 标准用法样板 |

### 接口契约在哪

- **`Transport` 抽象基类**（`include/transport/transport.h:44-623`）是引擎核心与所有链路实现之间唯一的契约面。它把 `TransferEngine`/`TransferEngineImpl`/`MultiTransport` 声明为 friend（`:45-47`），把 `registerLocalMemory`/`unregisterLocalMemory`/`registerLocalMemoryBatch`/`unregisterLocalMemoryBatch`/`getName` 放在 **private 纯虚**（`:606-622`）——只有引擎能注册内存，外部拿到 `Transport*` 也不能调。
- **`submitTransferTask` 的同步发布契约**（`:550-552`）：实现必须在返回前同步发布所有 slice，返回后任何 worker 不得再追加或发布。`ScatterTransferOperation::Impl::legacy_submit_failed_`（`transfer_engine.cpp:1861-1863`）正是依赖这条契约来判定"提交失败且 `slice_count == 0` ⇒ 未发布 ⇒ 可安全放弃"。
- **两个能力查询**：`supportsGroupedScatter()`（`:566`，默认 false）与 `supportsBatchCompletionCounter()`（`:572`，默认 false）。后者注释明确要求"所有 task 都不需要 `getTransferStatus()` 驱动进度才能 opt in"。
- **数据怎么流过去**：`TransferRequest`（值语义，但被 task 以裸指针引用）→ `TransferTask`（引擎侧，`BatchDesc::task_list` 按值持有）→ `Slice`（transport 侧，thread-local 池化，`task.slice_list` 持裸指针）→ 硬件 WR。返回方向：CQ worker → `Slice::markSuccess/markFailed` → `TransferTask` 计数器 → `getTransferStatus` → `TransferStatus{s, transferred_bytes}`。
- **元数据怎么流过去**：`SegmentDesc` 通过 `shared_ptr` 快照流动。transport 拿到 `metadata_->getSegmentDescByID(...)` 的 `shared_ptr` 后，即使 `syncSegmentCache` 并发替换表里的条目，它看到的仍是一致快照。
- **TENT 边界**：classic API 在 TENT 模式下仍可用，但 `installTransport`/`allocateSharedMemory`/`showLinks` 是兼容性占位；错误码经过 `tentToClassicError` 映射；`USE_ASCEND_DIRECT` 等构建会改变 `getTransferStatus` 的 notify 副作用路径。

---

## 设计取舍与易踩坑点

### 为什么这样设计

**1. 为什么 `BatchID` 是指针而不是索引/自增 ID？**
热路径上 `submitTransfer`/`getTransferStatus` 每次都要拿 `BatchDesc`。map 查找意味着每次轮询取锁 + 哈希。`toBatchDesc` 是一次 `reinterpret_cast`，零开销（`transport.h:151-161` 注释明确说 "intentionally bypasses any map or lookup to minimize overhead on hot paths"，并把"保证对象存活"的责任交给调用方）。代价是 BatchID 不可校验，传野值是 UB；`batch_desc_set_` 那套调试映射默认不编译。

**2. 为什么完成状态要打包进一个字？**
见 §3.5。两个独立原子量的 Dekker 竞态会让 batch 永久卡在 WAITING，且只在极窄时序出现、几乎无法复现。打包后靠单变量全序天然免疫。代价是 `completion_state` 语义不直观（bit63 是标志、低 63 位是计数），且 `success_slice_count`/`failed_slice_count` 仍作为冗余计数保留（注释："packing both into one word is a follow-up"）。

**3. 为什么需要 `CompletionCallbackPin` 而不是让 `freeBatchID` 等 `is_finished`？**
`is_finished`（或计数相等）表示"完成已发布"，但发布动作本身还在 CQ 线程栈上——`finishIfComplete` 之后还要 `fetch_add(finished_task_count)`，EDC 下还要取 mutex 和 `notify_all`。这些都是对 `BatchDesc` 的裸访问。pin 计数是唯一能表达"回调还在栈上"的量。代价是 `freeBatchID` 可能返回 `BatchBusy`，需要 MultiTransport 的延迟清理线程把它对引擎调用方隐藏成 `BatchCleanupDeferred`。

**4. 为什么 `getBatchTransferStatus` 要有计数器快路径？**
一个 batch 可能有几百个 task，每个都要走一次虚函数 + 原子读。RDMA 的完成由 CQ worker 自驱，前台轮询**不推进任何事情**，纯粹是读计数。既然 `finished_task_count` 已由 `finishIfComplete` 维护，那么"能力满足 + 无超时需求 + 无失败 + 未全完成"时直接返回 WAITING 是 O(1)。代价是 WAITING 期间字节数报 0——被明确写进 API 文档作为合法下界。

**5. 为什么元数据要 COW + 版本仲裁 + 三态 `GetResult`？**
`SegmentDesc` 在飞传输期间被并发读取，发布者可能有多个（多 transport 追加 protocol、buffer 增删）。COW 保证读者看到一致快照；`metadata_version`（墙上微秒）保证不会用旧数据覆盖新数据，并能检出"同版本不同内容"这种发布者 bug；三态 `GetResult` 保证元数据服务抖动不会误清缓存把在飞传输搁浅。三者缺一都会在生产环境出难查的问题。

**6. 为什么 `registerLocalMemory` 要两阶段预留？**
见流程 C。单表方案在长耗时 MR 注册期间对重叠注册完全不设防。

**7. 为什么 P2P 是独立模式而不是存储插件？**
P2P 场景没有中心元数据服务，SegmentDesc 只能向对端实时换取。把 `storage_plugin_` 置空、让 `getSegmentDescInternal` 走 `exchangeMetadata`，避免在插件接口里塞一个"没有后端"的特殊实现。代价是 publish 系列 API 全部退化为 no-op，且 P2P 下没有"权威缺失"的概念——所有失败都是 `kUnavailable`。

**8. 为什么 Topology 矩阵要随 SegmentDesc 发布？**
发起端既要选本地 RNIC，也要选远端 RNIC。远端 NIC 的 NUMA/GPU 亲和只有远端自己知道，所以拓扑必须随 SegmentDesc 一起广播；发起端用**远端发布的拓扑**来选远端 device_id，再按这个下标去索引远端 `devices[]`/`rkey[]`。

### 读代码/改代码时容易误解的地方

1. **`src/transfer_engine.cpp` 里同名函数有两份。** `openSegment` 在 `:172`（classic）与 `:773`（TENT）各一份，`init` 在 `:128`/`:138` 与 `:662`/`:670` 各一份。引用行号前必须说明分支（`#ifndef` 段 `:38-385`，`#else` 段 `:386-1254`，`:1256` 之后共享）。

2. **两套互不兼容的错误模型。** 返回 `int` 的旧 API 用 `include/error.h` 的 `ERR_*`（`ERR_INVALID_ARGUMENT=-1` … `ERR_NOT_IMPLEMENTED=-303`）；返回 `Status` 的新 API 用 `include/common/base/status.h:33-55` 的 `Status::Code`（`kOk=0` … `kNotImplemented=999`）。数值完全对不上，TENT 分支专门写 `tentToClassicError`。读代码时按函数签名判断该用哪套。

3. **`submitTransfer` 返回 OK ≠ 每个请求都成功入队，也 ≠ 一个请求对应一个 task。** 合并只发生在 scatter 路径（见 §A.2⑤）；部分提交不回滚（见 §A.2⑥）。

4. **`openSegment` 不建连接，`closeSegment` 是空实现。** 但 `openSegment` 可能触发网络 I/O，失败返回 `(uint64_t)-1` 而不是具名常量；classic 的 `CheckSegmentStatus` 恒 OK。

5. **`installTransport` 的 "Only for testing" 注释已过期。** Store 与 Python 绑定都在生产路径上调它。

6. **`Topology::selectDevice` 返回下标不是名字**，且 `ERR_DEVICE_NOT_FOUND = -6` 与下标共用返回值域。调用方必须区分 `<0` 与 `>=0`。

7. **`showLinks` 不做链路协商**，只是本地诊断渲染。

8. **`getTransferStatus` 有发送 notify 的副作用**（`impl.h:351-361`），`USE_ASCEND_DIRECT` 构建下没有。

9. **`SegmentDesc::buffers[i].name` 存的是 location 字符串**，不是名字。RDMA 的 `pickTopologyDevice` 直接把它当 `storage_type` 用。

10. **`getBatchTransferStatus` 的 `has_failure.store` 失败分支不可达**（`multi_transport.cpp:561-563`）。真正的失败标记在 `markFailed` 的 `finishIfComplete` 里。

11. **`transfer_metadata*.cpp` 是三个文件**：`transfer_metadata.cpp`、`transfer_metadata_plugin.cpp`、`transfer_metadata_dump.cpp`。不存在 `transfer_metadata_c.cpp`；C API 是 `src/transfer_engine_c.cpp` + `include/transfer_engine_c.h`，其中 `segment_id_t` 是 `int32_t`（`transfer_engine_c.h:26`）而 C++ 的 `SegmentID` 是 `uint64_t`——**跨语言边界有窄化**。

12. **`graceful_shutdown` 不是 RAII**，必须显式 enable；装上后主线程的 handler 永远停在 `pause()`，真正的清理与 `_Exit` 在 detached 线程。

13. **`ENABLE_MULTI_PROTOCOL` 默认 OFF。** 默认构建下 `selectTransport` 的逗号协议打分分支（`multi_transport.cpp:799-875`）不参与编译，`BufferDesc::protocol` 字段不存在，`allocateLocalSegmentID` 直接覆盖 `protocol` 而不是追加，`MC_FORCE_SHM` 走"SHM-only 短路"而不是"追加 SHM"。

14. **`USE_EVENT_DRIVEN_COMPLETION` 默认 OFF。** 默认下 `finishIfComplete` 只走 `fetch_add(finished_task_count)`；正常完成路径上 `task.is_finished` 由前台 `getTransferStatus` 写，CQ 线程不写（例外：`tryFreeBatchID` 对 `slice_count==0` 的空 task、scatter 的 `legacyBatchPhysicallyDrained` 会兜底写）。EDC 下 CQ 线程与前台都写，`legacyBatchPhysicallyDrained` 直接返回 false 让完成线程自己发布。

15. **`CONFIG_USE_BATCH_DESC_SET` 从未被定义**，`batch_desc_set_` 默认永远是空的。

16. **`MC_FORCE_TCP` / `MC_FORCE_HCA` / `MC_INTRANODE_NVLINK` 是"存在即真"**（`getenv(...) != nullptr`），`MC_FORCE_TCP=0` 也会强制 TCP；而 `MC_FORCE_SHM` 用 `envFlagEnabled`，`0/false/off/no` 视为关闭。同一个前缀，两种语义。

17. **EFA 排除只发生在 RDMA 发现路径。** `Topology::discover(filter, exclude_efa)` 的第二个参数由 `autoDiscoverUsesEfa()` 决定；配置 EFA transport 时必须不排除。

18. **多协议的本地 IPC 门控只覆盖 `hip`/`musa`/`shm`**（`multi_transport.cpp:841-845`），`maca` 与 `hylink` 不在门控里。跨主机请求遇到 `maca` buffer 不会被跳过，判断路由时要留意。

19. **`getRpcMetaEntryInternal` 在写锁内做网络 I/O**（`transfer_metadata.cpp:1983-1996`），与 segment 缓存"锁外抓取"的风格相反。元数据服务慢时会串行阻塞所有 RPC 地址查询；设计意图未完全确认。

20. **`syncSegmentCache` 永远返回 0。** 抓取失败只进日志统计；想据此做健康检查得自己解析或另建指标。

21. **deferred 清理后的 BatchID 不可再次释放。** `freeBatchID` 第二次调用会 `toBatchDesc` 解引用可能已 delete 的指针。

22. **`C API` 的 `BATCH_CLEANUP_DEFERRED(5)` 与 `Status::Code::kBatchCleanupDeferred(5)` 对齐，但与 `ERR_BATCH_CLEANUP_DEFERRED(-5)` 只差符号。** 三个常量分散在两个头文件，改动必须同步。

23. **`ScatterTransferOperation` 的析构会阻塞等待传输完成**，且 `waitFor` 超时不会取消；buffer 必须保持存活。

24. **TENT 模式下 `installTransport` 返回的 `Transport*` 是占位对象**，不能拿它 `submitTransfer`；`allocateSharedMemory` 返回 nullptr。

---

## 精读清单

按下面的顺序读，每一步都标注"该看懂什么"。

### 第 1 步：契约面（`include/transport/transport.h`，约 630 行）

1. `:60-99` `TransferRequest` / `TransferStatusEnum` / `TransferStatus` —— 记住 `target_offset` 是远端地址；`task_group_id` 只在 scatter 路径生效；`nic_hint` 只影响首次提交。
2. `:151-164` `toBatchDesc` 与注释 —— **建立"BatchID 就是指针"的心智模型**，后面所有 `*((BatchDesc*)(batch_id))` 才看得懂。
3. `:116-149` `SliceTaskPtr` —— 理解为什么不能直接放 `std::atomic`。
4. `:168-262` `Slice` 字段与 union —— 只需知道"各 transport 往里塞私有字段"，不必记具体分支。
5. `:264-310` `kCompletionSealBit`/`markSuccess`/`markFailed`/`seal`/`unseal` —— **本章最难的一段，慢读**。看懂：pin 在最外层；终态判据同时依赖 `success+failed == slice_count` 与 `completion_state`；`markSuccess` 用 `add_fetch` 的返回值直接判定。
6. `:314-371` `finishIfComplete` —— 看懂 seal bit 检查、`completion_published` 的 exchange 去重、以及 EDC 两个分支各自谁写 `is_finished`。
7. `:372-402` `CompletionCallbackPin` —— 看懂它防的是哪个 UAF 窗口。
8. `:405-451` `ThreadLocalSliceCache` —— 看懂 `from_cache` 与 `cleanup_callback` 的清空时机。
9. `:453-530` `TransferTask` / `BatchDesc` 全字段 —— 对照 §3.3/§3.4；特别注意 `completion_state` 默认值与 cacheline 注释。
10. `:532-580` 虚接口 + 三条契约注释（同步发布、`supportsGroupedScatter`、`supportsBatchCompletionCounter`）。
11. `:606-622` private 纯虚 —— 理解 friend 边界。

配套读 `src/transport/transport.cpp`（74 行）：thread-local 缓存实例、基类版 `allocateBatchID`/`freeBatchID`（注意基类版**没有延迟清理**，直接返回 `BatchBusy`）、`install()` 只存 name 与 metadata。

### 第 2 步：批量传输语义（`src/multi_transport.cpp`，约 1000 行）

1. 头文件 `:51-54` `ScatterSubmission`、`:58-60` `freeBatchID` 语义、`:112-118` `allows_reuse` 的正确性前提。
2. `.cpp:107-120` `allocateBatchID` —— 确认指针即 ID，并认出 `ERR_MEMORY` 死代码。
3. `:122-201` `freeBatchID` / `tryFreeBatchID` —— **重点**：三重检查（task 终态、pin 归零、before_delete）、幂等的 deferred 判定、惰性起线程。
4. `:203-247` `deferredCleanupLoop` —— 100ms 轮询、永久错误也移除、停机时故意泄漏。
5. `:249-328` `submitTransfer` —— **本章第二难的一段**：按 §A.2 的六步逐段对照。确认 `reserve` 的必要性、`unseal` 时机、合并的四个前提、seal 的补偿循环、部分提交不回滚。
6. `:330-341` `submitScatter` —— `allocateBatchID(0)` + 传 `task_sizes` 的组合。
7. `:393-464` `getTransferStatus` —— 委托给 transport 的理由（NVLink 的 `cudaStreamQuery`）、超时叠加、遗留回退路径。
8. `:466-501` `getScatterRequestStatuses` —— slice 布局假设与失败恢复。
9. `:503-565` `getBatchTransferStatus` —— **重点**：三条路（cached / counter 快路径 / 全量遍历）与 `has_failure` 短路径；注意 FAILED 分支不可达。
10. `:567-744` `installTransport` —— 扫一遍协议→类映射与两条互斥规则即可，不必细读 `#ifdef`。
11. `:746-898` `selectTransports` / `selectTransport` —— 批量选路缓存的正确性前提；多协议打分分支（默认不编译）；`rdma → rdma_twosided` 降级；`isLocalIpcReachableTarget` 剔除跨主机 GPU-IPC。

### 第 3 步：引擎生命周期（`src/transfer_engine_impl.cpp` + 头文件）

1. `impl.h:57-68` `shouldInstallAscendTransport` —— 三个构建分支的语义。
2. `impl.cpp:77-94` `setFilesLimit`、`:96-106` `loadTopologyJsonFile`。
3. `:108-221` `init` 的前半：fd 上限、名字解析、RPC 三种绑定方式。**重点理解 P2P 为什么改写 `local_server_name_`。**
4. `:223-255` metadata/MultiTransport 创建、`addRpcMetaEntry`、`MC_FORCE_TCP` 短路。
5. `:257-314` SHM/Ascend/UBSHMEM/CXL 分支。
6. `:316-539` auto-discover 决策树 —— 按表格逐个分支确认；理解 `MC_CUSTOM_TOPO_JSON` 的回落逻辑与"discover 返回码未检查"。
7. `:541-543` 收尾。
8. `:545-551` `freeEngine` —— 只摘 RPC meta，别以为它会拆 transport。
9. `:554-592` `installTransport` —— 补注册内存、`args[0]` 拓扑矩阵。
10. `:722-764` `openSegment`/`CheckSegmentStatus`/`closeSegment`。
11. `:780-841` `registerLocalMemory`/`unregisterLocalMemory`。
12. `:963-1028` 批量注册；`:1046-1114` 重叠判定与两阶段表。

### 第 4 步：门面与 TENT 边界（`src/transfer_engine.cpp`，约 1900 行）

1. `:38-385` classic 转发段 —— 扫一遍即可，重点是 `:147-155` `freeEngine` 的 `use_count()` 判断、`:96-124` 移动语义对 shutdown token 的处理。
2. `:386-530` TENT 段开头：`tentToClassicError`、`TentCompatibilityTransport`。
3. `:534-708` TENT 运行期开关、`buildTentConfig`、TENT `init`。
4. `:726-808` TENT 的 `installTransport`/`openSegment`/`CheckSegmentStatus`/`closeSegment`。
5. `:817-960` TENT 的内存注册与提交转换。
6. `:1256-1329` 共享段：opcode/intent 转换、`getSegmentBuffers`。
7. `:1331-1865` `ScatterTransferOperation::Impl` —— **重点**：`build` 的 fragment 展开与错误回调、`poll` 的 task→fragment 映射、`legacyBatchPhysicallyDrained`、`requestAbort` 的 TENT/classic 差异。
8. `:1893-1910` `submitScatter`/`transferScatter`。

### 第 5 步：元数据（`src/transfer_metadata.cpp` 约 2100 行 + `transfer_metadata_plugin.cpp` 约 1550 行）

1. `:50-78` `GetResult`/`nextWallTimeMetadataVersion`。
2. `:124-156` `shouldInstallFetchedSegmentDesc` —— 四条仲裁规则。
3. `:186` `kStaleSegmentFailureThreshold` 与 `:304-355` 构造函数（key 前缀、P2P 分支）。
4. `:417-432` `getFullMetadataKey` —— 名字含 `/` 的特殊规则。
5. `:558-768` `encodeSegmentDesc` —— 至少看懂 P2P 模式下哪些字段被编码（`priority_matrix`、buffers、devices）。
6. `:997-1309` `decodeSegmentDesc` —— fail-closed 校验。
7. `:1335-1421` `getSegmentDescInternal` —— P2P vs 存储两条路、tcp 兼容回填。
8. `:1456-1468` `SegmentDesc::operator==` 排除了哪些字段。
9. `:1470-1656` `syncSegmentCache` —— **本章第三难**：三阶段、kNotFound vs kUnavailable、连击阈值、RPC 缓存失效。
10. `:1658-1760` `getSegmentDescByName`/`getSegmentDescByID`/`getSegmentID`。
11. `:1762-1864` `updateLocalSegmentDesc`/`addLocalMemoryBuffer`/`removeLocalMemoryBuffer` —— COW 与版本号。
12. `:1866-2016` `addRpcMetaEntry`/`removeRpcMetaEntry`/`getRpcMetaEntryInternal` —— P2P 下启动 daemon；注意写锁内网络 I/O。
13. `:2018-2119` `startHandshakeDaemon`/`sendHandshake`/`sendNotify`/`sendProbe`。
14. `transfer_metadata_plugin.cpp:708-760` `Create` 分派与 `LOG(FATAL)`。
15. `:181-214` / `:374-416` / `:537-554` / `:627-652` 四个 `getWithStatus` 的权威缺失判据。
16. `:782-1008` SocketHandShakePlugin 监听线程；`:1118-1217` 非阻塞 connect + poll；`:1258-1293` `exchangeMetadata`；`:1429-1432` `Create` 恒返回 socket 插件。
17. `:1434-1479` `findLocalIpAddresses`；`:1480-1554` `findAvailableTcpPort`（保留已 bind 的 fd）。

### 第 6 步：拓扑、配置与 location（约 980 + 960 + 215 行）

1. `topology.h:38-146` 三个结构体与 `selectDevice` 声明。
2. `topology.cpp:49-143` 设备可用性检查；`:164-203` `MC_TE_FILTERS` 与 EFA 判定。
3. `:206-323` `listInfiniBandDevices` 过滤链。
4. `:436-509` `discoverCpuTopology`（两份重载：UB / IB）。
5. `:515-673` `discoverCudaTopology`：Data Direct fail-closed 与 PCIe 距离。
6. `:704-772` `discover`/`parse`。
7. `:794-884` `getDeviceIndex`/`selectDevice` 三个重载 —— 返回下标、随机 vs 轮转。
8. `:886-926` `resolve()` —— wildcard 合成与 `hca_list_` 构建。
9. `:928-977` `disableDevice` —— 为什么不重跑 resolve。
10. `config.h:42-201` 字段；`config.cpp:108-843` 环境变量解析（不必逐行，挑与本章相关的字段）；`:858-873` 硬件 clamp；`:932-937` 单例；`:941-961` 端口校验。
11. `memory_location.h` 全部；`memory_location.cpp:27-53` 页大小探测、`:85-155` NUMA 解析、`:161-212` segments 编解码。

### 第 7 步：两个辅助件

1. `graceful_shutdown.cpp` 全 213 行 —— 重点 `:98-144` 的 watcher 启动顺序、`:146-167` 的 async-signal-safe handler、`:181-211` 的安装顺序。
2. `show_links.cpp` 全 143 行 —— 只需知道它是诊断渲染，不参与传输。

### 第 8 步：下游用法与测试（验证理解）

1. `example/transfer_engine_bench.cpp:421-501` —— 标准 batch 生命周期；`:549-584` 手动装 transport 的两种方式。
2. `mooncake-store/src/transfer_task.cpp:928-960` —— EDC 的 `BatchDesc::completion_cv` 等待方，理解 TE 为什么需要 EDC。
3. `mooncake-store/src/client_service.cpp:918-1007` —— Store 如何在 `init` 之后按协议 `installTransport`。
4. `mooncake-transfer-engine/tests/batch_counter_wait_test.cpp` —— 用 `ProbeTransport` 精确验证了：快路径不轮询、WAITING 字节为 0、追加提交使缓存失效、失败可见、空 task 在 seal 后完成、提交失败保留回退轮询。**读完这 256 行，§A.5 就彻底清楚了。**
5. `mooncake-transfer-engine/tests/multi_transport_batch_test.cpp` —— 选路缓存、元数据替换、混合协议按地址路由、未支持目标不部分提交。
6. `mooncake-transfer-engine/tests/graceful_shutdown_test.cpp` —— 多引擎注册、重复 enable、移动后 token 重注册。

### 未完全确认之处

- `getRpcMetaEntryInternal` 在写锁内做网络 I/O 是否是刻意的（代码无注释，见 §D.4）。
- `TransferEngineImpl::init` 忽略 `Topology::discover` 返回码，是否期望上层用 `numContexts()==0` 自查（代码无注释）。
- `task_group_id` 在普通 `submitTransfer` 路径被完全忽略，是接口演进遗留还是刻意只服务 scatter，文档未说明。

---

# 03 RDMA 传输实现

本章讲经典 Transfer Engine（`mooncake-transfer-engine`）里真正执行跨节点单边 RDMA READ/WRITE 的那套代码，范围是
`include/transport/rdma_transport/`、`src/transport/rdma_transport/` 以及在其上派生的
`rdma_twosided`。所有路径均相对仓库根目录 `D:/xiaojun/work/Mooncake`。

> 先纠正一个容易找错门的名字：这套代码里**没有 `RdmaWorker` 这个类**。每个 `RdmaContext`
> 私有的工作线程组叫 `WorkerPool`（`include/transport/rdma_transport/worker_pool.h`，include guard
> 还是历史遗留的 `WORKER_H`）。仓库里叫 `RdmaWorkers*` 的是
> `mooncake-transfer-engine/tent/tests/` 下的测试夹具，测的是 TENT 的另一套
> `mooncake::tent::Workers`。TENT 里还存在一套同名但不同命名空间的
> `RdmaTransport/RdmaContext/RdmaEndPoint/EndpointStore/Workers`（声明在
> `mooncake-transfer-engine/tent/include/tent/transport/rdma/`，实现在
> `mooncake-transfer-engine/tent/src/transport/rdma/`），与本章无关，grep 时务必看目录。

---

## 定位与职责

### 它解决什么问题

Transfer Engine 对上层暴露 `BatchTransfer`：一批 `TransferRequest`，每条描述“从本地 `source`
读/写 `length` 字节到远端 `target_id` 的 `target_offset`”。RDMA 传输层的职责是把这批请求：

1. **切成 slice**（受 `slice_size`、两侧 MR 边界、fragment 阈值三重约束）；
2. **为每个 slice 选本地网卡**（并取到 `source_lkey`）；
3. **为每个 slice 选对端网卡**（并取到 `dest_rkey`）；
4. **按需建立 QP 连接**（本地 NIC ↔ 对端 NIC 一条 rail）；
5. **投递 WR、轮询 CQE、把完成/失败写回 `TransferTask` 计数器**；
6. **失败时换本地卡或换对端 rail 重投，并在持续故障时熔断**。

它对下直接调用 libibverbs（`ibv_post_send` / `ibv_poll_cq` / `ibv_modify_qp` / `ibv_reg_mr` /
`ibv_reg_dmabuf_mr`），可选调用 mlx5dv（LAG 端口、UDP 源端口、Data Direct、dma_buf 注册）。

### 四层分工

| 层 | 类 | 粒度 | 关键职责 |
|---|---|---|---|
| 传输层 | `RdmaTransport` | 每进程一个 | 实现 `Transport` 接口；切片；本地 NIC/lkey 选择；段元数据发布；握手 RPC 服务端；native notify 轮询线程 |
| 设备层 | `RdmaContext` | 每张本地 RNIC 一个 | `ibv_context`/PD、comp channel、CQ 列表、MR 表、`EndpointStore`、`WorkerPool`、epoll fd |
| 连接层 | `RdmaEndPoint` | 每对“本地 NIC ↔ 对端 NIC”一个 | N 个 QP；握手状态机；`ibv_post_send`；两阶段销毁；通知 QP |
| 执行层 | `WorkerPool` | 每 `RdmaContext` 一个池 | 对端 NIC/rkey 选择；按 peer 哈希归属 worker；CQ 轮询；故障分类与重派；三级熔断；异步事件恢复 |

一个容易讲错的地方：**选卡发生两次，分处两个类、两个线程**。
`RdmaTransport::submitTransferTask` 只选本地 NIC 与 `source_lkey`；
对端 NIC 与 `dest_rkey` 要等到 `WorkerPool::submitPostSend` 才选
（`src/transport/rdma_transport/worker_pool.cpp:296`）。只看 `rdma_transport.cpp` 会误以为提交时就路由完了。

### 线程模型

`RdmaContext::construct` 里 `transfer_worker_count_ = globalConfig().workers_per_ctx`（默认 2），
并把 `num_cq_list` 强行抬到不小于 worker 数（`src/transport/rdma_transport/rdma_context.cpp:243-249`；
`initializeRdmaResources` 里还有一次同样的抬升，`rdma_transport.cpp:1343-1349`）。

`WorkerPool` 构造时为每个 worker 起一个 `transferWorker(thread_id)`，**再额外起一个
`monitorWorker`**（`worker_pool.cpp:227-244`）。因此每张 NIC 的线程数是
`workers_per_ctx + 1`，外加 `RdmaTransport` 的一个全局 notify 线程（若启用 native notify）。

worker / CQ / peer 是**三重绑定**的：

- `postingThreadForPeer(path) = hash(path) % transfer_worker_count_`
  （`rdma_context.cpp:1110-1119`）；
- `cqIndexForPostingThread(thread_id)` 基本就是 `thread_id`（超出 CQ 数才取模，
  `rdma_context.cpp:1121-1126`）；
- `lockEndpointLifecycle(path)` 按 owner worker 取 `endpoint_lifecycle_locks_[owner]`
  （`rdma_context.cpp:1132-1141`）。

所以同一条 rail 永远由同一个 worker 投递、同一个 worker 轮询、落在同一个 CQ 上。
这解释了 `performPollCq` 里那句
`assert(postingThreadForPeer(completed->peerNicPath()) == thread_id)`
（`worker_pool.cpp:815`），也解释了 endpoint 生命周期锁为什么按 worker 而不是按 endpoint 分配。

---

## 目录结构速览

| 文件 | 一句话职责 |
|---|---|
| `include/transport/rdma_transport/rdma_transport.h` | `RdmaTransport` 声明；两个 `selectDevice` 静态重载 + `selectDeviceByLocalHca`；`context_list_`、`chunk_map_` |
| `src/transport/rdma_transport/rdma_transport.cpp` | install / 资源初始化 / 内存注册与分片 / 切片与本地选卡 / 握手被动端 / 选卡策略 |
| `include/transport/rdma_transport/rdma_context.h` | `RdmaContext`、`RdmaCq`、`MemoryRegionMeta`、`DmabufExport`、GID 快照、`clampRdAtomicDepth` |
| `src/transport/rdma_transport/rdma_context.cpp` | 单 NIC 全部资源：开设备、GID 选择/重探、CQ/comp channel、MR 注册、endpoint 索引、notify CQ |
| `include/transport/rdma_transport/rdma_endpoint.h` | `RdmaEndPoint`：七态 `Status`、两阶段销毁、`NotifyState`、`posted_fifo_`、`wr_depth_list_` |
| `src/transport/rdma_transport/rdma_endpoint.cpp` | QP 创建、主动/被动握手、RESET→INIT→RTR→RTS、`ibv_post_send`、native notify、销毁 |
| `include/transport/rdma_transport/worker_pool.h` | `WorkerPool`、`RailState`、三级熔断成员与阈值 |
| `src/transport/rdma_transport/worker_pool.cpp` | 运行时心脏：对端选卡、worker 归属、投递、CQ 轮询、错误恢复、事件恢复 |
| `include/transport/rdma_transport/endpoint_store.h` | `EndpointStore` 抽象 + `FIFOEndpointStore` + `SIEVEEndpointStore` |
| `src/transport/rdma_transport/endpoint_store.cpp` | 两种缓存的插入/淘汰/waiting list 异步回收 |
| `include/transport/rdma_transport/rdma_gid_probe.h` | 自动 GID 选择的纯函数库（七级分类、排序、重试决策） |
| `include/transport/rdma_transport/rdma_posted_fifo.h` | 非全信号投递的信号周期与前缀退休算法 |
| `include/transport/rdma_transport/rdma_batch_cache.h` | 批量提交时首次选卡结果的函数内（function-local）缓存 |
| `include/transport/rdma_transport/connect_pause_tracker.h` | 按 peer server name 记主动重连暂停截止时间 |
| `include/transport/rdma_transport/data_direct_topology.h` | ConnectX-8 Data Direct 的 PCI root 配对判定 |
| `src/transport/rdma_transport/data_direct_topology.cpp` | sysfs PCI root 解析 + 运行时 `dlvsym` libmlx5 |
| `src/transport/rdma_transport/CMakeLists.txt` | OBJECT library 构建，`USE_MLX5DV`/`USE_CUDA` 开关来源 |
| `include/transport/rdma_twosided/rdma_twosided_transport.h` | `RdmaTwoSidedTransport`（继承 `RdmaTransport`）与 `ConnectScope` |
| `include/transport/rdma_twosided/ctrl_channel.h` | 每 peer 一条专用 RC QP 的双边控制通道 |
| `src/transport/rdma_twosided/ctrl_plane.cpp` | `ensureCtrlChannel` 连接复用/抢占/超时、帧分发、轮询线程 |
| `include/transport/rdma_twosided/ctrl_frame.h` | 44 字节头线格式、10 种帧类型、版本严格校验 |
| `include/transport/rdma_twosided/sender_credit.h` | 信用账本（当前未接线，见下文） |

强耦合但不在目录内的文件：`include/topology.h`（`preferred_hca`/`avail_hca` 两级网卡表）、
`include/config.h`（全部 `MC_*` 默认值）、`include/transfer_metadata.h`（`SegmentDesc`/`HandShakeDesc`）、
`include/transport/transport.h`（`Slice`/`TransferTask` 完成协议）。

---

## 核心数据结构

### `Transport::Slice` 与 `TransferTask`：完成计数协议

`Slice` 是数据面最小单位，由 thread-local 的 `ThreadLocalSliceCache` 分配（底层是堆对象，
缓存容量 4096；复用出来的 slice 带 `from_cache=true`，`transport.h:405-450`），
**`markSuccess()`/`markFailed()` 恰好调用一次**；
它内含一个联合体，RDMA 用的是 `rdma` 分支：
`dest_addr`、`source_lkey`、`dest_rkey`、`qp_depth`（指向 endpoint 的每 QP 深度原子量）、
`retry_cnt`/`max_retry_cnt`、`endpoint`、`qp_index`
（`include/transport/transport.h:203-215`）。

完成状态有两条配合使用的通道：

1. **状态查询**：`getTransferStatus` 只看
   `success_slice_count + failed_slice_count == task.slice_count`
   （`rdma_transport.cpp:1192`、`1222`）。
2. **任务封口/批次完成回调**：`markSuccess()/markFailed()` 除了累加各自的计数与
   `transferred_bytes`，还对 `completion_state` 做一次原子加；`finishIfComplete` 只在
   **封口位已置位且低 63 位计数等于 `slice_count`** 时才发布完成
   （`transport.h:265-335`）。封口位由 `MultiTransport` 在 `submitTransferTask()` 返回后
   调用 `sealTaskSubmission()` 置上（`multi_transport.cpp:302-319`）。

因此 RDMA 的提交路径必须保证**所有 slice（以及最终 `slice_count`）在
`submitTransferTask()` 返回前发布完毕**——这正是 `supportsBatchCompletionCounter() == true`
的语义（`transport.h:550-571`）。注意 `slice_count` 在函数内部是边切边增长的，并不是
“投递前就固定”。

### `RdmaTransport`

关键成员（`rdma_transport.h:161-177`；`local_server_name_` 继承自 `Transport`，
见 `transport.h:598`）：

- `std::vector<std::shared_ptr<RdmaContext>> context_list_`：下标即 `device_id`，
  与 `Topology::getHcaList()` 以及 `BufferDesc::lkey/rkey` 向量**严格位置对齐**。
- `rdma_server_name_` vs `local_server_name_`：设了 `MC_RDMA_BIND_ADDRESS` 时二者分叉，
  前者用于构造 NIC path（`RdmaContext::nicPath()`，`rdma_context.cpp:1088-1090`），
  后者用于 TCP/P2P 路由与段名（`rdma_transport.cpp:219-229`）。
- `chunk_map_`：`基址 → 各 chunk 起始地址`，只为 `unregisterLocalMemory(base_addr)` 服务
  （因为大 buffer 被拆成多个 MR/BufferDesc，#2017）。

### `RdmaContext`

一个实例 = 一张本地 RNIC 的全部资源。核心字段（`rdma_context.h:344-407`）：

- `context_`/`pd_`：uverbs 设备上下文与保护域。**占位 Context 的 `pd_ == nullptr`**。
- `comp_channel_[]` + `event_fd_`：`epoll` 聚合 `context_->async_fd` 与各 comp channel fd
  （`rdma_context.cpp:298-318`），monitor 线程在 `event_fd_` 上等待。
- `cq_list_`（`std::vector<RdmaCq>`）：`RdmaCq` 只有 `ibv_cq *native` 和
  `std::atomic<int> outstanding`（`rdma_context.h:83-96`）。
  这里有一个很隐蔽的技巧：创建 CQ 时把 `&cq_list_[i].outstanding` 直接当作
  `ibv_create_cq` 的 `cq_context` 传进去（`rdma_context.cpp:320-333`），
  之后 `RdmaEndPoint::construct` 再 `static_cast<std::atomic<int>*>(cq->cq_context)`
  取回来存成裸指针 `cq_outstanding_`（`rdma_endpoint.cpp:137-139`）。
  于是 endpoint 持有的是 **`cq_list_` 元素内部对象的地址**。`RdmaCq` 因此显式删除拷贝赋值/
  移动赋值、只保留移动构造（`rdma_context.h:84-92`），而 `cq_list_` 的 `resize` 必须发生在
  任何 endpoint 创建之前——这是个隐含生命周期前提。
- `memory_region_map_`：`std::map<uintptr_t, MemoryRegionMeta>`，按起始地址排序，
  查找用 `upper_bound` 再 `prev`（`rdma_context.cpp:963-972`）；`MemoryRegionMeta` 自己记录
  `addr/length`，因为 iova-based MR 的 `mr->addr` 不一定是起始地址（`rdma_context.h:98-104`）。
- `endpoint_store_`：FIFO 或 SIEVE（`config.endpoint_store_type`，默认 SIEVE）。
- `worker_pool_`：构造末尾 `new WorkerPool(*this, socketId())`，`socketId()` 读
  `/sys/class/infiniband/<dev>/device/numa_node` 做 NUMA 绑定（`rdma_context.cpp:345`、`404-416`）。
- `active_`：context 级健康位，熔断与异步事件都会清它。

### `RdmaEndPoint`

代表“本地 NIC ↔ 对端 NIC”一条 rail，内部是 `num_qp_per_ep`（默认 2）个 RC QP。

`Status` 七态（`rdma_endpoint.h:52-60`）：
`INITIALIZING → UNCONNECTED → CONNECTING → CONNECTED_WAIT_READY_ACK → CONNECTED`，
以及 `DESTROYING → DESTROYED`。注意 `connected()` 把 `CONNECTED_WAIT_READY_ACK` 也算连接中
（`rdma_endpoint.h:117`、`215-217`），但 **只有 `CONNECTED` 才能 `submitPostSend`**
（`readyToSend()`，`rdma_endpoint.h:121`）。

关键字段：

- `qp_list_`、`qp_generation_`：握手等待者用它判断“我等的过程中 QP 是否被重建”。
- `wr_depth_list_`：每 QP 一个 `std::atomic<int>`，记录已投递未退休的 WR 数；
  `finishDestroy` 等它全零。
- `posted_fifo_`：每 QP 一个 `std::deque<Slice *>`，用于非全信号模式下“一个 signaled CQE
  退休它之前所有 unsignaled WR”（`rdma_endpoint.h:291-293`）。
- `cq_`、`cq_outstanding_`：见上。
- `notify_`（`NotifyState`）：native notify 的专用 QP、收发 buffer/MR、发送槽游标、
  条件变量。**它有自己的 mutex**，因为 `sendNotification` 可能阻塞等槽位，不能占着 endpoint 的
  `lock_`（`rdma_endpoint.h:259-278`）。
- `has_connected_`：一旦为 true，后续握手失败不再“重置复用”，而是退休重建（见流程 3）。

### `WorkerPool`

- `collective_slice_queue_`：`vector<unordered_map<peer_nic_path, SliceList>>`，按 worker 下标。
- `worker_slice_queue_` + 每 owner 一把锁：跨 worker 投递的入口（`worker_pool.h:149-153`）。
- `posted_slices_` + `posted_slices_mutex_`：可选诊断注册表（`track_rdma_posted_slices`）。
- `last_poll_ts_ns_` / `last_poll_interval_ns_` / `max_poll_interval_ns_`：monitor 用来区分
  “传输超时”与“轮询线程卡住”（`worker_pool.h:137-139`）。
- `RailState`：`peer_nic_path → {error_count, pause_until_ns, last_error_ns}`
  （`worker_pool.h:83-87`），配 `paused_rail_count_` 让热路径在无暂停时零锁。
- `context_failure_count_` / `breaker_reactivate_after_ns_` / `recovery_activate_after_ns_`：
  context 级熔断与物理事件恢复，状态迁移统一由 `context_state_lock_` 串行化
  （`recovery_activate_after_ns_` 本身是原子量，读取可以不加锁）。

### `EndpointStore`

抽象接口定义在 `endpoint_store.h:37-74`。两个实现都维护：

- `endpoint_map_`（活跃池，`peer_nic_path → endpoint`）；
- `fifo_list_` + `fifo_map_`（淘汰顺序）；
- `waiting_list_`（已 retire 但尚未析构）；
- `waiting_list_len_`（原子计数，**为 0 时 `reclaimEndpoint` 直接返回，不拿锁**）。

`FIFOEndpointStore` 就是先进先出。`SIEVEEndpointStore` 实现了 NSDI'24 SIEVE：
每个 entry 带一个 `atomic_bool visited`，`getEndpoint` 命中时置 true（读锁下幂等，安全）；
淘汰时 hand 从尾部向前扫，遇到 visited=true 就清位继续，遇到 false 就淘汰
（`endpoint_store.cpp:309-335`）。`insertEndpoint` 前会 `while (getSize() >= max_size_) evictEndpoint()`
（`endpoint_store.cpp:72`、`250`）。

### GID 选择相关

`AutoGidCandidateClass` 七级（`rdma_gid_probe.h:29-44`），从好到差：

1. `kNetworkRoutable`
2. `kNoNetworkRoutable`
3. `kNetworkPrivateV4`（10/8 等私网 IPv4-mapped）
4. `kNetworkDegraded`（overlay 网卡名或 fe80:: link-local）
5. `kNoNetworkPrivateV4`
6. `kNoNetworkDegraded`
7. `kFallbackNonzero`

`classifyAutoGidCandidate` 只接受 `IBV_GID_TYPE_ROCE_V2` 与 `IBV_GID_TYPE_IB`，
并显式把“私网 IPv4”从“降级”里拆出来（`rdma_gid_probe.h:98-131`，注释引用 #2729）。
`rankAutoGidCandidates` 用 `stable_sort`，同级按 `gid_index` 升序，最后再追加一个
“第一个 query 成功但未分类”的兜底候选（`rdma_gid_probe.h:154-195`）。

### `rdma_twosided` 的结构

- `CtrlChannel`：每 peer 一条独立 RC QP + 独立 CQ + 一个 send buffer + `recv_count_`
  个 recv buffer。它有自己的 `resource_mutex_` 与 `send_mutex_`，嵌套顺序固定为
  `send_mutex_ → resource_mutex_`（`ctrl_channel.h:126-136`）。
- `CtrlFrame`：44 字节头 `magic:u32 | ver:u8 | type:u8 | flags:u16 | session:u64 |
  epoch:u64 | seq:u64 | ack_seq:u64 | payload_len:u32`（`ctrl_frame.h:26-33`）。
  版本严格等于 1，未知版本拒绝；`flags` 只允许 `NeedsAck` 位（`ctrl_frame.h:30-55`）。
- `SenderCreditLedger`：按 `(peer, session)` 记账的累计信用账本，带 epoch 防陈旧写入
  （`sender_credit.h:34-97`）。**目前生产路径没有接它**，只有
  `sender_credit_test.cpp` 在用；`ctrl_plane.cpp:219-231` 对除 `NOTIFY_COMPAT` 外的所有
  帧类型都走 `VLOG(1) << "... (PR2 notify-only)"`。

---

## 关键流程剖析

### 流程 1：注册内存——rkey/lkey 从哪里来

调用链：

```
TransferEngine::registerLocalMemory
  → RdmaTransport::registerLocalMemory
    → RdmaTransport::registerLocalMemoryInternal
      → RdmaContext::exportDmabuf                (每 buffer 一次)
      → RdmaContext::registerMemoryRegion        (每 chunk × 每 context)
        → RdmaContext::registerMemoryRegionInternal
          → ibv_reg_mr / ibv_reg_dmabuf_mr / mlx5dv Data Direct
      → metadata_->addLocalMemoryBuffer          (每 chunk 一次, update=false)
      → metadata_->updateLocalSegmentDesc        (全部 chunk 完成后一次)
```

分步说明：

1. **拼访问权限**：`IBV_ACCESS_LOCAL_WRITE`；`remote_accessible` 时再加
   `REMOTE_WRITE|REMOTE_READ`；若运行时探测到 `ibv_reg_mr_iova2` 符号存在，再加
   `IBV_ACCESS_RELAXED_ORDERING`（`rdma_transport.cpp:344-349`、`165-190`）。
2. **单 fd 导出 dma_buf**：`exportDmabuf` 只对整块 buffer 调一次
   （`rdma_transport.cpp:360-371`）。`DmabufExport` 的注释解释了原因：每个 NIC 各导一次会让
   GPU 驱动为同一 buffer 预留 N 个 BAR1 窗口，共享单 fd 只占一个
   （`rdma_context.h:106-121`）。代价是硬生命周期契约：**fd 必须在所有 NIC 的注册都返回之后
   才能关**，因此用 `DmabufCloser` RAII 保证任意返回路径恰好关一次
   （`rdma_transport.cpp:368-371`；`rdma_context.cpp:749-756`）。主机内存走 `kHostReg`，
   不开 fd；`with_nvidia_peermem` 且非 Data Direct 的显存也走普通 `ibv_reg_mr`
   （`rdma_context.cpp:533-541`）。
3. **按 `max_mr_size` 切 chunk（#2017）**：`chunk_limit = globalConfig().max_mr_size`
   （默认 1 TiB，`config.h:48`）。注释写明原因：`ibv_reg_mr` 会**静默截断**到设备
   `max_mr_size`，但元数据仍广播完整长度，越界访问就会得到 `IBV_WC_REM_ACCESS_ERR`
   （`rdma_transport.cpp:373-379`）。HugeTLB 页（`detectBufferPageSize > sysconf(_SC_PAGESIZE)`）
   时把 `chunk_limit` 向下对齐到页大小，否则 `ibv_fork_init` 的 `MADV_DONTFORK` 会在非 hugepage
   边界失败（`rdma_transport.cpp:380-424`）。Data Direct 额外要求 IOVA 页对齐，所以每个 chunk
   预留 `prefix = addr % page` 的空间（`rdma_transport.cpp:425-451`）。
4. **逐 chunk 逐 context 注册**：可用 `MC_ENABLE_PARALLEL_REG_MR` 控制，默认
   `parallel_reg_mr == -1` 表示“多 context 且做过 pre-touch”才并行
   （`rdma_transport.cpp:533-592`、`config.h:119`）。
5. **收集每 chunk 的 key**：对每个 context 取 `lkey(chunk_addr)`，远端可访问时再取
   `rkey(chunk_addr)`，**向量顺序就是 `context_list_` 顺序**；然后构造一个 `BufferDesc`
   （`rdma_transport.cpp:617-628`）。
6. **元数据只发布一次**：每个 chunk 以 `update_metadata=false` 加入本地段，所有 chunk 完成后
   统一 `updateLocalSegmentDesc`（`rdma_transport.cpp:629-651`）。
7. **记录 chunk 起始地址**：`chunks.size() > 1` 时写 `chunk_map_[base]`
   （`rdma_transport.cpp:653-661`）。
8. **反注册**：先查 `chunk_map_` 并 `erase`；然后**即使某个 chunk 失败也要把所有 chunk 都走一遍**
   （注释明确说提前 return 会泄漏剩余 chunk 的 MR 与元数据），只记第一个错误最后返回；
   MR 释放按 context 并行（`rdma_transport.cpp:669-744`）。

`RdmaContext` 侧的三个细节：

- `registerMemoryRegionInternal` 发现 `length > max_mr_size` 直接报错而不是继续注册
  （`rdma_context.cpp:762-773`），这是防止上层绕过切片的最后一道闸。
- Data Direct 配对失败时是**静默降级**：写入 `mrMeta.mr = nullptr` 并 `return 0`
  （`rdma_context.cpp:793-798`）。于是该 NIC 的 `lkey()/rkey()` 读作 0
  （`rdma_context.cpp:940-961`）。设计前提是 discovery 阶段已经 fail-closed 拒绝了配对不成立的
  拓扑，所以传输不会路由到这张卡。
- `unregisterMemoryRegion` 用 `mr->length` 反推带前缀的真实起始地址，并在 `ibv_dereg_mr`
  之后对同一区间 `madvise(MADV_DOFORK)`，撤销注册时施加的 `MADV_DONTFORK`（#3639，
  `rdma_context.cpp:890-918`）。

### 流程 2：一次 BatchTransfer 的切片与本地选卡

入口是 `RdmaTransport::submitTransfer`（`rdma_transport.cpp:914-940`），
它把 `TransferRequest` 填成 `TransferTask` 后立即调用 `submitTransferTask`
（`rdma_transport.cpp:942-1175`）。

关键常量：

```
kBlockSize       = globalConfig().slice_size        // 65536
kFragmentSize    = globalConfig().fragment_limit    // 16384
kMaxRetryCount   = globalConfig().retry_cnt         // 9
kSubmitWatermark = max_wr * num_qp_per_ep           // 256 * 2 = 512
```

（`rdma_transport.cpp:958-962`；`config.h:51-69`、`113`。）

#### 2.1 一个 request 切成几个 slice

`SliceLengthCalculator::calculate`（`rdma_transport.cpp:121-150`）取三者最小值：

```
slice_length = min(
    (remaining <= block_size + fragment_size) ? remaining : block_size,
    bytesUntilBufferEnd(local_desc,  source + offset, require_remote_key=false),
    bytesUntilBufferEnd(target_desc, target_offset + offset, require_remote_key=true)
)
```

- 第一项是常规切片，但**当剩余量不超过 `block_size + fragment_limit` 时整块一次发完**，
  避免产生一个只有几 KB 的尾 slice。
- 后两项保证 **一个 WR 绝不跨 MR**：一个 `ibv_send_wr` 只有一个 sge、一个 rkey，
  跨 MR 的 WR 在语义上无法表达。
- `bytesUntilBufferEnd` 如果描述符有唯一覆盖索引（`buffer_range_index.size() == buffers.size()`
  且 `!overlaps()`）就走 `findCovering` 的 O(log n) 快路径；否则退化成线性扫描并取
  **剩余量最大的那个 buffer**（`rdma_transport.cpp:65-118`）。`last_buffer_idx` 是跨 slice、
  跨同一批请求的“上次命中”缓存。

#### 2.2 本地 NIC 选择（每 request 一次）

```cpp
request_hint = request.advise_retry_cnt == 0
             ? nic_hint_name(request.nic_hint) : {}
local_hint_device_id = (hint 为空 && last_local_device_buffer_id == last_local_buffer_id)
                     ? last_local_device_id : -1
local_device_cache.select(local_segment_desc, source, length, ...,
    [&]{ return selectDevice(local_segment_desc.get(), source, length,
                             request_hint, request_buffer_id, request_device_id,
                             0, last_local_buffer_id, local_hint_device_id); });
```

（`rdma_transport.cpp:1028-1055`。）含义：

- `request.nic_hint` 是上层给的建议卡；`advise_retry_cnt > 0` 说明这个 request 已经在
  重试，**重试时必须丢掉 hint 重新跑拓扑选择**。
- `local_hint_device_id` 是“同一个 MR 内连续请求复用上一张卡”的亲和提示。
- `BatchRdmaDeviceCache`（`rdma_batch_cache.h:28-67`）只在以下条件满足时缓存：
  同一个 `SegmentDesc`、缓存覆盖当前地址区间、选中 buffer 合法、
  **对端 HCA 列表大小为 1**、`buffer_range_index.size() == buffers.size()` 且不重叠、
  buffer 名不以 `segments:` 开头。它缓存的是 `buffer_id/device_id`，命中时**不再考虑本 request 的
  hint**，因此其前提是同一批请求的选择策略一致（文件顶部注释写的是
  “caller keeps the selection policy fixed”）。

#### 2.3 选卡策略本体

`selectDeviceImpl` + `pickTopologyDevice`（`rdma_transport.cpp:1402-1490`）：

```cpp
if (desc->buffer_range_index 唯一覆盖) {
    if (try_buffer(hint_buffer_id, /*allow_device_reuse=*/true)) return 0;
    int found = index->findCovering(offset, length);
    if (found != hint_buffer_id && try_buffer(found, false)) return 0;
    return ERR_ADDRESS_NOT_REGISTERED;
}
for (int candidate = 0; candidate < buffers.size(); ++candidate)
    if (try_buffer(candidate, candidate == hint_buffer_id)) return 0;
```

`try_buffer` 先检查 `isSelectableRdmaBuffer`（多协议构建下 `protocol` 必须为空或 `"rdma"`，
否则 hip 等设备 buffer 会因没有 lkey/rkey 被误选，`rdma_transport.cpp:1389-1400`）
和 `bufferCoversRange`。然后是“复用上一张卡”的短路：

```cpp
if (allow_device_reuse && retry_count == 0 && hint_device_id >= 0 &&
    buffer.name.rfind(kSegmentsLocationPrefix, 0) != 0) { ... selected = hint_device_id; }
```

注意条件里的 `retry_count == 0`：**一旦是故障切换（retry_count > 0），必须重新跑
`selectDevice`**，不能复用 hint 卡。否则“换路”会原地打转。

`pickTopologyDevice` 有两条路：

- `by_local_hca`（`selectDeviceByLocalHca`，供 peer 选卡时用）：按 buffer location 与
  `local_hca` 找配对网卡，失败回落 `kWildcardLocation`。
- 普通路径：先解析 location（支持 `segments:` 前缀的区间映射），
  **若显式给了 hint**，先试 `getDeviceIndex(location, hint)` 再试
  `getDeviceIndex(kWildcardLocation, hint)`；都不行才走 `topology.selectDevice(location, retry)`
  与 wildcard 回落。注释特别说明：这里**不能调用带 hint 的 `selectDevice` 重载**，
  因为那个重载在 location miss 时会立刻做普通选择，使 wildcard+hint 这一次尝试永远不可达
  （`rdma_transport.cpp:1422-1435`）。

#### 2.4 逐 slice 提交与多网卡聚合

对每个 slice（`rdma_transport.cpp:1064-1169`）：

1. 分配 `Slice`，填 `source_addr`、`length`、`opcode`、`rdma.dest_addr`、
   `retry_cnt = advise_retry_cnt`、`max_retry_cnt = kMaxRetryCount`、`task`、`target_id`。
2. 若 request 级选卡结果有效且 context 仍 `active()`，直接复用；否则进入
   `while (retry_cnt < kMaxRetryCount && !found_device)` 循环，逐次 `selectDevice(..., retry_cnt++)`。
   首轮仍允许 hint，后续丢弃 hint 以便换卡。
3. 命中后写 `slice->rdma.source_lkey = local_segment_desc->buffers[buffer_id].lkey[device_id]`
   （`rdma_transport.cpp:1149-1150`）。
4. `slices_to_post[context].push_back(slice)`——**按本地 Context 分组**，这就是“多网卡聚合”
   的第一处体现：一条大请求的不同 slice 可以落到不同本地 NIC。
5. `task.total_bytes += length`，`__sync_fetch_and_add(&task.slice_count, 1)`。
6. 每当本轮新分配 slice 数达到 `kSubmitWatermark`，就把当前分组全部提交并清空
   （`rdma_transport.cpp:1161-1166`）。注意计数器只统计“新分配”：从 thread-local
   `SliceCache` 复用出来的 slice 带 `from_cache=true`，不会让它自增
   （`rdma_transport.cpp:1069-1071`）。

#### 2.5 失败清理为什么要伪造零长 slice

`fail_unposted_slices` 把已排队但还没提交的 slice 全部 `markFailed`；
`fail_unstarted_tasks` 为后续每个尚未开始的 task **新建一个 `source_addr=nullptr, length=0`
的 slice**，挂到 task 上、`slice_count++`，再 `markFailed`
（`rdma_transport.cpp:963-994`）。

原因：`getTransferStatus` 的终止条件是 `success + failed == slice_count`，而从未开始的
task 的 `slice_count` 仍是 0——**0 == 0 会让它被报成 `COMPLETED`（静默成功）**，
这比挂起更糟。伪造一个零长失败 slice 把 `failed_slice_count` 抬到 1，才能让这个 task
落到 `FAILED`；封口后的 `finishIfComplete` 依赖同一个计数，所以两条通道会被一起修正
（`transport.h:315-335`）。`~TransferTask` 会回收这个 slice（`transport.h:492-495`）。
这是“用数据协议消化控制流”的典型写法。

#### 2.6 提交后的第二阶段选卡

`RdmaContext::submitPostSend` 只是转发到 `worker_pool_->submitPostSend`
（`rdma_context.cpp:1829-1832`）。真正的工作在 `WorkerPool::submitPostSend`
（`worker_pool.cpp:296-451`）：

1. 解析每个 `target_id` 的 `SegmentDesc`。
2. `failed_target_ids` 是 **thread_local 的 100 ms 黑名单**：同一目标刚失败过就直接
   `markFailed`，避免每个 slice 都去撞同一个坏目标（`worker_pool.cpp:342-350`）。
3. `peer_device_cache.select(...)` 选对端 `buffer_id/device_id`；失败则**强制 reload 元数据**
   （`getSegmentDescByID(id, true)`）再试一次，仍失败就登记黑名单、`dumpMetadataContent`
   （`worker_pool.cpp:358-388`）。
4. 写 `slice->rdma.dest_rkey = buffers[buffer_id].rkey[device_id]`。
5. 若选中的 rail 被暂停，**遍历该 peer 的其他 devices 找可用 rail**，找到就改写
   `dest_rkey` 并让缓存失效（`worker_pool.cpp:400-427`）。这是“多网卡聚合”的第二处体现。
6. `slice->interned_peer_nic_path` 指向 `SegmentDesc` 进程级 intern 池里的字符串，
   避免每 slice 一份 `std::string`（`worker_pool.cpp:429`；`transport.h:176-183`）。
7. `enqueuePreparedSlices` 按 `postingThreadForPeer(peerNicPath())` 分桶，投到 owner worker
   的队列；若有 worker 在 park，唤醒它（`worker_pool.cpp:453-478`）。

`selectPeerDevice`（`worker_pool.cpp:180-225`）决定用哪套策略：

- `enable_hca_peer_affinity`：`selectDeviceByLocalHca(..., local_hca, ...)`；
- 否则若 `enable_dest_device_affinity`：把 `local_hca` 当 hint 传给普通 `selectDevice`；
- 两者都关：无 hint 普通选择。

最后还会做两次边界检查：`device_id < buffers[buffer_id].rkey.size()` 与
`device_id < devices.size()`，否则返回 `ERR_ADDRESS_NOT_REGISTERED`
（注释提到 `decodeSegmentDesc` 现在也会拒绝键数与设备数不一致的描述符，
这里的检查是对实际使用值的本地兜底）。

### 流程 3：QP 建连——主动、被动与 ready ACK

一条 rail 的建连由“首次有 slice 要发”触发：`performPostSend` 发现 endpoint 不
`connected()` 就调用 `setupEndpointByActiveOutsideLifecycleGate`（见流程 4），
后者把 `setupConnectionsByActive()` 甩到 8 线程 executor。

#### 3.1 主动侧状态机（`rdma_endpoint.cpp:653-1004`）

```
lock_
 ├─ readyToSend() → 直接返回 0
 ├─ 本端 nicPath == peer_nic_path → loopback：doSetupConnection(自己) 直接 CONNECTED
 ├─ status == UNCONNECTED → 自己置 CONNECTING，do_rpc = true（唯一赢家）
 └─ 否则 do_rpc = false
```

- **输家自旋 + 退避**：前 500 次 `PAUSE()`，之后睡眠 50us 起、每次翻倍、上限 2000us；
  总超时 10 秒（`kWaitExistingHandshakeTimeoutNano`、`kWaitExistingHandshakeSpinCount/
  InitialSleepUs/MaxSleepUs`，`rdma_endpoint.h:234-240`）。超时后拿写锁，
  先比较 `qp_generation_`：若 QP 已被别人重建，返回 `connected()?0:ERR_ENDPOINT`；
  否则 `resetConnection("wait existing handshake timeout")`。
- **RPC 在锁外执行**：赢家先复制出 `local_qp_num`/`local_gid_selection` 填 `HandShakeDesc`
  （`fillLocalHandshakeDesc`，`rdma_endpoint.cpp:46-60`），然后释放锁调用
  `sendHandshake`（`rdma_endpoint.cpp:761-762`）。注释说明这是为了支持
  **simultaneous open**：对端可能同时也在主动连我们，被动路径会先把它连上。
- **回来先验代际**：重新拿写锁后若 `qp_generation_` 变了，丢弃陈旧回复。
- **RPC 失败但已连接**：若当前 QP 号与发出时一致，说明同时开放已经成功，直接复用
  （`rdma_endpoint.cpp:806-821`）；否则 `resetConnection`。
- **RPC 成功**：若已处于 connected 且 peer QP 号与通知 QP 号都一致，且对端支持 ready ACK，
  就发 ready ACK；不支持则直接置 `CONNECTED`。
- **正常建链**：用 `peer_desc.local_gid/local_lid/qp_num` 调 `doSetupConnection`，
  目标状态由 `peer_desc.ready_ack_supported` 决定是 `CONNECTED_WAIT_READY_ACK` 还是
  `CONNECTED`。
- **auto GID 重试闭环**：只有 `stage == kRtr && errno == EINVAL` 且自动 GID 才重试
  （`shouldAttemptAutoGidHandshakeRetry`，`rdma_gid_probe.h:206-212`）。
  先 `reprobeAutoGid(expected_selection, tried...)`，再
  `decideAutoGidRetryAction` 判断“重探真的换了”还是“外部已经换了”，然后
  `resetConnection`、`++auto_gid_retry_count`、`retry_with_new_gid = true`
  （`rdma_endpoint.cpp:930-970`）。

#### 3.2 被动侧（`rdma_endpoint.cpp:1006-1183`）

入口是 `RdmaTransport::onSetupRdmaConnections`（`rdma_transport.cpp:1279-1336`）：
用 `peer_desc.peer_nic_path` 的 NIC 名找到本地 `context_list_[index]`，
再以 `peer_desc.local_nic_path` 作为 `peer_nic_path` 取/建 endpoint，调用
`setupConnectionsByPassive`。

被动侧逻辑：

1. **ready_ack 帧**：必须是已 connected，且 peer QP 号与通知 QP 号都要匹配，否则拒绝；
   匹配则清 `ready_wait_start_ts_`、置 `CONNECTED`（`rdma_endpoint.cpp:1009-1034`）。
2. **已连接且 QP 号相同**：复用；若对端不支持 ready ACK，直接把状态从
   `CONNECTED_WAIT_READY_ACK` 推到 `CONNECTED`（`rdma_endpoint.cpp:1036-1054`）。
3. **QP 号不同**（典型是对端重启）：`resetConnection("re-establishing connection (passive)")`。
4. **路径一致性校验**：`peer_desc.peer_nic_path == context_.nicPath()` 且
   `peer_desc.local_nic_path == peer_nic_path_`，否则拒绝（`rdma_endpoint.cpp:1070-1079`）。
5. 置 `CONNECTING`，在 `attempt_setup_with_peer` 里循环：填本地描述、
   `doSetupConnection`、失败时同样走 auto GID 重试。
6. **本地竞态吸收**：`RdmaTransport::onSetupRdmaConnections` 在
   `setupConnectionsByPassive` 返回 `ERR_ENDPOINT` 且 endpoint 已 retired 时，
   认为这是本地 endpoint store 的竞态（对端 QP 号变化导致旧 endpoint 被退休），
   于是清空 `local_desc`、新建一个 endpoint 再试一次（`rdma_transport.cpp:1313-1334`）。
   日志上这看起来像“对端失败”，实际是本地重试。

#### 3.3 单 QP 的硬件配置（`rdma_endpoint.cpp:1519-1723`）

`doSetupConnection(qp_index, ...)` 是建连的硬件层真相：

```
ibv_query_qp(IBV_QP_STATE)
  ├─ 当前不是 RESET → ibv_modify_qp(RESET)
  │   原因：irdma(E810) 把 RESET 实现成先进 ERROR，直接 RESET→INIT 会 EINVAL
  ├─ RESET→INIT: port_num, pkey_index,
  │    access = LOCAL_WRITE|REMOTE_READ|REMOTE_WRITE|REMOTE_ATOMIC
  ├─ INIT→RTR: path_mtu = min(context.activeMTU(), config.mtu_length)
  │    grh.dgid = peer_gid, grh.sgid_index = local_gid_index,
  │    grh.hop_limit = 16, grh.traffic_class 可选,
  │    ah_attr.dlid = peer_lid, sl 可选, is_global = 1, port_num,
  │    dest_qp_num = peer_qp_num, rq_psn = 0,
  │    max_dest_rd_atomic = context.maxQpRdAtom(), min_rnr_timer = 12
  └─ RTR→RTS: timeout = 14, retry_cnt = 7, rnr_retry = 7, sq_psn = 0,
       max_rd_atomic = context.maxQpInitRdAtom()
```

常量 `kMaxHopLimit=16`、`kTimeout=14`、`kRetryCount=7` 在
`rdma_endpoint.cpp:42-44`。RD-atomic 深度由 `clampRdAtomicDepth` 从设备属性夹取：
上限 16、下限 1，避免设备报 0 时静默失败（`rdma_context.h:45-57`；
`rdma_context.cpp:1690-1703`）。每步失败都会填
`SetupConnectionFailureInfo.stage`（`kReset/kInit/kRtr/kRts/kPeerValidation`），
上层据此决定是否重探 GID。

两个可选尾部（都要求 `USE_MLX5DV`，失败只告警不致命）：

- `MC_MLX5_QP_LAG_PORT_BALANCE`：`num_lag_ports > 1` 时按 `qp_index % n + 1` 把 QP 绑到
  LAG 物理口（`rdma_endpoint.cpp:1662-1696`）。
- `MC_MLX5_QP_UDP_SPORTS`：按 `qp_index % sports.size()` 覆盖 RoCEv2 UDP 源端口，
  让不同 QP 走不同 ECMP 路径（`rdma_endpoint.cpp:1698-1720`）。

#### 3.4 握手 RPC 的传输与元数据

`sendHandshake` 默认走 `metadata_->sendHandshake(...)`（`rdma_transport.h:127-132`），
服务端由 `startHandshakeDaemon` 启动，回调就是 `onSetupRdmaConnections`
（`rdma_transport.cpp:1377-1382`）。建连前双方已经通过 `allocateLocalSegmentID` 把
`devices[{name,lid,gid}]` 与 `topology` 发布到元数据（`rdma_transport.cpp:794-821`）；
`buffers[{addr,length,lkey[],rkey[]}]` 则来自之后的内存注册：`allocateLocalSegmentID`
先复制旧 `SegmentDesc`，`registerLocalMemoryInternal` 再按 chunk 追加并统一重发布。
因此主动侧在 `peer_desc.local_gid` 为空时，可以从 `SegmentDesc` 按 NIC 名反查 GID/LID
（`rdma_endpoint.cpp:902-917`）。

### 流程 4：投递、CQ 轮询与故障恢复

#### 4.1 worker 主循环

`transferWorker`（`worker_pool.cpp:1212-1249`）：

```cpp
while (running) {
    if (processed == submitted && !hasOutstandingCq(thread_id)) {
        // 忙等：空闲时间未满 MC_RDMA_WORKER_IDLE_SPIN_US（默认 100ms）时直接 continue；
        // 超时后 parked_worker_count_++，在 cond_var_ 上最多等 1 秒，然后也 continue
        continue;  // 空闲时不会执行下面的 post/poll
    }
    performPostSend(thread_id);
    performPollCq(thread_id);
}
```

注意 park 条件的双重检查：`parked_worker_count_` 在检查前递增，生产者提交后只要看到它
大于 0 就会 `notify_all`，避免丢失唤醒（`worker_pool.cpp:1228-1240`）。

#### 4.2 `performPostSend`：投递前半段

（`worker_pool.cpp:580-775`。）

1. 把 `worker_slice_queue_[thread_id]` 合并进 `collective_slice_queue_[thread_id]` 并清空，
   合并时持 owner 锁。
2. **context 不活跃**：整队克隆后清空，`redispatch(..., handoff_to_local_worker=true)`
   ——本地卡坏了，换对端 rail 没意义。
3. **`redispatch_counter_` 触发的批量重派**：每个 worker 用 thread_local 记住已见计数，
   发现全局计数增加就整队重派。
4. 对每个 `peer_nic_path`：
   - 先查 `isRailAvailable(path)` 与 `isConnectPaused(path)`，任一为真就进
     `failed_slice_list`。注释解释了为什么把 pause 检查放在这里：
     pause 是策略决定，不该伪装成一次新的连接失败而刷新 TTL
     （`worker_pool.cpp:638-649`）。
   - `lockEndpointLifecycle(path)`，然后 `context_.endpoint(path, cqIndexForPostingThread(thread_id))`
     ——取或建 endpoint。
   - **未连接**：调用 `setupEndpointByActiveOutsideLifecycleGate`。该函数先释放生命周期锁，
     把握手提交到 8 线程 executor，等待期间**继续 drain 本 CQ**（`drain_cq_while_waiting`），
     等待期从 1ms 指数退避到 25ms。结束后重新拿锁，并确认
     “endpoint 仍是 store 里那个、仍 active、未 retired”（`worker_pool.cpp:128-178`）。
     这段并发设计的目的：握手可能阻塞在 TCP connect/read 超时上，不能让 owner worker 停止
     轮询 CQ，否则已完成的数据也无法退休。
   - 握手失败时的分类（`worker_pool.cpp:686-730`）：
     - 有可用对端 rail → `markRailFailed(path, immediate=true)`、`redispatch_counter_++`；
     - 否则本地 context 已 inactive → `set_active(false)`、刷新拓扑、`redispatch_counter_++`；
     - 两者都不是 → 只记日志，不暂停 rail；
     最后 `deleteEndpointByPtr`，能本地换 worker 的就 `tryHandoffToAnotherLocalWorker`，
     其余进 failed 列表。
   - **等待 ready ACK 超时**：`readyAckTimedOut()` 为真则暂停 rail、删 endpoint、
     整队失败（`worker_pool.cpp:732-745`）。
   - 为每个 slice 写 `slice->rdma.endpoint = endpoint.get()`，调用
     `endpoint->submitPostSend(entry.second, failed_slice_list)`。
   - 若 endpoint 在投递后变为不活跃，立即 `deleteEndpointByPtr`。
5. 处理 `failed_slice_list`：`shouldRetrySlice` 会先 `retry_cnt++` 再判断
   `< max_retry_cnt`；context 不活跃的走 `local_retry_list`（换本地卡），否则走
   `retry_list`（换对端 rail）；不重试的 `markFailed`。

#### 4.3 `RdmaEndPoint::submitPostSend`：唯一真正敲门的地方

（`rdma_endpoint.cpp:1295-1414`。）

- 前置检查：`active_` 且 `status == CONNECTED`，否则整批进 failed 列表。
- `cq_remaining = max_cqe - cq_outstanding_`，`<= 0` 直接返回 0，
  **slice 留在队列里等下一轮**——这是背压。
- 信号周期：`use_fifo = (rdma_signal_interval > 1)`；
  `period = rdmaSignalPeriod(interval, max_wr_depth_)`；
  `shouldSignalRdmaWr(i, wr_count, period)` 保证“每 period 个信号一次，且本链最后一个必信号”
  （`rdma_posted_fifo.h:26-39`）。
- QP 间均摊：`chunk = ceil(remaining_slices / remaining_qps)`，
  `wr_count = min(assigned_count, qp_avail, cq_remaining)`。
- 填 WR：`wr_id = (uint64_t)slice`，`opcode` 由 `TransferRequest::READ` 映射为
  `IBV_WR_RDMA_READ`，否则 `IBV_WR_RDMA_WRITE`；`num_sge=1`；
  `remote_addr = slice->rdma.dest_addr`、`rkey = slice->rdma.dest_rkey`；
  `slice->rdma.qp_depth = &wr_depth_list_[qp_index]`、`qp_index` 一并记录。
- **先预扣额度再敲门**：`wr_depth += wr_count`、`cq_outstanding += wr_count`
  （`rdma_endpoint.cpp:1372-1373`）。
- **`trackPostedSlices` 必须在 `ibv_post_send` 之前**：注释说明快速完成可能在上报诊断注册表
  之前就被轮询到（`rdma_endpoint.cpp:1374-1376`）。
- `ibv_post_send` 失败时：`posted = bad_wr - wr_list.data()`，
  `untrackPostedSlices(start+posted, wr_count-posted)`，
  沿 `bad_wr->next` 收集失败 slice，并把 `failed_count` 从 `wr_depth`/`cq_outstanding` 扣回
  （`rdma_endpoint.cpp:1379-1394`）。
- 非全信号模式仍要按**每个** WR 预扣 CQ：注释明确说失败/被 flush 的 WR 即使没有
  `IBV_SEND_SIGNALED` 也会产生 CQE（`rdma_endpoint.cpp:1316-1319`）。
  非全信号省下的是成功路径上的 CQE 数量，不是 CQ 深度。
- `use_fifo && posted > 0` 时把本批已投递 slice 追加到 `posted_fifo_[qp_index]`；
  一旦 `rc != 0` 立刻 `beginDestroyLocked()`（`rdma_endpoint.cpp:1395-1401`）。
- 最后把已投递前缀从 `slice_list` 里 erase；未投递的尾部保留，等下一轮。

#### 4.4 `performPollCq` 与 `processCompletions`

`performPollCq`（`worker_pool.cpp:777-852`）：

1. 更新 `last_poll_ts_ns_` 与轮询间隔统计（供 monitor 诊断）。
2. `kPollCount = 64`，`cq_index = cqIndexForPostingThread(thread_id)`，`context_.poll(...)`。
3. 对每个 CQE：
   - `completed = (Slice *)wc.wr_id`；
   - `assert(postingThreadForPeer(completed->peerNicPath()) == thread_id)`；
   - 若 `use_fifo` 且有 endpoint，调 `collectPostedCompletions`：
     `RdmaEndPoint::collectPostedCompletions` 在 `posted_fifo_[qp_index]` 中定位 `completed`，
     **退休它及其之前的所有 WR**（RC 顺序），返回数量并扣 `wr_depth`
     （`rdma_endpoint.cpp:1281-1293`；`rdma_posted_fifo.h:45-56`）。
   - 返回 0 表示这个 CQE 对应的 WR 已经被更早的 CQE 一起退休过，**必须整体 `continue`**：
     不能二次 `markSuccess`，也不能二次扣 `cq_outstanding`（`worker_pool.cpp:817-823`）。
   - 非 FIFO 模式：`drained = {completed}`，并从 `slice->rdma.qp_depth` 扣 1。
   - 展开 prefix：`prefix_status = (wc.status == WR_FLUSH_ERR) ? WR_FLUSH_ERR : SUCCESS`；
     对前缀中除最后一个之外的 slice 用 `prefix_status`，最后一个保留真实 `wc.status`
     （`worker_pool.cpp:835-843`）。这条规则的含义：一个 signaled CQE 之前的所有 unsignaled WR
     都已完成，而该 CQE 自身的状态只属于最后一个 WR。
   - `wr_retired` 汇总后一次性从 `cq_outstanding` 扣减。
4. 交给 `processCompletions`。

`processCompletions`（`worker_pool.cpp:854-1000`）是错误恢复的枢纽：

- **成功**：`markSuccess()`，并累计 `succeeded_slice_count`。
- **`IBV_WC_WR_FLUSH_ERR`**：显式排除在“真实网络错误”之外。它只是 QP 进 ERR 后硬件丢弃残留
  WR 的正常产物，是 `beginDestroy` 的预期结果（`worker_pool.cpp:889-916`）。
  但它仍然会走 `shouldRetrySlice` 重投；且当 context 已 inactive 时进
  `local_failed_slice_list`（换本地卡），否则进 `failed_slice_list`（换对端 rail）。
- **其他错误**：
  - 先拿 `slice->peerNicPath()` 的生命周期锁。
  - `local_wc_failure = isLocalWcFailure(wc)`（`worker_pool.cpp:1705-1730`）：
    只把 `LOC_LEN/LOC_QP_OP/LOC_PROT/MW_BIND/LOC_ACCESS/LOC_RDD_VIOL/LOC_EEC_*`
    当作本地 RNIC 故障；**`IBV_WC_GENERAL_ERR` 故意不算**，因为 provider 也拿它报连接/路径错误，
    误判会掩盖 GID 重探与远端 rail 恢复。
  - `!context_.active() || local_wc_failure` → 本地侧处理：
    - 每个 peer path 在一次 poll pass 里只记一次 `markRailFailed(path)`（**不立即暂停**），
      用 `local_failed_peer_paths` 集合去重。注释（#3299）解释了原因：一个 QP 本地故障会把
      它持有的所有 WR 一起完成，一次突发必须只算一次错误，否则会瞬间把这条 rail 熔断。
    - 每 pass 只调一次 `handleLocalFailure`（它递增 context 失败计数、`redispatch_counter_++`、
      删除 endpoint）；其余 slice 若其 endpoint 还没在本次 pass 删过，也各自
      `deleteEndpointByPtr` 一次，用 `local_failed_endpoints` 去重。
    - 重试列表换成 `local_failed_slice_list`。
  - 否则远端/default 故障：
    - 若有可用对端 rail，`markRailFailed(path, immediate=true)` + `redispatch_counter_++`；
    - 删除该 endpoint；
    - 进 `failed_slice_list`。
  - 不重试则 `finalize_slice(slice, false)`。
- 统计：`processed_slice_count_` 汇总所有终止 slice；**只有 `succeeded_slice_count > 0`
  才 `markContextSuccess()`**。注释明确说：完成错误（包括 FLUSH）不是“这张卡还能搬数据”的证据，
  所以不能清连续失败计数（`worker_pool.cpp:986-991`）。
- 最后分别 `redispatch(local_failed_slice_list, ..., handoff=true)` 与
  `redispatch(failed_slice_list, ..., handoff=false)`。

`redispatch`（`worker_pool.cpp:1002-1132`）：

- 达到 `max_retry_cnt` 直接失败（判断是 `retry_cnt >= max_retry_cnt`）。
- **本地 handoff**：`tryHandoffToAnotherLocalWorker` 只换源 RNIC，保持
  `target_id/peer_nic_path/dest_addr/dest_rkey` 不变，重算 `source_lkey`，
  把 `slice->rdma.endpoint` 置空，投到另一个 context 的
  `worker_pool_->submitPreparedPostSend`（`worker_pool.cpp:1134-1202`）。
  若没有别的本地 context 能接管，立即失败——注释说本地 RNIC 故障不能靠换远端 rail 修复。
- **远端重派**：重新 `selectPeerDevice(..., retry_cnt)`，改写 `dest_rkey` 与 peer path；
  若新选中的 rail 也被暂停，遍历其他 devices；全部暂停则失败。
  最后按 owner thread 入队：若 owner 就是自己且没有被要求延迟本地重派
  （`!defer_local_redispatch`），直接进 `collective_slice_queue_`，否则
  `enqueueSliceToOwner`（跨线程）。`defer_local_redispatch` 是
  `setupEndpointByActiveOutsideLifecycleGate` 等待期间 drain CQ 时置上的，避免在投递
  路径里递归重派（`worker_pool.cpp:1116-1123`）。

#### 4.5 两阶段销毁与 endpoint 回收

`EndpointStore::deleteEndpoint*`/`evictEndpoint` 不会立刻析构，而是：

```
beginDestroy():
  active_ = false; status_ = DESTROYING; 停 notify;
  若 has_connected_：把所有 QP 置 IBV_QPS_ERR（让硬件 flush 残留 WR 到 CQ）
waiting_list_.insert(endpoint); waiting_list_len_++
```

（`rdma_endpoint.cpp:273-307`；`endpoint_store.cpp:82-136`。）

`finishDestroy()` 有三道 gate（`rdma_endpoint.cpp:309-377`）：

1. 已 `DESTROYED` → 直接 true。
2. 非 `DESTROYING`（例如测试直接塞进 waiting list）→ 只有 `active_ == false` 才允许；
   `qp_list_` 为空说明从没 `construct` 过，直接标 `DESTROYED`，避免 `delete[]` 野指针。
3. `DESTROYING` → 检查 `wr_depth_list_` 是否全零；不零且未超过
   `kFinishDestroyTimeoutSec = 30s` 就返回 false；超过则**宁可永久泄漏也不强拆 QP**，
   因为陈旧 slice 引用还在，强拆会 UAF（`rdma_endpoint.h:242-246`）。

真正销毁调 `deconstructLocked()`：先按 `wr_depth` 修正 `cq_outstanding_`（即使随后
`ibv_destroy_qp` 失败也不会漏账），再逐个 `ibv_destroy_qp`；失败最多重试 3 次
（`kFinishDestroyMaxRetries`）。回收有两条触发路径：`RdmaContext::endpoint()` 插入新 endpoint 时
顺带一次，以及 `monitorWorker` 每秒一次 `reclaimEndpoints()`——后者是 issue #1845 专门加的，
因为故障负载下插入会停滞而淘汰仍在继续，waiting list 会无界增长
（`worker_pool.cpp:1518-1528`；`endpoint_store.cpp:138-147`、`337-346`）。

#### 4.6 异步事件与三级熔断

`monitorWorker` 每秒做一次 `reclaimEndpoints` / `pruneConnectPause`，并做 CQ 停滞诊断：
`poll_gap >= 5s` 或 `outstanding_age >= 30s` 时打日志（每 5 秒最多一次）；
开了 `track_rdma_posted_slices` 时还会按 peer 分组打印卡住的 slice 明细
（`worker_pool.cpp:1530-1626`）。之后 `epoll_wait(event_fd_, ..., 100ms)`，
若事件来自 `context()->async_fd` 就 `doProcessContextEvents()`。

`doProcessContextEvents`（`worker_pool.cpp:1251-1308`）：

- 循环 `ibv_get_async_event` 直到 `EAGAIN`。注释解释 async fd 是 edge-triggered 且每次只返回
  一个事件，必须一次排空，否则后面的事件（包括端口/设备错误）会被积压。
- `IBV_EVENT_QP_FATAL`：从 `qp_context` 取 endpoint，**先 `ibv_ack_async_event` 再操作 endpoint**。
  注释写明了死锁：另一个线程可能正持 `endpoint->lock_` 调 `ibv_destroy_qp`（eRDMA 下会阻塞到
  事件被 ack），此时先拿 lock 就会互等。ack 后再按 peer path 拿生命周期锁删除 endpoint。
- 其余事件交给 `handleContextEvent`。

`handleContextEvent`（`worker_pool.cpp:1318-1378`）：

- `DEVICE_FATAL / CQ_ERR / WQ_FATAL / PORT_ERR / LID_CHANGE`：
  `recovery_activate_after_ns_ = 0`；`resetContextBreaker(false)`（让异步事件独占 inactive 状态）；
  刷新已发布拓扑（把该设备从 HCA 列表里 disable）；**先 ack**；再锁所有生命周期锁并
  `disconnectAllEndpoints()`。
- `GID_CHANGE`：`refreshPublishedLocalGid()`；ack；若结果不是 UNCHANGED，断连所有 endpoint。
- `PORT_ACTIVE`：`scheduleContextRecovery()`，延迟 30 秒
  （`kContextRecoveryDelayNs`）再探测。注释解释：link-up 之后真实数据面仍可能拒绝 RTR 一段时间。
- 其他返回 false，由调用方 ack。

`maybeActivateRecoveredContext`（`worker_pool.cpp:1398-1430`）在截止时间到达后：
先刷新 GID；FAILED 就保持 inactive 并重新排期；CHANGED 则断连所有 endpoint；
最后 `resetContextBreaker(true)`、清 deadline、刷新拓扑。

**三个熔断器作用域与 TTL 完全不同**（这是最容易混的地方）：

| 熔断器 | key | 触发 | TTL / 默认 | 代码 |
|---|---|---|---|---|
| `ConnectPauseTracker` | peer server name（IP） | `RdmaContext::deleteEndpointByPtr`（路径失败 / QP fatal） | `MC_CONN_PAUSE_TTL_MS`，默认 0（关闭） | `connect_pause_tracker.h`；`rdma_context.cpp:1039-1072` |
| `RailState` | `peer_nic_path` | 5 次错误（`kRailErrorThreshold`）或 `immediate_pause` 强制；两次间隔 > 5 秒则重新计数 | `MC_RDMA_RAIL_PAUSE_SECONDS`，默认 30s | `worker_pool.cpp:1645-1696`；`worker_pool.h:165-172` |
| context 熔断 | 本 `RdmaContext` | 连续 32 个本地完成失败批次（每个 poll pass 至多记一次，`kLocalCompletionFailureThreshold`） | `MC_CONTEXT_PAUSE_TTL_MS`，默认 5000ms | `worker_pool.cpp:502-558` |

`RailState` 的 `paused_rail_count_` 让热路径在无暂停时完全跳过锁
（`isRailAvailable` 首行，`worker_pool.cpp:1675`）；锁内会顺便清理**所有**过期项，
避免一个再也不会被查询的 dest 让计数永远大于 0。

补充两个并发细节：`markLocalContextFailure` 在 context 已经 inactive 时直接返回 false
（`worker_pool.cpp:507`），这样 fatal 事件夺走恢复所有权之后，迟到的本地 CQE 不会再把
breaker 的 TTL 重新武装起来；`maybeActivateRecoveredContext` 的 deadline 是原子量，
读取不加 `context_state_lock_`，但写入由该锁串行化（`worker_pool.cpp:1398-1427`）。

#### 4.7 native notify（经典 rdma 独有）

`RdmaContext::construct` 里：

```cpp
native_notify_enabled_ = globalConfig().rdma_notify_enabled &&
                         std::string(engine_.getName()) == "rdma";
```

（`rdma_context.cpp:335-344`。）所以装 `rdma_twosided` 时父类这套 notify CQ 根本不会创建。

启用后 `RdmaEndPoint::constructNotification` 建一条专用 RC QP，256 个 slot × 64 KB
（`kNotifySlots`/`kNotifySlotBytes`，payload 上限 65528 字节），收发各注册一块 16 MB MR，
`postNotificationReceive` 预先投满 256 个 RECV（`rdma_endpoint.cpp:404-485`）。
`sendNotification` 在 `connected`/`readyToSend` 后等一个空 slot，编码为
`[name_len][name][msg_len][msg]`（native-endian 长度前缀），可选 inline SEND
（`rdma_endpoint.cpp:541-583`）。

`RdmaTransport::install` 末尾只要有任一 context 启用 native notify，就起
`notifyWorkerThread`：它轮询所有 context 的 `notify_cq_`，只有一整轮都没有完成时才
`sleep_for(10us)`（`rdma_transport.cpp:256-274`）。`pollNotificationCq` 通过 `notify_endpoints_`
把完成映射回 endpoint，再 `meta()->pushNotify(message)`（`rdma_context.cpp:365-402`）。

native notify 还有两条锁顺序约束：`dispatchNotificationCompletion` 先在 `notify_mutex_`
下取出 endpoint 的强引用，释放 `notify_mutex_` 后才回调 endpoint
（`rdma_context.cpp:376-387`），与 teardown 的相反加锁顺序错开；`sendNativeNotify` 则在
握手、等发送槽和实际 SEND 期间临时释放生命周期锁，并在每次重新拿锁后校验 endpoint
仍是 store 里那个、仍 active、未 retired——注释明确说**不重放送达状态不确定的 notify**
（`rdma_transport.cpp:1256-1276`）。

---

## 与其他模块的边界

### 被谁依赖

| 调用方 | 交互点 | 契约 |
|---|---|---|
| `MultiTransport` | `installTransport("rdma" / "rdma_twosided")` | 二者**互斥**，同时安装直接报错（`src/multi_transport.cpp:578-590`） |
| `MultiTransport::selectTransport` | 协议路由 | 目标段 protocol 是 `"rdma"` 时若只装了 twosided，会映射过去；twosided 发布到元数据的 protocol 仍是 `"rdma"`（`multi_transport.cpp:881-889`） |
| `TransferEngine` / `TransferEngineImpl` | `submitTransfer` / `submitTransferTask` / `getTransferStatus` / 注册接口 | `Transport` 基类虚函数（`transport.h:547-622`）；注册接口是 `private virtual`，靠 `friend class TransferEngineImpl` 调用（`transport.h:45-46`、`607-622`） |
| `HeterogeneousRdmaTransport`（ascend） | 调用**非虚**的 `getTransferStatus(BatchID, vector<TransferStatus>&)` | 不是多态入口，别当基类接口用 |

### 依赖谁

| 依赖 | 用途 | 关键位置 |
|---|---|---|
| libibverbs | QP/CQ/MR/PD、post/poll、async event | 全局 |
| libmlx5 + mlx5dv（可选） | LAG 端口数、LAG 绑定、UDP 源端口、Data Direct、dma_buf 注册 | `rdma_context.cpp:347-356`、`rdma_endpoint.cpp:1662-1720` |
| CUDA/HIP/MACA/MLU/SUPA Driver（可选） | 判断 host/device 内存、导出 dma_buf | `rdma_context.cpp:510-748` |
| `TransferMetadata` | 段描述发布/加载、握手 RPC、notify 推送 | `rdma_transport.cpp:250-254`、`1377-1382` |
| `Topology` | `getHcaList`、`selectDevice`、`selectDeviceByLocalHca`、`getDeviceIndex`、`disableDevice` | `rdma_transport.cpp:1402-1436` |
| `BufferRangeIndex` | 段内按地址区间选 buffer 的 O(log n) 索引 | `rdma_transport.cpp:65-72`、`99-106` |
| `memory_location` | `segments:` 前缀解析、`kWildcardLocation`、`detectBufferPageSize` | `rdma_transport.cpp:50-58`、`407` |
| `GlobalConfig` | 所有 `MC_*` 参数 | `include/config.h` |
| Linux epoll | 聚合 async_fd 与 comp channel | `rdma_context.cpp:298-318`、`1795-1817` |

### 数据怎么流过去

```
上层 BatchTransfer
  → RdmaTransport::submitTransferTask        (切片 + 本地 NIC/lkey, 按 context 分组)
  → RdmaContext::submitPostSend              (纯转发)
  → WorkerPool::submitPostSend               (对端 NIC/rkey, 按 peer 哈希分桶)
  → WorkerPool::performPostSend              (取/建 endpoint, 必要时握手)
  → RdmaEndPoint::submitPostSend             (填 WR, ibv_post_send)
  → 硬件
  → WorkerPool::performPollCq                (ibv_poll_cq)
  → RdmaEndPoint::collectPostedCompletions   (前缀退休, wr_depth--)
  → WorkerPool::processCompletions           (markSuccess/markFailed, 重派/熔断)
  → TransferTask 计数器 → getTransferStatus
```

注意 `getTransferStatus` 只读计数、不驱动状态迁移；批次完成的发布由
`Slice::markSuccess/markFailed` 对 `completion_state` 的封口协议完成
（`transport.h:265-335`），封口位则是在 `submitTransferTask` 返回后由 `MultiTransport` 置上。
vector 版本与单 task 版本逻辑相同（`rdma_transport.cpp:1177-1232`），
后者是 `override`，前者不是。

---

## 设计取舍与易踩坑点

1. **两阶段选卡不是冗余，而是信息边界**。本地选卡只需要本地段描述；对端选卡依赖对端的
   `SegmentDesc`（可能被 reload）与 rail 暂停状态，而且必须在 owner worker 线程里与握手、
   投递串行化。把两者塞进 `submitTransferTask` 会让提交线程承担远端元数据读取与握手阻塞。

2. **slice 长度不等于 `slice_size`**。实际长度还要被两侧 MR 边界截断，并在剩余量
   `<= block + fragment` 时整块发送（`rdma_transport.cpp:132-148`）。看到 `slice_size=65536`
   就假设每个 slice 都是 64 KB 会算错 QP 额度和日志。

3. **`fail_unstarted_tasks` 的零长 slice 是协议补丁，不是脏数据**。它让
   `success+failed==slice_count` 对所有 task 都成立（`rdma_transport.cpp:969-986`），
   更关键的是把未开始 task 的 `0==0`（会被误报成 COMPLETED）变成 `1==1` 的 FAILED。

4. **`BatchRdmaDeviceCache` 只在很窄的条件下生效**：单 HCA、索引无重叠、同 desc、
   buffer 覆盖区间、非 `segments:` buffer（`rdma_batch_cache.h:47-60`）。
   它缓存的是“首次选卡结果”，命中时不会重新考虑 hint；调用方必须保证同一批请求策略一致。

5. **`cq_context` 裸指针技巧**：`RdmaEndPoint::cq_outstanding_` 指向
   `cq_list_[i].outstanding` 的内部地址。任何在 endpoint 存活期间让 `cq_list_` 重新分配的改动
   都会造成悬垂指针。`RdmaCq` 删除拷贝/移动赋值就是为此，但**移动构造仍然会让地址变化**，
   真正的保障是“`resize` 只发生在 endpoint 创建之前”。

6. **非全信号投递不减少 CQ 额度占用**。失败/被 flush 的 WR 即使 unsignaled 也产生 CQE，
   所以 `cq_outstanding_` 必须按每个 WR 预扣（`rdma_endpoint.cpp:1316-1319`）。
   省下的是成功路径的 CQE 数，不是 CQ 深度。

7. **陈旧 CQE 必须整体跳过**。`collectPostedCompletions` 返回 0 时若继续
   `markSuccess`，会二次计数、二次扣 `cq_outstanding`，最终把 CQ 额度算成负数
   （`worker_pool.cpp:817-823`）。

8. **`IBV_WC_WR_FLUSH_ERR` 不是网络错误**。它是 `beginDestroy` 把 QP 置 ERR 后的正常产物；
   但它仍要重投，且 context 已 inactive 时走本地 handoff 而不是换对端 rail
   （`worker_pool.cpp:889-916`）。`markContextSuccess` 只在真正成功时清零，
   FLUSH 不能算“这张卡还能用”的证据。

9. **`IBV_WC_GENERAL_ERR` 被故意排除在本地故障之外**，否则会掩盖 GID 重探和远端 rail 恢复
   （`worker_pool.cpp:1705-1730`）。

10. **#3299 的去重**：一次 QP 本地故障会让该 QP 上所有 WR 一起完成，若每个 CQE 都记一次
    rail 错误，5 次阈值瞬间触发，具体路径被误熔断。因此用
    `local_failed_peer_paths` 每 pass 每 path 只记一次，且**不立即暂停**
    （`worker_pool.cpp:938-952`）。

11. **endpoint 可能被“故意泄漏”**。`finishDestroy` 等 `wr_depth_list_` 全零，30 秒不排空就
    留在 waiting list。这比强拆 QP 导致陈旧 slice 引用 UAF 更安全
    （`rdma_endpoint.cpp:336-359`；`rdma_endpoint.h:242-246`）。

12. **`resetConnection` 对“曾经连上过”的 endpoint 是退休而不是重置**。只要
    `has_connected_ == true`，任何握手失败都 `beginDestroyLocked()`；只有从未成功连过的
    endpoint 才 `disconnectUnlocked()` 重建 QP。理由是已连过的 QP 上可能还有用户 WR
    （`rdma_endpoint.cpp:1190-1234`）。`disconnectUnlocked` 在 pre-connected 情况下还会
    `CHECK_EQ(wr_depth, 0)` 后 `reconstruct()`，并 `++qp_generation_` 让等待者失效。

13. **占位 Context 不是 bug**。某张网卡 `construct` 失败时，`initializeRdmaResources`
    仍 push 一个 `set_active(false)`、`pd_ == nullptr` 的空壳，以保持
    `context_list_` 下标与 `getHcaList()`、`BufferDesc::lkey/rkey` 对齐
    （`rdma_transport.cpp:1353-1368`）。所有 MR/rkey/lkey/preTouch 入口都用
    `if (!pd_) return` 静默跳过（`rdma_context.cpp:864`、`876`、`921`、`942`、`953`）。
    因此死卡的 key 读作 0 是预期行为。

14. **Data Direct 配对失败是静默降级**：写 `mr = nullptr` 并返回 0
    （`rdma_context.cpp:793-798`）。看到 `if (!iter->second.mr)` 不要以为漏了错误处理。
    `unregisterMemoryRegion` 与 `deconstruct` 都会跳过这类条目。

15. **单 dma_buf fd 的生命周期契约很硬**。fd 必须在所有 NIC 注册完成后才能关；
    单个 NIC 的注册各自持引用。`registerLocalMemoryInternal` 用 `DmabufCloser` 保证任意返回路径
    恰好关一次（`rdma_transport.cpp:360-371`）。中途失败时，代码先用
    `unregisterChunkMRs`/`rollbackChunks` 回滚本 chunk 的半成品 MR 与已提交 chunk 的
    元数据，再整体返回错误，不会留下“部分 NIC 注册成功”的状态。

16. **`chunk_limit` 的两种对齐互斥**：HugeTLB 向下对齐只对主机内存生效（Data Direct 只对显存，
    显存映射报告 base page）；Data Direct 的页对齐前缀又要求 `prefix < chunk_limit`
    （`rdma_transport.cpp:407-451`）。改 `MC_MAX_MR_SIZE` 时要同时考虑这两条。

17. **`chunk_map_` 的反注册语义**：`unregisterLocalMemory` 只拿到基址；`chunk_map_` 被
    `erase` 后必须把所有 chunk 都处理完，提前 return 会泄漏剩余 chunk 的 MR 与元数据
    （`rdma_transport.cpp:684-744`）。

18. **`MC_MAX_EP_PER_CTX` 的报错文案具有误导性**。`doSetupConnection` 发现
    `qp_list_.size() != peer_qp_num_list.size()` 时提示检查 `MC_MAX_EP_PER_CTX`
    （`rdma_endpoint.cpp:1470-1481`），但 QP 数来自两侧的 `num_qp_per_ep`；
    这是跨节点配置一致性问题，不是本地 endpoint 数量问题。另外
    `RdmaContext::construct` 头文件默认 `max_endpoints = 256` 也是误导性的，
    生产调用方传的是 `globalConfig().max_ep_per_ctx`（默认 65536），
    并会被 `updateGlobalConfig` 按 `device_attr.max_qp / num_qp_per_ep` 向下夹取
    （`config.cpp:858-873`）。

19. **三个熔断器不要混为一谈**（见 4.6 表格）：作用域分别是 peer IP、peer rail、本地 context；
    TTL 默认分别是关闭、30s、5s。`ConnectPauseTracker` 默认 TTL=0 时
    `pauseConnect`/`isConnectPaused` 直接返回（`rdma_context.cpp:1056-1070`）。

20. **GID 自动选择不是“挑第一个能用的”**。默认 `MC_GID_INDEX=-1` 会走七级排序；
    私网 IPv4 从“降级”里单独拆出（#2729），真正可靠的降级信号是 overlay 网卡名
    （docker*/cni*）而不是私网地址本身（`rdma_gid_probe.h:110-131`）。
    RTR `EINVAL` 会触发 `reprobeAutoGid` 换候选重试，默认最多 2 次
    （`config.h:70`；`rdma_endpoint.cpp:930-970`）。

21. **显式 GID 与自动 GID 在 `IBV_EVENT_GID_CHANGE` 时行为不同**：自动模式重跑整套候选
    过滤排序（可能换 index）；显式模式保持配置的 index，只刷新其 GID 值
    （`rdma_context.cpp:1407-1521`，注释在 `rdma_context.h:273-277`）。

22. **`RdmaTwoSidedTransport` 只换通知通道，不换数据面**。数据面 100% 继承父类；
    `getName()` 是 `"rdma_twosided"`，但发布到元数据的 segment protocol 仍是 `"rdma"`。
    两者在同一 `TransferEngine` 实例里互斥（`multi_transport.cpp:578-590`、`885-890`）。
    并且 `native_notify_enabled_` 要求 `getName() == "rdma"`，所以 twosided 安装时父类
    native notify CQ 不会创建（`rdma_context.cpp:335-336`）。

23. **twosided 的控制面目前是半成品**：除 `NOTIFY_COMPAT` 外，`SESSION_OPEN`、
    `CREDIT_GRANT`、`DATA_ACK` 等帧全部落到 `VLOG(1) << "... (PR2 notify-only)"` 被忽略
    （`ctrl_plane.cpp:205-237`）；`postSessionOpen()` 直接 return 0
    （`ctrl_channel.cpp:413-417`）。`SenderCreditLedger` 只有单测在用。
    不要根据头文件里的帧类型数量推断“信用流控已经工作”。

24. **twosided 的 `ConnectScope` 与 `ctrl_mutex_` 是一套精细的抢占协议**。构造时发布占位
    channel 并 `ctrl_connecting_[peer]++`；析构时**总是**撤销 in-flight 标记并唤醒等待者，
    占位 channel 只有在 `markSuccess()` 之后才保留，否则一并撤回。调用方必须在整个 scope
    生命周期持 `ctrl_mutex_`，因此阻塞握手必须用 `UnlockGuard` 临时解锁而不是手动释放
    （`rdma_twosided_transport.h:70-93`；`ctrl_plane.cpp:40-80`）。
    `ensureCtrlChannel` 用绝对 deadline（默认 10s）避免重试无限延长；发现“未连接且无 in-flight”
    的 channel 会主动回收重建，而不是等一个不会来的唤醒
    （`ctrl_plane.cpp:125-192`）。

25. **`CtrlChannel` 的 send 是“入队成功”而非“送达成功”**。`sendCtrlFrame` 返回 0 只表示
    WR 已投递；失败由 SEND completion 的 WC 异步发现，随后 `connected_ = false`
    （`ctrl_channel.h:71-82`；`ctrl_channel.cpp:482-485`、`568-583`）。下一次
    `ensureCtrlChannel` 会把这个“未连接且无 in-flight”的 channel 回收重建；如果仍然拿不到
    可用 channel，`TransferEngineImpl::sendNotifyByName` 会按
    `MC_RDMA_NOTIFY_OOB_FALLBACK`（默认开）回退到 OOB notify
    （`transfer_engine_impl.cpp:682-699`）。发送路径会在 SQ 满时**主动 poll SEND 完成**，
    避免调用者与 ctrl worker 互相等待造成死锁（`ctrl_channel.cpp:438-447`）。

26. **native notify 有两条彼此独立的实现，且互斥启用**：经典 rdma 用
    `RdmaEndPoint` 内嵌通知 QP + `RdmaContext::notify_cq_`；twosided 用 `CtrlChannel`。
    看到 `kNotifySlots=256, 64KB` 就以为 twosided 也用这套是错的。

27. **文档来源的坑**：`docs/source/design/transfer-engine/transport/` 下没有
    `rdma_transport.md`；RDMA 说明散落在 `docs/source/design/transfer-engine/index.md`
    的 “Topology Aware Path Selection / Endpoint Management / Fault Handling” 三节
    （`index.md:52-91`），而且通篇用 “Mooncake Store employs...” 的口吻，
    容易让人误以为讲的是 Store 而不是 Transfer Engine。

28. **`RdmaTransport::install` 的双名字**：`MC_RDMA_BIND_ADDRESS` 让
    `rdma_server_name_`（NIC path 用）与 `local_server_name_`（TCP/P2P 与段名用）分叉
    （`rdma_transport.cpp:216-229`）。双网卡环境下只按 `local_server_name_` 反推 NIC path
    会对不上；`nicPath()` 一律用 `rdma_server_name_`（`rdma_context.cpp:1088-1090`）。

29. **`getTransferStatus(BatchID, vector<TransferStatus>&)` 不是 override**。
    基类只声明了 `(batch_id, task_id, status)` 这一个虚函数（`transport.h:578`），
    vector 版本是 `RdmaTransport` 自己加的非虚重载（`rdma_transport.h:107-108`），
    目前只有 ascend 异构传输调用。不要把它当成多态入口。

30. **`ptr` 生命周期**：`Slice::rdma.endpoint` 是裸指针，指向 `EndpointStore` 里的
    `shared_ptr` 对象。endpoint 被删除后进入 waiting list 才析构，正是为了让仍在 CQ 中的
    slice 指针不悬垂。不要在 `deleteEndpointByPtr` 后立即假设对象已释放。另外
    `getEndpointByPtr` 同时搜索活跃 map 与 `waiting_list_`，而 `deleteEndpointByPtr` 在
    store 锁内**只做指针身份比较、绝不解引用**（`endpoint_store.h:46-54`），所以
    `IBV_EVENT_QP_FATAL` 携带的陈旧 `qp_context` 指针也不会造成 UAF。

> 关于 grouped scatter：`TransferTask` 的 `request` 指针 + `request_count` 与
> `supportsGroupedScatter() == true` 确实支持“一个 task 挂多个 request”。
> `submitTransferTask` 用 `task.request[request_index++]` 遍历
> （`rdma_transport.cpp:1009-1013`），而 `RdmaTransport::submitTransfer` 每次只填一个 request
> （`rdma_transport.cpp:926-938`）。多 request 的入口在
> `MultiTransport::submitTransfer(batch_id, entries, task_sizes)`：它把连续的、具有相同
> `task_group_id` 且路由到同一 transport 的请求合成一个 task，并把 `request_count` 设为组大小
> （`multi_transport.cpp:283-308`）。上层入口是 `MultiTransport::submitScatter()`，
> 它分配 batch 并以 `task_sizes` 调 `submitTransfer`（`multi_transport.cpp:327-337`）。
> 调用方需让 transport 声明 `supportsGroupedScatter()`，且请求带相同的 `task_group_id`，
> 否则每组退化为单 request。

---

## 精读清单

建议按以下顺序读，每步都先读头文件再读实现，并带着“谁在哪个线程、持什么锁”的问题。

1. **`include/transport/transport.h:166-335`**
   看懂 `Slice` 的联合体、`markSuccess/markFailed` 的封口协议、`TransferTask` 的
   `slice_count` 终止条件。这是理解后面所有“伪造 slice”“重派不重复计数”的前提。

2. **`include/config.h:40-181`**
   把 `slice_size`、`fragment_limit`、`max_wr`、`num_qp_per_ep`、`workers_per_ctx`、
   `num_cq_per_ctx`、`max_ep_per_ctx`、三个熔断 TTL、`rdma_signal_interval` 的默认值记下来。
   后面每个魔数都能在这里找到来源。

3. **`src/transport/rdma_transport/rdma_transport.cpp:204-264`（install）**
   搞清资源初始化、段 ID 分配、握手 daemon、元数据发布、notify 线程启动的顺序，
   以及失败时各自返回什么。

4. **`rdma_transport.cpp:339-663`（registerLocalMemoryInternal）**
   重点看：单 fd dma_buf、`max_mr_size` 切 chunk、HugeTLB/Data Direct 两套对齐、
   每 chunk 的 `BufferDesc`、只发布一次、`chunk_map_`。
   然后读 `rdma_context.cpp:758-918`，看 `mr == nullptr` 的降级路径与反注册的 `MADV_DOFORK`。

5. **`rdma_transport.cpp:942-1175`（submitTransferTask）**
   手画一遍：一个 request → N 个 slice → 按 Context 分组的 map。
   特别关注 `SliceLengthCalculator`、`request_hint`/`local_hint_device_id`、
   `BatchRdmaDeviceCache`、`kSubmitWatermark`、三个失败 lambda。

6. **`rdma_transport.cpp:1387-1520`（selectDeviceImpl / pickTopologyDevice）**
   理解唯一覆盖索引快路径、`retry_count > 0` 禁止复用 hint、显式 hint 为什么必须绕开
   带 hint 的重载、`kWildcardLocation` 回落。

7. **`worker_pool.cpp:296-451`（submitPostSend）**
   第二次选卡：对端 NIC/rkey、`failed_target_ids` 100ms 黑名单、段 reload、
   rail 暂停时遍历 devices 换路、`internedNicPath`、按 owner 哈希分桶。

8. **`worker_pool.cpp:580-775`（performPostSend）**
   逐行读故障分支：context inactive 整队 handoff、`redispatch_counter_`、
   connect pause 与 rail pause 两道前置检查、`setupEndpointByActiveOutsideLifecycleGate`
   的“释放锁 + 8 线程 executor + 等待期间 drain CQ”、ready ACK 超时。

9. **`rdma_endpoint.cpp:1295-1414`（submitPostSend）**
   唯一调 `ibv_post_send` 的函数。重点：QP 间均摊、双重额度、信号周期、
   `trackPostedSlices` 在敲门之前、bad_wr 回滚、`rc != 0` 立即 `beginDestroyLocked`。

10. **`worker_pool.cpp:777-1000`（performPollCq + processCompletions）**
    重点：FIFO 前缀退休与陈旧 CQE 跳过、prefix_status 展开、
    FLUSH 特殊分支、`isLocalWcFailure` 的二分、#3299 去重、
    `markContextSuccess` 只认成功。

11. **`worker_pool.cpp:1002-1202`（redispatch + tryHandoffToAnotherLocalWorker）**
    搞清“换本地卡”与“换对端 rail”两条互斥策略，以及各自的 retry 计数与失败条件。

12. **`rdma_endpoint.cpp:653-1004` 与 `1006-1183`（主动/被动握手）**
    画状态机：UNCONNECTED→CONNECTING 的唯一赢家、输家退避与 `qp_generation_`、
    RPC 锁外执行、simultaneous open 复用、ready ACK、
    `resetConnection` 的“连过就退休”语义、被动侧的竞态吸收。

13. **`rdma_endpoint.cpp:1519-1723`（doSetupConnection）**
    对照 verbs 手册看 RESET/INIT/RTR/RTS 的每个属性，尤其 irdma 的 RESET 特例、
    `path_mtu` 取 min、`min_rnr_timer=12`、`timeout=14/retry_cnt=7`、LAG/UDP 源端口两个尾部。

14. **`rdma_endpoint.cpp:219-377` 与 `endpoint_store.cpp`**
    把两阶段销毁、`waiting_list_`、`reclaimEndpoint`、SIEVE hand 串起来。
    理解“30 秒排不空就泄漏”的取舍。

15. **`rdma_context.cpp:226-363`（construct）**
    理解 `num_cq_list` 抬升、CQ 的 `cq_context` 技巧、`endpoint_lifecycle_locks_` 按 worker 分配、
    FIFO/SIEVE 选择、`native_notify_enabled_` 的条件、`WorkerPool` 与 NUMA 绑定。

16. **`rdma_context.cpp:1237-1521` + `rdma_gid_probe.h`**
    自动 GID 的七级分类与重探，对照 `IBV_EVENT_GID_CHANGE` 的处理一起读。

17. **`worker_pool.cpp:1212-1643`**
    worker 主循环的 park 协议、monitor 的 CQ 停滞诊断、async event 的 ack 顺序与死锁注释、
    PORT_ACTIVE 的 30 秒延迟恢复。

18. **`rdma_twosided/`（ctrl_plane.cpp → ctrl_channel.cpp → ctrl_frame.cpp）**
    先读 `ensureCtrlChannel` 与 `ConnectScope`，再读 `CtrlChannel::connectActive/acceptPassive`、
    `sendCtrlFrame` 的槽位与轮询、`drainCompletions` 的错误 WC 处理。
    最后确认哪些帧类型仍是 no-op，避免把“定义了”当成“实现了”。

完成以上 18 步后，再回头看 `docs/source/design/transfer-engine/index.md:52-91`，
应当能把文档里的三节与代码中的具体函数一一对应；若对不上，以代码为准。

---

# 04 其它传输后端与共享内存段

本章讲 Transfer Engine（下称 TE）里 RDMA 之外的那一大票传输后端，以及两个经常被误认为"传输后端"的独立子系统：`shared_segment`（跨进程共享主机内存）和 `nvlink-allocator` / `ubshmem-allocator`（供 Python 侧加载的 fabric 内存分配器）。

RDMA 后端与引擎主体不在本章展开。`src/transport/rdma_transport/`、`rdma_twosided/`、`device/`（GPU-initiated 路径）、`cxi_transport/` 属于同一抽象层但另有章节；`tent/` 是一套并行引擎，也不是本章对象。

本章的读法建议：**先接受"所有后端都只是 `Transport` 的一种数据搬运方式"这个前提，再把差异归到两件事上——数据怎么搬、对端地址怎么翻译**。抓住这两条，剩下几十个后端都只是参数不同的实例。

---

## 定位与职责

### 在整体中的位置

一次典型的 TE 操作从公开 API 进入，按下面的链路由上到下：

```
TransferEngine (include/transfer_engine.h)
  -> TransferEngineImpl (src/transfer_engine_impl.cpp)
     -> MultiTransport (src/multi_transport.cpp)
        -> Transport 的某个具体子类（本章对象）
           -> 硬件/内核/运行时接口
```

`MultiTransport` 做两件事：

1. **安装**：`MultiTransport::installTransport(proto, topo)` 按字符串协议名 `new` 出具体类（`mooncake-transfer-engine/src/multi_transport.cpp:567`）。
2. **选路**：`MultiTransport::selectTransport(entry, transport, allows_reuse)` 读取目标 `SegmentDesc::protocol`，决定把这条请求交给哪个后端（`src/multi_transport.cpp:778`）。

因此本章子系统解决的问题是：**同一套批量提交 API（`allocateBatchID` / `submitTransfer` / `getTransferStatus`），要能跑在 TCP、POSIX 共享内存、CXL、各家 GPU 的 IPC/scale-up 链路、集合通信库、NVMe-oF 之上，并且绝大多数后端是编译期可选、运行期按环境变量/拓扑自动安装的**。

### 按搬运机制分类

| 类别 | 代表后端 | 数据如何移动 | 对端地址如何得到 |
|---|---|---|---|
| 同步 memcpy | `shm`、`cxl` | 调用线程内直接 `memcpy` | `shm` 用 `shm_name` 重新 `mmap` 后换算；`cxl` 用 `offset + 本地 cxl_base_addr` |
| socket/网络流 | `tcp`、`efa`、`cxi`、`ub`、`barex` | 内核 socket / libfabric / URMA / ACCL | `tcp` 用 host:port 建连，header 里带目标进程的绝对 VA，服务端校验后直接写；其余靠握手交换 rkey/端点 |
| GPU IPC / scale-up | `nvlink`、`nvlink_intra`、`hip`、`maca`、`musa`、`hylink`、`sunrise_link` | `cudaMemcpyAsync` / `hipMemcpyAsync` / 厂商拷贝 API（Sunrise 用 `tang`/`ptml`） | 导入对端 IPC/fabric handle，得到本进程可访问的 VA |
| 集合通信库 | `nccl`、`flagcx`、`hccl` | `ncclPutSignal` / FlagCX P2P engine / HCCL | 对端注册 buffer 的对称窗口或 MR |
| 存储 | `nvmeof`、`ascend_direct` | cuFile / 昇腾传输引擎 | 文件偏移或设备地址 |
| 独立子系统 | `shared_segment` | memfd + mmap（或昇腾 VMM） | 各 rank 用 `local_base + offset`，**不要求 VA 一致** |
| 独立子系统 | 两个 allocator `.so` | 让 torch 分配的显存可被 IPC 导出 | 由 Python 经 ctypes 调用 |

### 一条必须记住的共性

每个后端都在填 `Transport::Slice` 里那个 **per-backend union**（`include/transport/transport.h:203`），搬完数据后调用 `Slice` 自带的 `markSuccess()` / `markFailed()`。完成计数、Batch 通知、回调 pin 全部由 `Slice` 的完成协议处理，后端只负责"这一片数据成了还是败了"。这是读任何后端 `submitTransfer` 时最容易忽略的收尾。

---

## 目录结构速览

路径均相对 `mooncake-transfer-engine/`。

| 文件 / 目录 | 一句话职责 |
|---|---|
| `include/transport/transport.h` | `Transport` 抽象基类、`Slice`（含 union）、`TransferTask`、`BatchDesc`、完成协议 |
| `src/transport/transport.cpp` | 基类唯一实现：`allocateBatchID` / `freeBatchID` / `install` / 线程本地 slice 缓存 |
| `src/multi_transport.cpp` | 后端注册、协议到后端的映射、多协议按 buffer 选路、批量状态聚合 |
| `include/multi_transport.h` | `MultiTransport` 声明：`transport_map_`、延迟释放线程 |
| `src/transport/CMakeLists.txt` | 所有后端的编译开关总表；`shm_transport` 是唯一无条件编译的 |
| `include/transport/tcp_transport/tcp_transport.h` | `TcpTransport` + `TcpIoPool` + lane 状态机全部类型 |
| `src/transport/tcp_transport/tcp_transport.cpp` | TCP 的 `Transport` 接口层（843 行） |
| `src/transport/tcp_transport/tcp_transport_session_impl.h` | 被 `.cpp` include 的线协议与会话实现（982 行） |
| `src/transport/tcp_transport/tcp_transport_lane_impl.h` | 被 `.cpp` include 的 lane 调度状态机（1674 行） |
| `include/transport/shm_transport/shm_transport.h` | `ShmTransport` + `MappingPin` + 名字判别辅助 |
| `src/transport/shm_transport/shm_transport.cpp` | SHM 全部实现（1065 行） |
| `include/transport/nvlink_transport/nvlink_transport.h` | `GpuIpcTransportPolicy` 策略接口 + `NvlinkTransport` |
| `src/transport/nvlink_transport/nvlink_transport.cpp` | `NvlinkTransport` 的 IPC 拷贝核心；实际编译条件为 `USE_MUSA OR (USE_MNNVL AND NOT USE_HIP)`（`src/transport/CMakeLists.txt:84-87`），HIP/MACA 各有独立实现 |
| `include/transport/musa_transport/musa_transport.h` | 37 行，只是 `final : public NvlinkTransport` |
| `src/transport/cxl_transport/cxl_transport.cpp` | `/dev/daxX.Y` mmap + `memcpy`，offset 寻址 |
| `src/transport/nccl_transport/nccl_transport.cpp` | `NcclHostTransport`，用 NCCL 对称窗口（`ncclCommWindowRegister`）+ `ncclPutSignal`；只支持 WRITE，READ 直接返回 `NotSupportedTransport`（`include/transport/nccl_transport/nccl_transport.h:43-45`） |
| `src/transport/ascend_transport/` | 4 个昇腾后端 + `ascend_allocator.cpp` |
| `src/transport/kunpeng_transport/` | `UbTransport`（URMA / OBMM 两种 endpoint） |
| `src/transport/efa_transport/` | AWS EFA（libfabric） |
| `src/transport/flagcx_transport/` | FlagCX P2P engine |
| `src/transport/nvmeof_transport/` | cuFile / GPUDirect Storage |
| `src/transport/rpc_communicator/` | **不是** Transport 子类，是基于 coro_rpc 的控制面 RPC |
| `include/shared_segment/shared_segment.h` | `SharedSegment` 公开接口与两阶段设计注释 |
| `src/shared_segment/shared_segment.cpp` | 后端选择、blob 编解码、fingerprint、Create/Complete |
| `src/shared_segment/mmap_shared_segment.cpp` | memfd + THP 后端（默认路径） |
| `src/shared_segment/ascend_shared_segment.cpp` | 昇腾 VMM / fabric handle 后端 |
| `nvlink-allocator/` | 编成 `nvlink_allocator.so` 的 fabric 显存分配器 |
| `ubshmem-allocator/` | 编成 `ubshmem_fabric_allocator.so` 的昇腾 fabric 分配器 |
| `fabric_allocator.cmake` | `add_fabric_allocator_build_target()`：用 `add_custom_target` + `add_custom_command` 调 `build.sh` 产出上述 `.so`；`.so` 不经过普通 CMake 编译/链接规则 |

---

## 核心数据结构

### 1. `Transport`：统一抽象

`include/transport/transport.h:44` 定义的 `Transport` 是纯虚基类，关键虚函数：

- `allocateBatchID(batch_size)` / `freeBatchID(batch_id)`：有默认实现（`src/transport/transport.cpp:27,42`）。
- `submitTransfer(batch_id, entries)`：纯虚，老式"一次交一批 Request"。
- `submitTransferTask(task_list)`：默认返回 `NotImplemented`，**实际主路径**是它——`MultiTransport` 总是构造 `TransferTask` 后调这个（`src/multi_transport.cpp:314`）。
- `submitTransferTaskGroup(task_list)`：默认转调 `submitTransferTask`；TCP 也覆写了它（`include/transport/tcp_transport/tcp_transport.h:169-170`），但仓库内没有调用者——真正生效的 grouped scatter 顺序由 `TcpTransport::submitTransferTask` 内部按 `task_group_id` 聚合并走 `startTransferSequence`（`tcp_transport.cpp:572-594`）。
- `supportsGroupedScatter()`：TCP 返回 true（`tcp_transport.h:172`），因为组内 slice 按请求顺序追加。
- `supportsBatchCompletionCounter()`：目前**只有 RDMA 打开**（`include/transport/rdma_transport/rdma_transport.h:67`）；其余后端返回 false，意味着批量状态查询必须逐 task 走。
- `getTransferStatus(batch_id, task_id, status)`：纯虚。
- `registerLocalMemory*` / `unregisterLocalMemory*`：private 纯虚，`TransferEngine`/`TransferEngineImpl` 是友元，由它们广播给所有已安装后端。

### 2. `Slice`：所有后端的差异都在这一个 union 里

`include/transport/transport.h:168-262`。

公共字段：`source_addr`、`length`、`opcode`、`target_id`、`status`、`task`（`SliceTaskPtr`）、`dest_rkeys`、`cleanup_callback`、`ts`。

union 分支（`transport.h:203-262`）：

| 分支 | 关键字段 | 使用者 |
|---|---|---|
| `rdma` | `dest_addr`、`source_lkey`/`dest_rkey`、`qp_depth`、`qp_index`、`endpoint` | RDMA |
| `ub` | `dest_addr`、`jetty_depth`、`r_seg`/`l_seg`、`endpoint` | UB/URMA |
| `local` | `dest_addr`、`cuda_stream`、`cuda_device` | **shm 也复用它**（`shm_transport.cpp:929`）以及本地异步拷贝 |
| `nccl` | `event`、`device_id` | NCCL |
| `tcp` | `dest_addr` | TCP |
| `nvmeof` | `offset`、`cufile_desc`、`file_path` | NVMe-oF |
| `cxl` | `dest_addr` | CXL |
| `hccl` | `dest_addr` | HCCL |
| `ascend_direct` | `dest_addr`、`handle`、`start_time`、`engine_id` | 昇腾直接传输 |
| `ubshmem` | `dest_addr` | 昇腾 UBShmem |
| `flagcx` | `dest_offset` | FlagCX |

注意 `shm` 没有自己的分支，它借用了 `local`。这解释了一个初读时的困惑：为什么在 SHM 代码里看到 `slice->local.dest_addr`。

`SliceTaskPtr`（`transport.h:116-149`）是对 `std::atomic<TransferTask*>` 的手写包装。存在的理由写在 110-115 行注释里：`Slice::task` 跨线程读写必须是原子的，但 `std::atomic` 不可拷贝/移动，会破坏那些按值重置 `Slice` 的后端和测试。它提供 `operator->`、隐式转指针、`load()`。

`toBatchDesc`（`transport.h:162-164`）把 `BatchID` 直接 `reinterpret_cast` 成 `BatchDesc*`，**不做任何 map 查找**。生命周期完全由调用方保证：`freeBatchID` 之前不能有未完成的 task。

### 3. 完成协议：`markSuccess` / `markFailed` / seal

`transport.h:273-371`。

一次 `markSuccess()` 做四件事：

1. `CompletionCallbackPin pin(task)`：给 `BatchDesc::active_completion_callbacks` 加一，防止回调还没结束就被 `freeBatchID` 删掉 `BatchDesc`。
2. `status = SUCCESS`。
3. `transferred_bytes += length`（relaxed）、`success_slice_count += 1`（acq_rel）。
4. 对 `completion_state` 做一次 `__atomic_add_fetch`（acq_rel），得到 `state`，交给 `finishIfComplete`。

`completion_state` 的打包方式是本子系统最值得记住的细节（`transport.h:265-269`）：

```
bit 63        = sealed（提交方已确认"不会再追加 slice"）
bits 0..62    = 已完成 slice 数
```

`finishIfComplete`（`transport.h:315-371`）只有在 **sealed 且 done == slice_count** 时才发布完成；然后用 `completion_published` 做一次性交换，防止重复发布。发布后：

- 未开 `USE_EVENT_DRIVEN_COMPLETION`：只 `finished_task_count++`，`is_finished` 交给 `getTransferStatus` 的遍历去写。
- 开启 EDC：直接写 `is_finished`，并在最后一个 task 完成时用 `completion_mutex` + `completion_cv` 唤醒等待者。

提交侧由 `MultiTransport` 在 `submitTransferTask` 返回后对仍未 seal 的 task 调用 `sealTaskSubmission`（`src/multi_transport.cpp:315-319`）；只有 `supportsBatchCompletionCounter()` 为 true 的后端才会在提交前 `unsealTaskSubmission`（`multi_transport.cpp:302-303`），另外 `USE_EVENT_DRIVEN_COMPLETION` 下 grouped scatter（`count > 1`）也会 unseal（`multi_transport.cpp:304-306`）。默认 `completion_state` 就是 sealed（`transport.h:462-465`），所以不参与计数机制的后端（TCP、SHM 等）天然按"slice 全部完成即 task 完成"工作。

批量查询还有一个容易被忽略的门槛：`getBatchTransferStatus` 只有在 `counter_eligible == true`、`globalConfig().slice_timeout <= 0` 且 batch 尚无 failure 时才跳过逐 task 遍历；一旦启用 slice 超时，即使 RDMA 也必须逐 task 查询（`multi_transport.cpp:517-529`）。

### 4. `TransferTask` 与 `BatchDesc`

`TransferTask`（`transport.h:453-496`）：`slice_count`、`success_slice_count`、`failed_slice_count`、`transferred_bytes`、`batch_id`、`transport_`、`request`、`request_count`、`slice_list`。析构时把 `slice_list` 里的 slice 还给线程本地缓存（`transport.h:492-495`）。

`BatchDesc`（`transport.h:498-530`）：`batch_size`、`task_list`、`context`（后端私用，如 `NVMeoFBatchDesc`）、`has_failure`、`is_finished`、`status_cached`、`finished_task_count`、`counter_eligible`、`active_completion_callbacks`，EDC 下还有 mutex/cv。

`ThreadLocalSliceCache`（`transport.h:405-451`）：每线程容量 4096 的环形缓存。`allocate()` 命中时复用并置 `from_cache=true`；`deallocate()` 先把 `cleanup_callback` 摘出并清空、再执行回调，避免回调泄漏到下一次使用；缓存已满则直接 `delete` 该 slice 而不是覆盖旧项。

### 5. `MultiTransport` 的路由状态

`include/multi_transport.h:132-143`：`transport_map_`（`std::map<string, shared_ptr<Transport>>`）、`batch_desc_set_`、延迟释放 map 与线程。

延迟释放的存在理由在 `multi_transport.h:58-60`：`freeBatchID` 可能返回 `BatchBusy`（还有 task 未完成，或完成回调的 pin 尚未退完，见 `multi_transport.cpp:177-191`），`MultiTransport` 把这类 batch 放进后台线程、以 100 ms 间隔重试（`multi_transport.cpp:203-232`），对上层呈现为 `BatchCleanupDeferred`；永久性错误会被丢弃而不是无限轮询。

### 6. TCP 后端的核心类型

`include/transport/tcp_transport/tcp_transport.h`：

- `TcpIoPool`（61-150）：N 个 `asio::io_context`，每个一个线程；`nextLaneExecutor()` 轮转分配 lane，`coordinatorExecutor(hash)` 把同一个 peer group 的协调动作固定到某个分片。默认 `MC_NUM_TCP_IO_THREADS=1`（`tcp_transport.cpp:375-379`）。
- `TcpContext`（`tcp_transport_session_impl.h:923-982`）：acceptor 只存在于 shard 0，先尝试 IPv6 双栈（`v6_only(false)`），失败再退回 IPv4；`doAccept` 把每个新连接轮转投到某个 shard 的 `io_context` 上。
- `ConnectionKey`（219-233）：`{host, port}`，hash 用 host 的 hash 异或 port 左移 1。
- `WorkFailureReason` / `FailureCounters`（235-251）：六类失败原因各自的原子计数，用于日志与观测。
- `TcpWorkItem`（253-275）：`slice`、`use_v2`、`continuation`、`admission_deadline`。move-only，`static_assert` 强制（293-300）。
- `TerminalAction`（277-291）：把"要完成的 work + 终态 + 连接是否干净"打包，避免在锁内做 `markSuccess`。
- `ConnectionLane`（330-349）：一个固定 lane，`state`、`connect_stage`、`operation_epoch`、`resolver`、`socket`、`session`、`current`。
- `PeerConnectionGroup`（351-399）：一个 `(host,port)` 的所有 lane + 有界队列 `queue` + `pending_admissions` + 字节配额 + 准入/重试计时器 + 退休状态。
- `ConnectionLaneState`（401-417）：全局配置（`lanes_per_peer`、队列容量、准入超时）、`groups`、`current_key_by_peer`、`retiring_groups`。

`operation_epoch` 是这套状态机的"防迟到"核心：每次状态迁移递增，异步回调必须校验自己拿到的 epoch 仍然匹配，否则视为 stale 直接返回。

### 7. TCP 线协议类型

在 `src/transport/tcp_transport/tcp_transport_session_impl.h`：

- `SessionHeader`（45-49）：`{uint64_t size; uint64_t addr; uint8_t opcode;}`，用 `htole64`/`le64toh` 做小端转换。两侧都用 `sizeof(SessionHeader)` 读写，因此结构体填充也是一致的一部分。
- `kOpcodeV2Flag = 0x80`（118）：v2 请求把 opcode 最高位置 1。
- `kStatusMagic = 0x4D435456 << 32`（126）：高 32 位是 `"MCTV"`，低 32 位是状态码。
- `kStatusOk` / `kStatusAddrRejected`（127-128）。
- `TcpStagingBuffer`（68-82）：GPU 内存走 socket 时的中转缓冲，按需 `resize`。
- `forceLegacyTcpProto()`（136-141）：`MC_TCP_PROTO=1` 时强制 v1。

### 8. SHM 后端的核心类型

`include/transport/shm_transport/shm_transport.h`：

- `AllocatedShmEntry`（153-156）：本进程自己创建的共享内存，`{name, length}`。
- `OpenedShmEntry`（101-107）：本进程映射的**对端**共享内存，`{shm_addr, length, shm_name, pin_count, stale}`。
- `MappingPin`（112-129）：RAII 持有 `OpenedShmEntry` 的一个 pin，析构时 `unpinLocked` 并延迟 `munmap`。
- `RelocateMap`（158-159）：`unordered_map<owner_buffer_addr, shared_ptr<OpenedShmEntry>>`，按 target segment 再分一层。
- `PendingUnmap`（160）：`vector<pair<void*, uint64_t>>`，把 munmap 推迟到锁外执行。
- `relocate_lock_`（190）：保护 `relocate_map_` 的 `RWSpinlock`。
- `shm_path_mutex_` / `shm_path_map_`（193-194）：保护本进程自建对象的 `void* -> AllocatedShmEntry`。
- `kMaxMappingsPerTarget = 32`（197）：每个 target 最多缓存 32 个映射。

名字判别（37-74）：

- `kPosixShmNamePrefix = "/mooncake_"`。
- `isPosixShmName(name)` 取 basename 后判断是否以 `mooncake_` 开头，所以 `/dev/hugepages/mooncake_x` 也命中。
- `isFilesystemShmPath(name)` 要求绝对路径且至少两个段、无 `..`。
- `pathHasDotDotComponent` 专门防 `..` 逃逸。

### 9. GPU IPC 策略对象

`include/transport/nvlink_transport/nvlink_transport.h:33-93` 的 `GpuIpcTransportPolicy` 把厂商差异抽成 10 个虚钩子（5 个纯虚、5 个带默认实现）：

`protocol()`、`displayName()`、`selectStreamDevice()`、`groupTransfersByDevice()`、`requiresStreamDeviceGuard()`、`preserveExistingMetadata()`、`normalizeMemoryLocation()`、`openIpcMemHandle()` / `closeIpcMemHandle()`、`submitBatchCopies()`。

`NvlinkTransport` 的构造函数接受 policy（`nvlink_transport.h:115-116`），`getName()` 返回 `policy_->protocol()`（140 行）。所以 `MusaTransport` 只有 37 行：换一个 policy 就得到一个新协议名（`include/transport/musa_transport/musa_transport.h:26`）。

不要被目录名骗了：`HipTransport`、`MacaTransport`、`HylinkTransport` 都是直接继承 `Transport` 的**独立实现**——Hip/Maca 各自维护 `remap_entries_` 缓存（Hip 另有 `StreamPool`/`EventPool`），Hylink 维护自己的 `OpenedMapping` 表和逐传输的 `hipEvent_t`；只有 MUSA 走策略复用。三种风格并存是历史演进的结果。

### 10. `SharedSegment`

`include/shared_segment/shared_segment.h:71-115`：

- `Create(name, size, options, segment, out_blob)`：阶段一，预留地址窗口；owner 额外分配并导出物理页。
- `Complete(blobs)`：阶段二，收到所有 rank 的定长 blob 后校验并映射。
- `ready()`、`base_addr()`、`size()`、`device_addr()`。
- 私有成员：`name_`、`size_`、`options_`、`fingerprint_`、`alloc_size_`、`base_addr_`、`ready_`、`backend_`。

设计注释（27-51）明确了三个反直觉点：

1. **VA 不要求跨 rank 一致**，peers 只约定字节布局，定位方式是 `local_base + offset`。这样保留的地址空间是 segment 大小而不是 `size × world_size`。
2. **两阶段**是因为唯一可用的 communicator 属于框架（实际是 torch process group），从 C++ 驱动它必须回调 Python。
3. **peers are trusted**：校验的是"配置不一致"，不是"对端撒谎"。

`SharedSegmentBackend`（`src/shared_segment/shared_segment_internal.h:30-67`）六个虚函数：`Granularity`、`CreateOwner`、`ReserveLocal`、`ImportAndMap`、`BackendId`、`DeviceAddr`。

`SegmentBlobHeader`（`shared_segment_internal.h:81-89`）：

```cpp
uint32_t magic; uint16_t version; uint16_t backend_id;
uint32_t rank_id; uint32_t handle_bytes;
uint64_t fingerprint; uint64_t alloc_size;
```

`static_assert(sizeof == 32)`（93 行），头是裸 `memcpy` 进 blob 的，所以布局就是线格式。`kMaxHandleBytes = 128`（按 `aclrtMemFabricHandle` 尺寸定），`kSegmentBlobBytes = 160`（96-97 行）。

### 11. 两个 allocator 的 C ABI

`nvlink-allocator/nvlink_allocator.cpp` 导出：

- `mc_probe_fabric_support(int)` -> `MemoryBackendType`
- `mc_allocator_probe(int)` -> int
- `mc_allocator_malloc(ssize, int, cudaStream_t)`
- `mc_allocator_free(...)`
- `mc_nvlink_malloc` / `mc_nvlink_free`（别名）

`ubshmem-allocator/ubshmem_fabric_allocator.cpp` 导出同样的探测/分配/释放**符号名**，但基于 ACL，且签名不同——`mc_allocator_malloc` / `mc_allocator_free` 只有 `(size, device)` 两个参数，没有 CUDA 版的 `cudaStream_t`：

- `aclrtMallocPhysical` + `aclrtReserveMemAddress` + `aclrtMapMem`（32-70 行）。
- 另有 `mc_probe_ub_fabric_support` / `mc_ub_fabric_malloc` / `mc_ub_fabric_free`（93-113 行）。

消费方是 Python：`mooncake-integration/allocator.py:86-88` 用 `CUDAPluggableAllocator(so_path, "mc_allocator_malloc", "mc_allocator_free")`；`allocator_ascend_npu.py:66-69` 用 `NPUPluggableAllocator`。探测走 `fabric_allocator_utils.py:probe_allocator_backend` 的 ctypes。

---

## 关键流程剖析

### 流程 A：运行期安装与选路

**调用链**

```
TransferEngine::init / TransferEngineImpl::init        (src/transfer_engine_impl.cpp:236 起)
  -> MultiTransport::installTransport(proto, topo)       (src/multi_transport.cpp:567)
       -> new XxxTransport(); transport->install(...)     (multi_transport.cpp:578-740)
  -> （提交时）MultiTransport::selectTransports(entries, transports)  (multi_transport.cpp:746)
       -> selectTransport(request, transport, &reuse_allowed)        (multi_transport.cpp:778)
```

**分步说明**

1. `MC_FORCE_TCP` 最优先：跳过所有探测，只装 TCP 并直接返回（`transfer_engine_impl.cpp:240-255`）。如果编译期没有 `USE_TCP`，直接报错返回。
2. `MC_FORCE_SHM` 在**未**开 `ENABLE_MULTI_PROTOCOL` 时是"只装 SHM 并返回"（262-274）。开了多协议时**故意不返回**，让后面的自动探测继续装 RDMA/TCP，最后由 `maybeInstallShmTransport` 追加 SHM（`transfer_engine_impl.cpp:57-68`）。这一点很容易读反。
3. 自动探测按编译宏分支：昇腾、UBShmem、CXL（只有设置 `MC_CXL_DEV_PATH` 才装）、EFA、UB、MACA、MNNVL/IntraNode-NVLink、其它（RDMA/TCP）（`transfer_engine_impl.cpp:276-539`）。注意昇腾与 EFA 分支会各自先调一次 `maybeInstallShmTransport` 再 `return`；其余路径在末尾统一调用（541 行）。
4. `installTransport` 内部有两条**排他约束**（567-590）：
   - `USE_NCCL_HOST` 下，NCCL 必须是进程内唯一后端，双向检查。
   - `rdma` 与 `rdma_twosided` 不能共存。
5. 同一个 proto 字符串 `"ascend"` 会按宏的 `else if` 顺序映射到三个类：`USE_ASCEND_DIRECT -> AscendDirectTransport`、`USE_ASCEND -> HcclTransport`、`USE_ASCEND_HETEROGENEOUS -> HeterogeneousRdmaTransport`（616-630）。
6. `installTransport` 最后调用 `transport->install(local_server_name_, metadata_, topo)`，失败则 `delete` 并返回 nullptr（737-743）。
7. 选路时，`selectTransports` 逐个 request 调 `selectTransport`，并在"同一 target、该 segment 是单协议、且 `globalConfig().metacache` 开启"时复用上一个 `Transport*`（`multi_transport.cpp:755-773`）。
8. 单协议路径就是查表：`transport_map_[proto]`；`rdma` 找不到但装了 `rdma_twosided` 时重定向（885-897）。
9. 多协议路径（`ENABLE_MULTI_PROTOCOL`，`protocol` 里含逗号）才走 offset 匹配（799-875），细节见下一节。

**多协议选路的三个关键点**（`multi_transport.cpp:807-873`）：

- 遍历目标 segment 的 `buffers`，找覆盖 `entry.target_offset` 的那个；`hip`/`musa`/`shm` 的 buffer 在跨主机时直接 `continue` 跳过（`isLocalIpcReachableTarget` 判定）。这个判定在 `include/multi_transport_locality.h` 里：先剥掉 `:port`、单独处理 `[IPv6]:port` 与裸 IPv6，再对 host 做大小写不敏感比较——直接 `rfind(':')` 会在裸 IPv6 段名上判错，从而错误地丢掉本机 IPC 快路径。
- 同一 buffer 被多协议注册时按硬编码优先级取最高：`hylink 5 > hip/maca/musa/shm 4 > cxl 3 > rdma 2 > tcp 1`；`MC_DISABLE_HYLINK/HIP/MACA/MUSA` 可把它们降为 0（808-821）。
- CXL 的 buffer 起点用 `buffer.offset + desc->cxl_base_addr`，其它协议用 `buffer.addr`（833-838）。这是 CXL 地址空间模型不同的直接体现。

### 流程 B：TCP 一次 WRITE 的完整生命周期（v2）

**调用链**

```
MultiTransport::submitTransfer
  -> TcpTransport::submitTransferTask(task_list)           (tcp_transport.cpp:570)
       -> prepareTransfer(task, request)                   (tcp_transport.cpp:611)
       -> startTransfer(slice, continuation, reuse=true)   (tcp_transport.cpp:718)
            -> enqueuePooledTransfer(...)                  (lane_impl.h:541)
                 -> requestGroupPumpLocked / postGroupPump (lane_impl.h:405,727)
                      -> runGroupPump                     (lane_impl.h:770)
                           -> startLaneSession             (lane_impl.h:1218)
                                -> ClientSession::initiate (session_impl.h:443)
```

**数据面时序（v2 WRITE）**

1. `prepareTransfer` 把 `request.target_offset` 写进 `slice->tcp.dest_addr`，`slice->task = task`，`task.slice_count++`（`tcp_transport.cpp:611-626`）。
2. `startTransfer` 读取目标 `SegmentDesc`，从 `tcp_data_host`/`tcp_data_port` 构造 `ConnectionKey`，并用 `desc->tcp_proto_version >= 2 && !forceLegacyTcpProto()` 决定 `use_v2`（`tcp_transport.cpp:732-764`）。
3. `enqueuePooledTransfer` 在 `lane_state_->mutex` 下找到/创建 `PeerConnectionGroup`。准入策略（`lane_impl.h:635-669`）：
   - 先过期 `pending_admissions`，再尽量把 pending 提升到 `queue`；
   - 字节配额不足 -> 直接 `QUEUE_FULL` 拒绝；
   - 队列未满且无 pending -> 直接入 `queue`；
   - 否则进 `pending_admissions` 并设置 `admission_deadline`；
   - pending 也满 -> 拒绝。
4. `postGroupPump` 用 `asio::post` 把 `runGroupPump` 投到该 group 的 coordinator executor。`pump_epoch` 保证同一时刻只有一个 pump 在跑（`lane_impl.h:405-414`）。
5. `runGroupPump`（`lane_impl.h:793-933`）：
   - 先把 IDLE lane 与 queue 头部配对。配对前会对 socket 做一次 `MSG_PEEK` 读 1 字节：如果对端已经关闭，会得到 `0` 或非 `would_block` 错误，于是把 lane 置为 DISCONNECTED（806-822）。注释明确说明 `is_open()` 只描述本地描述符。
   - 没有可用 lane 时，为 DISCONNECTED 的 lane 启动连接探测；`kMaxConcurrentLaneProbes = 1`，一次只探一个（`lane_impl.h:21,888-908`）。
   - 一轮探测全部失败且没有任何可用 lane 时，把队列整体判失败、进入 1 秒冷却（`kReconnectRoundCooldown`，`lane_impl.h:24`），并用 `retry_timer` 在冷却后开启新一轮（848-874）。
   - 对每个选中的 lane，连接走 `asio::dispatch` 到 lane 自己的 executor 执行 `startLaneConnect`（951-958），会话走同样的 `dispatch` 执行 `startLaneSession`（959-966）。单 shard 时 inline，多 shard 时跨线程。
6. `startLaneSession` 在 group 锁内创建 `ClientSession`，`initiate` 发 header（`lane_impl.h:1227-1274`）。注意注释：Asio 的 initiating function 不会 inline 调完成回调，所以可以在锁内发起。
7. `ClientSession::writeHeader` 成功后：v2 WRITE **同时**启动 `readWriteAck()` 和 `writeBody()`（`session_impl.h:632-636`）。
8. `writeBody` 按 `getChunkSize()`（默认 64 KB，`session_impl.h:20-43`）分块 `async_write`。全部写完后：
   - v1：直接 `finalize(COMPLETED, true)`（855-859）——**这只代表数据离开了发起端 socket buffer**。
   - v2：置 `write_body_done_`，若 ack 已到则完成，否则 `armStatusDeadline()` 等待状态帧（838-854）。
9. 服务端 `ServerSession::readHeader`（`session_impl.h:187-233`）：
   - 用 `header.addr`（对端给的本地目标地址）和 `header.size` 调 `validate_addr_`，校验是否落在本进程已注册 buffer 内。失败时 v2 回 `kStatusAddrRejected`，v1 只能关连接（206-218）。
   - WRITE -> `readBody()`；READ 且 v2 -> 先 `sendStatus(kStatusOk)` 再 `writeBody()`。
10. 服务端 `readBody` 收满最后一块后，v2 先发 `kStatusOk` 再 `start()` 处理下一个请求；v1 直接 `start()`（`session_impl.h:298-308`）。这正是 v2 让 COMPLETED 等价于"目标内存已写入"的关键。
11. 客户端 `readWriteAck` 校验 magic 和状态码；若 ack 先于 body 完成，记录 `write_acked_ok_`，由 body 路径完成 finalize（`session_impl.h:693-748`）。
11b. v2 WRITE 的提前失败路径值得单独记：ack 读到拒绝帧或非法 magic 时调用 `abortWrite()` 直接关 socket，但只要 `write_body_in_flight_` 仍为真就**不**发布终态——必须等 body 的 Asio 完成回调跑完（此时源 buffer 才不再被 Asio 引用）再 `finalize(FAILED)`（`session_impl.h:397-405,601-608,889-911`）。这是"调用方看到终态即可释放源 buffer"这个契约的前提。
12. `ClientSession::finalize` 只执行一次（`terminal_reported_`），取消两个 deadline，然后把终态交给 `handleLaneTerminal`（586-596）。
13. `handleLaneTerminal`（`lane_impl.h:1290-1376`）：
   - 用 `operation_epoch` 判 stale；
   - 先把 socket **关闭**（如果连接不干净），再 `completeTerminalAction` 发布 slice 终态。注释说明了顺序理由：脏协议流必须在调用方看到终态前关闭（1324-1327）。
   - 干净完成且 socket 仍可用 -> lane 回 IDLE，连接可复用；否则 DISCONNECTED，并决定是否进入冷却或立即开新一轮（1342-1361）。
14. `completeTerminalAction` 调 `slice->markSuccess()` / `markFailed()`，再执行 continuation（`lane_impl.h:1378-1400`）。continuation 是 grouped scatter 的顺序推进器。

**grouped scatter 的顺序推进**（`tcp_transport.cpp:628-694`）

`startTransferSequence` 用一个 `Sequence{mutex, slices, next, advancing, resume_requested}` 和 `advance` 闭包实现 trampoline：

- 取下一个 slice，设置 `continuation = advance`，调用 `startTransfer`。
- 若 `startTransfer` 同步失败并立即调用 continuation，`advancing`/`resume_requested` 让控制流回到 `while` 循环而不是递归。
- 若异步完成，本次调用返回，之后由完成回调再次进入 `advance`。

**超时与失败路径**

- `MC_TCP_STATUS_TIMEOUT_SEC`（默认 30s）：v2 等状态帧的上限，超时关 socket（`session_impl.h:415-422,473-496`）。它只覆盖状态帧本身，不覆盖 payload。
- `MC_TCP_PROGRESS_TIMEOUT_SEC`（默认 30s）：整个请求的滑动活性超时，只在真正有进展时刷新（428-439,506-558）。
- lane 侧的超时：`admission_timeout` 默认 1000 ms（`tcp_transport.cpp:335,369-373`）；`expirePendingAdmissionsLocked` 只负责收集过期项（`lane_impl.h:190-203`），调用方再把它们以 `QUEUE_TIMEOUT` 失败（`lane_impl.h:716-717,967-969`）。

### 流程 C：SHM 一次 WRITE

**调用链**

```
MultiTransport::submitTransfer
  -> ShmTransport::submitTransferTask(task_list)        (shm_transport.cpp:1002)
       -> relocateSharedMemoryAddress(dest, len, target_id, &pin)  (shm_transport.cpp:755)
       -> copySlice(task, request, dest_addr)           (shm_transport.cpp:923)
            -> memcpy + slice->markSuccess()
```

**前提：对象是本进程 `allocateSharedMemory` 创建的**

1. `allocateSharedMemory(length, opt)`（`shm_transport.cpp:459-484`）：
   - 若 `use_hugepage`，先 `prepareHugetlbfsAllocate` 校验挂载点、页大小与长度对齐（273-309）；
   - 循环最多 8 次生成随机名字 `randomShmPath(dir)`；
   - `createSharedMemory` 用 `O_CREAT|O_EXCL` 打开（POSIX 用 `shm_open`，hugetlbfs 用 `open`），`ftruncate` 到 size，`mmap`，记录到 `shm_path_map_`（397-453）。
2. `registerLocalMemory`（507-560）：只接受 `shm_path_map_` 里的**精确基址**。传子区间或与已有分配重叠会返回 `ERR_INVALID_ARGUMENT`；长度超过分配长度也报错。随后写 `BufferDesc{addr, length, name, shm_name}` 并发布。
3. 提交时 `relocateSharedMemoryAddress`（755-921）做地址翻译：
   - 从 `SegmentDesc` 里找覆盖 `dest_addr` 的、且 `shm_name` 是 mooncake 对象名（`isPosixShmName`，POSIX 或 hugetlbfs）的 buffer；找不到返回 `AddressNotRegistered`（`shm_transport.cpp:771-782`）。
   - 在 `relocate_map_[target_id]` 里按 `shm_name` + 地址区间命中缓存（`tryResolve`，599-618），再在写锁下对 `!stale` 的项加 pin（784-807）。
   - **缓存命中后还要 `shmObjectExists(buffer->shm_name)` 再探一次名字**（811）。因为 `mmap` 在 `shm_unlink` 之后依然有效；如果对端 free+realloc 了，本端会静默写进一个孤儿对象。
   - 名字不存在：unpin 旧项，继续走重新映射。
   - 未命中：在写锁下 `pruneAndCapLocked`，找是否有同 `shm_name` 的旧映射可复用；复用前同样校验名字存在。
   - 仍不行：`mapSharedMemory` 打开并映射**整个文件**（按 `fstat.st_size`，不是 `BufferDesc.length`，因为 hugetlbfs 必须页对齐，`shm_transport.cpp:198-207`），再在写锁下安装。如果并发安装者赢了，输者把自己刚 mmap 的地址放进 `pending` 等锁外 `munmap`（881-907）。
   - 最终地址换算：`dest_addr = requested_addr - buffer->addr + shm_addr`（909-910）。
4. `attempt` 跑两遍（915-920）：第一遍用缓存元数据；如果返回 `Status::Memory`（说明缓存描述指向的对象没了），强制 `force_update=true` 重新拉一次 `SegmentDesc` 再试。
5. `copySlice`（923-952）：分配 slice，填 `slice->local.dest_addr`，按 opcode 选方向，`memcpy`，`markSuccess()`。
6. `MappingPin` 在 `submitTransferTask` 函数结束时析构，`unpinLocked` 把 pin 减一；若是最后一个 pin 且 `stale`，把映射放进 `PendingUnmap`，**在释放 `relocate_lock_` 写锁之后** `flushUnmaps` 才 `munmap`（`shm_transport.cpp:634-650,733-747`）。

**为什么必须延迟 munmap**

如果在写锁内直接 `munmap`，另一个线程可能已经通过 `tryResolve` 拿到同一个 `OpenedShmEntry` 并正在 `memcpy`，映射会被抽走。`MappingPin` + 延迟 unmap 是唯一能同时保证"缓存可裁剪"和"memcpy 期间映射有效"的组合。

**孤儿对象与 hugetlbfs 的安全边界**

- `shmObjectExists` 对 hugetlbfs 路径走 `openHugetlbfsPeerFd`：校验绝对路径、无 `..`、basename 是 `mooncake_`、`realpath` 后仍在 hugetlbfs、`O_NOFOLLOW` 打开、再次 `fstatfs` 校验（`shm_transport.cpp:89-136`）。这防的是符号链接和路径逃逸。
- `resolveHugetlbfsMount` 还要求 `f_bsize == hugepage_size`（235-271），避免把 1GB 请求分配到 2MB 挂载点。

### 流程 D：SharedSegment 的两阶段创建

**调用链**

```
Python: create_shared_segment(...)          (mooncake-integration/shared_segment.py)
  -> _CppSharedSegment.create(...)          (mooncake-integration/transfer_engine/shared_segment_py.cpp:36)
       -> SharedSegment::Create             (src/shared_segment/shared_segment.cpp:242)
  -> torch all_gather(blob)
  -> segment.complete(blobs)                (shared_segment_py.cpp:59)
       -> SharedSegment::Complete           (src/shared_segment/shared_segment.cpp:301)
```

**阶段一：Create**

1. `ValidateOptions`：`world_size != 0`，`rank_id/owner_rank < world_size`，`host_register` 必须配 `mmap=true`（65-80）。
2. 尺寸限制 `(0, 1<<46]`（250-253）。
3. `SelectSharedSegmentBackend(options.mmap)`：`mmap=true -> "mmap"`；否则 `USE_ASCEND_DIRECT -> "ascend"`，其他构建返回空（41-50）。
4. 计算 `fingerprint = FNV(name, size, world_size, owner_rank, mmap, host_register)`（145-154）。**不包含 rank_id**，因为各 rank 的 rank_id 本来就不同。
5. `alloc_size = AlignUp(size, backend->Granularity(options))`。
6. owner：`CreateOwner(alloc_size, options, base_addr, handle)`；非 owner：`ReserveLocal(...)`（276-282）。
7. `EncodeSegmentBlob`：固定 160 字节，头 32 字节 + handle 最多 128 字节（156-174）。只有 owner 的 handle 非空。

**阶段二：Complete**

0. 入口先拒绝重复调用：`ready_` 已为 true、或 `backend_` 为空（不是 `Create` 返回的对象）都会直接报错（`shared_segment.cpp:302-309`）。
1. `ExtractOwnerHandle` 检查 `blobs.size() == world_size`（116-120）。
2. 对每个 rank 调 `CheckPeerBlob`（82-109）：解码、magic/version、`rank_id` 必须等于下标、`backend_id` 一致、`fingerprint` 一致、`alloc_size` 一致。
3. 取出 `owner_rank` 的 handle；空则报错。
4. 非 owner 调 `ImportAndMap(alloc_size, options, owner_handle)`（318-323）。
5. `ready_ = true`。

**mmap 后端细节**（`src/shared_segment/mmap_shared_segment.cpp`）

- owner：`memfd_create("mcss", MFD_CLOEXEC)` + `ftruncate` + `ReserveAlignedWindow`（按 THP PMD 对齐）+ `mmap(MAP_FIXED)` + `MADV_HUGEPAGE` + 每 PMD 触碰一字节预缺页 + `MADV_DONTFORK`（196-228）。handle 是字符串 `/proc/<pid>/fd/<fd>`（230-235）。
- 非 owner：先 `ReserveAlignedWindow` 占位（365-377）；`Complete` 时 `open` 该 `/proc/<pid>/fd/<fd>`，再用 `MAP_FIXED` 映射进自己预留的窗口（379-420）。
- 这意味着 **owner 进程必须活得比所有 peer 久**；否则 `/proc/<pid>/fd/<fd>` 失效，peer 无法导入。
- `MmapSharedSegmentBackend::DeviceAddr()` 返回 `ascend_dev_ptr_`，只有 `host_register=true` 在昇腾上才有值（310-312）。

**昇腾后端细节**（`src/shared_segment/ascend_shared_segment.cpp`）

- `CreateOwner`：`adxl::AdxlEngine::MallocMem(MEM_HOST, size, &addr)` + `ExportToShareableHandle`（169-204）。
- `ReserveLocal`：A3 SoC 优先尝试 `aclrtReserveMemAddressNoUCMemory`（固定 78TB 起始 VA），不支持再回退普通 `aclrtReserveMemAddress`（206-237）。
- `ImportAndMap`：`aclrtMemImportFromShareableHandleV2` + `aclrtMapMem` + `GrantDeviceAccess`（239-270）。
- 弱符号 `aclrtReserveMemAddressNoUCMemory`（45-47）让老运行时仍能加载；`CreateAscendSharedSegmentBackend` 又用 `IsAdxlExportShareableHandleAvailable`（52-54）在运行时缺少 `ExportToShareableHandle` 时直接不创建后端（293-299），而不是在加载期失败。

---

## 与其他模块的边界

### 依赖谁

| 依赖 | 用途 | 证据 |
|---|---|---|
| `TransferMetadata` | `SegmentDesc`/`BufferDesc` 的发布与查询、握手 daemon | `tcp_transport.cpp:399-431`、`shm_transport.cpp:362-395` |
| `Topology` | 只被 RDMA/EFA 等真正用；SHM/TCP 基本忽略 | `shm_transport.cpp:365`、`tcp_transport.cpp:401` |
| `asio` | TCP 的 io_context、socket、timer、resolver | `tcp_transport.h:38-43` |
| `yalantinglibs` | TENT/控制面 RPC；TCP header 也引入 `coro_io` | `tcp_transport.h:49` |
| CUDA/HIP/MUSA/MACA 运行时 | GPU IPC、staging、fabric 分配 | `cuda_alike.h`、`nvlink_transport.cpp:980` |
| libfabric / libcxi / NCCL / FlagCX / HCCL / URMA | 对应后端 | 各 CMakeLists |
| cuFile | NVMe-oF | `nvmeof_transport.cpp` |
| 内核 | shm_open/hugetlbfs/memfd/mmap、`/dev/daxX.Y` | `shm_transport.cpp:17-24`、`cxl_transport.cpp:33-36` |
| torch / pybind11 | `shared_segment` 与 allocator 的消费方 | `shared_segment.py`、`allocator.py` |

### 被谁依赖

- **`TransferEngineImpl`**：决定运行时装哪些后端（`transfer_engine_impl.cpp:236-541`）。
- **`MultiTransport`**：所有提交/状态查询的入口。
- **`mooncake-store`**：`TransferSubmitter` 用 `engine_.isTcpOnly()` 决定是否走同进程 memcpy 快路径（`mooncake-store/src/transfer_task.cpp:1066`）。`MultiTransport::isTcpOnly()` 会先减掉 `shm` 再判断（`multi_transport.cpp:977-983`），所以 `tcp+shm` 仍算"只有 TCP"。
- **Python 绑定**：`shared_segment`、`allocator`、`RpcInterface`。

### 接口契约

1. **`submitTransferTask` 的同步性**：`include/transport/transport.h:551-556` 明确要求"实现必须在返回前同步发布所有 slice；返回后不得再追加"。这是 `MultiTransport` 能在返回后 `sealTaskSubmission` 的前提。
2. **`markSuccess`/`markFailed` 每个 slice 恰好一次**（`transport.h:271-272`）。终端发布是单次的，pin 协议依赖这一点。
3. **`freeBatchID` 只能在所有 task `is_finished` 且回调 pin 归零后**（`transport.cpp:42-65`）。
4. **`registerLocalMemory` 是广播语义**：`TransferEngineImpl::registerLocalMemory` 逐个 transport 调，任一失败就回滚已注册的（`transfer_engine_impl.cpp:796-817`）。所以 `tcp+shm` 时同一块内存会在两个 transport 各登记一次；SHM 对非自建指针返回 0（`shm_transport.cpp:543`），不会误报。
5. **多协议 segment 的 `protocol` 是逗号分隔串**。SHM 在 `ENABLE_MULTI_PROTOCOL` 下追加 `,shm`，否则直接覆盖（`shm_transport.cpp:375-390`）。

### 数据怎么流过去

- **TCP**：控制面（SegmentDesc）只提供 `tcp_data_host`/`tcp_data_port`/`tcp_proto_version`；数据面是独立的 TCP 连接，`SessionHeader{size, addr, opcode}` 里携带目标进程内的虚拟地址，服务端按自己的注册表校验后直接写内存。
- **SHM**：控制面提供 `shm_name`；数据面是本进程对同名对象的 `mmap`，地址换算后 `memcpy`。
- **GPU IPC**：控制面提供序列化后的 `cudaIpcMemHandle_t` 或 `CUmemFabricHandle`；数据面是导入后的设备 VA + `cudaMemcpyAsync`。
- **CXL**：控制面提供 `cxl_base_addr` 与 `BufferDesc.offset`；数据面是本进程 `/dev/daxX.Y` 的 mmap + `memcpy`。
- **SharedSegment**：控制面是固定 160 字节 blob 的 all-gather；数据面是各 rank 自己映射出来的 VA，谁都不需要知道别人的 VA。

---

## 设计取舍与易踩坑点

1. **TCP v1 的 COMPLETED 是"假成功"**。v1 只保证数据进了发起端内核 socket buffer；对端可能还有几 MB 在途，服务端地址校验拒绝的请求也会报成功。测试文件把这一点写得很直白：`tests/tcp_write_visibility_test.cpp:17-22` 记录了实测 166/400 次撕裂、最坏整块 2.4 MB 未送达。v2 用 `kOpcodeV2Flag` + 8 字节状态帧修正；协商不是握手，而是读对端 `SegmentDesc::tcp_proto_version`（`tcp_transport.cpp:453-471,763`）。

2. **陈旧元数据会让 v2 发起端打到 v1 服务端**。代码用 `kStatusMagic` 高 32 位让第一帧快速失败（`session_impl.h:118-131`），并明确承认仍有 `2^-64` 的 in-band 混淆残余风险。另外 v1 服务端对未知 opcode 按 READ 处理并立即流 payload，所以对端可能一直不关连接；`statusFrameTimeoutSec()` 的 30 秒兜底就是为这个场景准备的（`session_impl.h:406-422`）。

3. **TCP 的真正实现不在 `.cpp` 里**。`tcp_transport.cpp` 只有 843 行，`tcp_transport_lane_impl.h`（1674 行）和 `tcp_transport_session_impl.h`（982 行）是被它从 `namespace mooncake` 内部 `#include` 进来的实现片段（`tcp_transport.cpp:268,716`）。只 grep `.cpp` 会漏掉约 2600 行，包括整个线协议和 lane 状态机。

4. **lane 状态机的复杂度全部来自"慢 peer 隔离"**。固定 lane 数 + 有界队列 + 准入超时 + 单探测 + 1 秒轮次冷却 + group 退休，这几样组合起来才保证：一个慢/挂的 peer 不会耗尽本进程的 fd 或内存，也不会拖住其它 peer。代价是状态机很难读，必须靠 `operation_epoch` / `pump_epoch` / `retry_epoch` / `admission_epoch` 四个 epoch 判断异步回调是否过期。读的时候建议先只看 `enqueuePooledTransfer -> runGroupPump -> handleLaneTerminal` 这条主线。

5. **lane 复用前要做 `MSG_PEEK`**。`asio::socket::is_open()` 只说明本地描述符没关；空闲连接可能已被对端关闭。`runGroupPump` 在把工作交给 IDLE lane 前偷看 1 字节（`lane_impl.h:806-822`），这是避免"把请求写进半关闭连接"的关键。

6. **脏连接的关闭顺序**。`handleLaneTerminal` 先关 socket、再发布 slice 终态（`lane_impl.h:1324-1327`）。如果反过来，调用方可能在协议流尚未清理时释放源 buffer 或复用连接。

7. **SHM 的孤儿对象陷阱**。`mmap` 在 `shm_unlink` 之后仍然有效，所以缓存命中不能只信指针；必须 `shmObjectExists` 再探一次名字（`shm_transport.cpp:808-817`）。否则对端 free+realloc 后，本端会静默写进一个已经 unlink 的对象。

8. **SHM 的 munmap 必须在写锁外**。`queueUnmap` 在 `pin_count > 0` 时只置 `stale`，否则把地址放进 `PendingUnmap`；`flushUnmaps` 在 `RWSpinlock::WriteGuard` 析构后执行（`shm_transport.cpp:620-650,733-747`）。这是 `MappingPin` 存在的唯一理由。

9. **SHM 的 `mapSharedMemory` 按 `st_size` 映射**，不是按 `BufferDesc.length`（`shm_transport.cpp:198-207`）。hugetlbfs 要求页对齐；用逻辑长度映射会 `munmap` 失败或截断。

10. **`shm_transport` 是唯一无条件编译的后端**（`src/transport/CMakeLists.txt:44-45`），但"编得进"不等于"装得上"。默认只有 `MC_FORCE_SHM` 或显式 `installTransport("shm")` 才装；开了 `ENABLE_MULTI_PROTOCOL` 时 `maybeInstallShmTransport` 也只在 `MC_FORCE_SHM` 下追加（`transfer_engine_impl.cpp:57-68`）。

11. **未开多协议时，SHM 会吞掉同进程的 RDMA/TCP**。`ShmTransport::install` 把 `desc->protocol` 整体替换成 `"shm"`，只打一条 WARNING（`shm_transport.cpp:381-390`）。虽然它保留了旧的 `buffers`，但选路只看 `protocol`，等于废掉其它后端。这是"先装 TCP 再装 SHM"最容易踩的坑。

12. **`isTcpOnly()` 会先把 `shm` 减掉**（`multi_transport.cpp:977-983`）。所以 `tcp+shm` 的引擎仍会被 Store 判定为"只有 TCP"，从而启用同进程 memcpy 快路径。注释明确说 POSIX SHM 是 intra-node、不改变这个分类。

13. **CXL 的地址空间模型与其它后端不同**。`BufferDesc` 里存的是相对 `cxl_base_addr` 的 `offset`，不是绝对 VA；`BufferDesc.addr` 在 `CxlTransport::registerLocalMemory` 里**根本没有赋值**（`cxl_transport.cpp:242-265`）。单协议选路不使用 `addr`，多协议选路专门用 `buffer.offset + desc->cxl_base_addr`（`multi_transport.cpp:833-838`），`getBaseAddr()` 也只对 CXL 返回非 0（`multi_transport.cpp:992-1001`）。另外 `CxlTransport::submitTransfer` 从 `getSliceCache()` 借出 Slice 后，**没有** `task.slice_list.push_back(slice)`（对比 `submitTransferTask` 的 `cxl_transport.cpp:388`）。全部回收路径只有 `TransferTask::~TransferTask()` 遍历 `slice_list` 归还线程本地缓存（`include/transport/transport.h:492-495`），所以这条路径上的 Slice 永远不会归还，按当前实现是逐请求泄漏；`BufferDesc.addr` 之外，这是 CXL 后端第二个值得修的遗留问题。改动前仍应先用 `tests/cxl_transport_test.cpp` 确认没有依赖这一行为的用例。

14. **`BufferDesc::addr` 未初始化还影响 `BufferRangeIndex`**。`addLocalMemoryBuffer` 会 `rebuildBufferRangeIndex()`（`include/transfer_metadata.h:145`），而索引按 `buffers[i].addr` 排序（`include/buffer_range_index.h:63-80`）。CXL 的未初始化 `addr` 会进入这个排序。CXL 数据路径不依赖该索引，但这是一个值得修的可疑点。

15. **两条安装排他约束写在 `installTransport` 开头**，不在文档里：NCCL 必须唯一（`multi_transport.cpp:569-576`），`rdma` 与 `rdma_twosided` 互斥但对外都发布 `protocol="rdma"`，所以选路有一段重定向（885-891）。NCCL 还必须在 `registerLocalMemory` **之前**安装，否则 peer buffer 顺序不确定（`include/transport/nccl_transport/nccl_transport.h:22-27`）。

16. **`"ascend"` 一个 proto 映射三个类**，取决于宏的 `else if` 顺序（`multi_transport.cpp:616-630`）；`ascend_transport/` 目录下并列 4 个子实现 + `ascend_allocator.cpp`，不是一个后端。`HcclTransport::getName()` 返回 `"hccl"`，但注册用的 proto 是 `"ascend"`。

17. **路径名与类名大面积不一致**：
    - `src/transport/sunrise_link/` -> `SunriseLinkTransport`，proto `"sunrise_link"`，CMake target `sunrise_link_transport`，头文件在 `include/transport/sunrise_link_transport/`。
    - `kunpeng_transport/` -> 类 `UbTransport`，源文件 `ub_transport.cpp`，proto `"ub"`。
    - `nccl_transport/` -> `NcclHostTransport`。
    - `intranode_nvlink_transport/` -> `IntraNodeNvlinkTransport`，`getName()` 返回 `"nvlink_intraNode"`，但 `installTransport` 用的 proto 是 `"nvlink_intra"`（`multi_transport.cpp:632-636`）。
    - `flagcx` -> `FlagCxTransport`；`nvmeof` -> `NVMeoFTransport`。

18. **`rpc_communicator/` 不是 Transport 子类**。它基于 yalantinglibs `coro_rpc`，只被 Python 绑定使用（`include/transport/rpc_communicator/rpc_communicator.h`、`rpc_interface.h`）。把它当传输后端讲是错的。

19. **两个 allocator 的 `.so` 不走普通 CMake 编译目标**。`fabric_allocator.cmake` 的 `add_fabric_allocator_build_target` 建的是 `add_custom_target`，用 `add_custom_command` 调 `build.sh` 产出 `.so`，然后让该自定义目标依赖 `transfer_engine`（`add_dependencies(${FAB_TARGET_NAME} transfer_engine)`，方向是 allocator 依赖引擎）。在 C++ 里 grep `dlopen` 找不到它们，容易误判成死代码；消费方是 Python 的 `CUDAPluggableAllocator` / `NPUPluggableAllocator`（`mooncake-integration/allocator.py:86-88`、`allocator_ascend_npu.py:66-69`）。

20. **`mc_nvlink_malloc` 与 `mc_allocator_malloc` 是别名**（`nvlink_allocator.cpp:176-178`），但 Python 只用后者。`cuMemCreateTryFabric` 在 FABRIC 不被支持时清标志位重试（6-18 行），这是从 NCCL `allocator.cc` 抄来的兼容逻辑；`ProbeAllocatorBackend` 会真的分配 4096 字节试一次（24-53 行），所以探测本身有副作用（分配/释放一次）。

21. **`SharedSegment` 故意不要求各 rank VA 一致**（`shared_segment.h:33-37`）。定位方式是 `local_base + offset`；保留的地址空间是 segment 大小而不是 `size × world_size`。交换的 blob 是固定 160 字节，一致性靠 FNV fingerprint 而不是序列化框架。两阶段拆开的唯一原因是 communicator 属于框架（torch process group），从 C++ 驱动它得回调 Python。

22. **`SharedSegment` 的校验前提是"peers are trusted"**（`shared_segment.h:49-51`）。`CheckPeerBlob` 只检查配置一致性（rank id、backend id、fingerprint、alloc size），不防恶意对端。owner 必须比所有 peer 活得久，否则 peer 的映射失效。

23. **`maybeInstallShmTransport` 的行为随 `ENABLE_MULTI_PROTOCOL` 改变**：无多协议时它是 no-op，有多个协议时只在 `MC_FORCE_SHM` 下追加（`transfer_engine_impl.cpp:57-68`）。读这段一定要先把宏分支在纸上画出来。

24. **`NVMeoFTransport` 不支持 `MultiTransport` 的批量路径**：`submitTransferTask` 直接把 task 标记完成并返回 `NotImplemented`（`nvmeof_transport.cpp:131-141`）。它有自己的 `NVMeoFBatchDesc` 和 `allocateBatchID` 重写（`include/transport/nvmeof_transport/nvmeof_transport.h:41-65`）。所以它更像"另一套 batch 模型"，而不是标准后端。

25. **hugetlbfs 的崩溃遗留比 tmpfs 更糟**。SIGKILL 不会 unlink；hugetlbfs 文件会一直占用大页直到被删或重启（`include/common.h:84-95` 的 `SharedMemoryOptions` 注释）。没有启动清理器，因为多进程共享同一挂载点，清 `mooncake_*` 会误删活着的 peer。

---

## 精读清单

按这个顺序读，每一步都给出"应该看懂什么"：

1. **`include/transport/transport.h:110-371`**
   看懂 `Slice` 的 union、`SliceTaskPtr`、`markSuccess`/`markFailed`、seal 位、`CompletionCallbackPin`。这是所有后端的公共语法。读完后问自己：一个 slice 完成后，`BatchDesc` 什么时候知道？谁负责 `is_finished`？

2. **`src/transport/transport.cpp` 全文（74 行）**
   看懂 `allocateBatchID` 为什么把指针当 ID、`freeBatchID` 的两道检查（`is_finished` + `active_completion_callbacks`）、`CONFIG_USE_BATCH_DESC_SET` 的两种模式。

3. **`src/multi_transport.cpp:249-328` 与 `:567-720`**
   看懂 Task 是怎么按 transport 分组的、`counter_eligible` 与 seal 的关系、`installTransport` 的宏分发表和两条排他约束。

4. **`src/multi_transport.cpp:746-898` 与 `:901-1001`**
   看懂单协议查表、多协议按 buffer 覆盖选路、优先级表、CXL 特例、`isLocalIpcReachableTarget` 的门控、`isTcpOnly` 与 `getBaseAddr`。

5. **`include/transport/tcp_transport/tcp_transport.h` 全文**
   先只建立类型地图：`TcpIoPool`、`ConnectionLane`、`PeerConnectionGroup`、`ConnectionLaneState`、`TcpWorkItem`、`TerminalAction`。不要急着读实现。

6. **`src/transport/tcp_transport/tcp_transport.cpp:315-471`**
   看懂所有 `MC_TCP_*` / `MC_NUM_TCP_IO_THREADS` 的默认值和上限、`install` 如何分配数据端口并发布 `tcp_proto_version=2`。

7. **`src/transport/tcp_transport/tcp_transport.cpp:544-694`**
   看懂 `prepareTransfer`、`submitTransferTask` 的 task_group 合并条件、`startTransferSequence` 的 trampoline。

8. **`src/transport/tcp_transport/tcp_transport_session_impl.h:20-141`**
   看懂 `getChunkSize`、`SessionHeader`、v1/v2 的设计动机、`kStatusMagic`、`forceLegacyTcpProto`。

9. **`tcp_transport_session_impl.h:146-367`（ServerSession）与 `:370-921`（ClientSession）**
   对照读：服务端 `readHeader -> validate -> readBody/writeBody`，客户端 `writeHeader -> readWriteAck + writeBody`。重点看 v2 的 ack 时序、两个 deadline、`finalize` 的单次性。

10. **`src/transport/tcp_transport/tcp_transport_lane_impl.h:541-975`**
    看懂 `enqueuePooledTransfer` 的三级准入、`postGroupPump`/`runGroupPump` 的 epoch、IDLE lane 的 `MSG_PEEK`、探测上限、冷却与 `retry_timer`。

11. **`tcp_transport_lane_impl.h:1218-1400` 与 `:1468-1670`**
    看懂 `startLaneSession` 的锁内发起、`handleLaneTerminal` 的 stale 判断与关闭顺序、`completeTerminalAction` 的 continuation、shutdown 的"先停 pool 再清理"屏障。

12. **`include/transport/shm_transport/shm_transport.h` 全文 + `src/transport/shm_transport/shm_transport.cpp:41-309`**
    看懂命名规则、hugetlbfs 的安全校验、`mapSharedMemory` 为什么按 `st_size` 映射。

13. **`shm_transport.cpp:599-753`**
    看懂 `tryResolve`、`queueUnmap`/`unpinLocked`/`flushUnmaps`、`pruneAndCapLocked`、`MappingPin`。这是 SHM 的生命周期核心。

14. **`shm_transport.cpp:755-921`**
    逐行读 `relocateSharedMemoryAddress` 的两遍 attempt、缓存命中后的 `shmObjectExists`、并发安装时输者的处理、最终地址换算。这是整个子系统最容易读错也最值得读的函数。

15. **`shm_transport.cpp:923-1063`**
    看懂 `copySlice` 的 READ/WRITE 方向、`submitTransfer` 与 `submitTransferTask` 的差异（前者在重定位阶段失败时直接返回，尚未创建 task；后者会给所有 task 调 `failSlice`，包括重定位成功的那些）。

16. **`include/transport/nvlink_transport/nvlink_transport.h:29-192` + `src/transport/nvlink_transport/nvlink_transport.cpp:210-245,570-655,980-1080,1319-1432`**
    看懂策略对象如何把厂商差异抽走、`supportFabricMem` 的两条路径（`cudaIpcGetMemHandle` vs `cuMemExportToShareableHandle`）、`relocateSharedMemoryAddress` 的 `remap_entries_` 缓存。

17. **`include/transport/musa_transport/musa_transport.h`**
    验证"37 行就是一个后端"这件事，理解策略复用的收益。

18. **`src/transport/cxl_transport/cxl_transport.cpp:110-266,324-407`**
    看懂 CXL 的 offset 地址模型、`cxlMemcpy` 的边界检查、`registerLocalMemory` 的 `BufferDesc.addr` 缺失。带着第 13、14 条坑去读。

19. **`include/shared_segment/shared_segment.h` + `src/shared_segment/shared_segment_internal.h`**
    看懂两阶段设计、VA 不一致的取舍、`SegmentBlobHeader` 的线格式、`kMaxHandleBytes` 的由来。

20. **`src/shared_segment/shared_segment.cpp:32-326`**
    看懂 `SelectSharedSegmentBackend`、`ValidateOptions`、`CheckPeerBlob`、`ComputeSegmentFingerprint`、`Create`/`Complete`。

21. **`src/shared_segment/mmap_shared_segment.cpp:131-228,339-420`**
    看懂 memfd + THP + 对齐预留 + `/proc/<pid>/fd/<fd>` handle 的导入流程。

22. **`src/shared_segment/ascend_shared_segment.cpp:169-270`**
    看懂 `MallocMem`/`ExportToShareableHandle`/`ImportFromShareableHandleV2` 的对称关系，以及 A3 的 NoUCMemory 预留策略。

23. **`mooncake-integration/shared_segment.py:80-200`**
    看懂布局如何折叠进 name（fingerprint）、all-gather 如何驱动两阶段、`tensors()` 如何按 `local_base + offset` 生成零拷贝视图。

24. **`nvlink-allocator/nvlink_allocator.cpp` 全文 + `mooncake-integration/allocator.py:24-90`**
    看懂 `cuMemCreateTryFabric` 的降级、三态 probe、`CUDAPluggableAllocator` 的调用方式。

25. **`src/transport/CMakeLists.txt` + `fabric_allocator.cmake` + `nvlink-allocator/CMakeLists.txt`**
    最后读构建，把"哪些后端编得进、哪些 .so 是 build.sh 产出的"对上号。

---

# 05 TENT 新一代传输引擎

> 本章只讲 `mooncake-transfer-engine/tent/` 以及它与旧 Transfer Engine（下称 TE）的接缝。
> 所有行号基于本章写作时的 `main`（`67df8c41`）。文中 `tent/...` 形式的路径相对于 `mooncake-transfer-engine/`，其余路径相对于仓库根目录 `D:/xiaojun/work/Mooncake`。

## 定位与职责

TENT（Transfer Engine NEXT）是 Mooncake 经典 TE 的继任者。它解决的核心问题是：**在异构、会抖动的集群里，应用的传输代码不应该绑定到某个具体后端**。

经典 TE 的模型是「构造 → `init` → `installTransport("rdma", ...)` → 注册内存 → 提交 batch → 轮询」。后端在初始化阶段就被固定下来，多网卡 striping 也是静态的。一旦某条链路退化，应用只能看到变慢或失败。TENT 把这三件事整体搬进运行时：

1. **动态选路**：应用只提交「搬什么」（`Request`：opcode / source / target / offset / length），不指定「怎么搬」。每个请求在提交时由 `TransportSelector` 从策略表里选出一个 transport。
2. **细粒度调度**：大传输被切成 slice，按遥测（完成时间、队列深度、带宽估计）分配到多条 lane/QP 上（见 `docs/source/design/tent/slice-spraying.md`、`deadline-scheduling.md`）。
3. **运行时容错**：跨 transport 的 failover 由引擎内部完成，应用侧的 `getTransferStatus` 不会因为「还有健康路径」而返回 FAILED（见 `docs/source/design/tent/failover.md`）。

在 Mooncake 整体中的位置：TENT 是**数据面搬砖工**，向上被 Store / Reshard / RL 训练侧调用（原生 C++、C API、Python `tent` 模块三种入口），向下依赖网卡、GPU 驱动、元数据服务。它不负责数据布局、副本策略、对象语义——这些在 `mooncake-store/`。两套引擎在同一进程、同一仓库里并存，靠 `MC_USE_TENT`/`MC_USE_TEV1` 切换（见「与其他模块的边界」）。

一句话概括架构：**`TransferEngineImpl` 是唯一的编排者，`transport_list_` 是它的能力表，`Transport` 只负责搬字节，`ControlService` 是控制面/RPC 门面（段描述交换、bootstrap、通知；TCP 的 `SendData`/`RecvData` 也经它）。**

## 目录结构速览

| 路径 | 职责 |
|---|---|
| `tent/include/tent/transfer_engine.h` | 公共 API：`extern "C"` 的 `tent_*` + C++ 类 `mooncake::tent::TransferEngine` |
| `tent/include/tent/common/types.h` | `TransportType`、`Request`、`Notification`、`TransferStatusEnum`、`MemoryOptions` |
| `tent/include/tent/common/config.h` | `Config`：单棵 `nlohmann::json` + `'/'` 路径键；`ConfigHelper::loadFromEnv` |
| `tent/include/tent/common/config_lifecycle.h` | 配置生命周期视图：`BootstrapConfig` / `RuntimeConfig` / `RuntimeConfigSnapshot` / `TentConfigBundle` |
| `tent/include/tent/runtime/transfer_engine_impl.h` | 引擎核心声明：`TaskInfo`、分片 batch 锁、`DeferredStageTeardown` |
| `tent/src/runtime/transfer_engine_impl.cpp` | 引擎实现（3837 行）：装配、提交、轮询、failover、notification、batch 生命周期 |
| `tent/src/runtime/transport_loader.cpp` | `loadTransports()`：编译期 `USE_*` × 运行期 `transports/<name>/enable` 双重门控 |
| `tent/include/tent/runtime/transport.h` | `Transport` 抽象契约（含全有或全无提交约定） |
| `tent/include/tent/runtime/transport_selector.h` / `src/runtime/transport_selector.cpp` | 配置驱动选择器：policy → 有序 transport 候选表 + device_mask |
| `tent/include/tent/runtime/control_plane.h` / `src/runtime/control_plane.cpp` | 控制面：`ControlClient`（RPC 客户端）/ `ControlService`（RPC 服务端） |
| `tent/include/tent/runtime/topology.h` / `src/runtime/topology.cpp` | 本地拓扑：NIC 列表、内存列表、3 级 `device_list` 优先级 |
| `tent/include/tent/runtime/segment_manager.h` + `segment_registry.h` + `segment_tracker.h` | 段描述缓存、本地 buffer 注册表、远端拉取与失效 |
| `tent/include/tent/runtime/proxy_manager.h` | 经主机内存的 staging 中转（头文件标注 "Beta version -- use with own risk"） |
| `tent/include/tent/runtime/admission_queue.h` / `deadline_mlu.h` / `progress_worker.h` | opt-in 的准入队列、deadline 调度、事件驱动 progress |
| `tent/include/tent/device_plugin.h` | device plugin C ABI：`device_plugin_t` + `tent_register_device_plugin` |
| `tent/src/runtime/memory_prober.cpp` | `MemoryProber`：dlopen `*.so` 加载 device plugin（当前无调用方，见「易踩坑点」） |
| `tent/src/platform/*` | 编译期选出的 `Platform` 实现（CUDA/ROCm/Ascend/MLU/XPU/TPU/Sunrise/CPU） |
| `tent/src/transport/*` | 18 种后端实现；`rdma/`、`ub/` 最重，`hp_tcp/` 次之 |
| `tent/src/metastore/*` | redis / http / etcd 三种 `MetaStore` 实现；p2p 没有对应插件，由 `ControlService` 直接处理（`control_plane.cpp:297`） |
| `tent/src/metrics/*` | `TentMetrics` 单例 + 9100 HTTP 报表（编译期开关，默认 OFF） |
| `tent/src/transfer_engine_c.cpp` | C API 实现 |
| `tent/src/python/pybind.cpp` | Python 模块 `tent`（内部名 `tentpy`） |
| `tent/config/transfer-engine.json` | 完整参考配置（policy、每个 transport 的参数块） |
| `tent/tests/` | 78 个测试源、83 个 `add_test`；`notification_fallback_test.cpp`、`engine_failover_e2e_test.cpp` 是关键回归 |
| `tent/benchmark/hip_bandwidth_bench.cpp` | 唯一一个独立 hipcc 程序；真正的 TENT 性能后端在 `mooncake-transfer-engine/benchmark/tent_backend.cpp` |

## 核心数据结构

### `TransportType`：既是枚举也是 wire 契约

`tent/include/tent/common/types.h:46-68` 定义了 18 个后端 + `UNSPEC` 哨兵，最后是长度哨兵 `kNumTransportTypes`。三条约束必须记住：

- **只能尾部追加**。数值被 C 宏（`tent/include/tent/transfer_engine.h:102-120`）和 Python 绑定同时引用；`tent/src/python/pybind.cpp:36-51` 用一串 `static_assert` 把 8 个值钉死等于 C 宏，任何重排都是编译失败，而不是线上协议破裂。
- `UNSPEC = 0` 被当作「无 hint」的哨兵使用（`Request::transport_hint`、`c_to_transport_hint()` 越界也返回 `UNSPEC`）。
- `kNumTransportTypes` 是数组长度，所有 `std::array<..., kSupportedTransportTypes>` 依赖它。

### `Request` / `TaskInfo`：应用视角 vs 引擎视角

`Request`（`types.h:156-180`）按 7 组字段看，每组都影响选路：

| 字段 | 作用 |
|---|---|
| `opcode` | `READ`/`WRITE`，影响 TCP 的 payload 上限（`control_plane.h:128-134`） |
| `source` | 本地地址；**同时**用于推导本地内存类型（`Platform::getLoader().getMemoryType`） |
| `target_id` / `target_offset` / `length` | 远端定位，`target_id == 0`（`LOCAL_SEGMENT_ID`）表示本进程 |
| `priority` | `PRIO_HIGH/MEDIUM/LOW`，进 `SelectionContext::priority_level`，可被 policy 的 `priority` 精确匹配 |
| `policy_name` | 强制绑定某个 policy，跳过「首个匹配」逻辑 |
| `transport_hint` | 非 `UNSPEC` 时被 `reorderWithHint` 提到候选表首位；若该后端不可用则**直接拒绝**而不是降级（`transport_selector.cpp:584-591`） |
| `deadline_ns` / `intent_type` | 分别是 SLO 绝对时间戳与业务意图；只有 opt-in 的 admission queue / policy 过滤才消费 |

`TaskInfo`（`transfer_engine_impl.h:73-110`）是引擎内部的任务记录，字段几乎每一条都对应一个设计决定：

- `derived` + `physical_owner_task_id`：请求合并的产物。多个 public 请求合并成一个物理任务后，`derived=true` 的别名不自己发包，只镜像 owner 的状态。
- `xport_priority`：候选 transport 表的下标，**failover 时递增**；`failover_count` 是预算计数器。
- `suppress_failover`：把「这个失败是永久的」钉死，例如 TCP 的 WRITE ACK 丢失（远端结果未知）或 HP_TCP 的元数据刷新重试耗尽。
- `failure_stage`（`-1/0/1`，`impl.h:102-109`）：归因用，**first failure wins**——在失败发生地写入，之后永远不覆盖。所以「提交期失败 → 恢复 → 后来轮询又失败」仍然算提交期失败。
- `attempt_type` / `attempt_active` / `attempt_post_time`：区分「逻辑请求」与「当前物理尝试」。`post_time` 故意保留逻辑请求的首次 post 时间（`impl.h:94-99`），用于端到端延迟指标。
- 拷贝/移动构造是手写的（`impl.h:113-227`），因为 `staging_status` 是 `std::atomic`——这决定了 `TaskInfo` 不能 `= default`。

### `Batch`：batch_id 就是指针

`Batch` 定义在 `transfer_engine_impl.cpp:62-89`，`BatchID` 直接是 `reinterpret_cast<BatchID>(batch)`（`allocateBatch`，`impl.cpp:1149-1163`）。这带来两个后果：

- batch 生命周期必须靠**注册表**保护：`batch_shards_` 是 64 个 `BatchShard`（`impl.h:653-682`），每个 shard 自带 `active_batches`（`Batch*` 集合，析构用）与 `alive_batches`（`BatchID` 集合，合法性检查用）。registry 内嵌在 shard 里，所以 poll/alloc/free 热路径只需一把 shard 锁，N 个线程轮询 N 个 batch 不会互相串行。
- `freeBatch` 有快路径和慢路径（`impl.cpp:1165-1249`）：只有在「runtime queue 关闭 + 无 runtime refs + 首次 free + 最终轮询成功且已终态」时才原地释放；否则标记 `free_requested` 丢进 `batch_freelist_`，由 `lazyFreeBatch()`（`impl.cpp:1251-1353`）异步扫。扫不动会累计 `reclaim_failures`，到 4096 次后 `reclaim_abandoned`，交给 `deconstruct()` 兜底。

锁序是全章最需要背下来的一条（`impl.h:648-652`）：

```
progress_mutex_ -> shard
```

**永不反向**。`freeBatch` 的慢路径为此专门「先放 shard、再拿 `progress_mutex_`、再重拿 shard」（`impl.cpp:1209-1224`）。runtime queue 开启时，热路径直接持 `progress_mutex_`，shard 只用于成员检查（`progressLockFor`，`impl.h:666-669`）。

### `Transport`：后端契约

`tent/include/tent/runtime/transport.h:47-196`。最重的不是方法签名，而是**提交语义的注释**（`:98-106`）：

> 全有或全无：返回错误时，`request_list` 中任何一个请求都不得被派发，sub-batch 必须保持调用前的样子。

这条约定是跨 transport failover 的安全前提：引擎在 `submitTransferTasks` 失败后，会把**同一个**任务原样交给下一个后端（`impl.cpp:2397-2399`）。如果某个后端「发了一半再报错」，引擎就会重复执行前缀。写新 transport 时这是第一优先级。

其他关键点：

- `SubBatch`（`:49-64`）携带 `device_mask`、`qp_pool`、`progress_batch_id` + `notify_progress` 回调。其中 `device_mask` / `qp_pool` 从 `SelectionPolicy` 经 `SelectionResult`、`TaskInfo` 一路透传到 `RdmaTask`，是「策略真正生效」的通道；`progress_batch_id` 和回调则由引擎在 `attachProgressNotifier()` 里填（`impl.cpp:2208-2215`），用于后端主动唤醒 progress。
- `quiesce()`（`:85`）是 teardown 屏障：在注册范围与 sub-batch 被回收**之前**调用，让异步 I/O 收尾。
- `supportsCancellation()` 是 best-effort（`:125-129`）：已下发到设备的工作仍可能完成，调用方必须继续 poll 到终态。
- `supportNotification()` 默认 `false`；`sendNotification`/`receiveNotification` 默认返回 `NotImplemented`。

### `SelectionContext` / `SelectionPolicy` / `SelectionResult`

`transport_selector.h:74-166`。`SelectionPolicy` 的字段就是 policy JSON 的字段；`resolved_device_mask:149` 是**缓存**——`setTopology()` 时一次性把 `devices` 名单解析成 64 位掩码（`transport_selector.cpp:308-355`），避免每个请求重复查拓扑、重复打日志。

注意两个「已废弃但仍解析」的字段：`service_level` / `traffic_class`（`transport_selector.h:128-134`）。它们的注释写得很直白：链路层 QoS 属于 QP pool，不属于 policy，这里配了只会打一条 WARNING（`transport_selector.cpp:286-294`）。真正生效的是 `qp_pool`，它从 `SelectionResult` → `TaskInfo` → `SubBatch` → `RdmaTask` → `selectQpInPool()` 一路透传。

`SelectionResult::staging_params`（`:165`）由引擎填，不由 `select()` 填——这是「选择器不懂 staging」的分层。

### `Config`：一棵树 + 斜杠路径

`tent/include/tent/common/config.h:34-184`。`get()` 先按 `'/'` 逐段下钻（`:42-63`），失败再尝试**扁平键**（`:64-71`）。这个 fallback 是为了兼容老配置里 `"transports/rdma/enable": true` 这种把路径写成单键的写法。

`setFromString()`（`:131-161`）的类型推断值得记：`"true"/"false"`（大小写不敏感）→ bool；纯整数 → `long long`；否则 string。**注意 `"1"` 会被推断成整数 1**，所以 `config.get("x", false)` 读不到它。`ConfigHelper::loadFromEnv` 因此给 bool 型环境变量单独用了 `setBoolConfig`（`config.cpp:139-145`），先 `parseBool` 再 `set`。

`freeze()`（`config.h:175`）返回 `shared_ptr<const Config>` 快照，`paths()`（`:179`）枚举全部路径。这两个是 `config_lifecycle.h` 的基础：`buildTentConfigBundle()` 在 `construct()` 第一行先 `freeze()` 出一份不可变副本，再包出 `BootstrapConfig` 和 `RuntimeConfigSnapshot`（引擎只保留后者，存进 `runtime_config_snapshot_`）。快照带着 `max_failover_attempts` 和 `enable_auto_failover_on_poll`（默认 3 / true，`config_lifecycle.h:128-136`），在 `prepareSubmit` 时被**拷贝进每个任务**（`impl.cpp:2103-2113`）。于是换新快照只影响之后提交的任务，已经提交的任务保留它出生时的策略——这是一个刻意的「任务级策略快照」。注意目前唯一的运行期发布入口是 test-only 的 `publishRuntimeConfigForTest()`（`impl.h:363-373`），生产路径只在 `construct()` 发布一次。

### `BootstrapDesc` / `OnNotify`：控制面的两个交换结构

`control_plane.h:46-60`：`BootstrapDesc` 是 RDMA 握手信封，包含 `qp_num` 列表、LID/GID、`reply_msg`，以及 **`notify_qp_num`**（`:54`，0 表示对端不支持通知）。UB 因为 Jetty/JFC/EID 与 RDMA 的 QP/CQ/GID 不兼容，单独有 `UbBootstrapDesc`（`:65-119`），并且带 `protocol_version` 校验（`:103-105`）。

`OnNotify = std::function<int(const Notification&)>`（`:142`）——返回 int 而不是 `Status`，这个设计直接决定了提交 `5cf142e3` 的报错方式（见 notification 流程）。

### `Topology`：3 级设备优先级

`topology.h:45-79`。`NicEntry` 有 `name/pci_bus_id/type/numa_node`，`MemEntry` 有 `device_list[DevicePriorityRanks]`——经典优先级矩阵里 rank 0 是 `preferred` 列表、rank 1 是 `avail` 列表（`topology.cpp:301-313`）；自动探测时同 NUMA 的 NIC 进 rank 0、其余进 rank 2（`DevicePriorityRanks - 1`，`topology.cpp:157-162`），rank 1 只由矩阵填充。`findNearMem(location, type)` 就是「给定内存位置，找最近的 host DRAM staging 位置」（同 NUMA 优先，否则取第一个该类型），staging 路径完全依赖它。

## 关键流程剖析

### 流程 A：引擎装配 `construct()`

入口 `TransferEngineImpl::construct()`（`impl.cpp:369-591`）。它是**唯一的装配真源**，顺序本身承载语义：

```
ConfigHelper::loadFromEnv -> construct()
  1. buildTentConfigBundle(*conf_).runtime  (369-373, 先发布快照)
  2. ParseHpTcpTransportConfig                (374)
  3. metadata_type / metadata_servers / log_level / hostname / segment name
  4. rpc_server_port 校验 + rpc_server_threads 默认值策略 (381-393)
  5. merge_requests_ / notify_rpc_fallback_ / enable_progress_worker_
     / runtime_queue_* 全量读取 + 非零校验 (394-425)
  6. 本地 IP 发现                              (428-431)
  7. Topology::loadFromConfig(conf, {Platform::getLoader(conf_)})  (433-435)
  8. ControlService 构造 + start(port, ipv6, threads)         (437-440)
  9. local_segment_name: p2p -> host:port；否则随机名        (442-445)
 10. setupLocalSegment()                       (447)
 11. TransportSelector 构造 + setTopology     (450-451)
 12. legacy_mode = use_legacy_transport_selection    (454-458)
 13. loadTransports()                          (460)
 14. 注册"临时" notify 回调 -> local_notifi_list_   (468-472)
 15. 按槽位升序 install 所有 transport          (474-507)
 16. staging_proxy_ = ProxyManager(this)       (509)
 17. deadline 退化策略（可选）                  (511-544)
 18. ProgressWorker（可选）                     (546-552)
 19. Metrics（可选）                            (554-574)
```

几个必须点名的细节：

**rpc_server_threads 的默认值是条件式的**（`:386-393`）。TCP 未启用时默认 1；TCP 启用且用户没显式配置时取 `min(8, max(4, hardware_concurrency()))`。理由写在注释里：TCP 的 `SendData`/`RecvData` 拷贝虽然被 offload，但 RPC io_context 仍要读完整 attachment，单线程会把并发大块 TCP 串行化。

**HP_TCP install 失败是致命的**（`:482-498`）。其他 transport install 失败只打 WARNING 并把槽位清空（`:499-502`），但 HP_TCP 如果被显式要求（`hp_tcp_transport_config_.enabled`），失败会立刻 `deconstruct()` 回滚并返回错误。`deconstruct()` 因此必须是幂等的——析构函数里还会再调一次。

**临时 notify 回调的时序**（`:468-472` 与 `:479`）。`ControlService::start()` 在第 8 步就启动了 RPC server，而 RPC handler 在没有回调时**不会收下通知**（现在会写回原因）。所以第 14 步先注册一个「把通知塞进 `local_notifi_list_`」的临时回调；随后 transport 在 `install()` 里用自己的回调**接管唯一槽位**。`ControlService` 只有一个 `notify_callback_`，谁最后 install 谁赢。install 是升序的，所以 TCP（槽 7）先注册、HP_TCP（槽 13）后注册——两者都开时，走 RPC 到本机的通知进的是 **HP_TCP 的队列**。

**`local_segment_name` 的 p2p 规则**（`:442-445`）：`metadata_type == "p2p"` 时段名就是 `host:port`，否则由用户给或随机生成。旧 TE 兼容层 `getLocalIpAndPort()` 直接返回这个段名而不是重建 host:port（`mooncake-transfer-engine/src/transfer_engine.cpp:752-764`），注释明确说「重建出来的可能和实际段名不一致」。

`deconstruct()`（`:593-712`）是镜像顺序，且每一步顺序都有理由：

1. `progress_worker_->stop()`：先停，防止它拿 `BatchID` 去解引用已拆的 Batch；但**不销毁对象**，因为 worker 线程 join 期间完成路径还可能发一次 no-op 唤醒（`:600-602`）。
2. 所有 transport `quiesce()`（`:604-612`）。
3. `staging_proxy_.reset()` 必须在 `local_segment_tracker_` / `metadata_` 之前——它的析构会回调 `unregisterLocalMemory`/`freeLocalMemory`（`:614-617`）。
4. 遍历本地段描述，对每个 buffer 调所有 transport 的 `removeMemoryBuffer`（传的是**拷贝**，因为 transport 可能改写参数）。
5. 释放所有 batch（在销毁 transport **之前**），这样 `freeSubBatch()` 还能把 SubBatch/Slice 还给 Slab（`:640-664`）。
6. `transport.reset()`（worker 在这里 join）。
7. 处理 `DeferredStageTeardown`：超时未排空的 staging batch 与其 stage buffer **故意泄漏**到进程退出（`:679-700`）。原因写在注释里：`freeSubBatch()` 是刚销毁的 transport 的虚函数，`freeLocalMemory()` 走 `allocated_memory_` 里的裸 `Transport*`，此时回收就是 use-after-free。
8. 清 notify 回调、`deleteLocal()`、释放 `metadata_`。

### 流程 B：一次提交 `submitTransfer`

调用链：

```
TransferEngineImpl::submitTransfer(batch_id, req_list, notifi*, owner_kind, require_cancel)
  -> retainBatch / BatchRef           (2776-2778)
  -> prepareSubmit                    (2781)
       -> validateTransportHint       (2095-2098)
       -> 读 runtime snapshot         (2103-2113)
       -> resolveRequestBoundaries    (2116-2120)
       -> mergeRequests               (2121-2122)
       -> TCP 超限合并回退             (2130-2156)
       -> 每个合并后请求 resolveTransport(_, 0)  (2159-2186)
       -> public -> merged 映射       (2188-2202)
  -> shouldQueueSubmit ? enqueuePreparedSubmit : commitPreparedSubmit (2784-2795)
  -> addSubmitHook（若带 notifi）     (2797-2799)
```

分步说明：

**B1. 合并请求。** `resolveRequestBoundaries()`（`:1862-1940`）为每个请求找出源/目的 buffer 的 `{addr, length}` 身份键，并按 `target_id` 分组，每组只触发一次 `withCachedSegment`。合并条件很保守（`mergeRequests`，`:1790-1834`）：必须同 opcode、同 target、同 `transport_hint`、同 `priority`/`policy_name`/`intent_type`（因为这几个都会影响选路，合并会「把第一个请求的策略偷偷套给整段」），源和目的都必须**地址连续**，且总长不超过 `max_merge_bytes`。`max_merge_bytes` 对 HP_TCP 有单独上限，取自对端 buffer 上编码的 endpoint 属性（`:1888-1899`）。

还有一个容易漏的不变量（`:1919-1935`）：`withCachedSegment` 的 lambda 一旦返回 `NeedsRefreshCache`，本次写入的所有 `target_key` 必须**全部清空**。因为缓存失效后只会重拉一次，如果重拉失败，重试 pass 根本不会跑，残留的陈旧 key 会泄漏进 `mergeRequests`。

**B2. 选路（每个合并后请求一次）。** `resolveTransport(req, 0)` → `getTransportType(req, 0)`（`:1526-1679`）：

1. 取本地/远端段描述（`LOCAL_SEGMENT_ID` 走本地）。
2. 探测源地址内存类型 `local_mtype`。
3. **XPU/MLU staging 预判**（`:1541-1580`）：仅对**远端**目标生效。触发条件是「装了 XPU transport 且任一侧为 XPU」或「任一侧为 MLU」（`:1545-1552`）。命中后先 `findStagingPolicy` 求三段 `{server_addr, local_stage, remote_stage}`，再只在 `RDMA/FABRIC/TCP/HP_TCP` 里挑候选；未 staging 的一侧必须已注册该 transport，staging 的一侧由新建 host stage buffer 顶替（`carries` 对 nullptr 视为满足，`:1568-1572`）。
4. legacy 模式（`use_legacy_transport_selection=true`）走老逻辑：file 段 GDS→IOURING；memory 段取 `entry->transports`，并按机器本地性过滤 XPU/MLU/NVLINK/SHM/TPU（`:1584-1637`）。
5. 默认走 selector：构造 `SelectionContext`，把 `buffer_transports` 指向**对端** `entry->transports` 或 staging 候选，调 `TransportSelector::select`。

注意 `resolveTransport()` 的 `invalidate_on_fail` 默认是 `true`（`impl.h:517-518`）：第一次选不出 transport 时会先 `invalidateRemote(target_id)` 再重选一次，才把 `UNSPEC` 交回去（`impl.cpp:2080-2089`）。所以「缓存陈旧导致选路失败」在 submit/failover 路径上已经被吸收掉一次。

`select()`（`transport_selector.cpp:529-606`）的核心是三段：

- **首个匹配 policy 胜出**（`:537-543`），匹配维度见 `matchesPolicy`（`:396-460`）：segment_type、same_machine、local/remote memory pattern、min/max size、priority 精确相等、intent_type 精确相等。**没有 `intent_type` 的 policy 是 catch-all**——这是历史兼容行为，写新 policy 时要小心它会把别的意图也吃掉。
- `raw` 候选表 = force_tcp 且 memory 段时 `{TCP}`；否则 policy 的 `transports`；否则对端 buffer 注册的 transports（`:569-575`）。
- `reorderWithHint` 把 hint 提到首位（`:608-621`）；hint 不在候选表里返回 `nullopt` → 整个选择返回 `UNSPEC`，任务注定失败。hint 在候选表里但该后端不可用，也在 slot 0 直接拒绝（`:584-591`），**不会**悄悄降级到别的后端。
- 按 `transport_index` 在**过滤后**的候选表上走 N 步（`:593-598`）——这就是 failover 的「有序回退表」。

> **纠正一个常见误解**：`transport_list_` 的槽位顺序（枚举值顺序）**不是** failover 优先级。failover 优先级来自 policy 的 `transports` 数组顺序；未配 policy（内置的 `memory_default` policy 的 `transports` 为空）时来自**对端** `BufferDesc::transports`，而后者由 `getSupportedTransports()` 的注册顺序决定（`impl.cpp:914-944`：MNNVL → NVLINK → XPU → MLU → RDMA → UB → FABRIC → SUNRISE_LINK → AscendDirect → SHM → TCP → HP_TCP → GDS → MPCOMM → TPU → HYLINK → MEMFABRIC），它是**第三个**独立顺序。槽位顺序只影响两件事：`install()` 的顺序，和 notification 的尝试顺序。

**B3. 落地任务。** `commitPreparedSubmit()`（`:2217-2421`）是整套逻辑里最容易读错的一段：

- 先做容量检查，然后 `task_list.insert` 出所有 public 槽位（`:2229-2230`）。
- 对每个 public task：若是某个合并组的第一次出现，就填 `TaskInfo`（`type = owner.route.transport`、`device_mask`、`qp_pool`），并用 `owner_task_id_by_merged_task` 记住它的物理 id；否则直接拷贝 owner 的 `TaskInfo`、置 `derived=true`、写 `physical_owner_task_id`（`:2241-2254`）。
- `task.type == UNSPEC`（选不出后端）时不分配 sub-batch，只记 WARNING；后续 `pollTaskStatus` 对 `UNSPEC` 短路成 FAILED（`:2271-2276`、`:3084-3088`）。
- staging 任务交给 `staging_proxy_->submit()`，**不**在这里调 transport（`:2278-2293`）。
- 每个 transport 的 sub-batch 懒分配一次，并 `attachProgressNotifier`（`:2296-2307`）。
- **RDMA/UB 必须按 `(device_mask, qp_pool)` 分组提交**（`:2326-2339`）：一个 SubBatch 一次提交只带一组策略标量，同 transport 不同策略的请求必须分批。其他 transport 一批到底。
- `sub_task_id` 从 `sub_batch->size()` **当前值**开始分配（`:2350-2358`），这样前面一组同步失败不会给后面一组留下空洞。
- 调 `submitTransferTasks` 前对所有任务 `startTransportAttempt`（`:2371-2376`）。
- 失败时（`:2378-2416`）：先给整组打 `failure_stage=0` 并 `finishTransportAttempt(FAILED)`，然后**只对物理 owner** 调 `attemptSubmitStageFailover`（对 derived 重复提交会把合并后的整段传输再发一遍——`failover.md` 称之为 hazard 1），最后把 owner 恢复后的 `type/sub_task_id/status/failover_count/xport_priority` 镜像给所有 derived 别名。

**B4. 提交后钩子。** 带 `notifi` 的提交会记一个 `SubmitHook`（`:2821-2832`）；`maybeFireSubmitHooks`（`:2834-2876`）在该区间所有**非 derived** 任务 COMPLETED 后，对 hook 的每个 target 各发一次通知。目标逐个移除、失败的留到下次 poll；每个 target 每次 pass 都会重试，注释解释了为什么不能在第一个失败处停下：`targets` 是 `unordered_set`，顺序不定，若一个不可达的远端排在前面，会阻塞一个本来不可能失败的自通知。

### 流程 C：轮询与跨 transport failover

```
getTransferStatus(task_id)                       (3346-3419)
  -> progressLockFor(batch_id)
  -> refillDispatchWindow()                      (仅 runtime queue)
  -> derived -> poll_task_id = physical_owner
  -> pollTaskStatus                              (3070-3155)
  -> updateTaskStatusAfterPoll(allow_failover)   (3157-3200)
  -> recordTaskCompletionMetrics
  -> maybeFireSubmitHooks (if COMPLETED)
```

**C1. `pollTaskStatus` 的分支顺序就是错误语义的优先级：**

1. `task.status == FAILED` → 直接返回 FAILED（逻辑失败是终态，不再 poll）。
2. `task.staging` → 交给 `staging_proxy_->getStatus`。
3. `task.type == UNSPEC` → FAILED（选不出后端）。
4. 正常调 `transport->getTransferStatus`。HP_TCP 在调用前先把 `task_status` 预置成 `{PENDING, 0}`（`:3095-3097`），因为它的错误分类依赖「终态输出」，不能把上一轮的陈旧状态带进来。
5. TCP 特例（`:3100-3106`）：若返回 FAILED 且 `result.IsRpcServiceError()`（WRITE 的 ACK 丢了，远端结果未知）→ `finishTransportAttempt(FAILED)` + `suppress_failover = true`。**这类错误明确不允许 failover**，因为重发一个可能已生效的 WRITE 会造成重复写。
6. HP_TCP 特例（`:3107-3144`）：`NeedsRefreshCache` 表示段缓存陈旧 → 失效缓存并 `retryTransferTask`，最多 `kMaxHpTcpMetadataRefreshRetries = 1` 次（`:59`）；超限或失效失败则 `suppress_failover = true`。
7. 其余非 `TooManyRequests` 的错误 → 永久失败，`suppress_failover = true`（`:3146-3153`）。只有 `ShuttingDown`（映射成 `TooManyRequests`）保持可重试。

**C2. `updateTaskStatusAfterPoll` 才是 failover 的决策点**（`:3157-3200`）：

```
task.status = task_status.s
if terminal(FAILED/TIMEOUT/CANCELED) and failure_stage < 0: failure_stage = 1
if !allow_failover || cancel_requested || suppress_failover
   || status != FAILED || type == UNSPEC: return
finishTransportAttempt(FAILED)
while failover_count < max_failover_attempts:
    if resubmitTransferTask(...).ok(): status = PENDING; break
把 owner 的新 type/sub_task_id/... 镜像给 derived
```

`allow_failover` 来自 `task.runtime_policy.enable_auto_failover_on_poll`（默认 true）；`progressBatch` 传 `force_failover=true` 绕过它（`:3548-3551`、`:3501-3503`）。

**C3. `resubmitTransferTask`**（`:2966-3037`）是 failover 的唯一入口：

- `++failover_count > max_failover_attempts` → `InvalidEntry("Failover limit exceeded")`。
- 非 staging 任务 `xport_priority = failover_count`（烘焙「失败一次就下移一格」的语义）；staging 任务保持 0 并清掉 staging 标记。
- `resolveTransport(task.request, task.xport_priority)`；`UNSPEC` → "All available transports are failed"。
- 若新路由带 staging 参数 → 交给 `ProxyManager`（`:3006-3013`），注释强调「失败的设备 staging 绝不能退化成本地把 VRAM 地址发给纯主机网络后端」。
- 否则懒分配 sub-batch、写 `device_mask`/`qp_pool`（RDMA 还要写进 sub-batch）、`sub_task_id = sub_batch->size()`、`startTransportAttempt`、`submitTransferTasks`。同步失败时打 `failure_stage=0` 并 `finishTransportAttempt`。

**C4. 提交期 failover**：`attemptSubmitStageFailover`（`:3039-3068`）先把 `task.status` 置 FAILED（防止并发 poll 看到半迁移状态），再在一个循环里调 `resubmitTransferTask` 直到成功或预算耗尽；成功则改回 PENDING。彻底失败时先 `recordTaskCompletionMetrics(PENDING, FAILED)`（poll 路径只记录「离开 PENDING」的转换，不会看到这次），再把 `type` 清成 UNSPEC 让 poll 短路。

> `docs/source/design/tent/failover.md:9,21,76` 仍写着「提交期失败今天不重试（Known Gaps）」。**代码已经实现了提交期 failover**，而且同一份文档的第 129、188、193 行已经把它当作既有行为描述、`engine_failover_e2e_test.cpp:1649`（`SubmitStageFailoverWithDerivedTasks`）也把它钉成了断言——所以过时的是那几行 Known Gaps，不是整篇文档。精读时以代码为准。

**C5. 批量聚合**：`getBatchStatus`（`:3437-3541`）跳过 derived，按 `transferStatusSeverity`（`types.h:194-211`）取最严重的失败；只有 `success_tasks == total_tasks` 才 COMPLETED，只有「全部终态且至少一个失败」才返回失败，否则一律 PENDING。注释解释了为什么未知状态按 FAILED 的严重度排序：`getBatchStatus` 在 teardown 期间不能从 `unordered_map::at` 抛异常。

### 流程 D：notification 的四条路径

这是近期改动（`5cf142e3`）的落点，也是最容易被讲错的一块。

**D1. 发送：`sendNotification`**（`:3202-3250`）

```
if target_id == LOCAL_SEGMENT_ID:
    local_notifi_list_.push_back(notifi); return OK        (3204-3214)
for slot in 0..kNumTransportTypes:
    if !transport || !transport->supportNotification(): continue
    attempt = transport->sendNotification(...)
    if attempt.IsNotImplemented(): continue                (3231)  // SunriseLink
    attempted = true; status = attempt
    if status.ok(): return status
    if !notify_rpc_fallback_ || !(status.IsRdmaError() || status.IsDeviceNotFound()):
        return status                                      (3235-3238)
if !attempted: return InvalidArgument("Notification not supported")
return sendNotificationViaRpc(target_id, notifi)           (3249)
```

四个要点：

1. **自通知短路**（`:3204-3214`）。RDMA 的本地伪 endpoint 永远不会建立通知 QP，走 transport 会发失败、接收端会永远 poll 不到。
2. **只有两种错误码换下一条路径**：`RdmaError`（通道不可用）与 `DeviceNotFound`（endpoint 建不起来）。其他错误（参数错、缓存陈旧）直接返回给调用者。
3. **`NotImplemented` 被跳过**。`SunriseLinkTransport` 声明 `supportNotification() = true`（`tent/include/tent/transport/sunrise_link/sunrise_link_transport.h:87`）但没有 override send/receive，基类返回 `NotImplemented`——引擎靠这个特判把它当「不是通道」。
4. **「已发出但在途丢失」明确不在兜底范围**（`:3219-3222` 注释）：重发需要接收端去重，注释写明 out of scope。

实际生效的路径取决于哪些 transport 装了。升序下 `RDMA(1) → TCP(7) → SunriseLink(9, 跳过) → HP_TCP(13)`。注意 `TcpTransport::sendNotification` 本身就是走控制面 RPC（`tcp_transport.cpp:357-375`），所以**TCP 一开，显式的 `sendNotificationViaRpc` 基本走不到**——它只在 TCP/HP_TCP 都不可用（或都返回两种「通道不可用」错误）时才作为最后兜底。另外 TCP 在空 `rpc_server_addr` 时返回的是 `NeedsRefreshCache`，不是两种可跳过错误之一，所以它会**中止**循环。

**D2. 控制面投递与失败回报**（提交 `5cf142e3` 的核心）

`ControlClient::notify`（`control_plane.cpp:179-191`）：

```
CHECK_STATUS(tl_rpc_agent.call(server_addr, Notify, request, response));
if (response.empty()) return OK;
if (response.starts_with("malformed notification: ")) return InvalidArgument;
return InternalError(response);
```

`ControlService::onNotify`（`control_plane.cpp:608-651`）现在的行为：

| 情况 | 旧行为 | 新行为 |
|---|---|---|
| 没有注册回调 | 空响应（= 已送达） | `"notify callback not registered"` |
| 请求解析失败 | 空响应 | `"malformed notification: " + e.what()` |
| 回调返回非 0 | 空响应 | `"notify callback failed: returned N"` |
| 回调抛异常 | 空响应 | `"notify callback failed: " + e.what()` |
| 未知异常 | 空响应 | `"notify callback failed"` |

前缀 `kMalformedNotification` / `kNotifyCallbackFailed` 定义在 `control_plane.cpp:50-51`。**语义分界**：解析失败每次重试都会失败（所以映射成 `InvalidArgument`，不该重试）；缺少回调或回调失败是「对端应用层没收下」（映射成 `InternalError`）。兼容性：旧接收端回空 → 新发送方视为 OK；旧发送方忽略响应体 → 新接收端多写一个字符串也无害。

`onNotify` 的 in-flight 计数（`:620`、`:625-628`、`:427-431`）解决了另一个竞态：RPC server 比引擎先启动、teardown 比 server 先清回调。`setNotifyCallback`（`:402-419`）先把 `notify_callback_` 置空，再等 `notify_callbacks_in_flight_ == 0`（超时 5s，超时打 ERROR 并继续），最后才换上新回调。若在回调内部重入（`active_notify_service_ == this`，`:404-407`）则直接替换、不等待——否则会自死锁。

> 小瑕疵：`onNotify` 开头注释（`:613-614`）还留着「An empty response is the sender's ack」，与紧接着的 `response = "notify callback not registered"` 已经矛盾。以代码为准。

**D3. 接收：`receiveNotification`**（`:3291-3344`）

- 每次先 `notifi_list.clear()`：调用方会复用同一个 vector，追加会把上次已处理的再投一遍。
- 遍历**所有** transport（不是第一个），每个用独立临时 vector 接收（因为 `TcpTransport::receiveNotification` 会先 clear 参数），再拼起来。
- `NotImplemented` 的那个跳过；至少有一个真正 support 的 transport 才置 `has_transport`。
- 最后追加并清空 `local_notifi_list_`（`:3327-3336`）。
- 只要有内容返回 OK；空且没有任何通道才返回 `InvalidArgument("Notification not supported")`；否则返回**最后一个**非 ok 的 transport 错误——循环里的 `status` 会被每个失败覆盖（`notification_fallback_test.cpp:221` 的 `ReturnsTheLastTransportsErrorInsteadOfRpc` 就是钉这个行为）。

**顺序只保证在单路径内**：拼接顺序就是 transport 的槽位顺序（RDMA 的 notify QP 在前，TCP/HP_TCP 这类走控制面的队列在后），跨 transport 既不反映真实到达时间，也没有全序。

**D4. RDMA notify QP 的状态机**（`tent/include/tent/transport/rdma/endpoint.h:125-184`、`tent/src/transport/rdma/rdma_transport.cpp:982-1140`）

- 每个 endpoint 一个 notify QP；`notifyQpNum()` 经 `BootstrapDesc::notify_qp_num` 与对端交换（`control_plane.h:54`）。
- 收发各一块 64 KB 连续 host buffer，切成 `kNotifyMaxPendingSends = 256` 个槽（`endpoint.h:303-304`），各用一个 MR 注册。
- `shouldRearmNotifyRecv(status, notify_connected)`（`:167-170`）只在 `EP_READY && notify_connected` 时重挂 RECV；retiring endpoint 的 QP 已在 ERR，重挂是浪费。
- `notify_connected_` 是 `atomic<bool>`，故障后**单向关闭**、直到 endpoint 生命周期重建 QP；`notify_inflight_` 计数未轮询的 notify WR 完成，`finishDestroy()` 要等它归零，否则 provider 会把对端已确认的完成随 QP 一起丢掉（`:316-320`）。等待有上限（`kFinishDestroyTimeoutSec = 30s`），超时后打 WARNING 强制销毁（`endpoint.cpp:393-406`）。
- `classifyNotifyCompletion`（`rdma_transport.cpp:1036-1048`）把完成分成四种处置：

| 条件 | 动作 |
|---|---|
| `WR_FLUSH_ERR` 且 endpoint 不 ready | `SkipSilently`（retiring QP 的正常 flush） |
| endpoint 已不存活 | `ReportOnly` |
| 本地 fault（`LOC_LEN/QP_OP/PROT/ACCESS`、`MW_BIND`） | `DisableNotification`（只关通知，不动数据 QP） |
| 其他 | `RetireEndpoint`（可能对端重启/路径断了，重建整个 endpoint） |

- `sendNotification`（`:982-999`）的失败码映射在 `notifyStatusForEndpointFailure`（`:971-980`）：bootstrap RPC 本身失败 → `RpcServiceError`（**不再**兜底 RPC，因为控制面也到不了）；其他 → `DeviceNotFound`（可兜底）。

## 与其他模块的边界

### 依赖（TENT 依赖谁）

| 依赖 | 用途 | 边界文件 |
|---|---|---|
| yalantinglibs / ylt coro_rpc | 控制面 RPC、metrics HTTP | `tent/include/tent/rpc/rpc.h`、`tent/src/rpc/rpc.cpp` |
| nlohmann/json（vendored） | 配置树、RPC 信封 | `tent/include/tent/thirdparty/nlohmann/json.h` |
| pybind11 | Python 模块 `tent` | `tent/src/python/pybind.cpp` |
| libibverbs / liburing / hiredis / libcurl / etcd | 后端 | `tent/src/transport/rdma/ibv_loader.cpp`（dlopen `libibverbs.so.1`） |
| `mooncake_common`（`CreateRpcClientIoContextPool`/`Environ`）、`asio_shared` | 客户端 I/O 池、ASIO ODR 隔离 | `tent/src/runtime/CMakeLists.txt` 末段注释 |
| 各加速器 SDK | Platform 实现 | `tent/src/platform/*` |

### 被依赖（谁用 TENT）

- **旧 TE 兼容接缝**（`mooncake-transfer-engine/src/transfer_engine.cpp`）：构造函数读 `MC_USE_TENT` / `MC_USE_TEV1`（`:534-542`、`:571-582`）决定 `use_tent_`。切换后：
  - `init()` 把 `(metadata_conn_string, local_server_name)` 翻译成 TENT `Config`（`buildTentConfig`，`:642-660`）——`P2PHANDSHAKE` → `metadata_type=p2p`，否则按 `proto://servers` 拆；设备过滤写成 `topology/rdma_whitelist`。`protocol == "tcp"` 时调 `forceTcp()`（`:682-688`）。
  - `installTransport()` 变成 no-op，返回一个 `TentCompatibilityTransport` 垫片（`:447-492`、`:726-743`）——传输方法返回 `Status::NotImplemented`、内存注册方法返回 `ERR_NOT_IMPLEMENTED`、`getName()` 返回 `"tent_compat"`，目的只是让旧代码的 `Transport*` 变量还能拿到非空指针、编译运行不崩。
  - `getLocalIpAndPort()` 返回 TENT 段名（`:752-764`）。
- **Store / engine.so / store.so** 通过 `MC_USE_TENT=1` 走这条路径；`libtent_metrics.so` 被刻意编成 SHARED，避免两个 .so 各持一个 `TentMetrics` 单例（`tent/src/metrics/CMakeLists.txt` 整段注释：双份报表线程 + 9100 端口冲突）。

### 三套 API 的关系

```
C++ : mooncake::tent::TransferEngine        (tent/src/transfer_engine.cpp，逐方法转发 impl_)
        |
        +-- impl_ = unique_ptr<TransferEngineImpl>
C   : tent_* 函数                             (tent/src/transfer_engine_c.cpp，直接 new TransferEngineImpl)
Python: PYBIND11_MODULE(tent)                (tent/src/python/pybind.cpp)
```

三套 API 不是层层包装：C API 绕过 `TransferEngine` 壳直接持有 `TransferEngineImpl*`（`transfer_engine_c.cpp:41-54`），所以 C++ 壳上的任何默认值/校验都不会生效。C 结构体 `tent_request` **必须零初始化**（`transfer_engine.h:35-37` 的显式注释），因为 `transport_hint` 依赖 `UNSPEC == 0` 表示无 hint。

C API 也有覆盖不到的地方：`tent_request` 只有 opcode/source/target/offset/length/priority/transport_hint（`transfer_engine.h:38-46`），**没有 `deadline_ns` 和 `intent_type`**；`tent_submit` 的转换代码也确认没有填（`transfer_engine_c.cpp:212-222`）。要用 deadline 调度只能走 C++ 或 Python。

> **一个值得核实的实现细节**：`tent_load_config_from_file(path)` 只把 path 存进 `thread_local Settings`（`transfer_engine_c.cpp:35`），而 `tent_create_engine()` 调用的是 `config->load(tl_settings.path)`（`:44`）——`load()` 把参数当 **JSON 内容**解析，`loadFile()` 才是按路径读。按当前代码，传真实文件路径会解析失败、打 WARNING 并回退默认配置。这是有意（让"file"函数也能接受内联 JSON）还是缺陷，本章未能确认；使用 C API 加载配置文件时务必实测。

## 设计取舍与易踩坑点

### 1. 「插件」有两种，只有一种是真的

- **transport 不是插件。** `transport_loader.cpp:20-75` 是编译期 `#ifdef USE_RDMA/USE_UB/USE_CUDA/...` 的静态包含，`:80-178` 只做「编译期宏 × 运行期 `transports/<name>/enable`」双重门控后 `make_shared`（TCP/SHM/HP_TCP 三个不套 `#ifdef`，只受运行期开关约束，`transport_loader.cpp:81-91`）。没有 dlopen，没有注册表，新增后端必须重新编译。
- **device plugin 才是动态的**，但它当前是**休眠**的。`MemoryProber::loadPlugins`（`memory_prober.cpp:33-94`）会 `dlopen` 目录下所有 `*.so`、`dlsym("tent_register_device_plugin")`、`create_plugin` 建上下文，并支持 `alloc/free/memcpy_sync/query_location/get_device_pci_bus_id`（`device_plugin.h:33-46`）。但全仓库搜不到任何 `loadPlugins` 调用方（`memory_prober.cpp` 本身仍被 `file(GLOB)` 编进 `tent_runtime`）。生产路径上真正生效的是**编译期** `#ifdef` 选出的 `Platform::getLoader()`（`platform.cpp:43-70`），它再做 `call_once` 单例。
  → 因此 `tent/plugins/{cuda,rocm}/` 是可用的参考实现 + CMake 产物，但**不是运行时生效的扩展机制**。把它讲成「用户可热插拔设备后端」是错的。

### 2. 槽位顺序 ≠ 回退顺序 ≠ 注册顺序

三个独立顺序，混用会得出错误结论：

| 顺序 | 来源 | 影响 |
|---|---|---|
| 枚举槽位 | `types.h:46-68` | `install()` 顺序、notification 尝试顺序、`transport_list_` 下标 |
| policy transports | `policy[].transports` 数组 | 正常选路与 failover 的候选顺序 |
| 注册顺序 | `getSupportedTransports()`（`impl.cpp:914-944`） | policy 的 `transports` 为空时，对端 `BufferDesc::transports` 的顺序，即默认回退表 |

### 3. notification 是「尽力而为」，不是「可靠消息」

- 发送端只对两种错误码换路（`impl.cpp:3235-3238`），其他错误直接上抛。
- 发出但在途丢失**不重发**，因为需要接收端去重（注释明确 out of scope）。
- 接收端每次 poll 排空所有路径，但**只保证单路径内有序**。
- 提交 `5cf142e3` 之前，「RPC 成功」被当成「对端收下」；现在能区分「RPC 层成功」与「应用层收下」。讲可靠性时这两层必须分开。
- `ControlService` 只有一个 notify 回调槽 → **哪个 transport 最后 install，RPC 到达的通知就进哪个队列**。`receiveNotification` 因此必须排空所有 transport，不能只查一个。
- 自通知走 `local_notifi_list_`，与 transport 无关；即使一个 notification transport 都没装，自通知照样可收。

### 4. 默认关闭的东西比想象中多

| 开关 | 默认 | 说明 |
|---|---|---|
| `enable_progress_worker` | false | 不装事件驱动线程；poll 就是引擎 |
| `enable_runtime_queue` | false | admission queue / deadline 调度全关；开启会强制打开 progress worker（`impl.cpp:397-398`） |
| `transports/shm/enable` | false | 注释：当前路径不是 NUMA-aware |
| `transports/{ub,gds,hylink,memfabric}/enable` | false | UB 还需要 `USE_UB` 且拓扑里有 UB NIC |
| `transports/{rdma,tcp,io_uring,nvlink,ascend_direct,fabric,tpu,mlu,xpu,mpcomm,sunrise_link}/enable` | true（但受 `USE_*` 与硬件探测限制） | RDMA 还要求 `topology_->getNicCount(NIC_RDMA) > 0`（`transport_loader.cpp:94-96`） |
| `TENT_METRICS_ENABLED` | **编译期 OFF** | 关闭时宏展开为 `((void)0)`、零开销；配置里 `metrics.enabled=true` 不代表有指标 |
| `notification/rpc_fallback` | true | 关掉后退回「第一个 transport 说了算」 |
| `merge_requests` | true | 关掉后 `resolveRequestBoundaries` 都不跑 |

表外还有两处隐藏分叉：

- `USE_CUDA` 下由环境变量 `MC_ENABLE_MNNVL` 决定装 MNNVL 还是 NVLINK，同一次构建不会两者都装（`transport_loader.cpp:112-121`）。
- `enable_runtime_queue` 不接管普通提交里的 staging 请求：`shouldQueueSubmit()` 看到任一 staging owner 就退回同步提交，显式的 post-submit-cancellation 请求也绕过队列；只有 `QueueOwnerKind::StagingInternal`（ProxyManager 内部再提交）无条件入队（`impl.cpp:2762-2770`）。

把 deadline 调度 / MLU / 准入队列当成 TENT 默认行为是常见误解——它们是 RFC #2519 / #2792 的 opt-in 分层。

### 5. 「全有或全无」不是建议，是安全前提

`transport.h:98-106` 的约定如果在某个后端被破坏，后果不是「报个错」，而是**跨 transport failover 时重复执行前缀**。写 transport 时，凡是「边校验边提交」的实现都必须改成「先全量校验，再全量提交」；中途出错必须回滚到调用前状态。

### 6. batch 的锁与生命周期

- 锁序 `progress_mutex_ -> shard`，永不反向（`impl.h:648-652`）。`progress_mutex_` 本身是 `recursive_mutex`，因为 `freeBatch -> lazyFreeBatch -> getTransferStatus` 会重入（issue #2116，见 `impl.h:626-629`）。
- `TaskInfo` 手写拷贝/移动，因为含 `std::atomic`（`impl.h:113-227`）。
- `BatchID` 是指针。`retainBatch`/`releaseBatch` 用 `runtime_refs` 做线程安全的外部引用；`freeBatch` 之后再 `getTransferStatus` 返回 "Batch is not alive"，**不是** UB——但依赖 `alive_batches` 注册表。
- 永久卡住的 batch 在 4096 次 reclaim 失败后被 `reclaim_abandoned`，扫描停止告警，`deconstruct()` 无条件回收（`impl.cpp:54-58`、`:1262-1267`）。

### 7. staging 的 ownership 与 teardown

`ProxyManager` 头文件第一行就是 "Beta version -- use with own risk"。它的 shutdown 排空有硬超时（默认 30s，可用 `staging/shutdown_drain_timeout_ms` 覆盖）；超时后 `adoptDeferredStageTeardown`（`impl.cpp:1390-1433`）把 batch 和 stage buffer 的所有权转移给引擎：从 `active_batches`/`alive_batches`/`batch_freelist_` 摘掉，队列 owner 结算成 CANCELED，等 transport quiesce 后**故意泄漏**。这是理解 `deconstruct()` 顺序的关键——不能先销毁 `transport_list_` 再回收这些资源。

### 8. 其他容易读错的地方

- **`NotImplemented` 是控制流**。SunriseLink 在 notification 里靠它被跳过（`impl.cpp:3231`、`:3310`）；`submitTransferRequiringPostSubmitCancellation` 用它表示「该 transport 不支持 scatter」。
- **`UNSPEC` 两重含义**：`Request::transport_hint` 里是「无 hint」，`SelectionResult::transport` 里是「选不出来 / 没有更多候选」。同一个值，不同上下文。
- **`maybeFireSubmitHooks` 必须跳过 derived**（`:2846`）：被合并掉的 public 任务状态永远停在初始值，检查它会让 hook 永不触发。`getBatchStatus` 同理（`:3457`）。
- **`allocateLocalMemory` 的后端选择是硬编码优先级**（`impl.cpp:831-874`），不是 selector：host 位置优先 MNNVL(dram_to_dram) → RDMA → UB → FABRIC → TCP → HP_TCP → SHM。这与「传输选路走 policy」是两套逻辑。
- **`topology/priority_matrix` 优先于 `custom_json_path` 优先于自动探测**（`topology.cpp:349-391`）。`parseCustomTopology` 先看是不是原生 `{"nics","mems"}`，不是就按经典 TE 优先级矩阵解析（`:343-346`）。
- **device_mask 解析失败是 fail-open**（`transport_selector.cpp:334-346`）：`devices` 全部解析不出来时掩码设 `~0ULL` 并打 **ERROR**，传输继续但设备限制被忽略。看到一个 ERROR 说 "restriction is IGNORED" 就是这个。
- **`tent_notifi_record` 的 `handle` 永远是 0**（`transfer_engine_c.cpp:298`），C API 不告诉你通知来自哪个段。这是既有 ABI，不是笔误。

### 9. 文档与代码的已知偏差

| 位置 | 说法 | 实际 |
|---|---|---|
| `docs/source/design/tent/failover.md:9,21,76` | 提交期失败不重试 | `attemptSubmitStageFailover` 已实现（`impl.cpp:3039-3068`） |
| `control_plane.cpp:613-614` | 空响应是 ack | 现在会写失败原因 |
| `tent/include/tent/CMakeLists.txt` | install `transfer_engine_c.h` | 该文件不存在；C API 声明在 `transfer_engine.h`。且没有任何父 CMakeLists `add_subdirectory(include/)`，整个文件是死配置 |

## 精读清单

建议按下面顺序读，每步给出「读完应该能回答什么」。

1. **`docs/source/design/tent/overview.md`（142 行，10 分钟）**
   看懂三个设计决策（动态选择 / 遥测调度 / 运行时容错）和「应用只声明搬什么」。先建立意图，再读实现，否则会被 3800 行细节淹没。

2. **`tent/include/tent/common/types.h:29-258`**
   应能回答：`TransportType` 为什么不能重排？`UNSPEC` 有几种含义？`Request` 的哪些字段影响选路？`transferStatusSeverity` 为什么把未知值当 FAILED？

3. **`tent/include/tent/runtime/transport.h`（整份 200 行）**
   应能回答：`submitTransferTasks` 的 all-or-nothing 约定具体禁止什么？`SubBatch` 上哪些字段把策略与进度信息透传到后端？`quiesce()` 为什么必须在 buffer 回收前调？`supportsCancellation` 的 best-effort 到底 best-effort 在哪？

4. **`tent/include/tent/runtime/transfer_engine_impl.h`**
   重点 `TaskInfo`（73-110）、`DeferredStageTeardown`（562-571）、`batch_shards_` 与锁序注释（626-686）。应能回答：`xport_priority` 与 `failover_count` 分工是什么？`failure_stage` 的 first-failure-wins 为什么重要？为什么 runtime queue 开启后热路径退回 `progress_mutex_`？

5. **`tent/src/runtime/transfer_engine_impl.cpp:369-591`（`construct`）**
   对照本章「流程 A」逐步走。重点：`rpc_server_threads` 的条件默认值、HP_TCP 失败回滚、临时 notify 回调与 install 接管。应能回答：为什么 `ControlService::start()` 必须在 notify 回调注册之前？

6. **`tent/src/runtime/transport_selector.cpp:84-621`**
   先看 `getDefaultPolicies`，再看 `loadPolicies`，然后 `matchesPolicy` / `isTransportAvailable` / `select` / `reorderWithHint`。应能回答：没有 matching policy 会怎样？hint 不在候选表里会怎样？hint 在候选表里但后端不可用会怎样？`device_mask` 什么时候解析、失败怎么办？

7. **`tent/src/runtime/transport_loader.cpp`（181 行，快读）**
   建立「18 个后端 × 编译宏 × 配置默认值」的表。应能回答：哪些默认开？哪些默认关？RDMA 除了 enable 还要求什么？

8. **`tent/src/runtime/transfer_engine_impl.cpp:2091-2421`（`prepareSubmit` + `commitPreparedSubmit`）**
   这是全章最密的一段。配合本章「流程 B」读。应能回答：合并的条件有哪些（尤其是哪些字段不能合并）？为什么 RDMA/UB 要按 `(device_mask, qp_pool)` 分组？同步提交失败后为什么只对物理 owner 做 failover？`sub_task_id` 为什么从 `sub_batch->size()` 取而不是累加？

9. **`tent/src/runtime/transfer_engine_impl.cpp:2966-3200`（`resubmitTransferTask` / `attemptSubmitStageFailover` / `pollTaskStatus` / `updateTaskStatusAfterPoll`）**
   配合 `docs/source/design/tent/failover.md` 读，并在第 76 行处停下来记住「这条 Known Gaps 已过时」（同文档 129/188/193 行已按实现描述提交期 failover）。应能回答：哪些错误 `suppress_failover`？TCP 的 ACK 丢失为什么不能重试？提交期与轮询期两条 failover 路径如何共用同一张有序候选表？

10. **`tent/src/runtime/transfer_engine_impl.cpp:3202-3344` + `tent/src/runtime/control_plane.cpp:179-191,402-431,608-651`**
    两条一起读。应能回答：四条投递路径的触发条件各是什么？`NotImplemented` 为什么被跳过？`kNotifyCallbackFailed` → `InternalError` 与 `kMalformedNotification` → `InvalidArgument` 的区分依据是什么？`setNotifyCallback` 的 in-flight 排空解决哪个竞态？`receiveNotification` 为什么必须排空所有队列？

11. **`tent/include/tent/transport/rdma/endpoint.h:125-184` + `tent/src/transport/rdma/rdma_transport.cpp:971-1140`**
    应能回答：`notify_qp_num` 怎么交换？`shouldRearmNotifyRecv` 的两个条件为什么缺一不可？`classifyNotifyCompletion` 的四种处置分别对应什么故障？`notify_inflight_` 为什么必须在 QP 销毁前归零？

12. **`tent/src/common/config.cpp:165-264` + `tent/include/tent/common/config.h:34-184`**
    应能回答：`MC_TENT_CONF` 的「先文件后内联」怎么判断？旧 `MC_*` 环境变量映射到哪些路径键（这张表就是新旧引擎兼容的实际边界）？为什么 bool 型环境变量不能用 `setFromString`？`freeze()` 与 `RuntimeConfigSnapshot` 的关系？

13. **`mooncake-transfer-engine/src/transfer_engine.cpp:447-492,534-542,642-743,752-764`**
    理解两套引擎并存：`MC_USE_TENT` 开关、`buildTentConfig` 的翻译规则、`TentCompatibilityTransport` 为什么存在、`getLocalIpAndPort` 为什么返回段名。

14. **（可选，按需）**
    - `tent/src/runtime/topology.cpp:349-391`：三种拓扑来源的优先级。
    - `tent/include/tent/runtime/proxy_manager.h` + `impl.cpp:1390-1433`：staging 与 deferred teardown。
    - `tent/src/runtime/memory_prober.cpp:33-94`：确认 device plugin 是休眠路径。
    - `tent/src/runtime/admission_queue.cpp` / `tent/include/tent/runtime/deadline_mlu.h` / `tent/src/runtime/progress_worker.cpp`：opt-in 调度层。
    - `tent/tests/notification_fallback_test.cpp` + `engine_failover_e2e_test.cpp`：把前面读到的语义钉成断言。

15. **收尾自测**：合上文档，画一张图回答——
    「一个带 `transport_hint=RDMA` 的请求，从 `submitTransfer` 到 `getTransferStatus` 返回 COMPLETED，中间经过哪些函数？如果 RDMA 在 poll 期失败而 TCP 可用，状态怎么走？」能画出来，本章就通关了。

---

# 06 Store Master 服务与放置策略

> 精读范围：`mooncake-store` 的控制面——`master` 可执行入口、`MasterService` / `WrappedMasterService` / `MasterAdminServer`、对象元数据组织、副本放置与分配策略、segment 挂载与容量管理。
> 所有 `路径:行号` 相对仓库根 `D:/xiaojun/work/Mooncake`，以 main 分支 `67df8c41` 为准。
> 本章刻意不展开 HA 的选主/OpLog/Snapshot 细节（见第 08 章）与客户端数据面（见第 07 章），只在边界处交代接口。

## 定位与职责

Mooncake Store 是"控制面 / 数据面分离"的分布式 KV cache。Master 是控制面上唯一的权威：

- 它知道 **每个 key 的对象元数据**（谁写的、多长、有几个副本、每个副本在哪个介质/哪台机器/哪个虚拟地址、租约与 pin 状态）；
- 它**从不搬运对象字节**。`GetReplicaList` 返回的是一串 `Replica::Descriptor`（传输端点 + 远端地址 + 大小 + 协议），客户端拿着描述符经 Transfer Engine 直连持有数据的那个 Client。`master_service.cpp` 里出现的 `te_endpoint`、`buffer_address_` 都只是**寻址信息**（`types.h:463`、`allocator.h:143-150`），不是数据通道。
- 它同时是**集群容量账本**：segment 是 Client 贡献出来的一块 RDMA 可寻址内存（或 NoF SSD 区域），Master 为每块 segment 建一个 `BufferAllocatorBase`，据此分配/回收空间、统计水位、驱动淘汰。

把它当成"一个三件事的单点"来读最省力：

| 职责 | 主要载体 | 本章小节 |
|---|---|---|
| 对象元数据的组织与并发保护 | `std::array<MetadataShard, 1024> metadata_shards_` | 核心数据结构 1–4 |
| 副本放置决策 | `AllocationStrategy` + `AllocatorManager` + `SegmentAllocatorRegistration` | 核心数据结构 6–8、流程 1 |
| 空间注册、容量与生命周期 | `SegmentManager` / `NoFSegmentManager` + `ClientLivenessRecord` | 核心数据结构 5–6、流程 2 |

围绕这三件事还有一堆"控制面副产物"：两阶段写协议、lease / soft pin / group TTL、近 LRU 淘汰、多租户配额、promotion / offload / copy / move / drain 异步任务、OpLog 与 standby restore、HTTP 运维面。这些在本章只在它们侵入主干时展开。

一句话概括数据流方向：

```
Client ──PutStart(key,size,config)──> Master: 校验/配额/选 segment/占位/发布占位元数据
Client <──descriptor[]─────────────  Master
Client ──TE write 到 descriptor────> 另一个 Client 的 segment（Master 不参与）
Client ──PutEnd(key)───────────────> Master: 副本置 COMPLETE、撤销 processing、写 OpLog
Client ──GetReplicaList(key)───────> Master: 过滤可读副本 + 授读租约
Client <──descriptor[]─────────────  Master
Client ──TE read ─────────────────> 持有副本的 Client
```

**单点边界**：非 HA 模式下 `main` 直接起 `coro_rpc_server`，Master 进程即单点（`master.cpp:1703-1768`）；HA 模式下 `main` 把控制权交给 `MasterServiceSupervisor`（`master.cpp:1674-1702`），由它做选主、standby catch-up 与 leader 门控，但**用的是同一个 `WrappedMasterService` 实现**。所以"Master 内部逻辑"与"谁是 leader"是两件事——这是读本章时必须先建立的心智模型。

## 目录结构速览

| 文件 | 一句话职责 |
|---|---|
| `mooncake-store/src/master.cpp`（1769 行） | `mooncake_master` 入口：gflags/配置文件解析、校验、可选内嵌 `HttpMetadataServer`、HA 与非 HA 两条启动路径、RPC 服务与优雅关停 |
| `mooncake-store/include/master_service.h`（2471 行） | `MasterService` 声明 + 全部内部结构（`MetadataShard`/`TenantState`/`ObjectRecord`/访问器/锁顺序注释） |
| `mooncake-store/src/master_service.cpp`（13771 行） | 上帝对象实现：元数据 CRUD、两阶段 Put、分配、淘汰、租户配额、drain/copy/move、restore |
| `mooncake-store/include/rpc_service.h` | `WrappedMasterService`：对外完整 RPC 接口面（约 70 个方法，含 Batch 变体） |
| `mooncake-store/src/rpc_service.cpp` | 每个 RPC 的"租户解析 + 计时 + 指标"包装，末尾 `RegisterRpcService` 批量注册 |
| `mooncake-store/include/rpc_worker_pool.h` | `RpcWorkerPool` + `PooledHandler`：把 handler 从 io 线程挪到 worker 线程池 |
| `mooncake-store/include/master_admin_service.h` / `src/master_admin_service.cpp`（1490 行） | `MasterAdminServer`：coro_http 运维端点、指标序列化线程、HA 状态门控 |
| `mooncake-store/include/master_config.h`（1593 行） | `MasterConfig`（gflags/文件层）+ `WrappedMasterServiceConfig` + `MasterServiceConfig` + Builder |
| `mooncake-store/include/allocation_strategy.h`（732 行） | `AllocatorManager`、`AllocationStrategy` 抽象、Random/Ranked/FreeRatioFirst/SsdFreeRatioFirst/Cxl、工厂 |
| `mooncake-store/src/allocation_strategy.cpp`（69 行） | 只实现 `SsdFreeRatioFirst` 的两个打分重载与一个转发重载——算法本体全在头文件里 |
| `mooncake-store/include/segment.h`（572 行） | `SegmentManager`/`NoFSegmentManager` + `ScopedSegmentAccess`/`ScopedAllocatorAccess`/`SegmentView`/`SegmentSerializer` |
| `mooncake-store/src/segment.cpp`（1594 行） | 挂载/重挂/两阶段卸载、host 亲和排序、六个并行索引的一致性维护、CXL 全局分配器 |
| `mooncake-store/include/segment_allocator_registration.h` / `src/segment_allocator_registration.cpp` | 把 `BufferAllocatorBase` 与 `ClientLivenessRecord` 绑定，提供带双重存活检查的 `Allocate` |
| `mooncake-store/include/client_liveness.h` | `ClientLivenessRecord`：ACTIVE/SUSPECTED/OFFLINE 三态 + Serving/Retaining 两种 guard |
| `mooncake-store/include/lease.h` | `Lease`：单个 `atomic<int64_t>` 的单调 max 截止时间（group 共享 TTL 的基础） |
| `mooncake-store/include/object_metadata.h` | `ObjectMetadata`：`client_id`/`put_start_time`/`parts_`/`lease_`/`quota_ledger`/动态副本 |
| `mooncake-store/include/replica.h` | `Replica`、`ReplicaStatus`、五种 `Descriptor`、`ReplicateConfig`、`DetermineReplicaWriteMode` |
| `mooncake-store/include/types.h` / `src/types.cpp` | `ErrorCode`、`Segment`、`NoFSegment`、`UUID`、`AllocationStrategyType`、`BufferAllocatorType` 与错误码↔字符串 |
| `mooncake-store/include/metadata_store.h` / `src/metadata_store.cpp` | **HA/Standby 专用**的可序列化元数据容器（不是运行时索引，见"易踩坑"） |
| `mooncake-store/include/tenant/*` | 多租户配额表、policy store、sharded quota table |
| `mooncake-store/include/config/*` + `src/config/*` | 组件化配置（RPC 协议、metrics bootstrap、DFS、CXL…），`master.cpp` 在启动时逐项解析 |
| `mooncake-store/include/placement/*`、`include/segment/pool.h`/`catalog.h`/`region*.h`、`include/metadata/{object_route,tenant,tenant_registry,group_index}.h` | **下一代**放置/segment/元数据路由原型，已被编进 master 二进制但**生产路径未实例化**（见"易踩坑"第 1 条） |

## 核心数据结构

### 1. 锁顺序：先读这段注释，再读其它一切

`master_service.h:101-118` 的类头注释给出了**全局锁序**，任何改动都以此为准：

```
1. client_mutex_
2. tenant_quota_manager_ 的 policy mutex
3. master_state_mutex_
4. metadata_shards_[i].mutex
5. tenant_quota_manager_ 的 recompute mutex
6. ShardedTenantQuotaTable 内部 mutex  或  segment_mutex_
7. soft_pin_deadline_index_ 的 mutex
```

两条额外约束值得单独记：

- 需要同时拿 policy mutex 与 `master_state_mutex_` 的"租户准入/策略变更"路径，**必须先 policy 后 state**（`master_service.h:112-114`）。
- 第 6 级的两个锁被刻意设计成**永不嵌套**：recompute mutex 负责把容量快照与配额表更新串行化，`segment_mutex_` 在进入 quota table 之前就已释放（`master_service.h:115-117`）。`MasterService::UnmountSegment` 就是照这个顺序写的：先 `PrepareUnmountSegment`（拿 segment_mutex_，出作用域释放），再 `ClearInvalidHandles`，最后 `CommitUnmountSegment`（`master_service.cpp:2837-2885`）。

`master_state_mutex_` 是第二层"大闸"：它是 `std::shared_mutex`，读路径全走 shared，写路径多数也走 shared（真正的互斥靠 shard 锁），只有少数"全局状态变更"走 unique：`GracefulUnmountSegment`（`master_service.cpp:2904`）、`UnmountLocalDiskSegment`（`:8855`）、`ReMountSegment` 的 host-id 快照段（`:819`）以及 restore（`:3781`）。看到 shared 别误以为是只读——普通 `UnmountSegment` 虽然只拿 shared，仍靠 `segment_mutex_` 与元数据扫描互斥；unique 则用来制造"要么全在之前、要么全在之后"的一致性窗口。

### 2. ObjectIdentity / ObjectRecord / TenantState / MetadataShard

`master_service.h:1093-1135`：

```cpp
struct ObjectIdentity { TenantId tenant_id; std::string user_key; };   // 1093
static constexpr size_t kNumShards = 1024;                            // 1098

struct ObjectRecord {                                                 // 1102
    route::Generation generation;
    ObjectMetadata metadata;
    route::ObjectState state;
};
using ObjectMap = std::unordered_map<std::string, ObjectRecord>;      // 1113

struct TenantState {                                                  // 1115
    TenantQuotaHandle quota_account{nullptr};
    ObjectMap metadata;
};

struct MetadataShard {                                                // 1123
    mutable SharedMutex mutex;
    std::unordered_map<TenantId, TenantState, TenantIdHash> tenants GUARDED_BY(mutex);
    long disk_object_count GUARDED_BY(mutex) = 0;
    route::Generation next_generation GUARDED_BY(mutex) = 1;
};
std::array<MetadataShard, kNumShards> metadata_shards_;               // 1135
```

层级是 **shard → tenant → key → ObjectRecord**，即"三级索引"：

- **为什么 1024 个 shard**：`kNumShards` 是定值而非配置，`metadata_shards_` 是一个 `std::array`，没有任何动态分配——这让"按 key 找到锁"退化为一次取模，且内存布局稳定。代价是写路径会按 shard 争用（见易踩坑第 8 条）。
- **为什么 tenant 在 shard 内部**：租户是哈希维度的一部分，而不是一个全局表。`TenantState` 很薄：一个配额句柄 + 一张对象表。`quota_account` 是**稳定句柄**，配额策略变更时原地更新而不换句柄，所以 shard 内持有它是安全的（`metadata/tenant.h:19-25` 的注释描述的是同一设计，虽然那个类本身没被 master 使用）。
- **`generation`**：每次发布自增（`MetadataShardAccessorRW::NextGeneration()`，`master_service.h:1235`）。同一 key 永远落在同一 shard，所以"key 的 generation 不会重复"（`master_service.h:1131-1133` 注释）。它用于"我判定的那个对象是否还是当前对象"这类跨释放判断——`MetadataAccessorRW(service, tenant_id, ref)` 构造时会比对 generation，不匹配则视为未命中（`master_service.h:1766-1782`）。
- **`route::ObjectState`**：把一个 key 的所有 in-flight 任务（replication / offloading / promotion / dynamic replica / promotion candidate）收进 `HeapOptional` 字段，另有一个 `is_processing` 布尔与一个 dynamic-replication cooldown 时间点；绝大多数对象只多几个空指针（`metadata/object_state.h:18-37`）。这比老设计（TenantState 里挂一堆散装 map）省内存，但**注释与实现分离**：老代码里 `processing_keys_` / `replication_tasks_` 之类已不存在。
- **`disk_object_count`**：只计"至少有一个 COMPLETE 的 LOCAL_DISK 副本"的对象数，用来从淘汰分母里剔除纯磁盘对象（`master_service.h:1127-1130`）。维护点在 `MetadataShardAccessorRW::OnDiskReplicaAdded/Removed`（`master_service.h:1240-1263`），调用方必须成对。

**分片函数**（`master_service.h:1337-1350`）：

```cpp
if (tenant_id.IsDefault())
    return std::hash<std::string>{}(user_key) % kNumShards;      // 1340
size_t seed = std::hash<std::string>{}(tenant_id.value());
boost::hash_combine(seed, user_key);
return seed % kNumShards;                                        // 1344
```

default 租户**不做 hash_combine**，非 default 才把租户 id 混进去。效果是：默认单租户部署下分片完全兼容历史行为（同一 key 与旧版本落同一 shard），而多租户下同 key 在不同租户落到不同 shard。讲多租户路由时不要一句"hash(tenant,key)"带过。

### 3. 访问器：把"锁 + 查找 + 清理"打包成 RAII

这是本章最容易被低估的设计。核心不是"加锁"，而是**访问器在构造期就把引用失效的副本清掉**。

**`MetadataAccessorRW`**（`master_service.h:1741-1935`）：

| 成员 / 方法 | 语义 |
|---|---|
| 构造 `(service, object_id)` | 按 `getShardIndex` 拿 shard **写锁**，`Locate()` 找到 tenant 与 key；若已发布，立即 `CleanupInvalidMemoryReplicas`（1744-1755） |
| 构造 `(service, object_id, CreateKey{})` | 额外 `EnsureTenantState()`——**key 不存在的写路径**也持有 shard 写锁（1757-1764） |
| 构造 `(service, tenant_id, ref)` | generation 校验版，只在"还是那次发布"时持有关键（1766-1782） |
| `IsPublished()` / `Exists()` | 前者=表里有这条记录；后者=记录且 `metadata.IsValid()`（至少一个有效副本） |
| `Publish(args...)` | `assert(!IsPublished())`，取新 generation 并 `emplace`（1838-1851） |
| `Erase(...)` | 走 `EraseMetadata` 连带配额/任务清理，之后 `it_=end()` 但**仍持有 key**，允许立刻重新 Publish（1855-1865） |
| `QuotaAccount()` | 从 tenant_state 取绑定的 `TenantQuotaHandle`（1819-1821） |
| 析构 | `MaybeEraseEmptyTenant()`：访问器创建/清空的租户若已空则删除（1785、1919-…） |

关键点：**"持有 key"等于"持有该 shard 的写锁"**。头文件明确写了 "Until the store moves to per-key locks, holding a key means holding its metadata shard's write lock, so nothing in the accessor's scope may reach another key through another accessor"（`master_service.h:1737-1740`）。这条约束是很多"看起来可以顺手查另一个 key"的优化做不了的原因，也是死锁防线。

**`MetadataAccessorRO`**（`master_service.h:1940-1996`）：只在构造时拿 shard **共享锁**，内联缓存 tenant/key 迭代器；`Get()` 返回 `const ObjectMetadata&`。它**不做** `CleanupInvalidMemoryReplicas`（读路径不改状态）。

### 4. ObjectMetadata / Replica / Descriptor

`ObjectMetadata`（`object_metadata.h:46-160`）的关键字段：

- `client_id`：**写入者**的 UUID，`PutEnd` 用它校验 `ILLEGAL_CLIENT`（`master_service.cpp:5676-5680`）。
- `put_start_time`：配合 `put_start_discard_timeout_sec_` 判断"同 key 的上一次 PutStart 是否已超时、可被顶掉"（`master_service.cpp:5623-5626`）。
- `parts_`（`ObjectPart`）：目前实际只用 `GetPart(0)`，checksum 存在 part 上（`master_service.cpp:5776`）。
- `lease_`：`std::shared_ptr<Lease>`，group 内多个对象共享同一个 `Lease`（`RegisterGroupMember`，`master_service.h:1352-1359`）。`Lease` 本身只有一个 `atomic<int64_t> deadline_ns_`，`GrantReadLease` 做 CAS 单调 max（`lease.h:20-30`）。因此读路径**不需要 shard 写锁**就能延长 TTL；`ObjectMetadata::GrantReadLease` 只额外取一个保护 `lease_` 指针的极短 `SpinLock`（`object_metadata.h:313-318`）——这是 group TTL 既正确又便宜的原因。
- `quota_ledger`：租户配额的 pending/committed 账本，`InsertMetadata` 里 `AdoptPendingCharge`（`master_service.cpp:5374`）。
- `metadata.lease_` 在带 `group_id` 时被替换为 group 的共享 lease（`master_service.cpp:5396-5398`）。

`Replica`（`replica.h`）是"一个副本"的运行时对象：持有 `AllocatedBuffer`（MEMORY/NoF）或路径/端点信息（DISK/LOCAL_DISK/DFS），带 `ReplicaStatus`（正常路径 `INITIALIZED→PROCESSING→COMPLETE`，另有 `UNDEFINED`/`REMOVED`/`FAILED`，`replica.h:56-62`）与全局自增 `ReplicaID`（`next_id_.fetch_add(1)`）。

`Replica::Descriptor` 是一个 `std::variant`（`replica.h:674-677`）：

| 介质 | Descriptor | 客户端据此做什么 |
|---|---|---|
| MEMORY | `MemoryDescriptor{buffer_descriptor}`（远端地址 + te_endpoint + protocol） | TE RDMA/TCP 直读该 Client 的 segment |
| NOF_SSD | `NoFDescriptor{buffer_descriptor}` | 同上，走 SPDK/NoF 路径 |
| DISK | `DiskDescriptor{file_path, object_size}` | 读 Master 所在节点的本地文件（单机持久化） |
| LOCAL_DISK | `LocalDiskDescriptor{client_id, object_size, transport_endpoint}` | 向持有者 Client 发起 offload/读请求 |
| DFS | `DistributedFSDescriptor{file_path, offset, object_size, aligned_size, shard_idx}` | 走分布式文件系统 |

注意 `LocalDiskDescriptor` **不含 buffer 地址**——数据在 Client 自己的盘上，Master 只知道归属与端点。

### 5. SegmentManager：一把锁下的六个并行索引

`SegmentManager`（`segment.h:380-479`）的私有成员：

```cpp
mutable std::shared_mutex segment_mutex_;                        // 455
std::shared_ptr<AllocationStrategy> allocation_strategy_;        // 456（注意：master 实际不用这个字段）
const BufferAllocatorType memory_allocator_;                     // 457
const bool enable_cxl_;                                          // 461
std::shared_ptr<BufferAllocatorBase> cxl_global_allocator_;      // 462
AllocatorManager allocator_manager_;                             // 464，只含 OK 状态的 allocator
std::unordered_map<UUID, MountedSegment> mounted_segments_;      // 466  segment_id -> 记录
std::unordered_map<UUID, std::vector<UUID>> client_segments_;    // 468  client_id -> segment_ids
std::unordered_map<std::string, UUID> client_by_name_;           // 470  name -> client_id（有损）
std::unordered_map<std::string, UUID> segment_id_by_name_;       // 472  name -> segment_id
HostSegmentIndex segments_by_host_;                              // 473  host -> name -> {segment_id}
```

`HostSegmentIndex = std::map<std::string, std::map<std::string, std::set<UUID>>>`（`segment.h:25-26`）。**用 `std::map` 而不是 unordered 是有意的**：`BuildHostOrderedSegments` 要对 host 做环形遍历，需要稳定有序（`segment.cpp:49-92`）。

`MountedSegment`（`segment.h:28-33`）= `Segment` + `SegmentStatus` + `buf_allocator` + `allocator_registration`。后两者都是 `shared_ptr`：卸载时 **`registration->Invalidate()` 让所有快照上的分配失效**，然后 reset 成员；正在进行的 deallocate 持有局部 shared_ptr，会晚于卸载完成（`segment.cpp:449-456` 注释）。

`SegmentStatus`（`segment/status.h:7-14`）：`UNDEFINED / OK / DRAINING / DRAINED / GRACEFULLY_UNMOUNTING / UNMOUNTING`，并给出 `SegmentStatusAvailabilityRank`（`segment/status.h:30-46`）用于比较"哪个更不可用"。

`NoFSegmentManager`（`segment.h:481-570`）结构相同但少了 host 索引与 name 重索引逻辑（NoF 不做 host 亲和）。

### 6. AllocatorManager + SegmentAllocatorRegistration：分配安全的两道闸

`AllocatorManager`（`allocation_strategy.h:33-208`）不持锁，注释明确 "External synchronization is required … provided by `SegmentManager`'s `segment_mutex_`"（`:29-31`）。内部：

- `names_`：`std::vector<std::string>`，**为了随机取而不必遍历 map**（`:199-200`）。移除时用 swap-and-pop（`:94-98`）。
- `allocators_`：`name -> vector<shared_ptr<SegmentAllocatorRegistration>>`。一个 segment 名可以对应**多个** registration（同名 segment 被多个 Client 挂载；CXL 场景尤为明显）。
- `owner_by_name_`：只由 `Snapshot(owners)` 填充，供 `SsdFreeRatioFirst` 反查 owner client。
- `getServingNames()`（`:160-173`）：返回"至少有一个 serving registration"的名字。**每次调用都重建 vector**，代价 O(segments × registrations)。它每次 PutStart 至少跑两遍：`AllocateReplicas` 为计算 `has_enough_memory_segments` 在全量 manager 上算一次（`master_service.cpp:5176-5178`），随后策略又在快照上算一次（如 `allocation_strategy.h:317`、`:479`）。

`SegmentAllocatorRegistration`（`segment_allocator_registration.h:14-37`）是真正的安全闸，核心是 `Allocate`（`segment_allocator_registration.cpp:22-45`）：

```
1. allocation_lifetime_.isAvailable()?          否则返回 nullptr
2. client_liveness_ 已绑定且 !IsServing()？      否则返回 nullptr（SUSPECTED/OFFLINE 都拒绝）
3. buffer = GetAllocator()->allocate(size)      失败返回 nullptr
4. buffer 绑定 segment 生命周期 + client liveness（供后续 deallocate/校验）
5. 再检查一次：allocation_lifetime_ 可用、
   client_liveness_ 指针未变、状态仍 serving —— 任一不满足就丢弃 buffer
```

第 5 步是**分配后复检**：分配过程中 segment 可能被卸载、Client 可能被判离线；此时分配出来的 buffer 不能再交给调用方（否则会指向一块即将失效的内存）。丢弃 `unique_ptr` 是否即刻回收取决于 allocator 实现，但至少**不会泄漏到元数据里**。

`SegmentLifetime` 本身只是 `shared_ptr<atomic<bool>>`（`allocator.h:50-68`），`operator==` 比较指针身份——因此 `OwnsBuffer` 可以靠 `buffer.segment_lifetime_ == buffer_lifetime_` 判断归属（`segment_allocator_registration.cpp:70-73`）。

### 7. AllocationStrategy 家族

抽象基类 `AllocationStrategy`（`allocation_strategy.h:220-283`）有两个重载：

- `Allocate(const AllocatorManager&, slice_length, replica_num, preferred, excluded, replica_type)` —— **纯虚**，master 主路径用这个。
- `Allocate(const ScopedAllocatorAccess&, …)` —— 非纯虚，默认转发到上面；只有 `SsdFreeRatioFirst` 覆写它，因为打分需要 `GetOwnerClientId`（必须持锁读 `client_by_name_`）。

具体策略：

| 类 | 打分/选择 | 位置 |
|---|---|---|
| `RandomAllocationStrategy` | 先按 `preferred_segments` 顺序尝试，再从一个随机起点环形扫过全部 serving 名，跳过 excluded/used，最多重试 `min(100, names.size())` | `allocation_strategy.h:303-399` |
| `RankedAllocationStrategy`（模板方法 `AllocateRanked`） | 先 preferred；再取 `min(6×remaining, names.size())` 个候选（**从随机起点取连续环窗口，不是均匀采样**）按 score 降序尝试；不足则随机兜底扫 | `allocation_strategy.h:461-571` |
| `FreeRatioFirstAllocationStrategy` | score = 该 segment 名下所有 serving allocator 的聚合空闲率 | `allocation_strategy.h:573-617` |
| `SsdFreeRatioFirstAllocationStrategy` | score = **owner client 的 SSD 空闲率**（`LocalSsdManager::GetUsage`），未知按 1.0 | `allocation_strategy.cpp:18-67` |
| `CxlAllocationStrategy` | 只认 `preferred_segments[0]`，从全局 CXL allocator 分配一个，并 `buffer->change_to_cxl(name)` | `allocation_strategy.h:648-708` |

工厂 `CreateAllocationStrategy`（`allocation_strategy.h:713-730`）：`RANDOM` / `LOCAL_FIRST` / default 全部返回 `RandomAllocationStrategy`。**LOCAL_FIRST 不是一个独立的分配算法**——它的"本地优先"由 `AllocateReplicas` 注入 host-ordered `preferred_segments` 实现（见流程 1）。这是个非常容易看漏的点。

`CxlAllocationStrategy` 有个不显眼的行为：即使 `replica_num > 1`，它也只返回 **1 个** replica（`allocation_strategy.h:678-700` 只 emplace 一次）。constructor 在 `enable_cxl` 时把策略强制为 CXL（`master_service.cpp:210-212`），所以 CXL 部署下副本数不会按请求翻倍。

分配语义（抽象类注释 `allocation_strategy.h:215-247`）：**best-effort，至少 1 个，否则 `NO_AVAILABLE_HANDLE`**；同一 slice 的多副本保证落在不同 segment（靠 `used_segments`），不同 slice 可复用同一 segment。

### 8. Group / SoftPin / 其它辅助索引

- `GroupDomain`（`master_service.h:1140-1153`）：**全局一张表**（不是 shard 化），key 是 `tenant_id.MakeScopedKey(group_id)`，值是 `{member_keys, shared_ptr<Lease>}`。注释点明"路由仍是 hash(tenant,key)，group 只是成员集合 + 一个共享租约，淘汰时整体对待（all-or-none），读路径根本不碰这张表"（`:1137-1139`）。写路径低频，所以一把 `SharedMutex` 足够。
- `SoftPinDeadlineIndex`（`master_service.h:1157-1201`）：`priority_queue`（最小 deadline 在顶）+ `registrations_` map 做去重与删除，带 compaction（`kMinCompactionThreshold=4096`、2x 比例）。它是锁序第 7 级，**其实是一个定时器索引**，不是数据索引。
- `ShrinkBucketsIfSparse`（`master_service.h:91-99`）：`unordered_map` 删除不会缩 bucket，长期跑会保留历史峰值内存；因此在持有锁的前提下按 2x 目标 rehash。

### 9. MetadataStore 家族（HA 专用，务必区分）

`metadata_store.h` 里的东西**都不是 master 运行时的元数据索引**：

- `StandbyObjectMetadata`（`:27-51`）：可 `struct_pack` 序列化的对象元数据。**故意不存 lease 与 soft pin**（注释 `:31-36`）：standby 不淘汰，提升后也应重新授租约。
- `StandbySegmentInfo` / `StandbySnapshot` / `StandbyObjectEntry` / `MetadataPayload`：快照与 payload 结构，字段用 `struct_pack::compatible<T, N>` 做向后兼容。
- `StandbySegmentRegistry`（`:135-154`）：按 `transport_endpoint` 索引的 standby 侧 segment 视图，由 SEGMENT_MOUNT/UNMOUNT/UPDATE 事件维护。
- `MetadataStore`（`:163-219`）：抽象接口（tenant-aware Put/Get/Remove/Exists + weight metadata）。实现是 `StandbyMetadataStore` / mock。

运行时真正的索引只有 `master_service.h:1135` 的 `std::array<MetadataShard,1024>`。

### 10. 配置层

三层配置对象：

1. `MasterConfig`（`master_config.h:95-…`）：gflags + 配置文件合并后的"扁平大结构"，字段与命令行开关一一对应。
2. `WrappedMasterServiceConfig`：`WrappedMasterServiceConfig(const MasterConfig&, ViewVersionId)`（`master_config.h:622`）把字符串开关转成强类型（如 `"cachelib"` → `BufferAllocatorType::CACHELIB`），并补齐派生项（如 `ResolveConfiguredHABackendConnstring`）。
3. `MasterServiceConfig`（`master_config.h:1190`）+ `MasterServiceConfigBuilder`（`:845`）：`MasterService` 真正接收的配置，带默认值的类，供测试/嵌入式直接构造（Builder 目前只在头文件里定义，仓库内未见调用点）。

默认值里两个容易忽略：`memory_allocator = OFFSET`、`allocation_strategy_type = RANDOM`（`master_config.h:1255-1257`，`MasterServiceConfig` 段；`WrappedMasterServiceConfig` 的同名默认值在 `:594-596`）。`master.cpp` 只允许 `cachelib|offset` 两种 allocator，否则 `LOG(FATAL)`（`master.cpp:1518-1524`）。

## 关键流程剖析

### 流程 1：PutStart → PutEnd（两阶段写 + 放置决策）

调用链：

```
RPC "PutStart"
 └ WrappedMasterService::PutStart                     rpc_service.cpp:428-455
    └ WithWriteTenant(tenant_id, IsTenantQuotaEnabled())   rpc_service.cpp:67-75
       └ MasterService::PutStart                      master_service.cpp:5589-5661
          ├ PrepareWriteStart                         master_service.cpp:5442-5527
          ├ AdmitWithDfsRecovery(admit)               master_service.cpp:5562-5587
          │  └ admit lambda                           master_service.cpp:5601-5651
          │     ├ (shared) client_mutex_ → master_state_mutex_
          │     ├ GetRetainingClientIdsLocked
          │     ├ MetadataAccessorRW(CreateKey)       ← 拿 shard 写锁
          │     ├ ChargeTenantQuota(account, 0)
          │     ├ CleanupStaleBeforeWrite
          │     ├ [已发布] 判超时/已 COMPLETE → OBJECT_ALREADY_EXISTS
          │     │            否则弹出 processing 副本进 discarded_replicas_，Erase
          │     └ AllocateAndInsertMetadata           master_service.cpp:5404-5440
          │        ├ ChargeTenantQuota(account, requested)
          │        ├ AllocateReplicas                 master_service.cpp:5140-5320
          │        └ InsertMetadata                   master_service.cpp:5322-5402
          └ [失败且 TENANT_QUOTA_EXCEEDED] 打指标
```

分步说明（重点是"决策点在哪、锁在哪"）：

**① 前置校验（`PrepareWriteStart`，5442-5527）** —— 全在拿任何锁之前：

- 三者副本数全 0 或 key 空或长度 0 → `INVALID_PARAMS`（5446-5455）。
- `dfs_replica_num > 1` 拒绝（5456）；DFS 只支持 default 租户（5462）；DFS 未初始化 → `DFS_SERVICE_UNAVAILABLE`（5467）。
- `prefer_alloc_in_same_node && nof_replica_num > 0` 直接拒绝（5475-5482）——NoF 无本地亲和概念。
- 编译期没开 `USE_NOF` 时，任何 `nof_replica_num > 0` 都拒绝（5483-5490）。
- `ResolveSoftPinRequest(config)` 解析 soft-pin 三态（5492）。
- **`writer_host_id` 的确定条件**（5499-5504）：
  ```cpp
  if ((allocation_strategy_type_ == LOCAL_FIRST || config.prefer_alloc_in_same_node)
      && config.replica_num == 1)
      writer_host_id = config.host_id.empty() ? GetClientHostId(client_id) : config.host_id;
  ```
  注意 `replica_num == 1` 这个门槛：**多副本请求不会注入本地优先**。这是 LOCAL_FIRST 的实际语义边界。
- CACHELIB 下 `slice_length > kMaxSliceSize` 拒绝（5506-5512），因为 cachelib slab 有硬上限。

**② 准入尝试（admit lambda，5601-5651）**

```cpp
std::shared_lock client_lock(client_mutex_);       // 锁序 1
std::shared_lock shared_lock(master_state_mutex_); // 锁序 3
auto retaining_clients = GetRetainingClientIdsLocked();
client_lock.unlock();                              // 提前放掉，只留 state 的 shared
MetadataAccessorRW accessor(this, object_id, CreateKey{});  // 锁序 4：shard 写锁
```

`retaining_clients` 是一个"资源仍需保留"的 Client 集合（`ShouldRetainResources()`：非 OFFLINE，即 ACTIVE 或 SUSPECTED），用于在清理时判断"这个副本的持有者是否已经走了"。拿着它跨整个写路径，注意它只是**某一时刻的快照**——这也是 `ClearLocalDiskHandlesOwnedBy` 采用"按 owner 定向清理"而不是"按补集清理"的原因（`master_service.h:1070-1076` 注释讲得很清楚）。

若 key 已发布：

- 已有 COMPLETE 副本，或 `put_start_time + put_start_discard_timeout_sec_ >= now` → `OBJECT_ALREADY_EXISTS`（5621-5627）。这就是"同名 PutStart 未超时不能顶掉"的保护。
- 否则（旧 PutStart 已超时）把 processing 副本 `PopReplicas` 出来，DFS 副本立即释放，其余放进 `discarded_replicas_` 并记一个 `put_start_release_timeout_sec_` 之后的释放时间（5635-5643），然后 `accessor.Erase()`（5644）——**仍在同一个 shard 写锁内**，因此"擦旧 + 建新"对其它写者原子。

**③ 分配（`AllocateReplicas`，5140-5320）**

```cpp
AllocatorManager allocator_snapshot;
{
    ScopedAllocatorAccess allocator_access = segment_manager_.getAllocatorAccess(); // segment_mutex_ 共享锁
    has_enough_memory_segments = allocator_access.getAllocatorManager()
                                     .getServingNames().size() >= config.replica_num;
    // 显式 preferred 在前；writer_host_id 非空时追加 host-ordered
    ...
    allocator_snapshot = allocator_access.SnapshotAllocatorManager();
}   // ← 出作用域即释放 segment_mutex_
auto allocation_result = allocation_strategy_->Allocate(
    allocator_snapshot, value_length, config.replica_num,
    preferred_segments, {}, ReplicaType::MEMORY);
```

三个设计点：

1. **先快照、后分配**。分配动作（allocator 内部可能是 offset/cachelib 的锁）不在 `segment_mutex_` 下做，避免拖慢 mount/unmount。安全性由 `SegmentAllocatorRegistration` 的 lifetime/liveness 检查兜住（核心数据结构 6）。
2. **`preferred_segments` 的拼装顺序**：显式 `config.preferred_segment`（旧字段）优先于 `config.preferred_segments`，二者互斥（5165-5171）；随后追加 `GetHostOrderedSegments(writer_host_id, key)`（5179-5192）。`append_preferred_segment` 会去重。
3. **`has_enough_memory_segments`** 是个"诊断用"布尔：它只回答"serving segment 数是否 ≥ 请求副本数"。分配失败时，只有它为真才设 `need_mem_eviction_ = true`（5208-5210）——segment 数本来就不够，淘汰也没用，不该触发淘汰线程。

`GetHostOrderedSegments` 的实现（`segment.cpp:49-92`）值得单独理解：从 writer 所在 host 开始（找不到就 `lower_bound`，再不行从头），按 `std::map` 顺序**环形遍历全部 host**；每个 host 内部按 segment 名有序，并用 `hash(key) % host_segments.size()` 旋转起点。结果是"本地优先 + 每 key 稳定打散"——既避免热点全落同机，又让同一 key 的多次分配落在同一处（缓存友好）。

**多介质拼装**：

- MEMORY 分配后，`#ifdef USE_NOF` 下再对 `nof_segment_manager_` 做一次分配（5219-5251），**用同一套 `allocation_strategy_`**（此时 preferred 用 `config.preferred_nof_segments`）。
- `use_disk_replica_` 直接追加一个 `DiskDescriptor` 形态的 PROCESSING 副本（5293-5298）。
- `dfs_replica_num > 0` 走 `dfs_allocator_->Allocate(key, value_length)`（5300-5317），并把"是否容量耗尽型失败"写回 `dfs_allocation_failed`，供上层做有界重试（`AdmitWithDfsRecovery`，5562-5587）。

**④ 严格度检查（`HasExpectedReplicaAllocation`，master_service.cpp:110-123）**

这段逻辑纠正了"分配总是 best-effort"的粗印象：

| 配置 | 判定 | 含义 |
|---|---|---|
| `nof=0 && dfs=0`（纯内存） | `allocated_memory_replicas > 0` | **best-effort**：要 3 个只给 1 个也算成功，只打 WARNING + `put_start_partial_allocations` 指标 |
| `FLEXIBLE_DUAL_REPLICA`（1 mem + 1 nof） | `mem + nof > 0` | 任一侧成功即可 |
| 其它（`nof>0` 且非 flexible） | 精确匹配 | 少一个就 `NO_AVAILABLE_HANDLE` |

注释（`master_service.cpp:92-109`）解释：纯内存多副本的"可靠"保证不在分配阶段兑现，而在**传输阶段**（所有已分配副本都必须写完，否则 revoke）。所以 `replica_num=2` 的纯内存 Put 有可能只拿到 1 个副本并成功。读代码时若不看这段，很容易把"partial allocation"当 bug。

**⑤ 发布（`InsertMetadata`，5322-5402）**

- 先遍历 replicas 生成 `Descriptor` 列表并收集 `eligible_replica_ids`（5343-5366）。
- `accessor.Publish(client_id, now, value_length, replicas, timeout, with_hard_pin, data_type, group_id, tenant_id, key)`（5368-5371）——**在 shard 写锁内 emplace 到 `tenant_state_->metadata`**，generation 自增。
- 多租户下 `quota_ledger.AdoptPendingCharge(...)`，失败要 `PopReplicas` + `FreeDfsReplicas` + `erase(accessor.it_)` 回滚（5373-5385）。
- `BeginSessionTag` / `BeginSoftPinAction`（5386-5388）、`soft_pin_deadline_index_.Upsert(...)`（5389-5393）。
- 有 `group_id` 时把 `metadata.lease_` 换成 group 的共享 lease（5396-5398）。
- `accessor.GetState().is_processing = true`（5399）—— **这是 `PutEnd` 的准入条件**。
- 回滚：`AllocateAndInsertMetadata` 在 `AllocateReplicas` 或 `InsertMetadata` 失败时都会 `ReleaseTenantQuota(pending_quota_charge)`（5427-5438），所以失败路径上的 pending 配额不会泄漏。

另外一个容易忽略的分支：`CleanupStaleBeforeWrite` 清完成 stale 句柄后，如果 `enable_oplog_` 为真，会直接返回 `OBJECT_ALREADY_EXISTS`（5549-5551），把"重建"交给客户端重试；只有未启用 OpLog 的路径才会就地清完继续本次写。`UpsertStart` 走的是同一段逻辑（6189-6193）。

返回的是 descriptor 列表，客户端据此写数据。此刻对象处于"PROCESSING"：`GetReplicaList` 会因为它不可读（`IsReplicaReadable` 要求 `is_completed()`）而返回 `REPLICA_IS_NOT_READY`。

**⑥ PutEnd（5663-5834）**

```cpp
std::shared_lock shared_lock(master_state_mutex_);
MetadataAccessorRW accessor(this, object_id);        // shard 写锁
if (!accessor.Exists()) → OBJECT_NOT_FOUND
if (client_id != metadata.client_id) → ILLEGAL_CLIENT
if (!accessor.InProcessing()) {
    // 幂等判定：目标介质的副本若已全 COMPLETE，视为重试成功
    → {} / INVALID_WRITE
}
```

随后按 `replica_type`（`ALL / MEMORY / NOF_SSD / DFS`）过滤目标副本：

1. DFS：先 `bucket_allocator_->MarkCommitted(key, descriptor)` 把这些 pending descriptor 转正，失败即 `FILE_WRITE_FAIL`（5725-5743）。
2. `VisitReplicas(is_processing && is_target)` 逐个 `CommitSessionTag` + `mark_complete()`，DFS 还 `UpdateAccess`（5748-5763）。
3. "本次提交让对象第一次拥有 COMPLETE 副本"时提交 pending soft pin（5765-5770）。
4. checksum 写到 part 0（5772-5777）。
5. `SettlePrimaryWriteQuotaIfReady` 结算租户配额（5779-5783）。
6. `enable_offload_ && !offload_on_evict_` 且没有进行中的 DFS 副本时，把 MEMORY 副本推进 offload 队列并记 `state.offloading_task`（5785-5811）。
7. 全部副本 COMPLETE 时清 processing 标志（5814-5817）。
8. `SyncCacheTotalAccounting`、`GrantReadLease(0ms)`、`PublishKvStored`、写 OpLog `PUT_END`（5819-5832）。

第 8 步的 `GrantReadLease(std::chrono::milliseconds::zero())`（5821）容易误读成"给个 0 租约"：由于 `Lease` 是单调 max，等价于把截止时间抬到"现在"，**不延长、也不缩短**已有读者租约。语义是"PutEnd 本身不为写者保留读租约"。

### 流程 2：MountSegment / UnmountSegment（segment 生命周期）

调用链（挂载）：

```
RPC "MountSegment"
 └ WrappedMasterService::MountSegment                 rpc_service.cpp:1002
    └ MasterService::MountSegment                     master_service.cpp:676-742
       ├ (unique) client_mutex_                       锁序 1
       ├ client_liveness_records_.try_emplace(client_id, new ClientLivenessRecord)
       ├ record->ObserveAndRun(now, [&]{
       │     (shared) master_state_mutex_              锁序 3
       │     ScopedSegmentAccess::MountSegment(...)    锁序 6，segment_mutex_ 独占
       │ })
       ├ [OK] oplog PersistSegmentOpForHAOrEnqueue(SEGMENT_MOUNT)
       └ [OK] UpdateClientHostId + RecomputeTenantEffectiveQuotas
```

`ScopedSegmentAccess::MountSegment`（`segment.cpp:169-274`）逐段：

1. **同名检查**（172-185）：`segment_id_by_name_` 里有这个名字但对应记录已不存在 → `INVALID_PARAMS`（offboarding 保留名字直到持久化卸载记录被接受）；同名记录处于 `UNMOUNTING` → `UNAVAILABLE_IN_CURRENT_STATUS`。
2. **同 id 幂等检查**（189-206）：已存在时必须 status=OK、确实归属本 client，且 name/base/size/te_endpoint/protocol 全部一致才返回 `SEGMENT_ALREADY_EXISTS`；否则 `INVALID_PARAMS`。注释强调这是"确认重放"的存活信号，对所有协议（含 CXL）统一（`segment.cpp:186-188`）。
3. **CXL 分支**（211-237）：`protocol == "cxl"` 时**不新建 allocator**，直接复用 `cxl_global_allocator_`（全局单例，多个 segment 记录共享），随后照常填六个索引。
4. **普通内存**：`buffer/size` 非 0（239），CACHELIB 下必须按 `Slab::kSize` 对齐（245-252），然后 `CreateBufferAllocator(...)`（254-260）。
5. **一致性写入**（262-271）：`AttachUsageTracker` → `allocator_manager_.addAllocator(name, allocator, liveness)` → `client_segments_[client].push_back(id)` → `mounted_segments_[id] = {segment, OK, allocator, registration}` → `client_by_name_[name]=client` → `segment_id_by_name_[name]=id` → `AddHostSegment(segments_by_host_, segment)` → `inc_total_mem_capacity`。

**六个索引必须在同一把 `segment_mutex_` 内一起更新**，否则"容量查询 / 归属反查 / 放置候选"会看到不一致的世界。这是 `SegmentManager` 存在的全部理由。

`ReMountSegment`（`segment.cpp:312-344`）是"逐个 Mount + 错误分级"：`UNAVAILABLE_IN_CURRENT_STATUS` / `INTERNAL_ERROR` 直接返回让客户端重试；`INVALID_PARAMS`、`SEGMENT_ALREADY_EXISTS` 与其它错误只记日志并继续，最后返回 OK。注释解释（326-340）：**不可解的错不该让客户端无限重挂**。`ValidateRemountSegment`（346-367）在挂载前比对已存在记录的全部字段（含 `host_id`），任何不一致即 `INVALID_PARAMS`。

**两阶段卸载**：

- `PrepareUnmountSegment`（`segment.cpp:420-461`）：从 `allocator_manager_` 移除 registration（若还在）→ `RemoveHostSegment` → `registration->Invalidate()` → reset `allocator_registration`/`buf_allocator` → `status = UNMOUNTING`。**不立刻 detach usage tracker**，因为在途 deallocate 还持有 shared_ptr，allocator 析构时 RAII 自己 detach（449-453 注释）。
- `CommitUnmountSegment`（`segment.cpp:544-594`）：从 `client_segments_` 摘掉（空了就删 client 条目）、`RemoveHostSegment`、`ReindexSegmentNameAfterRemoval`、`mounted_segments_.erase`、非 CXL 时 `dec_total_mem_capacity` + `remove_segment_metrics`。
- `ReindexSegmentNameAfterRemoval`（505-542）：因为**多个 Client 可能用同一个 segment 名**，删掉一个后要把 `segment_id_by_name_`/`client_by_name_` 重新指向剩下的同名记录（优先 OK 状态，退而求其次任意状态）。

`MasterService::UnmountSegment`（2837-2885）把两阶段串起来，中间插一步元数据清理：

```
PrepareUnmount → (enable_async_segment_cleanup_ ? replica_cleanup_worker_.Schedule()
                                               : ClearInvalidHandles(retaining_clients))
               → GetSegmentBasicInfo(缓存 name/endpoint)
               → CommitUnmount → oplog SEGMENT_UNMOUNT → RecomputeTenantEffectiveQuotas
```

注意 `enable_async_segment_cleanup_` 在构造时确定：`!enable_ha && !enable_cxl`（`master_service.cpp:167`）。HA/CXL 下必须同步清理，避免快照/共享 allocator 看到半清理状态。

`GracefulUnmountSegment`（2900-2935）：校验归属 → `PrepareGracefulUnmountSegment`（只 `SetAllocatable(false)` + 从 allocator/host 索引摘除，**保留 mounted 记录**，状态 `GRACEFULLY_UNMOUNTING`，`segment.cpp:463-503`）→ 交给 `graceful_unmount_scheduler_` 在 `grace_period_ms` 后回调真正的 `UnmountSegment`（构造参数见 `master_service.cpp:155-166`）。

**Client 失联时的卸载**由客户端监控线程驱动（`ClientMonitorFunc`，`master_service.cpp:11969-12088`）：

```
每 1000ms 快照 client_liveness_records_
  → record->EvaluateAndRetire(now, active_ttl, suspicion_ttl,
                              ReserveJob, RetireJob)
       ACTIVE --超 active_ttl--> SUSPECTED --超 suspicion_ttl--> OFFLINE
  → OFFLINE 的 retire 回调里：
       client_mutex_ 下擦 ok_client_/client_host_id_
       移除该 client 的 graceful unmount 预约
       对它的每个 segment 做 PrepareUnmountSegment，构造 ClientOffboardingJob
       交给 client_offboarding_worker_ 异步提交（ReserveJob/ScheduleReserved）
```

`ClientLivenessRecord::EvaluateAndRetire`（`client_liveness.h:136-174`）先置 OFFLINE 再执行 retire 回调，注释说明：这样"新 guard 再也进不来"，而 retire 回调可以在不嵌套 transition mutex 的情况下拿 segment/metadata 锁（`:167-172`）。

这套状态机解释了为什么 `MountSegment` 用 `ObserveAndRun` 而不是直接写（`master_service.cpp:688-701`）：

- `REJECTED_OFFLINE` → 直接 `UNAVAILABLE_IN_CURRENT_STATUS`（一个已被判定离线的 client **不能靠重挂复活**，必须重新注册）；
- 回调返回 false（挂载失败）→ 观测被"扣留"（`OBSERVATION_WITHHELD`），存活时间不刷新——避免一个不断失败重试的 client 永远不超时。

### 流程 3：GetReplicaList（读路径）

```
WrappedMasterService::GetReplicaList                    rpc_service.cpp:312-342
 └ WithRequestTenant(租户解析) → MasterService::GetReplicaList   master_service.cpp:4777-4875
```

主流程：

1. `std::shared_lock master_state_mutex_`（4780）——挡 unmount/restore。
2. `MetadataAccessorRO`：shard **共享锁**（4787）。
3. `Exists()` 否 → `OBJECT_NOT_FOUND`（4791-4794）。
4. `GetReadableReplicaDescriptors(metadata)`（4797）→ 内部 `IsReplicaReadable` = `is_completed() && is_available() && !IsAwaitingRemount(replica)`（4716-4719）。`invalid_replica_endpoints_` 只在 standby restore 到 owner 重挂之间的窗口非空（4685-4693），所以这是个"快路径零成本"的检查。
5. DFS 副本顺带 `UpdateAccess` 供其内部淘汰打分（4798-4805）。
6. 空列表：若全部 REMOVED → `OBJECT_NOT_FOUND`；否则 `REPLICA_IS_NOT_READY`（4807-4815）。
7. 指标打点 + `metadata.GrantReadLease(default_kv_lease_ttl_)`（4818-4833）——**在 shard 共享锁下完成**：deadline 更新是 `Lease` 内的原子 CAS，`GrantReadLease` 只额外取一个保护 `lease_` 指针的极短 `SpinLock`，不升级 shard 锁。
8. 判定 promotion-on-hit 资格（`!any_memory && any_local_disk`）与动态复制信号（4839-4860），把 `GetReplicaListResponse(replicas, ttl, checksum)` 建好（4862-4864）。
9. **访问器释放之后**再做 promotion 入队 / `RecordAccess`（4866-4873）。注释写明原因："avoid lock-upgrade complexity"——RO 持有 shard 共享锁时不能升级为写锁，只能先放后取。

这条路径上 Master 只做"过滤 + 计数 + 授租约 + 拼响应"，没有任何数据搬运。租约的意义是"在这段时间内别把这个对象淘汰掉，客户端正在读"。

### 流程 4：容量水位与淘汰

```
EvictionThreadFunc                                    master_service.cpp:10219-10307
  每 10ms:
    used_ratio = segment_manager_.GetMemoryUsage().used_ratio()   ← 无锁聚合
    if (used_ratio > high_watermark || (need_mem_eviction_ && eviction_ratio_ > 0))
        target      = max(eviction_ratio_, used - high + eviction_ratio_)
        lowerbound  = max(target*0.5, used - high)
        BatchEvict(target, lowerbound)
    else if 距上次 discard 超过 put_start_release_timeout_sec_
        逐 shard DiscardExpiredProcessingReplicas + ReleaseExpiredDiscardedReplicas
    每 1s（多租户开启且 tenant_eviction_high_watermark_ratio_ > 0）：EvictTenantsOverWatermark
    NoF：同样的水位逻辑 → NoFBatchEvict
    DFS：按 dfs_allocator_->GetEvictionCheckInterval() 节流 → RunDfsEviction
    有 promotion 候选：RunPromotionCandidateRetry
```

要点：

- `GetMemoryUsage()` 是 O(1) 无锁读（`segment.h:438-440`），注释明确它是 best-effort，mount/unmount 瞬间可能撕裂——为了不在 10ms 周期里抢 `segment_mutex_`，这个精度损失是刻意的。
- `need_mem_eviction_` 是**分配失败路径反向触发淘汰**的信号：`AllocateReplicas` 失败时设置（5208-5210、5260-5264），`BatchEvict` 开头用 `exchange(false)` 消费掉（10918-10920）。这形成"写失败 → 立刻淘汰 → 重试"的闭环，而不必等水位自然越线。
- `BatchEvict`（10916-…）注释描述了两轮：第一轮只淘汰无 soft pin 的对象，若实际淘汰比例仍低于 lowerbound，第二轮才考虑 soft-pinned（取决于 `allow_evict_soft_pinned_objects_`）。这就是"近 LRU"——排序依据是租约剩余时间而不是严格访问序（`master_service.h:1028-1036`）。
- `BatchEvict` 本身不是单线程扫全表：先起 `min(kNumShards, 16)` 个线程并行做候选"普查"（只收集 lease deadline），再按 target 物化一个有界候选前沿（reserve slack / frontier limit），最后串行按 key 重新查找并淘汰。普查与淘汰之间有窗口，所以淘汰阶段必须重新校验（`master_service.cpp:11248-11320`、`:11479-11560`）。淘汰结束后只对真正淘汰过的 shard 调 `ShrinkBucketsIfSparse`（`master_service.cpp:11699-11704`）。
- `need_mem_eviction_` 在 `BatchEvict` 开头被 `exchange(false)` 消费；若期间分配压力又把它置位、且本轮毫无进展（既没淘汰、也没释放 discarded、也没排队 offload），末尾会重新 `store(true)` 把压力留给下一轮（`master_service.cpp:11706-11713`）。
- offload-on-evict（10986-…）：有 LOCAL_DISK 副本直接删 MEMORY；无则先 `PushOffloadingQueue` 把一个副本 pin 住推进队列，每轮有 `offloading_queue_limit_ * offload_cap_ratio_` 的上限，超限且 `offload_force_evict_` 才强删。

### 流程 5：进程启动（非 HA）

`master.cpp:1411-1768` 的顺序值得当作"Master 由哪些部件组成"的清单：

1. glog 初始化 + gflags 解析（1412-1433）。
2. `MasterConfig` ← 配置文件（`DefaultConfig`）+ 命令行（`LoadConfigFromCmdline`），再补 metrics bootstrap / client liveness / RPC 地址（1439-1469）。
3. 一致性校验：HA 必须有 backend connstring（1481-1490）；`enable_oplog` 蕴含 `enable_ha` 且 backend 为 etcd/redis（1491-1499）；batch snapshot 配置校验（1500）；allocator 只能是 cachelib/offset（1518）。
4. RPC 协议选择（`RpcProtocolConfig::FromEnvironment`；未编译 IBV 时降级 TCP 并告警，1526-1535）。
5. metadata cleanup 拓扑：同城内嵌 vs 远端 URL，两者都不可用则**关掉该功能并告警**（1548-1569）。
6. 可选启内嵌 `HttpMetadataServer`（1650-1664）。
7. 分叉：
   - HA：`StartShutdownWatchdog` + `InstallShutdownSignalHandlers`（必须在拿任何 leader lease 之前，1674-1689）→ `MasterServiceSupervisor(supervisor_config).Start()`（1694-1695）。
   - 非 HA：构造 `coro_rpc_server`（1706-1710）→ `WrappedMasterService`（shared_ptr，1719-1722）→ `MasterAdminServer` 启动并 `SetServiceDelegate` / `SetServiceAvailable(true)`（1723-1733）→ **`RpcWorkerPool` 声明在 service 之后**（1735-1740）→ 注册 handler → 信号处理 → 另起线程跑 `server.start()`，主线程轮询 `shutdown_requested || server_done` → `server.stop()` → join（1742-1767）。

"`RpcWorkerPool` 必须声明在 service 之后"是析构顺序约束：局部对象逆序析构，worker pool 先销毁（`~RpcWorkerPool` 里 `threads_.join()` 等已入队调用跑完，`rpc_worker_pool.h:69`），之后才轮到 service。反过来会 join 到一半发现 `service_` 已析构，直接 UAF。

## 与其他模块的边界

**向上（谁调用 Master）**

| 调用方 | 入口 | 契约 |
|---|---|---|
| Store Client（`MasterClient`） | `coro_rpc` 按 **handler 名字**路由 | 方法签名见 `rpc_service.h:40-374`；客户端侧封装在 `include/master_client.h` |
| 运维/脚本 | `MasterAdminServer` 的 HTTP 路由表 | `master_admin_service.cpp:1365-1489` |
| HA 监督进程 | `MasterServiceSupervisor` 直接持有 `WrappedMasterService` | `RestoreFromStandby` / `RestoreFromBatchOpLogPromotion` 是**内部方法，不是 RPC**（`rpc_service.h:280-296` 注释） |
| embedded / 测试 | 直接 `new MasterService(MasterServiceConfig)` | `MasterServiceConfigBuilder`（`master_config.h:845`） |

**向下（Master 依赖谁）**

- **RPC/HTTP/序列化**：ylt 的 `coro_rpc`、`coro_http`、`struct_pack`、`async_simple`。`RegisterRpcService` 用变参模板一次注册约 70 个 handler（`rpc_service.cpp:1875-1951`）。
- **`BufferAllocatorBase`**（`allocator.h`）：cachelib 与 offset 两种实现，`CreateBufferAllocator(type, name, base, size, te_endpoint)`（`segment.cpp:254`）。
- **`LocalSsdManager`**：SSD 侧容量与 offload/promotion 邮箱；`SsdFreeRatioFirst` 的打分数据源。
- **Transfer Engine**：Master 只保存 `te_endpoint`（`types.h:463`）与 `AllocatedBuffer::Descriptor` 里的 `buffer_address_`/`transport_endpoint_`（`allocator.h:143-150`），不建立任何 TE 连接。
- **DFS 后端**：`ShardAllocator` / `ImmutableBucketAllocator` / `DfsAllocatorInterface`（`master_service.h:65-68`、`:2157-2163`）。
- **HA**：`OrderedOpLogWriter`、`OpLogBatchStorage`、`HaKvBackend`（etcd/redis）、snapshot codec；`enable_oplog_` 在构造时被收紧为 `config.enable_ha && config.enable_oplog && HaBackendSupportsOpLog(...)`（`master_service.cpp:194-195`）。
- **可观测性**：`MasterMetricManager`（单例，生命周期长于 `MasterService`）+ `HAMetricManager`；`KvEventPublisher` 把「对象已写入」事件推给外部。

**内部边界（`WrappedMasterService` 到底加了什么）**

`WrappedMasterService` 只持有 `MasterService master_service_`（值成员，`rpc_service.h:377`），每个方法都是同一个模板骨架（以 `PutStart` 为例，`rpc_service.cpp:428-455`）：

```
execute_rpc("PutStart",
    [&]{ return WithWriteTenant(tenant_id, IsTenantQuotaEnabled(),
                                [&](const TenantId& id){ return master_service_.PutStart(...); }); },
    timer_log, inc_requests, inc_failures);
```

叠三层语义：

1. **租户解析/规范化**：`WithRequestTenant`（读，未开多租户就强制 default）/ `WithWriteTenant`（写，未开多租户强制 default，开了则空 tenant 直接 `TENANT_NOT_REGISTERED`）（`rpc_service.cpp:22-75`）。
2. **请求指标与 VLOG 计时**：`execute_rpc` + `ScopedVLogTimer`。
3. **少量后处理**：如 `GetReplicaList` 回填 `resolved_tenant_id`（`rpc_service.cpp:324-331`）；`PutStart` 对 `OBJECT_ALREADY_EXISTS` 单独打指标（`:449-451`）。

注意：**lease / HA 语义不在这里**，它们都在 `MasterService` 内部完成。`WrappedMasterService` 是"协议适配 + 可观测性"层，不是业务层。

**RPC 线程模型**（`rpc_worker_pool.h`）

`coro_rpc` 把一条连接绑到一个 io 线程并**内联执行**普通 handler；一个慢 handler 会卡住该线程上所有其它连接（包括健康客户端的 Ping）。`RpcWorkerPool` 用 `PooledHandler` 把调用 `coro_io::post` 到 asio 线程池（`rpc_worker_pool.h:42-49`，设计说明见 `:58-63`），io 线程只负责收发。

分流规则在 `RegisterRpcService`（`rpc_service.cpp:1887-1893`）：

- `Ping`、`ServiceReady` **留在 io 线程**（必须永远能响应，不能排在 worker 队列后面）；
- 其余全部经 `RegisterPooledHandlers` 走 worker pool（`rpc_service.cpp:1874-1885`）。`worker_pool == nullptr` 时退化为全部内联注册。

**Admin 面的状态门控**

`MasterAdminServer` 不直接持有 `MasterService`，而是持 `shared_ptr<WrappedMasterService>` + 四个状态字段（`state_` / `leader_view_` / `service_` / `service_available_`，`master_admin_service.h:136-141`）：

- `RuntimeSnapshot SnapshotState()`：`state_mutex_` 下一把拷出 `{state, leader_view, service, service_available}`（`master_admin_service.cpp:380-388`）。
- `GetActiveService()`：`service_available==false` 就返回 nullptr，handler 经 `WithActiveService` 统一回 `503 service plane is not active`（`:551-579`）。
- `SetServiceDelegate` / `SetServiceAvailable` 都用 `storage_metrics_refresh_mutex_` 串行化；一旦服务不可用（撤下 delegate，或 `SetServiceAvailable(false)`）就立即 `project_storage_usage({})` 清空投影——避免"旧 leader 采样的容量指标在新任期被发布"（`master_admin_service.cpp:350-378`，注释见 `master_admin_service.h:133-135`）。

## 设计取舍与易踩坑点

1. **两套 segment/placement 代码，只有一套是活的。** `SegmentManager + AllocatorManager + AllocationStrategy` 才是 master 运行时真正用的；`include/segment/pool.h`、`catalog.h`、`region_driver.h`、`include/placement/*`（`SegmentPool`/`PlacementIndex`/`ReplicaAllocator`/`AllocationCandidate`）与 `src/placement/*`、`src/segment/pool*.cpp` 虽被编进 master 二进制，但 master 生产路径**没有任何实例化**。证据：`master_service.cpp:40` 只 `#include "segment/region_driver.h"`，全文再无 `RegionDriver` 引用；`grep` 只在 `src/placement/*`、`src/segment/pool*.cpp`、`src/segment/snapshot.cpp`、`src/ha/snapshot/*codec*.cpp` 与单测里命中。粗读时极易误判"放置逻辑已迁移"。

2. **`metadata_store.h/.cpp` 名不副实。** 它不是 master 主元数据索引，而是 HA/Standby 的可序列化容器（`StandbyObjectMetadata`/`StandbySnapshot`/`StandbySegmentRegistry`/`MetadataStore` 抽象）。运行时索引是 `master_service.h:1135` 的 `std::array<MetadataShard,1024>`。`metadata_store.cpp` 只有 52 行。

3. **`include/metadata/*` 是原型，不是现状。** `route::ObjectRoute`、`metadata::Tenant`、`TenantRegistry`、`StripedGroupIndex` 均未被 master 使用；master 只用其轻量类型 `route::ObjectState`（`master_service.h:1111`，定义在 `metadata/object_state.h:18`）与 `route::Generation`/`ObjectRef` 做身份标识。真正的租户/分片状态仍在 `MetadataShard.tenants`。

4. **设计文档落后于头文件。** `docs/source/design/store/metadata-management.md` 里的 `TenantState` 仍是"一堆散装 map + `unordered_map<string,ObjectMetadata>`"；当前头文件是 `ObjectRecord{generation, metadata, state}`。讲元数据组织**以 `master_service.h:1093-1135` 为准**，文档只用于建立心智模型。

5. **`allocation_strategy.cpp` 只有 69 行，不代表策略简单。** `RandomAllocationStrategy::Allocate`、`RankedAllocationStrategy::AllocateRanked`、`FreeRatioFirst`、`Cxl` **全部是 header-inline**（`allocation_strategy.h:303-708`）。只读 .cpp 会得出"就一个 SSD 打分"的错误结论。

6. **`LOCAL_FIRST` 映射到 `RandomAllocationStrategy`**（`allocation_strategy.h:725-726`）。局部性完全由"调用方把 host-ordered segments 塞进 `preferred_segments`"实现，且**只在 `replica_num == 1` 时启用**（`master_service.cpp:5499-5504`）。改策略时别指望在策略类里找到 local-first 逻辑。

7. **"best-effort" 只对纯内存/FLEXIBLE 成立。** 见 `HasExpectedReplicaAllocation`（`master_service.cpp:110-123`）。纯内存 `replica_num=3` 拿到 1 个也算成功（有 WARNING + `put_start_partial_allocations` 指标），含 NoF 的其它配置则必须精确匹配。粗读地图里"分配总是尽力而为"的说法需要按此收窄。

8. **没有 per-key 锁，写并发受 shard 粒度限制。** `MetadataAccessorRW` 的注释明确："Until the store moves to per-key locks, holding a key means holding its metadata shard's write lock"（`master_service.h:1737-1740`）。粗略地图里"4096 条对象操作 mutex"在 `master_service.h`/`master_service.cpp` 中**查无实据**（唯一出现的 4096 是 `SoftPinDeadlineIndex::kMinCompactionThreshold`，`master_service.h:1189`）。因此：同一 shard 内不同 key 的写会互相阻塞；写路径（含分配、OpLog 序列化准备工作）全程持锁。这解释了为什么 `ClearStaleHandles` 要"先共享锁挑选、再按 64 个一批加写锁清理"（`master_service.cpp:64-69` 的注释与 `kStaleHandleCleanupBatchSize`，实现见 `:2492-2526`）。

9. **`segment_mutex_` 与配额表锁永不嵌套。** `PrepareUnmountSegment` 单独一个作用域拿锁，`ClearInvalidHandles` 在作用域外（`master_service.cpp:2845-2872`）；`ScopedAllocatorAccess` 也只在作用域内取快照（`master_service.cpp:5172-5194`）。想"在一个 `ScopedSegmentAccess` 里顺手查配额"会破锁序。

10. **分配走的是快照，安全靠 registration 而不是锁。** `AllocateReplicas` 把 `AllocatorManager` 拷出来、放锁、再分配。语义正确性由 `SegmentAllocatorRegistration::Allocate` 的"分配前双检 + 分配后复检"保证（`segment_allocator_registration.cpp:22-45`）。反过来说：**绕过 `registration->Allocate` 直接 `allocator->allocate` 会丢掉这层保护**。

11. **`getServingNames()` 每次重建 vector，而且每次 PutStart 至少跑两遍。** 它是 O(segments×registrations) 操作，并会调用每个 registration 的 `IsServing()`（两次 atomic load）：`AllocateReplicas` 为算 `has_enough_memory_segments` 跑一遍，策略内部又对快照跑一遍（`master_service.cpp:5176-5178`、`allocation_strategy.h:317`）。segment 数很多时这是可测量的开销。

12. **同名 segment 让 `client_by_name_` 有损。** 注释直说 "client_by_name_ is intentionally lossy because several Clients may mount Segments with the same hostname"（`segment.cpp:634-636`）；`GetAllSegments(vector<pair<Segment,UUID>>&)` 改为反查 `client_segments_`。写任何"按名字找 owner"的新代码都不要用 `client_by_name_`。

13. **CXL 是共享 allocator 的特例。** 多个 segment 记录指向同一个 `cxl_global_allocator_`；`GetMemoryUsageSnapshot` 用 `counted_allocators` 去重（`segment.cpp:125-140`），`releaseCapacityMetrics` 也是为了让"跨任期复用的 `MasterMetricManager` 单例"不被重复计数/漏减（`segment.h:391-400` 注释）。

14. **CXL 策略忽略 `replica_num`。** 见核心数据结构 7：只返回 1 个副本。CXL 部署下如果有人配 `replica_num=2`，不会得到 2 个副本。

15. **`PutEnd` 的 `GrantReadLease(0ms)` 不是 bug。** `Lease` 单调 max：0 TTL 只把已经过期的下界抬到 now，对仍有有效期的租约是 no-op（`master_service.cpp:5821`、`lease.h:20-30`）。

16. **unmount 的幂等性写在各层。** `PrepareUnmountSegment` 找不到 → `SEGMENT_NOT_FOUND`；`MasterService::UnmountSegment` 把它翻译成 **成功**（`master_service.cpp:2850-2855`）。客户端可以安全重试。

17. **HA 的 `RestoreFromStandby` 不是 RPC。** 它没有注册进 `RegisterRpcService`，只在 supervisor 内部调用（`rpc_service.h:280-296` 有明确注释）。把它当外部接口会得出错误的攻击面/兼容性结论。

18. **`MasterService` 析构顺序即依赖顺序。** 先停各线程（含 `graceful_unmount_scheduler_.Stop()`、`replica_cleanup_worker_.Stop()`），再 join，再停 `dynamic_replication_`、`client_offboarding_worker_`、`ordered_oplog_writer_`，最后 `segment_manager_.releaseCapacityMetrics()` 并清空 liveness 记录（`master_service.cpp:550-612`）。在 HA 里 `MasterMetricManager` 活得比 `MasterService` 久，所以容量贡献必须显式释放。

19. **`SoftPinDeadlineIndex` 是锁序第 7 级，且只在构造期/淘汰期低频访问。** 读路径完全不碰它（读只做一次保护 `lease_` 指针的短 `SpinLock` + `Lease` 的原子 CAS）。把 soft pin 理解成"淘汰优先级标记 + 一个到期索引"，而不是"阻止删除的锁"。

20. **`MetadataAccessorRW` 构造期就会清理失效副本。** 任何"我只是想读一下这个 key 的元数据"的写路径，只要用了 RW 访问器，就可能触发 `CleanupInvalidMemoryReplicas` 并连带删除对象（`master_service.h:1751-1754`）。这不是副作用 bug，是设计：segment 卸载/Client 离线后，第一次有写者碰到这个 key 就地修复。

## 精读清单

按下面顺序读，每一步都标注"应该看懂什么"。全程只需 5 个文件，其余按需跳转。

**第 1 步：建立边界感（30 分钟）**

1. `docs/source/design/store/mooncake-store.md` 的 Master/Client 职责划分段落 —— 确认"Master 不搬数据"。
2. `master.cpp:1411-1768`（`main`）—— 能画出一张"配置 → 校验 → [HA | 非 HA] → server/admin/worker_pool"的框图，并说清 `RpcWorkerPool` 为什么声明在 service 之后。

**第 2 步：元数据结构与并发（1–2 小时，本章最核心）**

3. `master_service.h:101-118`（锁顺序）—— 背下 7 级顺序，特别是"policy 先于 state""segment 与 quota table 不嵌套"。
4. `master_service.h:1093-1135`（`ObjectIdentity`/`ObjectRecord`/`TenantState`/`MetadataShard`）—— 能默写三级索引与 `getShardIndex` 的两个分支（`:1337-1350`）。
5. `master_service.h:1741-1996`（`MetadataAccessorRW`/`RO`）—— 重点看三个构造函数、`Publish`/`Erase`、`QuotaAccount`、析构里的 `MaybeEraseEmptyTenant`，以及那句"per-key locks 尚未实现"的注释。
6. `object_metadata.h:46-160` + `replica.h:56-62`（`ReplicaStatus`）、`:674-777`（`Descriptor` 变体及其取值/判断辅助）—— 能说清五种介质描述符各自携带什么，以及为什么 `LocalDiskDescriptor` 没有 buffer 地址。
7. `lease.h` 全文（80 行）—— 理解"一个原子变量 + 单调 max"如何支撑 group 共享 TTL。
8. `metadata/object_state.h:18-38` —— 确认 `ObjectState` 只有几个 `HeapOptional`，别去找 `processing_keys` 之类的老结构。

**第 3 步：写路径（2–3 小时，本章篇幅最大处）**

9. `master_service.cpp:5442-5527`（`PrepareWriteStart`）—— 列出全部拒绝条件，尤其是 `writer_host_id` 的三重门槛。
10. `master_service.cpp:5589-5661`（`PutStart` + admit lambda）—— 逐行看锁的获取/释放顺序，以及"已发布且未超时"分支的三种结局。
11. `master_service.cpp:5140-5320`（`AllocateReplicas`）—— 画出"取快照 → 出锁 → 分配 → 多介质拼装 → 严格度检查 → partial 告警"的流程；对照 `master_service.cpp:110-123` 理解 best-effort 的边界。
12. `master_service.cpp:5322-5402`（`InsertMetadata`）—— 注意 `Publish` 在锁内、回滚路径、`is_processing = true`。
13. `master_service.cpp:5663-5834`（`PutEnd`）—— 关注幂等判定、DFS `MarkCommitted`、`mark_complete` 遍历、`GrantReadLease(0)`、`EraseFromProcessing` 条件。

**第 4 步：放置策略（1 小时，全在头文件里）**

14. `allocation_strategy.h:33-208`（`AllocatorManager`）—— `names_` 为什么单独存、`getServingNames` 的语义与代价。
15. `allocation_strategy.h:303-399`（Random）+ `:461-571`（Ranked）—— 注意 Ranked 的"环窗口采样"不是均匀采样，以及最后的随机兜底。
16. `allocation_strategy.h:573-646` + `allocation_strategy.cpp:18-67`（FreeRatio / SsdFreeRatio）—— 两种 score 的数据来源不同（allocator 自身 vs `LocalSsdManager`）。
17. `allocation_strategy.h:648-730`（Cxl + 工厂）—— 记住 `LOCAL_FIRST → Random`，以及 CXL 只返回 1 个副本。
18. `segment_allocator_registration.cpp` 全文（84 行）—— 画出"分配前双检 → allocate → 绑定 → 分配后复检"的四段式。

**第 5 步：segment 生命周期（1–2 小时）**

19. `segment.h:28-41`（`MountedSegment`）+ `:380-479`（六个索引）—— 能说出每个索引解决什么查询。
20. `segment.cpp:169-274`（`MountSegment`）—— 六个索引的一致性写入顺序 + CXL 分支 + 对齐校验。
21. `segment.cpp:312-367`（`ReMountSegment`/`ValidateRemountSegment`）—— 理解"哪些错该让客户端重试、哪些不该"。
22. `segment.cpp:420-461` + `:544-594` + `:505-542`（两阶段卸载 + 名字重索引）—— 重点看 `Invalidate()` 的位置与"不立即 detach usage"的理由。
23. `segment.cpp:49-92`（`BuildHostOrderedSegments`）—— 手工推演 `writer_host_id="h2"`、hosts={h1,h2,h3} 时的输出顺序。
24. `master_service.cpp:2837-2885`（`UnmountSegment`）—— 对照锁序，确认两次 `ScopedSegmentAccess` 作用域是刻意分开的。
25. `master_service.cpp:676-742`（`MountSegment`）+ `client_liveness.h:136-174`—— 讲清"OFFLINE 拒绝复活""失败不刷新存活时间"。
26. `master_service.cpp:11969-12088`（`ClientMonitorFunc`）—— 画 ACTIVE→SUSPECTED→OFFLINE→offboarding job 的状态图。

**第 6 步：读路径与淘汰（1 小时）**

27. `master_service.cpp:4685-4724` + `:4777-4875`（`GetReplicaList`）—— 注意"RO 访问器释放后再入队 promotion"的顺序约束。
28. `master_service.cpp:10219-10307`（`EvictionThreadFunc`）—— 记住 10ms / 1s 两个周期与 `need_mem_eviction_` 的 `exchange(false)`。
29. `master_service.cpp:10916-11030`（`BatchEvict` 开头 + offload-on-evict）—— 理解"近 LRU"的排序依据与 offload cap。

**第 7 步：接口面与运维面（1 小时）**

30. `rpc_service.cpp:1887-1951`（`RegisterRpcService`）—— 数清 Ping/ServiceReady 之外有多少 handler，理解为什么要分池。
31. `rpc_service.h:40-374` —— 当作接口字典浏览一遍，标出 Batch 变体与 Copy/Move/Drain/Promotion 分组。
32. `rpc_service.cpp:22-91`（租户解析三件套）+ `:428-455`（`PutStart` 包装）—— 确认 `WrappedMasterService` 只加协议适配与指标。
33. `master_admin_service.cpp:1365-1489`（`RegisterHandler`）+ `:350-388`/`:551-579`（状态门控）—— 列出全部 HTTP 端点，理解 `service_available_` 如何与 HA 交接串行化。

**第 8 步：反向验证（30 分钟，防止被文档带偏）**

34. `grep -rn "SegmentPool\|PlacementIndex\|ReplicaAllocator" mooncake-store/src --include=*.cpp` —— 确认命中只在 `src/placement/*`、`src/segment/pool*.cpp`、snapshot codec，master 主路径为零。
35. `grep -n "4096\|object_mutex" mooncake-store/include/master_service.h` —— 确认没有 per-key 锁表，只有 `kMinCompactionThreshold`。
36. `grep -rn "metadata::Tenant\|ObjectRoute" mooncake-store/src/master_service.cpp` —— 确认 `include/metadata/*` 的重型容器未被使用。

读完这 36 步，你应该能独立回答：一个新 key 的 Put 从 RPC 进来到可读，中间经过了哪几把锁、哪几次分配、哪几个索引；一个 segment 卸载时，为什么新分配不会再落到它上面；以及为什么 master 集群的写吞吐上限大致等于 `kNumShards × (1 / 单 shard 写临界区耗时)`。

---

# 07 Store Client 与数据面

> 精读范围：`mooncake-store` 客户端侧（RealClient / Client / DummyClient / MasterClient / TransferSubmitter / buffer / C ABI）。
> 所有 `路径:行号` 均相对仓库根 `D:/xiaojun/work/Mooncake`，以 main 分支 67df8c41 为准。
> 本章只描述客户端如何"要元数据、搬字节、做校验、收敛错误"，Master 的放置算法与副本状态机见第 08 章。

## 定位与职责

Mooncake Store 的数据面是两段式的：

1. **控制面**：客户端向 Master 要"这个 key 应该写到哪 / 从哪里读"，拿到的是副本描述符（endpoint + 远端虚拟地址 + 大小 + 协议）。Master 只分配空间、维护副本元数据与租约。
2. **数据面**：客户端经 Transfer Engine 直接和**持有数据的对端 Client** 搬字节。Master 从不经手数据——设计文档原话是 "The Master Service does not take over any data flow, only providing corresponding metadata information."（`docs/source/design/store/mooncake-store.md:56`）。

因此客户端子系统承担四件事：资源注册（把本进程内存交给 TE）、协议状态机（Put 两阶段、Get 租约/校验/hot cache）、传输策略（memcpy / TE / 文件 / SPDK NoF）、以及多种进程形态的适配（embedded、dummy-real、独立进程、C ABI 宿主）。

内部分层如下，自上而下职责递减、代码量递增：

| 层 | 代表 | 职责 |
|---|---|---|
| Python 契约层 | `PyClient`（`mooncake-store/include/pyclient.h:232`） | pybind（`store_py.cpp` 的 `shared_ptr<PyClient>`）使用的纯虚接口，`RealClient`/`DummyClient` 都继承它；C ABI 不经过这层，直接持有具体 `RealClient` |
| 资源与适配层 | `RealClient`（`real_client.cpp`，9144 行） | 持有 `Client`、buffer allocator、segment、FileStorage、dummy shm 映射表、UDS 服务端、KV session、ResourceTracker 清理 |
| 存储协议层 | `Client`（`client_service.cpp`，5664 行） | Query / Put / BatchPut / Get / BatchGet / Transfer*、副本选择消费、租约、checksum、hot cache、HA、后台 Copy/Move 任务 |
| 控制面 RPC 封装 | `MasterClient`（`master_client.cpp`，1472 行） | 全部 Master RPC、连接池、HA 快速失败、RPC drain |
| 数据面提交层 | `TransferSubmitter`（`transfer_task.cpp`，2099 行） | 策略选择、OperationState/Future 生命周期、memcpy/fileread/SPDK 工作池、scatter 提交 |
| 进程代理 | `DummyClient`（`dummy_client.cpp`，2241 行） | TP 多 rank 场景下不持有资源的轻量代理，经 coro_rpc + memfd 共享内存把请求转给同机 RealClient |

**Client 的双重身份**：同一个 `Client` 既是请求发起方，也是存储提供方。`global_segment_size > 0` 时 `setup_internal` 会 `MountSegment` 把自己的内存贡献进集群池（`real_client.cpp:1318-1530`），别的客户端可以 RDMA 直读；`local_buffer_size = 0` 时它不注册本地 staging buffer，退化为纯服务端。所以"对端 Client"往往就是另一个跑着同样代码的进程。

**三种部署形态**（`docs/source/design/store/mooncake-store.md:40-43`）：

- embedded：推理进程内直接嵌 `RealClient`；
- dummy-real：每个 TP rank 一个 `DummyClient`（不持资源），每个推理实例一个 `RealClient`（持资源）。TP=8 时是 8 dummy + 1 real，dummy 通过 RPC + 共享内存/零拷贝转发；
- standalone：独立 `mooncake_client` 进程（入口 `real_client_main.cpp:114-166`），对外提供全局内存/SSD 资源池，此时 embedded 客户端可把 `global_segment_size` 设为 0，只贡献网卡资源。

`store_py.cpp` 的 docstring 明确说进程内 embedded master 是 "test/eval plumbing, not a documented deployment mode"（`mooncake-integration/store/store_py.cpp:2531`），不要把它和上述三种部署形态混为一谈。

**范围纠正**（读代码前先避免走偏）：

- `mooncake-store/include/client` 目录**不存在**。客户端头文件全部平铺在 `mooncake-store/include/` 下；唯一例外是 `registered_pinned_memory.h`，它在 `mooncake-store/src/`，因为它是纯内部实现头（配套配置头也在 `src/config/`）。
- `rpc_service.cpp` / `rpc_service.h` **不是客户端代码**。它是 Master 侧的 `WrappedMasterService` 与 `RegisterRpcService`，在 `mooncake-store/src/CMakeLists.txt:20` 起被列入 `MOONCAKE_STORE_MASTER_SOURCES`。客户端对应的 handler 注册表是 `real_client_main.cpp:38-111` 的 `RegisterClientRpcService`。

## 目录结构速览

| 文件 | 一句话职责 |
|---|---|
| `include/pyclient.h` | PyClient 抽象基类、QueryResultCache、RangedReadSnapshot、ClientRequester（store→store offload RPC）、dummy/real IPC 报文结构、三层结果构造函数 |
| `include/real_client.h` / `src/real_client.cpp` | RealClient：setup、put/get_into/batch_*、dummy 地址翻译、UDS 服务端、KV session、ResourceTracker 清理 |
| `src/real_client_main.cpp` | `mooncake_client` 可执行入口；`RegisterClientRpcService` 一张表列全 dummy→real RPC 面 |
| `include/client_service.h` / `src/client_service.cpp` | Client：Put 两阶段、BatchPut 流水线、Get/BatchGet、TransferData、hot cache、checksum、HA、后台任务 |
| `include/master_client.h` / `src/master_client.cpp` | MasterClient：全部控制面 RPC；`invoke_rpc_with_client_pool` 模板 + 每个 RPC 的 `RpcNameTraits`；三个连接池 |
| `include/transfer_task.h` / `src/transfer_task.cpp` | TransferSubmitter：策略分派、OperationState/Future、Memcpy/Fileread/SpdkNof 工作池、scatter |
| `include/client_buffer.h` / `src/client_buffer.cpp` | ClientBufferAllocator（offset allocator 子分配，**整块基址**按 64B 对齐）+ BufferHandle（owning/view）+ slice 工具 |
| `include/aligned_client_buffer.h` / `src/aligned_client_buffer.cpp` | 4096 对齐分配器，**只**服务 O_DIRECT + io_uring 的 SSD staging，不在 put/get 主链路上 |
| `include/dummy_client.h` / `src/dummy_client.cpp` | DummyClient：coro_rpc 转发、memfd 共享内存、staging 决策、ping/重连 |
| `include/uds_transport.h` / `src/uds_transport.cpp` | UDS 一次性阻塞通道：SCM_RIGHTS 传 fd（数据从不走 UDS） |
| `include/shm_helper.h` / `src/shm_helper.cpp` | memfd_create + mmap 匿名共享段单例，dummy/real 零拷贝的物理基础 |
| `include/store_c.h` / `src/store_c.cpp` / `src/store_c_shared.cpp` / `src/store_c.map` | C ABI：句柄、配置转换、异常兜底、空 TU、符号导出版本脚本 |
| `include/client_metric.h` / `src/client_metric.cpp` | ClientMetric 聚合体 + `execute_timed_operation` 模板 + 上报线程 |
| `src/registered_pinned_memory.h/.cpp` | host 协议 Store segment 的 cudaHostRegister 进程级单例，带配额与三态注销 |
| `include/replica_selection.h` | 读路径副本优选 `SelectBestReplica` + 可注入的远端打分钩子 |
| `include/batch_read_fanout.h` | 批量读重复 key 的 Plan/FanOut 契约 |
| `include/local_hot_cache.h` | 本地热缓存：block、引用计数、LRU、generation token、shm 模式 |
| `src/CMakeLists.txt` | SHARED/MASTER/CLIENT 三份源列表；`mooncake_client` 与 `WITH_STORE_C_SHARED` 链接规则 |

## 核心数据结构

### QueryResult（`include/client_service.h:43-72`）

读路径的元数据载体，四个成员全是 `const`，构造后不可变：

- `replicas`：Master 返回的副本描述符列表；
- `lease_timeout`：**本地 steady_clock 时间点**。`Client::Query`（`client_service.cpp:1240-1252`）在发 RPC **之前**取 `start_time`，收到响应后算 `start_time + lease_ttl_ms`，把网络往返时间也扣进租约，偏保守但安全；
- `object_checksum`：可选的全对象 CRC；
- `resolved_tenant_id`：Master 解析出的租户，读方转发给盘副本校验请求。

`IsLeaseExpired()` 只是 `now >= lease_timeout`（`client_service.h:64-69`）。租约检查全部由调用方显式做，没有任何后台自动续约。

### Replica::Descriptor 与 AllocatedBuffer::Descriptor

副本描述符有五种类型：MEMORY / NOF_SSD / DISK / LOCAL_DISK / DFS（`include/replica.h`）。MEMORY/NOF 副本的核心是 `AllocatedBuffer::Descriptor`（`include/allocator.h:143-150`）：

```cpp
struct Descriptor {
    uint64_t size_;
    uintptr_t buffer_address_;       // 对端进程内的虚拟地址（远端 RDMA 地址）
    std::string protocol_;           // rdma/tcp/nvlink/...
    std::string transport_endpoint_; // 对端 TE 的 ip:port，路由键
};
```

`transport_endpoint_` 是 TE `openSegment` 的键，也是 LOCAL_MEMCPY 判定的唯一依据；`buffer_address_` 只有在**本进程**时才可以直接解引用——这正是 memcpy 判定必须比对完整 endpoint 的原因。LOCAL_DISK 副本额外带 `client_id` 与 `transport_endpoint`（`replica.h` 的 `LocalDiskReplicaData`），用于走 owner 的 offload RPC。

### ReplicateConfig 与 ReplicaWriteMode（`include/replica.h:108-237`）

写请求配置：`replica_num`（内存副本数）、`nof_replica_num`、`dfs_replica_num`、`soft_pin_action`、`soft_pin_ttl_ms`、`with_hard_pin`、`preferred_segments`、`prefer_alloc_in_same_node`、`group_ids`、`session_ids`。`session_ids` 是 `struct_pack::compatible<..., 1>` 字段，保证新旧版本 RPC 载荷双向可读；`ValidSessionIds` 还会用 `boost::locale::utf` 校验每个 id 是合法 UTF-8 且长度不超过 256（`replica.h:136-153`）。

`DetermineReplicaWriteMode`（`replica.h:226-237`）把配置归成三种模式：

- `SINGLE_REPLICA`：默认，1 个内存副本；
- `FLEXIBLE_DUAL_REPLICA`：恰好 `replica_num==1 && nof_replica_num==1 && dfs==0`，内存/NoF 任一侧成功即整体成功；
- `RELIABLE_MULTI_REPLICA`：多副本或带 DFS，所有已分配副本必须全部成功。

这三种模式直接决定后面 `DetermineFinalizeDecision` 的裁决，是理解 Put 错误语义的钥匙。

### ReplicaTransferSummary 与 FinalizeDecision（`client_service.cpp:262-418`，匿名 namespace）

写路径的记账与裁决：

- `ReplicaTransferSummary`（`262-306`）按 MEMORY / NOF / DFS 三类分别统计 allocated / successful / failed，并记录第一个错误 `first_error`；
- `NonDfsTransfersSucceeded`（`308-314`）：内存 + NoF 全部成功且零失败；
- `AllAllocatedTransfersSucceeded`（`316-320`）：再加上 DFS 全部成功；
- `HasExpectedReplicaAllocation`（`322-336`）：Master 实际分配是否满足请求。**纯内存请求（`nof_replica_num==0 && dfs_replica_num==0`）只要求至少分配到一个内存副本，不校验是否达到 `replica_num`**；FLEXIBLE 模式只要求 memory+nof 总数 > 0；只有 NoF/DFS 参与时才要求每类数量精确相等。这是"多副本写"语义比直觉宽松的地方；
- `DetermineFinalizeDecision`（`365-418`）：
  - 非 FLEXIBLE：分配满足且全部传输成功才 `end_type=ALL`；否则 `revoke_type=ALL`，错误优先取 `first_error`，分配不满足则归一为 `NO_AVAILABLE_HANDLE`；
  - FLEXIBLE：内存成功 → `end=MEMORY, revoke=NOF, success=true`；NoF 成功 → 反之；两侧都成功 → `end=ALL`；都失败 → `revoke=ALL, success=false`。

**"部分成功仍返回 0"只发生在 FLEXIBLE_DUAL_REPLICA**。SINGLE/RELIABLE 模式下，如果 Master 真的分配了 3 个副本而只有 1 个传输成功，会 revoke ALL 并返回错误。但要注意反过来的情形：纯内存请求只分配到 1 个副本却传输成功时，整体仍判成功——分配数量本身不在校验范围内（见上面的 `HasExpectedReplicaAllocation`）。这两种情形不要混为一谈。

### Slice 与 kMaxSliceSize（`include/types.h:445-452`）

```cpp
struct Slice { void* ptr{nullptr}; size_t size{0}; };
const static uint64_t kMinSliceSize = facebook::cachelib::Slab::kMinAllocSize;
const static uint64_t kMaxSliceSize = facebook::cachelib::Slab::kSize - 16;
```

`kMaxSliceSize` 等于 16MiB - 16（`Slab::kSize = 1 << 24`），注释只写了 "should be lower than limit"。这是 Store 与 cachelib 分配器常量的隐式耦合：改 cachelib 版本会静默改变切片粒度。Slice 是数据面最小搬运单元，Master 侧也使用同一常量做长度校验。

### ClientBufferAllocator 与 BufferHandle（`include/client_buffer.h:27-118`，`src/client_buffer.cpp`）

- 分配器持有一整块按 64 字节（cache line）对齐的内存（`client_buffer.cpp:32-66`），底层是 `offset_allocator::OffsetAllocator` 子分配；hugepage 模式走 `allocate_buffer_mmap_memory`，`sunrise_link` 走专用分配器，其余走协议感知分配器。
- `create(void* addr, size_t size, ...)`（重载在 `client_buffer.cpp:26-30`，转入私有构造 `68-75`）用于**外部内存**：dummy 共享过来的 shm 在 real 侧就是用这个重载建 allocator 的。`is_external_memory_=true` 时析构不 free（`77-86`）。
- `BufferHandle` 是 move-only RAII，有两种模式（`client_buffer.h:82-117`）：
  - owning：持 `shared_ptr<ClientBufferAllocator>` + `OffsetAllocationHandle`，析构时 offset handle 自动归还；
  - view：只持 `ptr/size/release_fn`，析构时调回调。DummyClient 用 view 模式包装"real 侧分配、dummy 侧使用"的 buffer：`allocate_client_buffer` 拿到 dummy VA 后，`release_fn` 是一次 `release_buffer_dummy` RPC（`dummy_client.cpp:622-637`）。
- `split_into_slices`（`client_buffer.cpp:132-143`）按 `kMaxSliceSize` 切；`allocateSlices`（`162-192`）按副本类型映射：**DISK 副本按 kMaxSliceSize 切多片**，MEMORY/NOF/LOCAL_DISK/DFS 都只产生一个 slice（目标端对象本身就是连续的）。`calculate_total_size`（`145-160`）从任意副本类型取对象大小。

### OperationState 家族与 TransferFuture（`include/transfer_task.h:94-346`）

数据面异步结果的统一封装。`OperationState` 持 `optional<ErrorCode> result_` + mutex + condition_variable：

- `MemcpyOperationState` / `FilereadOperationState` / `SpdkNofOperationState`：worker 线程 `set_completed()` 后 notify；
- `TransferEngineOperationState`（`transfer_task.h:239-274`，实现 `transfer_task.cpp:795-1013`）：持 `BatchID`，析构时 `freeBatchID`/`freeBatch`。`check_task_status()`（`814-903`）**必须等 batch 内全部 task 到终态才置结果**，即使已有失败也不提前完成——注释明确说这是为了防止调用方看到 completed 后析构、`freeBatchID` 与在飞传输竞争。`set_result_internal`（`905-918`）对二次置位只打日志不 crash。
- `wait_for_completion()`（`920-1013`）：`USE_EVENT_DRIVEN_COMPLETION` 且非 TENT 时直接睡在 `BatchDesc::completion_cv` 上（带 acquire/relaxed 内存序配对说明）；否则忙轮询；60 秒超时置 `TRANSFER_FAIL`。
- `TransferFuture`（`transfer_task.h:283-319`）：持 `shared_ptr<OperationState>`，move-only，`get()` 即 `wait()`。
- `StoreScatterTransferOperation`（`transfer_task.h:327-346`）：**非线程安全**，只允许一个线程 wait/析构。其 Impl 析构时若未完成，会用较短超时（`MC_TENT_DESTRUCTOR_TIMEOUT_S` 默认 5 秒）轮询后 `forceCleanup`（`transfer_task.cpp:1243-1259`）；`wait()` 的 TENT 超时默认 60 秒（`1264-1290`）。

### TransferSubmitter（`include/transfer_task.h:592-756`，实现 `transfer_task.cpp:1042-2078`）

数据面单 key / 批量传输的统一入口（scatter 路径另有 `Client::SubmitScatter` 直连 TE）。成员包括 `engine_`（经典 TE 引用）、`tent_engine_`（可空裸指针）、`local_endpoint_`（构造时缓存 `getLocalIpAndPort()`，避免每次传输分配字符串）、`memcpy_pool_` / `fileread_pool_` / `spdk_nvmf_pool_`、`memcpy_enabled_`。

构造函数（`transfer_task.cpp:1042-1075`）的默认策略值得单独记住：`MC_STORE_MEMCPY` 未设置时，`memcpy_enabled_ = engine_.isTcpOnly()`——纯 TCP 环境启用 memcpy（省 loopback 开销），有 RDMA 则禁用。也就是说 RDMA 集群里即使读写本进程内存，默认也走 TE，除非显式 `MC_STORE_MEMCPY=1`。

### PutOperation（`client_service.cpp:2329-2443`）

BatchPut 流水线的每 key 状态机：

```cpp
enum class PutOperationState {
    PENDING, MASTER_FAILED, TRANSFER_FAILED, FINALIZE_FAILED, SUCCESS
};
```

构造时 `result` 预置为 `INTERNAL_ERROR`（`2349-2351`），确保任何提前 return 都不会把未完成的 op 当成功；只有 `SetSuccess()` 才清空错误。`pending_transfers` 存 `(ReplicaType, TransferFuture)` 对；`IsResolved()` 即 `state != PENDING`（`2438`），后续每个阶段都先跳过已 resolved 的 op。`SetError` 会依据"有没有 replicas、有没有 pending_transfers"推断失败阶段（`2384-2398`）。

### RealClient 的关键内部结构（`include/real_client.h`）

- `WritableBufferRegion{base,size,offset}`（`125-129`）：一次 `resolve_writable_buffer_region` 查询的结果，判断用户 buffer 能否直接做 RDMA 目的地。
- `MappedShm`（`1084-1108`）：dummy 共享段在 real 侧的映射，核心字段 `shm_addr_offset = real_base - dummy_base`（注释直接写明）。另有 `spdk_registered` 与 `te_unregister_pending` 两个拆除状态位，用于 quarantine 机制。
- `ShmContext`（`1110-1118`）：每个 dummy `client_id` 一个，含 `mapped_shms` 向量、建在外部 shm 上的 `client_buffer_allocator`、以及 `active_handles`（dummy 借走的 buffer，按 dummy 地址键控保活）。由 `dummy_client_mutex_`（`shared_mutex`）保护，读多写少。
- `registered_buffer_sizes_` + `local_buffer_region_`（`1122-1124`）：`registered_buffer_mutex_` 保护。前者存**调用方原始 size**，不是 HugeTLB 拓宽后的注册长度；后者是本进程 client buffer 的隐式可写区。
- KV session 表（`1126-1145`）：`get_sessions_`（`FilterQueryResult` 后的单副本 QueryResult）与 `put_sessions_`（`PutSessionEntry{replicas, object_size, write_mode, writable, inflight_transfers}`），由 `session_mutex_` + `session_cv_` 保护。注释明确：ranges 只在本地比租约、不再问 Master；Put 会话要靠 end/revoke 封会话并等待在飞 range 写完成才能 finalize/free。进程内私有，不与 DummyClient 共享。
- `closed_`（`1191`）：原子 CAS 保证 `tearDownAll` 只跑一次。
- `dummy_client_ping_queue_`（`1211-1213`）：`boost::lockfree::queue<PodUUID>`（128K 容量），dummy ping 与监控线程之间的无锁通道。
- `SessionRangeReadRequest/Plan`（`1239-1260`）：KV session 的 ranged read 计划，按 memory/dfs/local_disk/disk 四类分发。

### DummyClient::PreparedBuffer（`include/dummy_client.h:219-225`）

```cpp
struct PreparedBuffer {
    void *original; void *dummy; size_t size;
    std::unique_ptr<BufferHandle> staging; bool copy_back;
};
```

`dummy` 是实际发给 real 的地址（原地址或 staging 地址）；`staging` 非空表示发生了真拷贝；`copy_back` 表示操作完成后要把 staging 拷回原 buffer。这个结构是"dummy 模式到底有没有拷贝"的唯一判据。

### C ABI 的 StoreHandle（`src/store_c.cpp:30-37`）

```cpp
struct StoreHandle { std::shared_ptr<mooncake::RealClient> client; };
typedef void* mooncake_store_t;
```

它存在的唯一理由是**保活 shared_ptr**：`RealClient::create()` 返回 shared_ptr 并向 `ResourceTracker` 注册 weak_ptr，如果 C 侧只存裸指针，shared_ptr 会在 create 返回后立刻析构。C ABI 的契约在 `include/store_c.h:56-62`：所有 `char*` 参数在函数返回后不再被使用，调用方可以立即 free。

## 关键流程剖析

### 流程一：单 key 写全链路（`put` / `put_from` → `Client::Put`）

调用链（embedded 或 RealClient 直连）：

```
RealClient::put (real_client.cpp:2543)
 └ execute_timed_operation { put_internal (2483) }
    ├ client_buffer_allocator_->allocate(value.size_bytes())
    ├ gather_maybe_device_to_host(handle.ptr(), value.data(), size)   // 可能是 D2H
    ├ split_into_slices(buffer_handle)
    └ client_->Put(key, slices, config)                               // client_service.cpp:1959
       ├ ComputeObjectChecksumForSlices (可选, 1294-1347)
       ├ hot_cache_->RemoveHotKey(key)                                // 1983-1985
       ├ master_client_.PutStart(key, slice_lengths, cfg)             // 1987
       │   └ OBJECT_ALREADY_EXISTS 且 heal 返回 kEvicted 时重试一次 (1988-1993)
       │   └ 仍是 OBJECT_ALREADY_EXISTS → return {} (幂等成功)          (1994-1999)
       ├ 逆序找 disk replica → PutToLocalFile → break                  (2018-2031)
       ├ 逐 memory/nof 副本 TransferWrite，失败只 RecordFailure+continue (2033-2046)
       ├ DFS 副本：仅当 NonDfsTransfersSucceeded 才 WriteDfsReplicas   (2048-2079)
       ├ DetermineFinalizeDecision                                    (2088)
       ├ end_type 有值 → PutEnd(ObjectMeta{key, checksum}, type)      (2091-2099)
       └ revoke_type 有值 → PutRevoke(key, type)                      (2101-2108)
```

逐步要点：

1. **拷贝语义**：`put` 先从 `client_buffer_allocator_` 分配一块，再 `gather_maybe_device_to_host`（用户数据可能在 GPU），然后切片。`put_from_internal`（`real_client.cpp:5385-5413`）则直接 `split_into_slices(buffer, size)`，零拷贝，但注释和头文件都警告 buffer 地址必须落在 Store 已注册的内存里，否则 RDMA 会失败。C ABI 同时暴露 `mooncake_store_put` 与 `mooncake_store_put_from`，选错就是多一次拷贝或直接传输失败。
2. **checksum 在 PutStart 之前**：`object_checksum_enabled_` 时先对整个对象算 CRC，写路径把 checksum 作为元数据交给 PutEnd。注意这会让 `InitLocalHotCache` 直接放弃启用 hot cache（`client_service.cpp:5518-5524`）。
3. **TransferWrite → TransferData → submit**（`4990-4994`、`4698-4729`）：`TransferData` 是**单 key** `TransferRead`/`TransferWrite` 到 TransferSubmitter 的桥；批量路径（`BatchGet`、BatchPut 的 `SubmitTransfers`）并不经过它，而是直接调 `transfer_submitter_->submit`（`1773-1777`、`2827-2844`）。NoF 副本要求 slices 连续（`GetContiguousSliceRange`），并把连续区间的 ptr/size 额外传给 submit 供 SPDK 使用；其余副本传 `nullptr, 0`。`future->get()` 同步等待。
4. **盘副本先写**：注释（`2018-2019`）解释逆序遍历 + break 的原因——盘副本的收尾是 `PutToLocalFile` 里异步完成的：它先在调用线程上用 pinned buffer 做同步 D2H/拷贝，然后 `write_thread_pool_.enqueue` 执行 `StoreObject`，成功则 `PutEnd(DISK)`、失败则 `PutRevoke(DISK)`（`4599-4696`）。先把它提交，后面内存副本 finalize 时盘的收尾任务至少已经入队（注意：入队不等于已完成，PutEnd 只保证顺序调度，不保证盘侧先落盘）。
5. **best-effort 复制**：多副本写失败不整体回滚。逐副本 `TransferWrite`，失败只 `RecordFailure` 后 `continue`；最后 `DetermineFinalizeDecision` 按成功数裁决。SINGLE/RELIABLE 模式任何一步失败 → `revoke_type=ALL, success=false`；FLEXIBLE 模式一侧成功即整体成功、另一侧 revoke。唯一更严格的是 DFS 参与时：DFS 写只在**非 DFS 传输全成功**后才发生，否则记 skipped（`2071-2079`），避免 DFS 成为"孤儿副本"。
6. **PutStart 遇 OBJECT_ALREADY_EXISTS 返回成功**（`1994-1999`）：对象不可变，重复 put 幂等视为成功，不搬数据。例外是"悬空 LOCAL_DISK"（文件被 RemoveAll 擦掉但 Master 元数据还在，issue #3709）：`healDanglingLocalDiskReplica`（`2142-2174`）先 `GetReplicaList`，确认**所有 COMPLETE 副本都是本 client 的 LOCAL_DISK**，再用 RealClient 安装的 `local_disk_probe_fn_`（`real_client.cpp:1504-1518`，FileStorage::Exists，租户 scoped key）判断文件是否真的没了。只有 `file_gone == true` 才 `EvictDiskReplica` 并重试 PutStart；present/unknown 都走旧的幂等路径——**健康副本绝不会被误杀**。
7. **失败路径也可返回成功**：`PutEnd` / `PutRevoke` 本身失败会直接返回该 RPC 的错误（`2091-2108`），即使数据已经写进去了。读代码时不要把"数据面成功"和"元数据面成功"混为一谈。

### 流程二：批量写流水线（`BatchPut`，`client_service.cpp:3596-3635`）

```
BatchPut = CreatePutOperations              (2445)
         → ComputeBatchObjectChecksums      (2456)
         → StartBatchPut                    (2572, 一次 BatchPutStart)
         → StageWriteBuffersForRemoteReplicas (3460, 可选 stager)
         → SubmitTransfers                  (2758, 全部 submit，不等)
         → WaitForTransfers                 (2854, 统一 wait)
         → SubmitDfsWrites                  (3035)
         → FinalizeBatchPut                 (3095, 按 end/revoke × ALL/MEMORY/NOF 分 6 组)
         → CollectResults                   (3419)
```

**为什么快**：`SubmitTransfers` 把 N 个 key × M 个副本的传输先全部提交，每个 `submit` 返回的 `TransferFuture` 存进 `op.pending_transfers`；`WaitForTransfers` 再逐个 `future.get()`。N 次串行 RTT 被压成一个并发窗口。

细节：

- `StartBatchPut`（`2572-2674`）先把所有未 resolved 的 op 的 key/slice_lengths 收集成一次 `BatchPutStart`；`session_ids` 按 active index 重建（`2581-2582, 2593-2595`）。开工前对整批 key 做一次 `hot_cache_->RemoveHotKeys`（`2584-2589`）。
- `OBJECT_ALREADY_EXISTS` 的 key 先攒进 `already_exists`；`healDanglingLocalDiskBatchStarts`（`2473-2571`）做一次批量 `GetReplicaList` + 一次批量 evict，只对被 evict 的 key 重发 PutStart。健康的幂等批次只多付一次 RPC。
- 分配不满足请求（`HasExpectedReplicaAllocation`）立即 `SetTerminalError(NO_AVAILABLE_HANDLE, MASTER_FAILED)`（`2650-2656`），不再尝试传输。
- `SubmitTransfers`（`2758-2852`）对每个 op 先处理盘副本（同样的逆序 + break），再逐个 memory/nof 副本 submit。NoF 副本的 slices 不连续会 `RecordFailure(INVALID_PARAMS)` 并跳过该副本，不影响其他副本。submit 失败同样 `RecordFailure + AppendFailureContext` 后 continue。
- `WaitForTransfers`（`2854-2882`）逐个 `future.get()` 并把结果记进 `transfer_summary`；这里不提前短路，保证所有 future 都被消费。
- `SubmitDfsWrites`（`3035-3093`）只为"非 DFS 全成功"的 op 发起 DFS 写，skipped 计数进指标。
- `FinalizeBatchPut`（`3095-3296`）把每个 op 的 end/revoke 动作按 `(is_end, ReplicaType)` 分到 6 个 group，每组一次 `BatchPutEnd` / `BatchPutRevoke`。`pending_finalize_actions[index]` 计数 + `finalize_rpc_errors[index]` 追踪：RPC 失败或计数不归零时，若此前已有传输级失败则保留该错误并记为 `TRANSFER_FAILED`，否则才记为 `FINALIZE_FAILED`（`3267-3285`）。注意对"已 resolved 但失败且有 replicas"的 op 会补一个 `revoke ALL`（`3164-3172`），防止 Master 侧留下半成品。
- `CollectResults`（`3419-3457`）把 `OBJECT_ALREADY_EXISTS` 的失败结果改写成成功（`3429-3437`），与单 key Put 的幂等语义对齐；同时统计 `NO_AVAILABLE_HANDLE` 的 key 数并打一条统一的"空间不足"提示。
- `prefer_alloc_in_same_node` 走另一条路 `BatchWriteWhenPreferSameNode`（`3483-3587`）：按目标 segment 分桶后用 `submit_batch` 每桶一次提交。该模式不支持 NoF，且要求 `replica_num == 1`（`ValidatePreferSameNodeWriteConfig`，`339-356`）。

`StageWriteBuffersForRemoteReplicas`（`3460-3482`）只在提供 stager 时生效：**所有副本都能 local memcpy 的 op 跳过 staging**，否则调用 stager 把用户 buffer 拷进 client buffer。dummy 转发来的 CUDA IPC 场景（`stage_nonlocal=true`）就靠这层避免把设备地址当远端 RDMA 源。

### 流程三：读全链路（`Client::Get` / `BatchGet` / `get_into` / `batch_get_into`）

**最短路径**：`Client::Get(key, slices)` = `Query` + `Get(key, qr, slices)`（`1168-1175`）。

**核心** `Client::Get(key, query_result, slices)`（`1382-1456`），顺序极易讲错，逐步列：

1. `FindFirstCompleteReplica`（`5459-5472`）**只取第一个 COMPLETE 副本**，不做本地性优选。真正的优选发生在 RealClient 层：`get_buffer_internal` / `batch_get_into_internal` 先用 `SelectBestReplica`（`include/replica_selection.h:122-172`：local MEMORY > local NOF > remote MEMORY > remote NOF > LOCAL_DISK > DFS > DISK），再用 `FilterQueryResult` 把 QueryResult 裁剪成只含该副本，保证 `Client::Get` 内部的 `FindFirstCompleteReplica` 只能看到已选中的那个（`get_buffer_internal` 内注释，`real_client.cpp:3828-3830`）。另外 LOCAL_DISK 的 offload 读失败时还有一条读侧自愈：`get_buffer_internal` 调 `heal_disk_replica_for_read`（`real_client.cpp:3685-3726`）请 owner 验证文件——evicted 或 present 都会带 `heal_dangling_disk_replica=false` 重试一次，`kUnknown` 才直接返回 miss（`3800-3814`）；批量读路径有对应的同一逻辑（`4392-4400`）。
2. `RedirectToHotCache`（`1934-1957`）：memory 副本且 hot cache 命中时，**原地改写** replica 描述符的 endpoint（指向自己）与 `buffer_address_`（指向本地 HotMemBlock）；size 不匹配则放弃。命中后传输目标变成进程内的 cache block：memcpy 可用时是本地 memcpy，否则仍走 TE 的本地段（至少省掉远端网络）。
3. `TransferRead`（`5010-5044`）或 DFS 走 `ReadDfsReplica`（`5046-5093`）。`TransferRead` 先校验 slices 总长 ≥ 对象大小；DFS 副本必须带 key 上下文，否则返回 `INVALID_REPLICA`。
4. 传输完成后 `ReleaseHotKey`（`1409-1411`）：hot cache block 有引用计数，读期间防逐出。
5. `VerifyObjectChecksum`（`1349-1372`）：只有 `object_checksum_enabled_` 且 Master 给了期望值才算；GPU slice 走 `ScopedObjectChecksumBuffer` 分块 D2H（`1294-1347`）。
6. `ShouldAdmitToHotCache(key, cache_used)`（`client_service.h:799-806`）：CountMinSketch 频率准入，`increment(key) >= admission_threshold_`（默认 2，`client_service.h:1166`）才 `ProcessSlicesAsync` 异步回填。回填的两个跳过条件：非 shm 模式下本地副本不缓存（`5620-5625`）；**目标 slice 在设备内存的逐片跳过**（`5627-5649`，host 片仍会回填）——写回用 host memcpy，读 CUDA 地址会 fault。
7. **最后**才 `query_result.IsLeaseExpired()`（`1444-1448`）。所以 `LEASE_EXPIRED` 时用户 buffer 里的数据其实已经完整、checksum 已过、hot cache 已准入，但传输期间租约过期意味着 Master 可能已淘汰/改写对象，必须报错。不能把它当成"读失败、buffer 未动"。

**BatchGet**（`Client::BatchGet(keys, query_results, slices)`，`1667-1932`）：所有 key 先并行 submit（DFS 副本单独攒成一次 `dfs_storage_backend_->BatchRead`，`1795-1861`），再统一 `future.get()` + 校验 + 准入，最后用**同一个 `now`** 批量检查租约（`1905-1915`，注释：租约过期是罕见事件，统一时间点省 syscall）。submit 失败会立即释放对应 hot cache 引用（`1779-1785`）。

**RealClient::get_into**（`4902-4915`）→ `get_into_range_internal`（`4884-4900`）→ `resolve_ranged_read_metadata` + `execute_ranged_read`（`4617-4882`）。`execute_ranged_read` 是"目的地能否直读"的决策树：

- **全对象 + LOCAL_DISK**：目的 buffer 直接交给 offload RPC（`4643-4662`），由 owner 的 SSD 侧 TE 写入；可选再校验 checksum。
- **全对象 + DISK/DFS**：文件 I/O 不能直接写任意用户 buffer，先分配临时 client buffer 读入，再 `scatter_host_to_maybe_device`（`4664-4696`）。
- **全对象 + MEMORY + 目的地是设备**：若"不能 local memcpy 且不能 `can_use_direct_memory_read`"，**或 hot cache 已启用**（回填需要 host 源），则先读进临时 buffer 再 H2D（`4698-4734`）；否则 `allocateSlices` 后直接以用户 buffer 为目的地调 `Client::Get`（该路径再按策略走 TE 或本地 memcpy，并非恒定 RDMA）（`4736-4747`）。
- **部分范围读**（`src_offset/size` 不等于全对象）：`4750-4882`。LOCAL_DISK 通常只需读 `src_offset + size` 字节（offload RPC 从远端 0 顺序传）；但若 owner 就是本进程且启用了 pinned restore arena，则走 `OffloadReadRange{src_offset, total_size}` 直接写用户 buffer（`4779-4797`）；DISK/DFS 必须整对象读入临时 buffer 再 scatter；MEMORY 且目的地是设备、不能直读时也先读临时 buffer 再 H2D，其余情况直接以 `{dst, size}` 作为单 slice 调 `Client::Get(..., src_offset)`。

`can_use_direct_memory_read`（`4570-4586`）的分支：`nvlink` 协议接受普通 CUDA 分配；`rdma` 协议要求 `resolve_writable_buffer_region` 命中且 size 放得下；其他协议一律 false。

**batch_get_into_internal**（`6280-6305` 是入口：先 `BatchQuery(keys)`；`6308-6699` 是主体）按副本类型分四路：

1. 每 key 先 `SelectBestReplica`（`6398-6401`）；buffer 容量不足直接 `INVALID_PARAMS`；
2. **LOCAL_DISK** → 按 owner endpoint 分组，攒进 `offload_objects`，每组一次 `local_disk_reader`（offload RPC，`6650-6665`），读后逐 key `VerifyObjectChecksum`（`6667-6692`）；
3. **DISK/DFS** → 每个 unique key 分配临时 buffer 攒进一次 `client_->BatchGet`，再 `scatter_host_to_maybe_device` 到用户 buffer（`6544-6615`）；
4. **MEMORY/NOF** → `allocateSlices(replica, buffers[i])` **直接以用户 buffer 为目的**攒进一次 `client_->BatchGet`（`6467-6480`）。

**重复 key** 必须专门处理，但三条路做法略有不同：MEMORY 与 DISK/DFS 两路用 `PlanDuplicateKeys`（`include/batch_read_fanout.h:41-56`），LOCAL_DISK 路在扫描阶段用 `valid_local_disk_operations.count(key)` 就地检测（`real_client.cpp:6420-6445`）。原因是 `batch_slices` 是以 string 为键的 map，重复 key 会让所有出现塌缩到同一个目的地。因此首次出现者做 primary 真正传输，传完 `FanOutDuplicates`（`batch_read_fanout.h:133`）用 device-aware 的 `CopyMaybeDevice` 把已校验字节镜像给重复项，并镜像 primary 的成功/失败结果。漏掉这层，重复 key 的批量读会静默丢数据。

**返回值语义混合**：成功项是传输字节数（正数），失败项是负 ErrorCode（`to_py_ret`，`include/common/result.h:26-37`）。`batch_get_into` 用 `sum_positive_results` 只把正数累加进指标，避免把 `-704` 当字节数统计（`real_client.cpp:5865`）。

### 流程四：`get_into_ranges` —— 三层嵌套的 scatter 汇聚（`real_client.cpp:4918-5199`）

签名形状是 `buffers[] × keys[][] × (dst_offsets/src_offsets/sizes)[][][]`，返回值同形三层 `int64_t`。这是给 vLLM/SGLang 的"多个目标 buffer，每个装多个 key 的多个不连续片段"一次调用表达。全错误结果由 `build_ranged_read_results_like` 模板族按输入形状构造（`pyclient.h:48-74`），避免各处手写三层循环。

主循环（`4968-5160`）对每个 (buffer i, key j)：

1. `metadata_for(key)`：请求级 `metadata_cache`，每 key 只 resolve 一次（`4951-4960`）。
2. **memory 副本 + 可 local memcpy + 目标是设备**（`5000-5082`）：不能进 scatter（scatter 是 TE 语义），逐 fragment 走 `client_->Get(key, qr, slices, src_offset)`。快照模式（`allow_query_refresh=false`）直接在本地比租约，过期即 `LEASE_EXPIRED`，**绝不重新 Query**；刷新模式则先重 query 换副本，并在租约剩余一半时再续（`refresh_at = now + (lease - now)/2`）。注释（`5020-5023`）特别说明：即使是全对象快照读也要绕过 key 级 hot cache——cache 里可能是比快照描述符更新的版本。
3. **memory 副本（一般情况）**（`5083-5116`）：转成一条 `ScatterTransferRange` 攒进 `memory_transfers`，`on_fragment_complete` 回调写结果槽位，并携带指向 `scatter_leases[key]` 的指针——回调触发时若该 key 租约已过期，即使传输成功也报 `LEASE_EXPIRED`（`5103-5113`）。
4. **文件类副本**（`5119-5133`）：逐 fragment `execute_ranged_read`。

提交前后的租约处理：

- 若允许刷新，先对全部 scatter key 做一次 `refresh_leases`（`BatchQuery`，`5150-5168`，注释：planning 本身可能耗掉短租约的大半）；
- `SubmitScatterNative` 一次性提交（`5169`）；
- 等待循环 `wait() / waitFor(delay)`（`5180-5192`）：delay 取所有未出错 key 的"剩余租约/2"的最小值（`5137-5150`），到点续租再等，直到返回非 `Clock` 状态。

`QueryResultCache` 与 `RangedReadSnapshot` 是两种不同的复用（`pyclient.h:237-310`）：前者是**单次请求内**的 batch_query 结果复用（注释明确不要跨请求保留）；后者是调用方持有的**跨请求**快照，`reset` 时取最早租约的一半算 `refresh_at`，`reusable()` 判过期。混用会导致读到已淘汰对象。

### 流程五：数据面策略分派（`TransferSubmitter::submit` / `submit_batch` / scatter）

单条 `submit`（`transfer_task.cpp:1077-1133`）：

```
submit(replica, slices, op_code, ptr, size, intent)
 ├ memory 副本:
 │   ├ READ  → submitMemoryReadOperation (1869-1886)
 │   │          └ selectStrategy → LOCAL_MEMCPY ? submitMemcpyOperation
 │   │                                          : submitTransferEngineOperation
 │   └ WRITE → selectStrategy (2040-2046) 同上二选一
 ├ nof 副本 (USE_NOF) → submitSpdkNofOperation (要求 ptr/size 且按 block 对齐)
 └ 其他(disk) → submitFileReadOperation (FilereadWorkerPool 异步读文件)
```

**纠正粗读地图的一个错误**：粗读说"单条 submit 里读操作恒定走 Transfer Engine、只有写才 selectStrategy"。实际 `submitMemoryReadOperation`（`1869-1886`）内部**同样调用 `selectStrategy`**，LOCAL_MEMCPY 时走 `submitMemcpyOperation(READ)`——单条路径读写都能 memcpy。

真正的不对称只在 `submit_batch`（`1135-1221`）：

- `use_local_memcpy = (op_code == WRITE && !replicas.empty())`（`1144-1145`），**READ 批次恒走 TE**；
- WRITE 批次对每个副本 `use_local_memcpy &&= canUseLocalMemcpy(...)`（`1157-1158`），只要有一个副本在远端，**整批**全部走 TE，不混合两种策略；
- 非 memcpy 路径每个副本 `openSegment` 一次，slice 逐个拼 `TransferRequest`（`1200-1209`）。

`selectStrategy`（`2040-2046`）就是 `canUseLocalMemcpy(endpoint) ? LOCAL_MEMCPY : TRANSFER_ENGINE`。`canUseLocalMemcpy`（`2048-2052`）= `memcpy_enabled_ && (endpoint == local_hostname_ || endpoint == local_endpoint_)`。`isSameProcessEndpoint`（`2054-2063`）要求**完整 transport endpoint（ip:port）相等**而非只比 IP，注释给出理由：`handle.buffer_address_` 必须是本进程有效 VA，同主机不同进程共享 IP 但地址空间独立，memcpy 过去会段错误。

推论：embedded 模式自己写自己读才可能走 memcpy；dummy-real 之间是不同进程，**不走** memcpy（它们靠共享内存 + 地址偏移）；RDMA 集群默认 `memcpy_enabled_=false`，loopback 也走 TE。

`MemcpyWorkerPool`（`transfer_task.cpp:650-790`）只有 **1 个** worker 线程（注释：memcpy 受内存带宽约束，多线程无益）。执行时对每个 operation 查 `AcceleratorRegistry`：两端都是 host 用 `std::memcpy`；涉及设备则按方向调 `AcceleratorDevice::Copy`（D2H/H2D/D2D）；跨 runtime 的 D2D 直接报错失败（`736-746`）。

`submitTransfer`（`1742-1812`）是 TE 提交的公共尾段：TENT 模式做 opcode/intent 翻译后 `tent_engine_->submitTransfer`，经典模式 `allocateBatchID + submitTransfer`；失败都会 freeBatch。返回的 `TransferEngineOperationState` 析构时释放 batch。

**scatter 路径**：`ScatterRangeBuilder`（`client_service.cpp:175-231`）持有 offset/length 存储并 reserve 到精确 fragment 数。原因是 `ScatterTransferRange` 持**非拥有 span**，push 时 vector 重分配会让活 span 悬垂（注释 `170-174` 写明）。每个 entry 一个 `error_slot`，回调在等待线程上跑，无需加锁（`184-185`）。`Client::BatchTransferReadRanges`（`4753-4804`）把所有 entry 的所有 fragment 塞进一个 builder，一次 `SubmitScatter`（经典 TE）→ `operation->wait()`；`BatchTransferReadOffloadRanges`（`4806-4880`）与 `BatchTransferWriteRanges`（`4882-4974`）走 `SubmitScatterNative`（TENT 用 TENT scatter，经典用 TE scatter）。失败时 `MarkScatterOperationFailure` 只覆盖没有 entry 级错误的槽位。

### 流程六：dummy-real 零拷贝闭环

**建立阶段**（一次性）：

1. `DummyClient::setup_dummy`（`dummy_client.cpp:496-580`）：`connect(server)`（coro_rpc，`ServiceReady` 探活）→ `ShmHelper::allocate(local_buffer_size)`（memfd_create + mmap）→ `register_shm_via_ipc`（`440-494`）：UDS 连 `@mooncake_client_<port>.sock`（abstract namespace），先发 `IPC_SHM_REGISTER` 类型字，再用 `sendFd`（SCM_RIGHTS，`uds_transport.cpp:185-212`）把 memfd 和 `ShmRegisterRequest{client_id, dummy_base_addr, shm_size, is_local_buffer}` 一起交给 real → 等 status。
2. `RealClient::handle_ipc_shm_register`（`real_client.cpp:8525-8540`）收 fd 后调 `map_shm_internal_with_device`（`2945-3133`）：mmap 同一物理页（NOF 场景可能 2MB 对齐 MAP_FIXED + `spdk_mem_register`，`3063-3126` 有大段 SPDK 回滚语义注释）→ 记 `MappedShm.shm_addr_offset = real_base - dummy_base`（`3029-3030`）→ `client_->RegisterLocalMemory` 把 real 侧映射注册进 TE（`3037-3045`）→ `is_local_buffer` 时在这块外部内存上建 `ClientBufferAllocator`（`3047-3056`）→ `mapped_shms.reserve(size+1)` 后才 push_back（`3058-3061`，防止注册成功后 vector 扩容抛 bad_alloc 泄漏注册）。
3. dummy 侧顺手 `request_hot_cache_fd`（`dummy_client.cpp:2188-2239`）：先发 `IPC_SHM_FD_REQUEST` + `ShmFdRequest{client_id, SHM_SEG_HOT_CACHE}`，real 侧校验 client_id 与 hot cache 可用后 `sendFd` 回 hot cache 的 shm fd（`real_client.cpp:8542-8592`）。dummy 以 `PROT_READ` 只读 mmap——之后 `get_buffer` 命中 hot cache 时**零 RPC 数据往返**，只发 acquire/release 控制 RPC，直接读本地映射（`dummy_client.cpp:1359-1402`）。

**运行阶段**（每次 put/get）：RPC 只传元数据和**地址整数**。以 `DummyClient::put`（`dummy_client.cpp:1174-1179`）为例：value span 在 dummy 进程里，`put_dummy_helper` RPC 到 real；real 侧（`real_client.cpp:2522-2541`）持 `dummy_client_mutex_` 共享锁查 `shm_contexts_[client_id]`，然后 `put_internal(key, value, config, context.client_buffer_allocator)`。span 的指针就是 dummy 的 shm VA，而 shm 两进程物理同页，`client_buffer_allocator` 又建在这块 shm 上，所以 allocate/gather/slice 全在 real 进程内完成，数据零拷贝。

读方向先试 CUDA IPC 快路径（已注册的设备 buffer 导出 IPC handle 走 `batch_get_into_cuda_ipc`），否则 `prepare_buffer`（`dummy_client.cpp:762-798`）按下列顺序决策：

1. Ascend 注册区 → 透传原地址（`766-772`）；
2. dummy 自己的 shm 区（`is_dummy_shm_buffer`，`639-647`）且不是设备指针 → 透传；
3. 已注册外部 buffer → **申请 staging**；`copy_to_staging` 为真时先 `CopyToHost` 拷入；`copy_back = copy_back || !copy_to_staging`（`780-798`）；
4. 其余（普通 malloc 未注册 / 未注册 CUDA 显存）→ 返回 nullopt，调用方报错。

所以 dummy 模式零拷贝是有前提的：写外部注册 buffer 会先拷进 staging（`copy_to_staging=true, copy_back=false`）；读外部注册 buffer 则读进 staging 再 `copy_from_staging` 拷回（`855-871`）。

real 侧收到 dummy VA 后用 `map_dummy_buffer_to_real`（`real_client.cpp:660-684`）换算：先试 `last_hit_shm` 局部性缓存，再遍历 `mapped_shms` 做区间检查（`map_dummy_range_in_shm`，`640-658`：三段溢出检查后 `out_real = dummy_addr + shm_addr_offset`）。批量场景 `map_dummy_addrs_to_real_ptrs`（`699-732`）复用同一个 `last_hit_shm` 指针。

**并发与生命周期**：real 侧 RPC handler 全程持 `dummy_client_mutex_` 共享锁。`batch_get_into_dummy_helper`（`real_client.cpp:5873-5935`）把 `shared_lock` 装进 `shared_ptr` 塞进堆上 `CallState`，再 `co_await coro_io::post` 到线程池执行可能阻塞的 SSD I/O——注释解释这是为了不让 coro_rpc IO 线程被占住（ping 需要它），同时避免 GCC 11 协程帧 double-destruction。unmap 需要独占锁，因此传输期间 shm 不可能被拆。

dummy 掉线由 ping 机制兜底：dummy 每 1 秒 `ping`（`dummy_client.cpp:2077-2180`），real 的 `ping`（`real_client.cpp:8399-8418`）把 client_id 压进 lockfree 队列，`dummy_client_monitor_func`（`8420-8464`）每秒消费、TTL 过期即 `unmap_shm_internal`。ping 失败一次即进入重连循环，**重注册全部 shm**（`2109-2140`）+ Ascend buffer 重注册，再 ping 验证才恢复 connected。

`prepare_ranged_read_buffer`（`dummy_client.cpp:800-853`）有个反直觉设计：staging 场景下 `dst_offsets` 被**原地改写**成 staging 内的紧凑偏移（`832-833`），越界 fragment 的 offset 标成 `size_t::max()`（`825`，注释：保留非法性让 RealClient 逐项报错，即使零长度 fragment 也不放行）；staging 只 reserve fragment 字节总数，不含稀疏空洞（`836-839`）。

### 流程七：控制面 RPC 形态（MasterClient）

所有 RPC 共用 `invoke_rpc_with_client_pool<ServiceMethod, ReturnType>`（`master_client.cpp:358-402`）：模板参数是 **`WrappedMasterService` 的成员函数指针**（如 `&WrappedMasterService::PutStart`），coro_rpc 编译期反射匹配服务端 handler。客户端和服务端签名改动会直接编译失败，没有字符串方法名的运行期错配风险；代价是每个 RPC 还要写一个 `RpcNameTraits<...>` 特化提供指标名（`master_client.cpp:26-355`，60+ 个），漏写编译不过。

调用被 `RpcDrainGuard::ScopedCall` 包住（`361-364`）：`~MasterClient`（`473-482`）先 `drain_for(30s)` 等在飞 RPC 落地再释放连接池（issue #3909）。DummyClient 析构同款（`dummy_client.cpp:278-299`），其注释解释了为什么必须"先 tearDownAll 再 drain"的反向顺序不可能成立（unregister_shm 本身就是 RPC）。

`syncAwait` 把协程结果拉回同步世界；`timed_out` 映射 `RPC_TIMEOUT`，其余 `RPC_FAIL`（`384-391`）。批量版 `invoke_batch_rpc`（`417-471`）在 RPC 级失败时给**每个元素**填同样的错误，保证返回 vector 长度与输入一致。

**三个连接池**（`master_client.h:100-107`）：

- `client_accessor_`：前台流量，YLT 默认重试；
- `ha_control_client_accessor_` 与 `ha_probe_client_accessor_`：`MakeMasterRpcClientPoolConfig(ha_enabled=true)`，即 `connect_retry_count=0` + 1 秒连接超时（`master_client.h:70-89`）。注释说明 HA 监控/心跳循环自己掌握重试节奏，不能让 YLT 默认的 4×30 秒重试预算阻塞 failover。

`Connect`（`master_client.cpp:492-533`）在 HA 模式先用 probe 池验证 `ServiceReady` + **版本严格相等**（`517-522`），才把前台流量切到新 leader（`523-528`）。非 HA 模式直接用前台池做同样检查。leader 跟随由 `Client::LeaderMonitorThreadMain`（`client_service.cpp:735-795`）watch 驱动，30 秒只是兜底 re-sync。

## 与其他模块的边界

- **上游 pybind**（`mooncake-integration/store/store_py.cpp`）：`MooncakeStorePyWrapper::init_real_client`（`542+`）建 RealClient；`setup_dummy` 绑定（`2533-2548`）建 DummyClient，**socket 名由 real client 端口推导**：`"@mooncake_client_" + port + ".sock"`。这与 `real_client_main.cpp:146` 的命名约定必须一致，否则 dummy 永远连不上 real。
- **C ABI**（`store_c.h/.cpp`）：
  - `mooncake_store_t` 实为 `StoreHandle*`（`store_c.cpp:34-36`），唯一职责是保活 `shared_ptr<RealClient>`；
  - 所有函数 try/catch 兜底返回 -1/nullptr；
  - `to_replicate_config`（`store_c.cpp:40-75`）**必须在 cast 之前**做 `soft_pin_action` 范围检查：`SoftPinAction` 底层是 `uint8_t`，直接 cast 会把越界整数回绕成合法枚举值；越界返回 nullopt，put 函数返回 -1；
  - `char*` 生命周期约定在 `store_c.h:56-62`；
  - `libmooncake_store.so` 的符号来自 whole-archive 静态 `mooncake_store` 库 + `store_c.map` 版本脚本（只 global `mooncake_store_*`，asio/glog/TE 全部 local 化，防止符号泄漏进宿主进程）。`store_c_shared.cpp` 是**故意留空的 TU**，只为满足 CMake "shared library 至少一个源文件"；`WITH_STORE_C_SHARED` 只支持 Linux/ELF（`CMakeLists.txt:784-831`）。
- **Transfer Engine**：Client 的数据面交互只经 `transfer_engine_->registerLocalMemory / unregisterLocalMemory / openSegment / submitTransfer / submitScatter / allocateBatchID`（控制/查询类另有 `getLocalIpAndPort` 等）；TENT 构建下 `TransferSubmitter` 与 `TransferEngineOperationState` 内部做双实现分派。`TransferIntent`（前台读/后台预取/迁移）透传给 TE 做 QoS 区分。
- **Master**：全部经 `MasterClient` → `WrappedMasterService`（成员函数指针编译期绑定）。租约、放置、副本状态机语义都定义在 Master 侧；客户端只消费 `GetReplicaListResponse{replicas, lease_ttl_ms, object_checksum, resolved_tenant_id}` 与 `PutStart` 返回的描述符列表。
- **store→store**：`ClientRequester`（`pyclient.h:143-229`）是 RealClient 之间的 offload RPC 客户端（`batch_get_offload_object` / `release_offload_buffer` / `verify_disk_replica`），对端 handler 注册在 real 自己的 offload coro_rpc server 上（`real_client.cpp:1461-1483`，端口 0 由内核分配）。LOCAL_DISK 读就是"请求方 Client → owner Client 的 FileStorage"这条边。
- **设备抽象**：主机/设备之间的**数据拷贝**统一走 `device::GetAcceleratorRegistry().RuntimeAccelerators()` 的 `CopyToHost/CopyFromHost/CopyAuto/FindDeviceForPointer`。但注意两类例外，别据此以为客户端完全不碰厂商 API：`registered_pinned_memory.cpp` 直接调 `cudaHostRegister/cudaHostUnregister`；Ascend 分支（`dummy_client.cpp` 的 `aclrt*` IPC/VMM handle 导出、`USE_ASCEND_DIRECT` 下的 `aclrtGetDevice` 等）也是直接调用。
- **编译边界**（`mooncake-store/src/CMakeLists.txt:6/20/101`）：SHARED（uds_transport、types 等双端共用）/ MASTER / CLIENT 三份源列表。`real_client.cpp`、`client_service.cpp`、`dummy_client.cpp`、`store_c.cpp` 都在 CLIENT 列表；`rpc_service.cpp` 在 MASTER 列表；`mooncake_client` 可执行在 `741` 行。
- **Rust 绑定**：`mooncake-store/rust` 用 bindgen 从 `store_c.h` 生成 FFI 并通过 dlopen 加载 `.so`；它与仓库根的 `mooncake-store-rs/`（独立重写的 Rust Store 实现）是两套东西，不要混谈。

## 设计取舍与易踩坑点

1. **LOCAL_MEMCPY 判定比"同主机"严格得多**。`isSameProcessEndpoint` 要求 endpoint 完整相等（`transfer_task.cpp:2054-2063`）。同主机不同进程（包括 dummy↔real、两个 embedded Client）都必须走 TE。另外默认只在 TCP-only 环境启用（`1058-1071`），RDMA 集群想开需 `MC_STORE_MEMCPY=1`。
2. **读写策略不对称只在 submit_batch**：单条 `submit` 读写都可 memcpy；`submit_batch` 中 READ 恒走 TE、WRITE 才可能 memcpy，且是**全批 AND**（`1144-1158`）——批量写只要混入一个远端副本，整批走 TE。不要把这条推广到单条路径。
3. **Put 的 best-effort 与幂等**：SINGLE/RELIABLE 模式任何传输失败都会 revoke 并返回错误；FLEXIBLE_DUAL（1 memory + 1 nof）一侧成功即返回 0，另一侧 revoke。但纯内存请求（未要求 NoF/DFS）的"分配是否满足"只检查 `allocated_memory_replicas > 0`（`client_service.cpp:322-336`）——即使 `replica_num=3` 而 Master 只分配到 1 个，只要这个副本传输成功，整体仍判成功。PutStart 遇 `OBJECT_ALREADY_EXISTS` **直接返回成功且不搬数据**（`client_service.cpp:1994-1999`）——上层拿到 0 不代表"这次写进去了"。
4. **LEASE_EXPIRED ≠ buffer 未动**：数据已完整落到用户 buffer、checksum 已过、hot cache 已准入，最后一步租约检查才失败（`1444-1448`）。批量路径同理（`1905-1915`）。
5. **ExistKey 授予读租约，ProbeKey 不授予**（语义见头文件 `master_client.h:134-166` 注释；实现侧 `ExistKey` 传 `grant_lease=true`、`ProbeKey` 传 `false`，见 `master_service.cpp:2988-2995`）：Probe 的 true 只说明调用那一刻存在。`BatchProbeKey` 带 `GrantLeasePolicy`，`LastHitOnly` 只给最后一个 all-true 候选上租约。用 Probe 结果去 Get 会遇到"检查过存在但读不到"。
6. **租约的本地时钟语义**：`lease_timeout` 用发起 RPC 前的 steady_clock 起点 + Master 的 ttl（`client_service.cpp:1240-1252`），网络耗时被计入租约，偏保守。所有比较都在客户端本地做，Master 不参与。
7. **kMaxSliceSize = 16MiB-16 来自 cachelib Slab**（`types.h:451-452`），减 16 的原因注释语焉不详（"should be lower than limit"）。改 cachelib 版本或 Slab 常量会静默改变切片粒度，且 Master 校验使用同一常量。
8. **register_buffer 会静默拓宽注册长度**：hugepage 模式下、且传入地址恰为 shm 段 base 时，注册长度被拓宽到整个物理段（`real_client.cpp:4471-4489`），否则 `ibv_fork_init` 后生效的 `madvise(MADV_DONTFORK)` 在内核拆分 hugetlb VMA 时会 EINVAL。但 `registered_buffer_sizes_` 里存的仍是**调用方原始 size**（`4496`）——两者不一致是故意的，`resolve_writable_buffer_region` 按逻辑 size 判界。同一"按段实际大小注册"的规则也用在 hot cache 的 shm 注册上：`InitLocalHotCache` 会优先取 `shm_seg->size` 而不是 `GetTotalSize()` 去 `registerLocalMemory`（`client_service.cpp:5554-5561`），原因同样是避免 MR 末端落在最后一个 huge page 中间。
9. **resolve_writable_buffer_region 排除 CUDA/UVA 误判**：设备地址在数值上可能落进 host 本地 buffer 区间，只有非设备指针才把 `local_buffer_region_` 当隐式可写区（`real_client.cpp:4549-4566`）。
10. **DummyClient 不是测试桩**，是生产代理；但它的"零拷贝"有前提——`prepare_buffer` 只透传 Ascend 注册区/dummy shm 区，外部注册 buffer 走 staging 双拷贝（写时 CopyToHost，读后 copy_from_staging）。用户随手传的 malloc 内存直接 nullopt。
11. **数据从不走 UDS/coro_rpc**：UDS 是一次性阻塞单连接控制通道（`uds_transport.h:16-19` 注释明确别拿它做高吞吐 I/O）；coro_rpc 只传元数据与地址整数。真实字节走共享内存 + TE。
12. **store_c_shared.cpp 是空的，而且必须空**（`store_c_shared.cpp:15-21` 注释）：C ABI 符号来自 whole-archive 的静态库，这个 TU 只为满足 CMake 要求。删掉它会导致 CMake 报"no source files"。
13. **C ABI 的 soft_pin_action 必须先校验再 cast**（`store_c.cpp:47-53`）：越界整数会回绕成合法枚举值，这是 C ABI 边界上典型的"类型安全靠自己"。
14. **registered_pinned_memory 的三态语义容易读反**：`UnregisterResult::kSuccess` 正常；`kRuntimeUnloading` 表示 `cudaHostUnregister` 返回 `cudaErrorCudartUnloading`（CUDA runtime 正在卸载），此时 `release()` **只打 warning 并返回 true**（`registered_pinned_memory.cpp:200-214`），因为进程即将退出，free 是安全的；真正危险的是 `kError`，`release()` 返回 false（`registered_pinned_memory.cpp:193-210`），`ReleasePinnedRegionForFree`（`real_client.cpp:166-177`）打印 "Leaking ... backing memory" 并拒绝 free，`setup_internal` 相应置 `setup_segment_memory_must_leak_`（`real_client.cpp:1418-1426`），teardown 时据此拒绝释放整块 segment（`1836-1843`）。**"宁可泄漏"针对的是 kError，不是 kRuntimeUnloading。**
15. **pin 只对 host 协议生效**：`IsHostStoreSegmentProtocol` 白名单是 empty/tcp/rdma/efa/cxi/rpc_only（`real_client.cpp:110-113`）。nvlink、ascend、cxl、sunrise_link 等设备内存协议直接跳过——它们的内存本来就设备可达。
16. **shm 拆除的 quarantine 机制**：TE unregister 或 SPDK unregister 失败的映射**绝不 munmap**（VA 被复用后旧注册会 DMA 到错误内存），而是移进 `quarantine_shms_`，`RetryQuarantinedShmsLocked`（`real_client.cpp:1911-1954`）在后续 teardown 重试，永远失败就保留到进程退出。读 `unmap_shm_internal` / `tearDownAll_internal` 时别把"泄漏"当 bug。
17. **tearDownAll 的 CAS 与析构顺序**：析构、显式 close、信号处理三条路径都可能触发清理，`closed_.compare_exchange_strong`（`real_client.cpp:1753-1757`）去重。`ResourceTracker` 必须在 `main` 第一行初始化（`real_client_main.cpp:115-119` 注释：否则主线程不应用信号掩码，后创建的线程错过信号处理）；它用 `dlsym(RTLD_DEFAULT, "Py_IsInitialized")` 探测 Python 环境决定是否自装信号线程（`real_client.cpp:787-796`），避免吞掉 KeyboardInterrupt。
18. **AlignedClientBufferAllocator 与主 buffer 无关**：4096 对齐是 O_DIRECT + io_uring 的硬要求，唯一使用者是 `file_storage.cpp`（SSD offload staging）。主 `ClientBufferAllocator` 只按 64B 对齐。把它当"主 buffer 的对齐版"会误判影响面。
19. **hot cache 有三道闸门**：读命中 `RedirectToHotCache` 改写描述符；未命中回填要过 `ShouldAdmitToHotCache`（CMS 频率准入，阈值默认 2）；写路径开头 `RemoveHotKey` / `RemoveHotKeys` 主动失效（`1983-1985`、`2584-2589`）。`LocalHotCache::RemoveHotKey` / `RemoveHotKeys` 递增 `key_generation_`（等价于 `BumpKeyGeneration`），使在飞异步回填 token 失效。设备内存目的地跳过回填（#4764）；`object_checksum_enabled_` 时 hot cache 直接不启用（`5518-5524`）。
20. **Client 不只是请求发起方**：它还有 HA leader monitor、storage heartbeat、task poll 三个后台线程（`client_service.h:1074-1085`）和 `task_thread_pool_`（4 线程），会轮询 Master 领取 Copy/Move 任务并作为执行方搬数据——`ExecuteReplicaTransfer`/`Copy`/`Move` 在 `client_service.cpp:4355-4560`，`TaskPollThreadMain` 在 `5125`，`ExecuteTask` 在 `5150`——副本复制/迁移的数据面同样跑在 Client 里。
21. **to_py_ret 把错误码直接当返回值**（`common/result.h:26-37`）：Python/C 侧看到 `-704` 是 OBJECT_NOT_FOUND、`-200` 是 NO_AVAILABLE_HANDLE，不是通用 -1。批量接口成功项是正字节数、失败项是负错误码，同一数组混合。
22. **put 与 put_from 的代价差异**：`put` 先分配 client buffer 再 gather（可能 D2H）后切片；`put_from` 直接对用户 buffer 切片，零拷贝，但地址必须落在已注册内存里。C ABI 同时暴露两者，选错就是性能差一倍或直接失败。
23. **TransferEngineOperationState 等全部 task 终态才置结果**（`transfer_task.cpp:814-903` 注释）：即使部分已失败也不提前 complete，否则调用方析构触发 `freeBatchID` 会和在飞传输竞争。`set_result_internal` 的二次置位只打日志不 crash（`905-918`）。
24. **embedded master 是测试/评估用途**：`store_py.cpp` docstring 原话 "This is test/eval plumbing, not a documented deployment mode"（`2531`）。`StartEmbeddedMaster` 在进程内起 Master（默认 P2PHANDSHAKE 元数据模式、内核分配端口），不要误以为生产特性。
25. **P2PHANDSHAKE 模式下 endpoint 语义不同**：segment/描述符里的 endpoint 用 TE 实际 `getLocalIpAndPort()` 而非逻辑 hostname（`RedirectToHotCache` 里按 `metadata_connstring_ == P2PHANDSHAKE` 分支，`client_service.cpp:1949-1953`；`IsReplicaOnLocalMemory` 同款，`5652-5664`）。做 endpoint 比较的代码都要意识到有两种口径。
26. **duplicate key 需要专门处理**：`batch_read_fanout.h:10-13` 解释了原因——传输批次结构以 string 为 key，同一 key 出现两次会让所有出现塌缩到同一个目标。MEMORY 与 DISK/DFS 路用 `PlanDuplicateKeys`，LOCAL_DISK 路在扫描时用 map 计数检测；传完再 `FanOutDuplicates`，字节用 device-aware 的 `CopyMaybeDevice` 搬。漏掉这层，重复 key 的批量读会静默丢数据。
27. **KV session 是 RealClient 里的完整状态机**：`get_sessions_` / `put_sessions_`（`real_client.h:1126-1145`）是进程内会话表，不与 DummyClient 共享。ranges 只在本地比租约、不再问 Master；Put 会话要靠 end/revoke 封会话并等待在飞 range 写完成才能 finalize/free。这是 vLLM/SGLang 增量写读的基础，粗读时很容易整段跳过。
28. **`mooncake-store/rust` ≠ `mooncake-store-rs/`**：前者是 C ABI 的 bindgen 绑定（`allowlist_function("mooncake_store_.*")`，dlopen 加载 `.so`）；后者是独立重写的 Rust Store 实现，有独立架构文档，与本子系统并行。

## 精读清单

按依赖顺序读，每步标注"该看懂什么"：

1. **`include/types.h:445-452` + `include/replica.h:108-237` + `include/allocator.h:143-150`**（30 分钟）
   看懂 Slice/kMaxSliceSize、ReplicateConfig 三种写模式、`AllocatedBuffer::Descriptor` 四字段。这是后面一切的地基。
2. **`include/client_buffer.h` + `src/client_buffer.cpp` 全文**（194 行，30 分钟）
   看懂 owning/view 双模式 BufferHandle、64B 对齐、`split_into_slices` 与 `allocateSlices` 对不同副本类型的切分差异。
3. **`src/client_service.cpp:1959-2174`（Client::Put + heal）**（1 小时）
   对照流程一走一遍；重点数清 `ReplicaTransferSummary` 的每次 Record 调用点，再读 `DetermineFinalizeDecision`（`262-418`）验证"什么时候 revoke、什么时候仍成功"。
4. **`src/client_service.cpp:1382-1495` + `5010-5044` + `5459-5472`（Get 家族）**（1 小时）
   背下七步顺序（选副本 → hot cache → 传输 → 释放 → 校验 → 准入 → 租约）。再跳读 `replica_selection.h:122-172` 和 `real_client.cpp:3830-3845`（FilterQueryResult），理解"优选在 RealClient 层、Client::Get 只见单副本"的分工。
5. **`src/transfer_task.cpp:1042-1221 + 1742-1905 + 2040-2078`**（1 小时）
   看懂 submit/submit_batch 的策略分派、`isSameProcessEndpoint` 注释、`memcpy_enabled_` 的 TCP-only 默认。再读 `TransferEngineOperationState`（`795-1013`）理解 batch 生命周期与两种等待模式。
6. **`src/client_service.cpp:2329-2443 + 2572-2674 + 2758-2882 + 3095-3297 + 3596-3635`（BatchPut 流水线）**（1.5 小时）
   以 PutOperation 状态机为主线，确认每个阶段都先跳 `IsResolved()`；FinalizeBatchPut 的 6 组分发与 `pending_finalize_actions` 计数值得逐行读。
7. **`src/real_client.cpp:990-1540`（setup_internal）**（1 小时）
   看懂：端口自动绑定重试、client buffer 分配 + 注册 + `local_buffer_region_`、segment 分块挂载循环（NUMA/hugepage/EGM 分支可略读）、`TryPinStoreSegment`、IPC server 与 offload RPC server 启动、FileStorage 与 `SetLocalDiskProbe` 的接线。
8. **`src/real_client.cpp:4464-4586 + 4617-4915`（register_buffer / execute_ranged_read / get_into）**（1 小时）
   看懂 HugeTLB 拓宽、`resolve_writable_buffer_region` 的 CUDA 排除、`can_use_direct_memory_read` 的协议分支，以及 execute_ranged_read 的四路决策树。
9. **`src/real_client.cpp:4918-5199`（get_into_ranges_internal）**（1 小时）
   全库最复杂的读路径。抓住三条线：metadata_cache、scatter_leases + on_fragment_complete 回调、refresh 循环（wait/waitFor）。对照 `pyclient.h:237-310` 理解 cache 与 snapshot 的契约差别。
10. **`src/real_client.cpp:6280-6699`（batch_get_into_internal）+ `include/batch_read_fanout.h`**（1 小时）
    看懂四路分流（LOCAL_DISK offload RPC / DISK-DFS 临时 buffer / MEMORY 直读用户 buffer）与重复 key 的 Plan → transfer → FanOut。
11. **`src/master_client.cpp:358-533` + `include/master_client.h:60-110`**（40 分钟）
    看懂成员函数指针路由、RpcNameTraits、RpcDrainGuard、三个连接池与 HA fast-fail 配置的理由。
12. **`src/dummy_client.cpp:440-580 + 622-871 + 1174-1402 + 2077-2239` 与 `src/real_client.cpp:2945-3133 + 8485-8592 + 640-732`**（1.5 小时）
    按流程六走一遍 dummy-real 闭环：memfd → UDS sendFd → mmap → addr_offset → 每次请求只传整数。重点读 `prepare_buffer` 分支、`prepare_ranged_read_buffer` 的偏移改写、`map_shm_internal_with_device` 的注册/SPDK/reserve 细节、ping 重连循环。
13. **`src/store_c.cpp` + `include/store_c.h` + `src/store_c.map` + `src/CMakeLists.txt:784-831`**（30 分钟）
    看懂 C ABI 三决定：句柄保活、先校验后 cast、whole-archive + version script。
14. **`src/registered_pinned_memory.cpp` + `include/client_metric.h:79-89`（`execute_timed_operation`）+ `958-1028`（`ClientMetric` 聚合体）+ `src/client_metric.cpp` + `src/aligned_client_buffer.cpp`**（40 分钟）
    小而精的收尾：三态 UnregisterResult 与"宁可泄漏"的真实边界、`execute_timed_operation` 模板与指标聚合、4096 对齐分配器的真实作用域。
15. **（选读）`src/client_service.cpp:4355-4555`（Copy/Move 任务执行）与 `735-807`（HA 线程）**
    理解 Client 作为"后台任务执行方"的第二身份，补全对 Client 生命周期的认识。

阅读时建议始终带着三个问题：

- 这一步失败时，数据/元数据分别处于什么状态？调用方看到什么错误？
- 这块内存/这个 fd/这条 shm 映射的所有权在谁手上，谁负责释放？
- 这个判断用的是逻辑 endpoint 还是 TE 实际 endpoint？是 host 内存还是设备内存？

未完全确认之处（阅读时请自行核实）：

- `kMaxSliceSize` 减 16 字节的精确动机。注释只有 "should be lower than limit"，推测与 cachelib 分配器 header/对齐预留有关，未在 allocator 代码中逐行验证。
- TENT 模式下 `StoreScatterTransferOperation::Impl` 的 segment 引用计数与 `forceCleanup` 的完整语义（`transfer_task.cpp:1223-1596` 通读了框架，未逐行核对 `retryCloseSegments` 的所有分支）。
- Ascend agent 模式（`USE_ASCEND_DIRECT`）下 VMM/fabric handle 的注册路径（`ascend_shm_internal` / `register_ascend_shm`）只确认了与 memfd 路径的接口对称性，设备侧细节未深读。
- `KV session` 的 ranged put 路径（`SessionRangeReadPlan` 之外的 `put_sessions_` 写侧）只读了会话结构与 finalize 注释，未逐行跟踪所有 range 提交/封会话分支。

---

# 08 高可用与主备复制

> 本章基于仓库 `D:/xiaojun/work/Mooncake`（commit `67df8c41`）的实际源码精读写成。所有技术断言均带 `相对路径:行号`，可直接跳转核对。
>
> **路径约定**：下文所有相对路径都以仓库根目录为基准。注意 HA 代码全部位于 `mooncake-store/` 子目录下（`mooncake-store/src/ha/...`、`mooncake-store/include/ha/...`），仓库根目录**没有**独立的 `src/ha/`。此外 `hot_standby_service.cpp`、`standby_state_machine.cpp`、`shutdown_request.cpp` 等单文件位于 `mooncake-store/src/` 而非 `mooncake-store/src/ha/`。

---

## 定位与职责

Mooncake Store 的元数据主节点（`mooncake_master`）是单点：所有 `PUT_START/PUT_END/GET_REPLICA_LIST/MountSegment` 等元数据操作都由它裁决。一旦它崩溃，整个 Store 集群的元数据面就瘫痪了（数据面 Transfer Engine 不受影响，但新的读写无法建立）。

HA 子系统解决这个问题，做法是**租约式选主 + 有序 OpLog 复制 + 分块快照**：

1. 多个 `mooncake_master` 进程同时运行，通过外部协调后端（etcd / Redis / Kubernetes Lease）抢一个租约，抢到的当 Primary，其余当 Standby。
2. Primary 在服务流量的同时，把每个元数据变更（`PUT_END`、`REMOVE`、`SEGMENT_MOUNT` …）编码成 `OpLogEntry`，攒成有序批次（`OpLogBatchRecord`）通过 CAS 事务写进 KV 后端，并推进一个单调的 `durable_prefix`。
3. Standby 轮询 `durable_prefix`，把批次拉下来严格按 `sequence_id` 顺序应用到本地的影子存储 `StandbyMetadataStore`。
4. Standby 可选地周期性把"已应用状态"物化成快照上传到本地/S3 对象存储，用于快速 bootstrap 和 OpLog 修剪。
5. Primary 挂掉后，Standby 抢到租约 → 追平 → 导出 `PromotionContext` → 恢复并校验 → 复验租约 → 才暴露 RPC。

它在 Mooncake 中的位置是一个**纵向切片**：位于 `MasterService` / RPC 层之下、KV 后端之上，完全不影响 Transfer Engine。

### 三个独立开关（最容易误解的地方）

`enable_ha` **不等于**开启复制。这是三个正交的开关，默认值都是保守的：

| 开关 | 默认 | 作用 | 定义位置 |
|---|---|---|---|
| `--enable_ha` | `false` | 走 supervisor 选主循环 | `mooncake-store/src/master.cpp:236` |
| `--enable_oplog` | `false` | 开启 Primary 批写 + Standby 跟随 | `mooncake-store/src/master.cpp:381` |
| `--enable_oplog_snapshot` | `false` | 开启 Standby 生成快照 | `mooncake-store/src/master.cpp:386` |

只配 `enable_ha` 得到的是"选主 + 空 standby 晋升"——`CreateStandbyController` 会退化成 `NoopStandbyController`（`mooncake-store/src/ha/standby_controller.cpp:468-481`），它的 `PromoteStandbyAndExport()` 直接返回一个**空的** `PromotionContext{}`（`standby_controller.cpp:89-92`）。也就是说：新 Primary 会以空元数据开始服务。这在"元数据可从客户端重建"的部署里可以接受，在需要保留元数据的场景里是数据丢失。

配置校验只保证了依赖方向（`mooncake-store/src/master.cpp:1491-1498`）：`enable_oplog` 要求 `enable_ha=true`，且后端必须是 etcd 或 redis。

还有一个**代码层面的硬约束**：`enable_oplog_snapshot=true` 只支持 etcd 后端。`CapabilityDrivenStandbyController` 的构造函数在 `spec_.type != HABackendType::ETCD`（或没有 OpLog 跟随、chunk count 为 0）时直接抛异常（`mooncake-store/src/ha/standby_controller.cpp:122-130`），进程启动失败。也就是说 Redis 可以跟 OpLog，但不能生成 batch 快照。

---

## 目录结构速览

| 文件 | 职责 |
|---|---|
| `mooncake-store/src/master.cpp:1674-1702` | `main()` 的 `enable_ha` 分支：先看门狗、再装信号处理器、最后构造 supervisor |
| `mooncake-store/src/shutdown_request.cpp` | 进程级优雅退出原语（112 行，语义极精密） |
| `mooncake-store/include/ha/leadership/master_service_supervisor.h` | supervisor 入口 + `ServingStateGate` + `kHaShutdownDeadline=15s` |
| `mooncake-store/src/ha/leadership/master_service_supervisor.cpp` | **HA 的心脏**：`RunSupervisorLoop`（1056 行文件，主循环占 416-1015） |
| `mooncake-store/include/ha/ha_types.h` | 核心类型：`MasterView` / `LeadershipSession` / `MasterRuntimeState` / `HaBackendSupportsOpLog` |
| `mooncake-store/include/ha/leadership/leader_coordinator.h` | 选主抽象接口（6 个纯虚函数 + 带默认实现的 `PublishServiceReady`） |
| `mooncake-store/src/ha/leadership/leader_coordinator_factory.cpp` | 三后端选主分发 |
| `mooncake-store/src/ha/leadership/backends/etcd/etcd_leader_coordinator.cpp` | 生产路径的选主实现（741 行） |
| `mooncake-store/src/ha/leadership/backends/{redis,k8s}/` | Redis（实验性）、K8s Lease（只选主） |
| `mooncake-store/include/ha/leadership/leader_label_reconciler.h` | K8s pod label 的后台收敛器 |
| `mooncake-store/include/ha/standby_controller.h` | `PromotionContext` + `StandbyController` 接口 |
| `mooncake-store/src/ha/standby_controller.cpp` | 能力驱动的 standby 控制器 + noop 退化 |
| `mooncake-store/src/hot_standby_service.cpp` | **Standby 全生命周期**（1365 行） |
| `mooncake-store/src/standby_state_machine.cpp` / `include/standby_state_machine.h` | 九态状态机 + 转移表 |
| `mooncake-store/include/ha/standby_metadata_store.h` | Standby 侧影子元数据存储（含快照游标） |
| `mooncake-store/include/ha/oplog/oplog_types.h` | 复制协议数据契约：`OpType` / `OpLogEntry` |
| `mooncake-store/include/ha/oplog/oplog_batch_types.h` + `src/ha/oplog/oplog_batch_types.cpp` | 批次记录结构 + KV key 布局 |
| `mooncake-store/src/ha/oplog/oplog_batch_storage.cpp` | **存储层真相**：CAS + fence + 修剪（573 行） |
| `mooncake-store/src/ha/oplog/ordered_oplog_writer.cpp` | Primary 侧有序批写（721 行） |
| `mooncake-store/src/ha/oplog/oplog_batch_standby_reader.cpp` | Standby 侧轮询读取 + 一致性判定（239 行） |
| `mooncake-store/src/ha/oplog/oplog_applier.cpp` | 按序应用到影子存储 |
| `mooncake-store/include/ha/kv/ha_kv_backend.h` | 最小 KV 抽象（`Get/Put/Range/DeleteRange/SupportsTxn/Txn`） |
| `mooncake-store/src/ha/snapshot/batch_oplog/batch_oplog_snapshot_coordinator.cpp` | 快照调度：capture→write→publish→GC→prune |
| `mooncake-store/src/ha/snapshot/batch_oplog/batch_oplog_snapshot_publisher.cpp` | `latest`/`fallback` 双指针轮转发布 |
| `mooncake-store/src/ha/snapshot/batch_oplog/batch_oplog_pruning_coordinator.cpp` | compaction floor + 删批次 |
| `mooncake-store/src/ha/snapshot/batch_oplog/batch_oplog_snapshot_provider.cpp` | 恢复：latest→fallback→全量回放 |
| `mooncake-store/src/ha/snapshot/snapshot_maintenance_lease.cpp` | 快照维护互斥租约 |
| `mooncake-store/src/ha_metric_manager.cpp` | HA Prometheus 指标（698 行） |
| `mooncake-store/src/etcd_helper.cpp` / `include/etcd_helper.h` | Go clientv3 的 C++ 薄封装（cgo） |
| `mooncake-store/src/k8s_lease_helper.cpp` | Go client-go leaderelection 的薄封装 |
| `mooncake-store/src/embedded_master.cpp` | **与 HA 无关**：进程内单点 master，全绑 loopback |
| `docs/source/deployment/mooncake-store-deployment-guide.md:342-505` | HA 的事实设计文档（含 Recovery from Unusable HA State、重置 legacy 命名空间） |
| `docs/source/design/store/mooncake-store.md:58-65` | Snapshot & Restore 设计说明 |

---

## 核心数据结构

### `MasterView` / `LeadershipSession`（`mooncake-store/include/ha/ha_types.h:106-117`）

```cpp
struct MasterView { std::string leader_address; ViewVersionId view_version = 0; };
struct LeadershipSession {
    MasterView view;
    OwnerToken owner_token;          // 后端私有，只有铸造它的后端能解释
    std::chrono::milliseconds lease_ttl{0};
};
```

`ViewVersionId` 是 `EtcdRevisionId`（`int64_t`）的别名（`mooncake-store/include/types.h:211-212`）。**但别名有误导性**：三个后端赋予它的语义并不相同：

- **etcd**：`CreateWithLease` 返回的 etcd revision（`etcd_leader_coordinator.cpp:229-231`）——真正的全局单调修订号。
- **K8s**：`lease_transitions`（`k8s_leader_coordinator.cpp:148`）——选举换手次数，单调但粒度粗。
- **Redis**：Lua 脚本里的 `INCR` 计数器（`redis_leader_coordinator.cpp:45-59`，`INCR` 在第 49 行；C++ 侧在 `:390-395` 解析脚本返回的版本号）。

`owner_token` 同理是后端私有的：etcd 是 `std::to_string(lease_id)`（`etcd_leader_coordinator.cpp:645-647`），Redis 是 `UuidToString(generate_uuid())` 随机 token（`redis_leader_coordinator.cpp:679-681`），K8s 是 `ns/lease/transitions/sequence/leader_address` 拼串（`k8s_leader_coordinator.cpp:28-37`）。这个设计让 `LeaderCoordinator` 接口保持后端中立，代价是 token 无法跨后端比较——这也是为什么 `ClaimProducerView` 用的是 `view_version` 而不是 `owner_token`。

### `MasterRuntimeState` 七态（`ha_types.h:136-144`，角色映射见 `:166-179`）

`kStarting / kStandby / kCandidate / kRecovering / kCatchingUp / kLeaderWarmup / kServing`。

它面向**外部上报**（`MasterAdminServer::SetRuntimeState`，供 Prometheus / HTTP 探针读取），`MasterRuntimeRoleToString` 把它们粗分成 `leader`（只有 `kLeaderWarmup`、`kServing`）和 `standby`（其余五个）。

注意 `kCatchingUp` 并不由 supervisor 直接设置，而是 `MapStandbyRuntimeState` 从 `StandbySyncStatus` 映射出来的：只有当"开了 oplog 跟随 + 有观察到的 leader + lag > 0"时才是 `kCatchingUp`，否则 `WATCHING` 映射成 `kStandby`（`standby_controller.cpp:66-71`）。

### `StandbyState` 九态（`include/standby_state_machine.h:49-76`）

`STOPPED / CONNECTING / SYNCING / WATCHING / RECOVERING / RECONNECTING / PROMOTING / PROMOTED / FAILED`。

这是**另一套层次**的状态机，面向 `HotStandbyService` 内部复制流程。两套状态机通过 `MapStandbyRuntimeState`（`standby_controller.cpp:53-77`）单向映射。转移表是手写的 switch（`standby_state_machine.cpp:10-`），关键约束：

- `IsReadyForPromotion()` 只在 `WATCHING` 返回 true（`standby_state_machine.h:246-248`）——所以 `SYNCING` 阶段无法被晋升，必须先 `SYNC_COMPLETE` 进入 `WATCHING`。
- `IsConnected()` 包含 `RECOVERING` 和 `PROMOTING`（`standby_state_machine.h:232-236`）。

### `OpLogEntry`（`include/ha/oplog/oplog_types.h:75-84`）

```cpp
struct OpLogEntry {
    uint64_t sequence_id{0};      // 全局严格递增，由 writer 分配
    uint64_t timestamp_ms{0};     // Commit 时被强制清零（见下）
    OpType op_type{OpType::PUT_END};
    std::string tenant_id{"default"};
    std::string object_key;
    std::string payload;          // 该 OpType 的负载（struct_pack / msgpack 二进制）
    uint32_t checksum{0};         // XXH32(payload)
    uint32_t prefix_hash{0};      // Commit 时被强制清零
};
```

`OpType` 共 11 种（`oplog_types.h:13-26`）：`PUT_END / PUT_REVOKE / REMOVE / LEASE_RENEW / SEGMENT_MOUNT / SEGMENT_UNMOUNT / SEGMENT_UPDATE / WEIGHT_METADATA_UPSERT / WEIGHT_METADATA_DELETE / WEIGHT_LEASE_UPSERT / WEIGHT_LEASE_DELETE`，外加哨兵 `OP_TYPE_MAX` 用于范围校验。

**刻意清零的字段**：`Commit` 和 `CommitBatch` 都强制 `entry.timestamp_ms = 0; entry.prefix_hash = 0;`（`ordered_oplog_writer.cpp:326-328`、`387-389`），然后重算 `checksum = ComputeOpLogChecksum(entry.payload)`。这意味着这两个字段虽然在结构里，但在批次路径上恒为 0——`VerifyOpLogChecksum` 对 `prefix_hash == 0` 显式放行（`oplog_types.cpp:29-33`）。这两个字段看起来是 legacy 单条 OpLog 布局的遗留（`RejectLegacyLayout` 仍在拒绝 20 位数字命名的单条 OpLog 键），代码里没有说明保留原因；读批次路径时不要被它们误导。

大小上限：`kMaxOpLogObjectKeySize = 4096`、`kMaxOpLogPayloadSize = 10MB`（`oplog_types.h:86-87`），在写入端 `ValidateOpLogBatchEntry`（`oplog_batch_types.cpp:81-95`）和读取端 `ApplyOpLogEntry`（`oplog_applier.cpp:50-56`）**双向**校验——读取端校验是 DoS 防护，防止恶意/损坏的 etcd 数据把 standby 打爆。

**一个协议实现细节**：`LEASE_RENEW` 虽然在 `OpType` 里占了一个枚举值，但全仓没有任何生产代码会发出它（`grep LEASE_RENEW` 只命中枚举定义），而 `OpLogApplier::ApplyOpLogEntry` 的 `switch` 也没有对应分支（`oplog_applier.cpp:86-123`，会走 `default` 返回 false，进而被 reader 判成 `INCOMPLETE_OPLOG_CATCH_UP`）。所以它目前只是一个保留值，不要据此推断"租约续期会被复制"。

### 批次记录的线上格式（`src/ha/oplog/oplog_batch_codec.cpp`）

批次记录不是直接序列化 `OpLogBatchRecord`，而是一个紧凑的 JSON 数组（`BatchRecordToJson`，`oplog_batch_codec.cpp:76-97`）：

```
[schema_version, batch_id, first_seq, last_seq,
 [ [op_type, tenant_id, object_key, base64(payload)], ... ],
 checksum]
```

三个容易看漏的点：

1. **`sequence_id` 不入盘**。解码时按位置推导：`entry.sequence_id = batch.first_seq + i`（`oplog_batch_codec.cpp:259-260`，赋值在 `:134`）。这正是 `ValidateOpLogBatchRecordShape` 要求条目序号连续的原因——序号是位置函数，不是独立字段。
2. **`timestamp_ms` 和 `prefix_hash` 不入盘**。解码时分别置 0，`checksum` 则用 `ComputeOpLogChecksum(payload)` 重算（`:134-141`）。所以这些字段在复制路径上不携带信息。
3. **批次自身有 XXH32 校验和**，覆盖前五个数组元素（`:250-263`）；解码先校验批次 checksum，再逐条校验 payload 的 base64 规范性和 `OpLogEntry` 大小/类型（`:110-146`）。payload 的 base64 还要求"解码再编码后字节完全一致"（`:128-131`），防止同一 payload 有多种编码表示。

### `DurablePrefix` 与 `OpLogBatchRecord`（`include/ha/oplog/oplog_batch_types.h`）

```cpp
struct DurablePrefix { uint64_t batch_id{0}; uint64_t last_seq{0}; };
struct OpLogBatchRecord {
    uint32_t schema_version{1}; uint64_t batch_id{0};
    uint64_t first_seq{0}; uint64_t last_seq{0};
    std::vector<OpLogEntry> entries; uint32_t checksum{0};
};
```

`DurablePrefix` 是整个复制协议的**唯一真相点**：它表示"batch_id 及之前的所有批次、直到 last_seq 的所有条目都已持久"。`batch_id == 0` 与 `last_seq == 0` 必须同时为零或同时非零（多处断言：`oplog_batch_storage.cpp:281-286`、`oplog_batch_standby_reader.cpp:34-36`）。

`ValidateOpLogBatchRecordShape`（`oplog_batch_types.cpp:34-78`）强制：`batch_id != 0`、`first_seq != 0`、`entries` 非空、`first_seq`/`last_seq` 与首尾条目一致、**序号必须连续**（`last_seq - first_seq == entries.size() - 1`，且逐条 `entries[i].sequence_id == first_seq + i`）。连续性是 standby 能"只读 durable_prefix 就断定历史完整"的前提。

### KV key 布局（`src/ha/oplog/oplog_batch_types.cpp`）

```
/oplog/<cluster_id>/batches/<20位零填充batch_id>   批次记录
/oplog/<cluster_id>/durable_prefix                 持久前缀
/oplog/<cluster_id>/producer_view                  当前 producer 的 view_version
/oplog/<cluster_id>/snapshot/{latest,fallback,maintenance,compaction_floor}  快照控制键
mooncake_master_snapshot/<cluster_id>/...          对象存储里的快照制品根
```

batch_id 用 `std::setw(20) << std::setfill('0')` 零填充（`oplog_batch_types.cpp:110-115`，`kOpLogBatchIdWidth = 20`），这样字典序 == 数值序，`Range` 扫描可以直接按序返回。`BuildBatchRecordRange` 用经典的 `PrefixEnd`（末字节 +1，全 `0xFF` 则退化成 `"\0"`）构造前缀区间（`oplog_batch_types.cpp:17-27, 140-152`）。

### `PromotionContext`（`include/ha/standby_controller.h:22-38`）

```cpp
struct PromotionContext {
    uint64_t applied_seq_id{0};
    std::vector<StandbyObjectEntry> objects;
    std::vector<StandbySegmentInfo> segments;
    WeightMetadataSnapshot weight_metadata;
    std::unique_ptr<StandbyMetadataStore> metadata_store;  // 快照模式走这条
    DurablePrefix applied_cursor;
    ViewVersionId producer_view_version{0};
    ReplicaID max_replica_id{0};
};
```

**只可移动、不可拷贝**（显式 `= delete` 拷贝构造/赋值）。它承载"Standby 的全部已验证状态"，所有权从 `HotStandbyService` 转移到 `MasterService`。

两条互斥的填充路径：

- **快照模式**（`metadata_store` 非空）：整个 `StandbyMetadataStore` 直接 move 过去，零拷贝，还带 `applied_cursor` 和 `producer_view_version`。
- **非快照模式**：`PromoteAndExportSnapshot()` 先把 `metadata_store_` 的内容**拷贝**进局部的 `StandbySnapshot`（`hot_standby_service.cpp:770-784`），再由 `standby_controller.cpp:369-376` 把 `objects` / `segments` / `weight_metadata` move 进 `PromotionContext`（注意这条路径不填 `applied_cursor` / `producer_view_version`）。

supervisor 靠 `promotion_ctx->metadata_store` 是否非空来决定调哪个 restore（`master_service_supervisor.cpp:759-768`）。

---

## 关键流程剖析

### 流程一：进程启动到进入 standby（`master.cpp` → supervisor）

**调用链**：
```
main()  mooncake-store/src/master.cpp:1674
  ├─ StartShutdownWatchdog(kHaShutdownDeadline)     :1679
  ├─ InstallShutdownSignalHandlers()                :1685
  ├─ MasterServiceSupervisor supervisor(config)     :1694
  └─ supervisor.Start()                             :1695
       ├─ BuildHABackendSpec(config_)      master_service_supervisor.cpp:1025
       ├─ CreateStandbyController(*spec, config_)   :1035
       ├─ MasterAdminServer admin_server(...)       :1043
       └─ RunSupervisorLoop(...)                    :1050
```

**分步说明**：

1. **看门狗必须在任何租约获取之前 arm**（`master.cpp:1675-1690`）。注释说得很直白：`without them a SIGTERM kills a leader mid-term`。顺序是有意义的——如果先拿租约再装处理器，中间窗口的 SIGTERM 会直接杀死一个已经持有租约的 leader，租约只能等 TTL 过期，故障切换被拖长。两个调用失败都 `return 1` 拒绝启动（不是降级运行）。

2. **信号处理器必须用 `sigaction` + `SA_ONSTACK`**（`shutdown_request.cpp:59-70`）。头文件注释解释了原因：`which the Go c-shared HA backends in the master process require`（`include/shutdown_request.h:13-15`）。etcd 和 k8s 后端本体是 Go 代码编译成 `c-shared` `.so`，Go runtime 对信号栈有自己的要求。处理器本身只做一件事：一次无锁 `exchange`，如果是第二次信号就 `_exit(128 + signo)`（`shutdown_request.cpp:31-37`）。**不打日志、不加锁、不分配内存**——这是 async-signal-safe 的硬约束。

3. **`BuildHABackendSpec`** 做三重校验（`master_service_supervisor.cpp:68-91`）：解析后端类型 → `ValidateHABackendAvailability` 检查编译期是否开启了对应 `STORE_USE_ETCD/REDIS/K8S_LEASE`（`ha_types.h:71-96`）→ 解析连接串。任何一步失败返回 `INVALID_PARAMS` 或 `UNAVAILABLE_IN_CURRENT_MODE`，这两者被 `IsFatalHABackendError` 判为致命（`:93-96`），不重试直接退出。

4. **`RunSupervisorLoop` 开局**（`:416-453`）：构造 `LeaderLabelReconciler`（只在 K8s + 有 pod identity 时启用，`:43-45`）→ `SetLeader(false)` → 状态置 `kStarting` → 注册 standby 状态回调（受 `accept_standby_runtime_updates` 原子门控，`:424-432`）→ **先检查一次 shutdown**，再 `EnterStandbyMode`。

   `EnterStandbyMode`（`:398-414`）打开 `accept_runtime_updates`、更新观察到的 leader、调 `standby_controller->StartStandby(leader_view)`。注意 `StartStandby` 失败**不会**让 supervisor 退出，只是记 warning 并把状态设成 `kStandby`——一个连不上 etcd 的 standby 应该继续重试，而不是自杀。

### 流程二：竞选 → 晋升 → 服务（supervisor 主循环，最核心）

**调用链**（`master_service_supervisor.cpp:455-1012`，一个 `while(true)` 的"任期循环"）：

```
while (true) {                                        :455
  coordinator = CreateLeaderCoordinator(spec)          :461
  while (!leadership_session) {                        :473   ← 内层竞选循环
    ReadCurrentView()                                  :479
    ├─ 有 leader → EnterStandbyMode + WaitForViewChange :550-563
    └─ 无 leader → TryAcquireLeadership()              :499
         ├─ ACQUIRED → leadership_session = ...        :520-525
         └─ CONTENDED → EnterStandbyMode + WaitForViewChange :530-547
  }
  term_lease = {.session = &*leadership_session}       :570
  release_term = [&]{...}                              :577   ← 租约交接闭包
  LeadershipRenewalGuard renewal(...)                  :595   ← 保活侧线程
  ClaimProducerViewForServing(...)                     :597   ← fence 旧 producer
  promotion_ctx = standby_controller->PromoteStandbyAndExport()  :615
  return_to_standby = [&]{...}                         :632   ← 保留游标
  if (renewal.Lost()) → return_to_standby + continue   :658
  WarmupLeadership(...)                                :671
  coro_rpc_server server(...)                          :695
  WrappedMasterService wrapped_master_service(...)     :714
  detail::ServingStateGate serving_state               :713
  SetBatchOpLogTerminalCallback(...)                   :743
  SetRuntimeState(kRecovering); RestoreFromBatchOpLogPromotion(...) :758-768
  if (renewal.Lost()) → release_service + return_to_standby        :793
  serve_preflight = RenewLeadership(...)               :808
  worker_pool + RegisterRpcService(server, ...)        :838-843
  StartLeadershipMonitor(..., on_leadership_lost)      :845
  ServingShutdownTrigger shutdown_trigger(...)         :879
  serving_state.RunIfActive([&]{ ec.emplace(server.async_start()) }) :891
  PublishServiceReady(...)                             :935
  serving_state.RunIfActive([&]{ ActivateServingState(...); 释放 handoff }) :938-949
  server_err = ec->get()                               :982   ← 阻塞在服务期
  // 服务结束后：worker_pool.reset() → StopLeadershipMonitor
  //   → StopBatchOpLogWriter → DeactivateServingState → release_term
}
```

**分步说明**（重点是每一步的门控语义）：

1. **每轮任期新建一个 coordinator**（`:461`）。这个决定很重要：`EtcdLeaderCoordinator` 的析构函数会调 `ShutdownKeepAliveThread()`，它会 **`RevokeLease`**（`etcd_leader_coordinator.cpp:155, 656-707`）。所以"函数返回"这个动作本身就有副作用。

2. **`shutdown_exit` 绝不 `return`**（`:437-444`）：
   ```cpp
   auto shutdown_exit = [&](std::string_view phase, const ShutdownLease& lease) -> int {
       LOG(INFO) << "Shutdown requested, finishing from phase=" << phase;
       admin_server.Stop();
       accept_standby_runtime_updates.store(false, ...);
       standby_controller->StopStandby();
       ExitAfterShutdown(FinishShutdown(lease));   // _exit()，不回卷栈
   };
   ```
   `:434-436` 的注释说明了原因：`Returning would destroy the per-iteration coordinator, and the etcd coordinator's destructor revokes a held lease.` 也就是说，优雅退出路径必须**绕过 C++ 栈回卷**，否则析构会把租约吊销掉，可能让一个还没准备好的节点抢到 leadership。`ExitAfterShutdown` 最终调 `_exit(status)`，**不跑任何析构**（`shutdown_request.cpp:104-110`）。

3. **`ShutdownLease` 与 `release_term` 的租约交接**（`:177-203, 570-590`）。这是本次 drain 提交（`#4745`）里最精妙的部分：
   ```cpp
   struct ShutdownLease {
       const LeadershipSession* session = nullptr;      // 仍持有，交给 TTL 过期
       std::optional<ErrorCode> prior_release{};        // 已经做过的普通释放的结果
   };
   ```
   `release_term` 闭包在释放**前后各检查一次** shutdown：
   - 释放**前**观察到 → `shutdown_exit(phase, term_lease)`，此时 `term_lease.session` 非空，`FinishShutdown` 打印"不释放，交给 TTL 过期"。
   - 释放**后**观察到 → `term_lease` 已被改写成 `{.prior_release = err}`，`FinishShutdown` 报告已完成的释放结果。

   语义就是：**shutdown 既不重复一次已经做过的释放，也不谎称"没持有租约"**。`:574-576` 的注释精确描述了这三态。

4. **`LeadershipRenewalGuard` 必须在 catch-up 之前启动**（`:592-595`）。注释记录了一个真实 bug：`etcd KeepAlive previously started only in the warmup phase, so a large Oplog replay expired the lease first`。这个侧线程每 `kRenewCheckInterval`（1s）调一次 `RenewLeadership`，失败就把 `lost_` 置 true（`:96-160`）。

   而 etcd 侧的 `RenewLeadership` 实际上是个**适配器**：etcd wrapper 只暴露阻塞式的 `KeepAlive` 循环，没有一次性 renew RPC（`:331-333` 注释）。所以第一次调用会 spawn 一个 keepalive 线程并 `WaitKeepAliveReady`（1s 超时，`:396-403`），后续调用只是检查那个线程还活着（`:343-356`）。keepalive 线程结束时会在锁内取出 monitor 回调并触发 `on_leadership_lost`（`:363-392`）。

5. **`ClaimProducerViewForServing` 是防脑裂的 fence**（`:374-396`）。它只在 `enable_oplog && !cluster_id.empty()` 时生效，要求后端支持 OpLog 且 `view_version != 0`，然后调 `OpLogBatchStorage::ClaimProducerView(view_version)`。

   `ClaimProducerView`（`oplog_batch_storage.cpp:191-258`）是**单调**的：
   ```cpp
   if (current_view > producer_view_version) → return ETCD_TRANSACTION_FAIL;  // 被 fence
   if (current_view == producer_view_version) → return OK;                    // 幂等
   // 否则 CAS 写入（key 不存在用 kKeyNotExists，存在用 kValueEquals）
   ```
   一个 view_version 更小的旧 Primary（比如网络分区后苏醒的那个）在这里就被永久挡住：它的 `ClaimProducerView` 返回 `ETCD_TRANSACTION_FAIL`，supervisor 走 `EnterStandbyMode` + `release_term` + `continue`（`:602-612`）。即使它侥幸通过，后续每次 `WriteBatchAndAdvancePrefix` 的事务里也带着 `producer_view` 的 compare（`oplog_batch_storage.cpp:377-382`），双保险。

   `:387-389` 的注释特别说明：fence 走的是 `spec` 选定的后端而不是硬编码 etcd，因为 Redis 也实现了 `ClaimProducerView` 需要的事务 CAS，所以旧 Redis primary 同样必须被 fence。

6. **`PromoteStandbyAndExport`**（`:615`，实现在 `standby_controller.cpp:297-377`）：先检查 `standby_running_`，然后按 `IsBatchOpLogSnapshotMode()` 分岔。快照模式走 `PromoteAndDetachBatchOpLogStore()`，把整个 `StandbyMetadataStore` move 进 handoff；非快照模式走 `PromoteAndExportSnapshot()` 值拷贝。

   **失败路径都会 `MarkAppliedCursorForRestart()`**（`standby_controller.cpp:316, 352`），这是"fail-closed 但不丢进度"的关键，下面流程四细讲。

7. **Restore 是服务门控**（`:754-788`）。`:754-757` 的注释：`Restore is the serving gate: do not register or expose a candidate service until the complete promotion context has been applied.` 状态置 `kRecovering`，调 `RestoreFromBatchOpLogPromotion`（有 `metadata_store`）或 `RestoreFromStandby`（否则）。失败 → `release_service()` → `DeactivateServingState` → `return_to_standby` → `release_term` → `continue`。**绝不带着未验证的元数据对外服务**。

   `RestoreFromBatchOpLogPromotion`（`master_service.cpp:3743-3754`）会校验 `chunk_object_count != 0` 和 cursor 的零字段一致性，然后转 `RestoreFromStandbyState`；后者还要求 `ValidateReplicaIds` 得到的 `max_live_id` 与 handoff 里的 `expected_max_replica_id` **完全相等**（`master_service.cpp:3792-3797`）。

8. **Restore 之后再查一次 `renewal.Lost()`**（`:790-803`）。注释：`Restore can outlast catch-up.` 恢复一个几百万 key 的影子存储可能比追平 OpLog 还慢，所以必须重查侧线程守卫，否则一个在 restore 中途丢掉的租约会继续"恢复中"。

9. **`serve_preflight` 复验租约**（`:805-836`）。`RenewLeadership` 再调一次，失败或过期都 `release_service()` + `return_to_standby` + `release_term` + `continue`。这是暴露 RPC **之前**的最后一道租约检查。

10. **注册顺序很讲究**（`:838-895`）：`RpcWorkerPool` → `RegisterRpcService` → `StartLeadershipMonitor` → `ServingShutdownTrigger` → `serving_state.RunIfActive([&]{ ec.emplace(server.async_start()) })`。

    `async_start` 被包在 `RunIfActive` 里（`:891-895`），并且内部**再查一次** `ShutdownObserved`。这是为了处理一个竞态：monitor 的 `on_leadership_lost` 回调可能在 `async_start` 之前就已经 `RequestShutdown` 了。如果 `ec` 没有值，说明被门控挡下了，走 `serve_start` 退出路径（`:896-909`）。

11. **`PublishServiceReady` 分离"当选"与"可服务"**（`:929-956`）。`:931-933` 的注释：这个后端调用可能花好几秒，所以**故意放在 transition gate 外面**，这样 leadership loss 能及时停掉 listener。成功后再进 `RunIfActive` 做 `ActivateServingState`，并**释放 handoff**（`:944-948`）：
    ```cpp
    promotion_ctx->metadata_store.reset();
    promotion_ctx->objects.clear();
    promotion_ctx->segments.clear();
    ```
    注释说明动机：`Serving owns the restored index now; drop the handoff so the standby copy is not held for the whole term.`——否则整个任期都会多持有一份元数据副本。

    **只有 etcd 后端 override 了 `PublishServiceReady`**（全仓 grep 确认，见 `leader_coordinator.h:35-38` 的默认实现返回 `OK`）。etcd 的机制是：`TryAcquireLeadership` 先写一个 `__mooncake_service_warming__:<owner_token>` 占位值（`etcd_leader_coordinator.cpp:227-231`），`ReadCurrentView` 看到这个前缀就返回 `std::nullopt`（`:200-202`）——即"有人在竞选，但还没有可路由的端点"。`PublishServiceReady` 再用 `kValueEquals` CAS 把占位值换成真实地址（`:283-317`），并且**幂等**：CAS 失败时重读，如果已经是自己的地址+版本就当成功（`:308-315`，处理"RPC 客户端超时但 etcd 已写成功"）。

    Redis 后端没有这个 warming 阶段，它在 acquire 的 Lua 脚本里就直接写了 `leader_address`（`redis_leader_coordinator.cpp:357-363`）。

    **`PublishServiceReady` 失败时的收尾**（`master_service_supervisor.cpp:958-980`）：先 `StopLeadershipMonitor`，再阻塞在 `ec->get()` 等 listener 真正停下（publish 失败前已经通过 `RequestShutdown` 调过 `server.stop()`），然后 `shutdown_trigger.Stop()`；随后按致命/可重试分流——可重试就 `release_service()` + `return_to_standby()` + `release_term("readiness_failure")` + `continue`。注意这里**先停 listener 再回 standby**，避免一个"选主成功但 endpoint 发布失败"的节点带着还在 accept 的端口回到 standby 状态。

12. **服务期结束后的拆除顺序**（`:982-1011`）：`ec->get()` 返回 → `shutdown_trigger.Stop()` → **`worker_pool.reset()`**（`:984-986` 注释：`Finish the calls the workers still hold before the term's OpLog writer and leadership go`）→ `StopLeadershipMonitor` → `StopBatchOpLogWriter` → `DeactivateServingState` → `release_term("serving")` → `ReadCurrentView` → `release_service` → 回到 standby。

    顺序不能乱：worker pool 里可能还有 in-flight 的 handler 在往 OpLog writer 提交，必须先排干 worker，再停 writer，最后才放租约。

### 流程三：Primary 写一条 OpLog（有序批写）

**调用链**（以 `PUT_END` 为例）：
```
MasterService::AppendOpLogWithDurableFinalize(...)   master_service.cpp:13630
  ├─ ReserveBatchOpLogSlot()                          :13648
  │    └─ ordered_oplog_writer_->Reserve()   ordered_oplog_writer.cpp:267 → ReserveBatch(1) :272
  └─ AppendReservedOpLogWithDurableFinalize(...)      :13692
       └─ ordered_oplog_writer_->Commit(reservation, entry, callback)  :13711
            ├─ ValidateOpLogBatchEntry                 ordered_oplog_writer.cpp:320
            ├─ entry.sequence_id = next_sequence_id++  :329
            ├─ committed_entries.push_back(...)        :336
            └─ SealCommittedEntriesIfIdle()            :341 → :82
                 （committed_entries → ready_entries，batch_busy = true）

writer_thread（Start() 里 spawn）                       :515
  ├─ 取走 ready_entries                                :528
  ├─ batch.batch_id = expected_prefix.batch_id + 1     :543
  └─ while(true):
       err = write_batch(batch, expected_prefix)       :582
         └─ OpLogBatchStorage::WriteBatchAndAdvancePrefix   oplog_batch_storage.cpp:339
              ├─ ValidateOpLogBatchRecordShape         :349
              ├─ 连续性断言 batch_id/first_seq          :357-368
              ├─ txn.compares += producer_view CAS     :377-382   ← fence
              ├─ txn.compares += durable_prefix CAS    :383-387   ← 乐观并发
              ├─ txn.puts += batch record              :388-389
              ├─ txn.puts += new durable_prefix        :390-394
              └─ backend_.Txn(txn)                     :395
       ├─ OK → durable_prefix 前进、TakeCoveredWaiters、
       │        callback_entries.push_back(...)        :605-641
       ├─ 非可重试 → EnterTerminalState(kFenced/…)      :644-652
       └─ 可重试 → accepting=false、指数退避 1ms→1s      :653-679

callback_thread（Start() 里 spawn）                     :479
  └─ callback_entry.callback(callback_entry.entry)     :511   ← DurableFinalizeCallback
```

**分步说明**：

1. **两段式提交：Reserve → Commit**。`Reserve()` 先占槽位（`open_waiting_slots += entry_count`，`ordered_oplog_writer.cpp:289-293`），准入上限是 `max_entries_per_batch`（默认 1024，来自 `--oplog_batch_max_entries`，`master.cpp:396-398`）。超限返回 `TASK_PENDING_LIMIT_EXCEEDED`（`:284-288`）。

   **为什么需要 Reservation？** 因为序号是在 `Commit` 里分配的（`:329`）。如果直接在 Commit 里既分配序号又做慢操作，两个并发 handler 就可能以与序号相反的顺序入队。Reservation 把"我打算提交 N 条"这件事先记账，配合 `Reservation` 的 RAII——析构时自动 `Abort` 归还槽位（`:239-243`）——保证任何提前返回的路径都不会漏槽。

2. **`SealCommittedEntriesIfIdle` 保证组原子性**（`:82-108`）。它从 `committed_entries` 取最多 `max_entries_per_batch` 条封成一批，但**如果切分点落在一个 commit group 中间，就扩展到组尾**：
   ```cpp
   while (count < committed_entries.size() &&
          committed_entries[count].group != 0 &&
          committed_entries[count].group == committed_entries[count-1].group) ++count;
   ```
   `:88-90` 的注释：`The cap is a batching efficiency knob; a group's atomicity is not.` group 由 `CommitBatch` 分配（`:383`），用于"修复类"的多条目集合——`AppendOpLogBatchWithDurableFinalize`（`master_service.cpp:13658-13690`）。头文件说明动机（`ordered_oplog_writer.h:90-95`）：拆开会让 durable prefix 捕获到一个真子集。

3. **`WriteBatchAndAdvancePrefix` 的双重 CAS**（`oplog_batch_storage.cpp:376-395`）。事务里放两个 compare：`producer_view == 我的 view_version`（fence）和 `durable_prefix == expected_prefix`（乐观并发）。然后两个 put：批次记录 + 新前缀。**批次记录和前缀在同一个事务里原子推进**，这是 standby 能信任前缀的根本原因。

4. **模糊失败的幂等重解**（`oplog_batch_storage.cpp:396-435`）。这是很容易看漏的一段：
   - 第一步（`:396-406`）：如果事务失败，重读 durable_prefix，**如果解码出来等于 expected_prefix 但原始字节不同**（说明存储里的编码形式变了，比如零填充差异），就用原始字节替换 compare 值重试一次。
   - 第二步（`:407-435`）：如果还是失败或错误码模糊（`ETCD_OPERATION_ERROR` 被 `IsAmbiguousTxnError` 判为"可能已经写成功了"，`:69-71`），就重读 producer_view 和 durable_prefix，再 `ReadBatch` 出来用 `SameBatchRecord` 逐字段比对（`:20-37`）。**如果存储里的内容和我想写的完全一致，返回 `OK`**——因为客户端 RPC 超时但服务端已成功的情况下，重试不应该报 fence。

   只有当 producer_view 对不上时才返回 `ETCD_TRANSACTION_FAIL`（`:411-422`），这个错误码在 writer 侧被翻译成 `kFenced`（`ordered_oplog_writer.cpp:644-649`）。

5. **终态如何触发下台**。`EnterTerminalState`（`:130-167`）设 `accepting = false`、记 `terminal_reason`、swap 出所有 durable waiters 并用错误码完成它们，然后**在锁外**调 terminal callback（`:164-166`）。supervisor 注册的那个 callback（`master_service_supervisor.cpp:743-752`）会 `serving_state.RequestShutdown([&]{ ... server.stop(); })`，从而让 `ec->get()` 返回，走服务期结束的拆除路径。

   **`ServingStateGate` 是这里的互斥核心**（`master_service_supervisor.h:17-48`）：`RunIfActive` 和 `RequestShutdown` 都在 `transition_mutex_` 下执行，`shutdown_requested_` 用 `exchange` 保证 callback 只跑一次（`first_request`）。这样"OpLog writer 终态"、"leadership loss"、"进程 shutdown"三个来源的竞争被收敛成一次 `server.stop()`。

6. **一个重要的实现事实：`retry_timeout` 在生产路径上从未被设置**。`OrderedOpLogWriterConfig::retry_timeout` 默认 `0`（`ordered_oplog_writer.h:20-24`），而 `InitializeBatchOpLogWriter` 只设了 `max_entries_per_batch` 和 `initial_durable_prefix`（`master_service.cpp:13558-13560`）。writer 里 `retry_deadline` 只在 `config.retry_timeout.count() > 0` 时才被赋值（`ordered_oplog_writer.cpp:666-670`）。

   全仓搜索确认：只有 `mooncake-store/tests/ha/oplog/ordered_oplog_writer_test.cpp:265-269, 1297, 1343` 设置了它。

   **后果**：生产环境下 `kRetryTimeout` 终态**不可达**。etcd 持续不可用时，writer 会以 1ms→1s 的指数退避**无限重试**可重试错误（`ETCD_OPERATION_ERROR / ETCD_CTX_CANCELLED / PERSISTENT_FAIL`，`:202-206`），期间 `accepting = false`（`:663`），所以新的 `Reserve()` 返回 `UNAVAILABLE_IN_CURRENT_STATUS`（`:278-280`）——元数据写操作会失败，但 writer **不会**触发 terminal callback 把服务停掉。

   真正兜住这个场景的是另一条路径：etcd 挂了 → `KeepAlive` 失败 → keepalive 线程结束 → `on_leadership_lost` → `server.stop()`。这两条路径是独立的，读代码时不要以为 OpLog 写失败会自动下台。

7. **两种可见性语义**。`master_service.h:2394-2399` 声明了两个 API，区别在于调用方是否等待持久化：
   - `AppendOpLogWithDurableFinalize`：返回 `OpLogEntry`（含已分配的 `sequence_id`），带一个 `DurableFinalizeCallback`，在批次真正 durable 后由 callback_thread 触发。用于"必须知道已持久才能推进的状态机"，比如 `REMOVE`（`master_service.cpp:13725-13735`）和副本修复（`:10468-10498`）。
   - `AppendOpLogVisibleBeforeDurable`：只返回 `sequence_id`，callback 传 `nullptr`（`:13622-13623`）。用于 segment 类操作——`PersistSegmentOpForHAOrEnqueue`（`:13744-13755`）失败时**只打 warning 就继续**，因为 segment mount/unmount 可以由客户端心跳重新驱动。

   还有一个协程化的等待接口 `AwaitDurable(sequence)`（`ordered_oplog_writer.cpp:422-440`），返回 `async_simple::Future<ErrorCode>`，用 `std::multimap<seq, Promise>` 存 waiter，`TakeCoveredWaiters` 用 `extract` 摘节点避免在推进前缀时分配内存（`:112-120`）。生产调用点只有 `master_store_backend.cpp:35`。

### 流程四：Standby 追平、快照与晋升

#### 4.1 bootstrap（`hot_standby_service.cpp:64-307`）

```
Start(primary_address, oplog_endpoints, cluster_id)     :64
  ├─ 收割上一轮遗留的 joinable 线程                       :75-84
  ├─ 若 !preserve_baseline：清空所有快照游标状态            :90-100
  ├─ state_machine_.ProcessEvent(START)                  :106
  ├─ CreateHaKvBackend(spec)                             :129
  ├─ ProcessEvent(CONNECTED)                             :146
  ├─ PrepareBootstrapBaselineLocked(baseline_seq_id)     :149 → :171
  │    ├─ 丢弃"有元数据但无可恢复序号边界"的本地状态         :179-184
  │    ├─ 新建 OpLogApplier(metadata_store_)             :186
  │    ├─ 快照模式 + preserve_promotion_baseline_ → 复用游标 :189-210
  │    ├─ 快照模式 → LoadBatchOpLogSnapshotBaselineLocked  :212 → :240
  │    ├─ 有可恢复本地状态 → warm start，applier.Recover(seq) :214-219
  │    └─ 否则 → LoadSnapshotBaselineLocked（清空，从 0 开始）:221 → :232
  └─ StartOplogFollowingLocked(baseline_seq_id)          :160 → :273
       ├─ new OpLogBatchStandbyReader(...)               :283
       ├─ SetBaselineCursor(*batch_snapshot_baseline_)   :287
       ├─ ProcessEvent(SYNC_COMPLETE)  → WATCHING        :295
       └─ replication_thread_ = std::thread(ReplicationLoop) :297
```

`LoadBatchOpLogSnapshotBaselineLocked`（`:240-271`）的关键设计是**先在临时对象上恢复，成功了才替换成员**：新建 `temporary_metadata` + `temporary_applier` + `temporary_registry`，调 `provider->RestoreBaseline(...)`，失败直接返回错误码而**不碰现有状态**（`:243-251`）。成功才 `metadata_store_ = std::move(temporary_metadata)`（`:258`）。

`RestoreBaseline`（`batch_oplog_snapshot_provider.cpp:407-500`）的三级降级：先读 `compaction_floor`（`:420-433`，用它抬高 `minimum_snapshot_batch`），然后依次尝试 `latest` 和 `fallback` 两个指针（`:435-487`），每次失败都 `metadata.Clear()` + `registry.Clear()` + `applier->Recover(0)` 回滚（`:478-483`）。区分 `kInfrastructure`（基础设施错误，直接返回）和内容错误（继续试下一个指针）。两个指针都不行且 `minimum_snapshot_batch != 0` → `INCOMPLETE_OPLOG_CATCH_UP`（`:489-491`）；否则退化成 `RestoreCompleteOpLog` 全量回放（`:492-499`）。

#### 4.2 稳态跟随（`ReplicationLoop`，`:1165-1326`）

每轮：
1. 不 connected → 睡 1s 重试（`:1189-1192`）。
2. `recovering_` → `RebootstrapBatchOpLog(recovery_floor)`（`:1194-1206`）。
3. 否则 `batch_standby_reader_->PollOnce()`（`:1211`），更新指标、更新 `last_applied_batch_snapshot_prefix_`、`HandleSnapshotCaptureRequest(result)`（`:1243`）、推进 `primary_seq_id_` 和 `applied_seq_id_`。
4. 按 disposition 分流（`:1256-1297`）：
   - `REBOOTSTRAP_REQUIRED` → 记 `recovery_floor`，置 `recovering_ = true`，`continue`（**不计入重试预算**）。
   - `RETRYABLE` → 指数退避，超过 `batch_oplog_retry_timeout_sec`（默认 180s）就 `FATAL_ERROR` 退出循环。
   - `FATAL` → 直接 `FATAL_ERROR` 退出。
5. 睡 `oplog_poll_interval_ms`（默认 1000ms）。

**`PollOnce` 的一致性判定**（`oplog_batch_standby_reader.cpp:52-237`）是 standby 侧最精密的逻辑：

- **错误分类只认两种可重试**：`IsRetryableBackendError` 仅把 `ETCD_OPERATION_ERROR` / `ETCD_CTX_CANCELLED` 判为 `RETRYABLE`（`:13-16`），其余后端错误（含 key 不存在且此前见过 batch 格式）都是 `FATAL`。所以"etcd 临时抖动"与"数据布局异常"在 reader 层被严格分开。

- **先读 floor**（`:56-67`）：如果 `last_applied_batch_id_ < floor`，说明我需要的历史已经被修剪掉了 → `REBOOTSTRAP_REQUIRED`。
- **前缀倒退检测**（`:112-121`）：四种非法组合（batch_id 变小、last_seq 变小、同 batch_id 但 last_seq 变了、batch_id 变大但 last_seq 没变）都判 `INCOMPLETE_OPLOG_CATCH_UP` + FATAL。这是防止 etcd 数据被外部篡改或回滚。
- **零前缀特例**（`:123-131`）：`batch_id == 0` 时 `last_seq` 必须也是 0，否则 FATAL。
- **已证游标的快路径**（`:133-142`）：`floor == prefix.batch_id == last_applied_batch_id_` 且序号对得上时，**跳过读批次**直接推进——因为那个批次可能已经被修剪掉了，读不到是正常的。
- **批次连续性校验**（`:173-194`）：`batch.batch_id` 必须等于 `last_applied_batch_id_ + 1`；若 `last_applied_batch_id_ == 0` 且 `require_complete_history_` 为真，则必须从 `batch_id == 1` 开始。`expected_first_seq` 同样严格衔接。
- **条目级校验**（`:195-214`）：超过 `prefix.last_seq` 的条目 break（前缀之后的还没 durable）；比 expected 旧的 skip（重试可能重放）；比 expected 新的 → FATAL（有洞）；`ApplyOpLogEntry` 失败 → FATAL。
- **收尾判定**（`:222-235`）：如果没有更多分页且 `expected <= prefix.last_seq`，说明没追平 → FATAL；追平了才推进 `last_applied_durable_prefix_`。
- **FATAL 之后的重分类**（`:69-88`）：`PollOnce` 在返回 FATAL 前会**再读一次 floor**。因为修剪可能在初次读 floor 之后推进了。如果新 floor 说明"我落后的部分已被修剪"，就改判 `REBOOTSTRAP_REQUIRED`；如果 `current_floor == last_applied_batch_id_`，就用新 floor 重跑一次 `PollBatches`。这个二次判定避免了把"正常修剪"误报成"数据损坏"。

`require_complete_history_` 只在 `SetBaselineCursor` 时根据 `cursor.batch_id == 0` 设置（`:42`）——从空基线开始的 standby 必须看到完整历史（从 batch 1 开始）；从快照基线开始的则允许中间有已被修剪的洞。

#### 4.3 快照捕获（coordinator ↔ standby 的握手）

这是本子系统里**并发设计最微妙**的一段，因为它让快照协调器线程和复制循环线程互相阻塞。

```
BatchOpLogSnapshotCoordinator::RunAttempt()      batch_oplog_snapshot_coordinator.cpp:311
  ├─ ReadLatestBatchId + GetLastAppliedBatchOpLogSnapshotPrefix  :319-329
  │    （没有新批次 → Skip::NoNewBatch，返回 OK）
  ├─ lease_factory_() + lease->Acquire()          :352-370   ← SnapshotMaintenanceLease 互斥
  ├─ 重读两侧（fence 之后）                        :403-415
  ├─ standby_.BeginBatchOpLogSnapshotCapture()    :417
  │    └─ hot_standby_service.cpp:938-960
  │         ├─ 置 state->requested = true, ++generation
  │         ├─ service_lock.unlock(); replication_loop_cv_.notify_all()   ← 唤醒复制循环
  │         └─ state->cv.wait(!requested)          ← 阻塞等复制循环处理
  │
  │    【复制循环侧】HandleSnapshotCaptureRequest(result)  :991-1037
  │         ├─ 一致性判定：result.error==OK && durable_prefix_present
  │         │   && applied_prefix == result.durable_prefix
  │         │   && expected-1 == applied_prefix->last_seq           :1006-1011
  │         ├─ 构造 BatchOpLogSnapshotCapture（含 BeginSnapshotTraversal 游标） :1013-1019
  │         ├─ state->active = true; state->cv.notify_all()          ← 放行协调器
  │         └─ state->cv.wait(!active || !replication_loop_running_) :1025-1028
  │              ↑↑↑ 复制循环在这里阻塞，直到快照拷贝完成 ↑↑↑
  │
  ├─ BatchOpLogSnapshotWriter::Write(...)          :460-463
  │    └─ 内部反复调 standby_.CopyNextBatchOpLogSnapshotChunk(count, capture, out)
  │         └─ hot_standby_service.cpp:962-973（校验 generation + lease_state_）
  ├─ （成功路径不显式调用 End；capture 离开作用域时析构 → Release()，
  │    见 capture.h:45, 80-91，清 active 并唤醒复制循环）
  ├─ BatchOpLogSnapshotPublisher::Publish(...)     :491-500
  ├─ BatchOpLogSnapshotGc::Run(...)                :523-537   ← best-effort
  └─ BatchOpLogPruningCoordinator::Run(...)        :540-551   ← best-effort
```

**关键事实：快照捕获期间 OpLog 应用是暂停的。** `HandleSnapshotCaptureRequest` 在复制循环线程里执行（`:1243` 调用点），它在 `:1025-1028` 阻塞等待 `state->active` 变 false。成功路径上 coordinator 并不显式调用 `EndBatchOpLogSnapshotCapture`：`active` 由 `BatchOpLogSnapshotCapture` 的析构函数调用 `Release()` 清零（`include/ha/snapshot/batch_oplog/capture.h:45, 80-91`）。`EndBatchOpLogSnapshotCapture`（`hot_standby_service.cpp:975-989`）只在 promotion 取消路径（coordinator `:441`）显式使用，它除了 `Release()` 还会触发 snapshot-released 生命周期回调。所以从 capture 开始到 chunk 拷完，复制循环不推进。代码用 `capture_pause_us` 指标度量这段停顿（`:1029-1036`）。

这是个明确的取舍：**用可度量的复制停顿换取"不需要对影子存储做 COW / 版本化"**。`SnapshotCursor` 的注释也印证了这点：`Mutations must remain frozen for the lifetime of the returned cursor`（`standby_metadata_store.h:90`）。

**取消路径**必须处处检查，因为晋升/停止随时可能插进来：capture 前（`:388-401`）、capture 后（`:433-445`）、write 后 publish 前（`:474-489`，此时还要 `DeleteObjectsWithPrefix` 清掉半成品制品）。`CancelSnapshotCapture`（`hot_standby_service.cpp:1039-1051`）会把 `requested` 和 `active` 都置 false 并 notify，让阻塞的复制循环脱身。

`generation` 字段是防陈旧 capture 的：`CopyNextBatchOpLogSnapshotChunk` 会校验 `capture.generation_ == state->generation`（`:967-971`），如果 capture 被取消后又发起了新一轮，旧 capture 的 chunk 拷贝会直接失败返回 false。

**维护租约的活性检查**：`SnapshotMaintenanceLease::IsHeld()` 不是读本地标志，而是每次都向 etcd 查询 maintenance session 是否仍存活（`snapshot_maintenance_lease.cpp:73-88`）；`Acquire()` 走 `EtcdHelper::AcquireMaintenanceSession`（`:50-69`），拿到的 `lease_id` 同时充当 `owner_token` 和快照 ID 后缀（`:63`，`batch_oplog_snapshot_coordinator.cpp:447-449`）。因此 publisher/pruning 事务里的 value + create_revision 双 compare 提供了 etcd 侧的最终 fence：一旦 maintenance key 的持有者或创建修订号变了，事务就会失败（这正是 publisher 注释里说的 "the compare remains the final etcd-side fence"，`batch_oplog_snapshot_publisher.cpp:113-118`）。这也是 snapshot 路径事实上绑定 etcd 的原因之一（见上文"三个独立开关"）。

#### 4.4 修剪协议（`batch_oplog_pruning_coordinator.cpp`）

顺序是**先发布 floor，再删批次**，而且要求两个独立验证过的快照：

1. 读 `latest` 和 `fallback` 两个指针（`:112-121`）。**首次发布没有 fallback → `Skip::NoFallback`，直接成功返回，不修剪**（`:117-121`）。
2. 解码两个 descriptor，`candidate = fallback_descriptor->last_included_batch_id`。如果 `latest.snapshot_id == fallback.snapshot_id` 或 `latest.batch_id <= candidate` → `Skip::InvalidPair`（`:123-135`）。
3. 读旧 floor，`candidate < floor` → `Skip::FloorAhead`（`:139-158`）。
4. **`ValidateSnapshot` 两个快照都验一遍**（`:160-164`）：descriptor key 内容要和指针字节完全一致、manifest 大小和 CRC32C 要对、manifest 里的每个 key 要等于按规则重建的 key、每个 chunk 用 `InspectObject` 查大小和 CRC（`:19-28`）。任何一项不过 → `INTERNAL_ERROR`。
5. **一个五路 compare 的事务**发布 floor（`:167-186`）：maintenance lease 的 value + create_revision、`latest` 的 value、`fallback` 的 value、floor 的 value（或 not-exists）。这保证只有在"我仍持有维护租约且两个指针都没被别人改过"的前提下 floor 才推进。
6. **floor 事务成功后才 `DeleteBatchesThrough(candidate)`**（`:193-200`）。`:193-196` 的注释：`Even if the lease is lost now, this range has already been advertised to readers. A crash or failed delete leaves the floor in place; a later successful publication can safely repeat this bounded deletion.`

`DeleteBatchesThrough`（`oplog_batch_storage.cpp:506-515`）本身很朴素——`DeleteRange(BuildBatchRecordKey(cluster,0), suffix.begin_key)`，头文件明确它**不选择也不校验 cutoff、不重试后端错误**（`oplog_batch_storage.h:31-33`），所有判断都在 pruning coordinator 里。

**运维上不可逆**：一旦 floor 推进、批次被删，落后于 floor 的 standby 在 `PollOnce` 里会因为读不到历史批次而得到 `REBOOTSTRAP_REQUIRED`；随后 `RebootstrapBatchOpLog` 把 floor 作为 `minimum_snapshot_batch` 传给 `RestoreBaseline`（`hot_standby_service.cpp:1096-1099`），而后者在 `minimum_snapshot_batch != 0` 时拒绝降级到全量回放（`batch_oplog_snapshot_provider.cpp:489-491`）——`latest`/`fallback` 都不可用时只能失败。部署文档在 `docs/.../mooncake-store-deployment-guide.md:371` 明确要求：开始修剪后，回滚必须使用理解 batch snapshot 与 floor 协议的二进制。

#### 4.5 晋升与"fail-closed 但不丢进度"

`PromoteAndDetachBatchOpLogStore`（`hot_standby_service.cpp:805-876`）的严格顺序：

1. 前置检查：provider/following/reader 都在 + `IsReadyForPromotion()`（必须 `WATCHING`）。
2. `ProcessEvent(PROMOTE)` → `PROMOTING`。
3. `PreparePromotionLocked(current_applied_seq_id, kBoundedNoProgress)`（`:703-723`）：
   - `NotifySnapshotPromotion()` → 通过生命周期回调把协调器置为 `promotion_requested_`（`batch_oplog_snapshot_coordinator.cpp:124-137`）：取消并等待进行中的 capture（避免晋升被复制线程的停顿拖住），已写好的候选不再清理、继续走 publish（`stop_before_publish` 要求 `stop_requested_ && !promotion_requested_`），GC/pruning 的 `cancelled` 谓词则立即返回。
   - `StopReplicationLoop()` → **join 复制线程**，此后没有并发应用。
   - `recovering_` 为真 → `PROMOTION_FAILED`（不能在重 bootstrap 中途晋升）。
   - `FinalCatchUpForPromotionLocked` → `FinalCatchUpBatchRecordsLocked`（`:525-600`）：循环 `PollOnce` 直到 `GetLocalLastAppliedSequenceIdLocked() >= result.durable_prefix.last_seq`。
     - **缺失前缀只在本地序号为 0 时接受**（`:587-591`），否则 `INCOMPLETE_OPLOG_CATCH_UP`。这正是文档 `:418` 说的 fail-closed。
     - 两种超时策略（`PromotionCatchUpPolicy`）：`kLegacyTotalDeadline`（总时限 30s，`:540-541`）和 `kBoundedNoProgress`（无进展超时 = `batch_oplog_retry_timeout_sec`，`:538-539`）。detach 路径用后者——只要还在进展就继续等。
4. **游标与序号必须对齐**（`:828-842`）：`applied_cursor->last_seq != latest_applied_seq_id` → `PROMOTION_FAILED` + `INCOMPLETE_OPLOG_CATCH_UP`。这一步排除了"条目应用了但游标没推进"或反之的中间态。
5. `ValidateReplicaIds(max_replica_id)`（`:844-848`）：副本 ID 必须合法（非 0 等），得到的 max 会带进 handoff 供 restore 时**双向校验**。
6. 读当前 `producer_view`（`:850-855`）：优先用 etcd 里的实时值，读不到才退回快照里的 `batch_snapshot_producer_view_version_`。
7. `CompletePromotionLocked()` → `PROMOTED`。
8. **move 出 handoff，然后 reset reader 和 applier**（`:863-871`）：`metadata_store = std::move(metadata_store_)`，之后 `batch_standby_reader_.reset(); oplog_applier_.reset();`。顺序是必要的——reader 持有 applier 的引用，applier 持有 metadata_store 的裸指针。
9. `lock.unlock(); Stop();`（`:873-874`）。

**失败后如何不丢进度**：三条互补的机制。

- `MarkAppliedCursorForRestart()`（`:602-624`）：把当前 reader 的游标（或快照基线）存进 `batch_snapshot_baseline_`，连同 segment registry 存进 `preserved_segments_`，置 `preserve_promotion_baseline_ = true`。**但有严格前提**：`cursor->last_seq == applied_seq && applied_seq != 0`，否则直接返回不保存（`:612-614`）——宁可从头 bootstrap 也不保存一个可疑游标。
- `PreservePromotionBaseline(...)`（`:626-655`）：supervisor 的 `return_to_standby` 在 handoff 仍持有 `metadata_store` 时调用它（经 `RestorePromotionBaseline`，`standby_controller.cpp:403-410`），把**已经 move 出去的** metadata_store 装回 standby，并重装游标、producer_view、segments。**只允许在 `STOPPED` 或 `FAILED` 状态下调用**（`:631-636`），否则 `UNAVAILABLE_IN_CURRENT_STATUS`。
- 下次 `Start()` 时 `preserve_promotion_baseline_` 为真就**不清空**快照状态（`:88-100`），`PrepareBootstrapBaselineLocked` 走 `:189-210` 分支：`applier->Recover(preserved_seq)` + `LoadSegmentRegistry(preserved_segments_)`，直接从该游标续传。

supervisor 侧对应的调用点在 `return_to_standby`（`master_service_supervisor.cpp:632-656`）：
```cpp
if (promotion_ctx->metadata_store && promotion_ctx->applied_cursor.last_seq != 0) {
    auto restore_err = standby_controller->RestorePromotionBaseline(std::move(*promotion_ctx));
    ...  // 失败只 LOG(ERROR) "Failed to preserve catch-up cursor"，不阻断
}
```
`:999-1003` 还处理了另一个场景：如果 listener 在 `ActivateServingState` 之前就停了（handoff 还没被释放），也要走 `return_to_standby` 保住游标。

### 流程五：优雅退出（`#4745` drain）

```
SIGTERM/SIGINT
  └─ HandleShutdownSignal(signo)                shutdown_request.cpp:31
       └─ g_shutdown_signal.exchange(signo)
            ├─ 首次 → 返回 0，继续
            └─ 非首次 → _exit(128 + signo)      ← 立即退出，不排水

RunWatchdog(deadline)（detached 线程，master.cpp:1679 arm，deadline=15s）
  ├─ while (!ShutdownRequested() && !ShutdownLatched()) sleep(50ms)   :42-44
  ├─ expires_at = now + deadline                :45   ← 只算一次，永不重置
  ├─ 睡到 expires_at（分段睡，每段 ≤50ms）        :46-51
  └─ _exit(EXIT_FAILURE)                        :52

supervisor 侧（三个并发的观察点）
  ├─ RunSupervisorLoop 每个阶段前后 ShutdownObserved(...)   :449,456,474,480,...
  ├─ ServingShutdownTrigger 侧线程（100ms 轮询）            :207-258, 879-887
  │    └─ serving_state.RequestShutdown([&]{ ...; server.stop(); })
  └─ shutdown_exit(phase, lease) → ExitAfterShutdown(...)   :437-444
       └─ _exit(status)，不跑析构                            shutdown_request.cpp:104-110
```

**`ShutdownObserved` 的 latch 语义**（`master_service_supervisor.cpp:162-175`）：
```cpp
bool ShutdownObserved(const std::function<bool()>& shutdown_requested) {
    if (ShutdownLatched()) return true;
    if (!shutdown_requested()) return false;
    LatchShutdownRequest();      // ← 第一次观察到就 latch
    return true;
}
```
`:162-165` 的注释解释了三个目的：所有 shutdown 检查都走这里；第一次 true 观察就 latch（**在任何 shutdown 工作之前启动看门狗 deadline**）；即使注入的谓词后来变 false，后续检查仍然是 true。

**`ServingShutdownTrigger` 故意在锁外跑回调**（`:236-244`）：
```cpp
lock.unlock();
on_shutdown_();
return;
```
`:237-241` 的注释：`Stop() must be able to signal and join while the callback is inside server.stop(). If that callback blocks, the join in Stop() waits, and the watchdog is what bounds the whole process, not this interlock.` ——即：这个互斥锁**不是**超时的保证者，看门狗才是。这是个很清醒的分层。

**`ExitAfterShutdown` 只在看门狗存活时刷 glog**（`shutdown_request.cpp:104-110`）：
```cpp
LatchShutdownRequest();
if (g_watchdog_running.load(acquire)) google::FlushLogFiles(google::GLOG_INFO);
_exit(status);
```
条件刷日志的理由在头文件里：`flushes glog only when the watchdog thread exists, so a blocked flush still ends at the deadline`（`include/shutdown_request.h:39-41`）。如果 glog 的锁被卡住，刷日志会阻塞——但有看门狗在，最多阻塞到 deadline。没有看门狗就别冒这个险。

**看门狗线程不拿锁、不做 I/O**（`:39-53`）：只 sleep-poll 两个 atomic。注释：`so a stalled cleanup, a held glog mutex or a full stderr pipe cannot stop it`。测试（`mooncake-store/tests/shutdown_request_test.cpp`）专门覆盖了"stderr 是满管道"和"退出线程卡在 log flush 里"这两种情况——每个 case 在子进程里跑。

**`main()` 返回后的分流**（`master.cpp:1696-1702`）：
```cpp
const int result = supervisor.Start();
if (mooncake::ShutdownRequested() || mooncake::ShutdownLatched()) {
    mooncake::ExitAfterShutdown(result);   // 不跑 main/static 的 teardown
}
return result;
```
`:1696-1698` 注释：`Finish a requested process exit without further main/static teardown; supervisor-owned cleanup may already have run on this error path.`

**硬时间上界**：从第一次观察到 shutdown 起 15 秒（`kHaShutdownDeadline`，`master_service_supervisor.h:54`）。到点 `_exit(EXIT_FAILURE)`。二次信号立即 `_exit(128+signo)`。没有任何路径能突破这个上界。

---

## 与其他模块的边界

### 向下依赖：`HaKvBackend`（`include/ha/kv/ha_kv_backend.h`）

只有 6 个方法的极简抽象：`Get / Put / Range / DeleteRange / SupportsTxn / Txn`。`KvTxn` 是 `{vector<KvCompare> compares, vector<KvPair> puts}`，`KvCompareKind` 三种：`kValueEquals / kKeyNotExists / kCreateRevisionEquals`。

- `kCreateRevisionEquals` 是 etcd 特有的（比较 key 的创建修订号），快照维护租约用它防止"租约被别人重新创建过"（`batch_oplog_pruning_coordinator.cpp:171-174`、`batch_oplog_snapshot_publisher.cpp:123-126`）。
- `SupportsTxn()` 让不支持事务的后端能优雅拒绝：`InitDurablePrefix`（`oplog_batch_storage.cpp:97-99`）、`ClaimProducerView`（`:193-196`）、`WriteBatchAndAdvancePrefixImpl`（`:345-347`）都在开头检查它。

两个实现：`EtcdHaKvBackend`（走 `EtcdHelper` → cgo → Go clientv3）和 `RedisHaKvBackend`（hiredis + Lua 脚本模拟 CAS）。工厂在 `src/ha/kv/ha_kv_backend_factory.cpp`。

### 向下依赖：Go cgo 库

`EtcdHelper`（`src/etcd_helper.cpp`）和 `K8sLeaseHelper`（`src/k8s_lease_helper.cpp`）都是**薄封装**：真正的 etcd/k8s 逻辑在 Go 里（`mooncake-common/etcd/etcd_wrapper.go`、`mooncake-common/k8s-lease/k8s_lease_wrapper.go`），编译成 `c-shared` 的 `libetcd_wrapper.so` / `libk8s_lease_wrapper.so`。

两个文件都有 `#ifdef STORE_USE_ETCD` / `#else` 双份实现（`etcd_helper.cpp:15, 562, 788`；`k8s_lease_helper.cpp:14, 184, 245`），未编译进来时 etcd 侧每个方法都 `LOG(FATAL) << "Etcd is not enabled in compilation"`（`etcd_helper.cpp:563-568` 起），k8s 侧多数为 `LOG(FATAL)`、少数清理路径为 `LOG(ERROR)`（`k8s_lease_helper.cpp:187, 235, 241`）。

这个 cgo 边界解释了信号处理器为什么必须 `SA_ONSTACK`——Go runtime 在 c-shared 库里对信号栈有自己的要求。

### 向上被依赖：`MasterService` / `WrappedMasterService`

`MasterService` 持有三个 HA 成员（`master_service.h:2380-2383`）：`batch_oplog_kv_backend_`、`batch_oplog_storage_`、`ordered_oplog_writer_`。

初始化在构造函数里（`master_service.cpp:381-413`），条件是 `enable_oplog_ && !cluster_id_.empty() && !config.ha_backend_connstring.empty()`。关键的一行是 `:406`：
```cpp
InitializeBatchOpLogWriter(std::move(backend.value()),
                           /*require_fenced_writer=*/view_version_ > 0);
```
`:402-403` 的注释：`Direct MasterService construction has no leadership session; supervisor-created services carry a non-zero acquired view.` 也就是说，**只有 supervisor 造出来的 service（带非零 `view_version_`）才是 fenced writer**；测试里直接构造的 MasterService 拿到的是 unfenced writer（不带 producer_view 的 CAS）。

`InitializeBatchOpLogWriter`（`:13532-13592`）里 fenced 路径还会再调一次 `ClaimProducerView(view_version_)`（`:13549-13556`），并且 `require_fenced_writer && view_version_ == 0` 直接 `INVALID_PARAMS`（`:13537-13541`）。

supervisor 通过 `WrappedMasterService`（`rpc_service.cpp:98-104, 1846-1868`）转发这几个调用：`SetBatchOpLogTerminalCallback`、`StopBatchOpLogWriter`、`RestoreFromStandby`、`RestoreFromBatchOpLogPromotion`。

### 平级交互：`MasterAdminServer`

supervisor 通过它上报运行态：`SetRuntimeState`、`SetServiceAvailable`、`SetServiceDelegate`、`SetObservedLeader`。`ActivateServingState` / `DeactivateServingState`（`master_service_supervisor.cpp:332-346`）把这三个调用打包，注意 `Deactivate` 的顺序是**先 `SetServiceAvailable(false)` 再 `SetServiceDelegate(nullptr)`**——先停止对外宣告可用，再摘掉委托。

### 平级交互：`LeaderLabelReconciler`（`include/ha/leadership/leader_label_reconciler.h`）

只在 K8s + `pod_name`/`pod_namespace` 都非空时启用（`master_service_supervisor.cpp:43-45`）。它把"desired leader label"这个布尔量交给后台线程去收敛，失败就按 `kLabelReconcileRetryInterval`（1s）重试直到实际状态与期望一致。设计动机在头文件注释里：`so a transient failure cannot leave a stale leader label on a pod that is no longer serving`。调用方 `SetLeader(bool)` 是非阻塞的。

Label 是 `mooncake.io/store-role=leader`（`:40-41`），供 K8s Service selector 把流量导向当前 leader。

### 向上被依赖：Store Client（选主结果的消费者）

HA 的"暴露服务"不是 master 自己说了算，而是由客户端读取 `master_view` 决定的。`Client::ConnectToMaster`（`src/client_service.cpp:631-688`）在连接串被解析成 HA 后端时，会自己 `CreateLeaderCoordinator`，然后 `ReadCurrentViewOrWaitForReady(..., 30s)`（`:647-660`，超时常量在 `:71`）。拿到 `MasterView` 后 `SwitchLeader` 按 `view_version` 单调过滤：旧版本直接忽略，同版本同地址且上次 ping 成功也忽略（`:708-733`）。

之后 `LeaderMonitorThreadMain`（`:735-795`）用 `WaitForViewChange(known_version, 30s)` 阻塞等待变更；注释明确说明 30s 不是轮询周期，而是"watch 漏事件时的兜底重读"（`:736-743`）。etcd 的 warming 占位值在这里表现为"changed 但 current_view 为空"，客户端会清掉本地 view 并继续等（`:764-784`）——这就是 master 侧"当选 ≠ 可服务"在客户端侧的对应处理。

因此读 HA 代码时必须把 **master 端的 `master_view` 写入**和 **client 端的 `master_view` 读取**当成同一个协议的两半来看：`EtcdLeaderCoordinator::ReadCurrentView` 对 `__mooncake_service_warming__:` 前缀返回 `nullopt`（`etcd_leader_coordinator.cpp:179-207`），而客户端恰好把 `nullopt` 当作"暂不可用"。

### 数据怎么流过去

```
客户端 → RPC(coro_rpc) → WrappedMasterService → MasterService
                                                   │
                        ┌──────────────────────────┤ 元数据变更
                        ▼                          ▼
                 本地权威状态             OrderedOpLogWriter
                 (segment_manager,              │ Commit（分配 seq）
                  metadata shards)              ▼
                                        OpLogBatchStorage.WriteBatchAndAdvancePrefix
                                                │ Txn(producer_view CAS + prefix CAS)
                                                ▼
                                          HaKvBackend (etcd/redis)
                                                │
                        ┌───────────────────────┘ 轮询 durable_prefix
                        ▼
        OpLogBatchStandbyReader.PollOnce → OpLogApplier.ApplyOpLogEntry
                                                │
                                                ▼
                                      StandbyMetadataStore（影子）
                                                │ 晋升时 move
                                                ▼
                             PromotionContext → MasterService::RestoreFrom*
```

**Primary 与 Standby 之间没有直接的元数据 RPC 通道。** 复制拓扑完全由 KV 后端决定。`HotStandbyConfig::primary_address` 基本是遗留字段——`CreateStandbyService` 硬编码传 `""`（`standby_controller.cpp:33`），`HotStandbyService::Start` 的头文件注释也写明 `not used with OpLog backend-based sync`（`hot_standby_service.h:95-96`）。

---

## 设计取舍与易踩坑点

### 1. 为什么不用 Raft / 内建共识？

Mooncake 选择"外部协调后端 + 单 Primary 写"而不是内建 Raft。好处是复用了运维已有的 etcd/K8s，master 进程本身不需要实现日志复制的状态机；代价是 **etcd 成了新的单点依赖**，而且复制延迟受 etcd 往返和轮询间隔（默认 1s）限制。

`durable_prefix` 轮询而非 watch，是因为 standby 需要的是"批量拉取连续区间"，watch 的逐事件语义反而要自己攒批。

### 2. 为什么快照是 Standby 生成的，不是 Primary？

Primary 的权威状态是分片的、带锁的、在线服务的；对它做一致性快照需要停顿或 COW。Standby 的影子存储 `StandbyMetadataStore` 结构简单（两层 `unordered_map`，`standby_metadata_store.h:16-17`），而且**它本来就有一个天然的快照点：durable_prefix 对应的游标**。

代价就是 4.3 节讲的：捕获期间复制暂停。这个停顿被 `capture_pause_us` 指标暴露出来。

### 3. `shutdown_exit` 为什么不能 `return`

前面讲过，这里强调**改代码时的陷阱**：如果你在 `RunSupervisorLoop` 里新增一个 shutdown 检查点，用 `return shutdown_exit(...)` 是对的（`shutdown_exit` 内部 `_exit`，那个 `return` 只是为了让编译器闭嘴，实际永不执行）；但如果你写成一个普通的 `return -1` 或 `break`，就会析构本轮的 `EtcdLeaderCoordinator`，其析构函数 `ShutdownKeepAliveThread()` 会 `RevokeLease`（`etcd_leader_coordinator.cpp:155, 695-698`）——把租约白白让给可能还没准备好的节点。

注意 `HandleSupervisorError` 返回 true 时的 `return -1`（如 `:465, 488, 510`）是**故意**走析构路径的：那些是致命的后端配置错误，进程本来就要退出，租约被吊销反而是正确的。

### 4. `ServingStateGate` 不是超时机制

`:237-241` 那段注释值得反复读。gate 只保证"三个 shutdown 来源中只有第一个能跑回调"，**不保证回调会返回**。如果 `server.stop()` 卡住，`ServingShutdownTrigger::Stop()` 里的 join 就卡住，整个进程靠 15s 看门狗兜底。想在 gate 里加超时逻辑是错的方向。

### 5. `retry_timeout = 0` 意味着 writer 无限重试

见流程三第 6 点。这是**当前实现的事实**，不是文档里的意图。读代码时如果看到 `kRetryTimeout` 就以为"etcd 挂了会自动下台"，那是错的——生产路径到不了那个分支。真正让 Primary 下台的是 keepalive 失败触发的 `on_leadership_lost`。

### 6. Redis 后端的能力不对等

- `HaBackendSupportsOpLog` 对 K8S 返回 `false`（`ha_types.h:41-51`）——**K8s 后端只能选主，不能复制**。配 `enable_oplog + ha_backend_type=k8s` 会在启动时 `LOG(FATAL)`（`master.cpp:1496-1498`）。
- Redis 支持 OpLog，但部署文档明确警告：`A successful Redis write only means Redis accepted the record. Do not treat Redis persistence, including AOF appendfsync everysec, as a strict durability guarantee.`（`docs/.../mooncake-store-deployment-guide.md:364`）。
- Redis **没有** `PublishServiceReady` override，所以没有 etcd 那套 warming 占位值机制——acquire 成功后 leader 地址立即可见，客户端可能在 RPC listener 真正起来之前就尝试连接。
- Redis 的 `view_version` 是 Lua `INCR` 出来的计数器，不是全局修订号。跨后端比较 `view_version` 没有意义。

**生产 Primary-Standby 路径必须用 etcd。**

### 7. `ClientOffboardingJob` 的"快照屏障"——注释与代码不一致（此处需明确指出）

`include/client_offboarding.h:36-38` 的注释断言：
```
// Process-local residual work for one terminal Client incarnation. The job is
// intentionally not serializable: snapshots stay behind the pending-work
// barrier until the residual work converges.
```
并且 `ClientOffboardingWorker::HasPending()` 声明在 `:65-67`，标了 `[[nodiscard]]`。

**但我全仓搜索 `HasPending`，除了声明处没有任何调用点**（`grep -rn "HasPending" mooncake-store/` 只命中 `client_offboarding.h:65` 和无关的 `DynamicReplicationController::HasPendingLocked`）。`hot_standby_service.cpp` 和整个 `src/ha/` 目录里也搜不到 `offboarding` 字样。

所以我**无法确认**代码里存在一个实际生效的 pending-work 屏障。可能的解释有三种，我无法从当前代码判定是哪一种：屏障由别的机制间接实现（例如残留清理会写 OpLog，从而天然被 durable_prefix 排序覆盖——`ProcessClientOffboardingJob` 内部确实有 `AppendOpLogVisibleBeforeDurable` 调用，见 `master_service.cpp:2765`）；或者这是尚未落地的设计意图；或者屏障在快照 provider 内部通过别的方式实现而我没有找到。**这一点此处未完全确认**，读者若要依赖该语义，应自行验证或询问维护者。

可以确认的是：`ClientOffboardingJob` 确实不可序列化（成员里有 `std::shared_ptr<ClientLivenessRecord>`、`std::chrono::steady_clock::time_point`，`client_offboarding.h:40-52`），并且 `ReserveJob()`/`ScheduleReserved()` 两段式入队配合 `pending_jobs_` 原子计数（`client_offboarding.cpp:44-60`），worker 线程按 `next_attempt_at` 最小者优先、指数退避重试（`:62-67, 93-160`），master 停止时把队列里的 job 全部 `DropJob(..., "master_stopping")`（`:102-109`）。

### 8. `PrefixWatchGuard` 会**故意泄漏内存**

`etcd_leader_coordinator.cpp:80-129`。这是本子系统里最反直觉的一段代码，注释写得很完整：Go watch goroutine 持有 `ViewChangeWatchState*` 裸指针，`WaitWatchWithPrefixStopped` 有 5s 预算（`kWatchStopTimeoutMs`）。**如果超时，goroutine 可能还会回调**，此时释放 state 就是 use-after-free。所以：
```cpp
if (wait_err == ErrorCode::OK) delete state_;
else LOG(WARNING) << "... leaking ViewChangeWatchState to avoid UAF";
```
两害相权取其轻。改这段代码时不要"顺手修一下这个泄漏"。

同一函数 `:421-438` 还记录了另一个约束：**每个进程每个前缀只支持一个 waiter**，因为 Go wrapper 的 `storePrefixWatchCtx` 是 `map<prefix, ...>`。以及 issue #3059 的教训：早期实现每轮循环都新建 watch + 析构，每次 goroutine 错过 5s 预算就泄漏一个 state 和一个 OS 线程，高负载下线程数爆掉。现在改成整个 wait 期间复用一个持久 watch，只在 etcd 报告 broken 时重新 arm。

### 9. `WaitForViewChange` 的 arm-before-read 不变式

`:442-449` 的注释描述了一个经典的分布式竞态解法：watch 在 revision `R_watch` 建立，随后的 read 观察到 `R_read >= R_watch`。那么发生在 revision `C` 的变更，要么 `C < R_read`（被 read 看到），要么 `C >= R_watch`（被 watch 看到），**不可能从中间漏掉**。所以代码顺序是先 `arm_watch()` 再第一次 `ReadCurrentView()`（`:475, 491`）。

watch 从 `start_revision = 0`（即当前修订号）开始，避免依赖一个可能已被 compact 的历史修订号（`:446-448, 464-466`）。

### 10. `RebootstrapBatchOpLog` 的 try_lock 防死锁

`hot_standby_service.cpp:1059-1163`。这个函数在复制循环线程里跑，需要拿 `mutex_`；但 `Stop()`（`:355-377`）会**持有 `mutex_` 去 join 复制线程**。经典的死锁。

解法是 `lock_while_running` lambda（`:1072-1084`）：只在 `replication_loop_running_` 为真时 `try_lock`，拿不到就在 `replication_loop_cv_` 上等 1ms 重试；一旦 `replication_loop_running_` 变假就立刻返回 false 放弃。注释：`Stop/promotion can hold mutex_ while joining this worker. Never block on that mutex once the worker has been asked to stop.`

同一函数末尾的 swap 也讲究顺序（`:1144-1148`）：
```cpp
metadata_store_.swap(metadata);   // 局部变量现在持有旧对象
oplog_applier_.swap(applier);
batch_standby_reader_.swap(reader);
```
`:1144-1145` 注释：`Swapping preserves old state until all replacement references exist. Locals destroy the old reader before its applier and metadata store.` 局部变量按声明逆序析构（reader → applier → metadata），正好满足"先销毁持有引用的，再销毁被引用的"。

### 11. 两套状态机不要混用

`MasterRuntimeState`（七态，面向 admin/metrics 上报）和 `StandbyState`（九态，面向 HotStandbyService 内部复制流程）是不同层次的东西，通过 `MapStandbyRuntimeState`（`standby_controller.cpp:53-77`）单向映射。而且这个映射还依赖 `capabilities` 和 `observed_leader`——同样的 `WATCHING` 状态，开了 oplog 跟随且有 lag 时映射成 `kCatchingUp`，否则映射成 `kStandby`。

同一个进程在生命周期内会多次轮转角色（丢租约 → 回 standby → 再竞选），所以不要假设状态是单调的。

### 12. `accept_standby_runtime_updates` 原子门

`master_service_supervisor.cpp:424-432`。standby 控制器可以在**任意线程**（复制循环、状态机回调）上报运行态，但 supervisor 在晋升/服务期间不希望这些上报覆盖它自己设的状态。所以用一个 atomic bool 门控：`EnterStandbyMode` 打开（`:402`），claim 成功后、进入晋升前关闭（`:614`），`shutdown_exit` 里也关闭（`:441`）。

### 13. `embedded_master` 与 HA 无关

`src/embedded_master.cpp` 是测试/嵌入式场景的进程内单点 master。三个 listener 全部硬编码 `"127.0.0.1"`（`:35, 47, 72`），端口默认传 0 让内核分配（`:25-30` 注释解释了为什么不预先探测端口：`Probing free ports before binding races with other clients and processes`）。别把它当成"内嵌 HA 模式"。

### 14. legacy master snapshot 已被移除

`RejectLegacyLayout`（`oplog_batch_storage.cpp:519-571`）在 `InitDurablePrefix` 开头就检查三类遗留数据：`/oplog/<cluster>/latest` 键、20 位数字命名的单条 OpLog 键、`snapshot/` 下除四个控制键之外的 sidecar。发现任何一个都返回 `INCOMPLETE_OPLOG_CATCH_UP` 并打 ERROR 提示清理。这是**拒绝自动迁移**的显式选择：新旧布局混存的风险太高，宁可让运维手动清理。

`docs/source/design/` 下没有独立的 HA 设计文档。HA 设计细节的事实来源是 `docs/source/deployment/mooncake-store-deployment-guide.md:342-470` 的 High Availability 章节，加上 `docs/source/design/store/mooncake-store.md:58-65` 的 Snapshot & Restore 小节。

### 15. "RPC 已监听"不等于"已对外服务"——但端口确实已经 accept

supervisor 的启动顺序是：`RegisterRpcService`（`:842`）→ `StartLeadershipMonitor`（`:845`）→ `server.async_start()`（`:893`）→ `PublishServiceReady`（`:935`）→ `ActivateServingState`（`:942`）。也就是说 **listener 在 `PublishServiceReady` 之前就已经 bind + listen**。

"不暴露"靠的是 etcd 里的 `master_view` 仍然是 warming 占位值（`etcd_leader_coordinator.cpp:200-202`），客户端读不到可路由地址。但一个已经知道该进程 `host:port` 的客户端（例如上一任期缓存下来的地址，或运维手动连接）仍可能在此期间连上并调用 RPC handler。handler 本身不检查 leadership，只检查 `MasterService` 是否已经 restore 完成（restore 在 `async_start` 之前完成，`:758-788`）。所以：

- 在 restore 完成之前，RPC 端口即使被连上也没有可用的 service delegate（handler 注册在 `:842`，但 server 还没 start）。
- 在 restore 完成之后、`PublishServiceReady` 之前，端口可连且 metadata 已恢复，只是"选主协议上还没宣告"。

这不是数据正确性问题（restore 已经完成），但它解释了为什么排障时可能看到"客户端已经打进 RPC，admin 的 `ha/status` 却还显示 `leader_warmup`"。不要把 `ha/status` 的 `service_available` 当成 listener 是否存活的判据。

### 16. `enable_ha=false` 时根本不会用到本章的优雅退出

`shutdown_request` 是"模式无关"的工具（`include/shutdown_request.h:9-11`），但**只有 HA 分支调用它**（`master.cpp:1679-1688`）。非 HA 分支仍然用 `std::signal` + 一个局部 `std::atomic<bool>`（`master.cpp:1742-1747`），没有 watchdog、没有二次信号立即退出的语义。改退出逻辑时不要把两条路径混为一谈：非 HA 的"优雅退出"只是一个 100ms 轮询后 `server.stop()`（`:1756-1761`）。

### 17. `MasterAdminServer` 从进程启动就存在

`MasterServiceSupervisor::Start` 在进入主循环之前就构造并 `Start()` 了 admin server（`master_service_supervisor.cpp:1042-1049`），所以 standby 阶段也能提供 `/metrics`、`/health`、`/ha/status`。这正是 `MasterRuntimeState` 七态存在的意义：让外部能看到一个不服务数据的 standby。相对地，数据面的 `coro_rpc_server` 是**每个任期新建**的（`:695-697`），服务结束后销毁，下一任期再建。

---

## 精读清单

按这个顺序读，每一步都标注了"该看懂什么"。

**第 0 步：先读文档建立心智模型（20 分钟）**
- `docs/source/deployment/mooncake-store-deployment-guide.md:342-427`。重点看 `:413-425` 的六步晋升流程和 `:427` 的 fail-closed 动机（旧版"记日志继续服务"被明确定性为**正确性 bug 而非可用性回退**）。
- `docs/source/design/store/mooncake-store.md:48` 和 `:58-65`。

**第 1 步：类型契约（30 分钟）**
- `mooncake-store/include/ha/ha_types.h` 全文（217 行）。看懂 `MasterView`/`LeadershipSession`/`MasterRuntimeState` 七态，以及 `HaBackendSupportsOpLog`（`:41-51`）和 `ValidateHABackendAvailability`（`:71-96`）这两个能力判定。
- `mooncake-store/include/types.h:211-212`。确认 `ViewVersionId = EtcdRevisionId = int64_t`，并记住"别名不等于语义统一"。
- `mooncake-store/include/ha/oplog/oplog_types.h` + `oplog_batch_types.h`。看懂 `OpLogEntry` 的 8 个字段、11 种 `OpType`、`DurablePrefix` 的零字段约束、批次记录的连续性要求。
- `mooncake-store/src/ha/oplog/oplog_batch_types.cpp`。看懂 KV key 布局和 20 位零填充的 `FormatOpLogBatchId`（这是字典序 == 数值序的关键）。

**第 2 步：优雅退出（20 分钟，短小但语义精密）**
- `mooncake-store/include/shutdown_request.h` 全文（53 行）——注释就是规格说明。
- `mooncake-store/src/shutdown_request.cpp` 全文（112 行）。看懂：`HandleShutdownSignal` 的二次信号 `_exit`（`:31-37`）、`RunWatchdog` 的 deadline 只算一次（`:39-53`）、`ExitAfterShutdown` 的条件刷日志（`:104-110`）。
- `mooncake-store/src/master.cpp:1674-1702`。看懂为什么看门狗必须在任何租约获取前 arm。
- 配合 `mooncake-store/tests/shutdown_request_test.cpp` 看边界（子进程测试、满管道 stderr、卡在 log flush）。

**第 3 步：supervisor 主循环（90 分钟，本章重心）**
- `mooncake-store/include/ha/leadership/master_service_supervisor.h`。先看 `ServingStateGate`（`:17-48`）和 `kHaShutdownDeadline`（`:54`）。
- `mooncake-store/src/ha/leadership/master_service_supervisor.cpp` 按这个顺序：
  1. `:96-160` `LeadershipRenewalGuard`——保活侧线程。
  2. `:162-203` `ShutdownObserved` / `ShutdownLease` / `FinishShutdown`——latch 语义和三态租约交接。
  3. `:205-258` `ServingShutdownTrigger`——特别是 `:236-244` 的锁外回调。
  4. `:290-346` `WarmupLeadership` / `ActivateServingState` / `DeactivateServingState`。
  5. `:374-414` `ClaimProducerViewForServing` / `EnterStandbyMode`。
  6. `:416-568` `RunSupervisorLoop` 的竞选内循环。**看懂 `shutdown_exit` 为什么不能 return（`:434-444`）。**
  7. `:570-692` 任期门控：`term_lease` / `release_term` / renewal / claim / promotion / `return_to_standby` / warmup。
  8. `:694-956` 服务期：restore 门控 → preflight → worker pool → monitor → async_start → PublishServiceReady → ActivateServingState。
  9. `:982-1012` 服务期结束的拆除顺序。

**第 4 步：etcd 选主实现（45 分钟）**
- `mooncake-store/src/ha/leadership/backends/etcd/etcd_leader_coordinator.cpp`：
  - `:179-207` `ReadCurrentView`——warming 占位值如何被当成"无 leader"。
  - `:209-281` `TryAcquireLeadership`——`GrantLease` + `CreateWithLease`，失败要 `RevokeLease`。
  - `:283-317` `PublishServiceReady`——CAS 换掉占位值，以及幂等重解。
  - `:319-410` `RenewLeadership`——理解它其实是 keepalive 线程的适配器。
  - `:412-535` `WaitForViewChange`——**重点看 `:421-449` 的两段注释**（单 waiter 限制、arm-before-read 不变式）。
  - `:80-129` `PrefixWatchGuard`——故意泄漏的那段。
  - `:656-707` `ShutdownKeepAliveThread`——这就是"析构会吊销租约"的出处。

**第 5 步：OpLog 写入与存储（60 分钟）**
- `mooncake-store/include/ha/oplog/ordered_oplog_writer.h`——`Reservation`/`PendingHandle`/`CommitBatch` 的语义注释。
- `mooncake-store/src/ha/oplog/ordered_oplog_writer.cpp`：
  - `:21-198` `Impl` 的全部字段。
  - `:82-108` `SealCommittedEntriesIfIdle`——group 原子性扩展。
  - `:130-167` `EnterTerminalState`——锁外调 callback。
  - `:272-408` `ReserveBatch`/`Commit`/`CommitBatch`——准入上限、序号分配、强制清零 `timestamp_ms`/`prefix_hash`。
  - `:515-682` writer 线程主循环——**看懂 `:591-641` 的成功路径和 `:644-679` 的重试/终态分流**。
  - `:685-719` `Stop`——waiter 用 `UNAVAILABLE_IN_CURRENT_STATUS` 完成。
- `mooncake-store/src/ha/oplog/oplog_batch_storage.cpp`：
  - `:191-258` `ClaimProducerView`——单调 fence。
  - `:339-436` `WriteBatchAndAdvancePrefixImpl`——双 CAS + **`:396-435` 的模糊失败幂等重解**（最容易看漏）。
  - `:506-515` `DeleteBatchesThrough` + `:519-571` `RejectLegacyLayout`。
- `mooncake-store/src/master_service.cpp:13532-13592` `InitializeBatchOpLogWriter`——注意 `:13558-13560` 只设了两个字段（`retry_timeout` 保持 0）。
- `mooncake-store/src/master_service.cpp:13594-13718`——两种可见性语义的对比。

**第 6 步：Standby 侧（60 分钟）**
- `mooncake-store/include/standby_state_machine.h:49-76, 222-248` + `src/standby_state_machine.cpp:10-`。看懂九态和转移表，特别是 `IsReadyForPromotion()` 只认 `WATCHING`。
- `mooncake-store/src/ha/standby_controller.cpp`：
  - `:28-51` 能力构造——**注意 `primary_address = ""`**。
  - `:53-77` `MapStandbyRuntimeState`。
  - `:297-377` `PromoteStandbyAndExport`——两条互斥路径 + 失败时的 `MarkAppliedCursorForRestart`。
  - `:403-410` `RestorePromotionBaseline`。
- `mooncake-store/src/ha/oplog/oplog_batch_standby_reader.cpp` 全文（239 行）。**这是 standby 一致性的核心**，逐行读 `PollBatches`（`:92-237`），特别是前缀倒退检测（`:112-121`）、已证游标快路径（`:133-142`）、条目级校验（`:195-214`）、收尾判定（`:222-235`），以及 `PollOnce` 的 FATAL 重分类（`:69-88`）。
- `mooncake-store/src/ha/oplog/oplog_applier.cpp:48-137`——size 校验、checksum 校验、全局序号校验（旧 skip / 新 reject）。
- `mooncake-store/src/hot_standby_service.cpp`：
  - `:64-161` `Start`。
  - `:171-271` bootstrap 三条路径。
  - `:489-600` 最终追平（两种超时策略）。
  - `:602-655` `MarkAppliedCursorForRestart` / `PreservePromotionBaseline`。
  - `:805-876` `PromoteAndDetachBatchOpLogStore`——**逐步核对 9 个校验点**。
  - `:938-1051` 快照捕获握手 + `:1059-1163` `RebootstrapBatchOpLog`（try_lock 防死锁、swap 顺序）。
  - `:1165-1326` `ReplicationLoop`。

**第 7 步：快照与修剪（45 分钟）**
- `mooncake-store/include/ha/snapshot/batch_oplog/metadata.h` + `src/.../metadata.cpp:374-446`——key 布局和 `snapshot_id = <batch_id>-<lease_id>` 格式。
- `mooncake-store/src/ha/snapshot/snapshot_maintenance_lease.cpp` 全文（127 行）——`Acquire`/`IsHeld`/`Release`，`owner_token = std::to_string(lease_id)`。
- `mooncake-store/src/ha/snapshot/batch_oplog/batch_oplog_snapshot_coordinator.cpp:311-556` `RunAttempt`——**数一数总共有几个取消检查点**（`:388-401, 433-445, 474-489, 525-528`）。
- `mooncake-store/src/ha/snapshot/batch_oplog/batch_oplog_snapshot_publisher.cpp` 全文（153 行）——四路 compare 的事务，`fallback ← latest` 轮转。
- `mooncake-store/src/ha/snapshot/batch_oplog/batch_oplog_pruning_coordinator.cpp` 全文（203 行）——`ValidateSnapshot`（`:30-78`）和五路 compare 的 floor 事务（`:167-186`），**确认"先发布 floor 再删批次"的顺序（`:193-200`）**。
- `mooncake-store/src/ha/snapshot/batch_oplog/batch_oplog_snapshot_provider.cpp:407-500` `RestoreBaseline`——latest → fallback → 全量回放的三级降级。

**第 8 步：验证与延伸（按需）**
- `mooncake-store/tests/ha/leadership/high_availability_test.cpp`、`mooncake-store/tests/ha/oplog/ordered_oplog_writer_test.cpp`、`mooncake-store/tests/shutdown_request_test.cpp`。
- `mooncake-store/tests/ha/leadership/etcd_leader_hang_e2e.sh`、`etcd_leader_lease_ttl_e2e.sh`——真实 etcd 的端到端脚本。
- 提交 `3db34ceb`（drain，#4745）和 `292e8391`（可配置 lease TTL，#4706）的完整 message，里面有设计动机的第一手叙述。

---

### 附：本章未能完全确认的点

1. **`ClientOffboardingJob` 的 pending-work 屏障**：注释断言存在（`include/client_offboarding.h:36-38`），`HasPending()` 已声明（`:65-67`），但全仓无调用点，快照路径代码里也搜不到 offboarding 相关引用。是否通过"残留清理会写 OpLog"间接生效，我未能确认。
2. **`retry_timeout` 是否有计划接入生产配置**：当前事实是它恒为 0（只有测试设置）。这是有意为之（依赖 keepalive 失败来兜底）还是遗漏，代码和注释里都没有说明。
3. **Redis 后端的 `view_version` 计数器在 Redis 重启/flush 后的行为**：`ClaimProducerView` 的单调 fence 依赖 `view_version` 全局单调，而 Redis 的计数器来自 Lua `INCR`。Redis 数据丢失后计数器回退是否会导致 fence 失效，我没有追进 Lua 脚本（`kAcquireLeadershipScript`）和 Redis KV 后端的事务实现去确认。文档已把 Redis 定性为实验性且"写成功不等于持久"，实践上不应依赖。

---

# 09 持久化与分层存储后端

> 本章基于仓库 `D:/xiaojun/work/Mooncake`（commit `67df8c41`）的实际源码精读写成。所有技术断言均带 `相对路径:行号`，可直接跳转核对。
>
> **路径约定**：本章所有相对路径以仓库根目录为基准。注意范围提示里的文件都位于 `mooncake-store/` 子目录下：例如 `mooncake-store/src/storage_backend.cpp`、`mooncake-store/src/file_storage.cpp`。仓库根目录**没有**独立的 `src/storage_backend.cpp`。另外 `master_store_backend.cpp` 与磁盘分层无关（见“设计取舍”一节），`local_ssd/` 与 `spdk/` 也不在 client 本地 offload 路径上。

---

## 定位与职责

Mooncake Store 的热数据在 DRAM 里（memory replica），冷数据需要一条**比 DRAM 便宜、但比对象存储快**的落盘通道。本子系统就是这条通道：

1. **容量扩展**：master 在内存压力下通过心跳点名若干 key，real client 把它们写到本地 SSD / NVMe KV / 共享 DFS / 对象存储。
2. **副本登记**：写成功后 client 调用 `NotifyOffloadSuccess`，master 给对象追加一个 `LOCAL_DISK` 副本，并在副本描述符里记录属主 client 的 RPC 地址。DFS 副本**不**走这条通知：它在 `PutStart` 时由 master 的 DFS allocator 分配 descriptor，直接以 PROCESSING 副本写进元数据（见流程 E）。
3. **读回落**：读路径在内存副本不可用时，向“拥有该盘上副本的 real client”发 RPC，由它从盘上读进预注册的 `ClientBuffer`，再由 Transfer Engine 用 RDMA/TCP 拉到请求方。
4. **自动回收**：盘用量超过水位时，后端先通知 master 删副本，master 接受后才真正删文件——**盘上数据的删除权掌握在 master 手里**。

一个关键认知：**`LOCAL_DISK` 副本不是“共享盘”，而是“某个 client 进程拥有的盘”**。读请求永远经过拥有者进程转发（`docs/source/deployment/object-storage-offload.md` 明确写了 S3/OSS 也是如此），所以拥有者进程下线要先 `DrainLocalDiskSegment` 做优雅摘除。

子系统内部有两条完全不同的数据面：

- **本地/对象 offload 数据面**：`FileStorage` 心跳线程 → `StorageBackendInterface::BatchOffload/BatchLoad`。覆盖 file-per-key、bucket、offset-allocator、NVMe KV、对象存储五种实现。
- **DFS 描述符数据面**：master 在 `PutStart` 时分配 `DistributedFSDescriptor`，client 拿描述符直接 `BatchWrite/BatchRead` 到共享文件系统。它**不走** `LOCAL_DISK` 心跳 offload（`mooncake-store/src/file_storage.cpp:142-147` 在 DFS 模式下直接 return，连心跳线程都不启动）。

---

## 目录结构速览

| 文件 | 一句话职责 |
|---|---|
| `mooncake-store/include/storage_backend.h` | 子系统总契约：`StorageBackendInterface`、`FileStorageConfig`、五种后端声明、`RecordHeader`(v3 盘上格式)、`BucketMetadata` |
| `mooncake-store/src/storage_backend.cpp` | 5663 行：file-per-key 原语、`StorageBackendAdaptor`、`BucketStorageBackend`、`OffsetAllocatorStorageBackend`、`CreateStorageBackend` 工厂 |
| `mooncake-store/include/file_interface.h` | `StorageFile` / `PosixFile` / `UringFile` / `FileLockRAII` |
| `mooncake-store/src/posix_file.cpp` | `preadv/pwritev/fdatasync`，写失败时析构 unlink 损坏文件 |
| `mooncake-store/src/uring_file.cpp` | 每线程一个 io_uring ring、进程级固定缓冲区注册、O_DIRECT bounce buffer、`batch_read` |
| `mooncake-store/src/uring_submit.h` | `submit_all_pending`：SQ 提交重试（EINTR/EAGAIN/ENOMEM） |
| `mooncake-store/include/file_storage.h` + `src/file_storage.cpp` | client 侧总协调者：心跳、offload/promotion、`BatchGet` 远程读、buffer 租约、优雅 drain |
| `mooncake-store/src/config/file_storage_config.cpp` | `MOONCAKE_OFFLOAD_STORAGE_BACKEND_DESCRIPTOR` 等环境变量解析 |
| `mooncake-store/include/storage/distributed/` | DFS/对象存储：`DistributedStorageBackend`、`FileSystemAdapter`、`ObjectStorageAdapter`、`ShardAllocator`、`ImmutableBucketAllocator` |
| `mooncake-store/src/storage/distributed/` | 上述实现，含 S3/OSS REST 引擎（HMAC 签名 + curl_multi） |
| `mooncake-store/include/nvme_kv/` + `src/nvme_kv/` | NVMe KV：16B 物理 key 编解码、冲突槽、inline/manifest 对象布局、三级线程池 |
| `mooncake-store/include/local_ssd/` + `src/local_ssd/` | **master 侧**：每 client 的 offload/promotion 任务邮箱与容量账本 |
| `mooncake-store/include/spdk/` + `src/spdk/` | NVMe-oF 远程 SSD 池（NoF）封装，**不在本地 offload 路径** |
| `mooncake-store/include/hf3fs/` + `src/storage/distributed/hf3fs_adapter.cpp` | 3FS USRBIO 资源管理与 DFS 适配器 |
| `mooncake-store/include/config/{bucket_backend,offset_allocator_backend,file_per_key}_config.h` | 各后端的策略开关 |
| `docs/source/design/store/ssd-offload.md` | 架构与数据流总览（建议先读） |
| `docs/source/deployment/ssd/{ssd-offload,nvme-kv,nvmf-ssd-deployment-guide}.md` | 部署与环境变量 |
| `docs/source/deployment/object-storage-offload.md` | OSS/S3 拓扑与凭据配置 |

---

## 核心数据结构

### 1. `StorageBackendInterface`：六个纯虚函数是核心契约

定义在 `mooncake-store/include/storage_backend.h:262-318`。六个纯虚方法如下：

| 方法 | 语义 |
|---|---|
| `Init()` | 打开设备/恢复盘上元数据；必须只调用一次 |
| `BatchOffload(batch_object, complete_handler, eviction_handler)` | 批量写；**写成功但 master 未被告知**时必须走回滚/上报失败 |
| `BatchLoad(batched_slices)` | 批量读到调用方预分配的 `Slice` |
| `IsExist(key)` | 存在性探测（用于 dangling replica 自愈） |
| `IsEnableOffloading()` | 是否还能继续 offload（配额/水位） |
| `ScanMeta(handler)` | 扫描盘上已有对象并批量回调，用于重启后向 master 重注册 |

除了这六个纯虚方法，接口里还有 4 个**带默认实现**的虚函数，它们在真实流程中同样会被调用，读代码时不能漏看：`ResetScanIterator()`（bucket 的 `ScanMeta` 基于游标，重扫前必须调用，见 `FileStorage::ReRegisterOffloadedObjects`）、`RemoveAll()`（master 要求清盘时由 `Heartbeat` → `PollRemoveAll` 触发）、`EvictAboveDiskWatermark(high, low, handler)`（主动水位驱逐）、`SetTestFailurePredicate(...)`（仅测试注入，默认 no-op）。

两个回调的事务语义是全章最容易读错的地方：

- `complete_handler(keys, metadatas)` 由 `FileStorage` 提供，内部调用 `client_->NotifyOffloadSuccess`（`mooncake-store/src/file_storage.cpp:342-367`）。它返回非 OK 意味着**数据已经在盘上、但 master 不知道**。此时 bucket 后端会 `RollbackCommittedBucket` 把本地索引和文件一起撤掉（`mooncake-store/src/storage_backend.cpp:1667-1680`），offset-allocator 后端则直接返回错误、依赖下次 `ScanMeta` 兜底（`storage_backend.cpp:5243-5255`）。
- `EvictionHandler(evicted_keys)` 由 `FileStorage` 提供，内部调用 `client_->BatchEvictDiskReplica(..., ReplicaType::LOCAL_DISK)`（`file_storage.cpp:579-626`）。**只有它返回成功，后端才被允许删文件**。这是“两阶段驱逐”的核心。

### 2. `FileStorageConfig`

`storage_backend.h:205-260`。要点：

- `storage_backend_type` 默认 `kBucket`；`storage_filepath` 默认 `/data/file_storage`，且 `Validate()` 强制**目录必须已存在、绝对路径、非符号链接、可写**（`mooncake-store/src/config/file_storage_config.cpp:146-205`）。即使是对象存储模式也要这个目录，只是不拿它缓存数据。
- `local_buffer_size` 默认 1280 MiB（`storage_backend.h:213`），这是 staging/读缓冲池大小，不是磁盘配额。
- `total_size_limit` 默认 2 TiB、`total_keys_limit` 默认 1000 万；不同后端对它们的解释不同（bucket 用 `max_total_size`，offset-allocator 直接当文件容量 `capacity_`）。
- `use_uring`：全局开关，决定 `PosixFile` 还是 `UringFile`。
- `enable_dfs` 不是环境变量直接配的，而是 `FileStorage` 构造时根据 `DistributedStorageBackend::UsesObjectStorage()` 反向设置（`mooncake-store/src/file_storage.cpp:86-93`）。

### 3. `StorageBackend`（file-per-key 原语）

`storage_backend.h:328-655`。它**不是** `StorageBackendInterface` 的实现，而是 `StorageBackendAdaptor` 的底层工具类，负责：

- 目录状态重建：`Init(quota_bytes)` 递归扫描、按创建时间排序、重建 FIFO 队列、计算 `used_space_/total_space_/available_space_`；配额缺省为磁盘容量的 90%（`storage_backend.cpp:95-206`）。
- 空间预留与 FIFO 驱逐：`EnsureDiskSpace` 先 `CheckDiskSpace` 预留，不足则循环 `PopFileToEvictByFIFO`，凑够后再调用 `eviction_handler`，master 接受后才 `DeleteEvictedFile`（`storage_backend.cpp:936-1020`）。
- 路径级 64 路分段锁 `file_path_mutexes_`（`storage_backend.h:518-519`，取锁 `storage_backend.cpp:85-88`），把“同一路径的写/删”串行化；`pending_eviction_paths_` 防止正在被驱逐的文件被重新写入。
- 物理文件选择：`create_file` 只对 **读** 加 `O_DIRECT`；`use_uring_` 打开时读、写都返回 `UringFile`，但写模式的 `use_direct_io=false`（buffered，不加 `O_DIRECT`）（`storage_backend.cpp:701-737`）。注释给的理由是“写延迟不敏感，且 O_DIRECT 写要 4KiB padding，会破坏 meta 解析、浪费空间”。

### 4. `BucketMetadata` / `BucketReadGuard` / `StorageObjectMetadata`

- `BucketMetadata`（`storage_backend.h:40-98`）：`meta_size`、`data_size`、`keys`、`metadatas`（每个对象的 `{offset,key_size,data_size}`）。注意 `YLT_REFL(BucketMetadata, data_size, keys, metadatas)` **没有序列化 `meta_size`**（`storage_backend.h:98`），`meta_size` 是写 `.meta` 文件后回填的序列化长度（`storage_backend.cpp:3383`）。
- `inflight_reads_` / `last_access_ns_` 是 `mutable std::atomic` 的**运行时字段**，拷贝/移动构造时显式清零（`storage_backend.h:56-96`），保证不会被元数据序列化带走。
- `BucketReadGuard`（`storage_backend.h:112-154`）：构造时 `fetch_add(1, relaxed)`，析构 `fetch_sub(1, release)`。它让删除方可以“先摘索引，再等 in-flight 读归零，最后删文件”。
- `StorageObjectMetadata`（`mooncake-store/include/types.h:539-547`）：`{bucket_id, offset, key_size, data_size, transport_endpoint}`。它是 client 与 master 之间的副本描述符载体；`transport_endpoint` 由 `FileStorage` 在调用 `complete_handler` 前填入本机 RPC 地址（`file_storage.cpp:347-349`、`184-199`）。
- `RecordHeader`（`storage_backend.h:1249-1359`）：offset-allocator 后端的盘上记录头，v3 布局：

```text
[u32 key_len][u32 value_len][u64 seq][u32 flags][u32 crc32][key][zero pad][value]
```

  `SIZE=24`，`value` 起点按 `key_len` 纯函数对齐到 4 KiB（`ValueOffsetInRecord`），为 GDS/DMA 预留；`seq` 是写入时的 `insert_seq_` 戳；`flags & kFlagHasCrc` 决定 `crc32` 是否可信（flag 定义见 `storage_backend.h:1268-1272`）；未知 flag 位在恢复扫描时会导致该 extent 被丢弃并释放（`storage_backend.cpp:4225-4232`）。

### 5. `BucketStorageBackend` 的内部状态

`storage_backend.h:1064-1131`：

- `mutex_`（`SharedMutex`）保护 `object_bucket_map_`、`buckets_`、`total_size_`、`lru_index_`、`pending_eviction_*`、`pending_write_*`。
- `object_bucket_map_`：key → `StorageObjectMetadata`（含 bucket_id/offset）。`buckets_`：`std::map<bucket_id, shared_ptr<BucketMetadata>>`，bucket_id 单调递增，所以 FIFO 候选就是 `begin()`。
- `lru_index_`：`std::set<pair<last_access_ns, bucket_id>>`。它是**惰性索引**：读路径只原子更新 `last_access_ns_`，不碰这个 set；驱逐时才在独占锁下修复（`storage_backend.cpp:2724-2755`）。
- `pending_eviction_keys_` / `pending_eviction_size_`：已从索引摘除、但 master 尚未确认的 key 与字节数；`pending_write_keys_` / `pending_write_size_`：正在写、尚未提交的 key。它们参与配额投影，避免并发 offload 把同一份空间算两遍。
- `file_cache_`：读 fd 缓存。**当前代码里 `GetOrOpenFile` 只有头文件声明（`storage_backend.h:1127`）和定义（`storage_backend.cpp:3488`），没有任何调用点**，`BatchLoad` 实际每次 `OpenFile`（`storage_backend.cpp:1786`）。`file_cache_` 的 erase 逻辑仍然存在（`FinalizeEviction`/`CleanupOrphanedBucket`），属于“为缓存预留、目前基本闲置”的代码，读时不要误以为热路径命中了缓存。

### 6. `OffsetAllocatorStorageBackend`

- 单个预分配文件 `kv_cache.data` + 元数据 `kv_cache.meta`（`storage_backend.cpp:3632-3638`）。
- 元数据分 1024 个 shard，每个 shard 一把 `SharedMutex` + 一个 `unordered_map`（`storage_backend.h:1425-1449`），`ShardForKey` 用 `hash & 1023`。
- `ObjectEntry{offset,total_size,value_size,AllocationPtr allocation,fifo_seq}`（`storage_backend.h:1388-1414`）。`AllocationPtr = shared_ptr<RefCountedAllocationHandle>`：**读到一半被驱逐也不会 UAF**，因为物理 extent 由最后一个引用释放。
- FIFO 索引 `fifo_index_`（`seq → key`）与 `insert_seq_`（全局单调）；`eviction_mutex_` 保护两者，且**必须先于 shard 锁获取**（`storage_backend.h:1494-1504`）。
- 持久化结构 `OffsetAllocatorPersistedMetadata`：`{version, allocator_state, insert_seq, fifo_entries, evicted_keys_this_batch}`（`storage_backend.h:185-193`），当前版本 3。

### 7. 文件 I/O 层：`StorageFile` / `PosixFile` / `UringFile`

- `StorageFile`（`file_interface.h:57-163`）：纯虚 `write/read/vector_write/vector_read`，虚 `datasync()`，`error_code_` 在写失败时被置为 `FILE_WRITE_FAIL`，析构据此决定是否 `unlink`（`posix_file.cpp:25-42`）。`SetDeleteOnWriteFail(false)` 用于 fresh-start 新建的数据文件与恢复目标数据文件，避免写失败误删可恢复数据（`storage_backend.cpp:3847-3849`、`4640`）。
- `PosixFile`：`preadv/pwritev` 一次系统调用完成 scatter/gather；`write` 循环处理 EINTR；`vector_write` 要求**一次写满**，否则返回 `FILE_WRITE_FAIL`（`posix_file.cpp:113-132`）。
- `UringFile`：见流程 C 的第 3 步与“设计取舍”第 4 条。

### 8. DFS / 对象存储的数据结构

- `DistributedStorageMode{kFileSystem,kObjectStorage}`（`distributed_storage_backend.h:20-23`）；`DfsWriteRequest/DfsReadRequest = {key, descriptor, slices}`（`:25-35`）。DFS 读写必须带 `DistributedFSDescriptor`，因为 key 本身无法定位 shard/offset。
- `DistributedFSDescriptor{file_path, offset, object_size, aligned_size, shard_idx}` 定义在 `mooncake-store/include/replica.h:264-270`。bucket 模式下 `shard_idx` 复用为 bucket id（`dfs_allocator_interface.h:39-44`）。
- `FileSystemAdapter`（`fs_adapter.h:34-176`）：文件管理和 fd 级 `WriteAt/ReadAt`，可选 `BatchWriteAt/BatchReadAt`（默认串行）、`SupportsFdReopen()`（POSIX 为 true）、`SupportsBatchIo()`（当前 3FS 为 true）。`FdIoRequest{fd, iov, iovcnt, offset}` 是批 I/O 的单元（`fs_adapter.h:20-25`）。
- `ObjectStorageAdapter`（`object_storage_adapter.h:41-104`）：逻辑 key 的 `Put/PutV/Get/PutBatch/GetBatch/Exists/Delete/ListKeys`。逻辑 key 可以为任意字节，物理 key 由适配器负责可逆编码。
- `ShardAllocator::ShardState`（`shard_allocator.h:120-163`）：每 shard 一个 `OffsetAllocator` + `offset_to_handle` + 独立 LRU + `pending_free` 延迟释放队列。
- `ImmutableBucketAllocator`（`immutable_bucket_allocator.h:25-190`）：`BucketEntryState{PENDING,COMMITTED,TOMBSTONE}` × `BucketLifecycle{ACTIVE,SEALED,FROZEN,EVICTING,RETIRED}` 双状态机；`BucketState` 只存在于内存，`Init` 拒绝任何已有 bucket 文件的根目录（`immutable_bucket_allocator.cpp:96-109`）。

### 9. NVMe KV 的数据结构

- `NvmeKvPhysicalKey = std::array<uint8_t,16>`（`key_codec.h:12`）。16 字节物理 key 由 canonical identity 连同 role/slot/chunk_index 一起用两个 XXH64 seed 派生（`key_codec.cpp:32-50`、`:91-107`），另有一个 32 字节 `verify_hash`（4 个 XXH64）写进对象头（`object_layout.h:18-28`）。
- `NvmeKvObjectHeader{magic,object_type,payload_size,verify_hash,payload_checksum,header_checksum,identity_metadata_size}`（`object_layout.h:18-28`）。
- 对象两种形态：`kInline`（header+identity+payload 一个 value）与 `kManifest`（header+identity+manifest，manifest 指向多个 chunk 物理 key）（`object_layout.h:13-16`、`41-50`）。
- `NvmeKvStorageBackend` 内部：`manifest_cache_`（`shared_mutex` 保护）、`connector_`、`total_size_/total_keys_` 原子计数、三组 `ThreadPool`（`backend.h:83-96`）。

### 10. master 侧 `LocalSsdManager` / `LocalSsdTaskMailbox`

`local_ssd/manager.h:21-151`：

- 每个注册的 real client 一条 `ClientRecord`：`total_capacity_bytes`（受 `stats_mutex` 保护）、`used_bytes`（原子）、`mailbox`。
- mailbox 内是 `pending_offloads_`（按 scoped key 去重）与 `pending_promotions_`（去重 + `seq` 重新排序）。`TakePromotions` 用 `partial_sort` 按 seq 降序取前 N（`manager.cpp:63-88`），保证积压时也是“最近被再次触达的优先”，不会退化成哈希序。
- `ExportPersistedState/RestorePersistedState` 让 **pending offload 队列**（连同 `enable_offloading` 与 `total_capacity_bytes`）进 HA 快照（`persisted_state.h`、`manager.cpp:356-394`）；注意 `pending_promotions_` **不** 被持久化，重启后 promotion 队列为空。
- 容量上报 `ReportCapacity` 返回 `{previous,current}`，供 master 更新 `file_total_capacity_`；`AdjustUsedBytes` 只在 `NotifyOffloadSuccess`（加）与 `ReleaseLocalDiskUsage`（减）两处被调用（`master_service.cpp:9148`、`2139`），promotion 不直接调整这个账本。

---

## 关键流程剖析

### 流程 A：启动与盘上状态恢复

```text
RealClient::setup (real_client.cpp:1484-1518)
  └─ FileStorage::FileStorage            // 建 allocator、选 backend、注册 io_uring 全局 buffer
  └─ FileStorage::Init                   // file_storage.cpp:129-228
       ├─ RegisterLocalMemory            // ClientBuffer 以 remote_accessible=true 注册
       ├─ storage_backend_->Init()
       │    ├─ BucketStorageBackend::Init          // storage_backend.cpp:1906-2169
       │    ├─ OffsetAllocatorStorageBackend::Init // storage_backend.cpp:3642-3913
       │    ├─ StorageBackendAdaptor::Init         // storage_backend.cpp:1161-1191
       │    └─ DistributedStorageBackend::Init     // distributed_storage_backend.cpp:174-214
       ├─ MountLocalDiskSegment(enable_offloading_)        // master 注册 LOCAL_DISK 段
       ├─ ReportSsdCapacity(total_size_limit)              // 仅用于展示分母
       ├─ ScanMeta(handler) → NotifyOffloadSuccess         // 重注册盘上对象
       └─ 启动 heartbeat_thread_ + client_buffer_gc_thread_
```

分后端看恢复语义，这是本章最值得精读的部分：

**Bucket**（`storage_backend.cpp:1906-2169`）：

1. 先在 `storage_path_` 下取 `.mooncake_local_disk.lock` 的**非阻塞独占 flock**；失败即 `FILE_OPEN_FAIL`（`:1914-1938`）。这确立了“一个目录只能有一个活 client”的契约，直接排除了两个进程写同一批 bucket_id 的可能。
2. 扫描所有 `.meta`：解析失败、或对应 `.bucket` 数据文件缺失/不是普通文件、或 `data_size/meta_size/keys/metadatas` 任一为空 → 把该 bucket 的数据与元数据都删掉并跳过（`:1970-2054`）。
3. 对每个有效 bucket 重建 `object_bucket_map_`、`total_size_`、`lru_index_`（时间戳置 0，因而 LRU 下“从未被读”的桶最先被淘汰，等价于 FIFO）。
4. 第二遍扫描 `.bucket` 文件，删掉没有对应 `.meta` 的孤儿文件（`:2072-2131`）。这是“数据写成功但元数据写失败”崩溃窗口的兜底。
5. `max_total_size` 未配置时取磁盘容量的 90%（`:2140-2150`）；`BucketIdGenerator` 从最大已有 id 续号（`:2152-2160`）。

**OffsetAllocator**（`storage_backend.cpp:3642-3913` + `4499-4678`）：`TryRecoverFromMetadata` 返回四种结果，语义差别巨大：

| 结果 | 触发条件 | 行为 |
|---|---|---|
| `kRecovered` | 元数据、allocator、数据文件大小全部校验通过 | 恢复索引与 FIFO，直接可用 |
| `kNoMeta` | 没有 `.meta` | 真正的首次启动，清空重建 |
| `kCorrupt` | 版本不匹配、解析失败、capacity 不匹配、数据文件大小不匹配、数据文件 ENOENT | **允许从零开始**（会 `O_TRUNC` 数据文件） |
| `kTransientError` | `FILE_OPEN_FAIL`、`bad_alloc`、fstat 失败、非 ENOENT 的 open 失败 | **绝不能擦数据**：`Init` 直接失败，让运维重试 |

恢复细节：`RebuildShardMapsFromAllocator` 遍历 allocator 里标记为 used 的 extent，逐条读 header/key/value；任何 `seq >= checkpoint.insert_seq` 的记录被判为“checkpoint 之后写的、可能是撕裂写”而丢弃并释放 extent（`storage_backend.cpp:4258-4272`）；带 CRC 的记录会分块校验（1 MiB chunk，`:4294-4337`）；同一 key 出现两份时保留 seq 更大的一份（`:4339-4397`）。随后 `RestoreAndRepairFifoIndex` 恢复 FIFO（`:4422-4493`）：A 阶段丢弃 seq >= insert_seq 的条目；B 阶段给 shard map 里没有 FIFO 槽的 key 补新 seq；C 阶段根据 `evicted_keys_this_batch` 删除“复活”的已驱逐 key。

**FilePerKey**（`storage_backend.cpp:1359-1444`）：扫描两级哈希目录，读每个文件、`struct_pb::from_pb` 出 `KVEntry`，回填 `total_keys/total_size`，并按 `scanmeta_iterator_keys_limit` 分批回调。注意它在扫描时**持有 `mutex_` 整个回调过程**，因此大目录启动会阻塞。

**DFS**：`DistributedStorageBackend::Init` 只建目录、初始化 adapter；**不扫描、不恢复任何分配状态**（`distributed_storage_backend.cpp:198-213`）。DFS 的分配器在 master 侧（`master_service.cpp:518-548`），并且 `enable_oplog` 与 DFS 互斥（`master_service.cpp:503-508`），`single_tenant=false` 也被拒绝（`:510-516`）。`ImmutableBucketAllocator` 更激进：根目录里已有任何 bucket 文件就拒绝启动（`immutable_bucket_allocator.cpp:96-109`）。

**对象存储**：`Init` 后可选 `CheckHealth()`（写读删探针，`distributed_storage_backend.cpp:180-195`）；`ScanMeta` 用 `ListKeys()` 枚举逻辑 key 并按批回调（`:863-904`）。

最后 `FileStorage::Init` 对 DFS 有一个**提前返回**：`config_.enable_dfs` 为真时只启动 buffer GC 线程就返回（`file_storage.cpp:142-147`）。这意味着 DFS 模式**不 mount LOCAL_DISK 段、不跑 offload 心跳、不做 ScanMeta 重注册**——DFS 副本由 master 的 allocator 直接管理，与 `LOCAL_DISK` 是两套东西。

### 流程 B：一次 offload（内存 → 盘）

```text
heartbeat_thread_ (每 heartbeat_interval_seconds)
 └─ FileStorage::Heartbeat                         // file_storage.cpp:662-812
     ├─ draining_? return
     ├─ (可选) 重试 metadata_resync_pending_ 的后台 rescan
     ├─ [offloading_mutex_] client_->OffloadObjectHeartbeat(enable_offloading_, tasks)
     │    └─ SEGMENT_NOT_FOUND? → MountLocalDiskSegment + ReportSsdCapacity + 重试
     │         + 异步 ReRegisterOffloadedObjects()
     ├─ client_->PollRemoveAll() → RemoveAll()
     ├─ OffloadObjects(tasks)                      // file_storage.cpp:307-577
     │    ├─ tenant scoped key 化
     │    ├─ BucketStorageBackend::AllocateOffloadingBuckets  // 按 bucket_keys_limit / bucket_size_limit 分桶；跳过盘上已存在的 key；flush_partial_buckets=false 时余下的 key 存进 ungrouped_offloading_objects_ 留到下一轮
     │    ├─ 每个 bucket: BatchQuerySegmentSlices(user_keys, tenant) → 内存 Slice
     │    ├─ 设备指针 → PinnedBufferPool D2H staging
     │    ├─ storage_backend_->BatchOffload(host_batch_object, complete, eviction)
     │    └─ 未成功/未上报的 key 收集进 failed_tasks
     ├─ failed_tasks → NotifyOffloadSuccess(tasks, data_size=-1 哨兵)
     ├─ ProcessPromotionTasks()
     └─ RunDiskWatermarkEviction()
```

几个必须看懂的点：

1. **`BatchQuerySegmentSlices` 是 best-effort**（`file_storage.cpp:1047-1073`）：它调 `client_->BatchQuery` 查副本列表，只挑“在本进程内存里”的副本，拿不到就当作失败任务。注释明确说它现在总是返回 OK（`:400-403`），失败靠 `failed_tasks` 表达。
2. **D2H staging**：`storage_backend` 的所有实现都假定收到的是 host 指针（`ConcatSlicesToString`/`BuildBucket`/`WriteBucket` 直接 memcpy）。因此 GPU/XPU slice 必须先拷到 `pinned_buffer_pool_`（`file_storage.cpp:418-453`）。
3. **错误分级**：`INVALID_READ` 与 `OBJECT_ALREADY_EXISTS` 被视为“本 bucket 软失败”，继续处理后续 bucket；`KEYS_ULTRA_LIMIT` 会把 `enable_offloading_ = false` 并中止本轮（`file_storage.cpp:302-305`、`521-547`）。这个分级修复过 issue #2827（注释里写了）。
4. **失败上报哨兵**：`failed_metadatas` 全部填 `StorageObjectMetadata{-1,0,0,-1,""}`（`file_storage.cpp:558-571`），master 看到 `data_size=-1` 就知道要清理 offloading 任务并释放源副本引用。不这么做，任务要等 TTL reaper 才回收。
5. **重复 key 是幂等 no-op**：`PrepareEviction` 对已在 `object_bucket_map_` 或 `pending_write_keys_` 的 key 放进 `skipped_keys`（`storage_backend.cpp:2818-2841`），提交阶段跳过；但**已被 `pending_eviction_keys_` 覆盖的 key 会返回 `OBJECT_ALREADY_EXISTS`**，因为此时 skip 可能丢掉唯一副本。如果整个 bucket 的 key 都已在盘上，`BatchOffload` 会 `CleanupOrphanedBucket` 删掉刚写的冗余文件并直接返回 `bucket_id`，**不调用 `complete_handler`**（`storage_backend.cpp:1628-1635`）。

Bucket 后端的 `BatchOffload` 内部顺序（`storage_backend.cpp:1514-1684`）值得逐行读：

1. `BuildBucket` 把 `[key bytes][value bytes][key bytes][value bytes]...` 拼成 iovec 数组，同时算出每个对象的 `offset/key_size/data_size`（`storage_backend.cpp:2407-2437`）。
2. `PrepareEviction(required_size, bucket->keys)`：独占锁下按 FIFO/LRU 选旧桶，从 `buckets_/object_bucket_map_` 摘除，累加 `pending_eviction_size_`，返回 `PendingEviction`。**此时不删任何文件**。
3. `eviction_handler(pending.keys)` 通知 master；失败则 `RestorePreparedEviction` 原样放回。
4. `CommitPreparedEviction` 清掉 `pending_eviction_keys_`；`FinalizeEviction` 才真正删文件：**先删 `.meta`，再等 `inflight_reads_` 归零，最后删 `.bucket`**（`storage_backend.cpp:3088-3158`）。先删 meta 是刻意的：即使后面删数据失败，重启 `Init` 也不会把这个已被 master 摘除的桶恢复出来。
5. `WriteBucket`：写数据 → `datasync()` → 写 `.meta`（`store_bucket_metadata`）。`datasync` 失败会 `CleanupOrphanedBucket` 并返回错误（`:2537-2553`）。
6. 提交本地索引（独占锁）：再次过滤重复 key，插入 `object_bucket_map_`、更新 `total_size_`、插 `buckets_` 与 `lru_index_`（`:1597-1659`）。**先提交索引再通知 master**，这样 master 一旦把读请求转过来，本地一定能命中。
7. `complete_handler` 失败 → `RollbackCommittedBucket`：摘索引 → 自旋等 in-flight 读 → 删文件（`:2622-2713`）。超时则**故意不删文件**，留给下次 `Init` 的孤儿扫描清理。

### 流程 C：一次 SSD 读（远程 client → 盘 → RDMA）

```text
请求方 Client::Get
 ├─ BatchGet 查 master，得到 LOCAL_DISK(rpc_addr=owner)
 ├─ (不同进程) RPC batch_get_offload_object → owner RealClient  (real_client.cpp:8694-8720)
 │    └─ FileStorage::BatchGet(keys, sizes)       // file_storage.cpp:267-287
 │         ├─ LoadBatch(prefer_pinned=false)
 │         │    ├─ AllocateBatch(client_buffer_allocator_)   // 每 key 一条 BufferHandle
 │         │    └─ storage_backend_->BatchLoad(allocated_batch->slices)
 │         ├─ 用 slice.ptr 回填 pointers（O_DIRECT 会对齐偏移修正）
 │         └─ 把 AllocatedBatch 放进 client_buffer_allocated_batches_[batch_id]
 ├─ 响应 {batch_id, pointers[], TE addr, gc_ttl_ms}
 ├─ Transfer Engine 从 ClientBuffer 拉数据
 └─ release_offload_buffer(batch_id) → FileStorage::ReleaseBuffer   // file_storage.cpp:1202-1214
```

- 同进程 GPU 恢复可走 `BatchGetLocal` + `pinned_restore_arena_`（`file_storage.cpp:289-300`、`1092-1167`），避免再经过一次远程 TE。但 `use_uring` 打开时 pinned arena 会被禁用（`file_storage.cpp:56-77`，代码只打印 “Pinned SSD restore is disabled with io_uring”）；推测原因是 O_DIRECT 读要求缓冲区与长度都 4 KiB 对齐，而 arena 的分配策略不保证这一点。
- `AllocateBatch` 在 `use_uring` 下会为每个对象多分配 `2*4096` 并把指针向上对齐（`file_storage.cpp:1120-1155`），这是给 O_DIRECT 的“头尾各一个页”留余量。
- 租约 GC：`client_buffer_allocated_batches_` 每条有 `lease_timeout = now + client_buffer_gc_ttl_ms`；`ClientBufferGCThreadFunc` 周期清理（`file_storage.cpp:1169-1192`），正常路径靠 `ReleaseBuffer` 主动释放。

Bucket 的 `BatchLoad`（`storage_backend.cpp:1703-1891`）是并发正确性最密集的读路径：

1. 共享锁下为每个 key 查 `object_bucket_map_` → `buckets_`，校验 `data_size == slice.size`，把读计划按 bucket 分组，并为每个 bucket 创建一个 `BucketReadGuard`（`:1722-1771`）。同时更新 `last_access_ns_`。
2. **释放锁**，按 bucket 打开文件做 I/O；guard 在整个 I/O 期间保证 `inflight_reads_ > 0`。
3. 如果是 `UringFile`（即 `use_uring` 且读模式），走 `batch_read`：对每个 key 计算 `actual_offset = offset + key_size`，向下/向上按 4 KiB 对齐成 `aligned_offset/aligned_size`，每次提交最多 `QUEUE_DEPTH = 32` 条独立读（`uring_file.cpp:53`、`:151-190`）；成功后把 `batch_object[key].ptr` 调整为 `dst + offset_in_buffer`（`:1794-1867`）。这里真正解决的是**文件偏移与读长度**的页对齐，而不是缓冲区对齐：`dst` 本身仍必须 4 KiB 对齐（由 `AllocateBatch` 保证，`UringFile::batch_read` 会拒绝未对齐的 buf/len/off，`uring_file.cpp:774-786`）。
4. 否则逐条 `vector_read`，要求读到恰好 `data_size` 字节。

注意这个读批次的**失败粒度是整批**：任何一个 key 在索引里找不到（`INVALID_KEY`）、所属 bucket 找不到（`BUCKET_NOT_FOUND`）或 `data_size` 与调用方 `Slice.size` 不符（`INVALID_PARAMS`），`BatchLoad` 都会立刻返回错误，已经读出的 key 也一起作废（`storage_backend.cpp:1722-1771`）。调用方必须按整个 batch 重试或降级。

### 流程 D：promotion（L2 盘 → L1 DRAM）与水位驱逐

`ProcessPromotionTasks`（`file_storage.cpp:868-1016`）是 offload 的反向流水线：

1. `client_->PromotionObjectHeartbeat(tasks)` 拉任务；master 每次最多返回一条（注释 `:898-903` 解释这是为了不超心跳存活窗口）。
2. `PromotionAllocStart` 在 master 侧分配一个 PROCESSING 的 MEMORY 副本。失败（通常 `NO_AVAILABLE_HANDLE`）要立刻 `NotifyPromotionFailure` 释放 admission 名额，否则名额被 TTL reaper 占满（`:915-936`）。
3. `AllocateBatch` 从 `client_buffer_allocator_` 拿 staging → `BatchLoad` 从盘读 → `PromotionWrite(memory_descriptor, tx_slices)` 用 TE 写入 DRAM 副本 → `NotifyPromotionSuccess` 提交。**任意一步失败都要 `NotifyPromotionFailure`**，注释明确说这是为了避免瞬时抖动把 `promotion_queue_limit_` 占满（`:938-952`）。
4. `RunDiskWatermarkEviction`（`file_storage.cpp:628-647`）与写路径驱逐复用同一套 `EvictAboveDiskWatermark`：bucket 后端把 `synthetic_required_size = max_total_size - target` 传给 `PrepareEviction`（`storage_backend.cpp:3204-3211`），file-per-key 后端则从 FIFO 队列直接选文件（`storage_backend.cpp:1022-1120`）。

### 流程 E：DFS 描述符数据面（控制面/数据面分离）

```text
PutStart (master_service.cpp:5589)
  └─ AllocateAndInsertMetadata (:5404) → AllocateReplicas (:5140)
       └─ dfs_allocator_->Allocate(key, value_length) → DistributedFSDescriptor   // :5300-5317
            └─ 作为 PROCESSING 的 DFS replica 写进元数据，随 PutStart 响应下发
Put 数据面 (client_service.cpp:2900-3010)
  ├─ 设备 slice D2H staging 到 pinned buffer
  └─ DistributedStorageBackend::BatchWrite(DfsWriteRequest[])   // distributed_storage_backend.cpp:633-720
       ├─ 校验 descriptor 范围（IsDfsDescriptorRangeValid / IsBucketDescriptorRangeValid）
       ├─ 校验 sum(slice.size) == descriptor.object_size
       ├─ adapter->SupportsBatchIo() ? 收集后批量提交 : ExecuteSingleIo
       └─ shard 模式：缓存 fd；bucket 模式：每批最多打开 64 个 bucket 文件
PutEnd (master_service.cpp:5725-5743)
  └─ bucket 模式额外 MarkCommitted(key, descriptor) 把 PENDING 翻成 COMMITTED
```

- **shard 模式**（`ShardAllocator`）：每个 shard 是预分配的 `dfs_shard_XX.data`，`Allocate` 先按 `hash(key) % shard_count` 选 shard，满了再线性探测其它 shard（`shard_allocator.cpp:255-300`）。`alloc_size = aligned_size + alignment - 1`，保证任意偏移都能向上对齐到 4 KiB。`Free` 不立即还空间，而是进 `pending_free` 延迟队列，默认 30 秒后才真正释放（`shard_allocator.cpp:398-401`、`588-611`），这是给“读请求手里还握着旧 descriptor”留的安全窗口。
- **bucket 模式**（`ImmutableBucketAllocator`）：只追加，不回收单条；条目布局就是 `[value][zero pad]`，没有头、没有校验（`bucket_entry_layout.h:12-19`）。`ReserveInBucketLocked` 永不回退 `append_offset`，即使 key 后来变成 TOMBSTONE（`immutable_bucket_allocator.cpp:186-216`）。驱逐是整桶删除：`PrepareEviction` 只选 `SEALED` 且 `pending_entries==0` 的桶，翻成 `FROZEN`；master 在**稳定顺序锁住所有相关元数据 shard** 后逐条校验 descriptor/lease/pin，全过才 `CommitEvictionLogical`（`master_service.cpp:8410-8567`）；物理删除失败则留在 `failed_deletions_` 等重试（`immutable_bucket_allocator.cpp:541-593`）。另外，`PutStart` 遇到 DFS 分配失败时会先重新 admit，再调用 `TryRecoverDfsSpaceAfterAllocationFailure()` **强制驱逐一个整桶**（`PrepareEvictionForAllocationFailure`，`manager` 侧见 `master_service.cpp:5576-5587`、`8400-8425`），成功后重试 admit；这是“盘满”时 bucket 模式的唯一在线回收路径。
- **对象存储模式**：`BatchOffload` 直接 `PutBatch` 整个逻辑对象（`:359-444`），`BatchLoad` 直接 `GetBatch`（`:815-845`），`ScanMeta` 用 `ListKeys`。它**不支持驱逐**（收到 `eviction_handler` 只打一次 warning，`:372-376`）。

### 流程 F：NVMe KV 的写与读

写（`nvme_kv/backend.cpp:479-973`）：

1. `BuildPayloadView` 把多 slice 拼成连续 payload（单 slice 零拷贝）。
2. 配额检查（keys/size）→ `IsEnableOffloading`。
3. 对每个对象 `BuildWritePlan(identity, payload, slot=0, max_value_size)`（`key_conflict_policy.cpp:32-123`）：payload 能塞进一个 value 就 `kInline`；否则按 `max_value_size` 切 chunk，再生成一个 `kManifest` 根对象。
4. 三级流水线：prepare（`prepare_concurrency_`）→ chunk 提交（`batch_submit_concurrency_` 个 lane，每个 lane 一个 `IndexQueue`）→ 根对象提交（`root_submit_concurrency_` 个 consumer，`root_queue`）。根对象**最后写**，它相当于可见性标记（`:628-830`）。
5. 失败回退：任何一个 chunk 失败会标记 `needs_fallback`，清理已写的 chunk/root，再逐对象走 `OffloadOne` 的“换 slot 重试”路径（`:832-916`、`975-1165`）。
6. `BuildWritePlan` 的 `slot` 与 `ResolveExistingObject` 配合实现 **store-if-not-exists + 冲突槽**：物理 key 已存在且内容相同 → 幂等成功；内容不同 → `OffloadOne` 换下一个 slot 重试，最多 `kNvmeKvMaxPhysicalKeySlots = 64` 个（常量在 `key_codec.h:18`，循环在 `backend.cpp:995`；`ResolveExistingObject` 在 `key_conflict_policy.cpp:125-145`）。

读（`backend.cpp:1167-1485`）：

1. `ResolveRoot` 逐个 slot 试 `Retrieve`，解析 header/identity，校验 `logical_key` 与 `resolved_physical_key/resolved_slot` 一致（`:147-184`）。
2. inline 对象直接校验 checksum 后 memcpy；manifest 对象解析 chunk 记录并缓存到 `manifest_cache_`。
3. chunk 读分两种：目标地址满足 DMA 对齐 → `RetrieveIntoBatch` 直接落盘；否则 `RetrieveBufferBatch` 再 memcpy（`:1336-1422`）。
4. `RunPipelinedIo` 让“读下一组”和“校验上一组”重叠（`:1433-1469`）。

---

## 与其他模块的边界

### 依赖谁

| 依赖 | 用途 | 证据 |
|---|---|---|
| `liburing` | `UringFile`、NVMe KV io_uring executor | `uring_file.cpp:18`、`file_interface.h:13`、`nvme_kv/executor_io_uring.cpp:9` |
| `offset_allocator` | offset-allocator 后端与 `ShardAllocator` 的空间分配 | `storage_backend.h:23`、`shard_allocator.h:17` |
| `yalantinglibs` | 元数据 `struct_pb` 序列化、`tl::expected` | `storage_backend.cpp:25`、`storage_backend.h:38` |
| `libcurl + OpenSSL` | OSS/S3 REST 引擎（HMAC 签名、curl_multi） | `rest_object_storage_adapter.cpp:253-254`、`:403-425`（curl）；HMAC 签名在 `storage/distributed/object_storage_signing.h:8,50-56`、`oss_adapter.cpp:431-442` |
| `3FS hf3fs_usrbio` | `Hf3fsAdapter` 数据面 | `hf3fs_adapter.cpp:15`、`hf3fs.h:8` |
| `SPDK + DPDK` | NoF 远程 SSD 池、SPDK 内存分配/注册 | `spdk_wrapper.h:83-108`、`src/common/client_buffer_allocation.cpp:128-131` |
| master 控制面 | `MountLocalDiskSegment`、`OffloadObjectHeartbeat`、`PromotionObjectHeartbeat`、`NotifyOffloadSuccess`、`BatchEvictDiskReplica`、`ReportSsdCapacity`、`PollRemoveAll` | `file_storage.cpp` 全文 |
| Transfer Engine | SSD→DRAM 的 promotion 写、SSD→远端读 | `file_storage.cpp:986-988` |

### 被谁依赖

- `RealClient`：创建/持有 `FileStorage`，并在析构前调 `DrainLocalDiskSegment`（`real_client.cpp:1484-1518`、`2362-2375`）。
- `Client`：通过 `SetDfsStorageBackend` 拿到 DFS 后端，在 Put/Get 路径直接 `BatchWrite/BatchRead`（`client_service.cpp:4275`、`2988`、`1798`）。
- `MasterService`：持有 `LocalSsdManager` 与 `DfsAllocatorInterface`，负责调度 offload/promotion/DFS 驱逐（`master_service.h:2156-2166`）。
- 监控：`SsdMetric`（`file_storage.cpp:470-498`、`1030-1043`）。

### 接口契约在哪

- client ↔ backend：`StorageBackendInterface`（`storage_backend.h:262-318`）。
- client ↔ master：`Client` 的 RPC 方法（`master_client.h`），语义由 `FileStorage` 的调用顺序定义。
- backend ↔ 盘：`StorageFile`（`file_interface.h:57-163`）与 `ObjectStorageAdapter`（`object_storage_adapter.h:41-104`）。
- master ↔ DFS 存储：`DfsAllocatorInterface`（`dfs_allocator_interface.h:45-79`）+ `DistributedFSDescriptor`（随副本描述符下发）。
- 数据怎么流过去：`StorageObjectMetadata` / `DistributedFSDescriptor` 都通过 `YLT_REFL` 序列化进副本描述符（`types.h:545`、`replica.h:270`）。

---

## 设计取舍与易踩坑点

1. **`master_store_backend.cpp` 与磁盘无关**。它实现的是 `WeightStoreBackend`（接口在 `mooncake-store/include/weight_store_backend.h:18-34`，`MasterStoreBackend final : public WeightStoreBackend` 在 `:38-58`），负责把 oplog 持久化回调与权重组快照桥接到 `MasterService`（`master_store_backend.cpp:9-78`）。不要把它归入存储后端。

2. **SPDK 不在本地 offload 路径上**。`SpdkWrapper` 服务的是 NVMe-oF 远程 SSD 池（`OpenNofSegment`/`SubmitRequest`，`spdk_wrapper.cpp:370`、`450`）以及 SPDK 内存分配/注册（`src/common/client_buffer_allocation.cpp:128-131`、`real_client.cpp:1023`）。本地落盘的文件路径只有 POSIX/io_uring（file-per-key、bucket、offset-allocator），另有 `nvme_kv/` 直连本地 NVMe 设备的独立路径；两者都不经过 SPDK NoF。`local_ssd/` 同理是 **master 侧** 的任务邮箱与容量账本（`local_ssd/manager.h:21-151`），不是 client 写盘代码。

3. **O_DIRECT 只用于读，不用于写**。file-per-key 在 `create_file` 里只对读加 `O_DIRECT`（`storage_backend.cpp:714-721`）；bucket 的 `OpenFile` 同样只对读返回 `UringFile`（`:3464-3484`）。由此产生一个**实际不可达的分支**：`WriteBucket` 里 `dynamic_cast<UringFile*>` 后的 `write_aligned` 路径（`:2457-2519`）在当前代码中永远进不去，因为写模式返回的是 `PosixFile`。读代码时不要据此认为 bucket 写用了 O_DIRECT。反过来，offset-allocator 后端的数据文件在 `use_uring` 时是 **O_DIRECT 读写都开**（`:3813-3841`、`4593-4638`），这与前两者不同。

4. **io_uring 是“每线程一个 ring”，不是每文件也不是全局**。`SharedUringRing::instance()` 是 `thread_local`（`uring_file.cpp:56-59`），注释解释了旧全局 ring 的锁竞争 >1ms、而 `fdget()` 只有 ~50ns，所以故意不用 `IOSQE_FIXED_FILE`（`:37-50`）。缓冲区注册却是进程级 `g_buf`（`:31-35`），每个线程 ring 在首次 I/O 时懒注册（`:75-100`）。注册前还会 `madvise(MADV_NOHUGEPAGE)`，避免 THP 下 `FOLL_LONGTERM` pin 失败（`:985-996`）。

5. **`batch_read` 的对齐语义容易被误读**。`use_direct_io_` 时 `UringFile::batch_read` 会校验 `buf`、`len`、`off` 三者都按 4 KiB 对齐，任一不满足直接返回 `FILE_INVALID_BUFFER`（`uring_file.cpp:774-786`）；因此调用方传入的 `dest_slice.ptr` **必须** 4 KiB 对齐——`FileStorage::AllocateBatch` 在 `use_uring` 下正是为此多分配 `2*4096` 并把指针向上取整（`file_storage.cpp:1119-1155`）。`BucketStorageBackend::BatchLoad` 做的是另一件事：把**文件偏移**向下对齐、长度向上取整，从对齐的偏移读到缓冲区起始处，再把 `slice.ptr` 往后挪 `offset_in_buffer`（`storage_backend.cpp:1808-1865`）。所以读完后 `Slice.ptr` 和调用前不是同一个值，`FileStorage::LoadBatch` 专门回填了 `pointers`（`file_storage.cpp:253-262`）。

6. **两阶段驱逐：删除权在 master**。bucket 与 offset-allocator 都遵循“先摘索引 → 通知 master → master 接受后才释放物理空间”。bucket 在 master 拒绝时 `RestorePreparedEviction` 原样放回（`storage_backend.cpp:3002-3045`）；offset-allocator 在 `NotifyAndCommitPreparedEviction` 里回滚（`:4801-4834`）。注意 offset-allocator 的驱逐只有 master 接受后才写 tombstone，否则恢复会误删仍然有效的 key（`:4745-4751` 注释）。

7. **`total_size_` 不是物理磁盘用量**。bucket 的 `total_size_` 是逻辑计数：一个 256 MiB 的 bucket 文件只有在**桶内所有对象**都被驱逐后才删除，所以按对象驱逐时它会显著低于真实占盘。为此 `max_physical_bytes` + `ActualDiskBytesUsedLocked()` 用 `stat.st_blocks * 512` 实测（`storage_backend.cpp:2762-2810`），口径与 `du`/kubelet emptyDir 一致；扫描失败时**fail-closed**（报告已达上限，不缓存），避免静默放开上限。结果按 `disk_scan_cache_ms` 缓存，`FinalizeEviction` 后失效（`:3165-3170`）。

8. **DFS 模式的 `FileStorage::Init` 提前返回**（`file_storage.cpp:142-147`）。因此 DFS 下不会 mount LOCAL_DISK 段、不跑心跳、不 `ScanMeta`。DFS 的 `BatchLoad/IsExist/ScanMeta` 都是 `NOT_SUPPORTED`（`distributed_storage_backend.cpp:815-873`），`IsEnableOffloading()` 返回 `UsesObjectStorage()`，对 DFS 即 false（`:859-861`）。**DFS 的读必须带 descriptor 走 `BatchRead`**，不能按 key 读。

9. **对象存储仍是 `LOCAL_DISK` 副本**。master 只知道“某个 real client 拥有它”，读请求经该 owner 转发（`docs/source/deployment/object-storage-offload.md` 的 Topology 一节）。`MOONCAKE_OFFLOAD_FILE_STORAGE_PATH` 仍是必填的本地目录，但不做缓存。对象存储后端也不参与驱逐（`distributed_storage_backend.cpp:372-376`），容量回收要靠上层/外部生命周期策略。

10. **`ImmutableBucketAllocator` 不做任何持久化元数据**。它的正确性前提是“进程重启后根目录为空”，所以 `Init` 见到任何 bucket 文件就拒绝（`immutable_bucket_allocator.cpp:96-109`）。系统整体也要求 DFS 与 `enable_oplog` 互斥（`master_service.cpp:503-508`）。把它当成“可恢复的持久化层”会错得很离谱。

11. **`ShardAllocator` 的延迟释放是给读者留的安全窗口**。`Free` 只把 handle 放进 `pending_free`，30 秒后才真正还给 allocator（`shard_allocator.cpp:398-401`、`588-611`）；代码没有写明理由，推测是为“读请求手里还握着旧 descriptor”留窗口。`EffectiveUsage` 会把 `pending_free` 里的字节算成“即将可用”（`:613-635`），因此水位判断用的是偏低的逻辑用量：与只看物理 free 相比，驱逐会更晚、更少触发；但这些字节在 `deferred_free_duration` 到期前并不能真正分配，极端情况下会出现“水位没到、分配已失败”。调小 `deferred_free_duration` 会提高空间利用率，但可能让仍在飞的 RDMA 读读到已被复用的区域。

12. **offset-allocator 的持久化是“尽力而为的缓存恢复”**。元数据 checkpoint 与数据写分离，恢复时 `seq >= checkpoint.insert_seq` 一律丢弃（`storage_backend.cpp:4258-4272`）；CRC 是每条可选（`enable_record_crc`），DMA 写入的记录可以不带 CRC；旧版本元数据直接拒绝。`kTransientError` 与 `kCorrupt` 的区分是设计精髓：资源类错误**绝不能**触发清盘（`:3760-3782`）。

13. **`BucketStorageBackend` 的 `BatchLoad` 其实没有用 `file_cache_`**。`GetOrOpenFile` 定义了但无调用点（全仓 grep 只有 `storage_backend.cpp:3488`），`BatchLoad` 走 `OpenFile`（`:1786`）。`file_cache_` 只在 `ClearFileCache`/驱逐/清理时被 erase。把它当成“热桶 fd 缓存”会误判性能特征。

14. **`OffsetAllocatorStorageBackend` 有单写者前提**。`BatchOffload` 顶部的注释明确说：`total_size_/total_keys_` 的读-改-写不是原子的，当前由 `FileStorage` 的单心跳线程保证；若将来引入并发 offload，`EvictToMakeRoom` 里的 `DCHECK_GE` 必须重新评估（`storage_backend.cpp:4844-4854`）。不要在这个后端外面加并发调用。

15. **`StorageBackendAdaptor`（file-per-key）的读写都经过 `struct_pb`**。写路径把 `KVEntry{key,value}` 序列化后整文件写（`storage_backend.cpp:1246-1254`）；读路径先按目标大小构造一个“等长占位串”再读回来（`:1318-1335`）。这意味着它比 bucket 多一次完整内存拷贝，且 `BatchLoad` 是逐 key 串行的，没有批量提交。它适合小规模/可读性优先，不适合百万对象。

16. **bucket 的 `IsExist` 不只看索引**。它还会 `fs::exists` 检查对应 `.bucket` 文件（`storage_backend.cpp:2171-2192`），因为写入是“先文件后索引”，索引命中但文件被外部抹掉时必须回答 false。这正是 `Client::Put` 用来自愈 dangling `LOCAL_DISK` 副本的探测（`real_client.cpp:1504-1518`）。

17. **pinned restore arena 与 io_uring 互斥**。`FileStorage` 构造时若 `use_uring` 打开会打印 “Pinned SSD restore is disabled with io_uring” 并跳过 arena 分配（`file_storage.cpp:56-77`）。依赖 `HasPinnedRestoreArena()` 的同进程 GPU 恢复路径会因此退化到普通路径（`real_client.cpp:8876-8923`），这是配置组合上的隐式约束。

18. **NVMe KV 的 `ScanMeta` 是空实现**（`nvme_kv/backend.cpp:1516-1525`）。也就是说重启后它**不会**向 master 重注册已有对象；`total_size_/total_keys_` 也从 0 开始（`Init` 里显式 store 0，`:473-474`）。这是与 bucket/file-per-key 的重要差异，做容量规划时不能假设“盘上已有数据会被恢复”。

19. **NVMe KV 的 16 字节 key 是哈希，不是路径**。冲突处理靠“最多 64 个 slot + 盘上存 logical key 校验 identity”（`key_codec.h:12-31`、`key_conflict_policy.cpp:23-30`）。因此 `IsExist` 需要逐个 slot `Retrieve` 探测（`backend.cpp:1487-1501`），比本地索引后端贵得多。

20. **优雅下线是锁协议，不是简单置位**。`DrainLocalDiskSegment` 在 `offloading_mutex_` 下先置 `draining_` 再 `UnmountLocalDiskSegment`（`file_storage.cpp:829-866`）。`Heartbeat` 在入口和拿到锁后各检查一次（`:674-676`、`709-711`），因为一个已通过入口检查的 tick 可能正阻塞在这把锁上；等它拿到锁时若不复查，就会因 `SEGMENT_NOT_FOUND` 重新 mount 已摘除的段。后台 `ReRegisterOffloadedObjects` 也在每个批次检查 `draining_` 并主动中止（`:1240-1244`）。

21. **bucket 的 owner lock 与 pid salt 是两层防御**。`Init` 用 `flock(LOCK_EX|LOCK_NB)` 保证一个目录一个活 client（`storage_backend.cpp:1914-1938`）；`BucketIdGenerator` 再把 `getpid() & 4095` 混进起始 id，缩小“同秒启动”的碰撞窗口（`:1460-1470`）。`OpenFile` 写模式对不存在的路径用 `O_EXCL`，对已存在路径才 `O_TRUNC`（`:3452-3461`）。三者缺一不可，不能只依赖 salt。

22. **此处未完全确认**：master 侧 DFS 驱逐的完整接受/回滚分支（`master_service.cpp:8410-8710`）我只精读到“稳定锁序 + 逐条校验 + 接受后提交”这一层；其中 `RunShardDfsEviction` 对 `accepted` 向量的构造（`:8597-8708`）没有逐行核对。若要做 DFS 驱逐相关的修改，请单独精读这段。

---

## 精读清单

建议按下面顺序读，每一步都带着一个具体问题：

1. **先读设计文档**：`docs/source/design/store/ssd-offload.md`（架构图 + 数据流 + 三种本地后端对比），再看 `docs/source/deployment/ssd/ssd-offload.md` 的环境变量表。目标：建立“心跳驱动、master 点名、client 落盘、读经 owner 转发”的全局图。
2. **`mooncake-store/include/storage_backend.h:262-318`**：把 `StorageBackendInterface` 六个纯虚方法的语义抄一遍，特别是 `complete_handler` 与 `EvictionHandler` 的失败语义。然后对照 `file_storage.cpp:342-367` 和 `579-626` 看它们各自的实现。
3. **`file_storage.cpp:662-812`（Heartbeat）**：画出 STEP 1/2/3 的顺序，标注 `offloading_mutex_` 的持有范围、`SEGMENT_NOT_FOUND` 的重挂载与异步 rescan。再读 `829-866`（Drain），理解为什么需要两处 `draining_` 检查。
4. **`file_storage.cpp:307-577`（OffloadObjects）**：重点看分桶、D2H staging、错误分级、`failed_tasks` 哨兵。问自己：一个 key 从 master 队列被取出，到最终 `NotifyOffloadSuccess` 或 NACK，所有分支是否都覆盖了？
5. **`storage_backend.cpp:1514-1684`（Bucket BatchOffload）**：按“BuildBucket → PrepareEviction → notify → Commit → Finalize → WriteBucket → 提交索引 → complete_handler”逐步读，每步记下失败路径。再读 `2812-3000`（PrepareEviction）与 `3080-3176`（FinalizeEviction），确认 `pending_*` 计数与 in-flight 读的配合。
6. **`storage_backend.cpp:1703-1891`（Bucket BatchLoad）**：先读非 uring 路径，再读 uring 的 `batch_read` 对齐计算。重点确认 `BucketReadGuard` 的生命周期覆盖整个 I/O 阶段。
7. **`storage_backend.cpp:1906-2169`（Bucket Init）**：列出所有“删掉这个 bucket”的条件，理解孤儿 `.bucket` 清理与 owner lock。
8. **`storage_backend.cpp:3642-3913` + `4499-4678` + `4161-4493`（OffsetAllocator）**：先看 `RecoveryResult` 四态，再看 `RebuildShardMapsFromAllocator` 的 seq 守卫与 CRC 校验，最后看 `RestoreAndRepairFifoIndex` 的三阶段。这是持久化设计的精华。
9. **`storage_backend.cpp:4838-5258`（OffsetAllocator BatchOffload）**：按 (A) 水位驱逐 → (B) 通知 master → (C) allocate → (D) fallback 驱逐 → (E) 写记录 → (F) 更新索引 + FIFO → 持久化屏障 的顺序读。注意 `eviction_mutex_` 先于 shard 锁的顺序约束。
10. **`uring_file.cpp:24-450`**：先读 `SharedUringRing` 的设计注释，再读 `ensure_buf_registered`、`batch_read`、`submit_rw/collect`。最后读 `979-1055` 的全局注册。目标：说清楚“每线程 ring + 进程级 buffer + 懒注册 + 不用 FIXED_FILE”这四个决策。
11. **`posix_file.cpp` 全文（149 行）**：很短，重点看析构里的 unlink 与 `vector_write` 的“必须一次写满”语义。
12. **`distributed_storage_backend.cpp:174-214`（Init）+ `359-444`（对象存储 BatchOffload）+ `633-813`（DFS BatchWrite/BatchRead）+ `517-631`（批量执行）**：理解两种模式的装配与“控制面分配 descriptor、数据面直写 DFS”的分工。对照 `master_service.cpp:5300-5317`（Allocate）与 `5725-5743`（bucket MarkCommitted）。
13. **`immutable_bucket_allocator.cpp:69-123`（拒绝非空根）+ `186-216`（只追加）+ `415-561`（整桶驱逐）**：理解双状态机与“运行时态不落盘”的取舍。再读 `docs/source/design/store/immutable-dfs-bucket-allocator.md`。
14. **`shard_allocator.cpp:39-160`（Init/恢复容量）+ `255-300`（Allocate）+ `433-577`（Prepare/Resolve 驱逐）+ `588-635`（延迟释放）**：理解 shard 布局、线性探测、LRU 与 pending_free。
15. **`nvme_kv/backend.cpp:479-973`（BatchOffload 三级流水线）+ `975-1165`（OffloadOne 冲突槽）+ `1167-1485`（BatchLoad）**：对照 `key_conflict_policy.cpp:32-143` 与 `key_codec.h`。重点：store-if-not-exists、manifest 可见性、slot 冲突。
16. **`local_ssd/manager.cpp:13-115`（mailbox）+ `117-197`（容量/生命周期）**：站在 master 视角理解 offload/promotion 的调度队列与 HA 快照。
17. **最后读 `storage_backend.cpp:5578-5661`（CreateStorageBackend）与 `file_storage_config.cpp:47-144`**：把环境变量到具体实现的映射表补齐，作为部署时的查表依据。

读完后应该能回答四个问题：(a) 一个 key 从内存到盘、再回到另一个进程的内存，中间经过哪些进程/线程/锁？(b) 盘上文件被删除的唯一合法触发链是什么？(c) 崩溃在“写数据后、写元数据前”和“写元数据后、通知 master 前”分别由谁兜底？(d) 五种后端各自牺牲了什么换取了什么？

---

# 10 内存分配器体系

> 精读范围：`mooncake-store` 段分配器（CacheLib / OffsetAllocator）、`offset_allocator` 核心算法、客户端 mmap/arena/pinned 缓冲、设备与 GPU fabric 分配器、分配器指标。
> 所有 `路径:行号` 均相对仓库根 `D:/xiaojun/work/Mooncake`；行号以 2026-10 的 main 分支（`67df8c41` 附近）为准。

## 定位与职责

Mooncake Store 的分配器体系回答一个看起来简单、实际很容易读错的问题：**"一个 KV 对象的地址空间从哪来？"**

答案分三层，而这三层的"内存"含义完全不同：

1. **Master 侧的段分配器（segment allocator）**。客户端把一大块自己拥有的内存 `MountSegment` 进集群后，Master 需要在这块内存的地址区间 `[base, base+size)` 里为每个副本切出一个 `AllocatedBuffer`。请注意：**Master 从不拥有、也从不访问这块内存**，它只是在一段 `uint64` 地址区间上做区间分配，把 `(offset, size, protocol, endpoint)` 作为元数据下发给写者。真正的字节永远由客户端进程持有并通过 Transfer Engine 直接搬运（`docs/source/design/store/mooncake-store.md` 的 "Master Service does not take over any data flow"）。
   这里有两个可替换实现（`mooncake-store/include/types.h:521` 的 `BufferAllocatorType`）：
   - `CachelibBufferAllocator`：vendored 的 Facebook CacheLib slab 分配器子集，16MB slab + size-class；
   - `OffsetBufferAllocator`：移植自 sebbbi/OffsetAllocator 的 bin 分配器，**默认且推荐**（`mooncake-store/include/master_config.h:275`、`:1255`、`:885` 三处默认值都是 `OFFSET`）。

2. **客户端侧的本地缓冲分配器**。客户端要一块大缓冲来承接 put/get 的数据，这块缓冲是**真实内存**：`allocate_buffer_allocator_memory`（`mooncake-store/src/common/client_buffer_allocation.cpp:97`）按 `protocol` 分发到 CUDA/Ascend/Sunrise/UB/SPDK/aligned_alloc，hugepage 路径还可以走 `MmapArena`。拿到真实内存后，再用**同一个** `OffsetAllocator` 在其上做子分配（`ClientBufferAllocator`，`mooncake-store/src/client_buffer.cpp:64`）。另有 `RegisteredPinnedMemoryManager` 负责带预算的 `cudaHostRegister`。

3. **设备 / fabric / 指标层**。`mooncake-store/include/device/` 提供跨厂商 `AcceleratorDevice` 抽象；`mooncake-integration/allocator.py` 与 `mooncake-transfer-engine/nvlink-allocator/nvlink_allocator.cpp` 一起把 GPU fabric 内存（`cuMemCreate` + `cuMemMap` 的 VMM 路径）注册成 torch 的 pluggable allocator；`allocator_metric.cpp` 导出的是**进程 RSS 与 jemalloc 统计**，与上面两层的 BufferAllocator 没有关系。

一张图概括（箭头表示"谁持有谁"）：

```
                    ┌──────────────────────── Master 进程 ────────────────────────┐
                    │                                                            │
 客户端 mount ─────►│  SegmentManager::mounted_segments_[id].buf_allocator        │
 (base,size,proto)  │     ├── OffsetBufferAllocator  ──► offset_allocator::OffsetAllocator
                    │     └── CachelibBufferAllocator ──► facebook::cachelib::MemoryAllocator
                    │              （两者都只切 [base, base+size) 的地址，不碰内存）│
                    │         ▲ AllocatorManager + AllocationStrategy 选段         │
                    │         └── AllocatedBuffer（weak_ptr 回指 allocator，RAII 归还）│
                    └────────────────────────────────────────────────────────────┘
                                          │ Descriptor{size,addr,protocol,endpoint}
                                          ▼
                    ┌──────────────── 客户端进程（真实内存在这里）───────────────┐
                    │ allocate_buffer_allocator_memory / allocate_buffer_mmap_memory│
                    │      └── ClientBufferAllocator ──► OffsetAllocator（子分配）  │
                    │ RegisteredPinnedMemoryManager（cudaHostRegister 预算管理）    │
                    │ device::AcceleratorDevice / nvlink_allocator.so（GPU fabric） │
                    └────────────────────────────────────────────────────────────┘
```

本子系统最容易产生的三个误解，先在这里点名，正文会逐一展开：

- `OffsetAllocator` **从不碰真实内存**，它管理的只是地址/偏移；同一个类被复用在四个语义完全不同的地方（Master 段地址、客户端缓冲子分配、本地 SSD 文件内偏移、DFS 页偏移）。读代码前必须先确认 `base` 指向什么。
- `allocator_metric.cpp` / `allocator_status.h` 与 BufferAllocator 无关，导出的是进程 RSS 和 jemalloc 内部统计。
- 段容量超过 3.75GB 时，`OffsetAllocator` 会引入 `m_multiplier_bits`，所有 offset/size 以 quantum 为单位存储；把 `OffsetAllocation::getOffset()` 当字节偏移是最常见的误读。

## 目录结构速览

| 文件 | 一句话职责 |
|---|---|
| `mooncake-store/include/allocator.h` | 段分配器契约总入口：`BufferAllocatorBase`、`AllocatedBuffer`、两个实现、`DummyBufferAllocator`、`SimpleAllocator`、HA 导入 API |
| `mooncake-store/src/allocator.cpp` | 两实现的构造/分配/释放、CACHELIB 布局校验、每段 `MasterMetricManager` 记账、`ImportOffsetBufferAllocator` 的重放式恢复 |
| `mooncake-store/include/offset_allocator/offset_allocator.h` | `OffsetAllocator`（线程安全包装 + multiplier + RAII handle）、`__Allocator`、`OffsetAllocatorSnapshot` 声明与序列化模板 |
| `mooncake-store/src/offset_allocator.cpp` | bin 分配核心：`SmallFloat` size-class、两级位图、free-list、Mooncake 的 round-up-to-bin 修改、快照全图校验 |
| `mooncake-store/include/cachelib_memory_allocator/MemoryAllocator.h` | vendored CacheLib 子集入口：`MemoryAllocator → MemoryPool → AllocationClass → SlabAllocator` 层级，`importAllocations` |
| `mooncake-store/include/cachelib_memory_allocator/Slab.h` | `Slab::kSize = 1<<24 = 16MB`、`kMinAllocSize = 64`、`SlabHeader`（7 字节 packed） |
| `mooncake-store/src/cachelib_memory_allocator/*.cpp` | 独立 STATIC 库 `cachelib_memory_allocator` 的实现（`MemoryAllocator.cpp` / `MemoryPool.cpp` / `MemoryPoolManager.cpp` / `AllocationClass.cpp` / `SlabAllocator.cpp` / `Slab.cpp`） |
| `mooncake-store/include/allocator_metric.h` / `src/allocator_metric.cpp` | `AllocatorMetric`：进程 RSS 系列 + jemalloc 系列的 Prometheus 指标；`JemallocStatsCollector` 函数指针注入 |
| `mooncake-store/include/allocator_status.h` | jemalloc 侧采集器：`LogAllocatorStatus` / `CollectJemallocStats` / `InstallAllocatorStatsCollector`（仅 `STORE_USE_JEMALLOC` 下有内容） |
| `mooncake-store/include/mmap_arena.h` / `src/mmap_arena.cpp` | `MmapArena`：hugepage mmap + lock-free CAS bump 分配，`owns()` 用于区分释放路径 |
| `mooncake-store/src/common/mmap_buffer.cpp` | arena 与直接 mmap 的接线层：`allocate_buffer_mmap_memory` / `free_buffer_mmap_memory` / `is_mmap_arena_allocation`，全局 `g_mmap_arena` |
| `mooncake-store/src/common/client_buffer_allocation.cpp` | `allocate_buffer_allocator_memory` / `free_memory`：按 protocol 分发的总闸口 |
| `mooncake-store/src/registered_pinned_memory.cpp`（头在同目录 `src/`） | `RegisteredPinnedMemoryManager`：预算 + 重叠检测 + `cudaErrorCudartUnloading` 时宁可泄漏 |
| `mooncake-store/include/client_buffer.h` / `src/client_buffer.cpp` | `ClientBufferAllocator`（真实内存 + OffsetAllocator 子分配）+ `BufferHandle`（owning/view 双模式） |
| `mooncake-store/include/pinned_host_buffer.h` | `PinnedHostBuffer`：函数指针 deleter 的 move-only pinned 缓冲 |
| `mooncake-store/include/device/accelerator_device.h` | `AcceleratorVendor` / `PointerInfo` / `AcceleratorDevice` 抽象 |
| `mooncake-store/include/device/{runtime_accelerator,accelerator_registry,cuda_ipc_buffer}.h` | 运行时选设备、静态注册表、CUDA IPC 导出/映射 |
| `mooncake-store/include/allocation_strategy.h` | `AllocatorManager`（segment→registration 注册表）与 `Random/Ranked/FreeRatioFirst/SsdFreeRatioFirst/Cxl` 策略 |
| `mooncake-store/include/segment_allocator_registration.h` + `src/*.cpp` | 一个 segment 的注册项：两条 `SegmentLifetime` + `ClientLivenessRecord`，决定"这个段还能不能分配" |
| `mooncake-store/include/storage_usage.h` | `StorageUsageTracker` / `StorageUsageRegistration`：按存储域聚合用量，供水位/策略使用 |
| `mooncake-store/src/memory_alloc.cpp` | 19 行；`hugepage_memory_alloc` 在非 `USE_NOF` 构建下直接返回 `nullptr` |
| `mooncake-store/src/aligned_client_buffer.cpp` | 4096 对齐版客户端缓冲，仅 `file_storage.cpp` 的 O_DIRECT 路径使用 |
| `mooncake-integration/allocator.py` | `NVLinkAllocator` / `BarexAllocator`：把 `.so` 的 C 符号包成 `torch.cuda.memory.CUDAPluggableAllocator` |
| `mooncake-integration/fabric_allocator_utils.py` | `get_mooncake_so_path` / `probe_allocator_backend`（ctypes 探针封装） |
| `mooncake-transfer-engine/nvlink-allocator/nvlink_allocator.cpp` | GPU fabric 分配的 `.so` 源码：`cuMemCreate/cuMemMap` + `extern "C" mc_allocator_*` |
| `mooncake-transfer-engine/fabric_allocator.cmake` | `add_fabric_allocator_build_target`：调外部 `build.sh` 在 CMake 树外构建 `.so` |
| `docs/source/performance/mooncake/allocator-benchmark-result.md` | 官方基准：明确 MmapArena 与 OffsetAllocator 是互补层次，并给出 round-up-to-bin 的前后对比 |

## 核心数据结构

### 3.1 `AllocatedBuffer`：一个"副本句柄"，不是一个指针

`mooncake-store/include/allocator.h:70-177`。它是 Master 侧分配结果的唯一产物，也是整个子系统所有权关系的浓缩。

```cpp
std::weak_ptr<BufferAllocatorBase> allocator_;   // allocator.h:158
SegmentLifetime segment_lifetime_;               // :164
std::shared_ptr<ClientLivenessRecord> client_liveness_;  // :165
std::string segment_name_;                       // :166
void* buffer_ptr_{nullptr};                      // :167  ← 客户端虚拟地址，不是本地可解引用指针
std::size_t size_{0};                            // :168  ← 请求大小
std::string protocol{"tcp"};                     // :169
std::optional<offset_allocator::OffsetAllocationHandle> offset_handle_;  // :171
```

几个设计要点：

- **`allocator_` 是 `weak_ptr`**（`:158`）。这是刻意的：`AllocatedBuffer` 不能反向延长 allocator 的生命周期，否则 segment 卸载后 allocator 永远析构不掉。代价是析构时可能 lock 失败——此时走 `AllocatedBuffer::~AllocatedBuffer` 的 `if (alloc) alloc->deallocate(this)`（`mooncake-store/src/allocator.cpp:66-75`），注释说明 segment 已经拆掉、用量不再需要维护。
- **`buffer_ptr_` 是远端客户端的虚拟地址**。Master 无法解引用它，它只是 `Descriptor` 里下发给写者的搬运目标。CXL 协议下更极端：`change_to_cxl`（`allocator.cpp:118-123`）把它减去 `DEFAULT_CXL_BASE` 变成**相对偏移**，`get_vaddr_from_cxl`（`:125-128`）再还原。
- **`offset_handle_` 是 OFFSET 路径的 RAII 归还器**（`:171`）。CACHELIB 路径不用它，靠 `deallocate` 显式 `free`。
- **`Descriptor`**（`:143-150`）是可序列化视图，四个字段 `size_ / buffer_address_ / protocol_ / transport_endpoint_`，`YLT_REFL` 生成反射用于 msgpack。`get_descriptor()` 在 `allocator.cpp:112-116`；`getTransportEndpoint()`（`allocator.cpp:96-109`）在 protocol 为 `cxl` 时返回 `segment_name_`（即 CXL binding name），而不是 allocator 的 endpoint——这是 CXL 段的特殊寻址方式。

两个"有效性"谓词不能混用：

- `isAllocatorValid()`（`allocator.h:100-102`）= allocator 未过期 **且** `segment_lifetime_` 可用；
- `isAvailable()`（`:108-114`）= 上面那条 **且** 客户端 liveness 记录仍为 serving。

### 3.2 `SegmentLifetime` 与"两套生命周期"

`SegmentLifetime`（`allocator.h:50-68`）只是一个 `shared_ptr<atomic<bool>>` 的包装：拷贝即共享，`setAvailable(false)` 一次性把同一 lifetime 的所有持有者标成不可用。之所以要共享语义，是因为它是 `SegmentAllocatorRegistration` 的成员，需要被"复制"进每个 `AllocatedBuffer`。

`SegmentAllocatorRegistration`（`mooncake-store/include/segment_allocator_registration.h`、实现 `src/segment_allocator_registration.cpp`）持有**两条**独立 lifetime：

```cpp
std::shared_ptr<BufferAllocatorBase> allocator_;
SegmentLifetime allocation_lifetime_;   // 还能不能继续分配
SegmentLifetime buffer_lifetime_;       // 已发出的 buffer 还算不算数
std::shared_ptr<ClientLivenessRecord> client_liveness_;
```

`Invalidate()`（`segment_allocator_registration.cpp:79-82`）同时关两条；`SetAllocatable()`（`:75-77`）只关分配但不影响已发出的 buffer。`Allocate()`（`:22-45`）的流程值得逐行看：先查 `allocation_lifetime_`，再原子读 `client_liveness_`，分配成功后 `bindSegmentLifetime(buffer_lifetime_)` + `bindClientLiveness(record)`，**最后再检查一次** lifetime/liveness 是否在分配过程中被并发翻转，若翻了就返回 `nullptr`（buffer 随即析构归还）。这是典型的"分配后二次校验"模式，用来消除 unregister 与 allocate 的竞态窗口。

`OwnsBuffer()`（`:70-73`）通过比较 `SegmentLifetime` 的 `operator==`（即比较底层 `shared_ptr` 指针）判断某个 buffer 是否属于本注册项——这是 HA 恢复替换 allocator 时区分新旧 buffer 的手段。替换本身走 `BindAllocator`（`:52-56`），对 `allocator_` 做 `std::atomic_store_explicit`；`GetAllocator`/`Allocate` 用对应的 `std::atomic_load_explicit` 读取，所以"换 allocator"与"并发 allocate"之间不会读到半更新的 `shared_ptr`。

### 3.3 `BufferAllocatorBase`：两套记账同时存在

`allocator.h:183-227`：

```cpp
virtual std::unique_ptr<AllocatedBuffer> allocate(size_t size) = 0;
virtual void deallocate(AllocatedBuffer* handle) = 0;
virtual size_t capacity() const = 0;     // 段总大小
virtual size_t size() const = 0;         // 已用量（requested 之和）
virtual uintptr_t base() const = 0;
virtual size_t getLargestFreeRegion() const = 0;
void AttachUsageTracker(const std::shared_ptr<StorageUsageTracker>&);  // :211
```

私有成员只有两个：`std::atomic_size_t cur_size_{0}`（`:225`）与 `std::unique_ptr<StorageUsageRegistration> usage_registration_`（`:226`）。**这里有两套互不相干的记账**，混淆是常见错误：

| 记账 | 位置 | 用途 | 触发点 |
|---|---|---|---|
| `cur_size_` + `StorageUsageRegistration` | `allocator.cpp:44-56` | 分配策略的水位判断（`FreeRatioFirstAllocationStrategy`）与 `GetMemoryUsageSnapshot` | `RecordAllocation/RecordDeallocation` |
| `MasterMetricManager::inc_allocated_mem_size` | `allocator.cpp:258-264` 等 | Prometheus 每段指标 | 每次 allocate/deallocate/restore |

`StorageUsageRegistration`（`include/storage_usage.h:71-102`）在构造时把 allocator 的**当前** `used_bytes` 和 `capacity()` 加进 tracker，析构时减回去；它持有一个 `std::atomic_size_t&` 引用（指向 allocator 的 `cur_size_`）。成员声明顺序保证了析构顺序正确：`usage_registration_` 在 `cur_size_` 之后声明，因此先析构、先 detach，不会读悬空引用。

`AttachUsageTracker`（`allocator.cpp:35-42`）是幂等的、只允许在发布前调用一次；`SegmentManager::AttachMountedUsageTrackers`（`mooncake-store/src/segment.cpp:142-158`）负责在恢复后统一补挂（用 `unordered_set<const BufferAllocatorBase*>` 去重，因为 CXL 全局 allocator 可能被多条 segment 记录共享）。新版 `SegmentPool`/`RegionDriver` 路径不是批量补挂，而是在挂载/恢复事务里逐条挂：`SegmentPool::WriteAccess::PrepareWithLiveAllocations`（`segment/pool_mount.cpp:91-92`，`PrepareMount`/`PrepareRestore` 都汇入这里）与 `PublishMount`（`:186`，覆盖快照恢复走的 `PrepareAdopt` + `Commit`）；两处都可能命中同一个 allocator，靠 `AttachUsageTracker` 自身的幂等性去重。

`size()` 返回的是**请求字节数之和**，不是实际占用的 bin/slab 字节。这一点对 OFFSET 尤其重要（见 §6）。

### 3.4 `CachelibBufferAllocator`

`allocator.h:278-339`，实现 `allocator.cpp:142-349`。

字段：`segment_name_ / base_ / total_size_ / transport_endpoint_ / replica_type_` 五个不可变元数据，加上三块状态：

```cpp
std::unique_ptr<char[]> header_region_start_;      // SlabHeader 数组，master 本地堆分配
size_t header_region_size_;                        // :328
std::unique_ptr<MemoryAllocator> memory_allocator_; // :329
PoolId pool_id_;                                   // :330
```

构造函数（`allocator.cpp:185-222`）的关键点：

```cpp
header_region_size_ = sizeof(SlabHeader) * (size / sizeof(Slab)) + 1;   // :199-202
header_region_start_ = std::make_unique<char[]>(header_region_size_);   // :203
memory_allocator_ = std::make_unique<MemoryAllocator>(
    Config(generateAllocSizes()), header_region_start_.get(), header_region_size_,
    reinterpret_cast<void*>(base), size);                               // :208-212
pool_id_ = memory_allocator_->addPool("main", size);                    // :219
```

**元数据与段内存是分离的**：`SlabHeader` 数组由 Master 自己 `new` 出来（`header_region_start_`），而 `slabMemoryStart` 传的是客户端上报的远端虚拟地址。CacheLib 的原设计里两者通常在同一片 mmap 里，这里被拆开了——因为 Master 根本拿不到客户端内存。

`IsValidCachelibLayout`（`allocator.cpp:17-24`）给出 CACHELIB 的六条硬约束：

```cpp
base != 0 && base % Slab::kSize == 0 &&
size >= Slab::kSize && size % Slab::kSize == 0 &&
slab_count <= UINT_MAX && base <= SIZE_MAX - size
```

注意 `slab_count = size / sizeof(facebook::cachelib::Slab)`——用的是 `sizeof(Slab)`，而 `Slab` 里有一个 `char data_[kSize]`（`Slab.h:101`），所以 `sizeof(Slab) == kSize == 16MB`。这是"用类型大小当常量"的写法，改 `Slab` 布局会静默改变语义，读代码时容易看漏。

`MountSegment` 在创建前还会再拦一次（`segment.cpp:245-252`），错误信息里明确写了 "as required by Cachelib"。

`allocate`（`allocator.cpp:235-266`）：`padding_size = max(size, kMinSliceSize)`（`kMinSliceSize = 64`，`types.h:450`），调 `memory_allocator_->allocate(pool_id_, padding_size)`；成功后 `RecordAllocation(size)` 与 `inc_allocated_mem_size(segment_name_, size)` **都按请求大小记**，不是 `padding_size`，也不是 size-class 的实际占用。失败与异常都返回 `nullptr`，不抛。

`deallocate`（`:268-292`）：CXL buffer 要先用 `get_vaddr_from_cxl()` 还原绝对地址再 `free`（`:270-272`），因为 `change_to_cxl` 已经把它改成了相对偏移。

析构（`:224-233`）按 `replica_type_` 减指标：`MEMORY` 减 `allocated_mem_size` 并 `remove_segment_metrics`，`NOF_SSD` 减 `allocated_nof_size`。

### 3.5 `OffsetBufferAllocator` 与 `OffsetBufferAllocatorSnapshot`

`allocator.h:377-436`，实现 `allocator.cpp:352-565`。字段极简：五个不可变元数据 + 一个 `shared_ptr<OffsetAllocator>`（`allocator.h:435`）。

构造函数（`allocator.cpp:352-394`）里两个容量参数是从段大小推出来的：

```cpp
uint64_t init_capacity = clamp(size / 4096, 1024, 64 * 1024);          // :367-370
uint64_t max_capacity  = clamp(size / 1024, 1024*1024, kMaxNodes);      // :372-378
```

含义是：初始按"平均 4KB 一个 node"预算，上限按"平均 1KB 一个 node"预算，且节点总数不超过 `OffsetAllocatorSnapshot::kMaxNodes = 1<<26`（`offset_allocator.h:334`，约 6710 万）。所以：

- 64GB 段：`max_capacity = 64M`，恰好等于 `1<<26`（6710 万）这个上限，不多不少；
- 1TB 段：理论 10 亿，被夹到 6710 万 → **平均分配粒度不能低于约 16KB**，否则节点会先用完。

`OffsetBufferAllocatorSnapshot`（`allocator.h:354-370`）是"脱离运行时的快照"：外层的 `segment_name/base/capacity/used_bytes/transport_endpoint` 加内层 `OffsetAllocatorSnapshot`。`ValidateMetadata()`（`allocator.cpp:406-428`）做三件事：外层与内层 base/capacity 必须一致、`used_bytes <= capacity` 且不超 `int64` 上限、`used_bytes == allocation_state.allocated_size`。`Validate()`（`:430-434`）再叠加内层的全图校验。

`Restore`（`:436-461`）的顺序值得记住：**先校验元数据 → 再让 `OffsetAllocator::Restore` 校验完整图 → 构造对象 → 最后才 `inc_allocated_mem_size` + `SetUsageBytesForRestore`**。注释（`:455-456`）强调"No allocation handles exist yet. Pair restored usage with the same accounting that allocate()/destruction use, before publishing the object"。注意这里只更新了 Prometheus 指标与 `cur_size_`，**没有** `AttachUsageTracker`——头文件注释（`allocator.h:388-391`）说得很清楚："capacity accounting remains the owning pool's responsibility"。

`allocate`（`:480-523`）把 `OffsetAllocationHandle` 直接移进 `AllocatedBuffer`，`buffer_ptr = allocation_handle->ptr()`（也就是 `m_base + (offset << multiplier_bits)`）。`deallocate`（`:525-546`）只做 `handle->offset_handle_.reset()`，真正的归还发生在 handle 析构里。

`getLargestFreeRegion`（`:548-565`）转调 `offset_allocator_->storageReport().largestFreeRegion`，**加锁**取一次权威值；而 `OffsetAllocator` 内部的 `m_largest_free_region` 是一个可能偏大的无锁提示值。两者的差异见流程 B（B1/B6）与 §6.5。

### 3.6 `OffsetAllocator` / `__Allocator`：bin 分配的全部状态

`offset_allocator.h:144-256` 是线程安全外壳，`offset_allocator.h:258-319` 是算法内核。

外壳的关键字段：

```cpp
std::unique_ptr<__Allocator> m_allocator GUARDED_BY(m_mutex);  // :227
uint64_t m_base;              // :228  真实基址（字节）
uint64_t m_multiplier_bits;   // :231  所有 offset/size 的 quantum = 1<<bits
uint64_t m_capacity;          // :232  字节容量
mutable Mutex m_mutex;        // :233  注意：整段一把锁
uint64_t m_allocated_size GUARDED_BY(m_mutex) = 0;   // :236 请求字节和
uint64_t m_allocated_num  GUARDED_BY(m_mutex) = 0;   // :237
std::atomic<uint64_t> m_largest_free_region{0};      // :238 无锁提示
bool m_largest_free_region_tightened GUARDED_BY(m_mutex) = false;  // :239
```

`GUARDED_BY` / `REQUIRES` 来自 `mooncake-store/include/mutex.h:19-52`，`__clang__` 之外是空宏，所以只是文档。

内核 `__Allocator`（`offset_allocator.h:258-319`）：

```cpp
struct Node {                       // :290-300
    uint32 dataOffset = 0;
    uint32 dataSize   = 0;
    NodeIndex binListPrev = unused;   // 同一个 bin 内的双向链表
    NodeIndex binListNext = unused;
    NodeIndex neighborPrev = unused;  // 物理相邻节点（用于合并）
    NodeIndex neighborNext = unused;
    bool used = false;
};
uint32 m_size;                 // 总大小，单位是 quantum（不是字节！）
uint32 m_current_capacity;     // 已启用的节点数
uint32 m_max_capacity;         // 节点数上限
uint32 m_freeStorage;          // 空闲总量，单位是 quantum
uint32 m_usedBinsTop;          // 32 个 top bin 的位图
uint8  m_usedBins[32];         // 每个 top bin 下 8 个 leaf 的位图
NodeIndex m_binIndices[256];   // 256 = 32 * 8 个 leaf bin 的头节点
std::vector<Node> m_nodes;
std::vector<NodeIndex> m_freeNodes;
uint32 m_freeOffset;           // free-list 栈顶
```

`m_nodes` / `m_freeNodes` 在 `reset()` 里 `reserve(m_max_capacity)`（`offset_allocator.cpp:175-176`），随后 `resize(m_current_capacity)`。所以**虚拟地址空间按上限预留，物理页随实际写入的节点增长**；不要据此断言 RSS 会等于 `max_capacity × sizeof(Node)`。

`m_multiplier_bits` 由 `calculateMultiplier`（`offset_allocator.cpp:539-545`）算出：

```cpp
for (; SmallFloat::MAX_BIN_SIZE < (size >> multiplier_bits); multiplier_bits++) {}
```

`MAX_BIN_SIZE = 4026531840`（3.75GB，`offset_allocator.cpp:60`）。也就是说不超过 3.75GB 的段 `multiplier_bits == 0`，全部按字节计；超过则整个地址空间按 `1<<bits` 压缩，`m_size = size >> bits` 才能塞进 `uint32`。**`Node::dataOffset/dataSize` 永远是 quantum 单位**，真实值要 `<< multiplier_bits`。

### 3.7 `OffsetAllocation` / `OffsetAllocationHandle`

`OffsetAllocation`（`offset_allocator.h:35-53`）只有两个 `uint32`：`offset`（quantum 单位）与 `metadata`（节点下标）。`NO_SPACE = 0xffffffff` 同时用作两者的空值。`getOffset()`（`:47`）返回 `uint64_t` 但**仍是 quantum 单位**——这是最容易被误读的一行。

`OffsetAllocationHandle`（`:70-117`）是 RAII 归还款：

```cpp
std::weak_ptr<OffsetAllocator> m_allocator;   // :108  ← 同样用 weak_ptr
OffsetAllocation m_allocation;                // :110
uint64_t real_base;                           // :112  ← 这里才是真实字节地址
uint64_t requested_size;                      // :113
```

`ptr()`（`:102`）= `reinterpret_cast<void*>(real_base)`。析构（`offset_allocator.cpp:531-536`）通过 `freeAllocation(m_allocation, requested_size)` 归还。移动赋值（`:507-529`）会先归还自己原有的分配，再接管对方的——所以 `handle = std::move(other)` 不会泄漏。

### 3.8 `SmallFloat`：3 位尾数的分段对数 size-class

`offset_allocator.cpp:56-121`。这是整个分配器的"分级"来源，本质是把 `uint32` 映射到一个 8 bit 的 float 表示（`exp` + 3 位尾数）：

- `uintToFloatRoundUp(size)`（`:65-89`）：返回**能装下 size 的最小 bin 编号**；
- `uintToFloatRoundDown(size)`（`:91-109`）：返回**size 落入的 bin 编号**；
- `floatToUint(bin)`（`:111-120`）：bin 的**下界**字节数。

`size < 8` 走 denorm 分支（`mantissa = size`），否则取最高位、截 3 位尾数并按需进位。由于是分段线性对数分布，"每个 size class 的平均相对浪费比例相同"（注释 `:62-64`）。

256 个 leaf bin 用一级 `m_usedBinsTop`（32 位）+ 二级 `m_usedBins[32]`（每个 8 位）表示非空，因此找可用 bin 是常数次位运算：`findLowestSetBitAfter`（`:124-130`）做 `mask & ~((1<<start)-1)` 再 `tzcnt`。

### 3.9 `MmapArena` / `ClientBufferAllocator`

`MmapArena`（`include/mmap_arena.h:19-106`）：

```cpp
static constexpr size_t kMinAlignment = 64;      // :21
std::atomic<void*>  pool_base_;                  // :93
std::atomic<size_t> pool_size_;                  // :95
std::atomic<size_t> alignment_;                  // :97
std::atomic<size_t> alloc_cursor_;               // :100 单调递增的 bump 指针
std::atomic<size_t> peak_reserved_;              // :101
std::atomic<size_t> num_allocations_, num_failed_allocs_;  // :102-103
std::mutex init_mutex_;                          // :105 只保护 initialize()
```

**没有 free 方法**。这是 bump allocator 的本质：只进不退，靠 `owns()`（`mmap_arena.cpp:271-281`）在释放路径上识别"这是 arena 的，跳过 munmap"。

`ClientBufferAllocator`（`include/client_buffer.h:27-72`）持有一块真实内存 + 一个 `OffsetAllocator`：

```cpp
void* buffer_; size_t buffer_size_; bool use_hugepage_;   // :59-61
std::shared_ptr<offset_allocator::OffsetAllocator> allocator_;  // :67
std::string protocol; bool is_external_memory_; bool use_spdk_dma_;  // :69-71
```

`BufferHandle`（`:82-112`）是双模式：owning 模式持 `shared_ptr<ClientBufferAllocator>` + `OffsetAllocationHandle`；view 模式持裸指针 + `std::function<void()> release_fn_`。`ptr()`/`size()`（`client_buffer.cpp:117-123`）按 `view_ptr_` 是否为空二选一。

### 3.10 `RegisteredPinnedMemoryManager`

`src/registered_pinned_memory.h`（注意在 `src/` 而非 `include/`，是 store 内部头）：

```cpp
class RegisteredPinnedRegion {          // :16-35
    RegisteredPinnedMemoryManager* manager_;   // 非拥有，靠 manager 全局单例活着
    void* addr_; size_t size_; bool release_succeeded_ = true;
};
struct ActiveRegion { void* addr; size_t size; RegisteredPinnedRegion* region; }; // :56-60
const uint64_t limit_bytes_;  const PinOps pin_ops_;   // :75-76
mutable std::mutex mutex_; uint64_t pinned_bytes_ = 0;
std::vector<ActiveRegion> regions_;
```

`PinOps`（`:41-46`）是两个 C 函数指针（register/unregister），默认实现走 `cudaHostRegister(..., cudaHostRegisterPortable)` 与 `cudaHostUnregister`（`registered_pinned_memory.cpp:20-41`）。这个注入点让单元测试可以在没有 CUDA 的机器上跑（`MOONCAKE_STORE_TEST` 下构造函数被提到 public，`:63-70`）。

### 3.11 设备抽象、`PinnedHostBuffer` 与 fabric

`include/device/accelerator_device.h`：

```cpp
enum class AcceleratorVendor { kNvidia, kMusa, kMaca, kHygon, kCorex,
                               kSupa, kHip, kAscend, kSunrise, kIntel };  // :12-23
enum class MemoryKind { kHost, kDevice, kUnknown };                       // :25-29
struct PointerInfo { MemoryKind kind; int32_t device_id = -1; };          // :39-42
class AcceleratorDevice {                                                 // :44-56
    virtual AcceleratorVendor Vendor() const = 0;
    virtual bool Available(bool ensure = false) const = 0;
    virtual PointerInfo QueryPointer(const void* ptr) const = 0;
    virtual int32_t CurrentDeviceId() const = 0;
    virtual void SetContext(int32_t device_id) const = 0;
    virtual bool Copy(void* dst, const void* src, size_t size, CopyDirection) const = 0;
    virtual PinnedHostBuffer AllocatePinnedHost(size_t size) const = 0;
};
```

`ProbeCachedAcceleratorDevice`（`:58-67`）用 `mutable std::atomic<uint8_t> available_state_` 缓存"这个厂商在不在"的探测结果，避免每次 `Available()` 都去初始化运行时。

`PinnedHostBuffer`（`include/pinned_host_buffer.h:10-49`）是 move-only + 函数指针 deleter，CUDA-like 实现里 deleter 就是 `cudaFreeHost` 的包装（`src/device/cuda_like_accelerator_device.cpp:71-78` 用 `cudaMallocHost`）。

同一份 `nvlink_allocator.cpp` 通过 `cuda_alike.h` 的类型别名被 hipcc / mcc / maca 编译成 HIP/MUSA/MACA 版本，这是"一次源码、多厂商产物"的关键。

### 3.12 指标结构

`include/allocator_metric.h`：

```cpp
struct JemallocBinStats { uint64 size, regs_per_slab, regs, slabs; };  // :31-36
struct JemallocSnapshot {                                              // :38-67
    uint64_t allocated, active, metadata, resident, retained, mapped;
    uint64_t small_allocated, large_allocated, dirty_bytes, muzzy_bytes;
    uint64_t arenas, background_threads;
    int64_t  opt_dirty_decay_ms, opt_muzzy_decay_ms;
    uint64_t dirty_purged_pages, ... , small_deallocations;   // cumulative
    std::vector<JemallocBinStats> bins;
};
using JemallocStatsCollector = bool (*)(JemallocSnapshot&);            // :69
```

`AllocatorMetric`（`:100-208`）有 7 个进程 gauge、15 个 jemalloc gauge、8 个 counter、4 个带 `size` 标签的 `hybrid_gauge_1t`。`kMinRefreshInterval{1000}`（`:139`）是刷新限流。

## 关键流程剖析

### 流程 A：MountSegment → 创建分配器 → 切出一个副本

这是 Master 侧最主干的路径。

```
客户端 RPC: MountSegment(segment{base,size,protocol,te_endpoint})
  → ScopedSegmentAccess::MountSegment            (segment.cpp:169)
      ├─ 查重（同名/同 id/同 base,size,te_endpoint,protocol）      (:189-206)
      ├─ CXL 分支：enable_cxl_ && protocol=="cxl"                    (:211-237)
      │     └─ 直接用 SegmentManager::cxl_global_allocator_（CACHELIB）
      ├─ CACHELIB 对齐校验（base/size 均须 16MB 对齐）               (:245-252)
      ├─ CreateBufferAllocator(type, name, base, size, endpoint)     (:255)
      │     └─ allocator.cpp:158-183  按枚举分派
      ├─ allocator->AttachUsageTracker(usage_tracker_)               (:262)
      ├─ allocator_manager_.addAllocator(name, allocator, liveness)  (:263-264)
      └─ mounted_segments_[id] = {segment, OK, allocator, registration} (:266-267)
```

三个细节：

1. **CXL 是特例**：`segment.cpp:213-236` 完全绕过 `CreateBufferAllocator`，直接用全局 CXL allocator，且只支持 CACHELIB（否则 `INTERNAL_ERROR`）。同一逻辑在新版 RegionDriver 里也是硬编码：`CreateRegionDrivers` 用 `BufferAllocatorType::CACHELIB` + `DEFAULT_CXL_BASE` 建 CXL allocator（`mooncake-store/src/segment/region_driver.cpp:303-320`），与配置的 `memory_allocator` 无关。
2. **`addAllocator` 包注册项**：`AllocatorManager::addAllocator`（`include/allocation_strategy.h:51-68`）构造 `SegmentAllocatorRegistration`，把 `ClientLivenessRecord` 一起塞进去。之后所有分配都走 `registration->Allocate(size)`，而不是直接调 allocator。
3. **恢复后补挂 tracker**：快照恢复出来的 allocator 不会走上面的 `AttachUsageTracker`，由 `SegmentManager::AttachMountedUsageTrackers`（`segment.cpp:142-158`）统一补。

写路径上真正的分配发生在 `MasterService::PutStart`：

```
PutStart(key, value_length)
  → (CACHELIB 且 slice_length > kMaxSliceSize) 直接 INVALID_PARAMS   (master_service.cpp:5506-5512)
  → allocation_strategy_->Allocate(allocator_snapshot, value_length,
                                   replica_num, preferred_segments, {}, MEMORY)
                                                                     (master_service.cpp:5196-5199)
      → RandomAllocationStrategy::allocateSingle                      (allocation_strategy.h:423-452)
          → registration->Allocate(slice_length)                      (segment_allocator_registration.cpp:22-45)
              → BufferAllocatorBase::allocate(size)                   （虚分派）
                  ├─ OffsetBufferAllocator::allocate                  (allocator.cpp:480)
                  └─ CachelibBufferAllocator::allocate                (allocator.cpp:235)
```

上面画的是默认 `AllocationStrategyType::RANDOM` 的路径。若配成 `FREE_RATIO_FIRST`/`SSD_FREE_RATIO_FIRST`/`LOCAL_FIRST`，会先经过 `RankedAllocationStrategy::AllocateRanked`（`allocation_strategy.h:465-563`）做"采样 + 打分 + 排序 + 兜底重试（`kMaxRetryLimit = 100`）"，但最终仍然落到同一个 `allocateSingle`，所以"唯一分配入口是 `registration->Allocate`"这个结论不变。

`AllocatorManager` 本身不做锁（头文件注释 `allocation_strategy.h:29-32`：线程安全由 `SegmentManager::segment_mutex_` 提供），所以主路径先在锁内 `SnapshotAllocatorManager()` 拷一份，再在锁外跑策略（`master_service.cpp:5193-5196`）。

`FreeRatioFirstAllocationStrategy`（`allocation_strategy.h:573-618`）的评分函数是：

```cpp
total_free += capacity - buffer_allocator->size();   // :607-609
return total_free / total_capacity;                  // :614
```

也就是说它的水位完全建立在 `size()`（请求字节和）之上——这正是 §3.3 那套 `cur_size_` 记账的下游消费者。

### 流程 B：OffsetAllocator 的一次 allocate 与一次 free

这是整个子系统算法密度最高的部分，逐段读 `mooncake-store/src/offset_allocator.cpp:191-357`。

**B1. 外壳层前置检查**（`OffsetAllocator::allocate`，`:880-935`）：

```cpp
if (size == 0) return std::nullopt;                          // :881-883
if (size > getLargestFreeRegion()) return std::nullopt;      // :889-891  无锁快速失败
const size_t fake_size = m_multiplier_bits > 0
    ? ((size + (1<<bits) - 1) >> bits) : size;                // :893-897  字节→quantum，向上取整
if (fake_size > SmallFloat::MAX_BIN_SIZE) return std::nullopt;// :899-901
MutexLocker guard(&m_mutex);                                  // :903
OffsetAllocation allocation = m_allocator->allocate(fake_size);  // :908
if (allocation.isNoSpace()) { refreshLargestFreeRegion(); ... return nullopt; }  // :909-921
m_allocated_size += size;  m_allocated_num++;                 // :924-925
const uint64_t real_base = m_base + (allocation.getOffset() << bits);  // :927-928
guard.unlock();                                               // :929
return OffsetAllocationHandle(shared_from_this(), allocation, real_base, size);  // :933
```

这里有两个精心设计的点：

- **`size > getLargestFreeRegion()` 的快速失败不需要拿锁**。`m_largest_free_region` 是 `atomic`，可能偏大（成功分配后不缩小），因此失败时它只是"多走一次加锁路径"，不会误拒；而一旦在锁内观察到真实的 `NO_SPACE`，就调用 `refreshLargestFreeRegion()`（`:1001-1013`）把提示收紧到权威值。头文件注释（`offset_allocator.h:180-184`）明确写了这三条规则：可能偏大、失败时收紧、free 时抬高。
- **handle 的构造在 `guard.unlock()` 之后**（`:929-934`）。注释解释了原因：构造 handle 与 `shared_ptr` 引用计数不访问 allocator 状态，不应延长临界区。

**B2. 内核层选 bin**（`__Allocator::allocate`，`:191-289`）：

```cpp
if (m_freeOffset == m_max_capacity) return NO_SPACE;            // :193-196 元数据耗尽
if (m_freeOffset == m_current_capacity) {                       // :197-201 按需扩容一个节点
    m_freeNodes.push_back(m_current_capacity);
    m_nodes.emplace_back();
    m_current_capacity++;
}
uint32 minBinIndex = SmallFloat::uintToFloatRoundUp(size);      // :205 最小可容纳 bin
uint32 minTopBinIndex  = minBinIndex >> 3;                      // :207
uint32 minLeafBinIndex = minBinIndex & 0x7;                     // :208
if (m_usedBinsTop & (1 << minTopBinIndex))                      // :214-218
    leafBinIndex = findLowestSetBitAfter(m_usedBins[minTopBinIndex], minLeafBinIndex);
if (leafBinIndex == NO_SPACE) {                                 // :220-233
    topBinIndex = findLowestSetBitAfter(m_usedBinsTop, minTopBinIndex + 1);
    if (topBinIndex == NO_SPACE) return NO_SPACE;
    leafBinIndex = tzcnt_nonzero(m_usedBins[topBinIndex]);
}
```

这就是"两级位图 + `tzcnt`/`lzcnt`"的常数时间查找：先在请求落入的 top bin 内找同层或更高的 leaf，找不到就往更高的 top bin 走。因为更高 top bin 的所有 leaf 都必然装得下，所以直接从 bit 0 取最低位即可。

**B3. Mooncake 对上游最关键的修改：round up 到 bin 边界**（`:241-249`）：

```cpp
// Modified in Mooncake project: Round up to bin size. Otherwise when this
// node is freed, if it cannot be merged with neighbors, it will be inserted
// to a smaller bin.
#ifdef OFFSET_ALLOCATOR_NOT_ROUND_UP
    uint32 roundupSize = size;
#else
    uint32 roundupSize = SmallFloat::floatToUint(minBinIndex);
#endif
```

为什么重要：设请求落在 bin B，`floatToUint(B) >= size`。若只按 `size` 占用，free 时该节点无法与邻居合并，会按 `roundDown(size)` 插入**更小**的 bin；下次同样大小的请求 `roundUp(size)` 又会去找 bin B，于是这个空闲节点永远匹配不上——uniform-size 负载下利用率断崖式下跌。官方基准（`docs/source/performance/mooncake/allocator-benchmark-result.md`）里 "Before Optimization / After Optimization" 两组表对应的就是这个 round-up 优化（文档写的是"优化前/优化后"两版分配器，而非直接编译期 A/B 宏）。`OFFSET_ALLOCATOR_NOT_ROUND_UP` 只在 `offset_allocator.cpp:244`、`:611` 与测试 `tests/offset_allocator_test.cpp:2159` 出现，没有 CMake 开关，需要手工 `-D`。

**B4. 摘节点 + 拆剩余**（`:250-288`）：

```cpp
node.dataSize = roundupSize;  node.used = true;                 // :250-251
m_binIndices[binIndex] = node.binListNext;                      // :252 弹出 bin 头
m_freeStorage -= nodeTotalSize;                                 // :255 注意减的是原 node 大小
if (m_binIndices[binIndex] == Node::unused) { ...清两级位图... } // :261-270
uint32 reminderSize = nodeTotalSize - roundupSize;              // :273
if (reminderSize > 0) {                                          // :274-286
    uint32 newNodeIndex = insertNodeIntoBin(reminderSize, node.dataOffset + roundupSize);
    // 把新节点挂进物理邻居链：node.neighborNext ↔ newNode ↔ 旧 neighborNext
}
return OffsetAllocation(node.dataOffset, nodeIndex);            // :288
```

**B5. free 与双向合并**（`:291-357`）：

```cpp
uint32 nodeIndex = allocation.metadata;
Node& node = m_nodes[nodeIndex];
ASSERT(node.used == true);                                       // :299 双 free 检查（Release 下空宏）
uint32 offset = node.dataOffset, size = node.dataSize;
if (prev 物理相邻且未使用) { offset = prev.offset; size += prev.dataSize;
                            removeNodeFromBin(prev); node.neighborPrev = prev.neighborPrev; }
if (next 物理相邻且未使用) { size += next.dataSize;
                            removeNodeFromBin(next); node.neighborNext = next.neighborNext; }
m_freeNodes[--m_freeOffset] = nodeIndex;                          // :341 归还节点槽位（LIFO 栈）
uint32 combinedNodeIndex = insertNodeIntoBin(size, offset);       // :344 整块重新入 bin
// 再修复邻居链
return m_freeOffset < m_max_capacity ? size : 0;                  // :356
```

`free` 的返回值有两重语义：正常时是"合并后的大小"，元数据耗尽时返回 0（`offset_allocator.h:271-273` 的注释）。上层 `freeAllocation`（`:1015-1051`）把它当成"新空闲区大小"，用于抬高 `m_largest_free_region`。

**B6. free 侧的三处防御**（`freeAllocation`，`:1015-1051`）：

- 只有 `m_largest_free_region_tightened` 为真时才需要考虑抬高提示（`:1022-1036`）：未收紧时提示已经 ≥ 所有现存空闲区的可服务大小（初始值是整段算出来的最大可服务请求，之后成功分配只会让它"过时偏高"而不会偏低），无需更新；合并出的区域 >= 全容量时再把 `tightened` 复位为 false。
- `m_allocated_size` 用**饱和减法**而不是回绕：`if (size > m_allocated_size) { LOG(WARNING) ...; m_allocated_size = 0; }`（`:1041-1048`）。注释说明恢复坏记录时可能 free 一个尺寸未知的节点，无符号回绕会永久污染指标。
- `removeNodeFromBin`（`:395-442`）有个隐藏成本：如果被删节点是 bin 的头节点，它必须用 `roundDown(node.dataSize)` **重新算出 bin 下标**（`:408`），因为 `Node` 里不存 `binIndex`。

**B7. 元数据上限是硬约束**。`m_freeOffset == m_max_capacity` 时直接返回 `NO_SPACE`（`:193-196`），哪怕还有大量空闲字节。这是"碎片化"之外的第二种失败模式，排查"明明还有空间却分配失败"时要先看这个。

### 流程 C：HA 恢复的两条路径

OffsetAllocator 体系的恢复有**两套完全不同的机制**，对应两种可用信息：

```
路径 1（有完整内部布局）：
  SegmentPool::CaptureSnapshot                       (segment/snapshot.cpp:12-41)
    → OffsetBufferAllocator::CaptureSnapshot          (allocator.cpp:463-467)
      → OffsetAllocator::CaptureSnapshot              (offset_allocator.cpp:830-837) 深拷贝 __Allocator
  → AllocatorSnapshotCodec::Encode                    (ha/snapshot/allocator_snapshot_codec.cpp:9-31)
  ... 持久化 ...
  → AllocatorSnapshotCodec::Decode                    (:33-95) 解码后立刻 snapshot.Validate()
  → OffsetBufferAllocator::Restore                    (allocator.cpp:436-461)
    → OffsetAllocatorSnapshot::Validate               (offset_allocator.cpp:804-818)
    → OffsetAllocator::Restore                        (:838-851)

路径 2（只有 live allocation 列表）：
  BuildRegionLiveAllocations(spec, descriptors)       (region_driver.cpp:322)
  → ImportOffsetBufferAllocator                       (allocator.cpp:567-651)
  → ImportCachelibBufferAllocator                     (allocator.cpp:310-349)
```

**C1. 快照校验是"全图校验"**。`OffsetAllocatorSnapshot::Validate()`（`offset_allocator.cpp:804-818`）串起六个检查，顺序不能换：

| 步骤 | 函数 | 检查什么 |
|---|---|---|
| 1 | `ValidateHeader`（`:560-597`） | `layout` 非空；`capacity != 0` 且 `base + capacity` 不溢出；`multiplier_bits` 必须**等于重新算出来的期望值**（`:571-576`，这行同时守卫了后续所有移位）；`layout->m_size == capacity >> bits`；`ValidateNodeCapacity`；`nodes/freeNodes.size() == m_current_capacity` |
| 2 | `ValidateUsedNodes`（`:599-636`） | 每个 used 节点的 `dataSize` 必须**恰好是 bin 边界**（`:611-621`，仅在 round-up 生效时）；累计 `max_used_bytes` 不超 capacity；同时算出 `min_used_bytes`（该 bin 的最小可能请求字节） |
| 3 | `ValidateBins`（`:638-698`） | 遍历 256 个 leaf bin 的链表：节点下标在界内、未被别的 bin 认领（用 `NodeState` 数组分类，天然防环）、`roundDown(dataSize) == bin`、`binListPrev` 链一致；重算 `m_freeStorage`、`m_usedBinsTop`、`m_usedBins[]` 并逐一比对 |
| 4 | `ValidateFreeStack`（`:700-725`） | `m_freeNodes[m_freeOffset..]` 里的每个下标必须未被分类；`m_freeOffset == live_nodes`；最后不允许有 `kUnclassified` 残留 |
| 5 | `ValidateNeighborChain`（`:727-785`） | 物理邻居链必须**恰好一个头**、无环、`dataOffset` 严格连续递增、`offset + size == m_size` 且节点数 == live_nodes |
| 6 | `ValidateAccounting`（`:787-802`） | `allocated_num == used_nodes`；`allocated_size ∈ [min_used_bytes, max_used_bytes]` |

第 6 步的区间语义值得单独强调：快照里**不保存每个节点的原始请求 size**，所以 `allocated_size` 只能验证到一个区间里，而不是精确相等（`offset_allocator.h:342-345` 的注释）。这也解释了为什么 `used_bytes` 与 `allocated_size` 在元数据层要求严格相等（`allocator.cpp:422-426`），而在节点层只能区间校验。

`Decode` 在返回前就调用 `snapshot.Validate()`（`allocator_snapshot_codec.cpp:82-87`），`Restore` 再校验一次 `ValidateMetadata()`（`allocator.cpp:439-443`）。两次校验的注释解释：解码层负责把诊断信息带给调用方，工厂层负责拒绝非法状态。

**C2. 反序列化的两道容量闸门**。`__Allocator` 的反序列化构造（`offset_allocator.h:453-501`）在 `resize` 之前检查：

```cpp
static constexpr uint32 kMaxSerializedNodes = 1u << 24;   // :472
if (m_max_capacity == 0 || m_max_capacity > kMaxSerializedNodes ||
    m_current_capacity > m_max_capacity ||
    m_freeOffset > m_current_capacity || m_size == 0) throw ...
```

注释（`:466-471`）说明动机：一份"能解析但荒谬"的 meta 会先触发 OOM killer，而不是抛出 `bad_alloc`，从而绕过"corrupt meta → fresh start"的兜底。外层 `OffsetAllocator` 的反序列化（`:396-429`）另有一道：`m_multiplier_bits >= 32 || m_capacity == 0` 直接抛（`:407-414`）。注意这两个上限不一致：host 内存分配器允许 `kMaxNodes = 1<<26`（`offset_allocator.h:334`），而 wire 解码器只允许 `1<<24`——注释解释说 storage backend 的字节序列化器用更小的上限。

**C3. 重放式恢复：`ImportOffsetBufferAllocator`**（`allocator.cpp:567-651`）。

输入只有 `vector<LiveAllocation>{offset_from_base, requested_size}`。算法：

```
1. 造一个空的 OffsetBufferAllocator（新布局，整段一个 free node）    :575-577
2. 按 offset 升序排序（保留原下标，最后按输入顺序返回 buffers）        :579-584
3. cursor = base                                                     :588
4. 对每个 allocation（升序）：
     a. 校验 IsValidAllocation（size!=0、offset<capacity、不越界）      :627-629
     b. address = base + offset；若 address < cursor 说明重叠           :630-633
     c. occupied = normalizedAllocationSize(requested_size)             :634-635
        （必须按"分配时实际占用"算，因为 allocate 会 round up 到 bin）
     d. fill_gap(address - cursor)：把 [cursor, address) 这段空洞填满  :636-637
     e. allocate(requested_size)，并要求返回地址恰好 == address          :640-643
     f. cursor = address + occupied                                     :644
5. gaps.clear() —— 一次性释放所有填缝 buffer，让它们合并回空闲区         :648
6. 返回 {allocator, buffers}（buffers 按输入顺序）                      :649-650
```

`fill_gap`（`:590-623`）本身是"二分找最大可填请求 + 循环"：

```cpp
while (gap != 0) {
    // 在 [1, gap] 里二分，找最大的 request 使 normalizedAllocationSize(request) <= gap
    ...
    auto filler = allocator->allocate(request);
    if (!filler || (size_t)filler->data() != cursor) return false;   // 地址必须精确
    cursor += occupied; gap -= occupied; gaps.push_back(std::move(filler));
}
```

**为什么这一定成功而且布局逐字节一致**？关键在于不变量：整个过程按地址升序消费，任何时候 allocator 里**最多只有一个空闲节点**（即 `[cursor, end)` 这段尾巴）。因此：

- 每次 `allocate` 都必然从这段尾巴的头部切出，地址可预测；
- `fill_gap` 用 `normalizedAllocationSize` 精确算出"这次 allocate 会吃掉多少字节"，填到 `gap` 恰好归零；
- 最后所有 filler 都是物理相邻且空闲，`gaps.clear()` 触发一连串 free + 双向合并，重新合成一个 `[最后一个 live 区之后, end)` 的空闲节点。

测试 `tests/buffer_allocator_test.cpp:244-260` 专门验证了这个性质：65537 个"1 字节分配 + 1 字节空洞"的极端稀疏布局也能精确还原，并检查最后一个 buffer 的地址。同一文件 `:145-183` 还验证了"恢复后重新分配被删对象，地址回到原处"。

任何一个步骤失败都返回 `std::nullopt`，此时局部构造的 allocator 与 buffers 随作用域销毁，**不发布任何状态**（头文件 `allocator.h:443-445`）。

**C4. CacheLib 的恢复完全不是一回事**。`ImportCachelibBufferAllocator`（`allocator.cpp:310-349`）走的是 `MemoryAllocator::importAllocations`：

```cpp
// allocator.cpp
if (replica_type != MEMORY || !IsValidCachelibLayout(base, size)) return nullopt;  // :314-317
imports.push_back({ reinterpret_cast<void*>(base + offset),
                    static_cast<uint32_t>(std::max(requested_size, kMinSliceSize)) });  // :325-328
// 另外要求 requested_size <= UINT32_MAX（:321-323）
allocator->memory_allocator_->importAllocations(pool_id_, imports);                 // :337
```

而 `MemoryAllocator::importAllocations`（`src/cachelib_memory_allocator/MemoryAllocator.cpp:90-168`）做的事是：

1. 池必须是空的（`:93-95`）；
2. 用 `getSlabForIdx(0)` 反推 slab 区间 `[base, base + slabCount*16MB)`（`:105-112`）；
3. 对每个地址算 `slabIndex = (addr-base)/16MB`、`slabOffset = (addr-base)%16MB`、`chunkIndex = slabOffset/allocSize`，要求**地址必须落在 chunk 边界上**（`:127-134`）；
4. 同一 slab 内不允许两个不同 class（`:136-138`）、不允许重复 chunk（`:143-149`）；
5. 按 slabIndex 升序 `makeNewSlab`，并要求返回地址与 `base + idx*16MB` 精确匹配（`:151-157`）——这是"从 0 号 slab 顺序分配"这一 CacheLib 内部行为的强假设；
6. 有 allocation 的 slab 调 `pool.importSlab(slab, classId, occupied)`，没有的调 `pool.importFreeSlab(slab)`（`:158-165`）。

恢复出的 `AllocatedBuffer` 由 `adoptImportedBuffer`（`allocator.cpp:294-308`）构造，地址是 `base + offset_from_base`，`size` 用 `requested_size`（而不是 class size）。

两者的差异可以一句话概括：**OFFSET 的 import 是"重放操作序列"，CACHELIB 的 import 是"直接写入 slab 的 chunk 占用位图"**。前者不信任任何内部结构、只信地址与大小；后者依赖 CacheLib 的 slab/class/chunk 几何关系，因此对地址合法性要求严格得多（必须 chunk 对齐）。

### 流程 D：客户端缓冲——真实内存从哪来，怎么被 free

```
ClientBufferAllocator::create(size, protocol, use_hugepage, use_spdk_dma)   (client_buffer.cpp:19-24)
  → 构造 (client_buffer.cpp:32-66)
      ├─ alignment = 64                                          (:46)
      ├─ use_hugepage_ ? allocate_buffer_mmap_memory(size, 64)   (:47-48)
      │                 └─ mmap_buffer.cpp:203-260
      ├─ sunrise_link ? sunrise_allocate_memory(...)             (:50-53，仅在非 hugepage 分支内)
      └─ else allocate_buffer_allocator_memory(size, protocol, 64, use_spdk_dma)
      │                 └─ client_buffer_allocation.cpp:97-143
      └─ OffsetAllocator::create(reinterpret_cast<uint64_t>(buffer_), size)  (:64-65)
```

`allocate_buffer_allocator_memory`（`client_buffer_allocation.cpp:97-143`）的编译期分支顺序就是"协议优先级"：

| 条件 | 去处 |
|---|---|
| `USE_ASCEND_DIRECT`/`USE_UBSHMEM` 且 protocol ∈ {ascend, ubshmem} | `ascend_allocate_memory`（`:106-109`） |
| `USE_MEMFABRIC` 且 protocol == ascend | `memfabric_allocate_memory`（`:110-114`） |
| `USE_SUNRISE` 且 protocol == sunrise_link | `sunrise_allocate_memory`（`:115-121`） |
| `USE_UB` 且 protocol == ub | `ub_allocate_memory`（`:122-126`） |
| `USE_NOF` 且 `use_spdk_dma` | `SpdkWrapper::Alloc`（`:127-132`） |
| `USE_VRAM_SEGMENT` | `nvlink_intra ? allocateFabricMemory_intra : cudaMalloc`（`:133-140`、`:31-59`） |
| 兜底 | `aligned_alloc(alignment, total_size)`（`:142`） |

**free 必须镜像同一条路径**（`client_buffer_allocation.cpp:145-184`），注释（`:166-169`）特别点明：SPDK 大页池取出的内存必须 `spdk_free`，用 glibc `free()` 会 `abort with "free: invalid pointer"`。这是个典型的"分配器必须成对"陷阱，而它之所以危险，是因为 `#ifdef` 分支在错误的构建组合下会静默走错。

`ClientBufferAllocator::~ClientBufferAllocator`（`client_buffer.cpp:77-86`）只在 `!is_external_memory_` 时释放；`create(void* addr, size_t size, ...)`（`:26-30`、`:68-75`）用于外部已有内存（例如共享内存映射），此时 allocator 只做子分配，不负责底层释放。

`MmapArena` 路径（`mmap_buffer.cpp:203-260`）值得单独讲：

```cpp
std::call_once(g_arena_init_flag, initializeGlobalArena);          // :211 每进程只尝试一次
if (g_mmap_arena && g_mmap_arena->isInitialized()) {               // :215
    void* ptr = g_mmap_arena->allocate(total_size, alignment);     // :216
    if (ptr) return ptr;
    // OOM 才降级到直接 mmap（:222-229，LOG_FIRST_N 限流）
}
flags = MAP_PRIVATE | MAP_ANONYMOUS;                               // :235
if (!defer_direct_population) flags |= MAP_POPULATE;               // :236-238
hugepage_size = get_hugepage_size_from_env(&flags);                // :239
map_size = mmap_map_size(total_size, hugepage_size);               // :240
```

而 `free_buffer_mmap_memory`（`:267-297`）用 `g_mmap_arena->owns(ptr)` 区分：

```cpp
if (g_mmap_arena && g_mmap_arena->owns(ptr)) {                     // :275
    g_arena_noop_free_count++;                                     // :276-277
    LOG_FIRST_N(WARNING, 3) << "... does not individually release arena-owned pointer ...";
    return;                                                        // :284 ← 故意不 munmap
}
munmap(ptr, mmap_map_size(total_size, get_hugepage_size_from_env()));  // :288-296
```

**arena 里分配的内存永远不会被单独归还**，只有进程退出时 `MmapArena::~MmapArena`（`mmap_arena.cpp:43-52`）整体 `munmap`。这决定了它只适合"长生命周期、申请后不释放"的场景（SGLang HiCache 的主机 KV 池），不适合当作通用分配器。

`MmapArena::initialize`（`mmap_arena.cpp:54-166`）的决策树：

1. `std::lock_guard<std::mutex> lock(init_mutex_)`（`:60`）——注释（`:56-59`）说明这是为了修掉早期用 CAS 时"输掉的线程覆写 `alignment_/pool_size_`"的元数据竞态。
2. 校验 `pool_size > 0`、`alignment` 是 2 的幂、未初始化过（`:62-75`）。
3. 池大小按 `max(hugepage_size, 2MB)` 对齐（`:82-88`），并做溢出保护。
4. `flags = MAP_PRIVATE|MAP_ANONYMOUS|MAP_POPULATE`（`:94`），若开启大页则加 `MAP_HUGETLB` + 具体 size 位（`:97-104`）。
5. **降级策略**：mmap 失败时，若 `allow_regular_page_fallback == false`（即 `MC_STORE_USE_HUGEPAGE` 显式要求大页）直接失败（`:111-120`）；否则清掉 `MAP_HUGETLB` 与 size 位重试（`:122-141`）。
6. `madvise(MADV_DONTFORK)`（`:149-153`）：`MAP_POPULATE` 意味着所有页都有物理后备，fork 出的子进程会继承一个巨大的 COW 映射（注释举例 64GB），可能直接触发 OOM killer。
7. **发布顺序**：先 relaxed 写 `alignment_`/`pool_size_`，最后 release 写 `pool_base_`（`:155-160`）。`allocate()` 用 acquire 读 `pool_base_`，之后再 relaxed 读另外两个字段——靠 release/acquire 建立 happens-before，这个模式在 `allocate` 的注释里也重复了一遍（`:179-184`）。

`MmapArena::allocate`（`:168-258`）的 CAS 循环有三处容易写错但这里写对了的地方：

- **对齐的是 offset（而不只是 size）**（`:205-213`；`size` 在 `:188-194` 已对齐，随后在 CAS 循环里再把 offset 也对齐），否则"前一次小对齐分配把 cursor 停在非对齐位置"时，后续大对齐请求返回的指针会不满足契约；
- `next = aligned_offset + aligned_size` 后先检查 `next < aligned_offset`（回绕）与 `next > pool_size`（OOM）**再** CAS（`:215-225`）；
- CAS 的 expected 用 `raw`（原始 cursor）而不是 `aligned_offset`（`:230-234`），失败后重新读。

### 流程 E（补充）：GPU fabric 分配器

Python 侧（`mooncake-integration/allocator.py`）：

```python
class MemoryBackend(IntEnum):
    USE_CUDAMALLOC = 0; USE_CUMEMCREATE = 1; UNKNOWN = -1; UNSUPPORTED = -2   # :17-21

NVLinkAllocator.get_allocator(device)                                        # :82-89
  → CUDAPluggableAllocator(so_path, "mc_allocator_malloc", "mc_allocator_free")
```

注意：**`allocator.py` 里没有任何分配逻辑**。它只做两件事：(1) 用 `probe_allocator_backend`（`fabric_allocator_utils.py:33-58`，本质是 `ctypes.CDLL` + `getattr(symbol)` + 调一次；符号缺失时返回 `unsupported_value` 而不抛）探测后端类型；(2) 把 `.so` 的两个 C 符号注册成 torch 的 pluggable allocator，之后真正的分配由 torch 在需要时回调。

C 侧（`mooncake-transfer-engine/nvlink-allocator/nvlink_allocator.cpp`）：

```
mc_allocator_probe(device_id)                                    (:168-170)
  → ProbeAllocatorBackend                                        (:24-53)
      ├─ cuDeviceGet 失败 → unknown                              (:26-29)
      ├─ CU_DEVICE_ATTRIBUTE_HANDLE_TYPE_FABRIC_SUPPORTED 为 0 → use_cudamalloc  (:31-36)
      └─ 用 4KB 做一次 cuMemCreate(CU_MEM_HANDLE_TYPE_FABRIC) 试分配        (:38-52)
           成功 → cuMemRelease + use_cumemcreate；失败 → use_cudamalloc

mc_allocator_malloc(size, device, stream)                        (:172-174)
  → AllocateFabricMemory                                         (:55-135)
      ├─ cuDeviceGet                                              (:63-67)
      ├─ prop = PINNED / DEVICE / location.id = dev               (:68-70)
      ├─ fabric_supported ? prop.requestedHandleTypes = FABRIC    (:72-82)
      ├─ GPU_DIRECT_RDMA_WITH_CUDA_VMM_SUPPORTED → allocFlags.gpuDirectRDMACapable = 1  (:84-91)
      ├─ cuMemGetAllocationGranularity(MINIMUM)，size 向上对齐      (:92-99)
      ├─ cuMemCreateTryFabric                                     (:100-104)
      ├─ cuMemAddressReserve                                      (:105-110)
      ├─ cuMemMap                                                 (:111-117)
      └─ cuMemSetAccess 给**所有**设备 READWRITE                   (:118-133)
```

`cuMemCreateTryFabric`（`:7-18`，注明 ref nvidia/nccl v2.28.9-1 的 `allocator.cc`）是关键的降级点：当请求了 FABRIC handle 但驱动返回 `CUDA_ERROR_NOT_PERMITTED`/`NOT_SUPPORTED` 时，清掉 `CU_MEM_HANDLE_TYPE_FABRIC` 位再试一次。这样同一份代码在"支持 fabric"和"只有普通 VMM"两种机器上都能工作。

`cuMemSetAccess` 遍历 `device_count` 把**每一张** GPU 都设为可读写（`:118-126`），这是跨节点 NVLink 共享 KV 池能成立的前提——任何一张卡都能直接访问这块显存。

`FreeFabricMemory`（`:137-158`）的顺序是 `cuMemRetainAllocationHandle` → `cuMemGetAddressRange` → `cuMemUnmap` → `cuMemAddressFree` → `cuMemRelease`。注释（`:150`）提示"不是所有 runtime 都接受 NULL 的 base 输出，总是传一个真实指针"，这是踩过的坑。

构建侧（`mooncake-transfer-engine/fabric_allocator.cmake:1-44`）：`add_fabric_allocator_build_target` 用 `add_custom_command` 调外部 `bash ${FAB_BUILD_SCRIPT}` 在 `CMAKE_CURRENT_BINARY_DIR` 下产出 `.so`，并 `add_dependencies(${FAB_TARGET_NAME} transfer_engine)`。之所以走外部脚本，是因为要支持用 `hipcc`/`mcc`/`maca` 编译同一份 `.cpp`——这些编译器切换不适合塞进 CMake 的编译器探测。

### 流程 F（补充）：指标链路

`AllocatorMetric::Refresh()`（`allocator_metric.cpp:173-186`）：

```cpp
std::lock_guard<std::mutex> guard(refresh_mutex_);            // :175
if (now - last_refresh_ < kMinRefreshInterval) return;        // :176-178 1s 限流
last_refresh_ = now;  RefreshLocked();                        // :179-180
```

限流的原因写在头文件（`allocator_metric.h:110-118`）：推进 jemalloc epoch 会拿一把与分配线程共享的锁，如果 `/metrics` 是无认证端点，高频抓取会直接打击分配性能。

`RefreshProcess()`（`:188-215`）逐行读 `/proc/self/status`，把 `VmRSS` / `VmHWM` / `RssAnon` / `RssFile` / `RssShmem` / `VmSize` / `VmSwap` 的 kB 值乘 1024 后灌进 gauge。

`RefreshJemalloc()`（`:217-272`）先从原子变量取采集器指针：

```cpp
const JemallocStatsCollector collector = g_jemalloc_collector.load(std::memory_order_acquire);  // :218-219
if (collector == nullptr || !collector(snapshot)) { jemalloc_enabled_.update(0); return; }      // :221-224
```

**这一行解释了为什么"库本身不链接 jemalloc"也能有指标**：`SetJemallocStatsCollector`（`:40-42`）是个全局原子函数指针，由链接了 jemalloc 的可执行文件在 `main()` 里注入：

```
master.cpp:1432-1433        LogAllocatorStatus(); InstallAllocatorStatsCollector();
real_client_main.cpp:128-129  同上
```

`InstallAllocatorStatsCollector`（`allocator_status.h:220-222`）把 `CollectJemallocStats`（`:94-212`）注册进去；未链接 jemalloc 时它和 `LogAllocatorStatus`（`:226-228`）都是空函数，所以调用点不需要 `#ifdef`。**没装 collector 时 `mooncake_jemalloc_enabled = 0` 是正确值，不是故障**。

`AdvanceCounter`（`allocator_metric.cpp:274-283`）把 jemalloc 的累计值转成 ylt counter：

```cpp
if (total < previous) return;                 // :276-280 采样回退时保持单调，丢弃本次
counter.inc(total - previous);  previous = total;
```

`CollectJemallocStats` 的两个前置检查也值得记：`config.stats` 必须为真（`allocator_status.h:98-101`，否则"每次 stats.* 读都失败、整份快照看起来像健康的 0"），`arenas.page != 0`（`:110-113`，否则 dirty/muzzy 的页数换算成字节会静默变 0）。

## 与其他模块的边界

**向上（谁使用分配器）**

| 使用方 | 用的什么 | 接口 |
|---|---|---|
| `SegmentManager` / `ScopedSegmentAccess` | 段分配器 | `CreateBufferAllocator`、`AttachUsageTracker`、`AllocatorManager::addAllocator`（`segment.cpp:142-158`、`:255-264`） |
| `MasterService::PutStart` 等写路径 | `AllocationStrategy` | `allocation_strategy_->Allocate(...)`（`master_service.cpp:5196`、`:5229`、`:9896`） |
| `SegmentPool` / `RegionDriver`（新版快照路径） | `CaptureSnapshot` / `Restore` / `Import*`（`segment/snapshot.cpp:12-41`、`region_driver.cpp:215-245`） | |
| `FileStorage` / `AlignedClientBufferAllocator`（O_DIRECT） | `allocate_buffer_allocator_memory`（4096 对齐）（`aligned_client_buffer.cpp:37-56`） | |
| `ClientBufferAllocator` | `OffsetAllocator` 子分配 + `allocate_buffer_mmap_memory` | `client_buffer.cpp:47-65` |
| `RealClient` | `TryPinStoreSegment` / `ReleasePinnedRegionForFree`（`real_client.cpp:156-177`） | |
| `store_py.cpp` | `hugepage_memory_alloc` 的函数地址（`memory_alloc.cpp:7-19`） | |
| Python 推理进程 | `nvlink_allocator.so` 的两个 C 符号（`allocator.py:82-89`） | |

**向下（分配器依赖谁）**

- vendored CacheLib 子集：`mooncake-store/{include,src}/cachelib_memory_allocator/`，独立 STATIC 库，自带 `fmt`/`folly` 头桩；
- sebbbi/OffsetAllocator 的移植版（MIT），Mooncake 加了 thread-safe 外壳、multiplier、快照、round-up；
- yalantinglibs：`ylt::metric::{gauge,counter,hybrid_gauge_1t}`、`tl::expected`、`YLT_REFL`；
- `MasterMetricManager`（每段 allocated/capacity 指标）、`StorageUsageTracker`（水位）；
- msgpack（HA 快照）、`Serializer<T>` 模板（`include/serialize/serializer.h`）；
- CUDA Driver/Runtime（`cuMemCreate`/`cuMemMap`/`cudaHostRegister`/`cudaMallocHost`）、`cuda_alike.h` 多厂商别名；
- libnuma / hugetlb / THP（`mmap_buffer.cpp:178-434`）；
- SPDK（`USE_NOF`）、libacl（Ascend）、ubshmem、`libaccl_barex`、Sunrise `tang_runtime`；
- gflags/glog/fmt。

**关键契约**

1. **`base` 的语义随调用方而变**。同一个 `OffsetAllocator` 类在四个地方出现：
   - `OffsetBufferAllocator`：`base` 是**客户端上报的远端虚拟地址**（Master 不能解引用）；
   - `ClientBufferAllocator`：`base` 是**本进程的真实指针**（`client_buffer.cpp:64-65`）；
   - `OffsetAllocatorStorageBackend`：`base = 0`，分配的是**文件内字节偏移**（`src/storage_backend.cpp:5570` 的 `OffsetAllocator::create(0, capacity_)`；`ObjectEntry::offset` 见 `include/storage_backend.h:1388-1390`，是 `uint64_t` 文件偏移）；
   - DFS：同样 `base = 0`，分配的是 **shard 文件内偏移**（`src/storage/distributed/shard_allocator.cpp:237`）。
   改任何一处逻辑前，先确认 `base` 指向什么。
2. **`AllocatedBuffer` 的 `size()` 是请求大小，不是占用大小**。CacheLib 侧实际占用 >= `max(size, 64)` 且向上取整到 size-class；Offset 侧实际占用 = `normalizedAllocationSize(size)`（round up 到 bin，再乘 quantum）。这直接影响 `size()/capacity()` 的利用率读数。
3. **Descriptor 是跨进程契约**：`{size, buffer_address, protocol, transport_endpoint}`。`protocol == "cxl"` 时 `buffer_address` 是相对 `DEFAULT_CXL_BASE` 的偏移（`allocator.cpp:118-128`）。
4. **恢复接口不发布半成品**：`Restore` / `Import*` 全部"先校验、再构造、最后才记账"（`allocator.cpp:439-460`、`:648-650`）。

## 设计取舍与易踩坑点

### 6.1 为什么默认 OFFSET 而不是 CACHELIB

| 维度 | CACHELIB | OFFSET |
|---|---|---|
| 段对齐要求 | `base`、`size` 都必须 16MB 对齐（`allocator.cpp:17-24`、`segment.cpp:245-252`） | 无（只要求 `base + size` 不溢出） |
| 内部碎片 | slab 内按 size-class 切，同一 slab 只能服务一个 class；class 间不共享（`Slab.h:34-52`，`MemoryAllocator.h:31-53` 的层级说明） | bin 之间按物理相邻合并，碎片可回收 |
| 单对象上限 | `kMaxSliceSize = 16MB - 16`（`types.h:451`），且 `PutStart` 在 CACHELIB 下显式拒绝更大对象（`master_service.cpp:5506`） | 无 16MB 限制，只受 `MAX_BIN_SIZE`(3.75GB) 与 quantum 约束 |
| 最大空闲块查询 | 只能返回 `kAllocatorUnknownFreeSpace`（`allocator.h:301-308`）——因为 CacheLib 不暴露"最大连续空闲" | `storageReport().largestFreeRegion` 精确可查（`allocator.cpp:548-565`） |
| 并发粒度 | 每个 AllocationClass 一把锁 + MemoryPool 锁 + SlabAllocator 锁（`AllocationClass.h:384`、`MemoryPool.h:317`、`SlabAllocator.h:174`） | 每段一把锁（`offset_allocator.h:233`），临界区短（bitmap + 链表指针） |
| 元数据位置 | `SlabHeader` 数组在 Master 本地堆（`allocator.cpp:199-203`） | `Node` 数组在 Master 本地堆（`offset_allocator.cpp:175-179`） |
| 恢复方式 | `importAllocations` 写 chunk 位图 | 快照全图校验 / 地址重放 |
| HA 序列化支持 | 不支持（`segment.cpp:761-765` 明确拒绝非 OFFSET 的 SegmentManager 序列化） | 支持（`AllocatorSnapshotCodec`） |

结论：CACHELIB 的价值在于"成熟、被 Facebook 验证过"，但它的 16MB slab/class 语义与 Mooncake 的对象尺寸分布不匹配（LLM KV 对象常常远小于 16MB，且尺寸相近），加上对齐约束和不可查询空闲块，所以默认走 OFFSET。中文文档 `docs/source/zh_archive/mooncake-store.md` 也把 CacheLib 标为不推荐。

**但要注意历史残留**：`SegmentManager` 的构造函数默认参数（`segment.h:387`、`:488`）和 `RegionDriverConfig::memory_allocator`（`segment/region_driver.h:117`）仍写着 `CACHELIB`。真正的默认值在 `MasterServiceConfig`（`master_config.h:275`/`:1255`/`:885`）——`OFFSET`。读代码时别被构造函数默认参数带偏，要看谁实际传参（`master_service.cpp:209` 把 config 的值交给 `memory_allocator_type_`）。

### 6.2 round-up-to-bin 是"用一点浪费换碎片可控"

见流程 B 的 B3。补充两点：

- 这个改动让**分配尺寸统一化**：所有落进 bin B 的请求都占用 `floatToUint(B)` 字节，free 时能精确落回 bin B，从而与同 bin 的邻居合并概率大增。
- 它也让 `getLargestFreeRegion()` 在语义上精确：因为任何大于 `floatToUint(B_max)` 的请求都会 round up 到更高的 bin，而更高 bin 一定为空。所以"返回 bin 下界"与"返回最大可服务请求"等价。测试 `tests/buffer_allocator_test.cpp:128-143` 就是断言这一点（并顺带区分了"提示值"与"权威值"）。

### 6.3 `multiplier_bits`：不要直接把 offset 当字节

超过 3.75GB 的段会引入 `m_multiplier_bits`（`offset_allocator.cpp:539-545`、`:853-863`）。所有 `Node::dataOffset/dataSize`、`m_size`、`m_freeStorage`、`OffsetAllocation::offset` 都是 quantum 单位；只有 `OffsetAllocationHandle::real_base`（`offset_allocator.h:112`）与 `OffsetAllocator::m_base/m_capacity` 是字节。换算公式：

```
real_addr = m_base + (offset << multiplier_bits)
real_size = size << multiplier_bits
real_qty  = (real_qty_bytes + (1<<bits) - 1) >> bits    // 向上取整，见 allocate
```

`normalizedAllocationSize`（`:937-952`）把整个"字节 → quantum → bin round-up → 字节"的链条封装成一个函数，`fill_gap` 的正确性完全依赖它与 `allocate` 的一致性。

### 6.4 元数据上限是第二种 OOM

`m_freeOffset == m_max_capacity` 时 `__Allocator::allocate` 直接返回 `NO_SPACE`（`offset_allocator.cpp:193-196`）。对 Master 侧，`max_capacity` 被夹在 `1<<26`：

- 64GB 段 → 6710 万节点，平均 1KB/节点；
- 大于 64GB 的段 → 仍只有 6710 万节点，平均粒度变大。

所以**大段 + 小对象**的组合会先耗尽节点而不是字节。排查 "`allocation_failed` 但 `current_size` 远小于容量" 时，必须同时看节点数。

另外注意 `m_nodes.reserve(m_max_capacity)`（`:175`）只是预留虚拟地址空间；实际 RSS 随写入的节点增长。不要据此估算内存占用。

### 6.5 `getLargestFreeRegion` 有两种语义

- `OffsetAllocator::getLargestFreeRegion()`（`offset_allocator.h:182-184`）：**无锁提示**，可能偏大（成功分配后不缩小），失败后被收紧，free 时被抬高。只用于快速失败。
- `OffsetBufferAllocator::getLargestFreeRegion()`（`allocator.cpp:548-565`）：**加锁读权威值**（转调 `storageReport()`）。
- `CachelibBufferAllocator::getLargestFreeRegion()`：恒返回 `kAllocatorUnknownFreeSpace`（`allocator.h:306-308`），表示"不知道"，调用方必须按"可能装得下"处理。

顺带说明一个当前状态：`BufferAllocatorBase::getLargestFreeRegion()` 这个虚函数在 store 主路径上**目前没有调用点**（grep 全仓只有定义与测试），`AllocationStrategy` 的候选过滤实际用的是 `capacity()/size()`。头文件里"used for filtering"的注释属于设计意图而非既成事实。

### 6.6 两套记账、两个"用量"

- `BufferAllocatorBase::size()` / `cur_size_`：请求字节和，供 `FreeRatioFirstAllocationStrategy` 与 `GetMemoryUsageSnapshot` 使用。
- `MasterMetricManager::inc_allocated_mem_size(segment, bytes)`：Prometheus 每段指标，供运维观察。

两者都按**请求大小**记，因此都会**低估**真实占用（CacheLib 的 size-class、Offset 的 bin round-up、以及 64B 最小分配）。做容量规划时不要拿它当物理占用。

`AllocatedBuffer` 析构时如果 `weak_ptr` 已过期，`deallocate` 不会被调用，指标也不会减——但此时 allocator 的析构函数（`allocator.cpp:224-233` / `:469-478`）会 `dec_allocated_mem_size(segment_name_, size())` 一次性地把该段的量减掉，并 `remove_segment_metrics`。两条路径不会重复扣减，因为 allocator 死了之后 buffer 也不可能再持有它。

### 6.7 MmapArena 的 free 是 no-op（不是 bug）

见流程 D。三个直接后果：

1. `free_buffer_mmap_memory` 对 arena 指针**不 munmap**，只累加 `g_arena_noop_free_count` 并限流告警（`mmap_buffer.cpp:275-285`）。
2. 指针被 `owns()` 判定为 arena 的，就不能再传给普通 `munmap`；反过来，非 arena 指针走 `munmap(ptr, mmap_map_size(total_size, ...))`，**size 必须与分配时一致**，所以 `mmap_map_size` 被抽成公共函数（`mmap_buffer.cpp:72-76`）。
3. arena 初始化"每进程只尝试一次"（`std::call_once`，`:211`/`:273`），失败即永久回退直接 mmap。日志明确写了"This is only attempted once per process"（`:62-64`）。

arena 是 opt-in：`--use_mmap_arena_allocator` 或 `MC_MMAP_ARENA_POOL_SIZE`（`mmap_buffer.cpp:23-26`、`src/config/mmap_arena_config.cpp:24-48`）；`MC_DISABLE_MMAP_ARENA` 可强制关闭。默认池大小 8GiB（`mmap_buffer.cpp:25-26`）。

### 6.8 hugepage："静默降级"还是"硬失败"

由 `allow_regular_page_fallback` 决定，而这个参数来自 `MC_STORE_USE_HUGEPAGE`：

- 未设置 → 尝试大页，失败则普通页重试并 `LOG(WARNING)`（`mmap_arena.cpp:122-138`）；
- 显式设置 → 失败直接返回 false（`:111-120`），日志明确写 "regular-page fallback disabled because MC_STORE_USE_HUGEPAGE explicitly requested hugepages"。

`MmapArenaConfig::FromEnvironment` 里的注释（`src/config/mmap_arena_config.cpp:51-56`）把这条契约写成了两条状态机，改代码时不要破坏它。

### 6.9 `RegisteredPinnedMemoryManager` 的锁外注册与"故意泄漏"

`try_pin`（`registered_pinned_memory.cpp:90-170`）的并发设计：

1. 先在锁内做**重叠检测**（`start < region_end && end > region_start`，`:114-122`）与**配额扣减**（`size > limit || pinned > limit - size`，`:124-130`），并把 `{addr, size, nullptr}` 压进 `regions_`，`pinned_bytes_ += size`；
2. **释放锁之后**才调 `cudaHostRegister`（`:141-143`）——因为这是慢系统调用，不能持锁；
3. 失败则重新拿锁 `remove_inactive_region_locked` 回滚（`:144-153`）；
4. 成功则再拿锁把 `entry.region` 指向自己（`:155-165`）。

中间的 `region == nullptr` 是一个明确的"占位中"状态：其它线程的重叠检测会把这段当成已占用（这正是想要的，防止两个人同时 pin 同一段），但 `remove_inactive_region_locked`（`:211-220`）只清理 `region == nullptr` 的条目。

`release`（`:172-209`）的行为更微妙：

- 先把自己那条的 `region` 置空并记 `should_unregister`，**释放锁**后调 `cudaHostUnregister`；
- 若返回 `kRuntimeUnloading`（`cudaErrorCudartUnloading`，`registered_pinned_memory.cpp:36-39`），只 `LOG(WARNING)` 然后**当作成功**继续走清理（`:192-204`）——因为此时 CUDA runtime 正在卸载，调任何 API 都可能崩；
- 若返回 `kError`，`LOG(ERROR)` 并 **`return false`**，条目保留（`region` 已是 nullptr，`pinned_bytes_` 不减）。调用方 `ReleasePinnedRegionForFree`（`real_client.cpp:166-177`）看到 false 就打 `"Leaking ... backing memory because cudaHostUnregister failed"` 并**放弃释放内存**。

也就是说退化路径是"泄漏一段内存 > 在 runtime 卸载期崩溃"。段地址区间也会继续保持"被 pin 占用"的记账，防止后续重复注册同一段。

配置项是 `MC_STORE_PIN_MEMORY_MAX_BYTES`（`src/config/registered_pinned_memory_config.cpp:16-32`），默认 0 = 关闭（`src/config/registered_pinned_memory_config.h:8`）。`limit_bytes_ == 0` 时 `try_pin` 直接返回 nullptr（`:92`），所以这是一个显式 opt-in 的功能。

### 6.10 CXL 的三个特殊之处

1. **分配器类型硬编码 CACHELIB**，与配置无关（`segment.cpp:211-236`、`region_driver.cpp:303-320`）。
2. **`base` 固定为 `DEFAULT_CXL_BASE`**（`segment.cpp:1556`），所以 `AllocatedBuffer::change_to_cxl` 能把指针转成相对偏移（`allocator.cpp:118-123`），`get_vaddr_from_cxl` 再转回来。这也意味着 CXL buffer 的 `data()` **不能被直接解引用**。
3. **CXL 段共享一个全局 allocator**：`cxl_global_allocator_` 由 `SegmentManager` 独占持有，可被多条 segment 记录引用，所以 `AttachMountedUsageTrackers` 要去重（`segment.cpp:142-158`），`GetMemoryUsageSnapshot` 也要用 `counted_allocators` 去重（`:126-141`）。

`CxlRegionDriver::PrepareAdopt` 直接返回 `UNAVAILABLE_IN_CURRENT_MODE`（`region_driver.cpp:296-302`），即 CXL 段不支持快照恢复（`PrepareOpen` 也拒绝非空 `live_allocations`，`:282-285`）。

### 6.11 `CreateBufferAllocator` 并非"无异常"

`allocator.cpp:158-183` 返回 `tl::expected`，注释（`:174-175`）说 "Offset construction has no parameter-dependent throwing path; metadata allocation failures intentionally follow fail-fast"。但 `OffsetBufferAllocator` 的构造函数在 `OffsetAllocator::create` 失败时会 `throw std::runtime_error`（`:383-386`），而 `CreateBufferAllocator` 并没有 `try/catch`。也就是说：**元数据分配失败会以异常形式穿过这个 `tl::expected` 接口**。这是刻意的 fail-fast 策略（注释里写了"metadata allocation failures intentionally follow fail-fast"），但对调用方来说，"返回 expected 的函数不会抛"这个直觉在这里不成立，写错误处理时要知道。

CacheLib 侧同理：`CachelibBufferAllocator::Create` 只把布局非法转成 `INVALID_PARAMS`，构造过程中的 `bad_alloc` 不会被捕获（`:150-155` 的注释明确说"Do not catch allocation failures here"）。

### 6.12 名字带 allocator 但不是内存分配器

- `mooncake-store/src/memory_alloc.cpp`：全文 19 行，`hugepage_memory_alloc` 在非 `USE_NOF` 构建下直接 `return nullptr`（`:7-13`）。它的存在意义是被 `store_py.cpp` 以**函数地址**形式导出给 Python（SGLang 的 NoF/大页自定义内存池钩子），不是通用分配入口。
- `mooncake-store` 的 `allocator_metric.cpp` / `allocator_status.h`：进程 RSS + jemalloc，见 §3.12。
- `mooncake-store-rs/.../client/runtime_alloc.rs`：解析副本写目标，不是分配器。
- `mooncake-store/include/placement/replica_allocator.h`：副本放置需求。
- `mooncake-transfer-engine/tent/src/platform/*_allocator.cpp`：TENT 新传输引擎自己的 CPU/CUDA/ROCm/Ascend/Sunrise 分配体系，与 store 段分配器不是一条线。
- `SimpleAllocator`（`allocator.h:460-476`，实现 `allocator.cpp:653-765`）：唯一一个**分配真实内存**的 CacheLib 用户——它自己 `aligned_alloc` 出 `base_`，再在上面建 CacheLib allocator。仅测试代码使用（`tests/batch_remove_test.cpp:99`、`tests/client_integration_test.cpp:258`），不是生产路径。
- `DummyBufferAllocator`（`allocator.h:234-259`）：`allocate` 返回 `nullptr`，只为在 standby 提升时保留 endpoint 元数据（`master_service.cpp:3572`、`:3864`、`:4040`）。

## 精读清单

按这个顺序读，每一步都标注"该看懂什么"：

1. **`mooncake-store/include/allocator.h` 全文（480 行）**
   重点 `AllocatedBuffer`（:70-177）与 `BufferAllocatorBase`（:183-227）。
   *看懂*：`weak_ptr` 回指、`SegmentLifetime`/`ClientLivenessRecord` 双绑定、`size()` 是请求字节、`Descriptor` 四字段。
2. **`mooncake-store/src/allocator.cpp:1-130`**
   `IsValidCachelibLayout`、`AllocatedBuffer` 的构造/析构/descriptor/cxl 转换。
   *看懂*：CACHELIB 的 16MB 对齐硬约束从哪来、CXL 指针为何是偏移。
3. **`mooncake-store/src/offset_allocator.cpp:56-130`（SmallFloat）+ `:191-357`（allocate/free）**
   *看懂*：三个转换函数的方向（roundUp/roundDown/floatToUint）、两级位图查找、`roundupSize` 的动机、free 的双向合并、`m_freeOffset` 栈语义。
4. **`mooncake-store/src/offset_allocator.cpp:538-545` + `:853-935`**
   `calculateMultiplier` 与 `OffsetAllocator::allocate`。
   *看懂*：quantum 换算、无锁快速失败 + 失败后收紧、unlock 后再构造 handle。
5. **`mooncake-store/src/offset_allocator.cpp:1001-1051`**
   `refreshLargestFreeRegion` + `freeAllocation`。
   *看懂*：提示值的三条更新规则、饱和减法。
6. **`mooncake-store/src/allocator.cpp:352-565`**
   `OffsetBufferAllocator` 的构造/分配/释放/`Restore`/`CaptureSnapshot`。
   *看懂*：`init_capacity/max_capacity` 的推导、`Restore` 的记账顺序、`size()` 与真实占用的差。
7. **`mooncake-store/src/offset_allocator.cpp:547-818`（快照校验）**
   六个 `Validate*`。
   *看懂*：为什么 `multiplier_bits` 必须重算校验、为什么 `allocated_size` 只能区间校验、邻居链如何证明"节点铺满整个布局"。
8. **`mooncake-store/src/allocator.cpp:567-651`（重放式恢复）**
   *看懂*：`fill_gap` 的二分、"任何时刻只有一个空闲节点"的不变量、`gaps.clear()` 为什么能让布局逐字节还原。
9. **对照读 `mooncake-store/src/allocator.cpp:310-349` 与 `src/cachelib_memory_allocator/MemoryAllocator.cpp:90-168`**
   *看懂*：chunk 位图恢复 vs 地址重放恢复，为什么后者对地址对齐要求更严。
10. **`mooncake-store/src/segment.cpp:142-158, 171-270, 1546-1568`**
    挂载路径、CXL 特例、tracker 补挂。
    *看懂*：一个 segment 从"客户端上报"到"可被策略选中"的全过程。
11. **`mooncake-store/include/allocation_strategy.h:33-120, 423-452, 573-618`**
    *看懂*：`AllocatorManager` 无锁、`registration->Allocate` 是唯一入口、`FreeRatioFirst` 的水位依赖 `size()`。
12. **`mooncake-store/src/segment_allocator_registration.cpp` 全文（84 行）**
    *看懂*：两条 lifetime 的区别、分配后二次校验。
13. **`mooncake-store/src/mmap_arena.cpp:54-166` + `src/common/mmap_buffer.cpp:199-297`**
    *看懂*：hugepage 决策树、`MAP_POPULATE`/`MADV_DONTFORK` 的理由、release/acquire 发布顺序、arena no-op free 的识别。
14. **`mooncake-store/src/common/client_buffer_allocation.cpp` 全文**
    *看懂*：protocol 分发表，以及 free 必须镜像 allocate。
15. **`mooncake-store/src/registered_pinned_memory.cpp:90-220`**
    *看懂*：锁外注册、占位状态、`kRuntimeUnloading` 的泄漏选择。
16. **`mooncake-store/src/allocator_metric.cpp:173-283` + `include/allocator_status.h:94-222`**
    *看懂*：1s 限流的动机、函数指针注入、cumulative→counter。
17. **`mooncake-transfer-engine/nvlink-allocator/nvlink_allocator.cpp:7-158`**
    *看懂*：fabric 探测（4KB 试分配）、`cuMemCreateTryFabric` 的降级、`cuMemSetAccess` 全设备授权、释放顺序。
18. **最后跑一遍 `mooncake-store/tests/buffer_allocator_test.cpp`**
    `OffsetLargestFreeRegionRemainsExact`（:122-143）、`ImportOffsetAllocationsAtOriginalAddresses`（:145-183）、`ImportOffsetAllocationsHasNoArbitraryGapLimit`（:244-260）、`CachelibCreateRejectsInvalidMemoryLayout`（:264+）。
    *看懂*：这些测试实际上把本章流程 B / 流程 C 的关键不变量都钉死了，是最快的交叉验证。

---

**未完全确认的两处**：

- `__Allocator::Node` 的 `sizeof` 未在代码中显式断言。按字段（6 个 `uint32` + 1 个 `bool`，4 字节对齐）推算约 28 字节；`m_nodes.reserve(m_max_capacity)` 的虚拟地址预留量约为 `28 × max_capacity`，但这是推算而非实测（代码里既没有 `static_assert`，也没有 `#pragma pack`），不要当作精确值引用。
- `AttachUsageTracker` 的"只允许在发布前调用一次"是**约定**而非强制：`BufferAllocatorBase::AttachUsageTracker` 只是在 `usage_registration_` 非空时静默返回（`allocator.cpp:35-42`），不会断言也不会替换 tracker。若有人对同一个 allocator 先后挂两个不同 tracker，第二次会被无声忽略。旧路径（`SegmentManager::AttachMountedUsageTrackers`）与新路径（`pool_mount.cpp` 的 `PrepareWithLiveAllocations`/`PublishMount`）都遵守这个约定，但契约本身靠调用方自觉。

---

# 11 Store 特性层

> 本章基于仓库 `D:/xiaojun/work/Mooncake`（commit `67df8c41`）的实际源码精读写成。所有技术断言尽量带 `相对路径:行号`，可直接跳转核对；凡是没能从源码闭环确认的地方，文中明确标注"未完全确认"。
>
> **路径约定**：本章的相对路径一律以仓库根为基准。需要强调：范围提示里写的 `src/tenant/`、`tenant_quota_ledger.cpp`、`kv_event/`、`engram/` 等都不在仓库根，而在 **`mooncake-store/src/` 与 `mooncake-store/include/`** 之下。另外 `docs/source/design/store/store-rs/` 讲的是另一套 Rust 实现（`mooncake-store-rs`），与本层无关，读文档时最容易张冠李戴。

---

## 定位与职责

`mooncake-store` 的 Master（`MasterService`）本体的职责是"对象元数据 + 路由 + 分配"：`PutStart/PutEnd/GetReplicaList/Remove`、segment 管理、eviction、HA 复制。**特性层不实现对象数据面**，它是在这套核心之上挂的一组可开关组件，每个组件都只在某个配置标志打开时才产生行为：

| 特性 | 解决的问题 | 开关（默认） | 宿主 |
|---|---|---|---|
| 多租户配额 | 多个租户共享同一内存池时，给每个租户一个"承诺容量"，写路径上原子准入，超额拒绝 | `enable_multi_tenants`（false） | `MasterService` + `TenantQuotaManager` |
| KV 事件 | 把对象"哪一层介质可用/被删除/整租户被清空"以 RFC #1527 格式经 ZMQ 推给外部 indexer | `enable_kv_events`（false），且需编译期 `MOONCAKE_ENABLE_KV_EVENTS` | `KvEventPublisher` |
| 权重专用存储 | 把不可变的模型权重 revision 变成一等资源：身份发现、lease、manifest 校验、两段式导入 | 无 OpLog 时纯内存可用；有 OpLog 时还需 `--weight_management_oplog_capability_confirmed` | `WeightStoreManager` / `WeightMetadataStore` |
| 客户端热缓存 | 客户端把常读对象再缓存一份到本地（含跨进程 shm），读命中直接本地 memcpy | 环境变量 `MC_STORE_LOCAL_HOT_CACHE_SIZE`（0=关） | `LocalHotCache` + `LocalHotCacheHandler`（**客户端**） |
| 晋升（promotion on hit） | 只存在于 LOCAL_DISK 上的对象被读到时，异步搬回 MEMORY | `promotion_on_hit`（false），依赖 `enable_offload` | `MasterService` + `PromotionCandidateTracker` |
| 动态副本 | 热对象自动多发一张 MEMORY 副本的租约，由客户端执行 Copy | `dynamic_replication_mode`（"off"） | `DynamicReplicationController` |
| 任务邮箱 | Master 向客户端下发 REPLICA_COPY / REPLICA_MOVE 任务 | 随动态副本/迁移使用 | `ClientTaskManager` |

它在整体中的位置可以概括成一句话：**上接 RPC（`rpc_service.cpp`、`master_admin_service.cpp`），下接元数据分片与 segment 分配器，横向依赖 HA 的 OpLog/快照**。所有特性都寄生在 `MasterService` 的同一把"元数据分片锁"体系里，因此**读任何特性代码之前必须先读锁序契约**：

```
mooncake-store/include/master_service.h:101-118
1. client_mutex_
2. tenant_quota_manager_ 的 policy mutex
3. master_state_mutex_
4. metadata_shards_[shard_idx_].mutex
5. tenant_quota_manager_ 的 recompute mutex
6. ShardedTenantQuotaTable 内部 mutex 或 segment_mutex_
7. soft_pin_deadline_index_ mutex
```

注释还补了两条关键约束：需要同时拿"租户策略锁"和 `master_state_mutex_` 的路径必须**先策略锁后状态锁**；recompute mutex 串行化"容量快照 + 配额表更新"这一对操作，而 segment mutex 在进入配额表前已经释放，所以 5/6 两层不会嵌套。

一个贯穿全层的设计事实是：**特性状态大多不放在特性组件里，而挂在对象记录 `ObjectRecord` 上**（`mooncake-store/include/master_service.h:1102-1112`）：

```cpp
struct ObjectRecord {
    route::Generation generation;   // 同一 key 的每次发布唯一
    ObjectMetadata metadata;        // 内含 TenantQuotaLedger
    route::ObjectState state;       // 内含 promotion/dynamic 的 pending 与 candidate
};
```

`ObjectMetadata` 是 **non-copyable / non-movable**（`mooncake-store/include/object_metadata.h:95-98`），因此 `TenantQuotaLedger` 才能安全内嵌其中（`object_metadata.h:160`），`promotion_candidate` / `dynamic_replication_pending` 也就地存活、随对象一起销毁。所有 `*Locked` 后缀方法（`RecordLocked`、`ClearPendingLocked`、`DueLocked`…）的语义统一是：**调用者已经持有该对象所在元数据分片的锁**，组件本身只做规则判断和索引维护。

---

## 目录结构速览

| 文件 | 一句话职责 |
|---|---|
| `mooncake-store/include/master_service.h` | 特性层总宿主；101-118 行锁序契约；成员含 `tenant_quota_manager_`、`weight_manager_`、`promotion_candidates_`、`dynamic_replication_`、`task_manager_`、`kv_event_publisher_` |
| `mooncake-store/src/master_service.cpp` | 13.7k 行，所有特性的集成调用点（准入、结算、事件发布、晋升、动态副本、权重转发） |
| `mooncake-store/include/tenant/quota.h` | `TenantQuotaAccount`（准入位与计费字节共用一个原子字，有效配额另用一个原子字）与 `TenantQuotaTable`（每租户稳定账户注册表） |
| `mooncake-store/src/tenant/quota.cpp` | `TryCharge/Release` 的无锁实现；超额订阅时按比例缩减有效配额 |
| `mooncake-store/include/tenant/quota_sharded.h` + `quota_sharded_impl.h` | 1024 分片线程安全外壳，跨分片操作用 `recompute_mutex_` 串行化 |
| `mooncake-store/include/tenant/quota_manager.h` + `src/tenant/quota_manager.cpp` | 策略锁、策略持久化、`Recompute`/`RebuildUsage`，对外唯一入口 |
| `mooncake-store/src/tenant/quota_policy_store.cpp` | YAML 文件（原子写 + fsync 目录）与 etcd 两种连接器 |
| `mooncake-store/include/tenant_quota_ledger.h` + `src/tenant_quota_ledger.cpp` | **每对象**配额账本：pending/committed/replaced 三桶的结算、退款、转移 |
| `mooncake-store/include/kv_event/kv_event_config.h` / `kv_event_publisher.h` / `src/kv_event/kv_event_publisher.cpp` | KV 事件信封、有界 drop-oldest 队列、msgpack+ZMQ 三帧发布；关闭时是头文件内联 no-op |
| `mooncake-store/include/weight_management.h` | 权重领域类型、状态机合法性校验、canonical key 推导、SHA-256 |
| `mooncake-store/include/weight_metadata_store.h` + `src/weight_metadata_store.cpp` | 内存版权重 revision/lease/operation 存储（1328 行，Prepare*/Publish 两段式） |
| `mooncake-store/include/weight_store_manager.h` + `src/weight_store_manager.cpp` | 4096 分段组锁 + "先持久化后发布" |
| `mooncake-store/include/weight_store_backend.h` + `src/master_store_backend.cpp` | 权重持久化抽象到 `MasterService` OpLog 的桥 |
| `mooncake-store/include/local_hot_cache.h` + `src/local_hot_cache.cpp` | 客户端 LRU 热缓存、detached fill、令牌校验、异步 handler |
| `mooncake-store/src/client_service.cpp` | 热缓存读路径 `RedirectToHotCache`、写回 `ProcessSlicesAsync`、`InitLocalHotCache` |
| `mooncake-store/src/config/local_hot_cache_config.cpp` | `MC_STORE_LOCAL_HOT_*` 环境变量 → `LocalHotCacheConfig` |
| `mooncake-store/include/promotion_candidate_tracker.h` + `src/promotion_candidate_tracker.cpp` | 晋升重试候选的 due 索引 + 重试预算（kMaxRetries=64、kTtl=300s） |
| `mooncake-store/include/dynamic_replication_controller.h` + `src/dynamic_replication_controller.cpp` | 热度窗口、准入线程、选源/选目标、租约表、`PendingCopyGuard` |
| `mooncake-store/include/dynamic_replication_lease_table.h` | 每租户 proposal_id → lease 表 |
| `mooncake-store/include/task_manager.h` + `src/task_manager.cpp` | 按 client_id 组织的任务邮箱（REPLICA_COPY/MOVE）+ HA 序列化 |
| `mooncake-store/include/thread_pool.h` + `src/thread_pool.cpp` | 38 行通用线程池（本层几乎不直接用） |
| `mooncake-store/include/serialize/serializer.h` + `src/serialize/serializer.cpp` | HA 快照的 msgpack 编解码（allocator/segment/replica） |
| `mooncake-store/include/common/` | 纯工具：`zstd_util`、`result.h`、`timestamp.h`、`heap_optional.h` 等 |
| `mooncake-store/src/config/` | 60+ 个"环境变量/gflag → 结构体"加载器，不是业务配置 |

---

## 核心数据结构

### 1. 多租户配额

#### `TenantQuotaAccount`：把"准入位 + 已计费字节"压进一个 `atomic<uint64_t>`

```cpp
// mooncake-store/include/tenant/quota.h:50-82
static constexpr uint64_t kAdmissionClosed = 1ULL << 63;
static constexpr uint64_t kChargedBytesMask = kAdmissionClosed - 1;
alignas(64) std::atomic<uint64_t> charged_state_{kAdmissionClosed};
alignas(64) std::atomic<uint64_t> effective_quota_bytes_{0};
std::atomic<uint64_t> policy_sequence_{0};
uint64_t requested_quota_bytes_{0};  // 仅在控制面同步下访问
bool has_explicit_policy_{false};
```

设计要点：

- **bit 63 = 准入开关，bit 0..62 = 已计费字节**。由此推出硬上限 `kMaxChargedBytes = 2^63-1`，所有入口（`UpsertTenantPolicy`、`AdoptPendingCharge`、`RebuildUsage`、`SettleAdditional`、`ReleaseCommitted`）都先做范围检查（`mooncake-store/src/tenant/quota.cpp:241-243`、`367-371`，`tenant_quota_ledger.cpp:25-27`、`65-68`、`101-103`）。
- 计费状态与有效配额各占一条 cache line（`alignas(64)`），避免控制面写 `effective_quota_bytes_` 时把写热点的计费行打 invalid。
- `policy_sequence_` 是**顺序锁（seqlock）**：奇数表示"控制面正在改策略"。数据面 `TryCharge` 在读到奇数时自旋，读完后再次比对 sequence，保证"读到的一致快照"（`quota.cpp:94-156`）。**CAS 成功并不等于扣费成功**：如果 CAS 之后发现 sequence 变了（控制面中间插入），`TryCharge` 会把刚扣的字节 `Release` 回去再重试（`quota.cpp:141-155`），因此不会出现"按旧配额扣了费"的窗口。
- `Release` 是纯 CAS 循环，`bytes > charged_bytes` 直接返回 `kAccountingMismatch`；它不读 `effective_quota_bytes_`，所以配额缩小时已计费的字节不会被动改写（`quota.cpp:159-179`）。
- 账户指针 `TenantQuotaHandle = TenantQuotaAccount*`（`quota.h:84`）是**长期稳定**的：账户由 `std::map<TenantId, std::unique_ptr<TenantQuotaAccount>>` 持有（`quota.h:117`），只增不删，所以对象里缓存的 handle 不会悬空。

#### `TenantQuotaTable` / `ShardedTenantQuotaTable<1024>`

`TenantQuotaTable` 是单线程语义的注册表（调用者负责同步，`quota.h:89-91`）；生产用 `ShardedTenantQuotaTable<1024>` 包一层：

```cpp
// mooncake-store/include/tenant/quota_sharded.h:43-55
struct Shard { mutable std::mutex mutex; TenantQuotaTable table; };
std::array<Shard, kNumShards> shards_;
mutable std::mutex recompute_mutex_;
```

分片索引就是 `TenantIdHash{}(tenant_id) % 1024`（`quota_sharded_impl.h:155-159`）。**单租户操作只锁一个分片**（`UpsertTenantPolicy`/`DisableTenantPolicyIfEmpty` 仍会先拿 `recompute_mutex_`，因为随后要用新的策略重算全局有效配额）；`ApplyTenantPolicies`、`RebuildUsage`、`RecomputeEffectiveQuotas` 这类跨分片操作先拿 `recompute_mutex_`，再逐个分片加锁（`quota_sharded_impl.h:13`、`31`、`47-64`、`127-153`、`173-197`）。

#### `TenantQuotaManager`：控制面唯一入口

```cpp
// mooncake-store/include/tenant/quota_manager.h:94-99
const CapacityFn allocatable_capacity_;   // 回调：当前可分配总容量
std::unique_ptr<TenantQuotaPolicyStore> policy_store_;
mutable std::mutex policy_mutex_;
mutable std::mutex recompute_mutex_;
ShardedTenantQuotaTable<1024> table_;
```

`LockPolicy()` 返回 `unique_lock`（`quota_manager.h:64-66`），用于"准入检查 + 紧接着的写"之间不被并发删除策略插入——`MasterService::ResolveTenantIdForWrite` 是典型用法（`mooncake-store/src/master_service.cpp:1206-1226`）。

容量回调不是常量：`MasterService::GetTenantQuotaAllocatableCapacityBytes` 把所有已挂载 segment 的 `size` 相加（`master_service.cpp:1264-1278`）。因此 segment mount/remount 后必须重算（`master_service.cpp:737-740`、`1116`、`2792`、`2896`）。

#### `TenantQuotaLedger`：每对象的三个桶

```cpp
// mooncake-store/include/tenant_quota_ledger.h:59-62
uint64_t pending_bytes_{0};    // PutStart 已向账户预扣，但写还没结束
uint64_t committed_bytes_{0};  // 已完成的 MEMORY 副本实际占用
uint64_t replaced_bytes_{0};   // 覆盖写时从旧对象转过来的"待释放"额度
```

三条铁律（都写在头文件注释里，`tenant_quota_ledger.h:9-12`）：

1. **ledger 从不拥有账户**，每次变更都要显式传 `TenantQuotaHandle`；
2. **从不隐式释放**，对象析构不会退配额（必须走 `EraseMetadata` 显式 `ReleaseAll` 或分步结算）；
3. `Rebuild` 只改本地字段，账户由调用方另行重建（`tenant_quota_ledger.cpp:173-183`）。

### 2. KV 事件

```cpp
// mooncake-store/include/kv_event/kv_event_publisher.h:95-127
enum class EventKind { kStored, kRemoved, kCleared };
struct PendingEvent {
    EventKind kind;
    std::string object_key, medium, tenant_id, group_id, session_id;
};
std::mutex queue_mutex_;
std::deque<PendingEvent> queue_;
std::condition_variable queue_cv_;
std::thread worker_;
std::atomic<bool> stop_{false};
std::atomic<uint64_t> next_event_id_{1};
std::atomic<uint64_t> next_zmq_sequence_{1};
```

`KvEventConfig`（`kv_event_config.h:12-37`）里有几个容易忽略的字段：`queue_capacity=65536`（有界队列，满则丢最旧）、`emit_legacy_compat_fields=true`（同时吐 vLLM/SGLang 的 `type`/`block_hashes` 字段）、`emit_object_key=true`、`model_name/block_size/lora_name/additional_salt/dp_rank`（一个 publisher 服务一个固定模型上下文，租户按事件走）。

**同一头文件里有两个实现**：CMake 选项 `ENABLE_KV_EVENTS` 打开（即为 store 目标定义 `MOONCAKE_ENABLE_KV_EVENTS=1`）时是上面的真实现；关闭时是头文件内联 no-op stub（`kv_event_publisher.h:130-173`），`enabled()` 恒 false。所以调用方**不需要 `#ifdef`**，这也解释了为什么 `master_service.cpp` 里可以直接 `kv_event_publisher_->PublishCommitted(...)`。

### 3. 权重专用存储

领域类型全在 `mooncake-store/include/weight_management.h`：

- `WeightRevisionIdentity{tenant_id, name_space, resource_id, revision, weight_generation}`（:76-94）——五元组唯一确定一个不可变 revision。
- `WeightManifestReference{manifest_key, manifest_sha256, payload_group_id, payload_keys_sha256, payload_count, logical_bytes}`（:96-108）。
- `WeightRevisionMetadata`（:110-126）：`availability`（IMPORTING/READY/DEGRADED/DELETING/DELETED）+ `residency`（UNKNOWN/HOT/COLD/MIXED/ABSENT）+ `operation`（NONE/EVICTING/REHYDRATING/REPAIRING）+ `metadata_generation` 与时间戳。
- `WeightRevisionLease`（:128-139）：`fenced_metadata_generation` 是**栅栏**——租约只对"不旧于该代"的 revision 有效。
- `WeightResidencyOperation`（:141-159）：一次驻留迁移的进度（`processed_members/total_members/cursor`）。

canonical key 由身份哈希推导，而不是调用方随意给：

```cpp
// mooncake-store/src/weight_metadata_store.cpp:111-125
MakeWeightPayloadGroupId(identity) => "weight:" + SHA256(identity)
MakeWeightRevisionMetadataKey(identity) => "weight-revision:" + SHA256(identity)
MakeWeightLeaseMetadataKey(lease_id) => "weight-lease:" + lease_id
```

`MakeWeightManifestKey` 是另一套可读路径 `weights/<ns>/<resource>/<revision>/<gen>/manifest`，其中 `name_space`/`resource_id`/`revision` 三段都做 URL 编码（`weight_management.h:298-323`）。

`WeightMetadataStore` 的成员（`weight_metadata_store.h:132-138`）说明它就是个"加锁的纯内存库"：

```cpp
mutable std::mutex mutex_;
std::map<WeightRevisionIdentity, WeightRevisionMetadata> revisions_;
std::map<std::string, WeightRevisionIdentity> group_index_;   // payload group -> identity
std::unordered_map<uint64_t, WeightRevisionLease> leases_;
std::unordered_map<uint64_t, WeightResidencyOperation> operations_;
uint64_t next_lease_id_{1}, next_operation_id_{1};
```

`WeightStoreManager` 在它之上加了两层（`weight_store_manager.h:65-68`）：`backend_`（持久化）与 `std::array<std::mutex, 4096> group_locks_`——**按 payload group 分段的操作锁**，让不同权重组可以并行导入。

### 4. 客户端热缓存

```cpp
// mooncake-store/include/local_hot_cache.h:36-51
struct HotCachePutToken { uint64_t cache_epoch = 0; uint64_t key_generation = 0; };
struct HotMemBlock {
    void* addr; size_t size;
    std::atomic<int> ref_count;
    std::string key_;
    std::atomic<bool> accessed{false};
};
```

关键区分：

- `addr/size` 里 `size` 是**上一次写入对象的逻辑长度**，不是块容量。块容量是 `LocalHotCache::block_size_`（:242）。
- `key_` 对"已发布块"由缓存拥有；对"detached fill 块"由调用方（`SubmitPutTask`）在无锁状态下写入，这正是 #4334 的根因（见"易踩坑点"）。
- `accessed` 是延迟 LRU touch 标志：读路径只 `store(true)`，真正 `splice` 到队首发生在下一次持写锁的 `drainDeferredTouches()`（`local_hot_cache.cpp:420-443`）。

守护结构（`local_hot_cache.h:255-262`）：

```cpp
mutable std::shared_mutex lru_mutex_;
std::list<HotMemBlock*> lru_queue_;   // 队首最近使用，队尾最冷
std::unordered_map<std::string, std::list<HotMemBlock*>::iterator> key_to_lru_it_;
std::unordered_map<std::string, uint64_t> key_generation_;
std::atomic<uint64_t> cache_epoch_{0};
```

**`GetFreeBlock` 会把块从 `lru_queue_` 和 `key_to_lru_it_` 中彻底摘掉**（`local_hot_cache.cpp:374-418`），此块既不在 LRU 也不在索引里，处于"detached fill"状态；只有 `PutHotKey` 成功才重新入索引。

### 5. 晋升候选

```cpp
// mooncake-store/include/promotion_candidate_tracker.h:110-139
static constexpr size_t kLimit = 50000;
static constexpr uint32_t kMaxRetries = 64;
static constexpr std::chrono::milliseconds kTtl{300000};
static constexpr std::chrono::milliseconds kInitialBackoff{10};
static constexpr std::chrono::milliseconds kMaxBackoff{5000};
std::atomic<uint64_t> count_{0};
mutable std::mutex mutex_;
std::unordered_map<Key, TimePoint, KeyHash> due_;
std::set<std::pair<TimePoint, const Key*>> order_;
```

`order_` 里存的是**指向 `due_` 节点内 key 的裸指针**。这能成立的前提是 `std::unordered_map` 的节点地址在 rehash 时稳定（节点式容器），所以 `File/Unfile` 只需 `order_.erase({it->second, &it->first})`（`promotion_candidate_tracker.cpp:210-232`）。`count_` 是"建议性"计数（`std::memory_order_relaxed`，注释明说是 advisory，不是屏障）。

候选本体不在这里，而在 `route::ObjectState::promotion_candidate`（`mooncake-store/include/metadata/object_state.h:20-31`）；tracker 只持索引与规则。

### 6. 动态副本

```cpp
// mooncake-store/include/dynamic_replication_controller.h:236-282
kActionCooldown{30000}; kLeaseTtl{30000}; kRecreateCooldown{60000};
kWindowCleanupInterval{1000}; kAdmissionThreadSleepMs = 100;
kWindowEntryLimit = 50000; kWindowCleanupBudget = 256;
kAdmissionQueueLimit = 50000; kAdmissionBatchSize = 64;
kTargetHighWatermark = 0.85;

std::mutex heat_mutex_;                                  // key -> {window_start,hits}
std::mutex admission_mutex_; admission_queue_; admission_queued_;
std::thread admission_thread_;
std::mutex lease_mutex_;  unordered_map<TenantId, DynamicReplicationLeaseTable>;
std::mutex pending_mutex_; unordered_map<TenantId, unordered_set<string>> pending_keys_;
```

`DynamicReplicaPending`（定义在 `mooncake-store/include/object_runtime_state.h:17-26`，由 `metadata/object_state.h` 的 `route::ObjectState` 以 `HeapOptional` 持有）记录 `proposal_id/lease_id/source_segment/target_segment/target_domain/version_epoch/expire_at_ms_epoch/task_id`。`version_epoch` 由对象 `put_start_time` 的毫秒数推导（`dynamic_replication_controller.cpp:675-681`），作用是让"针对旧版本对象的租约"在对象被覆盖写后自动失效。

### 7. 任务邮箱

```cpp
// mooncake-store/include/task_manager.h:228-260
mutable SharedMutex mutex_;
unordered_map<UUID, Task, boost::hash<UUID>> all_tasks_;
unordered_map<UUID, queue<UUID>, boost::hash<UUID>> pending_tasks_;         // client -> FIFO
unordered_map<UUID, unordered_set<UUID, boost::hash<UUID>>, boost::hash<UUID>> processing_tasks_;
deque<UUID> finished_task_history_;                                       // LRU 淘汰顺序
size_t total_pending_tasks_, total_processing_tasks_;
```

`Task` 有 `PENDING/PROCESSING/FAILED/SUCCESS` 四态（:36-41），payload 是 JSON（`struct_json` 序列化，:130-135）。`ScopedTaskReadAccess/WriteAccess` 是 RAII 持 `SharedMutex` 的访问器（:140-202），**所有读写都通过它们**，避免调用方忘记加锁。

---

## 关键流程剖析

### 流程 A：一次多租户 Put 的配额准入、计费与结算

这是本层最需要"从头到尾"讲清楚的一条链，因为它横跨 `PutStart`、`PutEnd` 和 `EraseMetadata` 三个入口。

#### A.1 PutStart：准入探针 → 预扣 → 记账

```
Client::Put
  → MasterClient::PutStart
    → MasterService::PutStart                        (master_service.cpp:5589)
      → PrepareWriteStart                            (:5442，参数校验 + soft-pin/group/host_id 解析，不碰对象元数据)
      → AdmitWithDfsRecovery(config, admit)          (:5655)
        admit lambda:
          1) client_mutex_ shared + master_state_mutex_ shared   (:5604-5605)
          2) MetadataAccessorRW accessor(CreateKey{})            (:5610-5611)
          3) ChargeTenantQuota(quota_account, 0)   ← 准入探针    (:5612)
          4) CleanupStaleBeforeWrite                             (:5616)
          5) AllocateAndInsertMetadata                           (:5647)
```

第 3 步的 `bytes=0` 不是笔误，而是**只查准入、不开票**：`TryCharge(0)` 会检查 `admission_closed` 位与 `policy_sequence_` 的稳定性，但 `effective_quota_bytes_`（虽然被读进局部变量）在 `bytes == 0` 时根本不参与比较，也不会走 CAS 扣减（`mooncake-store/src/tenant/quota.cpp:111-139`）。它只保证"此刻该租户的准入是打开的"。真正的扣费在第 5 步：

```cpp
// master_service.cpp:1240-1262 两个计费公式
CompletedMemoryQuotaCharge(metadata)
    = metadata.TotalLength() * count(completed && is_memory_replica)
RequestedMemoryQuotaCharge(value_length, config)
    = value_length * config.replica_num        // 用 __int128 防溢出
```

```cpp
// master_service.cpp:5417-5439  AllocateAndInsertMetadata
pending_quota_charge = RequestedMemoryQuotaCharge(value_length, config);
ChargeTenantQuota(accessor.QuotaAccount(), pending_quota_charge);  // 真正的 TryCharge
AllocateReplicas(...);          // 失败 → ReleaseTenantQuota 回滚
InsertMetadata(..., pending_quota_charge, ...);  // 失败 → ReleaseTenantQuota 回滚
```

`InsertMetadata` 在 `accessor.Publish(...)` 之后、仍持有分片写锁时调用账本：

```cpp
// master_service.cpp:5368-5385
accessor.Publish(...);
auto& metadata = accessor.Get();
if (enable_multi_tenants_) {
    auto adopt_result = metadata.quota_ledger.AdoptPendingCharge(
        accessor.QuotaAccount(), pending_quota_charge);
    if (!adopt_result) {
        // 回滚：弹出全部 replica、释放 DFS、从 tenant map 中抹掉这条 metadata
        auto failed_replicas = metadata.PopReplicas();
        FreeDfsReplicas(key, failed_replicas);
        accessor.tenant_state_->metadata.erase(accessor.it_);
        return tl::make_unexpected(ErrorCode::INTERNAL_ERROR);
    }
}
```

`AdoptPendingCharge` 要求账本此前 `pending_bytes_ == 0`，且 `pending + committed + replaced + bytes` 不溢出（`tenant_quota_ledger.cpp:23-33`）。

**注意一个语义差**：预扣按 `config.replica_num` 计算，但结算按"实际完成的 MEMORY 副本数"计算。如果配置里混了 NOF/DFS 副本，或部分副本最终没完成，`PutEnd` 会把差额退回去。

#### A.2 PutEnd：结算

```
MasterService::PutEnd                          (master_service.cpp:5663)
  → 校验 client_id / processing 状态
  → VisitReplicas(processing && is_target) 逐个 mark_complete()  (:5748-5763)
  → SettlePrimaryWriteQuotaIfReady(account, metadata)            (:5779)
  → PublishKvStored(...)                                         (:5822)
  → （可选）写 OpLog PUT_END                                     (:5824-5832)
```

`SettlePrimaryWriteQuotaIfReady` 有两个前置条件（`master_service.cpp:1632-1658`）：

1. `metadata.IsValid()`；
2. **没有任何 `is_memory_replica() && is_processing()` 的副本**——否则说明还有 MEMORY 副本在传，不能结算。

然后调用：

```cpp
// tenant_quota_ledger.cpp:35-60
primary_bytes = pending_bytes_ + committed_bytes_;
total_bytes   = primary_bytes + replaced_bytes_;
account->Release(total_bytes - actual_committed_bytes);   // 先动账户
pending_bytes_ = 0;
committed_bytes_ = actual_committed_bytes;
replaced_bytes_ = 0;
```

注释（`tenant_quota_ledger.h:27-30`）明确写了顺序契约：**先校验、先释放账户，再改本地字段**。这样 `Release` 失败（`kAccountingMismatch`）时本地账本保持原样，调用方可以重试或走错误路径，而不会出现"本地认为已释放、账户里还挂着"的分裂。

`replaced_bytes_` 在这个函数里**无条件清零并释放**：覆盖写场景下旧对象的额度已经通过 `TransferReplacementCharge` 转到新账本的 replaced 桶，结算时一并退还。

#### A.3 UpsertStart 的覆盖写：额度转移而不是重复计费

`UpsertStart` 分三种情况：Case A（对象不存在，走普通创建）、Case B（同尺寸原地覆盖）、Case C（尺寸变化或还有读者，需要重新分配）。只有 Case C 涉及配额转移（`master_service.cpp:6414-6575`）：

```
1) 若存在 read lease（旧 descriptor 仍可能被读）：
     replacement_pending = RequestedMemoryQuotaCharge(new_size, cfg)
     ChargeTenantQuota(account, replacement_pending)      // 先给新副本预扣
     AllocateReplicas(...)                                // 失败则 Release
2) 若旧账本 TotalChargedBytes() != 0：
     metadata.quota_ledger.TransferReplacementCharge(account, replacement_charge)
        —— 旧账本的 committed+replaced 搬进栈上临时账本 replacement_charge
        —— 旧账本清零，**账户余额不变**
3) PopReplicasWithCacheTotalAccounting(metadata) 把旧副本放进 discarded_replicas_，
   延迟到 max(kv_lease_ttl, put_start_release_timeout) 之后才真正释放
4) accessor.Erase(QuotaEraseMode::kPreserveOld, previous_kv_media)
   —— kPreserveOld 只 RefundPending；若旧账本还有 committed/replaced 会打 ERROR
5) InsertMetadata / AllocateAndInsertMetadata 发布新对象
6) replacement_charge.TransferReplacementCharge(account, new_metadata.quota_ledger)
   —— 把额度落到新账本的 replaced 桶，等 UpsertEnd/CopyEnd 结算时释放
   任一步失败 → ReleaseReplacement 回滚
```

`TransferReplacementCharge` 的约束（`tenant_quota_ledger.cpp:118-137`）：

```cpp
if (pending_bytes_ != 0 || destination.replaced_bytes_ != 0 ||
    AddOverflows(committed_bytes_, replaced_bytes_)) return kAccountingMismatch;
destination.replaced_bytes_ = committed_bytes_ + replaced_bytes_;
committed_bytes_ = 0; replaced_bytes_ = 0;
```

即：源账本必须没有未结算的 pending（否则说不清转多少）；目标账本的 replaced 桶必须为空（避免叠加后无法区分批次）。这也解释了为什么 Case C 里"有 read lease"的路径要先把新副本的 pending 记在 `InsertMetadata` 的账本上，而不是直接塞进 replacement 账本。

#### A.4 回收路径：谁在什么时候退配额

| 场景 | 代码位置 | 行为 |
|---|---|---|
| 对象正常删除/驱逐 | `master_service.cpp:2085-2094` | `kFull` → `quota_ledger.ReleaseAll(account)`（一次性退 pending+committed+replaced） |
| 覆盖写临时摘除旧对象 | `master_service.cpp:2095-2111` | `kPreserveOld`/`kAbortOnly` → 只 `RefundPending`；若仍有 committed/replaced 打 ERROR（设计上这些额度已转移） |
| 内存副本句柄失效被清理 | `master_service.cpp:1976-1994` | 先算 `before/after CompletedMemoryQuotaCharge`，差额 `ReleaseCommitted` |
| 复制/迁移任务失败 | `master_service.cpp:1873`、`2070-2071`、`10547`、`10602` | `ReleaseTenantQuota(task->pending_quota_charge_bytes)` |
| 复制/迁移任务成功 | `master_service.cpp:7196-7205`、`7588-7597` | `SettleAdditional(account, pending, actual)`：退差额、把实际字节并入 committed |
| 晋升成功/取消 | `master_service.cpp:10028-10044` | 成功 `SettleAdditional`，取消 `ReleaseTenantQuota` |
| HA 恢复/启动重建 | `master_service.cpp:1343-1387`、`4231-4233` | 扫全部分片按 `CompletedMemoryQuotaCharge` 汇总 → `RebuildUsageOrThrow` |

`SettleAdditional` 的语义（`tenant_quota_ledger.cpp:62-82`）：`actual_bytes` 只能 ≤ `pending_bytes`，释放差额，把 `actual` 并入 committed。它**不碰 replaced 桶**，因为复制/晋升不是覆盖写。

#### A.5 有效配额如何算出来

`Recompute` 触发点是 segment mount/remount（`master_service.cpp:739`、`1116`）、segment 卸载（`2896`）以及 offboarding job 在后台摘除离线 segment 的路径（`2792`）。执行链：

```
TenantQuotaManager::Recompute                          (quota_manager.cpp:129-132)
  → lock recompute_mutex_
  → ShardedTenantQuotaTable::RecomputeEffectiveQuotas  (quota_sharded_impl.h:78-83)
      → 逐分片 ListTenantSnapshots() 汇总
      → TenantQuotaTable::BuildEffectiveQuotaAssignments (quota.cpp:28-85)
      → 逐分片 ApplyEffectiveQuotas
```

`BuildEffectiveQuotaAssignmentsImpl` 的算法（`quota.cpp:28-85`）：

1. 只统计 `has_explicit_policy` 的租户，用 `unsigned __int128` 求 `requested` 总和；
2. 若总和 ≤ 可分配容量：每人拿到自己请求的值，未显式配置的租户拿 0；
3. 否则按比例缩减：`base_i = capacity * requested_i / sum`，余数用 `__int128` 保存；
4. **最大余额法**：按 `remainder` 降序、`tenant_id` 升序排序，把剩余字节逐个 +1；
5. 结果写回每个账户的 `effective_quota_bytes_`，并 `SetAdmissionClosed(!has_explicit_policy_)`（`quota.cpp:220-224`）。

所以 `requested_quota_bytes` 与 `effective_quota_bytes` 是两回事：前者是"承诺",后者是"当前容量下的实际可用"。`TenantQuotaSnapshot::over_quota` 用后者判定（`quota.cpp:401-414`）。

配额耗尽的处理策略也值得注意：写路径**不做内联驱逐**，直接返回 `TENANT_QUOTA_EXCEEDED`，把"腾地方"交给后台 `EvictTenantsOverWatermark`（`master_service.cpp:6579-6591` 的注释，实现在 `10171-10210`）。后台按 `effective_quota_bytes` 的水位算出目标字节数再驱逐，避免每个处于上限的并发写都去扫 1024 个分片。

### 流程 B：客户端热缓存的"读命中 → 写回 → 失效"

#### B.1 初始化

```
Client::InitLocalHotCache                          (client_service.cpp:5518-5590)
  → LocalHotCacheConfig::FromEnvironment()         (config/local_hot_cache_config.cpp:45-89)
       MC_STORE_LOCAL_HOT_CACHE_SIZE      总大小，0=关闭
       MC_STORE_LOCAL_HOT_BLOCK_SIZE      默认 16MB
       MC_STORE_LOCAL_HOT_CACHE_USE_SHM   "1" 才开
       MC_STORE_LOCAL_HOT_ADMISSION_THRESHOLD  [1,255]
  → hot_cache_ = make_shared<LocalHotCache>(total, block, use_shm)
  → transfer_engine_->registerLocalMemory(base, size, ...)   // 必须整段注册
  → hot_cache_handler_ = make_unique<LocalHotCacheHandler>(hot_cache_, 2)
  → admission_sketch_ = make_unique<CountMinSketch>()
```

两个细节：

- 如果开了 `object_checksum_enabled_`，热缓存直接关闭（`client_service.cpp:5524-5526`，代码没有写注释说明原因；从两条路径的语义看，应与"读校验和依赖真实副本地址/长度"这一约束有关）。
- shm 模式注册的是 **segment 的真实大小**而不是逻辑大小（:5550-5559）。原因是 `ShmHelper::allocate` 会按 hugepage 向上取整，按逻辑大小注册可能让 MR 结束在最后一个 huge page 中间，后续 `madvise(MADV_DONTFORK)` 会 EINVAL。

#### B.2 读命中：改写 descriptor，而不是搬数据

```
Client::Get / BatchGet
  → RedirectToHotCache(key, replica)              (client_service.cpp:1934-1957)
      if (!replica.is_memory_replica() || !hot_cache_) return false;
      blk = hot_cache_->GetHotKey(key);           // 命中则 ref_count++
      if (blk == nullptr) return false;
      if (mem_desc.buffer_descriptor.size_ != blk->size) return false;  // 尺寸必须一致
      mem_desc.buffer_descriptor.transport_endpoint_ =
          (metadata_connstring_ == P2PHANDSHAKE) ? GetTransportEndpoint() : local_hostname_;
      mem_desc.buffer_descriptor.buffer_address_ = (uintptr_t)blk->addr;
      return true;
  → TransferRead(replica, slices)                 // 走本地 memcpy 分支
  → hot_cache_->ReleaseHotKey(key)                // 传输完成后释放
```

这里的关键是 **`transport_endpoint_` 被改成自己的端点、地址改成热缓存块地址**，于是 Transfer Engine 认为"目标就在本机"，走本地 memcpy 路径。`GetHotKey` 在 `shared_lock` 下把 `ref_count++` 并置 `accessed`（`local_hot_cache.cpp:133-152`），保证在 `ReleaseHotKey` 之前该块不会被 `GetFreeBlock` 摘走。

#### B.3 写回：两级准入 + 异步发布

```
ShouldAdmitToHotCache(key, cache_used)            (client_service.h:798-806)
  cache_used == true       → false（命中过就不再计数/回填）
  admission_sketch_ == null→ true
  否则 increment(key) >= admission_threshold_
ProcessSlicesAsync(key, slices, replica)          (client_service.cpp:5613-5650)
  if (!(hot_cache_ && replica.is_memory_replica())) return;
  if (!hot_cache_->IsShm() && IsReplicaOnLocalMemory(replica)) return;  // 本地内存无需缓存
  for each slice:
      if (device::GetAcceleratorRegistry().RuntimeAccelerators()
              .FindDeviceForPointer(slice.ptr) != nullptr) { ++device_dest_slices; continue; }
      hot_cache_handler_->SubmitPutTask(key, slice);
```

`ProcessSlicesAsync` 对每个 slice 单独提交，意味着一次读如果被拆成多段目标缓冲（`vector<Slice>`，例如 batch_get_into 的非连续 buffer），只有 host 段会进缓存，device 段被跳过。`#4764` 的注释把原因写得很直白：`SubmitPutTask` 用 **host `std::memcpy`** 从读目标拷进缓存块，对 CUDA 指针会直接段错误（`client_service.cpp:5626-5638`）。读路径不受影响：缓存命中时 TE 的本地 memcpy worker pool 本身是 device-aware 的（commit message 所述）。

`SubmitPutTask` 的执行顺序（`local_hot_cache.cpp:492-562`）值得逐步看：

```
1) TouchHotKey(key) 命中 → 直接返回 true（只 touch，不重复拷贝）
2) 队列容量检查（默认 1024），满了丢任务并 LOG_EVERY_N
3) token = AcquirePutToken(key)      // 读 cache_epoch + key_generation
4) block = GetFreeBlock()            // 从队尾找一个 ref_count==0 的块，detach
5) slice.size > hot_cache_->GetBlockSize() → 归还 block 并失败
6) std::memcpy(block->addr, slice.ptr, slice.size)   // 不持锁
7) block->size = slice.size; block->key_ = key
8) 加 queue_mutex_ 检查 shutdown_，push 任务，notify_one
```

第 5 步的注释（:527-533）是个真实的坑：**不能拿 `block->size` 当容量**，因为它是上一个对象的逻辑长度，`GetFreeBlock` 复用时不会重置。如果按它比较，块的可容纳尺寸会随着每次复用单调收缩。

worker 侧（:564-609）只做一件事：`PutHotKey(block, token)`。

```cpp
// local_hot_cache.cpp:82-93
std::unique_lock lk(lru_mutex_);
if (block && !block->key_.empty() && !isPutTokenValidLocked(block->key_, token)) {
    block->key_.clear();       // 令牌过期：丢弃数据，块回到空闲池
}
return putHotKeyLocked(block);
```

**令牌校验与发布在同一把写锁内完成**，这是"取消语义"的关键：如果 `RemoveHotKey` 在拷贝期间发生，它会 `key_generation_[key]++`（:211-222），于是 worker 拿着旧 generation 一定校验失败，不会把过期数据复活。

`putHotKeyLocked`（:95-126）再做一次竞争检查：如果 key 已经在 `key_to_lru_it_` 里，或存在 `ref_count>0` 的同名活跃块（`hasActiveBlockForKeyLocked`），就把块当空闲块塞回队尾。

#### B.4 失效与释放

- `RemoveHotKey` / `RemoveHotKeys` / `RemoveHotKeysByRegex`：逐个 `key_generation_[key]++`，再摘索引、把块塞回队尾（:192-278）。
- `RemoveAllHotKeys` / `Clear`：`cache_epoch_++` 后清空全部，`key_generation_.clear()`（:280-334）。
- `BumpKeyGeneration` / `BumpCacheEpoch`：只失效 in-flight fill，不驱逐条目（:300-318）。
- `ReleaseHotKey`（:154-180）：先查 `key_to_lru_it_` 快路径；找不到则扫 `blocks_`，找 `ref_count > 0 && key_ == key` 的块递减。这条慢路径是为"条目已被删、但读者还握着块"准备的。

### 流程 C：LOCAL_DISK → MEMORY 晋升

#### C.1 触发与准入

```
MasterService::GetReplicaList                     (master_service.cpp:4777-4875，其中判定在 4835-4874)
  在 RO accessor 释放后：
  if (promotion_eligible)  TryPushPromotionQueue(object_id)
      promotion_eligible = !any_memory && any_local_disk    (:4850)
```

`TryPushPromotionQueue`（:9633-9785）的闸门顺序是：

```
1. promotion_on_hit_ && promotion_sketch_ 否则 kDisabled
2. CMS increment(tenant-scoped key) < threshold → kFrequencyRejected
3. 内存水位 >= eviction_high_watermark_ratio_ → kWatermarkRejected
      （record_candidate=true 时 RecordLocked(..., kWatermark)）
4. 取 RW accessor；对象不存在 → kNotFound
5. accessor.InProcessing() → EraseCandidate + kAlreadyInFlight
6. state.promotion_task 已存在 → TouchPromotion（刷新邮箱顺序）+ EraseCandidate
7. 已有 MEMORY 副本 → EraseCandidate + kMemoryReplicaPresent
8. 找不到 LOCAL_DISK 源 → EraseCandidate + kNoLocalDiskSource
9. promotion_in_flight_ >= promotion_queue_limit_ → kQueueCapRejected（记候选）
10. source->inc_refcnt()；PushPromotionQueue → local_ssd_manager_.EnqueuePromotion
      失败：dec_refcnt；
            OBJECT_ALREADY_EXISTS → kAlreadyInFlight（并清候选）
            SEGMENT_NOT_FOUND/INVALID_PARAMS → kNoLocalDiskSource（并清候选）
            其他（含客户端已不 serving）→ 记 kPushFailed 候选
11. ConsumeLocked() 取出候选链上的 execution_failures（无候选=0）
12. state.promotion_task = PromotionTask{source_id, object_size, holder_id, execution_failures,...}
13. promotion_in_flight_++；返回 kQueued
```

注意第 10 步是**把任务投进"持有该 LOCAL_DISK 副本的客户端"的专用邮箱**，走的是 `local_ssd_manager_.EnqueuePromotion`，不是 `ClientTaskManager`。客户端通过 `PromotionObjectHeartbeat` 拉取（:9787 起），服务端限制"一次心跳最多 `promotion_max_per_heartbeat_` 个"（默认 1），原因是客户端每个任务要做"SSD 同步读 + RDMA 写"，给多了会阻塞超过 liveness 窗口（:9796-9800 的注释）。

#### C.2 失败与重试预算

`NotifyPromotionFailure`（:10069 起）在清掉任务后重新记候选（:10143-10160）：

```cpp
if (!metadata.HasReplica(fn_is_memory_replica) &&
     metadata.HasReplica(fn_is_local_disk_replica)) {
    if (prior_failures >= kMaxPromotionExecutionFailures) {
        // 放弃：毒 key 不再自我重记
    } else {
        freq = promotion_sketch_->count(key);        // 只读，不再 increment
        promotion_candidates_.RecordLocked(..., kExecutionFailed, OK,
                                           prior_failures + 1);
    }
}
```

`kMaxPromotionExecutionFailures = 3`（`master_service.h:2079`）是**防自激**的：没有它，一个 SSD 文件坏掉但 LOCAL_DISK 副本还在的 key 会"失败→重记→再准入→再失败"无限循环，且没有真实读需求，白白占投递槽。注释还强调这里用 `count()` 而不是 `increment()`，因为执行失败不是需求信号，不能污染频率统计。

重试候选的规则（`promotion_candidate_tracker.h:110-121` 的注释是最权威的说明）：

- 预算 `kMaxRetries=64`、TTL `kTtl=300s`、退避 `10ms` 起、上限 `5s`；
- 这些数值是按"客户端 offload 心跳 10s 级"推导的：水位/队列上限/推送失败这些闸门要等心跳才会放开。旧预算（8 次 ≈ 2.3s）会在条件放开前就过期；
- 64 次 × 5s 上限 ≈ 5 分钟 ≈ 30 个心跳周期；TTL 限制"一个没人读的 key 能占候选槽多久"。

`DueLocked` 在过期时会区分两种指标（`promotion_candidate_tracker.cpp:119-144`）：`retry_count == 0` 说明"一轮都没轮到就 TTL 了"（批太小），记 `expired_unevaluated`；否则记 `expired_evaluated`。

#### C.3 重试轮

`RunPromotionCandidateRetry`（:9361-9434）由后台线程驱动（每次最多 `kPromotionRetryBatchSize=128`）：

```
1. 持 master_state_mutex_ shared，promotion_candidates_.Due(now, 128)
2. 对每个 (tenant,key) 取 RW accessor：
     - 对象不存在 → ForgetLocked
     - DueLocked() 过期/未到期 → 跳过（过期会顺带 Erase）
     - 快速预过滤：IsValid / !is_processing / !promotion_task /
                   !有 MEMORY / 有 LOCAL_DISK，否则 EraseLocked
3. 释放锁后，再持 shared lock 对候选逐个 TryPushPromotionQueue(record_candidate=false)
     kQueued          → admitted
     IsTransient      → BackoffCandidate（kWatermark/kQueueCap/kPushFailed）
     其他（永久）      → EraseCandidate
```

`BackoffLocked`（`promotion_candidate_tracker.cpp:146-176`）每次 `retry_count++`，按本次闸门更新 `last_reason`（只有 watermark/queue-cap 会把 `last_error` 重置为 OK；push-failed 沿用 `RecordLocked` 当时记下的错误码），然后 `retry_after = now + Backoff(retry_count)`；`Stale` 判定是 `now - last_seen >= kTtl || retry_count >= kMaxRetries`（:204-208）。

### 流程 D：动态 MEMORY 副本

#### D.1 热度统计 → 准入队列

```
GetReplicaList 读路径
  if (dynamic_replication_.Enabled()) {
      memory_replicas = count(is_memory_replica)
      observed = WantsAnotherMemoryReplica(memory_replicas)   // 0 < mem < max
  }
  释放 RO accessor 后：
  dynamic_replication_.RecordAccess(tenant, key)              (dynamic_replication_controller.cpp:154-167)
      ObserveAccess() 返回 true（本窗口恰好达到阈值）→ Enforce 模式 Enqueue
```

`ObserveAccess`（:98-130）：

- key 用 `tenant_id.MakeScopedKey(key)`（NUL 分隔）；
- 窗口内 `hits++`，跨窗口重置；
- 返回 `hits == AdmissionMinHits()`，`AdmissionMinHits = ceil(qps_threshold * window_seconds)`（:74-78）。用"恰好等于"而不是"大于等于"，保证**一个窗口只入队一次**；
- 表大小到 50000 时，每 1s 最多清理 256 个（`CleanupWindowsLocked`，:80-96），窗口存活超过 2×window 才删。

注意：`RecordAccess` 只在"已有 1..max-1 个 MEMORY 副本"时被调用。**0 个 MEMORY 副本（纯盘对象）不属于动态副本的职责，那是晋升**。两条特性在这里是明确分工的。

#### D.2 准入线程

```cpp
// dynamic_replication_controller.cpp:182-210
while (admission_running_) {
    wait_for(100ms, 队列非空或停止);
    while (队列非空 && batch.size() < 64) { pop; admission_queued_.erase(scoped_key); }
    for (object : batch) submit_(tenant_id, key);   // 不持锁
}
```

`Enqueue`（:169-180）用 `admission_queued_` 去重、`kAdmissionQueueLimit=50000` 限长。`submit_` 是 `MasterService` 在构造时注册的 lambda，**它自己先拿 `master_state_mutex_` shared，再调 `TrySubmitDynamicReplicaProposal`**（`master_service.cpp:456-460`），所以准入线程不会破坏锁序。

#### D.3 提案 → 选源/选目标 → 发租约

```
TrySubmitDynamicReplicaProposal                    (master_service.cpp:9436-9451)
  if (!Enforcing || !HeatAdmitted) return;
  SubmitReplicaActionProposalLocked(MakeAutoProposal(...))
    → ValidateProposal                              (dynamic_replication_controller.cpp:225-245)
        仅 Enforce；action==ADD；key 非空；proposal_id 非零；
        requester_domain/target_domain 必须为空（域感知留给下一阶段）；
        expire_at_ms_epoch 不能早于 now
    → ResolveTenantIdForWrite（拿 policy 锁）
    → MetadataAccessorRW accessor
    → FindReusableLease                             (dynamic_replication_controller.cpp:247-284)
        幂等重试：同一 proposal_id 的活租约直接返回；过期则删掉当没有；
        同一 proposal_id 但请求参数不同 → INVALID_PARAMS
    → HeatAdmitted 再查一次
    → CheckCooldownLocked                           (:492-504)
        state.dynamic_replication_cooldown 未到期 → UNAVAILABLE_IN_CURRENT_STATUS
    → InProcessing / HasReplicationTask / HasPendingLocked → OBJECT_HAS_REPLICATION_TASK
    → version_epoch = VersionEpoch(metadata)（put_start_time 毫秒）
        校验 proposal.observed_version_epoch / object_size_bytes
    → DynamicReplicationRecreateBlocked(now) → UNAVAILABLE_IN_CURRENT_STATUS
    → SelectPlan(...)                               (:286-431)
    → IssueLeaseLocked(...)                         (:506-539)
    → SubmitDynamicReplicaCopyTask(...)             (master_service.cpp:9590-9625)
        源 client 仍 serving、segment 归属未变，才 submit REPLICA_COPY 任务
    → lease.task_id = task; CommitLeaseLocked(...)  (:541-552)
```

`SelectPlan` 的选法（:286-431）：

- 收集所有 `is_readable && is_memory_replica` 的源 segment；若源为空、或 MEMORY 副本数 ≥ `max_memory_replicas_`，返回空；
- 源：在去重排序后的候选里，按 `StableScore(object_key, segment)` 取最小——稳定的 FNV-1a 哈希，保证同一对象在多次提案里倾向同一源，减少抖动（:58-72、341-345）；
- 目标：必须不在 `existing_segments`、`IsSegmentAllocatable`、剩余容量 ≥ 对象大小，且**放入后利用率 < 0.85**（:347-364）；
- 打分三元组 `(different_host, util, stable_score)`：优先"不同主机"，再优先"利用率低"，最后用稳定哈希打破平局（:366-383、397-413）；
- 只有 `preferred_target_segment` 合法时才用它，否则走打分。

`IssueLeaseLocked`（:506-539）把租约的过期时间取 `min(proposal.expire, now + 30s)`，把 pending 写进 `state.dynamic_replication_pending`，并把 key 放进 `pending_keys_` 索引（过期清扫只扫这些 key，而不是遍历所有对象）。

#### D.4 CopyStart：`PendingCopyGuard` 的失败回滚

`CopyStart` 拿到分片写锁后先验证 pending（`master_service.cpp:6889-6900` → `dynamic_replication_controller.cpp:583-629`）：

```cpp
bool dynamic_copy = (lease_id != UUID{});
if (!state.dynamic_replication_pending) {
    return dynamic_copy ? UNAVAILABLE_IN_CURRENT_STATUS : OK;
}
pending = *state.dynamic_replication_pending;    // 按值拷贝，后面会 reset
if (pending.expire_at_ms_epoch < NowMs()) {
    ClearPendingLocked(...);                     // 顺手清 cooldown + leases
    return dynamic_copy ? UNAVAILABLE_IN_CURRENT_STATUS : OK;
}
if (!dynamic_copy) return OBJECT_HAS_REPLICATION_TASK;
if (pending.version_epoch != current_version_epoch) { ClearPendingLocked; return INVALID_VERSION; }
if (pending.lease_id != lease_id || pending.version_epoch != lease_version_epoch)
    return INVALID_VERSION;
if (pending.source_segment != source_segment || targets != {pending.target_segment})
    return OBJECT_HAS_REPLICATION_TASK;
return OK;
```

验证通过后立刻构造 `PendingCopyGuard`（`master_service.cpp:6899-6900`，类定义在 `dynamic_replication_controller.h:180-207`）：

```cpp
PendingCopyGuard guard(controller, tenant, key, state, dynamic_copy);
// 之后任何失败返回都会析构 guard → ClearPendingLocked：
//   pending.reset(); cooldown 清零; EraseLeasesForObject; UnindexPendingLocked
```

成功路径在 `RegisterCopyStartLocked`（`dynamic_replication_controller.cpp:631-660`）返回后立刻 `pending_guard.Release()`（`master_service.cpp:7015-7019`）。`RegisterCopyStartLocked` 里先 `pending.reset()`，再核对 source/expiry/version，最后 `metadata.MarkDynamicReplica(replica_id, {created_at, source, target, domain, complete=false})`。注意它**不写 cooldown**——cooldown 是在 `CommitLeaseLocked`（发完任务时）就设成 `now+30s` 的（:548-549），也就是"从发出租约起算冷却"，而不是从副本建好起算。

`CopyEnd` 成功后 `SyncKvObjectState` 发布介质变化（`master_service.cpp:7212`）；失败/撤销路径会 `ClearPendingLocked` 并把 `pending_quota_charge_bytes` 退掉（:7281-7284、:7099-7102）。删除副本时 `RecordReplicaRemoval`（`dynamic_replication_controller.cpp:662-673`）会把该 replica 从 `dynamic_replicas` 里忘掉，并置 `recreate_after = now + 60s`，防止"删了立刻又被热起来重建"的抖动。

### 流程 E：KV 事件的发布与 RemoveAll 的 epoch 协议

#### E.1 事件从哪来

| 触发点 | 调用 | 语义 |
|---|---|---|
| `PutEnd` 成功 | `PublishKvStored`（`master_service.cpp:5822`） | 提交：对当前全部介质无条件发 `stored`（内容可能变了，即使介质集合没变） |
| 元数据内部变更（evict、copy、promotion、invalid handle 清理） | `SyncKvObjectState`（:13251） | **差分**：`previous_media` 有、`current` 没有 → `removed`；反之 → `stored` |
| 对象整体删除 | `PublishKvRemoved`（:13275）→ `EraseMetadata` :2045 | 对删除前可见的每种介质发 `removed` |
| 驱逐但对象存活 | `PublishKvRemovedAfterEvict`（:13289） | 把被驱逐介质并进 previous 集合，再走差分，避免重复 |
| `RemoveAll` | `PublishKvClearedIfEpochUnchanged`（:13325） | 每租户一条 `cleared` |

介质字符串只有两种：`cpu` 和 `disk`（`KvMediaForMetadata`，:13171-13198）。`cpu` 要求"completed 且 memory 且 handle 有效"；`disk` 覆盖 NOF/disk/local_disk/dfs。删除路径用 `KvMediaForRemoval`（:13200-13233），会把 PROCESSING/REMOVED 状态的副本也算进去——因为它们曾经被宣告过，删除时必须撤回。

`KvMediaSnapshot` / `KvRemovalSnapshot`（`master_service.h:2340-2347`）在 publisher 未启用时直接返回空 vector，避免热路径上白跑 `VisitReplicas`。

#### E.2 `RemoveAll` 的 epoch 抑制

跨 1024 分片扫描时，一个并发的 `PutEnd` 可能落在"已经扫过的分片"上。如果扫描结束后无条件发 `cleared`，就会把刚写进去的活对象从 indexer 里抹掉。协议是：

```
1. 扫描前：epoch_baseline = SnapshotKvTenantEpochs(track)     (master_service.cpp:7883)
   —— 整数组拷贝，而不是"第一次见到某租户时再读"：
      并发提交可能落在尚未见到的租户上，那时读会把它的提交折进自己的基线
2. 扫描中：tenants_seen / tenants_with_remaining_objects 两个集合  (:7884-7885)
   —— 只有真正遍历到对象才记 seen（空 tenant 行不算"被清空"）
3. 扫描后：对每个 seen 且没有 remaining 的租户
      PublishKvClearedIfEpochUnchanged(tenant, epoch_baseline[slot])  (:8001-8009)
4. 该函数在 epoch 锁内重读并比较，相同才发布 cleared              (:13325-13344)
```

`PublishKvStored` 的 epoch bump 在 enqueue **之前**、且在对象分片锁内完成（:13238-13242）：所以并发的 `RemoveAll` 要么读到新 epoch 从而放弃 `cleared`，要么它的 `cleared` 早已排在 `stored` 之前。`SyncKvObjectState` 只在"当前仍有介质"时才 bump（:13261-13266）：一个以"没有任何介质"结尾的同步是撤回，`cleared` 跟在它后面是安全的。

epoch 数组是 **1024 个哈希槽**而不是 per-tenant map（`master_service.h:2285-2292`）：哈希碰撞只会让两个租户共享 epoch，后果是"该发的 clear 被抑制"，永远不会发出不安全的 clear。

#### E.3 Publisher 内部

```
EnqueueBatch                                     (kv_event_publisher.cpp:296-314)
  队列满（默认 65536）→ pop_front + dropped_events++ + next_zmq_sequence_++  // 预留 seq 空洞
WorkerLoop                                       (:333-352)
  wait 直到 stop 或非空；每次最多取 64 条；PublishBatch
  stop 后 DrainRemainingQueue 把剩余排空
PublishBatch                                     (:354-501)
  1. 过滤：非 cleared 且 (emit_object_key=false 或 key 为空) → skipped_keyless++
  2. 分配 event_id（单调递增）
  3. msgpack: [timestamp_ms, [event...], dp_rank]
     每条事件是一个 map，字段数由 ComputeEventMapSize 精确计算（:37-65）
  4. ZMQ 三帧：空 topic / 8 字节大端 seq / payload
  5. 任一帧发送失败 → dropped_events += encoded.size()
```

几个工程细节：

- 发送高度水位 `ZMQ_SNDHWM=10000`、`ZMQ_LINGER=0`（:19、:134-137），进程退出不阻塞。
- 丢弃事件时**额外把 ZMQ 序列号 +1**（:307-308），消费者据此能发现"中间少了一批"，而不是静默丢数据。
- `emit_object_key=false` 在启动时打 WARNING（:108-117）：这个 flag 名字像格式开关，实际会让所有 stored/removed 事件变成"没有可操作 key"而被丢弃。
- 事件里的 `tenant_id` 统一规范化成 `"default"`（:85-89），订阅者不需要处理空值。
- **epoch 的 bump 与"有没有订阅者"无关**：`PublishKvStored` 先 `BumpKvTenantEpoch` 再看 publisher 是否 enabled（`master_service.cpp:13238-13245`）。这让 RemoveAll 的抑制协议在测试里可以脱离真实 ZMQ 验证（`kv_track_tenant_epochs_` 也由构造期独立设置）。

### 流程 F：权重导入的"先持久化后发布"

#### F.1 两段式变更模型

`WeightMetadataStore` 把每个写操作拆成 **Prepare（只读、算出 mutation）** 和 **Publish（在锁内校验并落地）**：

```cpp
// weight_metadata_store.h:21-42
struct WeightMetadataMutation {
    Kind kind;                       // UPSERT / ERASE
    WeightRevisionIdentity identity;
    std::optional<WeightRevisionMetadata> previous;  // 乐观并发基线
    std::optional<WeightRevisionMetadata> next;
    bool no_op{false};               // 幂等重试：不写 OpLog，直接返回
};
```

`Publish` 的并发控制非常直接（`weight_metadata_store.cpp:304-351`）：

```cpp
if (mutation.previous) {
    if (current == end || current->second != *mutation.previous)
        return STALE_GENERATION;
} else if (current != end) {
    return STALE_GENERATION;
}
```

也就是 **CAS：整条 metadata 必须与 Prepare 时看到的一模一样**。`metadata_generation` 每次真正变更 +1（`PrepareCommitImport` 在 :246），租约的 `fenced_metadata_generation` 与它比较。

#### F.2 一次 Commit 的完整链路

```
WeightStoreManager::CommitWeightImport                    (weight_store_manager.cpp:85-112)
  → canonical_group = MakeWeightPayloadGroupId(identity)；manifest.payload_group_id 必须等于它
  → LockGroup(identity)                                   (:54-60)
      key = TenantId(tenant).MakeScopedKey(MakeWeightPayloadGroupId(identity))
      return group_locks_[hash(key) % 4096]
  → weight_metadata_.PrepareCommitImport(request, now_ms)  (:188-253)
      校验身份/manifest/canonical key/generation≠0
      READY：manifest 相同 + 代匹配 → no_op；否则 CONFLICT/STALE
      IMPORTING：代必须匹配、payload_group_id/count/logical_bytes 必须与 Begin 时一致
      CanAdvanceWeightMetadataGeneration 防 0 与 UINT64_MAX
      next.availability=READY; next.residency=HOT; ++metadata_generation
  → if (!mutation->no_op) ValidateWeightGroupForCommit(request)  (:149-199)
  → PersistAndPublishWeightMutation(*mutation)             (:201-246)
```

`ValidateWeightGroupForCommit` 是"元数据说的"与"存储里真实存在的"之间唯一的桥：

```
expected_manifest_key = MakeWeightManifestKey(identity)      // 不匹配 → CONFLICT
members = backend_.SnapshotWeightGroup(identity, group_id)
members.size() != payload_count + 1                          → CONFLICT
逐个成员：
    !readable                                                → NOT_READY
    key == manifest_key:
        必须恰好一次、data_type == METADATA                   → 否则 CONFLICT
    其他:
        data_type 必须 == WEIGHT、size 累加不溢出              → 否则 CONFLICT
        payload_keys.push_back(key)
没有 manifest                                                → NOT_FOUND
logical_bytes 不等 / 数量不等 /
ComputeWeightPayloadKeysSha256(sorted keys) 不等              → CONFLICT
```

`SnapshotWeightGroup` 实际是去 Master 的 group domain 里取成员，再逐个对象读元数据（`master_store_backend.cpp:45-76`）。它同时校验 `metadata.group_id == payload_group_id`，防止 group 索引与对象实际归属不一致。

#### F.3 `PersistAndPublish`：为什么需要条件变量

```cpp
// weight_store_manager.cpp:13-50
auto completion = make_shared<Completion>();      // mutex + cv + optional<Result>
auto persisted = backend.AppendOpLogWithDurableFinalize(
    type, tenant, key, payload,
    [completion, publish](const DurableResult& durable) {
        lock(completion->mutex);
        if (completion->result.has_value()) return;   // 只允许一个终态
        completion->result = durable ? publish(*durable)
                                     : unexpected(DURABILITY_FAILED);
        completion->cv.notify_all();
    });
if (!persisted) return unexpected(DURABILITY_FAILED);   // 入队就失败
unique_lock lock(completion->mutex);
completion->cv.wait(lock, [&]{ return completion->result.has_value(); });
```

关键在 `WeightStoreBackend::DurableFinalize` 的契约注释（`mooncake-store/include/weight_store_backend.h:21-23`）：**"stop 可能在一次成功的在途写完成之前就报失败；消费者必须仲裁终态后才能发布元数据"**。所以写路径要同时接受"持久化成功回调"和"持久化失败回调"两条路，用 `completion->result.has_value()` 保证 `publish` 只发生一次。`MasterStoreBackend::AppendOpLogWithDurableFinalize` 的具体做法是（`master_store_backend.cpp:22-43`）：

- 入队成功后，再挂 `AwaitDurable(sequence_id).thenValue(...)`，**只在 error != OK 时**调用 `finalize(unexpected)`；成功路径由 OpLog writer 的 durable 回调负责；
- 入队失败直接返回错误，调用方不会再等。

`mutation.no_op || !backend_.IsOpLogEnabled()` 时直接 `Publish`，不写 OpLog（`:208-210`）：幂等重试和"没有 HA"的部署都不产生无谓日志。

还有一把只在这两个 `PersistAndPublishWeight*Mutation` 里用到的 `std::shared_mutex mutation_mutex_`（`weight_store_manager.h:67`）：写路径持 **shared** 锁（`weight_store_manager.cpp:204`、`:326`），而快照导出走 `TryLockSnapshot()` 的 **unique** try_lock（`weight_store_manager.h:24-26`）。因此导出快照与"append 到 publish 之间"的窗口互斥，多个互不相关的权重写入却可以并发；`TryLockSnapshot` 用 try_lock 而不是阻塞，是为了让摘快照的一方自己决定退让策略。

还有一层门控（`master_service.cpp:194-198`）：

```cpp
enable_oplog_ = config.enable_ha && config.enable_oplog && ha::HaBackendSupportsOpLog(...);
weight_management_mutations_enabled_ = !enable_oplog_ || config.weight_management_oplog_capability_confirmed;
```

即：**没有 OpLog 时权重变更允许（纯内存、易失）；有 OpLog 但没确认 capability 时，所有非 no-op 变更都返回 `DURABILITY_FAILED`**。这是"宁可拒绝，也不写一半"的 fail-closed 选择。

#### F.4 租约与 HA 恢复

- `PrepareAcquireLease`（`weight_metadata_store.cpp:421-480`）：revision 必须 READY、代匹配、没有 operation；同一 holder 已有活租约且 `fenced_metadata_generation <= current` → `no_op` 返回原租约（幂等）；否则分配 `next_lease_id_`。
- `Publish(WeightLeaseMutation)`（:564-619）：再次检查 revision 代 ≥ fence、READY、无 operation；`previous` 必须完全匹配（CAS）。注意 `fenced_metadata_generation >= revision.metadata_generation` 是合法的（栅栏只要求"不旧于"）。
- `PrepareRenewLease`（:482-510）：**`expires_at_ms = max(expires_at_ms, now + ttl)`**——租约续期必须单调，否则 standby 重放会得到"时间倒流"的状态。注意 `WeightStoreManager::RenewWeightRevisionLease`（`weight_store_manager.cpp:267-294`）会**先 Prepare 一次拿到 identity，加 group 锁后再 Prepare 一次**：第一次只用于定位锁，真正的 mutation 必须在锁内重新计算 `now_ms`，否则会拿锁外的旧时间戳发布。
- `PrepareExpireLeases`（:542-562）：扫描所有过期租约，生成按 `lease_id` 排序的 ERASE 列表，保证重放确定性。
- `ExportSnapshot` / `RestoreSnapshot`（:1099、1229-1259）：恢复前跑 `ValidateWeightMetadataSnapshot`（:1135-1227），检查 schema、身份唯一、group 唯一、lease 指向存在的 revision、`fence <= generation`、`next_*_id > max`、operation 与 revision 状态互相自洽（包括 completed operation 要求 revision 已经前进一步）。`MasterService::RestoreFromStandby` 在装载任何新状态前还会先独立校验一次（`master_service.cpp:3776-3778`，失败直接返回、尚未安装任何东西）；随后 `weight_manager_.RestoreSnapshot` 内部再校验，此处失败才走 `rollback_restored_state()` 并让整个 restore 失败（`master_service.cpp:4262-4269`）。

**未完全确认**：`WeightMetadataStore::PrepareStartOperation/PrepareFinishOperation` 等驻留操作的完整状态机（`weight_metadata_store.cpp:636-908`）逻辑较密，本章只覆盖了快照校验里体现出的约束（EVICTING↔COLD、REHYDRATING↔HOT），未逐条推演其失败分支。

### 流程 G：Reshard 权重传输的有界在途（#4628）

这块是**纯 Python**，与 Master 特性层没有直接调用关系，但它是"权重传输"这条用户链路的执行端，且最近有重要改动。

`mooncake-reshard/python/mooncake/reshard/transfer_engine/executor.py`：

```python
# :355-527
_execute_batches(batches, direction, *, max_inflight_batches, submission):
    if max_inflight_batches == 1:                       # 默认路径
        return tuple(submission.execute_batch(b, direction) for b in batches)
    # >1：要求引擎有 submit_scatter_read / submit_scatter_write
    published: deque[_PublishedBatch] = deque()          # 滑动窗口
    receipts: list[TransferBatchReceipt | None]
    unresolved: list[_CompletionTicket]
    while ...:
        if len(published) >= max_inflight_batches:
            drain_published(published.popleft())         # 先回收最老
        batch = next(batch_iterator)
        prepared = self._prepare_batch(batch, direction, require_async=True)
        ticket = submit_method(*prepared.arguments)
        # 入窗前探测 ticket.drain / ticket.status 都是可用属性
        published.append(_PublishedBatch(index, prepared, ticket))
    while published: drain_published(published.popleft())
```

要点：

- 默认 `max_inflight_batches=1`（`executor.py:125`、`weight/_te/execution.py:73`），也就是改动前后行为一致；只有显式调大才走异步提交。
- 窗口满时**先 drain 最老再提交新批**，因此同时在途的物理 IO 数量有上界；任一批失败/完成未知/被中断，循环立刻停止提交，但**已提交的仍然会逐个 drain**，不会留下未回收的资源。
- `_CompletionUnknown` 的 ticket 会进入 `unresolved`；循环结束后若 `unresolved` 非空，包装成 composite ticket 交给 `_pending_manager._retain_pending_ticket(...)`（`executor.py:496-512`、`:532`、`:557`）。语义是"完成状态未知 → 注册资源暂不回收，等 pending 管理器裁决"。
- 预先探测 `drain/status` 属性、`_mark_physical_io_started()` 等细节说明这条路径对"原生提交返回了不可 drain 的 ticket"是 fail-closed 的。

**未完全确认**：`completion.py` 中 `_CompletionTicket` 的具体回收时机（何时算"safe to unregister"）本章只按调用点语义描述，未逐行核对 pending manager 的内部状态机。

---

## 与其他模块的边界

### 依赖谁

| 被依赖方 | 用途 | 契约位置 |
|---|---|---|
| 元数据分片/路由（`metadata/object_state.h`、`object_route.h`、`MetadataShardAccessorRW`） | 对象锁、`route::Generation`、`ObjectState` | `master_service.h:1098-1135`、`1741-1851` |
| segment 管理/分配器（`segment_manager_`、`ScopedSegmentAccess`） | 可分配容量、segment 利用率、源/目标选择 | `master_service.cpp:1264-1277`、`dynamic_replication_controller.cpp:286-431` |
| HA OpLog（`OpLogEntry`、`OrderedOpLogWriter::AwaitDurable`） | 权重变更持久化、restore 的 durable repair | `master_store_backend.cpp:22-43` |
| HA 快照（`serialize/`、`WeightMetadataSnapshot`） | 权重状态随快照迁移 | `master_service.cpp:4262-4269` |
| `CountMinSketch` | 热缓存准入、晋升频率 | `count_min_sketch.h:12-79`（内部 mutex，`increment/count` 可在任意线程调用） |
| Transfer Engine 客户端 | 热缓存注册/本地读、设备指针识别 | `client_service.cpp:5560-5575`、`5630-5638` |
| `ShmHelper` | 热缓存跨进程共享 | `local_hot_cache.cpp:38-49` |
| yalantinglibs / msgpack / zstd / yaml-cpp / OpenSSL | expected、struct_pack、事件信封、任务快照压缩、策略文件、SHA-256 | 见各文件 include |

### 被谁依赖

| 依赖方 | 如何用 |
|---|---|
| `rpc_service.cpp`（`WrappedMasterService`） | 转发 Put/Get/Remove/`ListTenantQuotaSnapshots`/`UpsertTenantQuotaPolicy`/`DeleteTenantQuotaPolicy`（`rpc_service.cpp:1579-1630`） |
| `master_admin_service.cpp` | `/metrics` 里的 `mooncake_tenant_quota_*` 指标（:418-460）、租户配额 HTTP 管理接口（:1241、1300、1326） |
| `master.cpp` | gflag/env → `MasterServiceConfig`（`tenant_eviction_high_watermark_ratio`、`weight_management_oplog_capability_confirmed` 等） |
| `mooncake-integration` | `store_py.cpp` 暴露 Store Python API；`engram_store_py.cpp` 暴露 `EngramStore` |
| `mooncake-reshard` | Python 层用 `mooncake.store` 普通对象 API + `ReplicateConfig(group_ids, data_type=WEIGHT)` 上传权重；**不调用 weight revision RPC**（因为还没暴露） |

### 一个重要边界：权重 API 目前没有客户端 RPC

`MasterService` 有 `BeginWeightImport/CommitWeightImport/AbortWeightImport/AcquireWeightRevisionLease/...`（`master_service.h:149-165`），但 `rpc_service.h` 里**没有对应方法**，只有一个被 supervisor 调用、明确注明不是 RPC 端点的 `RestoreFromStandby(..., const WeightMetadataSnapshot& weight_metadata)`（`rpc_service.h:282-286`）。`docs/source/design/mooncake-reshard/weight-management.md` 的 "Delivery Status" 也明确说客户端 RPC 是后续工作。所以：

- 权重 revision 的**元数据能力**目前只能在 Master 进程内 / 测试中使用；
- 实际的权重上传走的是普通对象路径：manifest 存 `METADATA`、payload 存 `WEIGHT`、同一个 `payload_group_id`，由 `WeightStoreManager::CommitWeightImport` 的 `SnapshotWeightGroup` 去校验。

### 数据怎么流过去

```
Client(reshard) --put(group_id, data_type=WEIGHT)--> Master 元数据分片
                                                        |
                          WeightStoreManager.CommitWeightImport
                                |                       |
                     SnapshotWeightGroup          OpLog(WEIGHT_METADATA_UPSERT)
                                |                       |
                        group 成员/大小/类型         standby 重放 / 快照
                                |
                      WeightMetadataStore.Publish(revision READY)
```

---

## 设计取舍与易踩坑点

1. **`requested_quota_bytes` 不等于 `effective_quota_bytes`。** 显式策略总和超过可分配容量时按比例缩减（最大余额法）。排查"为什么我的租户只有 60% 配额"时先看 `Recompute` 的容量回调（所有已挂载 segment 之和），而不是策略文件。

2. **`TryCharge(0)` 只是准入探针，不是预留。** `PutStart` 先用 0 字节确认租户没被关闭，真正扣费在 `AllocateAndInsertMetadata`。因此两个并发写可能都通过探针，然后一个在扣费时失败。错误码是 `TENANT_QUOTA_EXCEEDED`，且会打 `inc_tenant_quota_reject` 指标（`master_service.cpp:5656-5659`）。

3. **配额账本是"每对象三桶"，不是全局账本。** 账户在 `ShardedTenantQuotaTable<1024>` 里每租户一个、地址稳定；对象只持有 handle。所以 `TenantQuotaLedger` 的析构**不会**退配额——漏调 `EraseMetadata(kFull)` 或 `ReleaseAll` 就是永久泄漏。`kAbortOnly` 目前在主代码里**只定义未使用**（全仓库只在 `master_service.h:1410` 与 `master_service.cpp:2096` 出现），读代码时不要假设有调用方。

4. **`SettlePrimaryWrite` 的顺序是刻意的。** 先 `account->Release`、后改本地字段。反过来的话，`Release` 失败会留下"本地已清零、账户仍有余额"的状态，后续 `ReleaseAll` 会算错。

5. **覆盖写的额度是"转移"不是"重扣"。** `TransferReplacementCharge` 不动账户余额，只把旧账本的 committed+replaced 搬到目标的 replaced 桶。目标账本在 `UpsertEnd/PutEnd` 结算时把 replaced 桶一起释放。若覆盖写在中途失败，`ReleaseReplacement` 是唯一的回滚手段；漏掉它就会多扣一份额度。

6. **热缓存的 `block->size` 是逻辑长度，不是容量。** 复用块时不重置，所以 `SubmitPutTask` 必须拿 `GetBlockSize()` 做容量判断（`local_hot_cache.cpp:527-539`），否则块的可容纳尺寸会单调收缩。

7. **detached fill 块的 `key_` 归调用方所有。** `GetFreeBlock` 把块从 LRU 和索引里摘掉后，`ref_count` 为 0；`SubmitPutTask` 在**不持锁**的情况下 `block->key_ = key`。因此任何扫描 `blocks_` 的代码必须**先看 `ref_count`、再看 `key_`**——这就是 #4334 的全部内容（`git show ac4d29bf` 只改了两行谓词顺序）。旧顺序在扫描时会读到一个正在被并发赋值的 `std::string`，属于数据竞争（UB），而不是"逻辑上偶尔匹配错"。

8. **令牌校验与发布必须同锁。** `PutHotKey(block, token)` 在校验失败时把 `key_` 清空，让块回到空闲池；校验和发布之间如果放锁，`RemoveHotKey` 就能在中间把 generation 推进，随后过期数据会被重新发布。

9. **热缓存写回是 host memcpy，不能碰设备指针（#4764）。** `ProcessSlicesAsync` 用 accelerator registry 的 `FindDeviceForPointer` 逐 slice 判断，命中就跳过。这会导致"device 目的地的读不回填缓存"——这是有意的取舍，因为读路径本身不依赖缓存；缓存命中进设备内存仍然可用（TE 的本地拷贝是 device-aware 的）。注意：`ProcessSlicesAsync` 里某个 slice 提交失败会 `return`，后面的 slice 不再提交，但已经提交的任务不会回滚。

10. **晋升与动态副本的分工不是直觉的。** 晋升处理"0 个 MEMORY 副本、只有 LOCAL_DISK"的对象；动态副本只处理"已有 1..max-1 个 MEMORY 副本"的对象（`WantsAnotherMemoryReplica`，`dynamic_replication_controller.cpp:51-54`）。一个纯盘对象永远不会被动态副本计数，因为它连一个可读的 MEMORY 源都没有。

11. **晋升候选的 `count_` 是建议性上限。** `RecordLocked` 用 CAS 抢槽，`kLimit=50000` 是软上限；`EraseLocked/ForgetLocked` 必须成对调用，否则索引槽会被占满直到重试轮扫到它。`EraseMetadata` 里显式 `promotion_candidates_.EraseLocked(...)`（`master_service.cpp:2075-2077`）就是为此。

12. **候选刷新不重置失败历史。** `RecordLocked` 在已有候选时刷新 `last_seen/last_reason/last_error/sketch_score` 并重置 `retry_after/retry_count`，但**不动 `execution_failures`**（`promotion_candidate_tracker.cpp:42-60` 的注释）。真实读可以延长预算，但不能抹掉"这条准入链已经失败过 N 次"的事实，否则毒 key 会借读重新获得 3 次执行机会。

13. **动态副本的 pending 和 cooldown 不在 controller 里。** 它们在 `route::ObjectState`，与对象同生共死；`ClearPendingLocked` 会同时 `EraseLeasesForObject`——**清掉该 key 的所有租约**，不只是当前 proposal。这也是为什么 `ValidateCopyStartLocked` 在版本不匹配时会"顺手清 pending 再返回错误"：让下一个提案不被废弃租约拖住。

14. **动态副本的 `version_epoch` 是 `put_start_time` 的毫秒值。** 它不是单调版本号，但同一对象的每次覆盖写都会产生新的 `put_start_time`，足以让旧租约失效。`proposal.observed_version_epoch == 0` 表示"不校验版本"，是给外部调用者的兼容口子。

15. **KV 事件的 `cleared` 是"每租户"而不是"全局"。** 全局 `RemoveAll` 也要逐个租户发（`master_service.cpp:8001-8009`），因为订阅者按 tenant 建索引。`tenants_seen` 只在真正遍历到对象时插入，空 tenant 行不算被清空——这个细节修掉了一类"空租户导致 spurious cleared"的 bug（:7903-7907 注释）。

16. **epoch 哈希碰撞只会抑制 clear，不会误发。** 两个租户共享槽位时，任一提交都会 bump 同一个计数，于是另一个租户的 clear 会被判定"世代变了"而放弃。这是安全方向的偏差。

17. **publisher 的 `emit_object_key=false` 不是格式开关。** 它会让所有 stored/removed 事件被计为 `skipped_keyless_events` 并丢弃，只剩 `cleared` 能出去（`kv_event_publisher.cpp:108-117`、:361-366）。

18. **权重 `Publish` 的 CAS 是"整条 metadata 相等"。** 任何字段（包括 `updated_at_ms`）不同都会 `STALE_GENERATION`。所以 Prepare 与 Publish 之间不能有别的东西改同一 revision；`WeightStoreManager` 用 4096 个 group 锁把"同一 payload group 的 Prepare→Persist→Publish"串起来，而不是靠重试。

19. **权重写路径可能永久阻塞，如果 backend 不回调。** `PersistAndPublish` 的 `cv.wait` 没有超时（`weight_store_manager.cpp:47-49`）。它的正确性完全依赖 `WeightStoreBackend::DurableFinalize` 的契约：成功或失败都必须回调一次。改 backend 时这是首要约束。

20. **`mutation.no_op` 不写 OpLog，但仍然走 `Publish`。** 幂等重试（重复 Begin/Commit/Acquire）在 standby 上不会产生额外记录，这是"重放确定性"的基础。

21. **权重状态机校验很严，别绕过 `ValidateWeightRevisionMetadata`。** 例如 IMPORTING 必须 residency=UNKNOWN、READY 不能 ABSENT、DELETED 必须 ABSENT 且无 operation、`operation` 与 `operation_id` 必须同为零或同为非零（`weight_management.h:400-470`）。`Publish` 每次都会重跑校验（`weight_metadata_store.cpp:331-334`），所以 Prepare 里算出的非法 `next` 会在发布时被拒。

22. **`ClientTaskManager` 不是调度器，是邮箱。** 它只按 `client_id` 保存 PENDING 队列和 PROCESSING 集合，客户端通过心跳 `pop_tasks` 拉取（`master_service.cpp:12620`）。Master 的 `job_dispatch_thread_` 是另一套机制（`JobDispatchThreadFunc`），两者不要混。任务超时（pending 按 `created_at`、processing 按 `last_updated_at`）由后台清理线程周期调用 `prune_expired_tasks/prune_finished_tasks`（:2825-2830）。

23. **特性层里有四五种"后台线程"，没有统一线程池。** `ThreadPool`（38 行）几乎不被这些特性使用；实际存在的是 `KvEventPublisher::worker_`、`LocalHotCacheHandler` 的 2 个 worker、`DynamicReplicationController` 的 admission 线程、Master 的 eviction/job-dispatch/task-cleanup 线程。排查并发问题时不要假设"所有任务都跑在同一个池里"。

24. **`common/` 与 `config/` 不含业务逻辑。** `config/local_hot_cache_config.cpp` 只是环境变量解析（含"负数/非法值回退默认并 WARNING"的老兼容行为）；`common/result.h` 只是 `tl::expected → Python int` 与调试打印工具。读特性逻辑时可以直接跳过它们。

25. **一处读代码时注意到的引用计数边界（未在测试中验证）。** `RedirectToHotCache` 在 `GetHotKey` 成功（`ref_count++`）之后才检查尺寸，尺寸不一致时直接 `return false`（`client_service.cpp:1941-1949`），**没有调用 `ReleaseHotKey`**。而调用方只在 `cache_used == true` 时才在传输结束/失败路径释放（`client_service.cpp:1409-1411`、`1586-1589`、`1607-1611`、`1622-1627`）。因此"缓存里是旧尺寸、查询结果里是新尺寸"的竞态（例如对象被其他客户端覆盖写后本客户端下次读）会让这个块的 `ref_count` 永久 +1，`GetFreeBlock` 再也不会复用它，直到 `LocalHotCache` 析构。是否属于已知问题、是否有意为之，本次精读**未完全确认**；改这段代码前建议先补一个"覆盖写后读旧缓存"的并发测试。

---

## 精读清单

按下面顺序读，每一步都给出"读完应该能回答什么"。

1. **锁序契约**：`mooncake-store/include/master_service.h:101-118`。
   应能回答：为什么策略锁必须在 `master_state_mutex_` 之前？recompute 锁和 segment 锁为什么不会嵌套？

2. **对象记录的三个成员**：`master_service.h:1093-1135`、`object_metadata.h:60-110`、`metadata/object_state.h:1-40`。
   应能回答：为什么 `ObjectMetadata` 必须 non-movable？`quota_ledger`、`promotion_candidate`、`dynamic_replication_pending` 分别归谁所有、受哪把锁保护？

3. **配额的原子字**：`tenant/quota.h:50-82` + `tenant/quota.cpp:89-230`。
   应能回答：bit63 的作用？`TryCharge` 的 seqlock 在保护什么？为什么 CAS 之后还要再读一次 sequence？

4. **有效配额算法**：`tenant/quota.cpp:28-85` + `327-331`。
   手推一个例子：容量 100、三个租户请求 60/60/30，最终各得多少？（提示：最大余额法，余数相同按 tenant_id 升序）

5. **分片外壳**：`tenant/quota_sharded_impl.h:47-64`、`127-153`、`173-197`。
   应能回答：`ApplyTenantPolicies` 为什么要先按 shard 分组？`RecomputeEffectiveQuotasLocked` 为什么要先收集全部快照再统一 apply？

6. **账本三桶**：`tenant_quota_ledger.h:19-63` + `tenant_quota_ledger.cpp:35-137`。
   应能回答：`SettlePrimaryWrite` 为什么先 `Release` 后改字段？`TransferReplacementCharge` 的三个前置条件分别防什么？

7. **Put 全链路**：`master_service.cpp:5589-5660` → `5404-5440` → `5330-5401` → `5663-5834`。
   边读边画：探针、预扣、Adopt、Settle、PublishKvStored 各在哪个锁区间内。

8. **覆盖写 Case C**：`master_service.cpp:6414-6575`（先读 6167-6195 的锁获取）。
   应能回答：为什么"有 read lease"时要先给新副本预扣？`replacement_charge` 临时账本在成功/失败两条路径上分别怎么收尾？

9. **回收路径**：`master_service.cpp:2020-2120`（`EraseMetadata` 全函数）。
   应能回答：三种 `QuotaEraseMode` 各释放哪些桶？为什么 `kPreserveOld` 下还有 committed 会打 ERROR？

10. **KV 事件发布器**：`kv_event/kv_event_publisher.h:19-128` + `src/kv_event/kv_event_publisher.cpp:296-352`。
    应能回答：drop-oldest 时为什么要额外推进 ZMQ sequence？批次大小和队列容量分别是多少？

11. **RemoveAll 的 epoch 协议**：`master_service.h:2270-2347` + `master_service.cpp:7870-8015`、`8017-8143`、`13325-13344`。
    应能回答：为什么 baseline 要在扫描前整数组快照，而不能"见到租户再读"？哈希碰撞的后果是什么方向？

12. **热缓存发布**：`local_hot_cache.cpp:82-126`、`374-443`、`492-609`。
    应能回答：detached fill 期间块在哪些数据结构里"不存在"？令牌校验失败时块怎么回到池里？

13. **#4334 与 #4764**：`git show ac4d29bf`、`git show 67df8c41`，配合 `local_hot_cache.cpp:154-190` 与 `client_service.cpp:5613-5650`。
    应能回答：两个修复各自防的是什么故障（数据竞争 vs 段错误）？为什么读路径不用改？

14. **晋升准入**：`master_service.cpp:9633-9785` + `promotion_candidate_tracker.cpp:34-176`。
    应能回答：五个闸门分别对应哪个 `PromotionQueueResult`？哪些会记候选，哪些会清候选？

15. **晋升重试**：`master_service.cpp:9361-9434` + `promotion_candidate_tracker.cpp:194-232`。
    应能回答：`kMaxRetries=64 / kTtl=300s` 是按什么推导的？`Due/Record/Backoff` 如何避免每轮全量扫对象？

16. **动态副本提案**：`dynamic_replication_controller.cpp:98-210`（热度+准入线程）、`286-431`（选型）、`master_service.cpp:9493-9588`（Master 集成）。
    应能回答：为什么 0 个 MEMORY 副本的对象不会被动态副本统计？目标 segment 的三个排序键是什么？

17. **动态副本 CopyStart**：`dynamic_replication_controller.cpp:583-660` + `master_service.cpp:6889-6904`。
    应能回答：`PendingCopyGuard` 在哪些返回路径上会触发 `ClearPendingLocked`？`RegisterCopyStartLocked` 成功时它为什么不会触发？

18. **权重导入**：`weight_metadata_store.cpp:127-351`、`weight_store_manager.cpp:13-112`、`master_store_backend.cpp:22-76`。
    应能回答：Prepare/Publish 的 CAS 基线是什么？`PersistAndPublish` 的条件变量在等哪个回调？`ValidateWeightGroupForCommit` 校验了哪五件事？

19. **权重租约与快照**：`weight_metadata_store.cpp:421-634`、`1135-1259`。
    应能回答：`fenced_metadata_generation` 的方向（`>=` 还是 `<=`）？为什么续期要取 `max`？快照校验里 `completed operation` 的特殊条件是什么？

20. **任务邮箱**：`task_manager.cpp:29-160`、`171-259`。
    应能回答：`pop_tasks` 为什么可能跳过 `all_tasks_` 里不存在的 id？pending/processing 两个计数在哪些路径上增减？

21. **Reshard 有界在途**：`mooncake-reshard/python/mooncake/reshard/transfer_engine/executor.py:355-527`，配合 `weight/_te/execution.py:73`、`weight/_te/reader.py:433`、`weight/_te/sink.py:437`。
    应能回答：默认值是多少？窗口满时的动作顺序？`_CompletionUnknown` 为什么要 retain 而不是直接回收？

22. **兜底设施**：`serialize/serializer.h`、`common/result.h`、`config/local_hot_cache_config.cpp`、`thread_pool.h`。
    快速过一遍即可，确认它们不含特性语义；需要时再回来查具体序列化字段或环境变量名。

---

### 附：本章核实过的关键常量速查

| 常量 | 值 | 位置 |
|---|---|---|
| 元数据分片数 | 1024 | `master_service.h:1098` |
| 配额表分片数 | 1024 | `quota_manager.h:99` |
| 配额计费上限 | \(2^{63}-1\) | `tenant/quota.h:54` |
| 权重组锁 | 4096 | `weight_store_manager.h:68` |
| KV 队列容量 / 批大小 / HWM | 65536 / 64 / 10000 | `kv_event_config.h:24`、`kv_event_publisher.cpp:19-20` |
| KV epoch 槽 | 1024 | `master_service.h:2289` |
| 晋升候选上限 / 重试 / TTL | 50000 / 64 / 300s | `promotion_candidate_tracker.h:110-121` |
| 晋升执行失败上限 / 每轮批量 | 3 / 128 | `master_service.h:2071-2079` |
| 动态副本窗口条目 / 准入队列 / 批 | 50000 / 50000 / 64 | `dynamic_replication_controller.h:241-244` |
| 动态副本 cooldown / lease / recreate | 30s / 30s / 60s | `dynamic_replication_controller.h:236-238` |
| 动态副本目标水位 | 0.85 | `dynamic_replication_controller.h:245` |
| 热缓存默认块大小 | 16 MiB | `local_hot_cache.cpp:13` |
| 热缓存 pending 队列 | 1024 | `local_hot_cache.h:305` |
| 任务邮箱 finished 历史上限 | 由 `TaskManagerConfig` 决定 | `task_manager.h:210` |

---

# 12 Rust / Go 实现

> 精读范围：`mooncake-store-rs/`（Rust 原生 Store，内部称 Store-RS）、`mooncake-store/rust/`（C++ Store 的 Rust 绑定）、`mooncake-store/go/`（同一份 C ABI 的 Go 绑定），以及 `docs/source/deployment/store-rs/`。
> 文中所有 `路径:行号` 均相对仓库根 `D:/xiaojun/work/Mooncake`，行号以本次精读时的 main 分支（`67df8c41` 附近）为准。

## 定位与职责

这个标题下其实塞了**三份互不相同的代码**，先把它们分开，否则后面每一句话都会讲串：

| 代码 | 语言/形态 | 行数量级 | 运行时依赖 | CMake 开关 |
|---|---|---|---|---|
| `mooncake-store-rs/` | 14 个 crate 的 Rust 原生实现（Store-RS） | ~18.3 万行 Rust（`find crates -name '*.rs' \| xargs wc -l`） | Redis/etcd + 对端 client；**不需要 master** | `WITH_STORE_RS`，默认 **OFF**（`CMakeLists.txt:29-32`） |
| `mooncake-store/rust/` | crate `mooncake_store`，约 2.4k 行的薄安全绑定 | `src/` + `build.rs` + examples/tests 合计 2461 行 | `mooncake_master` + metadata server + `libmooncake_store` | `WITH_STORE_RUST`，默认 **ON**（`CMakeLists.txt:26`） |
| `mooncake-store/go/` | package `mooncakestore`，CGo 绑定 | 约 977 行（含 `tests/`、`examples/`） | 同上 | `WITH_STORE_GO`，默认 **OFF**（`CMakeLists.txt:21`） |

后两者是**同一个东西的两种语言前端**：都只是 `mooncake-store/include/store_c.h` 那 19 个 C 函数的安全封装，实际语义（路由、副本、驱逐、租约）全部由 C++ 的 `RealClient`/`MasterService` 决定。Store-RS 则是**从零重写的另一套 Store**，它与 C++ 的唯一真实接触面是数据面（Transfer Engine / TENT 的 `.so`），通过运行时 `dlopen` 装配。

用一句话概括三者关系：

```text
                 ┌─────────────────────────────────────────────┐
Python/CLI/e2e → │ Store-RS (mooncake-store-rs, masterless)    │
                 │   route mesh + metadata backend + tenant    │
                 │   └─ dlopen libmooncake_{classic,tent}_shim  │
                 └──────────────────┬──────────────────────────┘
                                    │ 数据面
                       C++ Transfer Engine / TENT (.so)

Rust/Go 用户 → mooncake_store / mooncakestore（薄绑定）
                 └─ store_c.h → store_c.cpp → RealClient → mooncake_master（C++ 元数据主）
```

**为什么会有两套 Rust**：`mooncake-store/rust` 出现的动机是"让 Rust/Go 生态能调用已有的 C++ Store"，成本低但受制于 master；`mooncake-store-rs` 的动机写在 `docs/source/design/store/store-rs/architecture.md:5-11`——保留 Mooncake 的编程模型、复用 TE/TENT 数据面、**把 master 从路由热路径上去掉**。这两条路线互相独立，代码零共享，连构建入口都不同（Cargo workspace vs 单 crate）。

Store-RS 的定位细节（必须记住的三条不变式）：

1. **热路径不许打后端**。`architecture.md:13-20` 原文："Steady-state request hot paths must not depend on backend round trips." Redis/etcd 只在租约、segment 公告、路由策略、`MetadataOnly` 模式等控制面上出现；正常 put/get 的每一次路由查询走的是 client 之间的 authority mesh。
2. **路由与存储分离**。route owner（authority）负责"对象在哪"的权威版本，storage owner 负责"字节在哪、还剩多少容量"；两者通过 CAS + `BatchTrackReplicaRoutes` 解耦（`architecture.md:239-251`）。
3. **元数据后端只做四类持久化**：client lease、segment 公告与生命周期、需要持久化的 object route、tenant 策略与配额（`architecture.md:394-403`）。

能力覆盖上，Store-RS 远多于两个绑定。Store-RS 有：单/批量 put/get/remove/exist/size、注册缓冲区零拷贝、multi-buffer 读写、scatter-gather range read、张量并行重建（`mooncake-store-rs-tensor`）、多租户与严格配额、CLOCK 驱逐 + route-owner CAS 回收、冷层（Cold Tier）与 NoF、滚动升级 handoff/drain、admin HTTP、Prometheus/OTLP。两个绑定则严格等于 `store_c.h` 的 19 个函数对应能力，没有租户、没有配额、没有 handoff。

## 目录结构速览

### `mooncake-store-rs/`（Store-RS）

| 路径 | 一句话职责 |
|---|---|
| `mooncake-store-rs/Cargo.toml:1-18` | workspace 成员清单，14 个 crate，`resolver = "2"` |
| `mooncake-store-rs/Cargo.toml:25-26` | **`warnings = "deny"`**：读代码时看到的 `#[allow(dead_code)]` 都是刻意标注而非历史垃圾 |
| `mooncake-store-rs/CMakeLists.txt:7-12` | 硬前置：必须有 `transfer_engine` 与 `tent_shared` 两个 target |
| `mooncake-store-rs/CMakeLists.txt:37-69` | 只编两个 shim：`libmooncake_classic_shim.so`、`libmooncake_tent_shim.so` |
| `mooncake-store-rs/CMakeLists.txt:81-119` | 两条 Cargo 构建线：`build_store_rs_cli`（无 PyO3）与 `build_store_rs_python_extension` |
| `mooncake-store-rs/CMakeLists.txt:122-133` | wheel 只装 shim + `_store_rs.<soabi>.so` + 三个 bin |
| `crates/mooncake-store-core/src/traits.rs` | 三个核心 trait：`MetadataBackend` / `RouteDirectory` / `PlacementStrategy` |
| `crates/mooncake-store-core/src/route.rs` | 路由与租户词汇表：`ObjectKey`/`ObjectRoute`/`ReplicaRoute`/`ClientLease`/`CasResult` |
| `crates/mooncake-store-core/src/identity.rs` | `ClientStableId` / `ClientEpoch` / `ClientRuntimeId` / `NamespaceScope` |
| `crates/mooncake-store-client/src/client/mod.rs` | `StoreClient` 组装点与全部默认常量 |
| `crates/mooncake-store-client/src/client/builder.rs` | `StoreClientBuilder::build`：启动校验、租约、策略引导、后台句柄 |
| `crates/mooncake-store-client/src/client/facade.rs` | `MooncakeCompatibilityFacade`：54 个方法（**不是**粗读地图说的 119 个） |
| `crates/mooncake-store-client/src/client/runtime_write.rs` | put/batch_put 与远端副本写 |
| `crates/mooncake-store-client/src/client/runtime_io.rs` | get/batch_get、按 segment 分组、本地/远端分派、迁移读 |
| `crates/mooncake-store-client/src/client/runtime_alloc.rs` | 本地/远端分配预约 |
| `crates/mooncake-store-client/src/client/state_store.rs` | `StorageOwnerState` 的实现（结构体本体在 `state_core.rs:329`）：CLOCK、track_route、evict_one |
| `crates/mooncake-store-client/src/client/membership_sync.rs` | 后台刷新 live-client 快照与 tenant 策略缓存 |
| `crates/mooncake-store-client/src/control_plane/` | tonic 版控制面 client/server（路由/分配/驱逐/迁移/冷层） |
| `crates/mooncake-store-client/proto/control_plane.proto` | 30 个 rpc + 双向 `ControlStream`（`control_plane.proto:5-35`） |
| `crates/mooncake-store-client/build.rs` | workspace 里唯一发 link 指令的地方：protoc 生成 + 可选 `kvcs-capi`/`nof-spdk` |
| `crates/mooncake-store-route/src/shim/` | `RouteOperations`、`build_route_directory`、元数据适配 |
| `crates/mooncake-store-route/src/table/` | 单 authority 的本地路由表与 CAS |
| `crates/mooncake-store-route/src/mesh/` | Embedded WRH：选主、镜像、读修复、ranked fallback |
| `crates/mooncake-store-rs-transport-sys/` | `dlopen`/`dlsym` 装配层（classic/tent 两个函数指针表） |
| `crates/mooncake-store-rs-transport/` | `TentEngine` / `ClassicTransferEngine` 的安全封装 |
| `crates/mooncake-store-transport-core/src/lib.rs` | 只有 107 行，却定义了 `StoreTransport`/`StoreTransportFactory` 两个 trait |
| `crates/mooncake-store-rs-metadata/` | Redis/etcd/in-memory 三套 `MetadataBackend` 实现 |
| `crates/mooncake-store-rs-runtime/` | Python 兼容运行时：`CompatRuntimeArgs`、`StoreDispatcher`、shm、hot cache、dummy 服务 |
| `crates/mooncake-store-py/` | PyO3 扩展 `_store_rs`（`lib.rs` 只有 2 行 mod 声明） |
| `crates/mooncake-store-rs-cli/` | 三个命令：`mooncake-store-rs-client` / `-bench` / admin |
| `crates/mooncake-store-e2e/` | 1951 行的可运行系统验证 + `bin/nof_multi_client.rs` |
| `mooncake-store-rs/scripts/{e2e,tests,build,lib}/` | shell 级 e2e（route migration、hot cache、NoF、redis ttl recovery 等） |

### `mooncake-store/rust/` 与 `mooncake-store/go/`

| 路径 | 一句话职责 |
|---|---|
| `mooncake-store/rust/src/lib.rs:44-69` | feature 门禁：至少 `link`/`dlopen` 之一；两者都开时 `link` 胜出 |
| `mooncake-store/rust/src/store.rs:27-38` | backend 无关的关键：`cfg` 二选一，`include!(OUT_DIR/bindings.rs)` 或 `ffi_dlopen` |
| `mooncake-store/rust/src/store.rs:85-124` | `ReplicateConfig::to_ffi`：CString/指针数组的所有权契约 |
| `mooncake-store/rust/src/store.rs:639-645` | `Drop` 调 `mooncake_store_destroy` |
| `mooncake-store/rust/src/ffi_dlopen.rs:49-111` | 平台 `DEFAULT_LIBRARY` 常量、进程级 `OnceLock<MooncakeStoreLib>` + `INIT_LOCK` + `load_library`/`ensure_loaded` |
| `mooncake-store/rust/build.rs:190-191` | 纯 `dlopen` 构建时 `build.rs` 什么都不做 |
| `mooncake-store/rust/build.rs:193-457` | `link` 构建：大量 `rustc-link-search`（含从 `CMakeCache.txt` 解析路径）、17 条无条件 `rustc-link-lib`（`mooncake_store`/`transfer_engine`/`base`/`asio`/…）外加按存在性条件追加的 `asan`/`etcd_wrapper`/`hiredis`/`curl`/`cuda`/`cudart`/`mlx5`/`uring`/`zmq`/`crypto`/`gcov`，最后跑 bindgen |
| `mooncake-store/rust/src/generated/ffi_dlopen_bindings.rs` | **提交进仓库的生成物**（397 行），dlopen 后端的真实 ABI |
| `mooncake-store/go/mooncakestore/store.go` | 19 个 `*Store` 方法 + `New` + `toCConfig` |
| `mooncake-store/go/mooncakestore/config.go` | `ReplicateConfig` / `SoftPinAction`（Go 侧独立定义） |
| `mooncake-store/go/mooncakestore/errors.go` | 14 个哨兵错误 |
| `mooncake-store/go/build.sh:30-91` | CGO flags：`-Wl,--start-group` 里链 6 个静态库（`mooncake_store`/`mooncake_local_ssd`/`cachelib_memory_allocator`/`transfer_engine`/`base`/`mooncake_common`）+ 一堆系统库 |

## 核心数据结构

### Store-RS：三个 trait 是整个抽象的地基

`crates/mooncake-store-core/src/traits.rs` 只有 933 行，却定义了全部可替换边界。

**`MetadataBackend`**（`traits.rs:21-408`）——持久化面。方法大致分六组（另有冷层设备记录、object-id 查询、版本 floor 等零散方法未单列）：

- 租约：`allocate_client_lease` / `upsert_client_lease` / `update_client_state` / `get_client_lease` / `list_live_clients`；
- segment：`publish_segment` / `unpublish_segment` / `reserve_segment` / `release_segment` / `update_segment_state`；
- 路由：`get_object_route` / `compare_and_swap_object_route` / `list_object_routes_by_cold_backing` / `list_object_routes_by_nof_backing`；
- 策略与租户：`get_route_policy` / `put_route_policy_if_absent` / `get_tenant_policy(s)` / `put_tenant_policy`；
- 配额：`reserve_tenant_quota` / `finalize_tenant_quota` / `abort_tenant_quota` / `list_tenant_eviction_candidates`；
- handoff：`put_handoff` / `get_handoff`。

最值得读的是 `traits.rs:38-67` 两段文档注释，它们定义了 masterless 正确性的前提：

- `upsert_client_lease`：**必须拒绝** `epoch` 不大于该 `stable_id` 历史观测值的写入，除非 lease key 已存在（视为 refresh）。唯一的例外是"lease 已过期消失、但没有更高 live epoch"时的同 epoch 回收。
- `allocate_client_lease`：调用方**不提供 epoch**；实现方用 `max(active epochs, historical HWM) + 1` 原子分配，并在同一原子步内提交 lease key、per-stable-id 活跃索引和 HWM。

三套实现都照此契约写，但手段不同：in-memory 靠一把写锁；Redis 用一段 Lua（脚本常量 `ALLOCATE_CLIENT_LEASE_SCRIPT` 定义在 `crates/mooncake-store-rs-metadata/src/redis_backend.rs:309`，调用点在 `:1704-1718`，6 个 key + 5 个 arg，返回分配到的 epoch 字符串）；etcd 用多 key txn 循环。**之所以要这么严格**：没有 master 做仲裁，epoch 单调性就是"谁是当前活着的同一个 client"的唯一依据；一旦允许回退，旧进程的租约可能覆盖新进程，路由会指向已死 runtime。

**`RouteDirectory`**（`traits.rs:410-578`）——路由查询/发布面，几乎所有方法都带 `observer: &ClientLease`（`metadata()` 例外）。这个参数不是摆设：authority 候选集要按 observer 的 compatibility 与 `route_scope` 标签过滤，等于"每个 client 看到的路由拓扑可以不同"。几个刻意的默认实现：

- `get_object_routes_bounded`（`:431-437`）：存在性检查的专用路径，**不做穷尽 fallback 探测**。注释写得很直白："a miss is common and must not fan out to every route authority in the cluster"。
- `compare_and_swap_local_route`（`:490-498`）：绕过 mesh 选主的逃生口。
- `visit_routes_by_replica_owner`（`:511-528`）：兼容路径有 256 条硬上限，超出直接 `Unsupported`，逼调用方走分页 visitor。

**`PlacementStrategy`**（`:580-587`）：只有一个 `select_write_targets`，输入本地 runtime、live clients、副本数，输出目标 runtime 列表。它很小，因为真正的放置热路径在 `mooncake-store-client/src/placement.rs` 的 `PlacementPlanner` 里，这个 trait 更像"可注入策略"的最小契约。

### Store-RS：路由词汇表（`mooncake-store-core/src/route.rs`）

- `ObjectKey(pub String)`（`:16`）：**不是**用户 key，而是 scope 编码后的全局 key。`from_scope`（`:23-38`）在默认 domain/object_set 下编码成 `"<tenant>::<key>"`，否则 `"<tenant>::ns/<domain>/<object_set>/<key>"`；每段都过 `encode_key_component`（`:45-59`）做百分号转义，只保留 `[0-9A-Za-z-_.]`。这解释了为什么 Store-RS 能天然多租户而 C++ 绑定不能——tenant 被编进了物理 key。
- `RouteVersion(pub u64)`（`:73-79`）：单调递增，`next()` 用 `saturating_add`。所有写路径都是"读当前 version → 构造 next → CAS(expected=旧 version)"。
- `RouteState`（`:82-88`）：`Active` / `Deleting` / `Tombstone`。注意 `Tombstone` 已注明 "Legacy: no longer produced by local remove/eviction paths"，只为线兼容保留。
- `ReplicaRoute`（`:107-118`）：**两个 offset 是本章最容易讲错的地方**。
  - `offset: Option<u64>` —— 传输目标地址（TE/TENT 的 target_offset），是 payload 在 segment storage target map 中的字节位置；
  - `segment_offset: u64` —— 本地分配器/reclaim 使用的偏移。
  二者在"transport target 坐标 ≠ 进程虚拟地址"时**不相等**，本地直读也必须走 `offset` 再映射回本地注册区（架构文档 `:194-202`）。
- `ObjectRoute`（`:121-141`）：除 key/version/state/replicas 外，还带 `namespace`、`logical_key`、`canonical_key`、`sharing_scope`（共享/复用标识字符串）、`qos_tier`、`cold_backing`、`nof_backing`，以及 `compatibility: CompatibilityDescriptor`（参与 authority 选择与路由身份比较，常被漏看）。
- `ClientLease`（`:143-150`）：`runtime` + `state` + `compatibility` + `endpoints` + `expires_at_ms`。**租约即成员关系**：没有独立的成员表，`list_live_clients()` 就是"列出未过期 lease"。
- `SegmentAnnouncement`（`:152-173`）：容量、已用、`target_chunks`、`alignment_bytes`、`tags`，以及 `transport_endpoint` / `transport_segment_descriptor`（classic TE P2P 场景下用来免掉 Redis 段元数据查询，见架构文档 `:186-192`）。
- `CasResult`（`:190-196`）：`applied` + `current` + `version_floor`。`version_floor` 专门处理"我读的时候这个 key 还不存在，CAS 时发现已有人建了更高版本"的竞争。
- `RouteControlMode`（`:205-210`）：`MetadataOnly` / `EmbeddedWrh`（默认）。

### Store-RS：`StoreClient` 字段即子系统地图

`crates/mooncake-store-client/src/client/mod.rs:144-193`，一次读完就知道客户端持有什么：

```rust
metadata, route_directory, _control_plane, control_client,
allocator: Arc<Mutex<LocalAllocatorState>>,
storage_owner: Arc<StorageOwnerState>,
lease: ClientLease, lease_ttl_ms: u64,
live_client_cache, suspect_runtime_cache, membership_sync,
_async_eviction, async_replica_tracking, async_route_hit_reporting,
default_tenant, local_memory, transport, transport_factory,
write_mode, route_control, route_topk,
transfer_stall_timeout, request_timeout_override,
lifecycle_state, route_write_gate, startup_activation_pending,
heartbeat_repair_pending, tenant_quota_reservation_counter,
namespace_quota, execution_fairness, bandwidth_shaping, placement_policy,
state: Arc<Mutex<StoreState>>,
owns_local_state_lifecycle, owns_cold_tier_lifecycle, cold_tier_shutdown_mode,
restore_promotions, cold_tier, cold_restore_flights, deferred_cold_tier_reconciles,
```

配套默认值（`mod.rs:57-92`）读出来的运行参数：`route_topk = 2`、live-client 同步 `1s`、suspect runtime TTL `5s`、传输 stall 超时 `10s`、请求超时基/上限 `1s`/`60s`、吞吐下限 `32MiB/s`、put 重试 4 次、segment publish 租约 `30s`、tenant 配额预留 TTL `60s`、tenant 策略缓存 TTL `1s`/空闲 `30s`。

三个 `#[allow(dead_code)]` 字段（`:183-192`）：`owns_cold_tier_lifecycle`（`:184`）与 `cold_tier_shutdown_mode`（`:187`）确实只给 `Drop` 用；`deferred_cold_tier_reconciles`（`:192`）的 allow 则略陈旧——它在 `runtime_core.rs:87`（启动激活完成时）也被读取，并非纯 Drop 字段。这正是 `warnings = "deny"` 下的常见写法，但读的时候要分清哪条 allow 仍然必要。

### 绑定侧：`MooncakeStore` / `ReplicateConfig` / `mc_store_t`

Rust 绑定的核心只有两个结构体：

- `MooncakeStore`（`mooncake-store/rust/src/store.rs:145-147`）：`handle: ffi::mooncake_store_t`，即 `*mut c_void`。`:151-152` 手写 `unsafe impl Send/Sync`，理由是"底层 C 对象内部已同步"。`Drop`（`:639-645`）调 `mooncake_store_destroy`。
- `ReplicateConfig`（`:64-77`）：`replica_num` / `soft_pin_action` / `soft_pin_ttl_ms` / `with_hard_pin` / `preferred_segments`。它 `#[derive(Default)]`——**这一点后面会变成一个坑**。

`to_ffi`（`:85-124`）是所有权教学样本：先把 `Vec<String>` 转成 `Vec<CString>`，再从中取 `Vec<*const c_char>`，最后把 `ffi::mooncake_replicate_config_t` 和两个 Vec **一起返回**。调用方必须同时绑定这三个值，否则指针悬垂。`store.rs:611-617` 的注释把这个约束写死了："The caller must bind the returned tuple to a named variable in its own stack frame *before* taking a raw pointer"。

Go 侧对应 `Store`（`mooncake-store/go/mooncakestore/store.go:29-32`）只持有 `handle C.mooncake_store_t`；`ReplicateConfig` 在 `config.go` 里独立定义（`ReplicaNum int`、`SoftPinTTLMs *uint64`），与 Rust 侧结构相同但**校验更严**（见后文）。

C 侧唯一契约 `mooncake-store/include/store_c.h`：

- `:25` `typedef void *mooncake_store_t;`
- `:33-37` `enum mooncake_soft_pin_action`：`PRESERVE=0` / `ENABLE=1` / `DISABLE=2`，零初始化即 PRESERVE
- `:39-54` `struct mooncake_replicate_config`：7 个字段（`replica_num`、`with_hard_pin`、`preferred_segments`、`preferred_segments_count`、`soft_pin_action`、`has_soft_pin_ttl`、`soft_pin_ttl_ms`）
- `:56-62` 全章的"所有权宪法"：**所有 `char *` 参数在函数返回后不再被使用**，调用方可立即释放。
- `:68-146` 19 个函数：create/destroy/setup/init_all/health_check、put/put_from/batch_put_from、get_into/batch_get_into、is_exist/batch_is_exist/get_size/get_hostname、remove/remove_by_regex/remove_all、register_buffer/unregister_buffer。

实现端 `mooncake-store/src/store_c.cpp`：

- `:34-36` `StoreHandle` 包 `std::shared_ptr<RealClient>`。注释解释了为什么不能只存裸指针：`RealClient::create()` 返回 shared_ptr，并在 ResourceTracker 里注册了 weak_ptr。
- `:42-70` `to_replicate_config`：先**校验 action 范围再 cast**（注释说明 enum 底层是 uint8_t，越界会回绕成合法值），`config.replica_num = c_config->replica_num` 是**原样透传**。
- 几乎所有导出函数都套 `try { ... } catch (...) { return -1; }`，把 C++ 异常挡在 ABI 之外。

### Store-RS：路由 authority 与成员快照

- `EmbeddedWrhRouteDirectory`（`crates/mooncake-store-route/src/mesh/directory.rs:62-71`）：`route_topk`、`namespace`、`local_stable_id`、`metadata`、`authority_client`、`membership`、`async_mirror`、`baseline: Mutex<BaselineState>`。
- `BaselineState`（`:56-60`）：`member_ids` + `candidates` + `created_at`，TTL 300s（`:54`）。它是"membership 抖动时的兜底候选集"。
- `LiveClientCache`（`crates/mooncake-store-client/src/client/state_core.rs:5-10`）：`refreshed_at` + `leases` + `tenant_quota_policies`。注意 `snapshot()`（`:13-16`）在 `refreshed_at` 为 None 时返回 None，上层会抛 `InvalidState("live client snapshot is not initialized; build() must prewarm membership before serving requests")`（`:173-182`）——所以 builder 里"先 prewarm 再提供服务"不是优化，是**正确性要求**。
- `shared_live_client_cache(namespace)`（`:184-193`）：进程级 `OnceLock<Mutex<BTreeMap<String, SharedLiveClientCache>>>`，**按 route_namespace 共享**。同一进程内多个 `StoreClient`（例如 Python 多 worker）只要连同一个 keyspace，就共用同一份成员快照和 tenant 策略缓存。
- `SuspectRuntimeCache`（`:76-167`）：被 quarantine 的 runtime。`is_recovered_by`（`:148-167`）先要求 quarantine 时间已过，再判：若此前没有观测快照直接恢复；否则三个判据（expires_at 变大、状态变成允许写入、control address 变化）任一成立即视为"同一个 stable_id 换了进程"。要让这些判据生效，membership 刷新必须先 `reconcile_suspects(leases)`。
- `HotCacheKey` / `LocalHotCache`（`crates/mooncake-store-rs-runtime/src/hot_cache.rs`）：`HotCacheKey` 在 `:16-19`（`(tenant, key)`），`LocalHotCache` 在 `:92`；由 `MC_STORE_LOCAL_HOT_CACHE_SIZE`（总容量）与 `MC_STORE_LOCAL_HOT_BLOCK_SIZE` 等环境变量启用，块大小默认 16MiB（`DEFAULT_BLOCK_SIZE` `:14`），另可用 `MC_STORE_LOCAL_HOT_CACHE_USE_SHM=1` 走共享内存。

## 关键流程剖析

### 流程 A：`StoreClientBuilder::build` —— 一个 client 如何加入集群

入口 `crates/mooncake-store-client/src/client/builder.rs:393`。这是全章最值得逐行读的函数，因为它把所有子系统的启动顺序固定下来了。

调用链（按执行顺序）：

```text
StoreClientBuilder::build(expires_at_ms)
├─ 参数校验                         builder.rs:394-403
├─ 标签归一化                        builder.rs:407-419 → normalize_storage_label(:87)
├─ endpoints 补全（rpc/segment）     builder.rs:420-432
├─ 解析 effective tenant policy      builder.rs:445-459
├─ 创建 state/allocator/lifecycle    builder.rs:460-469
├─ 创建 control_client               builder.rs:470
├─ validate_startup_conflicts        builder.rs:482 → builder.rs:7-28
├─ allocate_client_lease             builder.rs:483
├─ bootstrap_route_policy            builder.rs:491-497 → builder.rs:846-869
├─ build_route_directory             builder.rs:517-525
├─ 冷层设备 bootstrap/别名            builder.rs:539-633
├─ StorageOwnerState::new            builder.rs:634-653
├─ ControlPlaneHandle::spawn_*       builder.rs:683-690
├─ bind_local_authority_service      builder.rs:691-695
├─ upsert_client_lease（含 labels）   builder.rs:700-710
├─ prewarm_live_client_cache         builder.rs:711
├─ prewarm_segment_target_chunk_cache builder.rs:729-734
├─ MembershipSyncHandle::spawn       builder.rs:757-765
├─ AsyncEviction/ReplicaTrack/Hit    builder.rs:766-783
├─ ColdTierHandle::spawn/disabled    builder.rs:788-798
└─ 组装 StoreClient                  builder.rs:799-842
```

分步说明：

1. **启动校验**（`:394-403`）：`default_tenant` 不能为空；`route_topk >= 2`。后者不是拍脑袋——topk=1 意味着没有镜像 authority，读修复与 CAS 失败重试全部失效。
2. **存储角色归一化**（`:87-101`）：`storage=true` 但没有 `storage_bytes` → `InvalidState`；`storage_bytes=0` 且未显式声明 → 自动写 `storage=false`。同一段还补一个 `route` 标签（`:103-111`，`normalize_route_label`）：**仅在标签缺失时**按 `has_storage()` 填 `true`/`false`，用户显式设置的值不会被覆盖。**这个标签决定了它能否成为路由 authority**（`crates/mooncake-store-route/src/util.rs:12-18` 的 `route_capable` 只认 `labels["route"] == "true"`）。
3. **冲突检测**（`:482` → `:7-28`）：如果本地 segment 名与某个 *Active* 且 *可达* 的远端 runtime 冲突，直接失败。`validate_reachable_runtime`（`:63-85`）用 `control_client.probe_reachability` 探测；**探测失败（Unknown）也报错**，只有明确 Unreachable 才放行——这是防"脑裂后两个进程抢同一 segment"的保守策略。
4. **租约分配**（`:483`）：`allocate_client_lease(&template)`，模板里的 epoch 是 `ClientEpoch::default()`，会被后端忽略并替换成原子分配的新 epoch。
5. **路由策略引导**（`:491-497`，实现 `:846-869`）：第一个 client 用 create-if-absent 写入集群默认策略；随后所有 client 读回 effective 策略并**校验与本地一致**（`validate_route_policy`），不一致就启动失败。目的是防止不同 client 悄悄用不同的 `topk`/`route_control` 组合。
6. **构造路由目录**（`:517-525`）：把 `effective_route_control`/`effective_route_topk`、metadata、provisional lease、control client、共享快照、suspect 缓存一起交给 `build_route_directory`。
7. **控制面 server**（`:683-690`）：`ControlPlaneHandle::spawn_with_migration(host, local_authority, storage_adapter, storage_adapter, migration_adapter, cold_tier_control)`——注意 `storage_adapter` 传了两次，分别作为 allocator service 与 eviction service。这是"每客户端既是 RPC 客户端也是服务端"的实现点。
8. **成员快照预热**（`:711`）：`prewarm_live_client_cache` 必须成功，否则后续任何 `cached_live_client_snapshot` 都会报错。
9. **后台句柄**（`:757-783`）：membership sync、异步驱逐、异步副本跟踪、异步 route hit 上报，四条线程/任务在这里起。
10. **冷层**（`:788-798`）：只有真配置了 cold tier/NoF 才 `ColdTierHandle::spawn`，否则 `disabled` 占位。

一个容易忽略的细节：`startup_activation_pending`（`:434-443`）。当调用方要求 `initial_state=Active` 且开启 `activate_on_local_memory_registration` 时，**先以 Standby 发布租约**，等本地内存注册完成后再激活。这样别的 client 不会在内存还没注册好时把路由指向它。

### 流程 B：Store-RS 的 put —— 从一次写入到路由发布

调用链：

```text
facade.rs put/put_in_tenant (:1309/:1313)
└─ put_object (runtime_io.rs:663)
   ├─ route_ops().load_route(key)                 runtime_io.rs:677-680
   └─ put_object_with_policy_current              runtime_io.rs:413
      ├─ ensure_local_memory / flush_due_reclaims :420-421
      ├─ resolve_replication_policy               :449
      ├─ reserve_tenant_quota_for_put             :460
      ├─ reserve_replica_targets                  :471
      ├─ write_reserved_replicas                  :495 → runtime_write.rs:127
      ├─ 构造 ObjectRoute（version+1, replicas）   :528-566
      ├─ route_ops().publish_route (CAS)          :571-573
      ├─ track_remote_storage_owners_best_effort  :638
      ├─ finalize_published_route(quota)          :639-646
      ├─ storage_owner.track_route                :647
      └─ reclaim_route(previous)                  :648-658
```

分步：

1. **读当前路由**（`runtime_io.rs:677-681`）。在 `EmbeddedWrh` 下这一步是 authority RPC，不是 Redis 查询。
2. **QoS 与策略解析**（`:443-454`）：`qos_tier` 缺省继承旧路由；`resolve_replication_policy` 把请求策略与租户/默认策略合并；`ranked_candidate_count` 决定写重试上限。
3. **配额预留**（`:460-465`）：`reserve_tenant_quota_for_put` 在分配之前，因此"配额失败"不会浪费一次内存分配。注意顺序：**quota → alloc → write → publish → finalize quota**（架构文档 `:415-417` 明确写了这个顺序）。
4. **副本目标预约**（`:471-482`）：`reserve_replica_targets` 返回 `(targets, reservations)`；失败时先 abort 配额再返回。target 里带 `storage_runtime`、`segment_name`、`target_chunks`、`transport_endpoint`。
5. **写数据**（`runtime_write.rs:127-349`）。这是数据面核心：
   - 逐 target 判定本地/远端（`:149-156`）：`target.storage_runtime == self.lease.runtime && memory.has_storage_segment(...)`。
   - **本地**：`storage_address(segment, reservation.offset_bytes)` 拿到虚拟地址后 `ptr::copy_nonoverlapping`（`:157-169`）。注意这里用的是 `reservation.offset_bytes`（本地坐标）。
   - **远端**：先 `ensure_remote_runtime_reachable`，再 `open_segment_with_info`，然后 `storage_target_offset(target_chunks, segment_name, reservation.offset_bytes, len)` 把**分配器坐标翻译成传输坐标**（`:186-204`）。若 chunk 映射不可用，退回 `segment_relative_target_offset`（按 segment 内 buffer 顺序累加）。
   - 远端请求按 `max_registration_bytes` 切分（`target_buffer_transfer_requests`），按 `remote_write_chunk_limit` 分 chunk 提交；每 chunk 一次 `allocate_batch → submit_with_hints → wait_for_batch_completion_detailed → free_batch`（`:260-310`）。
   - 失败重试分两层：`write_reserved_replicas` 内部只在"远端 segment 缓存过期"且尚未刷过缓存时做**一次**整轮重试（`:336-345`），避免把真实网络故障当成缓存问题无限重试；外层 `put_object_with_policy_current` 的 `loop` 会在 `should_retry_put_after_write_failure`（`:2141-2151`：无 `required_preferred_segments` 且错误属 `Transport`/`NotFound`/`InvalidState`）且 `attempt < max_write_attempts` 时重新走"配额→分配→写"，每次重试前都会释放上一轮的分配并 abort 配额。
6. **构造路由**（`runtime_io.rs:534-570`）：`version = old.version.next()` 或 1；`replicas[priority] = { owner, segment_name, offset: 写侧返回的 absolute_offset, segment_offset: 本地分配偏移, length, checksum, tier: Dram }`。**两个 offset 在这里被分别赋值**，这是全章最该盯住的三行。注意 `offset` 的来源并不对称：本地 target 记的是 `storage_address()` 返回的本地虚拟地址，远端 target 记的是经 chunk 映射后的 transport target_offset；两者在本地 segment 上通常数值一致，但语义由 `replica_storage_target_offset` 统一收口（见流程 C）。
7. **路由 CAS 发布**（`:571-636`）：`publish_route` 走 `RouteOperations` → `RouteDirectory.compare_and_swap_object_route`。失败且 `expected_version` 为 None、同时 `cas.current` 也为 None 时，会用 `version_floor.next()` 或重新探测版本（`next_route_version`）再试一次（`:596-618`）。仍冲突则释放已写的分配、abort 配额、把 `current` 更新为 CAS 返回的 current 后 **continue 重试**，但受 `attempt < max_write_attempts` 上限约束，超过后返回 `Conflict`（`:619-635`）。
8. **发布后的收尾**（`:637-658`）：先 best-effort 通知远端 storage owner（`BatchTrackReplicaRoutes`），再 finalize 配额，再让本地 `StorageOwnerState` 记录热数据，最后对旧路由 `reclaim_route`。顺序刻意如此：**先让新路由权威可见，再回收旧字节**；旧路由回收失败只 warn，不影响 put 成功。

批量写的差异在 `batch_put_scoped_routed`（`runtime_write.rs:351` 起）：按 scoped key 排序后统一做配额准入（`reservation_order.sort_by` `:736`），任一条失败就 abort 已获得的预留（架构文档 `:417`）；远端传输按 tenant 分组，再按 scratch 窗口/公平性与 shaping 上限切 chunk，减少 completion 等待次数（`runtime_write.rs:1132-1240`）。另外 `batch_put_from_statuses` 在整批失败时会二分拆批重试，把错误细化到每个 key（`runtime_write.rs:393-433`）。

### 流程 C：Store-RS 的 get —— 路由解析 + 本地/远端分派

```text
facade.rs get/get_into/batch_get_into (:1562/:1651)
└─ resolve_objects (runtime_io.rs:2418)
   ├─ route_ops().load_routes(keys)             :2429-2445
   ├─ local_storage_segments / readable_runtime :2446-2447
   └─ 逐 key: select_readable_replica           :2466-2492
              └─ 无副本时 refresh membership 再选 :2473-2491
                 └─ 仍无且启用冷层 → cold backing read
└─ execute_batch_get_into (runtime_io.rs:2676)
   ├─ Phase 1 一把锁：分类 local/remote + 命中缓存算 copy 指令 :2736-2814
   ├─ Phase 2 缓存未命中：补 open_segment + target_chunks        :2824-2870
   ├─ 本地项 execute_copy_instructions / copy_local_replica_direct
   └─ 远端项 execute_remote_batch_get_chunk                     :4699
```

关键点：

- `select_readable_replica`（`runtime_io.rs:1925`）的排序规则**不是** `route_state_rank`/`replica_tier_rank`：那两个 rank 只在 `util.rs` 的 `canonical_route_key`（路由身份比较）里用到。真实规则是：先按 owner/segment 过滤掉不可读副本；若本地有可读副本，取 `priority` 最小的那个；否则对远端可读副本按 `priority` 排序后用 `replica_balance_hash(reader.stable_id, key) % n` 选一个——**这是刻意的读负载均衡，不是简单挑主副本**（`runtime_io.rs:1932-1956`，hash 函数 `:5576-5585`）。路由状态本身由调用方先过滤（`route.state != Active` 直接 `NotFound`）。**读热路径允许一次 membership 强制刷新**（`:2474`/`select_readable_replica_with_refresh` `:2034`），这是"读失败一次就重新看成员"的容错，不属于常规热路径的后端访问。
- `execute_batch_get_into` 明确分两阶段（注释 `:2734-2735` 与 `:2823`）：Phase 1 只持锁一次，把所有能算的 copy 指令算完；Phase 2 处理 cache-miss：先为每个缺失 segment 分别加锁 `open_segment_with_info`，最后再持一次锁统一算 copy 指令。这是为了避免"每 item 一把锁"的放大，而不是严格意义上的"Phase 2 只开一次锁"。
- 本地读也要做坐标翻译：`replica_storage_target_offset(info, chunks, replica)`（`:2800-2801`），而不是直接用 `replica.offset` 当虚拟地址。`copy_local_replica_direct`（`:5288`）同样走这条映射。**结论：本地读和远端读看到的是同一份 target 坐标语义**，这是架构文档 `:196-202` 强调的不变式。
- 读成功会报告 hit（`report_get_hits_best_effort` `:4072`；`report_grouped_route_hits_best_effort` `:4175`），本地直接进 `StorageOwnerState`，远端按 owner 聚合走 `BatchReportRouteHits`。hit 上报是 best-effort，不影响正确性。
- `is_exist`（`facade.rs:1232-1240`）走 `contains_active_route` → `RouteOperations::contains_active_route` → `RouteDirectory::contains_object_route`；`batch_is_exist`（`:1242-1257`）走 `contains_active_routes_bounded`。对 `EmbeddedWrh` 而言，单条与批量最终都落到 `contains_object_routes_bounded`（`mesh/directory.rs:1042-1046` 把单条委托给批量），因此**不做 fallback 探测也不上报 hit**（架构文档 `:259-260` 明确"existence checks stay boolean probes"）。一个容易看漏的细节：这两个 API 名字里的 "active" 在 `RouteDirectory` 这一层**并没有被强制执行**——`contains_object_route` 的默认实现就是 `get_object_route(...).is_some()`（`traits.rs:439-441`），`EmbeddedWrh` 的 bounded contains 也只是问 authority "有没有这条 route"（`directory.rs:262-330`），并不检查 `RouteState`。因此若某条路由处于 `Deleting`，`is_exist` 仍可能返回 true；需要严格判定可读性时应查 `query_route`/`get_size`。另一个细节是 bounded contains 会按 `observer_start_rank`（`directory.rs:1550-1559`，observer stable_id 的简单哈希）轮转起始 rank，避免所有 client 都先打同一个主 authority；并且在结果不全时会用 `baseline`（TTL 300s）里的候选集做一次兜底查询——这仍然只在 `route_topk` 范围内，不是全集群 fan-out。

### 流程 D：Embedded WRH —— 一次路由查询到底打了谁

这是 Store-RS 与 C++ master 模型分歧最大的地方。入口 `EmbeddedWrhRouteDirectory::get_object_routes`（`mesh/directory.rs:957-1009`）：

1. **取候选**（`:965`）：`authority_candidates(observer, force=false)` → 走 `ClientRouteMembership::live_clients`（`client/src/route_directory.rs:138-152`），即**本地快照**，不打 Redis。
2. **过滤**（`mesh/selection.rs:31-62`）：只保留 `state == Active`、`route_capable`（`labels["route"]=="true"`）、非 suspect、compatibility 匹配、`route_scope` 一致的 lease；同一 `stable_id` 只留最高 epoch。
3. **排序**（`selection.rs:64-89`）：对每个 key 计算 `weighted_rendezvous_score(namespace, key, stable_id, weight)`。实现是 `-ln(u)/w`，其中 `u` 由 FNV-1a 哈希 `namespace|key|stable_id` 映射到 (0,1)（`util.rs:99-123`）。分数**越小越优先**（`ranked.sort_by` 升序），weight 越大分数越小即越容易被选中。同分用 stable_id 字典序打破，保证跨进程确定性。
4. **按 rank 轮询**（`directory.rs:980-1005`）：
   - rank `0..route_topk`：主 + 镜像，带读修复观测；
   - rank >= topk：继续问更低排名的 authority，直到没有候选（fallback）。
   每一步内部先按 authority 分组，把同一 authority 的多个 key 合成一次 batch（`read_ranked_authorities` `:545-666`），减少 RPC 次数。
5. **合并**（`merge_fresher_route`，实现在 `mesh/read_repair.rs:70`）：同一 key 若多个 authority 返回不同版本，保留更新的。
6. **回填**（`backfill_missing_authorities` `:712-754`）：这不是"补偿读"而是**补偿写**——`compute_backfill_requests`（`read_repair.rs:142-170`）依据读修复观测为缺失/落后的镜像 authority 生成 `RouteCasRequest`，再经 `cas_authority_batch` 把较新的路由 CAS 写回镜像；失败只 warn 并标 suspect，不影响本次读结果。

CAS 路径 `compare_and_swap_route_selections`（`directory.rs:778-933`）同样是 rank 循环：

- 对每个 rank 分组；**本地 authority 直接调 `LocalRouteAuthority` 的 service**（走注册表，不过网络，`:814-873`），远端走 `authority_client.batch_compare_and_swap_routes`（gRPC，`:876-924`）。
- 一个请求在某个 rank 成功后立刻 `resolved[index] = Some(...)`，不再尝试更高 rank；失败才降级到下一个 rank。**这就是"primary CAS + mirror 兜底"的全部实现**。
- 任一 authority 报错都会 `maybe_mark_authority_suspect`，进入 `SuspectRuntimeCache` 隔离。隔离不是永久的：`reconcile_with_leases` 会在租约变大/状态变化/control 地址变化时解除（`state_core.rs:148-166`）。

`MetadataOnly` 模式则是另一条完全不同的路径：`shim/metadata.rs:13-17` 的 `build_metadata_route_directory` 直接包出一个只持有 `MetadataBackend` 的 `MetadataRouteDirectory`，所有读写都落到后端。它的 owner 列表上限用的是 `DEFAULT_ROUTE_OWNER_PAGE_SIZE`（`shim/contracts.rs:10`，值 256），超出时报 `Unsupported`（`metadata.rs:51-66`）。**它被定位为 bring-up/调试模式，而不是性能目标**（architecture.md `:19`）。

镜像发布是异步的：`AsyncRouteMirrorWorker` 在 `EmbeddedWrhRouteDirectory::new` 时 spawn（`directory.rs:135-140`），`Drop` 时 `shutdown` 并注销本地 authority（`:936-941`）。`RouteDirectory::flush_pending_route_mirrors`（`traits.rs:572-577`）是优雅退出时的补发钩子。

### 流程 E：Python 兼容层 —— `setup()` 如何落到同一个 StoreClient

`python/mooncake/store/__init__.py:95-117` 是分叉点：`MOONCAKE_STORE_BACKEND` 只接受 `cpp`/`rs`，分别 import `mooncake._store` 或 `mooncake.store.rs.store`（后者再 import `mooncake._store_rs`）。

rs 后端里，`PyMooncakeDistributedStore::setup(...)`（`crates/mooncake-store-py/src/python.rs:157` 起，函数体组装到 `:240`）：

```text
PyMooncakeDistributedStore::setup
├─ 组装 CompatSetupArgs（tenant/domain/object_set/labels/route_topk/timeouts...）
├─ CompatRuntimeArgs { setup, local_segment_name, initial_state, route_control }.build()
│    └─ runtime.rs:48
│        ├─ plan = setup.build()                     config.rs:218
│        │    └─ build_store_metadata_backend(url)   store_backend.rs:9
│        ├─ 选 transport factory                     runtime.rs:67-79
│        ├─ LocalMemoryConfig（storage/scratch/水位/hugepage）
│        ├─ StoreClientBuilder...build()             runtime.rs:104-135
│        └─ 可选 HTTP transport server publish
├─ StoreDispatcher::spawn_with_timeout_config_and_namespace_scope   dispatcher.rs:241
└─ finalize_real_dispatcher_setup                    runtime.rs:14-25
     ├─ register_local_memory()
     ├─ initial_state == Active → activate()
     └─ start_heartbeat_loop(lease_ttl_ms)
```

要点：

- `metadata_url` 是**必填**（`config.rs:255-259` 显式拒绝空串），且只认 `redis://` 或 `etcd://`；`http(s)://` 会明确报 "http metadata endpoints are not supported in store-rs"（`store_backend.rs:34-38`）。而 `transport_metadata_url` 是 TE 的输入，允许 `P2PHANDSHAKE`（仅 classic_te；`config.rs:401-407` 对 Tent 后端明确拒绝）。
- `StoreDispatcher`（`dispatcher.rs:170-189`，**不是** `config.rs`）持有 `Arc<StoreClient>`、`Arc<HealthChannel>`、`Arc<Mutex<RegionMap>>`、`Arc<tokio::Runtime>`、`hot_cache: Option<Arc<LocalHotCache>>`，以及 heartbeat 循环句柄与超时配置。所有 Python 调用都通过 `run` / `run_async`（`dispatcher.rs:509-540`）投递到 tokio runtime；PyO3 侧再配合 `allow_threads` 释放 GIL。
- 心跳循环（`dispatcher.rs:1901-1960`）默认 30s（`DEFAULT_HEARTBEAT_INTERVAL_MS` `:31`），失败退避为 `heartbeat_interval_ms.clamp(500, 1000)`（`:1897-1899`）；当**距上次成功的时间** `>= lease_ttl_ms` 时打 CRITICAL 日志（不是"连续失败次数超过 TTL"）。心跳本身是 `prepare_heartbeat(...).publish()`（`facade.rs:752-755`），成功后如果之前有失败，还会 `repair_local_metadata_after_heartbeat_recovery()` 补发本地元数据。
- `dummy` 路径（`setup_dummy`，`python.rs:243-256`）只做 `DummySession::connect(server_address, worker_scope)`，即连到独立的 `mooncake-store-rs-client` 进程；gRPC 承载 put/get，Unix domain side channel 传 shm fd（architecture.md `:512-522`）。`StoreDispatcher::hot_cache_fd()`（`dispatcher.rs:583`）就是 fd 传递的一端。
- **能力面差异**：`_CPP_EXPORTS` 47 个名字，`_RS_EXPORTS` 20 个名字（`__init__.py:9-93`，我逐个数的），交集只有 `MooncakeDistributedStore`/`MooncakeHostMemAllocator`/`BufferPool`/`RegisteredBufferPool`/`ReplicateConfig` 这 5 个；rs 独有的有 `TensorParallelism`、`ParallelAxis`、`ReadTarget`、`READ_MODE_*` 与 metrics 系列。切到 rs 后端后，访问 `EngramStore`、`Descriptor` 系列、`ObjectDataType` 枚举等会得到带专门文案的 `AttributeError`（`:133-137`）。

### 流程 F：薄绑定的一次 put —— 从 Rust/Go 到 C++ RealClient

Rust（`link` 后端）：

```text
MooncakeStore::put(key, value, config)          store.rs:239
├─ CString::new(key)?                            :245
├─ prepare_config(config)                        :619-636 → to_ffi :85-124
├─ ffi::mooncake_store_put(handle, key, ptr, len, cfg_ptr)  :252-258
│   （link：bindgen 静态符号；dlopen：ffi_dlopen 的 shim 转发到 MooncakeStoreLib） :251
└─ rc != 0 → StoreError::OperationFailed(rc)     :260-263
```

C++（`store_c.cpp:156-169`）：判空 → `to_replicate_config` → `std::span<const char>(value, size)` → `RealClient::put(key, span, *rc)`。**注意 `span` 不拥有数据**，依赖 `store_c.h:56-62` 的"返回后不再使用 char*"约定；这里传的是 `void*` 但语义相同。

Go（`store.go:175-199`）：

```text
Store.Put(key, value, config)
├─ toCConfig(config) → (C 结构体, []*C.char, error)   :180
├─ defer freeCStrings(cSegs)                           :184
├─ cKey := C.CString(key); defer C.free(...)           :186-187
├─ ptr = &value[0]（len>0 时）                          :189-192
└─ C.mooncake_store_put(handle, cKey, ptr, len, &cc)    :194
```

Go 侧的内存管理有三层，读的时候要分清：

1. `C.CString` 分配在 C 堆，必须 `C.free`（`defer`）；
2. `preferred_segments` 的每个字符串单独 `CString`，由 `freeCStrings` 统一释放（`:168-171`）；
3. **调用方的 Go slice 指针**通过 `unsafe.Pointer(&value[0])` 临时交给 C。Go 的 GC 在本调用期间不会移动堆对象（当前实现的保证），但**调用返回后 C 不得再持有该指针**——这正是 `store_c.h` 那条约定的另一半。

Go 的校验比 Rust 严（`store.go:126-166`）：`cfg == nil` 时显式填 `replica_num = 1`；`ReplicaNum <= 0` 直接 `ErrInvalidArgument`；`SoftPinAction` 越界拒绝；`SoftPinTTLMs != nil && action != Enable` 拒绝。Rust 的 `to_ffi` 只检查最后一条（`store.rs:95-99`），不做 `replica_num` 检查——这直接导致后文那个默认值坑。

## 与其他模块的边界

### Store-RS ↔ C++：只有数据面，且是运行时装配

这是本章最需要强调的边界事实：**`mooncake-store-rs/crates/` 下没有 `store_c.h`，也没有 `libmooncake_store`**。它与 C++ 的全部耦合是 CMake 产出的两个 shim：

- `mooncake-store-rs/CMakeLists.txt:37-52`：`libmooncake_classic_shim.so`，链接 `transfer_engine`；`OUTPUT_NAME mooncake_classic_shim`，`INSTALL_RPATH "$ORIGIN"`。
- `:54-69`：`libmooncake_tent_shim.so`，链接 `tent_shared`。
- `:122-125`：两者与 `tent_shared` 一起被装进 wheel 的 `mooncake/` 目录。

Rust 侧通过 `crates/mooncake-store-rs-transport-sys/src/lib.rs` 在**运行时** `dlopen`：

- `:13-28` 四个 `LibrarySpec`：`libtransfer_engine.so`（`MOONCAKE_CLASSIC_TE_LIB_PATH`）、`libtent_shared.so`（`MOONCAKE_TENT_SHARED_LIB_PATH`）、`libmooncake_classic_shim.so`、`libmooncake_tent_shim.so`。
- `:36-45` `load_library`：`dlopen(path, RTLD_NOW | RTLD_GLOBAL)`。`RTLD_GLOBAL` 在这里是必须的，但理由不是"shim 的符号要能被 dlsym 到"（对已拿到的 handle 做 `dlsym` 与 `RTLD_GLOBAL` 无关），而是**让先加载的库进入全局符号作用域**：`libmooncake_classic_shim.so` 的未定义符号要解析到早先加载的 `libtransfer_engine.so`，`libmooncake_tent_shim.so` 同理要解析到 `libtent_shared.so`。
- `:47-59` `load_symbol`：`dlsym` + `transmute_copy` 成函数指针。
- `:72-80` `load_library_from_spec`：逐个候选路径尝试 `load_library`，全失败后回落到裸 `file_name` 交给系统 loader；`:82-93` `resolve_library_candidates` 负责生成候选：先看环境变量（文件或目录），再按 `library_search_dirs()`（`:95-143`，模块自身路径、可执行文件路径、`MOONCAKE_BUILD_DIR` 下的 `mooncake-common`/`mooncake-asio`/`mooncake-store-rs/native-shims`/TE 与 TENT 的 src）。
- `:241-314` `ClassicApi`：25 个函数指针，其中 **16 个来自 `libtransfer_engine.so`**（`createTransferEngine`、`getLocalIpAndPort`、`installTransport`、`destroyTransferEngine`、`registerLocalMemory(Batch)`、`unregisterLocalMemory(Batch)`、`openSegment`/`openSegmentNoCache`、`closeSegment`、`allocateBatchID`、`submitTransfer`、`getTransferStatus`、`freeBatchID`、`syncSegmentCache`），**9 个来自 classic shim**（`mooncake_classic_republish_local_metadata`、`..._get_local_segment_descriptor_json(_size)`、`..._cache_segment_descriptor_json`、`..._get_batch_transfer_status`、`..._get_segment_first_buffer`、`..._get_segment_buffer_count`、`..._get_segment_buffer`、`..._get_max_mr_size`）。
- `:681-710` `TentApi`：21 个函数指针，其中 **19 个直接 `dlsym` 自 `libtent_shared.so` 的 `tent_*` 符号**，只有 2 个来自 tent shim（`:738-739`）。
- `:319-322` / `:715-718`：`OnceLock<Option<Api>>`，进程内只加载一次；加载失败返回 None，之后每次调用都是 `map_or(null)` 的空操作——**不会重试**。

`tent_shim.cc` 全文 49 行，只导出两个符号：

- `mooncake_tent_platform_free_memory`（`:8-14`）：转调 `Platform::getLoader().free`；
- `mooncake_tent_probe_rdma_max_mr_size`（`:16-49`）：遍历 IB 设备取 `max_mr_size` 的最小值。

**所以"按 shim 行数判断 FFI 规模"会严重低估**：真正的 ABI 面是 `libtent_shared.so` 的 19 个 `tent_*` 符号。

构建期校验只有"路径存在"：`mooncake-store-rs/crates/mooncake-store-rs-transport-sys/build.rs:27-37` 取出这些路径，`required_directory`/`required_file`（`:40-66`）再断言它们是绝对路径、必须是目录/已存在文件；可用 `MOONCAKE_SKIP_NATIVE_BUILD`（整体跳过）或 `MOONCAKE_SKIP_CLASSIC_TE`（跳过 classic shim 校验）绕过。**该文件没有任何 `cargo:rustc-link-lib`**。这与 `mooncake-store/rust/build.rs` 的 link 后端形成鲜明对照。

**但"Store-RS 完全不静态链接 C"是错的**（本次精读新发现）：`crates/mooncake-store-client/build.rs` 是 workspace 里唯一的链接点，且只挂在两个可选 feature 上：

- `kvcs-capi`（`:36-147`）：校验 `KVCS_SDK_ROOT` 下的 `COMMIT_ID` 版本必须是 `0.4.0`、架构与目标三元组一致，然后把 SDK 自带的 `rust/src/ffi.rs` 拷进 `OUT_DIR`，`rustc-link-lib=dylib=kvcs`（mock 模式下是 `kvcsmock`）。仅支持 Linux GNU + x86_64/aarch64（`:39-55`）。
- `nof-spdk`（`:160-200`）：用 `cc` 编译 `src/client/cold_tier/nof/extent_store/nof_spdk_shim.c`，`ar` 打成 `libmooncake_nof_spdk_shim.a`，再按 `pkg-config spdk_nvme spdk_env_dpdk spdk_syslibs` 的结果逐条发 link 指令。

这两个 feature 连的是外部设备/对象 SDK，**不是 `libmooncake_store`**。同一份 `build.rs` 还负责用 `tonic_build` + `protoc-bin-vendored` 生成 `control_plane.proto` 的 Rust 代码（`:4-13`），这是 CMake 完全不参与的一步。

安全封装层 `crates/mooncake-store-rs-transport/src/tent.rs`：

- `TentEngine::new`（`:159-172`）→ `create_engine_handle`（`:174-187`）：全局 create 锁 + 临时设置 Redis 环境变量（`TentRedisEnvGuard`）→ `config.apply()` → `ffi::tent_create_engine()`。
- `TentEngine` 内部：`raw: RwLock<handle>`（支持 `rebuild_engine_for_metadata_recovery` 原子替换，`:240-249`）、`registered_regions`、`owned_allocations`、`generation: AtomicU64`。`max_registration_bytes` 只在 RDMA 开启时才有值（`:164-167`）。

### Store-RS ↔ 元数据后端

`crates/mooncake-store-rs-metadata/src/store_backend.rs:9-45` 是唯一分派点：`redis://` → `RedisMetadataBackend`；`etcd://`（逗号分隔多 endpoint）→ `EtcdMetadataBackend`；`http(s)://` → 明确不支持；其他 → `unsupported metadata url scheme`。in-memory 后端不在这里，只给单测/`mooncake-store-test-utils` 用。

### Python 包 ↔ 两个原生扩展

- C++ 后端：`mooncake._store`（pybind，`mooncake-integration/store/store_py.cpp`）。
- Store-RS 后端：`mooncake._store_rs`，由 `crates/mooncake-store-py/Cargo.toml` 的 `[lib] name = "_store_rs"`、`crate-type = ["cdylib", "rlib"]` 产出；`python.rs:1542-1549` 的 `#[pymodule] fn _store_rs` 注册 6 个类。
- CMake 侧的安装名做了 SOABI 适配：`_store_rs.${Python3_SOABI}${CMAKE_SHARED_MODULE_SUFFIX}`（`mooncake-store-rs/CMakeLists.txt:34-35, 126-130`）。
- 三个 CLI 的 Python 入口在 `python/mooncake/_launcher.py:41-55`，只是 `exec` 对应二进制。

### 构建边界：CMake 管原生产物，Cargo 管 Rust 产物

`mooncake-store-rs/CMakeLists.txt` 的分工非常清晰：

- CMake 编：两个 shim、`transfer_engine`、`tent_shared`、Python 解释器探测（`:19-22` 要求 CPython SOABI）。
- Cargo 编：三个 bin（`build_store_rs_cli` `:81-97`）与 `_store_rs` 扩展（`build_store_rs_python_extension` `:99-116`）。CLI 目标传 **6 个环境变量**：`CARGO_TARGET_DIR`、`MOONCAKE_ROOT_DIR`、`MOONCAKE_BUILD_DIR`、`MOONCAKE_STORE_RS_DIR`、`MOONCAKE_CLASSIC_SHIM_LIB_PATH`、`MOONCAKE_TENT_SHIM_LIB_PATH`（`:83-89`）；Python 目标再多一个 `PYO3_PYTHON`（`:103`），共 7 个。粗读地图说 4 个，是低估了。
- 测试目标单独一套 `CARGO_TARGET_DIR`（`:137-150`），避免和 wheel 构建互相踩 target 目录。

### CI / 工具链边界

- Store-RS **不在任何 GitHub Actions workflow 里**：`grep -rl mooncake-store-rs .github/` 只命中 `.github/CODEOWNERS`。已发布 wheel 构建都不传 `WITH_STORE_RS=ON`（默认 OFF 生效），但 **`WITH_STORE_RUST=OFF` 并非"一律"**：`.github/workflows/_build-wheel.yaml` 只在三个 arm64 变体里显式传 `-DWITH_STORE_RUST=OFF`，三个 x86_64 变体没传，沿用该 option 的默认值 **ON**，所以 x86_64 wheel 实际会构建 Rust 绑定。
- 两个绑定则在 nightly 里有任务：`nightly.yml:299`（`run_store_rust_smoke.sh`）与 `:466`（同一脚本的 ASan/sanitizer 变体，`MOONCAKE_STORE_RUST_LINK_ASAN=1`）；Go 侧是 `:335`（普通 integration）与 `:477`（`MOONCAKE_STORE_GO_SANITIZED=1` 的 ASan 构建）。
- `.pre-commit-config.yaml:16` 显式 `skip: [store-rust-dlopen-bindings, store-rs-rustfmt, store-rs-clippy]`：这三个 hook 需要本地 CMake 构建树或 Cargo，pre-commit.ci 上不跑。hook 定义在 `:42-62`，其中 `store-rust-dlopen-bindings` 会在 `store_c.h` 变化时重生成绑定，CI 用 `git diff --exit-code` 卡过期绑定。

## 设计取舍与易踩坑点

### 1. 两个 Rust Store，名字只差两个字母

`mooncake-store/rust`（crate `mooncake_store`）与 `mooncake-store-rs`（workspace）完全无关。判别方法：

- 看有没有 `Cargo.toml` 的 `[workspace]`；
- 看代码里有没有 `store_c.h`/`RealClient`（有 = 绑定，无 = Store-RS）；
- 看 CMake 开关：`WITH_STORE_RUST`（默认 ON）vs `WITH_STORE_RS`（默认 OFF）。

讲反了会导致整章结论错误，例如"Store-RS 需要 master"（错）或"绑定支持多租户配额"（错）。

### 2. `route_topk` 不是 `replica_count`

`route_topk` 是**路由元数据**的 authority 副本数（主 + 镜像），决定"多少个 client 保存这个 key 的路由"；`replica_count` 是**数据副本**数，决定"多少个 storage owner 保存字节"。两者独立设置，`config.rs` 文档也专门提醒（`docs/source/deployment/store-rs/configuration.md:333`）。`route_topk < 2` 会被 builder 直接拒绝（`builder.rs:399-403`）。

### 3. `ReplicaConfig::default()` 的 `replica_num = 0` 是个真坑

- Rust 侧 `ReplicateConfig` 是 `#[derive(Default)]`（`store.rs:64`），`replica_num` 默认 **0**。
- 文档注释说 "0 means use the server default"（`store.rs:66`），`docs/source/api-reference/rust/mooncake-store.md:184` 也这么写。
- 但实现路径不支持：`store_c.cpp:54` 原样透传 `replica_num`；`master_service.cpp:5446-5451` 在 `replica_num == 0 && nof_replica_num == 0 && dfs_replica_num == 0` 时返回 `INVALID_PARAMS`。C++ 结构体的默认值 `replica_num{1}`（`mooncake-store/include/replica.h:109`）**只在 `config == nullptr` 时生效**。
- 结论：显式传 `Some(&ReplicateConfig::default())` 会被 master 拒绝；传 `None` 反而正常。
- Go 侧没这个问题：`toCConfig(nil)` 填 `replica_num = 1`，`ReplicaNum <= 0` 直接拒绝（`store.go:130-137`）。

**这是两套绑定语义不一致的真实缺陷，不是文档措辞问题**；修的时候要么改 Rust 的 `Default` 为 1，要么在 `to_ffi` 里补校验。

### 4. Rust 绑定漏了 `init_all`

`store_c.h:79-81` 有 `mooncake_store_init_all`，生成的 `ffi_dlopen_bindings.rs` 里也有（`:58`、`:253-260`），但 `store.rs` 没有任何安全包装，`ffi_dlopen.rs:137-214` 的 `shims!` 列表也**故意不含它**。Go 侧则有 `Store.InitAll`（`store.go:82-99`）。所以"19 个 C 函数 = 19 个 Rust 方法"的印象是错的，实际是 18 个。

### 5. `StoreTransport` 的定义地和"看起来的实现地"不是同一个 crate

`crates/mooncake-store-transport-core/src/lib.rs:19-107` 只有 107 行，定义了 `StoreTransport` 与 `StoreTransportFactory`，以及 `MemoryRegistration`。`crates/mooncake-store-client/src/transport.rs:104` 只是 `pub use mooncake_store_transport_core::{StoreTransport, StoreTransportFactory};`，具体实现 `TentStoreTransport`/`ClassicTeTransport`/`HttpStoreTransport` 在 client crate。按路径找 trait 定义会找错方向。

### 6. `offset` 与 `segment_offset` 不能混用

见"核心数据结构"。判断规则：

- 要交给 transport（submit/open segment/本地 copy）→ 用 `replica.offset`（再经 target chunk map 翻译）；
- 要交给 allocator（reserve/release/reclaim）→ 用 `replica.segment_offset` 或 `SegmentReservation.offset_bytes`。

`runtime_write.rs:186-204` 与 `runtime_io.rs:2799-2809` 是两处标准写法。架构文档 `:194-202` 明确写了"Scratch buffers ... are never published as object-addressable storage"。

### 7. 成员快照是"进程级 + 按 namespace 共享"的可变全局

`shared_live_client_cache`/`shared_suspect_runtime_cache`（`state_core.rs:184-206`）用 `OnceLock<Mutex<BTreeMap<...>>>` 做进程内缓存。影响：

- 同进程多个 `StoreClient` 共享同一份快照，一个 client 的 `force_refresh` 会刷新所有人的视图；
- 测试里如果并行跑多个不同 keyspace，必须用不同 namespace，否则互相污染；
- 缓存永不释放（`OnceLock` 静态表），长时间运行的多租户进程会累积条目。这不是泄漏（有界于 keyspace 数），但要知道。

### 8. `warnings = "deny"` 塑造了代码风格

`Cargo.toml:25-26` + pre-commit 的 `cargo clippy -- -D warnings`（`.pre-commit-config.yaml:56-62`）意味着：

- 所有"暂时没用"的字段/常量都必须 `#[allow(dead_code)]` 并附注释（例如 `mod.rs:76-89` 的 7 个 `DEFAULT_*`、`mod.rs:183-192` 的 Drop 相关字段）；
- 任何 rustc/clippy warning 都会让构建失败（clippy 侧是 `-D warnings`），没有"留几个无害 warning"的空间；
- 读代码时看到大量 `#[allow]`，应理解为"编译期强制的显式声明"，而非技术债。

### 9. `tent_shim.cc` 的行数会骗人

49 行里只有 2 个 `extern "C"` 符号，但 Rust 侧 `TentApi` 有 21 个函数指针。**评估 FFI 面要看 `load_api` 里的 `load_symbol` 数量，不是 shim 文件大小**。

### 10. Store-RS 是 Linux 取向，且默认关

- `mooncake-store-rs/crates/mooncake-store-rs-runtime/src/shm.rs` 直接用 `std::os::fd::OwnedFd`、Unix domain socket，并有 `#[cfg(target_os = "linux")]` 分支；
- `WITH_STORE_RS` 默认 OFF，wheel 默认不构建，CI 不跑；
- `mooncake-store/rust` 的 dlopen 后端虽然为 macos/windows 准备了 `DEFAULT_LIBRARY` 常量（`ffi_dlopen.rs:48-55`），但产出 `libmooncake_store.so` 的 `WITH_STORE_C_SHARED` 在 CMake 里硬性限制为 Linux/ELF（`mooncake-store/src/CMakeLists.txt:784-790` 直接 `FATAL_ERROR`，README 只是提了该开关，并没有"Linux-only"这句话）。

**改代码前先确认目标平台**，否则会在非 Linux 上撞到 fd/shm 相关编译或运行错误。

### 11. 生成物是提交进仓库的

`mooncake-store/rust/src/generated/ffi_dlopen_bindings.rs`（397 行）不是构建产物，而是受版本控制的 ABI 定义，由 `examples/generate_dlopen_bindings.rs` 生成、pre-commit hook 维护。改 `store_c.h` 后不重新生成，dlopen 后端会静默使用旧 ABI（bindgen 的 `dynamic_link_require_all` 只保证符号存在，不保证签名/布局匹配）。

### 12. 绑定层的错误码是"一锅端"

`store_c.h` 的所有函数只用 -1 表示失败（`get_into`/`get_size`/`remove_by_regex` 用负值），没有区分 NotFound/Timeout/Conflict。因此 Rust 的 `StoreError::NotFound` 变体实际上**从内部路径永远不会产生**（`error.rs:44-51` 的注释自己承认了），Go 侧也只能映射成 `ErrGet`/`ErrExist` 等粗粒度哨兵。想要细粒度错误，只能用 Store-RS。

### 13. Store-RS 的 batch 语义与绑定不同

- 绑定的 `batch_put_from`/`batch_get_into` 返回"每 key 结果码数组"，整体 `rc != 0` 才表示调用级失败（`store_c.cpp:185-215`、`:232-258`）。
- Store-RS 的 `batch_put_from` 返回 `Result<Vec<ObjectRoute>>`，`aggregate_batch_put_statuses`（`facade.rs:642-651`）遇到第一个 `Err` 就整体返回该错误，**已成功的条目不会回滚**（它们已经写完并发布了路由）；所以"整体 Err"只描述返回语义，不代表没有副作用（`facade.rs:1515-1518`）。部分失败语义由内部重试/回滚尽量收敛，但不暴露给调用方。
- Store-RS 的 `get_into_ranges`（`facade.rs:141-150`）返回 `Vec<Vec<Vec<i64>>>` 三层嵌套，且是 `unsafe`，它没有 C ABI 对应物——这是 Store-RS 独有的能力。

### 14. 未完全确认的点

- `mooncake-store-rs` 的 `cold_tier`/NoF 代码量很大（`state_store.rs` 2024 行、`cold_tier_state.rs` 997 行、`extent_store_engine.rs` 6287 行），本次只读了入口与类型，**没有逐行验证其持久化格式与恢复语义**；相关结论以 `docs/source/deployment/store-rs/cold-tier.md`、`nof.md` 为准。
- `control_plane.proto` 有 30 个 rpc，本次只核对了 rpc 名单（`:5-35`）与 client 侧的分组，**没有逐个核对 server 端 handler 的实现完整性**；`UnsupportedColdTierControlService` 的存在说明未启用冷层时部分 rpc 返回不支持。
- `MooncakeCompatibilityFacade` 的方法数我按 `fn <name>` 去重数得 **54**，与粗读地图的"约 119"不符；若按"含 tenant/in_scope 变体"或"含 impl 内部辅助方法"统计口径不同，数字会变，但 trait 声明本身是 54。

## 精读清单

建议按下面顺序读，每步都给出"读完应该能回答什么"。

1. **`docs/source/design/store/store-rs/architecture.md:1-120`**
   目标：建立 Store-RS 的心智模型。读完能回答：热路径不变式是什么、`EmbeddedWrh` 与 `MetadataOnly` 的区别、组件表里每个组件的热路径角色。

2. **`mooncake-store-rs/Cargo.toml` + `mooncake-store-rs/CMakeLists.txt`**
   目标：搞清 14 个 crate 的依赖方向与 CMake/Cargo 分工。读完能回答：谁编 shim、谁编 bin、wheel 装哪些文件、为什么 `WITH_STORE_RS` 需要 `WITH_TE` + `USE_TENT`。

3. **`crates/mooncake-store-core/src/traits.rs:1-200`**
   目标：掌握 `MetadataBackend` 的租约契约与 `RouteDirectory` 的 observer 语义。读完能回答：为什么 epoch 必须单调、`StaleEpoch` 在什么情况下允许同 epoch 回收、`get_object_routes_bounded` 为什么存在。

4. **`crates/mooncake-store-core/src/route.rs:14-220` + `identity.rs:14-140`**
   目标：把 `ObjectKey`/`ObjectRoute`/`ReplicaRoute`/`ClientLease`/`ClientRuntimeId` 的字段含义与序列化形态刻进脑子。读完能回答：`offset` 与 `segment_offset` 的区别、`route_topk` 与 `replica_count` 的区别。

5. **`crates/mooncake-store-client/src/client/mod.rs:57-200`**
   目标：用字段列表建立子系统地图。读完能回答：一个 `StoreClient` 持有哪些后台句柄、默认超时/心跳/重试参数各是多少。

6. **`crates/mooncake-store-client/src/client/builder.rs:393-843`**
   目标：完整走一遍启动流程。读完能回答：启动校验有哪些、租约如何分配、路由策略如何引导、本地 authority 如何绑定、后台任务何时起。

7. **`crates/mooncake-store-route/src/mesh/selection.rs` + `util.rs:1-125`**
   目标：看懂 WRH。读完能回答：候选如何过滤、分数怎么算、为什么是 `-ln(u)/w`、同分如何打破。

8. **`crates/mooncake-store-route/src/mesh/directory.rs:545-1009`**
   目标：读懂读路径的 rank 轮询与 CAS 的 primary/mirror 降级。读完能回答：一次 miss 最多打几个 authority、读修复在什么时候触发、suspect 如何隔离与恢复。

9. **`crates/mooncake-store-client/src/client/runtime_io.rs:413-736` + `runtime_write.rs:127-349`**
   目标：吃透 put 的"quota → reserve → write → CAS → finalize → reclaim"六段式。读完能回答：本地写与远端写分别怎么算地址、失败时按什么顺序回滚、重试条件是什么。

10. **`crates/mooncake-store-client/src/client/runtime_io.rs:2345-2566` + `2676-2900` + `5288-5324`**
    目标：吃透 get 的两阶段锁与本地/远端统一坐标。读完能回答：为什么 Phase 1 只持一次锁、本地读为什么也要做 target 映射、cold backing 在什么条件下接管。

11. **`crates/mooncake-store-rs-transport-sys/src/lib.rs:1-380, 606-748` + `tent_shim.cc`**
    目标：确认 Store-RS 与 C++ 的唯一接触面。读完能回答：加载了哪 4 个 `.so`、`RTLD_GLOBAL` 为什么必要、classic/tent 各有多少函数指针、shim 实际只导出几个符号。

12. **`crates/mooncake-store-rs-runtime/src/runtime.rs:14-157` + `dispatcher.rs:170-320, 1901-1960`**
    目标：理解 Python real 路径。读完能回答：metadata URL 如何选后端、transport factory 三选一、心跳循环的退避与告警条件、`StoreDispatcher` 的 tokio runtime 扮演什么角色。

13. **`crates/mooncake-store-py/src/python.rs`（`setup` 与 `_store_rs` 两处）+ `python/mooncake/store/__init__.py`**
    目标：看清 backend 选择与能力面差异。读完能回答：rs 后端缺哪些 C++ API、`MOONCAKE_STORE_BACKEND` 非法值会怎样、dummy 路径连的是谁。

14. **`mooncake-store/include/store_c.h` 全文 + `mooncake-store/src/store_c.cpp:29-98, 156-215`**
    目标：掌握 ABI 契约。读完能回答：19 个函数分别是什么、`char*` 生命周期约定、`to_replicate_config` 的校验点、`StoreHandle` 为什么持有 shared_ptr。

15. **`mooncake-store/rust/src/store.rs` 全文**
    目标：把安全封装的样板刻下来。读完能回答：`to_ffi` 的三个返回值为什么必须同时存活、`prepare_config` 的悬垂指针防护、`Drop` 做了什么、哪些 `unsafe` 是调用方责任。

16. **`mooncake-store/rust/src/ffi_dlopen.rs` + `build.rs` + `Cargo.toml`**
    目标：理解双后端。读完能回答：`link` 与 `dlopen` 的 feature 优先级、`ensure_loaded` 的并发保护、`link` 后端链接了哪些库、为什么 ASan 默认不链。

17. **`mooncake-store/go/mooncakestore/store.go` 全文 + `build.sh:30-91`**
    目标：理解 CGo 生命周期与链接图。读完能回答：`toCConfig` 比 Rust 严在哪、`defer freeCStrings` 覆盖哪些分配、`CGO_LDFLAGS` 为什么要 `--start-group`。

18. **`scripts/ci/run_store_rust_smoke.sh` / `run_store_go_integration.sh` / `mooncake-store-rs/scripts/e2e/*.sh`**
    目标：知道怎么验证。读完能回答：哪些场景需要真实 master/metadata、哪些跑在 CI、Store-RS 的 e2e 如何覆盖 route migration 与 hot cache。

按这 18 步读完，应能独立回答三类问题：**（a）** 给定一个 put/get 调用，说出它经过哪些线程/进程/RPC 与哪些锁；**（b）** 给定一个"路由指向已死 client"的现象，定位到 suspect cache、membership refresh、read fallback 中的哪一环；**（c）** 给定一处 ABI 或语义改动，判断需要同步修改 Store-RS、Rust 绑定、Go 绑定、生成绑定文件与文档中的哪几处。

---

# 13 Python 绑定与打包

## 定位与职责

这一层解决两个彼此耦合的问题：

1. **把 C++ 内核变成 Python 可调用的模块**。Mooncake 的数据面（RDMA/TCP 传输、张量序列化、零拷贝缓冲区）全部在 C++，pybind11 扩展负责把这些能力暴露成 `mooncake.engine`（TransferEngine）、`mooncake._store`（Store）、`mooncake._conductor`（Conductor 客户端），以及 `mooncake._ep` / `mooncake._fast_copy` 等辅助模块。
2. **把一堆 `.so`、原生可执行文件和 Python 源码打包成一个可 `pip install` 的 wheel**。仓库里有三棵互不自洽的源码树（`python/mooncake/`、`mooncake-integration/`、`mooncake-wheel/mooncake/`），由两套构建后端（scikit-build-core 与 setuptools）在构建期拼装成同一个 `mooncake` 包。

它在整体架构中的位置：向下依赖 `mooncake-store`（`PyClient`/`RealClient`/`DummyClient`）、`mooncake-transfer-engine`（`TransferEngine`、`Transport`、`SharedSegment`）、`mooncake-conductor`；向上被 vLLM/SGLang connector、REST 服务、CLI 以及 `structured_object_store` 使用。

一个必须记住的事实：**对外只有三个公开入口，其余都是私有扩展**。

| Python 侧导入 | 扩展模块名 | 定义位置 |
|---|---|---|
| `mooncake.engine` | `engine`（无下划线，直接公开） | `mooncake-integration/transfer_engine/transfer_engine_py.cpp:1743` |
| `mooncake.store.MooncakeDistributedStore` | `_store`（`OUTPUT_NAME "_store"`） | `mooncake-integration/store/store_py.cpp:2137`、`mooncake-integration/CMakeLists.txt:157-158` |
| `mooncake._conductor.ConductorClient` | `_conductor` | `mooncake-integration/conductor/conductor_py.cpp:286` |
| `mooncake._fast_copy` | C 扩展 | `python/mooncake/_fast_copy.c:192` |

`_store` 和 `_conductor` 刻意私有：用户应通过 `mooncake.store` 门面访问 Store，Store 门面才能在后端（cpp/rs）之间切换（`python/mooncake/store/__init__.py:95-138`）；Conductor 则由 `python/mooncake/conductor.py` 包一层（`from mooncake._conductor import ...`），而不是走 Store 门面。

打包有两条并行路径，分清它们才能读懂 CMake 里的分叉：

- **legacy（当前发布路径）**：`scripts/build_wheel.sh` + `mooncake-wheel/pyproject.toml`（setuptools）。`.github/workflows/_build-wheel.yaml:219` 调用的正是这个脚本；`mooncake-wheel` 的版本是 `0.3.13`（`mooncake-wheel/pyproject.toml:26`）。
- **unified（目标路径）**：仓库根 `pyproject.toml` 用 `scikit_build_core.build`，版本 `0.3.12.post1`（`pyproject.toml:11,15`），CMake 侧用 `if(SKBUILD)` 分支（`mooncake-integration/CMakeLists.txt:3-19`、`CMakeLists.txt:7-9`）。SKBUILD 路径的冒烟测试靠 `MOONCAKE_TEST_WHEEL` 环境变量（`python/tests/packaging/test_installed_wheel.py:26-39`：变量名在参数化列表里，未设置时 `pytest.skip`），仓库里没有 workflow 设置该变量——也就是说 CI 上它默认 skip。

## 目录结构速览

| 路径 | 职责 |
|---|---|
| `mooncake-integration/CMakeLists.txt` | 绑定层安装总入口：定义 `engine` / `store` / `_conductor` 三个 pybind11 模块，按 `WITH_TE` / `WITH_STORE` / `WITH_CONDUCTOR` / `USE_CUDA` / `USE_UBSHMEM` / `SKBUILD` 决定产物 |
| `mooncake-integration/store/store_py.cpp` | 3541 行，Store 绑定主体：`MooncakeStorePyWrapper`（对外名 `MooncakeDistributedStore`）、张量编解码、GIL 管理、`PYBIND11_MODULE(_store)` |
| `mooncake-integration/store/store_py_internal.h` | 被 `store_py.cpp:494` 在匿名 namespace 内 include 的“续页”：并行度规格、`WriterShardManifest`、`TensorIntoPlan`、分片计划构造函数 |
| `mooncake-integration/store/store_py_parallel_write.h` | 被 include 进 `MooncakeStorePyWrapper` **类体内部**（`store_py.cpp:1674`）：`batch_write_tensor_impl`、CUDA IPC 写路径 |
| `mooncake-integration/store/store_py_parallel_read.h` | 同类体内部 include（`store_py.cpp:1729`）：元数据前缀读、分片重建、`execute_tensor_into_plan_transfers` |
| `mooncake-integration/store/buffer_pool.cpp` | `BufferPool` / `BufferLease` / `BufferLeaseView` 绑定；阻塞式 acquire、溢出错配、`register_buffer` 回调的 GIL 处理 |
| `mooncake-integration/store/engram_store_py.cpp` | `EngramStore` / `EngramStoreConfig`，通过 `_get_pyclient_capsule()` 复用同一个 `PyClient` |
| `mooncake-integration/integration_utils.h` | 跨模块张量线格式：`TensorDtype`、`array_creators`、`TensorMetadata`、`Build/Validate/ParseTensorMetadata` |
| `mooncake-integration/transfer_engine/transfer_engine_py.cpp` | 2032 行，`PYBIND11_MODULE(engine)`；协议→分配器选择、buddy slab 分配、同步/异步/批量/页聚集传输 |
| `mooncake-integration/transfer_engine/transfer_engine_py.h` | `TransferEnginePy` 声明：`handle_map_`、`free_list_`、slab 尺寸表 |
| `mooncake-integration/transfer_engine/shared_segment_py.cpp` | `SharedSegment` 两阶段 create/complete 绑定 |
| `mooncake-integration/shared_segment.py` | `SharedSegment` 的 Python 侧：布局计算、all-gather blob、裸地址包成 torch 视图 |
| `mooncake-integration/allocator.py` / `allocator_ascend_npu.py` | `NVLinkAllocator` / `BarexAllocator` / `UBShmemAllocator`：注册为 torch 可插拔分配器 |
| `mooncake-integration/fabric_allocator_utils.py` | 定位包内 `.so`（`get_mooncake_so_path`）与 ctypes 探测（`probe_allocator_backend`） |
| `mooncake-integration/conductor/conductor_py.cpp` | `ConductorClient` 绑定，`tl::expected`→`{"ret": int, ...}` dict |
| `mooncake-integration/tests/*.py` | 只有两个 TENT 测试，直接 `import engine`，靠 `PYTHONPATH` 指向构建目录（`CMakeLists.txt:88-108`） |
| `python/mooncake/` | 规范 Python 包源：门面、async 包装、CLI、`_fast_copy.c`、`structured_object_store.py` |
| `python/mooncake/store/__init__.py` | 后端门面：`MOONCAKE_STORE_BACKEND` → cpp/rs，白名单懒转发 |
| `python/mooncake/store/rs/store.py` | Rust 后端兼容层：`_CACHE_COMPAT_POLICY` 降级策略 |
| `python/CMakeLists.txt` | 仅 SKBUILD 下加入：编译 `_fast_copy`，安装 `mooncake-wheel/mooncake/*.py` 与 reshard |
| `mooncake-wheel/pyproject.toml` + `setup.py` | legacy 打包元数据与 wheel tag / `_fast_copy` 扩展声明 |
| `mooncake-wheel/mooncake/` | 迁移未完成的模块暂存地（`_partial_read.py`、`dataproto_catalog.py`、connector、REST 服务、`pg.py`） |
| `scripts/build_wheel.sh` | 661 行 legacy 构建主流程：拷 `.so` → sed 改变体 → `python -m build` → auditwheel repair → 事后注入 EP/PG `.so` |

## 核心数据结构

### PyTensorInfo：一次张量操作的“句柄包”

```cpp
struct PyTensorInfo {          // store_py.cpp:42
    uintptr_t data_ptr;
    size_t tensor_size;
    TensorMetadata metadata;
    py::object owner;          // 保活
};
```

四个字段各有深意：

- `data_ptr` / `tensor_size` 是从 Python 对象上鸭子类型取出来的（`data_ptr()`、`numel()*element_size()`，`store_py.cpp:186-189`）。
- `metadata` 是线格式结构体。`BuildTensorMetadata` 先把它置 0，`build_full_tensor_metadata` 在返回前就填成 `tensor_size`（`store_py_internal.h:276`），`valid()` 再覆盖同一字段并做整体校验。
- `owner` 是最容易被忽略的字段：`extract_tensor_info_impl` 对输入张量会调用 `.contiguous()`（`store_py.cpp:183-185`），如果原张量非连续，这一步产生一个**新对象**；把新对象存进 `owner` 才能保证 C++ 后续做 RDMA 时那块内存不被 GC 回收。

`valid()`（`store_py.cpp:48-83`）不是简单的空指针检查，它把 `data_bytes` 覆盖成 `tensor_size` 后重新跑一遍 `ValidateTensorMetadata`，并要求 `data_offset + tensor_size` 与线格式自洽。

### 张量线格式（wire format）

`integration_utils.h:189-232` 定义了跨语言共享的内存布局，全部是 POD：

```cpp
struct TensorObjectHeader {   // 40 字节（4+2+2+4+4+4+4+8+8，自然对齐）
    uint32_t magic;           // 0x4d4f4f4e ('MOON')
    uint16_t version;         // 1
    uint16_t header_size;     // sizeof(TensorMetadata)
    int32_t  dtype;
    int32_t  ndim;
    uint32_t layout_kind;     // FULL=0 / SHARD=1
    uint32_t reserved_flags;
    uint64_t data_offset;
    uint64_t data_bytes;
};
struct TensorLayoutMetadata { // global_shape + local_shape + axis_count + axes[4]
    TensorShape global_shape; // int64_t dims[8]
    TensorShape local_shape;
    uint32_t axis_count;
    uint32_t reserved0;
    LayoutAxis axes[kMaxLayoutAxes];
};
struct TensorMetadata { TensorObjectHeader header; TensorLayoutMetadata layout; };
```

关键常量：`kTensorObjectMagic=0x4d4f4f4e`、`kTensorObjectVersion=1`、`kMaxTensorDims=8`、`kMaxLayoutAxes=4`（`integration_utils.h:171-174`）。按自然对齐计算 `sizeof(TensorMetadata) = 304` 字节（header 40 + layout 264），这个数值就是后文“只读元数据前缀”每次 ranged read 的长度。

`ValidateTensorMetadata`（`integration_utils.h:308-381`）的校验顺序值得逐条读，因为它同时是“跨版本兼容契约”：

1. magic / version / `header_size == sizeof(TensorMetadata)` 必须精确匹配；
2. `dtype` 必须在 `[0, NR_DTYPES)`，`ndim` 在 `[0, 8]`，`axis_count <= 4`，`layout_kind <= SHARD`；
3. `data_offset >= sizeof(TensorMetadata)` 且 `<= total_length`；
4. **`data_bytes` 必须精确等于 `total_length - data_offset`**；
5. 每个维度的 global/local shape 非负，且 `data_bytes` 必须等于 `TensorMetadataExpectedDataBytes()` 算出来的 `numel * element_size`（`integration_utils.h:280-306`）；
6. `ndim` 之后的 shape 槽位必须保持 `-1`（`MakeTensorShape` 初始化时填 `-1`，`integration_utils.h:234-241`）；
7. 每个 axis 的 `kind`、`shard_count>0`、`0<=shard_rank<shard_count`、`-1<=split_dim<ndim`。

这套校验意味着：**在枚举中间插入新 dtype 会破坏所有已存对象**。`TensorDtype` 的值（`FLOAT32=0 … NR_DTYPES=15`）已经写进 header 并被严格比对（`integration_utils.h:25-43`）。

`ParsedTensorMetadata`（`integration_utils.h:228-232`）把 header 里两个 `uint64_t` 转成 `size_t`，避免调用方反复做宽度转换。

### 分片 / 并行度元数据

`store_py_internal.h:17-55` 是一组**只存在于 C++ 侧、不进入线格式**的规格类型：

- `ParallelAxisSpec { kind, rank, size, split_dim, expert_id, stage_id }`（:17）——描述一条并行轴（DP/TP/EP/PP）。
- `TensorParallelismSpec { vector<ParallelAxisSpec> axes }`（:26）。
- `WriterShardManifest`（:41-55）：`magic=0x574d414e`（`kWriterShardManifestMagic`，:170）、`version=1`、`dtype`、`ndim`、`split_dim`、`shard_count`，加一个 `global_shape`。设计上它作为**独立的 Store 对象**存在，读取端按 `get_parallelism_manifest_key_name(base_key)` 取（:177），使“读回完整张量”不必先知道所有 shard 的 key 命名规则。**但要注意本版本的写入端并未接通**：`put_manifest_impl` / `upsert_manifest_impl`（`store_py.cpp:1566-1584`）在仓库里没有任何调用点或绑定；读取侧的 `load_parallelism_manifest`（`store_py.cpp:1708`）同样没有调用者。也就是说这条“独立 manifest 对象”的路径在当前 C++ 绑定中既不会被写出、也不会被走到，阅读时应把它当作预留设计而不是现状。

`TensorIntoPlan`（`store_py_internal.h:101-110`）是“把分片重新拼进用户缓冲区”的执行计划：`fragments` 描述每一次 ranged read 的 `(read_key, dst_offset, src_offset, size)`，`registered_buffer_ptr` 是用户缓冲区**所属的已注册 region 的基址**（不是用户传进来的指针，见 `store_py.cpp:1683-1706`），`materialized_metadata` 用于零长度/纯元数据情形。

### TransferEnginePy

`transfer_engine_py.h:45-336`。数据成员（:324-335）：

```cpp
std::shared_ptr<TransferEngine> engine_;
Transport *xport_;                                  // 声明后从未被赋值/使用
std::mutex mutex_;
std::vector<std::stack<char *>> free_list_;         // 16 个 size class
std::vector<char *> buffer_list_;                   // 从 buddy 顶层申请的大块
std::unordered_set<char *> large_buffer_list_;      // 超过最大 class 的独立分配
std::unordered_map<std::string, Transport::SegmentHandle> handle_map_;
bool auto_discovery_;
uint64_t transfer_timeout_nsec_;
```

slab 表在头文件里（`transfer_engine_py.h:37-43`）：`kDefaultBufferCapacity = 2GiB`，16 个 class 从 8KB 到 256MB。这是**经典的 buddy 分配器**：

- `findClassId(size)`（`transfer_engine_py.cpp:527-532`）返回能容纳 `size` 的最小 class；`size > 256MB` 返回 -1。
- `doBuddyAllocate(class_id)`（:534-554）递归向上一级借块，把一个块劈成两个压回 `free_list_[class_id]`。
- 顶层 class 一次性 `allocateRawBuffer(2GiB)` 并切成 256MB 块。
- `allocateManagedBuffer` / `freeManagedBuffer`（:556-584）用 `mutex_` 保护所有 free list 操作。**只有走 buddy 的块**（`class_id >= 0`）是“压回 stack、不真正归还内存”；超过最大 class 的大块走 `large_buffer_list_`，释放时会 `unregisterLocalMemory` 并真正 `freeMemory`（:576-580）。

关键副作用：`allocateRawBuffer`（:512-525）在分配后立刻 `engine_->registerLocalMemory(buffer, capacity, kWildcardLocation)`——即 **RDMA 注册发生在第一次分配时**，失败就 `freeMemory` 并返回 nullptr。所以 `allocate_managed_buffer` 返回 0 既可能表示 OOM，也可能表示 MR 注册失败。

### ScatterTransferTicket

`transfer_engine_py.cpp:73-131`。它把异步的 `ScatterTransferOperation` 包成一个可查询、可 drain 的对象：

- `status()`/`drained()` 加锁读（:88-96）。
- `drain(timeout_ms)`（:98-123）：如果已 drain 直接返回缓存状态；否则 `operation_->waitFor(bounded_timeout)`；`IsClock()` → `COMPLETION_UNKNOWN` 且**不置 drained**（可以再试）；成功/失败才置 `drained_` 并 `operation_.reset()`。
- 绑定里 `submit_scatter_write/read` 带 `py::keep_alive<0,1>()`（:1877、1884），保证 ticket 活着时 engine 不被回收——这是 DMA 生命周期正确性的关键。

### BufferPool 的三层对象

`buffer_pool.cpp:78-196`：

- `BufferPoolNative`：持有 `store_obj_`（`py::object`，让 Python 侧 Store 包装器活下去）、`py_client_`（`shared_ptr<PyClient>`）、`regions_`、`in_use_`、容量统计、`condition_variable`。
- `BufferLeaseNative`：`shared_ptr<BufferPoolNative>` + `shared_ptr<BufferHandle>` + `std::atomic<size_t> exports_`。`release_lease`（:252-264）在有导出的 view 时**拒绝释放**并抛错。
- `BufferLeaseViewNative`：每个 view 构造时 `add_export()`、析构时 `release_export()`（:198-204），从而让 lease 的析构知道是否还有 memoryview 活着。

`Region { handle, size, overflow_registered }`（:150-154）——`overflow_registered` 决定释放时是否需要回调 Python 的 `unregister_buffer`。

### Python 侧的三个“策略对象”

- `_CPP_EXPORTS` / `_RS_EXPORTS`（`python/mooncake/store/__init__.py:9-93`）：两个白名单的差集就是 cpp/rs 后端的 API 兼容性缺口清单。rs 模式下访问 C++ 专有名字会抛带说明的 `AttributeError`（:131-137），而不是静默返回 None。
- `_CACHE_COMPAT_POLICY`（`python/mooncake/store/rs/store.py:41-63`）：把缓存面方法（put/get/register_buffer…）的异常映射成失败码/空值；控制面方法保持严格抛错。
- `_NativeStoreWorker`（`rs/store.py:177-226`）：一个把原生 Store 关进守护线程、用 `queue.Queue + Future` 串行化的执行器。**注意：本版本中它只是被定义，`self._worker` 在 `__init__` 里恒为 `None`（:276），`_invoke`（:343-346）因此直接调原生对象。** 读代码时不要把这段当成实际线程模型。

## 关键流程剖析

### 流程一：`put_tensor` 从 Python 对象到 RDMA

调用链（全部在 `store_py.cpp`）：

```
put_tensor(key, tensor)                          :1162
  └─ put_tensor_impl                              :1154
       ├─ extract_tensor_info(tensor, key)        :213 → :160
       │    ├─ 类名含 "Tensor" 判断               :165-168
       │    ├─ tensor.contiguous()（可能拷贝）    :183-185
       │    ├─ data_ptr/numel/element_size        :186-189
       │    ├─ get_tensor_dtype(dtype)            integration_utils.h:105
       │    └─ build_full_tensor_metadata         store_py_internal.h:270
       └─ put_tensor_info_impl                    :1545
            └─ batch_put_tensor_infos_impl        :1291
                 ├─ try_dummy_cuda_ipc_batch_put_tensor_impl  parallel_write.h:161
                 └─ batch_write_tensor_impl                   parallel_write.h:208
```

分步说明：

1. **鸭子类型识别**（`store_py.cpp:165-168`）：只检查 `__class__.__name__` 里是否包含 `"Tensor"`。任何名字含 `Tensor` 的对象都会被继续当作 torch 张量使用 `data_ptr()`/`numel()`/`element_size()`。torch 本身只在运行时通过 `py::module_::import("torch")` 引入（`integration_utils.h:23`），构建期无依赖。
2. **连续性策略不对称**：输入张量允许非连续，会被静默 `.contiguous()`（可能多一次 D2D/H2D 拷贝，新对象由 `info.owner` 保活）；目标张量（`extract_tensor_destination_info`）强制 `is_contiguous()`，否则报错（`store_py.cpp:175-181, 218-222`）。
3. **元数据构造**：`build_full_tensor_metadata` 把全局/局部 shape 都设成同一个张量形状，`layout_kind=FULL`、`data_offset=sizeof(TensorMetadata)`；`BuildTensorMetadata` 先置 `data_bytes=0`，`build_full_tensor_metadata` 紧接着把它填成 `tensor_size`（`store_py_internal.h:270-277`），`PyTensorInfo::valid` 再覆盖一次并做完整校验。
4. **选择写路径**：
   - `use_dummy_client_` 且张量在 CUDA 上：走 CUDA IPC（见下）。
   - 否则进入 `batch_write_tensor_impl`。RealClient 路径把每个张量拆成**两个 buffer**——`PyTensorInfo` 内的 `TensorMetadata` 和 payload 指针（`parallel_write.h:183-192`），调用 `batch_put_from_multi_buffers(..., true)`（`store_py.cpp:1308-1314`）。DummyClient 路径则先 `allocate_client_buffer(total_size)`，`memcpy` 元数据 + `CopyToHost` payload，再一次性 `batch_put_from`（`parallel_write.h:246-292`）。
5. **GIL**：整个 `batch_write_tensor_impl` 的执行体在 `py::gil_scoped_release` 内（`parallel_write.h:229`）。也就是说，只有参数提取和最终的 Python 对象构造持有 GIL，传输期间不持有。
6. **结果回填**：`apply_tensor_batch_results`（`parallel_write.h:194-206`）用 `original_indices` 把“跳过的非法项”映射回去，保证返回列表长度与输入 keys 一致。

CUDA IPC 分支（`parallel_write.h:17-159`）是全书最微妙的一段：

- 在**持有 GIL** 时提取每个 owner tensor 的 `get_device()` 与 `torch.cuda.current_stream(device).cuda_stream`（:30-50），异常一律 `PyErr_Clear()` 吞掉。
- 然后在**释放 GIL** 后逐个 `ExportCudaIpcBuffer(data_ptr, tensor_size)`（:63-67）。如果任一失败，整体返回 `nullopt`，回退到普通 staged 路径。
- 校验 stream 的 `device_id` 必须与 IPC handle 的 `device_id` 一致（:71-87），不一致时**所有**已导出项都置为 `INTERNAL_ERROR`（而不是只错一个）。
- 元数据单独 `allocate_client_buffer` 并 `memcpy`（:101-108），以 `CudaIpcShmBufferRef` 传给 DummyClient——因为 DummyClient 在另一个进程里，不能解引用调用方的指针。
- 写之前对每个**唯一** (device, stream) 调 `SynchronizeCudaStream`（:141-150），确保 torch 侧的生产者 kernel 已完成。

### 流程二：`batch_get_tensor_into_cuda` 的两阶段读

`store_py.cpp:954-1103`。这是“把张量直接读进用户 CUDA 张量”的主路径，也是理解快照（snapshot）语义的最佳入口。

RealClient（非 dummy）路径：

1. **查询快照**：一次 `batch_query(keys)` 装入 `PyClient::QueryResultCache snapshot`，键为 key（:972-984）。整个操作后续只用这一份快照。
2. **只读元数据前缀**：`batch_get_tensor_metadata_prefixes`（:875-943）分配 `keys.size() * sizeof(TensorMetadata)` 的 client buffer，构造 `all_src_offsets = {{0}}`、`all_sizes = {{sizeof(TensorMetadata)}}`，用 `get_into_ranges` 只把每个对象的**元数据前缀（`sizeof(TensorMetadata)`，当前 304 字节）**拉进 scratch，然后 `parse_tensor_metadata_from_prefix` 解析（`store_py_parallel_read.h:25-43`）。
   - 为什么值得这么麻烦：走 `get_buffer`/`batch_get_buffer` 的整对象路径会把整份 payload 一起取回，对大张量（尤其需要 host staging 时）等于多一次全量拷贝。
3. **目标校验**：对每个目标张量 `extract_tensor_destination_info`，再 `tensor_destination_matches_metadata`（`store_py.cpp:416-431`）比对 dtype、ndim、字节数与 shape。不匹配的项**不报错**，只是留在 `results[i] = INVALID_PARAMS` 初值上。
4. **只读 payload**：对通过校验的项构造 `all_dst_offsets={{0}}`、`all_src_offsets={{stored.data_offset}}`、`all_sizes={{stored.data_bytes}}`，再次 `get_into_ranges_from_snapshot`（:1031-1037）。`data_bytes==0` 的项直接 `results[i]=0` 跳过（:1016-1019）。
5. **结果校验**：循环先把每一项写成 `INTERNAL_ERROR`，然后只有在“结果为负（错误码原样透传）”或“结果为正且**恰好等于** `data_bytes`”时才覆盖；其它正值（短读/长读）都保留 `INTERNAL_ERROR`（:1038-1051）。注意这里并**不是**“`result >= 0` 即成功”。

DummyClient 路径（:1055-1102）：只调用一次 `batch_get_tensor_metadata_prefixes`（:1055，未传 snapshot），然后用 `ExportCudaIpcBuffer` 把**目标**张量导出成 IPC handle，构造 `CudaIpcReadRequest{key, destination, source_offset, size}`，交给 `dummy_client->batch_get_into_cuda_ipc()`。也就是说，读方向也依赖 CUDA IPC，只是导出的是本地目标缓冲区。

**为什么必须用同一个 snapshot**：代码注释写得很直接（:970-972）——如果在“读前缀”和“读 payload”之间重新 query，中间发生的 upsert 可能让两次读到不同版本，最终拼出一个不存在的数据。**但这份保护只覆盖非 dummy 分支**：DummyClient 分支在 :1055 取前缀时没有传 snapshot，随后才用前缀里的 `source_offset/size` 发起 IPC 读，因此 dummy 路径仍存在“两次读之间被 upsert 改写”的窗口，只是 payload 读取用的是第一次读到的偏移量。

### 流程三：零拷贝边界——裸指针 API 族

真正的零拷贝不是 `get_tensor()`。`get_tensor()` 走 `buffer_to_tensor`（`store_py.cpp:233-368`），它：

1. 从 `BufferHandle` `new char[total_length]`；
2. `CopyToHost` 整块拷到 host（:254-261）；
3. `ParseTensorMetadata`；
4. `array_creators[dtype](exported_data, data_offset, tensor_size, take_ownership)` 造 numpy 数组（`integration_utils.h:45-57`，带 capsule 析构）；
5. `torch.from_numpy(...)`，再对 bf16/fp16/fp8 做 `.view(...)` 重解释（`store_py.cpp:353-361`）。

numpy 没有 bf16/fp8 dtype，所以存储类型是 `uint16_t`/`uint8_t`（`integration_utils.h:73-76`），必须在 `from_numpy` 之后用 torch 的 view 改名。**顺序不能改**，先 view 再 from_numpy 会失败。

零拷贝 API 族接受 `(uintptr_t, size)`：

- 写：`put_from`、`put_from_with_metadata`、`batch_put_from`、`batch_put_from_multi_buffers`、`upsert_from`、`batch_upsert_from`。
- 读：`get_into`、`batch_get_into`、`get_into_ranges`、`batch_get_into_multi_buffers`、`batch_get_into_multi_buffer_ranges`。
- 张量对象版：`put_tensor_from` / `get_tensor_into` / `batch_*_from` / `batch_*_into`。

使用前提是**先 `register_buffer(ptr, size)`**（`store_py.cpp:3004-3014`），由 store 把这段内存注册进 Transfer Engine。绑定注释明确说“Register a memory buffer for direct access operations”，绑定层本身没有“必须先注册”的前置断言。在需要 MR 的传输（如 RDMA）上，提交时按注册表查不到源地址会返回 `AddressNotRegistered`/“Memory region not registered by any active device(s)”（`mooncake-transfer-engine/src/transport/rdma_transport/rdma_transport.cpp:1131-1139`）；而 TCP 等不需要注册的传输可能照常工作。所以“不注册就一定会内存损坏”并不准确，但依赖具体传输、且错误会在很晚的阶段才暴露，仍然应当先注册。

`put_tensor_from`（`store_py.cpp:1477-1495`）的关键设计在 `make_raw_tensor_object_parts`（:99-149）：

- 缓冲区布局是 `[metadata][payload]`，`metadata_size = header.data_offset`、`payload_size = header.data_bytes`。
- 元数据通过 `AcceleratorRegistry.RuntimeAccelerators().CopyToHost(...)` 读取（:107-115），所以即使 `buffer_ptr` 是设备地址也能解析。
- **但 `parts.metadata_ptr` 保留调用方原始地址**（:143-147）。注释解释了原因：DummyClient 用注册表校验外部缓冲区；如果把设备元数据指针替换成私有 host 拷贝，这个指针就会“看起来未注册”。
- `batch_put_tensor_from`（:1497-1543）把每个对象拆成 `{metadata_ptr, metadata_size}` 与 `{payload_ptr, payload_size}` 两个 buffer，调用 `batch_put_from_multi_buffers`。这才是真正的零拷贝写。

`store_py_parallel_read.h` 里还有更复杂的“分片重建”路径：把多个 `key_tp_<rank>` 的 payload 用 ranged read 拼进一个完整的 `[TensorMetadata][全量 payload]` 缓冲区，必要时先 `register_buffer` 自己分配的重建缓冲区（:486、:560）。注意它的入口是 `get_tensor_with_writer_shard_full`（:456）和 `get_tensor_with_tp_full`（:521），两者都定义在这个类体内但**没有注册进 `_store` 的 `PYBIND11_MODULE`**，在本版本中不可从 Python 直接调用；真正注册的 `get_tensor_with_tp_into` / `batch_get_tensor_with_tp_into`（`store_py.cpp:1105-1152`）只是把 key 换成 `_tp_<rank>` 后转调 `get_tensor_into` / `batch_get_tensor_into`，并**不**走分片重建。名字相近的 `get_tensor_with_parallelism_into` 只存在于 Rust 后端（`python/mooncake/store/rs/store.py:848`），不是这条 C++ 路径的调用者。细节见 `TensorIntoPlan`。

### 流程四：TransferEngine 的一次同步写

```
te = engine.TransferEngine()
te.initialize(hostname, meta, "rdma", devices)        transfer_engine_py.cpp:372
  └─ initializeExt                                     :381
       ├─ initMemoryAllocator(protocol)                :391（定义 :229）
       ├─ TransferEngine 构造 / EFA/CXI 拓扑发现       :400-426
       ├─ engine_->init(conn_string, ...)              :428-438
       └─ free_list_.resize(16)                        :506
buf = te.allocate_managed_buffer(n)                    :556
te.transfer_sync_write(peer, buf, peer_addr, n)        :613
  └─ transferSync(...)                                 :962
       ├─ handle_map_ 查/开 segment（持 mutex_）        :971-983
       ├─ resolveNicHint / parseTransportHint          :993、:1007
       ├─ allocateBatchID + submitTransfer             :995-1015
       └─ 轮询 getTransferStatus / 超时                 :1041、:1057-1069
```

要点：

1. **分配器是文件级静态函数指针**（`transfer_engine_py.cpp:63-65`）：

   ```cpp
   static void* (*allocateMemory)(size_t) = nullptr;
   static void (*freeMemory)(void*) = nullptr;
   static std::string g_protocol;
   ```

   `initMemoryAllocator`（:229-306）按**第一次** `initialize` 的 protocol 选一次：`nvlink`→MNNVL、`musa`、`hip`、`nvlink_intra`、其它→64KB 对齐 `posix_memalign`（:293-300）。第二次调用只打 `WARNING` 并返回 true（:230-234）。**同进程内混用 rdma 与 nvlink 不会得到各自的分配器**。另外注意 `musa` 分支在未编译 `USE_MUSA` 时只打 ERROR 而**没有 return false**（:249-260，其它分支都有），随后仍会执行 `g_protocol = protocol; return true;`——此时两个函数指针仍是 nullptr。只要后续 `engine_->init` 没有因为未知协议名而失败，`initializeExt` 就会返回 0，而第一次 `allocateManagedBuffer` 才以 "Memory allocator is not initialized" 失败（:513-516）。这是与其它分支明显不一致的行为，读到这里值得停下来确认意图（本环境未构建验证 protocol=musa 时 `engine_->init` 是否必然成功）。
2. **64KB 对齐不是随便选的**：注释说明 Ascend SVM 的 `_devmm_mem_remote_map` 需要 64KB 页对齐的 src VA，否则 `aclrtHostRegister` 返回 EINVAL（:289-292）。
3. **buddy 分配器在 `mutex_` 内**，但 `allocateRawBuffer` 内部会调 `engine_->registerLocalMemory`——也就是**持锁做 MR 注册**（:512-525 在 :557 的 lock_guard 内被调用）。这是潜在的锁竞争点。
4. **segment handle 缓存**：`handle_map_` 以 target_hostname 为键，首次 `openSegment` 后缓存。`batchTransferSync` 在提交失败且 `CheckSegmentStatus` 不 OK 时会 `closeSegment`、`removeSegmentDesc`、`erase`（:1146-1161），其中 `removeSegmentDesc` 被 `if (auto metadata = engine_->getMetadata())` 包住（:1158-1159）：注释说明 TENT 模式下 `getMetadata()` 返回 nullptr，只有 classic 路径需要清 metadata，避免解引用空指针（#3995 的 P0 stale-handle 修复）。
5. **重试策略**：`max_retry = engine_->numContexts() + 1`（:1104）。第一次提交保留调用方给的 `nic_hint`，之后所有 entry 的 `nic_hint` 置 -1（:1128-1132），让调度器自由选本地 RNIC。
6. **超时**：`transfer_timeout_nsec_` 默认 30s（:308-316），实际超时是 `transfer_timeout_nsec_ + total_length`，注释写着“1GiB per second”（:1189-1190）——即按字节数线性放宽。
7. **GIL**：`transferSync` 第一行就 `py::gil_scoped_release`（:969），所以轮询期间不持 GIL。`transferSyncWrite/Read` 自身不释放，靠转调 `transferSync` 完成。
8. **buffer-protocol 变体**：`batch_transfer_sync_*_arrays`（:912-936）把手写 Python list 换成 numpy/array 的缓冲协议，`vectorFromIntegerBuffer`（:729-768）做严格校验：一维、8 字节整数、连续、非负。页聚集版 `batchArraysFromPages`（:824-863）把 `[F,5]` 的 field 表与页 id 展开成 `F*P` 条描述符，再按 `max_batch_size` 分批提交（:887-910）。

### 流程五：BufferPool 的 acquire/release 与溢出错配

`buffer_pool.cpp:315-351`：

```
acquire(size, block, timeout)
  ├─ block 缺省取 block_on_exhaustion_；timeout 缺省取 default_timeout_
  ├─ deadline = steady_clock::now() + timeout（无 timeout 则 time_point::max）
  ├─ size_class > max_bytes_ → 抛错
  ├─ py::gil_scoped_release                     :333
  └─ while(true):
       raise_if_open_locked()
       try_acquire_locked(lock, size)            :390
       ├─ 容量检查 → reserve_locked
       ├─ lock.unlock()                          :397  ← 关键
       ├─ client_buffer_allocator_->allocate()
       │    失败 → allocate_overflow_unlocked()  :440
       │            ├─ posix_memalign
       │            ├─ gil_scoped_acquire        :449
       │            └─ store_obj_.register_buffer(ptr, size)
       ├─ lock.lock(); unreserve_locked()
       └─ closing 检查；regions_/in_use_ 记账
     make_lease_locked → in_use_.insert(ptr)
     失败则 wait_until/wait 后重试
```

三个设计要点：

1. **分配期间不持锁**（:397 主动 unlock）。分配可能触发 `register_buffer` 这种昂贵的、需要 GIL 的调用，若持锁会阻塞所有其他 lease。
2. **持锁时碰 Python 对象**：`allocate_overflow_unlocked` 在**没有** mutex 的情况下 `gil_scoped_acquire` 回调 `register_buffer`（:448-452）；`unregister_overflow_unlocked`（:461-468）同样。函数名里的 `unlocked` 就是这个意思。反过来，`release_internal`（:470-491）先用 `lock_guard` 完成记账，再在锁外做 `unregister_overflow_unlocked`——顺序是刻意安排的，不要在锁内加 Python 调用。
3. **`close()` 的两阶段**（:358-378）：先 `closing_=true` 并解锁，然后在**释放 GIL** 的状态下等待 `reserved_regions_ == 0`；若有活跃 lease，回滚 `closing_` 并抛错。`acquire` 循环顶部 `raise_if_open_locked()` 让等待者尽快退出。

`BufferLeaseNative` 的析构（:217-222）吞掉所有异常并 `release_lease(false)`（不检查导出 view）。这意味着**如果 Python 侧还持有 memoryview，析构会绕过检查直接释放**；而显式 `release()` 会抛错。这是一个有意的不对称：析构路径不能抛。

### 流程六：SharedSegment 的两阶段创建

C++ 侧只有两个动作（`shared_segment_py.cpp`）：

- `SegmentCreate`（:36-57）：构造 `SharedSegmentOptions`，`SharedSegment::Create(name, size, options, segment, blob)`，**释放 GIL**，返回 `(segment, blob)`。
- `SegmentComplete`（:59-67）：把各 rank 的 blob 交回 `segment->Complete(blobs)`，同样释放 GIL。

Python 侧（`shared_segment.py:54-150`）负责所有编排：

1. 校验 `host_register` 必须 `mmap=True`（:88-89），`supported()` 探测（:90-101）。
2. 按**名字排序**计算布局（:105-109），保证各 rank 的字节偏移一致，与 dict 顺序无关。
3. 把布局编码进 segment 名字：`f"{name}|{tag}"`（:113、:120-129）——C++ 的 fingerprint 校验因此能发现 peer 之间 count/shape/dtype 不一致，而不必传完整声明。
4. `create` 之后做**失败收敛**：`_raise_if_any_rank_failed`（:310-335）通过 `all_gather_object` 让所有 rank 以相同错误退出，避免一半成功一半挂起。
5. `_all_gather_blob`（:338-342）用 CPU 字节张量做 all-gather（要求 gloo group；vLLM `GroupCoordinator` 通过 `cpu_group` 暴露）。
6. `tensors(block)`（:173-208）把裸地址包成视图：有 `device_addr()` 时走 `torch_npu._C._construct_storage_from_data_pointer`（:352-375），否则用 `ctypes.c_int8.from_address` + `torch.frombuffer`（:378-384）。每个 view 都挂 `_mooncake_segment` 反向引用保活。

### 流程七：legacy wheel 构建

`scripts/build_wheel.sh` 的骨架：

1. **拷产物到 `mooncake-wheel/mooncake/`**（:30-144）：`engine.*.so`→`engine.so`、`_store.*.so`、`libasio.so`、`libtransfer_engine.so`（BUILD_SHARED_LIBS 时）、`libmooncake_store.so`、`nvlink_allocator.so`、`ubshmem_fabric_allocator.so`、`mooncake_master`/`mooncake_client`/`transfer_engine_bench`、`ascend_transport*.so`……
2. **暂存迁移中的 Python 模块**（:190-241）：`MIGRATED_PYTHON_MODULES` 列表里的模块从 `python/mooncake/` 拷进来，`store/` 整目录、`reshard/` 整目录 `cp -R`。退出时由 `trap cleanup_migrated_python_staging EXIT`（:232）删除，`__init__.py` 用 `mktemp` 备份后还原（:234-235）。
3. **按变体重写 pyproject**（:281-362）：8 个变体（non-cuda / cuda13 / npu / efa / efa-cuda13 / efa-non-cuda / musa / hip）用 `sed -i` 改 name、description、keywords、classifier、甚至 `requires-python`。备份到 `pyproject.toml.backup`，EXIT trap 还原（:253-258）。
4. **构建**：`python -m build --wheel`（NPU 用 `--no-isolation`，:466-472）。
5. **auditwheel repair**（:487-581）：超长 `--exclude` 列表，包括 glibc、CUDA、torch、Ascend 全系、`libmpcomm.so*` 等。
6. **事后注入**（:586-614）：CUDA EP/PG 的 `.so` 在 repair **之后** `wheel unpack`→`cp`→`wheel pack`，因为 patchelf 会破坏 CUDA fatbin（注释明确提到 `cudaErrorInvalidKernelImage`）。同时给 `_ep*.so` 补 `rpath=$ORIGIN`。
7. **NPU 特化**（:620-653）：把 auditwheel 的 `.libs/` 摊平进 `mooncake/`，删 `.pth`，对所有 ELF 强制 `--force-rpath --set-rpath '$ORIGIN'`，并逐个验证 RPATH。

## 与其他模块的边界

**向下（C++ 内核）**

- Store 绑定依赖 `mooncake-store` 的 `PyClient`（抽象）、`RealClient`、`DummyClient`、`BufferHandle`、`ReplicateConfig`、`ResourceTracker`（`store_py.cpp:13-20`）。
  - `init_real_client()`（:542-551）负责 `RealClient::create()` 并把结果注册进 `ResourceTracker::getInstance()`。
  - `setup_dummy`（:2532-2548）切到 `DummyClient`，并生成 IPC socket 名 `"@mooncake_client_<port>.sock"`。
  - `ResourceTracker` 是进程级单例，故意 `new` 出来泄漏以避免静态析构顺序问题，构造时 `std::atexit(exitHandler)`，并用 `dlsym(RTLD_DEFAULT, "Py_IsInitialized")` 判断是否嵌在 Python 里——在 Python 内不安装自己的信号线程，以免 SIGINT 被屏蔽（`mooncake-store/src/real_client.cpp:776-799`）。
- TransferEngine 绑定依赖 `transfer_engine.h`、`transport.h`、`rdma_transport.h`、`transfer_engine_c.h`（`transfer_engine_py.h:27-31`），并链接 `$<TARGET_OBJECTS:rpc_communicator>`（`mooncake-integration/CMakeLists.txt:72`）。
- Conductor 绑定只依赖 `conductor_cpp_core`（`CMakeLists.txt:176`）。

**向上（Python 集成层）**

- `mooncake.store` 门面是唯一稳定的 Store API 面；`async_store.py` 用 `__getattr__` 把任意 `async_xxx` 映射到 `xxx` 并 `run_in_executor`（`python/mooncake/async_store.py:8-39`）。这意味着**任何同步方法都自动获得异步版本**，但也意味着线程池默认大小（`None` → 默认 executor）会限制并发。
- `structured_object_store.py` 依赖 `mooncake._partial_read`（只存在于 `mooncake-wheel/mooncake/`，`structured_object_store.py:27`）和 `mooncake._fast_copy`（:34）。这是三棵树互相依赖的最直观例子。
- `_fast_copy.c` 提供 `concat_arrays_into(list[ndarray], dest_ptr, ...)`，内部 `Py_BEGIN_ALLOW_THREADS` + `pthread_create` 并行拷贝（`python/mooncake/_fast_copy.c:34,133,138,180`），被结构化对象序列化热路径使用。

**构建期边界**

- `engine` 与 `_store` 都链接 `transfer_engine`；`_store` 还链 `mooncake_store`、`cachelib_memory_allocator`（`CMakeLists.txt:72-86, 159-170`）。
- `asio_shared`（`libasio.so`）被安装进包目录（:283-286），`engine.so` 靠 `INSTALL_RPATH "$ORIGIN"`（:57）与 `CMAKE_BUILD_WITH_INSTALL_RPATH TRUE`（:47）找到它；`tent_metrics` 同理（:295-298，注释解释了共享 `TentMetrics` 单例必须可加载）。
- `WITH_EP` 时创建一个 `engine.so -> engine<EXT_SUFFIX>` 符号链接（:318-333），因为 `ep.so`/`pg.so` 以裸名 `engine.so` 链接。缺这个链接时 `import mooncake.engine` 会以“找不到库”的形式失败，报错不会指向真正原因。

## 设计取舍与易踩坑点

1. **三个 `.h` 不是头文件，是 `store_py.cpp` 的续页**。`store_py_internal.h`、`store_py_parallel_read.h`、`store_py_parallel_write.h` 都没有 `#pragma once`；后两个被 include 在 `MooncakeStorePyWrapper` 类体内部（`store_py.cpp:1674`、`1729`）。单独打开会以为是独立 TU，且里面的成员函数定义离开类上下文根本编译不了。任何“把头文件当独立单元读”的结论都不可靠。

2. **GIL 是手工管理的，不是 `py::call_guard`**。三种模式并存：
   - 整个执行体包一层 `py::gil_scoped_release`（`batch_write_tensor_impl`、`transferSync`）；
   - release 块内再 `gil_scoped_acquire` 回来构造 Python 对象（`get`/`get_batch`，`store_py.cpp:664-689`、`701-737`）；
   - 持 C++ mutex 时重新 acquire GIL 回调 Python（`buffer_pool.cpp:448-452`、`461-467`）。

   第二种写法依赖“acquire 先析构、release 后析构”的逆序规则，能工作但非常脆弱。`get()` 里还有一个类型不一致：函数声明返回 `pybind11::bytes`，CUDA 拷贝失败分支却 `return pybind11::none()`（`store_py.cpp:683`）。此处**未完全确认**具体异常行为（本地未构建验证），但从签名看它会在返回处走一次 `bytes` 转换而不是把 `None` 交给 Python——错误路径上很可能二次抛错。

3. **“零拷贝”只在裸指针 API 上成立**。`get_tensor`/`batch_get_tensor`/`load_tensor_from_safetensor` 都会完整 host 拷贝。判断标准：参数里有没有 `uintptr_t buffer_ptr, size`。而且裸指针 API 要求先 `register_buffer`；绑定层不强制，但需要 MR 的传输会对未注册的源地址返回 `AddressNotRegistered`（见上文流程三），所以错误通常表现为传输失败而非静默成功——但并非所有传输都要求注册。

4. **输入/目标张量的连续性策略不对称**：输入自动 `.contiguous()`，目标强制要求连续。写代码时如果从 `get_tensor_into_cuda` 换到 `get_tensor_into`，很容易因为一个转置视图而失败。

5. **dtype 枚举只能追加**。`TensorDtype` 的值进入 header 并被严格校验（`integration_utils.h:316-318`），`array_creators` 是 `std::array<...,15>` 且靠下标索引（:61-77）。在中间插入会同时破坏 `ValidateTensorMetadata`、`array_creators`、以及已落盘对象的兼容性。

6. **bf16/fp16/fp8 的 numpy 绕行顺序固定**：先 `create_typed_array<uint16_t/uint8_t>`，再 `torch.from_numpy`，最后 `.view(bfloat16/...)`（`store_py.cpp:339-361`）。numpy 不认识这些 dtype。

7. **张量识别靠类名子串**，不是 `isinstance(torch.Tensor)`。任何 `__class__.__name__` 含 `"Tensor"` 的对象都会走鸭子类型；反之自定义 `nn.Parameter`（名字不含 Tensor）会被拒绝，除非其 `__class__.__name__` 恰好匹配。这是零构建期依赖 torch 的代价。

8. **静态分配器是隐式全局状态**。`allocateMemory`/`freeMemory` 是文件级指针（`transfer_engine_py.cpp:63-65`），一旦进程内第一个 `TransferEngine` 以 `rdma` 初始化，后续 `nvlink` 实例也用 64KB `posix_memalign`。多协议同进程场景必须把 `initialize` 顺序纳入设计。

9. **`file(GLOB SOURCES "*.cpp")` 与 `${CACHE_ALLOCATOR_SOURCES}` 都是空的**。前者匹配不到 `mooncake-integration/` 顶层的任何文件（该目录只有 `.py`/`.h` 和子目录），后者在整个仓库里从未被 `set` 过——只有两处使用（`mooncake-integration/CMakeLists.txt:54,152`）。读 CMake 时不要以为“还有别处源码”。

10. **模块命名不对称**：`engine` 直接公开，`store` 被 `set_target_properties(... OUTPUT_NAME "_store")` 改成私有（`CMakeLists.txt:157-158`）。Python 侧没有任何 `.pyi` 存根，IDE 补全全靠 `.so` 内省。`InnerTransferEngine` 只是把 C++ 的 `TransferEngine` 类型再注册一次、且没有任何方法绑定（`transfer_engine_py.cpp:2021-2022`），用于类型暴露/隔离，不是一个不同的 C++ 类型。

11. **两个 capsule 辅助函数不等价**。`make_pyclient_capsule` 生成**带名字** `"mooncake.PyClient.shared_ptr"` 的 capsule（`store_py.cpp:2358`），`_get_pyclient_capsule` 用它（:2431-2436）；而 `_get_pyclient_from_wrapper`（:2394-2415）生成的是**无名** capsule。`buffer_pool.cpp:41-46` 与 `engram_store_py.cpp:32-37` 都要求名字匹配，所以它们只能走前者。用错会得到 "unexpected PyClient capsule type"。

12. **`_NativeStoreWorker` 当前是死代码**。`rs/store.py:177-226` 定义完整、注释也描述了串行化模型，但 `self._worker` 在 `:276` 初始化为 `None` 后再无赋值，`_invoke`（:343-346）直接调原生对象。`setup` 里的 `worker_scope` 是传给 Rust 侧的参数，不是这个 Python 线程。**不要把注释里的模型当成现状**。

13. **cpp 与 rs 后端的错误语义不可互换**。rs 的 `_CACHE_COMPAT_POLICY` 会把 `put/get/register_buffer` 等缓存面异常降级成 `-1`/`b""`/`[b""]*n`（`rs/store.py:66-80`），cpp 后端直接抛/返回原始错误码。依赖“失败会抛异常”的代码切到 rs 后会静默拿到失败码。

14. **`__init__.py` 的归属取决于构建路径**。`mooncake-wheel/mooncake/__init__.py` 只有 `extend_path`（构建期占位）；`build_wheel.sh:236-239` 会用 `python/mooncake/__init__.py` 覆盖它并在退出时还原；`python/CMakeLists.txt:18-24` 安装 legacy 目录时显式 `PATTERN "__init__.py" EXCLUDE`。所以“这个文件到底是什么”必须先问走哪条路径。

15. **跨目录源码依赖是双向的**。`python/CMakeLists.txt:18-24` 从 `mooncake-wheel/mooncake/` 装 Python 文件；`mooncake-integration/CMakeLists.txt:245-246, 259, 315` 又从 `mooncake-wheel/mooncake/` 装 `mooncake_connector_v1.py`、`vllm_v1_proxy_server.py`、`mooncake_store_service.py`、`pg.py`。反过来 `structured_object_store.py` 依赖只存在于 wheel 树的 `_partial_read.py`。没有任何一棵树是自洽的包。

16. **发布矩阵靠 sed 维持，两个数组必须单行**。`mooncake-wheel/pyproject.toml:34-36` 的 `keywords` / `classifiers` 被 8 个变体的 `sed -i 's/^keywords = \[\(.*\)\]$/.../'` 原地改写（`build_wheel.sh:289, 299, 308, 320, 329, 338, 348, 360`——最后一个是 ROCm/HIP 变体）。格式化换行会**静默**让变体构建不再改包名/关键词，产物仍能构建但元数据错误。文件顶部注释（:7-11）就是在警告这一点。

17. **`libmpcomm.so` 故意不 vendor**（`build_wheel.sh:478-486`）：MPComm 独立升级，且它可能带 CUDA fatbin，patchelf 会破坏它。vendor 会有两个后果：RPATH 覆盖 `LD_LIBRARY_PATH` 导致新版本被旧副本遮蔽；以及 fatbin 损坏。

18. **wheel tag 来自构建机 glibc**。`build_wheel.sh:459` 用 `PLATFORM_TAG=${PLATFORM_TAG:-"manylinux_${GLIBC_VERSION}_${ARCH_SUFFIX}"}`，glibc 由 `detect_glibc_version` 探测（:416-436，最终 fallback `2_17`）。`setup.py:28-105` 的 `_detect_manylinux_tag` 是另一套实现。`mooncake-wheel/tests/test_release_wheel_tags.py` 之所以断言 CI 必须跑固定镜像，就是因为 aarch64 的 floor 曾在无代码变更时漂移。

19. **测试环境的 import 解析与真实安装不同**。`mooncake-wheel/tests/conftest.py:11-29` 手工往 `sys.path` 和已导入 `mooncake.__path__` 里塞 `python/` 与 `mooncake-wheel/mooncake/`。在这组测试里能 import 到的东西，装 wheel 后不一定能。`mooncake-integration/tests/*.py` 则直接 `import engine`（`test_tent_send_probe.py:3`），依赖 `PYTHONPATH=$<TARGET_FILE_DIR:engine>`（`CMakeLists.txt:94`），只在 `BUILD_UNIT_TESTS AND USE_TENT` 时注册（:88）。

20. **`BufferPool` 的析构与解释器关闭**。`BufferLeaseNative::~BufferLeaseNative` 吞异常调 `release_lease(false)`（`buffer_pool.cpp:217-222`），后者可能走到 `unregister_overflow_unlocked` → `py::gil_scoped_acquire`（:461-464）。如果发生在解释器 finalize 阶段，acquire GIL 的行为本身就是未定义的灰区。**此处未完全确认** CPython 3.10-3.13 各版本的表现，但设计上应避免把 `BufferLease` 留到进程退出。

21. **`close()` 的返回值语义**：`MooncakeStorePyWrapper::close`（`store_py.cpp:2694-2704`）无论 `tearDownAll` 成功与否都会 `reset` 内部指针，文档字符串写明“nonzero result is terminal, not retryable”。

22. **快照对象绑定到特定 Store**：`RangedReadSnapshotPy::belongs_to`（`store_py.cpp:506-508`）比较裸指针，`get_into_ranges_from_snapshot` 在不匹配时抛 `invalid_argument`（:3096-3099）。跨 Store 复用快照会直接失败，这是有意的。

## 精读清单

建议按以下顺序读，每步都有明确的“读懂标志”：

1. **`mooncake-integration/integration_utils.h` 全文（401 行）**。
   先读 `TensorDtype`（:25-43）与 `array_creators`（:61-77），再读 `TensorObjectHeader`/`TensorLayoutMetadata`/`TensorMetadata`（:189-232），最后逐条过 `ValidateTensorMetadata`（:308-381）。
   *读懂标志*：能说出“为什么 `data_bytes` 必须等于 `numel*element_size`”“为什么 `MakeTensorShape` 用 -1 填充剩余槽位”“新增 dtype 只能追加在哪里”。

2. **`store_py.cpp:1-540`**。
   `PyTensorInfo`（:42-84）→ `RawTensorObjectParts`/`make_raw_tensor_object_parts`（:92-149）→ `extract_tensor_info_impl`（:160-211）→ `buffer_to_tensor`（:233-368）→ `RangedReadSnapshotPy`（:498-530）→ 注意 :494 的 `#include "store_py_internal.h"`。
   *读懂标志*：能画出 `get_tensor()` 的一次完整拷贝路径，并指出 `owner` 在哪一行被赋值、为什么。

3. **`store_py_internal.h:1-140` + `:1345-1504`**。
   只看类型与三个关键函数：`calculate_shard_range`（:1345）、`parse_tensor_metadata_from_raw_buffer`（:1402）、`build_raw_tensor_shard_write_plan`（:1417）。
   *读懂标志*：能解释 `data_ranges` 里每个 `(src_offset, row_bytes)` 是怎么从 `(global_shape, split_dim, shard_start)` 算出来的。

4. **`store_py.cpp:533-1103`（类体前半）**。
   重点 `init_real_client`（:542）、`setup` 的两个重载（:2437 / :2484，可以跳读到绑定处）、`get`/`get_batch`（:656/:692）、`get_tensor_into`（:795）、`batch_get_tensor_metadata_prefixes`（:875）、`batch_get_tensor_into_cuda`（:954）。
   *读懂标志*：能说清“为什么元数据前缀要单独读”“为什么必须复用同一个 snapshot”。

5. **`store_py.cpp:1154-1672` + `store_py_parallel_write.h` 全文**。
   `put_tensor_impl` → `batch_put_tensor_infos_impl` → `try_dummy_cuda_ipc_batch_put_tensor_impl`（:17-159）→ `batch_write_tensor_impl`（:208-334）。
   *读懂标志*：能对比 RealClient（两个 buffer、零拷贝）与 DummyClient（staged、一次拷贝）的差异，并指出 GIL 在哪一行释放、在哪里重新获取。

6. **`store_py.cpp:2137-3541`（模块绑定）**。
   先扫 `PYBIND11_MODULE` 里注册的类型与函数名（:2137-3538），再回头看 `MooncakeDistributedStore` 的 `.def` 列表，把方法分成四类：缓冲/注册、裸字节、张量、控制面。
   *读懂标志*：拿到 Python 侧 TE/Store 的完整契约；知道 `BufferPool` 的别名（`RegisteredBufferPool` 等，`buffer_pool.cpp:586-588`）。

7. **`mooncake-integration/store/buffer_pool.cpp:315-534`**。
   `acquire`（:315）→ `try_acquire_locked`（:390）→ `allocate_overflow_unlocked`（:440）→ `release_internal`（:470）→ `has_capacity_for_locked`（:513）。
   *读懂标志*：能指出“持锁期间从不调用 Python”这一不变量在哪些行被打破（以及为什么那里要 `lock.unlock()`）。

8. **`mooncake-integration/transfer_engine/transfer_engine_py.cpp:63-306` 与 `:505-585`**。
   `initMemoryAllocator`（:229）→ `allocateRawBuffer`（:512）→ `findClassId`/`doBuddyAllocate`（:527/:534）→ `allocateManagedBuffer`/`freeManagedBuffer`（:556/:572）。
   *读懂标志*：能解释 8KB…256MB 的 class 表、2GiB 顶层块、以及“buddy 块 free 不归还内存（超过最大 class 的大块例外）”。

9. **`transfer_engine_py.cpp:962-1273` 与 `:1743-2032`**。
   `transferSync`（:962）→ `batchTransferSync`（:1075）→ `batchTransferAsync`（:1207）→ 模块绑定（:1743）。
   *读懂标志*：能说出重试次数、超时公式、失败时 segment 缓存如何清理、`batch_id_t` 为什么用 0 表示失败。

10. **`mooncake-integration/shared_segment_py.cpp` + `shared_segment.py`**。
    两阶段协议、布局排序、名字编码指纹、all-gather、视图构造。
    *读懂标志*：能解释“为什么虚拟地址不要求跨 rank 一致”以及 `host_register`/`mmap` 四种组合分别得到什么张量类型。

11. **`python/mooncake/store/__init__.py` 全文 + `python/mooncake/store/rs/store.py:1-80, 273-460`**。
    白名单差异、懒转发、rs 降级策略、`_invoke` 的真实调用路径。
    *读懂标志*：能列出 cpp 有而 rs 没有的 API，并知道 rs 模式下它们抛什么。

12. **打包三件套**：`mooncake-integration/CMakeLists.txt`（尤其 :1-19、:46-57、:148-179、:236-334）、`python/CMakeLists.txt`、`scripts/build_wheel.sh:180-280, 438-661`。
    最后看 `python/tests/packaging/test_project.py`——它把“哪棵树是唯一权威源”“哪些模块被迁移”“CMake 必须怎么写”固化成可执行断言，是打包不变量的最佳文档。
    *读懂标志*：拿到一张“某个 `.py`/`.so` 最终会出现在 wheel 的哪个路径、由哪条规则安装”的映射表。

---

# 14 Expert Parallelism 与 PyTorch Backend

> 本章覆盖两个源码目录与两个设计文档：`mooncake-ep/`（MoE 专家并行的 dispatch/combine 运行时）、`mooncake-pg/`（基于 Mooncake Transfer Engine 的进程组/集合通信库），以及它们在 `python/mooncake/`、`mooncake-wheel/mooncake/` 下的 Python 门面。设计侧对应 `docs/source/design/mooncake-ep.md` 与 `docs/source/design/mooncake-backend-pg.md`。
>
> 行号基准是仓库根目录 `D:/xiaojun/work/Mooncake` 当前 `main`（`67df8c41`）。路径一律写成仓库相对路径。
>
> 本章开头先做三处**事实修正**，它们都来自实际读码而非粗读摘要：
> 1. `SWITCH_RANKS` / `SWITCH_RDMA_RANKS` / `SWITCH_RANKS_WITH_DTYPE` 这三个宏在 `mooncake-ep/include/mooncake_ep_launch.cuh:8-53` 有定义，但**全仓库没有任何使用者**（`grep` 只命中定义）。legacy 内核并不按 `num_ranks∈{2,4,8}` 做模板特化；它特化的是 `hidden`、warp-group 数与 `use_fp8`（`mooncake-ep/src/mooncake_ep_kernel.cu:476-511`），`num_ranks` 是纯运行期参数。
> 2. `ElasticBuffer` 的**编译期形状集合比 legacy 还窄**：所有实例都固定在 `hidden=4096, num_experts=256, num_topk=8, num_max_tokens_per_rank=128, num_sms=24`；非 hybrid 允许 `num_scaleup_ranks∈{2,8}`（`mooncake-ep/src/mooncake_ep_elastic_kernel.cu:700-728`、`:1023-1044`），hybrid 只允许 `(scaleout,scaleup)∈{(2,4),(2,8),(4,4)}`（`:633-637`）。不匹配直接 `unsupported_elastic_config()`。
> 3. `docs/source/design/mooncake-ep.md:163-167` 描述的 zero-copy combine（`get_next_combine_buffer`）在 legacy Python 门面里是 **`NotImplementedError`**（`python/mooncake/mooncake_ep_buffer.py:568-569`、`:705-706`）。文档描述的是原生能力/规划，不是当前 Python 可用路径。

---

## 定位与职责

这两个目录合起来是 Mooncake 的**推理侧通信栈**，但它俩解决的是两个不同层面的问题：

| | `mooncake-pg` | `mooncake-ep` |
|---|---|---|
| 抽象层级 | 通用 ProcessGroup：collective + P2P | MoE 专用算子：dispatch / combine |
| 对上的接口 | `torch.distributed`（`c10d::ProcessGroup` 后端，注册名 `mooncake` / `mooncake-cpu`） | DeepEP low-latency 风格的 Python API（`mooncake.mooncake_ep_buffer.Buffer`、`mooncake.mooncake_elastic_buffer.ElasticBuffer`） |
| 数据面 | 主机 worker 线程经 `TransferEngine::submitTransfer` 直写对端注册缓冲；P2P 走信用制代理 | GPU 内核直接从显存向对端发起 NVLink P2P 写或 IBGDA RDMA 写，**不经 host、不经集合通信库** |
| 控制面 | 自带 Coordinator/Agent（动态成员、故障协调、视图 2PC） | 无控制面；借外部 process group 做 bootstrap，并从 PG 读 `active_ranks` |
| 容错语义 | 成员可增删、故障可协调、可恢复 | 内核超时可把 `active_ranks[src]=0` 并跳过该源；成员恢复要靠上层重连 |

**为什么需要 EP**：MoE 推理里每个 token 按 top-k 路由到全局专家，专家的 host rank 与 token 的 host rank 通常不同。dispatch 要按目标专家把 token 搬到专家所在 rank，combine 要按路由权重把专家输出搬回并按权重求和。这两步是纯"小消息 + 高频 + 强时延敏感"的 all-to-all 变体，用 NCCL 的通用集合通信会因为建链/同步/规约的通用开销而吃亏。Mooncake EP 的做法是把整个协议编进一个 cooperative kernel：发送方直接对远端显存做 put，再用信号缓冲做计数与完成通知。

**为什么需要 PG**：NCCL 的进程组是固定成员的，成员变化要重建组。Mooncake PG 把"组"建模成**固定座位的餐桌**（`docs/source/design/mooncake-backend-pg.md:13-32`）：`max_group_size` 是座位数，`active_ranks` 是当前有人坐的座位；成员掉线只清 active 位，不重排 rank，恢复后回原座位。这套语义 NCCL 没有。

**两者的耦合点只在 Python 层**：`mooncake-ep/` 的 C++ 代码对 `mooncake-pg/` **零引用**（在 `mooncake-ep/` 下 grep 不到任何 `mooncake_pg` / `mooncake-pg` 符号；用点号写法 `mooncake.pg` 只会在 `mooncake-ep/benchmarks/` 的两个 Python 脚本里命中，它们同样是在 Python 层 import PG 做 bootstrap）。EP 的 `Buffer.connect()` 接受任意 `dist.ProcessGroup`，用 `all_gather` / `all_to_all` 交换 RDMA/IPC 元数据；只有 `_active_ranks_tensor()` 会去 `from mooncake.pg import get_active_ranks`（`python/mooncake/mooncake_ep_buffer.py:343-348`），且组不是 mooncake 后端时只警告并按全 active 处理（`:325-341`）。

整体位置（自上而下）：

```
推理框架 (SGLang / vLLM / 自研 MoE runtime)
   │  dispatch/combine                       │  all_reduce/all_gather/P2P
   ▼                                         ▼
mooncake-ep (Buffer / ElasticBuffer)      mooncake-pg (MooncakeBackend = c10d::ProcessGroup)
   │  GPU kernel: mc_route_put/mc_rdma_put  │  host worker / P2PProxy
   ▼                                         ▼
mooncake-transfer-engine/transport/device  ←→  TE::submitTransfer / 段注册 / Topology
   (P2pTransport=NVLink+IPC, RdmaTransport=IBGDA)
```

---

## 目录结构速览

### EP 原生层

| 文件 | 一句话职责 |
|---|---|
| `mooncake-ep/include/mooncake_ep_buffer.h` | legacy `MooncakeEpBuffer` 声明：`BufferPair` 双缓冲布局、传输所有权、元数据交换接口 |
| `mooncake-ep/src/mooncake_ep_buffer.cpp` | legacy host 中枢：传输初始化/降级、dispatch/combine 校验与流同步、`BufferPair` 翻转、QPN/IPC 交换 |
| `mooncake-ep/src/mooncake_ep_kernel.cu` | legacy dispatch/combine CUDA 内核与模板启动器（按 hidden 特化；dispatch 另按 warp-group 数与 FP8 特化） |
| `mooncake-ep/include/mooncake_ep_configs.cuh` | `MAX_QP_COUNT=256`、`NUM_WORKSPACE_BYTES=32MiB`、`FINISHED_SUM_TAG=1024`、phase 位 |
| `mooncake-ep/include/mooncake_ep_device.h` | 平台分支的启动方式：CUDA 用 cooperative `cudaLaunchKernelEx`，MUSA/MACA 用普通 `<<<>>>` |
| `mooncake-ep/include/elastic/mooncake_ep_elastic_buffer.h` | `MooncakeElasticBuffer`：`ElasticConfig`/`ElasticTopology`、大参数表 dispatch/combine、`reconfigure_nccl` |
| `mooncake-ep/src/mooncake_ep_elastic_buffer.cpp` | Elastic host 实现：`NcclElasticState`、`make_launch_context` 选择 kIbgda/kNccl、拓扑发现、尺寸计算 |
| `mooncake-ep/include/elastic/mooncake_ep_elastic_transport.cuh` | **EP 与底层传输关系的核心**：`IbgdaOps` / `NcclOps` 把 DeepEP 的 GIN 面映射到 Mooncake Device API / NCCL |
| `mooncake-ep/include/elastic/*_official.cuh` | 从 DeepEP official elastic 源码移植的 dispatch/combine/hybrid 内核（文件头注释注明来源） |
| `mooncake-ep/src/mooncake_ep_elastic_kernel.cu` | Elastic 内核启动器：按 backend 选择 `Ops`，按编译期形状选实例 |
| `mooncake-ep/src/ep_py.cpp` | pybind11 模块 `_ep`：`Buffer`、`ElasticBuffer`、`EventHandle`、`create_nccl_unique_id` 等 |

### PG 原生层

| 文件 | 一句话职责 |
|---|---|
| `mooncake-pg/include/mooncake_pg.h` | 实验性 C API：context/comm/completion 句柄、所有 CPU/GPU 集合操作与 send/recv、成员管理入口 |
| `mooncake-pg/include/mooncake_communicator.h` | `MooncakePGContext`（进程级单例）与 `MooncakeCommunicator`（每组建一个）、超时常量 |
| `mooncake-pg/src/mooncake_communicator.cpp` | 组 bootstrap、collective 路由（Legacy / v2）、P2P 入队、`checkOpState`、`applyGroupState`、`joinGroup` |
| `mooncake-pg/src/mooncake_worker_thread.cpp` | legacy collective 执行引擎：Task 状态机、`submitTransfer` 直写、数据同步区、失败证据 |
| `mooncake-pg/src/mooncake_worker_host.cpp` | `putTaskCpu/putTaskCuda`：切 chunk、双缓冲轮换、常驻 enqueue 内核、等待提交完成 |
| `mooncake-pg/include/mooncake_worker.cuh` | `TransferGroupMeta`、`CollectiveExtensionState`、`WorkCompletion` |
| `mooncake-pg/include/mooncake_worker_kernels.cuh` | host/设备共享的 `Task` 结构与常驻内核 |
| `mooncake-pg/src/p2p_proxy.cpp` | 信用制接收方驱动 P2P：分块状态机、Credit/Ack 环、超时与失败上报 |
| `mooncake-pg/include/p2p_proxy.h` | P2P 内存布局、`CreditSlot`/`AckSlot`、池与 lane 状态（文件头有完整协议图） |
| `mooncake-pg/src/control_plane/coordinator.cpp` | 成员/健康状态机：健康集推导、reconciliation 窗口、激活准入、ViewUpdate 2PC |
| `mooncake-pg/include/control_plane/control_types.h` | `RankState`/`GroupMemberState`/`GroupView`/`GroupEndpointInfo`/`LinkEvent` |
| `mooncake-pg/include/device_comm/**` | collective v2：设备发起的 `DeviceCollectiveRuntime` + `RingAllReduceAlgorithm` + 路由层 |
| `mooncake-pg/torch/src/pg_py.cpp` | 后端注册（`__attribute__((constructor))`）、`initControlPlane`、Python 绑定 |
| `mooncake-pg/torch/src/mooncake_backend.cpp` | `MooncakeBackend`（继承 `c10d::ProcessGroup`）、`MooncakeBackendShim`、collective/P2P 模板 |
| `mooncake-pg/torch/include/work_handles.h` | `FailedRanksHint`、`MooncakeWorkCpu/Cuda/P2PWork`、`MooncakeWorkTracker` |

### Python 门面（两个 C++ 目录之外）

| 文件 | 一句话职责 |
|---|---|
| `python/mooncake/mooncake_ep_buffer.py` | legacy `Buffer`：connect 元数据交换、fallback 判定、EventOverlap、纯 torch 回退 |
| `python/mooncake/mooncake_elastic_buffer.py` | `ElasticBuffer`：transport 共识协商、`EPHandle`、NCCL unique id 广播、`update_ep_member` |
| `python/mooncake/ep.py` | 只是把 `mooncake._ep` 的符号 re-export |
| `mooncake-wheel/mooncake/pg.py` | `mooncake.pg` 的运行时 JIT 加载器（首次 import 编译 Torch 适配器并缓存） |
| `tests` | `mooncake-pg/tests/`（PG 契约）与 `python/tests/ep/`（EP 正确性/恢复） |

---

## 核心数据结构

### 1. `BufferLayout` / `BufferPair`（legacy EP 的双缓冲布局）

`mooncake-ep/include/mooncake_ep_buffer.h:24-65`。一个 `BufferPair` 描述**两套**逻辑缓冲，每套含四个区：`rdma_send_signal_buffer`、`rdma_recv_signal_buffer`、`rdma_send_data_buffer`、`rdma_recv_data_buffer`。

构造公式（`:42-64`）：

```
signaling_buffer_bytes = num_experts * sizeof(int)
send_recv_buffer_bytes = num_experts * num_max_dispatch_tokens_per_rank
                         * (2 * sizeof(int4) + hidden * 2)
pair i 的基址 = 2*i*signaling + 2*i*data
total_bytes   = 4*signaling + 4*data        // 恰好两套
```

值得注意的两点：

- 公式里**没有用到 `num_ranks`**，尽管构造函数签名有它。接收数据区的大小只按 `num_experts × 每源 rank 最大 token 数` 算；每个专家的数据区内部再按 `num_ranks * num_max_dispatch_tokens_per_rank` 细分（见内核 `mooncake-ep/src/mooncake_ep_kernel.cu:186-189`）。也就是说 `num_ranks` 的维度被折叠进了 `num_experts`（因为 `num_experts % num_ranks == 0` 且每个 rank 的专家数 = `num_experts/num_ranks`，每专家每 rank 的容量 = `max_tokens`）。
- `get_ep_buffer_size_hint()`（`mooncake-ep/include/mooncake_ep_buffer.h:204-210`）只是构造一个假 `BufferPair(nullptr, ...)` 读 `total_bytes`，不分配。Python 侧 `Buffer.get_ep_buffer_size_hint` 直接转发到 `_ep`（`python/mooncake/mooncake_ep_buffer.py:353-364`）。

### 2. `MooncakeEpBuffer`（legacy 原生 Buffer）

声明在 `mooncake-ep/include/mooncake_ep_buffer.h:67-202`，实现从 `mooncake-ep/src/mooncake_ep_buffer.cpp:123` 开始。关键字段：

| 字段 | 含义 / 设计理由 |
|---|---|
| `gdr_buffer` | 由 `p2p_transport_->allocateBuffer()` 分配的 GDR 缓冲；**P2P transport 拥有分配权**，即使只用 RDMA 也走它（`:159-172`） |
| `p2p_transport_` / `rdma_transport_` | 指向传输对象。`P2pTransport` 管 NVLink+IPC，`RdmaTransport` 管 IBGDA QP/MR |
| `owned_p2p_transport_` / `owned_rdma_transport_` | EP 自己创建时持有所有权；外部传入 `TransferEngine*` 时由 engine 拥有，这两个为空（`:161-166`、`:175-210`） |
| `buffer_idx` | 双缓冲下标，dispatch/combine 里用 `buffer_idx ^= 1` 翻转（`:280`、`:443`） |
| `dispatch_shadow_slots` + `next_shadow_slot` | 见下 |
| `ibgda_disabled_` / `p2p_enabled_` | 快速路径判定；`use_fast_path()` 只在 IBGDA 不可用时才看 P2P 全可达（`:149-158`） |
| `USE_QP_COUNT` | `MAX_QP_COUNT / num_ranks * num_ranks`，即 `floor(256/n)*n`，保证能被 `num_ranks` 整除（`:131`） |
| `active_qps_cap_` | RoCE 下每对端的活跃 QP 上限，可由 `MOONCAKE_EP_ACTIVE_QPS_PER_RANK` 覆盖（`:134-152`） |

`dispatch_shadow_slots` 是个容易被忽略但很关键的设计：注释说“TBO 拥有多个逻辑 dispatcher 但共享一个 native Buffer”（`mooncake-ep/include/mooncake_ep_buffer.h:78-82`）。dispatch 会把 `topk_idx` 复制到 `workspace` 里的一块 shadow 区（每个 dispatch 元数据指针分配一个槽），combine 再用 `src_info_ptr` 找回同一个槽（`mooncake-ep/src/mooncake_ep_buffer.cpp:282-289`、`:435-440`）。原因：combine 需要原始 `topk_idx` 来计算路由权重，但调用方在 dispatch 与 combine 之间可能已经释放/复用了那块 tensor；同时两个逻辑 dispatcher 交错调用时不能互相覆盖。槽位常量是 `kTopkShadowMaxTokens=1024 / kTopkShadowMaxTopk=32 / kTopkShadowSlots=8`（`:63-67`），超出 topk 上限会被 host assert 拒绝（`:323`）。

生命周期上有两个不对称：

- **析构**（`:217-250`）：engine 拥有 RDMA transport 时要手动 `unregisterMemory`；自己拥有时交给 `RdmaTransport` 析构。P2P 缓冲则总是 `freeBuffer` 后置空。
- **失败降级**（`:94-113`、`:174-210`）：`initRdmaTransport` 是“initialize → registerMemory → allocateControlBuffer → createQueuePairs”四步，任一步失败会回滚 MR 并返回 false，调用方把 `ibgda_disabled_=true`，随后 `use_fast_path()` 退回“P2P 全可达才快”的判定。

### 3. `EventHandle` / `EventOverlap`

`EventHandle`（`mooncake-ep/include/mooncake_ep_event.h:8-38`）本质是“在某个 stream 上 record 一个 disable-timing event + 可选的 keepalive 共享指针”。`EventHandle(uint64_t stream_ptr, keepalive)` 构造即 `cudaEventRecord`；`current_stream_wait()` 是对目标 stream 的 `cudaStreamWaitEvent`。

`EventOverlap`（`python/mooncake/mooncake_ep_buffer.py:37-93`）在 Python 侧包一层，额外持有 `extra_tensors` 来模拟 `record_stream`——因为 CUDA Graph 下不能调用 `record_stream`，所以用“把 tensor 挂在事件对象上延迟释放”的办法保证生命周期（`async_finish=True` 或 `return_recv_hook=True` 时才挂，见 `:544-552`、`:697-703`）。

### 4. `ElasticConfig` / `ElasticTopology`（ElasticBuffer）

`mooncake-ep/include/elastic/mooncake_ep_elastic_buffer.h:21-47`。

`ElasticTopology` 是“物理域 × 逻辑域”的双层描述：

- 物理域：`num_rdma_ranks`（跨节点域大小）× `num_nvlink_ranks`（节点内域大小）。
- 逻辑域：`num_scaleout_ranks` × `num_scaleup_ranks`，以及 `scaleout_rank_idx` / `scaleup_rank_idx`。
- `hybrid_enabled` 表示是否同时用两条链路（节点内 NVLink + 节点间 RDMA）；`scaleup_lsa` 表示 scale-up 域是否真是 LSA（NVLink 全互通）。

`ElasticConfig` 是用户参数快照（max tokens、hidden、topk、fp8、deterministic、hybrid、multiple reduction、overlap、`sl_idx`（RDMA service level）、QP 数、CPU/GPU 超时秒数）。注意 host 实现里 `allow_multiple_reduction=false` 直接抛错（`mooncake-ep/src/mooncake_ep_elastic_buffer.cpp:447-451`）。

### 5. `ElasticLaunchContext`

`mooncake-ep/include/elastic/mooncake_ep_elastic_launch.cuh:32-58`。这是 host 传给内核启动器的**统一参数包**：backend 枚举（`kIbgda`/`kNccl`）、device、NCCL context、GDR 基址、NVLink 可达表、IPC peer 指针表、raddr/rkey/qp_devctx、信号缓冲、数据区、workspace、mapped host workspace、rank/team 元数据、QP 数、超时周期。

它把两种 backend 的差异压成“同一套字段 + 一个 backend 枚举”；内核侧再用 `Ops::kIsNccl` 做编译期分叉。

### 6. `NcclElasticState`

`mooncake-ep/src/mooncake_ep_elastic_buffer.cpp:155-284`。持有 `device::NcclTransport`、`NcclBufferRegistration`、`NcclDeviceContext`、`NcclTransportProperties`、`NcclLsaTopology`、对称内存分配、GIN context 数、comm stream。

构造流程（`:169-218`）值得逐行看：先 `createNcclDeviceTransport()`，再填 `NcclTransportConfig`（`enable_gin = num_ranks > 1`、GIN context 数、queue depth 1024、signal count `num_ranks+4`、traffic class = `sl_idx`），然后 `initialize(unique_id)`、`allocateAndRegisterBuffer()`、`deviceContext()`，最后用 `allRanksSucceeded(device_context.valid())` 做一次**跨 rank 状态集合**——任何 rank 失败则全组抛错。

`release()` 与 `abort()` 的区别很重要（`:220-276`）：`release()` 走正常的 deregister → free → shutdown；`abort()` 只在 `reconfigure_nccl` 替换旧代时用，因为它假设旧 communicator 可能已经没有完整的对端（`:650-656`）。

### 7. `IbgdaOps` / `NcclOps` 与 team 标签

`mooncake-ep/include/elastic/mooncake_ep_elastic_transport.cuh` 是**本章最该先读的文件**。文件头（`:24-38`）直接给了映射表：

```
get_sym_ptr  -> mc_route_put，返回 local/P2P 对端 VA 或 nullptr
put          -> 本地/P2P warp copy，否则 mc_rdma_put
put_value    -> 本地/P2P release store，否则 mc_rdma_put/mc_signal
flush        -> no-op（Device API 的操作由 release/fence 与显式 barrier 排序）
```

三个空类型 `WorldTeam` / `ScaleupTeam` / `ScaleoutTeam`（`:17-19`）只是模板标签，用来让 DeepEP 官方模板代码保持原样，真正的路由由 `world_rank<team_t>()`（`:82-92`）换算成全局 rank。换算规则：

- `ScaleoutTeam`：`dst_rank * num_scaleup_ranks + scaleup_rank_idx`（跨节点，同一 scale-up 位置）。
- `ScaleupTeam`：`scaleout_rank_idx * num_scaleup_ranks + dst_rank`（节点内）。
- `WorldTeam`：原样。

`IbgdaOps` 的关键实现细节：

- `put()`（`:116-159`）：`mc_route_put` 非空就走本地/P2P 拷贝，按 16 字节对齐选择 `int4` 批量或逐字节；然后 `__threadfence_system()`。注释明确说**不能在传输原语里放整 warp barrier**，因为 notify lane 是分支执行的，会死锁（`:149-153`）。
- `put_value()`（`:161-214`）：4 字节走 `mc_st_release` / `mc_signal`；8 字节在远端下降级为**两次 32 位 RED add**，并区分“低字最后发布”还是“高字最后发布”（`kRedAddReleaseLowWordLast`，`:21-22`）。原因写在注释里：IBGDA WQE 的源地址必须是注册过的 GDR 地址，线程局部标量不能做 DMA 源（`:184-188`）。
- `red_add_rel()`（`:216-283`）：4 字节同 `put_value`；8 字节在本地/P2P 用 `ptx::red_add_rel_sys` 做**单条 64 位系统域原子加**，在远端拆两次 32 位并“把 ready 字最后发”。这段注释解释了两种协议（高字 ready vs 低字 finish）为什么不能随便拆（`:236-259`）。
- `flush()`（`:312`）：`__threadfence_system()`。

`NcclOps`（`:320-556`）是同一编译期接口的 NCCL 实现：

- `kNumQPs = 65`（hybrid 默认 1 notify + 64 data，对应 `nccl_gin_context_count()` 的默认值，见 `mooncake-ep/src/mooncake_ep_elastic_buffer.cpp:137-151`）。
- `put()` 按 team 分派：`ScaleupTeam` 直接用 `lsaPeerPointer` 本地拷贝；`ScaleoutTeam` 用 `gin.put<kRail>`；`WorldTeam` 用 `gin.put<kWorld>`（`:393-426`）。
- `red_add_rel()` 对 64 位值用 `gin.signalAdd<team>`；对本地/P2P 走 `ptx::red_add_rel_gpu` 或 `red_add_rel_sys`（`:450-488`）。
- `flush()` 要遍历**所有** GIN context（`flushContextWarp`），因为 NCCL 的 flush 是 per-context 的（`:546-555`）。

### 8. `EPHandle`

`python/mooncake/mooncake_elastic_buffer.py:270-320`。字段名刻意对齐 DeepEP official elastic handle：`do_expand`、`num_experts`、`expert_alignment`、`num_max_tokens_per_rank`、`num_sms`、`topk_idx`、`psum_num_recv_tokens_per_scaleup_rank`、`psum_num_recv_tokens_per_expert`、`recv_src_metadata`、`dst_buffer_slot_idx`、`token_metadata_at_forward`、`channel_linked_list`、`native_handle`、`_generation`。

两个语义点：

- `num_recv_tokens = recv_src_metadata.shape[0]`（`:320`）——没有 CPU sync 时它是**推断上界**，CPU sync 后才是真实值（注释明说与 DeepEP 同约定）。
- `_generation` 只在 NCCL 模式下有值（`tuple(nccl_unique_id)`）；`_validate_handle_generation()` 拒绝旧代 handle（`:526-532`）。IBGDA 模式下 `_generation` 为 `None`，校验直接跳过。

### 9. `mooncakePgCommConfig_t`

`mooncake-pg/include/mooncake_pg.h:88-133`。这是 C ABI 的结构体，带 `structSize/magic/version` 三件套做版本校验，`MOONCAKE_PG_COMM_CONFIG_INITIALIZER` 给默认值。

字段分四组：

- 身份：`groupId`（PyTorch 的 bootstrap id）、`rank`、`size`、`maxGroupSize`、`globalRanks/globalRankCount`。
- 设备：`deviceType`、`deviceIndex`。
- 策略：`idResolvePolicy`（`CreateOrAttach` / `AttachOrExtend`）、`autoDeactivateOnFailure`、`autoSyncOnFailure`、`preferredGpuCollectiveBackend`。
- 镜像：`activeRanksMirror` + count + isDevice + deviceIndex（调用方拥有的 `active_ranks` tensor 存储）。

Torch 侧到 C 侧的翻译在 `mooncake-pg/torch/src/mooncake_backend.cpp:361-407`，其中最容易踩的是 `auto_sync_on_failure` 必须搭配 `auto_deactivate_on_failure`（`:399-402`，C++ communicator 里还有二次校验 `mooncake-pg/src/mooncake_communicator.cpp:502-504`）。

### 10. `MooncakePGContext`

`mooncake-pg/include/mooncake_communicator.h:48-107`。**进程级单例**（Python 侧用 `std::call_once` 保证，`mooncake-pg/torch/src/pg_py.cpp:35-43`）。持有：

- `owned_engine` / `engine`：默认自建 `TransferEngine(true)`，也可被外部 engine 替换（`setExternalEngine`）。
- `link_manager`：TE 的段/链路解析。
- `worker_manager` / `p2p_device_worker_manager`：按设备共享的 worker。
- `coordinator_host`（仅 global rank 0）/ `agent_host`（所有 rank）。
- collective v2 的 `device_transfer_service` / `device_collective_workspace` / `device_collective_strong_stream` / `device_collective_recovery_worker`（`#if MOONCAKE_PG_HAS_COLLECTIVE_V2`）。
- `comm_use_count_` + `state_mutex_`：引用计数，保证最后一个 communicator 销毁前 context 不析构。

`initialize()`（`mooncake-pg/src/mooncake_communicator.cpp:191-263`）的顺序约束写在注释里：**engine 和 LinkManager 必须先初始化，再 connectCoordinator 启动 AgentHost**，因为 `AgentHost::start()` 会立刻发 `registerAgent`，里面用到 `LinkManager::localServerName()` 与 `getWarmupRecvAddr()`（`:211-225`）。这段注释在 `pg_py.cpp:50-54` 又重复了一遍，说明它踩过坑。

### 11. `MooncakeCommunicator` + `TransferGroupMeta` + `CollectiveExtensionState`

`MooncakeCommunicator`（`mooncake-pg/include/mooncake_communicator.h:131-344`）每个 group 一个。它持有：

- `meta_`（`std::shared_ptr<TransferGroupMeta>`）：**host 与设备共享的组元数据**。
- `worker_`：legacy collective worker。
- `send_buffer_[2] / recv_buffer_[2] / cpu_sync_send_region_[2] / cpu_sync_recv_region_[2]`：双缓冲 + 每 rank 一个 int32 同步字。
- `p2p_proxy_` + `p2p_device_worker_`：P2P 数据面。
- `active_ranks_mirror_*`：调用方镜像。

`TransferGroupMeta`（`mooncake-pg/include/mooncake_worker.cuh:84-112`）是理解 PG 数据面的钥匙：

```cpp
InGroupRank rank; GlobalRank globalRank;
GlobalRank rank_order[kMaxNumRanks];   // InGroupRank -> GlobalRank
int maxGroupSize;                      // 固定槽位数
std::atomic<int> activeSize{0};        // 最高活跃 in-group rank + 1（不是人数！）
std::atomic<uint64_t> epoch{0};        // GroupView epoch
std::atomic<CollectiveExtensionState> extensionMode{Isolated};
bool* activeRanks; bool* activeRanksDevice;   // mapped pinned，内核/worker 共享
RankState rankStates[kMaxNumRanks];    // 按 GlobalRank 索引
uint64_t rankEpochs[kMaxNumRanks];
TransferMetadata::SegmentID segmentIDs[kMaxNumRanks];
GroupEndpointInfo segmentInfos[kMaxNumRanks];
MooncakeCommunicator* communicator;    // 回调用于上报 link event
```

`CollectiveExtensionState`（`:78-82`）是**本地执行阶段**，不是成员状态：

- `Isolated`：只做 {self} 的本地集合操作（用于新 rank 预热）。
- `Quiescing`：拒绝新集合操作，等激活。
- `Normal`：使用 Coordinator 提交的成员关系。

注意 `activeSize` 的注释：“Highest active InGroupRank plus one”。这与 `getSize()` 的实现完全一致（`mooncake-pg/src/mooncake_communicator.cpp:781-787`、`:1784-1794`）。

### 12. 控制面数据模型

`mooncake-pg/include/control_plane/control_types.h`：

- `RankState`（`:41-45`）：`Offline` / `Synced` / `Healthy`，描述**进程级**健康，与组无关。
- `GroupMemberState`（`:163-169`）：`None` / `Inactive` / `AwaitingActivation` / `Active` / `Left`，描述**组内**成员状态。状态迁移图见 `:145-162`。
- `GroupView`（`:212-225`）：`group_id`、`status`（Bootstrapping/BootstrapSyncing/Ready）、`epoch`、`auto_deactivate`、`gpu_collective_backend`、`max_group_size`、`rank_order`、`members`。`epoch` 是成员关系的单调版本号。
- `GroupEndpointInfo`（`:123-143`）：每 `(group, rank)` 的 buffer/sync/P2P 地址 + `device_collective` 子结构。
- `LinkEvent`（`:227-239`）：`events` 与 `target_rank_epochs` 并行，**用 rank epoch 防止迟到的旧进程事件被算到新进程头上**。

一个极易混淆的点：`rank_order` 是 **InGroupRank → GlobalRank** 的映射，不是“当前参与者列表”；参与者看 `members[i].isActive()` 或本地的 `activeRanks[]`。

### 13. `P2PProxy` 的结构与协议

`mooncake-pg/include/p2p_proxy.h`。文件头 `:24-163` 有完整的内存布局图与五步数据流，非常值得对照读。

- `CreditSlot`（`:202-223`）64 字节，布局 `[header_token][recv_addr][chunk_len][pad][footer_token]`。**header/footer 双 token 防伪**：生产者先写 payload 与 footer，最后 release 写 header；消费者读 header → payload → footer，只有 `header == footer != kInvalidControlToken` 才接受（`:185-201`）。
- `AckSlot`（`:230-248`）同样双 token，payload 只有 `chunk_len`。
- `ControlToken = uint64_t`，`(epoch << 32) | sequence`（`:176-183`）。
- `P2PChunkPool`（`:252-269`）：无锁，因为**只被一个 worker 线程访问**；`free_stack_` 是 `vector<void*>`。
- 发送侧状态机：`kCopyIn → kWriteRemote → kAck → kFinished`（`:340-349`）。
- 接收侧状态机：`kIssueCredit → kWaitAck → kCopyOut → kFinished`（`:351-358`）。
- 每对端：`SendPeerLane` / `RecvPeerLane`，含 pending 队列、active op、序号、`copy_ready_events_`（`:455-478`）。
- `P2PDeviceWorker`（`:601-657`）：**每设备共享**的两个轮询线程（send/recv）+ 两个固定 chunk 池；多个 communicator 注册到同一个 worker（`proxies_`）。

### 14. `Task`（legacy collective 的 host/设备共享结构）

`mooncake-pg/include/mooncake_worker_kernels.cuh`（被 `mooncake_worker.cuh:16` 引入）。worker 有 4 个槽位：2 个 CPU + 2 个 CUDA（`mooncake_worker.cuh:173-176`）。每个 `Task` 含 `opType`、`dataSize`、`broadcastRoot`、`bufferOffset`、`submitSequence`、`failedRanksHint`、`resetFailedRanksHint`、`transferGroupMeta`、`active`，以及设备侧的 `activeRanksMirrorRequestGeneration/AppliedGeneration`。

`submitSequence` 是 host 提交线程与常驻 enqueue 内核之间的接力协议。实际方向是：`enqueueTaskKernel` 把参数写进设备可见的 `Task` 槽（`tasks[taskId].submitSequence = submitSequence`），`__threadfence_system()` 后置 `active=true`，然后**自旋等 host 清 `active`**（`mooncake-pg/src/mooncake_worker.cu:19-58`）；host worker 线程看到 active 后在 `mooncake-pg/src/mooncake_worker_thread.cpp:159-160` / `:233-234` 把序号发布到 host 侧的 `submitted_task_sequence_[i]`；提交线程用 `waitUntilTasksSubmitted()`（`:66-89`）等这个 watermark，之后才在 enqueue 流上 record event_end。这样保证槽位不会被下一轮提前复用。

---

## 关键流程剖析

### 流程一：legacy EP 的 `connect()`——借 process group 完成元数据交换

入口：`python/mooncake/mooncake_ep_buffer.py:184-311`。`Buffer.__init__`（`:96-122`）先建原生 `ep.Buffer(rank, group_size, num_ep_buffer_bytes, disable_p2p)`，再把 `_use_fallback` 初始化成 `runtime.ibgda_disabled()`，然后调 `connect()`。

分步：

1. **RDMA MR 元数据（all_gather）**：`get_mr_info()` 返回 `(raddr, rkey)`，转成 CUDA tensor 后 `all_gather` 再 `cat().tolist()`（`:186-203`）。
2. **可选 QP 重建**：`is_update=True` 时先 `runtime.update_local_qpns()`（`:205-206`），即 `RdmaTransport::recreateQueuePairs`；失败会把 `ibgda_disabled_` 置真（`mooncake-ep/src/mooncake_ep_buffer.cpp:522-529`）。
3. **QPN 的 all_to_all**：本端 QPN 列表按 `all_to_all_size = len(local_qpns)/group_size` 切成每对端一段，`all_to_all` 后得到 `peer_qpns[r]`（`:208-222`）。
4. **LID 同法**（`:224-237`），再 **GID（subnet_prefix + interface_id）all_gather**（`:239-258`）。
5. **`sync_ibgda_peers(...)`**：把每 rank 的 QPN/LID 展平成 `USE_QP_COUNT` 长度，调 `RdmaTransport::connectPeers`；失败则 `ibgda_disabled_=true`（`mooncake-ep/src/mooncake_ep_buffer.cpp:531-563`）。
6. **IPC handle 交换**：`group_size==1` 直接跳过并强制 `_use_fallback=False`（`:271-275`）；否则 `get_ipc_handle()` → all_gather → `sync_nvlink_ipc_handles()`（`:276-300`）。异常只警告，不立即失败。
7. **重新评估 fallback**：`use_fast_path()` 再判一次（`:304-311`）。注释（`:114-117`）解释了为什么必须重估：`sync_nvlink_ipc_handles()` 可能把 C++ 侧的 `ibgda_disabled_` 从 True 翻回 False——`importPeerHandles()` 成功即意味着 P2P 全可达，此时 `use_fast_path()` 返回真。

`update_ep_member()` 就是 `connect(True)`（`:313-314`），即重建 QP + 重新交换。

这里有一个**集成契约**：`connect()` 全程使用 `dist.all_gather/all_to_all`，所以 process group 必须在 EP 构造前就健康；且这些是**定长 tensor**，所有 rank 必须用同样的 `group_size` 与 QP 数，否则 collective 形状不匹配。

还有一个按代码推演出来的可疑路径（本书未运行验证，建议实测）：若 IBGDA 不可用但 P2P 全可达，构造末尾的 `_use_fallback` 会被重估为 `False`（`:304-311`）；此后调用 `update_ep_member()` → `connect(True)` 会重新进入 RDMA 元数据交换分支，而此时 `get_local_qpns()` 返回空列表、`all_to_all_size = 0`，`torch.tensor([]).view(-1, 0)` 在 PyTorch 里通常会直接抛“无法推断 -1”的 reshape 错误。也就是说“IBGDA 关闭 + 纯 P2P”的部署上，首次 `connect()` 能过，但恢复路径未必能过。读到这里应把它当作需要验证的边界，而不是已保证的行为。

### 流程二：legacy EP 一次 `dispatch()` 的端到端

调用链：

```
Python Buffer.dispatch
  -> runtime.dispatch(...)                       [ep_py.cpp:57 -> mooncake_ep_buffer.cpp:252]
     -> 校验 + 选 launch_stream + 选 shadow slot
     -> launcher(SEND | RECV) / launcher(SEND) + hook -> launcher(RECV)
        -> mooncake::dispatch(...)                [mooncake_ep_kernel.cu:435]
           -> SWITCH_HIDDEN -> dispatch<fp8?, groups, 4, hidden><<<coop>>>
```

#### 2.1 Python 侧准备

`python/mooncake/mooncake_ep_buffer.py:367-552`：

- 断言 `x` 是 2D contiguous bf16、`topk_idx` 2D int64、`num_experts % group_size == 0`、`hidden % 128 == 0`。
- `active_ranks` 为 `None` 时用 `_active_ranks_tensor()`（优先 `mooncake.pg.get_active_ranks`，否则全 1）；显式传入但长度不等于 `group_size` 时则直接替换成全 1，**不会**再去问 PG（`:322-351`、`:393-402`）。
- fallback 模式走 `_fallback_dispatch`（见 2.5）；否则分配输出：
  - `packed_recv_x`：`[num_local_experts, group_size*max_tokens, hidden]`，fp8 或 bf16；
  - `packed_recv_count`：`[num_local_experts]` int32，**必须每次清零**。注释写了原因：rank 1 的本地专家计数器不会被发送阶段重置（`:464-471`）；
  - `packed_recv_src_info`：`[num_local_experts, group_size*max_tokens]` int32；
  - `packed_recv_layout_range`：`[num_local_experts, group_size]` int64（每项打包 `(begin<<32)|count`：高 32 位是起始偏移、低 32 位是该源 rank 的 token 数；与内核 `pack2<int,int64_t>(num_recv_tokens, recv_token_begin_idx)` 及回退路径 `(begin << 32) | count` 一致）；
  - fp8 时还有 `packed_recv_x_scales`，形状先按 `[E, hidden/128, ...]` 分配再 `transpose(1,2)`（`:486-496`）。
- `handle = (src_info, layout_range, max_tokens, hidden, num_experts)`，combine 原样传回（`:528-534`）。

#### 2.2 Host 侧：双缓冲、shadow、stream

`mooncake-ep/src/mooncake_ep_buffer.cpp:252-394`：

1. 断言链（`:261-265`）：`num_experts % num_ranks == 0`、`USE_QP_COUNT % num_ranks == 0`、`hidden % 16 == 0 && hidden % 128 == 0`、`num_tokens <= max_tokens`。
2. `graph_capture = stream_is_capturing(compute_stream)`（`:272`）。
3. 构造 `BufferPair layout`，`current_buffer_idx = buffer_idx`，`next_buffer = layout.buffers[buffer_idx ^= 1]`（`:275-280`）。**翻转发生在取 `next_buffer` 之前**：本次 SEND/RECV 用的都是 `buffer`（即 `current_buffer_idx` 那一套），`next_buffer` 只作为 `next_clean_buffer` 传进内核，由 SEND 阶段提前把它的计数器清零，供下一次调用使用（`:352`、内核 `:251-271`）。combine 的取用顺序完全相同（`:432-443`、`:478`）。
4. 分配 shadow slot（`buffer_mutex` 保护），把 `topk_idx` 拷进 `workspace`（`:281-329`）。
5. `launch_stream = (return_recv_hook || graph_capture) ? compute_stream : comm_stream`（`:294-295`）。`async && return_recv_hook` 被禁止（`:296`）。
6. 若非 hook 且非 capture：`stream_wait(comm_stream, compute_stream)`，让通信流等计算流（`:297-298`）。
7. 取设备侧指针：`raddrsPtr/rkeysPtr/qpDevCtxsPtr`（RDMA）、`availableTablePtr/peerPtrsTablePtr`（P2P），算 `active_qps_per_rank`（`:335-343`）。
8. 用 lambda 捕获全部参数，按模式 launch：
   - hook 且非 capture：只发 SEND；
   - capture：一次发 SEND|RECV；
   - 否则：发 SEND|RECV，或若编译期 `MOONCAKE_EP_SPLIT_SEND_RECV` 则分两次（`:357-368`）。
9. 同步返回：`async` 时返回 `EventHandle(launch_stream)`；非 async 且非 hook/capture 时 `stream_wait(compute_stream, launch_stream)`；hook 模式返回 `recv_hook` lambda（`:371-393`）。

`active_qps_per_rank_for_ep()`（`:13-34`）是平台相关的微调：MUSA 恒为 1；RoCE 下取 `min(qps_per_rank, cap)`，cap 默认 8（MACA 是 `max(8, num_local_experts)`）。注释解释了原因：CUDA RoCE 多卡共享 HCA，小消息摊到所有专家 QP 上会增加 doorbell/progress 开销。

#### 2.3 内核 SEND：按专家分工、计数、直写

`mooncake-ep/src/mooncake_ep_kernel.cu:44-433`。

拓扑映射：

```
warp_group_id = warp_id / kNumWarpsPerGroup (4)
responsible_expert_idx = sm_id * kNumWarpGroups + warp_group_id
num_sms = max(2, ceil(num_experts / kNumWarpGroups))
```

即一个 warp group 负责一个全局专家，一个 SM 负责 `kNumWarpGroups` 个专家。启动器为 3..8 个 warp group 各准备了一份模板实例（`:497-506`），但实际只有两条路径会取到非 8 的值：CUDA/MACA 固定 8，MUSA 用 `max(5, min(8, ceil(num_experts / SM 数)))` 计算（`:453-462`）；`hidden` 另由 `SWITCH_HIDDEN` 特化，dispatch 还按 `use_fp8` 二选一。combine 侧则把 warp group 数写死为 8（`:857-862`）。

发送阶段（`:122-237`）：

- **数据 warp**（除最后一个 warp 外）：`token_idx` 以 `sm_id` 为起点、`num_sms` 为步长遍历 token（`:129`），所以不同 SM 处理不同 token，不重不漏。
- 每个数据 warp 用 `warp_id < num_topk` 选自己的 top-k 槽（`:136`）；因此 `num_topk <= kNumMaxTopK=17` 且要 ≤ 数据 warp 数（`:464-469`）。
- `token_idx` 的源索引写进 RDMA 发送缓冲的首个 int（`:137`），随后做 FP8 分块量化（每 128 元素一组，`half_warp_reduce_max`，`:145-170`）或直接 bf16 copy。
- `mc_bar_sync(1, num_threads)` 在数据 warp 之间做同步（`:176`）。注意它**不是**整 CTA 的 `__syncthreads`：CUDA 上落到 PTX 命名 barrier `bar.sync 1, num_send_threads`，只有 `num_send_threads` 个线程参与（`cuda_ops.cuh:126-128`）；MUSA/MACA 上退化成 `__syncthreads()`（`musa_ops.cuh:116-118`、`maca_ops.cuh:68-71`），所以计数路径必须补等次数的 `__syncthreads()`（见下）。
- **槽位分配**：每个 warp 的 lane 0 对 `atomic_counter_per_expert[dst_expert_idx]` 做 `atomicAdd` 拿 `slot_idx`，再 `__shfl_sync` 广播（`:180-181`）。这样同一专家在多个 token 上的写入位置是唯一的。
- **目标地址**（`:185-189`）：`rdma_recv_data_buffer + local_expert*num_ranks*max_tokens*msg + rank*max_tokens*msg + slot*msg`——即“目标 rank 上、该本地专家、来自本 rank、第 slot 个 token”。
- **传输选择**：`mc_route_put(comm_ctx, dst_rank, dst_ptr)`（`:191`）。返回非空 = 本地或 NVLink P2P，做 warp 协作拷贝 + fence；返回空 = 需要 IBGDA，`mc_rdma_put(...)` 直接从源缓冲发 RDMA WRITE（`:203-222`）。
- 每个 token 发完后 `mc_atomic_add_release(atomic_finish_counter_per_expert + dst_expert_idx, 1)`（`:230-235`），作为“本地发送完成”计数。

计数阶段（`:249-301`）：最后一个 warp（MUSA/MACA 上是最后一个 warp group）做两件事：

- SM 0 负责清空 **next buffer** 的计数器（`:251-271`），并先加 `FINISHED_SUM_TAG`（1024）作为“本专家已被清空/就绪”的基准。
- 每个 SM 统计自己负责的专家在 `topk_idx` 中出现的次数（`:274-300`），用 `warp_reduce_sum` 汇总到 `shared_num_tokens_sent_per_expert`，并加 `FINISHED_SUM_TAG - sum`。
- 一个容易漏掉的并发细节：在 `MOONCAKE_EP_SPLIT_SEND_RECV`（MUSA/MACA）下 `mc_bar_sync` 就是 `__syncthreads()`，而数据 warp 每个 token 迭代都要过它一次（`:176`），所以计数路径必须空转同样次数的 `__syncthreads()`（`:238-247`），否则 CTA barrier 到达次数不匹配会直接挂死。这是“为什么计数分支里有一段看起来什么都不做的 token 循环”的原因。

这里的计数协议很巧：发送完成后计数器的值应恰为 `FINISHED_SUM_TAG + (FINISHED_SUM_TAG - sum) + sum = 2*FINISHED_SUM_TAG`。所以负责发送计数的 lane 只需等 `== FINISHED_SUM_TAG*2`（`:315-324`），不需要额外的 barrier/共享内存握手。

计数发送（`:309-346`）：每个专家由 `sub_warp_id==0 && lane_id==0` 的线程向目标 rank 写 `-num_tokens_sent - 1`：跨 rank 用 `mc_red_add`，本地用 `mc_st_release`（`:325-337`）。**用负数**是因为接收端把“非 0”当作“已到达”，而 0 表示未到达；`-1` 编码了“0 个 token”这一合法情况。随后清零两个计数器，并在 `dst_rank==0` 时清 `packed_recv_count`（`:340-345`）。

#### 2.4 内核 RECV：等信号、按源偏移、打包

`mooncake-ep/src/mooncake_ep_kernel.cu:350-432`：

- 只有 `phases & SEND` 时才做 `mc_grid_sync()`（`:355-356`），保证 `packed_recv_count` 对所有 block 可见。
- 每个专家由 `src_rank = responsible_expert_idx / num_local_experts` 决定“等待哪个源 rank 的计数”（`:360-364`）。
- **超时检测**（`:378-390`）：`sub_warp_id==1 && lane_id==0` 的线程 `clock64()` 轮询 `rdma_recv_signal_buffer[local_expert*num_ranks + src_rank]`；超过 `timeout_ticks` 就把 `active_ranks[src_rank]=0`；一旦 `active_ranks[src_rank]==0` 立即 `num_recv_tokens=-1` 跳出。之后 `num_recv_tokens = -num_recv_tokens - 1`，把 `-1` 还原成 0。
- **接收偏移**：`recv_token_begin_idx = atomicAdd(packed_recv_count + local_expert, num_recv_tokens)`（`:391`），同一专家的多个源 rank 因此串行拼接。
- `recv_range[src_rank] = pack2<int,int64_t>(num_recv_tokens, recv_token_begin_idx)`（`:394`），这就是 combine 用来定位的 `layout_range`。
- 用 `mc_bar_sync(warp_group_id + 2, kNumWarpsPerGroup*32)` 在 warp group 内同步（`:396`），随后按 `i += kNumWarpsPerGroup` 并行拷贝源信息/数据/scales（`:402-429`）。FP8 的 scales 是以 `num_ranks*max_tokens` 为 stride 写到 `packed_recv_x_scales` 的转置视图里（`:420-428`）。

#### 2.5 combine 的反向走法

`mooncake-ep/src/mooncake_ep_kernel.cu:514-840`：

- 每个 warp group 负责一个**全局专家 id**；`dst_rank = responsible_expert_idx / num_local_experts` 语义上是“这条数据要回送的源 rank”，`local_expert_idx = responsible_expert_idx % num_local_experts`。
- 从 `layout_range[local_expert*num_ranks + dst_rank]` 解出 `(offset, num_tokens_to_send)`（`:586-595`），然后按 token 发。
- `mc_route_put` 非空：直接 warp copy 到对端（`:610-617`）；空：非 zero_copy 先 staging 到发送缓冲，再 `mc_rdma_put`（`:618-641`）。
- 完成后向目标 rank 的 `rdma_recv_signal_buffer + global_expert_idx` 发 `mc_signal(..., 1)`（`:652-666`）。注意信号索引是**发送方的全局专家 id**，目标端正好按自己的 top-k 专家 id 去等。
- 接收端每个专家由 `sub_warp_id==0 && lane_id==0` 等信号（发送侧的 `sub_warp_id` 用于轮转 token，信号由 `sub_warp_id==1 && lane_id==0` 在 `mc_bar_sync(warp_group_id+1, ...)` 之后发出，`:650-674`），超时同样把 `active_ranks[src_rank]=0`（`:693-705`）。
- `mc_grid_sync()`（`:715`）后做规约：每个线程负责 hidden 的一段，遍历 `num_topk`，跳过 `topk_idx<0` 与 `active_ranks[expert_src_rank]==0` 的项，按 `topk_weights` 累加，最后写回 bf16（`:771-838`）。MUSA 分支对 `bfloat162` 做了成对处理（`:805-813`）。

#### 2.6 纯 torch 回退

`python/mooncake/mooncake_ep_buffer.py:730-946`（dispatch）与 `:948-1085`（combine）：

- 先 `all_gather` 各 rank 的 token 数，按 `active_ranks` 把不可达 rank 的 token 数置 0（`:746-759`）。
- pad 到 `max_num_tokens`，`all_gather_into_tensor` 拿全量 `x` 与 `topk`（`:761-794`）。
- 对每个本地专家，找出所有 rank 中路由到它的 token 位置，按 `src_rank` 顺序拼接，写入 `layout_range[le, src_rank] = (begin << 32) | count`（`:812-842`）。
- 物化 `packed_recv_x`（fp8 时调 `_fp8_cast`），返回 dummy event/hook（`:896-946`）。
- combine 反向构造 `send_buf[src_rank, token]`，累加后 `all_reduce`，再取本 rank 切片（`:1030-1081`）。

回退路径的语义与原生路径**不完全等价**：它用全局 `all_gather`/`all_reduce`，通信量与显存占用都是 O(world × tokens)，且 `_DummyEvent` 只是 `torch.cuda.synchronize()`（`:711-716`）。

### 流程三：ElasticBuffer——传输协商、dispatch/combine 与 NCCL 重配置

#### 3.1 构造与组内传输共识

入口 `python/mooncake/mooncake_elastic_buffer.py:347-469`。

1. 规范化请求：`_requested_transport()` 把 `auto` 展开成 `MOONCAKE_EP_TRANSPORT` 环境变量的值（`:16-26`）。
2. 先算逻辑/物理域：`_calculate_logical_domain_size()` / `_calculate_physical_domain_size()`，再 `_refresh_topology_from_runtime()` 用原生返回值覆盖（`:516-524`）。注释说明：NCCL 的 LSA team 才是本地域的权威，CUDA 设备数只是启发式。
3. **组内共识**：`_select_transport_for_group()`（`:168-243`）用一个 4 元 int32 tensor `[requested_code, nccl_ready, required_code, num_ranks]` 做 `all_gather`，然后 `_resolve_transport_consensus()`（`:68-131`）：
   - 所有 rank 的 `requested` 必须一致，否则报错并列出每个 rank；
   - `required`（旧 generation 的要求）必须一致；
   - 若请求/要求是 `nccl` 而任一 rank 不 ready → 全组报错；
   - `auto` 模式下**只要有一个 rank 不能 NCCL，全组退回 ibgda**（`:126-131`）。
4. `_nccl_topology_supported()`（`:29-44`）的形状约束：单节点 `num_nvlink∈{2,8}`；多节点需要 hybrid，且 `(rdma=2, nvlink∈{4,8})` 或 `(rdma=4, nvlink=4)`。
5. NCCL 模式还要 `_require_full_nccl_membership()`（`:488-514`）：从 PG 取 active mask，all_gather，要求**所有逻辑槽位都已被恢复**，否则 NCCL 不能重建 communicator（`_resolve_nccl_membership_masks`，`:134-159`）。
6. `_exchange_nccl_unique_id()`（`:642-703`）：rank 0 调 `create_nccl_unique_id()`，先广播一个 status 再广播 32 个 int32 的 payload——**先把错误广播出去，避免非 root rank 在 payload 广播里永久等待**。
7. 构造原生 `ep.ElasticBuffer(...)`，再 `_connect_native()`（`:534-640`）做 IBGDA 元数据交换（与 legacy 流程同构，但用 `ep.MAX_QP_COUNT // num_ranks` 定 all_to_all 宽度）。
8. 两次 `torch.cuda.synchronize()` + `dist.barrier`（`:467-469`）。

#### 3.2 原生 dispatch 与编译期形状

`mooncake-ep/src/mooncake_ep_elastic_buffer.cpp:830-973`：

1. 断言 `num_experts % num_ranks == 0`、`num_recv_tokens <= max_tokens*num_ranks`；非 cached 模式必须 `== max`（`:845-858`）。
2. `max_num_recv_tokens = num_max_tokens_per_rank * num_ranks`——注释特意说明 hybrid 下每个 scale-up 对端可能收到来自所有 scale-out rank 的 token，所以容量与 sentinel 要覆盖**完整逻辑世界**（`:848-855`）。
3. `stream_wait(launch_stream, compute_stream)`（`:906-907`），**即使 `async_with_compute_stream=true` 也会先等**；之后再按需 `stream_wait(compute_stream, launch_stream)`。
4. `make_launch_context(timeout_cycles)`（`:914`）：
   - NCCL：`local_base = nccl_state_->allocation`，`ctx.num_qps = gin_context_count`；
   - IBGDA：`local_base = gdr_buffer`，`rdma_recv_signal_buffer = local_base`，`rdma_send_signal_buffer = local_base + workspace_bytes`，`ctx.buffer = local_base + workspace_bytes + scratch_bytes`（`:368-385`、`:390-392`）。
   注意这与 legacy `BufferPair` 的四段布局**完全不同**：Elastic 的布局是“recv signal 区（与 workspace 同址，都从 `local_base` 开始）+ send signal 区 + atomic scratch + payload”，后三段的边界由 `elastic_workspace_num_bytes()` / `elastic_atomic_scratch_num_bytes()` 固定（`:46-72`）。也就是说 `ctx.workspace` 与 `rdma_recv_signal_buffer` 指向同一块内存，读代码时不要当成两段独立区域。
5. 可选 deterministic prologue（`:916-935`）：仅当 `deterministic && !cached && !hybrid`，MUSA 恒不跑。
6. 主 dispatch `launch_mooncake_elastic_dispatch` + copy epilogue `launch_mooncake_elastic_dispatch_copy_epilogue`（`:937-962`）。
7. `(void)active_ranks;`（`:964`）——**原生 Elastic 当前不使用 `active_ranks`**，成员固定。这是集成时最容易误判的一点：传进来的 mask 只影响 Python 侧与 PG，不影响这个内核。

内核启动器的 backend 选择在 `mooncake-ep/src/mooncake_ep_elastic_kernel.cu:742-761`：

```cpp
#ifdef USE_NCCL_DEVICE
if (ctx.backend == kNccl) { launch_..._backend<NcclOps>(..., ctx.nccl, ...); return; }
#endif
if (ctx.backend != kIbgda) throw ...
launch_..._backend<IbgdaOps>(...);
```

形状匹配是**逐参数全等**（`TRY_DISPATCH` 宏，`:700-711`）：

```cpp
hidden == H && num_experts == E && num_topk == K &&
num_max_tokens_per_rank == M && num_sms == S &&
num_scaleup_ranks == R && elem_size == EL && num_sf_packs == SFP &&
expert_alignment == 1 && !do_cpu_sync
```

不匹配就 `unsupported_elastic_config()` 抛错（`:726-728`）。hybrid 分支的形状宏再叠加 `(scaleout, scaleup) ∈ {(2,4),(2,8),(4,4)}`（`:633-637`）；combine 的非 hybrid 列表同样只有 `(4096,256,8,128,24,{2,8})`（`:1023-1044`）。

这里有个容易误读的 `do_cpu_sync`：它出现在形状匹配里，但 host 侧 `MooncakeElasticBuffer::dispatch` 调启动器时**恒传 `false`**（`mooncake-ep/src/mooncake_ep_elastic_buffer.cpp:937-946`）。Python 的 `do_cpu_sync` 只控制 Python 层是否把 prefix-sum 拷回主机并裁剪输出（`python/mooncake/mooncake_elastic_buffer.py:1187-1224`），与内核形状无关。

Python 侧 `num_sms` 由 `get_theoretical_num_sms()` 给出：`prefer_overlap_with_compute` 时 `max(1, min(24, sm_count/4))`（`mooncake-ep/src/mooncake_ep_elastic_buffer.cpp:817-828`，Python 同式 `:928-933`）。在 132 SM 的 H100 上正好 24；在 SM 数不是 ~96 的卡上可能得到别的值，从而“形状不支持”。

#### 3.3 combine

`mooncake-ep/src/mooncake_ep_elastic_buffer.cpp:975-1038`：`launch_mooncake_elastic_combine` 返回一个 `reduce_buffer`，再交给 `launch_mooncake_elastic_combine_reduce_epilogue`。同样 `(void)active_ranks`（`:1032`），且 `stream_wait` 语义与 dispatch 一致。

#### 3.4 `update_ep_member()` 与 NCCL 重配置

IBGDA 模式下就是 `_connect_native(True)`（重建 QP + 重交换元数据）。NCCL 模式下（`python/mooncake/mooncake_elastic_buffer.py:705-748`）：

1. `torch.cuda.synchronize()` + 切到构造时记录的 device。
2. 重新跑一次组内 transport 共识，并传入 `required_transport=self.transport`（`:721-730`）——如果拓扑已经不再支持 NCCL，直接报错让用户改 `ibgda`。
3. `_require_full_nccl_membership()` + `_exchange_nccl_unique_id()`。
4. 原生 `reconfigure_nccl(unique_id)`（`mooncake-ep/src/mooncake_ep_elastic_buffer.cpp:605-662`）：
   - `cudaStreamSynchronize` 明确静止边界；
   - **先完整构造 candidate**（新 allocation、新 NCCL communicator），再 `std::move` 替换；
   - `allRanksSucceeded(local_workspace_valid)` 让替换 rank 与存活 rank 走同一条集合序列；
   - 最后 `previous->abort()` 养老代；注释解释替换进程从未拥有旧 communicator，不能要求正常集体销毁（`:650-656`）。
5. Python 侧在 `reconfigure_nccl` 返回后**立刻**更新 `_generation`（`:739-742`），再刷新拓扑、把 `_active_ranks` 填 1。
6. 最后 `synchronize` + barrier。

补充一个同样依赖“全体 rank 同步静止”的生命周期点：`ElasticBuffer.destroy()` 在 NCCL 模式下先 `torch.cuda.synchronize()` + `dist.barrier`，再调原生 `destroy()`（内部 `NcclElasticState::release()` 做 deregister → free → shutdown），最后再 barrier 一次（`python/mooncake/mooncake_elastic_buffer.py:750-768`）。对称窗口（symmetric window）的注销必须是集合操作，所以销毁不能由单个 rank 单方面决定时序——只查 C++ 析构会低估这里的约束。

### 流程四：PG 的初始化与一次 collective

#### 4.1 `import mooncake.pg` → backend 注册

`mooncake-wheel/mooncake/pg.py` 的 `_load_jit_adapter()`：

- 源码目录优先取 wheel 内的 `_pg_jit`，否则回退到源码树的 `mooncake-pg/torch`（`_source_dir()`）。
- 缓存 key 是 `torch.__version__ + cuda/musa + CXX11_ABI + git_version + backend + core .so 路径 + sys.implementation.cache_tag + 源文件内容 + core lib 字节内容`的 sha256 前 20 位（`_cache_key()`）。
- 用 `fcntl.flock` 做跨进程构建锁，`torch.utils.cpp_extension.load(...)` 现场编译 3 个 .cpp（`pg_py.cpp`、`mooncake_backend.cpp`、`work_handles.cpp`）。
- 找不到 `libmooncake_pg.so` 或 `ninja` 就直接报错。

`pg_py.cpp` 里的 `__attribute__((constructor)) MooncakeBackendConstructor()`（`:108-127`）在模块加载时向 `torch.distributed.Backend.register_backend` 注册 `mooncake-cpu`（device=cpu）与 `mooncake`（cuda 或 musa）。这就是“`mooncake-pg/torch/` 没有 CMake 目标”的原因——它不是预编译的 `.so`，而是运行时 JIT 出来的扩展模块。

#### 4.2 进程级控制面引导

`initControlPlane()`（`mooncake-pg/torch/src/pg_py.cpp:45-82`），`std::call_once` 保证每进程一次：

1. `mooncakePgContextInitialize(context, rank, MOONCAKE_PG_MAX_RANKS)`：初始化 TE engine、LinkManager，可能初始化 collective v2 的设备资源（`mooncake-pg/src/mooncake_communicator.cpp:191-263`）。
2. `rank==0` 时 `mooncakePgContextLaunchCoordinator()`，把返回地址 `store->set("coordinator_addr", ...)`。
3. 所有 rank `store->wait({"coordinator_addr"})` 后 `connectCoordinator()`，启动 AgentHost。
4. **保活 bootstrap store**：`g_ctx.bootstrap_store = store`。注释解释：默认 PG 被销毁重建时，rank 0 的 TCPStore server 不能跟着死（`:76-79`）。
5. `atexit` 注册 `shutdownProcessContext`，注释说明为什么不用 `Py_AtExit`：Python 的 TE wrapper 可能已被 decref（`:254-258`）。

#### 4.3 建 communicator（含组注册 2PC）

`MooncakeBackend` 构造（`mooncake-pg/torch/src/mooncake_backend.cpp:299-421`）：

1. 处理 `activeRanks_`：用户给了 tensor 就**只用它的 storage**（内容由 Coordinator 填），否则按 `max_group_size` 分配；校验 dtype/contiguous/大小/设备（`:325-340`）。
2. 处理 `global_ranks_in_group`，缺省是 `iota(size)`（`:345-359`）。
3. 填 `mooncakePgCommConfig_t`，其中 `idResolvePolicy` 由 `options_->isExtension_` 决定（`:387-389`），`preferredGpuCollectiveBackend` 从环境变量 `MOONCAKE_PG_PREFERRED_GPU_COLLECTIVE_BACKEND` 读（`:374-386`）。
4. `mooncakePgCommCreate()` → `MooncakeCommunicator::create()`（`mooncake-pg/src/mooncake_communicator.cpp:460-473`）→ `initialize()`（`:492-767`）：
   - 校验 rank/global rank 映射；
   - 分配并 `registerLocalMemory` 双缓冲 + 同步区（`:571-596`）；
   - 拿 worker 与 P2P device worker（按 device 共享），`worker_->Start()`（`:598-608`）；
   - 构造 `P2PProxy` 并 `bindMeta`（`:610-642`）；
   - 分配 `activeRanks`（GPU 用 `cudaHostAlloc` + `cudaHostGetDevicePointer`，映射给内核）（`:644-657`）；
   - 填本地 `GroupEndpointInfo`（send/recv buffer、sync、P2P credit/ack、device collective endpoint）（`:659-679`）；
   - `agent_.waitUntilRegistered(30s)`；
   - `agent_.registerGroup(bootstrap_id, max_group_size, rank_order, policy, auto_deactivate, this)`（`:733-740`）。bootstrap id 加 `cpu:` / `device:` 前缀，CPU 与 GPU 命名空间隔离；
   - 注册被拒 → 本地降级为 `{self}` 成员、`autoSyncOnFailure=false`、返回（`:742-755`）；
   - 成功 → `publishLocalEndpoint()` + `waitUntilGroupReady(300s)`，再对每个本地槽位 `refreshSegmentID()`（`:757-765`）。
5. 注册 `MooncakeBackendShim`（`:412-420`）。Shim 存在的原因写在注释里（`:183-196`）：PyTorch 的 P2P 派发（`batch_isend_irecv`/`isend`/`irecv`）与 2.13 的 `all_gather_single`/`reduce_scatter_single` 都要通过 `ProcessGroup::getBackend(dev)` 找到**已注册的 `c10d::Backend`**，而 `MooncakeBackend` 继承的是 `ProcessGroup` 不是 `Backend`。

#### 4.4 一次 `dist.all_reduce` 的完整路径

```
Python: dist.all_reduce(t)
  -> MooncakeBackend::allreduce(tensors, opts)                  [mooncake_backend.cpp:557-569]
     -> makeContiguous / post_completion
     -> launchCollective<mooncakePgAllReduceCpu, ...Gpu>(...)   [:442-483]
        -> currentCudaStream(tensor) -> mooncakePgAllReduceGpu(...)
           -> MooncakeCommunicator::allReduceGpu(...)           [mooncake_communicator.cpp:990-1040]
              -> checkOpState + 参数校验
              -> [v2] device_collective_->enqueueAllReduce(...)  or
              -> [legacy] worker_->putTaskCuda(..., copyToSend, copyFromRecv)
                 -> MooncakeWorker::putTaskCuda                   [mooncake_worker_host.cpp:256-344]
                    -> 按 chunk 切分，轮换 bufferOffset = taskCount % 2
                    -> launchEnqueueTaskKernel(..., tasks_device_, taskId, enq_stream)
                    -> copyFromRecvBuffer(...) 排在 enqueue 流上
                    -> waitUntilTasksSubmitted(submitted_tasks)
                    -> issue_stream.waitEvent(event_end)
        -> event->record(stream) + 捕获状态检查 -> MooncakeWorkCuda
```

legacy worker 线程的循环（`mooncake-pg/src/mooncake_worker_thread.cpp:91-481`）是**四槽状态机**（2 CPU + 2 CUDA），每个槽：

- `IDLE`：为每个 `activeRanks[j]` 且未失败的 peer 生成一条 `TransferRequest{WRITE, source=本地 send_buffer, target_id=对端 segmentID, target_offset=对端 recv_buffer + 自己的槽偏移, length=dataSize}`，`allocateBatchID` + `submitTransfer`，记录 `submitted_task_sequence_`，进入 `TRANSFERRED_1`（`:168-237`）。
- `TRANSFERRED_1`：轮询 `getTransferStatus`。`FAILED` 直接判死；超过 `collectiveTimeoutUs` 且 `probePeerAliveByID` 不是 Alive 才判死（`:238-289`）。全部完成后 freeBatchID，再向每个 peer 的 `recv_sync + rank*sizeof(int32_t)` 写 1（`:301-330`），进入 `SIGNALED_1`。
- `SIGNALED_1`：等本地的 `recv_sync[j] == 1` 且状态 COMPLETED（`:331-380`）。
- 完成时：清 `signal_ptr`、把 `failedRanks[i][rank]` OR 进 `failed_ranks_hint`、按需设置 mirror generation、构造 `LinkEvent`（成功/失败 + `target_rank_epochs`）推给 Agent、freeBatchID、若 `task_detected_failure && autoSyncOnFailure` 则 `communicator->syncAfterFailure()`（`:385-476`）。

Collective 的语义就是“每个 rank 把自己的 send_buffer 写到所有 active peer 的 recv_buffer 的固定槽”，再在本地对 recv_buffer 做 reduce（`launchReduceKernel`，`mooncake_communicator.cpp:1035-1037`）。`AllToAll/ReduceScatter/Scatter` 只是改变源/目标偏移的计算方式（`:190-217`）。同步区是**每 rank 一个 int32**，所以“信号写入 recv_sync + rank”是非原子的单写者协议。

上面这句“每个 rank 写给所有 peer”只对全参与算子成立：`Broadcast`/`Scatter` 的非 root 与 `Barrier` 走 `skipTransfer` 分支（`:133-137`、`:158-164`），**不发任何数据请求**，但仍然要进入后面的同步字阶段；`Gather`/`Reduce` 则只有 root 收数据，其它 rank 只发不写自己（`:174-178`）。所以把 collective 一律理解成“全交换”会在读 Broadcast/Gather 的偏移计算时对不上号。

#### 4.5 P2P：接收方驱动的信用制

`MooncakeCommunicator::enqueueSend/Recv`（`:843-906`）只是把 `SendOp/RecvOp` 塞进 `P2PProxy` 的队列并唤醒设备 worker；真正协议在 `p2p_proxy.cpp`：

1. **接收方**从 RecvPool 取 chunk，把 `CreditSlot{epoch, seq, recv_addr, chunk_len}` 通过 RDMA WRITE 写进**发送方的本地 CreditLane**（`p2p_proxy.h:61-70`）。
2. **发送方**轮询自己的 CreditLane；`tryLoad()` 校验 header/footer 一致（`p2p_proxy.cpp:70-99`）；`slot_epoch < lane.minimum_credit_epoch_` 的旧信用被丢弃（`:768-771`），同一 op 内 epoch 变化直接报错（`:781-803`）；然后从 SendPool 取 staging、拷贝用户数据（GPU 走 `cudaMemcpyAsync`，CPU 走 `memcpy`）、RDMA WRITE 到 `recv_addr`、再写 `AckSlot` 回接收方。
3. **接收方**轮询 AckLane，匹配 `credit_epoch_`（`:1125-1139`），然后 `cudaMemcpyAsync` 拷出到用户 buffer、归还 chunk。
4. 任一步超时或传输失败 → `reportPeerFailure()`（`:327-350`）：`resetPeerState()`、按 `rank_order` 换算 global rank、填 `LinkEvent{target_rank_epoch}` 推给 Agent、`autoSyncOnFailure` 时 `syncAfterFailure()`。

**为什么接收方驱动**：接收方才知道自己哪块显存可用；发送方被动等待 credit，天然避免了“发送方写爆接收方”的流控问题。**为什么 SendPool/RecvPool 严格分离**：如果共用池，可能出现“所有 chunk 都被预留给接收、没有 chunk 可发送”的死锁（`p2p_proxy.h:80-86`）。**为什么分配不阻塞**：池耗尽时步骤函数直接返回 false，下一轮轮询重试（同处注释）。

#### 4.6 故障：从数据面证据到成员决策

这是 PG 最有价值也最容易讲错的部分。

1. **数据面只产生证据**：worker 写 `failed_ranks_hint`（按 InGroupRank 索引，长度 `max_group_size`），并推 `LinkEvent`；P2PProxy 同理。worker **不做成员决策**（`docs/source/design/mooncake-backend-pg.md:296-310`）。
2. **Coordinator 收证据**：`processLinkEventReport()`（`mooncake-pg/src/control_plane/coordinator.cpp:482-542`）校验 session/rank epoch/report_id 单调，更新 `link_status[reporter][peer]`。
3. **负证据开窗**：任一 `Failure` 就 `tryOpenReconciliationWindow()`，窗口默认 30s（`kDefaultFaultReconciliationWindowUs = 3 * 10s`，`mooncake-pg/include/mooncake_communicator.h:45-46`）；窗口内的 `sync_after_failure` 请求被挂进 `pending_syncs`（`:576-581`）。
4. **窗口到期决策**：`tick()`（`:621-694`）→ `tryCloseReconciliationWindow()`（`:465-479`）依次做 `updateRankStates`、`applyAutoDeactivate`、`checkGroupTransitions`、`resolvePendingSyncs`。
5. **健康集推导**：`extendHealthySet()`（`:839-916`）先取所有 `Healthy` 且自连通的 rank，然后**反复踢掉连接度最低的 rank 直到剩余集合是团（clique）**；单点且无任何连接则清空；最后把与集合内所有人互通的新候选并进来。
6. **状态落库**：`updateRankStates()` 在 `Healthy ↔ Synced` 间迁移并递增 `rank_state_version`、产生 `makeRankStateEffect`（`:918-939`）；`applyAutoDeactivate()` 对 `Ready` 且 `auto_deactivate` 的组把不健康成员改为 `Inactive`、清 endpoint、`epoch++`、推 `PushViewUpdate`（`:941-973`）。
7. **视图 2PC**：Coordinator 推 `ViewUpdate` 并等所有相关 rank 的 ACK（`handleViewUpdateAck`，`:594-619`）；超时未 ACK 的 rank 被记入 `dropped_ranks` 并 `handleTimedOutAgent`（`tick()` 的 barrier 过期分支，`:638-672`）。
8. **本地应用**：Agent 收到视图后调 `MooncakeCommunicator::applyGroupState()`（`mooncake-pg/src/mooncake_communicator.cpp:1720-1912`）：
   - `view.epoch < current` 直接丢弃；
   - 若 `self` 在新视图中 `Active` → `next_mode = Normal`；否则保持原模式（新 joiner 留在 `Isolated`，等待激活的留在 `Quiescing`，被自动去激活的留在 `Normal` 以便下次集合操作快速失败）；
   - `Isolated/Quiescing` 时 `activeRanks[] = {self}`；`Normal` 时按 `rank_order` 映射 `view.members[g].isActive()`（`:1804-1824`）；
   - 只有执行模式或有效参与者变化才 `taskCount = 0`（`:1826-1834`）；
   - 更新 `rank_order`/`segmentInfos`/`rankStates`/`rankEpochs`/`maybeActivatable`，`activeSize` 用 release 发布（`:1862-1864`）；
   - v2 后端时调 `device_collective_->applyGroupView(view)` 或 `useLocalOnly(epoch)`（`:1866-1878`）；
   - **最后**才 release 发布 `epoch`（`:1902-1907`），保证“看到新 epoch 的线程能看到完整成员状态”。
9. **`sync_after_failure`**：`communicator.syncAfterFailure()` 只是 `agent_.syncAfterFailure(group_id)`（`:1714-1718`）；Coordinator 侧 `handleSyncAfterFailure` 在窗口未关时挂起，窗口关闭后返回权威视图 + 状态（`coordinator.cpp:545-591`、`makeSyncResponse`/`resolvePendingSyncs`）。

---

## 与其他模块的边界

### EP ↔ Transfer Engine

EP 只依赖 TE 的**设备传输抽象**与 `comm_device.cuh` 的 `CommCtx`/`mc_*` 原语：

- `device::P2pTransport`（`mooncake-transfer-engine/include/transport/device/device_transport.h:48-95`）：`allocateBuffer`、`exportIpcHandle`、`importPeerHandles`、`availableTablePtr`、`peerPtrsTablePtr`。
- `device::RdmaTransport`（同文件 `:121-184`）：`initialize`、`registerMemory`、`allocateControlBuffer`、`createQueuePairs`、`connectPeers`、`raddrsPtr`、`rkeysPtr`、`qpDevCtxsPtr`、`recreateQueuePairs`。
- `mc_route_put`（`mooncake-transfer-engine/include/transport/device/comm_device.cuh:65-78`）：**“返回非空 = 本地/P2P 可直写；返回空 = 必须走 RDMA”**，这是 EP 两条数据路径的唯一分叉点。

EP 的**原生 C++ 构造函数**支持传入外部 `TransferEngine*` 复用其 transport（engine 拥有传输、EP 只持有裸指针）；不传时 EP 用 `createP2pDeviceTransport` / `createIbgdaDeviceTransport` 自建并持有（`mooncake-ep/src/mooncake_ep_buffer.cpp:161-210`）。注意 pybind 的 `Buffer` 只暴露了 4 个参数（`mooncake-ep/src/ep_py.cpp:41-43`），所以 Python 用户走不到“外部 engine”这条路径，永远由 EP 自建传输。

### EP ↔ PG

- **C++ 层无依赖**（已验证）。
- 唯一接口在 Python：`Buffer.connect()` 借 PG 的 `all_gather/all_to_all`；`_active_ranks_tensor()` 调 `mooncake.pg.get_active_ranks`；`update_ep_member()` 在 PG 视图变化后刷新。
- 两个 `active_ranks` 语义不同（见下节）。

### PG ↔ PyTorch

- `MooncakeBackend : c10d::ProcessGroup`，通过 `MooncakeBackendShim : c10d::Backend` 暴露给 P2P 派发。
- bootstrap 用 `c10d::Store`（`coordinator_addr` key）。
- `distBackendOpts.group_id` 只是 bootstrap id；Coordinator 把它解析成进程生命周期的 `GroupId`（CPU/GPU 命名空间独立）。
- 工作句柄 `MooncakeWorkCpu/Cuda/P2PWork` 实现 `c10d::Work`，附加 `get_failed_ranks_hint` / `get_local_success`（`mooncake-pg/torch/src/pg_py.cpp:185-209`）。

### PG ↔ 控制面 RPC

Coordinator↔Agent 走 `yalantinglibs` 的 `coro_rpc`（`mooncake-pg/src/control_plane/`），**不走 Transfer Engine**。`CoordinatorHost` 只在 global rank 0 的进程内起，无 HA。

### 打包边界

CUDA fatbin 被刻意拆成 `libmooncake_ep_device.so` / `libmooncake_pg_device.so`：host 扩展先过 auditwheel，设备库事后注入 wheel（根 `CMakeLists.txt:249-264`；`mooncake-ep/src/CMakeLists.txt:16-30` 的 `add_library(mooncake_ep_device SHARED ...)`）。在源码树里找“EP 内核在哪个 so”时不要找 `_ep.so`。

`mooncake-pg/torch/` 没有 CMake 编译目标；根 `CMakeLists.txt:176-191` 的 `EP_USE_IDE` 分支只为 clangd 建一个 OBJECT 库，并自带 “DO NOT USE IN PRODUCTION” 警告。

---

## 设计取舍与易踩坑点

1. **`getSize()` 不是活跃成员数**。它是“最高活跃 InGroupRank + 1”；空洞不重编号（`mooncake-pg/src/mooncake_communicator.cpp:781-787`、`:1784-1794`，语义文档 `docs/source/design/mooncake-backend-pg.md:78-101`）。扩展期（`Isolated`/`Quiescing`）返回声明的 `initial_size_` 以通过 PyTorch 的 rank 校验。要数参与者请数 `active_ranks`。

2. **两个 `active_ranks` 不会自动同步**。PG 层是 Coordinator 提交的配置；EP dispatch/combine 的 `[num_ranks] int32` 张量是观测，legacy 内核超时时会**就地改写为 0**（`mooncake-ep/src/mooncake_ep_kernel.cu:382-384`、`:697-699`）。集成方必须自己把健康状态在调度器、PG、EP 之间传播。文档也明确说了这一点（`docs/source/design/mooncake-ep.md:170-179`）。
   还有一个更隐蔽的存储别名：`Buffer.dispatch(active_ranks=None)` 时，`_active_ranks_tensor()` 调 `mooncake.pg.get_active_ranks()`，而 `pg_py.cpp:129-133` 返回的正是 backend 的 `activeRanks_` tensor（按值返回但**共享 storage**），其背后就是 `config.activeRanksMirror` 指针（`mooncake-pg/torch/src/mooncake_backend.cpp:403-407`、`mooncake-pg/src/mooncake_communicator.cpp:1566-1593`）。因此内核超时写 0 会直接写进 PG 的镜像。想避免这种跨层副作用，就显式传一个自己的 `active_ranks` 张量。

3. **ElasticBuffer 原生内核当前完全忽略 `active_ranks`**（`mooncake-ep/src/mooncake_ep_elastic_buffer.cpp:964`、`:1032`）。它的成员是固定的（NCCL 甚至是固定 communicator），所以“EP 级超时降级”只在 legacy Buffer 上成立。

4. **legacy 内核的形状约束是硬失败，不是慢路径**。`hidden ∈ {2048,2560,3072,3584,4096,5120,6144,7168,8192}`（`mooncake-ep/include/mooncake_ep_launch.cuh:66-89`），dispatch 的 warp-group 数 3..8（`mooncake-ep/src/mooncake_ep_kernel.cu:497-506`；实际 CUDA 恒为 8、MUSA 取 5..8，combine 固定 8），top-k ≤ 17（`:464-465`），`num_ranks ≤ MAX_QP_COUNT=256`（`mooncake-ep/include/mooncake_ep_configs.cuh:5`）。Elastic 更窄：固定 `(4096,256,8,128,24)`，非 hybrid 的 scaleup ∈ {2,8}，hybrid 仅 `(2,4)/(2,8)/(4,4)`。

5. **`num_sms` 是形状的一部分**。Elastic 的 `get_theoretical_num_sms()` 在 `prefer_overlap_with_compute=true` 时给 `min(24, sm/4)`；只有 SM 数 ≥ 96 的卡才会得到 24。换卡就可能 `unsupported_elastic_config`。

6. **`MOONCAKE_EP_SPLIT_SEND_RECV` 与 cooperative launch**。CUDA 用 `cudaLaunchKernelEx` + `cudaLaunchAttributeCooperative`（`mooncake-ep/include/mooncake_ep_device.h:112-127`），一次内核里用 `mc_grid_sync()` 跨 block 同步；MUSA/MACA 没有 cooperative launch，只能拆成 SEND/RECV 两次 launch（宏 `MOONCAKE_EP_SPLIT_SEND_RECV` 在 `mooncake-ep/include/mooncake_ep_configs.cuh:51-53` 定义，`mooncake-ep/src/mooncake_ep_buffer.cpp:363-368` 使用），并在 MACA 上额外用一次 P2P 集合 fence 对齐相位（`python/mooncake/mooncake_ep_buffer.py:124-182`）。

7. **`throw` 在析构里**。`MooncakeEpBuffer::~MooncakeEpBuffer() noexcept(false)`（`mooncake-ep/include/mooncake_ep_buffer.h:121`）允许抛出；`MooncakeElasticBuffer::~MooncakeElasticBuffer()` 则捕获并只记日志（`mooncake-ep/src/mooncake_ep_elastic_buffer.cpp:557-565`）。改动清理路径时要注意这个不一致。

8. **`dispatch_shadow_slots` 不是缓存，是生命周期保险**。它让 combine 能用 `src_info_ptr` 找回 dispatch 时的 topk 快照；TBO 交错的多个逻辑 dispatcher 共用一个 native Buffer 时，进程级 `buffer_idx` 翻转本身不足以配对（`mooncake-ep/src/mooncake_ep_buffer.cpp:281-289`、`:428-440`）。但 `buffer_idx` 仍是**进程级**的，所以交替调用 dispatch 的 `next_buffer` 语义依然依赖调用顺序——这是“同一 Buffer 上交错两个 dispatcher”仍需外部串行的原因。

9. **combine 的 `zero_copy` 在 legacy Python 门面不可用**。原生 host 与内核都支持 `zero_copy`（`mooncake-ep/src/mooncake_ep_buffer.cpp:403`、`mooncake-ep/src/mooncake_ep_kernel.cu:621`），但 Python `combine()` 一进来就 `raise NotImplementedError`（`python/mooncake/mooncake_ep_buffer.py:568-569`），`get_next_combine_buffer()` 同样（`:705-706`）。设计文档描述的是原生能力。

10. **PG 的 GPU collective 有两代实现**。Legacy = `MooncakeWorker` 常驻 enqueue 内核 + host 线程 `submitTransfer`；v2 = `device_comm/` 下设备发起的 `DeviceCollectiveRuntime` + `RingAllReduceAlgorithm`（仅 `USE_CUDA`，`MOONCAKE_PG_HAS_COLLECTIVE_V2` 门控，按 `preferred_gpu_collective_backend`/环境变量在组内共识，冲突则退回 legacy，`mooncake-pg/src/control_plane/coordinator.cpp:1240-1295`）。只跟一条线读会误判行为。

11. **P2P 是接收方驱动的信用制**。发送方完全被动；SendPool/RecvPool 严格分离；chunk 分配永不阻塞轮询线程（`mooncake-pg/include/p2p_proxy.h:59-86`）。这与“发送方 push”直觉相反。默认池是 128 MiB/方向、16 MiB/chunk（`:167-171`）——注意文件头示意图用的是 8 MiB × 32 的示例，和实际默认值不一致（`:24-44`），读的时候别被图带偏。

12. **控制面 RPC 用 coro_rpc，Coordinator 无 HA**。rank0 进程死了，成员管理就没了；这与数据面的容错叙事形成反差。

13. **EP 的 Python fallback 是完整实现，不是占位**。`_fallback_dispatch`/`_fallback_combine` 用 `all_gather_into_tensor` + `all_reduce` 实现语义等价的全量搬运（`python/mooncake/mooncake_ep_buffer.py:730-1085`），但通信量 O(world×tokens)，且 `_DummyEvent` 是全局同步。`_use_fallback` 在 `sync_nvlink_ipc_handles` 之后必须重估（`:304-311`）。

14. **`auto_sync_on_failure` 依赖 `auto_deactivate_on_failure`**，非法组合在两处被拒（`mooncake-pg/torch/src/mooncake_backend.cpp:399-402`、`mooncake-pg/src/mooncake_communicator.cpp:502-504`）。默认 `auto_sync=true` 意味着失败集合会**卡在 reconciliation 窗口**（默认 30s）才返回，这是“安全但慢”的取舍（`docs/source/design/mooncake-backend-pg.md:368-371`）。

15. **`syncAfterFailure` 的语义**：既是上报路径也是同步点；请求里 piggyback Agent 未 ACK 的 link 观测（`mooncake-pg/src/control_plane/coordinator.cpp:566-581`）。`local_success` 只代表本调用方观察到所有必需传输完成；`failed_ranks_hint` 是证据而非结论。

16. **`checkOpState` 的拒绝矩阵**：有效组内 Offline 永远拒；`Quiescing` 拒一切；P2P 要求有效组；集合操作要求 `mode == Isolated`（本地模式）或 `activeRanks[self]` 为真（`mooncake-pg/src/mooncake_communicator.cpp:789-814`）。注册被拒的 communicator 会用 `Isolated` + `{self}` 继续本地执行。写集成代码时不要指望“无效组还能做 P2P”。

17. **每个 op 都分配一个 `failed_ranks_hint` tensor**（`mooncake-pg/torch/src/mooncake_backend.cpp:447-448`）。高频小集合操作下这是可见的分配开销；`MooncakeWorkTracker` 用机会式回收 + CUDA Graph 期间保留来平衡生命周期（`mooncake-pg/torch/include/work_handles.h:30-66`）。

18. **`MooncakeBackendShim` 是必需的，不是装饰**。PyTorch 2.13 的 `all_gather_single`/`reduce_scatter_single` 会绕过 `ProcessGroup` 的 `_allgather_base` 重载，直接走 `getBackend(dev)`；没有 shim 就会落到基类报错（`mooncake-pg/torch/src/mooncake_backend.cpp:183-196`）。

---

## 精读清单

建议按下面顺序读，每一步都先明确“我要回答什么问题”。

1. **`mooncake-transfer-engine/include/transport/device/comm_device.cuh:19-135`**
   看懂 `CommCtx`、`make_comm_ctx`、`mc_route_put`/`mc_rdma_put`/`mc_signal`/`mc_red_add`。要能回答：`mc_route_put` 返回空代表什么？为什么 EP 的两条路径可以在一个 `if` 里分叉？

2. **`mooncake-ep/include/elastic/mooncake_ep_elastic_transport.cuh:24-313`**
   把文件头的映射表与 `IbgdaOps` 的 `put/put_value/red_add_rel/flush` 一一对上。要能回答：为什么 `put` 里不能放 warp barrier？为什么 64 位 RED 在远端要拆两次 32 位？

3. **`mooncake-ep/include/mooncake_ep_buffer.h:24-210` + `mooncake-ep/src/mooncake_ep_buffer.cpp:123-394`**
   画出 `BufferPair` 的两套四区布局与 `buffer_idx` 翻转；看懂 `dispatch_shadow_slots` 的存在理由；列出 `launch_stream` 的三种选择与对应的流同步。

4. **`mooncake-ep/src/mooncake_ep_kernel.cu:44-433`（dispatch）**
   跟一遍“数据 warp 读 topk → 量化 → 抢 slot → `mc_route_put`/`mc_rdma_put` → 计数 warp 汇总 → `FINISHED_SUM_TAG*2` → 发负数计数”。要能回答：为什么计数用 `-num_tokens-1`？超时标记 `active_ranks[src]=0` 发生在哪一行？

5. **`mooncake-ep/src/mooncake_ep_kernel.cu:514-840`（combine）**
   跟一遍“按 layout_range 取 token → 写对端 → 发 `global_expert_idx` 信号 → 等信号 → 按 topk_weights 累加”。要能回答：combine 的接收端为什么按“全局专家 id”等信号，而不是按源 rank？

6. **`python/mooncake/mooncake_ep_buffer.py:184-311`**
   把 `connect()` 的 7 步元数据交换画成时序图。要能回答：为什么 `_use_fallback` 必须在 IPC handle 交换之后重估？

7. **`python/mooncake/mooncake_elastic_buffer.py:16-131` + `:347-469` + `:534-640`**
   看懂 transport 共识的三层（rank 本地选择 → all_gather → 组级裁决）。要能回答：`auto` 模式下为什么“一个 rank 不行就全组 ibgda”？

8. **`mooncake-ep/src/mooncake_ep_elastic_buffer.cpp:350-403` + `:830-973`**
   对比 `make_launch_context` 的两个分支（NCCL vs IBGDA）与 legacy 的 `BufferPair` 布局差异；确认 `active_ranks` 在原生 elastic 路径未被使用。

9. **`mooncake-ep/src/mooncake_ep_elastic_kernel.cu:700-728` + `:1023-1044`**
   核对编译期形状列表；再对照 `get_theoretical_num_sms()` 算一遍你手上 GPU 的 `num_sms` 是否会命中。

10. **`mooncake-pg/torch/src/pg_py.cpp:45-127` + `mooncake-pg/torch/src/mooncake_backend.cpp:299-421`**
    看懂进程级 context、Coordinator 引导顺序、`MooncakeBackendOptions → mooncakePgCommConfig_t` 的翻译与 shim 注册。

11. **`mooncake-pg/src/mooncake_communicator.cpp:460-767`**
    跟一遍建组：内存注册 → worker/P2PProxy → endpoint → `registerGroup` → `publishLocalEndpoint` → `waitUntilGroupReady` → `refreshSegmentID`。要能回答：注册被拒时为什么可以“本地降级”继续跑？

12. **`mooncake-pg/src/mooncake_worker_host.cpp:256-344` + `mooncake-pg/src/mooncake_worker_thread.cpp:91-481`**
    看懂 4 槽 Task 状态机与 `submitSequence` 接力；列出一次 collective 的“数据写 → 同步字 → 完成”三段。

13. **`mooncake-pg/include/p2p_proxy.h:24-163` + `mooncake-pg/src/p2p_proxy.cpp:298-350` + `:734-810` + `:1100-1150`**
    对着图走一遍 send/recv 五步；确认 header/footer 防伪与 epoch 丢弃逻辑。

14. **`mooncake-pg/src/control_plane/coordinator.cpp:456-542` + `:621-694` + `:839-973` + `:1186-1238`**
    看懂“负证据开窗 → 到期推导健康团 → 自动去激活 → 2PC 推视图 → 激活准入”。要能回答：`extendHealthySet` 为什么要反复踢掉连接度最低的 rank？

15. **`mooncake-pg/src/mooncake_communicator.cpp:1720-1912` + `:789-814`**
    看懂 `applyGroupState` 的模式推导、`activeSize` 的“上界”语义、`epoch` 的 release 发布顺序，以及 `checkOpState` 的拒绝矩阵。

16. **`mooncake-pg/tests/test_pg_elastic.py` + `python/tests/ep/test_elastic_buffer_recovery.py`**
    把上面的语义落到可执行断言：缩放、进程替换、优雅退出、空洞、原地重入。

读完之后，如果只能记住三句话：**EP 的快来自“内核直接 put + 信号计数”，PG 的强来自“控制面协调成员、数据面只产生证据”；两者在 C++ 层解耦、只在 Python 层借 process group 交换元数据；NCCL 在 EP 里不是被替代者，而是可选的第三个传输后端。**

---

# 15 Reshard / Conductor / RL / P2P Store / HiCache

> 本章覆盖 Mooncake 平台上层的五个"再分发 / 索引 / 分发 / 分层缓存"能力：`mooncake-reshard`（权重与 KV cache 再分片）、`mooncake-conductor`（KV-cache 前缀索引服务）、`mooncake-rl`（RL 数据流示例）、`mooncake-p2p-store`（BitTorrent 式对象分发）、HiCache（SGLang 侧三级缓存，本仓库只有设计文档与配套脚本）。
>
> 所有 `路径:行号` 引用均以仓库根目录 `D:/xiaojun/work/Mooncake` 为基准，行号对应 main 分支提交 `67df8c41` 的时点。凡是仅凭阅读无法确定的结论，文中会显式标注"此处未完全确认"。

## 定位与职责

这五个子系统都构建在两大底座之上：**Transfer Engine**（RDMA/TCP 数据面，见第 2~5 章）与 **Mooncake Store**（分布式 KV 池，见第 7~10 章）。它们解决的是"数据放对位置之后，如何在角色 / 并行度 / 介质之间搬动、索引和分发"的问题：

| 子系统 | 语言/形态 | 一句话职责 |
|---|---|---|
| mooncake-reshard | 纯 Python 库（无 C++、无独立 CMake target；靠 `python/CMakeLists.txt:39-45` 的 `install(DIRECTORY ... FILES_MATCHING PATTERN "*.py")` 把 `mooncake-reshard/python/mooncake/reshard/` 下的 `.py` 装进 wheel 的 `mooncake/reshard`） | 训练/推理并行度变化（TP/PP/EP/DP）时，权重与 KV cache 的"地址无关逻辑规划 → 运行时绑定 → 有界物理执行"三段式再分片契约 |
| mooncake-conductor | C++（cinatra `coro_http` + `coro_rpc`），附 C/C++/Python/Rust 四套客户端 | 订阅 vLLM/SGLang/Mooncake Store 的 KV 事件，维护全局前缀缓存索引表，向 cache-aware 路由器回答"这个前缀在哪个实例、哪个介质层有多少命中" |
| mooncake-p2p-store | Go + cgo（直调 `transfer_engine_c.h`） | 无中心的 checkpoint/模型大规模分发：etcd 存元数据，gold/replica 分片模型，GetReplica 成功后本机自动成为新数据源 |
| mooncake-rl | 目录里只有一个示例脚本 | 演示 rollout↔training 之间用 Store `put_tensor`/`get_tensor` 传样本；真正的 RL 数据面在 `python/mooncake/structured_object_store.py` |
| HiCache | 本仓库**没有实现代码** | SGLang 侧 L1(GPU)/L2(host)/L3(Mooncake Store) 三级缓存；仓库内只有设计文档、集成/部署文档、hugepage 估算脚本和拷自 SGLang 的 e2e 测试 |

一个必须一开始就说清楚的混淆点：`mooncake-store` 客户端里的 **LocalHotCache 不是 HiCache 的 L2**。它是 Store 客户端自己的本地热缓存（`mooncake-store/include/local_hot_cache.h:56`），用于加速对 Store 对象的重复读；HiCache 的 L2 是 SGLang 进程内的 host memory 池，由 SGLang 仓库管理。两者唯一的连接点是：HiCache 的 L3 storage backend（SGLang 侧 `--hicache-storage-backend mooncake`，见 `scripts/e2e/python/test_hicache_storage_mooncake_backend.py:205`）读写 Store 时，Store 客户端可能顺带用 LocalHotCache 加速（是否启用取决于部署配置，此处未在 HiCache 侧代码中逐条确认）。

`mooncake-reshard` 内部又分成四个互相独立的包（`mooncake-reshard/README.md:14-22`）：

- `contracts`：`ResourceManifest`/`PlacementManifest`/`RuntimeBindingManifest` 等结构 `Protocol`，只描述身份与生命周期，不绑定框架；
- `transfer_engine`：资源中立的物理批次、完成追踪、pending 资源隔离（quarantine）、注册租约；
- `weight`：模型权重的 placement / runtime binding / N-D 规划 / Store 快照 / TE 适配器；
- `kv_cache`：KV cache 的拓扑、序列化、逻辑规划与执行——**注意它与 weight 是两套执行器**，没有 ticket/drain 机制（见"易踩坑点"）。

## 目录结构速览

| 路径 | 职责 |
|---|---|
| `mooncake-reshard/README.md` | reshard 总览：四个子包职责、placement/binding/plan/execution 生命周期、in-flight 窗口语义（270 行，信息密度极高，必读） |
| `mooncake-reshard/python/mooncake/reshard/weight/types.py` | `ParallelRank`、`SplitAxis`/`ReplicatedAxis`/`OwnershipAxis`、`TensorDescriptor`、`PlacementFragment` |
| `mooncake-reshard/python/mooncake/reshard/weight/topology.py` | `ParallelTopology`：四轴尺寸 + 显式选中参与者（`world_size` 是选中数，不是乘积） |
| `mooncake-reshard/python/mooncake/reshard/weight/_planner/api.py` | 三个逻辑规划入口 |
| `mooncake-reshard/python/mooncake/reshard/weight/_planner/core.py` | `_plan_transfer`：N-D overlap 求解 + `PlanningLimits` fail-closed |
| `mooncake-reshard/python/mooncake/reshard/weight/_planner/contracts.py` | `TransferRegion`（紧凑 N-D 区域）、`PlanningLimits`、`LogicalTransferPlan`、`PlacementExecutorPlan` |
| `mooncake-reshard/python/mooncake/reshard/weight/_planner/binding.py` | `bind_logical_transfer_plan`：逻辑计划绑定运行时快照 → `TransferPlan` |
| `mooncake-reshard/python/mooncake/reshard/transfer_engine/executor.py` | 资源中立批次执行器；`_execute_batches` 实现 `max_inflight_batches` 有界在途窗口（PR #4628 核心） |
| `mooncake-reshard/python/mooncake/reshard/transfer_engine/completion.py` | 完成语义：`PendingTransferManager`、composite ticket、unknown/interrupted 隔离与 drain |
| `mooncake-reshard/python/mooncake/reshard/transfer_engine/registration.py` | `registered_targets/sources` 注册上下文、`BufferRegistrationLease`、ContextVar 注册帧与 pending handoff |
| `mooncake-reshard/python/mooncake/reshard/transfer_engine/contracts.py` | `TransferBatchRange`/`TransferBatch`/`TransferBatchReceipt` |
| `mooncake-reshard/python/mooncake/reshard/lifetime.py` | `AllocationFence`/`AllocationLifetimeToken`/`AllocationTokenSet`/`TerminalTransferState` |
| `mooncake-reshard/python/mooncake/reshard/weight/_te/reader.py`、`sink.py` | 权重读/写适配器：绑定复核 → N-D 降级 → 有界执行 |
| `mooncake-reshard/python/mooncake/reshard/weight/_te/execution.py`、`batching.py` | `iter_endpoint_batches` 按 endpoint 分组批量；各上限统一校验；N-D → 物理 segment |
| `mooncake-reshard/python/mooncake/reshard/weight/_store/store.py`、`writer.py`、`load.py` | `WeightStore`：快照写入（begin_weight_snapshot/commit）与恢复（plan_load/load → get_into_ranges） |
| `mooncake-reshard/python/mooncake/reshard/kv_cache/executor.py` | KV cache 侧独立执行器（同步 `batch_transfer_sync_write` + fail-closed uncertain 标志） |
| `mooncake-integration/transfer_engine/transfer_engine_py.cpp` | 原生层：`ScatterTransferTicket` 三态、`submitScatterWithTicket`、pybind 绑定 |
| `mooncake-conductor/src/main.cpp` | conductor 进程入口（75 行）：ParseConfig → EventManager → HTTP/RPC → 信号等待 |
| `mooncake-conductor/src/kvevent/event_manager.cpp` | `EventManager`（1300 行）：订阅生命周期、HTTP 路由、锁序（含死锁注释） |
| `mooncake-conductor/src/kvevent/event_handler.cpp` | `KVEventHandler`（1131 行）：vLLM/SGLang/Mooncake 三类事件归一化、stale 全量撤回 |
| `mooncake-conductor/src/kvevent/conductor_service.cpp` | HTTP 与 RPC 共享的服务层（Query/Register/Unregister/GlobalView/ListServices） |
| `mooncake-conductor/src/zmq/zmq_client.cpp` | ZMQ SUB/DEALER 客户端：序列号追踪、replay 补洞、`MarkStale` |
| `mooncake-conductor/src/prefixindex/prefix_indexer.cpp` | `PrefixCacheTable`：四层介质命中查询、`write_order` 容量淘汰 |
| `mooncake-conductor/src/prefixindex/hash_strategy.cpp` | vLLM v1 / SGLang / SGLang-bigram 块哈希策略与惰性 `HashChain`（1166 行） |
| `mooncake-conductor/include/conductor/prefixindex/prefix_indexer.h` | `ContextState`/`BlockPresence`/`CacheHitResult`/`PrefixCacheTable` 声明 |
| `python/mooncake/conductor.py`、`mooncake-conductor/rust/src/lib.rs` | Python / Rust 客户端封装（Rust 是 FFI 静态链接 C++ core，不是独立实现） |
| `mooncake-p2p-store/src/p2pstore/core.go` | `P2PStore` 本体：Register/GetReplica/performTransfer/updatePayloadMetadata |
| `mooncake-p2p-store/src/p2pstore/metadata.go` | etcd 元数据 schema 与 ModRevision CAS 更新 |
| `mooncake-p2p-store/src/p2pstore/registered_memory.go` | 引用计数的本地内存注册（4GB 分块并发注册） |
| `mooncake-p2p-store/src/example/p2p-store-example.go` | trainer/inferencer 两角色示例 |
| `mooncake-rl/examples/rl_samples.py` | RL 数据流 dummy 示例（450 行，训练/rollout 逻辑全 mock） |
| `python/mooncake/structured_object_store.py` | `MooncakeBundleTransfer`/DataProto 结构化对象——实际 RL 数据面（7390 行） |
| `docs/source/design/hicache-design.md` | HiCache 唯一权威设计文档（122 行） |
| `docs/source/design/conductor/conductor-architecture-design.md` | conductor 架构文档（注意：部分内容明显滞后于代码，见"易踩坑点"） |
| `docs/source/design/mooncake-reshard/model-weight-reshard-planner.md` | 规划器设计：N-D region、四并行轴语义、fail-closed |
| `docs/source/design/p2p-store.md` | P2P Store 设计与 Go API 说明（注意签名已过时） |
| `mooncake-store/include/local_hot_cache.h`、`src/client_service.cpp` | Store 客户端热缓存及其写回路径（PR #4764） |
| `scripts/check_hicache_hugepage_requirements.py` | HiCache L2 hugepage 容量估算（Docker 内安装为 `mooncake-hicache-sizing`，见 `docker/mooncake.Dockerfile:130`） |

## 核心数据结构

### reshard：逻辑层（地址无关）

**`ParallelTopology`（`weight/topology.py:34-45`）**：`tp_size/pp_size/ep_size/dp_size` 加 `participants: tuple[TopologyParticipant, ...]`，以及一个 `topology_id`。文档注释明确写了设计意图：`world_size` 是**显式参与者个数**，不能由四轴乘积推导，因为框架可能把 TP 与 EP 映射到同一批 worker，且一个 placement 可以只选一个 DP 副本而保留运行时声明的 `dp_size`。这是理解后面 DP 规划的钥匙。

**`TensorDescriptor` / `SplitAxis` / `ReplicatedAxis` / `OwnershipAxis`（`weight/types.py:51-201`）**：三种轴语义分别是"逻辑切分（只有 TP/EP 允许）"、"完整副本"、"所有权路由"；注意 `shard_dims` 是 `TensorDescriptor` 的字段，不是 fragment 的字段。`PlacementFragment`（`weight/types.py:204-278`）是一个参与者在全局张量里的一个逻辑片段：`tensor_id` + `global_offset` + `local_shape` + `nbytes` + `rank`，另有可选的 `pipeline_stage_id` 与 `aliases`（别名组）。

**`PlanningLimits`（`weight/_planner/contracts.py:97-113`）**：三个上限 `max_transfer_regions=1_000_000`、`max_segments_per_region=1_000_000`、`max_total_lowered_segments=10_000_000`，`__post_init__` 里全部要求正整数。它是"紧凑 N-D 表示不得在绑定/执行阶段被展开成无界操作"的闸门。

**`TransferRegion`（`weight/_planner/contracts.py:116-355`）**：整个 reshard 最核心的几何对象。一个 region 表示源片段与目标片段的一个 N-D 盒交集，字段含义：

- `overlap_offset` / `overlap_shape`：交集在**全局张量坐标系**里的位置和形状（`__post_init__` 用 `_box_contains` 校验它落在两侧片段内，`contracts.py:170-183`）；
- `source_base_offset` / `target_base_offset`：该交集在两侧片段内的**字节**起始偏移；
- `inner_bytes`：最内层连续拷贝的字节数；
- `outer_loop_counts` / `source_strides` / `target_strides`：外层循环的重数与此侧字节步长。

`segment_count = prod(outer_loop_counts)`（`contracts.py:257-259`），`repeat` 就是它（`contracts.py:277-279`）。`iter_segments(max_segments)`（`contracts.py:323-355`）用里程表式进位把 N-D 循环展开成 `(source_offset, target_offset, inner_bytes)` 三元组流；`max_segments` 在展开**之前**就拒绝超限。`validate_bounds()`（`contracts.py:297-321`）用 `(count-1)*stride + inner_bytes` 判越界——注意这个公式假设最内层连续，外层只需校验最后一个元素。

**`LogicalTransferPlan`（`weight/_planner/contracts.py:463-654`）**：只含 `resource_id/revision/source_placement|source_manifest/target_placement/tensors/operations/planning_limits/source_executors/target_executors`。`__post_init__` 做了几件很硬的事：

1. 源侧"恰好一种 provenance"：`(source_placement is None) == (source_manifest is None)` 即必须且只能有一个（`contracts.py:503-506`）；
2. 两侧 `resource_id/revision` 必须一致，`weight_generation` 必须一致（`contracts.py:517-541`）；
3. 逐 operation 复核 segment 预算（`contracts.py:542-552`）；
4. `validate_source_manifest_snapshot()`（`contracts.py:593-616`）：若源是 Store 快照，则逐 fragment 与 manifest 比对，防止"计划被重定向到另一个 Store 对象"；
5. `__getstate__/__setstate__` 剔掉 `_operation_views` 缓存再在反序列化时重建（`contracts.py:644-654`）——这是为了让 pickle 往返后仍然执行完整校验；
6. 构造末尾还会调用 `_validate_logical_target_coverage` 与 `validate_source_placement_snapshot`（`contracts.py:570-575`），所以反序列化同样会重新验证目标覆盖与源 placement 快照。

**`TransferPlan`（`weight/_planner/bound_contracts.py:460-520`）**：绑定后的计划。与 `LogicalTransferPlan` 的区别是 operation 的 source/target 已是 `BoundWeightFragment`（带 `address/storage_address/storage_nbytes/storage_offset_bytes/worker_id/endpoint/device/lease_generation`）或 `StoredFragmentSnapshot`，并且携带 `ExecutorTransferPlan` 投影。**它没有 `execute()`**——这是刻意的：执行必须在框架 allocation guard 重新 pin 之后进行（`docs/source/design/mooncake-reshard/model-weight-reshard-planner.md:105-120`）。

### reshard：物理层

**`TransferBatchRange`（`transfer_engine/contracts.py:17-68`）**：一段共享同一源/目标 backing allocation 的片段集合。四个 base/capacity 必须为正，且 `capacity > _MAX_U64 - base` 直接拒绝溢出；每个 `size` 必须落在两侧 offset+capacity 内。它存在的理由是：原生 `submitScatter` 需要"本地缓冲区 + 容量 + 偏移向量"，只有保留 allocation 边界才能把 N-D 降级结果安全地交给它。

**`TransferBatch`（`contracts.py:71-129`）**：`endpoint` + 扁平地址三元组 + 可选 `ranges`。构造时会做一件很妙的交叉校验：如果给了 `ranges`，则把 ranges 展开后的 `base+offset` 序列与扁平 `source_addresses/target_addresses/sizes` **逐项比对**，不一致就抛错（`contracts.py:108-129`）。`from_ranges`（`contracts.py:131-155`）是唯一推荐构造方式，它负责生成两份一致的数据。

**`_CompletionTicket`（`completion.py:16-21`）**：只需要 `status` 属性与 `drain(timeout_ms)`。Python 层不依赖原生具体类型，只依赖这个结构契约，因此测试里可以用假 ticket。

**`_PendingTransfer`（`completion.py:192-203`）**：一条隔离记录，字段包括 `engine`、`engine_identity`、`ticket`、`registrations: set[int]`、`resources`、`allocation_tokens`、`restart_required`、`terminal_state`、`resources_handed_off`、`draining`。所有 pending 记录放在模块级字典 `_pending_transfers`（`completion.py:206-208`），用一把全局 `Lock` 保护——注意这是**进程级**而非"每 engine 一把锁"。

**`AllocationLifetimeToken` / `AllocationTokenSet`（`reshard/lifetime.py:64-111`）**：框架侧的 allocation pin 抽象，只有 `fence` 与 `release_after_terminal(state)` 两个方法。`AllocationTokenSet` 是三态：正常、`_pending`（已交给隔离路径）、`_released`。`release_after_terminal` 在 pending 或已释放时是 **no-op**（`lifetime.py:107-111`），这是防止"隔离中的 token 被 finally 提前释放"的关键。

### conductor：索引层

**`ContextKey`（`prefixindex/types.h:15-22`）**：`(tenant_id, model_name, lora_name, block_size)`。它不含 `cache_salt` 与 `instance_id`：salt 进入哈希链，instance 进入 `ContextState::instance_ranks`。`std::hash<ContextKey>` 是手工组合的 64 位混合（`types.h:101-115`），所以 `ContextKey` 可用作 `unordered_map` 键。

**`BlockPresence`（`prefix_indexer.h:26-36`）**：一个前缀（`ProjectedPrefix`，即 64 位投影哈希）在四层介质上的 owner 集合：`device_owners` / `cpu_local_owners`（都是 `EngineOwner`）、`cpu_share_owners`（`SharedObjectOwner`）、`disk_owners`（`TierOwner = variant<EngineOwner, SharedObjectOwner>`）。`Empty()` 决定条目能否删除。

**`ContextState`（`prefix_indexer.h:43-63`）**：每个 `ContextKey` 一份，含 `mutable std::shared_mutex mutex`、不可变的 `HashProfile profile`、`instance_ranks: map<instance, set<dp_rank>>`、`blocks: unordered_map<ProjectedPrefix, BlockPresence>`、`max_blocks`、`evicted_by_capacity`。另外两个字段是为容量淘汰服务的：`write_order: list<ProjectedPrefix>`（新在前）与 `order_pos`（前缀 → 链表迭代器）。头文件里那句 **"Lock order is global context-map mutex, then this mutex. Code holding this mutex must never reacquire the global mutex."**（`prefix_indexer.h:48-49`）是读这份代码的第一条纪律。

**`CacheHitResult` / `RankCacheHitResult`（`prefix_indexer.h:65-84`）**：按实例聚合的结果，四层各一个 token 数，外加 `dp`（rank→device 命中）与 `rank_matches`（rank→四层结构）。`longest_match_tokens` 取 `disk`（最深层），因为"可复用的前缀"由最深可达层决定。

**`HashChain`（`hash_strategy.h:26-40`）**：惰性哈希链。`At(i)` 会把 0..i 全部算出来并缓存；`BlockCount()` 的语义由策略决定（vLLM 只算完整块，SGLang 含尾部不完整块）。`CreateChain` 借用 `token_ids` 的 span，调用方必须保证其生命周期长于链。

### p2p-store：元数据层

**`Payload` / `Shard` / `Location`（`metadata.go:46-63`）**：etcd 键 `mooncake/checkpoint/<name>`（`core.go:32`），值是 JSON：

```
{ name, size, size_list[], max_shard_size, shards: [ { size, gold: [loc], replica_list: [loc] } ] }
```

`gold` 是种子节点（第一个 `Register` 的调用方）的原始副本，永不自动清除；`replica_list` 是克隆出该分片的节点。`Shard.Count() = len(replica_list)+len(gold)`（`metadata.go:73-75`）。`GetLocation(0)` 随机挑一个源，`GetLocation(i>0)` 按顺序重试（`metadata.go:65-98`）——这就是"多源 + 重试"的全部策略。

**`Catalog`（`catalog.go:19-62`）**：进程内、带互斥锁的 `map[name]CatalogParams`，记录本机打开过哪些 payload（`IsGold` 区分是 Register 还是 GetReplica 打开的）。它的作用不是缓存数据，而是**所有权账本**：`Unregister/DeleteReplica` 只能对账本里的名字操作，且用它拿到 addr/size 去做内存反注册。

**`RegisteredMemory`（`registered_memory.go:22-38`）**：`[]BufferHandle{addr,length,refCount}`。同一个 (addr,length) 重复 Add 只加引用计数；相交但不相同的区间直接 `ErrAddressOverlapped`；真正注册时按 `MAX_CHUNK_SIZE = 4GB`（`core.go:31`）切块**并发**调 `registerLocalMemory`，任一失败则回滚已成功块（`registered_memory.go:70-124`）。

## 关键流程剖析

### 流程一：一次权重再分片从规划到落地（plan → bind → read）

调用链：

```
plan_placement_transfer             (weight/_planner/api.py:16)
  ├─ _collect_placements            (weight/_planner/core.py:50)
  ├─ _plan_transfer                 (weight/_planner/core.py:146)
  └─ _logical_transfer_plan         (weight/_planner/core.py:327)
bind_logical_transfer_plan          (weight/_planner/binding.py:427)
MooncakeTransferEngineReader.execute(weight/_te/reader.py:96)
  ├─ validate_* 系列                 (weight/_te/execution.py)
  ├─ acquire_weight_lifetime_tokens  (weight/_lifetime.py:51)
  └─ _execute_reserved              (weight/_te/reader.py:246)
       ├─ iter_endpoint_batches     (weight/_te/execution.py:26)
       ├─ registered_targets        (transfer_engine/registration.py:579)
       └─ TransferSubmission.execute_batches (transfer_engine/executor.py:120)
            └─ MooncakeTransferEngineExecutor._execute_batches (executor.py:355)
                 └─ engine.submit_scatter_read(...)  (native binding)
```

**第 1 步：逻辑规划。** `plan_placement_transfer` 先核对源/目标 `resource_id`、`revision`、`weight_generation` 三者相等（`api.py:34-39`），再收集两侧的 tensor 目录与逻辑片段。`_plan_transfer` 的核心是一个"以目标片段为驱动"的双层循环：

1. 先按 `geometry_key` 给每个 tensor 的源片段建候选索引（`core.py:202-218`）；
2. 对每个目标片段，查候选、按 DP 规则过滤、求 N-D 交集盒（`core.py:222-277`）；
3. `boxes_exactly_cover(...)` 要求交集盒**恰好完整覆盖**目标片段，否则 fail-closed（`core.py:279-286`）——这保证了"不会悄悄漏拷贝"；
4. 每个交集生成一个 `TransferRegion`，并立即检查 region 数与 segment 预算（`core.py:295-309`）；
5. 最终按 `(target.fragment_id, target_base_offset, source.fragment_id, source_base_offset)` 排序（`core.py:311-318`），使得计划是确定性的、可复现的。

DP 的两条分支值得单独记住（`core.py:236-261`）：若张量声明了 `OwnershipAxis(dp)`，直接找声明 owner 的源片段；否则按 `source_dp_by_target_dp` 做 **round-robin** 映射（`core.py:194-201`），即 `source_dp = source_dp_ranks[target_dp % len]`。

**第 2 步：绑定。** `bind_logical_transfer_plan`（`binding.py:427-557`）先重新校验逻辑覆盖、tensor 快照、fragment 快照，再把每个 `PlacementFragment` 换成运行时 `BoundWeightFragment`；随后 `_validate_target_physical_ranges(...)` 检查物理别名/区间（`binding.py:509-516`，注释解释了"同一 allocation 只有在证明完整 alias group 与同一 lease scope 后才可共享"），最后生成源/目标 executor 投影。

**第 3 步：读适配器执行。** `MooncakeTransferEngineReader.execute` 的路子很讲究：

1. 先做一次**纯诊断性**的 `validate_manifest_pair`（`reader.py:120-122` 的注释写明：真正的校验会在 framework guard 返回新快照后再做一遍）；
2. 用 `planned_executors_for_binding` + `select_worker_executors` 选出本地目标 executor；若为空则直接返回 `()`（`reader.py:123-129`）；
3. 由目标 operation 反推需要哪些源 executor 与 fragment（`executor_requirements_for_operation_indices`，`execution.py:362-395`；它同时检查"每个 fragment 确实归该 executor 所有"与"索引被完整覆盖"）；
4. 进入 `with self.transfer_executor.submission()` —— 这一步在 `PendingTransferManager._reserve_submission` 里**预约整个原生引擎**，有 pending 或已有活动 submission 都会直接抛错（`completion.py:454-481`）；
5. 依次 `acquire_weight_lifetime_tokens` 拿源侧、目标侧 allocation pin；两次都成功后才拼成一个合并的 `AllocationTokenSet`（`reader.py:176-200`）；
6. `_execute_reserved` 里再做一次 `validate_scoped_executor_snapshot`，并逐 operation 用 `same_runtime_snapshot` 校验片段未被替换、`validate_registration` 校验注册租约、`operation.validate_bounds()` 校验几何（`reader.py:359-399`）；
7. 在 `registered_targets` 上下文里调用 `submission.execute_batches(...)`，把 `max_inflight_batches` 透传下去（`reader.py:411-434`）；
8. `finally` 里按 `terminal_state` 终结 token 集合（`reader.py:231-244`）。`terminal_state` 的取值逻辑是：正常完成 `COMPLETED`；`TransferCompletionFailedError` → `FAILED_DRAINED`；其他异常若 `submission.physical_io_started` 也记 `FAILED_DRAINED`，否则保持 `ABORTED`（`reader.py:222-230`）。

**第 4 步：N-D 降级为物理批次。** `iter_endpoint_batches`（`execution.py:26-50`）做两件事：

- 端点排序：`for endpoint in sorted(...)`，保证跨端点提交顺序确定；
- 端点内排序键是 `(target.address + target_offset, source.address + source_offset)`（`execution.py:40-43`）。这个键让**落在同一目标 allocation 上的 operation 彼此相邻**，于是同一 batch 内各 `TransferBatchRange` 的目标地址也是递增的，便于原生 scatter 按目标局部性提交。注意 range **不会跨 operation 合并**：`_range_from_fragments` 每次只处理一个 `(source fragment, target fragment)` 对的段列表。

`iter_transfer_batches`（`batching.py:9-71`）遍历 operation 的 `iter_segments`，累加 `(source_offset, target_offset, size)`，每累计到 `max_batch_operations` 就 `_range_from_fragments(...)` 并 yield 一个 `TransferBatch.from_ranges(...)`。`_range_from_fragments`（`batching.py:74-93`）把 fragment 的 `storage_address/storage_nbytes/storage_offset_bytes` 与段内 offset 相加，得到 allocation 相对地址——这就是"保留 backing allocation 边界"的落点。注意：**一个 operation 可以被切成多段、跨多个 batch**，但一个 range 永远属于同一个 (source fragment, target fragment) 对。

**第 5 步：提交。** `_prepare_batch`（`executor.py:239-343`）把 `TransferBatch` 翻译成原生参数元组。方向决定"谁是 local"：

- READ：local = target（本机接收），remote = source（`executor.py:278-288`）；
- WRITE：local = source（本机发送），remote = target（`executor.py:289-299`）。

参数顺序是 `(endpoint, local_base_addresses, local_capacities, remote_base_addresses, remote_capacities, local_offsets, remote_offsets, lengths)`，与 `submitScatterWithTicket` 的 C++ 签名一一对应（`transfer_engine_py.cpp:149-158`）。有 `ranges` 且引擎提供了 `scatter_transfer_sync_*_with_ticket`（或在 `require_async=True` 下）时走 scatter 入口；否则退回 `batch_transfer_sync_read/write` 扁平入口（`executor.py:267-334`）。

`WeightStore.load()` 是另一条并行的落地路径：它不经过 TE reader，而是用 `get_into_ranges` / `prepare_get_into_ranges_snapshot` 直接从 Store 拉进目标绑定（`weight/_store/load.py:233, 327`；`weight/_store/backend.py:118-156`）。

### 流程二：有界在途窗口 `_execute_batches`（PR #4628 核心）

这是整章最需要逐行读的函数（`transfer_engine/executor.py:355-525`）。

**入口分叉。** `max_inflight_batches` 必须是正整数，否则 `ValueError`（`executor.py:363-364`）。等于 1 时**完全退化**为逐批同步：

```python
return tuple(submission.execute_batch(batch, direction) for batch in batches)
```

（`executor.py:365-368`）。这条路径允许扁平 batch；而大于 1 时，函数在**消费输入之前**就检查引擎是否具备 `submit_scatter_read/write`，没有就抛 `TransferEngineError`（`executor.py:372-377`），并且 `_prepare_batch(require_async=True)` 会拒绝没有 `ranges` 的 batch（`executor.py:259-262`）与缺失的 `submit_method`（`executor.py:267-271`）。这是明确的 **fail-closed**：绝不静默降级到 `batch_transfer_sync_*`。

**窗口主循环。** 状态是几个变量（`executor.py:379-385`）：`published: deque[_PublishedBatch]`、`receipts: list[Receipt|None]`、`unresolved: list[_CompletionTicket]`、`first_failure`、`interruption`、`deferred_error`、`unknown_cause`。

循环条件写成了"全部清空才继续"（`executor.py:414-420`）：

```python
while (deferred_error is None and first_failure is None
       and not unresolved and interruption is None):
```

每轮先做背压：`if len(published) >= max_inflight_batches: drain_published(published.popleft())`（`executor.py:421-422`）。所以**窗口的上界恰好是 `max_inflight_batches`**：第 N+1 次提交前必定先 drain 最旧的一个。drain 后若出现任何失败信号，立即 `break`（`executor.py:423-429`）。

接着取下一批：`next(batch_iterator)` 的 `StopIteration` 是正常结束；其他 `BaseException` 记为 `deferred_error` 并 break（`executor.py:430-436`）。注意这里**不关闭**调用方的迭代器——`README.md:160-164` 明确说 `execute_batches` 只是借用，提前退出时资源由调用方负责关闭。

`_prepare_batch` 失败同样记 `deferred_error`（`executor.py:437-445`）。**顺序很关键**：先 `receipts.append(None)` 占位，再提交（`executor.py:446-447`），这样 receipt 的下标天然等于提交顺序。

**提交与探针。** `submission._mark_physical_io_started()` 之后调用 `submit_method(*prepared.arguments)`（`executor.py:454-456`）。三条异常路径：

| 情况 | 记录 | 后果 |
|---|---|---|
| `Exception`（提交抛错，且没有 ticket） | `_UndrainableCompletionUnknownTicket()` | `unknown_cause`，restart-required |
| `BaseException`（KeyboardInterrupt 等） | 同上 | `interruption` |
| 返回 ticket 但 `drain`/`status` 属性探针失败 | `_UndrainableCompletionUnknownTicket(ticket)` | `unknown_cause`，restart-required |

（`executor.py:455-482`）。探针必须在**把 ticket 放进窗口之前**做（`executor.py:466` 的注释："Probe both attributes before admitting the ticket to the window"），否则窗口里会混入无法 drain 的元素。

注意 `_UndrainableCompletionUnknownTicket` 的 `restart_required` 是**类属性且恒为 True**（`completion.py:85-87`），所以"探针失败"即使保住了原 ticket，仍然要求重启引擎。

**drain 的三路分流。** `drain_published`（`executor.py:387-412`）：

- `_CompletionUnknown` → `unresolved.append(error.ticket)`，首次记录 `unknown_cause`；
- `_CompletionWaitInterrupted` → `unresolved.append(error.ticket)`，首次记录 `interruption`；
- 正常返回但 `result != 0` → 记 `first_failure = (prepared, result)`；
- 返回 0 → 生成 receipt 填回 `receipts[item.index]`；**连 receipt 构造本身抛异常**也要接住记 `deferred_error`（`executor.py:408-412`）。

任何出口都会先 `break` 主循环，然后进入尾部排空 `while published: drain_published(published.popleft())`（`executor.py:487-488`）——**所有已发布的 ticket 在返回一个"已知失败"之前一定都被 drain 过**。这正是 `README.md:149-150` 的承诺。需要注意 `first_failure` 记录的是"最先被 drain 到的失败"，而 drain 顺序是提交顺序，所以它是提交序最靠前的失败；但尾部排空阶段若再出现 unknown，最终会以 unknown 上报（见下）。

**最终异常优先级。** 顺序是硬编码的（`executor.py:490-524`）：

1. 如果 `deferred_error` 是 `BaseException` 而非 `Exception`，且还没有 `interruption`，把它升级为 `interruption`（`executor.py:490-495`）——例如迭代器抛出 `KeyboardInterrupt`；
2. `unresolved` 非空 → 把全部 ticket 合成一个 `_composite_completion_ticket(unresolved, terminal_failed=(first_failure or deferred_error))`；有 interruption 就先抛 `TransferCompletionInterrupted`，否则抛 `TransferCompletionUnknownError`（`executor.py:496-512`）；
3. 否则 `first_failure` → `TransferCompletionFailedError`（`executor.py:513-517`）；
4. 否则 `deferred_error` 原样抛出（`executor.py:518-519`）；
5. 最后检查 `receipts` 里没有 `None`，否则抛"completed batch is missing its receipt"（`executor.py:520-524`）。

优先级汇总：**interruption > unknown > failed > deferred_error**。测试 `test_execute_batches_unknown_outweighs_known_failure`（`mooncake-reshard/tests/reshard_transfer_engine/test_executor.py:558-574`）固化了"unknown 压过 failed"；`test_execute_batches_preserves_stream_interruption_with_pending_ticket`（同文件 `:261-281`）固化了 interruption 的保留。

**隔离（quarantine）。** `_raise_completion_unknown` / `_raise_completion_interrupted`（`executor.py:527-563`）做同一件事：先 `_retain_pending_ticket(ticket)` 拿到一个 uuid，再 `_handoff_active_registration_frames(pending_transfer_id)` 把当前上下文里所有已注册缓冲区与 allocation token 转交给这条 pending 记录，最后抛出带 `pending_transfer_id` 与 `engine_identity` 的异常。若 `_handoff_active_registration_frames` 返回 False（当前上下文没有可交接的帧），就调用 `_seal_pending_resource_handoff` 把记录标记为"交接完成"（`executor.py:565-573`）。

注册帧是一条 **ContextVar 栈**（`transfer_engine/registration.py:158-183`），`registered_targets` 在进入时压帧、退出时弹帧，并在捕获到 `TransferCompletionUnknownError`/`TransferCompletionInterrupted` 且 `engine_identity` 匹配时调用 `_handoff_pending_resources`（`registration.py:579-645`）。这意味着：**隔离交接依赖异常在注册上下文内被处理**。`_raise_completion_unknown/_interrupted` 在当前 ContextVar 里找不到可交接帧时，会用 `_seal_pending_resource_handoff` 把记录封成"空交接"（`executor.py:565-573`）：`drain_pending_transfer` 不会因此报错，但那条 pending 记录里没有 registrations/tokens，隔离路径接管不到任何缓冲区。写跨线程/跨 context 的异步包装时要格外小心——要么在同一上下文内捕获，要么显式 `retain_pending_resources` 补交。

**composite ticket 的共享超时预算。** `_CompositeCompletionTicket.drain(timeout_ms)`（`completion.py:129-148`）先算出唯一 deadline：`deadline_ns = monotonic_ns() + timeout_ms*1e6`；随后对每个子 ticket，只有当其 `status` 仍为 `COMPLETION_UNKNOWN` 时才 `drain(remaining_ms)`，`remaining_ms = max(0, (deadline - now)//1e6)`。因此**后来的子 ticket 只能拿到剩余预算，甚至可能拿到 0**（0 在原生侧对应一次非阻塞 poll）。任一子 ticket 抛异常时，仍会把其余子 ticket 走完，记录 `COMPLETION_UNKNOWN`，最后再抛出第一个异常（`completion.py:142-147`）。`status` 属性只做聚合、不触发 drain（`completion.py:125-127`）；`_aggregate` 的规则是"有 UNKNOWN 就是 UNKNOWN；否则只要 `terminal_failed` 或任一非 COMPLETED 就是 FAILED_DRAINED"（`completion.py:150-156`）。

**为什么 `_UndrainableCompletionUnknownTicket` 要持有原 ticket。** 它的实现只有一行关键点：`self._owner = owner`（`completion.py:89-92`），注释写明"Keep the original ticket alive so its native destructor cannot run while quarantined. This placeholder never delegates drain to it."。原因是原生 `ScatterTransferOperation::Impl` 的析构函数会 `wait()` 到物理完成（`mooncake-transfer-engine/src/transfer_engine.cpp:1354`）；如果不持有 owner，Python 侧一旦丢弃引用就可能触发这个阻塞析构。placeholder 自身 `drain()` 直接返回 `"COMPLETION_UNKNOWN"`（`completion.py:94-95`），所以它永远不会"假装成功"。

**drain_pending_transfer 的收尾顺序。** `PendingTransferManager.drain_pending_transfer`（`completion.py:521-656`）：

1. 前置检查：`resources_handed_off` 必须为真、不能已被别的线程 drain（`draining`），否则抛错（`completion.py:529-540`）；
2. `ticket.drain(timeout_ms)`；若抛 `Exception` 直接返回 `"COMPLETION_UNKNOWN"`（**不**抛给调用者，`completion.py:544-548`）；
3. 把仍持有的 registrations **按地址降序**排序后逐个 `unregister_memory`（`completion.py:559-596`）。成功的从集合里移除，非零结果收集成 `failures` 最后一起抛；反注册抛异常则把 ticket 换成 `_UndrainableCompletionUnknownTicket(current.ticket)` 并置 `restart_required=True`（`completion.py:575-592`）。降序是代码里显式写的 `sorted(owned, reverse=True)`；作者意图（便于分配器合并）**此处未完全确认**；
4. 逐个释放 allocation token，**从尾部开始**（`current.allocation_tokens[-1]`），并在释放后复核"尾部仍是它"（`completion.py:603-629`）。这样即使中途失败，已释放的 token 不会在下次重试时被回放；
5. 全部成功才 `_pending_transfers.pop(...)`（`completion.py:648-650`），并在 `finally` 里清掉 `draining` 标志（`completion.py:651-655`）。

**逐批路径的收尾。** `_execute_batch`（`executor.py:206-237`）走 `_batch_transfer_with_completion_fence`：`_CompletionUnknown` → `_raise_completion_unknown`，`_CompletionWaitInterrupted` → `_raise_completion_interrupted`，其他异常包装成 `TransferEngineError`，`result != 0` 抛 `TransferCompletionFailedError`。在 `TransferSubmission` 层，前两种异常会把 submission 置为 `POISONED`（`executor.py:111-118, 135-147`），之后任何 `execute_batch(es)` 都会被拒绝（`executor.py:104-110`）。

### 流程三：原生 Scatter ticket 的三态与取消路径

**Python 侧看到的三态。** `ScatterTransferCompletionStatus`（`transfer_engine_py.cpp:67-71`）：`COMPLETED=0`、`FAILED_DRAINED=-1`、`COMPLETION_UNKNOWN=-2`。Python 的 `_completion_status_name` 为了兼容枚举对象/整数/字符串，按 `.name` → `int(status)` → `str(status)` 三级降级（`completion.py:211-230`）。

**ticket 的 drain。** `ScatterTransferTicket::drain`（`transfer_engine_py.cpp:98-123`）持自身互斥锁，`drained_` 后直接返回缓存状态；否则把毫秒转成纳秒（并对 `nanoseconds::max` 做饱和），调 `operation_->waitFor(...)`。**关键映射**：`result.IsClock()`（超时）→ 返回 `COMPLETION_UNKNOWN`，且**不置 `drained_`、不 reset `operation_`**（`transfer_engine_py.cpp:114-116`）。也就是说"超时"不等于"失败"，DMA 可能仍在跑，ticket 必须继续可 drain。非超时则 `ok() ? COMPLETED : FAILED_DRAINED`，置 `drained_` 并 `operation_.reset()`。

`IsClock()` 的来源是 `Status::Code::kClock`（`mooncake-transfer-engine/include/common/base/status.h:149`），而 `waitFor` 在 deadline 到达时返回 `Status::Clock("scatter transfer wait timed out")`（`transfer_engine.cpp:1379-1380`）。

**提交。** `submitScatterWithTicket`（`transfer_engine_py.cpp:149-224`）做了三层校验：各数组长度一致、本地 allocation 不溢出地址空间、每个 range 内 fragment 数组对齐；任何一层失败都返回一个 `terminal(FAILED_DRAINED)` 的 ticket（**不是抛异常**），所以 Python 侧看到的是"可 drain 的失败"而不是"提交未知"。校验通过后构造 `ScatterTransferRange` 并 `engine->submitScatter(ranges)`（`transfer_engine_py.cpp:221-223`）。

`ScatterTransferRange` 的字段（`transfer_engine.h:190-202`）与 Python 的 `TransferBatchRange` 一一对应：`opcode / remote_segment / remote_base_offset / remote_size / local_buffer / local_capacity / local_offsets / remote_offsets / lengths`。

**同步入口 vs 异步入口。** 两个 `scatter_transfer_sync_*_with_ticket` 都在 C++ 内部先 drain 再返回 ticket（`transfer_engine_py.cpp:685-707`），超时预算写成 `transfer_timeout_nsec_ + submission.total_length` 纳秒（即"每字节额外 1ns"的经验预算），再 `nanosToCeilingMillis`。而 `submit_scatter_read/write`（pybind lambda，`transfer_engine_py.cpp:1825-1842`）只提交、不 drain，把 drain 责任交给 Python 窗口。

**GIL 与生命周期。** 所有 ticket 相关绑定都在入口 `py::gil_scoped_release`（`transfer_engine_py.cpp:695, 1805, 1810, 1814, 1834`），所以 drain 期间不阻塞其他 Python 线程。四个 scatter 入口都带 `py::keep_alive<0, 1>()`（`transfer_engine_py.cpp:1877, 1884, 1891, 1898`），含义是"返回值（ticket）活着就保住第 1 个参数（`self`，即 `TransferEnginePy`）"。**这条 keep_alive 是有界窗口能跨调用持有 ticket 的前提**：否则 Python 侧可能在 ticket 尚未 drain 时就把 engine 适配器回收掉。

**TENT 的取消保证。** `TransferEngine::submitScatter` 在 TENT 后端下用 `submitTransferRequiringPostSubmitCancellation`（`transfer_engine.cpp:1710-1712`），该入口在 `prepareSubmit` 里做两件事（`tent/src/runtime/transfer_engine_impl.cpp:2091-2173`）：

1. 关闭 failover（`max_failover_attempts=0`、`enable_auto_failover_on_poll=false`）——因为重试会让"取消"失去意义；
2. 逐个 request 要求 `transport_list_[type]->supportsCancellation()`，不支持就返回 `Status::NotImplemented("scatter transfer requires a cancellable transport")`；解析出 staging 参数的路由同样以 `NotImplemented("scatter transfer does not support staged TENT routes")` 拒绝（`tent/src/runtime/transfer_engine_impl.cpp:2180-2184`）。

调用方对这个 `NotImplemented` 的处理是"整批在发布前失败"：`transfer_engine.cpp:1739-1758` 用"task 0 是否越界"区分"未发布的 prepare 失败"与"已发布但传输失败"，未发布的情况直接 `failPending + finish()`。**所以未支持/走 staging 的路由会在 DMA 提交前被拒绝，从而保证"状态查询失败时一定存在取消路径"**（与 `README.md:110-113` 一致）。已发布后出错则 `requestAbort()` → `tent->cancelTransfer(batch_id_, i)`（`transfer_engine.cpp:1490-1505`）。

### 流程四：conductor 的事件摄取与查询

**订阅。** 进程入口 `main.cpp:47-61`：`ParseConfig(&http_port, &rpc_port)` → 构造 `EventManager` → `StartHTTPServer()` → `StartRPCServer()` → `Start()`（订阅静态配置里的服务）→ 等 SIGINT/SIGTERM → `Stop()`。默认 HTTP `13333`、RPC `13334`，`rpc_server_port=0` 表示禁用 RPC（`main.cpp:41-46`，`event_manager.h:172-175`）。

`EventManager::Start`（`event_manager.cpp:734-765`）对每个服务**单独加写锁**再 `SubscribeToService`，锁在服务之间释放，避免长时间阻塞 HTTP 处理器。`SubscribeToService`（`event_manager.cpp:824-927`）的顺序值得背下来：

1. `stopped_` 直接读（注释说明：调用方已持 `mu_`，不能再递归拿非递归 `shared_mutex`，`event_manager.cpp:826-828`）；
2. 校验配置；`svc_key = MakeServiceKey(instance_id, tenant_id, dp_rank)`；
3. 若在 `unregistering_` 或 `cleanup_quarantined_` 里，拒绝；
4. 已存在同 key：配置相同则幂等返回，不同则冲突；
5. 同 endpoint 已被别的 key 占用则冲突（`event_manager.cpp:856-860`）；
6. vLLM/SGLang 先 `indexer_.Register(RegistrationFromService(svc))`；Mooncake 发布者则校验 profile 绑定（`event_manager.cpp:881-899`）；
7. `ZMQClient::Start()` 失败时要回滚刚插入的 indexer 注册（`event_manager.cpp:902-912`）；
8. 四张表一起写：`subscribers_ / handlers_ / active_configs_ / active_endpoints_`（`event_manager.cpp:914-917`）。

**死锁规则。** `UnsubscribeFromService`（`event_manager.cpp:929-1008`）的注释是全仓最直白的并发说明：`HandleBatch` 会取 `mu_` 读锁；如果在持 `mu_` 时调用 `client->Stop()` 等 ZMQ 事件循环线程 join，而该线程正卡在 `HandleBatch` 里等 `mu_`，就死锁。所以流程被拆成：

1. 持写锁：把 `unregistering_.insert(svc_key)`（**保留 key，防止替换注册被误删**），`handler->MarkUnavailable()`；
2. 放锁：`client->Stop()` → `handler->WaitForIdle()`（`event_manager.cpp:958-964`）；
3. `handler->InvalidateEndpoint()` 撤回该端点在索引中的所有贡献；vLLM/SGLang 再 `indexer_.Unregister(...)`；
4. 清理失败 → 进 `cleanup_quarantined_` 并返回错误（**不会**退化成"部分删除"，`event_manager.cpp:975-983`）；成功才清四张表。

`Stop()`（`event_manager.cpp:767-822`）复刻同一模式：持锁 `MarkUnavailable` → 放锁停 HTTP/RPC → 收集 clients/handlers → 放锁逐个 `client->Stop()` → `handler->WaitForIdle()` → 置 `stop_complete_` 并 notify。幂等性由 `stopped_`/`stop_complete_` 加 `stop_cv_` 提供。

**事件归一化。** `KVEventHandler::HandleBatch`（`event_handler.cpp:295-330`）先 `BeginDispatch()`（`lifecycle_mu_` 下检查 `available_` 并 `++in_flight_`），用 `DispatchLease` 保证退出时 `--in_flight_`（`event_handler.cpp:268-283`），然后校验 `manager_->IsStopped()`、`metadata.publisher_kind/endpoint` 与可信注册一致，再按 kind 分派到 vLLM/SGLang/Mooncake 三条处理链。vLLM 批次还额外校验批次里的 `data_parallel_rank` 与注册的 `dp_rank` 一致（`event_handler.cpp:354-365`）。

vLLM 处理链对每个事件 `std::visit` 到 Stored/Removed/Cleared（`event_handler.cpp:387-401`）；单个事件失败只记 warning 并继续（**批次级别不是原子**）。SGLang 的 `HandleSglangStored` 会拒绝 `lora_id`（无法与可信注册校验，`event_handler.cpp:456-458`），然后把 block hash 直接投影成 `ProjectedPrefix{value=hash}` 并 `StoreEngine`（`event_handler.cpp:466-479`）。

Mooncake 共享对象事件（`kMooncake` 发布者）走另一套模型：`PoolObjectKey{source_endpoint, backend_id, tenant_id, object_key, tier}` → `PoolObjectBinding{context, projected prefix, connector_block_hash, owner, tenant_id, group_id, physical_components}`，由一个逻辑 SGLang 哈希可能对应多个物理对象（如 key/value 分量）。只有当一个逻辑键的**最后一个** `physical_component` 被移除时，才向索引 `RemoveShared`（`event_handler.cpp:665-687`）——这就是"物理分量引用计数"的写法。

**stale 撤回。** `OnSourceStale`（`event_handler.cpp:332-350`）不做增量修补，直接 `InvalidateEndpoint()`：`MarkUnavailable()` 后，vLLM/SGLang 用 `ClearEngine({..., tier = std::nullopt})` 清掉该 engine owner 在**所有本地层**的存在（`event_handler.cpp:1116-1129`，`prefix_indexer.cpp:405-429`），Mooncake 则 `ClearMooncakeBindings(nullopt, nullopt)` 清空全部共享绑定。注释解释了为什么这是安全的：`InvalidateEndpoint()` 运行在源自己的事件循环上、两次 dispatch 之间，因此撤索引时没有该源的 batch 在飞（`event_handler.cpp:337-339`）。序列号缺口无法用 replay 补齐时，`ZMQClient` 调 `MarkStale`（`zmq_client.cpp:316-321, 337-340`），之后该源的事件被丢弃直到完整重同步（`zmq_client.cpp:291-300`）。

**查询。** HTTP `POST /query` 只接受 `application/msgpack`（JSON 会在长上下文下被 JsonCpp DOM 解析拖垮，`event_manager.cpp:1015-1031`）；RPC 路径绕过 HTTP 解析，但 `ConductorService::Query` 重复同样的值校验（`conductor_service.cpp:67-88`），并在 `tenant_id` 为空时归一化为 `"default"`（`conductor_service.cpp:76-80`）。其余 HTTP 路由是 `/register`（POST）、`/unregister`（POST）、`/global_view`（GET）、`/services`（GET），每个都成对注册了"错误方法返回 405"的处理器（`event_manager.cpp:1020, 1055, 1061, 1101, 1107, 1177, 1183, 1222, 1229, 1249`）。

`PrefixCacheTable::Query`（`prefix_indexer.cpp:505-679`）是本章第二个必须逐行读的函数：

1. `LoadContextState` 拿到 `shared_ptr<ContextState>`；未注册则返回空（`prefix_indexer.cpp:510-513`）。用 `shared_ptr` 的好处是：即使别的线程把 context 从全局表里删掉，本次查询仍持有合法状态；
2. **锁外**构造 `HashStrategy` 与 `HashChain`（`prefix_indexer.cpp:515-531`），因为哈希是纯 CPU 工作，不应占用 `state->mutex`；
3. `queried_tokens` 对 `sglang_bigram` 要做 `token_ids.size() - 1`（bigram 链是 token 对，n 个 token 只有 n-1 个位置），否则"满命中"会上报一个没有任何索引块持有的位置（`prefix_indexer.cpp:532-540`）；
4. 复制 `selected_instances`（可选 `instance_filter`），后续探测期间不再依赖全局表（`prefix_indexer.cpp:544-556`）；
5. **分块探测**：块大小从 8 起翻倍，上限 512；每块先在锁外算哈希，再取一次 `shared_lock` 做纯查表；首个 miss 立即 `probe_stalled`（`prefix_indexer.cpp:563-589`）。注释明确"每个 chunk 只在查表时持共享锁"；
6. 最终在**一把共享锁**内对每个实例、每个 rank 做四级游标推进（`prefix_indexer.cpp:591-673`）。四个 `present` 谓词是嵌套的：`device_present` ⊂ `local_present = device || cpu_local` ⊂ `shared_present = !cpu_share_owners.empty() || local_present` ⊂ `disk = shared_present || accessible(disk_owners)`。所以四个 `advance_cursor` 可以**复用同一个 cursor**，每个成功前缀在每个 rank 上只被访问一次（`prefix_indexer.cpp:645-647` 的注释）；
7. 每级的 token 数由 `MatchedTokens(cursor, block_size, queried_tokens)` 计算，即 `min(cursor*block_size, queried_tokens)`（`prefix_indexer.cpp:200-213`）——**永不把尾部不完整块报成整块**；
8. `longest_match_tokens = result.disk`（最深可达层），并按实例写回 `results`（`prefix_indexer.cpp:671-672`）。

`accessible` 谓词是"介质可用 ≠ 别的引擎可访问"这一点的落点：`TierOwner` 若是 `SharedObjectOwner`（共享 Store 对象），任何引擎都可访问；若是 `EngineOwner`，则必须 instance/rank 匹配（`prefix_indexer.cpp:633-640`，`types.h:51-52` 的注释原话）。

**容量淘汰。** `StoreEngine`/`StoreShared` 在插入后调 `EvictIfOverCapacity`（`prefix_indexer.cpp:169-189`）：超过 `max_blocks`（默认 200000）就按 `write_order` 从尾部（最旧）删到 `0.9*max_blocks`，并累加 `evicted_by_capacity`。注意注释里的告警："capacity eviction can reduce reported cache coverage"——即淘汰会让 `Query` 报告比真实引擎缓存更少的前缀，这是安全的（宁可少报，不可多报）。`write_order` 只在写路径 `TouchOrder` 更新，`Query` 完全不碰它，所以读路径可以一直用共享锁（`prefix_indexer.h:55-57`）。

**哈希策略的差异。** `VllmV1HashChain` 的 `block_count_ = token_ids.size() / block_size_`（`hash_strategy.cpp:699`）——**丢弃尾部不完整块**；SGLang 链的 block count 是向上取整（`hash_strategy.cpp:890-896`）——**包含尾部不完整块**。因此两种策略产生的 `ProjectedPrefix` 不可互换，`ContextKey` 里也没有策略字段，策略由 `HashProfile` 承载并在 `Register`/`ValidateProfileBinding` 时校验（`prefix_indexer.cpp:279-297`）。vLLM 链是"父 digest + 本块 token/lora/salt"的链式 SHA256（`hash_strategy.cpp:750-783`），`ProjectDigest` 取 digest 最后 8 字节大端拼成 u64（`hash_strategy.cpp:673-681`）；SGLang 链在 `cache_salt` 存在时先算 `sha256("sglang-cache-salt-v1\0" + salt)` 作为父 digest（`hash_strategy.cpp:901-915`），块内容是小端 int32 token 拼接（bigram 时是 token 对）。

### 流程五：P2P Store 的 BitTorrent 式滚雪球

**Register（做种）。** `Register(ctx, name, addrList, sizeList, maxShardSize, location, forceCreate)`（`core.go:141-210`，注意签名比 `docs/source/design/p2p-store.md:56` 多一个 `forceCreate bool`）：

1. 校验两个列表等长非空；`catalog.Contains(name)` 则拒绝（**同一实例对同一名字只能打开一次**）；
2. 对每段内存 `memory.Add(...)`（引用计数注册），失败则回滚已注册的；
3. 按 `maxShardSize` 切 shard，每个 shard 的 `gold = [{SegmentName: localServerName, Offset: addr+offset}]`，`replica_list = nil`；
4. `metadata.Create(...)`（etcd 事务：`Version(key) == 0` 才 Put，`metadata.go:131-148`）——保证不会覆盖已有 payload；`forceCreate` 时才用无条件 `Put`；
5. 成功后写本地 `Catalog{IsGold: true}`。

**GetReplica（下载 + 成为新源）。** `GetReplica`（`core.go:426-478`）先 `metadata.Get` 拿 `payload + revision`，`validateReplicaLayout` 校验调用方 buffer 布局与注册布局**逐项一致**（`core.go:386-404`；注释解释了为什么必须校验：`doGetReplica` 与 `updatePayloadMetadata` 都按 `MaxShardSize` 步进并用位置索引 `payload.Shards`，布局不符会越界或短写）。然后进入一个"下载 → 重读元数据 → 判断是否要重试"的循环：

- `doGetReplica`（`core.go:275-325`）为**每个 shard 起一个 goroutine**，用 `sync.WaitGroup` 等待，错误通过容量为 1 的 `errChan` 只保留第一条；任一失败则先 `wg.Wait()` 再反注册（注释：不能反注册还在飞的传输所用的内存）；
- `performTransfer`（`core.go:480-502`）按 `GetLocation(retry)` 依次换源，最多 `max(3, shard.Count())` 次；
- `performTransferOnce`（`core.go:525-563`）分配 batch → `openSegment` → 提交一个 READ 请求 → `waitTransfer` → 无论成功失败都在 defer 里 `freeBatchID`；
- `waitTransfer`（`core.go:591-618`）是 PR #3851 的 spin budget：前 128 次背靠背 poll，之后从 100µs 指数退避到 1ms。注释的理由很实在：一个 checkpoint 可能有几百个 poller，全速 spin 只会打满 CPU；而且某些传输（如 NVLink async）只有被 poll 才推进状态。**ctx 取消时仍会把任务驱动到终态再 free**（引擎拒绝释放有在飞任务的 batch），然后以 `ctx.Err()` 为准返回错误（`core.go:580-584, 614-616`）；
- 下载完成后再 `metadata.Get` 一次，`reconcilePayloadAfterTransfer`（`core.go:409-424`）判断：layout 变了 → `ErrInvalidArgument`；revision 变了但 layout 没变 → 把新 revision 带下去（避免用旧 CAS 版本收尾）；只有"本次用过的某个源消失了"（`!isSubsetOf(old, new)`）才需要重试。重试前先反注册本轮 buffer 再重新注册（`core.go:468-470`）；
- 最后 `updatePayloadMetadata`（`core.go:620-664`）把本机地址追加进每个 shard 的 `replica_list`，用 `Update` 的 **ModRevision CAS**（`metadata.go:160-185`）提交；CAS 失败就重新 `Get` + 重新校验布局 + 再追加，直到成功。成功后写 `Catalog{IsGold: false}`——**本机自此成为新数据源**。

`Update` 还有一个容易忽略的行为：若 `payload.IsEmpty()`（所有 shard 的 gold 与 replica_list 都空了），CAS 成功后执行的是 **`OpDelete`** 而不是 `OpPut`（`metadata.go:162-170`）。所以 `Unregister` 把 gold 清空、`DeleteReplica` 把本机 replica 清空，两者叠加会让键自动消失，无需显式删除。

## 与其他模块的边界

**reshard → Transfer Engine。** Python 侧只依赖 `mooncake.engine` 暴露的方法名与参数形状：`scatter_transfer_sync_{read,write}_with_ticket`、`submit_scatter_{read,write}`、`batch_transfer_sync_{read,write}`（`executor.py:251-266, 315-334`）。它不 import C++ 类型，native 只通过 `_CompletionTicket` 的 `status`/`drain` 契约参与（`completion.py:16-21`）。因此换一个"长得像"的 fake engine 就能跑全部 Python 测试（见 `tests/reshard_transfer_engine/test_executor.py:42-104`）。引擎身份用 `get_engine_ptr()`（`transfer_engine_py.h:301-303`，绑定在 `transfer_engine_py.cpp:2017`）做 canonical id；拿不到就退化为 `("python", id(engine))`（`completion.py:170-189`）。

**reshard → Store。** Weight 快照走 `WeightStore`，底层要求 `begin_weight_snapshot`/`commit`/`get_into_ranges`/`prepare_get_into_ranges_snapshot`；`StoreBackend` 对可选方法做 `_optional_method` 探测，对必需方法 `_required_method` 缺失即报错（`weight/_store/backend.py:118-156`）。`get_into_ranges` 的返回值支持"buffer 结果"与"object 结果"两种形状（`backend.py:229-247`）。

**conductor → 上游发布者。** 只依赖 ZMQ + msgpack 线格式：`[topic, 8 字节大端 sequence, payload]`（`zmq_client.cpp:17`；`docs/source/design/conductor/publisher-design.md:8-14` 给出 master 侧 PUB 的完整格式）。发布者类型由注册时的 `publisher_kind` **信任并锁定**，运行时只校验消息自带的 kind/endpoint 与注册一致（`event_handler.cpp:304-309`）。

**conductor → 下游路由器。** 三条通道共享同一份业务逻辑 `ConductorService`（`conductor_service.h:12-16`）：HTTP（`coro_http`）、`coro_rpc`、以及 C/C++/Python/Rust 客户端。Rust crate 静态链接 `libconductor_cpp_core`（`rust/src/lib.rs:37-39`），Python 扩展是 `mooncake._conductor`（`mooncake-integration/conductor/conductor_py.cpp:286`），Python 包装层刻意保留 pybind 的原始返回约定（int rc / `{"ret", ...}` dict）以和 `mooncake.store` 一致（`python/mooncake/conductor.py:1-6`）。

**p2p-store → Transfer Engine / etcd。** Go 通过 cgo 直接包含 `transfer_engine_c.h`（`p2pstore/transfer_engine.go:25-27`），`NewTransferEngine` 调 `createTransferEngine`（`transfer_engine.go:39-52`）；`installTransport` 的协议选择很有意思：`nicPriorityMatrix` 为空时装 `"tcp"`，否则装 `"rdma"`（`core.go:88-92`）。元数据只依赖 etcd v3 的 Get/Put/Txn（`metadata.go:114-125`）。

**RL → Store。** `mooncake-rl` 的示例用的是 `MooncakeDistributedStore` 的 `put_tensor`/`get_tensor`（`mooncake-rl/examples/rl_samples.py:98-106, 142, 306`）；真正的 RL 结构化数据面 `python/mooncake/structured_object_store.py` 也是建在 Store 的 `put/get/remove/is_exist` 四个原语上（`BundleStore` Protocol，`structured_object_store.py:57-64`），并用 `_MooncakePayloadTransport` + `_BundleManifestStore` + `_StructuredObjectLayer` 三层把"张量/非张量字段"打包成 stage 级对象，DataProto 只保留一个可 JSON 序列化的 ref（`export_dataproto_ref`，`structured_object_store.py:380-410`）。

## 设计取舍与易踩坑点

**1. 为什么 `max_inflight_batches` 默认是 1，且 >1 必须 fail-closed？**
默认 1 走的是**逐批同步**路径：`_execute_batch` 优先调用 `scatter_transfer_sync_*_with_ticket`（C++ 内先 drain 再返回 ticket），只有引擎没有该入口时才退到扁平 `batch_transfer_sync_*`（`executor.py:365-368`，`completion.py:242-259`）；行为与旧版逐批等待完全一致，风险最低。收益来自 >1：多个 endpoint 的批次可以真正重叠。但重叠需要一个"提交后立即可返回句柄"的原生入口，即 `submit_scatter_*`；老引擎只有 `batch_transfer_sync_*`，**没有可 drain 的句柄**。此时任何非零返回都无法证明 DMA 已停止，所以设计上宁可拒绝启动窗口，也不静默降级（`executor.py:372-377`，`README.md:143-147`）。同理，扁平 batch 在 >1 时被拒绝（`executor.py:259-262`），因为扁平入口同样没有 ticket。

**2. 为什么非零结果在扁平路径上被当成"未知"而不是"失败"？**
`_batch_transfer_with_completion_fence` 对 legacy 路径的处理是：抛异常 → `_CompletionUnknown`；返回非零 → **也** `_CompletionUnknown`（`completion.py:243-259`）。注释解释：同步 API 的非零可能只是超时，而 native DMA 仍持有 allocation。代价是"一次普通失败也要求重启引擎"，收益是"绝不释放可能仍被 DMA 引用的缓冲区"。这是典型的 fail-closed 取舍。相比之下，带 ticket 的路径可以把 FAILED_DRAINED 与 UNKNOWN 区分开（`completion.py:261-279`）。

**3. 已知失败为什么也要把所有已发布 ticket drain 完？**
因为窗口里可能还有在飞 DMA，而调用方拿到异常后大概率会释放 allocation。`_execute_batches` 在 break 之后无条件排空 `published`（`executor.py:487-488`），并在排空过程中把 unknown/interrupted 收集进 `unresolved`，从而把"部分失败"升级成"必须隔离"。代价是失败路径延迟更高。

**4. `_CompositeCompletionTicket` 的子 ticket 只拿到剩余预算。**
`drain(timeout_ms)` 只对 `status == COMPLETION_UNKNOWN` 的子 ticket 调 drain，且 `remaining_ms` 可能为 0（`completion.py:133-141`）。这不等于"没 drain"：0 传给原生 `waitFor` 会走一次 `poll()` 后立刻检查 deadline（`transfer_engine.cpp:1376-1381`）。**不要**以为每个子 ticket 都能拿到完整的 `timeout_ms`。

**5. `resources_handed_off` 是 quarantine 的一致性问题。**
`drain_pending_transfer` 与 `pending_transfer_status` 都要求 `resources_handed_off` 为真（`completion.py:511-515, 531-535`），否则抛"handoff is incomplete"。这个标志只有两条置真路径：`registered_targets/sources` 在异常属于本 engine 时把帧内注册与 token 交接过来；或者 `_raise_completion_*` 在找不到任何可交接帧时主动 `_seal_pending_resource_handoff`（`executor.py:565-573`）。因此它是一个**防御性/瞬时状态**：`_stage_pending_resources` 先把标志置假、再单独加锁封口，并发 drain 恰好落在中间时才会看到 incomplete。**踩坑场景**：在 `registered_targets` 之外自行注册缓冲区后直接调用 `MooncakeTransferEngineExecutor.execute_batch()`，或把异常跨 context 传递后再构造隔离——此时记录会被封成"空交接"，drain 能正常返回，但那条记录并不持有任何 registrations/tokens，缓冲区只能靠原上下文的 cleanup 或重启处理，隔离路径帮不上忙。正确做法是在同一注册上下文内 catch，或显式 `retain_pending_resources` 补齐（`executor.py:575-588` 是公开入口）。

**6. `_UndrainableCompletionUnknownTicket` 是"不可 drain 的终态"，不是"待重试"。**
它的 `drain()` 立即返回 `COMPLETION_UNKNOWN`（`completion.py:94-95`），而 `restart_required=True`。所以当 `PendingTransferManager._reserve_submission` 看到这种记录时会直接抛"restart-required"（`completion.py:464-471`），而不是提示你 drain。只有真正带原生句柄的 ticket 才可能 drain 到终态。

**7. 同一个原生引擎不能有并发 submission，但这是"进程级"约束。**
`_active_submission_engine_ids` 与 `_pending_transfers` 都是模块级全局（`completion.py:206-208`），`_reserve_submission` 用 `engine_identity` 过滤。这意味着同一进程内两个 Python 线程操作**同一个** engine 会被拒绝；不同 engine 之间互不影响。但注册/反注册（`register_memory`/`unregister_memory`）并不受这套 fencing 保护——**此处未完全确认**多线程下直接操作同一 engine 注册表是否另有原生锁。

**8. reshard 的 KV cache 执行器是另一套，别套用 weight 的语义。**
`kv_cache/executor.py` 只用同步 `batch_transfer_sync_write`（`kv_cache/executor.py:164`），出错后把 `_uncertain = True`（`kv_cache/executor.py:179`）并拒绝后续所有操作，要求重建 transport；它没有 ticket、没有 pending quarantine、也没有共享超时预算。所以"reshard 有完善的 unknown 隔离"这句话只对 weight 路径成立。

**9. `plan_placement_transfer_to_local_target` 的隐含约束。**
不传 `target_participant_id` 时要求 `target_placement.topology.world_size == 1`（`api.py:71-77`）；传了则必须在 parts 里找到。最后还要求结果恰好一个 target executor（`api.py:116-118`）。这意味着它不能用来"只规划多参与者目标里的某几个"——那是 `select_worker_executors` 在执行期的职责。

**10. conductor 文档与代码已经脱节。**
`docs/source/design/conductor/conductor-architecture-design.md:154-181` 仍在讲 Go 的 `conductor-ctrl/`、`go build`、`main.go`；实际代码是 C++（`mooncake-conductor/src/main.cpp`，CMake target 在 `mooncake-conductor/CMakeLists.txt`），且 HTTP 只支持 msgpack（文档未提）。`docs/source/design/p2p-store.md:38, 56, 87` 的 `NewP2PStore`（两参数）与 `Register`（六参数）也已过时：代码是 `NewP2PStore(metadataConnString, localServerName, nicPriorityMatrix)`（`core.go:75`）与 `Register(..., location, forceCreate)`（`core.go:141-147`）。**以代码为准**。

**11. `ContextKey` 不含 `cache_salt`，但 salt 会改变哈希。**
salt 由 `CreateChain` 参与计算（vLLM 在 block 0 编码里，SGLang 在根 digest 里），所以同一 `ContextKey` 下不同 salt 的请求会得到不同的 `ProjectedPrefix`。这意味着索引条目天然按 salt 隔离，但**注册时并不声明 salt**——如果同一实例对同一上下文先后用不同 salt 发事件，索引会同时保留两套前缀，`Query` 只能命中与本次请求 salt 匹配的那套。这是设计选择，不是 bug（`docs/source/design/conductor/conductor-architecture-design.md:71-74` 也提到 vLLM 忽略尾部不完整块、SGLang 包含尾部块）。

**12. `Query` 的首个 miss 即终止，且 `longest_match_tokens` 取最深可达层。**
`advance_cursor` 一旦遇到不存在的 prefix 就 `break`（`prefix_indexer.cpp:595-608`），不会跳过空洞继续找更长的匹配——这是"前缀连续性"的定义。另外注意 `result.dp` 记录的是 **device** 命中数，而 `result.device/cpu_local/cpu_share/disk` 是**跨 rank 取最大值**（`prefix_indexer.cpp:664-671`）。所以"某实例的 disk 层命中 100 token"不代表某个具体 rank 能拿到 100 token；要看 `rank_matches`。

**13. P2P 的"同一实例一次"约束是本地账本，不是全局约束。**
`catalog.Contains(name)` 只查本进程（`catalog.go:38-43`）。两个进程用同一 `localServerName` 注册会撞在 etcd 的 endpoint/segment 名上，而不是被 catalog 拦住。另外 `doGetReplica` 里 goroutine 闭包捕获 `source`/`shard` 依赖 Go 1.22+ 的 per-iteration 循环变量语义；模块声明 `go 1.25.0`（`src/p2pstore/go.mod:3`），所以是安全的——但在更老的 Go 上编译这段代码会得到所有 goroutine 共享最后一个变量值的经典 bug。

**14. HiCache 在本仓库没有实现，别在 `mooncake-store` 里找。**
`docs/source/design/hicache-design.md` 明确说 HiRadixTree 在 SGLang 侧（`:15-20`），Mooncake 提供的是 L3 backend 的零拷贝 RDMA 与 batch 读写（`:93-107`）。本仓库只有：设计文档、集成/部署文档、`scripts/e2e/python/test_hicache_storage_mooncake_backend.py`（拷自 SGLang 的 e2e）、`scripts/check_hicache_hugepage_requirements.py`（Docker 内叫 `mooncake-hicache-sizing`）。**Store 客户端的 LocalHotCache 是另一回事**：它的配置是 `total_size_bytes/block_size_bytes(16MB)/use_shm/admission_threshold(2)`（`local_hot_cache.h:22-29`），用 `HotCachePutToken{cache_epoch, key_generation}` 做"异步填充期间 key 被删/覆盖则丢弃"的取消（`local_hot_cache.h:36-39, 81-96`）。PR #4764 修的是一个真实崩溃：`ProcessSlicesAsync` 把**每个**目的 slice 都交给 `LocalHotCacheHandler::SubmitPutTask`，而后者用 host `std::memcpy` 从目的地址拷出；当目的是 CUDA 指针时直接 SIGSEGV。修法是用 accelerator registry 的 `FindDeviceForPointer` 探测并跳过 device 驻留的 slice（`mooncake-store/src/client_service.cpp:5613, 5626-5650`）。**读代码时的教训**：LocalHotCache 的写回源是"读的目的缓冲区"，不是"Store 的副本"，所以目的介质决定了这条路径是否合法。

**15. conductor 的 `BlockPresence` 是"存在性集合"，不是"副本计数"。**
同一 engine 重复 `StoreEngine` 同一前缀不会增加计数（`std::set`），所以 `disk_owners.size()` 不能当副本数用。真正影响"能不能从别处读"的是 `accessible()` 谓词（`prefix_indexer.cpp:633-640`）。另外 `cpu_share_owners` 用 `SharedObjectOwner`（含 `backend_id/object_id`），而 `disk_owners` 用 `variant`——同一个物理落盘对象可能同时以 engine 和 shared 两种身份出现，删除时必须两边都清（`ClearEngine` 与 `ClearShared` 各管一半，`prefix_indexer.cpp:405-429, 482-503`）。

**16. HiCache 的 prefetch timeout 是"动态公式"，不是固定值。**
设计文档给出的公式是 `timeout = prefetch_timeout_base + prefetch_timeout_per_ki_token * num_token_to_fetch / 1024`（`docs/source/design/hicache-design.md:61-65`），并且 `best_effort` 的等待时间是 0（`:50`）。读 SGLang 实现时要先确认这三个策略与两个参数确实落到了同一处，不要假定本仓库有对应代码。

## 精读清单

建议按下面顺序读，每步都先问"这一层的输入/输出契约是什么"，再问"失败时谁负责清理"。

1. **`mooncake-reshard/README.md`（全文，271 行）**。先建立四包分工、三段式生命周期、in-flight 窗口语义的整体印象。重点看 `:40-45`（world_size）、`:76-78`（PlanningLimits）、`:110-126`（ticket 三态与 legacy 路径）、`:142-164`（窗口、composite 预算、迭代器借用）。
2. **`weight/types.py` + `weight/topology.py` + `weight/_planner/contracts.py:97-355`**。看懂"一个 tensor 如何被三种轴描述""一个 N-D region 的 11 个字段各是什么"。把 `iter_segments` 用纸笔对一个 2-D 例子展开一遍。
3. **`weight/_planner/core.py:146-324`**。顺着 `_plan_transfer` 读：候选索引 → DP 过滤 → overlap → exactly-cover 校验 → 预算检查 → 排序。这里能看清"为什么 planner 是 copy-only 且不做 dtype 变换"。
4. **`weight/_planner/binding.py:427-557`**。注意它"只产出快照、不执行"的边界，以及 `_validate_target_physical_ranges` 对别名/allocation 的限制。
5. **`transfer_engine/contracts.py` + `lifetime.py`**。把 `TransferBatch` 的扁平/range 一致性和 `AllocationTokenSet` 的三态搞清。
6. **`transfer_engine/completion.py:16-309`**。先看 `_CompletionTicket` 契约，再看 `_drain_completion_ticket` 的"状态→drain→重试→终态"循环，最后看 `_batch_transfer_with_completion_fence` 对 legacy 与 ticket 两条路径的差异。
7. **`transfer_engine/completion.py:312-656`**。`PendingTransferManager` 的预约/隔离/排空。重点 `_reserve_submission`（454-481）、`_retain_pending_resources`（384-437，注意重复 token/资源去重与 "handing off while draining" 的拒绝）、`drain_pending_transfer`（521-656，注意反注册降序与 token 从尾部释放）。
8. **`transfer_engine/executor.py:355-525`**。逐行读 `_execute_batches`，把几个状态变量与四路异常优先级画成状态图。对照测试 `tests/reshard_transfer_engine/test_executor.py:163-220`（窗口上界与 receipt 顺序）、`:220-337`（失败/中断/unknown）、`:558-574`（unknown 压过 failed）。
9. **`weight/_te/execution.py:26-114, 274-395`**。理解 endpoint 分组与排序键为什么能让窗口跨 endpoint 重叠；`validate_scoped_executor_snapshot` 的六项复核（placement id/digest、lease、fragment 覆盖、快照相等）。
10. **`weight/_te/batching.py` + `weight/_te/reader.py:96-447`**。按"预检 → 预约 → 取 pin → 复核 → 降级 → 注册 → 执行 → 终结 token"走一遍。再对照 `sink.py:94-451` 看对称性（source/target 互换、`registered_sources` 替代 `registered_targets`）。
11. **`mooncake-integration/transfer_engine/transfer_engine_py.cpp:67-224`**。`ScatterTransferTicket` 的三态与 `IsClock()` 映射；`submitScatterWithTicket` 为什么用 `terminal(FAILED_DRAINED)` 报参数错误而不是抛异常。
12. **同文件 `:653-707, 1792-1898`**。对比 sync 入口（C++ 内 drain）与 submit 入口（不 drain）；确认 `gil_scoped_release` 与 `keep_alive<0,1>` 的作用。
13. **`mooncake-transfer-engine/src/transfer_engine.cpp:1331-1384, 1483-1505, 1581-1771`**。原生 `ScatterTransferOperation` 的 build/poll/abort；`waitFor` 的 Clock 语义；TENT 的 `requestAbort→cancelTransfer`。再看 `tent/src/runtime/transfer_engine_impl.cpp:2091-2173` 理解"为什么必须选可取消 transport"。
14. **`mooncake-conductor/src/main.cpp` + `event_manager.h`**。先建立组件关系，再读 `event_manager.cpp:734-1008` 的订阅/退订与锁序（两处死锁注释是重点）。
15. **`event_handler.cpp:268-350, 352-479`**。`BeginDispatch/DispatchLease/WaitForIdle` 的 in-flight 计数；`OnSourceStale → InvalidateEndpoint` 的全量撤回。再看 `:533-700` 的共享对象物理分量引用计数。
16. **`prefixindex/prefix_indexer.h` + `prefix_indexer.cpp:505-679`**。画一张"四层嵌套游标"的图；确认 `MatchedTokens` 的上界裁剪；再看 `:169-189` 的容量淘汰与 `:238-297` 的注册/profile 绑定。
17. **`prefixindex/hash_strategy.cpp:683-705, 887-997`**。自己算一遍"8 个 token、block_size=4、SGLang bigram"的 block 数与 `queried_tokens`，确认尾部块语义。
18. **`conductor_service.cpp:67-200` + `conductor_client.h` + `python/mooncake/conductor.py`**。看三条通道如何共享同一 service、错误码如何映射（`-2000/-2001/-600/-900`）。
19. **`mooncake-p2p-store/src/p2pstore/core.go:75-210, 275-325, 426-502, 525-664`**。按 Register → GetReplica → doGetReplica → performTransfer/waitTransfer → updatePayloadMetadata 走一遍；对照 `metadata.go:46-98, 131-185` 看 schema 与 CAS；最后看 `registered_memory.go:46-175` 的引用计数与 4GB 分块。
20. **`mooncake-rl/examples/rl_samples.py:335-388` + `python/mooncake/structured_object_store.py:543-604, 380-410`**。理解"示例只是 Demo、真正数据面是 DataProto ref + stage 对象"这一层。
21. **`docs/source/design/hicache-design.md`（全文 122 行）**。把三级缓存、prefetch 三策略与动态 timeout 公式、write-back 三策略、page-first-direct 布局、MLA 单 rank 写回、多 rank `all_reduce(min)` 记下来；然后去 SGLang 仓库看实现。
22. **`mooncake-store/include/local_hot_cache.h:22-96, 173-235` + `src/client_service.cpp:5613-5650`**。最后读这块，避免把 LocalHotCache 误当 HiCache L2；重点理解 `HotCachePutToken` 的 stale-fill 取消语义与 #4764 的 device 目的地址判定。

---

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

---

# 附录 A · 跨子系统的重要发现

粗读阶段最有价值的产出，是那些**只看单个文件永远发现不了、但会让整体理解跑偏**的事实。以下按主题归纳。

## 1. 仓库里并存着多套实现，必须分清

| 名字 | 实际是什么 | 如何区分 |
| --- | --- | --- |
| `WITH_STORE_RUST`（默认 ON） | 给 **C++ Store** 生成 Rust 绑定（走 `store_c.h`） | 不是 Rust 实现 |
| `WITH_STORE_RS`（默认 OFF） | **独立的 Rust 原生 Store**（14 个 crate，18.3 万行，无 Master） | 真正的新实现 |
| `mooncake-store/rust/` | C++ Store 的薄安全绑定（约 1.2k 行） | 有 `link` / `dlopen` 两种后端 |
| `mooncake-store/go/` | 同一份 `store_c.h` 的 CGo 绑定 | 约 550 行 |
| `mooncake-store-rs/` | 从零重写的 Store，嵌入式 WRH 路由 | 强制要求 `USE_TENT=ON` |

Python 侧通过 `MOONCAKE_STORE_BACKEND=cpp|rs` 二选一，`python/mooncake/store/__init__.py` 里用两份 frozenset 白名单精确刻画两套 API 的差异面。

## 2. 两套错误模型并存

- 旧 API 返回 `int`，用 `include/error.h` 的 `ERR_*`（`ERR_INVALID_ARGUMENT = -1` … `ERR_NOT_IMPLEMENTED = -303`）
- 新 API 返回 `Status`，用 `include/common/base/status.h` 的 RocksDB 风格 `Status::Code`（`kOk = 0` … `kNotImplemented = 999`）

两套数值完全对不上，所以 TENT 兼容层专门写了 `tentToClassicError()` 做映射。判断某个函数该用哪套错误码，只能看签名。

## 3. 文档与代码存在若干不一致

粗读中实际核对出的几处（**照抄文档会踩空**）：

- `scripts/run_ci_test.sh` 被 README 与项目内 CI skill 反复引用，但 `git ls-files` 确认它**未被跟踪**。实际入口是 `scripts/ci/` 下的若干脚本与 `scripts/run_tests.sh`。
- 版本号有两处且不一致：根 `pyproject.toml` 是 `0.3.12.post1`（scikit-build-core 路径），`mooncake-wheel/pyproject.toml` 是 `0.3.13`（CI 实际发布路径）。
- 根 `pyproject.toml` 的 `sdist.include` 写了 `extern/yalantinglibs/**`，但该目录并不存在——`yalantinglibs` 是由 `mooncake-common/FindYLT.cmake` 用 FetchContent 下载的。仓库里唯一的 submodule 只有 `extern/pybind11`。
- `TransferEngineImpl::installTransport` 的头文件注释写着 "Only for testing"，但 `mooncake-store/src/client_service.cpp` 与 Python 绑定都在生产路径调用它。注释已过期。

## 4. 命名陷阱

- **`BatchTransfer` 不存在同名类型**（见前文）。
- **`Client` 有两层**：`mooncake-store/include/client_service.h:77` 的 `class Client` 是内部核心逻辑；Python 看到的 `MooncakeDistributedStore` 实际绑定的是 `MooncakeStorePyWrapper` → 抽象基类 `PyClient` → `RealClient` / `DummyClient`。设计文档里说的 "Client 类" 通常指前者。
- **`showLinks()` 不是连接管理**，只是本地链路诊断输出（GID/NUMA/速率），TENT 模式下直接返回占位串。
- **`openSegment` 不建立任何连接**，`closeSegment` 在 classic TE 里是空实现。它只是 `getSegmentID(name)`，可能触发一次元数据拉取，失败返回 `-1`。
- Conductor 的索引主类叫 `PrefixCacheTable`，文件却叫 `prefix_indexer.h`。
- `memory_location.h` 里的 `pagesize = 4096` 是**硬编码常量**，不是运行时页大小。

## 5. 设计哲学：fail-closed

这类系统里"宁可不可用，也不带着错误状态服务"的倾向非常明显，三个例子：

- **HA 选主成功 ≠ 可以服务**。被选中的节点还必须完成 standby 追赶、导出并校验完整 `PromotionContext`、恢复状态、复验租约，任一步失败就保持不可用。
- **元数据缓存不会因服务抖动而清空**。`GetResult` 分三态：`kUnavailable`（超时/抖动）不计入失败、不驱逐缓存；只有 `kNotFound`（后端权威地表示 key 已不存在）才累加，且要连续 2 次才真正 `erase`。
- **元数据插件创建失败是 `LOG(FATAL)`**。`MetadataStoragePlugin::Create` 找不到匹配协议时直接 abort，而不是返回 `nullptr`。

## 6. 性能相关的关键机制（精读时要重点验证）

- **完成状态打包**：`Slice::completion_state` 把 "seal bit(63)" 与 "已完成 slice 计数" 压进同一个 64 位字，避免两个独立原子量之间的竞态。这是全引擎最微妙的并发协议。
- **快慢两条轮询路径**：`getBatchTransferStatus` 在 `counter_eligible` 且无失败时只看计数器、不逐 task 轮询；此时 `transferred_bytes` 返回 0 是**设计而非 bug**。
- **两阶段内存注册**：`local_memory_regions_` 与 `registering_memory_regions_` 分开，使得耗时的 MR 注册期间就能阻止另一线程注册重叠区间。
- **切片与选路**：`submitTransfer` 会把相邻同组请求合并成一个 `TransferTask`（`request_count > 1`），此后 `task_id` 不再等于请求下标。
- **隐式副作用**：`getTransferStatus` 在 task 完成时会顺手发出挂在该 batch 上的通知——**不轮询就不会发通知**。

## 7. 构建开关是理解"功能为何不存在"的钥匙

`mooncake-common/common.cmake` 集中定义了 60+ 个 `USE_*` / `WITH_*` 开关，顶层 `CMakeLists.txt` 还有互斥校验，例如：

- `STORE_USE_K8S_LEASE` 与 `STORE_USE_ETCD` 不能同开（会在同进程内各自构建 Go c-shared 库）
- `USE_TPU` / `USE_XPU` 强制要求 `USE_TENT`

遇到"这台机器装了 RDMA 却走 TCP"之类的问题，答案通常在 `transfer_engine_impl.cpp` 的 `init()` 决策树里。

---


