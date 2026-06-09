## <font color="grape">attention scores / Logits</font>

self-attention 中，第 n 个 token 的 query 要和之前 token 的 k 做 $Q \times K^T$ 的计算，得到 $(a_1, a_2...a_{n-1}，a_n)$ n 个 attention score，注意力分数，这 n 个注意力分数也叫做 Logits。
<font color="orange">Logits 反应的这个位置上的token，和前面每个token的原始相似度分数。</font>

## <font color="grape">attention weights</font>

得到 Logits 之后，我们需要对Logits做归一化操作，Normalization, 使其转化为一个概率分布。
这个概率分布所表达的现实意义是，这个位置的token, 对前面各个key关注度的分布或注意力占比。

比如，现在是第 4 个token，先对前面4个token(包括自己)做attention计算，得到Logits，然后做归一化操作，得到概率分布 `[0, 0.85, 0.1, 0.05]`, 它表示的意思就是 “往前回头看，我把注意力几乎全押在第 2 个 token 上，第 3 个瞄一眼，第 1 个直接忽略。”

第一，那个 `0` 严格说不是真的 0，是 softmax 出来一个极小的数（比如 0.001），感性上当成"几乎不看"就行。真正的硬 0 只发生在 causal mask 把后面的 token 屏蔽掉的时候。

第二，"关注度高"不等于"语义相似"或"重要"。它只是说在当前这个 query 的投影空间里，第 2 个 token 的 key 跟我的 query 最对得上。同一个 token，换一个注意力头，可能就把 0.9 押到别人身上了——每个头关注的"那种关系"不一样（有的头盯语法主谓、有的盯指代、有的盯距离）。

归一化操作之后，得到的概率分布，就是 attention weights，它表示的就是，"我对前面各个 token 的关注度分配表"。

## <font color="grape">softmax</font>

前面提到的归一化操作，Normalization。我们用得最多的就是 softmax。

> [!NOTE] softmax 算法描述
> softmax 将每一个 Logit 取 exponential 作为分子，将所有 Logits 取 exponential之和作为分母，这样就得到了归一化之后的各个 attention weights。实际计算过程中，每个 Logit 在取exoponential 的时候，都会减去最大的Logit，这是为了防止计算过程中的溢出，因为取expoenntial 的值可能会非常大。所以，softmax 归一化公式为：
> $$
> \hat{a_i} = \frac{e^{a_i-a_{max}}}{\sum_{i=1}^{n}{e^{a_i-a_{max}}}}
> $$
> $$
> a_i = \frac{Q \times {K^T}}{\sqrt{d_k}}
> $$ 

在做softmax前，我们通常要对 attention score 除以 $\sqrt{d_k}$, $d_k$ 是每个head里，Q 和 K 向量的长度或者维度。这样做的目的，是为了让方差回归到约为1，这样不至于让softmax一下子就饱和，直接输出类似于`[0，0，0，1，0，0]`这样的结果。

做完 softmax 之后，我们得到的概率分布，就是 attention weights。attention weights 是一个 query token 对**前面所有 token 的关注程度分布**。它回答的是:"我这个位置在聚合信息时,应该从哪些历史 token 那里、各取多少比例?"

#### <font color="#b48ff4">非线性放大</font>

softmax 在工程实践上的效果很好。因为softmax 会用 exp 把 Logit 之间的差距<font color="orange">非线性放大</font>。exp让大的 Logit 在归一化之后的占比更大，但是如果模型给出的 Logit 是错的，这个错误也会被放大。
softmax本身不负责找出赢家，它只是忠实的归一化函数。分布是尖锐还是平坦，完全取决于Logits值本身，softmax 只是会把它们之间的差距放大。对于本来就很接近的 Logits， softmax 放大后出来的概率分布也会很接近，softmax不会无中生有制造确定性。

softmax 是个忠实的归一化器,它保留并(通过 exp)适度放大输入的相对格局,但不负责"制造"一个明确赢家。分布尖不尖锐由 logits 和 temperature 决定;真要挑赢家是 argmax 的活。

尖锐不尖锐由输入 logits 的差距决定,softmax 只是忠实地把它们映射成概率分布,而 exp 带来非线性。当 logits 差距大时,exp 确实拉得更开(看起来像放大)，但当 logits 差距小时,exp 反而让它们更接近(像压缩)。原因是 exp 放大的是**差值**,而 softmax 真正关心的是 logit 之间的**差**。差大就拉得开,差小就拉不开。softmax 通过 exp,把 logits 之间的"差"指数化地转成概率比。

**softmax 通过 exp,把 logits 之间的"差"指数化地转成概率比。** 两个 logit 差 1,它们的概率比就固定是 e¹≈2.7 倍,无论它们绝对值是多少。差 2 就是 e²≈7.4 倍。

所以与其说"放大",不如说 softmax 用一个**固定的指数尺度**去解释 logits 的差距——这个尺度既能在差距大时显得"放大",也能在差距小时显得"压平"。而能直接调这个尺度的旋钮就是 **temperature**:除以一个大的 T,等于把所有差都缩小,分布变平;除以小的 T,差被放大,分布变尖。

一句话修正:softmax 不是"放大原分布",而是"按指数尺度把 logits 的差转成概率比"——表现为放大还是压缩,取决于差本身有多大。

