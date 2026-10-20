# 6. CUDA 编程(一)

GPU 特别适用于，密集计算，高度可并行计算。它的出发点是图形学。晶体管主要用于执行计算，而不是缓存数据，或者控制指令流。

CPU 和 GPU 两者架构示意图如下，图中能够看到 GPU 和 CPU 各自晶体管的用途：

![image.png](6%20CUDA%20%E7%BC%96%E7%A8%8B(%E4%B8%80)/image.png)

上图绿色的部分是主要用于计算的 ALU 部分，CPU 的 ALU 部分所占比例并不大，GPU 用来做控制逻辑和缓存的部分很少，大量的芯片面积是用来做数据处理的。

# 1. 通用并行计算GPGPU与CUDA

2001/2002，研究人员把 GPU 当作数据的 并行协处理器，开始通过对 shader 进行编程，来处理一些特定的计算任务。**GPGPU(General Purpose Graphics Processing Unit)**，这个领域从此诞生。把 GPU 从最初图形图像计算的目的，推向通用计算，这是往前进了一大步。

2007年，Nvidia发布了**「CUDA，全称 Compute Uniform Device Architecture, 即统一计算设备架构」**。在这个架构下，把 GPU 抽象成为一个可以进行通用计算的平台，所以 GPGPU 就由原来的协处理器，发展成为一个完整的进行并行计算的，称为**GPU Computing 的完整结构**。
2008年，Khronos发布了OpenCL规范，它支持把整个通用并行计算，扩展到更多的设备平台上去。

# 2. CUDA基本信息

CUDA 包含了很多线程性的东西。在GPU的平台上，它包含一些 **层次化的线程集合，A hierarchy of thread groups**. 同时这些线程具有 **共享存储(Shared Memories)** 以及 **同步的特性(Barrier synchronization)**

CUDA 里面有两个独立的视角，一个是 Host, 一个是 Device。

**Host, 即主机端**，通常指 CPU，采用 ANSI 标准C语言编程。

**Device，即设备端**，通常指 GPU。该设备上通常是一些数据可并行化的代码。采用 ANSI 标准C的扩展语言编程。除了个别修饰符，本质上就是C语言。

**Host 和 Device 端有一个重要的分割点，就是他们拥有各自独立的存储器**。

CUDA编程，包括**「主机端」**和**「设备端」**两部分代码。 Host 与 Device 具有各自的存储器，这是他们重要的分割点。随着整个架构的进步发展，存储器编址的方式以及访问的方式，慢慢会进行一些为了便利的统一，但是从物理上来说，主机端和设备端的物理存储器空间，现在是分开的。了解这点，对我们进行整体的数据管理和数据存储分布的设计，至关重要。

**kernel，核函数**，指的是数据并行处理函数。kernel 函数，实际上是在「设备端去执行并发的函数」。在主机端通过调用 kernel 函数，就会使得整个 CUDA 系统，在设备创建大量轻量级的线程。这些线程是由硬件管理的线程，跟我们常见的软件的线程不太一样，由硬件负责创建和调度。

下面是一个简单的 CUDA 核函数(kernel)

```c
//kernel definition
__global__ void Vecadd(float *A, float *b, float *C){
	int i = threadIdx.x;
	C[i] = A[i] + B[i];
}

int main(){
	...
	// kernel invocation with N threads
	vecAdd<<<1, N>>>(A, B, C);	
}
```

CUDA 程序的结构跟标准的 C语言本质上没有什么不同。但是它多了一些修饰词。这些修饰词，通过编译系统翻译，能够把这段代码，翻译成 GPU 系统可以去执行的机器码。这段代码，跟常规写的 C语言代码，最大的不同之处在于，**这段代码是在 N 个不同的 CUDA 线程上并行执行的**。当然这里有一个调度的策略问题。

**__global__** 叫做修饰词或者**声明指定符(Declaration Specifier)**， 它表示这是一个核函数。
**<<<1, N>>>**叫做**启动配置(Execution Configuration)**. 它告诉执行系统，我们需要多少个线程块，多少个线程的配置，来启动核函数。
**threadIdx是线程标志ID**，表明该线程在整个CUDA系统它在哪个部分。整个CUDA系统中，线程的ID有它的编码方式。

CUDA 程序的执行逻辑如下：

