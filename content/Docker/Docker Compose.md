
docker compose 是一个 docker 容器的编排工具，主要通过 `compose.yaml` 文件来配置容器的具体编排，以前，配置文件叫做 `docker-compose.yml`。

docker compose 中的一些标准字段，受 Docker Compose Specification 规定的约束
比如经常使用的<font color="orange"> Docker Compose Specification 规定的标准字段</font>，部分如下：
```yaml
name
services
build
image
init
restart
ipc
stop_grace_period
environment
volumes
healthcheck
ulimits
logging
profiles
ports
command
deploy
```

# 以 vllm 服务的 compose为例

下面的讨论，以一个 vllm 的 docker compose 编排容器的 compose.yml 文件为例。文件详见[vllm-compose-file](Docker/Assets/vllm-compose-file)
# 0. compose 自定义扩展配置

`x-vllm-common: &vllm-common`  
这是 compose 的扩展配置名称。注意它不会启动任何服务，它只是一个配置。
`x-` 开头表示它不是一个真正的服务。Docker Compose 不会为它创建容器。作用是可以用来存放可以复用的公共配置。
`&vllm-common` 这是 YAML 的“锚点”，把下面的整段配置命名为 `vllm-common`. 后面可以使用

```yaml
vllm-main:
	<<: *vllm-common
```

其中，\*vllm-common 表示引用这个锚点，<<: 表示把锚点中的配置合并进当前的服务。
比如，使用
```yaml
vllm-main:
	<<: *vllm-common
vllm-backup:
	<<: *vllm-common
```
这样公共的镜像、环境变量、挂载、健康检查和日志配置只需要写一次。
注意，前面的 `x-vllm-common` 叫做配置字段名，`&vllm-common` 才是真正的YAML锚点名。
配置字段名，只是起到一个说明作用，只是一个“存放公共配置的自定义扩展字段”。
<font color="orange">自定义compose扩展字段必须以 x- 开头</font>.不加 x-, compose 会把它当做未知的顶层标准字段报错。

# 1. services

compose.yml 下有顶级的配置项 services，表示配置服务，一个 service 就对应一个docker 容器的服务进程。
```yaml
services:
  app:
    image: my-app

  db:
    image: postgres

  adminer:
    image: adminer
    profiles:
      - debug
```
比如，这里就有三个容器服务，app, db, 和 adminer

术语上，已经启动了的，叫做 docker 容器，没有启动的叫服务 service。

services 下面的每一个子配置项都是一个具体的服务
# 2. profiles

每个service下面有一个子配置项，叫做 profiles。profiles 是 docker compose 的可选服务分组功能，即用来控制哪些服务需要启动。

执行 `docker compose up`, 默认只会启动没有配置 profiles 的 services. 上面的例子中，就只会启动app 和 db. 配置了 profiles 之后，我们要启动这个服务，就要通过指定 profile 来启动。比如
`docker compose --profile debug up` 除了启动没有配置 profiles 的服务外，还会启动 adminer服务，因为它的 profiles 是 debug。

profiles 常用于，开发调试工具，测试服务 mock server，测试数据库，监控组件，不同环境的附加服务等，这些服务是根据需要启动的，而不是每次都默认启动。

>注意⚠️：没写 profiles 的服务始终启用，写了 profiles 的服务，只有对应 profile 被激活时，才会启动。配置了 `profiles` 的服务默认不参与 Compose 操作；激活对应 profile，或者直接点名该服务时，它才会参与。

profiles 配置项下面的每一项都是一个 profile 名字，可以配置多个。比如
```yaml
services:
  adminer:
    image: adminer
    profiles:
      - debug
      - dev
```
这表示 adminer 属于 debug 和 dev 两个 profile。激活其中一个，它都会启动，`docker compose --profile debug up` 或者 `docker compose --profile dev up` 都会让 adminer 启动。配置上也可以写成：
```yaml
services:
  adminer:
    image: adminer
    profiles: ["debug", "dev"]
```
注意：多个 profile 是“或”的关系——匹配任意一个即可启用该服务，不要求全部同时激活。
从术语上，debug 和 dev 是两个不同的profile。adminer 这个服务属于两个profiles，一个是 debug, 一个是 dev。


