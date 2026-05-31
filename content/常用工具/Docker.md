通常，应用程序的部署，环境配置的过程纷繁复杂。在开发环境好用，但是到了测试和生产环境都不好用情况常常发生。新的成员加入项目组，需要花费大量时间来配置开发环境，经常需要花费一天的时间，一步一步按照配置部署文档来配置环境，但是经常就卡在中间某个步骤上，再也过不去了。Docker就可以帮助我们完美解决这些问题。

## <font color="grape">Docker简介</font>

Docker是一个用于<font color="orange">构建(build)</font>，<font color="orange">运行(run)</font>，<font color="orange">传送(share)应用程序的平台</font>。可以将我们的应用程序打包成一个个集装箱，然后它就会帮我们将其运送到任何需要的地方。<font color="grass">有了Docker，我们就可以将应用程序，和它运行时需要的各种依赖包第三方软件库，配置文件等打包在一起。以便在任何环境中都可以正确的运行</font>。如下所示：

![[Pasted image 20260527112455.png\|600]]
我们将这些环境全部打包，放在一个集装箱内，运送到任何地方去运行。只要我们在开发环境中是运行成功了，那么我们可以在任何环境下运行成功。

### <font color="#b48ff4">虚拟机(Virtual Machine)</font>

我们使用过<font color="deeppink">vmware, virtual box, Parallels Desktop</font>等虚拟机软件以及<font color="deeppink">windows wsl </font>和 <font color="deeppink">Hyper-V </font>功能。我们可以在Windows上使用WSL安装和使用Linux系统，也可以在Mac中，使用虚拟机运行windows和各种Linux系统。它们是完整的操作系统，和实际的 windows 和 Linux系统一样。可以在这个操作系统中运行应用程序。这是用一种叫做<font color="deeppink">虚拟化(Virtualization)</font>的技术来实现的。

虚拟化是一种将物理资源，虚拟化为多个逻辑资源的技术。它可以将一台物理服务器虚拟为多台逻辑服务器，每个逻辑服务器都有自己的操作系统，CPU，内存，硬盘，网络接口等。它们之间是完全隔离的，可以独立运行。它在一定程度上实现了资源整合，可以将一台物理服务器的计算资源，存储能力，网络资源，分配给多个逻辑服务器，实现多台服务的功能。虚拟机运行模式如下：

![[Pasted image 20260527114552.png\|500]]

缺点是每台服务器都需要占用大量资源，比如CPU，内存，硬盘，网络等等。而且启动速度极慢，但是大多情况下，我们一台服务器上只需要运行一个对外提供服务的应用程序就可以了，并不需要一个操作系统所提供的所有功能，比如我们可能只需要启动一个web服务器，但是虚拟机要启动一个完整的操作系统，包括操作系统内核和各种系统服务，各种工具，甚至图形界面等。这些我们不需要的功能会导致大量资源浪费和启动速度慢等问题。

### <font color="#b48ff4">容器(Container)</font>

Docker概念上和虚拟机很类似，但是轻量很多。Docker不会模拟底层的硬件，只会为每一个应用提供完全隔离的运行环境。<font color="orange">可以在环境中配置不同的工具软件，并且不同环境互不影响。这个环境在Docker中被称为</font><font color="grass"> 容器(container).</font>
<font color="orange">Docker</font>和<font color="orange">容器(Container)</font>是两个不同的概念。很多人把Docker和容器混为一谈。其实Docker只是容器的一种实现。
<font color="grass">Docker是一个容器化的解决方案和平台，而容器是一种虚拟化技术。容器和虚拟机一样也是一个独立的环境，我们可以在这个环境中运行程序</font>。容器的运行模式如下：

![[Pasted image 20260527115533.png\|500]]


> [!NOTE] 容器与虚拟机的不同
> <font color="orange">与虚拟机不同的是，它并不需要在容器中运行一个完整的操作系统，而是直接使用宿主机的操作系统。所以启动速度非常快，通常只需要几秒钟，因为需要资源少，所以可以在一台物理服务器上运行更多的容器，可以更加充分利用服务器资源。比如一台物理服务器上可以运行几台虚拟机，却可以运行上百个容器。</font>
> 


### <font color="#b48ff4">镜像(image)</font>

