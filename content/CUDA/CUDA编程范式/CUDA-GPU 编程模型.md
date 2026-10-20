# 5. CUDA-GPU 编程模型

# 1. CPU 与 GPU 的交互模式

![image.png](5%20CUDA-GPU%20%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B/image.png)

![image.png](5%20CUDA-GPU%20%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B/image%201.png)

CPU 和 GPU 之间有相互独立的物理空间。

上面图左边部分是 CPU 系统，它有自己的处理单元，核心，有独立内存，还有一些接口芯片Chipset。右边是 GPU，它具有大量的处理芯片，同时 GPU 也有自己的存储系统，我们称为 GPU 显存。GPU 显存现在常用的技术叫做 **GDDR， Graphics Double Data Rate**。它的带宽比较大，速度也比较快。此外，GPU上通常有一个叫做”金手指”的接口，这就是 PCIe 接口。

CPU(我们也可以叫做主机系统) 与 GPU 之间就是通过 PCIe 接口连接起来的。PCIe 2.0 / 3.0, 大概能获得 8GB/s 到 16GB/s 的带宽。这个带宽算是比较大的，但是 CPU 与 GPU 之间交互的开销也很大。要注意数据必须通过 PCIe 总线，在 CPU 和 GPU 之间来回传递。

# 2. GPU存储器层次结构

GPU 中除了自己有硬件处理单元之外，很重要的就是 GPU 上的各个存储单元。各存储单元对应不同层次。(下图是 OpenCL 的图，CUDA 和 OpenCL 在很多方面都是一致的，学习 CUDA 之后，理解OpenCL 就会非常快)

![image.png](5%20CUDA-GPU%20%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B/image%202.png)

<aside>
💡

注意⚠️：

上图是 OpenCL 示意图，架构和 NVIDIA CUDA 逻辑上大体一致，但是还是有一差别

- work item 等同于 CUDA 架构中的 CUDA Core 或 ALU。
- Private Memory 等同于 CUDA 架构中的 Register File，是给线程用的寄存器。
- Compute Unit, CU 等同于 CUDA架构中的 SM。它们都是一个完整的计算单元或核。
- Local Memory, 等同于 CUDA 架构中的 Shared Memory, 它在每个 CU 或 SM 内单独存在的「片内存储器」，它只在 CU 或 SM 内部共享。用做上下文切换等。
⚠️注意：CUDA 架构中也有 Local Memory，但它指的是片外显存上划分给线程独占的一段空间，用于存放片内寄存器溢出的数据。它的思想类似于「虚拟内存」。
- Global / Constant Memory Data Cache 是针对显存上的 Global Memory / Constant Memory 在芯片内部设置的 Cache。
- Global / Constant Memory 通过 PCIe 总线和主机端的主存进行交互。
</aside>

在 CUDA 视角下，有以下存储器层次。存储器架构和层次的存在，主要是因为访存速度和成本之间做的一个折中(否则直接做成一块巨大的寄存器就行了，不需要任何层次结构)。访存层次的设计是一个不得已而为之的策略。

> ① Register, 最快的是寄存器，它由硬件支撑(HW)，访存速度是单时钟周期。它是 SRAM「详细查看下面Callout」
② Shared Memory, 共享存储器。它由硬件支撑，也是 SRAM，访存速度也是单时钟周期。它的速度和寄存器是一样的。差别就是寄存器是硬件直接控制，而 shared memory 是可以由用户进行软件层面的控制。「详细查看下面Callout」
③ Local Memory, 本地存储器。它是一个逻辑上的概念，它本质上是属于 SM 中某个线程的，但是它的物理位置在 SM 片外，在显存 DRAM 上。当某线程使用寄存器不够的时候，将其暂存在 local memory 中。现在它是具有 cache 的。「详细查看下面Callout」
④ Global Memory，全局存储器，DRAM，是显存中的一部分，也有若干层cache。
⑤ Constant Memory, 常量存储器，DRAM，也是显存中的一部分，也是有cache。它是一段固定的存储器，它的作用是提供给多个线程使用的，不变化的数据，对 GPU 它是只读。只读就没有任何冲突和同步问题，访问速度较快。对于 CPU 的 Host 端代码，可以对它进行读和写操作。GPU device 端只能进行读操作。
⑥ Texture Memory, 纹理存储器。DRAM，它也是存在于显存 DRAM 里面的。它也有cache.这些 cache 是针对一些不规则的杂乱的小数据访问的灵敏性支持。
⑦ Instruction Memory, 指令存储器。它也是放在显存 DRAM 里面。但是它由系统进行管理，对我们程序设计者是不可见的，主要是存放一些指令。也是有缓存。
> 

