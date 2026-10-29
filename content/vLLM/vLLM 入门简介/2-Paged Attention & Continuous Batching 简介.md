vLLM 是一个开源的大模型推理框架。它与 HuggingFace Transformer 相比，主要有两方面区别。
- <font color="orange">① 它不仅能把大模型跑起来，还实现了服务层面的事情，提供了API。</font>
- <font color="orange">② 大模型推理期间的一些技术优化，以更小的算力成本，获得更快的加速效果， 实现了比 HuggingFace Transformer </font><font color="red">高14 - 24 倍吞吐量</font>。

vLLM 最核心的贡献是提出了 PagedAttention 加速策略，能够更有效的管理 attention 计算过程中缓存的张量。本质就是提升了模型在推理过程中，对 GPU 显存的利用效率。以前的方式，比如 HuggingFace Transformer 在推理的时候，会造成大量的显存碎片。当我们需要去做大矩阵计算的时候，会发现显存被切得非常碎，无法提供矩阵计算需要的一块完整内存/显存，导致等待, 只有等别人释放之后再合并，这样就会拖慢计算的效率。

vLLM 框架最核心的机制就是，<font color="#eb4349">「Continuous Batching」</font> 和 <font color="#eb4349">「Paged Attention」</font>。

# 1. Continous Batching

## `◼︎`static batching & dynamic batching

vLLM 在处理的时候，是一个批次一个批次进行处理的。传统的 static batching, 需要等待，例如4个 request 一个批次，那么需要凑齐 4 个 request 组成一个完整的 batch, 才会交给 LLM 处理，但是一开始的那个 request，就会等待较长时间。

![[Pasted image 20260603133509.png]]

然后就出现了 Dynamic Batching, 我们可以设置一个时间窗口，让一个 rquest 等待一个时间窗口，超过这个时间窗口，已经到达的请求就会作为一个 batch, 送给 LLM 进行处理。

此外，一个批次中的 request 是并行处理的，必须等到最长的 request 被处理完后，这个 batch 所占用的资源，才会被统一释放。
## `◼︎`传统批处理机制浪费极大

传统的 KV Cache 处理方式，对显存的利用率不高。对于企业级的应用，可能同一时间，需要进行 batch process, 批处理多个提示词请求。大模型接受批量输入，但批量输入的长度是不一样的，一个batch中，每个请求完成的步骤差异很大，每个输出的结束时间也是不一致的。

传统的 KV Cache, 会给每个请求预分配一个显存空间，供该请求使用。在同一批次中，不同的request / prompt 长度不同，但是我们必须按照 context window 的最大长度，为每一个 request 预分配 KV Cache。在批处理的过程中，要求 KV Cache 的长度是对齐的。这样的方式，计算不会出问题，但是有大量的显存浪费。 真实的显存利用率只有 20%～30%。所以

>- 模型接受批量输入。即多个 request 组成一个 batch 输入到模型中。
>- batch 中的 每个 request / prompt 长度不一致。按最长上下文 context window 为每一个 request 分配 KV Cache 缓存空间。这样批处理过程中，KV Cache 的长度是对齐的。
>- 一个 batch 中所有的 request 完成计算以后，才能统一释放显存。 

这是批处理 batching 早期的朴素实现。不管是 「static batching」还是 「dynamic batching」, 只是组成一个 batch 的方式不同，前者按个数，后者按时间。执行批处理时，都是上面的规则。即
<font color="orange">已经生成答案的请求要等待所有请求完成后，才能统一释放显存，这样 GPU 就会未充分利用。同一batch中的任务，释放资源是同时释放的。</font>

![[Pasted image 20260602162814.png]]

上图中，有四个不同任务的时序，黄色部分是输入，蓝色部分是推理。右边是推理结束时的情况。输入 token 长度不同，需要的时间不同，推理输出的 token 长度不同，输出需要时间也不同。

比如，第一个任务，T1～T3 都在输入，T4 开始输出，T6完成，第二个任务T1～T2都在输入，T3开始输出，T8完成…以此类推。

所以第二个任务需要时间是最长的，只有等到它完成了推理，这个 Batch 中的显存才会被释放。所以右图中的白色方框，就是显存闲置的时候。
## `◼︎`Continuous Batching  

解决方案就是，<u>一旦一个 Batch 中的某个 seq 完成生成，就会发射一个 end-of-seq token，就可以在其他位置插入新的 seq 继续生成 token, 从而达到更高的GPU利用率</u>。

![[Pasted image 20260602163201.png]]

不管是 static batching 还是 dynamic batching。我们都是在一个 batch 中所有的 request 都处理完之后，才能释放资源，然后处理第二个 batch。这就导致可能前一个 batch 中某个 request 处理时间很长，而其他 request 处理完成后，必须要等待这个长 request 处理完之后，才能一起释放资源。这个过程中会产生大量的等待时间，所以显存的利用率不高。