> [!NOTE] docker compose 启动
> `docker compose --profile xxx up -d`, 启动 compose.yml 文件中配置的 profile 分组为 xxx 的服务，以及所有没有配置任何 profiles 的服务。`-d` 是  --detach，表示在后台执行。

# 3. container_name

container_name 和 profiles 一样也都是 docker compose specification 中规定的标准字段。它表示的是容器的名字。在一个宿主机上，一个docker容器的名字，必须是唯一的。

如果不设置 container_name, 就会自动生成类似 `heteroserve-vllm-main-1`的名字，其中heteroserve是开始设定的 compose.yml 的名字 `name: heteroserve`, `vllm-main` 是这个service的名字，`1` 是给的一个编号。如果设定了 `container_name: heteroserve-main` 就会把容器名字固定为 `heteroserve-main`。
而 <font color="orange">profiles 是指这个容器所属服务分组，它决定的是这个服务是否参与本次docker容器服务的编排启动</font>。比如 `docker compose --profile main up` 才会启动属于 main profile的服务。所以：

```
container_name → 容器叫什么
profiles       → 该服务什么时候被启用
```

container_name 有两个限制：
- 同一个 Docker 主机上名称必须唯一
- 设置后该服务不能扩展为多个副本
通常不必设置 container_name， Compose 可以通过服务名管理容器。

# 3.  build

build表示，这个服务的镜像需要根据指定的 Dockerfile 在本地构建。根据提供的 Dockerfile 构建出容器进程来提供服务。比如：
```yaml
build:
  context: .
  dockerfile: deploy/Dockerfile
  args:
    VLLM_VERSION: "${VLLM_VERSION:-v0.21.0}"
```

`context .` 表示构建上下文为当前目录。Dockerfile 中的 COPY，ADD等命令，只能访问构建上下文目录内的文件。
`dockerfile: deploy/Dockerfile`, 指定 Dockerfile 的位置，这个位置是相对 context 的，所以，实际位置是 `./deploy/Dockerfile`。
`args`:  向 Dockerfile 传递的构建参数。`VLLM_VERSION: "${VLLM_VERSION:-v0.21.0}"` 对应 Dockerfile中的`ARG VLLM_VERSION` 这个变量。这个 args 是在 build 下面的，所以它对应的是构建时期使用的参数 ARG。

执行：
```
docker compose --profile main up -d --build
```

过程是：
```
读取 build 配置
    ↓
读取 deploy/Dockerfile
    ↓
基于 vllm/vllm-openai:v0.21.0 构建
    ↓
命名为 heteroserve/vllm-openai:v0.21.0-sm75
    ↓
使用该镜像创建并启动容器
```
`build` 只负责构建镜像，本身不会启动容器。启动由 `docker compose up` 完成。

说明：
理论上我们可以通过 Dockerfile, 指定镜像的构建过程，再使用 docker build 完成镜像的构建。再使用 docker run 启动镜像进程提供服务，这是手动起docker的过程。
在 docker compose, 我们直接把 Dockerfile 给compose, 在 docker compose up 的时候，它就会先调用 docker build, 根据 Dockerfile 以及我们配置的其他参数，构建出这个镜像。然后再使用这个镜像启动 docker 容器。这个过程完全可以用手动分开执行，但是 docker compose 帮助我们把这个过程全部一气呵成了。

 <font color="#b48ff4">--build 参数</font>

