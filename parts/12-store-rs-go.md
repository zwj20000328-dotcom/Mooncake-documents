# 12 Rust / Go 实现

> 精读范围：`mooncake-store-rs/`（Rust 原生 Store，内部称 Store-RS）、`mooncake-store/rust/`（C++ Store 的 Rust 绑定）、`mooncake-store/go/`（同一份 C ABI 的 Go 绑定），以及 `docs/source/deployment/store-rs/`。
> 文中所有 `路径:行号` 均相对仓库根 `D:/xiaojun/work/Mooncake`，行号以本次精读时的 main 分支（`67df8c41` 附近）为准。

## 定位与职责

这个标题下其实塞了**三份互不相同的代码**，先把它们分开，否则后面每一句话都会讲串：

| 代码 | 语言/形态 | 行数量级 | 运行时依赖 | CMake 开关 |
|---|---|---|---|---|
| `mooncake-store-rs/` | 14 个 crate 的 Rust 原生实现（Store-RS） | ~18.3 万行 Rust（`find crates -name '*.rs' \| xargs wc -l`） | Redis/etcd + 对端 client；**不需要 master** | `WITH_STORE_RS`，默认 **OFF**（`CMakeLists.txt:29-32`） |
| `mooncake-store/rust/` | crate `mooncake_store`，约 2.4k 行的薄安全绑定 | `src/` + `build.rs` + examples/tests 合计 2461 行 | `mooncake_master` + metadata server + `libmooncake_store` | `WITH_STORE_RUST`，默认 **ON**（`CMakeLists.txt:26`） |
| `mooncake-store/go/` | package `mooncakestore`，CGo 绑定 | 约 977 行（含 `tests/`、`examples/`） | 同上 | `WITH_STORE_GO`，默认 **OFF**（`CMakeLists.txt:21`） |

后两者是**同一个东西的两种语言前端**：都只是 `mooncake-store/include/store_c.h` 那 19 个 C 函数的安全封装，实际语义（路由、副本、驱逐、租约）全部由 C++ 的 `RealClient`/`MasterService` 决定。Store-RS 则是**从零重写的另一套 Store**，它与 C++ 的唯一真实接触面是数据面（Transfer Engine / TENT 的 `.so`），通过运行时 `dlopen` 装配。

用一句话概括三者关系：

```text
                 ┌─────────────────────────────────────────────┐
Python/CLI/e2e → │ Store-RS (mooncake-store-rs, masterless)    │
                 │   route mesh + metadata backend + tenant    │
                 │   └─ dlopen libmooncake_{classic,tent}_shim  │
                 └──────────────────┬──────────────────────────┘
                                    │ 数据面
                       C++ Transfer Engine / TENT (.so)

Rust/Go 用户 → mooncake_store / mooncakestore（薄绑定）
                 └─ store_c.h → store_c.cpp → RealClient → mooncake_master（C++ 元数据主）
```

**为什么会有两套 Rust**：`mooncake-store/rust` 出现的动机是"让 Rust/Go 生态能调用已有的 C++ Store"，成本低但受制于 master；`mooncake-store-rs` 的动机写在 `docs/source/design/store/store-rs/architecture.md:5-11`——保留 Mooncake 的编程模型、复用 TE/TENT 数据面、**把 master 从路由热路径上去掉**。这两条路线互相独立，代码零共享，连构建入口都不同（Cargo workspace vs 单 crate）。

Store-RS 的定位细节（必须记住的三条不变式）：

1. **热路径不许打后端**。`architecture.md:13-20` 原文："Steady-state request hot paths must not depend on backend round trips." Redis/etcd 只在租约、segment 公告、路由策略、`MetadataOnly` 模式等控制面上出现；正常 put/get 的每一次路由查询走的是 client 之间的 authority mesh。
2. **路由与存储分离**。route owner（authority）负责"对象在哪"的权威版本，storage owner 负责"字节在哪、还剩多少容量"；两者通过 CAS + `BatchTrackReplicaRoutes` 解耦（`architecture.md:239-251`）。
3. **元数据后端只做四类持久化**：client lease、segment 公告与生命周期、需要持久化的 object route、tenant 策略与配额（`architecture.md:394-403`）。

能力覆盖上，Store-RS 远多于两个绑定。Store-RS 有：单/批量 put/get/remove/exist/size、注册缓冲区零拷贝、multi-buffer 读写、scatter-gather range read、张量并行重建（`mooncake-store-rs-tensor`）、多租户与严格配额、CLOCK 驱逐 + route-owner CAS 回收、冷层（Cold Tier）与 NoF、滚动升级 handoff/drain、admin HTTP、Prometheus/OTLP。两个绑定则严格等于 `store_c.h` 的 19 个函数对应能力，没有租户、没有配额、没有 handoff。

## 目录结构速览

### `mooncake-store-rs/`（Store-RS）

| 路径 | 一句话职责 |
|---|---|
| `mooncake-store-rs/Cargo.toml:1-18` | workspace 成员清单，14 个 crate，`resolver = "2"` |
| `mooncake-store-rs/Cargo.toml:25-26` | **`warnings = "deny"`**：读代码时看到的 `#[allow(dead_code)]` 都是刻意标注而非历史垃圾 |
| `mooncake-store-rs/CMakeLists.txt:7-12` | 硬前置：必须有 `transfer_engine` 与 `tent_shared` 两个 target |
| `mooncake-store-rs/CMakeLists.txt:37-69` | 只编两个 shim：`libmooncake_classic_shim.so`、`libmooncake_tent_shim.so` |
| `mooncake-store-rs/CMakeLists.txt:81-119` | 两条 Cargo 构建线：`build_store_rs_cli`（无 PyO3）与 `build_store_rs_python_extension` |
| `mooncake-store-rs/CMakeLists.txt:122-133` | wheel 只装 shim + `_store_rs.<soabi>.so` + 三个 bin |
| `crates/mooncake-store-core/src/traits.rs` | 三个核心 trait：`MetadataBackend` / `RouteDirectory` / `PlacementStrategy` |
| `crates/mooncake-store-core/src/route.rs` | 路由与租户词汇表：`ObjectKey`/`ObjectRoute`/`ReplicaRoute`/`ClientLease`/`CasResult` |
| `crates/mooncake-store-core/src/identity.rs` | `ClientStableId` / `ClientEpoch` / `ClientRuntimeId` / `NamespaceScope` |
| `crates/mooncake-store-client/src/client/mod.rs` | `StoreClient` 组装点与全部默认常量 |
| `crates/mooncake-store-client/src/client/builder.rs` | `StoreClientBuilder::build`：启动校验、租约、策略引导、后台句柄 |
| `crates/mooncake-store-client/src/client/facade.rs` | `MooncakeCompatibilityFacade`：54 个方法（**不是**粗读地图说的 119 个） |
| `crates/mooncake-store-client/src/client/runtime_write.rs` | put/batch_put 与远端副本写 |
| `crates/mooncake-store-client/src/client/runtime_io.rs` | get/batch_get、按 segment 分组、本地/远端分派、迁移读 |
| `crates/mooncake-store-client/src/client/runtime_alloc.rs` | 本地/远端分配预约 |
| `crates/mooncake-store-client/src/client/state_store.rs` | `StorageOwnerState` 的实现（结构体本体在 `state_core.rs:329`）：CLOCK、track_route、evict_one |
| `crates/mooncake-store-client/src/client/membership_sync.rs` | 后台刷新 live-client 快照与 tenant 策略缓存 |
| `crates/mooncake-store-client/src/control_plane/` | tonic 版控制面 client/server（路由/分配/驱逐/迁移/冷层） |
| `crates/mooncake-store-client/proto/control_plane.proto` | 30 个 rpc + 双向 `ControlStream`（`control_plane.proto:5-35`） |
| `crates/mooncake-store-client/build.rs` | workspace 里唯一发 link 指令的地方：protoc 生成 + 可选 `kvcs-capi`/`nof-spdk` |
| `crates/mooncake-store-route/src/shim/` | `RouteOperations`、`build_route_directory`、元数据适配 |
| `crates/mooncake-store-route/src/table/` | 单 authority 的本地路由表与 CAS |
| `crates/mooncake-store-route/src/mesh/` | Embedded WRH：选主、镜像、读修复、ranked fallback |
| `crates/mooncake-store-rs-transport-sys/` | `dlopen`/`dlsym` 装配层（classic/tent 两个函数指针表） |
| `crates/mooncake-store-rs-transport/` | `TentEngine` / `ClassicTransferEngine` 的安全封装 |
| `crates/mooncake-store-transport-core/src/lib.rs` | 只有 107 行，却定义了 `StoreTransport`/`StoreTransportFactory` 两个 trait |
| `crates/mooncake-store-rs-metadata/` | Redis/etcd/in-memory 三套 `MetadataBackend` 实现 |
| `crates/mooncake-store-rs-runtime/` | Python 兼容运行时：`CompatRuntimeArgs`、`StoreDispatcher`、shm、hot cache、dummy 服务 |
| `crates/mooncake-store-py/` | PyO3 扩展 `_store_rs`（`lib.rs` 只有 2 行 mod 声明） |
| `crates/mooncake-store-rs-cli/` | 三个命令：`mooncake-store-rs-client` / `-bench` / admin |
| `crates/mooncake-store-e2e/` | 1951 行的可运行系统验证 + `bin/nof_multi_client.rs` |
| `mooncake-store-rs/scripts/{e2e,tests,build,lib}/` | shell 级 e2e（route migration、hot cache、NoF、redis ttl recovery 等） |

### `mooncake-store/rust/` 与 `mooncake-store/go/`

