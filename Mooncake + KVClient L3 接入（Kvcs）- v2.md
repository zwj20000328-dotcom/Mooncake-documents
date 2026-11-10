# Mooncake + KVClient L3 接入（Kvcs）v2

更新日期：2026-11-10。本文取代 `Mooncake + KVClient L3 接入（Kvcs）.md`（v1，2026-11-06）。

代码核对基线：Mooncake `77689203`；KVClient SDK `e6717d04`（v1 声称的 `e25e7e1a` 在本地仓库不存在，不可核对）。

---

## 1 背景

### 1.1 目标

通过 KVClient 将远端 KV 存储接入 Mooncake 的 DFS 副本流程，作为共享 L3。应用沿用 Put / Get 接口，在请求内完成配置中的内存与存储写入；内存副本淘汰后，其他 Client 通过 Master 查询并读取 L3 数据。

### 1.2 v1 → v2 的六条裁决

| # | 裁决 | 对 v1 的影响 |
| --- | --- | --- |
| 1 | 只增加 `UsesKvClient()`，与 `UsesKvcs()` **平级**；不引入 `UsesManagedKvStorage()` | v1 §2.2 的公共能力开关作废。Master 侧需要两个独立 flag，因为决策 6 要求恢复门保持 KVCS-only |
| 2 | `ReconcileMissingKvcsReplica()` 推广为 `ReconcileMissingKvReplica()` | v1 §5.3 成立，但改动面是 Client + Master 共 5 处，不是「推广一个函数名」 |
| 3 | 512B 对齐：`aligned_size = object_size = N`，且 **N 必须是 512 的倍数**；存储 value 就是应用提供的 N 字节原始数据 | v1 §1.2「长度按 512 对齐」与 §3.2「N 本身满足对齐」的矛盾消解为：**约束 N 的取值，而非填充 value**。零拷贝因此得以保留 |
| 4 | 两副本（memory + ASU）硬逻辑；ASU 失败 → 整个 Put 失败 + memory 副本一起 revoke | **零代码改动**，现有 `RELIABLE_MULTI_REPLICA` 即是该语义。v1 §7.3 描述准确 |
| 5 | 删除顺序改为**先删 value，再删元数据** | 推翻 v1 §7.1。与决策 2 的 miss 自愈互相成全（见 §2.7） |
| 6 | 不支持从 snapshot / OpLog 恢复对象目录 | v1 §7.3 与 §8.3 场景 8 作废。恢复门保持 KVCS-only，并加启动期显式拒绝 |

### 1.3 与 KVCS 的关系

v2 不新建 `ObjectStorageAdapter` 实现类，而是**实现一个与 `KvcsLowLevelDriver` 平级的 `KvcsDriver`**，注入现有 `KvcsObjectStorageAdapter`。理由见 §2.4。

```
ObjectStorageAdapter（复用，不改）
  └── KvcsObjectStorageAdapter（复用，2 处逻辑下推）
        └── KvcsDriver（复用接口）
              ├── KvcsLowLevelDriver（既有，EFC / gRPC）
              └── KvClientDriver（新增，Host RDMA）
```

---

## 2 设计

### 2.1 接入位置与后端选择

`fs_adapter_type = "kvclient"`，与 `"kvcs-lowlevel"` 平级。配置解析、工厂、后端注入沿用现有对象存储路径：

```
RealClient → FileStorage → DistributedStorageBackend
                              └── object_storage_adapter_ = KvcsObjectStorageAdapter(KvClientDriver)
```

`UsesObjectStorage()` 已完全抽象化数据面（`distributed_storage_backend.cpp:664`、`:790`、`:990`、`:1032`、`:1050`），**`DistributedStorageBackend` 的读写路径零改动**。

### 2.2 key 身份模型 ⚠️ 待裁决

**v1 的「string key 贯通」不成立。** SDK 的 `CacheKey` 是 8 字节定长（`kv_types.h:39-40`），线格式 SQE 中 key 字段固定 8B，Delete/Exist 的 key entry 后 8B 是 reserved 且校验必须全 0（`kv_protocol.cc:107-113`）。能力协商字段 `ServerKvCapabilities.keyLength` 只被打印日志，不参与 clamp（`kv_transport_impl.cc:124-139`）。SDK 曾主动从 `std::string` 改为 8 字节定长（commit `64a22ad6`）。

