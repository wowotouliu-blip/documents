# Venus ICD 初始化流程与 Guest↔Host 能力协商分析

## 一、整体架构概览

Venus 是 Mesa 中实现的一套 **Vulkan over virtio-gpu** 机制。其核心思路是：

```
Guest(Linux)                          Host(QEMU)
┌─────────────────────────┐          ┌──────────────────────┐
│ Vulkan App              │          │                      │
│   ↓                     │          │  virglrenderer /     │
│ Mesa Venus ICD          │          │  rutabaga_gfx (Vulkan│
│ (vn_*.c)                │          │  renderer backend)   │
│   ↓                     │          │        ↑             │
│ Venus Protocol Encoder  │          │        │             │
│   ↓                     │          │        │             │
│ Command Stream (shmem)  │          │        │             │
│   ↓                     │          │        │             │
│ DRM ioctl (EXECBUFFER)  │──────────│──> virtio-gpu 设备   │
│ virtio-gpu 驱动         │ virtio   │        │             │
│                         │ queue    │  rutabaga_submit_    │
│ /dev/dri/renderD128    │          │  command()           │
└─────────────────────────┘          └──────────────────────┘
```

**关键组件：**
- **Mesa**: `src/virtio/vulkan/vn_*.c` — Venus ICD (Vulkan 驱动)
- **Linux 内核**: `drivers/gpu/drm/virtio/virtgpu_*.c` — virtio-gpu DRM 驱动
- **QEMU**: `hw/display/virtio-gpu*.c` — virtio-gpu 设备模型
- **Host 渲染器**: virglrenderer (Venus 后端) 或 rutabaga_gfx (FFI 到 host Vulkan)

---

## 二、初始化全流程

### 阶段1：Linux 内核 virtio-gpu 驱动初始化

文件: `linux-6.15/drivers/gpu/drm/virtio/virtgpu_kms.c:117` (`virtio_gpu_init()`)

```
virtio_gpu_init()
  ├─ 检查 VIRTIO_F_VERSION_1 feature bit
  ├─ 查询 virtio feature bits:
  │   ├─ VIRTIO_GPU_F_VIRGL       → has_virgl_3d (Venus 需要)
  │   ├─ VIRTIO_GPU_F_EDID        → has_edid
  │   ├─ VIRTIO_GPU_F_RESOURCE_BLOB → has_resource_blob (Venus 需要)
  │   ├─ VIRTIO_GPU_SHM_ID_HOST_VISIBLE → has_host_visible (Host 可见内存)
  │   ├─ VIRTIO_GPU_F_CONTEXT_INIT → has_context_init (Venus 需要)
  │   └─ VIRTIO_GPU_F_RESOURCE_UUID → has_resource_assign_uuid (跨设备共享)
  ├─ 创建 virtqueues (control + cursor)
  ├─ 读取 num_capsets (来自 virtio config space)
  └─ virtio_gpu_get_capsets():
      └─ 循环获取每个 capset 的 info (id, max_version, max_size)
         └─ 构建 capset_id_mask (用于后续 Venus 查询 capset)
```

**关键 feature bits 对 Venus 的影响：**

| Feature Bit | 含义 | Venus 是否强制 |
|---|---|---|
| `VIRTIO_GPU_F_VIRGL` | 3D 渲染能力 | **是** — 无此 Venus 无法工作 |
| `VIRTIO_GPU_F_RESOURCE_BLOB` | Blob 资源 (更高效的 mem 模型) | **是** |
| `VIRTIO_GPU_F_CONTEXT_INIT` | 上下文初始化协议 | **是** — DRM_VIRTGPU_CONTEXT_INIT |
| `VIRTIO_GPU_SHM_ID_HOST_VISIBLE` | Host 可见共享内存 | 可选 (有则用于 BO 映射) |
| `VIRTIO_GPU_F_RESOURCE_UUID` | 跨 virtio 设备共享 | 可选 (CROSS_DEVICE) |

### 阶段2：Venus ICD 打开 DRM 设备并初始化渲染器

文件: `mesa/src/virtio/vulkan/vn_renderer_virtgpu.c:1766` (`virtgpu_init()`)

