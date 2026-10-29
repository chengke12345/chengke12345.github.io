# 1.  Decoder-Only 架构

<font color="orange">「Decoder-only Architecture」</font>：2017年 Google 推出的经典架构 Transformer, 它原本是 Encoder- Decoder 架构。但是现在我们市面上的使用的大模型，90%都是采用的 Decoder-Only 的架构。它把原始架构里面的 Encoder-Decoder 架构去掉了 Encoder 部分，只保留了 Decoder 部分的架构。因为 Decoder-only 这样的架构更适合生成式。

LLM 的工作原理如下，它的本质就是 <font color="#eb4349">「Predict Next Token」</font>. 每一刻都在输出下一刻的预测词。

![[Pasted image 20260603093205.png]]

我们把大模型 LLM 一个时刻一个时刻生成新 token 的过程，叫做<font color="#eb4349">「大语言模型的推理,Inference」。</font>

![[Pasted image 20260603093352.png]]

推理的过程，我们可以划分为两个阶段：
- 第一个阶段叫做 <font color="orange">「Pre-filling，预填充」</font>。Pre-filling 阶段，整个提示词会一起送入LLM，然后输出一个token。对 LLM而言，<u><b>Pre-filling 是一个 N 对 1 的阶段</b></u>。
- 第二个阶段叫做 <font color="orange">「Decoding, 解码」</font>。对 LLM 而言，<u><b>Decoding 是一个 1 对 1 的阶段</b></u>。

<font color="#eb4349">所以，推理分为两个阶段，Pre-filling(N-to-1), Decoding(1-to-1)。</font>

# 2. 大模型推理

大模型的推理就是<font color="orange">「使用大模型」</font>。

### <font color="#b48ff4">STEP 1: Tokenizer</font>

使用大模型我们要提供一个输入或者问题。在把句子交给大模型之前，首先要做 tokenize。tokenizer 首先会对 prompt 进行tokenizing.

![[Pasted image 20260603094300.png]]

tokenizer 将原始文本切割成一个个「词元， token」，然后将每个 token 映射到一个整数 token_id 上去。大模型只认识这个整数 token_id。在大模型的词库中，每个 token，都有一个对应的整数 token_id。这个过程中文可以叫做<font color="orange">「分词」</font>。也就是把一个句子分成一个一个token。负责这个工作的模块叫做 <font color="orange">「分词器，tokenizer」</font>

### <font color="#b48ff4">STEP 2: Scheduling</font>

<font color="orange">Scheduling, 调度</font>。prompt 在 tokenize 之后, 请求发送给服务器，服务器进行调度 schedule，分配 KV Cache 内存块，并将输入序列调度到下一个 prefill-batch 的队列中，排队等待 prefill 阶段处理。注意⚠️，此时没有进行向量化，向量化是在 Scheduling 之后，Prefill 之前才进行的步骤。

![[Pasted image 20260603095957.png]]

### <font color="#b48ff4">STEP 3: Prefill</font>

这个过程把所有 tokenize 以后的 input tokens，并行一次正向传播 forward pass。即，一起经过模型的 n 层神经网络。同时，把 keys 和 values 写入 kv cahce。

![[Pasted image 20260603100215.png]]

注意⚠️：prompt 的 tokens，在完成 tokenize，进入模型的第一步 foward，就是 Embedding, 把token_ids 完成向量化。Embedding 发生在 Prefill 阶段，是模型 forward 的第一步，在 scheduling 之后。

### <font color="#b48ff4">STEP 4: Decode/Output</font>

![[Pasted image 20260603101008.png]]

注意⚠️ ：生成内容的第一个 token 是来自 Prefill 阶段，Prefill 的输出就是第一个 output 的 token。除了第一个 token，后面产生的 token 都来自 decode 阶段。

所以，大语言模型的推理，是由<font color="#eb4349">「1次 Prefill + N 次 Decode 构成的」</font>。 在此之前，输入还有一个 tokenize 和 scheduling 的过程。

大模型每次向前传播只生成一个词，它是一个词一个词往外蹦，每生成一个词就流回客户端，这种方式就叫做 <font color="#eb4349">streaming, 流式输出</font>。

总结：
![[Pasted image 20260603101237.png]]

我们加载模型的时候，通常是加载两个东西，一个是 tokenizer, 分词器。一个是模型参数权重。Decode 过程完成以后，输出是 token id，通过 id 再进行 Detokenization, 经过解码之后才能正常打印字符串。

从用户输入 prompt，到产生第一个 First Ouput Token，等待的时间叫做 <font color="#eb4349"> 「Time to First Token(TTFT)」</font>。事实上，TTFT 就跟 Prefill 有关，与 Decode 是无关的。

推理速度除了与 TTFT 相关，还与后续 Decode 的过程也相关，优化的时候，要看瓶颈在哪里

![[Pasted image 20260603101429.png]]

Decode 阶段就是 Prefill 给出 T0 之后，要产生，T1，T2，T3…..。Decode 阶段，每产生一个token，需要等待的时长，我们称为 <font color="orange">ITL(Inter Token Latency)</font>。如果要优化 ITL，就需要看 Decode阶段进行了哪些计算，涉及了哪些资源，才能优化它。注意，Detokenization 的过程，发生在每一个 token 产生之后，然后立即输出字符或字符串，而不是等到最后一个 token 产生之后才进行。

![[Pasted image 20260603101524.png]]

左边的 iteration 1 是 Prefill 阶段，右边的 iteration 2, 3, 4 就是 Decode 阶段。相同意思的一个图示如下：
![[Pasted image 20260603101548.png]]

注意区分两个术语，
- <font color="#eb4349">TTFT(Time to First Token), Prefill 阶段产生第一个输出 token 需要的时间。</font>
- <font color="#eb4349">TTOT(Time to Output Token), Decode阶段产生一个Token需要的时间。</font>

<font color="orange">如果要实现一个推理引擎(inference engine)，一定是需要把这两个阶段分别实现的。</font>

Ollama 更多的侧重于个人使用层面，而 vLLM 等企业级的推理引擎，更看重的是并行批处理的能力。更多的是部署在服务器上，让各种客户端可以同时访问。

# 3. 推理的核心过程

![[Pasted image 20260603102253.png]]

先看 Prefill 的部分，4 代表 prompt中有 4 个token, 每个token对应一个向量，64 代表一个 token 对应的 embedding 是 64 维向量。

$Q\times K^T \times V$ 是计算 attention 的时候，每一个 token 的 query，要和它之前每一个 token 的 key 做点乘，也就是和 K 向量的转置相乘，得到一排 attention 值。再用每个 attention 值和相应 token 的 value，V向量做 weighted sum。最后得到结果。

Prefill 阶段，是不需要从 Cache Memory 里面读取缓存的，但是需要往 Cache memory 里面，写入 K 和 V 向量。如上所示的橘黄色部分，这个 Cache Memory 就是在 scheduling 阶段，分配好的。 这个 Cache memory 对于提速来说非常重要。如果没有 Cache Memory, 就意味着在decode阶段，每计算一个 token，都要把之前的所有 token 的 KV 向量都要重新计算一次。