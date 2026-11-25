启动虚拟机后，假设我们使用 RockyLinux。RockyLinux 9 以后的版本，全部采用 NetworkManager 代替老的 Network 去实现网络的管理。所以，登陆系统后设置网络，需要修改的配置文件是

```bash
cat /etc/NetworkManager/system-connections/ens160.nmconnection
```

有几张网卡就有几个 `ensxxx.nmconnetion`配置文件，数字最小的代表第一张网卡，以此类推。比如下面就有两张显卡， ens160 和 ens192

![](Pasted%20image%2020261124120942.png)

打开 ens160 显卡，这里可以把当前 IPV4 的工作方案从自动改为手动。然后再添加 `address1=192.168.66.11/24` 这里的 /24 表示 255.255.255.0 的子网掩码。

![image|360](Pasted%20image%2020261124121212.png)![image|340](Pasted%20image%2020261124121748.png)

如果有网关，可以加上 `address1=192.168.66.11, gw=xxx`, 当然还可以添加 dns 选项，指定 dns 服务器，只需要在 address1 下面，加上 `dns=xxx.xxx.xxx.xxx; yyy.yyy.yyy.yyy`, 多个 dns 服务器之间用分号分隔。

>注意⚠️：这里是把 IPv4 的工作方式，改为了手动 manual, 并且把 IP 改为了一个固定的 IP，192.168.66.11。意思就是这个网卡连接到的网络中，这个网卡就是固定 IP 192.168.66.11。

修改完后，执行 `systemctl restart NetworkManager` ，这样就可以通过终端连接服务器了。

# 1. vmnet1 和 vmnet8 虚拟网络(先不讨论 vmnet0)

VMWare 软件启动的时候，系统会默认初始化两个网络，即创建两个默认虚拟网络，`vmnet1` 和 `vmnet8`。宿主机默认会加入这两个网络。

`vmnet1` 和 `vmnet8` 是 VMWare 启动时默认会被占用的名字，中间的编号 2～7 可以用来进行创建自定义的虚拟网络，8 之后的编号也可以用，vmware 17 可以同时最多创建 20 个虚拟网络，即 `vmnet[0-19]`。

这里的 vmnet1 和 vmnet8 可以理解为虚拟交换机，即宿主机和虚拟机可以连接到 vmnet1 交换机上加入 vmnet1 网络。同样，通过另一张网卡，连接到 vmnet8 交换机上加入 vmnet8 。加入网络一定通过 Network Adapter (网络适配器，即网卡) 接入。

创建虚拟网络以后，并不一定要求宿主机要加入网络。但宿主机默认会加入，vmware 就是在宿主机上默认生成两张虚拟网卡，分别加入 vmnet1 和 vmnet8。

> 所以，在 vmware 启动的时候，默认会初始化两个虚拟网络，vmnet1 和 vmnet8 (可以看成交换机名字), 然后还要宿主机上虚拟出两张显卡。分别连接到 vmnet1 网络 和 vmnet8 网络上。

![](Pasted%20image%2020261124132903.png)

在 Virtual Network Editor 上可以查看网络 vmnet1 和 vmnet8. 下方的勾选项 Connect a host virtual adapter to this network, 默认情况勾选，就是让宿主机产生一个虚拟网卡(virtual adapter)，然后连接到本网络。


> [!attention] 网段
> - 下面的 subnet IP 是子网网段的地址，这是默认分配的，通常要修改，如果宿主机和虚拟机不在同一个网段下，即使在一个虚拟网络中也无法通信。这里我们将宿主机置于 192.168.66.0 网段，由子网掩码 255.255.255.0 控制。 
> - 上面虚拟机中，设置的是固定 IP 是 192.168.66.11/24，所以它的网段是 192.168.66.0。如果没有把vmnet1的网段改为 192.168.66.0(一开始默认分配的是 192.168.35.0，是后面改成 192.168.66.0的)，那么不在同一个网段下的机器是无法通信的。
> <font color="orange">所以，即使两个虚拟机加入了同一网络，如果 IP 配置不在同一网段下，也是无法通信的。</font>
> - subnet IP 设置的是网段 IP，而 connect  a host virtual adapter to this network 以后，宿主机网卡在网络中的具体 IP 是通过 use local DHCP service to distribute IP address to VMs, 即通过 DHCP 进行自动分配。 