```
vn_CreateInstance()  (vn_instance.c:251)
  ├─ vn_instance_init_renderer():
  │   └─ vn_renderer_create() → vn_renderer_create_virtgpu()
  │       └─ virtgpu_init():
  │           ├─ virtgpu_open():
  │           │   ├─ drmGetDevices2() 枚举 DRM 设备
  │           │   ├─ 过滤: PCI vendor=0x1af4 device=0x1050 (virtio-gpu)
  │           │   │  或 DRM_BUS_PLATFORM (如 crosvm)
  │           │   ├─ 打开 /dev/dri/renderD* (render node)
  │           │   └─ 验证: drmGetVersion() → name=="virtio_gpu" && major==0
  │           ├─ virtgpu_init_params():
  │           │   ├─ DRM_IOCTL_VIRTGPU_GETPARAM 逐个查询:
  │           │   │   ├─ VIRTGPU_PARAM_3D_FEATURES       (必须)
  │           │   │   ├─ VIRTGPU_PARAM_CAPSET_QUERY_FIX  (必须)
  │           │   │   ├─ VIRTGPU_PARAM_RESOURCE_BLOB     (必须)
  │           │   │   ├─ VIRTGPU_PARAM_CONTEXT_INIT      (必须)
  │           │   │   ├─ VIRTGPU_PARAM_HOST_VISIBLE → bo_blob_mem = HOST3D
  │           │   │   ├─ VIRTGPU_PARAM_GUEST_VRAM  → bo_blob_mem = GUEST_VRAM
  │           │   │   └─ VIRTGPU_PARAM_CROSS_DEVICE      (可选)
  │           │   └─ max_timeline_count = 64
  │           ├─ virtgpu_init_capset():
  │           │   └─ DRM_IOCTL_VIRTGPU_GET_CAPS:
  │           │       cap_set_id = VIRTGPU_DRM_CAPSET_VENUS (=4)
  │           │       cap_set_ver = 0
  │           │       → 获取 virgl_renderer_capset_venus 结构体
  │           └─ virtgpu_init_context():
  │               └─ DRM_IOCTL_VIRTGPU_CONTEXT_INIT:
  │                   ├─ VIRTGPU_CONTEXT_PARAM_CAPSET_ID = VENUS(4)
  │                   ├─ VIRTGPU_CONTEXT_PARAM_NUM_RINGS = 64
  │                   └─ VIRTGPU_CONTEXT_PARAM_POLL_RINGS_MASK = 0
  └─ vn_instance_init_renderer_versions():
      ├─ vn_call_vkEnumerateInstanceVersion() → renderer instance version
      ├─ 验证: renderer_api_version >= VN_MIN_RENDERER_VERSION
      └─ 协商版本 = MIN(renderer_api, app_requested, vk_xml_version)
```

### 阶段3：Capset 协商 — Venus 能力交换核心

文件: `mesa/src/virtio/virtio-gpu/venus_hw.h:29`

**Venus Capset 结构体** (`virgl_renderer_capset_venus`):

```c
struct virgl_renderer_capset_venus {
    uint32_t wire_format_version;          // Venus 协议版本
    uint32_t vk_xml_version;               // Host 侧 Vulkan XML 版本
    uint32_t vk_ext_command_serialization_spec_version;  // CS 扩展版本
    uint32_t vk_mesa_venus_protocol_spec_version;         // Venus 私有协议版本
    uint32_t supports_blob_id_0;           // 支持 blob_id=0 (shmem 分配)
    uint32_t vk_extension_mask1[32];       // 1023 个 Vulkan 扩展的位掩码
    uint32_t allow_vk_wait_syncs;          // 允许 guest 侧 wait
    uint32_t supports_multiple_timelines;  // 多时间线支持
    uint32_t use_guest_vram;               // 从 guest 专用堆分配
};
```

**协商逻辑** (`vn_instance_init_renderer()` + `virtgpu_init_renderer_info()`):