构建好的镜像，仍然是放在默认的本地缓存镜像目录中。如果再次执行 `docker compose up` 的时候。如果不带 `--build` 参数，即 `docker compose up -d` compose 会直接使用已有的镜像和容器，不再执行镜像构建。如果第二次还是带了 `--build`，compose 仍然会请求执行构建，但docker build 工具会检查，如果没有变化就使用已有的镜像和容器启动。
- 不带 `--build`：通常完全不进入构建流程，直接使用已有镜像。
- 带 `--build`：会进入构建检查，但没有变化时直接复用缓存，不会重新执行耗时构建。
镜像通常只在第一次真正计算和生成；以后不带 `--build` 时直接复用。即使带了 `--build`，只要 Dockerfile和构建上下文没有变化，BuildKit通常也只做缓存检查，不会重新执行构建步骤。

# 4. 变量设置 

我们采用 `VLLM_VERSION: "${VLLM_VERSION:-v0.21.0}"` 设置参数。前面一个 VLLM_VERSION 和后面一个$中的VLLM_VERSION表示的含义完全不同。

后面的 VLLM_VERSION 表示从宿主机上读取名为 VLLM_VERSION 的变量值，如果没有这个变量或者这个变量的值为空，那么就使用值 v0.21.0。然后将这个值赋给 Dockerfile 中名为 VLLM_VERSION(前面一个) 的参数。

这两个名字可以不同， 例如 `VLLM_VERSION: "${IMAGE_VERSION:-v0.21.0}"` 表示从宿主机上读取变量 IMAGE_VERSION, 如果没有这个变量或者该变量值为空，就使用v0.21.0这个值，把它传递给Dockerfile中的 VLLM_VERSION 这个参数。

所以，`${IMAGE_VERSION:-v0.21.0}` 的真正含义是获取宿主机上定义的 IMAGE_VESION 这个变量值，如果没有或为空，就采用默认值 v0.21.0

# 5. image

这里 `image: "heteroserve/vllm-openai:${VLLM_VERSION:-v0.21.0}-sm75"` 表示build好之后的镜像打标签。
- `build`：规定如何构建镜像。
- `image`：规定构建完成后镜像叫什么名字

# 6. init

`init: true`： 它表示在容器中加入一个轻量级的init进程，让它作为容器的根进程。没有配置 `init: true` 时，容器内通常是 ：
```
PID 1  vllm serve                             
        ├── worker 1
        ├── worker 2
        └── worker 3
```
启动了 `init:true` 时，容器内通常是：
```
PID 1  docker-init/tini
        └── vllm serve
              ├── worker 1
              ├── worker 2
              └── worker 3

```

这个轻量级的 init 进程主要负责两件事：
第一：转发停止信号。比如 Docker 向容器 PID1发送 SIGTERM，init 会把信号正确转发给 vllm serve。使得vllm 及其 worker 能够正常退出。
第二：回收僵尸进程。vLLM 使用 PP/TP 或多进程 worker 时会创建子进程。如果某个子进程退出，但父进程没有及时读取其退出状态，它可能变成僵尸进程。init 进程会负责回收这些进程。

需要注意，`init: true`：
- 不是用来初始化模型。
- 不是用来初始化数据库。
- 不是 `systemd`。
- 不负责健康检查。
- 不负责启动多个 Linux 服务。
- 不会替代 `vllm serve`。
它只是一个非常小的进程管理器。

# 7. restart

`restart: unless-stopped` 配置的是容器的自动重启策略，含义是：只要容器不是被用户明确停止，Docker 就会在它异常退出，或者 Docker 服务重启后自动启动它。

当 healthcheck 变成 unhealthy 本身不会触发 `restart: unless-stopped`, 只有容器主进程真正退出，重启策略才会生效。所以，这个策略不能仅凭健康检查失败就自动重启容器，若要根据 unhealthy 自动恢复，还需要额外的监控和编排。

# 8. IPC

`ipc: host` 表示容器直接使用宿主机的 IPC namespace, 而不再创建要个独立的 IPC namespace.
IPC, Inter-Process Communication, 即进程间通信。包括进程间的，共享内存，信号量，消息队列 /dev/shm 等等。
默认情况下，Docker容器拥有独立的 IPC namespace, 且 /dev/shm 通常比较小，64MiB.
`/dev/shm` shared memory, 共享内存系统，看起来像是文件系统，但是它是在内存中。它是Linux为多个进程提供的一块高速，临时，可共同访问的内存空间。