| 路径 | 一句话职责 |
|---|---|
| `mooncake-store/rust/src/lib.rs:44-69` | feature 门禁：至少 `link`/`dlopen` 之一；两者都开时 `link` 胜出 |
| `mooncake-store/rust/src/store.rs:27-38` | backend 无关的关键：`cfg` 二选一，`include!(OUT_DIR/bindings.rs)` 或 `ffi_dlopen` |
| `mooncake-store/rust/src/store.rs:85-124` | `ReplicateConfig::to_ffi`：CString/指针数组的所有权契约 |
| `mooncake-store/rust/src/store.rs:639-645` | `Drop` 调 `mooncake_store_destroy` |
| `mooncake-store/rust/src/ffi_dlopen.rs:49-111` | 平台 `DEFAULT_LIBRARY` 常量、进程级 `OnceLock<MooncakeStoreLib>` + `INIT_LOCK` + `load_library`/`ensure_loaded` |
| `mooncake-store/rust/build.rs:190-191` | 纯 `dlopen` 构建时 `build.rs` 什么都不做 |
| `mooncake-store/rust/build.rs:193-457` | `link` 构建：大量 `rustc-link-search`（含从 `CMakeCache.txt` 解析路径）、17 条无条件 `rustc-link-lib`（`mooncake_store`/`transfer_engine`/`base`/`asio`/…）外加按存在性条件追加的 `asan`/`etcd_wrapper`/`hiredis`/`curl`/`cuda`/`cudart`/`mlx5`/`uring`/`zmq`/`crypto`/`gcov`，最后跑 bindgen |
| `mooncake-store/rust/src/generated/ffi_dlopen_bindings.rs` | **提交进仓库的生成物**（397 行），dlopen 后端的真实 ABI |
| `mooncake-store/go/mooncakestore/store.go` | 19 个 `*Store` 方法 + `New` + `toCConfig` |
| `mooncake-store/go/mooncakestore/config.go` | `ReplicateConfig` / `SoftPinAction`（Go 侧独立定义） |
| `mooncake-store/go/mooncakestore/errors.go` | 14 个哨兵错误 |
| `mooncake-store/go/build.sh:30-91` | CGO flags：`-Wl,--start-group` 里链 6 个静态库（`mooncake_store`/`mooncake_local_ssd`/`cachelib_memory_allocator`/`transfer_engine`/`base`/`mooncake_common`）+ 一堆系统库 |

## 核心数据结构

### Store-RS：三个 trait 是整个抽象的地基

`crates/mooncake-store-core/src/traits.rs` 只有 933 行，却定义了全部可替换边界。

**`MetadataBackend`**（`traits.rs:21-408`）——持久化面。方法大致分六组（另有冷层设备记录、object-id 查询、版本 floor 等零散方法未单列）：

- 租约：`allocate_client_lease` / `upsert_client_lease` / `update_client_state` / `get_client_lease` / `list_live_clients`；
- segment：`publish_segment` / `unpublish_segment` / `reserve_segment` / `release_segment` / `update_segment_state`；
- 路由：`get_object_route` / `compare_and_swap_object_route` / `list_object_routes_by_cold_backing` / `list_object_routes_by_nof_backing`；
- 策略与租户：`get_route_policy` / `put_route_policy_if_absent` / `get_tenant_policy(s)` / `put_tenant_policy`；
- 配额：`reserve_tenant_quota` / `finalize_tenant_quota` / `abort_tenant_quota` / `list_tenant_eviction_candidates`；
- handoff：`put_handoff` / `get_handoff`。

最值得读的是 `traits.rs:38-67` 两段文档注释，它们定义了 masterless 正确性的前提：

- `upsert_client_lease`：**必须拒绝** `epoch` 不大于该 `stable_id` 历史观测值的写入，除非 lease key 已存在（视为 refresh）。唯一的例外是"lease 已过期消失、但没有更高 live epoch"时的同 epoch 回收。
- `allocate_client_lease`：调用方**不提供 epoch**；实现方用 `max(active epochs, historical HWM) + 1` 原子分配，并在同一原子步内提交 lease key、per-stable-id 活跃索引和 HWM。

三套实现都照此契约写，但手段不同：in-memory 靠一把写锁；Redis 用一段 Lua（脚本常量 `ALLOCATE_CLIENT_LEASE_SCRIPT` 定义在 `crates/mooncake-store-rs-metadata/src/redis_backend.rs:309`，调用点在 `:1704-1718`，6 个 key + 5 个 arg，返回分配到的 epoch 字符串）；etcd 用多 key txn 循环。**之所以要这么严格**：没有 master 做仲裁，epoch 单调性就是"谁是当前活着的同一个 client"的唯一依据；一旦允许回退，旧进程的租约可能覆盖新进程，路由会指向已死 runtime。

**`RouteDirectory`**（`traits.rs:410-578`）——路由查询/发布面，几乎所有方法都带 `observer: &ClientLease`（`metadata()` 例外）。这个参数不是摆设：authority 候选集要按 observer 的 compatibility 与 `route_scope` 标签过滤，等于"每个 client 看到的路由拓扑可以不同"。几个刻意的默认实现：

- `get_object_routes_bounded`（`:431-437`）：存在性检查的专用路径，**不做穷尽 fallback 探测**。注释写得很直白："a miss is common and must not fan out to every route authority in the cluster"。
- `compare_and_swap_local_route`（`:490-498`）：绕过 mesh 选主的逃生口。
- `visit_routes_by_replica_owner`（`:511-528`）：兼容路径有 256 条硬上限，超出直接 `Unsupported`，逼调用方走分页 visitor。

**`PlacementStrategy`**（`:580-587`）：只有一个 `select_write_targets`，输入本地 runtime、live clients、副本数，输出目标 runtime 列表。它很小，因为真正的放置热路径在 `mooncake-store-client/src/placement.rs` 的 `PlacementPlanner` 里，这个 trait 更像"可注入策略"的最小契约。

### Store-RS：路由词汇表（`mooncake-store-core/src/route.rs`）

- `ObjectKey(pub String)`（`:16`）：**不是**用户 key，而是 scope 编码后的全局 key。`from_scope`（`:23-38`）在默认 domain/object_set 下编码成 `"<tenant>::<key>"`，否则 `"<tenant>::ns/<domain>/<object_set>/<key>"`；每段都过 `encode_key_component`（`:45-59`）做百分号转义，只保留 `[0-9A-Za-z-_.]`。这解释了为什么 Store-RS 能天然多租户而 C++ 绑定不能——tenant 被编进了物理 key。
- `RouteVersion(pub u64)`（`:73-79`）：单调递增，`next()` 用 `saturating_add`。所有写路径都是"读当前 version → 构造 next → CAS(expected=旧 version)"。
- `RouteState`（`:82-88`）：`Active` / `Deleting` / `Tombstone`。注意 `Tombstone` 已注明 "Legacy: no longer produced by local remove/eviction paths"，只为线兼容保留。
- `ReplicaRoute`（`:107-118`）：**两个 offset 是本章最容易讲错的地方**。
  - `offset: Option<u64>` —— 传输目标地址（TE/TENT 的 target_offset），是 payload 在 segment storage target map 中的字节位置；
  - `segment_offset: u64` —— 本地分配器/reclaim 使用的偏移。
  二者在"transport target 坐标 ≠ 进程虚拟地址"时**不相等**，本地直读也必须走 `offset` 再映射回本地注册区（架构文档 `:194-202`）。
- `ObjectRoute`（`:121-141`）：除 key/version/state/replicas 外，还带 `namespace`、`logical_key`、`canonical_key`、`sharing_scope`（共享/复用标识字符串）、`qos_tier`、`cold_backing`、`nof_backing`，以及 `compatibility: CompatibilityDescriptor`（参与 authority 选择与路由身份比较，常被漏看）。
- `ClientLease`（`:143-150`）：`runtime` + `state` + `compatibility` + `endpoints` + `expires_at_ms`。**租约即成员关系**：没有独立的成员表，`list_live_clients()` 就是"列出未过期 lease"。
- `SegmentAnnouncement`（`:152-173`）：容量、已用、`target_chunks`、`alignment_bytes`、`tags`，以及 `transport_endpoint` / `transport_segment_descriptor`（classic TE P2P 场景下用来免掉 Redis 段元数据查询，见架构文档 `:186-192`）。
- `CasResult`（`:190-196`）：`applied` + `current` + `version_floor`。`version_floor` 专门处理"我读的时候这个 key 还不存在，CAS 时发现已有人建了更高版本"的竞争。
- `RouteControlMode`（`:205-210`）：`MetadataOnly` / `EmbeddedWrh`（默认）。

### Store-RS：`StoreClient` 字段即子系统地图

`crates/mooncake-store-client/src/client/mod.rs:144-193`，一次读完就知道客户端持有什么：

```rust
metadata, route_directory, _control_plane, control_client,
allocator: Arc<Mutex<LocalAllocatorState>>,
storage_owner: Arc<StorageOwnerState>,
lease: ClientLease, lease_ttl_ms: u64,
live_client_cache, suspect_runtime_cache, membership_sync,
_async_eviction, async_replica_tracking, async_route_hit_reporting,
default_tenant, local_memory, transport, transport_factory,
write_mode, route_control, route_topk,
transfer_stall_timeout, request_timeout_override,
lifecycle_state, route_write_gate, startup_activation_pending,
heartbeat_repair_pending, tenant_quota_reservation_counter,
namespace_quota, execution_fairness, bandwidth_shaping, placement_policy,
state: Arc<Mutex<StoreState>>,
owns_local_state_lifecycle, owns_cold_tier_lifecycle, cold_tier_shutdown_mode,
restore_promotions, cold_tier, cold_restore_flights, deferred_cold_tier_reconciles,
```

