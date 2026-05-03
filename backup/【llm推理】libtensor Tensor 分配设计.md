# libtensor Tensor 分配设计分析

本文分析 libtensor 中 Tensor 的内存分配设计。核心结论是：Tensor 本身只是轻量句柄，真实元数据放在 `TensorImpl`，真实内存生命周期放在 `Storage` / `MemoryBlock`，具体分配策略由 `ExecutionContext` 按 `Device + MemoryKind` 选择 allocator。

## 1. 设计目标

libtensor 的 Tensor 分配设计主要解决四个问题：

1. 让 Tensor 拷贝足够轻量，避免在 C++ 算子链路中频繁复制大对象。
2. 将形状、步长、dtype、device 与底层内存解耦，支持 view、sub-storage、外部借用内存等能力。
3. 将内存用途显式表达为 `MemoryKind`，避免普通 tensor、权重、workspace、KV cache、host pinned memory 混在一个分配池里。
4. 让 CPU / CUDA / pinned / caching allocator 都通过同一套抽象接入，算子层不直接依赖 `malloc`、`cudaMalloc` 或具体缓存策略。

整体结构可以概括为：

```text
Tensor
  -> TensorImpl
       -> Storage
            -> intrusive_ptr<MemoryBlock>
                 -> raw pointer
                 -> owner allocator
                 -> device / memory kind / bytes
```

## 2. Tensor 是轻量句柄

`Tensor` 定义在 `include/libtensor/tensor.h`。它内部只有一个 `TensorImpl* impl_`，对外暴露 `dtype()`、`device()`、`sizes()`、`strides()`、`data()` 等查询接口。

关键点：

1. `Tensor` 拷贝只增加 `TensorImpl` 的 intrusive 引用计数，不复制底层数据。
2. `Tensor` 移动只转移裸指针，不改变底层内存。
3. `Tensor()` 表示 undefined tensor，`defined()` 只检查 `impl_ != nullptr`。
4. `Tensor` 不拥有 allocator，也不直接释放 raw pointer，释放行为由 `TensorImpl -> Storage -> MemoryBlock` 逐层触发。

这种设计让算子 wrapper 可以安全地用栈数组传递输入输出：

```cpp
const Tensor inputs_arr[2] = { a, b };
Tensor outputs_arr[1] = { out };
```

这里复制的是句柄，而不是 tensor 数据。

## 3. TensorImpl 保存逻辑视图

`TensorImpl` 定义在 `src/core/tensor_impl.h`，保存：

1. `Storage storage_`：底层内存。
2. `storage_offset_`：以元素为单位的起始偏移。
3. `sizes_`：逻辑形状。
4. `strides_`：逻辑步长。
5. `dtype_`：元素类型。
6. `is_contiguous_`：构造时计算出的连续性标记。

`TensorImpl::data()` 的计算方式是：

```text
storage.data() + storage_offset * dtype_size_bytes(dtype)
```

这说明 `TensorImpl` 负责的是“逻辑 tensor 视图”，而不是“物理内存块”。同一块 `Storage` 可以被多个 `TensorImpl` 以不同 `storage_offset`、`sizes`、`strides` 解释。

当前实现中，大多数 kernel 只接受 contiguous tensor，因此很多 op wrapper 会在调度前检查 `is_contiguous()`。这不是 Tensor 层的限制，而是当前 kernel 能力的边界。

## 4. Storage 管理物理内存切片

`Storage` 定义在 `src/core/storage.h`，它持有 `intrusive_ptr<MemoryBlock>`，并额外保存 `offset_` 与 `nbytes_`。

`MemoryBlock` 是真正的内存生命周期对象：

1. `ptr`：allocator 返回的原始指针。
2. `bytes`：整块内存大小。
3. `device`：内存所属 device。
4. `kind`：内存用途分类。
5. `owner`：释放时调用的 allocator。
6. `last_stream`：预留给 stream-aware deferred free。

`MemoryBlock::~MemoryBlock()` 中，如果 `ptr && owner`，会调用：

```cpp
owner->deallocate(ptr, bytes);
```

这带来两个重要语义：

1. allocator 不需要被 Tensor 显式持有，`MemoryBlock` 只保存 `IAllocator* owner`。
2. 借用外部内存时 `owner == nullptr`，析构不会释放外部指针。

## 5. Storage 的三类来源

### 5.1 由 allocator 分配

`Storage(Device device, size_t nbytes, std::shared_ptr<IAllocator> alloc, MemoryKind kind)` 是普通 tensor 分配路径。

如果 `nbytes == 0`，不会创建 `MemoryBlock`，但会保留 `device_`，这样 zero-size tensor 仍能回答 device 信息。

