# 8. CUDA编程(三):矩阵相乘GEMM

# 0. Phase 0: CPU 上实现矩阵乘法

矩阵相乘 Matrix Multiply

![image.png](8%20CUDA%E7%BC%96%E7%A8%8B(%E4%B8%89)%20%E7%9F%A9%E9%98%B5%E7%9B%B8%E4%B9%98GEMM/image.png)

M 和 N 是两个原始矩阵，P 是一个结果矩阵。假定 M 和 N 都是方阵。M 的一行，乘累加上 N 的一列，得到 P 里面的一个结果元素。

矩阵相乘有很好的并行性，P 结果矩阵的每一个元素 [x, y]，都是由 M 中的第 x 行和 N 中的第 y 列进行乘累加得到的。彼此之间的计算过程都是独立的，P 中每个元素的计算过程和其他元素的计算过程都不相干。每一个结果的输出，都可以有一个特定线程去计算。

两个1000 维方阵相乘，需要计算 1000 * 1000 = 1,000,000 次点乘(dot product)。点乘就是一行和一列对应元素相乘再累脚，所以每次点乘需要做 1000 次乘法 和 1000次加法。如果用 CPU 实现

![image.png](8%20CUDA%E7%BC%96%E7%A8%8B(%E4%B8%89)%20%E7%9F%A9%E9%98%B5%E7%9B%B8%E4%B9%98GEMM/image%201.png)

# 1. Phase 1: GPU 上的朴素实现

在 GPU 上实现矩阵乘法，我们有一个基本框架。这个框架对于大多数 GPU 程序都是成立的。

> 
> 
> - 第一步，在GPU上分配空间 M， N， P，用来存放原始数据以及输出的结果。把CPU端读入的数据，拷贝到GPU上。
> - 第二步，在GPU上启动 kernel 并行处理数据的计算。
> - 第三步，把计算的结果拷贝回CPU。

```cpp
int main(){...}
/* STEP 1: Allocate and initialize the matrices M, N, P in GPU memory
 *       I/O to read the input matrices M and N 
 */

/* STEP 2: compute M * N on the GPU Devece */
MatrixMulOnDevice(M, N, P, Width)

/* STEP 3: I/O to write the ouput matrix P back to CPU 
 *         Free matrix M, N, P 
 */
```

1. 处理数据传输框架

![image.png](8%20CUDA%E7%BC%96%E7%A8%8B(%E4%B8%89)%20%E7%9F%A9%E9%98%B5%E7%9B%B8%E4%B9%98GEMM/image%202.png)

注意⚠️：这段代码是在主机端运行的，即在 CPU 端运行。

1. CUDA C 实现 kernel
    
    结果矩阵中的每个元素都用一个线程计算，即 M 中的一行与 N 中的一列点乘。结果矩阵是二维矩阵，使用二维索引的线程块 block 最方便。我们严格限制一个线程计算结果矩阵中的一个元素，所以某线程索引 `threadIdx.x` 与 `threadIdx.y` 就对应结果矩阵中元素的坐标。
    
    ![image.png](8%20CUDA%E7%BC%96%E7%A8%8B(%E4%B8%89)%20%E7%9F%A9%E9%98%B5%E7%9B%B8%E4%B9%98GEMM/image%203.png)
    
    注意⚠️，CPU版本中的外面两层循环都不见了。因为外面的双重循环已经通过并行的方式处理掉了。核函数的代码是一个所有线程都共享指令流，所有线程都要执行这一段程序，这个机制就天然把循环去掉了。
    
    当所有线程都执行完之后，也就是 Pd 结果矩阵里面已经填满了各个线程计算的结果之后, Pd 就是我们最终求的结果矩阵。
    
    由于每一个结果矩阵里面的元素都是彼此独立的，它不会用到或依赖别的元素的结果，所以这里不需要使用到并行计算中的锁存或者同步的机制。
    