<font color="grass">镜像,image</font>是一个虚拟机的快照(snapshot), 里面包含了要部署的应用程序，以及它所关联的所有库和软件。通过镜像我们可以创建许多不同的container，容器。这里容器就像是一台台运行起来的虚拟机。里面运行了应用程序，每一个容器是独立运行，相互之间不影响。

![[Pasted image 20260527124640.png]]

镜像是一个只读的模板，它可以用来创建容器，容器是Docker的运行实例，它提供了一个独立可移植环境，可以在这个环境中运行应用程序。<font color="deeppink">镜像是一个只读模板，而容器是一个运行实例。</font>

镜像与容器就如同类与对象，或者食谱与做出来的菜肴。“食谱”也可以分享给别人。
### <font color="#b48ff4">Dockerfile</font>

<font color="orange">容器化(containerization)，就是将应用程序打包成镜像image，然后用镜像生成容器，并在容器中运行应用程序的过程</font>。


> [!NOTE] 容器化的三个步骤
> - <font color="grass">① 首先要创建一个Dockerfile, 来告诉 Docker 构建应用程序镜像所需要的步骤和配置。</font>
> - <font color="grass">② 然后使用 Dockerfile 来构建镜像。</font>
> - <font color="grass">③ 使用这个镜像来创建和运行容器。</font>

<font color="deeppink">Dockerfile 是一个自动化脚本，主要用来创建镜像，简单说就是自动化打包环境生成镜像。</font>

Dockerfile 是一个文本文件，里面包含了一条条指令，用来告诉 Docker 如何构建镜像。这个镜像中包含了应用程序执行的所有命令，也就是各种依赖，配置环境，和运行应用程序所需的所有内容。
一般包含如下内容，

>- 精简版的操作系统，比如Alpine。
>- 应用程序的运行时环境，比如java, python等。
>- 应用程序，比如SpringBoot 打包好的jar包。
>- 应用程序第三方依赖库或包。应用程序的配置文件环境变量等等。

<font color="orange">一般来说，我们会在项目的根目录下，创建一个叫 Dockerfile 的文件，在这个文件中写入构建镜像所需要的各种指令之后，Docker 就会根据这个 Dockerfile 文件来构建一个镜像。有了镜像之后，我们就可以创建容器，然后在容器中运行应用程序。</font>

一个典型的Dockerfile大概长这样

```dockerfile
FROM python:3.11-slim          # ① 基础镜像
WORKDIR /app                   # ② 工作目录
COPY requirements.txt .        # ③ 把 build context 里的文件复制进镜像
RUN pip install -r requirements.txt  # ④ 在镜像里执行命令，产物留下
COPY . .                       # ⑤ 把项目代码复制进去
ENV PYTHONUNBUFFERED=1         # ⑥ 设置环境变量
EXPOSE 8000                    # ⑦ 元数据：声明端口
CMD ["python", "main.py"]      # ⑧ 启动命令
```

**最终镜像里有什么：**

- **基础镜像的全部内容**（FROM 那一层）——比如 `python:3.11-slim` 自带的 Debian 用户空间 + Python 解释器 + pip + 必要的库。这通常是镜像体积的大头。
- **COPY/ADD 进来的文件**——只有显式 COPY 的才会进，build context 里其他文件不会自动进去。
- **RUN 指令的产物**——比如 `pip install` 装的包、`apt-get install` 装的软件，都会沉淀到镜像里。
- **元数据**——ENV、EXPOSE、CMD、ENTRYPOINT、WORKDIR、LABEL 这些不是文件，但作为镜像配置存在。
注意⚠️，区分④ 和 ⑧， RUN 是build时候，build完了之后，要执行的命令，命令的效果会沉淀到镜像中。CMD是在使用镜像启动容器时，执行的命令，通常为启动要跑的服务或者应用。
### <font color="b48ff4">Docker仓库</font>

镜像分享给其他人，要通过 Docker仓库(registry)。Dokcer仓库是用来存放Docker镜像的地方。最流行和最常用的仓库就是 DockerHub。它是一个公共的Docker仓库，用来集中存储和管理Docker镜像。我们可以在这里下载各种镜像，也可以将自己的镜像上传到这里，这样就可以实现镜像的共享和复用。

### <font color="#b48ff4">Docker Compose</font>

实际使用过程中，我们的应用程序可能用到多个容器协作，比如一个容器来运行 web 应用，另一个容器，另一个容器来运行数据库系统。这样，可以做到数据和应用逻辑的分离。比如web容器down掉了，但是数据库服务器还在运转。此时，我们只需要修复 web 服务器就可以了。docker compose就可以做到这点。