配套默认值（`mod.rs:57-92`）读出来的运行参数：`route_topk = 2`、live-client 同步 `1s`、suspect runtime TTL `5s`、传输 stall 超时 `10s`、请求超时基/上限 `1s`/`60s`、吞吐下限 `32MiB/s`、put 重试 4 次、segment publish 租约 `30s`、tenant 配额预留 TTL `60s`、tenant 策略缓存 TTL `1s`/空闲 `30s`。

三个 `#[allow(dead_code)]` 字段（`:183-192`）：`owns_cold_tier_lifecycle`（`:184`）与 `cold_tier_shutdown_mode`（`:187`）确实只给 `Drop` 用；`deferred_cold_tier_reconciles`（`:192`）的 allow 则略陈旧——它在 `runtime_core.rs:87`（启动激活完成时）也被读取，并非纯 Drop 字段。这正是 `warnings = "deny"` 下的常见写法，但读的时候要分清哪条 allow 仍然必要。

### 绑定侧：`MooncakeStore` / `ReplicateConfig` / `mc_store_t`

Rust 绑定的核心只有两个结构体：

- `MooncakeStore`（`mooncake-store/rust/src/store.rs:145-147`）：`handle: ffi::mooncake_store_t`，即 `*mut c_void`。`:151-152` 手写 `unsafe impl Send/Sync`，理由是"底层 C 对象内部已同步"。`Drop`（`:639-645`）调 `mooncake_store_destroy`。
- `ReplicateConfig`（`:64-77`）：`replica_num` / `soft_pin_action` / `soft_pin_ttl_ms` / `with_hard_pin` / `preferred_segments`。它 `#[derive(Default)]`——**这一点后面会变成一个坑**。

`to_ffi`（`:85-124`）是所有权教学样本：先把 `Vec<String>` 转成 `Vec<CString>`，再从中取 `Vec<*const c_char>`，最后把 `ffi::mooncake_replicate_config_t` 和两个 Vec **一起返回**。调用方必须同时绑定这三个值，否则指针悬垂。`store.rs:611-617` 的注释把这个约束写死了："The caller must bind the returned tuple to a named variable in its own stack frame *before* taking a raw pointer"。

Go 侧对应 `Store`（`mooncake-store/go/mooncakestore/store.go:29-32`）只持有 `handle C.mooncake_store_t`；`ReplicateConfig` 在 `config.go` 里独立定义（`ReplicaNum int`、`SoftPinTTLMs *uint64`），与 Rust 侧结构相同但**校验更严**（见后文）。

C 侧唯一契约 `mooncake-store/include/store_c.h`：

- `:25` `typedef void *mooncake_store_t;`
- `:33-37` `enum mooncake_soft_pin_action`：`PRESERVE=0` / `ENABLE=1` / `DISABLE=2`，零初始化即 PRESERVE
- `:39-54` `struct mooncake_replicate_config`：7 个字段（`replica_num`、`with_hard_pin`、`preferred_segments`、`preferred_segments_count`、`soft_pin_action`、`has_soft_pin_ttl`、`soft_pin_ttl_ms`）
- `:56-62` 全章的"所有权宪法"：**所有 `char *` 参数在函数返回后不再被使用**，调用方可立即释放。
- `:68-146` 19 个函数：create/destroy/setup/init_all/health_check、put/put_from/batch_put_from、get_into/batch_get_into、is_exist/batch_is_exist/get_size/get_hostname、remove/remove_by_regex/remove_all、register_buffer/unregister_buffer。

实现端 `mooncake-store/src/store_c.cpp`：

- `:34-36` `StoreHandle` 包 `std::shared_ptr<RealClient>`。注释解释了为什么不能只存裸指针：`RealClient::create()` 返回 shared_ptr，并在 ResourceTracker 里注册了 weak_ptr。
- `:42-70` `to_replicate_config`：先**校验 action 范围再 cast**（注释说明 enum 底层是 uint8_t，越界会回绕成合法值），`config.replica_num = c_config->replica_num` 是**原样透传**。
- 几乎所有导出函数都套 `try { ... } catch (...) { return -1; }`，把 C++ 异常挡在 ABI 之外。

### Store-RS：路由 authority 与成员快照

- `EmbeddedWrhRouteDirectory`（`crates/mooncake-store-route/src/mesh/directory.rs:62-71`）：`route_topk`、`namespace`、`local_stable_id`、`metadata`、`authority_client`、`membership`、`async_mirror`、`baseline: Mutex<BaselineState>`。
- `BaselineState`（`:56-60`）：`member_ids` + `candidates` + `created_at`，TTL 300s（`:54`）。它是"membership 抖动时的兜底候选集"。
- `LiveClientCache`（`crates/mooncake-store-client/src/client/state_core.rs:5-10`）：`refreshed_at` + `leases` + `tenant_quota_policies`。注意 `snapshot()`（`:13-16`）在 `refreshed_at` 为 None 时返回 None，上层会抛 `InvalidState("live client snapshot is not initialized; build() must prewarm membership before serving requests")`（`:173-182`）——所以 builder 里"先 prewarm 再提供服务"不是优化，是**正确性要求**。
- `shared_live_client_cache(namespace)`（`:184-193`）：进程级 `OnceLock<Mutex<BTreeMap<String, SharedLiveClientCache>>>`，**按 route_namespace 共享**。同一进程内多个 `StoreClient`（例如 Python 多 worker）只要连同一个 keyspace，就共用同一份成员快照和 tenant 策略缓存。
- `SuspectRuntimeCache`（`:76-167`）：被 quarantine 的 runtime。`is_recovered_by`（`:148-167`）先要求 quarantine 时间已过，再判：若此前没有观测快照直接恢复；否则三个判据（expires_at 变大、状态变成允许写入、control address 变化）任一成立即视为"同一个 stable_id 换了进程"。要让这些判据生效，membership 刷新必须先 `reconcile_suspects(leases)`。
- `HotCacheKey` / `LocalHotCache`（`crates/mooncake-store-rs-runtime/src/hot_cache.rs`）：`HotCacheKey` 在 `:16-19`（`(tenant, key)`），`LocalHotCache` 在 `:92`；由 `MC_STORE_LOCAL_HOT_CACHE_SIZE`（总容量）与 `MC_STORE_LOCAL_HOT_BLOCK_SIZE` 等环境变量启用，块大小默认 16MiB（`DEFAULT_BLOCK_SIZE` `:14`），另可用 `MC_STORE_LOCAL_HOT_CACHE_USE_SHM=1` 走共享内存。

## 关键流程剖析

### 流程 A：`StoreClientBuilder::build` —— 一个 client 如何加入集群

入口 `crates/mooncake-store-client/src/client/builder.rs:393`。这是全章最值得逐行读的函数，因为它把所有子系统的启动顺序固定下来了。

调用链（按执行顺序）：

```text
StoreClientBuilder::build(expires_at_ms)
├─ 参数校验                         builder.rs:394-403
├─ 标签归一化                        builder.rs:407-419 → normalize_storage_label(:87)
├─ endpoints 补全（rpc/segment）     builder.rs:420-432
├─ 解析 effective tenant policy      builder.rs:445-459
├─ 创建 state/allocator/lifecycle    builder.rs:460-469
├─ 创建 control_client               builder.rs:470
├─ validate_startup_conflicts        builder.rs:482 → builder.rs:7-28
├─ allocate_client_lease             builder.rs:483
├─ bootstrap_route_policy            builder.rs:491-497 → builder.rs:846-869
├─ build_route_directory             builder.rs:517-525
├─ 冷层设备 bootstrap/别名            builder.rs:539-633
├─ StorageOwnerState::new            builder.rs:634-653
├─ ControlPlaneHandle::spawn_*       builder.rs:683-690
├─ bind_local_authority_service      builder.rs:691-695
├─ upsert_client_lease（含 labels）   builder.rs:700-710
├─ prewarm_live_client_cache         builder.rs:711
├─ prewarm_segment_target_chunk_cache builder.rs:729-734
├─ MembershipSyncHandle::spawn       builder.rs:757-765
├─ AsyncEviction/ReplicaTrack/Hit    builder.rs:766-783
├─ ColdTierHandle::spawn/disabled    builder.rs:788-798
└─ 组装 StoreClient                  builder.rs:799-842
```

分步说明：

1. **启动校验**（`:394-403`）：`default_tenant` 不能为空；`route_topk >= 2`。后者不是拍脑袋——topk=1 意味着没有镜像 authority，读修复与 CAS 失败重试全部失效。
2. **存储角色归一化**（`:87-101`）：`storage=true` 但没有 `storage_bytes` → `InvalidState`；`storage_bytes=0` 且未显式声明 → 自动写 `storage=false`。同一段还补一个 `route` 标签（`:103-111`，`normalize_route_label`）：**仅在标签缺失时**按 `has_storage()` 填 `true`/`false`，用户显式设置的值不会被覆盖。**这个标签决定了它能否成为路由 authority**（`crates/mooncake-store-route/src/util.rs:12-18` 的 `route_capable` 只认 `labels["route"] == "true"`）。
3. **冲突检测**（`:482` → `:7-28`）：如果本地 segment 名与某个 *Active* 且 *可达* 的远端 runtime 冲突，直接失败。`validate_reachable_runtime`（`:63-85`）用 `control_client.probe_reachability` 探测；**探测失败（Unknown）也报错**，只有明确 Unreachable 才放行——这是防"脑裂后两个进程抢同一 segment"的保守策略。
4. **租约分配**（`:483`）：`allocate_client_lease(&template)`，模板里的 epoch 是 `ClientEpoch::default()`，会被后端忽略并替换成原子分配的新 epoch。
5. **路由策略引导**（`:491-497`，实现 `:846-869`）：第一个 client 用 create-if-absent 写入集群默认策略；随后所有 client 读回 effective 策略并**校验与本地一致**（`validate_route_policy`），不一致就启动失败。目的是防止不同 client 悄悄用不同的 `topk`/`route_control` 组合。
6. **构造路由目录**（`:517-525`）：把 `effective_route_control`/`effective_route_topk`、metadata、provisional lease、control client、共享快照、suspect 缓存一起交给 `build_route_directory`。
7. **控制面 server**（`:683-690`）：`ControlPlaneHandle::spawn_with_migration(host, local_authority, storage_adapter, storage_adapter, migration_adapter, cold_tier_control)`——注意 `storage_adapter` 传了两次，分别作为 allocator service 与 eviction service。这是"每客户端既是 RPC 客户端也是服务端"的实现点。
8. **成员快照预热**（`:711`）：`prewarm_live_client_cache` 必须成功，否则后续任何 `cached_live_client_snapshot` 都会报错。
9. **后台句柄**（`:757-783`）：membership sync、异步驱逐、异步副本跟踪、异步 route hit 上报，四条线程/任务在这里起。
10. **冷层**（`:788-798`）：只有真配置了 cold tier/NoF 才 `ColdTierHandle::spawn`，否则 `disabled` 占位。

