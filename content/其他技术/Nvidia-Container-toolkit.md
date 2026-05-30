<font color="#45ce6e">Nvidia-Container-toolkit 是让容器能访问宿主机 GPU 的一套工具。</font>

它解决的核心问题是：普通的 docker 容器是看不到 GPU 的，即<font color="orange">容器继承了宿主主机的内核，但是GPU 设备节点</font> `/dev/nvidia*`, <font color="orange">用户态驱动库</font> `libcuda.co`, <font color="orange">以及 NVIDIA 的管理接口，默认都不在容器的命名空间中。</font>只是把设备文件 `--device` 映射到容器中不够，<font color="orange">容器中的CUDA runtime 还需要和宿主机内核驱动版本严格匹配的用户态库。</font>

## <font color="grape">Nvidia-Container-toolkit 工作原理</font>

它首先注册一个 OCI runtime hook(present hook), 叫做 `nvidia-container-runtime-hook`. 当容器启动的时候，Docker/contained 调用底层 runtime(runc) 创建容器，在容器进程真正 exec 之前，这个hook会触发，由 <font color="orange">libnvidia-container 这个库负责把宿主机上正确版本的驱动库，nvidia-smi, 设备节点等一起 bind-mount 进容器。这样容器就只需要安装 CUDA Toolkit / runtime, 而不需要安装驱动，驱动始终是宿主机的那一份。</font>

意味着你可以在同一台机器上用不同 CUDA 版本的镜像（比如 vLLM 官方镜像带的是 CUDA 12.x），而宿主机只要装一个足够新的驱动就行，容器里的 CUDA runtime 向下兼容那个驱动即可（驱动版本 ≥ 容器 CUDA runtime 要求）。

几个核心组件为：

> - `libnvidia-container`：底层 C 库，干实际挂载的活
> - `nvidia-container-toolkit`（hook + CLI）：把上面的库接到 OCI 流程里
> - `nvidia-ctk`：配置工具，比如 `nvidia-ctk runtime configure --runtime=docker` 会帮你改 `/etc/docker/daemon.json`

装好之后典型用法：

```bash
docker run --gpus all nvidia/cuda:12.4.0-base nvidia-smi
```

`--gpus all` 就是触发这套机制的开关，`all` 也可以换成 `--gpus '"device=0,1"'` 指定卡。

## <font color="grape"> Nvidia-Container-toolkit 的安装与配置</font>

<font color="orange">nvidia-container-tookit 是安装在宿主机上的</font>，我们在宿主机上先安装 nvidia-container-tookit.

```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/libnvidia-container/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

在`sudo apt install -y nvidia-container-toolkit` 之前的部分是往 apt 中添加软件库的源。在安装之后，`sudo nvidia-ctk runtime configure --runtime=docker` , 它会修改 `/etc/docker/daemon.json`, 这是把 nvidia runtime 注册进去。
注册之后，要`sudo systemctl restart docker` 重新启动 docker，dockerd才能生效。

注意⚠️：`nvidia-container-toolkit` 中包含了 `nvidia-ctk` 配置工具，所以 `apt install nvidia-container-toolkit` 安装这个包之后，自然就有nvidia-ctk可以直接使用。但是 libnvidia-container并不在安装包中。`libnvidia-container` → 它本体在独立的 `libnvidia-container1` 包里,**不在** toolkit 包内部,但 toolkit 包**依赖**它,所以 `apt` 会自动一并装上。
不过，`nvidia-container-toolkit` 和 `libnvidia-container`都不在Ubuntu官方源里，它们只在 NVIDIA 维护的仓库里发布，所以必须先把这个仓库添加到 apt 的仓库源中，apt 才知道去哪儿拉这些包。
## <font color="grape">容器的启动</font>
#容器构建过程

容器启动时，从 `docker run` 到真正把容器启动起来，要经过以下步骤。

```
docker (CLI)
   │  REST API (/var/run/docker.sock)
   ▼
dockerd (Docker daemon)         ← 管镜像、网络、卷、API
   │  gRPC
   ▼
containerd                      ← 容器生命周期管理器（高层 runtime）
   │
   ▼
containerd-shim                 ← 每个容器一个，守着容器进程
   │  exec
   ▼
runc                            ← 真正动手创建容器的低层 runtime（OCI runtime）
   │  clone() + namespaces + cgroups + pivot_root
   ▼
