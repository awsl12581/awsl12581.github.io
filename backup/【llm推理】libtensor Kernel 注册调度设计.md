# libtensor Kernel 注册调度设计分析

具体代码见[源码](https://git.cinogama.net/nongchatea/libtensor)

本文分析 libtensor 的 kernel 注册与调度设计。核心结论是：libtensor 把“算子语义层”和“后端 kernel 层”用 `Dispatcher` 隔开，调用方只根据 op name、device、dtype、layout 准备 `KernelKey`，具体 CPU / CUDA / BLAS / cuBLAS / cuDNN / FlashAttention 实现都通过注册覆盖或条件注册接入。

## 1. 设计目标

kernel 注册调度机制主要解决五个问题：

1. 同一个 op API 可以在 CPU、CUDA 等后端上复用。
2. 同一个 op 可以按 dtype 注册不同实现。
3. 基础 fallback kernel 与加速库 kernel 可以共存，并通过注册顺序覆盖。
4. 高层 operator 可以在 prepare 阶段缓存 kernel lookup 结果，避免每次 forward 重复查表。
5. LLM 专用 kernel 可以和通用 unary / binary / matmul / conv / reduce 使用同一套调度接口。

整体链路可以概括为：

```text
op wrapper / PreparedOperator
  -> OperatorInitContext / invoke_dispatch_kernel
       -> Dispatcher lookup(KernelKey)
            -> backend register_*_kernels(DType)
                 -> KernelFn(KernelContext, inputs, outputs)
                      -> CPU loop / CUDA launch / BLAS / cuBLAS / cuDNN / FlashAttention
```

## 2. KernelKey 是调度索引

`KernelKey` 定义在 `src/runtime/dispatcher.h`：

```cpp
struct KernelKey
{
    std::string op_name;
    DeviceType device = DeviceType::CPU;
    DType dtype;
    Layout layout = Layout::Strided;
};
```

它表达了当前调度系统认为影响 kernel 选择的四个维度：

1. `op_name`：算子名，例如 `matmul.mm`、`binary.add`、`llm.flash_attention`。
2. `device`：后端类型，目前主要是 CPU / CUDA。
3. `dtype`：数据类型，例如 float32、float16、bfloat16、int64。
4. `layout`：布局，目前主要使用 `Layout::Strided`。

这个设计刻意没有把 shape、stream、workspace、scalar 参数等运行时信息放进 key。原因是这些信息不决定“选择哪个函数指针”，而是作为 `KernelContext`、`KernelParams` 或 Tensor metadata 传给已经选中的 kernel。

## 3. KernelFn 是统一 ABI

所有被 Dispatcher 管理的 kernel 都使用同一个函数指针类型：

```cpp
using KernelFn = void (*)(KernelContext& ctx, Span<const Tensor> inputs, Span<Tensor> outputs);
```

`KernelContext` 包含：

1. `Device device`：当前运行 device。
2. `Stream stream`：当前 stream。
3. `WorkspaceArena* workspace`：可选 workspace。
4. `KernelParams params`：类型擦除的小参数块。

这个 ABI 的好处是 kernel 注册表只需要保存函数指针，不需要为每个 op 定义不同签名。不同 op 的差异通过三类数据表达：

1. Tensor 输入输出数量与形状由 wrapper 约定。
2. 标量、超参数、attention 参数等通过 `KernelParams` 传递。
3. 临时内存通过 `ctx.workspace` 传递。

代价是 kernel 内部必须自己解释 inputs / outputs / params，因此 wrapper 层的校验非常重要。

## 4. KernelParams 处理非 Tensor 参数

`KernelParams` 是一个轻量类型擦除结构：

```cpp
struct KernelParams
{
    const void* data = nullptr;
    size_t size = 0;
    const std::type_info* type = nullptr;
};
```

调用方用：

```cpp
KernelParams::make(params)
```

kernel 内用：

```cpp
auto* p = ctx.params.as<AttentionParams>();
```

这让 scalar binary、RoPE、RMSNorm、FlashAttention varlen 等不必把配置 tensor 化。例如 `binary.add_scalar` 使用 `BinaryScalarParams`，attention 使用 `AttentionParams` 或 `AttentionVarlenParams`。

需要注意：`KernelParams::make()` 保存的是外部对象地址，不拷贝数据。因此该 params 对象必须在 kernel 调用期间保持有效。当前调用方式通常是在栈上创建 params 并立即同步调用 kernel function pointer，这个生命周期是安全的；如果未来引入异步延迟执行或 graph capture 参数缓存，需要重新设计 params ownership。

## 5. Dispatcher 是全局注册表

`Dispatcher` 内部保存两个 map：

1. `registry_`：`KernelKey -> KernelFn`。
2. `meta_registry_`：`KernelKey -> KernelMeta`。

核心接口是：

```cpp
void register_kernel(const KernelKey& key, KernelFn fn);
void register_kernel(const KernelKey& key, KernelFn fn, KernelMeta meta);
KernelFn lookup(const KernelKey& key) const;
KernelMeta lookup_meta(const KernelKey& key) const;
bool has_kernel(const KernelKey& key) const;
```

所有读写都用 mutex 保护，所以注册和查询在基本并发场景下是安全的。

注册语义是覆盖写入：

```cpp
registry_[key] = fn;
```

这点非常关键，因为 BLAS / cuBLAS / cuDNN 这类加速实现就是通过“后注册覆盖 fallback”接入的。

## 6. 注册入口：按后端和 dtype 批量注册

统一注册入口在 `src/runtime/op_registry.h`。

CPU 侧：

```text
register_all_cpu_kernels()
  -> register_cpu_kernels_for_dtype(float32)
  -> register_cpu_kernels_for_dtype(float64)
  -> register_cpu_kernels_for_dtype(bfloat16)
  -> register_cpu_kernels_for_dtype(int32)
  -> register_cpu_kernels_for_dtype(int64)
```

每个 dtype 会注册：

1. unary。
2. binary。
3. matmul。
4. conv。
5. reduce。
6. llm。
7. creation。
8. copy。

如果启用 `LIBTENSOR_USE_BLAS`，则在 fallback 后注册 BLAS matmul / conv，从而覆盖同 key 的 naive kernel。

CUDA 侧类似：

```text
register_all_cuda_kernels()
  -> float16 / bfloat16 / float32 / float64 / int32 / int64
```

每个 dtype 会注册：

1. unary。
2. binary。
3. matmul。
4. conv。
5. reduce。
6. llm。
7. gated_delta_net。
8. causal_conv1d。
9. creation。
10. copy。

如果启用 `LIBTENSOR_USE_CUBLAS` 或 `LIBTENSOR_USE_CUDNN`，同样在 fallback 后覆盖注册加速库 kernel。

## 7. 条件注册与能力边界

后端注册函数可以根据 dtype 和编译宏选择性注册。

例如 CUDA LLM：

1. `llm.softmax_last_dim` 只在 float dtype 下注册。
2. `llm.rms_norm`、`llm.rope` 当前只对 float32 / float64 注册。
3. `llm.flash_attention`、`llm.flash_attention_varlen` 只有启用 `LIBTENSOR_USE_FLASH_ATTN` 且 dtype 是 float16 / bfloat16 时注册。
4. `llm.embedding` 对传入 dtype 注册。

这种策略让“没有 kernel”成为正常状态。上层通过 `PreparedKernel::valid()` 或 `invoke_dispatch_kernel()` 的 bool 返回值判断是否可执行。

设计上，这比在 kernel 内部处理所有 unsupported case 更清楚：不支持的组合不注册，lookup 失败即可。

## 8. Lazy registration：首次 lookup 时补注册

op wrapper 不要求用户显式调用注册函数。`OperatorInitContext::prepare_kernel()` 的逻辑是：

```text
key = { op_name, device.type, dtype, layout }
fn = Dispatcher::lookup(key)
if fn == nullptr:
    ensure_backend_kernels_registered()
    fn = Dispatcher::lookup(key)
return PreparedKernel { key, fn, meta }
```

`ensure_backend_kernels_registered()` 根据 device type 调用：

1. CPU：`register_all_cpu_kernels()`。
2. CUDA：`register_all_cuda_kernels()`。

`invoke_dispatch_kernel()` 也有类似逻辑：先 lookup，失败后补注册，再 lookup。

这个设计减少了使用成本，但也带来一个特点：注册函数可能被多次调用。因为 `Dispatcher::register_kernel()` 是覆盖写入，多次注册不会改变最终语义，只是有额外开销。

如果未来 kernel 数量变大，可以考虑为 CPU / CUDA 各加一个 `std::once_flag`，避免重复批量注册。

## 9. 两种调用路径

libtensor 目前同时支持两种 kernel 调用方式：

### 9.1 直接 dispatch 路径

简单函数式 API 通常直接调用 `invoke_dispatch_kernel()`。

例如 unary：

```text
neg_out(x, out, ctx)
  -> unary_dispatch_out(..., "unary.neg", ctx)
       -> 校验输入输出
       -> invoke_dispatch_kernel("unary.neg", ctx, x.dtype(), inputs, outputs)
```

这种方式适合一次性调用，不需要长期保存 kernel。

### 9.2 PreparedOperator 路径

复杂 operator 会在 prepare 阶段缓存 `PreparedKernel`。

例如 MatmulOp：

```text
MatmulOp::prepare(batched, init_ctx)
  -> init_ctx.prepare_kernel("matmul.mm" 或 "matmul.bmm")
  -> bind_kernel(kernel)

MatmulOp::forward_out(a, b, out, ctx)
  -> 校验输入输出
  -> invoke(ctx, inputs, outputs)
```

`PreparedOperator` 保存 `PreparedKernel kernel_`，`bind_kernel()` 成功后把 operator 状态标为 Prepared。

这种方式适合模型推理：模型加载或 pipeline 初始化时 prepare 一次，之后每 token / 每 layer 复用 prepared kernel。

## 10. Wrapper 层负责语义校验

Dispatcher 只按 key 查函数，不理解 op 语义。因此 shape、dtype、device、contiguous、输出 shape 等检查主要在 op wrapper 中完成。

例如 matmul wrapper 检查：

1. 输入输出都 defined。
2. `a.dim() == 2`、`b.dim() == 2`、`out.dim() == 2`。
3. dtype 一致。
4. device 一致且等于 `ctx.device`。
5. `a.size(1) == b.size(0)`。
6. output shape 正确。
7. 三个 tensor 都 contiguous。

binary wrapper 检查 broadcast shape，attention wrapper 检查 q/k/v rank、head 数关系、cu_seqlens dtype 等。

这种设计让 backend kernel 可以假设输入已经基本合法，专注计算。但也要求新增 op 时不能只写 kernel，还必须写正确的 wrapper 校验。

## 11. 输出由调用方分配

多数 op 是 `*_out` 风格：调用方传入已经分配好的 output tensor，kernel 只写结果。

这与 Tensor 分配设计形成明确边界：

1. creation op 负责创建并填充 tensor。
2. 普通 compute op 不负责决定 output memory kind。
3. 模型 pipeline / execution plan 可以统一控制 workspace、persistent、KV cache 等内存用途。

例如 `zeros()` 的路径是先 `make_empty_like_shape()` 分配 output，再调度 `creation.zeros` kernel 填零。即使 dispatch kernel 不存在，也可以 fallback 到 `device_fill_zero()`。

## 12. KernelMeta 的作用

`KernelMeta` 当前只有一个字段：

```cpp
struct KernelMeta
{
    bool requires_dense = false;
};
```

CUDA unary 注册时会传入 dense meta。`dispatch_kernel_requires_dense()` 会查询 meta，用于上层判断某个 kernel 是否要求 dense/contiguous 输入。

当前 meta 还比较简单，但它是扩展调度策略的重要入口。后续可以加入：

1. 是否支持 in-place。
2. 是否支持 broadcast。
3. workspace 需求估计。
4. 是否 capture-safe。
5. 最小 dtype / alignment / shape 约束。

## 13. 加速库覆盖策略

注册顺序体现了 libtensor 的 fallback-first 策略。

以 CPU matmul 为例：

```text
register_cpu_matmul_kernels(dt)
  -> register matmul.mm naive
  -> register matmul.bmm naive

register_cpu_matmul_blas_kernels(dt)
  -> register matmul.mm BLAS
  -> register matmul.bmm BLAS
```

因为 key 相同，BLAS 会覆盖 naive。CUDA 的 cuBLAS、cuDNN 也是同样机制。

这个策略的优点：

1. fallback 永远存在，便于测试和最小构建。
2. 加速库接入不需要改上层 op API。
3. 编译宏决定最终注册哪个实现。

需要注意：覆盖是静默的。调试性能问题时，需要通过构建宏、日志或额外 registry introspection 确认最终使用的是 fallback 还是加速 kernel。

## 14. LLM kernel 与通用 kernel 的统一

LLM 相关 op 没有单独做一套 dispatcher，而是注册到同一个 `Dispatcher` 中，例如：

1. `llm.embedding`。
2. `llm.rms_norm`。
3. `llm.rope`。
4. `llm.flash_attention`。
5. `llm.flash_attention_varlen`。
6. `llm.gated_delta_net`。
7. `llm.causal_conv1d`。

这带来的好处是模型层可以用同一套 `OperatorInitContext` 和 `RuntimeContext` prepare / forward，不需要区分“通用算子 runtime”和“LLM runtime”。

例如 AttentionOp：

```text
AttentionOp::prepare(params, init_ctx)
  -> init_ctx.prepare_kernel("llm.flash_attention")
  -> bind_kernel(...)

AttentionOp::forward_out(q, k, v, out, ctx)
  -> KernelParams::make(params_)
  -> invoke(...)
```

这对 Qwen pipeline 这类模型执行层很重要：模型只需要组合 operator，而不需要知道某个 operator 最终落到 CPU loop、CUDA kernel 还是 FlashAttention。

## 15. RuntimeContext 与 OperatorInitContext 的分工

`OperatorInitContext` 用于 prepare 阶段，包含：

1. device。
2. dtype。
3. layout。
4. capture mode。

它回答“应该准备哪个 kernel”。

`RuntimeContext` 用于 forward 阶段，包含：

1. device。
2. stream。
3. workspace。
4. capture mode。
5. step index。

它回答“这次运行在哪个 stream、用哪个 workspace、处于哪个执行步骤”。

这种分工是合理的：prepare 阶段只绑定相对稳定的 kernel 选择维度，forward 阶段携带每次运行可能变化的上下文。

## 16. 当前设计的关键优点

### 16.1 调度核心很小

`Dispatcher` 本身只是一个线程安全 map，不掺杂 shape 推导、内存分配或后端策略。这让它容易理解和维护。

### 16.2 注册覆盖机制简单有效

fallback 与加速库使用同一个 key，后注册覆盖先注册。这个机制足以支持 BLAS / cuBLAS / cuDNN 这种“同语义更优实现”的场景。

### 16.3 prepare / forward 分离适合推理

LLM 推理中同一层同一 dtype/device 会重复执行很多次。`PreparedOperator` 缓存 `PreparedKernel`，避免重复 lookup，也为未来 CUDA graph capture、workspace planning 留出结构位置。

### 16.4 Tensor 分配与 kernel 调度解耦

kernel 接收已分配的 outputs，不决定 memory kind。这样模型层可以用 `Persistent`、`Workspace`、`KVCache` 控制内存生命周期，而 kernel 只负责计算。

## 17. 当前边界与风险

### 17.1 注册缺少 once 保护

lazy registration 每次 lookup 失败都可能调用 `register_all_*_kernels()`。由于注册是覆盖写入，语义上没问题，但会重复构造 key、写 map、加锁。

建议后续用 `std::once_flag` 或按后端的 atomic 标志保证批量注册只执行一次。

### 17.2 KernelKey 还不能表达更细的 specialization

当前 key 只有 op、device、dtype、layout。对于高性能 kernel，未来可能需要按更多维度选择：

1. compute capability。
2. tensor rank 或固定 head_dim。
3. matmul transpose/layout 组合。
4. quantization scheme。
5. causal / non-causal attention。
6. alignment 或 vector width。

现在这些差异只能在一个 KernelFn 内部继续分支。短期简单，长期可能影响性能调度。

### 17.3 KernelParams 不拥有数据

`KernelParams` 当前只借用外部参数地址。同步立即调用是安全的，但如果未来引入异步 graph 构建、延迟执行队列、跨线程调度，就可能悬空。

如果要支持这些场景，可以考虑：

1. 限制 KernelParams 只用于立即调用。
2. PreparedOperator 内保存 params 副本。
3. 引入小对象内联存储的 owning params。

### 17.4 错误反馈只有 bool

很多 wrapper 返回 `false`，但调用方无法区分是 shape 错、dtype 错、device 错还是 kernel 未注册。

这对测试和模型集成调试不够友好。后续可以在 debug build 或 runtime diagnostic 模式下提供失败原因。

### 17.5 meta 能力还偏弱

`KernelMeta` 当前只有 `requires_dense`。对于 planner 来说，未来还需要知道 workspace、capture safety、是否支持 varlen、是否支持 in-place 等信息。

如果 meta 扩展得足够完整，`ExecutionPlan` 可以更早地做 kernel 可用性检查和 workspace 规划。

## 18. 新增 kernel 的推荐流程

新增一个 op / kernel 时，推荐按以下步骤：

1. 定义 op name，遵守已有命名风格，例如 `category.name`。
2. 在 op wrapper 中实现输入输出校验。
3. 如果需要非 Tensor 参数，定义 params struct，并通过 `KernelParams::make()` 传递。
4. 实现 CPU fallback kernel。
5. 在 `register_cpu_*_kernels(DType dt)` 中注册支持的 dtype。
6. 如果有 CUDA 实现，在对应 `register_cuda_*_kernels(DType dt)` 中按 dtype / macro 条件注册。
7. 如果有加速库实现，在 fallback 后注册同 key 以覆盖。
8. 为 unsupported dtype 保持不注册，而不是注册后在 kernel 内失败。
9. 增加直接 dispatch 测试和 PreparedOperator 测试。

## 19. 总结

libtensor 的 kernel 注册调度设计采用了非常直接但有效的结构：用 `KernelKey` 做精确查表，用 `KernelFn` 统一 ABI，用 lazy registration 降低使用门槛，用 PreparedOperator 支持推理场景的预绑定，用注册覆盖接入加速库。

它当前最重要的价值在于把算子语义、后端实现、内存分配三者拆开：op wrapper 负责语义校验，Dispatcher 负责选择函数，allocator / MemoryKind 负责输出内存来源。后续如果要继续提升性能和可诊断性，优先方向应是 once 注册、扩展 KernelMeta、增强错误信息，以及让 KernelKey 支持更细粒度的高性能 specialization。