2. 在主机端调用 kernel 
    
    当主机端调用 kernel 的时候，系统就会自动在 GPU 上启动 kernel 的大量并行线程。调用的时候，我们必须要定义每个block里有多少线程，以及每个 Grid 里有多少block。
    
    ![image.png](8%20CUDA%E7%BC%96%E7%A8%8B(%E4%B8%89)%20%E7%9F%A9%E9%98%B5%E7%9B%B8%E4%B9%98GEMM/image%204.png)
    
    我们设定block中的线程数是 Width * Width，宽度 * 宽度(前面假设M 和 N 都是方阵)，每个线程对应处理一个结果元素。同时，我们只有一个block，但是这里是二维的，我们假设Grid也是二维的。
    

所以矩阵相乘并行处理的基本思路

![image.png](8%20CUDA%E7%BC%96%E7%A8%8B(%E4%B8%89)%20%E7%9F%A9%E9%98%B5%E7%9B%B8%E4%B9%98GEMM/image%205.png)

这里面出现了一个问题，我们的计算次数和片外访存次数的比率是接近 1 : 1 的。即计算一次乘法并累加需要访存2次，读取 Md 中一行的一个元素，再读取对应 Nd 中对应列上的一个元素。计算 2 次，乘法一次，加法一次。所以计算次数和访存次数的比率接近1 : 1。(注意⚠️，这里描述的访存不是按整行整列的读，而是一个数一个数的读)

访存的延迟非常巨大，所以整个系统的访存压力很重。

另外，由于是一个block处理一个结果矩阵，所以矩阵大小受限于一个线程块允许的线程数目，而一个 block 中的线程数是有约束的。

所以，我们要解决两个问题：

- 访存延迟巨大。要想办法减少访存次数。
- block 大小约束了矩阵大小。想办法突破 block大小，否则无法执行大矩阵乘法。

所以总结起来就两个问题：「大矩阵」和「少访存」。

# 2. phase 2: 优化实现—突破矩阵维度限制

问题定义：

- 按照前面的方法，以 G80 和 GT200 为例，一个 block 最多有 512 个线程。所以不能计算矩阵大小大于 512 的矩阵。实际需求就是要 GPU 计算非常巨大的矩阵，大矩阵乘法是 GPU 并行计算最主要的用途。处理图形图像时，需要处理 「像素 x 像素」 的矩阵，通常都是很大的矩阵。
- 相乘里面有很多读写访问，乘累加的时候，两次读显存的操作之后，才进行了一次乘累加。然后计算完成后还有一次写内存。这样一个读写memory比例，读写的开销是非常巨大的。

所以，接下来优化的时候，解决两个问题，「大矩阵」和「少访存」。

### **突破 block 尺寸对矩阵维度的限制**

我们可以把结果矩阵拆分一下。之前是一个 block 负责整个大的完整的结果矩阵，现在我们把结果矩阵拆分乘若干小块。一个小块，称为一个 **「tile，瓦片」**。每个 block 负责其中的一个小块。

(搬砖模型，整个公司去搬砖的时候，不再是派一个部门去搬，而是多个部门的人都去，每个部门负责一个砖堆，就是一个 tile。 自然，每个人就要去找到自己对应负责第几个砖堆里面的那几块。) 

通过 threadIdx 和 blockIdx 索引可以访问结果矩阵元素数据对应的坐标。先用一个简单的例子，假设结果矩阵是一个有 16 个元素，4 * 4 的方阵，我们把它分成 4 个 Tile, 每个 Tile 大小为 2 * 2。

![image.png](8%20CUDA%E7%BC%96%E7%A8%8B(%E4%B8%89)%20%E7%9F%A9%E9%98%B5%E7%9B%B8%E4%B9%98GEMM/image%206.png)

这四块 Tile 对应 4 个 block 负责计算，每个 block 大小为 2 * 2, 即用一个 block， 2 * 2 个线程对应计算 Tile 中的 2 * 2 个结果元素。Block(0,0)~Block(1,1), 由于计算结果是相互独立的，所以每个block 可以独立的做自己的事情。例如，第 1 个 block 要做的事情：

