Colab 是 Google 旗下的共享实验平台 Colaboratory。主要是用来跑机器学习和深度学习的测试和实验。主要是面向学习性质的一个共享计算平台。

# Colab 平台

Colab 是 Google 提供的一个免费云端 Jupyter Notebook 环境。<font color="orange">它能让你在浏览器直接编写和运行 Python 代码，而不需要在本地配置任何环境</font>。

<font color="#b48ff4">「免费 GPU / TPU 资源」</font> ： 免费版能分配 Tesla T4, 对于跑测试和实验够用。

<font color="#b48ff4">「零配置 Python 环境」</font> ： Pytorch, Tensorflow, HuggingFace Transformers 这些常用库都是预装好的。省去了折腾 CUDA 版本，驱动兼容之类的麻烦。但是，这让我们对环境的控制比较弱，它的预装版本，不一定是我们期望的版本。

<font color="#b48ff4">「基于 Jupytor Notebook 交互式开发」</font>： 代码以 `Cell`为单位执行，适合边学边试，逐步调试。对学习而言，边学边试的交互式场景，效率要高很多。

<font color="#b48ff4">「Google Drive 集成」</font> ： Google Drive 是直接挂载到 colab 上的，可以存取模型权重和数据集等。也可以将我们编写代码的 `.ipynb` 文件持久化存储。Google Drive 挂载到 Colab 上，相当于 Colab 把 Google Drive 当作它的后端存储系统，或者后端磁盘空间。


> [!NOTE] 限制🚫
> Colab 是一个「临时运行环境」。我们在环境中做的一切工作，包括下载的文件，模型，修改等等，<font color="deeppink">在连接断开后，就会全部抹去。Colab 是以连接为基础的临时运行环境</font>。
> 每次重新连接时，我们得到的都是一个新的纯净的工作环境。

<font color="#b48ff4">「云端硬盘中存储副本」</font>

一般我们可能会收到别人给我们 colab 的程序代码或文本文件，它通常以 .ipynb 结尾，它是  Jupitor Notebook 文件。
这个文件的物理位置是在别人存储器上的，他分享给我们的是文件URL的链接，我们打开，只能看它。
<font color="orange">如果要做修改我们首先需要把它复制到自己云端存储 Google Drive 里面，然后在自己拷贝的副本中修改运行。</font>
别人的文件我们可以阅读，可以执行，但是不能修改，只有存储副本到自己的 Google Drive 上，我们才有权限修改自己的文件，在自己的 Colab 环境上运行。

<font color="#b48ff4">「更改运行时类型」</font>

Colab 让我们可以选择跑在哪一个 GPU 里面，免费是 T4 GPU (Tesla)。如果想要使用 A100 或 H100 就需要额外付费。

我们可以选择运行时类型，Python3/Julia/R, 以及运行时版本。

<font color="#b48ff4">「执行程序」</font>: Colab 上，在每段程序代码的左上角的播放符号，就是执行这段代码的意思。

<font color="#b48ff4">「Cell」</font>：Colab 中的执行单元，叫做 <font color="orange">「Cell」</font>。

一个 Cell 执行完之后，这个 Cell 中定义的所有变量会保留在内存中，直到我们主动通过 del 删除，或者重启运行时。而普通的 Python(.py) 脚本，在跑完一次程序之后，进程结束，内存释放。

所以，Colab中加载模型一次，模型就会在内存中，直到删除或断开连接，重新启动。而在普通Python(.py)脚本中，跑完之后，进程结束，模型就被释放了。需要注意他们在内存管理中的差异。

# 2. Colab 支持三种运行时

## `◼︎`区分三个东西

Colab 这个系统中要区分三个东西， <font color="orange">「笔记本文件(.ipynb)」, 「Colab 网页编辑器」，「执行代码的运行时环境」</font>。它们三者是分离的:

> - <font color="#45ce6e">.笔记本文件</font>：.ipynb 文件可以放在 Google Drive 上，可以放在 Github 上等其他存储器上。
> - <font color="#45ce6e">Colab 网页编辑器</font>：读取笔记本文件的，可以是 Colab 的网页版编辑器，或者使用其他编辑器，例如本地编辑器或者终端等。
> - <font color="#45ce6e">运行时环境</font>：执行代码的运行时环境可以 Colab 的运行时环境，可以是 Google Cloud 运行时环境。也可以是本地的运行时环境。

## `◼︎`运行时连接方式

Colab 支持的连接方式分为四种：

>  - 第一：最常用的方式。我们在 Colab 网页编辑器中，编辑完了笔记本文件之后，直接连接 Google Colab 的运行时环境，即连接到默认的 Colab 托管运行时。这种方式可以有一些选项，比如用什么 GPU，Python 解释器版本等。

> - 第二：我们在 Colab 网页编辑器中编写完笔记本文件之后(不一定非要用Colab网页编辑器编写)，连接 Google Cloud 运行时。使用的是我们自己的 Google 云端服务器运行时环境。

> - 第三：我们可以在本地编写 .ipynb 文件，通过 SSH 连接到 Colab 托管运行时环境，执行本地编写的文件。

> - 第四：我们还可以使用 Colab 网页编辑器，编写好文件后，不使用 Colab 的托管运行时，而使用自己本地的运行时。(需要通过 Jupitor 服务连接, 不是直连)

虽然这四种方式 Colab 都支持，但是通常我们主要使用的是第一种方式。

## `◼︎`Colab 网页编辑器

我们在 Colab 网页中打开的文件，其实是用 Colab 网页编辑器打开了一个 .ipynb 文件。如果是自己的(比如从自己的 Google Drive 上打开)，就可以编辑。如果是别人分享的，本质上别人分享的是一个URI，我们打开的是别人存储器上的文件，只能读不能写。需要修改，就要现在自己的存储器上存储一个副本，然后修改自己的副本文件。

此时，只是阅读和编辑文件，还没有连接运行时环境，也还没有执行任何代码。执行代码需要连接运行时环境。

所以，这就是「笔记本文件」，「Colab 编辑器」，「运行时环境」，三者分离。
