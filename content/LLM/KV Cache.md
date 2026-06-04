语言模型整个生成的过程是这样的。首先有一个来自人类的 prompt，语言模型就会去做文字接龙。接出的下一个字，接在prompt后面成为新的输入prompt.
![[Pasted image 20260324210721.png]]
整个过程可以看作分成两段，前半段是一个输入非常长的 sequence， 这个过程叫做<font color="#45ce6e">Prefill, 预填充</font>。后半段一个一个的Token输出，这个过程叫做 <font color="#45ce6e">Decode</font>

## <font color="grape">prefill</font>

在 prefill 的时候，一次进来了非常大量的token，每一个Token我们都要一次算出它的Q,K,V 三个向量。下面假设输入有三个 Token，这三个 Token 分别算出它们的 Q, K, V，接下来，我们就会拿这些 Q, K, V去算 attention。
![[Pasted image 20260324211743.png]]
q1 和 k1 算 attention weight, 然后乘以v1的到o1。q2去和k1, k2算出 2个attention weight, 然后对前面两个v1, v2 做 weighted sum 的到 o2。q3 也做同样的操作，算出三个 Attention Weight，对 v1 到 v3 做 weighted sum, 得到o3。
<font color="red"><u>从 q1, q2, q3 算出 o1, o2, o3, 这是三件事，可以做平行计算的。</u></font>

## <font color="grape">KV Cache的概念</font>

由于sequence前面token的 k 和 v, 后续推理每一个token都会用到，所以推理过程中对 k, v向量的读取次数是 $O(n^2)$, 所以为了挑高效率，这些 v 和 k 计算出来之后会被缓存下来(这里是 k1~k3, v1~v3)，放在显存HBM中。但是，我们会丢掉 q，因为 q 只用一次，计算自己和前面token的attention值。
<font color="#45ce6e">把 v 和 k 在HBM中缓存存下来这件事，就叫做 </font>「<font color="#b48ff4">KV Cache</font>」。

假设现在模型生成出来了第4个Token。第四个Token会被当做新的输入，所以，我们现在得到了q4, k4, v4。如果我们需要把第4个Token, 和前面的3个Token再合在一起重新丢给LLM，再重新算这些 token 的 v 和 k 的话，就太浪费时间了。所以，我们会把 k 跟 v 都存下来，就不用再算第二次了。

![[Pasted image 20260325003447.png]]

所以 q4 直接去跟之前存下来的 k1-k3 算 attention, 也跟 k4 自己算 attention。然后得到 4 个 attention weight。然后把它们跟 v1-v4 做 weighted sum。得到下一个输出 o4。这样就可以<font color="#eb4349">节省输入token后，计算 v1-v3, k1-k3 需要的时间(要把输入乘上一个 Matrix 才能把 k 和 v 计算出来)</font>。所以，我们会把已经算出来的k 和 v 存下来，以留待日后使用。

![[Pasted image 20260325003942.png]]

之后再进来第五个token。虽然前面有 4 个 token，但是它们的 V 和 K 不需要再重新计算了。第五个Token进来只需要算自己的 q5, k5, v5 , 然后再去计算q5与 k1-k4 以及和 k5 自己的 attention值，再和 v1-v5 做 weighted sum 就可以得到o5了。它就不需要再算前面已经算出来的 k1-k4 和 v1-v4 了。这就是 KV Cache 的概念。

## <font color="grape">KVCache撑爆显存</font>

KV Cache 的概念和思想非常简单，但是在实现上，可能会遇到巨大的问题。之前我们说GPU的时候，我们考虑的是工作台(SRAM)很小，数据不够放。但是，KV Cache 这个东西，是会撑爆我们以为的非常大的仓库的(HBM显存)。

![[Pasted image 20260325005429.png]]
- 首先，我们要存的 k 和 v 非常多，每次产生一个Token 或 每次输入一个Token，就需要增加缓存一组 k 和 v。如果输入或输出的sequence很长，那就会撑爆我们的仓库(HBM显存)。
- 另一方面，一个token 的 k 和 v 不是只有一组而已，我们通常会有多组的QKV，这就是<font color="#45ce6e">多头注意力机制, Multi-head Attention</font>。一个Token，我们会有很多个 query，每一个Query 都有自己对应的 Value 和 对应的 Key。我们通常都会做 Multi-head 的 query。  
上图实际上，是同一个token有 4 个head 的 multi-head attention. 这个 Token 对应了 4 个 query。因此就有 4 组 k 和 v。
## <font color="grape">多头注意力机制 Multi-head Attention</font>