| Capset 字段 | 协商策略 |
|---|---|
| `wire_format_version` | 严格匹配 — 不匹配即初始化失败 |
| `vk_xml_version` | 取 MIN(host_vk_xml, guest_vk_xml)，低于 VN_MIN_RENDERER_VERSION 则失败 |
| `vk_ext_command_serialization_spec_version` | 取 MIN(host, guest) |
| `vk_mesa_venus_protocol_spec_version` | 取 MIN(host, guest)。 **<3 限制到 Vulkan 1.3** (无 VK_EXT_host_image_copy passthrough) |
| `vk_extension_mask1[32]` | 1023-bit 位掩码：Host 标记哪些 Vulkan 扩展协议层支持编码。bit 0 为 validity flag |
| `supports_multiple_timelines` | Host 支持多时间线 → guest 每个 VkQueue 绑定独立 ring_idx |
| `allow_vk_wait_syncs` | Host 允许 → guest 直通 vkWaitSemaphores 等阻塞调用 |
| `use_guest_vram` | 标记 blob 分配方式 (dedicated heap vs normal) |

### 阶段4：物理设备能力查询

文件: `mesa/src/virtio/vulkan/vn_physical_device.c`

```
vn_physical_device_init()
  ├─ vn_physical_device_init_renderer_version()   ← vkGetPhysicalDeviceProperties
  ├─ vn_physical_device_init_renderer_extensions() ← vkEnumerateDeviceExtensionProperties
  │   └─ 同时检查 Venus 协议编码器是否支持该扩展 (vn_extension_get_spec_version)
  ├─ vn_physical_device_init_supported_extensions()
  │   ├─ vn_physical_device_get_native_extensions()   ← Venus 原生实现
  │   │   ├─ KHR_external_memory_fd / EXT_external_memory_dma_buf
  │   │   ├─ KHR_external_fence_fd / KHR_external_semaphore_fd
  │   │   ├─ KHR_swapchain 等 WSI 扩展
  │   │   ├─ EXT_pci_bus_info, EXT_physical_device_drm
  │   │   └─ EXT_map_memory_placed, EXT_device_memory_report
  │   └─ vn_physical_device_get_passthrough_extensions() ← 直通 renderer
  │       ├─ Vulkan 1.0~1.4 所有 promoted extensions
  │       ├─ KHR_* (ray_tracing, fragment_shading_rate, mesh_shader, ...)
  │       └─ EXT_* (graphics_pipeline_library, descriptor_heap, ...)
  ├─ vn_physical_device_init_features()           ← vkGetPhysicalDeviceFeatures2
  │   └─ 特殊处理: 禁用 host commands (accelerationStructureHostCommands=false)
  ├─ vn_physical_device_init_properties()         ← vkGetPhysicalDeviceProperties2
  │   └─ vn_physical_device_sanitize_properties():
  │       ├─ apiVersion = MIN(host, VN_MAX_API_VERSION, vk_xml_version)
  │       ├─ Clamp 逻辑:
  │       │   ├─ venus_protocol < 3 → max Vulkan 1.3
  │       │   └─ 无 KHR_synchronization2 → max Vulkan 1.2
  │       ├─ driverID = VK_DRIVER_ID_MESA_VENUS
  │       └─ deviceName = "Virtio-GPU Venus (<Host GPU Name>)"
  ├─ vn_physical_device_init_memory_properties()  ← vkGetPhysicalDeviceMemoryProperties2
  ├─ vn_physical_device_init_queue_family_properties()
  ├─ vn_physical_device_init_external_memory()    ← EXT_external_memory_dma_buf
  ├─ vn_physical_device_init_external_fence_handles()
  └─ vn_physical_device_init_external_semaphore_handles()
```

### 阶段5：设备创建和队列初始化

文件: `mesa/src/virtio/vulkan/vn_device.c:56` (`vn_queue_init()`)

```
vn_CreateDevice()
  ├─ 调用 vn_call_vkCreateDevice() 在 Host 创建设备
  └─ vn_device_init_queues():
      └─ 每个 VkQueue:
          ├─ vn_instance_acquire_ring_idx() → 分配 ring_idx (1~63)
          ├─ VkDeviceQueueTimelineInfoMESA.ringIdx = ring_idx
          └─ vn_call_vkGetDeviceQueue2() 绑定到该 timeline
```

**Ring/Timeline 映射:**
- `ring_idx=0`: 保留给 CPU timeline (同步命令立即完成)
- `ring_idx=1~63`: 每个 VkQueue 绑定一个独立的 GPU timeline
- 共享队列模拟 (Android): 某些平台需要第二个 graphics queue (emulate_second_queue)