比如Docker中的vllm, pytorch服务，可能有多进程共享内存交换数据
```
vLLM 主进程
    ├── PP Worker 0
    ├── PP Worker 1
    └── PP Worker 2
          ↕
     共享内存通信
```
`ipc: host` 容器可以使用宿主机较大的 `/dev/shm`。
注意⚠️ 它不表示：
- 使用宿主机网络；那是 `network_mode: host`。
- 容器可以随意访问全部宿主机内存。
- 自动开启 GPU P2P。
- 自动解决所有 NCCL 问题。
它只取消 IPC namespace 隔离。

# 9. stop_grace_period

`stop_grace_period: 2m` 表示 Compose正常停止容器时，在发送终止信号后，最多等待两分钟让程序自行退出。2分钟后仍未退出，则发送信号 SIGKILL 强制终止。

# 10. environment

environment 中提供一些 `变量名:变量值`. 表示在 Docker 容器运行时，注入环境变量。如果Dockerfile中已经定义了同名 ENV，运行容器时，compose中的值会覆盖镜像默认值。如果Dockerfile 中没有定义同名 ENV，那么就直接添加。容器中的 vllm 仍然能够读取它。

因此 `environment` 可以：
- 覆盖镜像中已有的环境变量。
- 创建镜像中原本没有的新环境变量。

Compose 的 `environment` 只影响创建出来的容器，不会修改原来的 Docker镜像。
总结：`environment` 为容器设置运行时环境变量；它可以覆盖 Dockerfile中 `ENV` 定义的默认值，也可以添加 Dockerfile中从未定义过的新变量。

# 11.    /.env 文件

这里的 `/.env` 不是指宿主机根目录下的文件，而是项目目录中的一个隐藏文件。项目结构例如
```
HeteroServe-完整规划方案-v3.0/
├── .env
├── compose.yml
└── deploy/
    └── Dockerfile
```

`.env` 是一个普通文本文件，用来给 Docker Compose提供变量值。`.env` 通常是和 `compose.yml` 放在一起的宿主机配置文件，Docker Compose读取它来替换 `${...}` 变量, 它是compose.yml中 ${...} 读取变量值的来源；它不是 Docker镜像中的文件，也不会自动把所有变量注入容器。

对于 `compose.yml` 中, 如果使用变量 `${PORT:-8000}`, 处理顺序是
1. 优先读取宿主机 Shell 环境变量 `PORT`
2. 没有时读取 `.env` 中的 `PORT`
3. 如果没有定义或者值为空，则使用默认值 `8000`
因此 `.env` 不是必须的。只要 Compose 中相关变量都写了合理的默认值，文件不存在也可以正常运行。`.env` 主要用于集中修改默认配置，避免直接改动 `compose.yml`。

# 12. HF_Home 与 HF_HUB_OFFLINE

HF_Home 与 HF_HUB_OFFLINE 这两个配置关于 Huggingface 的。

`HF_Home` 配置的是一个本地的目录，它指定的是 HuggingFace的缓存，Token等存放的位置。当前的 compose 会把我们指定的目录挂载到容器中的 hf-cache 卷，因此重建容器后，缓存仍然存在。

`HF_HUB_OFFLINE=1`：表示禁止对 Hugging Face Hub 网络的请求，只使用本地文件或缓存。文件不完整时直接报错。它不代表容器完全断网，只限制 Hugging Face 的访问。

# 13. volumes

volumes 用来把容器外部的存储挂载到容器内部，使得数据可以持久化，而不依赖于容器本身的生命周期。有两种挂载形式，一种是 bind， 一种是 volume。
bind:
```yaml
volumes:
  - type: bind
    source: "${MODEL_ROOT:-/opt/models}"
    target: /models
    read_only: true
```

