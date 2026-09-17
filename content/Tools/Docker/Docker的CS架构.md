## <font color="grape">Docker的运行方式:  Client/Server</font>

Docker 的运行方式是采用 C/S 架构：
- Client: `docker`  CLI 命令
- Server: `dockerd` 守护进程(daemon). docker的守护进程 daemon，叫做 dockerd.
我们敲 `docker run` 命令的时候, CLI 把请求通过 Unix socket (`/var/run/docker.sock`，这是dockerd监听的Unix domain socket) 或 TCP发送给 dockered. dockerd 调用底层的 containerd -> runc 真正创建容器，然后把结果返回给 CLI。这套机制无论是哪种部署方式下都一样。

### <font color="#b48ff4">Docker Desktop</font>

我们平时说到的 Docker, 指的是 Docker Engine, 它是核心引擎，Docker Desktop 是包装了 Docker引擎的桌面应用。
Docker 是 C/S 架构，Server 是守护进程 `dockerd` , 它依赖于Linux内核的 namespaces, cgroups等。所以，它没法直接跑在 MacOS 和 Windows 上。
为了方便 MacOS 和 Windows 用户使用Docker, Docker 公司为 MacOS，Windows, Linux 环境做的一个桌面集成产品。
<font color="grass">它的核心机制是：在你的 Mac/Windows 上启动一个轻量级 Linux 虚拟机（macOS 上是基于 Apple Virtualization Framework 或 HyperKit，Windows 上是 WSL2 或 Hyper-V），然后在这个 VM 里跑真正的 Docker Engine。你在宿主机上敲的 `docker` 命令，通过 socket 转发到 VM 里执行。</font>
除了 VM + Engine，Docker Desktop 还打包了一堆周边东西：图形化 GUI、Kubernetes 集成(一键起一个单节点 K8s)、Docker Compose、镜像漏洞扫描、Dev Environments、Extensions 市场等。

> [!NOTE] 注意⚠️
> <font color="orange">Linux 用户其实不太需要 Docker Desktop，直接装 Docker Engine 更轻、更原生。Docker Desktop for Linux 也是跑 VM 的（KVM），主要为了和 Mac/Win 行为一致，但对 Linux 来说反而多了一层。</font>
> <font color="orange">如果要在 Mac 或 Windows 上本地跑容器化服务的测试，Docker Desktop 方便；如果是部署到生产服务器（基本都是 Linux），直接用 Docker Engine 更合适，也是行业标准做法。</font>

#####  - Docker Desktop 里的docker运行方式

Docker Desktop 不是把 Docker Engine 当成一个本地进程直接装在你的 macOS/Windows 上——而是把 Engine 装在它内部启动的那个 Linux VM 里。所以严格说是"**Docker Desktop 自带一个 Linux VM，VM 里跑着 Docker Engine**"。

宿主机上你看到的 `docker` 命令只是个 CLI 客户端，真正的 dockerd 守护进程在 VM 里。这一层对用户透明。当我们执行docker命令的时候，这个命令会被发送给 VM里面的 dockerd, 执行完之后，再将结果返回 CLI.

### <font color="#b48ff4">Docker Engine .VS. Docker Desktop</font>

**虚拟机 vs Daemon 不是对立的，dockerd 始终存在**
不管是 Docker Desktop 还是 Linux 上的 Docker Engine，**dockerd 这个守护进程都必须存在**——区别只在于 dockerd 跑在哪里：

|部署方式|dockerd 跑在哪|Client 怎么连|
|---|---|---|
|Linux 直接装 Docker Engine|宿主机 Linux 上原生进程|本地 `/var/run/docker.sock`|
|macOS/Windows 上的 Docker Desktop|Docker Desktop 启动的 Linux VM 里|宿主机 socket → VM 内 socket 转发|
|Linux 上的 Docker Desktop|Docker Desktop 启动的 Linux VM 里|同上|

所以 Docker Desktop 不是"用虚拟机替代了 daemon"，而是"**给 daemon 准备了一个 Linux VM 作为运行环境**"，因为 dockerd 本身只能跑在 Linux 上（依赖 Linux 内核的 namespaces、cgroups）。在 Mac/Windows 上没有 Linux 内核，所以必须先有个 VM。

### <font color="#b48ff4">启动 daemon</font>

dockerd 的 daemon 必须要正确启动起来，才能正常使用docker。dockerd 启动通常不需要用户手动操作。

**Linux 上**Docker Engine 装好后，dockerd 通过 systemd 管理。安装包时一般会自动 `systemctl enable docker`，开机自启。如果没启动，敲 `docker ps` 会报 `Cannot connect to the Docker daemon` 这种错，这时手动 `sudo systemctl start docker` 即可。

 **Docker Desktop**：你启动 Docker Desktop 这个应用程序，它会负责拉起 VM、在 VM 里启动 dockerd，整个过程对你透明。Docker Desktop 不退出，daemon 就一直在。

### <font color="#b48ff4"> Docker 命令执行的转发机制</font>

dockerd 启动以后，我们就可以执行 docker 命令了。dockerd 会监听一个 Unix domain socket, `/var/run/docker.sock`. docker 命令执行的时候，CLI默认连接到这个socket上，通过它发送HTTP请求，<font color="red">注意⚠️: Docker的API本质是 REST over HTTP </font>
dockerd 监听到请求时，就处理这个请求然后返回结果。

Docker Desktop 在宿主机和VM之间架了一座桥。现在系统里，没有 dockerd 的 daemon 在监听这个socket了。Docker Desktop 采用的是一个辅助进程在监听这个 socket，一旦监听到有命令发送，它就把这个命令转发到 VM里面的 dockerd 进行处理，然后再返回结果。