# vLLM 部署方法

现在的部署流程已经很简便了，基本上就是分为<font color="orange">「环境准备」</font>，然后<font color="orange">「准备模型」</font>，最后<font color="orange">「通过一行部署命令把整个服务给部署起来」。</font>

首先，我们要确保，我们要本地化部署的大语言模型，是在 vLLM 支持的列表中的。一般来说，现在主流大厂的模型，vLLM都是支持的。

在部署的时候，直接使用命令行参数：

```powershell
#使用两张卡来部署Qwen3-8B,如果没有device，默认使用0号单卡部署
CUDA_VISIBLE_DEVICES=0,1, vllm serve{local_path} \\
		--api-key chengke12345 \\ #设置api key
		--served-model-name Qwen/Qwen3-8B \\ #api服务的模型名称
		--max_model_len 4096 \\ #最大处理长度(输入prompt+生成内容的token总数)
		--tensor-parallel-size 2 \\ #张量并行数量，和使用的GPU卡数保持一致
		--port 7890 #访问端口
# ----------------------------------------------------------------------------
#示例(vllm serve 后面跟上面指定的参数)， 没有指定 Device 参数，默认只有一个 GPU，并且是0号
vllm serve /root/models/Qwen3-8B --api-key abc123 --served-model-name Qwen/Qwen3-8B --max_model_len 4096 --port 7890		
```

## 参数解析

- 「CUDA_VISIBLE_DEVICE」:  我们可以去指定用几张卡来部署，这里是指定具体用哪几张卡去部署，这里是 0号 和 1号 GPU. 如果没有指定，默认使用0号，单卡部署。

- 「local_path」: 指定我们下载的模型的本地路径。

- 「api-key」:  这是 OpenAI 提供的模型接入方式，做访问和鉴权，如果没指定默认为空。这个 api-key 可以给任何我们允许他接入模型的用户。

- 「served-model-name」: 我们给模型取的名字，因为我们访问 vLLM 服务，使用端口为7890。在显存足够的情况下，我们可能同时部署了多个模型。用模型名来区分使用的究竟是哪个模型。

- 「max_model_len」: 这是最大处理 token 的长度，包括输入的 prompt， 和生成内容的token. 也就是 context window. 注意⚠️，这是在框架层面限制的 context window, 不是模型自身的 context window 约束。

- 「tensor_parallel_size」: 张量并行数量，和使用的 GPU 卡数保持一致。

- 「port」： 访问服务的端口

<font color="orange">vllm 命令还有很多其他的参数配置，可以查看</font> `vllm --help` <font color="orange">查看相关的参数说明。</font>

## 关于 Chat Template

在请求大模型回答问题的时候，它会自动先把问题，按照对话模板的形式，包装成一个多轮对话的记录。这样，大模型就可以感知到与用户交互过程的历史内容。主流大模型都默认将对话模板写在了 tokenizer_config.json 配置文件中，这样vllm启动时就会自动加载。

现在的大模型一般都经过了指令和聊天数据的微调，但是某些模型并未提供 chat template. 这种情况下，我们在部署的时候，就需要去指定，要用哪一个聊天模板。vllm官方提供了一些 chat-template 的示例版本，在开源的代码框架里的 examples 下可以查看。