`source` 表示源, 具体的路径通过 `${MODEL_ROOT}`变量读取，变量没有或者值为空的情况下，就使用默认值 `/opt/models`。<font color="orange">source 是宿主机上的文件或目录路径</font>。
目标 `target` 是挂载到 Docker 容器内的 `/models` 目录下，访问权限是只读。这种方式，是要将宿主机中的某个目录绑定到 Docker 容器内的某个目录下，是为了读取宿主机上的某些文件或数据。

volume
```yaml
volumes
  - type: volume
    source: hf-cache
    target: /root/.cache/huggingface
```

这是Docker容器内部管理的命名卷。这里卷名是 hf-cache, 这个卷的挂载位置是 /root/.cache/huggingface. 用于持久化保存 huggingface 的缓存。
这个路径是容器内的路径。容器中的 Hugging Face 程序向 `/root/.cache/huggingface` 写入数据，实际上会保存到 `hf-cache` 卷中。因此，容器被删除后，缓存数据仍能保留。

这里的`source: hf-cache` 的命名卷，对应的是宿主机上磁盘空间中的由Docker管理的部分空间，就是用来存放Docker容器中指定的命名卷。所以，它实际上是 Docker 容器在宿主机磁盘上的一个空间。所以，容器停止甚至删除的时候，这个容器在磁盘上的命名卷，依然存在。除非使用 `docker compose down -v` 才会删除命名卷，或者直接使用 `docker volume rm <卷名>` 显示的删除卷。这里删除卷，会清空 huggingface 缓存，无法恢复。

`compose.yml`文件末尾通常还声明命名卷，如：
```yml
volumes:
  hf-cache:
```

<font color="deeppink">命名卷通常还需要载顶层 volumes: 中声明，而绑定挂载不需要</font>

<font color="#b48ff4"><b>volumes的短写法</b></font>

我们经常会看到 volumes 的短写法，

<font color="orange">绑定挂载,bind</font>

```yaml
volumes: ["./deploy/nginx.conf:/etc/nginx/nginx,conf:ro"]
```

等价于：

```yaml
volumes:
  - type: bind
    source: ./deploy/nginx.conf
    target: /etc/nginx/nginx.conf
    read_only: true
```

所以，绑定挂载短写法的格式是
```yaml
volumes: ["宿主机路径 : 容器内路径 : 挂载模式"]
```

<font color="orange">命名卷挂载, volume</font>

```yaml
volomes: [grafana-data:/var/lib/grafana]
```

等价于

```yaml
volumes:
  - type: volume
    source: grafana-data
    target: /var/lib/grafana
```

命名卷挂载短写法的格式是
```yaml
volumes: ["宿主机路径 : 容器内路径 : 挂载模式"]
```

命名卷挂载短写法格式是：

```yaml
volumes: ["命名卷名字 : 容器内路径"]
```

<font color="deeppink">冒号前以 /、./ 或 ../ 开头，通常表示宿主机路径，对应的挂载就是绑定挂载。如果是一个普通名称，就表示命名卷的名字，对应的挂载是命名卷挂载。</font>

# 14 healthcheck

`healthcheck` 是 Docker 对容器内服务进行的定期健康检查。它不仅可以检查 “容器进程是否还活着”，还可以检查 vLLM 服务是否真正能够响应。比如：

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
  interval: 30s
  timeout: 10s
  retries: 5
  start_period: 10m
```

>`test` : 是容器内执行的检查命令。
`interval`: 每隔 30 秒检查一次。
`timeout`: 单次检查超过10秒视为失败
`retries`: 连续失败 5 次后标记为 unhealthy
`start_period`: 启动后的10分钟，期间的测试失败不会累计为正式失败

容器有三种健康状态  <font color="orange">starting -> healthy -> unhealthy </font>
`healthcheck` 本身只负责<font color="deeppink">监测和标记状态，不会自动重启不健康的容器</font>。

# 15. ulimits

ulimits 用来设置容器内进程的 Linux 资源上限。例如：
```yaml
ulimits:
  memlock: -1
  stack: 67108864
