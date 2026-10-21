# 13 Python 绑定与打包

## 定位与职责

这一层解决两个彼此耦合的问题：

1. **把 C++ 内核变成 Python 可调用的模块**。Mooncake 的数据面（RDMA/TCP 传输、张量序列化、零拷贝缓冲区）全部在 C++，pybind11 扩展负责把这些能力暴露成 `mooncake.engine`（TransferEngine）、`mooncake._store`（Store）、`mooncake._conductor`（Conductor 客户端），以及 `mooncake._ep` / `mooncake._fast_copy` 等辅助模块。
2. **把一堆 `.so`、原生可执行文件和 Python 源码打包成一个可 `pip install` 的 wheel**。仓库里有三棵互不自洽的源码树（`python/mooncake/`、`mooncake-integration/`、`mooncake-wheel/mooncake/`），由两套构建后端（scikit-build-core 与 setuptools）在构建期拼装成同一个 `mooncake` 包。

它在整体架构中的位置：向下依赖 `mooncake-store`（`PyClient`/`RealClient`/`DummyClient`）、`mooncake-transfer-engine`（`TransferEngine`、`Transport`、`SharedSegment`）、`mooncake-conductor`；向上被 vLLM/SGLang connector、REST 服务、CLI 以及 `structured_object_store` 使用。

一个必须记住的事实：**对外只有三个公开入口，其余都是私有扩展**。

| Python 侧导入 | 扩展模块名 | 定义位置 |
|---|---|---|
| `mooncake.engine` | `engine`（无下划线，直接公开） | `mooncake-integration/transfer_engine/transfer_engine_py.cpp:1743` |
| `mooncake.store.MooncakeDistributedStore` | `_store`（`OUTPUT_NAME "_store"`） | `mooncake-integration/store/store_py.cpp:2137`、`mooncake-integration/CMakeLists.txt:157-158` |
| `mooncake._conductor.ConductorClient` | `_conductor` | `mooncake-integration/conductor/conductor_py.cpp:286` |
| `mooncake._fast_copy` | C 扩展 | `python/mooncake/_fast_copy.c:192` |

`_store` 和 `_conductor` 刻意私有：用户应通过 `mooncake.store` 门面访问 Store，Store 门面才能在后端（cpp/rs）之间切换（`python/mooncake/store/__init__.py:95-138`）；Conductor 则由 `python/mooncake/conductor.py` 包一层（`from mooncake._conductor import ...`），而不是走 Store 门面。

打包有两条并行路径，分清它们才能读懂 CMake 里的分叉：

- **legacy（当前发布路径）**：`scripts/build_wheel.sh` + `mooncake-wheel/pyproject.toml`（setuptools）。`.github/workflows/_build-wheel.yaml:219` 调用的正是这个脚本；`mooncake-wheel` 的版本是 `0.3.13`（`mooncake-wheel/pyproject.toml:26`）。
- **unified（目标路径）**：仓库根 `pyproject.toml` 用 `scikit_build_core.build`，版本 `0.3.12.post1`（`pyproject.toml:11,15`），CMake 侧用 `if(SKBUILD)` 分支（`mooncake-integration/CMakeLists.txt:3-19`、`CMakeLists.txt:7-9`）。SKBUILD 路径的冒烟测试靠 `MOONCAKE_TEST_WHEEL` 环境变量（`python/tests/packaging/test_installed_wheel.py:26-39`：变量名在参数化列表里，未设置时 `pytest.skip`），仓库里没有 workflow 设置该变量——也就是说 CI 上它默认 skip。

## 目录结构速览

| 路径 | 职责 |
|---|---|
| `mooncake-integration/CMakeLists.txt` | 绑定层安装总入口：定义 `engine` / `store` / `_conductor` 三个 pybind11 模块，按 `WITH_TE` / `WITH_STORE` / `WITH_CONDUCTOR` / `USE_CUDA` / `USE_UBSHMEM` / `SKBUILD` 决定产物 |
| `mooncake-integration/store/store_py.cpp` | 3541 行，Store 绑定主体：`MooncakeStorePyWrapper`（对外名 `MooncakeDistributedStore`）、张量编解码、GIL 管理、`PYBIND11_MODULE(_store)` |
| `mooncake-integration/store/store_py_internal.h` | 被 `store_py.cpp:494` 在匿名 namespace 内 include 的“续页”：并行度规格、`WriterShardManifest`、`TensorIntoPlan`、分片计划构造函数 |
| `mooncake-integration/store/store_py_parallel_write.h` | 被 include 进 `MooncakeStorePyWrapper` **类体内部**（`store_py.cpp:1674`）：`batch_write_tensor_impl`、CUDA IPC 写路径 |
| `mooncake-integration/store/store_py_parallel_read.h` | 同类体内部 include（`store_py.cpp:1729`）：元数据前缀读、分片重建、`execute_tensor_into_plan_transfers` |
| `mooncake-integration/store/buffer_pool.cpp` | `BufferPool` / `BufferLease` / `BufferLeaseView` 绑定；阻塞式 acquire、溢出错配、`register_buffer` 回调的 GIL 处理 |
| `mooncake-integration/store/engram_store_py.cpp` | `EngramStore` / `EngramStoreConfig`，通过 `_get_pyclient_capsule()` 复用同一个 `PyClient` |
| `mooncake-integration/integration_utils.h` | 跨模块张量线格式：`TensorDtype`、`array_creators`、`TensorMetadata`、`Build/Validate/ParseTensorMetadata` |
| `mooncake-integration/transfer_engine/transfer_engine_py.cpp` | 2032 行，`PYBIND11_MODULE(engine)`；协议→分配器选择、buddy slab 分配、同步/异步/批量/页聚集传输 |
| `mooncake-integration/transfer_engine/transfer_engine_py.h` | `TransferEnginePy` 声明：`handle_map_`、`free_list_`、slab 尺寸表 |
| `mooncake-integration/transfer_engine/shared_segment_py.cpp` | `SharedSegment` 两阶段 create/complete 绑定 |
| `mooncake-integration/shared_segment.py` | `SharedSegment` 的 Python 侧：布局计算、all-gather blob、裸地址包成 torch 视图 |
| `mooncake-integration/allocator.py` / `allocator_ascend_npu.py` | `NVLinkAllocator` / `BarexAllocator` / `UBShmemAllocator`：注册为 torch 可插拔分配器 |
| `mooncake-integration/fabric_allocator_utils.py` | 定位包内 `.so`（`get_mooncake_so_path`）与 ctypes 探测（`probe_allocator_backend`） |
| `mooncake-integration/conductor/conductor_py.cpp` | `ConductorClient` 绑定，`tl::expected`→`{"ret": int, ...}` dict |
| `mooncake-integration/tests/*.py` | 只有两个 TENT 测试，直接 `import engine`，靠 `PYTHONPATH` 指向构建目录（`CMakeLists.txt:88-108`） |
| `python/mooncake/` | 规范 Python 包源：门面、async 包装、CLI、`_fast_copy.c`、`structured_object_store.py` |
| `python/mooncake/store/__init__.py` | 后端门面：`MOONCAKE_STORE_BACKEND` → cpp/rs，白名单懒转发 |
| `python/mooncake/store/rs/store.py` | Rust 后端兼容层：`_CACHE_COMPAT_POLICY` 降级策略 |
| `python/CMakeLists.txt` | 仅 SKBUILD 下加入：编译 `_fast_copy`，安装 `mooncake-wheel/mooncake/*.py` 与 reshard |
| `mooncake-wheel/pyproject.toml` + `setup.py` | legacy 打包元数据与 wheel tag / `_fast_copy` 扩展声明 |
| `mooncake-wheel/mooncake/` | 迁移未完成的模块暂存地（`_partial_read.py`、`dataproto_catalog.py`、connector、REST 服务、`pg.py`） |
| `scripts/build_wheel.sh` | 661 行 legacy 构建主流程：拷 `.so` → sed 改变体 → `python -m build` → auditwheel repair → 事后注入 EP/PG `.so` |

