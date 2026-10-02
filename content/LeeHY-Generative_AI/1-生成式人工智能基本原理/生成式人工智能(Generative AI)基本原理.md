# 1. 基本概念

> [!NOTE] 文字接龙
> ChatGPT, Gemini, Claude 等这些人工智能都是语言模型。语言模型本质上就是一个做<font color="orange">「文字接龙」</font>的人工智能。事实上，文字接龙是 chatgpt 等人工智能 <font color="red"><u>唯一会做的事</u></font>

- <font color="#b48ff4">「token」</font>:  语言模型做文字接龙时, 用来接龙输出的最小单位, 叫做token。
- <font color="#b48ff4">「Prompt」</font>: 给语言模型的输入叫做 prompt, 可以供选择的输出叫做 Token。
- <font color="#b48ff4">「Vocabulary」</font> : 语言模型的输出是下一个可以接龙的 token 的概率分布。语言模型会给所有可以用来做接龙的 token 一个概率分数，然后从中选择概率最高的(或用其他算法选一个 token，例如掷骰子)。<font color="orange">模型所有可以用来做文字接龙的 token 合在一起的集合叫做 「Vocabulary, 词库」。这是一个「有限集合」。</font>

> 注意⚠️：语言模型每一次产生的是输出 token 的概率分布，它会给每一个vocabulary 中的 token 一个概率分数。然后使用某个算法根据 Volcabulary 中每个 token 的概率选出一个 token 作为输出。
> 最终产出的 token 是根据概率选择的，所以它又叫做「预测下一个 token 」

现在的 Vocabulary 非常巨大，至少都有数十万个 token。

> [!NOTE] 人工智能与文字接龙
> 
一个语言模型要能够正确的进行文字接龙，必须拥有两方面的知识。
<font color="orange">「语言知识」</font> ：能够理解人类语言的语法。比较容易学，模型学习某种语言只要上百万份文章就足矣。 
<font color="orange">「世界知识」</font> ：物理世界运行的规则规律。难以学习，无穷无尽。


> [!NOTE] 大语言模型的参数
> 
大语言模型输出的是下一个 token 的概率分布。语言模型可以看作是一个函数 f(x) = ax + b; 把文本 x 看作输入，通过语言模型 ax + b, 得到输出概率分布 f(x). f 就是语言模型。a, b 叫做<font color="orange">「参数」</font>。通常一个语言模型都有非常大量的参数。现在的大模型，<u>百亿参数遍地走，十亿参数谁都有</u>。大语言模型的“大”，就是指非常大量的参数。<font color="lightblue">这些参数不是通过人工设定的，而是通过资料自动「学习」得到的。</font>![[Pasted image 20260219214332.png]]

  <font color="#b48ff4">「chat template」</font> : 为了让语言模型更好的理解用户意思，我们使用的平台通常会在 prompt 前后加入一些信息，再交给大语言模型处理。这些信息叫做 chat template。每一个模型用的 chat template 都不相同。注意，具体可以使用什么 chat template, 这和我们使用模型有关。
 
 > 比如，为了让语言模型能够正常回答问题，并且不会在遇到问题时，接龙出其他奇怪的内容，语言模型平台通常会在我们提问的前后加上一些内容，促使语言模型回答问题，如 “使用者问「问题内容」 AI答：”，这样语言模型收到 prompt 以后，就会正常回答了。
 
 <font color="#b48ff4">「system prompt」</font> : 每次我们提供 prompt，平台都会自动加入一些基本信息(今天的日期， 等等)。这些开发者加入的，系统自动每次交互都会自动加上的prompt，叫做 「system prompt」。

> [!NOTE] 区分「chat template」和 「system prompt」
> - 「chat template」 主要是负责把输入信息进行格式化规范处理。对话模板就是格式话对话规则。它是格式层面的处理。比如用户说"你好", 平台会格式化为 "用户说: 你好. AI 回答:", 这样发送给 AI 会让它更好的回答。
> - 「system prompt」主要是负责添加默认信息。比如用户名字，现在时间，今天的日期，天气等。这些附加信息能让 AI 更好的回答问题。
> - <font color="orange">总结：chat template 决定对话如何组织成模型的输入，是格式层面的事。system prompt 决定有哪些默认的附加信息要加在输入内容前后，要求模型如何行为，是内容层面的事。所以，chat template 是格式层，system prompt 是内容层。</font>

 
 <font color="#b48ff4">「对话记忆」</font> ：模型的记忆，通常仅限于在同一则聊天里面。如果我们按了“新聊天”，它会忘记过去跟它说的所有事情。
 
 <font color="#b48ff4">「多轮对话」</font> ：多轮对话，实际上是语言模型把前面所有的对话和答案拼在一起，一起作为 prompt, 发给语言模型，进行下一次文字接龙。
 
 <font color="#b48ff4">「AI 幻觉」</font> ： 语言模型在回答用户问题时，常常会产生一些不存在的东西，这就是AI 幻觉，AI hallucination。 
 
