# 专题一：GPU 线程层次结构

# 1. SM 和 CUDA Core

NVIDIA 把 一个**「完整的独立控制单元 +  ALUs 计算单元 + 上下文存储池」**叫做一个**「核 Core」**。

![image.png](%E4%B8%93%E9%A2%98%E4%B8%80%EF%BC%9AGPU%20%E7%BA%BF%E7%A8%8B%E5%B1%82%E6%AC%A1%E7%BB%93%E6%9E%84/image.png)

因为「核Core」的概念是属于 CPU，为了区分，NVIDIA 把一个「核Core」叫做 **「SM, Stream Multi-processor, 流多处理器」**。

SM 里面的一个ALU，NVIDIA 把它叫作一个**「CUDA核」**或 **「CUDA Core」。**由于 ALU 或 CUDA Core 的核心作用就是处理指令流，所以 CUDA Core 也叫做 **「流处理器，Stream Processor」**。而一个 SM 里有多个流处理器，所以叫做 「流多处理器」，这是 SM 名字的由来。

# 2. Thread 与 CUDA Core

**「Thread 线程」**就是我们编写的一段完成特定任务的代码。线程是这段代码或程序，实际执行的实体，它的硬件层面的执行资源对应一个 CUDA Core。从逻辑上可以认为一个 CUDA Core 执行的是一个 CUDA 线程对应的那份运算。CUDA Core 是执行 Thread 的实体硬件资源。

# 3. SM 内部分组 / SM 子分区(SM Subpartition, SMSP)

一个流多处理器 SM 内部，会把 CUDA Core 进行分组，每一组有自己控制单元。一个 SM 内可能有多个 「控制单元 + CUDA Core」的分组，分组之间是共享上下文存储池的。比如 GTX 480 

 

![image.png](%E4%B8%93%E9%A2%98%E4%B8%80%EF%BC%9AGPU%20%E7%BA%BF%E7%A8%8B%E5%B1%82%E6%AC%A1%E7%BB%93%E6%9E%84/image%201.png)

它就有两组。这样的一个组，叫做 「SM Sub-partition，SMSP」或 「SM 子分区」。SMSP 是 Warp **「实际执行的硬件资源单位」**。

# 4. Warp 与 SMSP

线程调度的最小单位是 **「线程束 Warp」**。NVIDIA 的体系里，一个 Warp 固定为 32 个 Thread，它是被调度的最小单位。

调度 Warp 执行的硬件资源就是 SM 内一个 「控制单元 + CUDA Core」的组，即 「SM 子分区」或「SMSP」。

在执行上下文里，每一个上下文实际上对应的是一个 Warp 的上下文。调度切换，是按 Warp 为单位进行的。

> **总结：Thread 是最小的执行单位，一个 Thread 对应执行的硬件资源是一个CUDA Core。Warp 是最小的调度单位，一个 Warp 对应执行的硬件资源是一个 SMSP，SM 子分区。**
> 

#### **Warp 调度**

一个 Warp 固定 32 个线程，一个线程对应一个 CUDA Core 的硬件执行单位。但这并不意味着一个SM 分区必须是 32 个 CUDA Core, 因为一个 Warp 调度执行可以分多个时钟周期，分时复用一个SMSP(SM子分区)的硬件资源。

例如上面 GTX 480，一个 warp 调度到一个 SM子分区执行的时候，SM 子分区只有 16个 CUDA Core。它会分两个时钟周期来执行，第一个时钟周期一个SM分组执行 Warp 内的 0～15 号线程，第二个时钟周期执行 Warp 内的 16～31 号线程。而两个组，可以同一时间调度给两个 Warp 执行。两个组分别用两个时钟周期执行两个 Warp 。

SM 内的分组，即 SM 子分区，逻辑上一个SM子分区分区有自己的 fetcher/decoder，相当于独立的指令控制单元，但是后来，随着技术的不断进步，多个子分区也可能共享 fetcher / decoder 的硬件资源，这和架构设计有关。但是每个 SM 子分区有独立的 Warp Scheduler 硬件控制单元，它是给Warp 调度执行的。

#### SIMT 与 并发度

在并行程序的执行中，同一份 kernel 程序是同一套指令流。同一个 kernel 的执行过程中，不同的SM 上运行的是同一份程序。各自处理不同的数据，进度和分支也可以各不相同。

并行计算中，现在采用的执行程序的方式是「SIMT(Single Instruction Multiple Thread)」. 也就是同一个 GPU 的多个 SM 上，运行同一套程序，每个Thread 也都是运行这一套相同程序，处理不同的数据。

同一个kernel 中的每个 Thread 都是这个程序的一份执行实例，从 kernel 入口开始，通过线程索引确定自己处理的数据范围，有自己的寄存器，也可以根据条件走不同分支。

这样大量的线程执行同一个 kernel，各自处理不同数据。线程被组织成 Warp, 共享指令发射，由warp scheduler 控制，在多个驻留 warp 之间交替发射指令。执行 warp 的硬件资源就是 SMSP，SM 子分区。

