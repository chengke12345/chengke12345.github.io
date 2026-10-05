# NvLink & NvSwitch

# 1. GPU 通信和数据传输接口

**PCIe：**Peripheral Component Interconnect Express, 外设互联高速通道。是一种总线标准协议。也是外设插入主板的一种板卡接口标准/形态。所以，PCIe 既指总线传输协议，也指接口。

**SXM**：是 Nvidia HGX 等专用底板使用的外设接口模块。这种专用底板，要承载供电，散热，PCIe 主机连接，以及大量 NvLink 通道的功能。

**NvLink**: Nvidia 提供的，GPU之间的专用高速链路，比 PCIe 带宽更高，延迟更低。

# 2. SXM、PCIe、NvLink 之间的关系

GPU 与主板之间的接口分为两种，一种是 PCIe，最新第五代接口是 PCIe 5Gen, 另一种是 SXM 接口。而 GPU 与 GPU 之间的通信，可以使用 PCIe 总线，也可以使用 NvLink 专用高速链路。

<aside>
💡

GPU接口 与 数据链路

- GPU 与 主板间的接口与 GPU 之间的数据传输链路是两个层次上的事。GPU 插入主板的接口可以是 PCIe 插卡形态或者 SXM 模块形态。GPU 之间数据传输，可以使用 PCIe 总线协议或者 NvLink 高速链路。
- 两张通过 PCIe 接口插入主板的 GPU 之间，通信或传输数据，可以使用 PCIe 总线协议，也可以使用 NvLink(GPU必须支持，比如 2080ti, 3090等)。两张通过 SXM 接口插入主板的 GPU 也可以选择使用 PCIe 或者 NvLink.
</aside>

# 3. NvLink 与 NvSwitch

## ◼︎ NvLink 专用高速链路

NvLink 是 Nvidia 提供的一种 GPU 间通信和数据传输的专用高速链路。具体的，NvLink 是一种高速互连技术，包含通信协议，电气信号规范和硬件接口，而不止是软件协议。

NvLink 既可以连接 GPU 与 GPU，也可以连接 GPU 与 NvSwitch, 它是一段点对点的数据链路。 NvLink 这个整体就是 GPU 间的专用高速数据链路。

NvLink 不是主板上的协议(对比，PCIe是主板上的协议)。因此，不能天然“打开” NvLink 使用, 必须要有相应的硬件支持，才能使用 NvLink。主要有两种方式，通过 NvLink 桥接器连接，或者通过GPU 接口底盘走线直连 NvSwitch。 

## **◼︎ NvLink 桥接器**

NvLink 桥接器是使用和实现 NvLink 专用高速数据链路的一种硬件方式。还有其他方式，比如通过NvLink 链路直连 NvSwitch。

我们在一些场景下提到 NvLink, 实际上是 NvLink 桥接器的简称。比如 “两个GPU用 NvLink 连接”。

NvLink 桥接器只能点对点的连接两张 GPU，它只是硬件上使用 NvLink 链路将两张 GPU 连接起来

```
桥接：
GPU A ═══ NVLink 桥接器 ═══ GPU B
```

## ◼︎NvSwitch

NvLink 只能连接两张 GPU，两张 GPU 之间可以实现高速数据链路，但是多GPU之间无法实现互连。NvLink 桥接器只是一个物理连接部件，不具备交换机那样的多端口转发功能， 而且在物理上，它只能点对点的连接。

NvSwitch 是交换和转发 GPU 之间 NvLink 数据的交换芯片。它可以连接到多张 GPU。GPU 通过NvLink 链路连接到 NvSwitch, 数据经它转发到目标 GPU 上。所以，NvSwitch 在功能上可以认为是”GPU 之间的交换机”。它的逻辑是

```
交换式互连（简化示意）：
GPU A ── NVLink ──┐
GPU B ── NVLink ──┤
                 NVSwitch
GPU C ── NVLink ──┤
GPU D ── NVLink ──┘
```

> **总结： NvLink 是互连链路。NvLink 桥接器是实现 直接连接的硬件形态。NvSwitch 是实现交换互连的芯片。**
> 

# 4. SXM

SXM 是 GPU 连接到主板上的一种模块形态，SXM 通常比 PCIe 具有更高的数据传输率。GPU 内部的 NvLink 接口电路，直接连接到 SXM 接口模块中 SXM 连接器上，连接器通过底板走线，直接连接到 NvSwitch 芯片上，即 GPU 与 NvSwitch 芯片间通过 SXM 模块连接了 NvLink 链路。GPU 与 NvSwitch 之间传输 NvLink 信号，由 NvSwitch 转发到目标GPU上，实现了 GPU 间高速数据传输。

```
GPU A ── NVLink ──→ NVSwitch ── NVLink ──→ GPU B
```

它的拓扑结构。

```
交换式互连（简化示意）：
GPU A ── NVLink ──┐
GPU B ── NVLink ──┤
                 NVSwitch
GPU C ── NVLink ──┤
GPU D ── NVLink ──┘
```