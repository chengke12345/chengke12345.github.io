
## <font color="grape">框架支持compute capability</font>

<font color="#45c36e">compute capability, CC, 叫做算力版本</font>，如 sm_75。本身是Nvidia定义的硬件属性，描述的是芯片的SM架构版本和它支持的指令集，不是vLLM框架的东西，严格说，不存在"vLLM的compatibility"，只有"vLLM对某个cc的支持"。

vLLM 框架本身来说也是一个软件，要运行在GPU硬件平台上，它编译出的代码对硬件arch有下界的要求，这就是vLLM支持的CC。说的是这个软件平台本身对硬件arch的要求。而框架中的算子实现，kernel实现时，会针对硬件arch，编译产生可执行程序，这是算子kernel实现所支持的cc。
所以vLLM整体的cc声明 和 算子内部实际的cc覆盖是两回事。vLLM的声明是编译执行vLLM这个框架软件的一个最低下限，算子内部 cc 的覆盖可以是算子自己的决策。


> [!NOTE] vLLM框架结构
>  <font color="orange">vLLM 大体上由框架软件本身和一些列的算子组成。框架软件本身负责主逻辑的运行，算子是提供功能的部件。</font>
>  <font color="#45c36e">框架本身</font>：<font color="lightblue">调度、内存管理（KV cache 分页/块表）、批处理（continuous batching）、请求队列、API server、并行编排（PP/TP 怎么切怎么通信）——这些是"组织和调度"的逻辑，主体是 Python，本身不怎么吃 CC。</font>
>  <font color="#45c36e">算子</font>：<font color="lightblue">真正在 GPU 上做数值计算的部件——attention、量化反量化、rotary、all-reduce 等，CUDA/SASS 编出来的，吃 CC 的就是这层。</font>
>  <font color="#eb4349">vLLM = 调度逻辑 + 一整包算子</font>。<font color="lightblue">vLLM 支持 sm_70" 这条门槛线，本来就不是框架那层（Python 调度）提的要求——它就是替算子那层提的。框架的调度逻辑本身（Python，不编 GPU 机器码）不因 CC 不同而跑不跑得动；但 vLLM 作为一个整体软件包，对外声明的那条 CC 下界，是被它打包的算子决定的。</font>

所以，vLLM框架声明的cc, 是框架能够正常编译运行支持硬件arch的最小 cc。本质上是替框架内部的算子实现，对硬件arch做初筛。只有高过这个版本，框架才会对这个硬件arch准入。
![[Pasted image 20260607185335.png]]

但是，对于这个最低的arch，算子内部kernel实现，不一定覆盖最低cc，这是由算子自己决定的。比如vLLM声明的最低cc是sm_75, 但是GEMM算子实现要求最低 ≥ sm_80. 意思就是sm_75框架对硬件arch才准入，但不是说里面的算子都覆盖sm_75, 是否覆盖这个arch是由算子自己决定的。vLLM自己提供的算子实现如此，第三方算子实现的cc覆盖，它就更管不着了。

## <font color="grape">算子实现覆盖 cc</font>

算子是一个抽象的逻辑规格。算子具体的会有不同的实现。例如 `Attention`的算子, 可以进行具体实现的可替换。它可以有多个具体的算子实现，`FlashAttention`/ `FlashInfer`/ `Triton`/ `XFomers` , 这些都是具体对 attention 的实现，是真正干活的组件。

这些算子本身需要调用底层的 kernel 在GPU上去完成真正的计算，kernel 本身是针对硬件架构 arch 编写的。不同kernel会支持不同的cc，其内部也会根据不同的arch 调用不同的kernel。就会有 attention算子的实现 Flash attention 支持 sm80, sm86, sm89 架构。而 attention 另一个实现 xFormers 支持 sm75，sm80等。

框架要能在某个硬件arch上运行，框架内算子的 kernel 实现，也要能够覆盖硬件arch的cc。
## <font color="grape">模型，dtype / 量化Kernel</font>