Docker Compose 是一个 Docker 官方的开源项目，是一个用来定义和运行多容器Docker应用程序的工具。比如，搭建一个网站，可能会使用到前端，后端，数据库，甚至缓存，负载均衡等多个服务器。这些服务是独立的，但是他们之间又是相互关联的。需要相互配合来工作，前端需要连接后端，后端需要连接数据库。这些服务之间的关联关系，就是 Docker Compose 需要解决的问题。它通过一个单独的 `docker-compose.yaml` 配置文件来将这一组相互关联的容器组合在一起，形成一个项目。然后使用一条命令，就可以启动，停止或者重建这些服务， 这样就可以非常方便管理这些服务了。比如，有一个新同事，之前可能需要半天时间去安装各种依赖和配置运行环境，现在有了Docker Compose 之后，只需要执行一下 Docker compose up 命令，就可以自动安装各种依赖和配置运行环境，然后在本地运行项目了，这样大大提高了开发效率。

我们创建 `docker-compose.yml` 文件。
```yml
version: "3"

services:
	web:
		build: .
		ports:
			-80:5000
	db:
		image: "mysql"
		environment:
			MYSQL_DATABASE: finance-db
			MYSQL_ROOT_PASSWORD: secret
		volumes:
			- my-volume-data:/var/lib/mysql
```

文件中我们使用services定义多个container, 比如这里定义了一个web容器里面运行 web 应用，然后再定义一个db容器，里面定义了 mysql 数据库系统, 这里通过两个环境变量，指定数据库的名字和连接密码, 通过 volumes 指定一个数据卷永久存放数据。
定义完毕后，我们保存文件，使用 `docker compose up -d` 来运行所有容器，-d代表后台运行所有容器。对应的，我们可以使用 `docker compose down` 来停止并删除所有容器。不过新创建的数据卷需要手动删除，除非加入 --volumes参数。

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

**Linux 上**Docker Engine 装好后，dockerd 通过 systemd 管理。装包时一般会自动 `systemctl enable docker`，开机自启。如果没启动，敲 `docker ps` 会报 `Cannot connect to the Docker daemon` 这种错，这时手动 `sudo systemctl start docker` 即可。

 **Docker Desktop**：你启动 Docker Desktop 这个应用程序，它会负责拉起 VM、在 VM 里启动 dockerd，整个过程对你透明。Docker Desktop 不退出，daemon 就一直在。

### <font color="#b48ff4"> Docker 命令执行的转发机制</font>

dockerd 启动以后，我们就可以执行 docker 命令了。dockerd 会监听一个 Unix domain socket, `/var/run/docker.sock`. docker 命令执行的时候，CLI默认连接到这个socket上，通过它发送HTTP请求，<font color="red">注意⚠️: Docker的API本质是 REST over HTTP </font>
dockerd 监听到请求时，就处理这个请求然后返回结果。

Docker Desktop 在宿主机和VM之间架了一座桥。现在系统里，没有 dockerd 的 daemon 在监听这个socket了。Docker Desktop 采用的是一个辅助进程在监听这个 socket，一旦监听到有命令发送，它就把这个命令转发到 VM里面的 dockerd 进行处理，然后再返回结果。

## <font color="grape">使用Docker</font>

按照容器化的三个步骤：首先我们编写 Dockerfile。

### <font color="#b48ff4">Step1: 创建 Dockerfile</font>

一般我们会在项目根目录下，创建一个Dockerfile文件
```dockerfile
FROM python: 3.8-slim-buster
WORKDIR /app
COPY . .
RUN pip3 install -r requrirements.txt
```

我们在第一行用FROM命令指定一个基础镜像(base image)。这可以帮我们节省很多软件安装时间。

在 Docker Hub 上有很多高质量的操作系统镜像，比如 ubuntu, debian, fedora, alpine等。不同的操作系统提供不同的包管理工具，比如 unbuntu上的apt，fedora上的dnf。DockerHub 还有方便某一种语言，某一种框架开发的镜像，比如 nginx，redis, node, python, tomcat等。

上面我们就使用DockerHub上的python镜像，这样免去了python的安装步骤。这里的 `python` 是官方镜像的名字，冒号后的 `3.8-slim-buster` 是版本号，也是标签 Tag。我们可以到DockerHub上，找到python镜像的页面，里面可以找到所有支持的标签，这里就是用的3.8版本。运行在debian buster的发行版本上。