我们在研究的时候，都是认为输入一个sequence, 首先转化为token ID，然后 token ID 会根据 embedding table，转化为 token embedding，即 embedding 向量。embedding 向量在进入第一层的 attention layer 的时候，embedding 会乘以 3 个参数矩阵 $M_q, M_k, M_v$ 计算得到 Q, K, V。  

一般我们研究的时候，认为一个 token embedding 生成一组 Q, K, V。实际的LLM系统中，一个token的 Q, K, V 不只一组，可能会有很多组。一个token embedding 会有多个query，每个query对应自己的 value 和 key. 通过同一个token embedding 乘以一组不同的矩阵，就可以得到不同的q , k, v。token的每一个 query 在各自的组内，跟自己组内的 k, v 计算自己的attention, 以及输出向量O，最后再综合考虑。这就是<font color="#45ce6e">多头注意力机制, Multi-head Attention</font>。

之所以使用这样的机制，是因为<font color="#45ce6e">一个token与其他 token之间的 atttention 可能不只一个方面</font>，比如，"外形相似程度", "颜色相似程度"，“是不是同一个种类”，“是否是同一领域的名词” 等等。一个head只能代表一个方面， 如果我们要考虑一个token和其他token之间多方面的attention关系，就要使用多头注意力机制。

一个embedding通过与不同的参数矩阵 $M_q, M_k, M_v$ 相乘就会得到多个头的Q,K,V。有多少组  $M_q, M_k, M_v$，就会有多少个头head。这一组一组的  $M_q, M_k, M_v$ 就是模型训练出来的参数。现在的LLM 一般会有 64 个 heads或甚至 128个 heads。

## <font color="grape">KVCache占用显存的计算</font>

我们 Gemma 2 为例
![[Pasted image 20260325124658.png\|400]]

我们看 27B 模型。<font color="deeppink">Layers</font> 说明它有46个Layer。<font color="deeppink">Head  type</font> 说明它用了一个叫做 GQA，Group-Query Attetion 的东西。<font color="deeppink">Num heads</font> 说明它的 Attention 用的是 32 个 head 的 multi-head attention。<font color="deeppink">Head size</font> 说明每一个head, 每一个QKV向量有128维。

每产生一个token，要存一组 key, 一组 value，需要消耗多少显存？

> [!NOTE] 每个 Token 的 KV Cache 占用量
>- <font color="lightblue">在一个头中有两个向量  key, value，每个128 维， 一共要存 2 x 128 个数值。</font>
>- <font color="lightblue">一共有32个头的attention, 就有 32 x 2 x 128。</font>
>- <font color="lightblue">一共有46 层 Layers, 每一层的输出，是下一层的输入，所以输入向量的QKV在每一层都不一样，所以要新占用显存。46 x 32 x 2 x 128。</font>
>- <font color="lightblue">每个数值，假设我们选用 FP16 精度，也就是 2个字节。一共需要 46 x 32 x 2 x 128 x 2 = 753644 bytes(736KB， 0.72MB)</font>

![[Pasted image 20260325124753.png\|342]]![[Pasted image 20260325125304.png\|342]]

假设我们现在用的是 A100，80GB Memory，也只够我们存114K tokens。而现在，我们对 Context 的需求是远超 10 万的。所以，当模型输入的 sequence 或输出的 sequence 太长的时候，就会 <font color="#45ce6e">CUDA Out of Memory</font>
为了避免显存溢出，就有了各式各样的发明。

## <font color="grape">Multi-query-Attention</font>

实现上，<font color="orange">我们都会使用 Multi-head Attention</font>。一个token 会对应多个头, 多个query，每个 query 都会对应一组 k, v 向量。有多少head, 就有多少组 k, v 向量。这样太占用显存空间了。所以提出了 [Multi-query attention].

![[Pasted image 20260325131148.png]]
我们一样，一个Token可以有多个query，但所有的query 公用同一组 value 和 key<font color="grass">这里的共用是指，所有的query共用一组 k,v 在显存中的缓存</font>。这样可以减少占用空间。
<font color="orange">实际上，这种方法的效果不好。</font>

## <font color="grape">Group-query-Attention</font>

由于Multi-head attention 的效果并不好，所以提出了一个介于 Multi-head attention 和 Multi-query Attention 之间的方法，叫做 <font color="#45ce6e">Group-query Attention</font>。这就是上面表格中的 <font color="#45ce6e">GQA</font> 。