## 核心数据结构

### PyTensorInfo：一次张量操作的“句柄包”

```cpp
struct PyTensorInfo {          // store_py.cpp:42
    uintptr_t data_ptr;
    size_t tensor_size;
    TensorMetadata metadata;
    py::object owner;          // 保活
};
```

四个字段各有深意：

- `data_ptr` / `tensor_size` 是从 Python 对象上鸭子类型取出来的（`data_ptr()`、`numel()*element_size()`，`store_py.cpp:186-189`）。
- `metadata` 是线格式结构体。`BuildTensorMetadata` 先把它置 0，`build_full_tensor_metadata` 在返回前就填成 `tensor_size`（`store_py_internal.h:276`），`valid()` 再覆盖同一字段并做整体校验。
- `owner` 是最容易被忽略的字段：`extract_tensor_info_impl` 对输入张量会调用 `.contiguous()`（`store_py.cpp:183-185`），如果原张量非连续，这一步产生一个**新对象**；把新对象存进 `owner` 才能保证 C++ 后续做 RDMA 时那块内存不被 GC 回收。

`valid()`（`store_py.cpp:48-83`）不是简单的空指针检查，它把 `data_bytes` 覆盖成 `tensor_size` 后重新跑一遍 `ValidateTensorMetadata`，并要求 `data_offset + tensor_size` 与线格式自洽。

### 张量线格式（wire format）

`integration_utils.h:189-232` 定义了跨语言共享的内存布局，全部是 POD：

```cpp
struct TensorObjectHeader {   // 40 字节（4+2+2+4+4+4+4+8+8，自然对齐）
    uint32_t magic;           // 0x4d4f4f4e ('MOON')
    uint16_t version;         // 1
    uint16_t header_size;     // sizeof(TensorMetadata)
    int32_t  dtype;
    int32_t  ndim;
    uint32_t layout_kind;     // FULL=0 / SHARD=1
    uint32_t reserved_flags;
    uint64_t data_offset;
    uint64_t data_bytes;
};
struct TensorLayoutMetadata { // global_shape + local_shape + axis_count + axes[4]
    TensorShape global_shape; // int64_t dims[8]
    TensorShape local_shape;
    uint32_t axis_count;
    uint32_t reserved0;
    LayoutAxis axes[kMaxLayoutAxes];
};
struct TensorMetadata { TensorObjectHeader header; TensorLayoutMetadata layout; };
```

关键常量：`kTensorObjectMagic=0x4d4f4f4e`、`kTensorObjectVersion=1`、`kMaxTensorDims=8`、`kMaxLayoutAxes=4`（`integration_utils.h:171-174`）。按自然对齐计算 `sizeof(TensorMetadata) = 304` 字节（header 40 + layout 264），这个数值就是后文“只读元数据前缀”每次 ranged read 的长度。

`ValidateTensorMetadata`（`integration_utils.h:308-381`）的校验顺序值得逐条读，因为它同时是“跨版本兼容契约”：

1. magic / version / `header_size == sizeof(TensorMetadata)` 必须精确匹配；
2. `dtype` 必须在 `[0, NR_DTYPES)`，`ndim` 在 `[0, 8]`，`axis_count <= 4`，`layout_kind <= SHARD`；
3. `data_offset >= sizeof(TensorMetadata)` 且 `<= total_length`；
4. **`data_bytes` 必须精确等于 `total_length - data_offset`**；
5. 每个维度的 global/local shape 非负，且 `data_bytes` 必须等于 `TensorMetadataExpectedDataBytes()` 算出来的 `numel * element_size`（`integration_utils.h:280-306`）；
6. `ndim` 之后的 shape 槽位必须保持 `-1`（`MakeTensorShape` 初始化时填 `-1`，`integration_utils.h:234-241`）；
7. 每个 axis 的 `kind`、`shard_count>0`、`0<=shard_rank<shard_count`、`-1<=split_dim<ndim`。

这套校验意味着：**在枚举中间插入新 dtype 会破坏所有已存对象**。`TensorDtype` 的值（`FLOAT32=0 … NR_DTYPES=15`）已经写进 header 并被严格比对（`integration_utils.h:25-43`）。

`ParsedTensorMetadata`（`integration_utils.h:228-232`）把 header 里两个 `uint64_t` 转成 `size_t`，避免调用方反复做宽度转换。

### 分片 / 并行度元数据

`store_py_internal.h:17-55` 是一组**只存在于 C++ 侧、不进入线格式**的规格类型：

- `ParallelAxisSpec { kind, rank, size, split_dim, expert_id, stage_id }`（:17）——描述一条并行轴（DP/TP/EP/PP）。
- `TensorParallelismSpec { vector<ParallelAxisSpec> axes }`（:26）。
- `WriterShardManifest`（:41-55）：`magic=0x574d414e`（`kWriterShardManifestMagic`，:170）、`version=1`、`dtype`、`ndim`、`split_dim`、`shard_count`，加一个 `global_shape`。设计上它作为**独立的 Store 对象**存在，读取端按 `get_parallelism_manifest_key_name(base_key)` 取（:177），使“读回完整张量”不必先知道所有 shard 的 key 命名规则。**但要注意本版本的写入端并未接通**：`put_manifest_impl` / `upsert_manifest_impl`（`store_py.cpp:1566-1584`）在仓库里没有任何调用点或绑定；读取侧的 `load_parallelism_manifest`（`store_py.cpp:1708`）同样没有调用者。也就是说这条“独立 manifest 对象”的路径在当前 C++ 绑定中既不会被写出、也不会被走到，阅读时应把它当作预留设计而不是现状。

`TensorIntoPlan`（`store_py_internal.h:101-110`）是“把分片重新拼进用户缓冲区”的执行计划：`fragments` 描述每一次 ranged read 的 `(read_key, dst_offset, src_offset, size)`，`registered_buffer_ptr` 是用户缓冲区**所属的已注册 region 的基址**（不是用户传进来的指针，见 `store_py.cpp:1683-1706`），`materialized_metadata` 用于零长度/纯元数据情形。

### TransferEnginePy

`transfer_engine_py.h:45-336`。数据成员（:324-335）：

```cpp
std::shared_ptr<TransferEngine> engine_;
Transport *xport_;                                  // 声明后从未被赋值/使用
std::mutex mutex_;
std::vector<std::stack<char *>> free_list_;         // 16 个 size class
std::vector<char *> buffer_list_;                   // 从 buddy 顶层申请的大块
std::unordered_set<char *> large_buffer_list_;      // 超过最大 class 的独立分配
std::unordered_map<std::string, Transport::SegmentHandle> handle_map_;
bool auto_discovery_;
uint64_t transfer_timeout_nsec_;
```

slab 表在头文件里（`transfer_engine_py.h:37-43`）：`kDefaultBufferCapacity = 2GiB`，16 个 class 从 8KB 到 256MB。这是**经典的 buddy 分配器**：

