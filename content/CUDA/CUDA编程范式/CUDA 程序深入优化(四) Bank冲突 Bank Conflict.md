# 12. CUDA程序深入优化(四): Bank冲突 Bank Conflict

# 1. Shared Memory 和 Bank 冲突

有时线程访问数据的地址，会跳着一个步长 stride 去访问，这会导致很明显的性能下降。这种访问模式性能是不优化的，非常不推荐。**但是这种情况又会经常发生，在多维问题里面，通常需要按步长访问内存，从很多维数据里面跳着取数据，这时候会不可避免的发生步长的问题**。

我们可以使用 shared memory 来解决，shared memory 很大程度上就是用来处理这样的问题的。我们可以通过对shared memory编程，有效规则化访存模式。

正常情况下，我们把显存上的数据，读到寄存器上，并且放到缓存 Cache 里，所以读取的数据块才叫 Cache Line。如果对 global memory的访问是不规则的，这会导致多次访存请求，效率降低。我们可以先用规则化的读取方式，把显存上的数据读到 shared memory 中，再从 shared memory 中进行非规则化访问，shared memory 是片内存储器，它的速度接近寄存器。这样就绕开了对DDRAM直接进行非规则化访问。 

shared memory 比 global memory 快上百倍。通过 shared memory 缓存数据可以减少 global memory 的访存次数。线程还可以和 shared memory 协作，用来避免不满足合并条件的访存，比如可以读入shared memory之后重排顺序，从而支持合并寻址。即，把 global memory 中零散的访问，可以按连续地址统一合并读到shared memory中，然后重新排序，完成合并访存模式。

shared memory 是被分为可以被多个线程访问的区块的，**这样的「一个区块叫做Bank」**。整个 shared memory 实际上是被分为若干 bank, 连续的 32-bit (4字节地址), 访存是被分配到连续的 banks 上的。例如下面

![image.png](12%20CUDA%E7%A8%8B%E5%BA%8F%E6%B7%B1%E5%85%A5%E4%BC%98%E5%8C%96(%E5%9B%9B)%20Bank%E5%86%B2%E7%AA%81%20Bank%20Conflict/image.png)

![image.png](12%20CUDA%E7%A8%8B%E5%BA%8F%E6%B7%B1%E5%85%A5%E4%BC%98%E5%8C%96(%E5%9B%9B)%20Bank%E5%86%B2%E7%AA%81%20Bank%20Conflict/image%201.png)

左图，整个 shared memory 被分为 15个 bank。连续的 32-bit 访存分配到连续的 bank 上面。

地址0就在 bank 0 上，地址1 就在 bank 1 上，地址2就在 bank 2 上，以此类推。到了地址16，又返回到 bank 0。所以，地址0，地址16，地址32…..都会落在 bank 0 里面， 地址1， 地址17， 地址33….都会落在bank 1 里面。

每个 bank, 对外有一个特定的接口，每个周期，bank 只能响应落在这个 bank 里面的一个地址的访问。如果多个线程访问的是不同 bank, 这些 bank 可以同时响应多个地址的访问申请。对同一个 bank 进行多个访存请求，将导致 bank 冲突，bank 冲突的访存必须串行执行。

右图是两个寻址举例，首先线性寻址，线程 0 访问 bank 0 中的地址，线程1访问 bank 1 中的地址，线程2 访问 bank 2 中的地址，以此类推。这样的访问就没有冲突
如果是1:1随机置换访问的话，如果每个线程的访问还是落在不同bank里面，这样也是没有冲突的。

<aside>
💡

**注意⚠️**

bank的编号，不是内存的编号。内存编号对bank数取余为0的在 bank 0 里面，取余为1的在 bank 1 里面，取余为2的在 bank 2 里面…..也就是内存编址，是以 bank 编号顺序编排，并且根据 bank 数循环使用 bank。 

</aside>

但是也有一些情况会造成冲突，我们称为几路冲突。

![image.png](12%20CUDA%E7%A8%8B%E5%BA%8F%E6%B7%B1%E5%85%A5%E4%BC%98%E5%8C%96(%E5%9B%9B)%20Bank%E5%86%B2%E7%AA%81%20Bank%20Conflict/image%202.png)

