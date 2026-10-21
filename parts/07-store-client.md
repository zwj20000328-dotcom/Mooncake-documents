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