因此 Mooncake 侧必须把 logical key 映射为 8 字节。两个候选：

#### 选项 A：XXH64 hash

```cpp
CacheKey key = XXH64(logical_key.data(), logical_key.size(), kAsuKeySeed);
```

| 项 | 结论 |
| --- | --- |
| 改动面 | 仅 driver 内部 |
| 零拷贝 | ✅ value 就是原始 N 字节 |
| 碰撞 | ❌ 64-bit 空间。10^7 key 时约 2.7×10^-6，10^8 时约 2.7×10^-4，10^9 时约 2.7% |
| 碰撞后果 | **静默数据损坏**：后写覆盖先写，读到错 value，无任何校验可发现 |
| 迟到 Delete | 同 key 重写映射到同一 CacheKey，滞后的删除可能误删新 value |

#### 选项 B：Master 铸造，复用 `ReplicaID`

`ReplicaID` 已是 `uint64_t`（`replica.h:32`），全局唯一，且 `DistributedFSDescriptor` 已随 `Replica::Descriptor` 返回给 Client（`replica.h:351-358`、`:765`）。直接令 `CacheKey = descriptor.id`。

| 项 | 结论 |
| --- | --- |
| 改动面 | driver + Remove 路径需先取 descriptor |
| 零拷贝 | ✅ value 就是原始 N 字节 |
| 碰撞 | ✅ 不可能，Master 保证唯一 |
| 迟到 Delete | ✅ 天然免疫：新副本有新 ReplicaID，旧删除打不到新 value |
| 重启稳定性 | ✅ 决策 6 放弃恢复后，不存在「重启后 ReplicaID 变化导致指向失效」的问题 |
| Upsert | 同尺寸 Upsert 走原地复用分支（`master_service.cpp:6196-6235`），ReplicaID 不变 → 同 CacheKey 覆盖写，符合整值替换语义 |
| 代价 | Remove 路径需先 `GetReplicaList` 取 descriptor（多一次 RPC） |

**推荐选项 B。** 它把 v1 的 §3.1 整节（string key 身份链）替换为「Master 铸造 + descriptor 传递」，消除碰撞这一唯一的静默数据损坏源，且与决策 5（先删 value）天然契合——删 value 时元数据尚在，ReplicaID 可取。

若选 B，`KvcsDriver` 的 key 参数需从 `ObjectKey`（string）扩展为携带 ReplicaID 的形式；`EncodeKey`（hex 编码）随之下推为 driver 各自的实现。

> 若选 A，必须同时接受「碰撞即静默损坏」并写明 key 数量级上限。加 value 内 trailer 校验可检测碰撞，但 I/O 长度将不再等于 `object_size`，需要 staging buffer，**零拷贝失效**，与决策 3 的「存储 value 就是原始 N 字节」冲突。三者不可兼得。

### 2.3 对象大小与对齐

决策 3 落地为对 N 的约束，而非对 value 的填充：

```
object_size = aligned_size = N
N % 512 == 0
0 < N <= 16,776,704
```

**上限的由来**：SDK 线格式 SQE 的 `length` 是 24-bit 且须 512B 对齐，故单 entry 上限 = `0xFFFFFF & ~511` = **16,776,704 B**（= 512 × 32767）。

**512 对齐自动消除了 v1 遗漏的大小窗口冲突**：

```
16,776,704 = 512 × 32767   ← ASU 上限，本身 512 对齐
16,777,216 = 512 × 32768   ← 下一个 512 倍数
16,777,200 = kMaxSliceSize ← Mooncake 上限（types.h:469 + Slab.h:82,88）
```

窗口 `(16776704, 16777200]` 内**不存在任何 512 的倍数**，故通过 512 对齐校验的 N 必然 ≤ 16,776,704。冲突自动消解，无需额外处理。

⚠️ 但 driver 侧仍须独立设 `MaxValueSize() = 16776704`。Master 的 slice 校验**只在 CACHELIB 分配器下生效**（`master_service.cpp:5287-5293`），DFS-only 写 + 非 CACHELIB 时这道门不开，`KvcsDriver::MaxValueSize()` 才是真正的守卫（`kvcs_capi_driver.cpp:375`）。