![image.png](6%20CUDA%20%E7%BC%96%E7%A8%8B(%E4%B8%80)/image%201.png)

CUDA 程序的执行，有两部分。首先，在CPU上有串行执行的代码，当执行到调用GPU上的核函数的时候，在GPU上会启动 kernel 函数，这个Kernel函数由大量的线程，在GPU上进行硬件并行执行。等结果处理完成之后，把这个结果返回到 CPU 端，然后再继续这样的执行过程，然后就形成**串并串并这样的一个交互的过程**。

# 3. 线程层次Thread Hierarchies

CUDA线程有一个线程组织形式，分为两个层次。第一层叫做 **Gridlock(线程网格)**, 第二层叫做 **Blcok 线程块**。一个Grid里面含有多个Block，一个 Block 里面含有多个线程。

Block(线程块)：线程块是一捆线程，它们可以相互合作，同步(synchronize)执行，同步不冲突的访问 shared memory，通过低延迟的 shared memory, 高效分享数据。shared memory 会以 block 为单位划分一个区域专属给它使用，block 内的线程都可以共享访问这个存储区。

Block 可以有自己的维度，可以是一维，二维或者三维的线程组成的块。Grid 同样有自己的维度，可以是一维或多维的线程块组成的网格。维度主要是用来便于线程或线程块的索引。

<aside>
💡

**线程层次的类比理解**

进行类比，每一个线程相当于公司里面的员工，Block 可以理解成公司里面不同的部门，Grid可以理解为整个公司。每个公司的部门有自己的划分，它可以划分为一维二维。Block就是每一个部门。每个部门内部由一个个员工组成，每个员工就有自己的员工号码，这个员工号码就有一个纬度，一维二维三维，我们可以在部门内部，还进行一个第一小组，第二小组，第三小组….的编制。

**即，Grid内部编排Block的时候，可以按照多维度编排。Block内部编排线程的时候，也可以按照多维度编排**。

</aside>

<aside>
💡

**线程层次的约束条件**

线程层次在划分的时候，有一定的约束条件:
**① 一个Grid里面，每个 Block 的线程数是一样的。
② Block 内部的每个线程，可以同步(synchronize), 以及访问共享存储器shared memory. shared memory是片内在SM内部的硬件存储结构，可划分给Block，供其中线程共享。**

</aside>

# 4. 线程索引

![image.png](6%20CUDA%20%E7%BC%96%E7%A8%8B(%E4%B8%80)/image%202.png)

我们在索引一个线程的时候，首先要在 Grid 中索引到 block， 再由 block 索引到具体的线程号。这是线程编号的模式或者线程的描述方式。这种线程描述方式，有助于组织以及处理特定维度的数据。

线程索引的详细讨论，参考 