- `findClassId(size)`（`transfer_engine_py.cpp:527-532`）返回能容纳 `size` 的最小 class；`size > 256MB` 返回 -1。
- `doBuddyAllocate(class_id)`（:534-554）递归向上一级借块，把一个块劈成两个压回 `free_list_[class_id]`。
- 顶层 class 一次性 `allocateRawBuffer(2GiB)` 并切成 256MB 块。
- `allocateManagedBuffer` / `freeManagedBuffer`（:556-584）用 `mutex_` 保护所有 free list 操作。**只有走 buddy 的块**（`class_id >= 0`）是“压回 stack、不真正归还内存”；超过最大 class 的大块走 `large_buffer_list_`，释放时会 `unregisterLocalMemory` 并真正 `freeMemory`（:576-580）。

关键副作用：`allocateRawBuffer`（:512-525）在分配后立刻 `engine_->registerLocalMemory(buffer, capacity, kWildcardLocation)`——即 **RDMA 注册发生在第一次分配时**，失败就 `freeMemory` 并返回 nullptr。所以 `allocate_managed_buffer` 返回 0 既可能表示 OOM，也可能表示 MR 注册失败。

### ScatterTransferTicket

`transfer_engine_py.cpp:73-131`。它把异步的 `ScatterTransferOperation` 包成一个可查询、可 drain 的对象：

- `status()`/`drained()` 加锁读（:88-96）。
- `drain(timeout_ms)`（:98-123）：如果已 drain 直接返回缓存状态；否则 `operation_->waitFor(bounded_timeout)`；`IsClock()` → `COMPLETION_UNKNOWN` 且**不置 drained**（可以再试）；成功/失败才置 `drained_` 并 `operation_.reset()`。
- 绑定里 `submit_scatter_write/read` 带 `py::keep_alive<0,1>()`（:1877、1884），保证 ticket 活着时 engine 不被回收——这是 DMA 生命周期正确性的关键。

### BufferPool 的三层对象

`buffer_pool.cpp:78-196`：

- `BufferPoolNative`：持有 `store_obj_`（`py::object`，让 Python 侧 Store 包装器活下去）、`py_client_`（`shared_ptr<PyClient>`）、`regions_`、`in_use_`、容量统计、`condition_variable`。
- `BufferLeaseNative`：`shared_ptr<BufferPoolNative>` + `shared_ptr<BufferHandle>` + `std::atomic<size_t> exports_`。`release_lease`（:252-264）在有导出的 view 时**拒绝释放**并抛错。
- `BufferLeaseViewNative`：每个 view 构造时 `add_export()`、析构时 `release_export()`（:198-204），从而让 lease 的析构知道是否还有 memoryview 活着。

`Region { handle, size, overflow_registered }`（:150-154）——`overflow_registered` 决定释放时是否需要回调 Python 的 `unregister_buffer`。

### Python 侧的三个“策略对象”

- `_CPP_EXPORTS` / `_RS_EXPORTS`（`python/mooncake/store/__init__.py:9-93`）：两个白名单的差集就是 cpp/rs 后端的 API 兼容性缺口清单。rs 模式下访问 C++ 专有名字会抛带说明的 `AttributeError`（:131-137），而不是静默返回 None。
- `_CACHE_COMPAT_POLICY`（`python/mooncake/store/rs/store.py:41-63`）：把缓存面方法（put/get/register_buffer…）的异常映射成失败码/空值；控制面方法保持严格抛错。
- `_NativeStoreWorker`（`rs/store.py:177-226`）：一个把原生 Store 关进守护线程、用 `queue.Queue + Future` 串行化的执行器。**注意：本版本中它只是被定义，`self._worker` 在 `__init__` 里恒为 `None`（:276），`_invoke`（:343-346）因此直接调原生对象。** 读代码时不要把这段当成实际线程模型。

## 关键流程剖析

### 流程一：`put_tensor` 从 Python 对象到 RDMA

调用链（全部在 `store_py.cpp`）：

```
put_tensor(key, tensor)                          :1162
  └─ put_tensor_impl                              :1154
       ├─ extract_tensor_info(tensor, key)        :213 → :160
       │    ├─ 类名含 "Tensor" 判断               :165-168
       │    ├─ tensor.contiguous()（可能拷贝）    :183-185
       │    ├─ data_ptr/numel/element_size        :186-189
       │    ├─ get_tensor_dtype(dtype)            integration_utils.h:105
       │    └─ build_full_tensor_metadata         store_py_internal.h:270
       └─ put_tensor_info_impl                    :1545
            └─ batch_put_tensor_infos_impl        :1291
                 ├─ try_dummy_cuda_ipc_batch_put_tensor_impl  parallel_write.h:161
                 └─ batch_write_tensor_impl                   parallel_write.h:208
```

分步说明：

1. **鸭子类型识别**（`store_py.cpp:165-168`）：只检查 `__class__.__name__` 里是否包含 `"Tensor"`。任何名字含 `Tensor` 的对象都会被继续当作 torch 张量使用 `data_ptr()`/`numel()`/`element_size()`。torch 本身只在运行时通过 `py::module_::import("torch")` 引入（`integration_utils.h:23`），构建期无依赖。
2. **连续性策略不对称**：输入张量允许非连续，会被静默 `.contiguous()`（可能多一次 D2D/H2D 拷贝，新对象由 `info.owner` 保活）；目标张量（`extract_tensor_destination_info`）强制 `is_contiguous()`，否则报错（`store_py.cpp:175-181, 218-222`）。
3. **元数据构造**：`build_full_tensor_metadata` 把全局/局部 shape 都设成同一个张量形状，`layout_kind=FULL`、`data_offset=sizeof(TensorMetadata)`；`BuildTensorMetadata` 先置 `data_bytes=0`，`build_full_tensor_metadata` 紧接着把它填成 `tensor_size`（`store_py_internal.h:270-277`），`PyTensorInfo::valid` 再覆盖一次并做完整校验。
4. **选择写路径**：
   - `use_dummy_client_` 且张量在 CUDA 上：走 CUDA IPC（见下）。
   - 否则进入 `batch_write_tensor_impl`。RealClient 路径把每个张量拆成**两个 buffer**——`PyTensorInfo` 内的 `TensorMetadata` 和 payload 指针（`parallel_write.h:183-192`），调用 `batch_put_from_multi_buffers(..., true)`（`store_py.cpp:1308-1314`）。DummyClient 路径则先 `allocate_client_buffer(total_size)`，`memcpy` 元数据 + `CopyToHost` payload，再一次性 `batch_put_from`（`parallel_write.h:246-292`）。
5. **GIL**：整个 `batch_write_tensor_impl` 的执行体在 `py::gil_scoped_release` 内（`parallel_write.h:229`）。也就是说，只有参数提取和最终的 Python 对象构造持有 GIL，传输期间不持有。
6. **结果回填**：`apply_tensor_batch_results`（`parallel_write.h:194-206`）用 `original_indices` 把“跳过的非法项”映射回去，保证返回列表长度与输入 keys 一致。

CUDA IPC 分支（`parallel_write.h:17-159`）是全书最微妙的一段：

- 在**持有 GIL** 时提取每个 owner tensor 的 `get_device()` 与 `torch.cuda.current_stream(device).cuda_stream`（:30-50），异常一律 `PyErr_Clear()` 吞掉。
- 然后在**释放 GIL** 后逐个 `ExportCudaIpcBuffer(data_ptr, tensor_size)`（:63-67）。如果任一失败，整体返回 `nullopt`，回退到普通 staged 路径。
- 校验 stream 的 `device_id` 必须与 IPC handle 的 `device_id` 一致（:71-87），不一致时**所有**已导出项都置为 `INTERNAL_ERROR`（而不是只错一个）。
- 元数据单独 `allocate_client_buffer` 并 `memcpy`（:101-108），以 `CudaIpcShmBufferRef` 传给 DummyClient——因为 DummyClient 在另一个进程里，不能解引用调用方的指针。
- 写之前对每个**唯一** (device, stream) 调 `SynchronizeCudaStream`（:141-150），确保 torch 侧的生产者 kernel 已完成。