**校验落点**：`PrepareWriteStart`（`master_service.cpp:5248-5255`）。放这里而非 `AllocateReplicas`，因为 Put 与 Upsert 都过这道门。descriptor 公式不变（`:5092-5093` 仍是 `object_size = aligned_size = value_length`）。

### 2.4 数据面：驱动复用

`KvcsDriver` 已是与 KVCS 无关的抽象（`kvcs_driver.h:62-78`）：

```cpp
struct KvcsIoRequest { ObjectKey logical_key; std::vector<Slice> slices; uint64_t size; };
class KvcsDriver {
    Init();  MaxValueSize();  MaxKeySize();
    GetMountpointStats();            // 默认 NOT_SUPPORTED ← 正好匹配「KVClient 无容量接口」
    BatchPut(span<KvcsIoRequest>);  BatchGet(...);  BatchDelete(span<ObjectKey>);
};
```

适配器已有注入构造函数 `KvcsObjectStorageAdapter(std::unique_ptr<KvcsDriver>)`（`kvcs_object_storage_adapter.h:19`），注入后 `Init()` 跳过 `LoadKvcsEfcTarget` / `CreateKvcsLowLevelDriver`（`:71-79`）。

**复用带来的收益**（相对 v1 新建适配器类）：

| v1 方案 | v2 复用 driver |
| --- | --- |
| 新增 `KvClientObjectStorageAdapter` 整个类 | 适配器类零新增 |
| 改 `BatchWrite`/`BatchRead` 加「连续 slices 整理」分支 | 后端零改动——`KvcsIoRequest.slices` 本就是多段，合并/staging 关在 driver 内 |
| §6.2「必须相邻、否则 NOT_SUPPORTED」的新契约 | 不需要。写侧 KVCS 已用 `value_segs` 表达多段；KVClient 在 driver 内自行决定合并或 staging |
| 大小上限零覆盖 | `MaxValueSize()` 天然承载 |
| 容量 gap 需新设计 | Master `:883-889` 已有降级：provider 返回 NOT_SUPPORTED → 只 WARNING、禁用水位淘汰、`enable_dfs_` 保持 true |

**需要下推的 2 处 KVCS 专属逻辑**（约 20 行移动，非新增）：

1. `EncodeKey`（hex 编码，`kvcs_object_storage_adapter.cpp:57-65`）→ 提为 driver 职责。KVCS 实现保持 hex，KVClient 实现按 §2.2 的裁决。
2. Delete-before-Put（`:155-170`）→ 移进 `KvcsLowLevelDriver::BatchPut`。该逻辑源于 KVCS Put 是 insert-only；ASU 若支持覆盖写（v1 §1.2 假设，**需真机验证**），KVClient driver 不需要这一步。

### 2.5 同步契约与内存注册 ⚠️ 关键约束

`ObjectStorageAdapter` 的契约是**硬同步**：

> Atomic, synchronous multi-region write: one commit and one result. Iovecs and payloads must remain valid and unchanged **until return**.
> —— `object_storage_adapter.h:51-52`、`:71-72`

而 SDK **在 `Wait` 超时后不取消任何在途操作**：

```
KvClientImpl::Wait → ClientTaskManager::Wait
                       ├─ WaitContext: cv.wait_for 超时 → result.status = TIMEOUT
                       └─ (void)Remove(taskId)   ← 只从 client 侧 map 摘除，不通知 transport
```
（`task_manager.cc:108-116`、`:366-384`）

- `AsuTransportImpl::Cancel` **无任何生产调用者**（全仓 grep `->Cancel(` 只命中 `Shutdown` 与 `Cancel` 自身定义）。
- transport 层 deadline 超时（`trans_task_executor.cc:391-405`）只在主机侧把 task 标为 COMPLETED，**已 post 到设备的 RDMA write 不会被中止**。
- 它调用的 `connManager_->ReportFailure(channel)` 只累加错误计数，达到阈值后把 channel 标 DRAINING（`connection_manager.cc:151-182`），同样不中止设备操作。
- `UnregisterRegions`（`kv_client_impl.cc:575-625`）**不检查 handle 是否仍被 in-flight task 引用**。SDK 头文件自己写了契约：「Handles referenced by queued or in-flight tasks must remain registered until those tasks complete」（`kv_client.h:58-59`）。