![image.png](8%20CUDA%E7%BC%96%E7%A8%8B(%E4%B8%89)%20%E7%9F%A9%E9%98%B5%E7%9B%B8%E4%B9%98GEMM/image%207.png)

图中着色的部分，就是第 1 个 block 要完成的计算。这样划分 Tile，让 Block 对应计算 Tile 中元素的方式，而不再是处理整个矩阵，就可以突破 block 对矩阵大小的限制。因为 block 的数目可以非常多，虽然 block 大小有限制，但是 block 的数量没有限制。

按照这种思路修改之后的代码为：

```cpp
__global__ void MatrixMulKernel(float *Md, float *Nd, float *Pd, int Width){
	int Row = blockIdx.y * blockDim.y + threadIdx.y;
	int Col = blockIdx.x * blockDim.x + threadIdx.x;
	
	float Pvalue = 0;
	for(int k = 0; k < Width; k++)
		Pvalue += Md[Row * Width + k] * Nd[k * Width + Col];
	pd[Row * width + Col] = Pvalue;
}

int main(){
	dim3 dimGrid(Width / TILE_WIDTH, Height / TILE_WIDTH);
	dim3 dimBlock(TILE_WIDTH, TILE_WIDTH);
	
	MatrixMulKernel<<<dimGrid, dimBlock>>>(Md, Nd, Pd, WIDTH);
}
```

main 函数在 CPU 端运行，它会调用 MatrixMulKernel 核函数，即启动 kernel。kernel 本身在 GPU 端执行。启动 kernel 时需要给出执行配置，定义 Grid 中有多少 block, block 中有多少 thread. 

dim3 是内置结构体类型，它是一个索引类型，存放维度信息，CUDA 最多支持三维，所以叫 dim3. 没有指定的维度，默认为1. 

dimBlock 是指定处理一块 Tile 的 block 大小，TILE_WIDTH * TILE_WIDTH 建立的是 block 的二维索引。整个矩阵的宽度除以一个小块的宽度 Width / TILE_WIDTH ，就得到在宽度方向上，需要多少个小块，同样可以得到在高度方向上需要多少小块。

因此，我们就得到我们需要 dimGrid 这么多个block，每个 block 的大小是 dimBlock。

<aside>
💡

**注意⚠️**

核函数是一个线程要处理的数据，即结果矩阵中一个元素的计算。它并不知道如何划分block，以及有多少block。它只是单个线程的执行指令流。

核函数的Width，是整个 Md, Nd, Pd 的宽度，不是Tile 的宽度。要特别注意。

</aside>

<aside>
💡

**矩阵尺寸问题**

如果结果矩阵是 1024 * 768，那可以划分为 32 * 32 或者 16 * 16 的小tile。但是如果矩阵尺寸是1025 * 768，那么处理方法是**把不能整除的那一部分填0**. 这样不会影响结果的正确性，但是会影响计算效率，最好在设计时就考虑到整除的问题。

</aside>

# **3: Phase3: 优化实现—减少访存**

每一次乘累加，我们需要读两次数据，一次从 Md 读一个数，一次从 Nd 读一个数。结果矩阵中一个元素的计算，需要 Md 中一行，对应 Nd 中一列，对应元素之间执行乘累加计算(每次乘累加两次访存)。因此，我们面临访存受限问题，我们受限于 global memory 的带宽。

> G80为例，峰值算力 346.5 GFLOPS, 即每秒钟可以做 346.5G次浮点数运算，每次浮点数运算需要给它提供一个浮点数，每个浮点数 4 Byte，就需要 1386 GB/s 带宽才能给他提供足够的浮点数，以达到峰值算力。实际 G80 的显存带宽只有 86.4 GB/s, 这就限制了代码的峰值算力只有 86.4 / 4 = 21.6 GFLOPS，由于一些开销，代码实际运行速度是 15 GFLOPS，峰值算力的 1/20 都不到。
> 

**必须大幅减少对 global memory 访问**。

