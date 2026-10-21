# 10 内存分配器体系

> 精读范围：`mooncake-store` 段分配器（CacheLib / OffsetAllocator）、`offset_allocator` 核心算法、客户端 mmap/arena/pinned 缓冲、设备与 GPU fabric 分配器、分配器指标。
> 所有 `路径:行号` 均相对仓库根 `D:/xiaojun/work/Mooncake`；行号以 2026-10 的 main 分支（`67df8c41` 附近）为准。

## 定位与职责

Mooncake Store 的分配器体系回答一个看起来简单、实际很容易读错的问题：**"一个 KV 对象的地址空间从哪来？"**

答案分三层，而这三层的"内存"含义完全不同：

1. **Master 侧的段分配器（segment allocator）**。客户端把一大块自己拥有的内存 `MountSegment` 进集群后，Master 需要在这块内存的地址区间 `[base, base+size)` 里为每个副本切出一个 `AllocatedBuffer`。请注意：**Master 从不拥有、也从不访问这块内存**，它只是在一段 `uint64` 地址区间上做区间分配，把 `(offset, size, protocol, endpoint)` 作为元数据下发给写者。真正的字节永远由客户端进程持有并通过 Transfer Engine 直接搬运（`docs/source/design/store/mooncake-store.md` 的 "Master Service does not take over any data flow"）。
   这里有两个可替换实现（`mooncake-store/include/types.h:521` 的 `BufferAllocatorType`）：
   - `CachelibBufferAllocator`：vendored 的 Facebook CacheLib slab 分配器子集，16MB slab + size-class；
   - `OffsetBufferAllocator`：移植自 sebbbi/OffsetAllocator 的 bin 分配器，**默认且推荐**（`mooncake-store/include/master_config.h:275`、`:1255`、`:885` 三处默认值都是 `OFFSET`）。

2. **客户端侧的本地缓冲分配器**。客户端要一块大缓冲来承接 put/get 的数据，这块缓冲是**真实内存**：`allocate_buffer_allocator_memory`（`mooncake-store/src/common/client_buffer_allocation.cpp:97`）按 `protocol` 分发到 CUDA/Ascend/Sunrise/UB/SPDK/aligned_alloc，hugepage 路径还可以走 `MmapArena`。拿到真实内存后，再用**同一个** `OffsetAllocator` 在其上做子分配（`ClientBufferAllocator`，`mooncake-store/src/client_buffer.cpp:64`）。另有 `RegisteredPinnedMemoryManager` 负责带预算的 `cudaHostRegister`。

3. **设备 / fabric / 指标层**。`mooncake-store/include/device/` 提供跨厂商 `AcceleratorDevice` 抽象；`mooncake-integration/allocator.py` 与 `mooncake-transfer-engine/nvlink-allocator/nvlink_allocator.cpp` 一起把 GPU fabric 内存（`cuMemCreate` + `cuMemMap` 的 VMM 路径）注册成 torch 的 pluggable allocator；`allocator_metric.cpp` 导出的是**进程 RSS 与 jemalloc 统计**，与上面两层的 BufferAllocator 没有关系。

一张图概括（箭头表示"谁持有谁"）：

```
                    ┌──────────────────────── Master 进程 ────────────────────────┐
                    │                                                            │
 客户端 mount ─────►│  SegmentManager::mounted_segments_[id].buf_allocator        │
 (base,size,proto)  │     ├── OffsetBufferAllocator  ──► offset_allocator::OffsetAllocator
                    │     └── CachelibBufferAllocator ──► facebook::cachelib::MemoryAllocator
                    │              （两者都只切 [base, base+size) 的地址，不碰内存）│
                    │         ▲ AllocatorManager + AllocationStrategy 选段         │
                    │         └── AllocatedBuffer（weak_ptr 回指 allocator，RAII 归还）│
                    └────────────────────────────────────────────────────────────┘
                                          │ Descriptor{size,addr,protocol,endpoint}
                                          ▼
                    ┌──────────────── 客户端进程（真实内存在这里）───────────────┐
                    │ allocate_buffer_allocator_memory / allocate_buffer_mmap_memory│
                    │      └── ClientBufferAllocator ──► OffsetAllocator（子分配）  │
                    │ RegisteredPinnedMemoryManager（cudaHostRegister 预算管理）    │
                    │ device::AcceleratorDevice / nvlink_allocator.so（GPU fabric） │
                    └────────────────────────────────────────────────────────────┘
```

本子系统最容易产生的三个误解，先在这里点名，正文会逐一展开：

- `OffsetAllocator` **从不碰真实内存**，它管理的只是地址/偏移；同一个类被复用在四个语义完全不同的地方（Master 段地址、客户端缓冲子分配、本地 SSD 文件内偏移、DFS 页偏移）。读代码前必须先确认 `base` 指向什么。
- `allocator_metric.cpp` / `allocator_status.h` 与 BufferAllocator 无关，导出的是进程 RSS 和 jemalloc 内部统计。
- 段容量超过 3.75GB 时，`OffsetAllocator` 会引入 `m_multiplier_bits`，所有 offset/size 以 quantum 为单位存储；把 `OffsetAllocation::getOffset()` 当字节偏移是最常见的误读。

## 目录结构速览

| 文件 | 一句话职责 |
|---|---|
| `mooncake-store/include/allocator.h` | 段分配器契约总入口：`BufferAllocatorBase`、`AllocatedBuffer`、两个实现、`DummyBufferAllocator`、`SimpleAllocator`、HA 导入 API |
| `mooncake-store/src/allocator.cpp` | 两实现的构造/分配/释放、CACHELIB 布局校验、每段 `MasterMetricManager` 记账、`ImportOffsetBufferAllocator` 的重放式恢复 |
| `mooncake-store/include/offset_allocator/offset_allocator.h` | `OffsetAllocator`（线程安全包装 + multiplier + RAII handle）、`__Allocator`、`OffsetAllocatorSnapshot` 声明与序列化模板 |
| `mooncake-store/src/offset_allocator.cpp` | bin 分配核心：`SmallFloat` size-class、两级位图、free-list、Mooncake 的 round-up-to-bin 修改、快照全图校验 |
| `mooncake-store/include/cachelib_memory_allocator/MemoryAllocator.h` | vendored CacheLib 子集入口：`MemoryAllocator → MemoryPool → AllocationClass → SlabAllocator` 层级，`importAllocations` |
| `mooncake-store/include/cachelib_memory_allocator/Slab.h` | `Slab::kSize = 1<<24 = 16MB`、`kMinAllocSize = 64`、`SlabHeader`（7 字节 packed） |
| `mooncake-store/src/cachelib_memory_allocator/*.cpp` | 独立 STATIC 库 `cachelib_memory_allocator` 的实现（`MemoryAllocator.cpp` / `MemoryPool.cpp` / `MemoryPoolManager.cpp` / `AllocationClass.cpp` / `SlabAllocator.cpp` / `Slab.cpp`） |
| `mooncake-store/include/allocator_metric.h` / `src/allocator_metric.cpp` | `AllocatorMetric`：进程 RSS 系列 + jemalloc 系列的 Prometheus 指标；`JemallocStatsCollector` 函数指针注入 |
| `mooncake-store/include/allocator_status.h` | jemalloc 侧采集器：`LogAllocatorStatus` / `CollectJemallocStats` / `InstallAllocatorStatsCollector`（仅 `STORE_USE_JEMALLOC` 下有内容） |
| `mooncake-store/include/mmap_arena.h` / `src/mmap_arena.cpp` | `MmapArena`：hugepage mmap + lock-free CAS bump 分配，`owns()` 用于区分释放路径 |
| `mooncake-store/src/common/mmap_buffer.cpp` | arena 与直接 mmap 的接线层：`allocate_buffer_mmap_memory` / `free_buffer_mmap_memory` / `is_mmap_arena_allocation`，全局 `g_mmap_arena` |
| `mooncake-store/src/common/client_buffer_allocation.cpp` | `allocate_buffer_allocator_memory` / `free_memory`：按 protocol 分发的总闸口 |
| `mooncake-store/src/registered_pinned_memory.cpp`（头在同目录 `src/`） | `RegisteredPinnedMemoryManager`：预算 + 重叠检测 + `cudaErrorCudartUnloading` 时宁可泄漏 |
| `mooncake-store/include/client_buffer.h` / `src/client_buffer.cpp` | `ClientBufferAllocator`（真实内存 + OffsetAllocator 子分配）+ `BufferHandle`（owning/view 双模式） |
| `mooncake-store/include/pinned_host_buffer.h` | `PinnedHostBuffer`：函数指针 deleter 的 move-only pinned 缓冲 |
| `mooncake-store/include/device/accelerator_device.h` | `AcceleratorVendor` / `PointerInfo` / `AcceleratorDevice` 抽象 |
| `mooncake-store/include/device/{runtime_accelerator,accelerator_registry,cuda_ipc_buffer}.h` | 运行时选设备、静态注册表、CUDA IPC 导出/映射 |
| `mooncake-store/include/allocation_strategy.h` | `AllocatorManager`（segment→registration 注册表）与 `Random/Ranked/FreeRatioFirst/SsdFreeRatioFirst/Cxl` 策略 |
| `mooncake-store/include/segment_allocator_registration.h` + `src/*.cpp` | 一个 segment 的注册项：两条 `SegmentLifetime` + `ClientLivenessRecord`，决定"这个段还能不能分配" |
| `mooncake-store/include/storage_usage.h` | `StorageUsageTracker` / `StorageUsageRegistration`：按存储域聚合用量，供水位/策略使用 |
| `mooncake-store/src/memory_alloc.cpp` | 19 行；`hugepage_memory_alloc` 在非 `USE_NOF` 构建下直接返回 `nullptr` |
| `mooncake-store/src/aligned_client_buffer.cpp` | 4096 对齐版客户端缓冲，仅 `file_storage.cpp` 的 O_DIRECT 路径使用 |
| `mooncake-integration/allocator.py` | `NVLinkAllocator` / `BarexAllocator`：把 `.so` 的 C 符号包成 `torch.cuda.memory.CUDAPluggableAllocator` |
| `mooncake-integration/fabric_allocator_utils.py` | `get_mooncake_so_path` / `probe_allocator_backend`（ctypes 探针封装） |
| `mooncake-transfer-engine/nvlink-allocator/nvlink_allocator.cpp` | GPU fabric 分配的 `.so` 源码：`cuMemCreate/cuMemMap` + `extern "C" mc_allocator_*` |
| `mooncake-transfer-engine/fabric_allocator.cmake` | `add_fabric_allocator_build_target`：调外部 `build.sh` 在 CMake 树外构建 `.so` |
| `docs/source/performance/mooncake/allocator-benchmark-result.md` | 官方基准：明确 MmapArena 与 OffsetAllocator 是互补层次，并给出 round-up-to-bin 的前后对比 |

## 核心数据结构

### 3.1 `AllocatedBuffer`：一个"副本句柄"，不是一个指针

`mooncake-store/include/allocator.h:70-177`。它是 Master 侧分配结果的唯一产物，也是整个子系统所有权关系的浓缩。

```cpp
std::weak_ptr<BufferAllocatorBase> allocator_;   // allocator.h:158
SegmentLifetime segment_lifetime_;               // :164
std::shared_ptr<ClientLivenessRecord> client_liveness_;  // :165
std::string segment_name_;                       // :166
void* buffer_ptr_{nullptr};                      // :167  ← 客户端虚拟地址，不是本地可解引用指针
std::size_t size_{0};                            // :168  ← 请求大小
std::string protocol{"tcp"};                     // :169
std::optional<offset_allocator::OffsetAllocationHandle> offset_handle_;  // :171
```

几个设计要点：

