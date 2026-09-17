# 1. Docker 容器的网络

使用 `docker run` 启动容器时，如果没有显式指定 `--network`, 容器默认加入 Docker 预置的bridge 网络。
`--network <网络名>` 是在创建容器时让 Docker 把该容器的一个网络接口接入指定 Docker 网络。

`--network <网络名>`  可以指定加入的网络有下面几种：
> - `bridge`: 默认桥接网络，最常用。
> - `host` : 不创建独立网络，直接共用宿主机的网络栈。
> - `none`: 只保留环回接口，不接入网络。
> - `用户自定义网络`：如 Compose 创建的 `项目名_default`网络，或者是 `docker network create app_network` 。也可以让 docker run 启动容器加入它们。
> - `overlay`：


Docker通常自带三种网络，可以让容器启动时选择

```
bridge # 默认网络
host   # 直接使用宿主机网络
none   # 不提供网络
```

<font color="grape"><b>◼︎ bridge 网络</b></font>

bridge 网络是默认网络，任何单独启动的 docker 容器，没有特殊指定(--network参数)，都默认加入这个 bridge 网络。bridge 是 Docker Engine (dockerd) 启动时就创建好的默认桥接网络，不是具体容器启动时才创建。
默认 bridge 网络中的容器有以下特点：

>- 有自己的容器 IP，通常是 172.x.x.x 网段
>- 可通过容器 IP 互相访问
>- 默认不能像 Compose 自定义网络那样用容器名/服务名互相解析。

<font color="grape"><b>◼︎ host 网络</b></font>

不创建独立网络，直接共用宿主的网栈

<font color="grape"><b>◼︎ none 网络</b></font>

none 表示容器拥有独立网络命名空间，但不连接任何 Docker 网络。例如：

```bash
docker run --network none my-image
```

这种容器通常只有自己的回环接口：

```text
lo -> 127.0.0.1
```

>none 网络中的容器有以下特点
>- 不能访问互联网、宿主机或其他容器
>- 其他容器也不能通过网络访问它
>- 没有容器 IP，端口映射 和 Docker DNS 服务名解析
>- 容器内的进程，仍然可以通过 localhost 相互通信

这种容器常用于完全不需要网络的离线计算，敏感批处理或安全隔离任务。它只隔离网络，挂载目录，权限，以及其他容器配置仍需另外控制。

<font color="grape"><b>◼︎用户自定义网络 </b></font>

<font color="orange">在 Docker 的 bridge 网络中，容器之间不能用容器名或服务名相互解析</font>。如果希望自行启动的容器间也能按名称相互访问和解析，那么可以创建用户自定义网络。

```bash
docker network create my_net

docker run -d --name docker_1 --network my_net nginx
docker run -d --name docker_2 --network my_net redis
```

此时 `docker_1` 可以用 `docker_2` 作为主机名访问 Redis。
- `--name` 表示指定启动之后容器的名字，如果没有指定，Docker 会为容器起一个随机名字。容器还有一个唯一的容器ID，可以通过 Docker ps 查看。
- `--network` 表示指定容器要加入的网络，my_net 是我们自定义网络。
- 末尾 `nginx` / `redis` 表示我们创建容器时使用的镜像。
<font color="deeppink">注意： --name 后面的名字是容器的名字。末尾的名字是镜像名字。</font>

只有用户自定义网络，才提供通过容器名解析容器 IP 地址的服务(容器DNS服务，由 dockerd 提供)。<font color="orange">compose 创建的网络，本质上就是用户自定义网络</font>。compose 在启动编排的容器时，会自动为compose中的容器，生成一个自定义网络，这个compose中的容器都自动加入这个网络。

# 2. Docker 网络的 DNS 服务

Docker 中的自定义网络或 Compose 创建的网络(两者本质一样)，相互独立及隔离。但是，Docker 对每个网络内部提供了 DNS 服务。

<font color="#b48ff4"><b>DNS 地址</b></font>

DNS服务是 Docker Engine(dockerd) 网络子系统的一部分。Docker为每个<font color="orange">自定义网络</font>的<font color="orange">容器</font>配置了`nameserver`, `127.0.0.11`, 所有自定义网络中的容器访问的 DNS 服务都是 `127.0.0.11:53`, 容器发往这个地址的 DNS 查询由 Dokcer的嵌入式解析器处理。
假设 nginx 是 docker 自定义网络中的一个容器服务，它要访问 DNS 时：
```
nginx 容器内的应用
  └─ 查询 127.0.0.11:53
       └─ Docker Engine 的嵌入式 DNS 逻辑
            ├─ 查 Docker 网络中的服务/容器名称
            └─ 外部域名则转发给宿主机配置的上游 DNS
```