### 流程二：`batch_get_tensor_into_cuda` 的两阶段读

`store_py.cpp:954-1103`。这是“把张量直接读进用户 CUDA 张量”的主路径，也是理解快照（snapshot）语义的最佳入口。

RealClient（非 dummy）路径：

1. **查询快照**：一次 `batch_query(keys)` 装入 `PyClient::QueryResultCache snapshot`，键为 key（:972-984）。整个操作后续只用这一份快照。
2. **只读元数据前缀**：`batch_get_tensor_metadata_prefixes`（:875-943）分配 `keys.size() * sizeof(TensorMetadata)` 的 client buffer，构造 `all_src_offsets = {{0}}`、`all_sizes = {{sizeof(TensorMetadata)}}`，用 `get_into_ranges` 只把每个对象的**元数据前缀（`sizeof(TensorMetadata)`，当前 304 字节）**拉进 scratch，然后 `parse_tensor_metadata_from_prefix` 解析（`store_py_parallel_read.h:25-43`）。
   - 为什么值得这么麻烦：走 `get_buffer`/`batch_get_buffer` 的整对象路径会把整份 payload 一起取回，对大张量（尤其需要 host staging 时）等于多一次全量拷贝。
3. **目标校验**：对每个目标张量 `extract_tensor_destination_info`，再 `tensor_destination_matches_metadata`（`store_py.cpp:416-431`）比对 dtype、ndim、字节数与 shape。不匹配的项**不报错**，只是留在 `results[i] = INVALID_PARAMS` 初值上。
4. **只读 payload**：对通过校验的项构造 `all_dst_offsets={{0}}`、`all_src_offsets={{stored.data_offset}}`、`all_sizes={{stored.data_bytes}}`，再次 `get_into_ranges_from_snapshot`（:1031-1037）。`data_bytes==0` 的项直接 `results[i]=0` 跳过（:1016-1019）。
5. **结果校验**：循环先把每一项写成 `INTERNAL_ERROR`，然后只有在“结果为负（错误码原样透传）”或“结果为正且**恰好等于** `data_bytes`”时才覆盖；其它正值（短读/长读）都保留 `INTERNAL_ERROR`（:1038-1051）。注意这里并**不是**“`result >= 0` 即成功”。

DummyClient 路径（:1055-1102）：只调用一次 `batch_get_tensor_metadata_prefixes`（:1055，未传 snapshot），然后用 `ExportCudaIpcBuffer` 把**目标**张量导出成 IPC handle，构造 `CudaIpcReadRequest{key, destination, source_offset, size}`，交给 `dummy_client->batch_get_into_cuda_ipc()`。也就是说，读方向也依赖 CUDA IPC，只是导出的是本地目标缓冲区。

**为什么必须用同一个 snapshot**：代码注释写得很直接（:970-972）——如果在“读前缀”和“读 payload”之间重新 query，中间发生的 upsert 可能让两次读到不同版本，最终拼出一个不存在的数据。**但这份保护只覆盖非 dummy 分支**：DummyClient 分支在 :1055 取前缀时没有传 snapshot，随后才用前缀里的 `source_offset/size` 发起 IPC 读，因此 dummy 路径仍存在“两次读之间被 upsert 改写”的窗口，只是 payload 读取用的是第一次读到的偏移量。

### 流程三：零拷贝边界——裸指针 API 族

真正的零拷贝不是 `get_tensor()`。`get_tensor()` 走 `buffer_to_tensor`（`store_py.cpp:233-368`），它：

1. 从 `BufferHandle` `new char[total_length]`；
2. `CopyToHost` 整块拷到 host（:254-261）；
3. `ParseTensorMetadata`；
4. `array_creators[dtype](exported_data, data_offset, tensor_size, take_ownership)` 造 numpy 数组（`integration_utils.h:45-57`，带 capsule 析构）；
5. `torch.from_numpy(...)`，再对 bf16/fp16/fp8 做 `.view(...)` 重解释（`store_py.cpp:353-361`）。

numpy 没有 bf16/fp8 dtype，所以存储类型是 `uint16_t`/`uint8_t`（`integration_utils.h:73-76`），必须在 `from_numpy` 之后用 torch 的 view 改名。**顺序不能改**，先 view 再 from_numpy 会失败。

零拷贝 API 族接受 `(uintptr_t, size)`：

- 写：`put_from`、`put_from_with_metadata`、`batch_put_from`、`batch_put_from_multi_buffers`、`upsert_from`、`batch_upsert_from`。
- 读：`get_into`、`batch_get_into`、`get_into_ranges`、`batch_get_into_multi_buffers`、`batch_get_into_multi_buffer_ranges`。
- 张量对象版：`put_tensor_from` / `get_tensor_into` / `batch_*_from` / `batch_*_into`。

使用前提是**先 `register_buffer(ptr, size)`**（`store_py.cpp:3004-3014`），由 store 把这段内存注册进 Transfer Engine。绑定注释明确说“Register a memory buffer for direct access operations”，绑定层本身没有“必须先注册”的前置断言。在需要 MR 的传输（如 RDMA）上，提交时按注册表查不到源地址会返回 `AddressNotRegistered`/“Memory region not registered by any active device(s)”（`mooncake-transfer-engine/src/transport/rdma_transport/rdma_transport.cpp:1131-1139`）；而 TCP 等不需要注册的传输可能照常工作。所以“不注册就一定会内存损坏”并不准确，但依赖具体传输、且错误会在很晚的阶段才暴露，仍然应当先注册。

`put_tensor_from`（`store_py.cpp:1477-1495`）的关键设计在 `make_raw_tensor_object_parts`（:99-149）：

- 缓冲区布局是 `[metadata][payload]`，`metadata_size = header.data_offset`、`payload_size = header.data_bytes`。
- 元数据通过 `AcceleratorRegistry.RuntimeAccelerators().CopyToHost(...)` 读取（:107-115），所以即使 `buffer_ptr` 是设备地址也能解析。
- **但 `parts.metadata_ptr` 保留调用方原始地址**（:143-147）。注释解释了原因：DummyClient 用注册表校验外部缓冲区；如果把设备元数据指针替换成私有 host 拷贝，这个指针就会“看起来未注册”。
- `batch_put_tensor_from`（:1497-1543）把每个对象拆成 `{metadata_ptr, metadata_size}` 与 `{payload_ptr, payload_size}` 两个 buffer，调用 `batch_put_from_multi_buffers`。这才是真正的零拷贝写。

`store_py_parallel_read.h` 里还有更复杂的“分片重建”路径：把多个 `key_tp_<rank>` 的 payload 用 ranged read 拼进一个完整的 `[TensorMetadata][全量 payload]` 缓冲区，必要时先 `register_buffer` 自己分配的重建缓冲区（:486、:560）。注意它的入口是 `get_tensor_with_writer_shard_full`（:456）和 `get_tensor_with_tp_full`（:521），两者都定义在这个类体内但**没有注册进 `_store` 的 `PYBIND11_MODULE`**，在本版本中不可从 Python 直接调用；真正注册的 `get_tensor_with_tp_into` / `batch_get_tensor_with_tp_into`（`store_py.cpp:1105-1152`）只是把 key 换成 `_tp_<rank>` 后转调 `get_tensor_into` / `batch_get_tensor_into`，并**不**走分片重建。名字相近的 `get_tensor_with_parallelism_into` 只存在于 Rust 后端（`python/mooncake/store/rs/store.py:848`），不是这条 C++ 路径的调用者。细节见 `TensorIntoPlan`。