- **`allocator_` 是 `weak_ptr`**（`:158`）。这是刻意的：`AllocatedBuffer` 不能反向延长 allocator 的生命周期，否则 segment 卸载后 allocator 永远析构不掉。代价是析构时可能 lock 失败——此时走 `AllocatedBuffer::~AllocatedBuffer` 的 `if (alloc) alloc->deallocate(this)`（`mooncake-store/src/allocator.cpp:66-75`），注释说明 segment 已经拆掉、用量不再需要维护。
- **`buffer_ptr_` 是远端客户端的虚拟地址**。Master 无法解引用它，它只是 `Descriptor` 里下发给写者的搬运目标。CXL 协议下更极端：`change_to_cxl`（`allocator.cpp:118-123`）把它减去 `DEFAULT_CXL_BASE` 变成**相对偏移**，`get_vaddr_from_cxl`（`:125-128`）再还原。
- **`offset_handle_` 是 OFFSET 路径的 RAII 归还器**（`:171`）。CACHELIB 路径不用它，靠 `deallocate` 显式 `free`。
- **`Descriptor`**（`:143-150`）是可序列化视图，四个字段 `size_ / buffer_address_ / protocol_ / transport_endpoint_`，`YLT_REFL` 生成反射用于 msgpack。`get_descriptor()` 在 `allocator.cpp:112-116`；`getTransportEndpoint()`（`allocator.cpp:96-109`）在 protocol 为 `cxl` 时返回 `segment_name_`（即 CXL binding name），而不是 allocator 的 endpoint——这是 CXL 段的特殊寻址方式。

两个"有效性"谓词不能混用：

- `isAllocatorValid()`（`allocator.h:100-102`）= allocator 未过期 **且** `segment_lifetime_` 可用；
- `isAvailable()`（`:108-114`）= 上面那条 **且** 客户端 liveness 记录仍为 serving。

### 3.2 `SegmentLifetime` 与"两套生命周期"

`SegmentLifetime`（`allocator.h:50-68`）只是一个 `shared_ptr<atomic<bool>>` 的包装：拷贝即共享，`setAvailable(false)` 一次性把同一 lifetime 的所有持有者标成不可用。之所以要共享语义，是因为它是 `SegmentAllocatorRegistration` 的成员，需要被"复制"进每个 `AllocatedBuffer`。

`SegmentAllocatorRegistration`（`mooncake-store/include/segment_allocator_registration.h`、实现 `src/segment_allocator_registration.cpp`）持有**两条**独立 lifetime：

```cpp
std::shared_ptr<BufferAllocatorBase> allocator_;
SegmentLifetime allocation_lifetime_;   // 还能不能继续分配
SegmentLifetime buffer_lifetime_;       // 已发出的 buffer 还算不算数
std::shared_ptr<ClientLivenessRecord> client_liveness_;
```

`Invalidate()`（`segment_allocator_registration.cpp:79-82`）同时关两条；`SetAllocatable()`（`:75-77`）只关分配但不影响已发出的 buffer。`Allocate()`（`:22-45`）的流程值得逐行看：先查 `allocation_lifetime_`，再原子读 `client_liveness_`，分配成功后 `bindSegmentLifetime(buffer_lifetime_)` + `bindClientLiveness(record)`，**最后再检查一次** lifetime/liveness 是否在分配过程中被并发翻转，若翻了就返回 `nullptr`（buffer 随即析构归还）。这是典型的"分配后二次校验"模式，用来消除 unregister 与 allocate 的竞态窗口。

`OwnsBuffer()`（`:70-73`）通过比较 `SegmentLifetime` 的 `operator==`（即比较底层 `shared_ptr` 指针）判断某个 buffer 是否属于本注册项——这是 HA 恢复替换 allocator 时区分新旧 buffer 的手段。替换本身走 `BindAllocator`（`:52-56`），对 `allocator_` 做 `std::atomic_store_explicit`；`GetAllocator`/`Allocate` 用对应的 `std::atomic_load_explicit` 读取，所以"换 allocator"与"并发 allocate"之间不会读到半更新的 `shared_ptr`。

### 3.3 `BufferAllocatorBase`：两套记账同时存在

`allocator.h:183-227`：

```cpp
virtual std::unique_ptr<AllocatedBuffer> allocate(size_t size) = 0;
virtual void deallocate(AllocatedBuffer* handle) = 0;
virtual size_t capacity() const = 0;     // 段总大小
virtual size_t size() const = 0;         // 已用量（requested 之和）
virtual uintptr_t base() const = 0;
virtual size_t getLargestFreeRegion() const = 0;
void AttachUsageTracker(const std::shared_ptr<StorageUsageTracker>&);  // :211
```

私有成员只有两个：`std::atomic_size_t cur_size_{0}`（`:225`）与 `std::unique_ptr<StorageUsageRegistration> usage_registration_`（`:226`）。**这里有两套互不相干的记账**，混淆是常见错误：

| 记账 | 位置 | 用途 | 触发点 |
|---|---|---|---|
| `cur_size_` + `StorageUsageRegistration` | `allocator.cpp:44-56` | 分配策略的水位判断（`FreeRatioFirstAllocationStrategy`）与 `GetMemoryUsageSnapshot` | `RecordAllocation/RecordDeallocation` |
| `MasterMetricManager::inc_allocated_mem_size` | `allocator.cpp:258-264` 等 | Prometheus 每段指标 | 每次 allocate/deallocate/restore |

`StorageUsageRegistration`（`include/storage_usage.h:71-102`）在构造时把 allocator 的**当前** `used_bytes` 和 `capacity()` 加进 tracker，析构时减回去；它持有一个 `std::atomic_size_t&` 引用（指向 allocator 的 `cur_size_`）。成员声明顺序保证了析构顺序正确：`usage_registration_` 在 `cur_size_` 之后声明，因此先析构、先 detach，不会读悬空引用。

`AttachUsageTracker`（`allocator.cpp:35-42`）是幂等的、只允许在发布前调用一次；`SegmentManager::AttachMountedUsageTrackers`（`mooncake-store/src/segment.cpp:142-158`）负责在恢复后统一补挂（用 `unordered_set<const BufferAllocatorBase*>` 去重，因为 CXL 全局 allocator 可能被多条 segment 记录共享）。新版 `SegmentPool`/`RegionDriver` 路径不是批量补挂，而是在挂载/恢复事务里逐条挂：`SegmentPool::WriteAccess::PrepareWithLiveAllocations`（`segment/pool_mount.cpp:91-92`，`PrepareMount`/`PrepareRestore` 都汇入这里）与 `PublishMount`（`:186`，覆盖快照恢复走的 `PrepareAdopt` + `Commit`）；两处都可能命中同一个 allocator，靠 `AttachUsageTracker` 自身的幂等性去重。

`size()` 返回的是**请求字节数之和**，不是实际占用的 bin/slab 字节。这一点对 OFFSET 尤其重要（见 §6）。

### 3.4 `CachelibBufferAllocator`

`allocator.h:278-339`，实现 `allocator.cpp:142-349`。

字段：`segment_name_ / base_ / total_size_ / transport_endpoint_ / replica_type_` 五个不可变元数据，加上三块状态：

```cpp
std::unique_ptr<char[]> header_region_start_;      // SlabHeader 数组，master 本地堆分配
size_t header_region_size_;                        // :328
std::unique_ptr<MemoryAllocator> memory_allocator_; // :329
PoolId pool_id_;                                   // :330
```

构造函数（`allocator.cpp:185-222`）的关键点：

```cpp
header_region_size_ = sizeof(SlabHeader) * (size / sizeof(Slab)) + 1;   // :199-202
header_region_start_ = std::make_unique<char[]>(header_region_size_);   // :203
memory_allocator_ = std::make_unique<MemoryAllocator>(
    Config(generateAllocSizes()), header_region_start_.get(), header_region_size_,
    reinterpret_cast<void*>(base), size);                               // :208-212
pool_id_ = memory_allocator_->addPool("main", size);                    // :219
```

**元数据与段内存是分离的**：`SlabHeader` 数组由 Master 自己 `new` 出来（`header_region_start_`），而 `slabMemoryStart` 传的是客户端上报的远端虚拟地址。CacheLib 的原设计里两者通常在同一片 mmap 里，这里被拆开了——因为 Master 根本拿不到客户端内存。

`IsValidCachelibLayout`（`allocator.cpp:17-24`）给出 CACHELIB 的六条硬约束：

```cpp
base != 0 && base % Slab::kSize == 0 &&
size >= Slab::kSize && size % Slab::kSize == 0 &&
slab_count <= UINT_MAX && base <= SIZE_MAX - size
```

注意 `slab_count = size / sizeof(facebook::cachelib::Slab)`——用的是 `sizeof(Slab)`，而 `Slab` 里有一个 `char data_[kSize]`（`Slab.h:101`），所以 `sizeof(Slab) == kSize == 16MB`。这是"用类型大小当常量"的写法，改 `Slab` 布局会静默改变语义，读代码时容易看漏。

`MountSegment` 在创建前还会再拦一次（`segment.cpp:245-252`），错误信息里明确写了 "as required by Cachelib"。

`allocate`（`allocator.cpp:235-266`）：`padding_size = max(size, kMinSliceSize)`（`kMinSliceSize = 64`，`types.h:450`），调 `memory_allocator_->allocate(pool_id_, padding_size)`；成功后 `RecordAllocation(size)` 与 `inc_allocated_mem_size(segment_name_, size)` **都按请求大小记**，不是 `padding_size`，也不是 size-class 的实际占用。失败与异常都返回 `nullptr`，不抛。

`deallocate`（`:268-292`）：CXL buffer 要先用 `get_vaddr_from_cxl()` 还原绝对地址再 `free`（`:270-272`），因为 `change_to_cxl` 已经把它改成了相对偏移。

析构（`:224-233`）按 `replica_type_` 减指标：`MEMORY` 减 `allocated_mem_size` 并 `remove_segment_metrics`，`NOF_SSD` 减 `allocated_nof_size`。

### 3.5 `OffsetBufferAllocator` 与 `OffsetBufferAllocatorSnapshot`

`allocator.h:377-436`，实现 `allocator.cpp:352-565`。字段极简：五个不可变元数据 + 一个 `shared_ptr<OffsetAllocator>`（`allocator.h:435`）。

构造函数（`allocator.cpp:352-394`）里两个容量参数是从段大小推出来的：

```cpp
uint64_t init_capacity = clamp(size / 4096, 1024, 64 * 1024);          // :367-370
uint64_t max_capacity  = clamp(size / 1024, 1024*1024, kMaxNodes);      // :372-378
```

含义是：初始按"平均 4KB 一个 node"预算，上限按"平均 1KB 一个 node"预算，且节点总数不超过 `OffsetAllocatorSnapshot::kMaxNodes = 1<<26`（`offset_allocator.h:334`，约 6710 万）。所以：

- 64GB 段：`max_capacity = 64M`，恰好等于 `1<<26`（6710 万）这个上限，不多不少；
- 1TB 段：理论 10 亿，被夹到 6710 万 → **平均分配粒度不能低于约 16KB**，否则节点会先用完。

`OffsetBufferAllocatorSnapshot`（`allocator.h:354-370`）是"脱离运行时的快照"：外层的 `segment_name/base/capacity/used_bytes/transport_endpoint` 加内层 `OffsetAllocatorSnapshot`。`ValidateMetadata()`（`allocator.cpp:406-428`）做三件事：外层与内层 base/capacity 必须一致、`used_bytes <= capacity` 且不超 `int64` 上限、`used_bytes == allocation_state.allocated_size`。`Validate()`（`:430-434`）再叠加内层的全图校验。

`Restore`（`:436-461`）的顺序值得记住：**先校验元数据 → 再让 `OffsetAllocator::Restore` 校验完整图 → 构造对象 → 最后才 `inc_allocated_mem_size` + `SetUsageBytesForRestore`**。注释（`:455-456`）强调"No allocation handles exist yet. Pair restored usage with the same accounting that allocate()/destruction use, before publishing the object"。注意这里只更新了 Prometheus 指标与 `cur_size_`，**没有** `AttachUsageTracker`——头文件注释（`allocator.h:388-391`）说得很清楚："capacity accounting remains the owning pool's responsibility"。

`allocate`（`:480-523`）把 `OffsetAllocationHandle` 直接移进 `AllocatedBuffer`，`buffer_ptr = allocation_handle->ptr()`（也就是 `m_base + (offset << multiplier_bits)`）。`deallocate`（`:525-546`）只做 `handle->offset_handle_.reset()`，真正的归还发生在 handle 析构里。

