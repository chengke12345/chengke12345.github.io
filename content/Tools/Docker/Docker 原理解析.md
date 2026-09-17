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

这个镜像不管在你 Mac、 Ubuntu、还是云服务器上跑，里面都是同一套 Python 3.10 + libfoo2，结果完全一致。**宿主机装的 Python 3.6 也好、3.11 也好，全都被屏蔽掉了**——容器进程只看到镜像 rootfs 里的 Python 3.10。

>- ① **Dockerfile 是"环境的源代码"** ——它声明性地描述了运行环境的每一寸。这也是为什么 Dockerfile 必须 check in 到 git 里、必须可重现 build。
>- ② **本地能跑不代表镜像里能跑**——经常遇到的坑：本地 `python main.py` 没问题，因为系统里早就装了某个库；镜像里没装这个库，build 出来跑不起来。本质就是镜像和本地是两个独立环境。
>- ③ **基础镜像决定底色**——`FROM ubuntu:22.04` 给你一个 Ubuntu 用户空间，`FROM alpine` 给你一个体积超小但用 musl libc（不是 glibc）的环境，`FROM scratch` 给你一个**完全空的**镜像（什么都没有，连 shell 都没有）。选 FROM 等于选了底层基底。
>- ④ **"COPY . ."** 是个例外，它确实把项目代码从宿主机搬进镜像——但搬的也只是**文件**本身，不是你本地的运行环境。本地 `pip install` 装的包不会跟着代码一起进镜像，必须在 Dockerfile 里再 `pip install -r requirements.txt` 一遍。

镜像 = **基础镜像提供的环境** + **Dockerfile 后续指令构建的增量**（装的依赖、复制进去的文件、设置的环境变量）。和你本地的运行环境唯一的交集，就是你通过 `COPY` 显式搬进去的那些文件。除此之外，本地和镜像是两个完全独立的世界。

Docker 不是"自动打包本地环境"，而是"**让你写一份配方，按配方生成一份与宿主机解耦的环境**"。"打包本地环境"是从用户体验角度的简化表述——你的目标确实是复刻本地环境，但你得自己把它翻译成 Dockerfile，dockerd 只认 Dockerfile，不认你本地装了什么。

<font color="deeppink">这种"环境与宿主完全脱钩"的特性，就是 Docker 提供可重现性和可移植性的根基。</font>

#### <font color="#b48ff4">Docker 与宿主机内核</font>

Docker 打包的是用户空间文件系统 + 应用程序 + 运行环境。如果应用程序要操作内核，那就会报错。因为Docker使用的是宿主机的内核，如果操作内核通过文件系统想要操作内核，
比如 :

>1. sysctl 参数(`/proc/sys/...`)—— 网络栈调优, 内存管理, 文件句柄数等,比如`net.ipv4.tcp_keepalive_time`  、`vm.swappiness`、`fs.file-max`
>2. **内核模块**（`insmod`、`modprobe`、`rmmod`）
>3. **`/proc`、`/sys` 下的设备和子系统配置**
>4. **cgroup、namespace 之外的全局内核状态**

容器默认对这些的访问权限是**严格受限**的。这些文件只具有只读操作。对他们写会报 read-only 错误。如果是测试脚本的容器，看到这里内核报权限错误，这就可以了，无需其他操作。

如果确实要动内核，Dokcer就不是一个适合的工具了。