<aside>
💡

**浅谈 Shared Memory 与 Local Memory**
- Shared Memory 是在每个 SM 中单独存在的一个共享存储空间，它是 SRAM。Shared Memory 在一个 SM 中，会以 Block为单位划分，一个 Shared Memory 区域划分给一个block 后，它就只能供这个block 中的线程共享，主要用于缓存数据和提供线程间通信。它是 SM 内部的高速缓存区。或者叫「片内存储」。它和一级缓存，二级缓存的空间选择上，有一个取舍问题。

- Local Memory 是在显存(DRAM)上的一块虚拟空间。它以线程为划分，一个区域划分给一个线程之后，它就只能供这个线程使用，用来存放该线程的上下文和一些寄存器溢出的数据。它是专门供某个线程使用的。它在显存上，是一个片外的存储区。

**总结**：shared memory 是片内高速共享存储区，每个 SM 都有自己的 shared memory。shared memory 上按照 block 划分，每个 block 获得一个 shared memory 分区。block 内的线程可以共享这个 block 的 shared memory 区域，用于存储共享数据和线程间通信。
Local memory 是片外的虚拟存储区，物理上存在于显存上，逻辑上属于某个线程。它由线程独享，其他部分不能访问。它的作用是存储线程上下文，寄存器溢出的数据。

**shared memory 是 block 内线程共享的 memory，所以叫 shared。local memory 是线程独占的一片 memory，所以叫 local to thread，它在片外显存上。这些存储区域命名，都是是站在线程角度的。**

</aside>

<aside>
💡

**浅谈寄存器 Register /  寄存器堆 Register File**

每个 SM 中有一个寄存器阵列，一大块 SRAM。寄存器阵列在 SM 中叫做 **Regsiter File,寄存器堆**。注意⚠️：这里的 File 应该翻译成堆，而不是文件。  

寄存器堆或寄存器堆中的寄存器，不属于任何一个 CUDA Core。只是在执行线程的时候，按需分配给线程，用完就释放。

**这里最关键的概念就是 CUDA Core 是无状态的，所有状态都在外部，寄存器是线程状态，不归Core所有**。

寄存器池会按照 SM sub-partition 划分，每一个 partition 内的 Warp 可以按需申请寄存器，所以可以简单认为一个大池子被分成了几个小池子。每一个 sub-partition 有一个自己的小寄存器池，用来存放线程上下文状态。由于 SMSP 的调度单位是 Warp，划分寄存器池按照 SMSP 划分，但是在 SMSP 内部，分配给线程时，是以 Warp 为颗粒度进行分配。

CUDA Core 就根据这些状态执行程序，CUDA Core 本身是无状态的。

Turing 或 Ampere 的 SM，寄存器堆有65536个寄存器(256KB SRAM)有4个sub-partition，每个 partition 16384 个寄存器。假设一个线程需要 32个 寄存器，一个block有256个线程，一个 block 需要 256 * 32 = 8192个寄存器。整个SM可以容纳 8个 block。一旦一个 block 被调度到某个SM上，它的每个线程就会被分配到某个寄存器编号区间。结束执行之前，都不会改变。