「Continous Batching」，与之前不同，在 batch 处理的时候，某个 request 处理完成，会发出一个END 信号，接受到END信号以后，系统就会立即调度处理下一个 batch 中的 request, 这个过程没有等待，提高了显存利用率。所以，<font color="orange">「Continuous Batching 是一个调度策略/机制」</font>

> [!info] Continuous Batching
>continuous batching，我们可以换一个角度来看待。它相当于是一直在处理一个批次，这个batch就没有停过，当一个 request 到来的时候，它会加入队列，一旦batch中有了空闲，就立即加入这个batch。所以这个 batch 启动之后，一直往前，有新的request，就加入batch。batch 持续不断的处理，所以叫做 continous batching。

# 2. KV Cache: Paged Attention
## `◼︎`Traditional KV Cache

传统的推理过程，我们需要显卡，显卡里面分为两部分，一部分是计算资源，这是真正做计算的，另一部分就是显存。KV Cache 就是要把 token 的 K V 向量缓存到显卡的显存里面，避免重复计算 token 的 K V 向量

推理时，每一个 token 的生成，都要用到之前所有 token 的 K, V 向量，包括新生成的 token。为了避免重复计算，我们需要将前面 token 的 K, V 向量做显存的缓存，这样可以避免每次都要把前面的token 通过与 Transform 矩阵做矩阵乘法，计算 K, V 两个向量，这就是 KV Cache。

KV Cache 受限于缓存大小，它是<font color="red">「以空间换时间的典型策略」</font>。
注意⚠️，<font color="orange">是否使用 KV Cache 并不影响 attention 计算的结果，只会影响计算的速度。</font>

KV Cache 需要的显存可以被计算出来，它和模型参数有关，包括，模型有多少层，即深度，还有模型中使用的向量的维数有关。

$$ 2 \times 2 \times b \times s \times d_{model} \times n_{layers} $$
b 是 batch size, 就是一个 batch 的大小。s 是序列的长度。 d 是向量维度，n 模型里面self-attention 有多少层。假设用半精度存储，即 FP16。所以一个数值占用 2 字节，K V 是2个值，所以最前面乘以 2。

<font color="orange">注意，这是推理出首个 token 需要的缓存大小. 即一个 batch 的 sequence 输入大模型后产生的 KV Cache 大小。</font>

这里是为了计算方便，假设是 static batching 的方式，一个 batch 中的 sequence 长度一致。但实际上，现在 vLLM 采取的内存管理方式是 paged Attention。

随着新 token 不断被推理出来，每次推理出来一个新 token 还需要增加缓存(增加也要求batch内sequence对齐)。每增加一个 token，缓存容量需要增加：

$$ 2\times2\times b \times 1\times d_{model} \times n_{layers} $$

## `◼︎` Paged Attention

Paged Attention 是一种优化策略，本身不涉及任何计算。传统 K V Cache 缓存过程中，K, V 张量非常大。显存的占用是动态的，取决于输入序列的长度，由于碎片化和过度预留，系统的60%-80%都是被浪费掉的。

![[Pasted image 20260603130711.png\|600]]

「Paged Attention」是 vLLM 在计算 self-attention 时采用的 KV Cache 策略。Paged Attention 思路源自操作系统<font color="orange">「虚拟内存」</font>和<font color="orange">「分页思想」</font>,<font color="deeppink">允许在<u>非连续</u>空间内存储<u>连续</u>的KV张量。</font>

### <font color="lightblue">基本思想</font>

>- <font color="#45ce6e">逻辑分块</font>: <font color="orange">Paged Attention 把每个序列的 K V 向量的缓存进行了分块，每块中包含固定长度 token 的 K V 向量，在计算 attention 的时候可以高效的检索并获取这些块。</font>
>- <font color="#45ce6e">物理分块</font>：<font color="orange">GPU 显存也被划分成同样大小的块(block)。每个 block 对应于同样数量 token 的 K V 向量 。block 中的 token 数量是预先设定好的。每个block中的存储单位叫做 slot, 一个slot中可以存放若干token 的 k, v 向量。</font>
>- <font color="#45ce6e">按块缓存</font>：<font color="orange">缓存 prompt 的 K, V 时，它和传统 KV Cache 进行预分配的机制不同，它是按需求，按 block 进行分配。每个逻辑上划分的 block，都可以一一对应到显存上的一个物理 block 中, 他们的对应关系就存放在「块表 block table」中</font>
>- <font color="#45ce6e">检索缓存</font>：<font color="orange">每个 request, 可以通过 block table，找到它的 cache 分散到了哪些物理 block 上，block table 就是逻辑块号到物理块号的映射表。每个固定长度的块可以看成虚拟内存中的页，token 可以看作字节，序列可以看作进程。那么通过一个块表就可以将逻辑块映射到非连续的物理块上，而物理块可以根据新生成的 token 按需分配。</font>

