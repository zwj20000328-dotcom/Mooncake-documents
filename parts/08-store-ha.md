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
