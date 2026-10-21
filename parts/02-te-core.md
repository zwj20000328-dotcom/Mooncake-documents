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
