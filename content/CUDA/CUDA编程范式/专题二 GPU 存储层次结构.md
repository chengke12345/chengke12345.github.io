# 专题二：GPU 存储层次结构

# 内存层次结构

┌─────────────────────────┐
│   计算单元 (CUDA Core)   │
│   只能从寄存器读操作数    │
└───────────┬─────────────┘
│ 1 周期
┌───────────▼─────────────┐
│   Register File (SRAM)  │  ← 每个线程私有。编译器静态分配。
│   每 SM 约 256 KB ｜← SM中寄存器阵列分为若干sub-partition. 每个组有自己的控制单元和调度器
└───────────┬─────────────┘
│ ~20-30 周期 (LDS 指令)
┌───────────▼─────────────┐
│   Shared Memory (SRAM)  │  ← block 内线程共享
│   每 SM 约 100 KB       │  ← block不能跨SM。block中的warp可以跨 SM 内部 sub-partition
│   分 32 个 bank         │ ← 但warp中的线程不可以跨sub-partition，warp是线程被调度的最小单元。
└───────────┬─────────────┘
│ ~400-800 周期 (LDG 指令)
┌───────────▼─────────────┐
│   Global Memory (DRAM)  │  ← 所有 block 共享
│   GB 量级               │
│   走 L2 / L1 cache      │
└─────────────────────────┘

一句话总结：CUDA编程的核心，就是在 Register → Shared memory →global memory 之间调度数据，让系统获得更好的性能。

# 线程层次结构

GPU中的线程的层次结构分为两个视角来看

软件视角：Thread → block → Grid

硬件视角：Thread → warp → SM 

我们在软件操作层面，可以将线程分为线程，若干线程组成一个block，若干个block组成一个Grid。我们在启动核函数的时候，可以指定需要启动多少个block，每个block多少线程。一个block只能分配到一个SM上去。但是 block 中的 warp 可以被分配到不同的 sub-partition 中去。SM 的计算单元和寄存器被分为若干sub-partition, 每个组有自己独立的warp-scheduler和控制逻辑。warp是被调度的最小单元。所以一个warp中的线程，不能跨partition存在。

一个warp固定32个线程，是硬件视角调度的最小单位，软件层面对它的感知不强烈。调度时，调度器只知道warp, 不知道block的存在，block是软件层面的划分。block是线程并行处理的逻辑单位，它是为了分层次更好的索引线程，并且它只能被分配到单个SM上。