`WORKDIR /app`：指定了这个命令之后所有的 Docker 命令的工作路径(working directory)。如果这个路径不存在，Docker会自动创建。这样可以避免使用绝对路径或手动cd切换路径。

`COPY . .` : 调用 COPY 命令将所有程序拷贝到 Docker 镜像中， 命令为 `COPY<本地路径><目标路径>`，第一个 `.` 代表根目录下的所有文件，也就是项目根目录下，即整个项目。第二个参数代表Docker镜像中的路径。这里的 `.` 代表当前工作路径，也就是之前指定的 `/app` 目录。

`RUN` 运行我们在创建镜像时，运行任意的 shell 命令。比如这里我们使用了 pip install 来安装 python程序的所有关联。

通过以上命令，我们就可以完成一个docker镜像的创建。

`CMD` ：在 Dockerfile 最后，我们会用 CMD 来指定当Docker 容器运行起来以后，要执行的命令。

注意区分 `RUN`和 `CMD`，<font color="orange">RUN是创建镜像时使用的，而CMD是运行容器时使用的</font>。

### <font color="#b48ff4">Step2: 创建镜像image</font>

我们可以使用 docker build 来创建一个镜像。我们在项目根目录下，运行`docker build -t my-proj .` ,  -t 表示 tag/标签，指定了创建镜像的名字，最后面的 `.` 告诉 docker 应该在当前目录下寻找 Dockerfile 文件。

第一次调用 `docker build` 会比较慢，因为会下载必要的镜像文件。再次调用就会快很多，因为它会缓存操作。

### <font color="#b48ff4">Step3: 启动容器 Container</font>

创建好了镜像之后，我们就可以使用 `docker run` 来启动一个容器。执行
```shell
docker run -p 80:5000 -d my-proj
```

`-p` 参数， 他会将容器上的一个端口映射到本地主机上，这样我们才能从主机上访问容器中的Web服务。前面的 80 是我们本地主机端口，后面的5000是容器上的端口。

`-d` 参数，--detached, 让容器在后台运行，这样容器的输出就不会直接显示在控制台。

`my-proj`: 表示要启动容器，使用的镜像名称。

我们通过Docker Desktop 图形界面，可以查看应用在后台的所有输出。这对调试很方便，同时也可以看到当前容器的各种信息状态等。在图形界面中可以直接进行一些操作，它们对应的命令行工具如下：
![[Pasted image 20260528072233.png\|500]]

当我们删除一个容器的时候，之前所做的修改，新添加的数据会全部消失。如果希望保留容器中的数据，可以使用 Docker 提供的 volume 数据卷功能。可以把它当作是在本地主机和不同容器中共享的文件夹。比如下面容器1中修改了数据，可能反应到容器3中。
![[Pasted image 20260528072632.png\|500]]

我们可以通过 docker volume create 来创建一个数据卷， `docker volume create my-vlume-data`. 在启动容器的时候
```shell
docker run -dp 80:5000 -v my-volume-data:/etc/finance mt-volume-data 
```
通过 `-v`参数指定，将这个数据卷挂载(mount)到容器中哪个路径上。这里将 my-volume-data 挂载到了 /etc/mount这个路径下，向这个路径写入的任何数据，都将被永久保存在这个数据卷中。

## <font color="grape">Docker 管理镜像文件的方式</font>

`docker build`  产生的镜像文件，dockerd 会把它放在本地的一个目录里，Linux默认是 `/var/lib/docker/` , 这个目录通常叫做<font color="orange"> local image store 或 本地镜像缓存</font>, 远程镜像仓库(Docker Hub, 或者配置的私有远程仓库)叫做 registry。

`docker run` 的查找逻辑：
1. 先查本地 image store，命中就直接用
2. 没命中，按镜像名解析 registry 地址（不带前缀的默认走 Docker Hub）
3. 从 registry 拉取（`docker pull`），存到本地 image store，再启动容器

<font color="orange">要把自己本地生成的镜像放到远程服务器上执行，我们通常是使用 registry 中转，这也是生产的标准做法。</font>

最规范、最常用的方式。流程：

