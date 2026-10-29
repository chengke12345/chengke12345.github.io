# 1. KV Cache 使用率

「KV Cache 使用率」指的是已经使用的 KV Cache 的显存容量占全部 KV Cache 容量的比例。

$$
KV\space Cache\space 使用率 = \frac{已占用\space KV\space Cache\space 容量}{KV\space Cache\space 缓存池总容量} \times 100\%
$$
KV Cache 缓存池总容量是推理引擎分配给 KV Cache 的总显存容量。

> [!example] KV Cache 使用率示例
> 假设 GPU 显存 80GB，其中 vLLM 分配给 KV Cache 缓存容量是 40GB，目前已使用 30GB，那么， KV Cache 的使用率就是 75%。

>- 注意⚠️：「KV Cache 使用率」和 「KV Cache 占用率」是同义词，是完全相同的。

具体统计口径取决于推理引擎。比如 vLLM V1 是按块计算，并且释放和可随时淘汰的前缀块都算作可用，即未被占用。

# 2. KV Cache 命中率

「KV Cache 命中率」是指 prompt 提交后，前缀 token 有多少能复用 Cache 中已有的 KV 向量。 
它的统计时点，是在 Prefill 计算前的调度阶段，确定 KV 命中。统计汇总按监控的时间窗口区分。

> 1. 请求的 Prompt 完成分词。
> 2. 调度器查找现有的 KV, 确定有多少连续的前缀 token 可以复用。
> 3. 记录「查询 token 数」和「命中 token 数」。再对剩余部分执行 prefill。

对于一次查询，假设输入 token 数是 $Q_{i}$，可复用的前缀 token 数为 $H_{i}$。那么该次查询的命中率为 

$$
单次查询 \space KV\space Cache\space 命中率 = \frac{H_{i}}{Q_{i}}
$$

运行一段时间后，或者一段时间内的整体命中率为：

$$
整体命中率 = \frac{\sum_iH_i}{\sum_iQ_i} \times 100\%
$$


> [!example] KV Cache 命中率示例
> 假设请求 A 有 1000 个 token, 查询下来命中了 800 个，请求 B 有 100 个 token, 查询下来 0 个命中。这样的整体命中率是 800 / (1000 + 200) = 72.7%。注意⚠️，不是把两个请求的命中率 80% 和 40% 求平均值 40%
> 监控时, vLLM 的 prometheus 指标 `prefix_cache_hits` 和 `prefix_cache_queries` 可以查看「累计数」。查看近 5 分钟的命中率，就是近 5 分钟的 「命中数的增量 / 查询数的增量」。
> 注意⚠️，prefix_cache_queries 是查询数，意思是输入 token 每个 token 都要去查是否命中了KV Cache,  因此查询数就是输入 token 的长度(特殊情况这里不考虑)。


> [!Warning] Decode 不计入前缀缓存命中率
> Decode 的时候，每一步都会读取当前请求已有的 KV。这种情况不计入前缀缓存命中率的计算。KV Cache 命中率指标是衡量 prefill 时能复用多少已有前缀计算。

# 3. GPU 显存使用率

显存使用率是指，当前显存使用量占总显存的百分比。在 vLLM 中，这是一个启动模型时，可设置的参数 `gpu_memory_utilization=0.9`。更准确的说是<font color="orange">「显存预算比例」</font>。

它让 vLLM 实例(模型实例) 按每张 GPU 总显存 90% 规划显存，其中包括<font color="orange">「模型权重」，「运行时激活与临时空间」，以及 KV Cache</font>。


> [!example] GPU 使用率设置
> 例如，一张 80GB 的卡，预算为 72GB。假设权重和运行时开销占 48GB，留给 KV Cache 的空间就约为 24GB，具体由启动时的内存分析决定。

注意⚠️：
- 参数配置 0.9：是规定的规划预算，并非驱动层，强制执行的硬隔离上限，也不保证不会 OOM
- 监控显示显存使用率为 0.9，表示当前已经占用了约 90%，本身不代表设置了上限。

这里的「GPU 显存使用率」，「GPU 显存占用率」，「GPU 显存利用率」是指一回事。
但是，有特别情况下，「GPU 显存利用率」是指采样时间内，显存被读写的时间占比。这种情况下，假设 80GB 的显存占用了 72 GB(90%), 但模型空闲时，显存读写利用率仍可能接近 0%。
# 4. 预分配机制

vLLM 采用的是显存「预分配」机制。启动 vLLM 时，如果设置了 `gpu_memory_utilization = 0.9`， vLLM 会申请 90% 的显存给 vLLM 先占着。

> 默认情况下，vLLM 在启动时会自动规划 KV Cache
> 1. 加载模型权重。
> 2. 通过试运行估算激活、临时缓冲等运行时开销。
> 3. 从总显存 90% 的预算中扣除这些开销，用剩余预算分配 KV Cache 缓存池。

$$
KV\space Cache\space 池容量 = (总显存 \times 0.9) - 权重 - 预估运行时开销
$$
<font color="orange">KV Cache 池在 vLLM 初始化时就分配显存，所以即使还没有请求，它也已经占据空间。后续请求主要是在池内分配，归还缓存块</font>。

启动后显存占用 90% 是常态，但并非必须等于 90%，预估的 0.9 峰值开销不一定持续存在，实际占用可以更低。内存估算偏差，额外运行时分配或其他进程，也可能使观测值超过 90%。

所以显存使用率 90%，KV Cache 使用率也可能接近 0% ，因为缓存池已经分配，但是里面还没有请求占用缓存块。

# 5. GPU 利用率

「GPU 利用率」与 「GPU 显存利用率」是完全不同的两回事。GPU 显存利用率是指容量占比，GPU 利用率是指计算占比。

「GPU 利用率」`GPU-util` 是指采样时间内，GPU 有计算内核正在执行计算的时间占比。GPU 利用率衡量的是忙碌时间，即使达到 100%，也不代表达到了 100% 的峰值算力。


> [!example] GPU 利用率
> 例如，模型的 KV Cache 缓存池已经分配好，显存占用 90%，但是没有请求运行，GPU 利用率也可能接近 0%.

