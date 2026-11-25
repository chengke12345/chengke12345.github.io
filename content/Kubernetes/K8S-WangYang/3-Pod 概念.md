___

<font color="#eb4349">「Pod 是最小的部署模块」</font>。即在 kubernetes 里，要让任务运行就必须放在 Pod 里(Kubernetes的部署和调度中没有容器概念). Pod 的中文翻译是毛豆。可以理解为，毛豆本身就是一个「Pod」，而毛豆里面有几个豆粒子，每个豆粒子可以理解为「容器 Container」.Pod 可以理解为一种<font color="#45ce6e">「容器组 Container Set」</font>。

![[Pasted image 20261024110121.png]]

一个 Pod 里面可以包含很多 Container. 这些容器组合在一起，这种所谓的逻辑分组，在 kubernetes 里的概念就是 Pod. 我们要运行业务，就是往 kubernetes 集群里部署一个一个的 Pod。

# 1. Pause 容器

上图中，Redis, Nginx, php-fpm 都是应用容器，有一个容器比较特殊，就是「Pause 容器」。
- <font color="#eb4349">「Pause 容器是 Pod 内部第一个启动的容器」</font>。
- 它需要去初始化网络栈，即它要先创建 Pod 内部容器共用的网络。
- 还要挂载需要的存储卷。
- 回收僵尸进程。
# 2. 僵尸进程

假设有一个 1 号进程，它启动了一个 Nginx 的 master 进程，master 进程会产生很多 worker process 进程，这些工作进程都是被 master 进程管理。master 进程负责分配工作给 worker 进程。

![image|340](Pasted%20image%2020261024140215.png)![image|340](Pasted%20image%2020261024140553.png)

当 master process 意外没了的时候，根据系统原则，子进程 worker process 1, worker process 2, worker process 3 就自动归属到 1 号进程。但是这三个子进程，既存在又不干活(1号进程无法指挥worker 进程干活)。这种进程就是<font color="#45ce6e">「僵尸进程」</font>。1号进程就要负责对这几个僵尸进程进行回收，杀死，释放对应资源。

在一个Pod 内部的容器中，也需要有类似1号进程这样身份的容器，去负责这样的事情，这就是 Pause 容器的职责。

# 3. Pod 内容器共享名字空间(namespace, ns)

一个 Pod 内的所有容器，都和 Pause 容器共享名字空间，Network, PID, IPC。不同的容器放在不同的名字空间下，就被隔离，但同样一个 Pod 内部的不同容器之间，在 network 级别，PID 级别和 IPC 级别，它们的名字空间是共享的。而且是由 Pause 容器提供，pod 内的其他容器来共享的。

<font color="deeppink">「pod 中的容器」和「容器中的应用」，是相同的逻辑概念</font>。其中最重要的概念就是「名字空间」，不同的容器在不同的名字空间下，是被隔离的。但同一个 pod 下，内部的不同容器之间在 <font color="orange">「network 级别」</font>，<font color="orange">「PID 级别」</font>和 <font color="orange">「IPC 级别」</font>它们的名字空间是共享的。
## <font color="#b48ff4"> I. Pod 中容器共享网络 IP(network)</font>

我们之所以会把若干容器放在同一个 Pod 里，是任务的自洽。它们一起是一个最小的模块，能完成单一任务场景，单一模块或微服务模块。所以一个 Pod 中的容器是共享网络 network 的。

同一个 Pod 中的容器共用 Pod 的 IP， 比如上面的 Nginx, Redius, php-fpm 都和 Pause 共用一个 IP，也就是说一个 pod 对外暴露只使用一个 IP。

更准确的说，它们共享一个网络命名空间(Network Namespace), 包括 IP，网络接口，端口空间，以及 localhost。从网络角度上看，Pod 就像一台机器，Pod 中的容器 Container, 就像这个机器上跑的各个进程。例如，一个 Pod 的 IP 是 10.244.1.5, 假设里面有两个业务容器，Nginx 与 Java：

| 容器       | 应用监听地址         | 集群内其他 Pod 访问该应用地址 |
| -------- | -------------- | ----------------- |
| Nginx 容器 | `0.0.0.0:80`   | `10.244.1.5:80`   |
| Java 容器  | `0.0.0.0:8080` | `10.244.1.5:8080` |
>- <font color="orange">Pod 内的容器可以直接通过</font> `localhost:xxxx` <font color="orange">互相访问</font>，例如 Nginx容器可以使用 localhost:8080 访问 Java 容器。
>- <font color="orange">端口空间是共享的，因此会发生端口冲突</font>。比如两个容器都想监听8080，后启动的通常会报地址被占用。
>- <font color="orange">端口是由容器的应用配置决定</font>。kubernetes 不会自动给每个容器分配端口。一个容器内的多个应用也可以监听多个端口。