```bash
# 本地：给镜像打 tag，指向目标 registry
docker tag myapp:latest registry.example.com/myteam/myapp:v1.0

# 本地：推到 registry
docker login registry.example.com
docker push registry.example.com/myteam/myapp:v1.0

# 远程服务器：拉下来运行
docker login registry.example.com
docker pull registry.example.com/myteam/myapp:v1.0
docker run -d registry.example.com/myteam/myapp:v1.0
```

Registry 的选择：

- **Docker Hub**：免费但公开（私有 repo 数量有限制）
- **阿里云 ACR、腾讯云 TCR、华为云 SWR**：国内速度快，国内云厂商基本都提供
- **GitHub Container Registry (ghcr.io)**：和 GitHub 仓库绑定，方便
- **私有自建**：用官方 `registry:2` 镜像跑一个，或者 Harbor（更完整的企业级方案）

## <font color="grape">Docker原理分析</font>

docker的核心是 dockerd 守护进程，它通过C/S的方式，让 dockerd 完成各种操作。
docker 本身打包的<font color="red">不是本地运行环境</font>，而是根据Dockerfile的指定，构建出一个运行环境。这个打包的运行环境里，主要包含<font color="orange">用户空间的文件系统 + 应用程序执行环境 + 应用程序及其依赖的包和软件。</font> 具体的，镜像image中包括：

>- **用户空间的文件系统**：`/bin`、`/lib`、`/usr`、`/etc` 等目录里的二进制和库
>- 应用程序及其依赖（动态链接库、Python 包、JAR 等）
>- 配置文件、环境变量、启动命令（ENTRYPOINT/CMD）
>- 元数据（暴露的端口、挂载点、labels）

<font color="red">注意⚠️，不包括内核和硬件驱动。</font>
所以你看 `ubuntu:22.04` 镜像大概 70MB，`alpine` 才 7MB——它们只是 Ubuntu/Alpine 的**用户空间发行版部分**（GNU 工具、glibc/musl、包管理器等），不是一个完整的 Ubuntu 系统。真正的"Ubuntu 系统"= 内核 + 用户空间，镜像里只有后半部分。

<font color="orange">容器不是 “虚拟机进程”，容器就是一个隔离的普通进程。容器**不是**一个里面跑着虚拟 OS 的进程，而是**宿主机上一个普通的 Linux 进程**——它和你 shell 里 `ps aux` 看到的其他进程没有本质区别。</font>

<font color="grass">容器能跨机器运行，是因为镜像把应用所需的整个用户空间环境都打包了，不依赖宿主机上装了什么库、什么版本。只要目标机器有 Linux 内核 + 容器运行时（containerd/runc 等），就能用这套打包好的 rootfs 启动进程。相当于，容器用自己的用户空间的rootfs文件系统, 应用程序跑在容器内的rootfs上，使用的是rootfs中的依赖包和软件，使用的是用户rootfs空间中环境变量和，配置文件等。但是，它是一个进程，使用的是宿主机的内核，被宿主机当作一个普通的进程调度。</font>

这正是 Docker 解决的核心问题——"在我机器上能跑"。不是因为它是独立进程（普通进程也是独立的），而是因为它**自带了完整的依赖环境**。<font color="deeppink">应用程序 + 依赖包和软件 + 用户空间文件系统，它们一起被打包成了一个进程运行，所以，应用程序的运行就不依赖操作系统环境了，只依赖它内部环境。</font>

#### <font color="#b48ff4">容器中多服务 和 容器之间</font>

容器中如果有两个服务A和B，那么它们会被启动成两个进程，这两个进程共享容器的运行环境，用户空间文件系统等容器内的资源。启动多个容器，同样是启动多个进程，但是他们就是完全独立的进程，互相之间不会任何影响。


#### <font color="#b48ff4">Docker不是打包本地运行环境</font>

<font color="deeppink">关键点：镜像里的环境完全来自 Dockerfile（主要是 FROM 那个基础镜像 + 后续 RUN/COPY 指令），跟你本地宿主机的环境没有任何继承关系。</font>
你本地装的 Python 3.11、那个特定版本的 CUDA、`~/.bashrc` 的配置、apt 装的某个库——**这些一概不会自动出现在镜像里**。dockerd 只看 Dockerfile，按它的指令从零搭起一个新环境。