一个容易忽略的细节：`startup_activation_pending`（`:434-443`）。当调用方要求 `initial_state=Active` 且开启 `activate_on_local_memory_registration` 时，**先以 Standby 发布租约**，等本地内存注册完成后再激活。这样别的 client 不会在内存还没注册好时把路由指向它。

### 流程 B：Store-RS 的 put —— 从一次写入到路由发布

调用链：

```text
facade.rs put/put_in_tenant (:1309/:1313)
└─ put_object (runtime_io.rs:663)
   ├─ route_ops().load_route(key)                 runtime_io.rs:677-680
   └─ put_object_with_policy_current              runtime_io.rs:413
      ├─ ensure_local_memory / flush_due_reclaims :420-421
      ├─ resolve_replication_policy               :449
      ├─ reserve_tenant_quota_for_put             :460
      ├─ reserve_replica_targets                  :471
      ├─ write_reserved_replicas                  :495 → runtime_write.rs:127
      ├─ 构造 ObjectRoute（version+1, replicas）   :528-566
      ├─ route_ops().publish_route (CAS)          :571-573
      ├─ track_remote_storage_owners_best_effort  :638
      ├─ finalize_published_route(quota)          :639-646
      ├─ storage_owner.track_route                :647
      └─ reclaim_route(previous)                  :648-658
```

分步：

1. **读当前路由**（`runtime_io.rs:677-681`）。在 `EmbeddedWrh` 下这一步是 authority RPC，不是 Redis 查询。
2. **QoS 与策略解析**（`:443-454`）：`qos_tier` 缺省继承旧路由；`resolve_replication_policy` 把请求策略与租户/默认策略合并；`ranked_candidate_count` 决定写重试上限。
3. **配额预留**（`:460-465`）：`reserve_tenant_quota_for_put` 在分配之前，因此"配额失败"不会浪费一次内存分配。注意顺序：**quota → alloc → write → publish → finalize quota**（架构文档 `:415-417` 明确写了这个顺序）。
4. **副本目标预约**（`:471-482`）：`reserve_replica_targets` 返回 `(targets, reservations)`；失败时先 abort 配额再返回。target 里带 `storage_runtime`、`segment_name`、`target_chunks`、`transport_endpoint`。
5. **写数据**（`runtime_write.rs:127-349`）。这是数据面核心：
   - 逐 target 判定本地/远端（`:149-156`）：`target.storage_runtime == self.lease.runtime && memory.has_storage_segment(...)`。
   - **本地**：`storage_address(segment, reservation.offset_bytes)` 拿到虚拟地址后 `ptr::copy_nonoverlapping`（`:157-169`）。注意这里用的是 `reservation.offset_bytes`（本地坐标）。
   - **远端**：先 `ensure_remote_runtime_reachable`，再 `open_segment_with_info`，然后 `storage_target_offset(target_chunks, segment_name, reservation.offset_bytes, len)` 把**分配器坐标翻译成传输坐标**（`:186-204`）。若 chunk 映射不可用，退回 `segment_relative_target_offset`（按 segment 内 buffer 顺序累加）。
   - 远端请求按 `max_registration_bytes` 切分（`target_buffer_transfer_requests`），按 `remote_write_chunk_limit` 分 chunk 提交；每 chunk 一次 `allocate_batch → submit_with_hints → wait_for_batch_completion_detailed → free_batch`（`:260-310`）。
   - 失败重试分两层：`write_reserved_replicas` 内部只在"远端 segment 缓存过期"且尚未刷过缓存时做**一次**整轮重试（`:336-345`），避免把真实网络故障当成缓存问题无限重试；外层 `put_object_with_policy_current` 的 `loop` 会在 `should_retry_put_after_write_failure`（`:2141-2151`：无 `required_preferred_segments` 且错误属 `Transport`/`NotFound`/`InvalidState`）且 `attempt < max_write_attempts` 时重新走"配额→分配→写"，每次重试前都会释放上一轮的分配并 abort 配额。
6. **构造路由**（`runtime_io.rs:534-570`）：`version = old.version.next()` 或 1；`replicas[priority] = { owner, segment_name, offset: 写侧返回的 absolute_offset, segment_offset: 本地分配偏移, length, checksum, tier: Dram }`。**两个 offset 在这里被分别赋值**，这是全章最该盯住的三行。注意 `offset` 的来源并不对称：本地 target 记的是 `storage_address()` 返回的本地虚拟地址，远端 target 记的是经 chunk 映射后的 transport target_offset；两者在本地 segment 上通常数值一致，但语义由 `replica_storage_target_offset` 统一收口（见流程 C）。
7. **路由 CAS 发布**（`:571-636`）：`publish_route` 走 `RouteOperations` → `RouteDirectory.compare_and_swap_object_route`。失败且 `expected_version` 为 None、同时 `cas.current` 也为 None 时，会用 `version_floor.next()` 或重新探测版本（`next_route_version`）再试一次（`:596-618`）。仍冲突则释放已写的分配、abort 配额、把 `current` 更新为 CAS 返回的 current 后 **continue 重试**，但受 `attempt < max_write_attempts` 上限约束，超过后返回 `Conflict`（`:619-635`）。
8. **发布后的收尾**（`:637-658`）：先 best-effort 通知远端 storage owner（`BatchTrackReplicaRoutes`），再 finalize 配额，再让本地 `StorageOwnerState` 记录热数据，最后对旧路由 `reclaim_route`。顺序刻意如此：**先让新路由权威可见，再回收旧字节**；旧路由回收失败只 warn，不影响 put 成功。

批量写的差异在 `batch_put_scoped_routed`（`runtime_write.rs:351` 起）：按 scoped key 排序后统一做配额准入（`reservation_order.sort_by` `:736`），任一条失败就 abort 已获得的预留（架构文档 `:417`）；远端传输按 tenant 分组，再按 scratch 窗口/公平性与 shaping 上限切 chunk，减少 completion 等待次数（`runtime_write.rs:1132-1240`）。另外 `batch_put_from_statuses` 在整批失败时会二分拆批重试，把错误细化到每个 key（`runtime_write.rs:393-433`）。

### 流程 C：Store-RS 的 get —— 路由解析 + 本地/远端分派

```text
facade.rs get/get_into/batch_get_into (:1562/:1651)
└─ resolve_objects (runtime_io.rs:2418)
   ├─ route_ops().load_routes(keys)             :2429-2445
   ├─ local_storage_segments / readable_runtime :2446-2447
   └─ 逐 key: select_readable_replica           :2466-2492
              └─ 无副本时 refresh membership 再选 :2473-2491
                 └─ 仍无且启用冷层 → cold backing read
└─ execute_batch_get_into (runtime_io.rs:2676)
   ├─ Phase 1 一把锁：分类 local/remote + 命中缓存算 copy 指令 :2736-2814
   ├─ Phase 2 缓存未命中：补 open_segment + target_chunks        :2824-2870
   ├─ 本地项 execute_copy_instructions / copy_local_replica_direct
   └─ 远端项 execute_remote_batch_get_chunk                     :4699
```

关键点：