你的容器进程（PID 1）
```

dockerd 接到 docker run 指令后，先调用containerd，containerd 调用 runc 真正动手创建容器。
关键分界线在 **containerd 和 runc 之间**。习惯上把它们叫两类：
- **高层 runtime（high-level）**：containerd、CRI-O。管的是"宏观"的事——拉镜像、解包成 rootfs、准备网络和存储、调度容器生命周期。它**自己不直接创建容器**。
- **低层 runtime（low-level）**：runc、crun、gVisor、Kata。真正调 Linux 内核那套系统调用，把一个进程关进 namespace + cgroup 的"笼子"里。这一层就是 **OCI runtime**。

## <font color="grape"> OCI 与OCI runtime</font>

OCI = Open Container Initiative. 是Docker牵头，Linux基金会下成立的标准化组织。<font color="orange">它是一个标准规范，用于规范容器和镜像的格式和行为。</font>

OCI 标准主要是两个核心规范
- <font color="#45ce6e"> ① Runtime Spec: 运行时规范，这是OCI runtime中的OCI的含义，它定义了两个东西：</font> 
	- filesystem bundle的格式：一个容器在磁盘上长什么样。它是一个目录，里面有解包好的rootfs(用户空间文件系统) 加一个 `config.json`。
	- `config.json` 非常关键，是容器的完整声明，用哪些 namespace、cgroup 限制多少 CPU/内存、挂载哪些路径、环境变量、要执行的命令、还有 **hooks**。
	- **命令行约定**：一个程序要想被称作"OCI runtime"，必须实现一组标准子命令——`create`、`start`、`kill`、`delete`、`state`，参数和行为都按 spec 来。

- <font color="#45ce6e">② Image Spec（镜像规范）：定义镜像的格式</font>。分层文件系统、manifest、config。管的是"镜像长什么样、怎么解包成 bundle"。和 runtime 互补image spec 管静态的包，runtime spec 管把包跑起来。

<font color="#b48ff4"><u>OCI Runtime 就是指的，一个遵守 OCI Runtime Spec 规范的可执行文件</u></font>。这个执行文件，你它一个 bundle 目录（含 `config.json`），调 `runtime create <id>`，它就照着 spec 把容器建出来。runc 是参考实现，crun（C 写的，更轻）、gVisor（Google 的用户态内核沙箱）、Kata（轻量虚拟机）都是符合这个规范的不同实现——可以互换，因为接口一致。

## <font color="grape">容器实体</font>

真正构建好了之后的容器的物理实体，在系统中有两部分，<font color="orange">内存中跑的进程</font> 和 <font color="orange">磁盘上的内容</font>。
磁盘部分就是磁盘上的 每个容器对应的 filesystem bundle, 每个容器都有的目录，里面有 rootfs 和 config.json。

bundle是每个容器都有一个自己的bundle，<font color="orange">它就是一个待运行容器在磁盘上的完整形态</font>。runc就是通过它来构建容器的。bundle大概长这样：

```
/run/containerd/.../<container-id>/    ← bundle 目录
├── config.json      ← 这就是 OCI spec
└── rootfs/          ← 容器的根文件系统（解包好的镜像）
```


`config.json` 是这个容器的**完整说明书**——把"怎么创建这个容器"的所有信息声明式地写在里面。关键字段：config.json是OCI Spec的物理形态, 它构建程序在构建容器时参考的文件。

另一部分就是内存中的容器进程。如果容器销毁了(`docker rm`), <font color="#45ce6e">那么容器进程就销毁了，同时这个容器对应的磁盘上的 filesystem bundle 也就一并销毁，所以容器一停，bundle就销毁。</font>

## <font color="grape">containerd 和 runc</font>

Docker 创建对象的时候，默认由 containerd调用 runc 来完成构建。<font color="#b48ff4"><u>runc 就是一个遵守 OCI Runtime Spec 标准实现的构建容器的程序</u></font>。

构建容器的过程为，containerd 接受到 docker run 的命令后，会根据此次传递的docker run 参数，它会做这几件事：
1. 准备 rootfs(把镜像的只读层叠加 + 一个可写层,挂成容器的根文件系统)
2. 读镜像的 image config + 合并你这次的运行时参数
3. **生成 config.json**,连同 rootfs 一起组成 bundle,放进 `/run/containerd/.../<id>/`
4. 调 runc(或你的场景里:先调 nvidia-container-runtime → 它改 config.json → 再调 runc)

所以 config.json 是"每次容器进入运行状态时"现生成**的。你 `docker stop` 再 `docker start` 同一个容器,会**重新生成一份(因为它要重新进入运行状态)。


> [!NOTE] 容器构建
> <font color="#45ce6e">containerd 接受到docker run命令，它会根据命令的参数和image的内容，生成容器的bundle，bundle中含有 config.json，它就是OCI Spec的物理体现, 然后调用 runc, runc只会根据config.json创建出容器进程。**runc 的行为 100% 由 config.json 决定**。同一个 runc 二进制,喂不同的 config.json,就创建出完全不同的容器。runc 本身是"哑"的,它只是个忠实的执行器.</font>
> <font color="orange">至此，容器创建完成，内存中有一个容器进程，磁盘上有这个容器对应的 bundle。</font>

## <font color="grape">nvidia-container-runtime </font>

 runc 遵守的时OCI runtime spec。它没有针对nvidia GPU驱动等资源进行处理的部分，所以，如果直接使用runc, 我们没法进行挂载驱动等一系列挂载容器的行为，所以它使用了另一个二进制可执行程序来构建容器，就是 nvidia-container-runtime。

#注册
 由于这是另一个容器构建程序，而Docker默认的是使用 runc, 所以我们需要把这个构建程序注册在dockerd中，让它在需要的时候，知道使用  nvidia-container-runtime 这个二进制来构建容器。所以，我们要把这个程序注册在 `/etc/docker/daemon.json`中. 注册之后，dockerd 就知道在特定情况下调用 nvidia-container-runtime 来创建容器，而不是默认的runc. 
 注册的时候我们使用的是
 
 ```bash
 sudo nvidia-ctk runtime configure --runtime=docker
 ```

nvidia-ctk 是 nvidia-container-toolkit 中包含的配置工具。这条命令执行后，它就会自动修改 /etc/docker/daemon.json, 把 nvidia runtime注册进去，完成"登记"。`nvidia-ctk runtime configure --runtime=docker` 往 `daemon.json` 写的是这么一段
```json
{
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "runtimeArgs": []
    }
  }
}
```

注意，这是在runtimes中增加了一个名叫 nvidia的可选 runtime，并没有动 `default-runtime`（默认仍是 `runc`）。只是登记了一个备选项。
注册之后，`sudo systemctl restart docker` 重新启动 docker，就会生效。

## <font color="grape">OCI Spec注入</font>

注册完成，当我们使用`docker run --gpus all` 这个参数的时候，dockerd就会调用 nvidia-container-runtime 来构建容器。首先，dockerd/containerd 合并生成相应的容器 bundle，包括 OCI Spec，即容器的 `config.json`.   

然后，dockerd 调用 nvidia-container-runtime 来构建容器。
nvidia-container-runtime 所做的工作就是把容器的`config.json` 读取进来，然后往 `hooks.prestart`里面塞一条记录：

```json
"hooks": {
  "prestart": [
    {
      "path": "/usr/bin/nvidia-container-runtime-hook",
      "args": ["nvidia-container-runtime-hook", "prestart"]
    }
  ]
}
```

改完,再把修改后的 spec 交给 runc。最终还是调用runc来完成构建容器。 runc 读到的 `config.json`已经带着这条 hook 了,它执行到 `run_hooks(spec.hooks.prestart)` 那一步时,就会去跑 `nvidia-container-runtime-hook`——GPU 挂载就此发生。
<font color="red">所谓注入,就是在 runc 读取 config.json 之前,修改这个 JSON 文件, 往里面追加一个hook条目。</font>
而`nvidia-container-runtime-hook`就是调用`libnvidia-container` 这个so库来完成挂载工作。

这个过程有点像设计模式中的装饰器模式，在runc前面增加一个 OCI 的注入操作，就形成了装饰器，nvidia-container-runtime程序。执行它的时候，就是先执行注入操作，再调用runc。


> [!NOTE] 总结
> <font color="#b48ff4">nvidia-container-tookit 解决的是包含CUDA的docker容器只能打包用户空间的环境的问题。对于GPU，容器中最多有CUDA Runtime。但是CUDA Runtime要依赖驱动，容器无法打包内核驱动。只有CUDA runtime和驱动严格匹配，CUDA程序才能正确执行。</font>
> 
> <font color="lightblue">所以，就把驱动等需要的宿主资源，在容器启动的时候，挂载到容器上。这个机制使得CUDA应用在容器中可以通过CUDA Runtime运行，只和容器内的环境有关，与外部环境无关。CUDA Runtime可以在不同的宿主机上调用不同挂载上来的nvidia驱动来完成工作。这就使得容器可以在任何环境中运行，只要 CUDA Runtime 和 驱动能兼容就可以。</font>
> 
> <font color="#45ce6e">它的运行原理是，Docker在构建容器时，传入 --gpus all 的参数，就不使用默认runc构建容器，而使用经过注册的 nvidia-container-runtime 程序来构建容器。nvidia-container-runtime 本质上还是调用runc来完成工作，不过在调用runc之前，通过OCI Spec 注入的方式，增加一个 prestart 的hook, 即修改容器bundle中的 `config.json`文件，增加一个hook。然后再调用runc。runc看到hook以后就会先去(这就是prestart)调用hook指向的程序，`nvidia-container-runtime-hook`，这个程序就会通过 libnvidia-container库(.so)，把驱动挂载到容器上，然后执行runc后续工作，完成容器的构建。这就是 nvidia-container-tookit 的基本原理。</font>