<font color="red">这正是 Docker 解决"在我机器上能跑"的核心机制.</font> 假设我使用Mac，在我机器上能跑，如果打包的是本地运行环境，那么镜像在Ubuntu下运行就会崩溃。所以，Docker不是在打包本地运行环境。
如果使用Docker，只有在Dockerfile里明确写的环境才生效：比如

```dockerfile
FROM python:3.10-slim         # 锁定 Python 3.10
RUN apt install -y libfoo2    # 锁定 libfoo
COPY . /app
CMD ["python", "/app/main.py"]
```

这个镜像不管在你 Mac、同事 Ubuntu、还是云服务器上跑，里面都是同一套 Python 3.10 + libfoo2，结果完全一致。**宿主机装的 Python 3.6 也好、3.11 也好，全都被屏蔽掉了**——容器进程只看到镜像 rootfs 里的 Python 3.10。

>- ① **Dockerfile 是"环境的源代码"** ——它声明性地描述了运行环境的每一寸。这也是为什么 Dockerfile 必须 check in 到 git 里、必须可重现 build。
>- ② **本地能跑不代表镜像里能跑**——经常遇到的坑：本地 `python main.py` 没问题，因为系统里早就装了某个库；镜像里没装这个库，build 出来跑不起来。本质就是镜像和本地是两个独立环境。
>- ③ **基础镜像决定底色**——`FROM ubuntu:22.04` 给你一个 Ubuntu 用户空间，`FROM alpine` 给你一个体积超小但用 musl libc（不是 glibc）的环境，`FROM scratch` 给你一个**完全空的**镜像（什么都没有，连 shell 都没有）。选 FROM 等于选了底层基底。
>- ④ **"COPY . ."** 是个例外，它确实把项目代码从宿主机搬进镜像——但搬的也只是**文件**本身，不是你本地的运行环境。本地 `pip install` 装的包不会跟着代码一起进镜像，必须在 Dockerfile 里再 `pip install -r requirements.txt` 一遍。

镜像 = **基础镜像提供的环境** + **Dockerfile 后续指令构建的增量**（装的依赖、复制进去的文件、设置的环境变量）。和你本地的运行环境唯一的交集，就是你通过 `COPY` 显式搬进去的那些文件。除此之外，本地和镜像是两个完全独立的世界。

Docker 不是"自动打包本地环境"，而是"**让你写一份配方，按配方生成一份与宿主机解耦的环境**"。"打包本地环境"是从用户体验角度的简化表述——你的目标确实是复刻本地环境，但你得自己把它翻译成 Dockerfile，dockerd 只认 Dockerfile，不认你本地装了什么。

<font color="deeppink">这种"环境与宿主完全脱钩"的特性，就是 Docker 提供可重现性和可移植性的根基。</font>

#### <font color="#b48ff4">Docker 与宿主机内核</font>

Docker打包的用户空间文件系统 + 应用程序 + 运行环境。如果应用程序要操作内核，那就会报错。因为Docker使用的是宿主机的内核，如果操作内核通过文件系统想要操作内核，
比如 :

>1. sysctl 参数(`/proc/sys/...`)—— 网络栈调优, 内存管理, 文件句柄数等,比如`net.ipv4.tcp_keepalive_time`  、`vm.swappiness`、`fs.file-max`
>2. **内核模块**（`insmod`、`modprobe`、`rmmod`）
>3. **`/proc`、`/sys` 下的设备和子系统配置**
>4. **cgroup、namespace 之外的全局内核状态**

容器默认对这些的访问权限是**严格受限**的。这些文件只具有只读操作。对他们写会报 read-only 错误。如果是测试脚本的容器，看到这里内核报权限错误，这就可以了，无需其他操作。

如果确实要动内核，Dokcer就不是一个适合的工具了。
## <font color="grape">Docker 与 Kubernetes 的关系</font>

虽然大家都在说，kubernetes在逐渐取代docker，但是指的是kubernetes中的容器引擎(container engine )而已。实际上 kubernetes 和 docker 不是一个层面的东西。之前的应用，web容器，数据库容器等，都运行在同一个计算机中，随着应用规模的增大，一台计算机没法满足我们的需求。当我们使用一个集群的计算机提供服务，并做到负载均衡，故障转移等。这时候就可以用kubernetes

![[Pasted image 20260528075938.png]]

简单说，kubernetes 所做的就是将我们的各个容器分发到一个集群(cluster)上运行。并且进行全自动化管理，包括应用部署和升级。
![[Pasted image 20260528080130.png]]

