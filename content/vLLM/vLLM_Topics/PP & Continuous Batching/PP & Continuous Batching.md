# 1. Continuous Batching

continuous batching 指的是如果一个batch 的内部，有 request 提前完成计算，它就立刻释放资源，让系统可以调度给下一个 request。但传统的 batch 处理，要求整个 batch 内所有 request 执行完之后，才能一起释放资源。

- <font color="b48ff4"><b>传统 Batching 处理方式</b></font>

![[Pasted image 20260602162814.png]]

上图中，有四个不同任务的时序，即这个 batch 中有 4 个 sequence。黄色部分是输入，蓝色部分是推理。右边是推理结束时的情况。输入 token 长度不同，需要的时间不同，推理输出的 token 长度不同，输出需要时间也不同。
比如，第一个任务，T1～T3 都在输入，T4 开始输出，T6完成，第二个任务T1～T2都在输入，T3开始输出，T8完成…以此类推。
所以第二个任务需要时间是最长的，只有等到它完成了推理，这个 Batch 中的显存才会被释放。所以右图中的白色方框，就是显存闲置的时候.

- <font color="b48ff4"><b>Continuous Batching 处理方式</b></font>
解决方案就是，<u>一旦一个 Batch 中的某个 seq 完成生成，就会发射一个 end-of-seq token，就可以在其他位置插入新的 seq 继续生成 token, 从而达到更高的GPU利用率</u>。

![[Pasted image 20260602163201.png]]

「Continous Batching」，与之前不同，在 batch 处理的时候，某个 request 处理完成，会发出一个END 信号，接受到END信号以后，系统就会立即调度处理下一个 request, 这个过程没有等待，提高了显存利用率。所以，<font color="orange">「Continuous Batching 是一个调度策略/机制」</font>

# 2. Batch 的滚动更新

实际上在一个 vLLM 实例处理 request 的过程中，<font color="#eb4349">「不存在多个 batch 的概念」</font>。逻辑上可以理解为<font color="#45ce6e">「只有一个 batch」</font>, 在里面的 request 完成后，立刻释放资源，然后调度下一个 request 加入这个 batch 形成新的 batch。整个过程就是围绕这一个 batch 做<font color="#45ce6e">「滚动更新」</font>。

![](Pasted%20image%2020261031133127.png)

系统维护一个活跃请求池。就由调度器调度加入 batch。在每轮计算后(每一轮就是每次向前计算，即 batch 中的每个请求，向前推理算出下一个 token)，如果有 request 结束退出，就从请求池中选择一个各方面条件允许的请求，加入 batch。

我们可以通过配置设置 batch size。所以正常情况下，request充分，系统中就始终有 batch size 个 request 在处理。但这通常是上限，实际 batch 运行时的大小，还受等待请求数量，token 预算，和 KV Cache 空间影响，不意味着每时每刻都有 batch size 个 request 在处理。

此外，batch size 的大小可以设置成小于系统能力，比如系统能力是处理 100 个 request, 但是设置的 batch size 只有 10.  当有 100 个请求到达的时候，只有 10 个 request 会组成一个 batch, 进行处理，其余 90 个必须等待。

系统始终处理的是这一个 batch，这个 batch 在处理的过程中进行滚动更新。滚动更新就是 <font color="#45ce6e">「Continuous Batching」</font> 这个词的由来。多个 batch，是在传统 static batching 和 dynamic batching 下的概念，因为它们设定 batch，系统必须以 batch 为调度单位，统一进行处理和释放资源，所以可以明确分割为 batch A , batch B, batch C。
Continuous Batching 中的 batch, 逻辑上就一个, 从头到尾不断滚动更新, 如果非要对应于多个 batch 的概念，我们可以说，一开始是 batch A，在有 request 结束退出时，新加入一个 request, 然后新旧 requests 一起又组成一个新的 batch，我们可以叫做 batch B, 以此类推。

这就是 Continuous Batching 的基本原理。

# 3. PP 空泡问题, Pipeline Bubble

<font color="orange">「PP, Pipeline Parallelism 流水线并行」</font>，会把模型分成几段，放在不同 GPU 上，每个 GPU 上的一段叫做一个 stage。
<font color="orange">「空泡，bubble」</font>就是某段 GPU 因为等待数据或计算依赖，暂时没有有效计算的空闲时间。
例如把一个模型分成 4 段：

```
输入 -> GPU 0(1/4层) -> GPU 1(2/4层) -> GPU 2(3/4层) -> GPU 3(4/4层) -> 输出
```

