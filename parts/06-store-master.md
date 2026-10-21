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