- `select_readable_replica`（`runtime_io.rs:1925`）的排序规则**不是** `route_state_rank`/`replica_tier_rank`：那两个 rank 只在 `util.rs` 的 `canonical_route_key`（路由身份比较）里用到。真实规则是：先按 owner/segment 过滤掉不可读副本；若本地有可读副本，取 `priority` 最小的那个；否则对远端可读副本按 `priority` 排序后用 `replica_balance_hash(reader.stable_id, key) % n` 选一个——**这是刻意的读负载均衡，不是简单挑主副本**（`runtime_io.rs:1932-1956`，hash 函数 `:5576-5585`）。路由状态本身由调用方先过滤（`route.state != Active` 直接 `NotFound`）。**读热路径允许一次 membership 强制刷新**（`:2474`/`select_readable_replica_with_refresh` `:2034`），这是"读失败一次就重新看成员"的容错，不属于常规热路径的后端访问。
- `execute_batch_get_into` 明确分两阶段（注释 `:2734-2735` 与 `:2823`）：Phase 1 只持锁一次，把所有能算的 copy 指令算完；Phase 2 处理 cache-miss：先为每个缺失 segment 分别加锁 `open_segment_with_info`，最后再持一次锁统一算 copy 指令。这是为了避免"每 item 一把锁"的放大，而不是严格意义上的"Phase 2 只开一次锁"。
- 本地读也要做坐标翻译：`replica_storage_target_offset(info, chunks, replica)`（`:2800-2801`），而不是直接用 `replica.offset` 当虚拟地址。`copy_local_replica_direct`（`:5288`）同样走这条映射。**结论：本地读和远端读看到的是同一份 target 坐标语义**，这是架构文档 `:196-202` 强调的不变式。
- 读成功会报告 hit（`report_get_hits_best_effort` `:4072`；`report_grouped_route_hits_best_effort` `:4175`），本地直接进 `StorageOwnerState`，远端按 owner 聚合走 `BatchReportRouteHits`。hit 上报是 best-effort，不影响正确性。
- `is_exist`（`facade.rs:1232-1240`）走 `contains_active_route` → `RouteOperations::contains_active_route` → `RouteDirectory::contains_object_route`；`batch_is_exist`（`:1242-1257`）走 `contains_active_routes_bounded`。对 `EmbeddedWrh` 而言，单条与批量最终都落到 `contains_object_routes_bounded`（`mesh/directory.rs:1042-1046` 把单条委托给批量），因此**不做 fallback 探测也不上报 hit**（架构文档 `:259-260` 明确"existence checks stay boolean probes"）。一个容易看漏的细节：这两个 API 名字里的 "active" 在 `RouteDirectory` 这一层**并没有被强制执行**——`contains_object_route` 的默认实现就是 `get_object_route(...).is_some()`（`traits.rs:439-441`），`EmbeddedWrh` 的 bounded contains 也只是问 authority "有没有这条 route"（`directory.rs:262-330`），并不检查 `RouteState`。因此若某条路由处于 `Deleting`，`is_exist` 仍可能返回 true；需要严格判定可读性时应查 `query_route`/`get_size`。另一个细节是 bounded contains 会按 `observer_start_rank`（`directory.rs:1550-1559`，observer stable_id 的简单哈希）轮转起始 rank，避免所有 client 都先打同一个主 authority；并且在结果不全时会用 `baseline`（TTL 300s）里的候选集做一次兜底查询——这仍然只在 `route_topk` 范围内，不是全集群 fan-out。

### 流程 D：Embedded WRH —— 一次路由查询到底打了谁

这是 Store-RS 与 C++ master 模型分歧最大的地方。入口 `EmbeddedWrhRouteDirectory::get_object_routes`（`mesh/directory.rs:957-1009`）：

1. **取候选**（`:965`）：`authority_candidates(observer, force=false)` → 走 `ClientRouteMembership::live_clients`（`client/src/route_directory.rs:138-152`），即**本地快照**，不打 Redis。
2. **过滤**（`mesh/selection.rs:31-62`）：只保留 `state == Active`、`route_capable`（`labels["route"]=="true"`）、非 suspect、compatibility 匹配、`route_scope` 一致的 lease；同一 `stable_id` 只留最高 epoch。
3. **排序**（`selection.rs:64-89`）：对每个 key 计算 `weighted_rendezvous_score(namespace, key, stable_id, weight)`。实现是 `-ln(u)/w`，其中 `u` 由 FNV-1a 哈希 `namespace|key|stable_id` 映射到 (0,1)（`util.rs:99-123`）。分数**越小越优先**（`ranked.sort_by` 升序），weight 越大分数越小即越容易被选中。同分用 stable_id 字典序打破，保证跨进程确定性。
4. **按 rank 轮询**（`directory.rs:980-1005`）：
   - rank `0..route_topk`：主 + 镜像，带读修复观测；
   - rank >= topk：继续问更低排名的 authority，直到没有候选（fallback）。
   每一步内部先按 authority 分组，把同一 authority 的多个 key 合成一次 batch（`read_ranked_authorities` `:545-666`），减少 RPC 次数。
5. **合并**（`merge_fresher_route`，实现在 `mesh/read_repair.rs:70`）：同一 key 若多个 authority 返回不同版本，保留更新的。
6. **回填**（`backfill_missing_authorities` `:712-754`）：这不是"补偿读"而是**补偿写**——`compute_backfill_requests`（`read_repair.rs:142-170`）依据读修复观测为缺失/落后的镜像 authority 生成 `RouteCasRequest`，再经 `cas_authority_batch` 把较新的路由 CAS 写回镜像；失败只 warn 并标 suspect，不影响本次读结果。

CAS 路径 `compare_and_swap_route_selections`（`directory.rs:778-933`）同样是 rank 循环：

- 对每个 rank 分组；**本地 authority 直接调 `LocalRouteAuthority` 的 service**（走注册表，不过网络，`:814-873`），远端走 `authority_client.batch_compare_and_swap_routes`（gRPC，`:876-924`）。
- 一个请求在某个 rank 成功后立刻 `resolved[index] = Some(...)`，不再尝试更高 rank；失败才降级到下一个 rank。**这就是"primary CAS + mirror 兜底"的全部实现**。
- 任一 authority 报错都会 `maybe_mark_authority_suspect`，进入 `SuspectRuntimeCache` 隔离。隔离不是永久的：`reconcile_with_leases` 会在租约变大/状态变化/control 地址变化时解除（`state_core.rs:148-166`）。

`MetadataOnly` 模式则是另一条完全不同的路径：`shim/metadata.rs:13-17` 的 `build_metadata_route_directory` 直接包出一个只持有 `MetadataBackend` 的 `MetadataRouteDirectory`，所有读写都落到后端。它的 owner 列表上限用的是 `DEFAULT_ROUTE_OWNER_PAGE_SIZE`（`shim/contracts.rs:10`，值 256），超出时报 `Unsupported`（`metadata.rs:51-66`）。**它被定位为 bring-up/调试模式，而不是性能目标**（architecture.md `:19`）。

镜像发布是异步的：`AsyncRouteMirrorWorker` 在 `EmbeddedWrhRouteDirectory::new` 时 spawn（`directory.rs:135-140`），`Drop` 时 `shutdown` 并注销本地 authority（`:936-941`）。`RouteDirectory::flush_pending_route_mirrors`（`traits.rs:572-577`）是优雅退出时的补发钩子。

### 流程 E：Python 兼容层 —— `setup()` 如何落到同一个 StoreClient

`python/mooncake/store/__init__.py:95-117` 是分叉点：`MOONCAKE_STORE_BACKEND` 只接受 `cpp`/`rs`，分别 import `mooncake._store` 或 `mooncake.store.rs.store`（后者再 import `mooncake._store_rs`）。

rs 后端里，`PyMooncakeDistributedStore::setup(...)`（`crates/mooncake-store-py/src/python.rs:157` 起，函数体组装到 `:240`）：

```text
PyMooncakeDistributedStore::setup
├─ 组装 CompatSetupArgs（tenant/domain/object_set/labels/route_topk/timeouts...）
├─ CompatRuntimeArgs { setup, local_segment_name, initial_state, route_control }.build()
│    └─ runtime.rs:48
│        ├─ plan = setup.build()                     config.rs:218
│        │    └─ build_store_metadata_backend(url)   store_backend.rs:9
│        ├─ 选 transport factory                     runtime.rs:67-79
│        ├─ LocalMemoryConfig（storage/scratch/水位/hugepage）
│        ├─ StoreClientBuilder...build()             runtime.rs:104-135
│        └─ 可选 HTTP transport server publish
├─ StoreDispatcher::spawn_with_timeout_config_and_namespace_scope   dispatcher.rs:241
└─ finalize_real_dispatcher_setup                    runtime.rs:14-25
     ├─ register_local_memory()
     ├─ initial_state == Active → activate()
     └─ start_heartbeat_loop(lease_ttl_ms)
```

要点：

- `metadata_url` 是**必填**（`config.rs:255-259` 显式拒绝空串），且只认 `redis://` 或 `etcd://`；`http(s)://` 会明确报 "http metadata endpoints are not supported in store-rs"（`store_backend.rs:34-38`）。而 `transport_metadata_url` 是 TE 的输入，允许 `P2PHANDSHAKE`（仅 classic_te；`config.rs:401-407` 对 Tent 后端明确拒绝）。
- `StoreDispatcher`（`dispatcher.rs:170-189`，**不是** `config.rs`）持有 `Arc<StoreClient>`、`Arc<HealthChannel>`、`Arc<Mutex<RegionMap>>`、`Arc<tokio::Runtime>`、`hot_cache: Option<Arc<LocalHotCache>>`，以及 heartbeat 循环句柄与超时配置。所有 Python 调用都通过 `run` / `run_async`（`dispatcher.rs:509-540`）投递到 tokio runtime；PyO3 侧再配合 `allow_threads` 释放 GIL。
- 心跳循环（`dispatcher.rs:1901-1960`）默认 30s（`DEFAULT_HEARTBEAT_INTERVAL_MS` `:31`），失败退避为 `heartbeat_interval_ms.clamp(500, 1000)`（`:1897-1899`）；当**距上次成功的时间** `>= lease_ttl_ms` 时打 CRITICAL 日志（不是"连续失败次数超过 TTL"）。心跳本身是 `prepare_heartbeat(...).publish()`（`facade.rs:752-755`），成功后如果之前有失败，还会 `repair_local_metadata_after_heartbeat_recovery()` 补发本地元数据。
- `dummy` 路径（`setup_dummy`，`python.rs:243-256`）只做 `DummySession::connect(server_address, worker_scope)`，即连到独立的 `mooncake-store-rs-client` 进程；gRPC 承载 put/get，Unix domain side channel 传 shm fd（architecture.md `:512-522`）。`StoreDispatcher::hot_cache_fd()`（`dispatcher.rs:583`）就是 fd 传递的一端。
- **能力面差异**：`_CPP_EXPORTS` 47 个名字，`_RS_EXPORTS` 20 个名字（`__init__.py:9-93`，我逐个数的），交集只有 `MooncakeDistributedStore`/`MooncakeHostMemAllocator`/`BufferPool`/`RegisteredBufferPool`/`ReplicateConfig` 这 5 个；rs 独有的有 `TensorParallelism`、`ParallelAxis`、`ReadTarget`、`READ_MODE_*` 与 metrics 系列。切到 rs 后端后，访问 `EngramStore`、`Descriptor` 系列、`ObjectDataType` 枚举等会得到带专门文案的 `AttributeError`（`:133-137`）。

