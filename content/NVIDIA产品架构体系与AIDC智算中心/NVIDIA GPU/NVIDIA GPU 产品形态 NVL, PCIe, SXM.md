# NVIDIA GPU 产品形态 NVL, PCIe, SXM

下面以 H100 为例进行分析。产品规格，技术逻辑，分层方式等分析，可以扩展至整个 NVIDIA GPU 产品线，作为 NVIDIA GPU 产品的「通用知识框架」。

H100 服务器有三个版本(不是每代 NVIDIA 设备都有三个版本，但划分的技术逻辑都一样)。H100 SXM，H100 PCIe 80GB, H100 NVL 94 GB.

# 1. H100 SXM

H100 SXM 版本的 GPU 与主板之间的接口是 SXM 模块。GPU 内部的 NvLink 接口线路连接模块内的SXM 连接器，通过模块下面的底板走线，直接连接到 NvSwitch 芯片上， 形成 NvLink 链路。

GPU 间通信和数据交换，就是通过 NvLink 链路，传输到 NvSwitch 芯片上，再由芯片转发到怕目标GPU上。

# 2. H100 PCIe 80GB

H100 PCIe 80GB 版本的 GPU 与主板间的接口形态，采用是 PCIe, 现在是第五代，PCIe 5Gen。80GB 是显存大小。PCIe 版本的显卡之间采用的是 PCIe 总线传输协议。所以 PCIe 在这里既表示接口的硬件形态，也表示数据传输的总线协议。

这个版本的 GPU 间通信和数据传输，就是采用 PCIe 总线，而没有 NvLink。但是我们可以外接 NvLink 桥接器，将两张 GPU 用 NvLink 连接起来。服务器 8 卡 GPU，分为 4 个双 GPU 组。组内2张 GPU 之间的通信使用 NvLink 链路。组与组之间的通信和数据传输，还是使用 PCIe.

总之，H100 PCIe 80GB 版本的服务器，使用 PCIe 接口形态，GPU通信使用 PCIe 总线。没有 NvLink, 但是我们可以自己外接 NvLink 桥接器。

# 3. H100 NVL 94GB

这个版本的 H100 服务器，GPU 与主板之间的接口形态采用 PCIe, 但是它默认将 8卡GPU，用 NvLink 桥接器两两桥接配置，形成 4 个双卡组。组内两张 GPU 之间，使用 NvLink 通信，组与组还是使用 PCIe 总线。

```
第 1 组：GPU 0 ══ NVLink ══ GPU 1
第 2 组：GPU 2 ══ NVLink ══ GPU 3
第 3 组：GPU 4 ══ NVLink ══ GPU 5
第 4 组：GPU 6 ══ NVLink ══ GPU 7

组与组之间：通过服务器的 PCIe 拓扑通信
```

8 卡不会形成 8 卡全互连的 NvLink 网络。

# 4. H100 NVL .vs. H100 PCIe 外接 NvLink

两个版本都是八卡服务器，通过 NvLink 两两组成四组，两者的拓扑结构基本一样，并且 H100 NVL版本就是使用的 NvLink 桥接器，与 PCIe 外接 NvLink 方案，并「没有差别」。它们之间的差异主要在 GPU 和显存规格。

NVL 的 GPU 是 94GB，双卡互连是 188GB. 而 PCIe 的 GPU 是 80GB，双卡互连时，只有 160GB.

| 对比项 | H100 PCIe 80GB，加装 NVLink 桥接器 | H100 NVL 94GB，使用 NVLink 桥接器 |
| --- | --- | --- |
| 硬件形态 | PCIe 插卡 | PCIe 插卡 |
| 每卡显存 | 80GB HBM2e | 94GB HBM3 |
| 每卡显存带宽 | 约 2TB/s | 约 3.9TB/s |
| 每对 GPU 显存容量合计 | 160GB | 188GB |
| 每对 NVLink 双向总带宽 | 600GB/s | 600GB/s |
| 八卡连接方式 | 四组双卡 | 四组双卡 |
| NVSwitch | 无 | 无 |