Pause 容器的作用就是维持这套共享网络环境的声明周期。常见的 Linux 实现中，运行时先建立 Pod 沙箱和 Pause 容器，再让业务容器加入同一网络命名空间。业务退出或重启时，Pod 的网络配置仍然能保留。

> 总结：Pod 内的容器之间共享 IP，即对外使用同一个 IP，共同使用 localhost, 以不同端口号对应到不同容器上。
## <font color="#b48ff4">II. Pod 中容器共享 PID</font>

Pod 内的容器共享 PID，是说 Pod 内的容器共享 Linux 下的 PID 命名空间。

默认情况下，容器内部有自己独立的一套 PID，容器主进程默认 PID 1. 容器的进程，是看不到其他容器或其他容器下的子进程的。只有在 Linux 宿主机层面可以区分 pod 内不同的容器是不同 PID，这种情况在 Pod 层面，就没有 PID 的概念了。

如果设置了 `shareProcessNamespace:true` (使用 Pod 一般都会打开这个设置)，那么整个 pod 里的进程就使用同一套 PID 编号，所以任何一个容器下的应用进程都能知道其他容器下的子进程，使用同一套 PID 就可以把同一个 Pod 下的所有进程识别出来。Pause 就是 PID 1 的进程。

如果使用默认的关闭设置，就仅仅是把多个容器放进同一个 pod, 容器各自使用各自的 PID 编号。

> [!example] Pod 中容器共享 PID
> 一个 pod 中有两个容器 A，B。默认情况下，A，B 各自有自己的 PID 命名空间。各有 PID 1 的进程。A 容器内运行的子进程并不知道 B 容器下运行的进程。设置`shareProcessNamespace:true` 后，一个  pod 内的所有容器下的所有进程共同使用同一套 PID。A， B 下运行的子进程都能知道对方容器下运行的每个进程。并且它们在 pod 内的 PID 1 的进程，就是 Pause 进程，

Pause 的其中一个任务是回收僵尸进程。Pause 首先得是这些容器的1号进程，它才有权限回收各个细分容器的僵尸进程。Pause 必须和容器进程共享 PID 和 IPC，才能看到容器的进程信息，才能保证 Pause 成为他们的 1 号进程，才能去回收 Pod 内的僵尸进程，有效保证当前 Pod 的稳定运行。

## <font color="#b48ff4">III. Pod 中容器共享 IPC</font>

在同一个 Pod 里，共享 IPC 指容器默认共享同一个 Linux IPC 命名空间。IPC，Inter process communication, 进程间通信, 实质上指的是通信资源。共享 IPC 命名空间后， A 和 B 中的进程可以访问同一组通信资源，例如 System V 共享内存，消息队列和信号量等。这些通信资源叫做 IPC 对象。共享 IPC，A 和 B 中的进程都可以找到同一个 IPC 对象，两边可以通过它交换数据，而无需知道对方的 PID。
所以这里指的是，共享 IPC 对象的命名空间，让 Pod 内的进程可以直接访问 IPC 对象，实现和其他进程的通信或交换数据。

## <font color="#b48ff4">IV. 总结</font>

> [!NOTE] 总结
> 一个Pod 是 kubernetes 里面部署的最小单位。但凡需要在集群内部运行任务，都需要封装在 Pod 内部运行。每个 Pod 都会有一个 Pause 容器，位于其他所有容器之前先启动，初始化网络栈，挂载可能的存储卷，以及回收僵尸进程。其他容器共享 Pause 容器的网络，IPC 和 PID 的名字空间。在 kubernetes 集群里，运行着数万数十万计的 Pod, 去完成各式各样的任务。


> [!Warning] 注意
> Pod 是一个<font color="orange">「逻辑概念」</font>, 它不真实存在。容器之间是独立存在的，我们让几个容器具备了一些特性(比如与一个 Pause 容器共享网络，PID，IPC等)。我们就把 它理解成，这几个容器合成了一个 Pod. 不代表这个 Pod 真的具象化。

因此，理论上，<font color="deeppink">可以不借助 kubernetes 集群，只需要 Docker 就可以实现 Pod 的概念。</font>

# 3. 回收僵尸进程

## 容器进程

容器本身是一个独立的容器进程，我们默认是里面是一个独立应用，但是实际上一个容器中可以运行多个应用。多个应用会以容器进程的子进程的形式运行。进程在结构上有两种形式，<font color="#45ce6e">假设应用 A, B 由两个进程运行，容器中进程总数取决于进程的启动方式</font>。