# 2. 网络接入方式 Host-Only, NAT, Bridge

<font color="orange">Host-Only </font>网络,意思是网络中的主机私有这个网络，即这是一个私有网络不对外连接。只有网络中的各个主机间才可以相互连接通信，所以叫做 Host-Only。
虚拟机配置网卡时，选择 Host-Only 的连接方式会让这张网卡加入默认网络 vmnet0。vmnet0 是 vmware 默认初始化启动的 Host-Only 网络。

<font color="orange">NAT, Network Address Transition, 网络地址转换</font>。意思是虚拟机只能通过宿主机向外发起连接，而外部无法直接连接虚拟机，虚拟机的内部 IP 对外是隐藏的。意思就是虚拟机对外，是共享宿主机的 IP 的。
虚拟机配置网卡时，选择 NAT 接入方式就会让这张网卡加入默认网络 vmnet8。vmnet8 是 vmware 默认初始化启动的 NAT 网络。

<font color="orange">Bridge 桥接</font>，就是让虚拟机通过宿主机的物理网卡，直接加入宿主机所在的局域网。虚拟机共享的是宿主机的物理网络连接。这时宿主机 windows 和 虚拟机 Rocky 各有自己的网卡和 IP。
windows 已经通过自己的 wi-fi 拿到一个IP，发起第二次 DHCP 请求的是 Rocky 虚拟机。所以这种模式下，宿主机 Windows 和 Rocky 各有自己独立的一套网络进程软件栈，包括网络接口，网络协议栈，DHCP 客户端。它们相互独立，但是它们共享复用宿主机上物理 wi-fi 连接。桥接模式下，在家庭局域网的 IP 通信角度看，Rocky虚拟机就是一台有自己独立 IP 的主机。强调⚠️，Rocky 和宿主机只是共享网络通信的物理信道，但是各自运行独立的网络软件栈，再逻辑层面它们是独立的家庭局域网的节点。
虚拟机配置网卡时，选择 Bridge 接入方式，就是虚拟机与宿主机共享物理网络连接。在通过物理连接之前，所有的桥接方式的虚拟机先连接到 vmnet0 网路，由 vmnet0 复用宿主机的物理网络接口和家庭局域网进行通信，所以宿主机的物理网络接口，可以看做 vmnet0 的上游网络。

![](Pasted%20image%2020261124171714.png)

正常情况下，前一张图里，我们是看不到 vmnet0 的，只有在点击右下角的 DHCP Settings 以后才能看到，因为它的接入是 Auto-Bridging 的方式，也就多个虚拟机连接到 vmnet0 上，由它集结多个虚拟机桥接到上游，复用宿主机的物理网络，严格意义上它并不是一个虚拟网络，只是一个hub的桥接方式，它的 Host Connection, DHCP, Subnet Address 都没有，都依赖家庭局域网。

> [!tip] 宿主机不通过 vmnet0
> 宿主机本身操作系统，如 windows 是通过自己的操作系统网络栈，直接连接物理网络对外通信，不会加入 vment0。所以，vmnet0是专属于，多个虚拟机公用的桥接网络。只有虚拟机才通过vmnet0 与家庭局域网其他节点通信。

![423](Pasted%20image%2020261124173233.png)

所以在宿主机上我们查看 ipconfig, 是看不到 vmnet0 的。

# 3. 默认初始化网路 vmnet0, vmnet1 和 vmnet8 