**v1 §6.3 的「按次注册、按次释放 + 超时即归还 buffer」因此不成立**：超时 → 注销 → 返回 → 调用者释放 buffer → 设备稍后写入已易主的内存，是 use-after-free 级别的数据损坏。

**v2 的强制结论**：`KvClientDriver` 要满足 `KvcsDriver` / `ObjectStorageAdapter` 的同步契约，就**必须自有 staging 并在超时后 quarantine**，不能把调用者 buffer 直接提交给 SDK。

- staging region 在 driver `Init()` 时一次性注册，持有 canonical `MRHandle`；`Shutdown()` 先排空 in-flight 再注销。
- 超时 / CANCELED 的 slot **永不归还、永不注销**，直到 SDK 确认在途操作结束。
- **禁止在 task 级 Wait 超时后读 `entryStatus`**：其初值是 `Status::OK()`（`kv_client_impl.cc:460`、`:514`），`BuildResult` 在 `!Done()` 时原样返回（`task_manager.cc:354-357`），会读到假成功。

这与决策 3 不冲突：staging 是 driver 内部的，对上层仍表现为「value = 原始 N 字节」。代价是 driver 内一次 memcpy。

> 注意：这条约束是**接口契约自动施加**的，不是可选优化。走 driver 复用路线后，无法在遵守同步语义的前提下写出「超时即归还原 buffer」的实现。

### 2.6 写入与成功判定（决策 4）

两副本配置 `replica_num=1, nof_replica_num=0, dfs_replica_num=1` **零代码改动**。

```
DetermineReplicaWriteMode（replica.h:313-324）
  dfs==0 ?  否 → 不进 FLEXIBLE_DUAL_REPLICA
  replica>1 || nof>1 || dfs>0 ?  是 → RELIABLE_MULTI_REPLICA
```

| 判据 | 求值 | 位置 |
| --- | --- | --- |
| `HasExpectedReplicaAllocation` | `memory==1 && nof==0 && dfs==1` | `client_service.cpp:346-348` |
| `NonDfsTransfersSucceeded` | memory 1/1、nof 0/0、零失败 | `:321-327` |
| `AllAllocatedTransfersSucceeded` | 上式 && dfs 1/1 && dfs 零失败 | `:329-333` |
| 不满足 | `PutRevoke(ALL)` + `success=false` | `:384-401` |

短路顺序已正确：`allocated_dfs_replicas > 0 && NonDfsTransfersSucceeded(...)`（`:2160-2161`）保证 memory 失败时**零 ASU 调用**，直接 revoke ALL。

DFS-only（`replica_num=0, dfs=1`）同样零改动：`HasExpectedReplicaAllocation` 要求 `memory==0`，`NonDfsTransfersSucceeded` 因 0==0 平凡成立。`PrepareWriteStart` 对 `replica_num` 无约束（`master_service.cpp:5240-5255`）。

⚠️ **配置陷阱**：`dfs > 0` 时 `FLEXIBLE_DUAL_REPLICA` 不可达，故 `memory + nof + ASU` 三副本会静默落入 RELIABLE，**nof 也变成必需**。建议在 `PrepareWriteStart` 加：

```cpp
if (use_kvclient_ && config.nof_replica_num > 0 && config.dfs_replica_num > 0)
    return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
```

把「只支持两副本 / DFS-only」变成启动期可发现的约束。这会让 KVClient 比 KVCS 多一条限制，属有意的不对称。

Upsert 复用同一判定：`FinalizeBatchUpsert` 不走 `DetermineFinalizeDecision`，但判据是 `HasExpectedReplicaAllocation && AllAllocatedTransfersSucceeded`（`client_service.cpp:3497-3500`），等价严格。

### 2.7 读取与 miss 自愈（决策 2）