127.0.0.11 是每个容器各自网络命名空间里的回环地址，所以不同容器都能使用同一地址，而不会冲突。这里的环回地址，就是 Docker 在容器网络命名空间中提供的特殊 DNS 入口。

<font color="deppink">127.0.0.11 不是 dockerd 的地址</font>
>注意⚠️：127.0.0.1:53 是 Docker 在每个<font color="orange">容器网络命名空间</font>里提供的 “虚拟DNS 服务入口”。它不是dockerd 在容器网络中的 IP，也不是 dockerd 在宿主机上的地址和端口。<font color="deeppink">它就是Docker给每个容器自定义网络定义的一个虚拟DNS服务入口。Docker 会利用 NAT规则，把 127.0.011:53 的访问转发到 dockerd 实际监听的端口上，然后dockerd 的libnetwork模块会处理 DNS 服务。</font>

> 一个 Compose 项目默认创建一张用户自定义网络，未显式修改网络配置的服务容器会加入它。容器的 DNS 查询会发送到 `127.0.0.11` 的标准 DNS 端口 53；这是 Docker Engine（`dockerd`）网络子系统提供的嵌入式 DNS，而不是独立容器。它依据 Docker 维护的网络成员、服务名和网络别名信息，解析当前容器所加入网络中的服务名。

- `127.0.0.11` 是**容器视角**的回环地址；每个容器都有自己的 loopback，所以请求不会经过物理网络。
- 它不仅有“DNS 表”。查询 `vllm` 时优先从容器已加入的 Docker 网络中找服务；找不到时，通常转发给宿主机配置的外部 DNS。

<font color="#b48ff4"><b>DNS 服务</b></font>

一台 Docker 宿主机内通常只有一个 dockerd. 在典型 Linux Docker Engine + Compose 自定义网络场景下，嵌入式 DNS 的处理逻辑在 dockerd 中的 libnetwork 模块中。<font color="orange">不是单独的DNS容器或独立进程。</font> 
假设 Compose 默认创建的自定义网络中有三个容器 docker_1, docker_2, docker_3, 它们之间可以通过统一网络中的服务名互相解析，例如，vllm, nginx, redis 等等。通常的过程是

```
docker_1 / docker_2 / docker_3
  └─ DNS 查询 → 127.0.0.11 → 本机 dockerd 的嵌入式 DNS
```

所以一般情况下，容器需要解析自己网络中的其他容器名或服务名的时候，就会访问 dockerd 中的嵌入式 DNS 服务。
但是，以下情况不支持 dockerd 的 DNS 服务：
>1. 默认的 bridge 网络，不支持按容器名互相解析。
>2. docker容器显示配置了 `--dns` 或 Compose 中配置 `dns:` 这时容器会查询指定的DNS服务器
>3. `--network host`，容器会直接使用宿主机网络栈和 DNS 设置。

# 3.一个宿主机上的多个网络

<font color="#b48ff4"><b>独立隔离的多网络</b></font>

一个 Compose 项目默认会创建一张独立的用户自定义网络，网络的名称通常是 `<项目名>_default`。不同的 Compose 项目创建的自定义网络是相互独立，相互隔离的, 即使它们都在同一个物理宿主机上。例如：
```
compose 项目 A → app_a_default
  ├─ nginx
  └─ vllm

compose 项目 B → app_b_default
  ├─ redis
  └─ worker
```

默认情况下：
A 中的 `nginx` 能解析 `vllm`。B 中的 `worker` 能解析 `redis`
A 不能直接通过服务名解析 B 的 `redis`。B 也不能直接通过服务名解析 A 的 `vllm`
所以<font color="orange">app_a_default 和 app_b_default 是完全独立的网络</font>。但是并不是启动两个 compose 就一定会创建两张网络，也有可能一个compose启动后，加入另一个compose的网络。

<font color="#b48ff4"><b>DNS 名称空间按网络隔离</b></font>

Docker 的容器名，服务名和 network alias 都是网络级的作用域。不同网络可以有同名的服务。