### 流程 F：薄绑定的一次 put —— 从 Rust/Go 到 C++ RealClient

Rust（`link` 后端）：

```text
MooncakeStore::put(key, value, config)          store.rs:239
├─ CString::new(key)?                            :245
├─ prepare_config(config)                        :619-636 → to_ffi :85-124
├─ ffi::mooncake_store_put(handle, key, ptr, len, cfg_ptr)  :252-258
│   （link：bindgen 静态符号；dlopen：ffi_dlopen 的 shim 转发到 MooncakeStoreLib） :251
└─ rc != 0 → StoreError::OperationFailed(rc)     :260-263
```

C++（`store_c.cpp:156-169`）：判空 → `to_replicate_config` → `std::span<const char>(value, size)` → `RealClient::put(key, span, *rc)`。**注意 `span` 不拥有数据**，依赖 `store_c.h:56-62` 的"返回后不再使用 char*"约定；这里传的是 `void*` 但语义相同。

Go（`store.go:175-199`）：

```text
Store.Put(key, value, config)
├─ toCConfig(config) → (C 结构体, []*C.char, error)   :180
├─ defer freeCStrings(cSegs)                           :184
├─ cKey := C.CString(key); defer C.free(...)           :186-187
├─ ptr = &value[0]（len>0 时）                          :189-192
└─ C.mooncake_store_put(handle, cKey, ptr, len, &cc)    :194
```

Go 侧的内存管理有三层，读的时候要分清：

1. `C.CString` 分配在 C 堆，必须 `C.free`（`defer`）；
2. `preferred_segments` 的每个字符串单独 `CString`，由 `freeCStrings` 统一释放（`:168-171`）；
3. **调用方的 Go slice 指针**通过 `unsafe.Pointer(&value[0])` 临时交给 C。Go 的 GC 在本调用期间不会移动堆对象（当前实现的保证），但**调用返回后 C 不得再持有该指针**——这正是 `store_c.h` 那条约定的另一半。

Go 的校验比 Rust 严（`store.go:126-166`）：`cfg == nil` 时显式填 `replica_num = 1`；`ReplicaNum <= 0` 直接 `ErrInvalidArgument`；`SoftPinAction` 越界拒绝；`SoftPinTTLMs != nil && action != Enable` 拒绝。Rust 的 `to_ffi` 只检查最后一条（`store.rs:95-99`），不做 `replica_num` 检查——这直接导致后文那个默认值坑。

## 与其他模块的边界

### Store-RS ↔ C++：只有数据面，且是运行时装配

这是本章最需要强调的边界事实：**`mooncake-store-rs/crates/` 下没有 `store_c.h`，也没有 `libmooncake_store`**。它与 C++ 的全部耦合是 CMake 产出的两个 shim：

- `mooncake-store-rs/CMakeLists.txt:37-52`：`libmooncake_classic_shim.so`，链接 `transfer_engine`；`OUTPUT_NAME mooncake_classic_shim`，`INSTALL_RPATH "$ORIGIN"`。
- `:54-69`：`libmooncake_tent_shim.so`，链接 `tent_shared`。
- `:122-125`：两者与 `tent_shared` 一起被装进 wheel 的 `mooncake/` 目录。

Rust 侧通过 `crates/mooncake-store-rs-transport-sys/src/lib.rs` 在**运行时** `dlopen`：

- `:13-28` 四个 `LibrarySpec`：`libtransfer_engine.so`（`MOONCAKE_CLASSIC_TE_LIB_PATH`）、`libtent_shared.so`（`MOONCAKE_TENT_SHARED_LIB_PATH`）、`libmooncake_classic_shim.so`、`libmooncake_tent_shim.so`。
- `:36-45` `load_library`：`dlopen(path, RTLD_NOW | RTLD_GLOBAL)`。`RTLD_GLOBAL` 在这里是必须的，但理由不是"shim 的符号要能被 dlsym 到"（对已拿到的 handle 做 `dlsym` 与 `RTLD_GLOBAL` 无关），而是**让先加载的库进入全局符号作用域**：`libmooncake_classic_shim.so` 的未定义符号要解析到早先加载的 `libtransfer_engine.so`，`libmooncake_tent_shim.so` 同理要解析到 `libtent_shared.so`。
- `:47-59` `load_symbol`：`dlsym` + `transmute_copy` 成函数指针。
- `:72-80` `load_library_from_spec`：逐个候选路径尝试 `load_library`，全失败后回落到裸 `file_name` 交给系统 loader；`:82-93` `resolve_library_candidates` 负责生成候选：先看环境变量（文件或目录），再按 `library_search_dirs()`（`:95-143`，模块自身路径、可执行文件路径、`MOONCAKE_BUILD_DIR` 下的 `mooncake-common`/`mooncake-asio`/`mooncake-store-rs/native-shims`/TE 与 TENT 的 src）。
- `:241-314` `ClassicApi`：25 个函数指针，其中 **16 个来自 `libtransfer_engine.so`**（`createTransferEngine`、`getLocalIpAndPort`、`installTransport`、`destroyTransferEngine`、`registerLocalMemory(Batch)`、`unregisterLocalMemory(Batch)`、`openSegment`/`openSegmentNoCache`、`closeSegment`、`allocateBatchID`、`submitTransfer`、`getTransferStatus`、`freeBatchID`、`syncSegmentCache`），**9 个来自 classic shim**（`mooncake_classic_republish_local_metadata`、`..._get_local_segment_descriptor_json(_size)`、`..._cache_segment_descriptor_json`、`..._get_batch_transfer_status`、`..._get_segment_first_buffer`、`..._get_segment_buffer_count`、`..._get_segment_buffer`、`..._get_max_mr_size`）。
- `:681-710` `TentApi`：21 个函数指针，其中 **19 个直接 `dlsym` 自 `libtent_shared.so` 的 `tent_*` 符号**，只有 2 个来自 tent shim（`:738-739`）。
- `:319-322` / `:715-718`：`OnceLock<Option<Api>>`，进程内只加载一次；加载失败返回 None，之后每次调用都是 `map_or(null)` 的空操作——**不会重试**。

`tent_shim.cc` 全文 49 行，只导出两个符号：

- `mooncake_tent_platform_free_memory`（`:8-14`）：转调 `Platform::getLoader().free`；
- `mooncake_tent_probe_rdma_max_mr_size`（`:16-49`）：遍历 IB 设备取 `max_mr_size` 的最小值。

**所以"按 shim 行数判断 FFI 规模"会严重低估**：真正的 ABI 面是 `libtent_shared.so` 的 19 个 `tent_*` 符号。

构建期校验只有"路径存在"：`mooncake-store-rs/crates/mooncake-store-rs-transport-sys/build.rs:27-37` 取出这些路径，`required_directory`/`required_file`（`:40-66`）再断言它们是绝对路径、必须是目录/已存在文件；可用 `MOONCAKE_SKIP_NATIVE_BUILD`（整体跳过）或 `MOONCAKE_SKIP_CLASSIC_TE`（跳过 classic shim 校验）绕过。**该文件没有任何 `cargo:rustc-link-lib`**。这与 `mooncake-store/rust/build.rs` 的 link 后端形成鲜明对照。

**但"Store-RS 完全不静态链接 C"是错的**（本次精读新发现）：`crates/mooncake-store-client/build.rs` 是 workspace 里唯一的链接点，且只挂在两个可选 feature 上：

- `kvcs-capi`（`:36-147`）：校验 `KVCS_SDK_ROOT` 下的 `COMMIT_ID` 版本必须是 `0.4.0`、架构与目标三元组一致，然后把 SDK 自带的 `rust/src/ffi.rs` 拷进 `OUT_DIR`，`rustc-link-lib=dylib=kvcs`（mock 模式下是 `kvcsmock`）。仅支持 Linux GNU + x86_64/aarch64（`:39-55`）。
- `nof-spdk`（`:160-200`）：用 `cc` 编译 `src/client/cold_tier/nof/extent_store/nof_spdk_shim.c`，`ar` 打成 `libmooncake_nof_spdk_shim.a`，再按 `pkg-config spdk_nvme spdk_env_dpdk spdk_syslibs` 的结果逐条发 link 指令。

这两个 feature 连的是外部设备/对象 SDK，**不是 `libmooncake_store`**。同一份 `build.rs` 还负责用 `tonic_build` + `protoc-bin-vendored` 生成 `control_plane.proto` 的 Rust 代码（`:4-13`），这是 CMake 完全不参与的一步。

安全封装层 `crates/mooncake-store-rs-transport/src/tent.rs`：

- `TentEngine::new`（`:159-172`）→ `create_engine_handle`（`:174-187`）：全局 create 锁 + 临时设置 Redis 环境变量（`TentRedisEnvGuard`）→ `config.apply()` → `ffi::tent_create_engine()`。
- `TentEngine` 内部：`raw: RwLock<handle>`（支持 `rebuild_engine_for_metadata_recovery` 原子替换，`:240-249`）、`registered_regions`、`owned_allocations`、`generation: AtomicU64`。`max_registration_bytes` 只在 RDMA 开启时才有值（`:164-167`）。

### Store-RS ↔ 元数据后端

`crates/mooncake-store-rs-metadata/src/store_backend.rs:9-45` 是唯一分派点：`redis://` → `RedisMetadataBackend`；`etcd://`（逗号分隔多 endpoint）→ `EtcdMetadataBackend`；`http(s)://` → 明确不支持；其他 → `unsupported metadata url scheme`。in-memory 后端不在这里，只给单测/`mooncake-store-test-utils` 用。