![[Pasted image 20260325132206.png]]

它的想法是，我们在显存中，存放多组 Value 和 Key. 但是每一组 Value 和 Key 对应的是多个 query。也就是把多个query放在一个Group里面，共享使用一组 Key，Value。上面的例子就是每一组 Value 和 Key, 会对应两个 query。
<font color="orange">这是一个非常广泛的使用方法，Llama, Gemma, etc. 这些知名的模型里面，都是使用的 Group-query attention。</font>

## <font color="grape">Muti-head-Latent-Attention, MLA</font>
## <font color="grape">MLA压缩向量算法</font>

这种方法的思想是，我们有一大堆的 Value 和 Key(所有head). 我们可以把它们全部压缩成一个向量，这个向量是更低维的向量。缓存在仓库(显存)的时候，就只存这个向量就好。

![[Pasted image 20260325132906.png]]

一般我们在算Value 和 Key 的时候，就是把一个输入的 token 向量x，乘以不同的Transformation矩阵，得到不同的 Value 和 Key。我们可以把 x 到多组的 Value 跟 Key 中间，放一个 Bottleneck 的 Layer。

![[Pasted image 20260325133723.png]]
即输入 x , 乘上一个 Transformation矩阵以后，得到一个Dimension比较小的向量
(下面桃红色的向量)。然后再用这个较小的向量乘以不同的Transform,   得到不同的 Value 和 Key。
<font color="orange">这个方法是需要训练模型的，训练模型的时候，一开始就要教模型，把 x 压成比较低维度的向量。然后再从这个低维度的向量乘上不同的Transformation, 变成 query 和 key。这就是deepseek模型里面用到的方式。</font>
把 x 乘以一个矩阵变成较小维度(桃红色)向量时，以及把这个压缩向量乘以不同矩阵，得到不同的 key 和 value 的时候。中间乘的这些矩阵，都是训练模型之后得到的参数。

> [!NOTE] 向量的压缩与解压缩
> <font color="orange">原本 x 向量我们经过不同组($W_q, W_k, W_v$)的参数矩阵，可以得到多个head的Q, K, V向量。现在我们经过不同训练的参数，让 x 经过参数矩阵，得到的(粉红色)是一个包含了所有head的所有 k, v 的压缩向量。这个压缩向量再经过另外一些参数矩阵，就可以解压缩，变回原来的 k, v 向量 </font><font color="#45ce6e">所以，我们只需要缓存这个压缩向量就好了，由于它是更低维度的向量，所以它所占用的内存空间就更小了</font>

整体来讲，位置 j 的 token $x_j$ 会经过Transform形成两个压缩向量，$C_j^{KV}， C_j^Q$。 $C_j^{KV}$压缩了该 token 的所有head的 k, v向量，通过另一个transform解压缩，就可以得到全部head的全部kv向量。
$C_j^Q$ 压缩了该 token 所有 head 的 Q向量，通过另一个tranform矩阵解压缩，就可以得到全部head的全部q向量。

解压缩以后再做 attention 计算。

所以，MLA(Multi-head Latent Attention) 实际上用的是<font color="orange">两个独立的压缩向量</font>，而不是一个：
- $c_j^{KV}=W_{DKV} \cdot x_j$  → 恢复出所有 head 的 K 和 V
- $c_j^Q = W_{DQ} \cdot x_j$​ → 恢复出所有 head 的 Q

Q 和 KV 是分开压缩的，不是从同一个压缩向量里恢复出 Q、K、V 三者。

##### <font color="lightblue">分开压缩的原因</font>
KV 压缩的核心目的是减少 KV Cache占用的内存量。推理时，之前所有 token 的 $c_j^{KV}$ 要缓存下来，所以它的维度要尽可能小。而当前 token 的 Q 是即算即用、不需要缓存的，所以 Q 的压缩维度可以和 KV 不同，各自选择最优的压缩比。
##### <font color="lightblue">完整流程</font>
对每个 token $x_j$​，产生两个压缩向量，分别解压出 Q 和 KV，然后正常做 multi-head attention。推理时只缓存 $c^{KV}$，不需要缓存 $c^{Q}$。

![[Pasted image 20260326154446.png]]

上图中的一个紫红色框，就是一个 token 的 k v 压缩向量，它压缩的是这个 token 所有 head 的 k v 向量。正常情况下，在生成第五个 Token 时，要把前面四个token的压缩向量解压缩，然后分别与第五个向量的query做attention。注意，每个头分开做自己的 attention 计算。