**关于SMSP，SM Sub-Partition**

在 SM 内部的寄存器堆会划分为若干 sub-partition，是出于工程简化的考虑。每个partition内部有一组独立的控制硬件，独立的 warp-scheduler。调度 warp 只在 partition 内部进行，这样的 warp 切换成本为 0。将整个寄存器堆分为若干组，多组硬件分别管理不同的寄存器组，在组内 0 成本切换 warp。这样的策略降低了工程复杂度。

注意⚠️：线程和 Warp 是全程独占这些寄存器的，所以它在硬件层面上的切换成本才为0.

sub-partition是硬件工程考虑的产物，block进入SM后，是按照线程分配寄存器编号。调度是调度器所在 partition 内部按 Warp 为单位进行调度，它感知不到 block 的存在。

</aside>

<aside>
💡

**浅谈寄存器堆 Register File 与 Shared Memory**

Register File 和 Shared Memory 都是 SM 内部的片上存储器，实现上都是采用的 SRAM, 可以看成两块独立的 SRAM。但是它们实现和使用并不完全相同，所以它们速度并不相同，register 更快，因为至少 Shared Memory 要独立寻址。

Register File 是专门分配给线程使用的，为线程提供操作数，程序控制，上下文状态等，但是它不专属于某个 CUDA Core，而是在线程到来后，分配给线程专属使用，CUDA Core 只是在执行线程的时候，会用到该线程的 Register File。CUDA Core 本身是无状态的。

Shared Memory 是划分给 block 专属使用的。block 内的线程在划分给它的 shared memory 内共享数据和通信。block 内的 Warp 共享划分给这个 block 的存储区域，即使这些 Warp 分散在不同的 SMSP 上，整个 Shared Memory 都在 SM 内部，SM 内部的 SMSP 都可以访问到。

假设一个 SM 同时驻留两个 block, A 和 B。A 和 B 中的每个线程，都各自有自己的寄存器，寄存器分配按线程分配。A 和 B 各自从 Shared Memory 中划分一个区域来使用，A 中的线程可以共享 Block A 的 shared memory存储区，B 中的线程可以共享 Block B 的shared memory存储区。然后 A 和 B 之间不能共享。

可以理解 Register 和 Shared Memory 是两套不同的 SRAM 存储资源，但实现上，不一定是两个完整的大阵列。

</aside>

#### GPU架构回顾(Maxwell 内核)

![image.png](5%20CUDA-GPU%20%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B/image%203.png)

每一个 SM 里面有很多小的处理单元(每个绿色颗粒是一个处理单元，即CUDA核)，这些处理单元被分在了若干组，每个组有自己的调度器。这样的一个组，叫做 SMSP，SM Sub-Partition, 子分区。

左边的图就是一个完整的SM结构。这个SM里面分了4 组处理单元，每组处理单元里有 32 个处理单元， 还有一些共享资源。

右图中，可以看到，每个芯片上有 5 个 SM，从左往右，每个 SM 里有 4 个 SMSP, 上下各两个。除了SM之外，还有片内的缓存，L2 Cache。L2 Cache 下面是 Memory Controller, 存储控制器，用于控制，这是和显存交互的控制逻辑。 顶端还有 PCIe 3.0 Host 接口，它可以直接和主机端通信。

我们编写的程序，最终都会映射到 SM 上去实际执行。真正干活计算的是 SM 里面一个个小的 CUDA Core，上面一个个小的方块。

# 3. GPU 线程组织模型

![image.png](5%20CUDA-GPU%20%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B/image%204.png)

这是一个简化的线程组织模型

![image.png](5%20CUDA-GPU%20%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B/image%205.png)

