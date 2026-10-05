# NVIDIA GPU 产品架构体系

# 1. GPU 架构

NVIDIA GPU 的架构现在主流的是 H(Hopper) 系列和 B 系列(Blackwell)。GPU 架构的演进历经 

```
volta -> Turing -> Ampere -> Hopper -> blackwell -> Rubin
```

不同的架构，对应不同的算力版本 CC, Compute Compatibility, 例如 Turing 的 CC 就是 sm_75, Ampere 的 CC 就是 sm_80。 

CUDA 以及其他软件(如vLLM) 通过对算力版本的支持，来指定对硬件平台 GPU 架构的支持。我们编写的 CUDA 程序要编译成匹配的 sm_xx 版本的目标程序后，才能在相应的 GPU 平台上运行。

| 核心代际 | 架构名称（代表GPU） | 关键进展与性能提升 | GPU 产品型号 | Compute Compatibility |
| --- | --- | --- | --- | --- |
| **第1代** | Volta (V100) | 引入FP16混合精度，AI训练性能比前代 Pascal提升12倍。 | V100 | sm_70 |
| **第2代** | Turing (RTX 20系列) | 首次进入消费级市场，支持 INT8/INT4 精度，AI推理性能大幅提升。 | T4, 2080ti 等20系显卡 | sm_75 |
| **第3代** | Ampere (A100) | 引入TF32、BF16 和FP64 精度，同时支持结构化稀疏计算，性能模式更灵活。 | A100，A800, A10, 30系 RTX  | 
sm_80/sm_86/sm_87 |
| **第3.5代** | Ada Lovelace | RTX 40系，算力版本sm_89. 还是属于 sm_8x | RTX 4090 等 | sm_89 |
| **第4代** | Hopper (H100) | 新增 Transformer 引擎，对 LLM 训练和推理进行针对性优化。 | H100，H200，GH200 | sm_90 |
| **第5代** | Blackwell (B100/RTX 50系列) | 引入FP4/FP6精度，能效比显著提升，可将游戏帧率提升至原来的8倍。 | B100, B200，GB200, 5090 | sm_100 (B100,B200) sm_120 / sm_121(RTX5090) |
| 第5.5代 | Blackwell Ultra | Blackwell 的升级架构 | B300，GB300 | sm_103 |
| 第6代 | Rubin | 搭配新的 CPU 架构 Vera | Vera Rubin GPU | sm_107 |

<aside>
💡

注意⚠️

从 Hopper 架构之后，就新增了 Transformer 引擎，对 LLM 的训练和推理进行了针对性优化。GPU 型号，A100，指的就是 Ampere 100。H100, 指的就是 Hopper 100。 B100 指的就是 Blackwell 100。这是 Nvidia 显卡的命名规则。 

</aside>

# 2. NVIDIA GPU 产品体系

这里列举 NVIDIA 推出的主流 GPU 产品以及其分类方式

## ◼︎ H100

H100 是 Hopper 架构下的首款产品，对应的算力版本为 0.9，编译目标版本为 `sm_90`.  产品形态分为三个版本，H100 PCIe 80GB, H100 NVL 94GB, H100 SXM.

- H100 PCIe 80GB 版本，GPU 显存 80GB，GPU 与主板间插卡接口是 PCIe, GPU 间通过 PCIe 总线通信和传输数据。可以自己外接 NvLink 桥接器
- H100 NVL 94GB 版本，GPU 显存 94GB，GPU 与主板间插卡接口是 PCIe, GPU 两两通过 NvLink 桥接器连接，8 卡形成 4 个双卡组。组内两张 GPU 通过 NvLink 通信，组与组之间通过 PCIe 总线通信。

![image.png](3%20NVIDIA%20GPU%20%E4%BA%A7%E5%93%81%E6%9E%B6%E6%9E%84%E4%BD%93%E7%B3%BB/image.png)

上面 8 张GPU，每两张之间用 3 个 NvLink 桥接器连接起来。

- H100 SXM，GPU 显存 80 GB, GPU 与主板间插卡接口是 SXM 模块。GPU内 NvLink 专用电路连接 SXM 连接器，通过底板走线，直接连接 NvSwitch 芯片。通过 NvSwitch 转发实现多 GPU间的 NvLink 互联。

| 产品规格 | 架构/算力版本 | 显存规格/大小/显存带宽 | 主板接口/GPU通信带宽 | 功耗TDP / 8卡服务器整机功耗 |
| --- | --- | --- | --- | --- |
| H100 PCIe 80GB | Hopper / sm_90 | HBM2e / 80GB / 2.0 TB/s | PCIe 5Gen / 128GB/s  (双工) | 350w / 10.2kw |
| H100 NVL 94GB | Hopper / sm_90 | HBM3 / 94 * 2=188GB,
双GPU / 3.9 TB/s | PCIe 5Gen / 双卡3个NvLink桥接器, 每卡 600GB/s  (双工) | 350~400w / 10.2kw |
| H100 SXM | Hopper / sm_90 | HBM3 / 80GB / 3.35 TB/s | SXM / NvSwitch  每卡 900GB/s (双工) | 最高 700w / 10.2kw |

