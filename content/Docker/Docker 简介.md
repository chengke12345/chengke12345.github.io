 应用程序部署，环境配置过程纷繁复杂。开发环境中好用，但到了测试和生产环境都不好用情况常常发生。新的成员加入项目组，需要花费大量时间来配置开发环境，经常需要花费一天的时间，一步一步按照配置部署文档来配置环境，但是经常就卡在中间某个步骤上，再也过不去了。Docker就可以帮助我们完美解决这些问题。

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

<font color="orange">容器化(containerization)，就是将应用程序及其运行环境打包成镜像image，然后用镜像生成容器，并在容器中运行应用程序的过程</font>。


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
- WORKDIR /app: 将工作目录设为 /app，这是镜像中操作系统系统下的目录。如果镜像中的操作系统没有这个目录，就创建它，并设置它为当前工作目录。
- **COPY/ADD 进来的文件**——只有显式 COPY 的才会进，build context 里其他文件不会自动进去。build context 表示执行 `docker build` 的目录。
  把 requirements.txt 文件先拷贝进去，是为了利用Docker缓存，如果这个文件内容没变，pip install 这一步就不用重复执行。
- `COPY . .` , 这一步是把本地目录复制到镜像中的目录，这里两个都是当前目录，第一个. 是本地的当前目录，第二个 . 是镜像中的当前目录，即 /app
- **RUN 指令的产物**——比如 `pip install` 装的包、`apt-get install` 装的软件，都会沉淀到镜像里。pip install -r 表示 --requirement, 读取 requirements.txt 的内容，并安装列出来的依赖包
- **元数据**——ENV、EXPOSE、CMD、ENTRYPOINT、WORKDIR、LABEL 这些不是文件，但作为镜像配置存在。
注意⚠️，区分④ 和 ⑧， RUN 是 build 完了之后，要执行的命令，命令的效果会沉淀到镜像中。CMD是在使用镜像启动容器时，执行的命令，通常为启动要跑的服务或者应用。
更详细的内容，参考 [Dockerfile](Docker/Dockerfile)

### <font color="b48ff4">Docker仓库</font>

镜像分享给其他人，要通过 Docker仓库(registry)。Dokcer仓库是用来存放Docker镜像的地方。最流行和最常用的仓库就是 DockerHub。它是一个公共的Docker仓库，用来集中存储和管理Docker镜像。我们可以在这里下载各种镜像，也可以将自己的镜像上传到这里，这样就可以实现镜像的共享和复用。

### <font color="#b48ff4">Docker Compose</font>

实际使用过程中，我们的应用程序可能用到多个容器协作，比如一个容器来运行 web 应用，另一个容器来运行数据库系统。这样，可以做到数据和应用逻辑的分离。比如web容器down掉了，但是数据库服务器还在运转。此时，我们只需要修复 web 服务器就可以了。docker compose就可以做到这点。

Docker Compose 是一个 Docker 官方的开源项目，是一个用来定义和运行多容器Docker应用程序的工具。比如，搭建一个网站，可能会使用到前端，后端，数据库，甚至缓存，负载均衡等多个服务。这些服务是独立的，但是他们之间又是相互关联的。需要相互配合来工作，前端需要连接后端，后端需要连接数据库。这些服务之间的关联关系，就是 Docker Compose 需要解决的问题。它通过一个单独的 `docker-compose.yaml` 配置文件来将这一组相互关联的容器组合在一起，形成一个项目。然后使用一条命令，就可以启动，停止或者重建这些服务， 这样就可以非常方便管理这些服务了。比如，有一个新同事，之前可能需要半天时间去安装各种依赖和配置运行环境，现在有了Docker Compose 之后，只需要执行一下 Docker compose up 命令，就可以自动安装各种依赖和配置运行环境，然后在本地运行项目了，这样大大提高了开发效率。

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

文件中我们使用 services 定义多个 container, 比如这里定义了一个web容器里面运行 web 应用，然后再定义一个db容器，里面定义了 mysql 数据库系统, 这里通过两个环境变量，指定数据库的名字和连接密码, 通过 volumes 指定一个数据卷永久存放数据。
定义完毕后，我们保存文件，使用 `docker compose up -d` 来运行所有容器，-d代表后台运行所有容器。对应的，我们可以使用 `docker compose down` 来停止并删除所有容器。不过新创建的数据卷需要手动删除，除非加入 --volumes参数。

## <font color="grape">Docker 与 Kubernetes 的关系</font>

虽然大家都在说，kubernetes在逐渐取代docker，但是指的是kubernetes中的容器引擎(container engine )而已。实际上 kubernetes 和 docker 不是一个层面的东西。之前的应用，web容器，数据库容器等，都运行在同一个计算机中，随着应用规模的增大，一台计算机没法满足我们的需求。当我们使用一个集群的计算机提供服务，并做到负载均衡，故障转移等。这时候就可以用kubernetes

![[Pasted image 20260528075938.png]]

简单说，kubernetes 所做的就是将我们的各个容器分发到一个集群(cluster)上运行。并且进行全自动化管理，包括应用部署和升级。
![[Pasted image 20260528080130.png]]