例如，上面假设有4个token，现在模型要产生第 5 个 token，前4个token的压缩向量如上粉红色小框所示。它们解压缩后是所有 head 的 KV 向量，假设有4个head。 因此，每个token的压缩向量解压缩出来应该是 4 组k v(一个蓝色代表一个value和一个橘色代表一个key, 这样是一组 k v)。每组 k v 去和对应的 query 做 attention。因为有4个head, query也有4个。

## <font color="grape">MLA 不解压缩的算法</font>

MLA(Multi-head Latent Attention)，可以不需要解压缩，就计算出attention。
![[Pasted image 20260326162127.png]]
我们假设压缩向量为 c, 解压缩变成 key 的时候, 就是把 c 乘上一个$W_k$，变成 k。不同的key 和 不同的 value, 其实就是乘上不同的向量。然后我们把 k 和 q 做dot product，得到 attention,  a。
$$
\begin{align}
a = q \cdot k = q^Tk=q^TW_kc\\
=(W_k^Tq)^Tc =(W_k^Tq)\cdot c
\end{align}
$$

经过这样的数学变换。本来要算的是 $W_k$ 和 c 相乘，把 c 解压缩，再跟 q 做 dot product。但是这件事可以等同于，用   $W_k^T$ 与 q 相乘，得到 q', 用 q' 去和 c 做 dot product。
这相当于是在压缩的维度下去做 dot product。<font color="orange">与其解压缩以后跟q做 dot product，不如让 q 做压缩，得到q', 用 q' 去和 c 做 dot product. 直接在压缩的情况下做 dot product，它们是等价的。</font>

把 q 压缩的过程，只需要做一次。<font color="#45ce6e">压缩的计算相对解压缩小很多，至少它跟sequence的长度无关，因为解压缩的话要把前面每个token的压缩向量都要解压缩以后再做attention。压缩就把q压缩一次，然后直接和压缩向量c做attention，然后做normalization. 然后就可以得到 attention 值</font>，$\hat{a}$ 。

![[Pasted image 20260326163109.png]]

得到 $\hat{a}$ 之后，我们需要做 weighted sum。这时候需要用到 v 向量，如果解压缩的话，我们会把每一个 c 乘以一个 $W_v$ 得到一个 v。所以，c1->v1, c2->v2，c3-> v3, c4->v4。

![[Pasted image 20260326163556.png]]

然后我们做 weighted sum

![[Pasted image 20260326163613.png]]
所以 weighted sum 不需要做在解压缩后的 v 上面，可以直接做在压缩过的 c 上面。
![[Pasted image 20260326164200.png]]

所以我们只需要在压缩的 c 上面用attention 做 weighted sum，计算出来之后，然后乘上 $W_v$ 解压缩一次就得到结果。我们不需要对 有 sequence 长度的 每个c，都去做解压缩。<font color="orange">先做 weighted sum，再解压缩一次，就可以得到结果。 </font>

## <font color="grape">Sliding-Window-Attention</font>

它的核心思想是，每次在做attention的时候，不要 attend 整个sequence，这样要存放太多的 K 和 V。我们只 attend 前面一个window的范围就好，比如只attend 4096范围。这样我们知道在存 k v cache的时候，有一个固定的上限。

![[Pasted image 20260326191157.png]]

如果这样，query可以attend的范围变少了，这样模型是不是没法考虑非常长的sequence，这个问题没那么严重。现在的 network 都是多层的。上面图中每一行序列代表其中一层attention。
最上面一层，只看前面两个key。这两个key是由query和前一层的k, v算出来的，它又看了更前面的两个Key。所以，如果network的深度够深，Sliding Window attention 可以看的范围是非常大的。
这个方法曾被用于 Mistral 7B 的某一版本中。
##### <font color="lightblue">混合方式</font>
我们开可以有些 layer 用 Sliding Window Attention, 有些 Layer 做完整的 Attention. 这样需要缓存的 k v 就变少了。在 GPT OSS 里面就是一层 Sliding Window，一层完整 Attention。 

## <font color="grape">streamingLLM</font>

sliding window attention 往往会让模型的表现变差，尤其在输入的sequence非常长的时候。但是，如果attention的范围能够包含sequence最开始的几个token的话，就没事了。这个方法甚至不需要做额外的training。在训练的时候，并不要特别教LLM要去看前面几个token。只要在 Inference 的时候，在sliding window, 额外加上最前面的几个token。就可以让 sliding window 的表现大增。