上图左边，每一个 bank 时会有两个线程要访问它里面的地址，这种就是 2-路 bank 冲突。右边就是 8-路 bank 冲突。一般来说，**「几路bank冲突，就会导致系统性能下降多少倍」**。

如果没有 bank 冲突，shared memory 的速度会和寄存器一样快。`warp_serialize profiler` 分析器可以反应冲突情况。

warp 内所有线程访问不同的 banks，就没有冲突，每个线程访问自己的bank的数据就可以了。如果所有的线程**「读取」**同一个地址，这样也没有冲突，因为硬件是支持广播的，从这个地址读取一份数据，就能同时送给所有请求它的线程。这两种情况下 Warp 在一个周期内，就能取得全部数据。

但是，如果这 32 个线程读取的同一个bank里的32个不同的地址，这就会造成 32 路 bank 冲突，需要串行读取。只要 warp 内多个线程访问同一个bank内的不同地址, 访存就必须串行化。代价就是多个线程同时访问同一个bank的线程数最大值。

# 2. 矩阵转置 Transpose

矩阵转置看起来是一个很简单的问题，但实际上它的计算非常耗时耗力，因为要把每一个元素从每行每列做一个交换。很多对延迟要求高的系统，对矩阵转置都会做一些处理。最简单的方式就是读入一行，写入一列。

![image.png](12%20CUDA%E7%A8%8B%E5%BA%8F%E6%B7%B1%E5%85%A5%E4%BC%98%E5%8C%96(%E5%9B%9B)%20Bank%E5%86%B2%E7%AA%81%20Bank%20Conflict/image%203.png)

矩阵转置对 global memory 的存有一个问题，就是不合并，我们必须要按照一个步长访问。

如果读入的时候是合并的，即按行读取，那么写入的时候就是按列写入，写入的时候是不合并的。如果我们写入的时候是合并的，即按行写入。那么读入的时候就不合并，读入的时候是按列读取。所以，读写总有一个是不合并的。一个最基本最原始的矩阵转置的算法如下：

![image.png](12%20CUDA%E7%A8%8B%E5%BA%8F%E6%B7%B1%E5%85%A5%E4%BC%98%E5%8C%96(%E5%9B%9B)%20Bank%E5%86%B2%E7%AA%81%20Bank%20Conflict/image%204.png)

我们希望通过 shared memory 来实现读写都合并。

我们先将 tile 的数据，从 idat 读入的时候就是以行读入 shared memory，这一行的数据就是 odata 的一列数据。idata 的数据全部读入 shared memory 后，再在shared memory中转置，然后再以行写入 odata, 最终完成转置操作。

![image.png](12%20CUDA%E7%A8%8B%E5%BA%8F%E6%B7%B1%E5%85%A5%E4%BC%98%E5%8C%96(%E5%9B%9B)%20Bank%E5%86%B2%E7%AA%81%20Bank%20Conflict/image%205.png)

（这里写的瓦片的多列元素，是站在 odata 角度的，这里就是指的 idata 的多行元素）

这里读和写都是行为单位，这就可以进行合并访存。有一点要注意，把数据从idat读入shared memory 的时候，需要同步，因为要等所有线程读取数据到 shared memory之后，才能进行转置，之后才能写入odata。

![image.png](12%20CUDA%E7%A8%8B%E5%BA%8F%E6%B7%B1%E5%85%A5%E4%BC%98%E5%8C%96(%E5%9B%9B)%20Bank%E5%86%B2%E7%AA%81%20Bank%20Conflict/image%206.png)

我们首先定义了shared memory 的一个区块tile，读数据的方式跟之前一致。但写数据时就有些变化。我们首先按行读入数据，将数据放入 shared memory 的 tile中。tile中的数据和读入的矩阵是1:1的复制，也是行优先存储。由于是按行读取的，warp可以合并访存。然后这里需要一个线程同步，目的是等待所有的线程把自己负责的数据都读入tile完毕之后，我们进行接下来的操作。
由于线程是按行写入tile的，现在要求tile的转置，就按列的数据, 以行的方式写入global memory中的odata，转置的操作实际上是在这里写操作执行的。