读路径沿用现有对象分支：`GetBatch` 的 `size` 取 `desc.object_size`（`distributed_storage_backend.cpp:817-818`），已有双层长度校验（`:838-841` + `kvcs_capi_driver.cpp:319-323`）。

`ReconcileMissingKvcsReplica` → `ReconcileMissingKvReplica`，门控从 `UsesKvcs()` 放宽。触发条件不变：仅 `OBJECT_NOT_FOUND`（`client_service.cpp:1624-1627`、`:1935-1939`），超时与其他 I/O 错误保留元数据不动。

Master 侧 `EvictDiskReplicaImpl` 的 DFS 是**条件性**白名单：

```cpp
const bool supports_dfs_condition = replica_type == ReplicaType::DFS &&
                                    expected_replica_id != nullptr &&
                                    use_kvcs_;                              // :6438-6440
```

必须放宽为 `use_kvcs_ || use_kvclient_`（`:6440`、`:6457`），否则 KVClient 下每次 miss 清理都返回 `INVALID_PARAMS`，坏记录永久残留、读取持续失败。CAS 语义本身与 v1 描述一致（`:6463-6473`）。

### 2.8 删除顺序（决策 5）

v1 的「先删元数据再删 value」改为**先删 value 再删元数据**。

**理由**：KVClient 的三条孤儿回收通道全断，metadata-first 的孤儿是永久且无界的。

| 回收通道 | KVCS | KVClient |
| --- | --- | --- |
| 水位淘汰（`KvcsDfsEviction::Reclaimer`，`master_service.cpp:423-434`） | ✅ 先删 value 后改元数据，失败则元数据保留待下轮重试 | ❌ 依赖 `kvcs_ll_mp_stat` 容量查询；KVClient 无容量 API → `usage.enabled=false` → 永不触发（`:7999-8004`） |
| Master 侧自持 driver | ✅ `CreateKvcsCapacityProvider` 在 Master 进程建 driver（`kvcs_capacity_provider.cpp:61-67`） | ❌ Master 进程通常无 RDMA 设备 / view service |
| key 枚举扫描 | ❌ | ❌ `ListKeys` → NOT_SUPPORTED |
| 同 key 重写 pre-delete | ✅ 偶然自愈 | ✅ 偶然自愈（唯一残存通道） |

**新顺序**：

```
① BatchDeleteObjects(全部 keys)          ← 先删 value
② 对成功项（含幂等 NOT_FOUND）调 master_client_.BatchRemove
③ 回填原下标，失败项保留各自错误
```

中间态「value 已删、元数据尚在」**由决策 2 自愈**：读到 `NOT_FOUND` → `ReconcileMissingKvReplica` → `EvictDiskReplicaIfCurrent` → Master 核对 ReplicaID 后移除 DFS 记录。**决策 5 依赖决策 2 必须同时落地**，否则中间态无人清理。

⚠️ **已知窗口（需接受）**：`RemoveObject` 的 `CheckRemovable` 会以 `OBJECT_HAS_LEASE` / `REPLICA_IS_NOT_READY` / `OBJECT_HAS_REPLICATION_TASK` 拒绝删除（`master_service.cpp:7401-7406`、`:7445-7463`）。value-first 下，value 已删但 Master 拒绝删元数据 → 一个仍在租约保护中的已发布对象变成不可读。

对 KV cache 而言这是「缓存命中变未命中」，不返回错数据，语义安全；但确实丢了已成功 Put 的数据。考虑到 metadata-first 的孤儿是永久无界泄漏，此取舍成立，须写进已知限制。

**决策 5 的连带必改项**：`RemoveAll` / `RemoveByRegex` 的 `NOT_SUPPORTED` 门必须一起放宽到 KVClient（`client_service.cpp:3843`、`:3864`）。这两个入口只删元数据、无 provider 清理，漏改等于一次性泄漏整个 L3。

### 2.9 元数据恢复（决策 6）：不支持

恢复能力整体挂在 HA leader 切换上，两个入口均为 HA 事件驱动：

| 入口 | 调用方 | 触发 |
| --- | --- | --- |
| `RestoreFromStandbySnapshot` | `rpc_service.cpp:2034` | standby 从 primary 接管 |
| `RestoreFromBatchOpLogPromotion` | `master_service_supervisor.cpp:799` | supervisor 提升新 leader |