### 重复访存问题

事实上，每个元素都是被若干线程反复读取的。

![image.png](8%20CUDA%E7%BC%96%E7%A8%8B(%E4%B8%89)%20%E7%9F%A9%E9%98%B5%E7%9B%B8%E4%B9%98GEMM/image%208.png)

例如上图，Md 中两行和 Nd 中同一列进行乘累加，计算结果矩阵中的两个元素。Nd 列上的元素就会被两个线程读取两次。每个元素都会被 width 个线程读取，因为每个元素都会参与结果矩阵中，同一行或者同一列的元素的计算。我们要重复读取同一个数据 Width 次。显存带宽产生巨大浪费。

### Shared Memory

我们的解决方法是使用 GPU 里面 SM中的 shared memory，批量计算。

![image.png](8%20CUDA%E7%BC%96%E7%A8%8B(%E4%B8%89)%20%E7%9F%A9%E9%98%B5%E7%9B%B8%E4%B9%98GEMM/image%209.png)

我们把 kernel 拆分成多个阶段，每个 block 依然负责其中一个子块。

事实上，我们需要把图中 Md 上的一横条完整数据和 Nd 中一竖条完整数据乘累加起来，才能得到图中结果矩阵中橘黄色块的最终结果。

**我们可以把整个一横条和一竖条的数据加载到 block 的 shared memory 中去**。然后在share memory中做乘累加。但是这个条带也可能很大，条带可能已经超过了shared memory的大小，所以我们把条带再切分成一个个更小的小块，每一个小块可以放到shared memory 中去。

上图中计算出 Pd 橘色块最终结果的过程是，首先把 Md 中紫色块和 Nd 中的紫色块作乘累加，得到是一个中间结果放在 Pd 对应的橘色块中。然后计算第二个 Md 和 Nd 中的红色块，直接再累加到 Pd 最终的橘色块，这样就得到紫色块和红色块两部分计算的中间结果在 Pd 橘色块中。然后，以此类推，将一行一列上划分的 block 分别全部计算完乘累加之后，Pd 中的橘色块，就是最终结果。

**总结：对应颜色的块先进行乘累加，得到的是一个中间结果，然后把这些不同颜色的块加到一起，最后才能得到一个完整的结果块**。

![image.png](8%20CUDA%E7%BC%96%E7%A8%8B(%E4%B8%89)%20%E7%9F%A9%E9%98%B5%E7%9B%B8%E4%B9%98GEMM/image%2010.png)

每个线程读入一个tile内的 Md 和 Nd 上的一个元素，存入 shared memory.

**代码分析**

```cpp
__global__ void MatrixMulKernel(float *Md, float *Nd, float *Pd, int Width){
	__shared__ float Mds[TILE_WIDTH][TILE_WIDTH];
	__shared__ float Nds[TILE_WIDTH][TILE_WIDTH];
	
	int bx = blockIdx.x; int by = blockIdx.y; // ①
	int tx = threadIdx.x; int ty = threadIdx.y; // ②
	
	int Row = by * TILE_WIDTH + ty; // ③
	int Col = bx * TILE_WIDTH + tx; // ④
	
	float Pvalue = 0;
	for(int m = 0; m < Width / TILE_WIDTH;++m){//⑤
		Mds[ty][tx] = Md[Row * Width + (m * TILE_WIDTH + tx)]; //⑥
		Nds[ty][tx] = Nd[Col + (m * TILE_WIDTH + ty) * Width]; //⑦
		__syncthreads();
		
		for(int k = 0; k < TILE_WIDTH; ++k) // ⑧
			Pvalue += Mds[ty][k] * Nds[k][tx]; // ⑨
		__syncthreads();
	}
	Pd[Row * Width + Col] = Pvalue；// ⑩
}
```

