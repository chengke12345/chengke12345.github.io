# 1. 默认网络 bridge 

Docker 服务启动时 `systemctl start docker`， 启动的 dockerd 后台进程会确保宿主机上有名为 bridge 的默认网络，它对应的网桥通常叫做 `docker0`(网桥理解为虚拟的交换机，连接到交换机就相当于连接到了这个网络，docker0 就是这个虚拟交换机的名字).  注意⚠️，bridge 是默认网络的名字，而 docker0 是 bridge 网络的网桥(交换机)的名字。

Docker 服务启动时，dockerd 进程就会维护这个默认网络。当一个 docker 容器被创建时，如果没有指定网络，docker 容器就会默认连接上这个网桥，加入这个默认网络。
当然，我们创建新的网路，指定容器加入自定义的网络。比如

```bash
docker network create net-a
docker run --network net-a nginx
```

这里创建了一个新的网络 net-a, 让 nginx 这个容器加入 net-a 网络。注意，这里的 net-a 对应于上面的 bridge, 是网络名，底层会在宿主机上创建一个独立的网桥(独立的虚拟交换机)，网桥的名字通常为 `br-a1b2c3d4e5f6` 这样的形式，这个名字不重要，实际操作中知道网络名 net-a 就够了。

# 2. Docker bridge 网络驱动

上面说的 bridge 是默认网络的名字，但是还有一个容易混淆的概念，就是 Docker bridge, 它是一种网络驱动。<font color="orange">「网络驱动」指的是 Docker 用来创建网络，把容器接入网络的一套实现</font>。docker 是使用这个驱动来创建新的网络，在宿主机上新建网桥或者虚拟交换机。同一台 Linux 服务器上的多个容器连接到同一个网桥上，就是加入了同一个网络，每个容器用自己的虚拟网卡接入。

| 网络名(NAME) | 驱动 DRIVER | 宿主机上的实体                                 |
| --------- | --------- | --------------------------------------- |
| bridge    | bridge    | 默认网络，名字叫bridge, 对应的 Linux 网桥通常叫 docker0 |
| net-a     | bridge    | 自定义网络，对应另一座新建的独立 Linux 网桥，名字不重要         |

# 3. Docker 容器端口映射

docker 的容器启动时，如果没有 --network xx 指定网络，就加入默认的 bridge 网络。下面就以默认网络为例讨论，所有结论在自定义网络下，都同样成立。
docker 容器启动后，默认由 bridge 网络的网桥分配 IP，通常为 172.17.0.xx, 172.17.0.1 通常分配给宿主机本身，所以宿主机默认是在 bridge 网络上的。

> 注意⚠️，这个 bridge 网络以及 docker 自定义网络，都是宿主机上的 docker 容器之间的局域网或私有网络。 

我们启动 docker 容器时，通常会提供参数 `-p 8080:80` -p 是 port，它表示的暴露端口，不是监听端口，具体的它表示，Docker 会把对宿主机 8080 端口的请求，转发到这个 docker 容器 IP 上的 80 端口。这是宿主机和容器之间的端口映射，意思就是容器的暴露端口，至于容器内的服务是在监听哪个端口是另外的设置(当然通常会一样，不然暴露这个端口，又没有服务监听，那就没有意义)

> 注意⚠️，这个IP 不一定是专属于容器，可能是一个 pod 内的容器在共享这个 IP，或者是容器下有多个应用进程在共享一个网络命名空间，所以更准确的说法是转发到这个 IP 下的 80 端口，由具体监听这个端口的容器或进程处理。

## 端口映射的实际实现

端口映射是在容器启动时，由 dockerd 后台进程建立的端口映射，后续连接通常由 Linux 内核按规则转发，不需要每次由 dockerd 接受处理。

以 Pod 示例，Pod IP 为 172.17.0.2，另一台机器访问 `宿主机IP:8080`，端口映射建立过程为：
1. 启动 pause 时，dockerd 根据 `-p 8080:80` 在宿主机上配置转发规则
2. 连接到达时，Linux 内核匹配到这条规则。把目的地址和端口从`宿主机IP:8080` 改为 172.17.0.2:80, 再通过 Docker 网桥转发送达。这一步叫「目的地址转换，DNAT」.
3. 到达 Pod 后再由具体监听 80 端口的容器比如 nginx 来接受处理。

# 4. Docker 实现 Pod

我们可以用 Docker 实现 Pod 来进一步理解 docker 管理的网络层次。假设我们使用 3 个容器组成一个 Pod。pause，nginx，ghost。
## I. nginx.conf 配置文件

```nginx.conf
error_log stderr;
events {worker_connections 1024;}
http {
	access_log /dev/stdout combined;
	server{
		listen 80 default_server;
		server_name example.com www.example.com;
		location / {
			proxy_pass http://127.0.0.1:2368;
		}
	}
}
```

首先指定当前的错误日志，通过标准错误输出。然后指定当前的线程数量1024。然后指定了一个 http 的区域，指定当前日志访问路径。定义了一个主机区域(server), 监听 80 端口，是一个默认服务器，当前服务名字是 example.com. 同时 location 还定义了一个根区域，对根的访问反向代理到本机回环接口 127.0.0.1:2368 端口上。

## II. 启动三个 Docker 容器