---

## 三、内存类型与 Blob 分配

### 3.1 内存类型获取

`vn_physical_device_init_memory_properties()` (vn_physical_device.c:978):

```
vkGetPhysicalDeviceMemoryProperties2() → host GPU 的内存类型
  └─ sanitize:
      ├─ 内核保证所有映射为 coherent → 无 HOST_COHERENT 但 HOST_CACHED 的剔除
      ├─ 如无 coherent-cached 类型 → 给第一个 coherent 类型追加 CACHED 属性
      └─ 内核保证 coherency → bo_flush/bo_invalidate 都是空操作 (nop)
```

### 3.2 Blob 内存分配策略

`vngpu_ioctl_resource_create_blob()` (vn_renderer_virtgpu.c:631):

| blob_mem | 值 | 含义 | 选择条件 |
|---|---|---|---|
| `VIRTGPU_BLOB_MEM_GUEST` | 0x0001 | Guest 系统内存 (sglist) | 旧的方式，QEMU 不易直接访问 |
| `VIRTGPU_BLOB_MEM_HOST3D` | 0x0002 | Host 3D 分配 | `VIRTGPU_PARAM_HOST_VISIBLE`=1 时 (bo_blob_mem) |
| `VIRTGPU_BLOB_MEM_HOST3D_GUEST` | 0x0003 | Guest mem, host 3D 可见 | 用于 guest_vram fallback |
| `VIRTGPU_BLOB_MEM_GUEST_VRAM` | 0x0004 | Guest VRAM 专用堆 | `VIRTGPU_PARAM_GUEST_VRAM`=1 时 |

**选择优先级** (`virtgpu_init_params()` vn_renderer_virtgpu.c:1624):
1. 先查 `VIRTGPU_PARAM_HOST_VISIBLE` → `VIRTGPU_BLOB_MEM_HOST3D`
2. 否则查 `VIRTGPU_PARAM_GUEST_VRAM` → `VIRTGPU_BLOB_MEM_GUEST_VRAM`
3. 两者都无 → 初始化失败

**共享内存 (shmem) 用 blob_id=0 标记：**
- `shmem_blob_mem = VIRTGPU_BLOB_MEM_HOST3D` 用于 Venus 内部 CS 缓冲
- 依赖 `supports_blob_id_0` capset flag (强制要求，由 render server 配置保证)

### 3.3 Blob Flags

`virtgpu_bo_blob_flags()` (vn_renderer_virtgpu.c:1176):

| Flag | 值 | 触发条件 |
|---|---|---|
| `VIRTGPU_BLOB_FLAG_USE_MAPPABLE` | 0x0001 | `VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT` |
| `VIRTGPU_BLOB_FLAG_USE_SHAREABLE` | 0x0002 | 有 external_handles |
| `VIRTGPU_BLOB_FLAG_USE_CROSS_DEVICE` | 0x0004 | `VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT` 且 `CROSS_DEVICE` 可用 |

### 3.4 Dma-buf 导入

`virtgpu_bo_create_from_dma_buf()` (vn_renderer_virtgpu.c:1194):
- 通过 `DRM_IOCTL_PRIME_FD_TO_HANDLE` 导入
- 通过 `DRM_IOCTL_VIRTGPU_RESOURCE_INFO` 获取实际的 blob_mem 和 size
- 验证 blob_mem 兼容性 (必须与 gpu->bo_blob_mem 一致)
- 支持同一 dma-buf 多次导入 (通过 gem_handle 匹配，使用 refcount)

---

## 四、能力协商总结

### 4.1 三阶段协商

