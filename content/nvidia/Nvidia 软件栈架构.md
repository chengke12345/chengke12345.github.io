## <font color="grape">三个CUDA 版本</font>

#### <font color="#b48ff4">nvidia-smi</font>

当我们安装完了nvidia GPU 驱动时，输入 `nvidia-smi`, 得到这样的输出
![[Pasted image 20260607051518.png]]
右上角的版本CUDA版本号。<font color="orange">它表示的是这个驱动能够支持的最高 CUDA 版本。</font> 它表示这个驱动能跑所有 CUDA Runtime ≤ 13.2 的版本，它表示的是<font color="deeppink">驱动内嵌 CUDA Driver API (libcuda.so) 的所对应的CUDA版本。</font>
#### <font color="#b48ff4"> CUDA Toolkit </font> 

我们安装CUDA Tookit 之后，输入 nvcc --version 可以得到如下输出结果：
![[Pasted image 20260607052357.png|500]]

这是编译器工具链 nvcc, 和配套的 CUDA Runtime API. nvcc 负责把 .cu 文件编译成 PTX/SASS。这个版本号只要小于驱动支持版本的上限就没有问题。

`nvcc --version` 显示的是我们安装 CUDA Toolkit 的版本，它同时决定两件事
- **编译期**：nvcc 这个编译器本身的版本，以及它默认链接的头文件、能识别的语言特性、能生成的 PTX/SASS 目标架构（比如新版 nvcc 才认识 `sm_90` Hopper）。
- **运行期**：你编译出的程序默认链接的 **CUDA Runtime (libcudart)** 版本——这份 runtime 来自同一个 Toolkit。
所以它不只是"编译时用的版本"，而是**这套 Toolkit 编译+运行配套库的版本**。

<font color="#eb4349">我们平时所说的 CUDA 版本，大多数时候，指的就是 CUDA Toolkit 版本。</font>
#### <font color="#b48ff4">pytorch 里的 cu130 </font>

我们安装 pytorch 的时候，

Pytorch安装的时候，自带CUDA Runtime。Pytorch 版本号中的 cu130, 表示 pytorch 库在编译的时候静态打包了 CUDA 13.0 的 Runtime 库 (libcudart等) 和 cuDNN / cuBLAS 等。

所以，<font color="orange">Pytorch 不依赖你系统的 CUDA Toolkit，它用自己捆绑的那套</font>。<font color="deeppink">这就是为什么有的系统中，没装 CUDA Toolkit， pip安装的 pytorch 照样能用GPU，只要驱动在就行(libcuda.so)。</font>只要 PyTorch 捆绑的 runtime 版本 ≤ 驱动上限，就能跑。系统 Toolkit 的 nvcc 版本与 PyTorch 能否运行**无关**

## <font color="grape">CUDA Runtime</font>

CUDA Runtime 是一个共享库，Linux上是 `libcudart.so`, 提供在 .cu 代码里调用的那些`cudaMalloc`、`cudaMemcpy`、`cudaLaunchKernel`、`cudaDeviceSynchronize` 之类的 API。你写的 `kernel<<<grid,block>>>()` 这种三尖括号语法，nvcc 会把它翻译成对 runtime 库函数的普通调用。<font color="orange">所以 CUDA Runtime, 本质上是一个动态链接库</font>。

libcudart 随 CUDA Toolkit 一起安装。同一个 Toolkit 还包括 cuBLAS, cuFFT 等库(cuDNN不在包里)。libcudart 本来就属于 Toolkit，所以它没有独立的版本号，CUDA Toolkit 13.2 里的 libcudart 就是13.2. 所以 "CUDA Runtime 版本" 和 "CUDA Toolkit 版本" 在数字上是一回事，只是指代库还是指代整套工具的区。