`getLargestFreeRegion`（`:548-565`）转调 `offset_allocator_->storageReport().largestFreeRegion`，**加锁**取一次权威值；而 `OffsetAllocator` 内部的 `m_largest_free_region` 是一个可能偏大的无锁提示值。两者的差异见流程 B（B1/B6）与 §6.5。

### 3.6 `OffsetAllocator` / `__Allocator`：bin 分配的全部状态

`offset_allocator.h:144-256` 是线程安全外壳，`offset_allocator.h:258-319` 是算法内核。

外壳的关键字段：

```cpp
std::unique_ptr<__Allocator> m_allocator GUARDED_BY(m_mutex);  // :227
uint64_t m_base;              // :228  真实基址（字节）
uint64_t m_multiplier_bits;   // :231  所有 offset/size 的 quantum = 1<<bits
uint64_t m_capacity;          // :232  字节容量
mutable Mutex m_mutex;        // :233  注意：整段一把锁
uint64_t m_allocated_size GUARDED_BY(m_mutex) = 0;   // :236 请求字节和
uint64_t m_allocated_num  GUARDED_BY(m_mutex) = 0;   // :237
std::atomic<uint64_t> m_largest_free_region{0};      // :238 无锁提示
bool m_largest_free_region_tightened GUARDED_BY(m_mutex) = false;  // :239
```

`GUARDED_BY` / `REQUIRES` 来自 `mooncake-store/include/mutex.h:19-52`，`__clang__` 之外是空宏，所以只是文档。

内核 `__Allocator`（`offset_allocator.h:258-319`）：

```cpp
struct Node {                       // :290-300
    uint32 dataOffset = 0;
    uint32 dataSize   = 0;
    NodeIndex binListPrev = unused;   // 同一个 bin 内的双向链表
    NodeIndex binListNext = unused;
    NodeIndex neighborPrev = unused;  // 物理相邻节点（用于合并）
    NodeIndex neighborNext = unused;
    bool used = false;
};
uint32 m_size;                 // 总大小，单位是 quantum（不是字节！）
uint32 m_current_capacity;     // 已启用的节点数
uint32 m_max_capacity;         // 节点数上限
uint32 m_freeStorage;          // 空闲总量，单位是 quantum
uint32 m_usedBinsTop;          // 32 个 top bin 的位图
uint8  m_usedBins[32];         // 每个 top bin 下 8 个 leaf 的位图
NodeIndex m_binIndices[256];   // 256 = 32 * 8 个 leaf bin 的头节点
std::vector<Node> m_nodes;
std::vector<NodeIndex> m_freeNodes;
uint32 m_freeOffset;           // free-list 栈顶
```

`m_nodes` / `m_freeNodes` 在 `reset()` 里 `reserve(m_max_capacity)`（`offset_allocator.cpp:175-176`），随后 `resize(m_current_capacity)`。所以**虚拟地址空间按上限预留，物理页随实际写入的节点增长**；不要据此断言 RSS 会等于 `max_capacity × sizeof(Node)`。

`m_multiplier_bits` 由 `calculateMultiplier`（`offset_allocator.cpp:539-545`）算出：

```cpp
for (; SmallFloat::MAX_BIN_SIZE < (size >> multiplier_bits); multiplier_bits++) {}
```

`MAX_BIN_SIZE = 4026531840`（3.75GB，`offset_allocator.cpp:60`）。也就是说不超过 3.75GB 的段 `multiplier_bits == 0`，全部按字节计；超过则整个地址空间按 `1<<bits` 压缩，`m_size = size >> bits` 才能塞进 `uint32`。**`Node::dataOffset/dataSize` 永远是 quantum 单位**，真实值要 `<< multiplier_bits`。

### 3.7 `OffsetAllocation` / `OffsetAllocationHandle`

`OffsetAllocation`（`offset_allocator.h:35-53`）只有两个 `uint32`：`offset`（quantum 单位）与 `metadata`（节点下标）。`NO_SPACE = 0xffffffff` 同时用作两者的空值。`getOffset()`（`:47`）返回 `uint64_t` 但**仍是 quantum 单位**——这是最容易被误读的一行。

`OffsetAllocationHandle`（`:70-117`）是 RAII 归还款：

```cpp
std::weak_ptr<OffsetAllocator> m_allocator;   // :108  ← 同样用 weak_ptr
OffsetAllocation m_allocation;                // :110
uint64_t real_base;                           // :112  ← 这里才是真实字节地址
uint64_t requested_size;                      // :113
```

`ptr()`（`:102`）= `reinterpret_cast<void*>(real_base)`。析构（`offset_allocator.cpp:531-536`）通过 `freeAllocation(m_allocation, requested_size)` 归还。移动赋值（`:507-529`）会先归还自己原有的分配，再接管对方的——所以 `handle = std::move(other)` 不会泄漏。

### 3.8 `SmallFloat`：3 位尾数的分段对数 size-class

`offset_allocator.cpp:56-121`。这是整个分配器的"分级"来源，本质是把 `uint32` 映射到一个 8 bit 的 float 表示（`exp` + 3 位尾数）：

- `uintToFloatRoundUp(size)`（`:65-89`）：返回**能装下 size 的最小 bin 编号**；
- `uintToFloatRoundDown(size)`（`:91-109`）：返回**size 落入的 bin 编号**；
- `floatToUint(bin)`（`:111-120`）：bin 的**下界**字节数。

`size < 8` 走 denorm 分支（`mantissa = size`），否则取最高位、截 3 位尾数并按需进位。由于是分段线性对数分布，"每个 size class 的平均相对浪费比例相同"（注释 `:62-64`）。

256 个 leaf bin 用一级 `m_usedBinsTop`（32 位）+ 二级 `m_usedBins[32]`（每个 8 位）表示非空，因此找可用 bin 是常数次位运算：`findLowestSetBitAfter`（`:124-130`）做 `mask & ~((1<<start)-1)` 再 `tzcnt`。

### 3.9 `MmapArena` / `ClientBufferAllocator`

`MmapArena`（`include/mmap_arena.h:19-106`）：

```cpp
static constexpr size_t kMinAlignment = 64;      // :21
std::atomic<void*>  pool_base_;                  // :93
std::atomic<size_t> pool_size_;                  // :95
std::atomic<size_t> alignment_;                  // :97
std::atomic<size_t> alloc_cursor_;               // :100 单调递增的 bump 指针
std::atomic<size_t> peak_reserved_;              // :101
std::atomic<size_t> num_allocations_, num_failed_allocs_;  // :102-103
std::mutex init_mutex_;                          // :105 只保护 initialize()
```

**没有 free 方法**。这是 bump allocator 的本质：只进不退，靠 `owns()`（`mmap_arena.cpp:271-281`）在释放路径上识别"这是 arena 的，跳过 munmap"。

`ClientBufferAllocator`（`include/client_buffer.h:27-72`）持有一块真实内存 + 一个 `OffsetAllocator`：

```cpp
void* buffer_; size_t buffer_size_; bool use_hugepage_;   // :59-61
std::shared_ptr<offset_allocator::OffsetAllocator> allocator_;  // :67
std::string protocol; bool is_external_memory_; bool use_spdk_dma_;  // :69-71
```

`BufferHandle`（`:82-112`）是双模式：owning 模式持 `shared_ptr<ClientBufferAllocator>` + `OffsetAllocationHandle`；view 模式持裸指针 + `std::function<void()> release_fn_`。`ptr()`/`size()`（`client_buffer.cpp:117-123`）按 `view_ptr_` 是否为空二选一。

### 3.10 `RegisteredPinnedMemoryManager`

`src/registered_pinned_memory.h`（注意在 `src/` 而非 `include/`，是 store 内部头）：

```cpp
class RegisteredPinnedRegion {          // :16-35
    RegisteredPinnedMemoryManager* manager_;   // 非拥有，靠 manager 全局单例活着
    void* addr_; size_t size_; bool release_succeeded_ = true;
};
struct ActiveRegion { void* addr; size_t size; RegisteredPinnedRegion* region; }; // :56-60
const uint64_t limit_bytes_;  const PinOps pin_ops_;   // :75-76
mutable std::mutex mutex_; uint64_t pinned_bytes_ = 0;
std::vector<ActiveRegion> regions_;
```

`PinOps`（`:41-46`）是两个 C 函数指针（register/unregister），默认实现走 `cudaHostRegister(..., cudaHostRegisterPortable)` 与 `cudaHostUnregister`（`registered_pinned_memory.cpp:20-41`）。这个注入点让单元测试可以在没有 CUDA 的机器上跑（`MOONCAKE_STORE_TEST` 下构造函数被提到 public，`:63-70`）。

### 3.11 设备抽象、`PinnedHostBuffer` 与 fabric

`include/device/accelerator_device.h`：

```cpp
enum class AcceleratorVendor { kNvidia, kMusa, kMaca, kHygon, kCorex,
                               kSupa, kHip, kAscend, kSunrise, kIntel };  // :12-23
enum class MemoryKind { kHost, kDevice, kUnknown };                       // :25-29
struct PointerInfo { MemoryKind kind; int32_t device_id = -1; };          // :39-42
class AcceleratorDevice {                                                 // :44-56
    virtual AcceleratorVendor Vendor() const = 0;
    virtual bool Available(bool ensure = false) const = 0;
    virtual PointerInfo QueryPointer(const void* ptr) const = 0;
    virtual int32_t CurrentDeviceId() const = 0;
    virtual void SetContext(int32_t device_id) const = 0;
    virtual bool Copy(void* dst, const void* src, size_t size, CopyDirection) const = 0;
    virtual PinnedHostBuffer AllocatePinnedHost(size_t size) const = 0;
};
```

`ProbeCachedAcceleratorDevice`（`:58-67`）用 `mutable std::atomic<uint8_t> available_state_` 缓存"这个厂商在不在"的探测结果，避免每次 `Available()` 都去初始化运行时。

`PinnedHostBuffer`（`include/pinned_host_buffer.h:10-49`）是 move-only + 函数指针 deleter，CUDA-like 实现里 deleter 就是 `cudaFreeHost` 的包装（`src/device/cuda_like_accelerator_device.cpp:71-78` 用 `cudaMallocHost`）。

同一份 `nvlink_allocator.cpp` 通过 `cuda_alike.h` 的类型别名被 hipcc / mcc / maca 编译成 HIP/MUSA/MACA 版本，这是"一次源码、多厂商产物"的关键。

### 3.12 指标结构

`include/allocator_metric.h`：

```cpp
struct JemallocBinStats { uint64 size, regs_per_slab, regs, slabs; };  // :31-36
struct JemallocSnapshot {                                              // :38-67
    uint64_t allocated, active, metadata, resident, retained, mapped;
    uint64_t small_allocated, large_allocated, dirty_bytes, muzzy_bytes;
    uint64_t arenas, background_threads;
    int64_t  opt_dirty_decay_ms, opt_muzzy_decay_ms;
    uint64_t dirty_purged_pages, ... , small_deallocations;   // cumulative
    std::vector<JemallocBinStats> bins;
};
using JemallocStatsCollector = bool (*)(JemallocSnapshot&);            // :69
```

`AllocatorMetric`（`:100-208`）有 7 个进程 gauge、15 个 jemalloc gauge、8 个 counter、4 个带 `size` 标签的 `hybrid_gauge_1t`。`kMinRefreshInterval{1000}`（`:139`）是刷新限流。

## 关键流程剖析

### 流程 A：MountSegment → 创建分配器 → 切出一个副本

这是 Master 侧最主干的路径。

```
客户端 RPC: MountSegment(segment{base,size,protocol,te_endpoint})
  → ScopedSegmentAccess::MountSegment            (segment.cpp:169)
      ├─ 查重（同名/同 id/同 base,size,te_endpoint,protocol）      (:189-206)
      ├─ CXL 分支：enable_cxl_ && protocol=="cxl"                    (:211-237)
      │     └─ 直接用 SegmentManager::cxl_global_allocator_（CACHELIB）
      ├─ CACHELIB 对齐校验（base/size 均须 16MB 对齐）               (:245-252)
      ├─ CreateBufferAllocator(type, name, base, size, endpoint)     (:255)
      │     └─ allocator.cpp:158-183  按枚举分派
      ├─ allocator->AttachUsageTracker(usage_tracker_)               (:262)
      ├─ allocator_manager_.addAllocator(name, allocator, liveness)  (:263-264)
      └─ mounted_segments_[id] = {segment, OK, allocator, registration} (:266-267)
```

