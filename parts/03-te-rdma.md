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
