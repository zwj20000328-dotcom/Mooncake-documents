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