```bash
docker run --name pause -p 8080:80 -d k8s.gcr.io/pause:3.1
```

docker run 启动容器，--name pause 指定当前容器名，-p 表示 port, 宿主机监听的 8080 端口对应到容器监听的 80 端口，即访问物理机端口 8080 代表访问容器内部端口 80。-d 表示 daemon, 放在后台运行。最后指定使用的是 kubernetes 的 pause 容器镜像。

```bash
docker run --name nginx -v `pwd`/nginx.conf:/etc/nginx/nginx.conf --net=container:pause --ipc=container:pause --pid=container:pause -d nginx 
```

指定当前容器名为 nginx, 然后 -v 进行卷挂载，把当前目录下的 nginx.conf 配置文件，挂载到容器内部，变成默认配置文件。 `--net=container:pause --ipc=container:pause --pid=container:pause`  表示跟 pause 容器共享网络，IPC 和 PID 命名空间。-d 放在后台运行，基于 nginx 的镜像启动。

```bash
docker run -d --name ghost -NODE_ENV=development --net=container:pause --ipc=container:pause --pid=container:pause ghost：5.8.2-alpine
```

ghost 是启动一个应用的容器，ghost 是基于 PHP 编写的个人博客系统，同样与 pause 共享网络，IPC 和 PID。我们会通过 nginx 反向代理到它。 指定具体的 ghost 版本和 `NODE_ENV=development` 是因为更新的版本已经必须要强制连接 Mysql 数据库，不再支持 sqlite，这里是为了简化处理，让它支持直接使用 sqlite, 而不需要额外配置。

pause, nginx, ghost 三个docker 容器启动完成后就可以通过 docker ps 看到正在运行的容器：

![700](Pasted%20image%2020261126174035.png)

## III.  网络分析

由于 nginx 和 ghost 都采用了 `--net=container:pause --ipc=container:pause --pid=container:pause` 的设置，所以它们是共享 pause 的网络 IP，PID 和 IPC 的。

这样对外，它们是一个 IP。逻辑上，对外它们就是一个机器，里面的容器是这个机器上的各个进程，或者说对外，它们就是一个拥有独立命名空间的容器，容器内部是共享网络IP，PID 和 IPC的各个应用进程。<font color="orange">这在逻辑上就是一个 Pod</font>。

上面的三个容器中，只有 pause 对外暴露了端口，ghost 和 nginx 的 ports 处都为空。pause 在启动的时候，由 docker0 给它分配了 IP，ghost 和 nginx 都是共享 pause 的 IP。这种情况下，我们无法通过访问宿主机访问到 ghost 和 nginx，因为宿主机上没有这两个容器的端口映射。

但是我们可以通过宿主机上的 bridge 网络直接访问 ghost 和 nginx，因为这两个容器自己是默认监听 2368 和 80 端口的。这和它暴露端口映射是两回事。
具体的，如果访问 `宿主机IP:2368` , `宿主机IP:80` 或者宿主机上访问 `127.0.0.1:2368` `127.0.0.1:80` 是无法访问到 ghost 和 nginx 容器的。但是在宿主机上可以通过 bridge 内网 `172.17.0.2:2368` 和 `172.17.0.2:80` 访问到 ghost 和 nginx。
所以，对外(除开宿主机通过 bridge 内网)，只有通过 pause 暴露的映射端口，才能访问到 pod 内部的容器。

![](Pasted%20image%2020261126182234.png)

# 5. Docker 容器中的多层网络

根据上面的讨论，Docker 容器中涉及三层网络：
1. 宿主机自己对外的网络。
2. 宿主机上 docker 容器之间自己组成的网络，默认是 bridge 网络，也可以是自定义网络。
3. 多个 docker 容器组成 Pod 或者是 一个容器的独立命名空间。Pod 内部多个 docker 容器，或者 docker 容器下的多个应用进程之间，它们共享 IP，PID，IPC。对外是一个网络，对内通过监听端口进行区分。

> [!attention] pause 容器不监听任何端口
> Kubernetes 标准下的 pause 镜像里的 ，pause 进程不监听任何端口，启动后只等待系统信号，完成管理 Pod 内各容器进程的任务。它的主要功能就是启动的时候，初始化网络，挂载需要的存储卷，以及杀死并回收僵尸进程。

## Pod 中 Docker 容器的架构

 Pod 中的 docker 容器有个架构问题。为什么要专门起一个 pause 容器去给 nginx 和 ghost 共享网络，PID，IPC 呢？从功能上来讲完全可以直接启动 nginx 容器，然后再启动 ghost 与 nginx 容器共享 IP，PID，IPC。

 这是因为如果使用 nginx, 那么它是一直在处理业务的。nginx 容器出错死掉的概率会很高，但是用一个 pause 容器，它几乎在启动执行初始化网络，挂载必要的磁盘文件之后，就休眠了。所以它的名字叫 pause。在 pause 为主的 Pod 里，pause 死的概率微乎其微。所以，用 pause 去初始化网络栈，去给别人共享网络，共享 IPC，共享 PID，会达到一个更稳定的状态。

 其次，如果用 nginx，它必要的时候也无法回收 ghost 僵尸进程，pause 的代码是独立开发的，它可以回收僵尸进程。

 这是要用独立的 pause 去给别的容器共享的主要原因。