如果 `nbytes > 0` 但 allocator 为空，会抛出 `std::invalid_argument`。上层 `make_empty_like_shape()` 通常会先检查 allocator 是否存在，失败时返回 undefined tensor。

### 5.2 借用外部内存

`Storage(Device device, void* data, size_t nbytes)` 与 `Storage::borrowed()` 可以包装外部指针。

区别是：

1. 构造函数默认 `MemoryKind::Default`。
2. `borrowed()` 可以显式指定 `MemoryKind`。
3. 两者都会让 `owner == nullptr`，因此不拥有外部内存。

这适合 safetensors、host buffer、外部 runtime 传入内存等场景，但调用方必须保证外部指针生命周期覆盖 Tensor 使用期。

### 5.3 sub-storage 共享底层块

`Storage::sub_storage(offset, nbytes)` 返回一个新的 `Storage`，共享同一个 `MemoryBlock`，只改变 `offset_` 与 `nbytes_`。

这给 view / slice 类操作提供了基础：多个逻辑 tensor 可以共享一块物理内存，引用计数保证最后一个 view 释放后才归还底层块。

## 6. empty 的分配链路

当前主要分配入口是 `empty()` 与内部工具 `make_empty_like_shape()`。

典型链路如下：

```text
empty(sizes, dtype, device, kind)
  -> make_empty_like_shape(sizes, dtype, device, kind)
       -> safe_numel_bytes(sizes, dtype)
       -> ExecutionContext::get_allocator(device, kind)
       -> Storage(device, nbytes, allocator, kind)
       -> compute_contiguous_strides(sizes)
       -> new TensorImpl(storage, 0, sizes, strides, dtype)
       -> Tensor(impl)
```

其中 `safe_numel_bytes()` 是一个关键防线：

1. 拒绝负数维度。
2. 任意维度为 0 时返回 0 字节。
3. 对 `numel * dtype_size` 做 overflow 检查。

这保证了分配前不会因为形状异常导致 size_t 溢出。

## 7. MemoryKind 是分配策略入口

`MemoryKind` 定义在 `include/libtensor/memory_kind.h`：

1. `Default`：普通 tensor。
2. `Persistent`：长期存活对象，例如模型权重。
3. `Workspace`：临时 scratch buffer。
4. `KVCache`：长期存活且可能按块增长的 KV cache。
5. `HostPinned`：适合 DMA 传输的 pinned host memory。
6. `HostPageable`：普通 host memory。

`ExecutionContext` 用 `DeviceKindKey { Device, MemoryKind }` 管理 allocator：

```text
(CPU, Default)      -> CPU caching/default allocator
(CPU, HostPinned)   -> pinned allocator 或 CPU fallback
(CUDA:0, Default)   -> CUDA caching/default allocator
(CUDA:0, Workspace) -> workspace allocator
(CUDA:0, KVCache)   -> KV cache allocator
```

`make_empty_like_shape()` 对 host memory kind 有一个显式约束：

```cpp
if ((kind == MemoryKind::HostPinned || kind == MemoryKind::HostPageable) && device.type != DeviceType::CPU)
{
    return Tensor { };
}
```

这避免出现“device 是 CUDA，但 memory kind 是 host pinned”的含混状态。Pinned host memory 在语义上仍属于 CPU device，只是物理页锁定、适合异步传输。

## 8. ExecutionContext 统一注册 allocator

`ExecutionContext` 是全局 runtime 状态，负责：

1. 默认 device。
2. `Device -> allocator` 的旧式映射。
3. `(Device, MemoryKind) -> allocator` 的新式映射。
4. `Device -> Stream` 的当前 stream。
5. allocator stats 查询。

`register_allocator(Device, alloc)` 会同时注册到 `MemoryKind::Default`。`register_allocator(Device, MemoryKind, alloc)` 则注册指定 memory kind。

设计上的好处是：

1. Tensor 分配入口不关心后端初始化细节。
2. 后端可以在 `init_cpu_backend()`、`init_cuda_backend()` 中集中配置 allocator。
3. 后续可以为 `Persistent`、`Workspace`、`KVCache` 替换不同 allocator，而无需改 Tensor API。

## 9. Allocator 抽象与当前实现

所有 allocator 都实现 `IAllocator`：

```cpp
class IAllocator {
public:
    virtual void* allocate(size_t bytes) = 0;
    virtual void deallocate(void* ptr, size_t bytes) = 0;
    virtual Device device() const = 0;
    virtual MemoryStats stats() const;
};
```

### 9.1 CpuAllocator