模型本质上就是一堆权重文件，如何使用这些权重文件，需要的是框架，比如 HuggingFace提供的Transformer或Ollama等，它们实现了attention的算法，flash attention加速，以及其他跟使用权重文件相关的工具。

模型本身不存在 "支持的GPU型号“ 这个说法，模型就是权重 + 架构,它对硬件无知。真正决定我们能不能使用某个GPU的是使用的部署框架。同一份权重, 在 vLLM / SGLang / llama.cpp / TensorRT-LLM 上对硬件的要求完全不同。各框架有自己的 hardware support matrix。各框架在使用模型的时候，都有自己对模型的使用方式。

模型文件本身和GPU无关，在repo里就是一堆权重+config, 权重以某个 dtype 存储, (BF16 / FP16/ FP8 / 或 IN4+scale这种量化打包), 问题不在于模型是否支持某显卡，<font color="orange">而在于这个 dtype 要跑起来，硬件arch支持这种数据格式，因为kernel在真实计算的时候，是把数据放到硬件上去执行，所以要求硬件必须要支持模型参数的数据格式</font>。

比如 2080ti 的Turing 架构的 sm_75 没有 BF16 的硬件计算路径，所以 vLLM 就不为 sm_75 提供 BF16 Kernel。所以实践结论确实是：**BF16 权重在 2080Ti 上必须 `--dtype float16` 转成 FP16 跑**——而 FP16 的 Tensor Core 路径 Turing 是有的。
## <font color="grape">GPU 架构</font>

GPU的架构 sm75 sm80 这样的规格来界定 arch 的。sm 代表的是 stream multiprocessor 是流多处理器。后面的数字是一个表示一个完整的芯片中，这样的流处理器有多少个？所以，用简单流处理器的数量表示 GPU 芯片架构的规格。

<font color="#45c36e">compute capability, CC, 叫做算力版本</font>，如 sm_75。本身是Nvidia定义的硬件属性，描述的是芯片的SM架构版本和它支持的指令集，不是vLLM框架的东西，严格说，不存在"vLLM的compatibility"，只有"vLLM对某个cc的支持"。

vLLM框架中的某些特性，要根据GPU架构来，因为vLLM中使用的 kernel 是针对不同的硬件架构编写的。kernel实现的算子是否提供了某个GPU架构的版本实现，就是这个feature是否支持该硬件。
![[Pasted image 20260608124427.png|600]]

GPU 中本身架构规格决定了它是否能支持一些数据格式.
比如 sm75 Turing 架构，支持FP16，但是没有BP16 和 FP8 硬件，所以不能做 BP16 和 FP8 的计算。也就是说这些数据格式的模型，在 Turing 架构上都得不到支持.


## <font color="grape">GPU架构适配性考虑</font>

我们在考虑GPU和模型适配性的时候，要从三个方面考虑 <font color="orange"> 部署框架 / GPU硬件架构 arch / 模型 dtype </font> 这三者之间相互适配，模型才能顺利在GPU上跑起来。

![[Pasted image 20260609064254.png]]
一个模型要能跑起来，需要考虑三个因素的适配，也就是模型，硬件，部署框架。

>- <font color="#45c36e">① 首先，框架本身对cc有一个最低要求的声明，框架软件要在支持的cc的arch上才能运行。每个框架有自己的 hardware support matrix，可以查看是否支持某种硬件arch。</font>
>- <font color="#45c36e">② 框架对cc的声明只是框架的准入门槛，我们还要查看框架内提供的算子实现，是否覆盖了硬件cc。如果我们要使用该算子，但是算子没有覆盖我们硬件arch的cc，那也不行。</font>
>- <font color="#45c36e">③ 部署框架本身，对模型的支持有自己的支持列表。模型必须是框架支持的模型，才能使用该框架进行模型的推理部署。</font>
>- <font color="#45c36e">④ 除了cc之外，GPU的显存还必须足够，否则无法装下权重文件。</font>
>- <font color="#45c36e">⑤ 模型本身对硬件 arch 没有要求，但是模型的参数是由数据格式的，数据格式必须是硬件arch支持的数据格式，推理的计算才能在硬件上执行。</font>