三个细节：

1. **CXL 是特例**：`segment.cpp:213-236` 完全绕过 `CreateBufferAllocator`，直接用全局 CXL allocator，且只支持 CACHELIB（否则 `INTERNAL_ERROR`）。同一逻辑在新版 RegionDriver 里也是硬编码：`CreateRegionDrivers` 用 `BufferAllocatorType::CACHELIB` + `DEFAULT_CXL_BASE` 建 CXL allocator（`mooncake-store/src/segment/region_driver.cpp:303-320`），与配置的 `memory_allocator` 无关。
2. **`addAllocator` 包注册项**：`AllocatorManager::addAllocator`（`include/allocation_strategy.h:51-68`）构造 `SegmentAllocatorRegistration`，把 `ClientLivenessRecord` 一起塞进去。之后所有分配都走 `registration->Allocate(size)`，而不是直接调 allocator。
3. **恢复后补挂 tracker**：快照恢复出来的 allocator 不会走上面的 `AttachUsageTracker`，由 `SegmentManager::AttachMountedUsageTrackers`（`segment.cpp:142-158`）统一补。

写路径上真正的分配发生在 `MasterService::PutStart`：

```
PutStart(key, value_length)
  → (CACHELIB 且 slice_length > kMaxSliceSize) 直接 INVALID_PARAMS   (master_service.cpp:5506-5512)
  → allocation_strategy_->Allocate(allocator_snapshot, value_length,
                                   replica_num, preferred_segments, {}, MEMORY)
                                                                     (master_service.cpp:5196-5199)
      → RandomAllocationStrategy::allocateSingle                      (allocation_strategy.h:423-452)
          → registration->Allocate(slice_length)                      (segment_allocator_registration.cpp:22-45)
              → BufferAllocatorBase::allocate(size)                   （虚分派）
                  ├─ OffsetBufferAllocator::allocate                  (allocator.cpp:480)
                  └─ CachelibBufferAllocator::allocate                (allocator.cpp:235)
```

上面画的是默认 `AllocationStrategyType::RANDOM` 的路径。若配成 `FREE_RATIO_FIRST`/`SSD_FREE_RATIO_FIRST`/`LOCAL_FIRST`，会先经过 `RankedAllocationStrategy::AllocateRanked`（`allocation_strategy.h:465-563`）做"采样 + 打分 + 排序 + 兜底重试（`kMaxRetryLimit = 100`）"，但最终仍然落到同一个 `allocateSingle`，所以"唯一分配入口是 `registration->Allocate`"这个结论不变。

`AllocatorManager` 本身不做锁（头文件注释 `allocation_strategy.h:29-32`：线程安全由 `SegmentManager::segment_mutex_` 提供），所以主路径先在锁内 `SnapshotAllocatorManager()` 拷一份，再在锁外跑策略（`master_service.cpp:5193-5196`）。

`FreeRatioFirstAllocationStrategy`（`allocation_strategy.h:573-618`）的评分函数是：

```cpp
total_free += capacity - buffer_allocator->size();   // :607-609
return total_free / total_capacity;                  // :614
```

也就是说它的水位完全建立在 `size()`（请求字节和）之上——这正是 §3.3 那套 `cur_size_` 记账的下游消费者。

### 流程 B：OffsetAllocator 的一次 allocate 与一次 free

这是整个子系统算法密度最高的部分，逐段读 `mooncake-store/src/offset_allocator.cpp:191-357`。

**B1. 外壳层前置检查**（`OffsetAllocator::allocate`，`:880-935`）：

```cpp
if (size == 0) return std::nullopt;                          // :881-883
if (size > getLargestFreeRegion()) return std::nullopt;      // :889-891  无锁快速失败
const size_t fake_size = m_multiplier_bits > 0
    ? ((size + (1<<bits) - 1) >> bits) : size;                // :893-897  字节→quantum，向上取整
if (fake_size > SmallFloat::MAX_BIN_SIZE) return std::nullopt;// :899-901
MutexLocker guard(&m_mutex);                                  // :903
OffsetAllocation allocation = m_allocator->allocate(fake_size);  // :908
if (allocation.isNoSpace()) { refreshLargestFreeRegion(); ... return nullopt; }  // :909-921
m_allocated_size += size;  m_allocated_num++;                 // :924-925
const uint64_t real_base = m_base + (allocation.getOffset() << bits);  // :927-928
guard.unlock();                                               // :929
return OffsetAllocationHandle(shared_from_this(), allocation, real_base, size);  // :933
```

这里有两个精心设计的点：

- **`size > getLargestFreeRegion()` 的快速失败不需要拿锁**。`m_largest_free_region` 是 `atomic`，可能偏大（成功分配后不缩小），因此失败时它只是"多走一次加锁路径"，不会误拒；而一旦在锁内观察到真实的 `NO_SPACE`，就调用 `refreshLargestFreeRegion()`（`:1001-1013`）把提示收紧到权威值。头文件注释（`offset_allocator.h:180-184`）明确写了这三条规则：可能偏大、失败时收紧、free 时抬高。
- **handle 的构造在 `guard.unlock()` 之后**（`:929-934`）。注释解释了原因：构造 handle 与 `shared_ptr` 引用计数不访问 allocator 状态，不应延长临界区。

**B2. 内核层选 bin**（`__Allocator::allocate`，`:191-289`）：

```cpp
if (m_freeOffset == m_max_capacity) return NO_SPACE;            // :193-196 元数据耗尽
if (m_freeOffset == m_current_capacity) {                       // :197-201 按需扩容一个节点
    m_freeNodes.push_back(m_current_capacity);
    m_nodes.emplace_back();
    m_current_capacity++;
}
uint32 minBinIndex = SmallFloat::uintToFloatRoundUp(size);      // :205 最小可容纳 bin
uint32 minTopBinIndex  = minBinIndex >> 3;                      // :207
uint32 minLeafBinIndex = minBinIndex & 0x7;                     // :208
if (m_usedBinsTop & (1 << minTopBinIndex))                      // :214-218
    leafBinIndex = findLowestSetBitAfter(m_usedBins[minTopBinIndex], minLeafBinIndex);
if (leafBinIndex == NO_SPACE) {                                 // :220-233
    topBinIndex = findLowestSetBitAfter(m_usedBinsTop, minTopBinIndex + 1);
    if (topBinIndex == NO_SPACE) return NO_SPACE;
    leafBinIndex = tzcnt_nonzero(m_usedBins[topBinIndex]);
}
```

这就是"两级位图 + `tzcnt`/`lzcnt`"的常数时间查找：先在请求落入的 top bin 内找同层或更高的 leaf，找不到就往更高的 top bin 走。因为更高 top bin 的所有 leaf 都必然装得下，所以直接从 bit 0 取最低位即可。

**B3. Mooncake 对上游最关键的修改：round up 到 bin 边界**（`:241-249`）：

```cpp
// Modified in Mooncake project: Round up to bin size. Otherwise when this
// node is freed, if it cannot be merged with neighbors, it will be inserted
// to a smaller bin.
#ifdef OFFSET_ALLOCATOR_NOT_ROUND_UP
    uint32 roundupSize = size;
#else
    uint32 roundupSize = SmallFloat::floatToUint(minBinIndex);
#endif
```

为什么重要：设请求落在 bin B，`floatToUint(B) >= size`。若只按 `size` 占用，free 时该节点无法与邻居合并，会按 `roundDown(size)` 插入**更小**的 bin；下次同样大小的请求 `roundUp(size)` 又会去找 bin B，于是这个空闲节点永远匹配不上——uniform-size 负载下利用率断崖式下跌。官方基准（`docs/source/performance/mooncake/allocator-benchmark-result.md`）里 "Before Optimization / After Optimization" 两组表对应的就是这个 round-up 优化（文档写的是"优化前/优化后"两版分配器，而非直接编译期 A/B 宏）。`OFFSET_ALLOCATOR_NOT_ROUND_UP` 只在 `offset_allocator.cpp:244`、`:611` 与测试 `tests/offset_allocator_test.cpp:2159` 出现，没有 CMake 开关，需要手工 `-D`。

**B4. 摘节点 + 拆剩余**（`:250-288`）：

```cpp
node.dataSize = roundupSize;  node.used = true;                 // :250-251
m_binIndices[binIndex] = node.binListNext;                      // :252 弹出 bin 头
m_freeStorage -= nodeTotalSize;                                 // :255 注意减的是原 node 大小
if (m_binIndices[binIndex] == Node::unused) { ...清两级位图... } // :261-270
uint32 reminderSize = nodeTotalSize - roundupSize;              // :273
if (reminderSize > 0) {                                          // :274-286
    uint32 newNodeIndex = insertNodeIntoBin(reminderSize, node.dataOffset + roundupSize);
    // 把新节点挂进物理邻居链：node.neighborNext ↔ newNode ↔ 旧 neighborNext
}
return OffsetAllocation(node.dataOffset, nodeIndex);            // :288
```

**B5. free 与双向合并**（`:291-357`）：

```cpp
uint32 nodeIndex = allocation.metadata;
Node& node = m_nodes[nodeIndex];
ASSERT(node.used == true);                                       // :299 双 free 检查（Release 下空宏）
uint32 offset = node.dataOffset, size = node.dataSize;
if (prev 物理相邻且未使用) { offset = prev.offset; size += prev.dataSize;
                            removeNodeFromBin(prev); node.neighborPrev = prev.neighborPrev; }
if (next 物理相邻且未使用) { size += next.dataSize;
                            removeNodeFromBin(next); node.neighborNext = next.neighborNext; }
m_freeNodes[--m_freeOffset] = nodeIndex;                          // :341 归还节点槽位（LIFO 栈）
uint32 combinedNodeIndex = insertNodeIntoBin(size, offset);       // :344 整块重新入 bin
// 再修复邻居链
return m_freeOffset < m_max_capacity ? size : 0;                  // :356
```

`free` 的返回值有两重语义：正常时是"合并后的大小"，元数据耗尽时返回 0（`offset_allocator.h:271-273` 的注释）。上层 `freeAllocation`（`:1015-1051`）把它当成"新空闲区大小"，用于抬高 `m_largest_free_region`。

**B6. free 侧的三处防御**（`freeAllocation`，`:1015-1051`）：

- 只有 `m_largest_free_region_tightened` 为真时才需要考虑抬高提示（`:1022-1036`）：未收紧时提示已经 ≥ 所有现存空闲区的可服务大小（初始值是整段算出来的最大可服务请求，之后成功分配只会让它"过时偏高"而不会偏低），无需更新；合并出的区域 >= 全容量时再把 `tightened` 复位为 false。
- `m_allocated_size` 用**饱和减法**而不是回绕：`if (size > m_allocated_size) { LOG(WARNING) ...; m_allocated_size = 0; }`（`:1041-1048`）。注释说明恢复坏记录时可能 free 一个尺寸未知的节点，无符号回绕会永久污染指标。
- `removeNodeFromBin`（`:395-442`）有个隐藏成本：如果被删节点是 bin 的头节点，它必须用 `roundDown(node.dataSize)` **重新算出 bin 下标**（`:408`），因为 `Node` 里不存 `binIndex`。

**B7. 元数据上限是硬约束**。`m_freeOffset == m_max_capacity` 时直接返回 `NO_SPACE`（`:193-196`），哪怕还有大量空闲字节。这是"碎片化"之外的第二种失败模式，排查"明明还有空间却分配失败"时要先看这个。

### 流程 C：HA 恢复的两条路径

OffsetAllocator 体系的恢复有**两套完全不同的机制**，对应两种可用信息：