nvcc 编译出的可执行文件，并不是一个能独立运行的完整GPU程序。它需要两层运行时支撑：
- <font color="#45c36e">CUDA Runtime (libcudart)</font>, 这是一个动态链接库，即我们生成的可执行文件里，没有 cudaMalloc等实现，它是运行时动态链接到 libcudart.so中去找的。它是动态程序依赖运行环境里的库这样的模式。
- <font color="#45c36e">GPU设备执行代码</font>。我们代码分两份，跑在CPU上的 host 代码，和跑在 GPU 上的 device 代码(kernel). device代码不能编译成 x86 指令，它要在GPU上执行。nvcc 把它编译成 **PTX**（一种虚拟 ISA，类似字节码）和/或 **SASS**（特定架构的真实 GPU 机器码），嵌在可执行文件里。
  程序运行时，CUDA Runtime 调用**驱动**（libcuda.so / Driver API），由驱动把这些 PTX/SASS 加载到 GPU、分配显存、启动 kernel。GPU 是一个独立的处理器和独立的地址空间，CPU 没法 "直接执行" GPU 上的指令——必须通过驱动这个中间层把工作提交过去。

<font color="#eb4349">CPU 端的指令直接跑，但 GPU 端的 kernel 必须经由 runtime → driver → GPU 这条链路才能被真正调度执行。</font>
```bash
你的 .cu 程序
  ├─ host 代码  → 直接在 CPU 跑
  └─ device 代码 (PTX/SASS)
        ↓ 通过
     libcudart (Runtime API，Toolkit 提供，你链接的)
        ↓ 调用
     libcuda  (Driver API，驱动提供，对应 nvidia-smi 那个上限)
        ↓ 提交给
     GPU 硬件执行
```

<font color="#eb4349">libcudart（Toolkit/PyTorch 那层, 即CUDA Runtime）必须能被 libcuda（驱动那层）支撑，即 runtime 版本 ≤ 驱动上限。两层各司其职，runtime 是你程序直接调用的上层 API，driver 是真正操作硬件的底层。</font>

注意⚠️：语法翻译是由nvcc干的，不是runtime。runtime负责在运行期间提供函数的实现。但runtime 不负责 GPU 代码的执行。函数执行过程中，Runtime 只是把请求转交给 driver，由driver实际操作硬件。runtime 是上层封装，driver 才是真正碰硬件的那层。runtime 自己不直接碰 GPU。


> [!NOTE] 总结
> - <font color="#45c36e">libcudart 是一个动态链接库，提供</font> `cudaMalloc` / `cudaMemcpy` / `cudaLaunchKernel` <font color="#45c36e">等函数的实现。</font>
> - <font color="#45c36e">nvcc 在编译期把 CUDA 特殊语法翻译成对这些函数的普通调用，并把 device 代码编成 PTX/SASS 嵌进可执行文件。</font>
> - <font color="#45c36e">运行期：host 代码直接在 CPU 跑；遇到需要操作 GPU 的调用（启 kernel、分配显存等），落到 libcudart，libcudart 再调用 libcuda（driver），由 driver 把 kernel 和数据真正提交给 GPU 执行。</font>

一句话：**nvcc 管翻译，runtime 管封装与转发，driver 管硬件。** 三层分工执行。

## <font color="grape">CUDA Toolkit 中的内容与层级关系</font>

CUDA Runtime 是 CUDA Tookit 中其中一个动态链接库，还有 cuBLAS、cuFFT 这两个（还有 cuSPARSE、cuRAND、cuSOLVER 等）。他们确实和 libcudart 一样，是 **CUDA Toolkit 自带的动态链接库**，版本随 Toolkit 走（Toolkit 13.2 → cuBLAS 就是配套那版）。它们是 NVIDIA 提供的高性能数学库（GEMM、FFT 等），底层同样通过 runtime/driver 调度 GPU。

#### <font color="#b48ff4">单独的cuDNN</font>

cuDNN **不在** CUDA Toolkit 里，是**独立分发、独立版本号**的库，需要单独下载安装。它有自己的版本线（如 cuDNN 9.x），只标注 "兼容某个 CUDA 主版本"。它是面向深度学习的（卷积、pooling、normalization、RNN/attention 等算子）cuDNN 是完全独立的组件。