> 在 shared memory 中创建两个临时空间, Mds 用来存放 Md 上一个 Tile 的数据, Nds 用来存放 Nd 上一个Tile 的数据。
**核函数中的代码，是所有线程都执行的一段共同指令流。shared memory 是按 Block 为单位进行划分的**。 我们定义了Mds, Nds 两块 shared memory，实际上是在划分给该 block 的 shared memory 区域上定义的两个临时矩阵。
shared memory 划分给一个 block 的分区，在这个 block 内部的线程中，只会定义一次，所有线程执行同一段定义时，其实是声明，所以block 线程内看到的是同一个 shared memory。 这就是这个存储被叫做共享存储的原因。
> 

<aside>
💡

**二维索引中的 x, y 轴 和二维数组行列下标不是一回事** 

我们在 Grid 中索引 block, 或者 block 中索引 thread，所使用的二维索引的 x, y 轴和二维数组的行列下标，是相反的。

![image.png](8%20CUDA%E7%BC%96%E7%A8%8B(%E4%B8%89)%20%E7%9F%A9%E9%98%B5%E7%9B%B8%E4%B9%98GEMM/image%2011.png)

比如这个点如果代表一个元素，它在二维数组中的索引是是 2 行 3列，即arr[2][3]。但是如果这个点代表 block 中某线程，block 内的线程是二维索引的。那么横轴是 x 轴，纵轴是y轴，所以要索引到这个线程 threadIdx.x = 3, threadIdx.y = 2. x 轴对应的是列，而y轴对应的是行，这和我们的直觉是相反的，这点要特别注意。 

CUDA 代码一般会写成

```cpp
int row = threadIdx.y; // y 对应行
int col = threadIdx.x; // x 对应列

// 当前线程对应处理 Matrix[row][col] 元素
```

</aside>

代码①②: 代码中的 bx 和 by 是 block 的 x 编号 和 y 编号。tx, ty 是 thread 在 block 内部的 x 编号 和 y 编号。

代码③④: Row 和 Col 是线程全局行编号和全局列编号。Row 对应的是 y 轴，Col 对应的是 x 轴

- Row 的计算是用 block 的 y 轴编号，即沿 y 轴向下走了多少个 block 再乘以 block 宽度，再加上这个线程在一个block内部的y坐标，就得到了线程在全局二维索引中的 y 坐标。这对应的是行编号 Row。
- Col 是用block的 x 编号，即沿 x 轴向右走了多少个 block，乘以 block 的宽度，再加上这个线程在一个block内部的 x 坐标，就得到了线程在全局二维索引中的 x 坐标，对应的是列编号 col。这样就得到这个线程在全局对应的行列编号 Row 和 Col。

Row 和 Col 是最终结果矩阵中元素对应的行列编号，或者说结果矩阵的二维数组行列编号。代表这个线程的任务就是生成结果矩阵中对应 Row 行 Col 列的这个结果元素。

但是特别注意 Row 是 y 轴，Col 是 x 轴。所以结果矩阵的 Pd[Col][Row] ←→ block[x][y]. 即 Pd 矩阵里面的行列索引和 block 中的二维索引，是反的。

一般会写成 `i = threadIdx.y` , `j = threadIdx.x`, 然后 `block[i][j] ←处理→ Matrix[i][j]`。

代码⑤：接下来的循环是为了使用 shared memory 做累加，而不是每次重复访存获取数据。

![image.png](8%20CUDA%E7%BC%96%E7%A8%8B(%E4%B8%89)%20%E7%9F%A9%E9%98%B5%E7%9B%B8%E4%B9%98GEMM/image%2012.png)

理论上，我们应该把 M 中那一整个横条数据和 N 中那一整个竖条数据整个放入shared memory中，直接做累加，就可以得到结果矩阵中一块 tile 的结果数据。但是 shared memory 的容量有限，整个横条数据和整个竖条数据已经超过了shared memory的大小。所以，我们的做法是将横条数据按tile的大小切割为tile大小的块，竖条也切割为tile大小的块。

如上图，我们首先对紫色的横条块和竖条块的数据放入shared memory，在shared memory 中做累加，然后放入结果矩阵的tile中去，此时结果矩阵中的这个 Tile 块是一个中间结果。

