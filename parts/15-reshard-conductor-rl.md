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