```
Phase 1: virtio device features (QEMU → Linux kernel)
  ├─ VIRTIO_GPU_F_VIRGL            [Venus 强制]
  ├─ VIRTIO_GPU_F_RESOURCE_BLOB    [Venus 强制]
  ├─ VIRTIO_GPU_F_CONTEXT_INIT     [Venus 强制]
  ├─ VIRTIO_GPU_F_EDID              [可选]
  ├─ VIRTIO_GPU_SHM_ID_HOST_VISIBLE [可选，优先使用]
  └─ VIRTIO_GPU_F_RESOURCE_UUID    [可选，CROSS_DEVICE]

Phase 2: DRM params (Linux kernel → Venus ICD)
  ├─ VIRTGPU_PARAM_3D_FEATURES      [强制]
  ├─ VIRTGPU_PARAM_RESOURCE_BLOB    [强制]
  ├─ VIRTGPU_PARAM_CONTEXT_INIT     [强制]
  ├─ VIRTGPU_PARAM_CAPSET_QUERY_FIX [强制]
  ├─ VIRTGPU_PARAM_HOST_VISIBLE     [二选一]
  ├─ VIRTGPU_PARAM_GUEST_VRAM       [二选一]
  └─ VIRTGPU_PARAM_CROSS_DEVICE     [可选]

Phase 3: Venus capset (Host renderer → Venus ICD)
  ├─ wire_format_version            [严格匹配]
  ├─ vk_xml_version                 [取 MIN]
  ├─ vk_ext_command_serialization_spec_version [取 MIN]
  ├─ vk_mesa_venus_protocol_spec_version        [取 MIN]
  ├─ vk_extension_mask1[32]         [按位有效性检查]
  ├─ supports_blob_id_0             [强制要求]
  ├─ supports_multiple_timelines    [推荐]
  ├─ allow_vk_wait_syncs            [推荐]
  └─ use_guest_vram                 [按需]
```

### 4.2 Vulkan 扩展协商

Venus 支持 **两类** Vulkan 设备扩展:

**1. Passthrough (直通)** — Host GPU 实际支持且 Venus 协议编码器支持 → 直接透传:

覆盖 Vulkan 1.0~1.4 所有 promoted extensions，以及大量 KHR/EXT 扩展：
`KHR_acceleration_structure`, `KHR_ray_tracing_pipeline`, `KHR_fragment_shading_rate`,
`EXT_mesh_shader`, `EXT_graphics_pipeline_library`, `EXT_descriptor_heap`,
`EXT_extended_dynamic_state3`, 等等。

限制条件：
- `KHR_synchronization2` 要求 renderer 支持 semaphore sync fd import
- `EXT_host_image_copy` 要求 `vk_mesa_venus_protocol_spec_version >= 3`
- `EXT_memory_budget` 需要 VN_DEBUG(MEM_BUDGET) 开启
- `EXT_graphics_pipeline_library` 需要 !VN_DEBUG(NO_GPL)
- `EXT_descriptor_heap` 需要 !VN_DEBUG(NO_DESC_HEAP)
- ray_tracing 系列需要 !VN_DEBUG(NO_RAY_TRACING)

**2. Native (原生)** — Venus ICD 自身实现，不依赖 Host GPU:

| 扩展 | 条件 |
|---|---|
| `KHR_external_memory_fd` | renderer 支持 EXT_external_memory_dma_buf |
| `EXT_external_memory_dma_buf` | renderer 支持 EXT_external_memory_dma_buf |
| `KHR_external_fence_fd` | has_external_sync 且 fence_exportable |
| `KHR_external_semaphore_fd` | has_external_sync 且 semaphore importable/exportable |
| `KHR_swapchain` + WSI 系列 | semaphore sync fd import 可用 |
| `EXT_pci_bus_info` | renderer pci info 或 renderer 支持 EXT_pci_bus_info |
| `EXT_physical_device_drm` | 始终 |
| `EXT_map_memory_placed` | 非 Windows |
| `KHR_map_memory2` | 始终 |
| `EXT_device_memory_report` | 始终 |
| `EXT_tooling_info` | 始终 |

**扩展 mask 协商**: Host 通过 capset 的 `vk_extension_mask1[32]` 位掩码告知 Guest 哪些扩展在协议层已支持编码和传输。bit 0 为 backward-compat flag — 设 1 表示 mask 有效，否则假定全部支持。

### 4.3 API Version Clamp 逻辑

最终广告给 Vulkan app 的版本:

```c
// 基础 clamp
apiVersion = MIN3(host_api, VN_MAX_API_VERSION, vk_xml_version)

// 协议特性 clamp
if (venus_protocol_spec_version < 3)
    apiVersion = MIN(apiVersion, VK_API_VERSION_1_3)  // 无 host_image_copy passthrough

if (!KHR_synchronization2)
    apiVersion = MIN(apiVersion, VK_API_VERSION_1_2)  // sync2 是 1.3 必需

// 版本覆盖 (环境变量)
if (version_override)
    apiVersion = version_override
```

