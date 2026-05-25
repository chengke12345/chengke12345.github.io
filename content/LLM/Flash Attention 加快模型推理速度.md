
我们训练好一个语言模型以后，使用它来做推理，这个过程叫做 <font color="#45ce6e">inference</font>。

# <font color="grape">简述语言模型内部工作原理 </font>
#token #Transformer #self-attention

语言模型就是文字接龙，给一个句子，它去预测接下来产生哪一个 <font color="#b48ff4">token</font>。今天主流类神经网络的架构就是<font color="#b48ff4">Transformer</font>. Transformer 里面有很多层的Layer，每层 Layer 中都有一个机制，叫<font color="#b48ff4">self-attention</font>。让 Tranformer 可以考虑整个输入sequence的所有信息。

![[Pasted image 20260324132940.png]]

self-attention, 输入一排向量 x1~x5, 输出一排向量 o1 ~ o5.
self-attention,首先把 x1~x5 各自乘以三个不同的矩阵，每个得到三个向量，q，k，v。 q 是 query, k 是 key, v 是 value。这个过程叫做，从 x 向量变成 Q K V 的过程。接下来以 x4 为例:

![[Pasted image 20260324134205.png]]

把 x4 的 query, q4 和它前面所有的 key, k1 ~ k4，做 dot product，内积， 记为 a1 ~ a4。他们值可能是负无穷到正无穷，所以对他们做softmax操作, 进行规则化normalization, 让它变成数值介于 0～1 之间的 $\hat{a_1}...\hat{a_4}$ ，它们的和为1。
然后$\hat{a_1}...\hat{a_4}$ 各自乘上 v 向量，做 weighted sum，最后得到 attention layer 在第4 个位置的输出 o4。

#### <font color="orange">加速Transformer计算的方法</font>

每一个加速 Transformer 计算的方法，都要关注其代价。有一些方法的代价是，它会改变 self-attention 的计算，也就是它不是真的计算 self-attention, 它是一个approximation。有的方法是模型绑定的，要训练特定的模型，对模型做一定的定制化, 这种方法不是随插即用的方法。

#### <font color="orange">Flash Attention的特点</font>
#FlashAttention特点

它不会改变原有Attention的计算结果，它不是一个近似 approximation。
它可以直接套用在任何以Transformer为基础的，使用 Attention 的模型上，它并没有绑定任何模型，是一个即插即用的方法。它的代价非常小。
Flash Attention 之所以可以达到加速的效果，它的核心思想是它考虑了GPU运算的底层逻辑。

#### <font color="orange">GPU显存 HBM与GDDR</font>
#HBM #GDDR

HBM，High Bandwidth Memory，与 GDDR 一样，都是用于显存的技术。但是它们侧重点不一样。HBM 追求<font color="orange">极致的带宽和能效</font>，GDDR追求<font color="orange">极致的性价比和通用性</font>。

#### <font color="orange">GPU运算底层逻辑</font> 
#GPU运算原理 #ExecutionUnit  #SRAM #HBM #工作台

在 GPU 里面，有一堆 Execution Unit, 它们负责做计算。这个 Execution Unit，在 GPU里面有多个，不止一个，它们可以同时执行任务。我们把它比作<font color="#b48ff4">小精灵</font>
它们的优点是数量多，所以他们的运算非常快速。它们的弱点是工作台太小了，它们的工作台能存放的数据是有限的。如果有一个大矩阵或者大向量，我们没法把它摆上工作台，每次工作台上都只能放非常少量的数据来进行计算。这个工作台就是<font color="#b48ff4">S-RAM</font>，或者更精确的说，是 <font color="#b48ff4">on-chip 的 SRAM</font>。我们称为<font color="#b48ff4">工作台</font>
如果GPU有大量数据，这些数据会被放在一个仓库里面，叫做 <font color="#b48ff4">HBM (显存), High Bandwidth Memory</font>。仓库大小是有限的，但是相较于工作台，仓库非常大，可以放非常多的内容。

![[Pasted image 20260324143612.png]]

真正做运算时，这些小精灵会去仓库里面把数据搬到工作台上，它只能搬正好可以放上工作台的数据，然后高速运算。完成后，把运算结果搬回仓库。所以，只要数据上了工作台，就可以做高速运算，在工作台上发生的一切，都可以想象成瞬间完成的。真正麻烦是<font color="orange">“搬数据是很花时间的”。</font>
<font color="red">搬数据，是拖慢运算的瓶颈。</font>

<font color="#45ce6e">Flash Attention 的思想就是，能不能改变 self-attention 计算的方法，结果一样，只是改变一些计算顺序，减少搬运数据的次数，这样让整体运算更快一点。</font>

# <font color="grape">一般attention的计算过程</font>
#chunk #Normalize #softmax算法思路 #softmax算法公式

query和key都是放在仓库中的。query 要和 key 做 dot product。实际运算过程中，GPU 可以同时处理多个query。下面我们只考虑一个query, 但是有多个query的情况可以直接类推。

![[Pasted image 20260324145723.png]]

