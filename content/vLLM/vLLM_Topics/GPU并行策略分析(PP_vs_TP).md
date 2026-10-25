## 1. 多 GPU 并行部署

vLLM 支持的多 GPU 并行部署策略，主要有两种：

<font color="orange">「张量并行(Tensor Parallelism), TP」</font>: 把同一层中的某些张量及其计算拆分到多个 GPU 上，各自计算一部分，再通过通信合并结果。

> [!example] TP 示例
> 例如 TP = 4，通常是 4 张 GPU 各运行一个工作进程。对应 4 个 TP rank。每张 GPU 上复制的是部分张量和张量的分片。 


<font color="orange">流水线并行(Pipeline Parallelism), PP</font>: 把整个模型的层按深度切成几段，每张卡负责连续的若干层，数据像流水线一样从第一段流到最后一段，**只在段与段的边界传一次 activation**。

>两者的通信特征完全不同，这是选型的关键：
>- TP 的 AllReduce 是每层都发生、对带宽和延迟极其敏感的高频通信。它几乎必须依赖 NVLink（GPU 间高带宽直连）才能不被通信拖死。
>- PP 的 activation 传递是每段边界才发生一次、数据量小的低频通信，普通 PCIe 就能扛。

## 2. 关键约束

本项目是 3 x 2080ti，vLLM 要求 `num_attention_heads % tensor_parallel_size == 0`。

| 模型          | Q heads | TP=2 | TP=3 | TP=4 |
| ----------- | ------- | ---- | ---- | ---- |
| Qwen3-32B   | 64      | ✅    | ❌    | ✅    |
| Qwen3-14B   | 40      | ✅    | ❌    | ✅    |
| Qwen3-8B    | 32      | ✅    | ❌    | ✅    |
| Llama-3-70B | 64      | ✅    | ❌    | ✅    |

**没有任何主流模型的 attention heads 能被 3 整除**，所以 3 卡场景下只有 PP=3 可行。

## 3. PP=3

**head 整除约束**：Qwen3-32B（dense）的 attention query head 数是 64。TP=N 要求把 head 数均分到 N 张卡，64 不能被 3 整除（64/3≈21.3），切不开。PP 按"层"切（如 64 层切成 22/21/21），不受 head 整除限制。

**无 NVLink 的通信瓶颈**：当前**没有 NVLink**。无 NVLink 时 TP 的 AllReduce 只能走 PCIe，主板上的三槽是非对称的 x8/x8/x4，第三张卡还挂在 PCH 上跨 DMI——TP 在这种拓扑上会被通信严重拖累。PP 只传 activation，受这套拓扑影响很小。

vLLM 官方文档明确：**"无 NVLink 优先 PP"**

**代价**：PP 有 pipeline bubble，小 batch 延迟略高于 TP，但 continuous batching 在高并发下能填满流水线。


### 附：查阅

| ✅✅  |
| :-: |