### Python 包 ↔ 两个原生扩展

- C++ 后端：`mooncake._store`（pybind，`mooncake-integration/store/store_py.cpp`）。
- Store-RS 后端：`mooncake._store_rs`，由 `crates/mooncake-store-py/Cargo.toml` 的 `[lib] name = "_store_rs"`、`crate-type = ["cdylib", "rlib"]` 产出；`python.rs:1542-1549` 的 `#[pymodule] fn _store_rs` 注册 6 个类。
- CMake 侧的安装名做了 SOABI 适配：`_store_rs.${Python3_SOABI}${CMAKE_SHARED_MODULE_SUFFIX}`（`mooncake-store-rs/CMakeLists.txt:34-35, 126-130`）。
- 三个 CLI 的 Python 入口在 `python/mooncake/_launcher.py:41-55`，只是 `exec` 对应二进制。

### 构建边界：CMake 管原生产物，Cargo 管 Rust 产物

`mooncake-store-rs/CMakeLists.txt` 的分工非常清晰：

- CMake 编：两个 shim、`transfer_engine`、`tent_shared`、Python 解释器探测（`:19-22` 要求 CPython SOABI）。
- Cargo 编：三个 bin（`build_store_rs_cli` `:81-97`）与 `_store_rs` 扩展（`build_store_rs_python_extension` `:99-116`）。CLI 目标传 **6 个环境变量**：`CARGO_TARGET_DIR`、`MOONCAKE_ROOT_DIR`、`MOONCAKE_BUILD_DIR`、`MOONCAKE_STORE_RS_DIR`、`MOONCAKE_CLASSIC_SHIM_LIB_PATH`、`MOONCAKE_TENT_SHIM_LIB_PATH`（`:83-89`）；Python 目标再多一个 `PYO3_PYTHON`（`:103`），共 7 个。粗读地图说 4 个，是低估了。
- 测试目标单独一套 `CARGO_TARGET_DIR`（`:137-150`），避免和 wheel 构建互相踩 target 目录。

### CI / 工具链边界

- Store-RS **不在任何 GitHub Actions workflow 里**：`grep -rl mooncake-store-rs .github/` 只命中 `.github/CODEOWNERS`。已发布 wheel 构建都不传 `WITH_STORE_RS=ON`（默认 OFF 生效），但 **`WITH_STORE_RUST=OFF` 并非"一律"**：`.github/workflows/_build-wheel.yaml` 只在三个 arm64 变体里显式传 `-DWITH_STORE_RUST=OFF`，三个 x86_64 变体没传，沿用该 option 的默认值 **ON**，所以 x86_64 wheel 实际会构建 Rust 绑定。
- 两个绑定则在 nightly 里有任务：`nightly.yml:299`（`run_store_rust_smoke.sh`）与 `:466`（同一脚本的 ASan/sanitizer 变体，`MOONCAKE_STORE_RUST_LINK_ASAN=1`）；Go 侧是 `:335`（普通 integration）与 `:477`（`MOONCAKE_STORE_GO_SANITIZED=1` 的 ASan 构建）。
- `.pre-commit-config.yaml:16` 显式 `skip: [store-rust-dlopen-bindings, store-rs-rustfmt, store-rs-clippy]`：这三个 hook 需要本地 CMake 构建树或 Cargo，pre-commit.ci 上不跑。hook 定义在 `:42-62`，其中 `store-rust-dlopen-bindings` 会在 `store_c.h` 变化时重生成绑定，CI 用 `git diff --exit-code` 卡过期绑定。

## 设计取舍与易踩坑点

### 1. 两个 Rust Store，名字只差两个字母

`mooncake-store/rust`（crate `mooncake_store`）与 `mooncake-store-rs`（workspace）完全无关。判别方法：

- 看有没有 `Cargo.toml` 的 `[workspace]`；
- 看代码里有没有 `store_c.h`/`RealClient`（有 = 绑定，无 = Store-RS）；
- 看 CMake 开关：`WITH_STORE_RUST`（默认 ON）vs `WITH_STORE_RS`（默认 OFF）。

讲反了会导致整章结论错误，例如"Store-RS 需要 master"（错）或"绑定支持多租户配额"（错）。

### 2. `route_topk` 不是 `replica_count`

`route_topk` 是**路由元数据**的 authority 副本数（主 + 镜像），决定"多少个 client 保存这个 key 的路由"；`replica_count` 是**数据副本**数，决定"多少个 storage owner 保存字节"。两者独立设置，`config.rs` 文档也专门提醒（`docs/source/deployment/store-rs/configuration.md:333`）。`route_topk < 2` 会被 builder 直接拒绝（`builder.rs:399-403`）。

### 3. `ReplicaConfig::default()` 的 `replica_num = 0` 是个真坑

- Rust 侧 `ReplicateConfig` 是 `#[derive(Default)]`（`store.rs:64`），`replica_num` 默认 **0**。
- 文档注释说 "0 means use the server default"（`store.rs:66`），`docs/source/api-reference/rust/mooncake-store.md:184` 也这么写。
- 但实现路径不支持：`store_c.cpp:54` 原样透传 `replica_num`；`master_service.cpp:5446-5451` 在 `replica_num == 0 && nof_replica_num == 0 && dfs_replica_num == 0` 时返回 `INVALID_PARAMS`。C++ 结构体的默认值 `replica_num{1}`（`mooncake-store/include/replica.h:109`）**只在 `config == nullptr` 时生效**。
- 结论：显式传 `Some(&ReplicateConfig::default())` 会被 master 拒绝；传 `None` 反而正常。
- Go 侧没这个问题：`toCConfig(nil)` 填 `replica_num = 1`，`ReplicaNum <= 0` 直接拒绝（`store.go:130-137`）。

**这是两套绑定语义不一致的真实缺陷，不是文档措辞问题**；修的时候要么改 Rust 的 `Default` 为 1，要么在 `to_ffi` 里补校验。

### 4. Rust 绑定漏了 `init_all`

`store_c.h:79-81` 有 `mooncake_store_init_all`，生成的 `ffi_dlopen_bindings.rs` 里也有（`:58`、`:253-260`），但 `store.rs` 没有任何安全包装，`ffi_dlopen.rs:137-214` 的 `shims!` 列表也**故意不含它**。Go 侧则有 `Store.InitAll`（`store.go:82-99`）。所以"19 个 C 函数 = 19 个 Rust 方法"的印象是错的，实际是 18 个。

### 5. `StoreTransport` 的定义地和"看起来的实现地"不是同一个 crate

`crates/mooncake-store-transport-core/src/lib.rs:19-107` 只有 107 行，定义了 `StoreTransport` 与 `StoreTransportFactory`，以及 `MemoryRegistration`。`crates/mooncake-store-client/src/transport.rs:104` 只是 `pub use mooncake_store_transport_core::{StoreTransport, StoreTransportFactory};`，具体实现 `TentStoreTransport`/`ClassicTeTransport`/`HttpStoreTransport` 在 client crate。按路径找 trait 定义会找错方向。

### 6. `offset` 与 `segment_offset` 不能混用

见"核心数据结构"。判断规则：

- 要交给 transport（submit/open segment/本地 copy）→ 用 `replica.offset`（再经 target chunk map 翻译）；
- 要交给 allocator（reserve/release/reclaim）→ 用 `replica.segment_offset` 或 `SegmentReservation.offset_bytes`。

`runtime_write.rs:186-204` 与 `runtime_io.rs:2799-2809` 是两处标准写法。架构文档 `:194-202` 明确写了"Scratch buffers ... are never published as object-addressable storage"。

### 7. 成员快照是"进程级 + 按 namespace 共享"的可变全局

`shared_live_client_cache`/`shared_suspect_runtime_cache`（`state_core.rs:184-206`）用 `OnceLock<Mutex<BTreeMap<...>>>` 做进程内缓存。影响：

- 同进程多个 `StoreClient` 共享同一份快照，一个 client 的 `force_refresh` 会刷新所有人的视图；
- 测试里如果并行跑多个不同 keyspace，必须用不同 namespace，否则互相污染；
- 缓存永不释放（`OnceLock` 静态表），长时间运行的多租户进程会累积条目。这不是泄漏（有界于 keyspace 数），但要知道。

### 8. `warnings = "deny"` 塑造了代码风格

`Cargo.toml:25-26` + pre-commit 的 `cargo clippy -- -D warnings`（`.pre-commit-config.yaml:56-62`）意味着：

- 所有"暂时没用"的字段/常量都必须 `#[allow(dead_code)]` 并附注释（例如 `mod.rs:76-89` 的 7 个 `DEFAULT_*`、`mod.rs:183-192` 的 Drop 相关字段）；
- 任何 rustc/clippy warning 都会让构建失败（clippy 侧是 `-D warnings`），没有"留几个无害 warning"的空间；
- 读代码时看到大量 `#[allow]`，应理解为"编译期强制的显式声明"，而非技术债。

### 9. `tent_shim.cc` 的行数会骗人

49 行里只有 2 个 `extern "C"` 符号，但 Rust 侧 `TentApi` 有 21 个函数指针。**评估 FFI 面要看 `load_api` 里的 `load_symbol` 数量，不是 shim 文件大小**。

### 10. Store-RS 是 Linux 取向，且默认关

- `mooncake-store-rs/crates/mooncake-store-rs-runtime/src/shm.rs` 直接用 `std::os::fd::OwnedFd`、Unix domain socket，并有 `#[cfg(target_os = "linux")]` 分支；
- `WITH_STORE_RS` 默认 OFF，wheel 默认不构建，CI 不跑；
- `mooncake-store/rust` 的 dlopen 后端虽然为 macos/windows 准备了 `DEFAULT_LIBRARY` 常量（`ffi_dlopen.rs:48-55`），但产出 `libmooncake_store.so` 的 `WITH_STORE_C_SHARED` 在 CMake 里硬性限制为 Linux/ELF（`mooncake-store/src/CMakeLists.txt:784-790` 直接 `FATAL_ERROR`，README 只是提了该开关，并没有"Linux-only"这句话）。

