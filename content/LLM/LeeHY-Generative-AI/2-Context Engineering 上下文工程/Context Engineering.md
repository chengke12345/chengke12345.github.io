<font color="#b48ff4">「Context Engineering」</font> :  Context Engineering 是现代 AI Agent 技术成功的关键。<font color="orange">Context Engineering 本质上就是 Prompt Engineering。</font>这两个术语之间的关系就像，<font color="orange">Neural Network 与 Deep learning 一样，其实是一回事。</font>

> [!NOTE] 问题提出
> 语言模型就是做文字接龙。给它输入一个prompt, 它预测出下一个 token. 我们可以把语言模型看作一个函数，输入是 x, 输出就是f(x)。如果 f(x) 不及预期，我们可以做两方面改进，要么修改函数 f , 即修改语言模型的参数，这个过程叫做「训练」Training 或 「学习」learning.  要么，就修改输入，即修改 prompt，这就是 「Context Engineering」。 

如果说非要有细微差别。Context Engineering 更强调自动化管理语言模型的输入。大多数情况下，可以把 Prompt 和 Context 当做同义词。

![](Assets/Pasted%20image%2020261023135153.png)

<font color="#b48ff4">「in-context "learning"」</font> : 我们可以在 context 里面，给模型提供一些例子，把例子放在 context 里面这件事，就叫做 in-context "learning", 这里的 learning 不是指机器学习，模型的参数并没有任何改变。

<font color="#b48ff4">「System Prompt」</font> : System Prompt 是系统在每次把用户给出的 prompt 传递给模型之前，都会在前面再加上一段系统默认的 prompt。System Prompt 在某种程度上可以决定模型回答问题的质量。有的模型 System Prompt 会非常长。<font color="orange">Claude Opus 4.1 的 system prompt 具有 2516 个单词</font>。