NVL 的主要优势是每张 GPU 的显存更大、显存带宽更高，而不是这四组 GPU 之间多了一种互连.

# 5. SXM .vs. NVL/PCIe

SXM 是 GPU 接入主板时使用的接口模块，里面有 SXM 连接器，它把 GPU 内部的 NvLink 接口电路直接通过底板走线，连接到 NvSwitch 芯片上。而 NVL/PCIe 是通过外接 NvLink 桥接器将两张 GPU进行 NvLink 互连。 NVL/PCIe 是不支持 NvSwitch 的。

| 场景 | NVLink 信号通过什么连接 |
| --- | --- |
| H100 PCIe / NVL 双卡桥接 | 外置 NVLink 桥接器 |
| H100 SXM 与 NVSwitch 连接 | SXM 连接器和底板走线 |

# 6. H100 PCIe/NVL GPU 的 NvLink 桥接口

H100 的 PCIe/NVL 版的 GPU，每卡有三个 NvLink 桥接口。因此一对 GPU 要用三个 NvLink 桥接器，连接两个 GPU 之间的 3 组 NVLink 桥接口。一个 NvLink 桥接器提供的带宽是 200 GB/s, 三个桥接器加起来就是 600GB/s。

```
H100 NVL 卡 A ═══ 三个 NVLink 桥接器 ═══ H100 NVL 卡 B
```

官方要求桥接时，安装全部三个NvLink 桥接器

![image.png](2%20NVIDIA%20GPU%20%E4%BA%A7%E5%93%81%E5%BD%A2%E6%80%81%20NVL,%20PCIe,%20SXM/image.png)

三个桥接器一起提供 600GB/s 的双向总带宽。

H200 NVL 提供了 2 卡 NvLink 桥接方式和 4 卡 NvLink 桥接方式。因此，NVL 版本提供了专供H200 的 「2-way NvLink Bridge」 和 「4-way NvLink Bridge」. 两路和四路 NvLink 桥接器都是一个整体式的桥接器，与 H100 使用 3 个 NvLink 桥接器的方式不同。

 4 张 GPU 用一个 4-way NvLink Bridge 连接起来

![image.png](2%20NVIDIA%20GPU%20%E4%BA%A7%E5%93%81%E5%BD%A2%E6%80%81%20NVL,%20PCIe,%20SXM/image%201.png)

![image.png](2%20NVIDIA%20GPU%20%E4%BA%A7%E5%93%81%E5%BD%A2%E6%80%81%20NVL,%20PCIe,%20SXM/image%202.png)

4-way NvLink Bridge

![image.png](2%20NVIDIA%20GPU%20%E4%BA%A7%E5%93%81%E5%BD%A2%E6%80%81%20NVL,%20PCIe,%20SXM/image%203.png)

2-way NvLink Bridge

# 7. SXM vs NVL vs PCIe

H100 的服务器有三种版本形态，SXM，NVL，PCIe. 它们之间的对比如下：

| H100 产品版本 | 硬件形态 | 显存（每 GPU） | NVLink 连接方式 | 是否使用
NvSwitch |
| --- | --- | --- | --- | --- |
| H100 SXM | SXM 模块 | 80GB | 通过服务器 GPU 底板连接，典型八卡系统使用 NVSwitch | 使用 |
| H100 PCIe | PCIe 插卡 | 80GB | 可通过 NVLink 桥接器连接两张卡，两卡合计 160GB | 不使用 |
| H100 NVL | **PCIe 插卡** | 94GB | 通过 NVLink 桥接器连接两张卡，两卡合计 188GB | 不使用 |

# 8. SXM 与 NVL 版本的使用对比

从对 GPU 的使用来说， SXM 版本追求的是极致的单卡性能和多卡并联扩展，对训练的场景做了极致优化，它更适合「大规模训练任务」。

