# 1. 加载模型到 GPU 设备

使用 Hugging Face 套件 Transformers 的 AutoTokenizer 和 AutoModelForCausalLM 下载和加载的模型的时候，如果没有任何设置，默认是加载到内存中，并且推理计算是由 CPU 直接使用内存完成。

```python
from transformers import AutoTokenizer, AutoModelForCausalLM

model_id = "google/gemma-3-1b-it"

tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(model_id)
```

tokenizer 会绑定下载模型的「词库」。model 会绑定下载模型的「权重参数」。下载模型之后，会自动把模型加载到系统中。<u>默认是加载到内存中，并由 CPU 完成后面的推理计算。</u>

如果要让模型加载到 GPU 中，并由 GPU 完成后面的推理计算任务，就需要设置，指定加载模型的设备。

```python
from transformers import AutoTokenizer, AutoModelForCausalLM

LLM_NAME = "google/gemma-3-1b-it"

tokenizer = AutoTokenizer.from_pretrained(LLM_NAME, device_map="auto")

model = AutoModelForCausalLM.from_pretrained(LLM_NAME,device_map="auto")
```

在构建`tokenizer` 和 `model` 对象的时候，我们使用了参数 `device_map = "auto"`，它表示如果有 GPU 且正常工作时，就将模型加载到 GPU 上。

> [!NOTE] 注意⚠️：只有model会被加载到GPU
>- 「tokenizer」 加上 `device_map = "auto"` 之后，`tokenizer` 不会被加载到 GPU，它本质是字符串处理(查表，BPE 分词)，它不涉及张量计算。写 `device_map ="auto"` 对 `tokenizer` 没有意义 (写上去不会报错，但也不会把它放 GPU 上)。
>- 「model」 `device_map = "auto"`对 model 才有意义，当你的 GPU 显存足够装下整个模型时，所有层都会被放到 GPU 上，CPU 上不会保留模型参数的副本。如果显存不够，`accelerate` 会自动把一部分层留在 CPU 内存甚至 offload 到磁盘上，这时候才会出现 "一部分在 GPU、一部分在 CPU"的情况。

# 2. 传送张量到 GPU

我们执行模型的时候可能如下：

```python
inputs = (tokenizer.encode("hello")).to("cuda")
outputs = model(inputs)
```

PyTorch 要求张量和模型参数在同一个设备上才能做运算。inputs 是 tokenizer 产生的，在 CPU 上，模型在 GPU 上，所以你需要把得到的张量 inputs，传送到GPU上，作为模型输入。

> [!NOTE] 有时数据可以自动传送
> 不过如果加载模型时用了 `device_map="auto"`，Transformers 其实会在内部自动把输入搬到正确的设备上，所以有时候不写 `.to("cuda")` 也能跑。但显式写上是更好的习惯。
> 自动搬运行为并不是所有情况都生效，实际表现取决于模型类型、`accelerate` 版本、以及具体的调用方式。

### 传送的含义

`.to("cuda")` 做的事情就是把张量数据从 CPU 内存（主存）通过 PCIe 总线复制到 GPU 显存上。可以类比 `memcpy`，只不过是从主存拷到显卡的 VRAM。拷完之后，GPU 上的模型就能直接读这些数据，因为它们现在同一块显存里。

# 3. 模型输出 outputs

模型推理的输出 outputs 会留在 GPU 上，不会自动搬回 CPU。我们代码中的 Python 变量 `outputs` 只是一个引用（类比 C++ 的指针），它指向的 tensor 数据实际存在 GPU 显存里。
如果需要把结果拿回 CPU 做后续处理，需要显式调用。大部分情况下你不需要手动拷回，因为后续的 `model.generate()` 或 loss 计算也都在 GPU 上完成。只有最终要打印结果或者用 numpy 处理时才需要搬回 CPU。

### CPU上访问模型输出是如何操作的

PyTorch 在这里做了一些对用户透明的处理。

执行 `print(outputs.logits)` 或 `print(outputs[0])` 时，Python 的 `print` 本身当然在 CPU 上执行，但它调用的是 tensor 的 `__repr__` 方法。
PyTorch 的实现会在内部自动把 GPU 上的数据临时拷回 CPU 来生成那个字符串表示，但这个拷贝是隐式的、临时的，不会改变原始 tensor 的设备归属。
打印完之后，`outputs.logits` 仍然在 GPU 上。

> 也就是说，outputs 本来是 CPU 上的变量，但是它指向的是GPU地址中的数据。当需要的时候，相当于跨设备远程读取。

注意⚠️，如果你对 GPU tensor 做需要 CPU 才能完成的操作，比如 `.item()`（取单个标量值）、`.numpy()`、或者用它做 Python 的 `if` 判断，PyTorch 有的会自动隐式拷贝（如 `.item()`），有的会直接报错要求你先手动 `.cpu()`（如 `.numpy()`）。行为不完全一致，所以显式 `.cpu()` 总是更清晰的做法。

> [!NOTE] 总结
> - <font color = "#45ce6e">outputs` 这个 Python 对象本身活在 CPU 的 Python 进程里（所有 Python 对象都在 CPU 上），但它内部持有的 tensor 数据的实际存储地址在 GPU 显存中。用 C++ 的视角理解，就像一个结构体在主存里，但里面有个指针指向显卡 VRAM 中的一块 buffer。</font>
> - <font color="lightblue">当你做 print, .item() 这类操作时，PyTorch 底层会通过 CUDA API 从显存把数据读回主存，本质上就是一次 PCIe 上的 DMA 传输。可以理解为"跨设备远程读取"</font>