典型的 SIMT 执行方式，就是现代 NVIDIA GPU 执行 CUDA kernel 的主流基础设计。核心机制是：

> 
> 
> - 大量线程 Thread 执行同一份 kernel 程序，各自处理一部分数据。
> - 线程组成 32 线程的 Warp, 以 Warp 为基本单位组织指令发射，即调度。
> - SM 内调度器从驻留 Warp 中选择就绪者, 向执行单元发射指令，通过交错执行掩藏延迟。

注意⚠️： Thread 对应执行计算的硬件是 CUDA Core, Warp 对应执行计算的硬件是 SMSP。

后续架构增加了更灵活的线程调度，Tensor Core 等能力，但没有取代这个基础模型。上面说的 kernel 程序是一次启动 kernel 执行的原理。整块 GPU 也可以并发运行不同的 kernel 程序。

# 5. Block 线程块 与 Shared Memory

我们在 kernel 里面会指定线程组织方式，包括 Grid 线程网格, block 线程块，thread 线程的大小和索引方式。

GPU 存储层次架构里面，Shared Memory 是 SM 内部的片上存储器。物理上是 SRAM，和寄存器使用同样的材料。每个 SM 都有自己的 Shared Memory, 物理上它是属于 SM 的。

同一个 SM 上会驻留多个 Thread Block, 每个 block 会划分一部分 Shared Memory 作为该 block 专属使用。划分给这个 block 的 Shared Memory 只在 block 内部的线程间共享使用，它是属于这个block的，block 内的线程，包括不同 Warp 间的线程，都可以访问该 block 的 shared memory, 主要用于线程间共享数据存储和通信。shared memory 说的就是 block 内线程共享的 memory。
不同 block 间的数据通常相互独立，即使运行在同一个 SM 上，也不会自动共享彼此的数据。

**调度**

线程的调度依然是以 Warp 为单位，发射指令。一个 block 是会分配到一个 SM 上执行，一个 block 不会垮 SM，可以认为 block 的调度单位是 SM。

一个 Block 整体分配到一个 SM 上执行，一个 SM 上可能驻留多个 block。block 和 SMSP(子分区)没有对应关系，意思就是 block 内 Warp 的硬件执行，并不一定在一个 SMSP上。事实上，典型的做法是把 block 内的 Warp，平均分散到 SM 上的多个 SMSP 上执行。因此，一个 SMSP 里 Warp Scheduler 调度执行的是多个 block 的 Warp. 

> 例如：一个 SM 里驻留 Block A, Block B, 每个 block 里假设有 4 个 Warp. SM 内也有四个SMSP。可以这样分配
> 
> 
> 
> | SM 子分区 | 分配的 Warp |
> | --- | --- |
> | SMSP 0 | Block A-Warp0、Blcok B-Warp 0 |
> | SMSP 1 | Block A-Warp1、Blcok B-Warp 1 |
> | SMSP 2 | Block A-Warp2、Blcok B-Warp 2 |
> | SMSP 3 | Block A-Warp3、Blcok B-Warp 3 |

# 6. 线程层次总结

线程的层次可以从两个角度看，「编程模型逻辑(软件层面)」 和「执行调度(硬件层面)」。

```
编程模型：Grid --> Block --> Warp --> thread
硬件调度：GPU  -->  SM   --> SMSP --> CUDA Core
```

> ⚠️Warp 比较特殊，一个 Warp 固定 32 线程，在编程模型上其实不能操作，但是它是硬件调度执行的最小单位。
> 
- 一个 kernel 通常包含一到多个 Grid，到一个 GPU 上执行。GPU 可以并发执行多个 kernel 和多个Grid。kernel 或 Grid 对应的硬件执行单元，就是整个 GPU。
- 一个 Grid 分为多个 block, 每个 Block 分配到一个 SM 上执行。一个 SM 内可以驻留多个 block。所以，block 对应的硬件执行单元就是 SM。
- block 内的 warp 共享一个 shared memory, 它是划分给 block 专属使用的。但是，block 内的Warp 是分散到多个 SMSP 上驻留调度执行。SMSP 调度的基本单位就是 Warp。SMSP 是 Warp 的硬件执行单元。
- warp 内的线程 thread 是一个完整的 kernel 执行实例，执行逻辑是完整的，只是不同 thread 处理不同数据。 一个 thread 对应的硬件执行单元就是一个 CUDA Core. 一个线程要执行的时候，会从 Register File 申请一些寄存器。寄存器专属分配给线程使用，但是不专属于 CUDA Core, CUDA Core 只是执行的物理计算单元。

**总结：GPU 上执行整个kernel 里面的Grid。一个 Grid 分为多个 Block，一个 Block 内部共享 shared memory 划分的 block 专属分区，一个block 分配到一个 SM 上执行。block 内的Warp，分散到 SM 内的多个 SMSP 上去。SMSP 调度 Warp 执行。 Warp 内的线程，实际执行单元是 CUDA Core. 典型 SIMT 任务下，每个 thread 都是一个完整的 kernel 执行逻辑, 只不过不同的 thread 负责处理不同的数据，可以有自己的执行进度，和自己的执行条件分支**。