### 流程四：TransferEngine 的一次同步写

```
te = engine.TransferEngine()
te.initialize(hostname, meta, "rdma", devices)        transfer_engine_py.cpp:372
  └─ initializeExt                                     :381
       ├─ initMemoryAllocator(protocol)                :391（定义 :229）
       ├─ TransferEngine 构造 / EFA/CXI 拓扑发现       :400-426
       ├─ engine_->init(conn_string, ...)              :428-438
       └─ free_list_.resize(16)                        :506
buf = te.allocate_managed_buffer(n)                    :556
te.transfer_sync_write(peer, buf, peer_addr, n)        :613
  └─ transferSync(...)                                 :962
       ├─ handle_map_ 查/开 segment（持 mutex_）        :971-983
       ├─ resolveNicHint / parseTransportHint          :993、:1007
       ├─ allocateBatchID + submitTransfer             :995-1015
       └─ 轮询 getTransferStatus / 超时                 :1041、:1057-1069
```

要点：

1. **分配器是文件级静态函数指针**（`transfer_engine_py.cpp:63-65`）：

   ```cpp
   static void* (*allocateMemory)(size_t) = nullptr;
   static void (*freeMemory)(void*) = nullptr;
   static std::string g_protocol;
   ```

   `initMemoryAllocator`（:229-306）按**第一次** `initialize` 的 protocol 选一次：`nvlink`→MNNVL、`musa`、`hip`、`nvlink_intra`、其它→64KB 对齐 `posix_memalign`（:293-300）。第二次调用只打 `WARNING` 并返回 true（:230-234）。**同进程内混用 rdma 与 nvlink 不会得到各自的分配器**。另外注意 `musa` 分支在未编译 `USE_MUSA` 时只打 ERROR 而**没有 return false**（:249-260，其它分支都有），随后仍会执行 `g_protocol = protocol; return true;`——此时两个函数指针仍是 nullptr。只要后续 `engine_->init` 没有因为未知协议名而失败，`initializeExt` 就会返回 0，而第一次 `allocateManagedBuffer` 才以 "Memory allocator is not initialized" 失败（:513-516）。这是与其它分支明显不一致的行为，读到这里值得停下来确认意图（本环境未构建验证 protocol=musa 时 `engine_->init` 是否必然成功）。
2. **64KB 对齐不是随便选的**：注释说明 Ascend SVM 的 `_devmm_mem_remote_map` 需要 64KB 页对齐的 src VA，否则 `aclrtHostRegister` 返回 EINVAL（:289-292）。
3. **buddy 分配器在 `mutex_` 内**，但 `allocateRawBuffer` 内部会调 `engine_->registerLocalMemory`——也就是**持锁做 MR 注册**（:512-525 在 :557 的 lock_guard 内被调用）。这是潜在的锁竞争点。
4. **segment handle 缓存**：`handle_map_` 以 target_hostname 为键，首次 `openSegment` 后缓存。`batchTransferSync` 在提交失败且 `CheckSegmentStatus` 不 OK 时会 `closeSegment`、`removeSegmentDesc`、`erase`（:1146-1161），其中 `removeSegmentDesc` 被 `if (auto metadata = engine_->getMetadata())` 包住（:1158-1159）：注释说明 TENT 模式下 `getMetadata()` 返回 nullptr，只有 classic 路径需要清 metadata，避免解引用空指针（#3995 的 P0 stale-handle 修复）。
5. **重试策略**：`max_retry = engine_->numContexts() + 1`（:1104）。第一次提交保留调用方给的 `nic_hint`，之后所有 entry 的 `nic_hint` 置 -1（:1128-1132），让调度器自由选本地 RNIC。
6. **超时**：`transfer_timeout_nsec_` 默认 30s（:308-316），实际超时是 `transfer_timeout_nsec_ + total_length`，注释写着“1GiB per second”（:1189-1190）——即按字节数线性放宽。
7. **GIL**：`transferSync` 第一行就 `py::gil_scoped_release`（:969），所以轮询期间不持 GIL。`transferSyncWrite/Read` 自身不释放，靠转调 `transferSync` 完成。
8. **buffer-protocol 变体**：`batch_transfer_sync_*_arrays`（:912-936）把手写 Python list 换成 numpy/array 的缓冲协议，`vectorFromIntegerBuffer`（:729-768）做严格校验：一维、8 字节整数、连续、非负。页聚集版 `batchArraysFromPages`（:824-863）把 `[F,5]` 的 field 表与页 id 展开成 `F*P` 条描述符，再按 `max_batch_size` 分批提交（:887-910）。

### 流程五：BufferPool 的 acquire/release 与溢出错配

`buffer_pool.cpp:315-351`：

```
acquire(size, block, timeout)
  ├─ block 缺省取 block_on_exhaustion_；timeout 缺省取 default_timeout_
  ├─ deadline = steady_clock::now() + timeout（无 timeout 则 time_point::max）
  ├─ size_class > max_bytes_ → 抛错
  ├─ py::gil_scoped_release                     :333
  └─ while(true):
       raise_if_open_locked()
       try_acquire_locked(lock, size)            :390
       ├─ 容量检查 → reserve_locked
       ├─ lock.unlock()                          :397  ← 关键
       ├─ client_buffer_allocator_->allocate()
       │    失败 → allocate_overflow_unlocked()  :440
       │            ├─ posix_memalign
       │            ├─ gil_scoped_acquire        :449
       │            └─ store_obj_.register_buffer(ptr, size)
       ├─ lock.lock(); unreserve_locked()
       └─ closing 检查；regions_/in_use_ 记账
     make_lease_locked → in_use_.insert(ptr)
     失败则 wait_until/wait 后重试
```

三个设计要点：

1. **分配期间不持锁**（:397 主动 unlock）。分配可能触发 `register_buffer` 这种昂贵的、需要 GIL 的调用，若持锁会阻塞所有其他 lease。
2. **持锁时碰 Python 对象**：`allocate_overflow_unlocked` 在**没有** mutex 的情况下 `gil_scoped_acquire` 回调 `register_buffer`（:448-452）；`unregister_overflow_unlocked`（:461-468）同样。函数名里的 `unlocked` 就是这个意思。反过来，`release_internal`（:470-491）先用 `lock_guard` 完成记账，再在锁外做 `unregister_overflow_unlocked`——顺序是刻意安排的，不要在锁内加 Python 调用。
3. **`close()` 的两阶段**（:358-378）：先 `closing_=true` 并解锁，然后在**释放 GIL** 的状态下等待 `reserved_regions_ == 0`；若有活跃 lease，回滚 `closing_` 并抛错。`acquire` 循环顶部 `raise_if_open_locked()` 让等待者尽快退出。

`BufferLeaseNative` 的析构（:217-222）吞掉所有异常并 `release_lease(false)`（不检查导出 view）。这意味着**如果 Python 侧还持有 memoryview，析构会绕过检查直接释放**；而显式 `release()` 会抛错。这是一个有意的不对称：析构路径不能抛。

### 流程六：SharedSegment 的两阶段创建

C++ 侧只有两个动作（`shared_segment_py.cpp`）：

- `SegmentCreate`（:36-57）：构造 `SharedSegmentOptions`，`SharedSegment::Create(name, size, options, segment, blob)`，**释放 GIL**，返回 `(segment, blob)`。
- `SegmentComplete`（:59-67）：把各 rank 的 blob 交回 `segment->Complete(blobs)`，同样释放 GIL。