我们不能一次把所有的 key 都放到工作台上，因为这些 key 太多了。key的数量就是我们要处理的 sequence 的长度。如果把 LLM当做 AI Agent 来使用，它的输入往往非常长。 我们把一整排 key 的长度假设为 L， 代表现在输入的 sequence 的长度。
我们要把这一整排key, 切成一个一个的 <font color="#b48ff4">chunk</font>， 假设每个 chunk 有 N 个 key 。chunk 的大小取决于工作台的大小。一个 chunk 中的 N 个 key，以及 query 被放到工作台上，然后一瞬间，就完成了query 和 所有 key 的 dot product。算完后，再把这些 dot product 的结果数值，放回到仓库里面， 工作台清除空间。

![[Pasted image 20260324145818.png\|330]] ![[Pasted image 20260324145859.png\|330]]

然后是第二个 chunk 的 key, 送到工作台与 q 做 dot product。然后是第三个chunk的key, 以此类推。一次可以把一个chunk里面，所有key跟query的dot product计算出来。 
输出的每一个黑色小框，我们称为 $a_i$, 它是一个 attention 的 weight ($q_i\times {k^T_i}$)
我们真正想要的，是接下来做过 normalization 以后的attention weight, 记为$\hat{a_i}$.

![[Pasted image 20260324151712.png]]

#softmax算法思路 

>nomalize的计算，是把每一个 $a_i$ 取 exponential 作为分子，把所有 $a_i$ 的exponential 加起来作为分母。得到 nomalize 的 $\hat{a_i}$。这是理论上的计算方法，实际上我们还会在每一个 $a_i$ 上减去这一排 $a_i$ 里面最大的一个 dot product(weight) 值 $a_{max}$ 。这样 e 的指数项最大数值是 0，因为 $a_i$ 可以很大，如果它很大，计算 $e^{a_i}$ 的时候，在 Exponential 指数项就会放大成一个很大的数值，计算完就可能overflow。所以用 $a_i$ 减掉 $a_{max}$，确保指数项最大的数值就是0. 这是为了防止计算overflow。

#softmax算法公式
所以 Normalize 计算的softmax算法公式为：
$$
\hat{a_i} = \frac{e^{a_i-a_{max}}}{\sum_{i=1}^{l}{e^{a_i-a_{max}}}}
$$

![[Pasted image 20260324152809.png]]
在把 $a_i$ normalize 转化为 $\hat{a_i}$ 时，我们不可以把所有 $a_i$ 一次放入工作台，然后完成计算以后，又一次搬回仓库。这样是不行的。
<font color="#45ce6e">工作台上不能放任何跟sequence长度 L 有关的东西，虽然这个 a 是scalar，一个一个的标量，但是实际上 L 可能是10万，100万。就算是简单数值，也无法放在工作台上</font>。

首先，我们要找出 $a_{max}$. 由于工作台只有一个 chunk 那么大，所以，我们在工作台上只能看到一个chunk 的 $a_i$, 所以

![[Pasted image 20260324153135.png]]

我们一次，先 Load 一个 chunk 大小里面的 $a_i$, 找出第一个chunk里面的最大 a, 假设最大的数值时$d_1$, 把这个数值存在工作台上的 d 空间处。d 中存放的就是当前这个chunk 里最大的 a。

然后把 a 中第二个chunk 读入工作台，比较这些 a 与 d 谁大，找出最大值 $d_2$ 放入 d 空间。

![[Pasted image 20260324153831.png]]
总共要做 B = L / N 次, 即分了多少个 chunk， 就要做多少次。做完之后，d 中存放的值就是 $a_{max}$
然后，我们再求分子，对 $a_i$ 再逐个以 chunk 为单位计算，得到分子 $a^{a_i-d}$，称为 $a_i$'。
得到分子之后，我们要算分母。再次按 chunk 为单位进行求和就行了。

![[Pasted image 20260324154505.png]]

分子分母求都出来之后，我们就可以算出真正的 $\hat{a_i}$。

![[Pasted image 20260324154743.png]]

然后把 $\hat{a_i}$ 去和 value 做 weighted sum。同样，每次只能处理一个chunk。

![[Pasted image 20260324155110.png]]
经过 B 次 weighted sum, 我们得到 $O_B$,  这个$O_B$ 就是最终 attention layer 在某一个位置上的输出。这就是 Q 跟 K 跟 V 做 attention 以后最终得到的结果。
<font color="orange">在整个过程中，我们对仓库HBM，做了很多次读写操作，访存次数非常多。这就是推理性能的瓶颈。</font>

# <font color="grape">Flash Attention 简化版 / softmax 简化版</font>
#FlashAttention简化版 #softmax简化版 #FlashAttention核心技巧

上面self-attention的计算方式，多次读写仓库(显存HBM)。<font color="#45ce6e">Flash Attention 的思想就是要减少这样对仓库的读写次数。</font>从上面的过程中，我们可以看出，从 $a_i$ 到 $\hat{a_i}$ 要读好几次仓库。
其实，$a_{max}$ 与 分母 $\sum_{i=1}^{L}e^{a_i-a_{max}}$ 可以一次被找出来。