vmware 软件在启动的时候，就默认初始化三个网络vmnet0, vmnet1 和 vmnet8。vmnet0 是桥接网络，vmnet1 的 Host-Only 私有网络, vmnet8 的 NAT 的地址转换，<font color="orange">这是默认的</font>。

实际上，系统为我们是默认创建了两种接入方式的网路。当我们配置多台虚拟机的多张虚拟网卡时，只要选择的接入 Host-Only，这张网卡就会自动加入 vmnet1 网络。如果选择的接入是 NAT, 这张网卡就会自动接入 vmnet8。vmnet1 和 vmnet8 实际上两种不同的接入方式。

如果我们自定义网络，就要在 `Virtual Network Editor --> Add Network` , 新增网络的时候可以选择接入方式，Host-Only 或者 NAT。然后我们在配置虚拟机网卡的时候，直接设置网卡为 Custom: Specific virtual network，比如时 vmnet2。此时，vmnet2 是什么样的网络，这个网卡就以什么方式接入，就不在网卡设置端指定接入方式了。

我们直接指定接入方式，<font color="orange">实际上是按照接入方式，对应接入到不同的默认网络中，不同的接入方式就可以一一对应到不同的默认网络中，所以实际是以接入方式，选择不同的默认网络</font>。而自定义网络，直接根据指定网络，确定接入方式接入就可以了。

vmnet0 严格意义上不算一个虚拟网络，更像一个hub，把选择桥接方式的虚拟机集结起来，
把宿主机物理网络接口当作上游网关，复用物理连接。所以它更多是一个桥接的 hub。

# 4. 总结

vmware 启动的时候，会默认启动三个网络 vmnet0, vmnet1, vmnet8, 三个网络分别对应不同接入方式的网络，vmnet0 是桥接网络，vmnet1 是 Host-Only 网络，vmnet8 是 NAT 网络。

创建虚拟机时，设置网卡的接入方式是 Bridge，就默认会加入 vmnet0。选择接入方式 Host-Only, 就默认加入 vmnet1。选择接入方式 NAT，就默认加入 vmnet8。选择网卡接入方式就是在选择加入的默认网络。

我们可以自定义网络，名字除了 vmnet0, vmnet1, vmnet8 之外都可以使用，当前vmware 版本下，虚拟网络数最大为20个。所以，就有了「定制化指定接入虚拟网络」的方式。自定义网络可以选择任意的接入方式(Bridge, Host-Only, NAT)。虚拟机在加入自定义网络时，直接配置网卡加入就行，不用选择接入方式，接入方式只能跟着自定义网络的方式来。

自定义网络中，不必要一定有宿主机。只有在自定义网络的时候勾选了 connect a host virtual adapter to this network, 才会在创建网络的时候，在宿主机上虚拟出一个网卡接入这个网络。

所以默认情况下，vmware 会虚拟出三个网络 vmnet0, vmnet1, vmnet8 。它们可以看作是交换机，通过连接 vmnet0, vmnet1,vmnet8 交换机就可以加入相应网络。宿主机默认不会加入 vmnet0, 但是会加入vmnet1 和 vmnet8，它是通过在宿主机上虚拟出网卡来加入的。

<font color="orange">网络中接入网络的节点，虽然是主机，但是本质上是网卡或网络适配器 Network Adapter。网络对网络中节点的感知，只有网卡</font>. 
默认情况下，vmware 会虚拟出交换机(网络) vmnet1 和 vmnet8, 并且在宿主机上虚拟出网卡，让宿主机加入相应网络，但宿主机不会加入 vmnet0。因为宿主机是用自己操作系统的物理机网络软件栈直接和局域网交换数据，所以宿主机上的 ipconfig 里看不到 VMware Network Adapter vmnet0 网卡，但是有 VMware Network Adapter vmnet1 和 VMware Network Adapter vmnet8 这两个网卡。这就是它加入这两个网络虚拟出的网卡。