然后计算计算第二组分块，即把橙色的横条块和竖条块的数据放入shared memory，在shared memory 中做累加，再和结果矩阵的 tile 相加，这样就得到处理完紫色和橙色分块后的中间结果。然后以此类推…..对后面的每组数据块都做这样的操作。

最后横条的 Tile 和竖条的 Tile 都累加完之后，就得到 Tile的最终结果了。这样既解决shared memory 不足的问题，又解决了对一个数据只做一次访存的问题。

实际操作中，我们使用一个 block 的线程来计算一个结果 Tile 块的数据，每个线程计算一个 Tile 块中的一个元素。记住，**核函数的代码是每个线程执行的指令流**。我们实际使用shared memory 的方式，把横条的紫色块读入Mds, 把竖条的紫色块读入 Nds，然后对 Mds 和 Nds, 即，M 和 N 中的紫色横竖条的 Tile 块内乘累加, 直接求得一个 Tile 块乘累加产生的中间结果。

代码⑤的外层循环，就是一个线程要负责计算出结果矩阵中的一个最终结果，它其实是负责 M 一整行和 N 一整列的乘累加，最后得到结果矩阵中的一个元素值。

但是这段代码要以 block 为单位运行，降低访存频率。 M 中一横条上所有的 Tile ，和 N 中一竖条上所有 Tile， 由一个 block 线程 Tile 的乘累加计算。线程负责计算出一个最终结果。所以**这个循环是对 Tile 块进行循环，表示逐个处理完一整行和对应的一整列的 Tile**。代码中，⑤处最外层循环。每处理完一个 Tile，就累加到中间结果 Pvalue 上去，最后横条和竖条处理完，Pvalue 中就是最终累加结果，最后将 Pvalue 放入 Pd 矩阵中相应位置。这个线程就计算完自己的最终结果。此时，整个block中所有线程执行完，就得到结果矩阵中的一个Tile的最终值。

![image.png](8%20CUDA%E7%BC%96%E7%A8%8B(%E4%B8%89)%20%E7%9F%A9%E9%98%B5%E7%9B%B8%E4%B9%98GEMM/image%2013.png)

> 简单说：一个线程负责计算一个最终结果值，但它是分 Tile 逐步计算出来的, 因为整个 block 共享数据，降低访存。block 中的线程每次循环一个 Tile 计算出来，并累加一个中间结果，处理完所有的Tile 后，得到最终结果。一个block 一起产生一个 Tile 的最终结果。但是 block 中一个线程还是负责一个结果，不过最终结果因为分Tile，它也是分阶段逐步产生并叠加中间结果，得到最终结果。
> 

代码⑥⑦：在⑤循环体内部，每一轮循环，是处理一个 Tile 的计算。

每个线程，都负责结果矩阵里面 Tile 中一个元素的计算，这个元素有一个行列位置，假如是 [i, j], 这个线程把 M[i, j] 元素读入Mds, 把 N 中 N[j, i] 的元素读入Nds。然后同步等待其他线程也将相应的元素放在shared memory中，使用 `__syncthreads()`, 即 block中的所有线程把 M tile 内的数据和N Tile 中的数据，都放入 shared memory 里的 Mds 和 Nds 中之后，再进行下面操作。

**注意⚠️，这两个shared memory的tile中的数据，是由每一个线程放一个M矩阵中tile的一个元素到Mds, 以及放一个N矩阵中tile的一个元素到Nds, 两次读取。一个线程读取一对 Md 和 Nd 数据**。该block中的线程同步执行到这一步之后，Mds 和 Nds 中的数据就准备好了。此时让 Mds 和 Nds的数据做乘累加，就不需要额外访存了。

需要注意的是，虽然一个线程在一次循环里，只把一对 M 和 N 的对应数据放到 shared memory, 但是整个 block 的线程执行完之后，单个线程计算这轮 Tile 中自己需要的 M 矩阵中的一行数据和 N 矩阵中的一列数据，都已经被 block 中其他线程读到了 shared memory 中的 Mds 和 Nds，该轮 Tile 需要的数据已经准备好。