两者汇聚到 `RestoreFromStandbyStateCore`（`master_service.cpp:3818`）。默认 `enable_ha=false`、`enable_oplog=false`、`enable_oplog_snapshot=false`，且 `enable_oplog` 强制要求 `enable_ha=true`，否则 `LOG(FATAL)`（`master.cpp:1506-1507`）。

恢复链路对 DFS descriptor 有三道 `use_kvcs_` 门，**v2 全部不改**：

| 门 | 位置 | 漏改后果 |
| --- | --- | --- |
| 恢复入口前置 | `master_service.cpp:3825-3828` | — |
| legacy standby 校验 | `:3533-3538` | 跳过该对象 |
| chunked metadata_store 校验 | `:4118-4126` | **整个恢复返回 INVALID_PARAMS** |

保持 `use_kvcs_` only 的现有行为是安全的：门 3825 会让 KVClient 的恢复**响亮失败**（`DFS_SERVICE_UNAVAILABLE`）而非静默丢数据。

**新增启动期拒绝**（把失败点从「崩溃后恢复时」提前到「启动时」）：

```cpp
// InitDfsAllocatorFromEnvironment，master_service.cpp:872 附近
if (dfs_config.UsesKvClient() && enable_oplog_) {
    LOG(FATAL) << "KVClient backend does not support metadata recovery; "
                  "enable_oplog must be false";
}
```

**默认部署行为**：Master 重启即对象目录全丢。L3 里的 value 仍在，但无任何索引可发现（不可枚举），等同丢失且永久占用存储空间。运维需知晓。

---

## 3 实现

### 3.1 改动清单

按决策 1，Master 侧需要**两个独立 flag**（`use_kvcs_` / `use_kvclient_`），因为决策 6 要求恢复门保持 KVCS-only。

#### 新增分支（不可放宽）

| 位置 | 改动 |
| --- | --- |
| `include/config/distributed_storage_config.h:27-31` | `UsesObjectStorage()` 列表加 `"kvclient"`；新增 `UsesKvClient()` |
| `src/config/distributed_storage_config.cpp:63-77` | KVClient 专属 `Validate()` 分支：`single_tenant` 必需；**不**要求 `eviction_check_interval`（无容量 API） |
| `src/storage_backend.cpp:6506` | 工厂新增 `UsesKvClient()` 分支：创建 driver → 注入 `KvcsObjectStorageAdapter` |
| `src/master_service.cpp:872-907` | 新增 `use_kvclient_` 分支：**不**加载 EFC target、**不**建 capacity provider、**不**调 `InitKvcsDfsEviction`；置 `enable_dfs_=true` 后 return。含 §2.9 的启动期拒绝 |

#### 放宽（语义相同）

| 位置 | 改动 |
| --- | --- |
| `src/master_service.cpp:847` | `enable_dfs_` 条件加 `\|\| UsesKvClient()` |
| `src/master_service.cpp:5090` | descriptor 分配：`use_kvcs_ \|\| use_kvclient_` |
| `src/master_service.cpp:5249` | `PrepareWriteStart` 门同上；**并加 512 对齐校验**（§2.3）与三副本拒绝（§2.6） |
| `src/master_service.cpp:6440`、`:6457` | `EvictDiskReplicaImpl` 的 DFS 条件白名单（§2.7） |
| `src/client_service.cpp:584` | `AttachHostId` 强制 `dfs_replica_num=1` |
| `src/client_service.cpp:3816` | `Remove` 路由到 `BatchRemove` |
| `src/client_service.cpp:3843`、`:3864` | `RemoveByRegex` / `RemoveAll` → `NOT_SUPPORTED`（§2.8 连带） |
| `src/real_client.cpp:1608`、`src/file_storage.cpp:126`、`src/config/file_storage_config.cpp:240` | 后端创建与注入链 |
| `include/storage/distributed/distributed_storage_backend.h:70-71` | 加平级的 `UsesKvClient()` 组合判断 |

#### 重构