> - <font color="orange">「容器主进程由应用担任」</font>：一个容器中的两个应用 A，B。A作为主进程，由 A 启动 B，那么这个容器内就有两个进程。
> - <font color="orange">「容器的主进程是管理进程」</font>：如果进程管理器作为主进程，由主进程分别启动应用进程 A 和 B。那容器内就有三个进程。

docker 容器是借助「名字空间 NS, Namespace」再配合 cgroup, bridge, netfilter, 还包括 chroot 等去实现容器。同一个容器内的应用进程之间是共享 cgroup 和命名空间等资源，但是应用进程监听端口，它们在系统中也是独立服务的进程。

## Pause 回收权限

> 进程回收：进程回收的是某个已退出的子进程留下的记录。僵尸进程已经停止执行，但是内核保留它的 PID 和 退出状态。回收这条记录时，容器内的其他应用可以继续运行。

Pause 在回收进程的时候，也要遵循回收的权限规则，不能垮级回收。假设应用进程 A 启动了子进程 B。B 先退出。这时谁来回收 B 取决于谁是它的父进程。
- 如果 A 还活着，就由 A 调用 wait() 或 waitpid() 回收 B。即使 A 迟迟没有执行回收，pause 也不能代劳。
- 如果 A 已经退出，B 仍待回收，内核会给 B 指定新的父进程。默认容器会有独立的 PID 命名空间，如果 pod 开启了 `shareProcessNamespace: true` , pause 通常是共享 PID 命名空间的 PID 1，所以通常业务进程都会由 pause 进行收养。当 B 被 pause 收养后，才由 pause 调用 waitpid() 回收它。
注意⚠️，每个业务容器默认有自己的 PID 命名空间，父进程先退出时，孤儿进程通常由盖容器的 PID 1 回收，如果存在更近的，被指定负责收养孤儿进程的祖先进程，则由它优先收养。开启 shareProcessNamespace: true 后，Pause 就是整个 Pod 共享的 PID 1。

> [!NOTE] 总结
> <font color="lightblue">Pause 回收僵尸进程，回收的颗粒度是进程，不是容器。一个容器中可能会运行多个应用，这些应用是各自独立运行的进程，当它的父进程先退出时，它就成了孤儿进程，进而成为僵尸进程。默认重新指定父进程由 PID 1 收养，如果容器中有指定祖先进程，优先由它收养。通常情况下，会开启 `shareProcessNamespace: true`, Pause 容器是 PID 1，当出现孤儿或僵尸进程的时候，它会接管。所以，Pause 进程用这种方式管理回收僵尸进程。</font>

# 附加：Rocky Linux 说明

我们现在使用的操作系统都是 Linux，它是一种内核。基于这个内核，外面还要封装很多生态，比如包管理工具，进程管理工具，防火墙等等。封装了不同软件或生态以后，各个厂商或各个社区就推出了自己的发行版。最早的就是 RedHat, 然后就是 CentOS, 包括后面的 Fedora, Unbuntu 等等。

Torvards Linux 写出了 Linux 操作系统内核，随后红帽公司成立，利用 Linux 内核，加上外围的软件生态，发布了 RedHat 版本。

后期为了保证 RedHat 代码的稳定性，又推出了 Fedora 这个桌面版。Fedora 其实是 RedHat 的上游，当 RedHat 开发了新功能代码，先会发布到 Fedora 上进行测试，因为是桌面版，很多人使用。功能测试没问题以后，再被下发给 RedHat 企业版，供各大公司使用。Fedora 是开源免费的。 

随后，CentOS 社区成立了。 RedHat 有一个特点，「服务收费，代码开源」。CenOS 社区拿到 RedHat 源代码之后，进行了一些修改，把其中一些授权软件移除，加上新的补充协议和优化等，就得到了一个新的发行版，「CenOS」。

然后 RedHat 看到竞品以后，一锅端把 CenOS 社区收购了。现在 CentOS 已经停止服役，并且一些中小企业已经在逐渐往其他系统上做迁移了。RedHat 把 CenOS 项目关闭以后，发布了一个新的项目，CentOS Stream。当有新的功能代码发布给 Fedora, Fedora 测试没有问题以后，会发布给 CentOS Stream，CentOS Stream 发布没问题以后，再发布给 RedHat. 相当于 RedHat 成了 CentOS Stream 的下游。CentOS Stream 被称为滚动版，它会不断滚动更新，因此它就不再适合真正的企业商用化了。

这时候，Rocky Linux 就出现了。它同样是从 RedHat 发布的源码进行修改，方式等同于之前的 CentOS. 而且，Rocky Linux 是由之前 CentOS 社区的创始人，去开创的一个新的社区，并且明确声明了要采取一些手段避免重蹈 CentOS 的覆辙。目前，几乎所有的云计算公司，都在跟投 Rocky Linux，也在各个云环境中支持 Rocky Linux。