![[Pasted image 20260602173254.png]]

上面，一个 Block 对应于一个页。一个 Block 中可以存放固定数量 token 的 K V，一个序列像一个进程一样，需要多个 Block 缓存。

以前，一个序列的 K V 缓存需要一大块连续空间，并且在推理阶段，可能进行显存的拷贝与释放，比如产生一个Token之后，预留显存不够了，需要重新分配一个更大的空间将原来的 K V 缓存拷贝过去，再在后面增加第 n+1 个 token 的 KV, 供后面的推理过程使用。内存的使用是一个动态分配的过程，这就导致了碎片化的问题。

现在，Paged Attention 将序列需要的 KV 空间(逻辑)切割成 Block。同样将，物理内存也按相同大小切割成若干block。然后在中间维护一个<font color="orange">「逻辑块到物理块的块映射表(Block table)」</font>, 如上图：

<font color="deeppink">映射表的从上往下的自然索引顺序就是序列的逻辑分块的编号，右边表项，physical block no, 表示其对应的物理显存中的块编号，Filled Slots, 表示这个块当前已经使用了多少个 slot。一个 slot 中可以包含一个或多个 KV</font>。

比如：第一行表示，该 request 序列, 逻辑划分的第 0 号 block 对应于物理显存中的第几号block，这个block当前使用了几个slot。这样我们在使用的时候，就可以快速索引，但是在存储的时候，可以灵活存储。


> [!NOTE] 每个 request 都有自己的 block table
> 一个 request 生成一条序列时，每个 request 都有自己的 block table, 覆盖 prompt + 已生成 tokens 的 KV Cache. 
> 一条 prompt 生成多条序列，或者不同的 prompt 在 prefix caching 命中相同前缀时，他们有各自独立的映射表(一个输入输出序列对应一张映射表)，但是它们可以<font color="orange">共享物理 KV 块</font>。




假设现在输入的 prompt 是 “Alan Turing is a computer scientist”，并且一个单词的 KV 向量占用一个 slot (也可以是多个 token 的 KV 向量占用一个 slot，这里使用一个 token 的 KV 向量占用一个 slot 是为了说明方便)

![[Pasted image 20260602215604.png]]

逻辑上它如上图左边，将 KV Cache 从前到后按顺序和固定大小划分成若干 block，但是实际存储到显存中时，它并不是要使用一个连续的大块物理显存来存储缓存，它是有空就可以用。上图，实际存储是从物理第7块开始存，即先使用 block 7, 再接着使用物理 block 1.

极端情况下，假设物理偶数 block 都被占用。那么以前的方法，碎片导致不能分配一个连续两个block 大小的显存空间，只能等待显存释放合并后再分配。但是，分页方式就让只要有空闲的 block 都可以拿来使用，这让显存碎片问题得到解决。

上面逻辑第 0 号 block 映射到物理第 7 号 block，并且占用了 4 个slot。逻辑第 1 号 block 映射到物理第 1 号 block, 并且已占用 2 个 slot. <font color="orange">我们要还原整个 KV Cache，只需要根据块映射表，将物理内存中的块拼接起来</font>。

![[Pasted image 20260602220135.png]]

如果产生一个新 token 的 K V, 物理 1 号 block还未填满，所以它继续使用后面的显存来缓存新产生的 KV。所以 1 号 block 当前使用为3 (填满之后为4)。通过这种方式，就可以最大化利用物理显存。

![[Pasted image 20260602220237.png]]

当一个Block填满之后，就会使用一个新的block，新 block 并不需要与之前的任何 block 连续存储

![[Pasted image 20260602220336.png]]

所以动态分配显存的颗粒度是 block，而不是要按整个 sequence 的长度，来分配连续显存空间。

![[Pasted image 20260602220432.png]]

隐藏优势：有可能进行<font color="orange">「高效内存共享」</font>。比如，我们需要同一个 prompt，多次生成输出序列，以进行并行采样时，这个 prompt 的 KV 向量计算和缓存，可以在多个输出序列之间共享。注意，高效共享的是原始 prompt 的 KV 向量, 后面生成的部分，每个输出不一样，不能共享。

![[Pasted image 20260602220902.png]]

这种高效内存共享，也出现在「Cache Hit」 上面。常发生在 AI Agent中，Agent的每次访问大模型，前面很长的一部分 system prompt 都是相同的，所以前面都是一样的 K，V，所以可以在缓存中命中，「KV Cache Hit」. 缓存命中在收费上，各大语言模型只要 1/10 的价格。 详细可以参考 [[KV Cache]] 那篇笔记。

vLLM最核心的就是 「continuous batching」 和 「paged attention」, 有了这两个机制，推理引擎的速度就会有很大提升。