> 循环体内部，一个线程准备 Mds 和 Nds 的数据，是为整个block的所有线程准备的。而接下来的计算，是为自己负责的那一个结果，进行的计算，也就是说下面的 Pvalue 只是为自己负责的最终结果，保存的中间结果。
> 

代码⑧⑨：接下来，该线程处理的只是该线程负责的M的tile中的一行(Mds中对应的一行)，与N的tile中的一列(Nds中的一列)，做累乘加，放入临时结果Pvalue中，Pvalue是线程私有的局部变量。**注意⚠️，这里只是一个临时结果，因为这里只有一个 tile 的临时处理结果**。

然后这里再用一个`syncthreads()`, 然后block里所有的线程都计算完这个tile中的数据，结果放在各线程自私有的Pvalue中以后。等 block 中所有线程的 Pvalue 计算完成后，就可以放入 Pd 结果矩阵的相应位置了。

计算完 Pvalue 后，使用 `syncthreads()`等待其他线程同步，是因为 Pvalue 的乘累加计算，要依赖其他线程往 shared memory 中放入的数据。如果不同步，不等所有线程都完成 Pvalue 的乘累加计算，那么有些执行速度快的线程就会在下一个循环往 shared memory 中放入新的数据，此时，旧的数据还没完成计算，就会造成冲突。同步就是解决这个问题。

然后就可以进入下一个循环，计算下一个 tile 了。

代码⑩：当线程把所有 tile 都做完，把所有乘累加的值都累加到临时变量 Pvalue 上之后。block中每个线程都各自有有一个最终的 Pvalue值. 这个 Pvalue 就是该线程负责计算的最终结果值。此时，再把最终值 Pvalue 写入结果矩阵 tile 中的相应位置。

<aside>
💡

注意⚠️

上面的讨论，是针对一个block的线程计算出最终一个 Tile 的过程。不是完整的结果矩阵P。完整的结果矩阵 P，就需要多个block一起并发/ 并行执行完成。

</aside>

### **title 大小的选择**

这里有个重要问题，就是 Tile 大小 TITLE_WIDTH 如何选择的问题。如果选太大了，可能一个 Tile 大小就会超过一个block的最大线程数。(G80 和 GT200 是 512, Fermi 是1024， Kerpler 是1024)。

另外，Tile 太大可能会超出 shared memory的极限。因为shared memroy 是要分配给block，一个block就要占用一块 shared memory 的大小。如果title太大，block的数量就会减少，(G80 的 shared memory 16KB/SM， 每个SM上8个blocks, 即 2KB/block, 1KB 给 Nds, 1KB 给Mds, 16 * 16 * 4[一个浮点数4byte] = 1KB，那就有 TILE_WIDTH = 16，即 16 * 16的大小，如果TILE_WIDTH 变大，那么在一个SM上使用的块数就会减少)。block数减少了，就会导致SM的占用率下降。

使用 shared memory 可以有效降低对 global memory 的访问次数。访问次数会减少TILE_WIDTH倍。16 * 16 tile，减少16倍。这样处理以后，系统吞吐就已经接近峰值算力了。

#### 原子操作 Atomic Functions

在GPU中，我们提供了很多 原子操作，Atomic Functions。原子操作就是针对特定的一个操作数，原子操作就是只有一个线程可以对个操作数进行相应原子操作。只有当这个操作完成之后，第二个线程才可以对它再执行操作。原子操作，是对同步和对全局数据共享的一种方法。常见的如下：

![image.png](8%20CUDA%E7%BC%96%E7%A8%8B(%E4%B8%89)%20%E7%9F%A9%E9%98%B5%E7%9B%B8%E4%B9%98GEMM/image%2014.png)

随着架构的提升，原子操作的速度也是越来越快。一般情况下，原子操作我们尽量少用，因为原子操作本身非常耗时，需要在系统里建立完整的队列。