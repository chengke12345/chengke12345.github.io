所有的算子都是抽象的概念层或者说是一种逻辑规格，它们都要绑定具体的实现。但是这种规格与实现的分离，对于普通用户是透明的。即普通用户只需要调用算子，而不需要关心算子真正使用的实现，以及底层使用什么样的 kernel 实现。
但是vLLM里面有些算子比较特殊，其中就有 attention 算子。

## <font color="grape">抽象算子与正式后端</font>

attention 算子是 vLLM 里面有正式后端的抽象算子。
所有的算子都是抽象层的逻辑规格，都是抽象的。但是一般的算子这层抽象，用户是没有感知的，直接调用就可以，算子内部dispatcher会自动选用哪个实现。但是attention算子不一样，用户能感知到它使用的是不同的算子实现，FlashAttention, FlashInfer, XFormers等等。还可以由第三方提供新的算子实现。所以 attention 算子在用户眼里就是一个抽象的算子。这里的“抽象”指的是用户感知层面，能感知不同实现，它就是抽象的。一般普通算子用户感知不到它的抽象。但是“抽象的逻辑框架”是所有算子的特性，算子架构就是 规格与实现分离。

这样，我们说，atttention 是有正式后端的算子。
<font color="orange">正式后端(backend)</font>: 指的是框架把"选哪个实现"做成一个有名字，有统一接口，可显示指定，可枚举的机制，而不是藏在算子内部的隐藏分支。正式后端通常具备以下特性：
- **有统一接口/抽象**:框架定义了一个 backend 该提供哪些方法(vLLM 里有 attention backend 的抽象),各实现都按这个接口来,所以能互换。
- **可枚举、有名字**:`FLASH_ATTN`、`FLASHINFER`、`TRITON`、`XFORMERS` 这种——每个后端是一个具名选项,能被列出来。
- **可显式选择**:用户能用 `VLLM_ATTENTION_BACKEND=FLASHINFER` 这种入口指定,框架也有自动选择逻辑。
- **可回退**:选不中或不支持当前 arch 时,按规则退到另一个。

## <font color="grape">算子实现的可插拔，可选择，默认绑定</font>

#### <font color="#b48ff4">算子与实现三种组织形式</font>

因此，vLLM 中的算子分为三类
第一类：该类算子提供正式后端，具体的算子实现，可枚举，有名字。框架提供一部分选择，另一部分可由第三方提供用户能够选择要使用的实现。这类算子我们称为可插拔。这个算子的实现和可插拔完全暴露给用户。
第二类：该类算子提供正式后端，具体的算子实现，可枚举，有名字。框架提供了几个实现，用户只能在这里面选择，不可以放入第三方实现。只提供了开关，可以供用户选择。
第三类：该算子没有正式后端，只有内部分派。算子实现，没有具体名字。内部按 arch/dtype 走不同 codepath,但**没有具名后端、没有统一 backend 接口、不可显式指定**。它有"多 codepath",但没有"正式后端"。

第一类完全可插拔的算子，目前主要是三个，attention，quantization, MoE。
第二类可选择实现的算法，包括 通信后端，KV Cache, CUDA graph等，虽然不允许第三方可插拔，但是框架提供了几个实现可以选择，比如通信后端可选择 NCCL/all-reduce。
第三类完全由框架提供，写死实现的算子。框架提供的算子直接绑定实现，不能改变。但是内部kernel可以根据 arch/dtype 选择不同的codepath。比如 RMSNorm这样的算子, 它的功能比较单一，保留多个实现，收益不高。

<font color="#eb4349">大多数普通算子，都是框架提供算子的时候附带提供的唯一实现，并默认绑定，是写死的算子，不能更改。且实现的大部分逻辑都是和GPU上的计算有关，即在kernel中实现。所以一般情况下，我们也会把 kernel 实现就说成是算子的实现。但是其实是算子的实现底层在调用kernel。</font>
#### <font color="#b48ff4">算子多个实现</font>

算子用哪种方式，可插拔，可选择，还是默认绑定完全取决于是<font color="orange">框架的工程选择，不是机制问题。</font>

<font color="#45c36e">原理层面</font>：算子是逻辑规格,实现与规格分离,所以"任何算子的实现原则上都可替换"，这是"算子/实现"这个抽象本身就蕴含的。没有哪个算子在原理上被钉死成只能有一种实现。
<font color="#45c36e">框架层面</font>：某个算子**实际上**暴不暴露可替换的后端、给不给你换、留几个候选,是 vLLM 的**工程决策**,而不是算子原理。

框架在考虑一个算子是否值得维护多个实现版本的时候一般会考虑
>- 这块算子是不是性能热点、值不值得维护多套实现(attention、GEMM、MoE 是大头,所以养了多后端;RMSNorm 收益小,一套够了)
>- 有没有成熟的第三方实现可接(FlashAttention 这种社区生态)
>- 维护成本、测试矩阵爆炸程度(每多一个可选后端,arch×dtype 的组合就翻倍)

所以"能不能换"是**有没有人去实现并接进来、框架愿不愿意暴露这个选择**的问题,不是"原理允不允许"的问题。原理永远允许;框架按收益决定做不做。
## <font color="grape">attention 算子 / attention backend</font>

attention算子是常常被拿出来讨论的抽象算子，它具有最高的灵活性。attention算子就是一个抽象的正式后端。框架提供的算子实现，都可枚举，有具体的名字，比如，`FLASH_ATTN`、`FLASHINFER`、`TRITON`、`XFORMERS`。用户能用 `VLLM_ATTENTION_BACKEND=FLASHINFER` 这种入口指定,框架也有自动选择逻辑。另外，它允许第三方提供的attention算子实现插入，这就是可插拔。

注意，attention 本身是一个算子，而`FLASH_ATTN`、`FLASHINFER`、`TRITON`、`XFORMERS`这些带名字的包都是attention算子的实现。由于attention算子是有正式后端的，所以这些实现就统称为 <font color="orange">attention backend</font>。<font color="deeppink">我们一般会说，attention backend可以选择不同的算子实现，或者说 attention backend 可替换可插拔</font>。

这些实现里面，各自根据不同的arch提供不同的kernel。因此，`FLASH_ATTN` 可能会有sm_80, sm_86, sm_89等kernel版本。`XFORMERS`可能支持sm_75, sm_86等。也就是说算子实现内部，也会根据 arch/dtype 进行不同codepath的选择。

同时，attention算子还接受第三方提供的实现。第三方实现，可以随时插拔到attention算子上。