```
路径 1（有完整内部布局）：
  SegmentPool::CaptureSnapshot                       (segment/snapshot.cpp:12-41)
    → OffsetBufferAllocator::CaptureSnapshot          (allocator.cpp:463-467)
      → OffsetAllocator::CaptureSnapshot              (offset_allocator.cpp:830-837) 深拷贝 __Allocator
  → AllocatorSnapshotCodec::Encode                    (ha/snapshot/allocator_snapshot_codec.cpp:9-31)
  ... 持久化 ...
  → AllocatorSnapshotCodec::Decode                    (:33-95) 解码后立刻 snapshot.Validate()
  → OffsetBufferAllocator::Restore                    (allocator.cpp:436-461)
    → OffsetAllocatorSnapshot::Validate               (offset_allocator.cpp:804-818)
    → OffsetAllocator::Restore                        (:838-851)

路径 2（只有 live allocation 列表）：
  BuildRegionLiveAllocations(spec, descriptors)       (region_driver.cpp:322)
  → ImportOffsetBufferAllocator                       (allocator.cpp:567-651)
  → ImportCachelibBufferAllocator                     (allocator.cpp:310-349)
```

**C1. 快照校验是"全图校验"**。`OffsetAllocatorSnapshot::Validate()`（`offset_allocator.cpp:804-818`）串起六个检查，顺序不能换：

| 步骤 | 函数 | 检查什么 |
|---|---|---|
| 1 | `ValidateHeader`（`:560-597`） | `layout` 非空；`capacity != 0` 且 `base + capacity` 不溢出；`multiplier_bits` 必须**等于重新算出来的期望值**（`:571-576`，这行同时守卫了后续所有移位）；`layout->m_size == capacity >> bits`；`ValidateNodeCapacity`；`nodes/freeNodes.size() == m_current_capacity` |
| 2 | `ValidateUsedNodes`（`:599-636`） | 每个 used 节点的 `dataSize` 必须**恰好是 bin 边界**（`:611-621`，仅在 round-up 生效时）；累计 `max_used_bytes` 不超 capacity；同时算出 `min_used_bytes`（该 bin 的最小可能请求字节） |
| 3 | `ValidateBins`（`:638-698`） | 遍历 256 个 leaf bin 的链表：节点下标在界内、未被别的 bin 认领（用 `NodeState` 数组分类，天然防环）、`roundDown(dataSize) == bin`、`binListPrev` 链一致；重算 `m_freeStorage`、`m_usedBinsTop`、`m_usedBins[]` 并逐一比对 |
| 4 | `ValidateFreeStack`（`:700-725`） | `m_freeNodes[m_freeOffset..]` 里的每个下标必须未被分类；`m_freeOffset == live_nodes`；最后不允许有 `kUnclassified` 残留 |
| 5 | `ValidateNeighborChain`（`:727-785`） | 物理邻居链必须**恰好一个头**、无环、`dataOffset` 严格连续递增、`offset + size == m_size` 且节点数 == live_nodes |
| 6 | `ValidateAccounting`（`:787-802`） | `allocated_num == used_nodes`；`allocated_size ∈ [min_used_bytes, max_used_bytes]` |

第 6 步的区间语义值得单独强调：快照里**不保存每个节点的原始请求 size**，所以 `allocated_size` 只能验证到一个区间里，而不是精确相等（`offset_allocator.h:342-345` 的注释）。这也解释了为什么 `used_bytes` 与 `allocated_size` 在元数据层要求严格相等（`allocator.cpp:422-426`），而在节点层只能区间校验。

`Decode` 在返回前就调用 `snapshot.Validate()`（`allocator_snapshot_codec.cpp:82-87`），`Restore` 再校验一次 `ValidateMetadata()`（`allocator.cpp:439-443`）。两次校验的注释解释：解码层负责把诊断信息带给调用方，工厂层负责拒绝非法状态。

**C2. 反序列化的两道容量闸门**。`__Allocator` 的反序列化构造（`offset_allocator.h:453-501`）在 `resize` 之前检查：

```cpp
static constexpr uint32 kMaxSerializedNodes = 1u << 24;   // :472
if (m_max_capacity == 0 || m_max_capacity > kMaxSerializedNodes ||
    m_current_capacity > m_max_capacity ||
    m_freeOffset > m_current_capacity || m_size == 0) throw ...
```

注释（`:466-471`）说明动机：一份"能解析但荒谬"的 meta 会先触发 OOM killer，而不是抛出 `bad_alloc`，从而绕过"corrupt meta → fresh start"的兜底。外层 `OffsetAllocator` 的反序列化（`:396-429`）另有一道：`m_multiplier_bits >= 32 || m_capacity == 0` 直接抛（`:407-414`）。注意这两个上限不一致：host 内存分配器允许 `kMaxNodes = 1<<26`（`offset_allocator.h:334`），而 wire 解码器只允许 `1<<24`——注释解释说 storage backend 的字节序列化器用更小的上限。

**C3. 重放式恢复：`ImportOffsetBufferAllocator`**（`allocator.cpp:567-651`）。

输入只有 `vector<LiveAllocation>{offset_from_base, requested_size}`。算法：

```
1. 造一个空的 OffsetBufferAllocator（新布局，整段一个 free node）    :575-577
2. 按 offset 升序排序（保留原下标，最后按输入顺序返回 buffers）        :579-584
3. cursor = base                                                     :588
4. 对每个 allocation（升序）：
     a. 校验 IsValidAllocation（size!=0、offset<capacity、不越界）      :627-629
     b. address = base + offset；若 address < cursor 说明重叠           :630-633
     c. occupied = normalizedAllocationSize(requested_size)             :634-635
        （必须按"分配时实际占用"算，因为 allocate 会 round up 到 bin）
     d. fill_gap(address - cursor)：把 [cursor, address) 这段空洞填满  :636-637
     e. allocate(requested_size)，并要求返回地址恰好 == address          :640-643
     f. cursor = address + occupied                                     :644
5. gaps.clear() —— 一次性释放所有填缝 buffer，让它们合并回空闲区         :648
6. 返回 {allocator, buffers}（buffers 按输入顺序）                      :649-650
```

`fill_gap`（`:590-623`）本身是"二分找最大可填请求 + 循环"：

```cpp
while (gap != 0) {
    // 在 [1, gap] 里二分，找最大的 request 使 normalizedAllocationSize(request) <= gap
    ...
    auto filler = allocator->allocate(request);
    if (!filler || (size_t)filler->data() != cursor) return false;   // 地址必须精确
    cursor += occupied; gap -= occupied; gaps.push_back(std::move(filler));
}
```

**为什么这一定成功而且布局逐字节一致**？关键在于不变量：整个过程按地址升序消费，任何时候 allocator 里**最多只有一个空闲节点**（即 `[cursor, end)` 这段尾巴）。因此：

- 每次 `allocate` 都必然从这段尾巴的头部切出，地址可预测；
- `fill_gap` 用 `normalizedAllocationSize` 精确算出"这次 allocate 会吃掉多少字节"，填到 `gap` 恰好归零；
- 最后所有 filler 都是物理相邻且空闲，`gaps.clear()` 触发一连串 free + 双向合并，重新合成一个 `[最后一个 live 区之后, end)` 的空闲节点。

测试 `tests/buffer_allocator_test.cpp:244-260` 专门验证了这个性质：65537 个"1 字节分配 + 1 字节空洞"的极端稀疏布局也能精确还原，并检查最后一个 buffer 的地址。同一文件 `:145-183` 还验证了"恢复后重新分配被删对象，地址回到原处"。

任何一个步骤失败都返回 `std::nullopt`，此时局部构造的 allocator 与 buffers 随作用域销毁，**不发布任何状态**（头文件 `allocator.h:443-445`）。

**C4. CacheLib 的恢复完全不是一回事**。`ImportCachelibBufferAllocator`（`allocator.cpp:310-349`）走的是 `MemoryAllocator::importAllocations`：

```cpp
// allocator.cpp
if (replica_type != MEMORY || !IsValidCachelibLayout(base, size)) return nullopt;  // :314-317
imports.push_back({ reinterpret_cast<void*>(base + offset),
                    static_cast<uint32_t>(std::max(requested_size, kMinSliceSize)) });  // :325-328
// 另外要求 requested_size <= UINT32_MAX（:321-323）
allocator->memory_allocator_->importAllocations(pool_id_, imports);                 // :337
```

而 `MemoryAllocator::importAllocations`（`src/cachelib_memory_allocator/MemoryAllocator.cpp:90-168`）做的事是：

1. 池必须是空的（`:93-95`）；
2. 用 `getSlabForIdx(0)` 反推 slab 区间 `[base, base + slabCount*16MB)`（`:105-112`）；
3. 对每个地址算 `slabIndex = (addr-base)/16MB`、`slabOffset = (addr-base)%16MB`、`chunkIndex = slabOffset/allocSize`，要求**地址必须落在 chunk 边界上**（`:127-134`）；
4. 同一 slab 内不允许两个不同 class（`:136-138`）、不允许重复 chunk（`:143-149`）；
5. 按 slabIndex 升序 `makeNewSlab`，并要求返回地址与 `base + idx*16MB` 精确匹配（`:151-157`）——这是"从 0 号 slab 顺序分配"这一 CacheLib 内部行为的强假设；
6. 有 allocation 的 slab 调 `pool.importSlab(slab, classId, occupied)`，没有的调 `pool.importFreeSlab(slab)`（`:158-165`）。

恢复出的 `AllocatedBuffer` 由 `adoptImportedBuffer`（`allocator.cpp:294-308`）构造，地址是 `base + offset_from_base`，`size` 用 `requested_size`（而不是 class size）。

两者的差异可以一句话概括：**OFFSET 的 import 是"重放操作序列"，CACHELIB 的 import 是"直接写入 slab 的 chunk 占用位图"**。前者不信任任何内部结构、只信地址与大小；后者依赖 CacheLib 的 slab/class/chunk 几何关系，因此对地址合法性要求严格得多（必须 chunk 对齐）。

### 流程 D：客户端缓冲——真实内存从哪来，怎么被 free

```
ClientBufferAllocator::create(size, protocol, use_hugepage, use_spdk_dma)   (client_buffer.cpp:19-24)
  → 构造 (client_buffer.cpp:32-66)
      ├─ alignment = 64                                          (:46)
      ├─ use_hugepage_ ? allocate_buffer_mmap_memory(size, 64)   (:47-48)
      │                 └─ mmap_buffer.cpp:203-260
      ├─ sunrise_link ? sunrise_allocate_memory(...)             (:50-53，仅在非 hugepage 分支内)
      └─ else allocate_buffer_allocator_memory(size, protocol, 64, use_spdk_dma)
      │                 └─ client_buffer_allocation.cpp:97-143
      └─ OffsetAllocator::create(reinterpret_cast<uint64_t>(buffer_), size)  (:64-65)
```

`allocate_buffer_allocator_memory`（`client_buffer_allocation.cpp:97-143`）的编译期分支顺序就是"协议优先级"：

| 条件 | 去处 |
|---|---|
| `USE_ASCEND_DIRECT`/`USE_UBSHMEM` 且 protocol ∈ {ascend, ubshmem} | `ascend_allocate_memory`（`:106-109`） |
| `USE_MEMFABRIC` 且 protocol == ascend | `memfabric_allocate_memory`（`:110-114`） |
| `USE_SUNRISE` 且 protocol == sunrise_link | `sunrise_allocate_memory`（`:115-121`） |
| `USE_UB` 且 protocol == ub | `ub_allocate_memory`（`:122-126`） |
| `USE_NOF` 且 `use_spdk_dma` | `SpdkWrapper::Alloc`（`:127-132`） |
| `USE_VRAM_SEGMENT` | `nvlink_intra ? allocateFabricMemory_intra : cudaMalloc`（`:133-140`、`:31-59`） |
| 兜底 | `aligned_alloc(alignment, total_size)`（`:142`） |

**free 必须镜像同一条路径**（`client_buffer_allocation.cpp:145-184`），注释（`:166-169`）特别点明：SPDK 大页池取出的内存必须 `spdk_free`，用 glibc `free()` 会 `abort with "free: invalid pointer"`。这是个典型的"分配器必须成对"陷阱，而它之所以危险，是因为 `#ifdef` 分支在错误的构建组合下会静默走错。

`ClientBufferAllocator::~ClientBufferAllocator`（`client_buffer.cpp:77-86`）只在 `!is_external_memory_` 时释放；`create(void* addr, size_t size, ...)`（`:26-30`、`:68-75`）用于外部已有内存（例如共享内存映射），此时 allocator 只做子分配，不负责底层释放。

