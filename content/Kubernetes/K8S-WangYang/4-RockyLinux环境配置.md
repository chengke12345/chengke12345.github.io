RockyLinux 是我们使用的 Kubernetes 节点使用的操作系统环境。所以，在创建虚拟机的时候，我们需要对虚拟机操作系统环境进行一定设置。
# 1. RockyLinux 简介

我们现在使用的操作系统都是 Linux，它是一种内核。基于这个内核，外面还要封装很多生态，比如包管理工具，进程管理工具，防火墙等等。封装了不同软件或生态以后，各个厂商或各个社区就推出了自己的发行版。最早的就是 RedHat, 然后就是 CentOS, 包括后面的 Fedora, Unbuntu 等等。

Torvards Linux 写出了 Linux 操作系统内核，随后红帽公司成立，利用 Linux 内核，加上外围的软件生态，发布了 RedHat 版本。

后期为了保证 RedHat 代码的稳定性，又推出了 Fedora 这个桌面版。Fedora 其实是 RedHat 的上游，当 RedHat 开发了新功能代码，先会发布到 Fedora 上进行测试，因为是桌面版，很多人使用。功能测试没问题以后，再被下发给 RedHat 企业版，供各大公司使用。Fedora 是开源免费的。 

随后，CentOS 社区成立了。 RedHat 有一个特点，「服务收费，代码开源」。CenOS 社区拿到 RedHat 源代码之后，进行了一些修改，把其中一些授权软件移除，加上新的补充协议和优化等，就得到了一个新的发行版，「CenOS」。

然后 RedHat 看到竞品以后，一锅端把 CenOS 社区收购了。现在 CentOS 已经停止服役，并且一些中小企业已经在逐渐往其他系统上做迁移了。RedHat 把 CenOS 项目关闭以后，发布了一个新的项目，CentOS Stream。当有新的功能代码发布给 Fedora, Fedora 测试没有问题以后，会发布给 CentOS Stream，CentOS Stream 发布没问题以后，再发布给 RedHat. 相当于 RedHat 成了 CentOS Stream 的下游。CentOS Stream 被称为滚动版，它会不断滚动更新，因此它就不再适合真正的企业商用化了。

这时候，Rocky Linux 就出现了。它同样是从 RedHat 发布的源码进行修改，方式等同于之前的 CentOS. 而且，Rocky Linux 是由之前 CentOS 社区的创始人，去开创的一个新的社区，并且明确声明了要采取一些手段避免重蹈 CentOS 的覆辙。目前，几乎所有的云计算公司，都在跟投 Rocky Linux，也在各个云环境中支持 Rocky Linux。
# 2. 网络配置

启动虚拟机后，假设我们使用 RockyLinux。RockyLinux 9 以后的版本，全部采用 NetworkManager 代替老的 Network 去实现网络的管理。所以，登陆系统后设置网络，需要修改的配置文件是

```bash
cat /etc/NetworkManager/system-connections/ens160.nmconnection
```

有几张网卡就有几个 `ensxxx.nmconnetion`配置文件，数字最小的代表第一张网卡，以此类推。比如下面就有两张显卡， ens160 和 ens192

![](Pasted%20image%2020261124120942.png)

打开 ens160 显卡，这里可以把当前 IPV4 的工作方案从自动改为手动。然后再添加 `address1=192.168.66.11/24` 这里的 /24 表示 255.255.255.0 的子网掩码(这是思科惯用的网络表达方式，后来在全世界普及开来)。

![image|360](Pasted%20image%2020261124121212.png)![image|340](Pasted%20image%2020261124121748.png)

如果有网关，可以加上 `address1=192.168.66.11, gw=xxx`, 当然还可以添加 dns 选项，指定 dns 服务器，只需要在 address1 下面，加上 `dns=xxx.xxx.xxx.xxx; yyy.yyy.yyy.yyy`, 多个 dns 服务器之间用分号分隔。

>注意⚠️：这里是把 IPv4 的工作方式，改为了手动 manual, 并且把 IP 改为了一个固定的 IP，192.168.66.11。意思就是这个网卡连接到的网络中，这个网卡就是固定 IP 192.168.66.11。

修改完后，执行 `systemctl restart NetworkManager` ，这样就可以通过终端连接服务器了。

# 3. 初始化环境

## I. RcokyLinux 软件源更换

首先，我们替换默认 RockyLinux 的 yum 源。默认的官方 yum 源比较慢，所以我们把它替换成阿里云的镜像源。
```bash
sed -e 's|^mirrorlist=|#mirrorlist=|g' \
	-e 's|^#baseurl=http://dl.rockylinux.org/$contentdir|baseurl=https://mirrors.aliyun.com/rockylinux|g' \
	-i.bak \
	/etc/yum.repos.d/[Rr]ocky*.repo
	
dnf makecache
```

通过 dnf makecache 可以建立元数据缓存。

## II. 防火墙设置

```bash
systemctl stop firewalld
systemctl disable firewalld

yum -y install iptables-services
systemctl start iptables
iptables -F
service iptables save
systemctl enable iptables
```

firewalld 是一个防火墙的管理程序。我们把它关闭，使用更通用的 iptables 防火期管理程序。
- 首先 system stop firewalld 临时关闭 firewalld。systemctl disable firewalld 永久关闭。
- 然后 yum -y install iptables-services 安装 iptables 用户态。
- 然后通过 systemctl start iptables 开启 iptables 的用户态。
- iptables -L 可以查看默认规则，要把它清空掉。清空命令就是 iptables -F。
- 然后 service iptables save 将「空规则」保存到持久化文件。
- 最后加一个开机自启，systemctl enable iptables. 
## III. selinux 设置