>不同的网络虽然相互独立和隔离，但是<font color="orange">并不是每一个网络启动一个独立的 DNS 进程</font>。本质上，DNS的数据和解析范围按网络隔离，但底层由 dockerd(Docker daemon)/libnetwork 统一管理。一个物理宿主机上通常只有一个 dockerd 守护进程。这个宿主机上的所有 docker 网络都是由这个dockerd/libnetwork 模块管理dns服务，这些容器网络，共享同一个 dns 服务。

对于 Compose 创建的用户自定义网络，容器里会看到：

```
nameserver 127.0.0.11
```

上面已经讨论过，这是 Docker 内置的 DNS 服务解析入口。 127.0.0.11 位于各容器自己的网络命名空间，所以每个容器都可以使用相同的地址。 当容器访问 127.0.0.11 的时候，Docker 根据NAT规则，将访问请求转发到 dockerd 实际监听端口，由 dockerd/libnetwork 模块实际处理 dns 服务。
```
容器中的应用
    ↓ DNS 查询
127.0.0.11:53（容器自己的网络命名空间）
    ↓ Docker 设置的转发规则
dockerd/libnetwork 中的 DNS 处理逻辑
    ├── 容器/服务名 → Docker 自己直接回答
    └── 公网域名     → 转发给宿主机配置的上游 DNS
```

<font color="#45ce6e">dockerd 根据：</font>
> <font color="#45ce6e">① 发起查询的是哪个容器</font>
> <font color="#45ce6e">② 该容器加入了哪些网络</font>
> <font color="#45ce6e">③ 查询名称在哪些网络中注册</font>
<font color="#45ce6e">来决定返回什么样的结果</font>

我们可以把 Docker 中的网络和DNS服务总结为以下几个关键点：

> [!NOTE] Docker 网络与DNS服务
> - <font color="orange">不同的 Compose 默认创建的网络是隔离的。</font>
> - <font color="orange">DNS 名称记录是按网络隔离的。每个网络都有自己的 DNS 名称空间</font>
> - <font color="orange">不是每个网络启动独立的 DNS 服务，而是共享 dockerd/libnetwork 模块提供的 DNS 服务</font>
> - <font color="orange">所有网络使用同一个DNS，容器看到的都是127.0.0.11, 这是Docker提供的统DNS入口。</font>
> - <font color="orange">一个容器若加入两个网络，同一个内置解析入口可以解析它所加入的两个网络中的名称。</font>
> 
>```
>同一个 dockerd
>├── 容器 A → 127.0.0.11 → 只能看到 network-a 的记录
>├── 容器 B → 127.0.0.11 → 只能看到 network-b 的记录
>└── 容器 C → 127.0.0.11 → 加入了 A、B，可看到两个网络的记录
>```

<font color="#b48ff4"><b>多网络解析冲突</b></font>

如果同一个容器在两个网络中，它要解析的名字，在两个网络中又是同名的服务，这种情况就是多网络的解析冲突。
例如同一个容器同时加入 net_a 和 net_b, 且两个网络中都有名为 db 的服务：
```
app 同时加入 net_a、net_b
net_a 中：db → 172.20.0.3
net_b 中：db → 172.21.0.5
```
此时，直接访问db, dockerd 的 DNS 按内部网络端点顺序找到第一个匹配项并返回其 IP；这个顺序不应作为应用依赖，因此不能保证你总是访问期望的那个 db。
我们应该<font color="orange">显示消除歧义</font>, 常用两种方式：

一、Docker 支持 "容器/服务名.网络名" 形式，能明确指定要在哪张指定网络中查找。
```
db.net_a
db.net_b
```

二、给不同网络中的服务设置不同别名，然后应用始终按别名访问
```
orders-db
billing-db
```

<font color="deeppink">结论：一个容器可以加入多个网络，但应避免在这些网络中使用相同的服务名，确实重名时，使用 服务名.网络名 或者不同的网络别名。</font>

# 4. 网络配置 networks

一个 compose 项目默认会创建一个自定义网络，默认网络的名字为 `<项目名>_default`，这个 compose 项目中的各个服务，如果没有特别设置，启动起来之后，就加入这个网络。但是这些服务并不是一定要加入默认网络，也可以通过配置加入其他网络。

<font color="#b48ff4"><b>顶层配置 networks </b></font>

在跟 services 同层的地方，也就是顶层，我们可以配置 networks, 例如：

```yaml
name: myapp               # Compose 项目名 

services:
  web:
    image: nginx
    networks:
      - frontend
      - backend

  db:
    image: mysql
    networks:
      - backend

  cache:
    image: redis
    # 未指定，自动加入 default

networks:
  frontend:              # Compose 文件内的网络标识
	  name: pub_net      # Docker 中的实际网络名
  backend:               # Docker 中实际网络名默认为 myapp_backend
```