`MmapArena` 路径（`mmap_buffer.cpp:203-260`）值得单独讲：

```cpp
std::call_once(g_arena_init_flag, initializeGlobalArena);          // :211 每进程只尝试一次
if (g_mmap_arena && g_mmap_arena->isInitialized()) {               // :215
    void* ptr = g_mmap_arena->allocate(total_size, alignment);     // :216
    if (ptr) return ptr;
    // OOM 才降级到直接 mmap（:222-229，LOG_FIRST_N 限流）
}
flags = MAP_PRIVATE | MAP_ANONYMOUS;                               // :235
if (!defer_direct_population) flags |= MAP_POPULATE;               // :236-238
hugepage_size = get_hugepage_size_from_env(&flags);                // :239
map_size = mmap_map_size(total_size, hugepage_size);               // :240
```

而 `free_buffer_mmap_memory`（`:267-297`）用 `g_mmap_arena->owns(ptr)` 区分：

```cpp
if (g_mmap_arena && g_mmap_arena->owns(ptr)) {                     // :275
    g_arena_noop_free_count++;                                     // :276-277
    LOG_FIRST_N(WARNING, 3) << "... does not individually release arena-owned pointer ...";
    return;                                                        // :284 ← 故意不 munmap
}
munmap(ptr, mmap_map_size(total_size, get_hugepage_size_from_env()));  // :288-296
```

**arena 里分配的内存永远不会被单独归还**，只有进程退出时 `MmapArena::~MmapArena`（`mmap_arena.cpp:43-52`）整体 `munmap`。这决定了它只适合"长生命周期、申请后不释放"的场景（SGLang HiCache 的主机 KV 池），不适合当作通用分配器。

`MmapArena::initialize`（`mmap_arena.cpp:54-166`）的决策树：

1. `std::lock_guard<std::mutex> lock(init_mutex_)`（`:60`）——注释（`:56-59`）说明这是为了修掉早期用 CAS 时"输掉的线程覆写 `alignment_/pool_size_`"的元数据竞态。
2. 校验 `pool_size > 0`、`alignment` 是 2 的幂、未初始化过（`:62-75`）。
3. 池大小按 `max(hugepage_size, 2MB)` 对齐（`:82-88`），并做溢出保护。
4. `flags = MAP_PRIVATE|MAP_ANONYMOUS|MAP_POPULATE`（`:94`），若开启大页则加 `MAP_HUGETLB` + 具体 size 位（`:97-104`）。
5. **降级策略**：mmap 失败时，若 `allow_regular_page_fallback == false`（即 `MC_STORE_USE_HUGEPAGE` 显式要求大页）直接失败（`:111-120`）；否则清掉 `MAP_HUGETLB` 与 size 位重试（`:122-141`）。
6. `madvise(MADV_DONTFORK)`（`:149-153`）：`MAP_POPULATE` 意味着所有页都有物理后备，fork 出的子进程会继承一个巨大的 COW 映射（注释举例 64GB），可能直接触发 OOM killer。
7. **发布顺序**：先 relaxed 写 `alignment_`/`pool_size_`，最后 release 写 `pool_base_`（`:155-160`）。`allocate()` 用 acquire 读 `pool_base_`，之后再 relaxed 读另外两个字段——靠 release/acquire 建立 happens-before，这个模式在 `allocate` 的注释里也重复了一遍（`:179-184`）。

`MmapArena::allocate`（`:168-258`）的 CAS 循环有三处容易写错但这里写对了的地方：

- **对齐的是 offset（而不只是 size）**（`:205-213`；`size` 在 `:188-194` 已对齐，随后在 CAS 循环里再把 offset 也对齐），否则"前一次小对齐分配把 cursor 停在非对齐位置"时，后续大对齐请求返回的指针会不满足契约；
- `next = aligned_offset + aligned_size` 后先检查 `next < aligned_offset`（回绕）与 `next > pool_size`（OOM）**再** CAS（`:215-225`）；
- CAS 的 expected 用 `raw`（原始 cursor）而不是 `aligned_offset`（`:230-234`），失败后重新读。

### 流程 E（补充）：GPU fabric 分配器

Python 侧（`mooncake-integration/allocator.py`）：

```python
class MemoryBackend(IntEnum):
    USE_CUDAMALLOC = 0; USE_CUMEMCREATE = 1; UNKNOWN = -1; UNSUPPORTED = -2   # :17-21

NVLinkAllocator.get_allocator(device)                                        # :82-89
  → CUDAPluggableAllocator(so_path, "mc_allocator_malloc", "mc_allocator_free")
```

注意：**`allocator.py` 里没有任何分配逻辑**。它只做两件事：(1) 用 `probe_allocator_backend`（`fabric_allocator_utils.py:33-58`，本质是 `ctypes.CDLL` + `getattr(symbol)` + 调一次；符号缺失时返回 `unsupported_value` 而不抛）探测后端类型；(2) 把 `.so` 的两个 C 符号注册成 torch 的 pluggable allocator，之后真正的分配由 torch 在需要时回调。

C 侧（`mooncake-transfer-engine/nvlink-allocator/nvlink_allocator.cpp`）：

```
mc_allocator_probe(device_id)                                    (:168-170)
  → ProbeAllocatorBackend                                        (:24-53)
      ├─ cuDeviceGet 失败 → unknown                              (:26-29)
      ├─ CU_DEVICE_ATTRIBUTE_HANDLE_TYPE_FABRIC_SUPPORTED 为 0 → use_cudamalloc  (:31-36)
      └─ 用 4KB 做一次 cuMemCreate(CU_MEM_HANDLE_TYPE_FABRIC) 试分配        (:38-52)
           成功 → cuMemRelease + use_cumemcreate；失败 → use_cudamalloc

mc_allocator_malloc(size, device, stream)                        (:172-174)
  → AllocateFabricMemory                                         (:55-135)
      ├─ cuDeviceGet                                              (:63-67)
      ├─ prop = PINNED / DEVICE / location.id = dev               (:68-70)
      ├─ fabric_supported ? prop.requestedHandleTypes = FABRIC    (:72-82)
      ├─ GPU_DIRECT_RDMA_WITH_CUDA_VMM_SUPPORTED → allocFlags.gpuDirectRDMACapable = 1  (:84-91)
      ├─ cuMemGetAllocationGranularity(MINIMUM)，size 向上对齐      (:92-99)
      ├─ cuMemCreateTryFabric                                     (:100-104)
      ├─ cuMemAddressReserve                                      (:105-110)
      ├─ cuMemMap                                                 (:111-117)
      └─ cuMemSetAccess 给**所有**设备 READWRITE                   (:118-133)
```

`cuMemCreateTryFabric`（`:7-18`，注明 ref nvidia/nccl v2.28.9-1 的 `allocator.cc`）是关键的降级点：当请求了 FABRIC handle 但驱动返回 `CUDA_ERROR_NOT_PERMITTED`/`NOT_SUPPORTED` 时，清掉 `CU_MEM_HANDLE_TYPE_FABRIC` 位再试一次。这样同一份代码在"支持 fabric"和"只有普通 VMM"两种机器上都能工作。

`cuMemSetAccess` 遍历 `device_count` 把**每一张** GPU 都设为可读写（`:118-126`），这是跨节点 NVLink 共享 KV 池能成立的前提——任何一张卡都能直接访问这块显存。

`FreeFabricMemory`（`:137-158`）的顺序是 `cuMemRetainAllocationHandle` → `cuMemGetAddressRange` → `cuMemUnmap` → `cuMemAddressFree` → `cuMemRelease`。注释（`:150`）提示"不是所有 runtime 都接受 NULL 的 base 输出，总是传一个真实指针"，这是踩过的坑。

构建侧（`mooncake-transfer-engine/fabric_allocator.cmake:1-44`）：`add_fabric_allocator_build_target` 用 `add_custom_command` 调外部 `bash ${FAB_BUILD_SCRIPT}` 在 `CMAKE_CURRENT_BINARY_DIR` 下产出 `.so`，并 `add_dependencies(${FAB_TARGET_NAME} transfer_engine)`。之所以走外部脚本，是因为要支持用 `hipcc`/`mcc`/`maca` 编译同一份 `.cpp`——这些编译器切换不适合塞进 CMake 的编译器探测。

### 流程 F（补充）：指标链路

`AllocatorMetric::Refresh()`（`allocator_metric.cpp:173-186`）：

```cpp
std::lock_guard<std::mutex> guard(refresh_mutex_);            // :175
if (now - last_refresh_ < kMinRefreshInterval) return;        // :176-178 1s 限流
last_refresh_ = now;  RefreshLocked();                        // :179-180
```

限流的原因写在头文件（`allocator_metric.h:110-118`）：推进 jemalloc epoch 会拿一把与分配线程共享的锁，如果 `/metrics` 是无认证端点，高频抓取会直接打击分配性能。

`RefreshProcess()`（`:188-215`）逐行读 `/proc/self/status`，把 `VmRSS` / `VmHWM` / `RssAnon` / `RssFile` / `RssShmem` / `VmSize` / `VmSwap` 的 kB 值乘 1024 后灌进 gauge。

`RefreshJemalloc()`（`:217-272`）先从原子变量取采集器指针：

```cpp
const JemallocStatsCollector collector = g_jemalloc_collector.load(std::memory_order_acquire);  // :218-219
if (collector == nullptr || !collector(snapshot)) { jemalloc_enabled_.update(0); return; }      // :221-224
```

**这一行解释了为什么"库本身不链接 jemalloc"也能有指标**：`SetJemallocStatsCollector`（`:40-42`）是个全局原子函数指针，由链接了 jemalloc 的可执行文件在 `main()` 里注入：

```
master.cpp:1432-1433        LogAllocatorStatus(); InstallAllocatorStatsCollector();
real_client_main.cpp:128-129  同上
```

`InstallAllocatorStatsCollector`（`allocator_status.h:220-222`）把 `CollectJemallocStats`（`:94-212`）注册进去；未链接 jemalloc 时它和 `LogAllocatorStatus`（`:226-228`）都是空函数，所以调用点不需要 `#ifdef`。**没装 collector 时 `mooncake_jemalloc_enabled = 0` 是正确值，不是故障**。

`AdvanceCounter`（`allocator_metric.cpp:274-283`）把 jemalloc 的累计值转成 ylt counter：

```cpp
if (total < previous) return;                 // :276-280 采样回退时保持单调，丢弃本次
counter.inc(total - previous);  previous = total;
```

`CollectJemallocStats` 的两个前置检查也值得记：`config.stats` 必须为真（`allocator_status.h:98-101`，否则"每次 stats.* 读都失败、整份快照看起来像健康的 0"），`arenas.page != 0`（`:110-113`，否则 dirty/muzzy 的页数换算成字节会静默变 0）。

## 与其他模块的边界

**向上（谁使用分配器）**

| 使用方 | 用的什么 | 接口 |
|---|---|---|
| `SegmentManager` / `ScopedSegmentAccess` | 段分配器 | `CreateBufferAllocator`、`AttachUsageTracker`、`AllocatorManager::addAllocator`（`segment.cpp:142-158`、`:255-264`） |
| `MasterService::PutStart` 等写路径 | `AllocationStrategy` | `allocation_strategy_->Allocate(...)`（`master_service.cpp:5196`、`:5229`、`:9896`） |
| `SegmentPool` / `RegionDriver`（新版快照路径） | `CaptureSnapshot` / `Restore` / `Import*`（`segment/snapshot.cpp:12-41`、`region_driver.cpp:215-245`） | |
| `FileStorage` / `AlignedClientBufferAllocator`（O_DIRECT） | `allocate_buffer_allocator_memory`（4096 对齐）（`aligned_client_buffer.cpp:37-56`） | |
| `ClientBufferAllocator` | `OffsetAllocator` 子分配 + `allocate_buffer_mmap_memory` | `client_buffer.cpp:47-65` |
| `RealClient` | `TryPinStoreSegment` / `ReleasePinnedRegionForFree`（`real_client.cpp:156-177`） | |
| `store_py.cpp` | `hugepage_memory_alloc` 的函数地址（`memory_alloc.cpp:7-19`） | |
| Python 推理进程 | `nvlink_allocator.so` 的两个 C 符号（`allocator.py:82-89`） | |

