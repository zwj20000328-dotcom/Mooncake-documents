```mermaid
flowchart TB
     RC["RealClient"]

    subgraph CP["Client"]
        direction TB
        CL["Client<br/>副本选择"]
        TE["Transfer Engine<br/>MEMORY / NOF 副本"]
        FS["FileStorage<br/>装配期注入 backend"]
        DSB["DistributedStorageBackend"]
        KOA["KvcsObjectStorageAdapter"]
        KLD["KvcsLowLevelDriver<br/>KVCS SDK"]

        CL --> TE
        CL -->|DFS 副本| DSB
        FS -.->|创建并持有| DSB
        FS -.->|SetDfsStorageBackend| CL
        DSB -->|PutBatch / GetBatch<br/>DeleteBatch| KOA
        KOA -->|BatchPut / BatchGet<br/>BatchDelete| KLD
    end

    subgraph MP["Master"]
        direction TB
        MS["MasterService"]
        KCP["KvcsCapacityProvider"]
        KLD2["KvcsLowLevelDriver<br/>独立 SDK 实例"]

        MS -->|GetCapacity<br/>DeleteObject| KCP
        KCP -->|GetMountpointStats<br/>BatchDelete| KLD2
    end

    STORE["KVCS Store"]

    RC --> CL
    CL <-->|PutStart / PutEnd<br/>GetReplicaList / BatchRemove| MS
    KLD -->|kvcs_ll_* C ABI<br/>SDK 边界，仅 Driver 内可见| STORE
    KLD2 -->|kvcs_ll_* C ABI| STORE

    classDef kvcs fill:#e8f0fe,stroke:#1a73e8,stroke-width:2px;
    class KOA,KLD,KLD2,KCP,STORE kvcs;
```

## 每条边对应的抽象层

| 边 | 契约 | 定义位置 |
| --- | --- | --- |
| `Client → DistributedStorageBackend` | DFS 数据面（`DfsWriteRequest` / `DfsReadRequest`） | [distributed_storage_backend.h:83-90](../Mooncake/Mooncake/mooncake-store/include/storage/distributed/distributed_storage_backend.h) |
| `DistributedStorageBackend → KvcsObjectStorageAdapter` | `ObjectStorageAdapter`：逻辑 key、整对象、逐请求结果 | [object_storage_adapter.h:44-149](../Mooncake/Mooncake/mooncake-store/include/storage/distributed/object_storage_adapter.h) |
| `KvcsObjectStorageAdapter → KvcsLowLevelDriver` | `KvcsDriver`：provider key、批量、deadline、mountpoint_index | [kvcs_driver.h:62-78](../Mooncake/Mooncake/mooncake-store/include/storage/distributed/kvcs/kvcs_driver.h) |
| `KvcsLowLevelDriver → KVCS Store` | `kvcs_ll_*` C ABI，**只有 Driver 这一层能看见** | [kvcs_capi_driver.cpp:250](../Mooncake/Mooncake/mooncake-store/src/storage/distributed/kvcs/kvcs_capi_driver.cpp) / [297](../Mooncake/Mooncake/mooncake-store/src/storage/distributed/kvcs/kvcs_capi_driver.cpp) / [349](../Mooncake/Mooncake/mooncake-store/src/storage/distributed/kvcs/kvcs_capi_driver.cpp) |
| `MasterService → KvcsCapacityProvider` | 控制面专用：容量 + 单 key 删除 | [kvcs_capacity_provider.h:23-24](../Mooncake/Mooncake/mooncake-store/include/storage/distributed/kvcs/kvcs_capacity_provider.h) |
| `KvcsCapacityProvider → KvcsLowLevelDriver` | 同一个 `KvcsDriver` 契约 | [kvcs_capacity_provider.cpp:26](../Mooncake/Mooncake/mooncake-store/src/storage/distributed/kvcs/kvcs_capacity_provider.cpp) / [55](../Mooncake/Mooncake/mooncake-store/src/storage/distributed/kvcs/kvcs_capacity_provider.cpp) |
