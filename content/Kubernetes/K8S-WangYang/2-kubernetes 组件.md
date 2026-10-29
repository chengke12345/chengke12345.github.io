
___
# 1.  架构

Kubernetes 的宏观架构里主要分为两个角色。一个是 master, 一个是 node。master 是「管理端」或「控制器端」， 它主要功能是维稳整个集群的正常运行。node 是实际运行当前业务的机器。
微观架构就详细一些

![[Pasted image 20261023101151.png]]

图中上半方框中的部分就是 master, 下半方框中的部分就是 node。
# 2. API Server

master 中，有一个「API Server」，应用程序编程接口，它是应用程序背后的处理端，所以叫 Server. 如果我们想和 Kubernetes 集群沟通交互的话，就需要有一个入口，API Server 就是整个集群的入口。
连接它的是比如 kubectl，webUI(虚线是因为官方默认没有配置图形化界面，但可以自己安装，比如官方社区开发了一个 dashboard 的仪表盘组件)。kubectl 就是 kubernetes 集群的客户端工具或命令行工具。所有的命令键入，都需要通过这个工具下发给当前的 Kubernetes 集群，具体的就是下发给了 API Server. 所以整个 API Server 就是对外访问的一个入口。
除此之外，旁边的 scheduler 和 replication controller 也是连接到 API Server。所以，<font color="#eb4349"><u>「API Server 是所有组件连接的中枢点」</u></font>。

# 3. Restful 

API Server 再将理解到的数据，存储至后端的 etcd. API Server 本身以 Restful 风格的范式。

![[Pasted image 20261023103506.png]]

Restful 或 Rest式的 Web 编程接口的风格，其实是一种「概念」或「定义」。在使用 Restful 之前，我们访问一个服务，我们可以根据不同的 URI 或者不同的后端地址，去确定执行的功能是什么。使用 Restful 之后，我们不再以 URI 去区分功能，而是通过不同的动作定义去定义不同的功能。比如上面，URI 都是 http://127.0.0.1/user, 但是 GET，POST，PUT，DELETE 定义了不同的功能或行为。这就是 Restful 的概念，它不是一个技术，而是一个开发思想，目的是尽可能让当前的接口复杂度降低，更便于理解。 

API Server 就使用了 Restful 的范式，开发了集群的暴露接口。API Server 就是接受到请求，审核请求通过以后，将请求的数据存放在后端 etcd 服务器内部。

# 4. etcd

etcd 叫做「键值对数据库」。这种存储方式，便于记录集群信息。etcd 是 Core OS 公司开发的基于 Golang 语言，通过 RAFT 协议开发的一个键值对数据库。简答说，当前集群的整个信息都会被保存到 etcd 中。多个 master 对应 多个 etcd，多个 etcd 就可以形成 etcd 集群，这个集群可以保证，任何一个节点的损坏，不会将当前信息丢失。

所以 etcd 的作用就是「存储当前集群配置以及信息参数」。

# 5. scheduler 与 controller manager

「scheduler, 调度器」，调度器可以去绑定需要运行的容器和节点之间的关系，对应的节点就会运行我们希望的容器。

「replication controller」在现在版本中，也叫做 controller manager. 集群管家。如果一个容器的应用死了，controller manager 会保证杀死这个容器，并创建一个新的容器来顶替它。它的作用是尽可能的保证当前集群可用。

「scheduler」, 「replication controller」, 「api server」这些组件就构成了 master 节点。

> [!NOTE] 流程总结
> 如果通过命令行工具 kubectl, 下发一个容器创建运行的任务，请求会发送至 API Server， API Server 会去审核权限，如果没问题，就会把这样的任务保存至 etcd 中。scheduler 调度器会定期从API Server 获取当前需要运行的任务，是否跟节点形成了绑定(任务是否已经分配给节点来执行，这里是绑定关系，并不是实际执行)，如果还有没跟节点绑定的，调度器就会为这个任务，挑选一个最合适节点进行绑定。replicaiton controller 会定期检查集群运行状态，是否和预期相符。就是这几个组件，保证整个集群的稳定，以及对后端 node 节点的管理。

# 6. Node

Node 中包含的组件，包括 Kubelet, Kube proxy, Pod, Container。
如果以 Kubernetes 运行云的概念，叫 Paas，平台即服务。它会通过容器的方案去运行业务。那就与我们常用的容器管理引擎高度相关，比如 docker，containerd, podman。它们才是真正帮我们创建容器，删除容器的组件。

## Kubelet

kubelet 并不会真正的创建容器，它会通过接口调用 Docker，Podman 去真正创建容器。Kubelet 主要起到的是一个承上启下的作用。一方面它要监听 API Server, 我这个节点是否有容器要创建，如果有，kubelet 就会去调用 docker, podman 或 containerd 等 CRI(Container Runtime Interface)，帮我们完成容器创建。

## Kube proxy

Kube proxy 也会监听 API Server，发现当前节点是否有一些防火墙规则，负载均衡规则需要创建。
也就是，Kubelet 解决的是承上启下，容器创建和删除的问题。Kube proxy 主要承上启下解决的是，当前节点的网络，负载均衡问题。有这两个组件，当前节点就会并入到整个 Kubernetes 的管理生态中去实际运行项目。

## 组件、插件、附件

以上这些是 Kubernetes 的核心组件，组件就是不可或缺，如果缺了任何一个，Kubernetes 的核心功能就不完整了。除了「组件」以外，我们还有「插件」，以及「附件」。

![[Pasted image 20261023113442.png]]

插件包括容器引擎，例如 Docker, Podman, Containerd 等。CoreDNS，提供 Kubernetes 集群内部的一个私有的域名解析服务，让节点在 Kubernetes 内部有自己的内部域名。Ingress Controller，提供一种 7 层的负载均衡, 我们常见的 LVS 是典型的 4 层负载均衡，Kubernetes 官方已经提供了一个 4 层负载均衡，叫做 Service。但要使用 7 层负载均衡，就需要安装 Ingress Controller 去实现。
这几个都是官方提供好的接口，把这些插件通过接口一连接，就可以集群扩展出对应功能了。所以叫插件。

附件。附件和我们本身没有连接，也不是组成的一部分。只是搭在主体上，这叫做附件。比如 Prometheus, Dashboard，Federartion.

总结：组件就是系统的组成部分，不可获取。插件就是系统留了接口，可以用外部插件进行连接，让系统扩展出对应功能。附件既不是系统的一部分，也不连接系统，而是作为外部软件搭在系统上，和系统一起协作工作。

这就是在整个 Kubernetes 生态中，可以见到的组件，以及常见的插件和附件