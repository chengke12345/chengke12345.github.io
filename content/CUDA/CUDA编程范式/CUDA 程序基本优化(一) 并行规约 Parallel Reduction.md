# 9. CUDA程序基本优化(一): 并行规约 Parallel Reduction

<aside>
💡

**`有效的数据并行算法` + `针对GPU架构特性的优化` = `最优性能`**

</aside>

GPU CUDA 并行程序包含两部分内容，首先需要「有效并行数据算法」，然后是「针对 GPU 架构特性的优化」，两种方法融合起来，我们才会得到最优性能。

# Parallel Reduction 并行规约

并行规约最简单的表现形式就是求和。所谓「规约 Reduction」，是把一组很大的数据，通过某种综合性运算，得到很少一组数据。不断减少数据规模。
Google  Hadoop 的 MapReduce 模型，就是一个规约过程。

比如，把 0～7 这 8 个数加起来，获得一组和，这组和就是一个规约的过程。

![image.png](9%20CUDA%E7%A8%8B%E5%BA%8F%E5%9F%BA%E6%9C%AC%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E5%B9%B6%E8%A1%8C%E8%A7%84%E7%BA%A6%20Parallel%20Reduction/image.png)

并行规约下，把这些数字「两两求和」。第一级，每个线程求出两两的和，然后得到4个部分的和，第二级再两两求和，得到两个部分和，第三级再做最后一个求和，得到最终结果。这就是一个并行规约过程。这个过程类似锦标赛淘汰赛，有 n 个元素要进行 log(n) 个回合才能得到最终结果。

思路为，CUDA线程，每个线程来做一个求和，4 个线程做第一级的求和，再由两个线程做第二级求和，最后由一个线程做第三级，得到最终结果。一个节点上方，竖直箭头表示线程要的执行这一步的数据从线程的上一个状态来，斜线是这个线程要访问其他线程的数据。第一级，线程只需要访问它相近的数据，第二级，线程需要跳过一个访问数据，第三级，需要跳 3 个去访问数据。这就是基本的线程处理模式

CUDA 系统上的实现过程如下：假设在shared memory 里面做并行规约的过程：

```cpp
__shared__ float partialSum[];
// ... load into shared memory
unsigned int t = threadIdx.x; // ①
for(unsigned int stride = 1; stride < blockDim.x; stride *= 2){//②
	__syncthreads();//③
	if(t % (2 * stride) == 0)//④
		partialSum[t] += partialSum[t+stride];//⑤
}
```

这是一段 GPU 上的线程代码，每个线程都要执行这段相同的指令流。①代码中我们获得线程编号，假设总共有8个线程来做这个事情。第一级规约过程，0，2，4，6 这四个线程需要做加法操作。

shared memory 中用一个数组存放基本数据。每个元素值代表处理的数，下标位置，代表对应线程的编号。注意⚠️，block 内的线程共享 shared memory，__shared__ 定义的变量对所有 block 中的线程是同一个实体，都可以访问它。放在 shared memory 中处理是为了尽量提升访存性能。

然后②我们做了三次循环，就是做了三级的规约操作，两两规约，8个数据，2的三次访，需要做 3 次。循环一开始，启动线程步长是1,  第二次循环，步长是2，第三次循环，步长是 4。所以步长是有一个 1，2，4 的跳动。

代码④⑤：循环内容，是把每间隔一个步长编号的线程，让它与相隔步长那么远的线程中的数据做加法。真正做加法的线程，是 0，2，4，6。③ 处有一个同步，是为了保证所有线程把每一级的相加做完之后，再接着往下执行，否则就会产生数据错误。

在计算累加的过程中，0，2，4，6做计算，然后 0，4 线程计算，最后是 0 做计算。过程中没有做计算的线程，它们不做任何有价值的事情，但是由于在一个 block 的 warp 内，这8个线程必须要同步，所以它们也是在做相同的操作，但是由于循环和分支条件，它们并不会实际操作存储器。

![image.png](9%20CUDA%E7%A8%8B%E5%BA%8F%E5%9F%BA%E6%9C%AC%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E5%B9%B6%E8%A1%8C%E8%A7%84%E7%BA%A6%20Parallel%20Reduction/image%201.png)

第1轮规约操作

![image.png](9%20CUDA%E7%A8%8B%E5%BA%8F%E5%9F%BA%E6%9C%AC%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E5%B9%B6%E8%A1%8C%E8%A7%84%E7%BA%A6%20Parallel%20Reduction/image%202.png)

第2轮规约操作

![image.png](9%20CUDA%E7%A8%8B%E5%BA%8F%E5%9F%BA%E6%9C%AC%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E5%B9%B6%E8%A1%8C%E8%A7%84%E7%BA%A6%20Parallel%20Reduction/image%203.png)

第三轮规约操作。

![image.png](9%20CUDA%E7%A8%8B%E5%BA%8F%E5%9F%BA%E6%9C%AC%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E5%B9%B6%E8%A1%8C%E8%A7%84%E7%BA%A6%20Parallel%20Reduction/image%204.png)

得到结果。每轮规约所需的线程数减半

## 优化思路

我们在安排线程的时候，排列组合，让线程处理数据的顺序发生一些变化。

![image.png](9%20CUDA%E7%A8%8B%E5%BA%8F%E5%9F%BA%E6%9C%AC%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E5%B9%B6%E8%A1%8C%E8%A7%84%E7%BA%A6%20Parallel%20Reduction/image%205.png)

我们让0，1，2，3 四个线程做计算，然后让 0，1 线程做第二轮，然后让 0 线程做第三轮。这样我们得到的是相同的并行规约的结果，但是规约对象的位置发生了变化。步长由原来的1，2，4 变为了4，2，1.

```cpp
__shared__ float partialSum[];
// ... load into shared memory
unsigned int t = threadIdx.x; // ①
for(unsigned int stride = blockDim.x / 2; stride > 0; stride /= 2){//②
	__syncthreads();//③
	if(t < stride)//④
		partialism[t] += partitalSum[t+stride];//⑤
}
```

![image.png](9%20CUDA%E7%A8%8B%E5%BA%8F%E5%9F%BA%E6%9C%AC%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E5%B9%B6%E8%A1%8C%E8%A7%84%E7%BA%A6%20Parallel%20Reduction/image%206.png)

![image.png](9%20CUDA%E7%A8%8B%E5%BA%8F%E5%9F%BA%E6%9C%AC%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E5%B9%B6%E8%A1%8C%E8%A7%84%E7%BA%A6%20Parallel%20Reduction/image%207.png)

![image.png](9%20CUDA%E7%A8%8B%E5%BA%8F%E5%9F%BA%E6%9C%AC%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E5%B9%B6%E8%A1%8C%E8%A7%84%E7%BA%A6%20Parallel%20Reduction/image%208.png)

![image.png](9%20CUDA%E7%A8%8B%E5%BA%8F%E5%9F%BA%E6%9C%AC%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E5%B9%B6%E8%A1%8C%E8%A7%84%E7%BA%A6%20Parallel%20Reduction/image%209.png)

整个工作过程，每一轮，我们需要的线程数依然是减半的，但是线程所处的位置不同。前面的方式，线程是交叉减半的，这种方式是前一半后一半的减半。

这样的优势是，可以把提前完成的线程占用的资源提前释放出来。第一种方式，由于它是交叉的，由于有warp的约束，就没法把中间那些不用的线程的资源释放出来。