`CpuAllocator` 使用 64 字节对齐分配：

1. MSVC 下使用 `_aligned_malloc`。
2. 其他平台使用 `std::aligned_alloc`。

它是简单直接的 allocator，适合作为基础 fallback。

### 9.2 CpuCachingAllocator

`CpuCachingAllocator` 使用 power-of-two bucket 缓存释放后的内存块。

分配时：

1. 将请求 size round up 到 2 的幂。
2. 查找对应 free list。
3. 命中则复用缓存块。
4. 未命中才调用底层 aligned allocation。
5. 用 `allocated_` 记录指针实际 rounded size。

释放时：

1. 不立即归还 OS。
2. 放回对应 bucket 的 free list。
3. 更新 cached bytes 与 free 统计。

这降低了频繁创建临时 tensor 时的系统分配成本，代价是 reserved memory 可能高于 active memory。

### 9.3 CUDA allocator

CUDA 侧同样有直接 `CudaAllocator` 与 `CudaCachingAllocator`。

`CudaCachingAllocator` 的重要行为：

1. 每次 `cudaMalloc` / `cudaFree` 前切换到 allocator 绑定的 device。
2. 使用 power-of-two bucket 缓存 GPU 内存。
3. `release_all_cached()` 可以释放缓存块。
4. 静态实例注册了 `atexit` shutdown，尽量在 CUDA driver 卸载前释放缓存。

代码中还存在 `DeferredDeleter`，通过 CUDA event 延迟释放 stream 上仍可能被使用的指针。当前 `MemoryBlock::last_stream` 也为这个方向预留了字段，但普通 `MemoryBlock` 析构路径仍是同步调用 `owner->deallocate()`。

### 9.4 Pinned allocator

Pinned allocator 用于 host pinned memory：

1. CUDA 可用时使用 `cudaMallocHost` / `cudaFreeHost`。
2. CPU-only fallback 使用 aligned host allocation。
3. `device()` 返回 CPU，因为 pinned memory 仍是 host memory。

## 10. WorkspaceArena 与 ExecutionPlan

除了普通 Tensor 分配，libtensor 还引入了 workspace 层。

`WorkspaceArena` 是 bump-pointer arena：

1. 构造或 `reserve()` 时一次性向 backing allocator 申请大块内存。
2. `allocate(bytes, alignment)` 只移动 `used_` 指针。
3. `reset()` 只重置 `used_`，不释放底层内存。
4. 析构时一次性释放底层块。

这适合单步推理中的临时 scratch buffer，避免在 kernel 链中频繁分配释放。

`ExecutionPlan` 则负责提前声明 buffer：

1. `add_buffer()` 记录 shape、dtype、device、memory kind、alignment。
2. `prepare_buffers()` 逐个调用 `empty()` 分配 tensor。
3. `workspace_bytes_` 记录按 alignment 对齐后的总需求。
4. `add_view_buffer()` 可以基于已有 buffer 创建 view。

需要注意：当前 `ExecutionPlan::prepare_buffers()` 虽然统计了 `workspace_bytes_`，但 buffer 本身仍通过 `empty()` 逐个分配，而不是从一个 `WorkspaceArena` 中切片。因此它更像“静态 buffer 计划与预分配”，还不是完整的 arena-backed tensor storage。

## 11. 当前设计的关键优点

### 11.1 句柄、视图、内存块分层清楚

`Tensor`、`TensorImpl`、`Storage`、`MemoryBlock` 各自职责明确：

1. `Tensor`：用户可见句柄。
2. `TensorImpl`：逻辑视图。
3. `Storage`：物理切片。
4. `MemoryBlock`：底层块与释放策略。

这种分层让 view、borrowed storage、workspace、KV cache 后续都能在现有结构上扩展。

### 11.2 MemoryKind 让用途进入 allocator 选择

很多框架早期只按 device 选 allocator，后续引入 workspace / pinned / KV cache 时会出现 API 兼容压力。libtensor 已经把 `MemoryKind` 放进 `empty()` 和 `ExecutionContext`，这是正确方向。

### 11.3 引用计数避免大规模 shared_ptr 开销

Tensor 和 MemoryBlock 使用 intrusive ref counter，减少了 shared_ptr 控制块分配，也让 Tensor 拷贝语义更接近底层 runtime 需要。

### 11.4 Kernel 层不负责分配策略

大部分 kernel wrapper 接收已经分配好的 output tensor，kernel 只写数据。分配发生在 creation op、模型 plan、调用方或上层 operator 中。这让 kernel 注册调度和内存策略保持相对独立。

## 12. 当前边界与风险