```bash
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
grubby --update-kernel ALL --args selinux=0
```

sentenforce 0 代表临时关闭。如果要永久关闭, 需要修改当前 /etc/selinux/config 文件，将当前的 enforcing 开启改为 disable 关闭。官方在新版里还建议在内核参数里添加 selinux=0 的标记。这样，重启以后 selinux 就可以达到关闭状态。

## IV.设置时区

```bash
timedatectl set-timezone Asia/Shanghai
```

## V. 环境复用

这样的虚拟机操作环境，在配置 kubernetes 的时候，会复用很多次。我们需要制作虚拟机快照。在VMWare 虚拟机里，关闭虚拟机，然后对这个设置拍摄快照：

![361](Pasted%20image%2020261125172126.png)![617](Pasted%20image%2020261125172257.png)

后续的 kubernetes 集群都会在这样的快照机器上，往后安装。也就是如果安装一个3个节点的机群，就需要初始化三台这样一模一样的机器。

# 4. 安装 Docker

```bash
yum install -y epel-release
yum install -y bridge-utils

modprobe br_netfilter
echo 'br_netfilter' >> /etc/modules-load.d/bridge.conf

echo 'net.bridge.bridge-nf-call-iptables=1' >> /etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-ip6tables=1' >> /etc/sysctl.conf
echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf

sysctl -p
```

- 首先安装 epel 源，它是为了给 bridge-utils 提供源支持。然后再安装 bridge-utils. 
- 然后需要加载一个模块 br_netfilter。它的含义是所有经过网桥的流量，必须要被防火墙处理。并且把这个模块添加到开机自启中。
- 然后，再在内核配置文件中添加三个选项。第一个是在 ipv4 下所有网桥的流量都必须被防火墙给回调，第二个是 ipv6 的回调，第三个是开启路由转发。 
- 最后用 sysctl -p 刷新生效。

添加中科大 ustc 的 docker-ce 源

```bash
sudo dnf config-manager --add-repo https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo
cd /etc/yum.repos.d
sed -e "s|download.docker.com|mirrors.ustc.edu.cn/docker-ce|g" docker-ce.repo

sed -e "s|download.docker.com|mirrors.ustc.edu.cn/docker-ce|g" docker-ce.repo > docker-ce-ustc.repo

mv docker-ce.repo docker-ce.repo.back

yum -y install docker-ce

cat > /etc/docker/daemon.json << EOF
{
	"default-ipc-mode": "shareable",
	"data-root": "/data/docker",
	"exec-opts": ["native.cgroupdriver=systemd"],
	"log-driver": "json-file",
	"log-opts": {
		"max-size": "100m",
		"max-file": "100"
	},
	"insecure-registries": ["harbor.xinxainghf.com"]
	"registry-mirrors": ["https://kfp63jaj.mirror.aliyuncs.com"]
}
EOF
mkdir -p /etc/systemd/system/docker.service.d

systemctl daemon-reload && systemctl restart docker && systemctl enable docker

```

首先添加中科大的 ustc 的 docker-ce 源，这句话就是把我们指定的网址处的 docker-ce.repo 下载下来，dnf --add-repo 会把这个文件保存到 /etc/yum.repos.d/ 目录下，供 DNF 读取。真正的下载源，仍然在 docker-ce.repo 里面的 baseurl 处指定。

添加完源之后，进入 /etc/yum.repos.repo 目录下， `cat docker-ce.repo` 可以看到当前的下载地址。这时候可以使用 sed 把 docker-ce 的源替换为中科大的源。
`sed -e "s|download.docker.com|mirrors.ustc.edu.cn/docker-ce|g" docker-ce.repo` 
这行代码，把文件 docker-ce.repo 文件中的 download.docker.com 替换为 mirrors.ustc.edu.cn。并且把替换后的结果打印到终端。因为参数没有加 i 所以源文件不会被修改。其中的参数 -e 表示 expression, 要执行的替换表达式，`s|旧内容|新内容|g`。所以上述代码只是查看了修改后的结果，然后我们把它输出内容重定向到 docker-ce-ustc.repo 就保存下来了。


替换完成后，将这个 docker-ce 源保存到 docker-ce-ustc.repo, 然后把原来官方的源保存成备用源 docker-ce.repo.back。防止有一天中科大源不能用了，还可以回过头来选择官方的。

然后，`yum -y install docker-ce`, 安装 docker-ce。

然后我们看一下需要准备的配置文件，配置文件在 /etc/docker/daemon.json中。第一行指定 IPC 模式是允许共享。第二行指定当前 docker 的根目录，默认是在 /var/lib/docker/。第三章指定一些额外的启动参数，比如这里指定 cgroup驱动为 systemd。第四行指定当前的日志驱动为 json-file。第五行指定日志文件最大大小和最大数量。第六行指定当前信任的仓库地址，以及当前镜像官方的仓库地址。

配置文件准备好之后，创建 docker service 的管理目录 docker.service.d。然后重启 docker 并且开机自启即可。

# 5. Docker 实现 Pod

Docker 实现 Pod, 要首先了解 Docker 的网络管理，然后了解 docker 实现 Pod 的容器形态。详细参考 [Docker网络管理](Kubernetes/Docker 网络管理)