| 位置 | 改动 |
| --- | --- |
| `src/client_service.cpp:3879-3912` | `BatchRemove` 的 KVClient 分支改为 value-first（§2.8） |
| `src/client_service.cpp:5299`、`:1626`、`:1936` | `ReconcileMissingKvcsReplica` → `ReconcileMissingKvReplica`，门控放宽（§2.7） |
| `src/storage/distributed/kvcs/kvcs_object_storage_adapter.cpp:57-65`、`:155-170` | `EncodeKey` 与 Delete-before-Put 下推为 driver 职责（§2.4） |

#### 不改（决策 6）

`src/master_service.cpp:3533`、`:3825`、`:4118` —— 保持 `use_kvcs_` only。

### 3.2 新增文件

```
mooncake-store/include/storage/distributed/kvclient/kvclient_driver.h
mooncake-store/src/storage/distributed/kvclient/kvclient_driver.cpp
```

`KvClientDriver final : public KvcsDriver`，职责：

- `Init()`：`CreateKvClient()` + 一次性注册 staging region，取得 canonical `MRHandle`
- `MaxValueSize()` → `16776704`；`MaxKeySize()` → 按 §2.2 裁决
- `GetMountpointStats()` → 沿用基类默认 `NOT_SUPPORTED`（触发 Master 既有降级）
- `BatchPut` / `BatchGet` / `BatchDelete`：logical key → CacheKey；staging 拷贝；`BatchStoreAsync` / `BatchLoadAsync` / `DeleteAsync` + `Wait`；超时 slot quarantine

**SDK API 选择**：只用 `BatchStoreAsync` / `BatchLoadAsync` / `DeleteAsync` / `QueryAsync`，**不用**单条 `StoreAsync` / `LoadAsync`——`GetSqeIoNum` 对 LOAD/STORE 强制返回 1（`io_scheduler.cc:82`），单条 API 在线格式层没有批量能力，N 个 entry 就是 N 个 SQE。

批量上限（`kv_types.h:196-199`）：写/读 110、删 254、查 256。按对象边界切批，绝不切开单个对象。

跨对象合批安全：SDK 自动按 nodeId 拆分 transport task 并用 `originalIndices` 回填（`task_manager.cc:266-302`、`:196-200`），`entryStatus` 与输入严格一一对应。

### 3.3 构建与可选依赖

沿用 KVCS 的可选依赖模式（`src/CMakeLists.txt:179-197`）：

```cmake
pkg_check_modules(KVCLIENT QUIET IMPORTED_TARGET "kvclient>=X.Y")
if(TARGET PkgConfig::KVCLIENT)
  set(MOONCAKE_KVCLIENT_SDK_FOUND ON)
endif()
```

SDK 缺失时 driver 提供显式 `NOT_SUPPORTED` stub，其他后端照常构建。`mooncake_store_shared_objects`（`:570-574`）与 `mooncake_store`（`:756-758`）需条件链接；**`mooncake_store_master` 不需要**（决策 6：Master 无 driver）。

---

## 4 测试

### 4.1 验证场景

1. 两副本（memory + ASU）写入成功；内存副本淘汰、写入 Client 退出后，另一 Client 从 L3 读取原数据。
2. DFS-only 完成写入、读取和显式删除。
3. **硬逻辑**：注入 ASU 写失败 → 整个 Put 失败，且 memory 副本被一起 revoke（Master 侧查不到该对象）。
4. **硬逻辑短路**：注入 memory 写失败 → ASU 零调用（driver 侧计数器断言），Put 失败。
5. **512 对齐**：N 非 512 倍数的 Put 在 `PrepareWriteStart` 返回 `INVALID_PARAMS`，不触达 driver。
6. **大小上限**：N = 16,776,704 成功；N = 16,777,216 被拒（driver `MaxValueSize`）。
7. **三副本拒绝**：`memory + nof + ASU` 返回 `INVALID_PARAMS`（若采纳 §2.6 建议）。
8. **删除顺序**：注入 value 删除成功但 Master `BatchRemove` 失败 → 元数据保留 → 后续读取触发 `ReconcileMissingKvReplica` → 记录被清理。
9. **删除孤儿窗口**：注入 value 删除成功但对象处于 lease 中 → `CheckRemovable` 拒绝 → 验证行为符合 §2.8 已知窗口（数据不可读但不返回错数据）。
10. **miss 自愈 CAS**：Client 观察到 R1 的 miss 后 Master 已发布 R2 → 条件清理返回不匹配，R2 保留。
11. **超时 quarantine**：注入 SDK `Wait` 超时 → 验证 ① 不读 `entryStatus` ② slot 未归还 ③ 调用者 buffer 未被复用 ④ 后续同 slot 不复用直到排空。
12. **`RemoveAll` / `RemoveByRegex`**：KVClient 下返回 `NOT_SUPPORTED`，且**元数据未被删除**。
13. **批量部分失败**：写/读/删部分失败时，结果与原 key 下标严格对应。
14. **恢复不支持**：`UsesKvClient() && enable_oplog=true` 时 Master 启动 `LOG(FATAL)`。
15. **SDK 缺失构建**：无 KVClient SDK 时其他后端正常构建，显式初始化返回 `NOT_SUPPORTED`。
16. 分层测量 driver staging 拷贝、SDK I/O、Master RPC 耗时，确定推荐 I/O 粒度。

