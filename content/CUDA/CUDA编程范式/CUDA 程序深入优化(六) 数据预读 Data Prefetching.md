# 14. CUDA 程序深入优化(六):数据预读 Data Prefetching

在一次 global memory 读操作和实际用到这个数据的语句中间，插入独立于以上数据的指令，可以隐藏访存延迟。数据预取本来也是我们用来隐藏访存延迟的一个技术手段。

```cpp
float m = Md[i];
float f = a * b + c * d;
float f2 = m * f;
```

我们把数据从存储器读取到 m 上，在第三条语句处使用 m，实际上这两条语句之间需要等待的延迟比较大。系统会切换 warp 来试图掩藏延时。比如第二条语句计算 f, 它不依赖于第一条语句，编译器就可以把第二条语句的执行插入到第一条语句的访存延迟中去。（这里假设 a, b, c, d 都可以直接从寄存器中获取到）

从 global memory 中预读数据可以有效提升独立指令的数量，这个方法其实是我们在帮助编译器达到延迟隐藏的效果。

# 1. 访存读取数据的过程

我们发起一个读取数据到 shared memory 的请求的时，数据是从显存里面读出，首先是进入 L2 Cache，然后到 L1 Cache，由于数据是首先读到 Cache, 所以单次访存，读取数据的单位才是 Cache Line。接着，从L1 Cache 才会进入寄存器，最后从寄存器再送入 shared memory. 这里如果要进行计算，就从寄存器把数据送入 CUDA Core 进行计算。

```
DRAM → L2 Cache → L1 Cache → 寄存器 → Shared Memory
└──── Global 读取阶段 ────┘    └─ Shared 写入阶段
```

一般情况下，从 DRAM 读取的数据，都要经过 L2 Cache, L1 Cache, 才到达寄存器。一般情况下，读取的数据，会自动成为 L2 Cache,  L1 Cache 的有效缓存数据。但是这不是必须的，少数情况下，数据经过 L2 Cache, L1 Cache的硬件模块，但是不进入Cache, 成为有效缓存，而直接进入寄存器。但是，寄存器只和 L1 硬件模块直接交互，这条数据链路是一定的。

# 2. 矩阵相乘的数据预取

瓦片化的矩阵相乘 matrix multiply

```cpp
for(/*...*/){
	//load current tile into shared memory
	__syncthreads();
	//Accumulate dot product
	__syncthreads();
}
```

我们是把 M 矩阵一个瓦片的数据读入 shared memory 中，N 矩阵的一个瓦片读入 shared memory中，然后在 shared memory 中完成乘累加，这样省去了重复读取数据的开销。

引入数据预取之后，瓦片化的 matrix multiply

```cpp
// load first tile into registers ①
for(/*....*/){
	// Deposit registers into shared memory ②
	__syncthreads();
	// Load next tile into registers ③
	// Accumulate dot product. ④
	__synthreads();
}
```

数据预取，我们首先把第一个瓦片数据直接读到寄存器中(准确说是两个瓦片数据，M 和 N的 Tile，这里说一个是指结果瓦片 )，而不是读到 shared memory 中。然后②再把寄存器里面的数据放到shared memory里面去。此时 shared memory 中就是准备好计算 M tile 和 N tile。此时③，不立刻进行乘累加计算，而是先预取下一个M tile 和 N tile 的数据到寄存器中。

每个瓦片之间的数据是完全独立的，所以彼此不会影响。但是在访存读取 M tile 和 N tile 数据的时候，此时延迟等待的时间，就可以用来计算 shared memory 中存放的 M tile 和 N tile 之间的乘累加。读取数据，计算乘累加，两个操作互不影响，如果有足够多的 warp 可以切换的话，计算乘累加的操作把读取数据的操作延迟隐藏了。shared memory 成为一个预取的空间。