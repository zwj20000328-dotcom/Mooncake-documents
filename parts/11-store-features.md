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