注意⚠️，这里线程编号没有变，所以可以合并访存的线程还是读取时候的那一组线程，我们做的是让刚刚按行读取一个 warp 的线程，现在按列从 tile 中取得数据，写入 odata 的时候，访存线程的编号没变，也是按行访存(写操作)，warp 仍然可以合并访存，只是现在写入的一行是 tile 中的一列，而 tile 是 idata 的副本，所以 odata 中就是 idata 的转置，并且，读和写都实现了合并访存。

<aside>
💡

注意⚠️

合并访存的概念，仅存在于对 global memory 访存的时候，对 shared memory 的存取数据，不存在合并访存的问题。shared memory 的问题，主要是 bank 冲突。

</aside>

# 3.Transpose 存在 Bank 冲突

假设 tile 是 16 * 16 的 float 数组，存在于shared memory。列中的数据存于相同的 bank。在准备写入 odata，要从 Tile 中读取一列数据时，存在 16-way bank conflict。

![image.png](12%20CUDA%E7%A8%8B%E5%BA%8F%E6%B7%B1%E5%85%A5%E4%BC%98%E5%8C%96(%E5%9B%9B)%20Bank%E5%86%B2%E7%AA%81%20Bank%20Conflict/image%207.png)

这是 shared memory 中的 tile，我们把数据读入 tile 的时候，使用的是“行”为单位，不会产生 bank-conflict, 但是当我们在 Tile 中按 “列” 取出数据，要写入 global memory 时，取数据的过程就会产生 bank conflict. 

这时候我们的做法是让 Tile多分配一列，定义tile的时候，定义成 `tile[TILE_DIM][TILE_DIM+1]`, 就如上图所示。现在就是 4 * 5 的 tile, 共 20个元素。二维数组是我们站在编程索引的角度，但是对于内存来说，就是一个一维数组，我们有 4个 bank，按顺序放入Tile 的话，就会造成 bank 错位:

![image.png](12%20CUDA%E7%A8%8B%E5%BA%8F%E6%B7%B1%E5%85%A5%E4%BC%98%E5%8C%96(%E5%9B%9B)%20Bank%E5%86%B2%E7%AA%81%20Bank%20Conflict/image%208.png)

![image.png](12%20CUDA%E7%A8%8B%E5%BA%8F%E6%B7%B1%E5%85%A5%E4%BC%98%E5%8C%96(%E5%9B%9B)%20Bank%E5%86%B2%E7%AA%81%20Bank%20Conflict/image%209.png)

![image.png](12%20CUDA%E7%A8%8B%E5%BA%8F%E6%B7%B1%E5%85%A5%E4%BC%98%E5%8C%96(%E5%9B%9B)%20Bank%E5%86%B2%E7%AA%81%20Bank%20Conflict/image%2010.png)

![image.png](12%20CUDA%E7%A8%8B%E5%BA%8F%E6%B7%B1%E5%85%A5%E4%BC%98%E5%8C%96(%E5%9B%9B)%20Bank%E5%86%B2%E7%AA%81%20Bank%20Conflict/image%2011.png)

最终在bank里面的分布就会变成

![image.png](12%20CUDA%E7%A8%8B%E5%BA%8F%E6%B7%B1%E5%85%A5%E4%BC%98%E5%8C%96(%E5%9B%9B)%20Bank%E5%86%B2%E7%AA%81%20Bank%20Conflict/image%2012.png)

<aside>
💡

**注意⚠️**

左边的图和右边的图，左图是逻辑的存储，右图是物理存储，逻辑存储上多加的一列，没有更多的 bank, 它只能循环到 bank 0 上。物理存储上，本来就是按地址顺序连续存储。只是我们多加了一列，把 bank 归属错开了而已。

没有任何区别。这个图展示的是bank 的归属分布。我们只有4个bank, 之前是 4 列数据，每一列放入一个bank。但是现在多填充了一列。因此，就会造成错位归属，例如上面的 m04 只能再次循环归属到 bank 0, 以此造成错位。

**这只是bank归属发生了错位变化，Tile 中数据存储的物理内存布局还是一样的**。

</aside>

这样处理之后，当我们要访问一列数据的时候，它就是访问的不同bank里面的数据，这样就避免了bank冲突。从行到列，都是落在不同的bank里面。所以 填充矩阵+1，就可以有效避免bank冲突。

<aside>
💡

**注意⚠️**

Bank 划分 和 Bank 冲突仅存在于 shared memory 中，而对 global memory 来说是不存在这个问题的。 

</aside>