<font color="#b48ff4">「RAG」</font> : Retrival Augmented Generatation, 检索增强生成。为了解决 AI 幻觉问题，将<font color="deeppink">「搜索引擎 和 AI」 </font>搭配起来使用，可以降低 AI 幻觉的概率。现在的AI， 一般都预设 RAG。<font color = "lightblue"><u>但是有RAG，并不表示AI产生的一定是正确答案。</u></font>

<font color="#b48ff4">「Context Engineering」</font> : 人类必须确保语言模型的输入有足够的信息，让语言模型在做文字接龙的时候，能够接出正确的输出。确保输入的内容信息足够，足够到大语言模型能够做出正确接龙的这件事。就叫做<font color="orange"> 「Context Engineering, 上下文工程」</font>。


> [!NOTE] 生成式人工智能
> 
让机器学会产生<font color="orange">复杂</font>而<font color="orange">有结构</font>的物件。有结构指的是，由有限的基本单位所构成，这些基本单位，叫做 token。复杂指的是，虽然基本单位是有限的，但是组合起来就是无限的。让机器学会生成复杂而有结构的物件，就是生成式人工智能做的事情。

生成式 AI 就是输入 一个 x 产生一个 y, 而 y 就是复杂而有结构的物件。

$$
x\{x_1, x_2...x_n\} ----->y\{y_1, y_2,...y_n\}
$$
要教会机器每次如何选择一个合适的 y，即生成策略，在有限集里面选择下一个 y 的策略。
其中，最常见最基本的生成策略就是「文字(Token)」接龙(这里的文字是泛指，接龙的「文字」可以是文字 token, 声音 token, 影相 token)。文字接龙的生成策略，叫做<font color="deeppink">「Autoregressive Generation，自回归生成」</font>。

<font color="#b48ff4"><b>文字接龙(Autoregressive Generation)生成策略的定义</b></font>

文字接龙可以描述为：
$$
\begin{array}{l}
x_1,x_2...x_j... -->y_1 \\
x_1,x_2...x_jy_1 -->y_2 \\
x_1,x_2...x_j,y_1y_2 -->y_3\\ 
...\\
x_1,x_2...x_jy_1y_2...y_{T-1} -->y_T
x_1,x_2...x_jy_1y_2...y_{T-1}y_T --> end
\end{array}
$$
输入部分的 x 也是 vocabulary 中的 token，我们可以把这个过程简化为，给一串 token, 预测下一个 token 的过程，即 $\{Z_1,Z_2...Z_{t-1}\}->Z_t$ .

<font color="orange">token 的集合大小是有限的，所以这是一个有限选择问题，也就是</font> <font color="red">机器学习中的分类问题。</font> 

将产生的 token 结果放到输入的后面，整体再作为新的输入，让语言模型再产生下一个token。这样循环生成，直到产生结束标志为止的过程，叫做<font color="deeppink">「文字接龙，Autoregressive Generation」.</font>

> [!NOTE] 生成策略
> 「文字」接龙 Autoregressive Generation, 并不是唯一的生成策略。可能会有其他的生成策略，其中 Diffusion Model 就是另外一种生成策略。它主要是做影像生成时采用的生成策略。 

# 2. 实践：模型输出过程
 
> [!NOTE] 开源模型
> 我们常用的 ChatGPT, Gemini, Claude, 这些模型背后的行为，也就是模型背后对应的 f 函数以及参数我们是不知道的，这些不知道的“函数”叫做「闭源模型」。
> 也有一些模型是「开源模型」，比如 Meta 的 LLaMA, Mistral, Gemma等。我们可以完全知道 f 长什么样子。我们知道背后有多少参数，每个参数是什么数值，这些都是公开的。<font color="pink"><u>但是，这些模型并没有告诉我们，他们是怎么被训练出来的, 使用的什么显卡，使用的什么训练数据集。</u></font>

<font color="#b48ff4">hugging Face</font> : Hugging Face 是一个网站，上面有各式各样的开源模型。这个网站本来是要做聊天机器人，后来演化为 <font color="orange">「一个放模型和资料集的平台」</font>。
这个平台上可以找到各式各样的模型。

<font color="#b48ff4">Llama-3.2-3B-Instruct</font> :  这是指的 Meta 开源模型 Llama, 版本号3.2， 3B 指的是3 billion 参数的模型。Instruct 表示这个模型具备理解指令并且回复的能力。

<font color="#b48ff4">Hugging Face Transformer 套件</font> ： Hugging Face 开发了一个 Transformer 套件。它可以让你使用托管在 Hugging Face 上的各种语言模型，无论这些模型背后的深度学习框架是什么(例如 Pytorch 或 JAX)。只要这个语言模型被放在 Hugging Face 上都可以通过 Hugging Face 的 Transformer 套件来使用。

有其他套件在使用模型的时候，可能比 Hugging Face Transformer 更有效率，比如 Ollama. 但是 Hugging Face Transformer 具有更高的弹性。