Python 侧（`shared_segment.py:54-150`）负责所有编排：

1. 校验 `host_register` 必须 `mmap=True`（:88-89），`supported()` 探测（:90-101）。
2. 按**名字排序**计算布局（:105-109），保证各 rank 的字节偏移一致，与 dict 顺序无关。
3. 把布局编码进 segment 名字：`f"{name}|{tag}"`（:113、:120-129）——C++ 的 fingerprint 校验因此能发现 peer 之间 count/shape/dtype 不一致，而不必传完整声明。
4. `create` 之后做**失败收敛**：`_raise_if_any_rank_failed`（:310-335）通过 `all_gather_object` 让所有 rank 以相同错误退出，避免一半成功一半挂起。
5. `_all_gather_blob`（:338-342）用 CPU 字节张量做 all-gather（要求 gloo group；vLLM `GroupCoordinator` 通过 `cpu_group` 暴露）。
6. `tensors(block)`（:173-208）把裸地址包成视图：有 `device_addr()` 时走 `torch_npu._C._construct_storage_from_data_pointer`（:352-375），否则用 `ctypes.c_int8.from_address` + `torch.frombuffer`（:378-384）。每个 view 都挂 `_mooncake_segment` 反向引用保活。

### 流程七：legacy wheel 构建

`scripts/build_wheel.sh` 的骨架：

1. **拷产物到 `mooncake-wheel/mooncake/`**（:30-144）：`engine.*.so`→`engine.so`、`_store.*.so`、`libasio.so`、`libtransfer_engine.so`（BUILD_SHARED_LIBS 时）、`libmooncake_store.so`、`nvlink_allocator.so`、`ubshmem_fabric_allocator.so`、`mooncake_master`/`mooncake_client`/`transfer_engine_bench`、`ascend_transport*.so`……
2. **暂存迁移中的 Python 模块**（:190-241）：`MIGRATED_PYTHON_MODULES` 列表里的模块从 `python/mooncake/` 拷进来，`store/` 整目录、`reshard/` 整目录 `cp -R`。退出时由 `trap cleanup_migrated_python_staging EXIT`（:232）删除，`__init__.py` 用 `mktemp` 备份后还原（:234-235）。
3. **按变体重写 pyproject**（:281-362）：8 个变体（non-cuda / cuda13 / npu / efa / efa-cuda13 / efa-non-cuda / musa / hip）用 `sed -i` 改 name、description、keywords、classifier、甚至 `requires-python`。备份到 `pyproject.toml.backup`，EXIT trap 还原（:253-258）。
4. **构建**：`python -m build --wheel`（NPU 用 `--no-isolation`，:466-472）。
5. **auditwheel repair**（:487-581）：超长 `--exclude` 列表，包括 glibc、CUDA、torch、Ascend 全系、`libmpcomm.so*` 等。
6. **事后注入**（:586-614）：CUDA EP/PG 的 `.so` 在 repair **之后** `wheel unpack`→`cp`→`wheel pack`，因为 patchelf 会破坏 CUDA fatbin（注释明确提到 `cudaErrorInvalidKernelImage`）。同时给 `_ep*.so` 补 `rpath=$ORIGIN`。
7. **NPU 特化**（:620-653）：把 auditwheel 的 `.libs/` 摊平进 `mooncake/`，删 `.pth`，对所有 ELF 强制 `--force-rpath --set-rpath '$ORIGIN'`，并逐个验证 RPATH。

## 与其他模块的边界

**向下（C++ 内核）**

- Store 绑定依赖 `mooncake-store` 的 `PyClient`（抽象）、`RealClient`、`DummyClient`、`BufferHandle`、`ReplicateConfig`、`ResourceTracker`（`store_py.cpp:13-20`）。
  - `init_real_client()`（:542-551）负责 `RealClient::create()` 并把结果注册进 `ResourceTracker::getInstance()`。
  - `setup_dummy`（:2532-2548）切到 `DummyClient`，并生成 IPC socket 名 `"@mooncake_client_<port>.sock"`。
  - `ResourceTracker` 是进程级单例，故意 `new` 出来泄漏以避免静态析构顺序问题，构造时 `std::atexit(exitHandler)`，并用 `dlsym(RTLD_DEFAULT, "Py_IsInitialized")` 判断是否嵌在 Python 里——在 Python 内不安装自己的信号线程，以免 SIGINT 被屏蔽（`mooncake-store/src/real_client.cpp:776-799`）。
- TransferEngine 绑定依赖 `transfer_engine.h`、`transport.h`、`rdma_transport.h`、`transfer_engine_c.h`（`transfer_engine_py.h:27-31`），并链接 `$<TARGET_OBJECTS:rpc_communicator>`（`mooncake-integration/CMakeLists.txt:72`）。
- Conductor 绑定只依赖 `conductor_cpp_core`（`CMakeLists.txt:176`）。

**向上（Python 集成层）**

- `mooncake.store` 门面是唯一稳定的 Store API 面；`async_store.py` 用 `__getattr__` 把任意 `async_xxx` 映射到 `xxx` 并 `run_in_executor`（`python/mooncake/async_store.py:8-39`）。这意味着**任何同步方法都自动获得异步版本**，但也意味着线程池默认大小（`None` → 默认 executor）会限制并发。
- `structured_object_store.py` 依赖 `mooncake._partial_read`（只存在于 `mooncake-wheel/mooncake/`，`structured_object_store.py:27`）和 `mooncake._fast_copy`（:34）。这是三棵树互相依赖的最直观例子。
- `_fast_copy.c` 提供 `concat_arrays_into(list[ndarray], dest_ptr, ...)`，内部 `Py_BEGIN_ALLOW_THREADS` + `pthread_create` 并行拷贝（`python/mooncake/_fast_copy.c:34,133,138,180`），被结构化对象序列化热路径使用。

**构建期边界**

- `engine` 与 `_store` 都链接 `transfer_engine`；`_store` 还链 `mooncake_store`、`cachelib_memory_allocator`（`CMakeLists.txt:72-86, 159-170`）。
- `asio_shared`（`libasio.so`）被安装进包目录（:283-286），`engine.so` 靠 `INSTALL_RPATH "$ORIGIN"`（:57）与 `CMAKE_BUILD_WITH_INSTALL_RPATH TRUE`（:47）找到它；`tent_metrics` 同理（:295-298，注释解释了共享 `TentMetrics` 单例必须可加载）。
- `WITH_EP` 时创建一个 `engine.so -> engine<EXT_SUFFIX>` 符号链接（:318-333），因为 `ep.so`/`pg.so` 以裸名 `engine.so` 链接。缺这个链接时 `import mooncake.engine` 会以“找不到库”的形式失败，报错不会指向真正原因。

## 设计取舍与易踩坑点

1. **三个 `.h` 不是头文件，是 `store_py.cpp` 的续页**。`store_py_internal.h`、`store_py_parallel_read.h`、`store_py_parallel_write.h` 都没有 `#pragma once`；后两个被 include 在 `MooncakeStorePyWrapper` 类体内部（`store_py.cpp:1674`、`1729`）。单独打开会以为是独立 TU，且里面的成员函数定义离开类上下文根本编译不了。任何“把头文件当独立单元读”的结论都不可靠。