![[Pasted image 20260326214132.png]]

下面两个图是一些模型的实验情况。横轴是输入的sequence长度，纵轴是perplexity. perlexity这个数值是越小越好。所以纵轴数值越大，代表LLM的表现越差。这个图是假设模型训练的时候，采用了固定的 sliding attention window，但是在做inference的时候，采用了不同的策略。
蓝色表示inference时，采用 dense attention, 就是完整的 attention. 它的输入sequence超过训练时窗口的长度之后，模型效果就会迅速恶化。
黄色是window Attention, 就是在 inference 的时候，采用sliding window attention，即推理时，只看一个固定的window长度，但是输入的sequence只要超过这个要看的window长度的时候，模型效果也会迅速恶化。
红色是 StreamingLLM，只要考虑了输入sequence的前几个 token，模型的表现就会好很多，而且可以处理非常长的输入，这些输入的长度即使是在训练的时候没有看过的，模型也有办法处理。

##### <font color="lightblue">是否attend第一个token影响很大</font>
现在的 Language Model 非常喜欢 attend 前面几个token，因为前面几个 token 几乎不需要做attention。attention的机制是强制性的，所有的 attention weight 合起来必须是1。所以对于每个query而言无论如何都需要 attend 一些东西，如果没什么好attention的情况呢？模型预设了一个行为，假设没什么好attend的，就attend在第一个token上。如果没有第一个token，即只考虑window的长度，没有考虑到第一个token，它的世界就会崩坏， 它的表现就会非常差。所以，在做attention的时候，有没有给模型看第一个token，对它的影响非常大。

## <font color="grape">Pruning-KV-Cache</font>

我们可以直接把没用的 kv 从仓库里面丢掉。论文发现，我们其实没必要存那么多的 k 和 v，多数的 k 和 v 后来都是没有被用上的。所以，我们可以做 [KV Cache prunning]， 把没有用到的 k 和 v 直接从仓库里面移除，就当它不存在。

![[Pasted image 20260326220943.png]]

上图是三个attention weight 权重图，分别显示在 178 的位置处，228 的位置，278 的位置，分别检查 attention 的情况。所以它是在这个位置，往前看，看看前面的 attend 怎么样。横轴的每一行，代表一个attention head, 上面就是展示了5个 attention head的情况。颜色越深的地方，就代表在这里有越大的 attention weight. 

从图中看出，结果只有非常少的token，有被attend，大部份的token, 从来没有被attend到。所以，存放这些 k, v 就是在占用浪费显存而已，因为这些 k 和 v 根本没有被用上。有少数token，是反复被attend 到的，这些 k 和 v 就应该被存下来。所以，这个算法的重点是，寻找哪一些 k 和 v 应该要被留下来。
基本思想是，假设有一组 k 和 v , 一直没有被attend到，它就会自己消失不见。仓库里有一个东西，一直没有人去拿，过几天就被丢掉了。

## <font color="grape">跨对话的Cache/ Cache Hit</font>

刚才讨论的 KV Cache, 它 Cache 的 K 和 V 是同一个对话里面的Key 和 Value。其实这个K V Cache，也有可能是跨对话的。假设模型生成了一个句子“大家好我是大金”。把它的 k v 缓存了下来。

![[Pasted image 20260326223803.png]]

假设现在有另一个人去 prompt 了模型，输入的是"大家好我是小金"。这两句话的前 5 个token其实是一模一样的。我们存了前5个token的kv cache。所以可以把前5个 k v 挪给第二句话，直接使用。事实上，前五个kv, 它们是一模一样的。
当然，“大” 的 kv, 和 “小” 的kv就不一样了。所以第6个位置处，就不能直接挪。
但是，上面一句话的金，也不能挪到下面使用。虽然这两个token一样，但是每个token会算出来的representation，都是取决于前面看过什么样的token。一旦前面的token有变化，比如"大" 变成 “小”, 我们后面得到的 k 和 v 就会不一样了.
所以，一个sequence的某位置有变，后面的token，就没办法再share key 和 value了。<font color="orange">两个不同的sequence，只有一样的前缀部分，可以共用 key  和 value。</font>

今天大多语言模型，比如 ChatGpt, Claude, Gemini。都有一个折扣的方案，对于ChatGPT，short context 短文本， 一百万token，cost \$2.5，长文本 Long context 一百万Token，\$5 。如果有 input cache，只收十分之一的钱。因为如果cache到input，对语言模型而言，几乎不需要做额外的计算，key 和 value 都是已经算好的，在显存中缓存的。Open AI官网上，还画了下面的图，解释Cache的原理