> [!NOTE] tokenizer 和 model
>
每一个模型的下载，我们都会从 Hugging Face Hub 上下载两个“物件”，一个是 tokenizer， 一个是 model.  
<font color="orange">tokenizer 记录模型所使用的 token, 也就是 vocabulary</font>. 而 <font color="orange">model 存储了模型的参数，也就是所谓的权重</font>。 

## Phase 1: 输出一个 token

<font color="#b48ff4">model</font> : model 本质上可以当做一个 function, 可以根据输入的 prompt, 一次产生一个Token。

![[Pasted image 20260204195727.png]]


>- 每一个 token 在都有一个编号(从0开始)，我们可以使用  tokenizer.encode 函数将文字转换为 token 的编号，使用 tokenizer.decode 函数将编号转回对应的文字。这个过程就是「编码」 与 「解码」。
>- model 只吃 token 的编号当作输入, 它不会读文字。要先用 tokenizer.encode 将 prompt 中的文字转换为 token_ids 编号序列. 然后 model 才把这些编号读进去。处理完之后再用 tokenizer.decode 将输出的 token 编号序列转换为token 文字，再返回给客户。

## Phase 2: 循环输出一连串 token

我们可以用循环让它连续多次产生多个Token，它的执行流程如下：

![[Pasted image 20260204201055.png]]

这里选用的是 <font color="orange">概率分布中最高的下一个 token</font>。而这只是其中一种选用的方法，我们也可以 <font color="orange">按照概率，随机掷骰子的方式，选择下一个 token</font>。

![[Pasted image 20260204201311.png]]

选择输出 token 时，如果完全按照掷骰子的方式，很容易出现奇怪的输出，并且只要中间一个 token 出现了错误，后面的都会错误。因为某 token 的概率再低也有可能被随机到。

所以，现在常用的方式是，只有几率足够高的 token，才能参与掷骰子的过程，这样可以避免选到几率很低的 token，这是今天实用语言模型的常用技巧。

![[Pasted image 20260204201844.png]]

## Phase 3: 一次输出一连串 token

上面的方式我们使用的 for 循环来产生Token。但是实际上 HuggingFace 的Transformer 套件里面，<font color="orange">提供了一个函数 model.generate</font>，可以使用它来产生一连串的Token, 而不需要我们手写循环。

![[Pasted image 20260204202804.png]]

> 注意⚠️：模型产生的输出 token，也是一串整数标识符，它无法直接产生文字。所以在把答案发送给用户之前，要进行「解码」，将 token ids 解码转换为用户能看懂的文字。

另外，model.generate() 产生的输出包含了输入的 token_ids。它生成 token 的结束条件是生成了代表结束的 token，或到达了长度上限。

model.generate() 有几个 config 可以设置，比如最大长度，`max_length`。是否需要掷骰子, `do_sample=True`, 以及几率最高的前几名 token 可以参与掷骰子, `top_k=3` 等等。 

## Phase 4: 使用 Chat Template

这是我们直接把 prompt 内容输入模型，产生的答案，效果并不好。因为我们没有加 chat template, 如果我们自己加 chat template, 那么执行流程如下：

![[Pasted image 20260205000954.png]]

自己加的 chat template 模型不一定能看懂。效果最好的是使用模型自己的官方chat template. 官方的 chat template 在 Llama 官方文档里有说明，内容还比较复杂，如果我们自己输入，很容易打错。
Transformers 提供了一个函数 ，「tokenizer.apply_chat_template」，可以直接调用这个函数，把官方的 chat template 加到我们输入的 prompt 上面。

输入prompt 时, 要先转换成某个特殊格式，tokenizer.apply_chat_template 要求特定格式的输入(比如某种 JSON 格式)。然后它才能将官方的 chat template 加到 prompt 上。

![[Pasted image 20260205001357.png]]

使用 「tokenizer.apply_chat_template」，我们就不用自己手动做 tokenizer.encode 了。它会自动帮我把 encode 的工作完成，直接返回处理好之后的 input_ids。

我们在输出的时候，可以只抽取 AI 的回答部分展示给用户。

![[Pasted image 20260205003501.png]]

## Phase 5: 多轮对话

目前我们是单轮对话，还可以进行多轮对话，多轮对话的关键是给模型完整的对话历史记录。

![[Pasted image 20260205003629.png]]

## Phase 6: pipeline

前面使用模型的方式，更多是为了说明如何使用模型的原理。真正使用 HuggingFace 的模型的时候，有一个更直接、更简单的方法，就是 pipeline.

> 一般情况下，我们需要大致三个步骤，从文件转换为input_ids，再将 input_ids 给模型产出 output_ids，再将 output_ids 转换为文字返回客户端。就是encode, decode 的过程。
> 我们可以直接调用 pipeline,  让 encode 和 decode 的部分全部可以省略。


![[Pasted image 20260207182916.png]]