---

## 五、同步机制

### 5.1 drm_syncobj 时间线

Venus 使用 Linux drm_syncobj 作为同步原语 (`virtgpu_sync_*`):

```
Host VkSemaphore/Fence  ←→  drm_syncobj timeline  ←→  Guest VkSemaphore/Fence
   (renderer 侧)            (内核同步对象)             (Venus ICD 侧)
```

**syncobj 操作:**

| 操作 | DRM ioctl |
|---|---|
| `virtgpu_sync_create()` | `DRM_IOCTL_SYNCOBJ_CREATE` + `DRM_IOCTL_SYNCOBJ_TIMELINE_SIGNAL` |
| `virtgpu_sync_destroy()` | `DRM_IOCTL_SYNCOBJ_DESTROY` |
| `virtgpu_sync_write()` | `DRM_IOCTL_SYNCOBJ_TIMELINE_SIGNAL` |
| `virtgpu_sync_read()` | `DRM_IOCTL_SYNCOBJ_QUERY` |
| `virtgpu_sync_reset()` | `DRM_IOCTL_SYNCOBJ_RESET` + signal |
| `virtgpu_wait()` | `DRM_IOCTL_SYNCOBJ_TIMELINE_WAIT` |
| `virtgpu_sync_export_syncobj()` | `DRM_IOCTL_SYNCOBJ_HANDLE_TO_FD` |
| `virtgpu_sync_create_from_syncobj()` | `DRM_IOCTL_SYNCOBJ_FD_TO_HANDLE` |

### 5.2 提交模型

`virtgpu_ioctl_submit()` → `DRM_IOCTL_VIRTGPU_EXECBUFFER`:

```
struct drm_virtgpu_execbuffer {
    .flags        = RING_IDX | FENCE_FD_OUT (可选)
    .size         = cs_data_size
    .command      = cs_data_ptr
    .bo_handles   = gem_handle_array
    .num_bo_handles = bo_count
    .ring_idx     = batch->ring_idx
};
```

- 携带 CS data、BO handles、ring_idx
- 返回 out fence fd → 转换为 syncobj signal
- `ring_idx=0` 提交给 CPU timeline (同步处理)
- 隐含 fencing: BO 列表用于 kernel 自动添加隐式同步

### 5.3 外部 Fence/Semaphore 导出

Venus 的外部同步目前采用 **emulation** 策略：

- **Fence export (`vkGetFenceFdKHR`)**: 提交一个空 batch 到 renderer，获取 out fence，再通过 venus 协议命令修复 renderer 侧 fence payload
- **Binary semaphore export (`vkGetSemaphoreFdKHR`)**: 同上
- **Timeline semaphore export**: 不直接支持，通过 sync_file 间接导出

未来方向是创建 `vn_renderer_sync` 来原生映射 guest fence/semaphore 到 host side。

---

## 六、DRM UAPI 交互总结

Venus ICD 通过以下 DRM ioctls 与内核通信：

| ioctl | 用途 | 调用阶段 |
|---|---|---|
| `DRM_IOCTL_VIRTGPU_GETPARAM` | 查询内核参数 (3D_FEATURES, RESOURCE_BLOB, CONTEXT_INIT 等) | renderer init |
| `DRM_IOCTL_VIRTGPU_GET_CAPS` | 获取 Venus capset 结构体 | renderer init |
| `DRM_IOCTL_VIRTGPU_CONTEXT_INIT` | 创建 Venus 上下文 (绑定 capset_id=4) | renderer init |
| `DRM_IOCTL_VIRTGPU_RESOURCE_CREATE_BLOB` | 创建 blob 资源 (BO/shmem) | 内存分配 |
| `DRM_IOCTL_VIRTGPU_RESOURCE_INFO` | 查询 blob 资源信息 | dma-buf 导入 |
| `DRM_IOCTL_VIRTGPU_EXECBUFFER` | 提交 CS 到 renderer | 命令提交 |
| `DRM_IOCTL_VIRTGPU_MAP` | 映射 blob 资源到用户空间 | BO 映射 |
| `DRM_IOCTL_GEM_CLOSE` | 释放 GEM handle | BO 销毁 |
| `DRM_IOCTL_PRIME_HANDLE_TO_FD` | 导出 BO 为 dma-buf fd | 外部共享 |
| `DRM_IOCTL_PRIME_FD_TO_HANDLE` | 导入 dma-buf fd | 外部共享 |
| `DRM_IOCTL_SYNCOBJ_CREATE/DESTROY` | 创建/销毁同步对象 | 同步 |
| `DRM_IOCTL_SYNCOBJ_TIMELINE_SIGNAL/WAIT/QUERY` | 时间线操作 | 同步 |