![[Pasted image 20260326224528.png]]

左边的图，是原来存在Cache里面的 sequence， 假设有一个新的 sequence(右上)，它只有最后两个token 跟原来的sequence不一样，前面的token就会cache hit，这些Token的计算它就会给你打一折。
但是同样的一个sequence(右下)，只是在这个sequence前面多了一个不一样的token，最前面的三角形。这样改变了这个sequence的前缀，就破坏了整个cache的结果。这种情况，就cache miss，就没办法打折了。

在真实的场景下，<font color="#eb4349">如果是AI Agent，就非常有可能是前缀一样</font>。

![[Pasted image 20260326225233.png]]

当我们对我们的AI Agent 说 “你好”的时候，它会在我们的句子前面加上非常长的 System Prompt，再丢给语言模型。<font color="#eb4349">system prompt 虽然很长，但它的内容通常是不变的，它代表的可能就是这个agent的名字，灵魂，存在的目标，它是一个长时间固定的东西，所以system prompt 就很有可能用上 cache hit.</font> <font color="orange">我们每次丢给语言模型的sequence，它前面的prefix，可能非常大部分都是一样的。这样就可以用上折扣</font>

![[Pasted image 20260326230430.png]]
所以我们在摆放system prompt时，里面的内容其实是有讲究的。<font color="#45ce6e"><u>越稳定不变的内容应该放在越前面。越可能变动的内容，应该放在越后面。</u></font>OpenClaw里面system prompt 的摆放顺序，其实是把这件事考虑进去的。比如有哪些工具可以使用，以及其使用说明就可以放在前面。而日期时间，就应该在system prompt中放在后面。OpenClaw是很认真考虑过这个事情的，OpenClaw在github上有一个issue，[]https://github.com/openclaw/openclaw/issues/27732 。很多人在讨论如何安排 system prompt 才是最有效率的方式。

同样输入的prompt，用不同写法，可能会得到不同程度的 cache hit。
![[Pasted image 20260326231132.png]]
这两个问题，现在cache hit 没有很大，只hit了"帮我定从" 这几个字。只要地点一换，cache hit 就没有用了，你的前缀就变了，后面的部分就没有得到折扣了。如果换一个写法。
![[Pasted image 20260326231421.png]]
我们让句子前面都一样，前面都是帮我定从 x 到 y 的班机，然后把变数写到sequence的最后面。这样就可能 hit 较长的cache，节省比较多的钱。
这是设计AI Agent 的时候，确实需要考虑的事。

总结：
![[Pasted image 20260326232057.png]]


> [!NOTE]  KV Cache 一般过程总结
> - <font color="grape">[prefill]</font>
>   <font color="pink">用户输入Prompt(假设有 n 个token)，一次并行送入模型。假设模型有L层深度的Layer。从第1层到第 L 层全部算完。每层都会有 n 个 k, n 个 v。各自填入 KV Cache。最后在第L层输出的最后一个向量经过 LM Head 得到第一个生成的token</font>
> - <font color="grape">[decode]</font>
>   <font color="pink">每次只有一个新生成的token，作为输入，从第 1 层，走到第 L 层。每一层做的事，就是对这个新 token, 算出它的q, k, v，把新的 k , v 添加(append)到这一层的kv cache里。用 q 对缓存中的所有 k 做 attention weights，再跟所有的 v 做weighted sum, 得到新的输出向量。传给下一层，走完 L 层之后，产生下一个生成的Token。然后重复这个过程。</font>

这个过程中有几点特别值得关注：
- <font color="#45ce6e">假设大模型是有 L 层深度的 network。每一层都是自己独立的 KV Cache，只能用于本层的attention计算。它们是按层独立维护的，第 n 层缓存的 KV 对第 n+1 层没用。</font>
- <font color="#45ce6e">每一层的 KV Cache 都会被缓存在显存中，每生成一个token, 每一层的KV Cache 都会新增新token的kv缓存起来。所以，每层的KV Cache 是独立维护，逐步增长(线性增长)。</font>
- <font color="#45ce6e">长序列生成时显存占用会持续增长——每层的缓存都在随 token 数线性增长整个生成过程中，每层的 KV Cache 一直在增长，直到生成结束。生成结束后，所有层的 KV Cache 才会释放。</font>