<aside>
💡

NvLink 带宽说明

我们通常说的 NvLink 的带宽，指的是双向带宽加起来的总和，比如双向 900 GB/s, 就意味着单向 450 GB/s。NvLink 双向带宽其实和每卡带宽是一回事，因为每个 GPU 有发送和接受两个方向，所以我们在说每卡/单卡带宽的时候，就是指 GPU 发送和接受两个方向的带宽总和，也就是双向带宽。

</aside>

## ◼︎H200

H200 是 H100 的升级产品，编译目标版本为 `sm_90`.  NVIDIA 为 H200 只提供了两个版本，H200 NVL 与 H200 SXM。产品上取消了类似 H100 PCIe 版本的产品形态，但是 NVL 的插槽仍然是PCIe, 组与组之间的 GPU 通信，也是采用 PCIe 总线。 

H200 NVL 的显存是 141GB HBM3e, GPU 主板之间的插卡接口是 PCIe 5Gen. H200 NVL 为 8 卡 GPU 服务器提供了两种桥接方案，一种是 2 卡 NvLink 桥接，一种是 4卡 NvLink 桥接。

双卡 NvLink 方案，使用的是专供 H200 的「2-way NvLink Bridge」，它是「一体式桥接器」，与H100 在双卡间使用 3个 NvLink 桥接器不同，但是它们都提供 600 GB/s 的带宽。四卡 NvLink 方案，使用的是专供 H200 的 「4-way NvLink Bridge」。它直接连接 4 张 GPU。

![image.png](3%20NVIDIA%20GPU%20%E4%BA%A7%E5%93%81%E6%9E%B6%E6%9E%84%E4%BD%93%E7%B3%BB/image%201.png)

上图是用了4-way NvLink Bridge 将 4 张 GPU 桥接，分为两组，形成 2 个四卡组。

H200 NVL 提供了 4个双卡组 和 2个四卡组 两种方式，一个八卡服务器可以这样划分

| 配置 | NVLink 分组 |
| --- | --- |
| **2 组 × 4 卡** | `{GPU 0、1、2、3}`、`{GPU 4、5、6、7}` |
| **4 组 × 2 卡** | `{GPU 0、1}`、`{GPU 2、3}`、`{GPU 4、5}`、`{GPU 6、7}` |

组内 GPU 之间的通信使用 NvLink, 组与组之间的 GPU 通信依然使用 PCIe

H200 SXM 版本，GPU 与主板间通过 SXM 模块连接，GPU内部的 NvLInk 电路通过 SXM 连接器，底板走线，直连 NvSwitch 芯片。8 卡 GPU 通过 NvSwitch 实现了 NvLink 的多卡互连。

| 产品规格 | 显存规格 / 大小 / 显存带宽 | 主板接口 / GPU 通信带宽  | 功耗TDP/ 8卡整机功耗 |
| --- | --- | --- | --- |
| H200 NVL 2-way桥接 | HBM3e / 141GB / 4.8 TB/s | PCIe 5Gen/ NvLink 每卡 900GB/s (双工) | 600w / 10.2kw |
| H200 NVL 4-way桥接 | HBM3e / 141GB / 4.8 TB/s | PCIe 5Gen/ NvLink 每卡 900GB/s (双工) | 600w / 10.2kw |
| H200 SXM | HBM3e / 141 GB/ 4.8 TB/s | SXM / NvSwitch 每卡 900 GB/s (双工) | 700w / 10.2kw |

## ◼︎B200 与 GB200

到了 Blackwell 时代， B200 和 B300 的 GPU，单卡 / 模组层面 (HGX / DGX 8-GPU 系统) 的主力产品形态已经只有 SXM 版本了，已经不再提供 NVL 版本了。

<aside>
💡

需要注意⚠️：GB200， GB300 服务器是按 NVL 版本提供的。但是含义已经变了。B系列中，SXM 和 NVL 已经不是同一级别的两个“显卡版本了”。B系列的单卡或者 8 卡服务器的产品，只提供 SXM 版本，抛弃了类似 H200 的 NVL 版本。而 GB 系列产品提供 NVL 版本，而这个 NVL 的含义已经发生了变化。

</aside>

GB200(以及GB300) 这样 GB 系列的产品，不是一个独立的产品线，而是一种特殊产品形态。GB系系列是以「核心计算模组」方式提供产品，即 「1 颗 Grace 架构的 CPU + 2 颗 Blackwell 架构的B200 GPU 」组成的芯片模组，这个产品叫做 「Superchip」。