2. **GIL 是手工管理的，不是 `py::call_guard`**。三种模式并存：
   - 整个执行体包一层 `py::gil_scoped_release`（`batch_write_tensor_impl`、`transferSync`）；
   - release 块内再 `gil_scoped_acquire` 回来构造 Python 对象（`get`/`get_batch`，`store_py.cpp:664-689`、`701-737`）；
   - 持 C++ mutex 时重新 acquire GIL 回调 Python（`buffer_pool.cpp:448-452`、`461-467`）。

   第二种写法依赖“acquire 先析构、release 后析构”的逆序规则，能工作但非常脆弱。`get()` 里还有一个类型不一致：函数声明返回 `pybind11::bytes`，CUDA 拷贝失败分支却 `return pybind11::none()`（`store_py.cpp:683`）。此处**未完全确认**具体异常行为（本地未构建验证），但从签名看它会在返回处走一次 `bytes` 转换而不是把 `None` 交给 Python——错误路径上很可能二次抛错。

3. **“零拷贝”只在裸指针 API 上成立**。`get_tensor`/`batch_get_tensor`/`load_tensor_from_safetensor` 都会完整 host 拷贝。判断标准：参数里有没有 `uintptr_t buffer_ptr, size`。而且裸指针 API 要求先 `register_buffer`；绑定层不强制，但需要 MR 的传输会对未注册的源地址返回 `AddressNotRegistered`（见上文流程三），所以错误通常表现为传输失败而非静默成功——但并非所有传输都要求注册。

4. **输入/目标张量的连续性策略不对称**：输入自动 `.contiguous()`，目标强制要求连续。写代码时如果从 `get_tensor_into_cuda` 换到 `get_tensor_into`，很容易因为一个转置视图而失败。

5. **dtype 枚举只能追加**。`TensorDtype` 的值进入 header 并被严格校验（`integration_utils.h:316-318`），`array_creators` 是 `std::array<...,15>` 且靠下标索引（:61-77）。在中间插入会同时破坏 `ValidateTensorMetadata`、`array_creators`、以及已落盘对象的兼容性。

6. **bf16/fp16/fp8 的 numpy 绕行顺序固定**：先 `create_typed_array<uint16_t/uint8_t>`，再 `torch.from_numpy`，最后 `.view(bfloat16/...)`（`store_py.cpp:339-361`）。numpy 不认识这些 dtype。

7. **张量识别靠类名子串**，不是 `isinstance(torch.Tensor)`。任何 `__class__.__name__` 含 `"Tensor"` 的对象都会走鸭子类型；反之自定义 `nn.Parameter`（名字不含 Tensor）会被拒绝，除非其 `__class__.__name__` 恰好匹配。这是零构建期依赖 torch 的代价。

8. **静态分配器是隐式全局状态**。`allocateMemory`/`freeMemory` 是文件级指针（`transfer_engine_py.cpp:63-65`），一旦进程内第一个 `TransferEngine` 以 `rdma` 初始化，后续 `nvlink` 实例也用 64KB `posix_memalign`。多协议同进程场景必须把 `initialize` 顺序纳入设计。

9. **`file(GLOB SOURCES "*.cpp")` 与 `${CACHE_ALLOCATOR_SOURCES}` 都是空的**。前者匹配不到 `mooncake-integration/` 顶层的任何文件（该目录只有 `.py`/`.h` 和子目录），后者在整个仓库里从未被 `set` 过——只有两处使用（`mooncake-integration/CMakeLists.txt:54,152`）。读 CMake 时不要以为“还有别处源码”。

10. **模块命名不对称**：`engine` 直接公开，`store` 被 `set_target_properties(... OUTPUT_NAME "_store")` 改成私有（`CMakeLists.txt:157-158`）。Python 侧没有任何 `.pyi` 存根，IDE 补全全靠 `.so` 内省。`InnerTransferEngine` 只是把 C++ 的 `TransferEngine` 类型再注册一次、且没有任何方法绑定（`transfer_engine_py.cpp:2021-2022`），用于类型暴露/隔离，不是一个不同的 C++ 类型。

11. **两个 capsule 辅助函数不等价**。`make_pyclient_capsule` 生成**带名字** `"mooncake.PyClient.shared_ptr"` 的 capsule（`store_py.cpp:2358`），`_get_pyclient_capsule` 用它（:2431-2436）；而 `_get_pyclient_from_wrapper`（:2394-2415）生成的是**无名** capsule。`buffer_pool.cpp:41-46` 与 `engram_store_py.cpp:32-37` 都要求名字匹配，所以它们只能走前者。用错会得到 "unexpected PyClient capsule type"。

12. **`_NativeStoreWorker` 当前是死代码**。`rs/store.py:177-226` 定义完整、注释也描述了串行化模型，但 `self._worker` 在 `:276` 初始化为 `None` 后再无赋值，`_invoke`（:343-346）直接调原生对象。`setup` 里的 `worker_scope` 是传给 Rust 侧的参数，不是这个 Python 线程。**不要把注释里的模型当成现状**。

13. **cpp 与 rs 后端的错误语义不可互换**。rs 的 `_CACHE_COMPAT_POLICY` 会把 `put/get/register_buffer` 等缓存面异常降级成 `-1`/`b""`/`[b""]*n`（`rs/store.py:66-80`），cpp 后端直接抛/返回原始错误码。依赖“失败会抛异常”的代码切到 rs 后会静默拿到失败码。

14. **`__init__.py` 的归属取决于构建路径**。`mooncake-wheel/mooncake/__init__.py` 只有 `extend_path`（构建期占位）；`build_wheel.sh:236-239` 会用 `python/mooncake/__init__.py` 覆盖它并在退出时还原；`python/CMakeLists.txt:18-24` 安装 legacy 目录时显式 `PATTERN "__init__.py" EXCLUDE`。所以“这个文件到底是什么”必须先问走哪条路径。

15. **跨目录源码依赖是双向的**。`python/CMakeLists.txt:18-24` 从 `mooncake-wheel/mooncake/` 装 Python 文件；`mooncake-integration/CMakeLists.txt:245-246, 259, 315` 又从 `mooncake-wheel/mooncake/` 装 `mooncake_connector_v1.py`、`vllm_v1_proxy_server.py`、`mooncake_store_service.py`、`pg.py`。反过来 `structured_object_store.py` 依赖只存在于 wheel 树的 `_partial_read.py`。没有任何一棵树是自洽的包。

16. **发布矩阵靠 sed 维持，两个数组必须单行**。`mooncake-wheel/pyproject.toml:34-36` 的 `keywords` / `classifiers` 被 8 个变体的 `sed -i 's/^keywords = \[\(.*\)\]$/.../'` 原地改写（`build_wheel.sh:289, 299, 308, 320, 329, 338, 348, 360`——最后一个是 ROCm/HIP 变体）。格式化换行会**静默**让变体构建不再改包名/关键词，产物仍能构建但元数据错误。文件顶部注释（:7-11）就是在警告这一点。

17. **`libmpcomm.so` 故意不 vendor**（`build_wheel.sh:478-486`）：MPComm 独立升级，且它可能带 CUDA fatbin，patchelf 会破坏它。vendor 会有两个后果：RPATH 覆盖 `LD_LIBRARY_PATH` 导致新版本被旧副本遮蔽；以及 fatbin 损坏。

18. **wheel tag 来自构建机 glibc**。`build_wheel.sh:459` 用 `PLATFORM_TAG=${PLATFORM_TAG:-"manylinux_${GLIBC_VERSION}_${ARCH_SUFFIX}"}`，glibc 由 `detect_glibc_version` 探测（:416-436，最终 fallback `2_17`）。`setup.py:28-105` 的 `_detect_manylinux_tag` 是另一套实现。`mooncake-wheel/tests/test_release_wheel_tags.py` 之所以断言 CI 必须跑固定镜像，就是因为 aarch64 的 floor 曾在无代码变更时漂移。