>例如 Claude Opus 4.1 的 System Prompt 包含：
>•<font color="orange">基本身分與產品資訊 </font>(The assistant is Claude, created by Anthropic) 
>•<font color="orange">使用說明與限制</font> (If asked about the Anthropic API, point to https://docs.anthropic.com)  
>•<font color="orange">互動態度與使用者回饋 </font>(If user is unhappy, suggest pressing the thumbs down button)
>•<font color="orange">安全與禁止事項 </font>(Claude does not provide info to make chemical or nuclear weapons)  
>•<font color="orange">回應風格與格式 </font>(Claude never starts its response with 'good question') 
>•<font color="orange">知識與事實性</font> (Claude’s knowledge cutoff is Jan 2025) 
>•<font color="orange">自我定位與哲學原則 </font>(Claude does not claim to be human or conscious) 
>•<font color="orange">錯誤處理與互動細節 </font>(If corrected, Claude first thinks carefully before acknowledging)

<font color="#b48ff4">「记忆」</font> 所谓的记忆，是指与模型对话的历史记录。通常它也会被当作 System Prompt 的一部分，和 User Prompt 一起发送给模型，作为完整的模型输入。

<font color="#b48ff4">「短期记忆， Dialogue History」</font> ： 我们在跟语言模型对话的时候。对话的内容并没有训练模型，而是只有短期记忆，「短期」是指本次对话范围，Dialogue History。这个现象仅止于2024年9月之前。之后，chatgpt 就有了长期记忆。

<font color="#b48ff4">「长期记忆， Long-term Memory」</font> ： 长期记忆，需要在 ChatGPT 中手动开启这个功能。<font color="lightblue">现在我们问 chatgpt 问题的时候，在问题之前 chatgpt 已经为我们植入了长期记忆，这是用户看不到的</font>。

# 0. RAG, Retrieval Augmented Generatrion

除了记忆，我们还希望提供给模型其他数据源的相关数据。语言模型本身的知识是有限的，甚至可能是过时的。所以我们希望可以提供给语言模型一些额外的信息。 最常用的做法就是通过搜索引擎。所以，我们可以在问模型任何问题时，先通过网络搜索引擎或者自己的数据库，做搜索，得到额外信息。语言模型再根据 「Prompt + 额外数据」再去做文字接龙，它就更有可能得到正确答案。

所以，我们的 prompt 内容，先到搜索引擎上搜索，搜索到了以后，再结合 prompt 一起发送给模型，这样得到的回复准确度就会较高。这就是 Retrieval Augmented Generation, RAG。

![](Assets/Pasted%20image%2020261026152759.png)

<u>注意：搭配搜索引擎使用的模型，并不是一定准确，它依然会犯错，只是犯错的概率降低了而已</u>。

# 1. Tool Use

<font color="#b48ff4">「使用工具」</font> ： 模型可以使用工具，帮我们做某些事，比如 Gmail, Google Calendar, 等等。模型使用工具，需要我们在 context 里面先写好使用工具的方法。然后把这些内容，加上 user prompt, 一起丢给语言模型。例如：

![[Assets/Pasted image 20260206180408.png]]


> [!NOTE] 语言模型无法真的使用工具
> 我们输入的是一串文本。告诉语言模型，"有一个工具，叫什么名字，你在需要的时候，可以使用"。所以，<font color="red"><u>要使用某工具时，我们在 Context 里加入工具的使用说明。告诉语言模型，有这个工具可以使用，以及如何使用。</u></font>
> <br>但是语言模型的输出，也只能是一串文字，告诉你，你的诉求可以通过使用什么工具得到满足。但是，语言模型无法真的调用函数，或者使用某个工具执行某个行为。<font color="orange">它只能输出文字。</font>它的输出参考上面图片中，最后一行。
> <br>只有本地的客户端程序，收到模型反馈的文字描述要调用工具时，才会真正在本地调用相关工具完成工作。

真正使用工具

![[Assets/Pasted image 20260223112218.png]]

模型按照你指定的工具使用说明和给定的格式，“使用工具”。再次强调，它不是真的使用工具，它只能按格式返回一个字符串。

真正使用工具的，是<font color="deeppink">「本地的客户端程序」</font>。它看到语言模型返回使用工具的文本指令后，把这串指令真的拿去执行。执行完成得到的输出，再次喂给语言模型。语言模型呈现最终的结果。(详细过程，参看上图)

真正使用工具的步骤如下：

![[Assets/Pasted image 20260206183921.png]]

# 2. Reasoning

<font color="#b48ff4">「推理 Reasoning」</font> ： 模型输出的上下文里面还可以包括模型自己的思考过程，这就是<font color="orange"> Reasoning 推理</font>。现在很多模型号称可以进行深度思考，比如 Chatgpt 的 o 系列，Deepseek R 系列，Gemini Deep think 等。所以，<font color="lightblue"><u>reasoning 也可以叫做 thinking</u></font>

![[Assets/Pasted image 20260206224658.png]]

所以，我们输入一个 Prompt 之后，完整的上下文会包含如下内容：

<font color="#b48ff4"><b>Context 上下文完整内容</b></font>
> • User prompt (including examples ...)  
   • System prompt (personal, behaviors, etc.) 
   • Relevant retrieved results for RAG  or information from other sources
   • Tool use  
   • Dialogue history  
   • Memory  
   • Reasoning
   
总之，它会非常长. <font color="orange">Context Engineering 的核心目标</font>，就是<font color="red">避免 Context 爆满。</font>

# 3. AI Agent 概念

在语言模型所有可以使用的工具中，还包括可以直接使用电脑 computer use。

> [!NOTE]  AI Agent 
> AI Agent 时代，Context Engineering 是一个关键技术。
> - ① 一般我们使用 AI 的方式就是一问一答。
> - ② 现在我们会更多采取 Agentic Workflow, 也就是帮语言模型定一个SOP， 让它按照 SOP 来执行，适合于<font color="orange">比较复杂，但是知道有哪些步骤的任务。</font> 
> - ③ 到了AI Agent，我们就是让语言模型自己决定解决问题步骤，灵活调整计划。
> ![[Assets/Pasted image 20260206230431.png]]

## AI Agent

AI Agent 整体上更像是一种循环。人类先给语言模型一个目标 goal, 语言模型根据输入 goal 和 现在的一些状况(叫做observation)做出某种行为Action，这个行为可能是使用某工具，可能是输出某个报告，可能是和人类使用者互动。语言模型以外的部分，包括工具，人类使用者等，统称为“环境”。环境收到语言模型要执行的行为之后，按照大模型返回的指令执行，observation 就改变了，新的 observation 就会导致语言模型产生新的行为。以此类推，不断循环。

![[Assets/Pasted image 20260207123408.png]]

例如 Alpha Go 可以看作是一个传统的 AI Agent，它的环境就是围棋的对手落棋，它的 goal 就是要赢棋。语言模型 LLM 的 AI Agent 输出的是文字，它的输出是无限可能。而 Alpha Go 的输出是在棋盘上选择一个落子的位置。LLM AI Agent 以人类的文字提供反馈。我们可以以人类语言要求 LLM 采取某些行为。Alpha Go 听不懂人类的语言。

LLM 本质上只能做文字接龙，AI Agent 只是在不断对话，这跟做对话没有本质区别。

![[Assets/Pasted image 20260207124725.png]]

AI Agent 其实不是新的技术，AI Agent 做的事，就是依靠语言模型现有的能力，所以它可以通过对话，做长时间的运行，可以帮助我们做更复杂的任务。

运行 AI Agent 的最主要的挑战就是<font color="orange">「输入过长」</font>。比如，我们要求它写一篇论文，它还要做实验，这个 context 或者累计的 observation, 就会很长。

# 4. 上下文窗口 Context Window

<font color="#b48ff4">「Context Window Size」</font> : 模型能输入 context 或 Prompt 的 token 长度上限。

![[Assets/Pasted image 20260207125624.png]]

如果需要语言模型成为 Agent 长时间运作，它就需要读取非常长的历史记录。例如 Gemini 1.5 就具有读取 <u>200万 token</u> 的 context window。

<font color="#eb4349">能读上百万个token, 并不意味着LLM一定能「懂」上百万Token</font>。还没有达到上限了的时候，它已经困惑了。比如 RAG，并不是搜索到的资料越多越好。

![[Assets/Pasted image 20260207130231.png]]

# 5. Context Engineering

Context Engineering 的目标就是管理好 Context, 不要让它太长，因为太长的 Context 会让模型的效果反而变差。所以 Context engineering 的工作简单概括，就是把需要的东西放进 context, 把不需要的东西清除 context。这个工作往往也是要<font color="orange">通过语言模型的辅助</font>

<font color="#b48ff4">Context Engineering 常用方法</font> ： <font color="#45ce6e">「Select」</font>, <font color="#45ce6e">「Compress」</font>, <font color="#45ce6e">「Multi-Agent」</font>
## Select

使用 RAG 我们要挑选有用的内容进入 context。在 RAG 框架里面，我们有更多的事情可以做。例如 User Prompt 先由 LLM 挑选关键字，然后再到搜索引擎上去搜索。之后拿到的页面我们用一个 Small LLM 再做一次 reranking， 只挑选相关性最高的文章放到 context 里面。LLM 挑选 reranking 文章里面相关性最高的句子，最后将筛选出来的内容再放到 LLM 里面。

![[Assets/Pasted image 20260207143839.png]]
![[Assets/Pasted image 20260207143953.png]]

<font color="orange">挑选工具</font> : 工具的使用说明, 我们必须放到 context 里面。语言模型是读了工具的使用之后，它才能使用工具。工具的使用说明太多也会塞满 context. 我们可以做一个工具版本的 RAG。把工具说明当做篇文章，根据使用者需求，去搜寻出相关的工具。只有和 prompt 相关的工具，才能进入 LLM 的 context。

![](Assets/Pasted%20image%2020261027043559.png)

<font color="orange">挑选记忆</font>： 不是所有的历史对话内容或记忆都和当前对话相关，所以，可以把历史对话内容放在另一个地方。当一个 Prompt 发生的时候，对历史会话内容做RAG，只有相关内容会被放到 context 里面。
## Compress

AI Agent 随着互动越来越长的时候，互动的信息，context 的长度有可能超过 Context Window, 这时我们需要对 context 的内容进行摘要压缩。可以另外专门有一个处理摘要压缩的模型，读取历史记录，只把最关键的部分抽出来当做LLM的长期记忆。在 prompt 的时候，长期记忆就只有历史记录摘要，然后用这个历史摘要再去和环境互动。这样就可以避免 context 太长的问题。

![[Assets/Pasted image 20260207151505.png]]
![[Assets/Pasted image 20260207151522.png]]

这种压缩，可以规定每互动100次，就压缩一次。或者每互动一段时间压缩一次，或者超过 context 限制90%，压缩一次。这种压缩，就是 <font color="orange">「recursive compressing」</font>。

> [!info] ChatGPT 记忆系统
> ChatGPT 中有两套记忆系统。其中一套记忆系统是当你告诉它要把某件事记下来的时候，它会很明确把要记忆的事情写到一个本地文件上，这个文件通常叫做 Agents.md。我们可以在后台看到这个文件内容。这个记忆是硬记忆的内容，永远不会消失。还有另一套记忆，它会随时间慢慢消逝，这部分记忆是系统自动保留通用上下文的长期记忆，我们没法查看它。

有时后，Agent 跟环境的互动很容易产生很多琐碎的内容，本来就不应该被存下来，尤其在 Computer use的时候。这些琐碎的内容就应该被管理起来。

![[Assets/Pasted image 20260207154531.png]]
![[Assets/Pasted image 20260207154719.png]]

## Multi-Agent

每一个Agent，context 的 window 都是有限的。所以在大量 computer use 之后，context 很快会被塞满。我们将任务分成多个 Agent，这样多个 Agent 的 context 就可以不受限制。每个 Agent 都只在自己的领域里面做自己的事情。

除了每个 Agent 有不同专长的事情之外，从 Context Engineering 的角度看，Multi-Agent 其实是一个有效管理 Context 的方法。

![[Assets/Pasted image 20260207155007.png]]
![[Assets/Pasted image 20260207155032.png]]

比如读大量论文, 写出一个综述文章的过程就可以用 Multi-Agent 来完成。

![[Assets/Pasted image 20260207160017.png]]