而 NVL 是专门为「大规模推理任务」优化的 “双芯板卡”。PCIe 外形更便于部署，并通过板载 NvLink 桥接器将两个 GPU 融合(Crossfire), 关键是释放了全部显存芯片，最大可以释放双卡 94GB x 2 = 188GB 显存。专门为在算力中心的巨型模型设计。

# 9. HGX / DGX

刚刚我们讨论的 H100 SXM / NVL / PCIe 是 GPU 产品形态层面的特性。NVIDIA 还以「集成模块」和「整机形式」提供产品，即 HGX 和 DGX。

HGX 是 NVIDIA 提供给服务器厂商集成的 GPU 加速平台。而 DGX 是 NVIDIA 提供的自己品牌的完整 AI 系统。它们属于 GPU 平台和整机系统之间的关系。它们之间的关系如下：

| 对比 | HGX | DGX |
| --- | --- | --- |
| 产品定位 | 供服务器厂商集成的加速平台 | 完整的软硬件系统 |
| 核心内容 | GPU、GPU 底板、NVLink/NVSwitch 等 | GPU 加速平台，加上 CPU、内存、存储、网络、散热及软件 |
| 谁提供整机 | 戴尔、HPE、联想、超微等服务器厂商 | NVIDIA 品牌，通常通过合作伙伴销售 |
| 配置方式 | 服务器厂商围绕平台设计不同整机 | 按具体 DGX 型号提供标准化配置 |

所以，平时我们看到的戴尔，联想，超微等服务器厂商销售的 H100 服务器就是用的 NVIDIA 提供的HGX GPU 加速平台，厂商自己做集成。而 DGX 是按标准化配置，由 NVIDIA 自己品牌做的集成。

```
HGX H100 八卡 GPU 平台
├── 8 张 H100 SXM GPU
├── 4 颗 NVSwitch
└── GPU 底板及互连
          │
          ├── 服务器厂商集成 → 该厂商的 HGX H100 服务器
          │
          └── NVIDIA 的完整系统方案 → DGX H100
```

# 10. NVIDIA GPU 产品体系层次

对于 H100， 我们可以按照下面的方式分层

| 层面 | 例子 | 描述什么 |
| --- | --- | --- |
| GPU 产品与形态 | H100 SXM、H100 PCIe 80GB、H100 NVL 94GB | 使用哪种 GPU、显存规格和安装形态 |
| GPU 加速平台 | HGX H100 八卡平台 | 如何将 GPU、底板、NVSwitch 等组合起来 |
| 完整服务器 | 厂商的 HGX H100 服务器、NVIDIA DGX H100 | 再加入 CPU、内存、存储、网络、供电散热等 |
| 集群系统 | DGX BasePOD / SuperPOD 等 | 将多台系统与网络、存储、管理软件集成起来 |

对 NVIDIA 的所有产品，它具备如下层次

| 层级 | 型号 | 性质 |
| --- | --- | --- |
| GPU芯片 | B100, B200, H100, H200, A100 | 单个GPU显卡产品型号。 |
| Superchip 模组 | GB200, GH200, GB300 | 1 颗CPU + 2 颗 GPU 封装组合，前面的 G，代表 Grace CPU, 加上两颗相应型号的GPU。 |
| 集成 GPU 加速平台 | HGX 系列 | 提供 GPU 加速平台给集成商。戴尔，超微等品牌的服务器，就是由这些厂牌集成的。 |
| NVIDIA服务器整机 | DGX 系列 | NVIDIA 官方以相应型号GPU 打造的 8卡服务器平台。 |
| 机柜方案 | GB200 NVL72， NVL36 | 36个Grace CPU + 72张GPU的整机柜方案，在一个NvLink 域内，72张GPU被当作一个超大的GPU 使用 |
| 集群方案 | SuperPOD | 多机柜集群 |