19. **测试环境的 import 解析与真实安装不同**。`mooncake-wheel/tests/conftest.py:11-29` 手工往 `sys.path` 和已导入 `mooncake.__path__` 里塞 `python/` 与 `mooncake-wheel/mooncake/`。在这组测试里能 import 到的东西，装 wheel 后不一定能。`mooncake-integration/tests/*.py` 则直接 `import engine`（`test_tent_send_probe.py:3`），依赖 `PYTHONPATH=$<TARGET_FILE_DIR:engine>`（`CMakeLists.txt:94`），只在 `BUILD_UNIT_TESTS AND USE_TENT` 时注册（:88）。

20. **`BufferPool` 的析构与解释器关闭**。`BufferLeaseNative::~BufferLeaseNative` 吞异常调 `release_lease(false)`（`buffer_pool.cpp:217-222`），后者可能走到 `unregister_overflow_unlocked` → `py::gil_scoped_acquire`（:461-464）。如果发生在解释器 finalize 阶段，acquire GIL 的行为本身就是未定义的灰区。**此处未完全确认** CPython 3.10-3.13 各版本的表现，但设计上应避免把 `BufferLease` 留到进程退出。

21. **`close()` 的返回值语义**：`MooncakeStorePyWrapper::close`（`store_py.cpp:2694-2704`）无论 `tearDownAll` 成功与否都会 `reset` 内部指针，文档字符串写明“nonzero result is terminal, not retryable”。

22. **快照对象绑定到特定 Store**：`RangedReadSnapshotPy::belongs_to`（`store_py.cpp:506-508`）比较裸指针，`get_into_ranges_from_snapshot` 在不匹配时抛 `invalid_argument`（:3096-3099）。跨 Store 复用快照会直接失败，这是有意的。

## 精读清单

建议按以下顺序读，每步都有明确的“读懂标志”：

1. **`mooncake-integration/integration_utils.h` 全文（401 行）**。
   先读 `TensorDtype`（:25-43）与 `array_creators`（:61-77），再读 `TensorObjectHeader`/`TensorLayoutMetadata`/`TensorMetadata`（:189-232），最后逐条过 `ValidateTensorMetadata`（:308-381）。
   *读懂标志*：能说出“为什么 `data_bytes` 必须等于 `numel*element_size`”“为什么 `MakeTensorShape` 用 -1 填充剩余槽位”“新增 dtype 只能追加在哪里”。

2. **`store_py.cpp:1-540`**。
   `PyTensorInfo`（:42-84）→ `RawTensorObjectParts`/`make_raw_tensor_object_parts`（:92-149）→ `extract_tensor_info_impl`（:160-211）→ `buffer_to_tensor`（:233-368）→ `RangedReadSnapshotPy`（:498-530）→ 注意 :494 的 `#include "store_py_internal.h"`。
   *读懂标志*：能画出 `get_tensor()` 的一次完整拷贝路径，并指出 `owner` 在哪一行被赋值、为什么。

3. **`store_py_internal.h:1-140` + `:1345-1504`**。
   只看类型与三个关键函数：`calculate_shard_range`（:1345）、`parse_tensor_metadata_from_raw_buffer`（:1402）、`build_raw_tensor_shard_write_plan`（:1417）。
   *读懂标志*：能解释 `data_ranges` 里每个 `(src_offset, row_bytes)` 是怎么从 `(global_shape, split_dim, shard_start)` 算出来的。

4. **`store_py.cpp:533-1103`（类体前半）**。
   重点 `init_real_client`（:542）、`setup` 的两个重载（:2437 / :2484，可以跳读到绑定处）、`get`/`get_batch`（:656/:692）、`get_tensor_into`（:795）、`batch_get_tensor_metadata_prefixes`（:875）、`batch_get_tensor_into_cuda`（:954）。
   *读懂标志*：能说清“为什么元数据前缀要单独读”“为什么必须复用同一个 snapshot”。

5. **`store_py.cpp:1154-1672` + `store_py_parallel_write.h` 全文**。
   `put_tensor_impl` → `batch_put_tensor_infos_impl` → `try_dummy_cuda_ipc_batch_put_tensor_impl`（:17-159）→ `batch_write_tensor_impl`（:208-334）。
   *读懂标志*：能对比 RealClient（两个 buffer、零拷贝）与 DummyClient（staged、一次拷贝）的差异，并指出 GIL 在哪一行释放、在哪里重新获取。

6. **`store_py.cpp:2137-3541`（模块绑定）**。
   先扫 `PYBIND11_MODULE` 里注册的类型与函数名（:2137-3538），再回头看 `MooncakeDistributedStore` 的 `.def` 列表，把方法分成四类：缓冲/注册、裸字节、张量、控制面。
   *读懂标志*：拿到 Python 侧 TE/Store 的完整契约；知道 `BufferPool` 的别名（`RegisteredBufferPool` 等，`buffer_pool.cpp:586-588`）。

7. **`mooncake-integration/store/buffer_pool.cpp:315-534`**。
   `acquire`（:315）→ `try_acquire_locked`（:390）→ `allocate_overflow_unlocked`（:440）→ `release_internal`（:470）→ `has_capacity_for_locked`（:513）。
   *读懂标志*：能指出“持锁期间从不调用 Python”这一不变量在哪些行被打破（以及为什么那里要 `lock.unlock()`）。

8. **`mooncake-integration/transfer_engine/transfer_engine_py.cpp:63-306` 与 `:505-585`**。
   `initMemoryAllocator`（:229）→ `allocateRawBuffer`（:512）→ `findClassId`/`doBuddyAllocate`（:527/:534）→ `allocateManagedBuffer`/`freeManagedBuffer`（:556/:572）。
   *读懂标志*：能解释 8KB…256MB 的 class 表、2GiB 顶层块、以及“buddy 块 free 不归还内存（超过最大 class 的大块例外）”。

9. **`transfer_engine_py.cpp:962-1273` 与 `:1743-2032`**。
   `transferSync`（:962）→ `batchTransferSync`（:1075）→ `batchTransferAsync`（:1207）→ 模块绑定（:1743）。
   *读懂标志*：能说出重试次数、超时公式、失败时 segment 缓存如何清理、`batch_id_t` 为什么用 0 表示失败。

10. **`mooncake-integration/shared_segment_py.cpp` + `shared_segment.py`**。
    两阶段协议、布局排序、名字编码指纹、all-gather、视图构造。
    *读懂标志*：能解释“为什么虚拟地址不要求跨 rank 一致”以及 `host_register`/`mmap` 四种组合分别得到什么张量类型。

11. **`python/mooncake/store/__init__.py` 全文 + `python/mooncake/store/rs/store.py:1-80, 273-460`**。
    白名单差异、懒转发、rs 降级策略、`_invoke` 的真实调用路径。
    *读懂标志*：能列出 cpp 有而 rs 没有的 API，并知道 rs 模式下它们抛什么。

12. **打包三件套**：`mooncake-integration/CMakeLists.txt`（尤其 :1-19、:46-57、:148-179、:236-334）、`python/CMakeLists.txt`、`scripts/build_wheel.sh:180-280, 438-661`。
    最后看 `python/tests/packaging/test_project.py`——它把“哪棵树是唯一权威源”“哪些模块被迁移”“CMake 必须怎么写”固化成可执行断言，是打包不变量的最佳文档。
    *读懂标志*：拿到一张“某个 `.py`/`.so` 最终会出现在 wheel 的哪个路径、由哪条规则安装”的映射表。