```

- `memlock:-1` 允许进程锁定不限量的系统内存，避免被交换到磁盘。CUDA、NCCL组件可能需要锁内存页。
-  `stack: 67108864` 将进程栈空间上限设置为 67108864 字节，即 64 MiB，避免复杂计算出现栈空间不足。
它们只是设置 "最大允许值", 并不会预先占用这些内存，也不涉及GPU显存。最终仍然受宿主机物理资源和内核限制。

# 16. logging

logging 配置 Docker 容器如何收集和保存容器主进程输出到 `stdout`, `stderr` 的日志。例如：
```yaml
logging:
  driver: json-file
  options:
    max-size: 100m
    max-file: "3"
```

>`driver: json-file` : Docker 将日志以 JSON格式保存在宿主机磁盘上。
>`max-size: 100m`：单个日志文件最大 100MB
>`max-file: "3"`: 最多保留3个轮转日志文件。

所以，每个容器的日志文件最多 300MB, 避免日志无限增长。`docker logs -f <服务名>` 或者 `docker compose logs -f <服务名>` 来查看日志。
注意⚠️：前面的 `VLLM_LOGGING_LEVEL` 决定 vLLM 输出哪些日志。logging 决定 Docker 如何保存这些日志。

# 17. ports

`ports` 表示<font color="orange">“宿主机端口 → 容器端口”</font>的映射关系。客户端访问宿主机:8000 → Docker转发 → 容器内vLLM:8000
两个容器内部可以都使用 8000，只要宿主机端口不同，比如：
```yaml
# 主模型
ports:
  - "8000:8000"
# 备用模型
ports:
  - "8001:8000"
```

访问宿主机 8000 时，Docker 会转发到主模型的容器8000端口，主模型容器监听8000端口，然后进行处理。访问宿主机 8001 时， Docker 会转发到备用模型容器的8000端口，备用模型容器监听 8000 端口，然后进行处理。这两者并不冲突。

# 18. deploy

`deploy` 用来配置服务部署时的资源，实例数量，更新策略等。例如我们用它来配置该服务可以使用哪张 GPU？
```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          device_ids: ["0"]
          capabilities: [gpu]
```

>  `resources` 表示服务需要的资源
>  `reservations` 表示预留或指定资源
>  `devices` 表示硬件设备
>  `driver: nvidia` 使用 NVIDIA 驱动
>  `device_ids: ["0"]` 将宿主机的第0号GPU提供给容器
>  `capabilities: [gpu]`: 要求该设备具有GPU能力，这是必填项
>  * `count:2`: 还有可能有配置 count, 它表示使用宿主机的几张GPU，这里使用了`device_ids`，就需要使用 count 了。  

这里并不是预留某个大小的显存，而是让容器获得指定 GPU 的访问权限，其他容器仍然可以同时使用这一张卡。
现代 docker compose up 支持这样的 GPU 配置，但 deploy 中某些面向集群的选项可能被普通compose忽略。


> [!NOTE] Deploy 规范
> <font color="orange">在 compose.yml 配置文件中，Deploy 要遵循单独的 Compose Deploy Specification, 不能随意编写字段。</font>

按照 Compose Deploy Specification。deploy 的常见结构为：
```yml
deploy:
  mode: replicated
  replicas: 2

  resources:
    limits:
      cpus: "4"
      memory: 16G
    reservations:
      devices:
        - driver: nvidia
          device_ids: ["0"]
          capabilities: [gpu]

  restart_policy:
  update_config:
  rollback_config:
  placement:
```

具体的 Compose Deploy Specification, 需要查询官方文档。
使用 `deploy...devices`的方式，<font color="orange">是当前 compose 使用的标准配置 GPU 的方式</font>。
# 19. docker compose exec 

`docker compose exec` 用来在已经运行的服务容器内执行一条额外的命令，不会新建容器，也不会替换主程序。比如

```bash
docker compose exec vllm-main nvidia-smi
```

表示在 vllm-main 容器内执行 nvidia-smi. 常用的方式为:

```bash
# 进入容器的交互式 Shell
docker compose exec vllm-main sh