**改代码前先确认目标平台**，否则会在非 Linux 上撞到 fd/shm 相关编译或运行错误。

### 11. 生成物是提交进仓库的

`mooncake-store/rust/src/generated/ffi_dlopen_bindings.rs`（397 行）不是构建产物，而是受版本控制的 ABI 定义，由 `examples/generate_dlopen_bindings.rs` 生成、pre-commit hook 维护。改 `store_c.h` 后不重新生成，dlopen 后端会静默使用旧 ABI（bindgen 的 `dynamic_link_require_all` 只保证符号存在，不保证签名/布局匹配）。

### 12. 绑定层的错误码是"一锅端"

`store_c.h` 的所有函数只用 -1 表示失败（`get_into`/`get_size`/`remove_by_regex` 用负值），没有区分 NotFound/Timeout/Conflict。因此 Rust 的 `StoreError::NotFound` 变体实际上**从内部路径永远不会产生**（`error.rs:44-51` 的注释自己承认了），Go 侧也只能映射成 `ErrGet`/`ErrExist` 等粗粒度哨兵。想要细粒度错误，只能用 Store-RS。

### 13. Store-RS 的 batch 语义与绑定不同

- 绑定的 `batch_put_from`/`batch_get_into` 返回"每 key 结果码数组"，整体 `rc != 0` 才表示调用级失败（`store_c.cpp:185-215`、`:232-258`）。
- Store-RS 的 `batch_put_from` 返回 `Result<Vec<ObjectRoute>>`，`aggregate_batch_put_statuses`（`facade.rs:642-651`）遇到第一个 `Err` 就整体返回该错误，**已成功的条目不会回滚**（它们已经写完并发布了路由）；所以"整体 Err"只描述返回语义，不代表没有副作用（`facade.rs:1515-1518`）。部分失败语义由内部重试/回滚尽量收敛，但不暴露给调用方。
- Store-RS 的 `get_into_ranges`（`facade.rs:141-150`）返回 `Vec<Vec<Vec<i64>>>` 三层嵌套，且是 `unsafe`，它没有 C ABI 对应物——这是 Store-RS 独有的能力。

### 14. 未完全确认的点

- `mooncake-store-rs` 的 `cold_tier`/NoF 代码量很大（`state_store.rs` 2024 行、`cold_tier_state.rs` 997 行、`extent_store_engine.rs` 6287 行），本次只读了入口与类型，**没有逐行验证其持久化格式与恢复语义**；相关结论以 `docs/source/deployment/store-rs/cold-tier.md`、`nof.md` 为准。
- `control_plane.proto` 有 30 个 rpc，本次只核对了 rpc 名单（`:5-35`）与 client 侧的分组，**没有逐个核对 server 端 handler 的实现完整性**；`UnsupportedColdTierControlService` 的存在说明未启用冷层时部分 rpc 返回不支持。
- `MooncakeCompatibilityFacade` 的方法数我按 `fn <name>` 去重数得 **54**，与粗读地图的"约 119"不符；若按"含 tenant/in_scope 变体"或"含 impl 内部辅助方法"统计口径不同，数字会变，但 trait 声明本身是 54。

## 精读清单

建议按下面顺序读，每步都给出"读完应该能回答什么"。

1. **`docs/source/design/store/store-rs/architecture.md:1-120`**
   目标：建立 Store-RS 的心智模型。读完能回答：热路径不变式是什么、`EmbeddedWrh` 与 `MetadataOnly` 的区别、组件表里每个组件的热路径角色。

2. **`mooncake-store-rs/Cargo.toml` + `mooncake-store-rs/CMakeLists.txt`**
   目标：搞清 14 个 crate 的依赖方向与 CMake/Cargo 分工。读完能回答：谁编 shim、谁编 bin、wheel 装哪些文件、为什么 `WITH_STORE_RS` 需要 `WITH_TE` + `USE_TENT`。

3. **`crates/mooncake-store-core/src/traits.rs:1-200`**
   目标：掌握 `MetadataBackend` 的租约契约与 `RouteDirectory` 的 observer 语义。读完能回答：为什么 epoch 必须单调、`StaleEpoch` 在什么情况下允许同 epoch 回收、`get_object_routes_bounded` 为什么存在。

4. **`crates/mooncake-store-core/src/route.rs:14-220` + `identity.rs:14-140`**
   目标：把 `ObjectKey`/`ObjectRoute`/`ReplicaRoute`/`ClientLease`/`ClientRuntimeId` 的字段含义与序列化形态刻进脑子。读完能回答：`offset` 与 `segment_offset` 的区别、`route_topk` 与 `replica_count` 的区别。

5. **`crates/mooncake-store-client/src/client/mod.rs:57-200`**
   目标：用字段列表建立子系统地图。读完能回答：一个 `StoreClient` 持有哪些后台句柄、默认超时/心跳/重试参数各是多少。

6. **`crates/mooncake-store-client/src/client/builder.rs:393-843`**
   目标：完整走一遍启动流程。读完能回答：启动校验有哪些、租约如何分配、路由策略如何引导、本地 authority 如何绑定、后台任务何时起。

7. **`crates/mooncake-store-route/src/mesh/selection.rs` + `util.rs:1-125`**
   目标：看懂 WRH。读完能回答：候选如何过滤、分数怎么算、为什么是 `-ln(u)/w`、同分如何打破。

8. **`crates/mooncake-store-route/src/mesh/directory.rs:545-1009`**
   目标：读懂读路径的 rank 轮询与 CAS 的 primary/mirror 降级。读完能回答：一次 miss 最多打几个 authority、读修复在什么时候触发、suspect 如何隔离与恢复。

9. **`crates/mooncake-store-client/src/client/runtime_io.rs:413-736` + `runtime_write.rs:127-349`**
   目标：吃透 put 的"quota → reserve → write → CAS → finalize → reclaim"六段式。读完能回答：本地写与远端写分别怎么算地址、失败时按什么顺序回滚、重试条件是什么。

10. **`crates/mooncake-store-client/src/client/runtime_io.rs:2345-2566` + `2676-2900` + `5288-5324`**
    目标：吃透 get 的两阶段锁与本地/远端统一坐标。读完能回答：为什么 Phase 1 只持一次锁、本地读为什么也要做 target 映射、cold backing 在什么条件下接管。

11. **`crates/mooncake-store-rs-transport-sys/src/lib.rs:1-380, 606-748` + `tent_shim.cc`**
    目标：确认 Store-RS 与 C++ 的唯一接触面。读完能回答：加载了哪 4 个 `.so`、`RTLD_GLOBAL` 为什么必要、classic/tent 各有多少函数指针、shim 实际只导出几个符号。

12. **`crates/mooncake-store-rs-runtime/src/runtime.rs:14-157` + `dispatcher.rs:170-320, 1901-1960`**
    目标：理解 Python real 路径。读完能回答：metadata URL 如何选后端、transport factory 三选一、心跳循环的退避与告警条件、`StoreDispatcher` 的 tokio runtime 扮演什么角色。

13. **`crates/mooncake-store-py/src/python.rs`（`setup` 与 `_store_rs` 两处）+ `python/mooncake/store/__init__.py`**
    目标：看清 backend 选择与能力面差异。读完能回答：rs 后端缺哪些 C++ API、`MOONCAKE_STORE_BACKEND` 非法值会怎样、dummy 路径连的是谁。

14. **`mooncake-store/include/store_c.h` 全文 + `mooncake-store/src/store_c.cpp:29-98, 156-215`**
    目标：掌握 ABI 契约。读完能回答：19 个函数分别是什么、`char*` 生命周期约定、`to_replicate_config` 的校验点、`StoreHandle` 为什么持有 shared_ptr。

15. **`mooncake-store/rust/src/store.rs` 全文**
    目标：把安全封装的样板刻下来。读完能回答：`to_ffi` 的三个返回值为什么必须同时存活、`prepare_config` 的悬垂指针防护、`Drop` 做了什么、哪些 `unsafe` 是调用方责任。

16. **`mooncake-store/rust/src/ffi_dlopen.rs` + `build.rs` + `Cargo.toml`**
    目标：理解双后端。读完能回答：`link` 与 `dlopen` 的 feature 优先级、`ensure_loaded` 的并发保护、`link` 后端链接了哪些库、为什么 ASan 默认不链。

17. **`mooncake-store/go/mooncakestore/store.go` 全文 + `build.sh:30-91`**
    目标：理解 CGo 生命周期与链接图。读完能回答：`toCConfig` 比 Rust 严在哪、`defer freeCStrings` 覆盖哪些分配、`CGO_LDFLAGS` 为什么要 `--start-group`。

18. **`scripts/ci/run_store_rust_smoke.sh` / `run_store_go_integration.sh` / `mooncake-store-rs/scripts/e2e/*.sh`**
    目标：知道怎么验证。读完能回答：哪些场景需要真实 master/metadata、哪些跑在 CI、Store-RS 的 e2e 如何覆盖 route migration 与 hot cache。

按这 18 步读完，应能独立回答三类问题：**（a）** 给定一个 put/get 调用，说出它经过哪些线程/进程/RPC 与哪些锁；**（b）** 给定一个"路由指向已死 client"的现象，定位到 suspect cache、membership refresh、read fallback 中的哪一环；**（c）** 给定一处 ABI 或语义改动，判断需要同步修改 Store-RS、Rust 绑定、Go 绑定、生成绑定文件与文档中的哪几处。