- **线程 Thread**: 线程是执行代码段的实体。它完成一个特定任务。一个 kernel 函数启动后，可能创建若干的线程执行运算，每个线程执行的都是完整的 kernel 函数逻辑。线程执行的程序片相同，但处理不同数据，也可以有自己的执行进度和条件分支。线程是 kernel 启动时，产生的运行实体。
- **线轴/线束 Warp**：翻译过来是线轴，GPU 中我们叫做线束，它是线程的组合，现在一个 Warp 大小固定由 32 个连续的线程组成的。
- **线程块 Block**:  若干线程组成一个block，线程块, 为了便于组织。线程块是分配资源的基本单位。一个线程块，会分配到一个 SM 上执行，但是线程块中 warp 一般会分配到不同 SMSP上执行。SM 中的 shared memory 会按 block 分配给 block 专属使用。
- **线程网 Grid**：若干 block 组成一个 Grid，线程网。kernel 核函数启动时，就会生成一个Grid。Grid 里面包含了若干 block, block 里又包含了若干thread。具体有多少线程，多少 block，每个 block 中有多少线程，是开发者在实际应用中根据问题来指定的。

总结：一个 kernel 启动，含有大量线程，一个 Kernel 就是执行在 GPU 上完整的一段代码，一个kernel 的执行，就会启动一个线程 Grid。这些线程被分成了若干不同线程块 block，线程块内部有若干线程。线程块内部有一些共享的资源, 比如 Shared Memory. 线程块中的线程可以同步syncthreads()访问共享这些资源。

Kernel 在启动 Grid 的时候，Grid 中包含若干线程块，每个线程块 block又包含若干线程。具体Grid包含多少线程块，线程块又包含多少线程，是由开发者来设定的。

**线程和线程块都有唯一标识。**

#### GPU  线程映射关系

我们编写的代码，最终要放在 GPU 硬件上执行，这个过程有一个映射关系。以前我们编写程序，默认在 CPU上执行，现在要在一个不同的硬件上执行，就需要一个调度，调度给程序需要的资源，资源的对应关系体现出来就是映射。

![image.png](5%20CUDA-GPU%20%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B/image%206.png)

- 软件的一个基本线程，是会被分配到 GPU 上的一个处理单元上，这个处理单元是一个基本的处理单元，叫做线程处理器。也就是我们常规理解的ALU单元或者CUDA Core。它是能够执行完整线程的最基本的单元。
- 一个线程块，就被分配到一个 SM 上。
- 整个 Grid 就被映射到一个完整的 GPU 设备上。

一个软件在 CPU 上同样存在映射关系。一个线程映射到 CPU 内最基本的计算单元，标量处理器。一个线程块映射到一个矢量处理器上，整个线程 Grid 映射到整个CPU的多核系统中。

例如，我们所说的CPU是 4 核 8线程，就是一个程序线程映射到一个线程处理器上(也就是标量处理器)，一个线程块映射到一个核中，一个核中有8个线程处理器。该程序的所有线程，被映射到整个CPU 的 4 个核上。也就是共有 32 个线程在处理。

#### 线程和存储器之间的关系

![image.png](5%20CUDA-GPU%20%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B/image%207.png)

**[线程存储空间]**: 每个线程有自己私有的存储空间，它有 **两部分一部分叫做寄存器，另一部分在显存中叫做Local Memory**。线程私有的 local memory 不会跟别的线程共享。 注意⚠️，线程的local memory是在显存中划分出来的一个区域。它是片外存储。

**[线程块存储空间] 每个线程块，都有一个共享存储区, Shared Memory**。注意⚠️，shared memory是片内存储，每一个SM都有自己独立的 shared memory 的物理硬件存储区域。不过在 SM 处理计算的时候，会划分给不同的 block。划分给这个 block 的 shared memory 区域，可以在这个 Block 内的线程之间共享。

**[全局共享的显存]** 整个 GPU 有一个共享 Global Memory，**这就是显存**。Global memory 就是一个全局的共享存储器，可以供全局所有线程共享，或者供整个 Kernel 空间内使用。