### 4.2 复用既有测试

`kvcs_object_storage_adapter_test.cpp` 已有 driver 注入模式（`MOONCAKE_KVCS_TEST_NO_SDK` / `MOONCAKE_KVCS_TEST_SDK`，`tests/CMakeLists.txt:416-424`）。`KvClientDriver` 可用同一 fake-driver 注入缝测试适配器层，无需真机。

真机必需项：§2.4 的「ASU Store 支持覆盖写」假设、§2.5 的超时后设备行为、实际 `singleValueMaxBytes`（只打日志，需读 Init 日志）。

---

## 5 已知限制

| 限制 | 说明 | 缓解 |
| --- | --- | --- |
| 删除孤儿窗口 | value-first 下，`CheckRemovable` 拒绝会留下不可读的已发布对象 | 语义安全（不返回错数据）；KV cache 场景可接受 |
| 无容量管理 | KVClient 无容量查询 API → Master 水位淘汰不生效 | 沿用 `:883-889` 降级；容量控制留待 SDK 提供接口 |
| 不支持元数据恢复 | Master 重启即目录全丢，L3 数据不可发现 | 启动期显式拒绝 `enable_oplog`；运维需知晓 |
| 不支持 key 枚举 | `ListKeys` / `RemoveAll` / `RemoveByRegex` 返回 `NOT_SUPPORTED` | 同 key 重写时 pre-delete 偶然清理 |
| 不支持范围读 | `BatchRead` 对象分支忽略 `range_read`，slice 容量 < object_size 时 `INVALID_PARAMS`（`distributed_storage_backend.cpp:802-818`） | 上层回退整对象读后截取 |
| driver 内一次 memcpy | §2.5 的同步契约强制 staging | 零拷贝需 SDK 提供真正的设备级中止能力 |
| 单对象上限 16,776,704 B | SDK 线格式 24-bit length + 512 对齐 | 无 |

---

## 6 代码依据

Mooncake `77689203`；KVClient SDK `e6717d04`。

| 引用 | 路径 |
| --- | --- |
| 适配器接口 | `mooncake-store/include/storage/distributed/object_storage_adapter.h` |
| KVCS 适配器 | `mooncake-store/src/storage/distributed/kvcs/kvcs_object_storage_adapter.cpp` |
| driver 抽象 | `mooncake-store/include/storage/distributed/kvcs/kvcs_driver.h` |
| 后端 | `mooncake-store/src/storage/distributed/distributed_storage_backend.cpp` |
| Client | `mooncake-store/src/client_service.cpp` |
| Master | `mooncake-store/src/master_service.cpp` |
| 副本与写模式 | `mooncake-store/include/replica.h` |
| SDK 接口 / 类型 | `kv_semantics/include/kv_client.h`、`kv_types.h` |
| SDK 任务管理 | `kv_semantics/src/client/task/task_manager.cc`、`src/client/kv_client_impl.cc` |
| SDK 传输 | `kv_semantics/src/trans/task/trans_task_executor.cc`、`src/trans/task/io_scheduler.cc` |
