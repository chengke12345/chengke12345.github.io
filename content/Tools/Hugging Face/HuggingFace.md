# 1. HuggingFace 网站

如果我们要找开源模型,，可以去一个网站，叫做 HuggingFace (https://huggingface.co). HuggingFace 上有各式各样的开源模型。HuggingFace 是一家公司的网站，本来这家公司是要做一个聊天机器人，所以LOGO是一双手托着一张笑脸。但是，后面做着做着就变成了<font color="orange">一个放开源模型和数据集的平台。</font>它有点像模型版的 Facebook。

# 2. Transformers 套件

HuggingFace 开发了一个Transformers 套件。这里的 Transformers 不是指现在最流行的类神经网络架构 Transformer。而是 HuggingFace 所开发的套件，这个套件可以让我们使用在 HuggingFace 上托管的各种语言模型。

无论这些模型背后的架构是什么(可能通过Pytorch或者是JAX来训练的)。只要放上了HuggingFace, 都可以通过 HuggingFace Transformers 套件来使用，它相当于为 HuggingFace 上提供了统一使用模型的工具或接口。

还有很多其他的套件，比 HuggingFace Transformer 更高效。比如Ollama. Ollama 本身也是一个使用模型的套件。

# 3. HuggingFace 权限

### Access Tokens

要从 Hugging Face下载模型的话，需要连接到 HuggingFace Hub 上。我们编写的程序在连接到 HuggingFace Hub 的时候，需要提交 User Access Token。程序提交的 Access Token 与 我们的账号在 HuggingFace 上创建的 Token 匹配时，HuggingFace Hub 才授权这个程序可以在 HuggingFace Hub 上执行操作。

我们在 Hugging Face上创建 User Access Token 的时候，本质上就是在 HuggingFace Hub 上注册一个Token，当应用程序提交的 Token 和注册的 Token 完成匹配 Pairing 的时候，它就获得了Token permission 授权，可以执行操作。

### Login

```python
from huggingface_hub import login
login(token="xxxxxx") 
```

Login 操作就是将本地程序连接到 HuggingFace Hub 上获得权限。我们通过 huggingface_hub 包中的函数 login，提供 access token,。如果能和 Hugging Face Hub 上的 User Access Token 匹配上，就可以获得权限。

login 函数中接受到的是一个字符串 token,  函数执行后， login 会把输入的 token 存放在本地文件中。即使程序结束，这个文件依然存在。下次遇到任何需要访问 Hugging Face 的代码的时候，会自动读取这个文件中的 Token，完成身份验证。而无需每次都 login 重复验证身份。

默认保存路径为：
`Linix/MacOS: ~/.cache/huggingface/token`
`Windows: C:\Users\你的用户名\.cache\huggingface\token`

第二次执行 login 登录代码时，不会重复登录。每次执行，都会用最近提交的 token 覆盖 token 文件中内容。虽然看起来像是重新登陆，但实际上只是更新了存储的token.


# 4. 模型下载与加载

使用 HuggingFace Transformers 下载模型的标准做法是

```python
from transformers import  AutoTokenizer, AutoModelForCausalLM

model_id = "meta-llama/Llama-3.2-3B-Instruct" # 模型名字

tokenizer = AutoTokenizer.from_pretrained(model_id) # 获得词库
model = AutoModelForCausalLM.from_pretrained(model_id) # 获得权重
```

使用 HuggingFace Transformers 包中的 AutoTokenizer 和 AutoModelCausalLM 两个类。 它们都有一个方法, from_pretrained()，首次执行会把模型从 Hugging Face 下载下来。

模型实体就是的是两个东西，<font color="orange">「词库」</font>和 <font color="orange">「模型权重」</font>。vocabulary 词库包含模型使用的所有 token，下载下来绑定在 tokenizer 变量上。模型权重参数下载下来绑定在 model 变量上。通过这两个变量我们可以访问模型的词库和参数。

这两行代码，让模型下载之后，默认直接加载到内存。第二次执行的时候，如果 from_pretrained() 方法发现本地缓存中已经下载了模型, 那它就直接把模型加载到内存中，不会重复下载。
模型默认会保存在：
`Windows: c:\Users\你的用户名\.cache\huggingface\hub`
`MacOS\Linux: ~/.cache/huggingface/hub`
在这个目录下会看到以模型命名的文件夹，如 `models--meta-llama--Llama-3.2-3B`。这里面会缓存所有从HuggingFace上下载过的模型。


> [!NOTE] 不会重复下载
> from_pretrained() 函数最终目的是将模型加载到内存，供程序访问使用。
> - 第一次执行：from_pretrained() 函数会从 huggingface_hub下载模型，保存到缓存目录。
> - 第二次之后执行：from_pretrained 函数会先检查缓存，发现已经存在，就从本地加载，如果不存在，就重新下载。

> 注意⚠️：虽然第二次之后执行，不会重复下载，每次执行这两行都要重新把模型加载到`tokenizer`和 `model`中。程序运行结束后，都会把模型权重参数清除出内存，它是伴随着 Python 程序的结束而结束的。

### CPU 加载模型

像上面这样仅使用 from_pretrained 方法加载模型，默认是加载到物理内存，不是 GPU。也就是说使用模型时，调用 model 和 model.generate 生成时，并没有使用到 GPU。模型加载的时候，默认加载到物理内存，之后也是使用 CPU 进行推理。

### GPU 加载模型

如果我们要指定将模型放到 GPU 上执行，可以使用

```python
model = AutoModelForCausalLM(model_id, device_map="auto")
#加上 device_map="auto", 如果有 GPU 可用，就将模型加载到 GPU，否则
# 就加载到 CPU。 
print(next(model.parameters()).device)
```

可以通过下面那条print语句查看model当前使用的哪个设备。如果只有一个GPU，那就是 cuda0。

加载到GPU之后，我们需要往模型输入文本，或数据的时候，<font color="orange"><u>不能直接用model去执行，必须要把数据首先移动到 GPU，再输入模型执行。</u></font>

```python
device = model.device #获取模型当前的设备, GPU 
input_ids = input_ids.to(device) #将数据移动到 GPU 上，使它成为GPU上的数据。
```

或者输入数据时，直接

```python
input_ids = tokenizer.encode(prompt, return_tensors="pt").to(model.device) #输入变成 GPU 输入
```

关于模型加载的设备问题，详见 [HuggingFace 模型运行设备](HuggingFace%20模型运行设备.md)

### 模型名字

「Llama-3.2-3B-Instruct」是模型的名字，它是 Meta 发布的开源模型 Llama 的 3.2版本。“Instruct” 表示 「该模型具备理解指令并进行回复的能力」。3B 是 3 Billion (30亿) 的缩写，代表模型具有30亿个参数。

# 5.  tokenizer

### encode 和 decode

每一个 token 都有一个编号(从0开始), 模型只认识 token 编号，而不认识任何文字。tokenizer 两个函数，encode 和 decocde。

tokenizer.decode() 把一个编号转回它对应的文字。tokenizer.encode()把文字转换为相应的编号。
# 6. model

HuggingFace Transformer 里，实现了一个函数 model.generate()。model 默认情况下，每次根据输入，只能产生一个token。如果要产生多个token, 则需要编写 for 循环。model.generate()函数可以简化这个过程，连续产生token，直到遇到代表结束的Token为止。

![[Pasted image 20260322190255.png]]

model.generate() 输出的 id, 包含我们输入 input 的 id，它是接着往后接龙。
generate 中的参数，就是一些可以设置的config

```python
output model.generate(
	input_ids, #prompt的 token ids
	malxlen=20, # 最大长度为 20 token
	do_sample=True, # 需要随机抽样，掷骰子
	top_k=3, # 每次从概率最高的 3 个token中掷骰子。
	pad_token_id=tokenizer.eos_token_id #podding token
	attention_mask = torch.ones_like(input_ids) 
)
```

# 7. chat template

每个模型，都是自己的 chat template, 把用户输入按照对话模版组织成一个标准的格式，再发送给模型处理，这样能够提高模型的回答质量。

如果使用我们自己加的 chat_template 模型不一定看得懂，不一定有好结果。通常，使用官方的 chat_template会有比较好的结果。官方 chat template 在官方文档里都有说明，但它比较复杂，自己输入很可能出错。

我们可以通过`tokenizer.apply_chat_template`这个函数来添加官方的chat template。<font color="orange">这个函数不仅帮你在传递给他的消息上添加官方 chat_template，它还顺便做了 tokenizer.encode() 做的事。</font>

![[Pasted image 20260322193919.png]]

我们输入一个 prompt，要先把 prompt 转换成有特定格式的 messages。因为 tokenizer.apply_chat_template 要吃某个特定格式才能运行。

```python
messages = [
	{"role": "user", "content": prompt}
]
input_ids_dic = tokenizer.apply_chat_template(
	#不止加chat template 还帮你 encode了
	messages,
	#信息末尾加一个特殊token,这会告诉模型现在该它回答了
	add_generation_prompt=True, 
	return_tensors="pt" # pytorch tensor 格式
)
```

把输入的 prompt 前面加一个身份 "role", 告诉模型现在这句话是谁说的。user表示是使用者。这是 `apply_chat_template` 要求的是消息格式。

# 8. 模型的输入与输出 

`tokenizer.apply_chat_template` 返回一个字典，大概长下面这样

```
tokenizer.apply_chat_template 的輸出：
 {'input_ids': tensor([[128000, 128006, 9125, 128007, 271,  38766, 1303, 33025, 2696, 25, 6790, 220, 2366, 18, 198, 15724,   2696, 25, 220, 1419, 2947, 220, 2366, 21, 271, 110310, 119395, 21043, 445, 81101, 128009, 128006, 882, 128007, 271, 57668,
21043, 112832, 30, 128009]]), 'attention_mask': tensor([[1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]])}
===============================================
```

它返回了一个字典，而不是一个 tensor 对象。

形如：`{'input_ids' : tensor(...), 'attention_mask' : tensor(...)`
字典的 key=‘input_ids’ ，<font color="orange">对应的值才是真正的tensor对象。</font>

所以，一般我们使用的时候，要把它先提取出来。`inputs = input_ids_dic['input_ids']`, 现在 inputs 中才是 tensor 对象。

```python
inputs = input_ids_dic['input_ids']
outputs = model.generate(
	input_ids
	...
)
print(outputs)
```

model.generate() 产生出来的 outputs 形如：

```
outputs: 
tensor([[128000, 128006, 9125, 128007, 271, 38766, 1303,  33025, 2696, 25, 6790, 220, 2366, 18, 198, 15724, 2696, 25,
220, 1419, 2947, 220, 2366, 21, 271, 110310, 119395, 21043,    445,  81101, 128009, 128006, 882, 128007, 271, 57668, 21043, 112832, 30, 128009, 128006, 78191, 128007, 271, 37046, 21043,    445,  81101, 3922, 87447, 103106, 30046, 101399, 37046, 51609,  13372, 9554, 27384, 25287, 102158, 78244, 123123, 1811,128009]], device='cuda:0')
```

model.generate 出来的结果 outputs，就是一个 tensor 对象。这个打印格式就是 Pytorch 中张量的标准表示。`device="cuda:0"`表示它在第 0 块 GPU 上。
tensor 中有两层方括号"\["。说明这是一个二维张量。外层方括号是第 0 维(batch，表示回答有多句话)，内层方括号是第 1 维 (序列，某一句话的内容)。

# 9. pipeline

HuggingFace 上使用模型最简单的方式是通过 「pipeline」，这样可以节省将文字转成 token ID，然后再转回来的过程。这也是最常见的方式。

![[Pasted image 20260323090204.png]]

我们直接把 messages 丢给一个叫 pipe 的函数。pipe 是 pipeline 通过model_id， 构造出来的可以直接访问模型的对象。

```python
pipe = pipeline("text-generation", model_id)
outputs = pipe(
	messages,
	max_new_token = 2000
	...
)
```

具体的，pipeline 返回的是一个 callable 对象，把它与 pipe 关联，然后通过pipe去 call 调用。

<u>pipe得到的输出，就直接是文字，整个过程不用再做 encode 和 decode </u>

>- `pipeline` 是 Hugging Face transformers 库提供的一个<font color="orange">工厂函数</font>。<font color="lightblue"><u>它把加载 tokenizer、加载模型、编码输入、推理、解码输出，五步, 全部封装起来。</u></font>
>- `pipeline("text-generation", model_id)` 一行就把 tokenizer 和 model 都加载好了，返回一个**可调用对象**（callable），你可以把它当函数用。之后 `pipe(messages)` 一行就完成了编码、推理、解码全过程。

`pipeline` 也是默认把模型加载到 CPU，如果要让模型加载到 GPU，需要指定设备。

```python
#方式一：指定 device 
pipe = pipeline("text-generation", model=LLM_NAME, device=0) # GPU 0 

#方式二：用 device_map 
pipe = pipeline("text-generation", model=LLM_NAME, device_map="auto")
```

但是，<font color="orange">我们在让模型生成输出的时候，不用再关心模型输入在哪个设备上。pipeline 会自动帮我们找到正确的设备，直接调用就可以了</font>：

```python
output = pipe("Hello, how are you?")`
```

不管模型在 CPU 还是 GPU 上，`pipeline` 都知道模型在哪个设备上，会自动把输入送到正确的位置。你只管给它原始文本，剩下的它全处理了。