**向下（分配器依赖谁）**

- vendored CacheLib 子集：`mooncake-store/{include,src}/cachelib_memory_allocator/`，独立 STATIC 库，自带 `fmt`/`folly` 头桩；
- sebbbi/OffsetAllocator 的移植版（MIT），Mooncake 加了 thread-safe 外壳、multiplier、快照、round-up；
- yalantinglibs：`ylt::metric::{gauge,counter,hybrid_gauge_1t}`、`tl::expected`、`YLT_REFL`；
- `MasterMetricManager`（每段 allocated/capacity 指标）、`StorageUsageTracker`（水位）；
- msgpack（HA 快照）、`Serializer<T>` 模板（`include/serialize/serializer.h`）；
- CUDA Driver/Runtime（`cuMemCreate`/`cuMemMap`/`cudaHostRegister`/`cudaMallocHost`）、`cuda_alike.h` 多厂商别名；
- libnuma / hugetlb / THP（`mmap_buffer.cpp:178-434`）；
- SPDK（`USE_NOF`）、libacl（Ascend）、ubshmem、`libaccl_barex`、Sunrise `tang_runtime`；
- gflags/glog/fmt。

**关键契约**

1. **`base` 的语义随调用方而变**。同一个 `OffsetAllocator` 类在四个地方出现：
   - `OffsetBufferAllocator`：`base` 是**客户端上报的远端虚拟地址**（Master 不能解引用）；
   - `ClientBufferAllocator`：`base` 是**本进程的真实指针**（`client_buffer.cpp:64-65`）；
   - `OffsetAllocatorStorageBackend`：`base = 0`，分配的是**文件内字节偏移**（`src/storage_backend.cpp:5570` 的 `OffsetAllocator::create(0, capacity_)`；`ObjectEntry::offset` 见 `include/storage_backend.h:1388-1390`，是 `uint64_t` 文件偏移）；
   - DFS：同样 `base = 0`，分配的是 **shard 文件内偏移**（`src/storage/distributed/shard_allocator.cpp:237`）。
   改任何一处逻辑前，先确认 `base` 指向什么。
2. **`AllocatedBuffer` 的 `size()` 是请求大小，不是占用大小**。CacheLib 侧实际占用 >= `max(size, 64)` 且向上取整到 size-class；Offset 侧实际占用 = `normalizedAllocationSize(size)`（round up 到 bin，再乘 quantum）。这直接影响 `size()/capacity()` 的利用率读数。
3. **Descriptor 是跨进程契约**：`{size, buffer_address, protocol, transport_endpoint}`。`protocol == "cxl"` 时 `buffer_address` 是相对 `DEFAULT_CXL_BASE` 的偏移（`allocator.cpp:118-128`）。
4. **恢复接口不发布半成品**：`Restore` / `Import*` 全部"先校验、再构造、最后才记账"（`allocator.cpp:439-460`、`:648-650`）。

## 设计取舍与易踩坑点

### 6.1 为什么默认 OFFSET 而不是 CACHELIB

| 维度 | CACHELIB | OFFSET |
|---|---|---|
| 段对齐要求 | `base`、`size` 都必须 16MB 对齐（`allocator.cpp:17-24`、`segment.cpp:245-252`） | 无（只要求 `base + size` 不溢出） |
| 内部碎片 | slab 内按 size-class 切，同一 slab 只能服务一个 class；class 间不共享（`Slab.h:34-52`，`MemoryAllocator.h:31-53` 的层级说明） | bin 之间按物理相邻合并，碎片可回收 |
| 单对象上限 | `kMaxSliceSize = 16MB - 16`（`types.h:451`），且 `PutStart` 在 CACHELIB 下显式拒绝更大对象（`master_service.cpp:5506`） | 无 16MB 限制，只受 `MAX_BIN_SIZE`(3.75GB) 与 quantum 约束 |
| 最大空闲块查询 | 只能返回 `kAllocatorUnknownFreeSpace`（`allocator.h:301-308`）——因为 CacheLib 不暴露"最大连续空闲" | `storageReport().largestFreeRegion` 精确可查（`allocator.cpp:548-565`） |
| 并发粒度 | 每个 AllocationClass 一把锁 + MemoryPool 锁 + SlabAllocator 锁（`AllocationClass.h:384`、`MemoryPool.h:317`、`SlabAllocator.h:174`） | 每段一把锁（`offset_allocator.h:233`），临界区短（bitmap + 链表指针） |
| 元数据位置 | `SlabHeader` 数组在 Master 本地堆（`allocator.cpp:199-203`） | `Node` 数组在 Master 本地堆（`offset_allocator.cpp:175-179`） |
| 恢复方式 | `importAllocations` 写 chunk 位图 | 快照全图校验 / 地址重放 |
| HA 序列化支持 | 不支持（`segment.cpp:761-765` 明确拒绝非 OFFSET 的 SegmentManager 序列化） | 支持（`AllocatorSnapshotCodec`） |

结论：CACHELIB 的价值在于"成熟、被 Facebook 验证过"，但它的 16MB slab/class 语义与 Mooncake 的对象尺寸分布不匹配（LLM KV 对象常常远小于 16MB，且尺寸相近），加上对齐约束和不可查询空闲块，所以默认走 OFFSET。中文文档 `docs/source/zh_archive/mooncake-store.md` 也把 CacheLib 标为不推荐。

**但要注意历史残留**：`SegmentManager` 的构造函数默认参数（`segment.h:387`、`:488`）和 `RegionDriverConfig::memory_allocator`（`segment/region_driver.h:117`）仍写着 `CACHELIB`。真正的默认值在 `MasterServiceConfig`（`master_config.h:275`/`:1255`/`:885`）——`OFFSET`。读代码时别被构造函数默认参数带偏，要看谁实际传参（`master_service.cpp:209` 把 config 的值交给 `memory_allocator_type_`）。

### 6.2 round-up-to-bin 是"用一点浪费换碎片可控"

见流程 B 的 B3。补充两点：

- 这个改动让**分配尺寸统一化**：所有落进 bin B 的请求都占用 `floatToUint(B)` 字节，free 时能精确落回 bin B，从而与同 bin 的邻居合并概率大增。
- 它也让 `getLargestFreeRegion()` 在语义上精确：因为任何大于 `floatToUint(B_max)` 的请求都会 round up 到更高的 bin，而更高 bin 一定为空。所以"返回 bin 下界"与"返回最大可服务请求"等价。测试 `tests/buffer_allocator_test.cpp:128-143` 就是断言这一点（并顺带区分了"提示值"与"权威值"）。

### 6.3 `multiplier_bits`：不要直接把 offset 当字节

超过 3.75GB 的段会引入 `m_multiplier_bits`（`offset_allocator.cpp:539-545`、`:853-863`）。所有 `Node::dataOffset/dataSize`、`m_size`、`m_freeStorage`、`OffsetAllocation::offset` 都是 quantum 单位；只有 `OffsetAllocationHandle::real_base`（`offset_allocator.h:112`）与 `OffsetAllocator::m_base/m_capacity` 是字节。换算公式：

```
real_addr = m_base + (offset << multiplier_bits)
real_size = size << multiplier_bits
real_qty  = (real_qty_bytes + (1<<bits) - 1) >> bits    // 向上取整，见 allocate
```

`normalizedAllocationSize`（`:937-952`）把整个"字节 → quantum → bin round-up → 字节"的链条封装成一个函数，`fill_gap` 的正确性完全依赖它与 `allocate` 的一致性。

### 6.4 元数据上限是第二种 OOM

`m_freeOffset == m_max_capacity` 时 `__Allocator::allocate` 直接返回 `NO_SPACE`（`offset_allocator.cpp:193-196`）。对 Master 侧，`max_capacity` 被夹在 `1<<26`：

- 64GB 段 → 6710 万节点，平均 1KB/节点；
- 大于 64GB 的段 → 仍只有 6710 万节点，平均粒度变大。

所以**大段 + 小对象**的组合会先耗尽节点而不是字节。排查 "`allocation_failed` 但 `current_size` 远小于容量" 时，必须同时看节点数。

另外注意 `m_nodes.reserve(m_max_capacity)`（`:175`）只是预留虚拟地址空间；实际 RSS 随写入的节点增长。不要据此估算内存占用。

### 6.5 `getLargestFreeRegion` 有两种语义

- `OffsetAllocator::getLargestFreeRegion()`（`offset_allocator.h:182-184`）：**无锁提示**，可能偏大（成功分配后不缩小），失败后被收紧，free 时被抬高。只用于快速失败。
- `OffsetBufferAllocator::getLargestFreeRegion()`（`allocator.cpp:548-565`）：**加锁读权威值**（转调 `storageReport()`）。
- `CachelibBufferAllocator::getLargestFreeRegion()`：恒返回 `kAllocatorUnknownFreeSpace`（`allocator.h:306-308`），表示"不知道"，调用方必须按"可能装得下"处理。

顺带说明一个当前状态：`BufferAllocatorBase::getLargestFreeRegion()` 这个虚函数在 store 主路径上**目前没有调用点**（grep 全仓只有定义与测试），`AllocationStrategy` 的候选过滤实际用的是 `capacity()/size()`。头文件里"used for filtering"的注释属于设计意图而非既成事实。

### 6.6 两套记账、两个"用量"

- `BufferAllocatorBase::size()` / `cur_size_`：请求字节和，供 `FreeRatioFirstAllocationStrategy` 与 `GetMemoryUsageSnapshot` 使用。
- `MasterMetricManager::inc_allocated_mem_size(segment, bytes)`：Prometheus 每段指标，供运维观察。

两者都按**请求大小**记，因此都会**低估**真实占用（CacheLib 的 size-class、Offset 的 bin round-up、以及 64B 最小分配）。做容量规划时不要拿它当物理占用。

`AllocatedBuffer` 析构时如果 `weak_ptr` 已过期，`deallocate` 不会被调用，指标也不会减——但此时 allocator 的析构函数（`allocator.cpp:224-233` / `:469-478`）会 `dec_allocated_mem_size(segment_name_, size())` 一次性地把该段的量减掉，并 `remove_segment_metrics`。两条路径不会重复扣减，因为 allocator 死了之后 buffer 也不可能再持有它。

### 6.7 MmapArena 的 free 是 no-op（不是 bug）

见流程 D。三个直接后果：

1. `free_buffer_mmap_memory` 对 arena 指针**不 munmap**，只累加 `g_arena_noop_free_count` 并限流告警（`mmap_buffer.cpp:275-285`）。
2. 指针被 `owns()` 判定为 arena 的，就不能再传给普通 `munmap`；反过来，非 arena 指针走 `munmap(ptr, mmap_map_size(total_size, ...))`，**size 必须与分配时一致**，所以 `mmap_map_size` 被抽成公共函数（`mmap_buffer.cpp:72-76`）。
3. arena 初始化"每进程只尝试一次"（`std::call_once`，`:211`/`:273`），失败即永久回退直接 mmap。日志明确写了"This is only attempted once per process"（`:62-64`）。

arena 是 opt-in：`--use_mmap_arena_allocator` 或 `MC_MMAP_ARENA_POOL_SIZE`（`mmap_buffer.cpp:23-26`、`src/config/mmap_arena_config.cpp:24-48`）；`MC_DISABLE_MMAP_ARENA` 可强制关闭。默认池大小 8GiB（`mmap_buffer.cpp:25-26`）。

### 6.8 hugepage："静默降级"还是"硬失败"

由 `allow_regular_page_fallback` 决定，而这个参数来自 `MC_STORE_USE_HUGEPAGE`：

- 未设置 → 尝试大页，失败则普通页重试并 `LOG(WARNING)`（`mmap_arena.cpp:122-138`）；
- 显式设置 → 失败直接返回 false（`:111-120`），日志明确写 "regular-page fallback disabled because MC_STORE_USE_HUGEPAGE explicitly requested hugepages"。

`MmapArenaConfig::FromEnvironment` 里的注释（`src/config/mmap_arena_config.cpp:51-56`）把这条契约写成了两条状态机，改代码时不要破坏它。

