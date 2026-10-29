Preemption 抢占，是指调度器暂停一个尚未完成的推理请求，把它占用的资源释放给其他请求，之后再回复执行，这个现象叫做「Preemption 抢占」。
# 1. 抢占为什么发生

在 vLLM 中常见的一个场景是：KV Cache 空间不足，调度器暂定部分的请求并回收 KV blocks，让其他请求继续生成。
因为在生成 token 的过程中， KV Cache 会随着序列长度增长，请求刚进入时能放下，不代表后续生成的所有 token 都能放下。Paged Attention 按需分配块，也无法消除总容量限制。

> [!example] Preemption 示例
> 假设 KV Cache 池总共有 100 个块。A，B，C 三个请求，分别占用 30，30， 40个块。A继续生成需要申请一个新块，但池中已经没有空闲块了。调度器选择暂停 C，回收它的块，给A，B继续使用。等资源足够后，再回复 C。
> 因此，即使没有新请求进入，正在运行的请求，也有可能因为 KV Cache 的增长而发生抢占。

# 2.  被抢占请求处理方式

被抢占请求，有两种经典方式处理

| 方式             | 抢占时                            | 恢复时                           | 主要开销 |
| -------------- | ------------------------------ | ----------------------------- | ---- |
| Recompute, 重计算 | 释放 KV Cache, 保留 token 和请求状态    | 根据已有 token 重建 KV Cache, 再继续生成 | 重复计算 |
| Swap, 换出       | 将 KV Cache 搬到 CPU 内存，释放 GPU 的块 | 将 KV Cache 搬回 GPU，再继续生成       | 数据传输 |


这两个处理方案，在额外的「计算开销」和「CPU-GPU 传输开销」之间做取舍。vLLM V1 默认使用 Recompute。Swap 是其他系统常见的实现方案。

# 3. Recompute 重计算

Recompute 实现下，已生成的 token 会保留，恢复时，可以把 `原始输入token + 已生成token` 作为已知序列，通过 Prefill 计算重建 KV Cache, 然后接着生成后面的 token。

抢占的时候，输入的token + 已生成的 token 以整数 Toekn ID 列表的形式保存在 CPU 内存中。Token ID 列表是一串整数，表示输入和已经生成的内容。


> [!example] Recompute 示例
> 输入 1000 个 token, 已经生成 200 个 token。恢复时，框架利用保存的 1200 个 token ID, 重新执行 Prefill 计算，重建 KV Cache，然后继续往后生成第 201 个 token。

注意⚠️：Token ID 只是整数序列，体积很小。传输开销比 Swap 开销小很多，所以 vLLM V1 选择 Recompute。

被抢占的请求需要经历等待和恢复，流式输出可能出现明显卡顿，端到端延迟增加。频繁重计算会消耗大量 GPU 资源。

# 4. 出现抢占时的解决方式

出现抢占的时候，就是因为 KV Cache 空间不足导致的。解决方案主要有两种 「扩大 KV Cache 容量」和「降低并发度」。
- 「扩大 KV Cache 容量」：显存有余量时，就提高 `gpu_memory_utilization`, 让框架分配更多的 KV Cache 空间。
- 「降低并发度」：降低 `max_num_seqs`,  减少同时驻留 GPU ，占用 KV Cache 的请求数量。
这是 vLLM 官方给出的调优方向
此外，相同并发下，请求上下文越长，占用 KV Cache 越多。限制上下文长度，`max-model-len` 也是一个解决方案。