一个GPU设备上可以同时跑多个 kernel, 那么它也会在不同的 kernal 间共享。只要把显存根据 kernel进行划分，每个 kernel 就可以使用自己在显存上的划分区域。

**[主机内存与GPU显存]** 然后我们一个主机系统的存储器，可以和多个 GPU 设备的存储器进行交互。逻辑上，这些 GPU 设备共享主机的内存。

下面就是线程和存储器之间的关系

![image.png](5%20CUDA-GPU%20%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B/image%208.png)

注意⚠️，Constant Memory 对于设备端来说是只读，对于主机端可以去读写。

# 4. 编程模型

常规意义上，GPU 用于处理图形图像，这是它的出发点。图形图像跟通用并行计算的处理模式非常一致。常规的图形图像处理，操作像素和像素块。对每个像素和像素块的操作都类似。例如做光照的渲染时，每一个像素都要算相同的方法，这就是天然的并行性。GPU 就是设计成天然的这种并行处理系统。它可以很好的应用到 SIMD 这样的计算结构中去。

SIMD，我们可以简单理解为**数据并行分割**的过程。当然 SIMD 是一个底层的指令级别的理解。更多使用的术语是，**STMD，Single Thread Multiple Data**。

在 GPU 中，它叫做**SIMT(Single Instruction Multiple Thread)**，**也就是GPU版本的SIMD**。**它是一个线程模型，不再是指令模型**。
但是本质逻辑是一样的。比如，线程要不断切换以获得延迟隐藏。多个线程执行相同的指令流，GPU 上大量的线程在调度。

CUDA 编程使用的语言，我们可以认为它就是一个扩展了的 C 语言，它的基本语言就是C语言，但是增加了一些关键词，或者函数调用模式，称之为 **前缀修饰词**。

首先我们要指明存储器是 GPU 上的。CPU 和 GPU 的存储空间是完全分开的。那我们如何知道哪个变量在 CPU 上，哪个变量是在 GPU 上呢？通常，不声明的时候，我们默认变量是在 CPU 上。如果变成 GPU 的变量，我们就需要在函数前面加上一些修饰词。这些修饰词代表放在 GPU 存储器上。

为了进行 GPU 线程的索引和一些底层操作，GPU 也提供了一些关键词和内置变量。还提供了一些GPU 上运行期的 API，通过一些函数在 GPU 上做特定的操作。

调用 GPU 函数有一些不同，需要提供一个特定的配置，告诉 GPU 这个任务要由多少线程去做，有多少线程块去做。通过  <<<N, M>>>  的形式。

关于 CUDA 函数的声明，最基本的包含以下三类

**__device__ float DeviceFunc()** 执行位置: device(GPU)。调用位置：device(GPU)。它代表这个函数要在 GPU 上执行，同时它的调用位置也在 GPU 上，也就是只有 GPU 上的函数才能去调用这个函数。这个函数也只能在 GPU 上执行。

**__global__ void KernelFunc()**   执行位置: device(GPU)。调用位置：host(CPU)。这就是我们常规上理解的kernel函数，它就是一个入口函数，也就是说，它在 CPU 上调用，在 GPU 上执行。

CPU上的代码执行到一定程度，调用这个入口函数，入口函数就会在 GPU 上启动起来我们需要的一些线程。当然，调用时需要提供一些配置。我们要通过入口函数的 <<<N, M>>> 告诉GPU，需要启动多少个线程块， 每个线程块多少个线程等等。入口函数必须返回 void 类型。

**__host__ float HostFunc()**  执行位置: Host(CPU)。   调用位置:   host(CPU)。只能在主机端执行，只能在主机端调用。

**__device__and__host__** 可以同时使用，同时修饰一个函数。这就代表，这个函数在主机端和设备端所有的代码是一样的。可以理解为，设备端肯定是由若干线程去执行的，每个线程执行的东西，跟host 端是一样的。