顶层 networks, 下面配置的 `frontend`, `backend`, 是 Compose 内部的网络标识符，也就是在 Compose 内部可以使用它们作为两个网络的名字，但是在 Compose 外，Docker 中使用的网络名就是在标识符下的 name 定义的。比如上面 <font color="orange">frontend是一个网络的标识符，但是在 docker 系统中，它使用的网络名字就是 pub_net。</font> <font color="lightblue">对于没有定义 name 属性的网络标识符，它在 docker 中的实际网络名，通常就定义为</font> `项目名_网络标识符`。

顶层 `networks` 本质上是，声明这个 Compose 项目允许服务使用哪些网络，以及这些网络如何获得。networks 先定义出网络的内部标识，然后通过 name 配置，要获该取网络要使用的外部名称。如果这个网络不存在，Compose就会创建，实际网络名，就是name配置或默认`项目名_网络标识符`

如果明确，一定要使用已有网络，那么可以使用
```yaml
services:
  app:
    networks:
      - backend

networks:
  backend:
    external: true
    name: existing_backend
```

`existing_backend` 必须要提前存在，否则 Compose 就会报错。这时，必须明确设置 `externel: true` 表示一定要使用外部网络。
如果外部网络不存在，Compose不负责创建和删除它。默认情况下，docker compose down 的时候，会把这个 compose 创建的网络一并删除。如果使用的是已经存在，且不由这个 Compose 管理的网络，compose 就只管使用它，不负责创建或删除。<font color="orange">这才是 external 的真正含义。</font>

<font color="#b48ff4"><b>顶层 networks 中配置 default</b></font>

在顶层 networks 设置中，也可以设置 default 网络。例如
```yaml
services:
  web:
    image: nginx
    # 没写 networks

networks:
  default:
    name: company_network
```

这样就把 compose 的默认网络设置成了 company_network, 而不是为默认网络取名 `项目名_default`。但是所有没配置 networks 的服务，还是默认加入这个网络，因为它们认的是 default的内部标识符，和外部实际网络名无关。
另外，如果配置了顶层networks，且没配 default, 那么如果compose中，有没配networks的服务，那么compose依然会创建默认网络，没配 networks的服务依然加入这个默认网络。

<font color="#b48ff4"><b>服务中配置 networks </b></font>

如果在服务中配置 networks, 就表示该服务会加入哪个网络，例如：

```yaml
services:
  app:
    networks:
      - backend

networks:
  backend:
```

它表示 app 这个服务起来以后，加入 backend 这个内部标识符的网络。

<font color="#b48ff4"><b>总结</b></font>

>- 顶层 `networks:`：声明、配置可用网络
>- 服务内的 `networks:`：选择加入哪些网络
>- 普通网络声明：由 Compose 创建和管理
>- `external: true`：使用 Compose 外部已有的网络
>- `default:` 是特殊网络，顶层 `networks:` 不一定都是“额外网络”，也可以用来配置默认网络

# 5. 服务/容器的网络 DNS 别名(alias)

服务/容器启动起来以后，要加入网络，我们可以指定该服务/容器在网络中的 DNS 别名 alias

```yaml
services:
  vllm-main:
    networks:
      default:                # 加入 default 网络
        aliases:
          - vllm-active       # 在这个网络内增加 DNS 别名
```

这里 vllm-main 是一个服务，它配置了networks，它默认会加入 default 网络。在这个网络中，该服务/容器有一个 DNS 别名叫做 vllm-active。这是节点在网络中的可被解析的 DNS 别名。

典型的用法是，多个容器中，仅有一个会在网络中的时候，可以让这些容器使用同一个 alias， 这样其他部分就可以使用这个 alias 进行 DNS 解析。这个容器更换成其他容器，对网络中的其他部分是透明的，它们不需要做任何修改，只要这些更换的容器使用同一个 alias 就可以了。
比如：
```yaml
services:
  vllm-main:
    networks:
      default:
        aliases:
          - vllm-active

  vllm-backup:
    networks:
      default:
        aliases:
          - vllm-active

networks:
  default:
    name: heteroserve-network
```

主备模型，默认都会加入 default 网络， 但是，同一时间只有一个启动，所以把它们的 alias 都设置成 vllm-active, 这样的话网络其他部分就可以直接访问 vllm-active, 不用管它究竟是主模型还是备用模型。