GB200 的存在是为了 NVIDIA 能够提供 GB200 NVL72 的机柜产品，它是由 36 颗 Grace CPU + 72 颗 B200 GPU 芯片组成的一体化机柜产品。GB200 芯片模组就是其组成部分。

注意⚠️：这里的 NVL 不再是 H100/H200 中 NVL 代表的 PCIe 插口形态了。它的含义是其中的 72 颗GPU 采用 NvLink 互联网络相互通信。这里的 B200 和主板间是焊死的，可以认为采用的板插接口就是 SXM。B系列已经放弃 PCIe 板插接口了，但是 GPU 与 CPU 等硬件之间的通信，还是使用 PCIe 总线。 

GB200 NVL72 过于庞大。NVIDIA 也为 GB200 提供了小一些的服务器产品，GB200 NVL4, 它是由 2 个 Superchip 组，即 2 颗 Grace CPU + 4 颗 B200 GPU 组成的服务器系统。

注意⚠️ GB200 没有提供 8卡服务器产品形态。只有 GB200 NVL4。

| 产品规格 | 显存 / 大小 / 带宽 | 主板接口 / GPU 通信带宽 | 单卡功耗TDP |
| --- | --- | --- | --- |
| B200 | HBM3e / 180GB / 7.7 TB/s | SXM / NvLink 每卡 1.8 TB/s (双工) | 1000w  |
| GB200 NVL4 | HBM3e / 186GB / 8 TB/s | SXM / NvLink 每卡 1.8 TB/s (双工) | 1200w |
| GB200 NVL72 | HBM3e / 186GB / 8 TB/s | SXM / NvLink 每卡 1.8 TB/s (双工) | 1200w |

## ◼︎B300 与 GB300

B300 使用的芯片架构是 Blackwell 的升级架构，叫做 Blackwell Ultra。算力版本是10.3, 编译目标是 sm_103。

B300 与 GB300 之间的关系，和 B200 与 GB200 之间的关系是完全一样的。B300 同样提供了 8卡 服务器产品。不同之处在于， GB300 只提供了GB 300 NVL72 的机柜产品，没有像 B200 NVL4 一样的小型服务器产品了。

 

| 产品规格 | 显存 / 大小 / 带宽 | 主板接口 / GPU 通信带宽 | 单卡功耗 TDP |
| --- | --- | --- | --- |
| B300 | HBM3e / 270 GB / 7.7 TB/s  | SXM / NvLink 每卡 1.8 TB/s (双工) | 1100w |
| GB300 NVL72 | HBM3e / 279 GB / 8 TB/s  | SXM / NvLink 每卡 1.8 TB/s (双工) | 1400w |

## ◼︎Vera Rubin

Rubin 是 NVIDIA 推出的最新一代 GPU 架构。并且还推出新一代的 CPU 平台，Vera。Vera Rubin 就是 CPU 搭配 GPU 的产品形态。

NVIDIA 提供的 Rubin 产品线更加丰富：

- Vera Rubin Superchip, 芯片模块组, 一颗 Vera CPU + 2 颗 Rubin GPU
- Vera Rubin NVL4, 搭载 2 个 Vera Rubin Superchip 的小型服务器
- Vera Rubin NVL8,  8 卡 Rubin GPU + 单个 Vera CPU 的服务器 HGX
- Rubin NVL8， 8 卡 Rubin GPU 的 HGX，不含 CPU，CPU 由 OEM 提供
- Vera Rubin NVL72，整机机柜产品，36颗 Vera CPU + 72 颗 Rubin GPU。

| 产品规格 | 显存 / 大小/ 带宽 | 主板接口 / GPU 通信带宽 |
| --- | --- | --- |
| Rubin GPU | HMB4 / 288 GB / 19.25 TB/s | SXM / 3 TB/s |

注意⚠️：Rubin 产品全面升级。不仅其搭载的 CPU 更新到了 Vera 架构。其显存还升级到了 HMB4，单卡大小 288 GB，显存带宽已经达到了 19.25 TB/s。NvLink 也升级到了第六代，每卡通信带宽达到了 3 TB/s 。(双工：“发送 + 接受” 的双向合计带宽)

## ◼︎GB100 与 GH100，GPU 芯片核心代号

H100，H200，B200，B300 以及 GB200, GB300, 这些都是 NVIDIA 基于 Hopper 架构和 Blackwell 架构，推出的 GPU 产品型号。

GB100 与 GH100 的编号比较特殊，它们不是指的产品型号。它们是指的 「GPU 芯片核心代号」。意思就是 H100，H200 的 GPU 是基于 GH100 芯片设计出来的 GPU 产品。B200 是基于 GB100 芯片设计出来的 GPU 产品。

B300 比较特殊，它基于的芯片核心代号为 GB110，因为 B300 是基于 Blackwell 的升级架构 Blackwell Ultra。