### 6.9 `RegisteredPinnedMemoryManager` 的锁外注册与"故意泄漏"

`try_pin`（`registered_pinned_memory.cpp:90-170`）的并发设计：

1. 先在锁内做**重叠检测**（`start < region_end && end > region_start`，`:114-122`）与**配额扣减**（`size > limit || pinned > limit - size`，`:124-130`），并把 `{addr, size, nullptr}` 压进 `regions_`，`pinned_bytes_ += size`；
2. **释放锁之后**才调 `cudaHostRegister`（`:141-143`）——因为这是慢系统调用，不能持锁；
3. 失败则重新拿锁 `remove_inactive_region_locked` 回滚（`:144-153`）；
4. 成功则再拿锁把 `entry.region` 指向自己（`:155-165`）。

中间的 `region == nullptr` 是一个明确的"占位中"状态：其它线程的重叠检测会把这段当成已占用（这正是想要的，防止两个人同时 pin 同一段），但 `remove_inactive_region_locked`（`:211-220`）只清理 `region == nullptr` 的条目。

`release`（`:172-209`）的行为更微妙：

- 先把自己那条的 `region` 置空并记 `should_unregister`，**释放锁**后调 `cudaHostUnregister`；
- 若返回 `kRuntimeUnloading`（`cudaErrorCudartUnloading`，`registered_pinned_memory.cpp:36-39`），只 `LOG(WARNING)` 然后**当作成功**继续走清理（`:192-204`）——因为此时 CUDA runtime 正在卸载，调任何 API 都可能崩；
- 若返回 `kError`，`LOG(ERROR)` 并 **`return false`**，条目保留（`region` 已是 nullptr，`pinned_bytes_` 不减）。调用方 `ReleasePinnedRegionForFree`（`real_client.cpp:166-177`）看到 false 就打 `"Leaking ... backing memory because cudaHostUnregister failed"` 并**放弃释放内存**。

也就是说退化路径是"泄漏一段内存 > 在 runtime 卸载期崩溃"。段地址区间也会继续保持"被 pin 占用"的记账，防止后续重复注册同一段。

配置项是 `MC_STORE_PIN_MEMORY_MAX_BYTES`（`src/config/registered_pinned_memory_config.cpp:16-32`），默认 0 = 关闭（`src/config/registered_pinned_memory_config.h:8`）。`limit_bytes_ == 0` 时 `try_pin` 直接返回 nullptr（`:92`），所以这是一个显式 opt-in 的功能。

### 6.10 CXL 的三个特殊之处

1. **分配器类型硬编码 CACHELIB**，与配置无关（`segment.cpp:211-236`、`region_driver.cpp:303-320`）。
2. **`base` 固定为 `DEFAULT_CXL_BASE`**（`segment.cpp:1556`），所以 `AllocatedBuffer::change_to_cxl` 能把指针转成相对偏移（`allocator.cpp:118-123`），`get_vaddr_from_cxl` 再转回来。这也意味着 CXL buffer 的 `data()` **不能被直接解引用**。
3. **CXL 段共享一个全局 allocator**：`cxl_global_allocator_` 由 `SegmentManager` 独占持有，可被多条 segment 记录引用，所以 `AttachMountedUsageTrackers` 要去重（`segment.cpp:142-158`），`GetMemoryUsageSnapshot` 也要用 `counted_allocators` 去重（`:126-141`）。

`CxlRegionDriver::PrepareAdopt` 直接返回 `UNAVAILABLE_IN_CURRENT_MODE`（`region_driver.cpp:296-302`），即 CXL 段不支持快照恢复（`PrepareOpen` 也拒绝非空 `live_allocations`，`:282-285`）。

### 6.11 `CreateBufferAllocator` 并非"无异常"

`allocator.cpp:158-183` 返回 `tl::expected`，注释（`:174-175`）说 "Offset construction has no parameter-dependent throwing path; metadata allocation failures intentionally follow fail-fast"。但 `OffsetBufferAllocator` 的构造函数在 `OffsetAllocator::create` 失败时会 `throw std::runtime_error`（`:383-386`），而 `CreateBufferAllocator` 并没有 `try/catch`。也就是说：**元数据分配失败会以异常形式穿过这个 `tl::expected` 接口**。这是刻意的 fail-fast 策略（注释里写了"metadata allocation failures intentionally follow fail-fast"），但对调用方来说，"返回 expected 的函数不会抛"这个直觉在这里不成立，写错误处理时要知道。

CacheLib 侧同理：`CachelibBufferAllocator::Create` 只把布局非法转成 `INVALID_PARAMS`，构造过程中的 `bad_alloc` 不会被捕获（`:150-155` 的注释明确说"Do not catch allocation failures here"）。

### 6.12 名字带 allocator 但不是内存分配器

- `mooncake-store/src/memory_alloc.cpp`：全文 19 行，`hugepage_memory_alloc` 在非 `USE_NOF` 构建下直接 `return nullptr`（`:7-13`）。它的存在意义是被 `store_py.cpp` 以**函数地址**形式导出给 Python（SGLang 的 NoF/大页自定义内存池钩子），不是通用分配入口。
- `mooncake-store` 的 `allocator_metric.cpp` / `allocator_status.h`：进程 RSS + jemalloc，见 §3.12。
- `mooncake-store-rs/.../client/runtime_alloc.rs`：解析副本写目标，不是分配器。
- `mooncake-store/include/placement/replica_allocator.h`：副本放置需求。
- `mooncake-transfer-engine/tent/src/platform/*_allocator.cpp`：TENT 新传输引擎自己的 CPU/CUDA/ROCm/Ascend/Sunrise 分配体系，与 store 段分配器不是一条线。
- `SimpleAllocator`（`allocator.h:460-476`，实现 `allocator.cpp:653-765`）：唯一一个**分配真实内存**的 CacheLib 用户——它自己 `aligned_alloc` 出 `base_`，再在上面建 CacheLib allocator。仅测试代码使用（`tests/batch_remove_test.cpp:99`、`tests/client_integration_test.cpp:258`），不是生产路径。
- `DummyBufferAllocator`（`allocator.h:234-259`）：`allocate` 返回 `nullptr`，只为在 standby 提升时保留 endpoint 元数据（`master_service.cpp:3572`、`:3864`、`:4040`）。

## 精读清单

按这个顺序读，每一步都标注"该看懂什么"：

1. **`mooncake-store/include/allocator.h` 全文（480 行）**
   重点 `AllocatedBuffer`（:70-177）与 `BufferAllocatorBase`（:183-227）。
   *看懂*：`weak_ptr` 回指、`SegmentLifetime`/`ClientLivenessRecord` 双绑定、`size()` 是请求字节、`Descriptor` 四字段。
2. **`mooncake-store/src/allocator.cpp:1-130`**
   `IsValidCachelibLayout`、`AllocatedBuffer` 的构造/析构/descriptor/cxl 转换。
   *看懂*：CACHELIB 的 16MB 对齐硬约束从哪来、CXL 指针为何是偏移。
3. **`mooncake-store/src/offset_allocator.cpp:56-130`（SmallFloat）+ `:191-357`（allocate/free）**
   *看懂*：三个转换函数的方向（roundUp/roundDown/floatToUint）、两级位图查找、`roundupSize` 的动机、free 的双向合并、`m_freeOffset` 栈语义。
4. **`mooncake-store/src/offset_allocator.cpp:538-545` + `:853-935`**
   `calculateMultiplier` 与 `OffsetAllocator::allocate`。
   *看懂*：quantum 换算、无锁快速失败 + 失败后收紧、unlock 后再构造 handle。
5. **`mooncake-store/src/offset_allocator.cpp:1001-1051`**
   `refreshLargestFreeRegion` + `freeAllocation`。
   *看懂*：提示值的三条更新规则、饱和减法。
6. **`mooncake-store/src/allocator.cpp:352-565`**
   `OffsetBufferAllocator` 的构造/分配/释放/`Restore`/`CaptureSnapshot`。
   *看懂*：`init_capacity/max_capacity` 的推导、`Restore` 的记账顺序、`size()` 与真实占用的差。
7. **`mooncake-store/src/offset_allocator.cpp:547-818`（快照校验）**
   六个 `Validate*`。
   *看懂*：为什么 `multiplier_bits` 必须重算校验、为什么 `allocated_size` 只能区间校验、邻居链如何证明"节点铺满整个布局"。
8. **`mooncake-store/src/allocator.cpp:567-651`（重放式恢复）**
   *看懂*：`fill_gap` 的二分、"任何时刻只有一个空闲节点"的不变量、`gaps.clear()` 为什么能让布局逐字节还原。
9. **对照读 `mooncake-store/src/allocator.cpp:310-349` 与 `src/cachelib_memory_allocator/MemoryAllocator.cpp:90-168`**
   *看懂*：chunk 位图恢复 vs 地址重放恢复，为什么后者对地址对齐要求更严。
10. **`mooncake-store/src/segment.cpp:142-158, 171-270, 1546-1568`**
    挂载路径、CXL 特例、tracker 补挂。
    *看懂*：一个 segment 从"客户端上报"到"可被策略选中"的全过程。
11. **`mooncake-store/include/allocation_strategy.h:33-120, 423-452, 573-618`**
    *看懂*：`AllocatorManager` 无锁、`registration->Allocate` 是唯一入口、`FreeRatioFirst` 的水位依赖 `size()`。
12. **`mooncake-store/src/segment_allocator_registration.cpp` 全文（84 行）**
    *看懂*：两条 lifetime 的区别、分配后二次校验。
13. **`mooncake-store/src/mmap_arena.cpp:54-166` + `src/common/mmap_buffer.cpp:199-297`**
    *看懂*：hugepage 决策树、`MAP_POPULATE`/`MADV_DONTFORK` 的理由、release/acquire 发布顺序、arena no-op free 的识别。
14. **`mooncake-store/src/common/client_buffer_allocation.cpp` 全文**
    *看懂*：protocol 分发表，以及 free 必须镜像 allocate。
15. **`mooncake-store/src/registered_pinned_memory.cpp:90-220`**
    *看懂*：锁外注册、占位状态、`kRuntimeUnloading` 的泄漏选择。
16. **`mooncake-store/src/allocator_metric.cpp:173-283` + `include/allocator_status.h:94-222`**
    *看懂*：1s 限流的动机、函数指针注入、cumulative→counter。
17. **`mooncake-transfer-engine/nvlink-allocator/nvlink_allocator.cpp:7-158`**
    *看懂*：fabric 探测（4KB 试分配）、`cuMemCreateTryFabric` 的降级、`cuMemSetAccess` 全设备授权、释放顺序。
18. **最后跑一遍 `mooncake-store/tests/buffer_allocator_test.cpp`**
    `OffsetLargestFreeRegionRemainsExact`（:122-143）、`ImportOffsetAllocationsAtOriginalAddresses`（:145-183）、`ImportOffsetAllocationsHasNoArbitraryGapLimit`（:244-260）、`CachelibCreateRejectsInvalidMemoryLayout`（:264+）。
    *看懂*：这些测试实际上把本章流程 B / 流程 C 的关键不变量都钉死了，是最快的交叉验证。

---

**未完全确认的两处**：

- `__Allocator::Node` 的 `sizeof` 未在代码中显式断言。按字段（6 个 `uint32` + 1 个 `bool`，4 字节对齐）推算约 28 字节；`m_nodes.reserve(m_max_capacity)` 的虚拟地址预留量约为 `28 × max_capacity`，但这是推算而非实测（代码里既没有 `static_assert`，也没有 `#pragma pack`），不要当作精确值引用。
- `AttachUsageTracker` 的"只允许在发布前调用一次"是**约定**而非强制：`BufferAllocatorBase::AttachUsageTracker` 只是在 `usage_registration_` 非空时静默返回（`allocator.cpp:35-42`），不会断言也不会替换 tracker。若有人对同一个 allocator 先后挂两个不同 tracker，第二次会被无声忽略。旧路径（`SegmentManager::AttachMountedUsageTrackers`）与新路径（`pool_mount.cpp` 的 `PrepareWithLiveAllocations`/`PublishMount`）都遵守这个约定，但契约本身靠调用方自觉。