# 查看容器内环境变量
docker compose exec vllm-main env

# 查看容器内模型目录
docker compose exec vllm-main ls -lah /models
```

它和 docker 命令的区别是：

```bash
docker exec <容器名> ...          需要知道实际容器名
docker compose exec <服务名> ...  使用 compose.yml 中的服务名
```

所以，即使没有设置 container_name, 也能通过服务名，让容器执行命令。比如

```bash
docker compose exec vllm-main sh
```

前提是该服务容器已经运行。它默认带交互终端，命令执行结束之后，这个额外进程也会结束，vLLM服务的容器则继续进行。

# 20. runtime

`runtime: runc` 字段是 Docker Compose Specification 的标准字段。它用来指定，Docker 应该使用哪一种容器运行时(构建容器的二进制可执行程序)来创建和启动该服务的容器。
<font color="orange">Docker 默认使用 runc，它负责把镜像进程启动为隔离的 Linux 容器进程</font>。它不是运行环境，而是更底层的容器启动的实现。

对于 Nvidia GPU，旧配置中常会看到
```
runtime: nvidia
```
意思是使用已在 Docker 守护进程 dockerd 中注册的 nvidia-container-runtime, 它会在容器启动前注入GPU设备，驱动库等。具体的关于 nvidia-container-runtime 的注入机制，参考 [Nvidia-Container-toolkit](Docker/Nvidia-Container-toolkit)

当前的 Compose 一般使用标准的 `deploy...devices` 的 GPU 配置方式，通常无需再写 `runtime: nvidia` 
<font color="deppink">宿主机正确安装并配置 Nvidia-Containter-Toolkit 才是前提。</font>

如果同时使用 `runtime: nvidia` 和 `deploy...devices` 也不会造成冲突，两者同时设置，通常可以正常工作。比如：
```yaml
runtime: nvidia
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          device_ids: ["0", "1"]
          capabilities: [gpu]
```
- `runtime: nvidia` 指定使用 Nvidia 容器运行时，即 nvidia-container-runtime 启动容器。
- `deploy...devices`: 指定容器使用宿主的哪些 GPU 设备，这里是 0 和 1。
它们不冲突，但对当前方案而言通常是**重复且不必要**的：Docker 官方的 Compose GPU 配置示例只使用 `deploy.resources.reservations.devices`，不需要额外声明 `runtime: nvidia`

# 21. depends_on

`depends_on` 用来声明服务之间的依赖关系，主要控制启动与停止顺序。它是 Docker Compose Specification 的标准服务字段。例如：
```yaml
services:
  gateway:
    depends_on:
      vllm-main:
        condition: service_healthy
  vllm-main:
    healthcheck:
      ...
```
含义是，先启动 vllm-main 这个服务，等待它的 healthcheck 变为 healthy 之后，再启动 gateway. gateway 这个服务的启动，是依赖于 vllm-main 这个服务启动完成之后，并且处于 healthy 的前提条件。
这样设置之后，Compose在停止服务的时候，会先停止 gateway, 再停止被依赖的 vllm-main。

# 22. cap_add

`cap_add` 用来给容器增加指定的Linux 内核权限。cap_add 实际上是 Linux capabilities addition。容器即使以 `root` 用户运行，默认也不会拥有全部内核特权，Docker会限制它可执行的敏感操作。`cap_add` 按需补充某一种权限：比如：
```yaml
cap_add:
  - IPC_LOCK
```
常见的 `cap_add` 的配置包括
- `IPC_LOCK`：允许锁定内存页面，常与 `ulimits: { memlock: -1 }` 配合
- `NET_ADMIN`：允许修改网络配置、路由、iptables 等
- `SYS_ADMIN`：权限极大，涉及挂载、命名空间等操作，应谨慎授予
- `ALL`：添加全部 capabilities，通常不建议
它是 Compose Specification 标准服务字段；相反的字段是 `cap_drop`，用于移除默认拥有权限。
<font color="orange">一般不建议额外添加 cap_add</font>。