如果按照默认的整个 batch 当成一个整体传递，GPU 0 工作时，其余 GPU 等待。轮到 GPU 1 时，GPU 0 又闲下来。模型虽然占用 4 张卡，但是同一时刻却可能只有一张卡在计算。

 <font color="b48ff4"><b>Continuous Batching 的空泡问题</b></font>

Continouous Batching 从头到尾只有一个 batch 在工作，按照上面的讨论，只有在结束一轮向前推理之后，有完成的 request 退出，新的 request 加入，这时才可以说有了下一个新 batch。因此整个流水线，实际上在处理这一个 batch。

此外，每个 GPU stage 往下一个 GPU stage 传递时, 是按 batch 传递的，意思就是说，一个 batch 的所有 request 必须都完成目前 stage 的计算，这个 batch 才能整体传递给下一个 stage。continuous batching 在这种情况下，就会造成同一时间，只有一个 GPU stage 在工作，其他所有 GPU 都空闲，因为 batch 作为整体，要一个 stage 一个 stage 的经过。

![](Pasted%20image%2020261031150238.png)

走完一次完整流水线，才能产生一个推理的 token，流水线上的任何时刻，对 batch 内的 request 都叫做「正在计算」状态。batch 中的 request 都是正在处理状态，此时即使有等待也是无法插入新 request 的，因为每个 request 在流水线上，都叫做正在处理。

# 4. microbatch

continuous batching 始终只有一个滚动 batch, 不可避免的就会形成 PP 空泡。

<font color="b48ff4"><b>PP 使用 microbatch 填满流水线</b></font>

这时候，前面讨论的物理意义上的划分多 batch, 在 PP 流水线部署下，就有意义了。可以把逻辑上一个大 batch 分割成多个较小的执行 batch，每个 batch 中含有相对较少的 requests 数量。 这个较小的执行 batch 就是用于流水线的<font color="#45ce6e">「microbatch」</font>。microbatch 依此送入流水线，不同的 stage 同时处理不同 batch, 尽量填满流水线。因此，如果 batch A, B, C, D 依此送入流水线，4个时间间隔后就会形成

```text
同一时刻下：
GPU0: batch D 的 stage 0
GPU1: batch C 的 stage 1
GPU2: batch B 的 stage 2
GPU3: batch A 的 stage 3
```

这些 batch 中的请求也在完成后滚动替换. 具体分成多少组，是否自动分组，由框架调度实现决定。

比如把 8 条请求拆成 4 个 microbatch：A、B、C、D，每份 2 条。GPU0 算完 A 并交给 GPU1 后，立即开始计算 B，无须等 A 跑完整个模型。假设各段耗时相同、忽略通信，前向计算的时间线如下；`·` 表示空泡：

|GPU / 时间片|1|2|3|4|5|6|7|
|---|---|---|---|---|---|---|---|
|GPU0|A|B|C|D|·|·|·|
|GPU1|·|A|B|C|D|·|·|
|GPU2|·|·|A|B|C|D|·|
|GPU3|·|·|·|A|B|C|D|

第 4 个时间片，4 张 GPU 都在工作。**同一份数据仍然依次经过各段，但多份数据可以同时处于不同段。** 这就是 microbatch 填充流水线的原理。


<font color="b48ff4"><b>TP 下只有一个 batch</b></font>

TP 下， 在一个 TP 组内的多张 GPU 共同完成一个 batch 的计算。在计算某一层时，各 GPU 分担这层的不同张量分片。

```
GPU0 ─┐ 
GPU1 ─┼─ 共同计算 batch A → 生成 token → 更新成员 → 下一轮 
GPU2 ─┤ 
GPU3 ─┘
```

# 5. 总结

Continuous Batching 下，不管是 PP 还是 TP，逻辑上就是一个大的 batch 在不断滚动更新。也就是选一组 request, 让他们各自向前推进，普通 decode 下就是各自生成一个token。完成的请求退出，新请求补入，形成新的 batch。

TP 执行时：多张 GPU 分担层内的计算，共同处理这个大的逻辑 batch。
PP 执行时：可以把这个大的逻辑 batch 拆分成多个 microbatch, 让不同 stage 同时处理不同 micro batch，减少空泡。

PP 下的 microbatch 的执行流程为:

```
batch 进入 GPU0 
→ 依次经过各 stage 
→ 最后一个 GPU 完成计算、采样得到 token 
→ 逐条判断 request 是否结束 
→ 回收已结束请求可释放的资源 
→ 补入新请求 
→ 更新后的 batch 从 GPU0 开始下一轮
```

其中，未结束的请求保留 KV Cache, 继续生成后续 token。流水线填满的情况下，某时刻，每个 GPU 上都有一个 microbatch 正在执行这个 stage 的计算。