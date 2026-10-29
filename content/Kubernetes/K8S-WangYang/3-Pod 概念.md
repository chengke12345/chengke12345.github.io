___

<font color="#eb4349">「Pod 是最小的部署模块」</font>。即在 kubernetes 里，要让任务运行就必须放在 Pod(这里没有容器). Pod 的中文翻译是毛豆。可以理解为，毛豆本身就是一个「Pod」，而毛豆里面有几个豆粒子，每个豆粒子可以理解为「容器 Container」.Pod 可以理解为一种<font color="#45ce6e">「容器组 Container Set」</font>。

![[Pasted image 20261024110121.png]]

一个 Pod 里面可以包含很多 Container. 这些容器组合在一起，这种所谓的逻辑分组，在 kubernetes 里的概念就是 Pod. 我们要运行业务，就是往 kubernetes 集群里部署一个一个的 Pod。

# Pause 容器

上图中，Redis, Nginx, php-fpm 都是应用容器，有一个容器比较特殊，就是「Pause 容器」。
- <font color="#eb4349">「Pause 容器是 Pod 内部第一个启动的容器」</font>。
- 它需要去初始化网络栈，即它要先创建 Pod 内容器共用的网络。
- 还要挂载需要的存储卷。
- 回收僵尸进程。
### `◼︎`僵尸进程

假设有一个 1 号进程，它启动了一个 Nginx 的 master 进程，master 进程会产生很多 worker process 进程，这些工作进程都是被 master 进程管理。master 进程负责分配工作给 worker 进程。

![image|340](Pasted%20image%2020261024140215.png)![image|340](Pasted%20image%2020261024140553.png)

当 master process 意外没了的时候，根据系统原则，子进程 worker process 1, worker process 2, worker process 3 就自动归属到 1 号进程。但是这三个子进程，既存在又不干活。这种进程就是<font color="#45ce6e">「僵尸进程」</font>。1号进程就要负责对这几个僵尸进程进行回收，杀死，释放对应资源。

在一个Pod 内部的容器中，也需要有类似1号进程这样身份的容器，去负责这样的事情，这就是 Pause 容器的职责。
### ◼︎ Pod 内容器共享名字空间(namespace, ns)

一个 Pod 内的所有容器，都和 Pause 容器共享名字空间，Network, PID, IPC。不同的容器放在不同的名字空间下，就被隔离，但同样一个 Pod 内部的不同容器之间，在 network 级别，PID 级别和 IPC 级别，它们的名字空间是共享的。而且是由 Pause 容器提供，其他容器共享过来的。
#### Pod 中容器共享网络 IP

我们之所以会把若干容器放在同一个 Pod 里，是任务的自洽。它们一起是一个最小的模块，能完成单一任务场景，单一模块或微服务模块。所以一个 Pod 中的容器是共享网络 network 的。

同一个 Pod 中的容器共用 Pod 的 IP， 比如上面的 Nginx, Redius, php-fpm 都和 Pause 共用一个 IP。更准确的说，它们共享一个网络命名空间(Network Namespace), 包括 IP，网络接口，端口空间，以及 localhost。从网络角度上看，Pod 就像一台机器，Pod 中的容器 Container, 就像这个机器上跑的各个进程。例如，一个 Pod 的 IP 是 10.244.1.5, 假设里面有两个业务容器，Nginx 与 Java：

| 容器       | 应用监听地址         | 集群内其他 Pod 访问该应用地址 |
| -------- | -------------- | ----------------- |
| Nginx 容器 | `0.0.0.0:80`   | `10.244.1.5:80`   |
| Java 容器  | `0.0.0.0:8080` | `10.244.1.5:8080` |
>- <font color="orange">Pod 内的容器可以直接通过 `localhost:xxxx` 互相访问</font>，例如 Nginx容器可以使用 localhost:8080 访问 Java 容器。
>- <font color="orange">端口空间是共享的，因此会发生端口冲突</font>。比如两个容器都想监听8080，后启动的通常会报地址被占用。
>- <font color="orange">端口是由容器的应用配置决定</font>。kubernetes 不会自动给每个容器分配端口。一个容器的应用也可以监听多个窗口。

