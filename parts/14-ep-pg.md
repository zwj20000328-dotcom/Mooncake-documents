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