---

## 七、关键文件索引

### Mesa Venus ICD

| 文件 | 核心内容 |
|---|---|
| `src/virtio/vulkan/vn_icd.c` | ICD 入口点 |
| `src/virtio/vulkan/vn_instance.c` | Instance 创建、renderer 连接、版本协商 |
| `src/virtio/vulkan/vn_instance.h` | Instance 结构定义 |
| `src/virtio/vulkan/vn_physical_device.c` | 物理设备特性/属性/内存/扩展枚举 (~3108 行) |
| `src/virtio/vulkan/vn_physical_device.h` | 物理设备结构定义 |
| `src/virtio/vulkan/vn_device.c` | 设备创建、队列初始化 |
| `src/virtio/vulkan/vn_device_memory.c` | 设备内存管理 |
| `src/virtio/vulkan/vn_renderer.h` | 渲染器抽象层接口 (sync, bo, shmem ops) |
| `src/virtio/vulkan/vn_renderer_virtgpu.c` | virtio-gpu 渲染器后端 (~1843 行，核心初始化) |
| `src/virtio/vulkan/vn_renderer_internal.h` | 内部结构 (shmem cache) |
| `src/virtio/virtio-gpu/venus_hw.h` | Venus capset 结构定义 |
| `include/drm-uapi/virtgpu_drm.h` | DRM uapi (含 VIRTGPU_DRM_CAPSET_VENUS=4) |

### Linux 内核 virtio-gpu 驱动

| 文件 | 核心内容 |
|---|---|
| `drivers/gpu/drm/virtio/virtgpu_drv.h` | 设备结构、fpriv 上下文定义 |
| `drivers/gpu/drm/virtio/virtgpu_kms.c` | 设备初始化、capset 获取、driver open/postclose |
| `drivers/gpu/drm/virtio/virtgpu_ioctl.c` | ioctl 处理 (全部 11 个 ioctl) |
| `drivers/gpu/drm/virtio/virtgpu_vq.c` | virtqueue 命令构建和通知 |
| `drivers/gpu/drm/virtio/virtgpu_object.c` | GEM 对象管理 |
| `drivers/gpu/drm/virtio/virtgpu_submit.c` | EXECBUFFER 实现 |
| `drivers/gpu/drm/virtio/virtgpu_fence.c` | Fence 分配和事件处理 |
| `drivers/gpu/drm/virtio/virtgpu_vram.c` | VRAM 分配 (GUEST_VRAM) |
| `include/uapi/drm/virtgpu_drm.h` | DRM UAPI 结构定义 |
| `include/uapi/linux/virtio_gpu.h` | virtio GPU 协议 (含 VIRTIO_GPU_CAPSET_VENUS=4) |

### QEMU

| 文件 | 核心内容 |
|---|---|
| `hw/display/virtio-gpu.c` | virtio-gpu 基础设备 (资源管理, 2D 命令) |
| `hw/display/virtio-gpu-base.c` | 设备基类 |
| `hw/display/virtio-gpu-virgl.c` | Virgl (OpenGL) 渲染器命令 |
| `hw/display/virtio-gpu-rutabaga.c` | Rutabaga (Venus/Vulkan) 渲染器命令 |
| `hw/display/virtio-gpu-pci.c` | PCI 设备实现 |
| `hw/display/virtio-gpu-pci-rutabaga.c` | PCI Rutabaga 变体 |
| `include/standard-headers/linux/virtio_gpu.h` | virtio GPU 协议头 (capset ids, config space) |