他们的层级关系是如下
```
cuBLAS / cuFFT / cuDNN   (高层算法库)
        ↓ 都依赖
     libcudart (Runtime)
        ↓
     libcuda (Driver)
        ↓
       GPU
```

即三者在逻辑分层上同级——都是建立在 Runtime 之上的高层库，自己不直接碰硬件。区别只在**打包归属**：cuBLAS/cuFFT 跟 Toolkit 绑在一起，cuDNN 独立。

<font color="orange">pyTorch 的 `cu130` wheel 通常把 **cuDNN、cuBLAS 等一并捆绑**进去了，所以你 pip 装 PyTorch 时既不用单独装 Toolkit，也不用单独装 cuDNN——它打包了自己需要的整套高层库，只留驱动这一项依赖在系统侧。</font>

## <font color="grape">切换 CUDA </font>

安装完CUDA Toolkit 之后，在 /usr/local 中可以看到
![[Pasted image 20260607053752.png]]

左边的cuda是一个软连接，它指向一个真实的 cuda 目录，这里是就是右边的 cuda-13.2。

![[Pasted image 20260607054252.png]]

我们可以通过修改配置文件 `~/.bashrc`，在里面加入
![[Pasted image 20260607054613.png|500]]
来指定CUDA的版本。这里是直接指定的软连接，如果把软连接指向其他的 cuda版本如 cuda-13.0, cuda-12.8 等。它自然就使用的是那个版本的cuda了。也可以把软连接换成真实cuda版本的绝对路径，不过如果需要更改使用的cuda版本，就需要重新修改 ~/.bashrc 配置文件。如果使用的软连接，那么我们就只需要修改软连接指向的 cuda 版本号。

## <font color="grape">CUDA 版本号</font>

当人们笼统说 "CUDA 13.2"、"装的是 CUDA 13.2"，**通常**指的就是 **CUDA Toolkit 13.2**，但严格讲，"CUDA 版本" 是个总称：
- **Toolkit 版本**：`nvcc --version` 显示的，编译工具链 + Runtime 那套。这是 "CUDA 13.2" 最常指的对象。
- **Driver API 版本**：`nvidia-smi` 右上角那个上限，由驱动决定。
- **Runtime API 版本**：libcudart 的版本，随 Toolkit。

所以更精确的说法是：**"CUDA X.Y" 是一个版本族的代号，Toolkit 是它最常见的具体指代；但同一时刻你机器上 driver 那层的 CUDA 版本可能是另一个数字。** 它代表的是驱动支持的CUDA版本上限，只要 CUDA 版本比它小，驱动就可以提供支持。

## <font color="grape">区分 libcuda 与 libcudart </font>

libcudart 是 **Toolkit 那层**的库，是你的程序去链接的上层封装。驱动那层是 **libcuda（Driver API）**，是真正操作硬件的底层。这是两个不同的库、两层不同的东西，不要把 Runtime 安到驱动头上。

所以 nvidia-smi 右上角那个版本号精确说是：**该驱动内置的 Driver API（libcuda）所对应的 CUDA 版本**，也就是这个驱动能支撑的 CUDA 上限。

它和 Runtime 的关系是 "谁能撑得起谁"：

```
libcudart (Runtime, Toolkit/PyTorch 那层)   ← 你程序调用的上层
     ↓ 调用
libcuda   (Driver API, 驱动那层)            ← nvidia-smi 右上角显示的就是这层的 CUDA 版本
     ↓
    GPU
```

nvidia-smi 显示的是 **driver 那层（libcuda）的 CUDA 版本上限**；Runtime（libcudart）是另外一层、另一个版本，只要 ≤ 这个上限就能被支撑运行。一句话：那个数字是**驱动/Driver API 的能力上限，跟 Runtime 不是一回事**。