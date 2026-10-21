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