[专题三：线程层次与索引](https://app.notion.com/p/3fb1c8a4210380dca810c78ccabb0538?pvs=21)

## ◼︎ block 块内线程索引

![image.png](6%20CUDA%20%E7%BC%96%E7%A8%8B(%E4%B8%80)/image%203.png)

Block 可以是一维，二维或三维的，用图形表示就是 线，面，体 或者 数组，矩阵，立方体。以二维 block 为例, x 轴 和 y 轴的方向如下所示：

![image.png](6%20CUDA%20%E7%BC%96%E7%A8%8B(%E4%B8%80)/image%204.png)

假设图中高亮方块代表的线程，在 block 内的索引是(x, y)。**Thread ID, 线程ID**，用变量 `thread_id` 表示，这是一个标量的标识符(编号), scalar thread identifier. 是线程的线性编号。

 `thread_id` 的计算方法就是 `thread_id = x + y * Dx` , Dx 是 block 每一行有多少线程数(x 的维度，即矩阵列数)。y 是偏移了多少行，x 是在行内偏移了多少个线程。

当使用三维 block 时，block 内的线程索引 (x, y, z) 转化为 thread_id 的计算方法就是 `thread_id = x + y * Dx + zDxDy`。DxDy 表示一层的大小，z 表示在第三维 z 坐标轴上，偏移了多少层。y 代表在一层里面偏移了多少行，x 代表在行内偏移了多少个线程。

![multilayer-grid-transparent.png](6%20CUDA%20%E7%BC%96%E7%A8%8B(%E4%B8%80)/multilayer-grid-transparent.png)

线程索引：每个线程必须要有一个 ID (如同每个员工都必须有一个编号)，我们通过这个编号，才能定位到某个线程。一个 ID 唯一确定一个线程。ID 由几部分组成。每个线程必然隶属于某个block(如同每个员工必然隶属于某个部门)。因此线程有 block 编号，同时线程还有一个 block 内部的编号。

一个线程在 block 内的二维索引转化为一维索引的 Thread ID 的 方式：

<aside>
💡

Thread ID

CUDA 原生的 **「Block 和 Grid 各自最多支持三维」**，二维三维的坐标，是线程在 block 中的位置信息，可以将它转换为一个**「标量的线程编号，Thread ID」**。这个编号是线程在全局线程中的编号。block 是一位二维三维时，我们计算 Thread ID 的方式为:

- **如果block是一维block, Thread ID == Thread index**
- **如果block是二维block(Dx, Dy), Thread ID of index(x, y) == x + yDx**
- **如果block是三维block(Dx, Dy, Dz), Thread ID of index(x, y, z) = x + yDx + zDxDy**
</aside>

## ◼︎CUDA 内置索引变量

CUDA 对 Grid 内部索引 Block，和 Block 内部索引 Thread. 都提供了一维二维三维的索引方式，CUDA 原生支持最多不超过三维，并且 CUDA 提供了一些原生「内置索引变量」，在 kernel 启动时，CUDA 会自动设置这些内置索引变量，我们可以通过读取这些变量，获得索引信息，

| 内置变量 | 含义 | 分量 |
| --- | --- | --- |
| threadIdx | 线程在 Block 内的位置  | x, y, z 对应 `threaIdx.x`, `threadIdx.y`, `threadIdx.z` , 分别代表线程在 Block 中的 x, y, z 坐标。 |
| blcokDim | Block 在每个维度上的大小, 即在某维度上有几个线程。 | x, y, z, 假设 N * M 的二维 block, `blockDim.x` = N, `blcokDim.y` = M, `blockDim.z` = 1 |
| blockIdx | Block 在 Grid 内的位置 | x, y, z 对应 `blockIdx.x`, `blockIdx.y`, `blockIdx.z`, 分别代表Block 在 Grid 中的 x, y, z 坐标。  |
| gridDim | Grid 在每个维度上的大小，即在某维度上有几个 block。 | x, y, z。假设 N * M 的二维 Grid, `gridDim.x` = N, `gridDim.y` = M, `gridDim.z` = 1 |

CUDA 核心提供的内置索引变量就是 `threadIdx`, `blockDim`, `blockIdx`, `gridDim` 这四个变量，它们本质上都是一个结构体，有 x, y, z 三个成员变量，保留了维度信息，可以通过 x, y, z 来访问各个维度分量上的值。`threadIdx` 和 `blockIdx` 存放的是索引信息，它们是 `uint3` 结构体类型的。`blockDim` 和 `gridDim` 存放的是维度大小信息，它们是 `dim3` 结构体类型的。

未使用的维度大小为1，索引为0. 例如二维 block 的 `threadIdx.z` = 0, `blockDim.z` = 1。通过 `threadIdx` 我们可以拿到线程在 block 中每个维度上的坐标, `blockDim` 可以拿到每个 block 在各个维度上的大小。同样，通过 `blockIdx`，可以拿到 block 在 Grid 中每个维度上的坐标。通过 gridDim 可以拿到 grid 在各个维度上的大小。 

## ◼︎线程索引总结

CUDA 中的线程层次有 Grid,  Block, Thread 三个层次。我们每启动一次 kernel，就会产生一个Grid，一个Grid中，有多个block, 一个block中有多个线程Thread。

调用核函数的 kernel<<<nB， nT>>>(args…)，尖括号中的内容被称为**执行配置**，它一共有4个参数，先关注前面两个:

- 第一个(nb)是 Dimension of Grid，即它设置一个 Grid 中有多少block，即 nB, number of blocks。
- 第二个参数(nT)是Dimension of Block，是设置一个 block 中有多少线程，即nT, number of threads。例如kernelFunc<<<4. 8>>>(), 表示启动一个具有4个block的Grid，每个block有8个线程，一共启动 4 * 8 = 32 个线程。注意⚠️，nB 和 nT 可以按照一维，二维，三维…来指定。

CUDA为我们提供了内置变量，主要是两个**Dim (gridDim 和 blockDim)** 和 **两个index (blockidx 和 threadidx)**

> **gridDim**: 一个Grid中有多少个block。它也可以按照一维，二维，三维…等描述。
**blockDim**：一个block中有多少个线程。它也可以按照一维，二维，三维…等描述。
**blockIdx**：它表示当前线程所属的block，在Grid中的编号。同一个block中的所有线程，都是共享同一个BlockIdx(block编号)。
**threadidx**：它表示当前线程在当前block中的编号。
> 

如果我们打印 blockIdx 和 ThreadIdx，会发现线程的 blockIdx 可能相同(在同一个block中)，threadIdx 也可能相同(在不同block中)。说明，**本质上其实 blockIdx 和 threadIdx 都是内部编号。blockIdx是Grid内部对block的编号，threadIdx 是 block 内部对 thread的编号**。

所以，不同block下的线程ThreadIdx是会重复的，单纯通过一个threadIdx，是没有办法在全局情况下去确认一个线程的。但是我们可以通过blockDim, blockIdx, threadIdx 的换算，得到线程的全局唯一编号。如，

```c
int i = threadIdx.x + blockDim.x * blockIdx.x;//得到线程在x分量上的全局编号
int j = threadIdx.x + blockDim.x * blockIdx.x;//得到线程在y分量上的全局编号
```

## ◼︎线程索引深度理解

我们在程序中使用的 Grid 和 Block 的一维索引，二维索引，三维索引都是「逻辑索引」。CUDA提供了两个 Dim (`gridDim`, `blockDim`) 和两个 Index (`blockIdx`, `threadIdx`) 的内置变量。我们可以根据这些内置变量，**按需计算出一维，二维，三维坐标，或者局部、全局标量编号**。它只是逻辑索引，是我们为了将线程和它要处理的数据，对应起来，方便映射而使用的逻辑索引。

**在硬件层面，真正调度执行的时候，它会按照自己的另一套规则定位线程**。而我们上面讨论的Grid, Block，一维二维三维索引，局部/全局标量编号等，都是为了在程序层面方便，方便指定数据和线程之间的对应关系使用的逻辑概念。

但是硬件层面 和 程序层面的索引并不是毫无关联，它们存在一定的映射关系。比如，一个block 放到一个SM上执行，block 内的 warp 分散到 SMSP(SM内子分区)上执行，一个 block 独享一块 Shared Memory, 一个 warp 被一个SMSP的 warp scheduler 调度，独享一个寄存器组。

这种对应关系，CUDA自己会处理，我们只需要把程序层面的索引维护正确就行。 

# 5.线程块 block 独立执行与扩展性

线程块之间彼此是独立执行的，线程块之间可以按任意顺序执行，可以并行也可以串行，可以被任意数量的处理器以任意顺序调度，处理器的数量具有可扩展性。

![image.png](6%20CUDA%20%E7%BC%96%E7%A8%8B(%E4%B8%80)/image%205.png)

例如，我们有一个CUDA程序，它有 8 个block, 它在运行期确定某个block放在某个SM上去执行，如果一个GPU有两个SM(图中左边部分)，我们可以通过4次调度完成8个block计算，每次2个block被调度到两个SM上执行(假设一个SM中一次执行一个block)。如果一个GPU有4个SM，相同的一段程序不做任何变化，通过两次调度完成，每次调度执行4个block。**这就是这种机制的一个优势，如果CUDA线程的Block数足够多，它就可以充分利用多个SM的硬件资源**。

一个线程块内部的线程具有一些重要特性：它们可以共享硬件上的低延迟的存储器, shared memory。它们可以彼此同步(一个部门内的所有成员，可以有一个同步方式)。

注意⚠️：一个 block 分配给一个 SM 后，它会驻留在这个 SM 上直到执行介绍，中途不会更换到其他SM 上。线程的调度，只是在 SM 内部以 Warp 为单位，进行调度。
但是，这种情况并不是绝对的。当有抢占的情况发生时，SM 被其他任务抢占，block 会被迫让出 SM, 但是等到 block 再次被调度的时候，它可能使用任何一个SM。但是，正常情况下，它是会在一个 SM 上执行结束的。 

# 6. CUDA 内存传输

**线程是实际执行的指令流的一个容器**。还有一个重要部分，就是内存中的数据传输。

![image.png](6%20CUDA%20%E7%BC%96%E7%A8%8B(%E4%B8%80)/image%206.png)

存储器具有一定架构。GPU 硬件设备上，有若干不同部分的存储器:

- 包含每一个线程所私有的寄存器(每个部门内的成员，有自己私有的办公桌)。
- 每一个线程块 block，有一部分划分给它使用的共享存储器shared memory(每个部门有公共区域或公共资源)，但是这部门存储器只在block内部共享使用，不与其他block共享(仅在部门内使用，其他部门的人不能使用本部门的东西)。
- 整个GPU具有一个很大的硬件存储器叫做，global memory，显存。所有的线程都可以访问 global memory(整个公司大的公共区域，比如食堂)。

对于设备端的代码(每一个员工)，可以读写线程私有的寄存器。它也可以读写线程块所共享的共享存储区 shared memory。它还可以读写 Grid 中所有线程共享的 global memory。

constant memory，是一段固定的存储器，它的作用是提供给多个线程使用的不变化数据，它是只读的，由于没有任何冲突和同步问题，所以访问它的速度较快。(公司的公示牌，只能读不能写)。

CPU主机端代码，可以读写整个 Global memory，及constant memory(公司高层可以发布公告牌)。

## ◼︎CUDA内存传输可以有几种方式

主机端 Host 可以和设备端 device 往返传输数据。主要是通过两部分，global memory 和 constant memory. Host 可以对两者都进行读写，device只能对 global memory 进行读写，对constant memory只能进行读操作。

![image.png](6%20CUDA%20%E7%BC%96%E7%A8%8B(%E4%B8%80)/image%207.png)

global memory 和 constant memory 都是在GPU设备上的。主机端可以通过底层 PCIe 总线进行对GPU 设备进行访存。操作可以通过几个具体函数，与 C语言中进行内存的分配和管理非常类似。

如果要在设备端分配 global memory, CUDA 系统提供了一个函数叫做 `cudaMalloc()`，当调用的时候，会在GPU设备端分配一个特定的global memory 的存储空间。对应的也有一个释放global memory空间函数叫做 `cudaFree()`。例如：

```cpp
float *Md;
int size = Width * Width * sizeof(float);

cudaMalloc((void**) &Md, size);
...
cudaFree(Md);
```

Md 指针，实际上是在GPU设备上的指针，用来返回设备端的内存地址。size 是一个大小，即希望在 global memory 上开辟一段空间的大小。我们调用cudaMalloc时，把指针和大小作为参数传进去。当这个函数调用成功以后，Md 指针就指向 GPU 设备上 global memory 中分配好的一段显存地址。然后使用完成以后，可以调用 cudaFree() 把这段显存地址释放掉。注意⚠️，Md 这个指针指向的是 GPU 设备上的存储空间， 是设备端的地址，只能在设备端使用，不能在主机端使用。

> 注意⚠️：Md 是 CPU 主机端的变量，但它指向的是设备端GPU显存的地址。所以，设备端分配内存的时候，使用 &Md，意思是指针变量本身的地址，往里面写入分配的显存地址，意思就是 Md 变量存放的内容是设备端显存的地址。所以 &Md 的类型是 (void **), 指向指针(Md)的指针。
> 

主机端和设备端之间的数据传输，可以使用 cudaMemcopy(), 它可以从主机端拷贝数据到主机端(host to host)，从主机端拷贝数据到设备端(host to device)，从设备端拷贝数据到主机端(device to host)，从设备端拷贝数据到设备端(device to device)。例如：

```cpp
cudaMemcpy(Md, M, size, cudaMemcpyHostToDevice);
cudaMemcpy(P, Pd, size, cudaMemcpyDeviceToHost);
```

第一条语句，第一个参数是目的地址，这是设备端地址，第二个参数是源地址，是主机端地址，第三个参数指明数据传输方向从主机端到设备端传输。

同样，第二条语句，第一个参数是目的地址，但是这里是主机端地址，第二个参数是源地址，这里是设备端地址，第三个参数指定数据传输方向从设备端到主机端传输。

例如，我们通过CPU系统接受到一个图像数据，读入CPU内存，就可以通过这个函数把图像拷贝到GPU上，通过GPU Kernel 函数的并发处理，处理完成后，还可以通过这个函数，把处理的结果从设备端拷贝会主机端。