### 12.1 `Tensor::empty()` 与 `empty()` 存在两条路径

`Tensor::empty()` 使用 `ExecutionContext::get_allocator(device)`，而 `ops::empty()` 使用 `get_allocator(device, kind)`。后者更完整，因为它支持 `MemoryKind`。

建议后续将 public/static 的 `Tensor::empty()` 收敛到统一的 `make_empty_like_shape()` 或标记为兼容入口，避免新增代码绕过 `MemoryKind`。

### 12.2 `MemoryBlock` 只保存 allocator 裸指针

`MemoryBlock::owner` 是 `IAllocator*`，不是 `shared_ptr<IAllocator>`。这要求 allocator 生命周期必须长于所有由它分配的 tensor。

当前 singleton allocator 与 `ExecutionContext` 注册方式基本满足这个假设。但如果未来允许动态卸载插件 allocator 或临时 allocator，需要重新评估 owner 生命周期。

### 12.3 caching allocator 没有按 stream 完整隔离

CUDA caching allocator 当前释放时直接把块放回 free list。对于异步 kernel，如果 tensor 在某个 stream 上仍被使用而 host 端引用已经释放，就需要 deferred free 保护。

代码中已经有 `DeferredDeleter` 和 `MemoryBlock::last_stream` 的设计痕迹，但还没有贯穿普通 Tensor 析构路径。后续做高并发 stream 或 CUDA graph capture 时，这是需要补齐的重点。

### 12.4 view 的边界校验仍需加强

`Storage::sub_storage()` 当前直接共享 block 并增加 offset，没有在这里检查 `offset + nbytes <= storage.nbytes()`。如果上层 view 计算错误，可能产生越界视图。

这类校验可以放在 view op 层，也可以在 Storage 层增加 defensive check。考虑到 Storage 是基础设施，建议至少在 debug 或始终启用的路径做边界保护。

### 12.5 WorkspaceArena 尚未成为 Tensor Storage 后端

`WorkspaceArena` 可以分配 raw pointer，但普通 Tensor 分配仍通过 allocator 创建独立 `MemoryBlock`。如果未来希望 workspace tensor 完全 arena-backed，需要一个“不释放单个切片，只由 arena 统一 reset”的 Storage owner 机制。

可能方向：

1. 引入 arena-backed `MemoryBlock`，owner 不直接释放切片。
2. 将 workspace tensor 表达为 borrowed storage，但由 `RuntimeContext` 保证 arena 生命周期。
3. 在 `ExecutionPlan` 中预分配大块 `Storage`，然后用 `sub_storage()` 创建 tensor view。

## 13. 扩展建议

### 13.1 新增 MemoryKind 时的接入步骤

新增一种内存用途时，应按以下路径接入：

1. 在 `MemoryKind` 中增加枚举和 `memory_kind_name()`。
2. 在后端初始化中注册 `(Device, MemoryKind) -> allocator`。
3. 在 `empty()`、模型 plan 或 cache 初始化中传入对应 kind。
4. 在测试中验证 `tensor.memory_kind()`、allocator stats 和释放行为。

### 13.2 新增 allocator 时的注意事项

新增 allocator 必须满足：

1. `allocate()` / `deallocate()` 线程安全。
2. `deallocate(ptr, bytes)` 能接受原始请求 size，或自行记录 rounded size。
3. `device()` 返回真实语义 device。
4. `stats()` 语义清楚区分 active / reserved / cached。
5. 如果是 CUDA allocator，需要明确 stream safety 与 device switching 策略。

### 13.3 大模型推理下的推荐分层

对于 LLM 推理，建议按用途使用不同 memory kind：

1. 权重：`Persistent`。
2. 每层中间激活：`Workspace`。
3. KV cache / recurrent state：`KVCache`。
4. CPU 到 GPU staging buffer：`HostPinned`。
5. 普通临时 tensor：`Default`。

这样 allocator stats 能直接回答“显存主要被权重、workspace 还是 KV cache 占用”，也方便后续做不同生命周期的复用策略。

## 14. 总结

libtensor 的 Tensor 分配设计已经具备一个推理 runtime 所需的核心骨架：轻量 Tensor 句柄、引用计数 Impl、共享 Storage、allocator 抽象、MemoryKind 分层和 workspace arena。它最重要的设计点不是某一个 allocator，而是把“逻辑 tensor 视图”和“物理内存生命周期”拆开，并把“内存用途”显式纳入分配决策。

后续最值得继续强化的方向是：统一分配入口、补齐 stream-aware deferred free、加强 view 边界校验，以及让 `WorkspaceArena` 真正成为高频临时 tensor 的 storage 后端。