Pause 容器的作用就是维持这套共享网络环境的声明周期。常见的 Linux 实现中，运行时先建立 Pod 沙箱和 Pause 容器，再让业务容器加入同一网络命名空间。业务退出或重启时，Pod 的网络配置仍然能保留。
#### Pod中容器共享 PID 和 IPC 

Pause 的其中一个任务是回收僵尸进程。Pause 首先得是这些容器的1号进程，它才有权限回收各个细分容器的僵尸进程。Pause 必须和容器进程共享 PID 和 IPC，才能看到容器的进程信息，才能保证 Pause 成为他们的 1 号进程，才能去回收 Pod 内的僵尸进程，有效保证当前 Pod 的稳定运行。


> [!NOTE] 总结
> 一个Pod 是 kubernetes 里面部署的最小单位。但凡需要在集群内部运行任务，都需要封装在 Pod 内部运行。每个 Pod 都会有一个 Pause 容器，位于其他所有容器之前先启动，初始化网络栈，挂载可能的存储卷，以及回收僵尸进程。其他容器共享 Pause 容器的网络，IPC 和 PID 的名字空间。在 kubernetes 集群里，运行着数万数十万计的 Pod, 去完成各式各样的任务。


> [!Warning] 注意
> Pod 是一个<font color="orange">「逻辑概念」</font>, 它不真实存在。容器之间是独立存在的，我们让几个容器具备了一些特性(比如与一个 Pause 容器共享网络，PID，IPC等)。我们就把 它理解成，这几个容器合成了一个 Pod. 不代表这个 Pod 真的具象化。
> 因此，理论上，可以不借助 kubernetes 集群，只需要 Docker 就可以实现 Pod 的概念。


# Rocky Linux 说明

我们现在使用的操作系统都是 Linux，它是一种内核。基于这个内核，外面还要封装很多生态，比如包管理工具，进程管理工具，防火墙等等。封装了不同软件或生态以后，各个厂商或各个社区就推出了自己的发行版。最早的就是 RedHat, 然后就是 CentOS, 包括后面的 Fedora, Unbuntu 等等。

Torvards Linux 写出了 Linux 操作系统内核，随后红帽公司成立，利用 Linux 内核，加上外围的软件生态，发布了 RedHat 版本。

后期为了保证 RedHat 代码的稳定性，又推出了 Fedora 这个桌面版。Fedora 其实是 RedHat 的上游，当 RedHat 开发了新功能代码，先会发布到 Fedora 上进行测试，因为是桌面版，很多人使用。功能测试没问题以后，再被下发给 RedHat 企业版，供各大公司使用。Fedora 是开源免费的。 

随后，CentOS 社区成立了。 RedHat 有一个特点，「服务收费，代码开源」。CenOS 社区拿到 RedHat 源代码之后，进行可以写修改，把其中一些授权软件移除，加上新的补充协议和优化等，就得到了一个新的发行版，「CenOS」。

然后 RedHat 看到竞品以后，一锅端把 CenOS 社区收购了。现在 CentOS 已经停止服役，并且在逐渐往其他的系统上做迁移了。RedHat 把 CenOS 项目关闭以后，发布了一个新的项目，CentOS Stream。当有新的功能代码发布给 Fedora, Fedora 测试没有问题以后，会发布给 CentOS Stream，CentOS Stream 发布没问题以后，再发布给 RedHat. 相当于 RedHat 成了 CentOS Stream 的下游。CentOS Stream 被称为滚动版，它会不断滚动更新，因此它就不再适合真正的企业商用化了。

这时候，Rocky Linux 就出现了。它同样是从 RedHat 发布的源码进行修改，方式等同于之前的 CentOS. 而且，Rocky Linux 是由之前 CentOS 社区的创始人，去开创的一个新的社区，并且明确声明了要采取一些手段避免重蹈 CentOS 的覆辙。目前，几乎所有的云计算公司，都在跟投 Rocky Linux，也在各个云环境中支持 Rocky Linux。