![[Pasted image 20260324161253.png]]

我们先算第一个chunk, chunk 里最大的 $a_i$ 为 $d_1$ , 我们假设它就是 $a_{max}$。我们得到一个分母, 求和式为 $s = s_1 = \sum_{i=1}^{N}e^{a_i-d_1}$

然后进入下一个 chunk，下一个chunk里会有上一个 chunk 留下来的 d 和 s。
假设我们在 chunk2 里面找到了一个更大的 $a_i$ 数值叫 $d_2$。现在我们应该把 $d_2$ 当做$a_{max}$。然后用 $d_2$ 求分母再和 $s_1$ 加起来。

![[Pasted image 20260324161344.png]]

前面是假设 $d_1$ 是 $a_{max}$ 现在又发现 $d_2$ 更大，因此前面的假设是错的。
所以这里就做了一个调整。做了这个调整就好像什么事都没有发生，把 $d_1$ 当做$a_{max}$ 的痕迹直接抹掉，变成把  $d_2$ 当做 $a_{max}$ 。

这个调整可行，是因为如果把 $S_1$ 带进去展开就可以得到
$$
\begin{align}
s_1 = \sum_{i=1}^{N}e^{a_i-d_1}(e^{d_1-d_2}) = \sum_{i=1}^{N}e^{a_i-d_2}
\\s_2 = s_1 + \sum_{i=N+1}^{2N}e^{a_i-d_2} = \sum_{i=1}^{2N}e^{a_i-d_2}
\end{align}
$$
这样就把 s1 修正为把 $d_2$当做 $a_{max}$了。
<font color="#eb4349">这就是 Flash Attention 的一个核心技巧。</font> #FlashAttention核心技巧

接下来的 chunk 都是相同的操作
![[Pasted image 20260324163058.png]]
最终得到的 $d_B$ 就是 $a_{max}$ 。现在，存放在 S 中 的数值 $S_B$ 就是我们最终要得到的分母那一项。

有了分母和$a_{max}$ , 我们就可以分chunk，求出每一个 $\hat{a_i}$
![[Pasted image 20260324163636.png]]
![[Pasted image 20260324163545.png]]

所以我们经过上面一番操作后，把从 $a_i$ 到 $\hat{a_i}$ 需要读写多次仓库，改为了只需要读写两次，这个两次说的是，每个 $a_i$ 到 $\hat{a_i}$ 都只需要把 $a_i$ 从仓库搬到工作台2次。一次求 $a_{max}$ 和分母，一次计算成 $\hat{a_i}$ 。
![[Pasted image 20260324164313.png]]

# <font color="grape">Flash Attention完整版 / softmax 完整版 </font>
 #FlashAttention完整版 #softmax的完整实现 #缺失AttentionWeight
 
但这并不是Flash Attention的全部，Flash Attention 最终有一个灵魂拷问。<font color="orange">我们真的需要计算出每一个attention weight, 才能计算最后的weighted sum吗？</font>
它跳过了把 $\hat{a}$ 真正算出来的这个步骤，直接得到最后的O。所以，用 Flash Attention，我们读不出来 attention weight。

它的做法是一步到位。K, Q 跟 V 通通放到这个工作台上以后，做一次运算就得到最终结果。

![[Pasted image 20260324165329.png]]

前面操作没有变化。q 要和一个chunk里面的 每个 k 做 dot product, 得到 $a_i$, 把最大的设为 $d_1$放在 d 中。同时，用 $d_1$ 算出一个分母 $S_1$ 放在 S 中。
这时候，把 v 在这个时间点就读进来。计算 weighted sum。
直接用向量 $v_i$ 乘以 $e^{a_i-d_1}$ 除以 $S_1$ ，得到一个输出向量 $O_1$ , 这相当于做了一次局部的 weighted sum。但这不对，因为 d 只是这个chunk中的局部最大值，所以这个 weight 根本不是我们要的attention的weight, $\hat{a}$ 的值。<font color="orange">一会想办法修补这个问题。</font>

下一个chunk, chunk2上，做同样的操作。d2 和 S2 分别是当前最大的 a 值 和 正确的分母。所以做如下对 O1 的调整，就可以把 d1 和 S1 错误抹掉
![[Pasted image 20260324171620.png]]

然后我们得到的 O2 就是当前局部的输出向量。我们真正要得到的是最终的O。这个过程反复进行下去。所以，最终在最后一个 B chunk以后，我们就得到我们真正想要的O。这就是 Flash Attention 的精神 
![[Pasted image 20260324172033.png]]

d 中的 $d_B$ 就是真正的 $a_{max}$, S 里面就是真正正确的分母，不过这两项，我们都不关心，真正的关心的是最终输出的 O 向量。这是这一层self-attention layer 在这个位置上产生的一个向量输出。最终的O向量是正确的，但是整个过程中，真正正确的 $\hat{a}$ 从来没有被算出来。但是我们最终得到了正确的weighted sum的结果，这就是 Flash Attention。

> [!NOTE] 总结
> <font color="#eb4349">总结：Flash Attention 是对softmax过程的加速优化</font>。







