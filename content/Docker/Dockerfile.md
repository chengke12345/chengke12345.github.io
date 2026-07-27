<font color="orange">容器化(containerization)，就是将应用程序及其运行环境打包成镜像image，然后用镜像生成容器，在容器中运行应用程序的过程</font>。


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

一般来说，我们会在项目的根目录下，创建一个叫 Dockerfile 的文件，在这个文件中写入构建镜像所需要的各种指令之后，Docker 在执行 `docker build <dir>`命令时就会根据这个 Dockerfile 文件来构建一个镜像。有了镜像之后，我们就可以创建容器，在容器中运行应用程序。

一个典型的 Dockerfile 大概长这样

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

Dockerfile 可以看做是 Docker 的配置文件，Docker通过读取和执行它，来知道该如何构建我们需要的镜像。通常我们使用 `docker build .` 来执行构建，执行构建时的目录叫做 build context, 这里是当前目录`.` 

**最终镜像里有什么：**

- <font color="orange">基础镜像的全部内容(FROM 那一层)</font>: 比如`python:3.11-slim` 自带的 Debian 用户空间 + Python 解释器 + pip + 必要的库。这通常是镜像体积的大头。这是基础镜像，我们在基础镜像的基础上，再逐步增加我们自己的环境内容。
- <font color="orange">WORKDIR /app</font>: 将工作目录设为 /app，这是镜像中操作系统系统下的目录。如果镜像中的操作系统没有这个目录，就创建它，并设置它为当前工作目录。
- <font color="orange">COPY/ADD 进来的文件</font>——只有显式 COPY 的才会进，build context 里其他文件不会自动进去。
  这里把 requirements.txt 文件先拷贝进去，是为了利用Docker缓存，如果这个文件内容没变，pip install 这一步就不用重复执行。
- <font color="orange">COPY . . </font>, 这一步是把本地目录复制到镜像中的目录，这里两个都是当前目录，第一个`.` 是本地的当前目录，第二个`.` 是镜像中的当前目录，即 /app
- <font color="orange">RUN 指令的产物</font>——比如 `pip install` 装的包、`apt-get install` 装的软件，都会沉淀到镜像里。pip install -r 表示 --requirement, 读取后面文本内容，requirements.txt 的内容，并安装列出来的依赖包
- <font color="orange">元数据</font>——ENV、EXPOSE、CMD、ENTRYPOINT、WORKDIR、LABEL 这些不是文件，但作为镜像配置存在。
注意⚠️，区分④ 和 ⑧， RUN 是 build 完了之后，要执行的命令，命令的效果会沉淀到镜像中。CMD是在使用镜像启动容器时，执行的命令，通常为启动要跑的服务或者应用。

# CMD & ENTRYPOINT

CMD 通常用于传递入口程序的参数，而不是直接给入口程序，如果像上面一样直接给入口程序,例如 启动 vllm 的 docker。 假设docker的名字叫 my-docker : 

```dockerfile
CMD ["python", "-m", "vllm.entrypoints.openai.api_server"]
```

执行 `docker run my-vllm` 就会启动 vLLM. 但是如果执行 `docker run my-vllm --port 9000`，参数就会把整个命令覆盖掉。Docker 会尝试执行 `--port 9000`. 即<font color="deeppink">它会把整个CMD命令覆盖掉，docker 解读为，启动 my-vllm容器, 启动命令为 --port 9000</font>。

更典型的做法是，我们是在 Dockerfile 里配置 ENTRYPOINT， 用CMD传递参数，例如：

```dockerfile
ENTRYPOINT ["python", "-m", "vllm.entrypoints.openai.api_server"]
CMD ["--host", "0.0.0.0", "--port", "8000"]
```

这样， `python -m, vllm.entrypoints.openai.api_server` 作为入口程序或主程序，CMD提供默认参数。我们构建 docker 时，如果给定了参数，它会整体覆盖 CMD 的内容。例如

```bash
docker run my-vllm --model Qwen/Qwen3-32B-AWQ --port 9000 
# 参数整体覆盖掉 CMD 内容，执行如下
python -m, vllm.entrypoints.openai.api_server --model Qwen/Qwen3-32B-AWQ --port 9000 # CMD 内容被整体替换，原来的 --host 0.0.0.0 不会被保留，--port 8000 整体替换为 9000
```

# CMD & command

整体上，docker compose 编排中在 `compose.yaml` 文件中的配置，会把 Dockerfile中的配置覆盖，比如 compose.yaml 中 command 会把 CMD 覆盖掉。Dockerfile 中的配置是 docker容器启动时的默认参数，而 compose.yaml 文件中的 command表示真实要编排启动该容器时，需要传递的参数，当然使用启动时传递的参数覆盖默认配置。

同样的，compose.yaml 中的 entrypoint 也会覆盖 Dockerfile 中的 ENTRYPOINT. 同样的道理。

```
Dockerfile:
ENTRYPOINT = 镜像默认入口程序
CMD        = 默认命令，或 ENTRYPOINT 的默认参数

Compose: 
entrypoint = 覆盖镜像的 ENTRYPOINT
command    = 覆盖镜像的 CMD
```

# Dockerfile 常用指令说明

<font color="orange">ARG</font> : `ARG VLLM_VERSIOn=v0.21.0` ARG 用来定义 Docker 镜像构建阶段使用的变量的默认值。 后面构建时可以用传入参数覆盖，例如 `docker compose build --build-arg VLLM_VERSION=v0.21.0`。 
写在 FROM 前面的 ARG 可以用于选择基础镜像，但不会自动进入 FROM 后面的构建阶段。如果后续指令要读取它，需要再次声明。例如
```dockerfile
ARG VLLM_VERSION=v0.21.0
FROM vllm/vllm-openai:${VLLM_VERSION}

ARG VLLM_VERSION
RUN echo "${VLLM_VERSION}"
```

<font color="orange">ENV</font>: `ENV PYTHONUNBUFFERD=1`, 定义环境变量。与 ARG 不同, ENV 定义的变量保存在镜像中，构建期间和容器运行期间都可以使用。

<font color="orange">LABEL</font>：这个指令用于给 Docker 镜像添加说明信息的元数据，不会改变程序的运行逻辑。构建完成后，可以通过 `docker image inspect heteroserve/vllm-openai:v0.21.0-sm75` 
查看镜像相关的信息。LABEL 可能用于说明镜像是什么，做什么的，标记镜像版本，维护者，源代码地址。供镜像仓库，CI/CD，安全扫描工具识别。

<font color="orange">EXPOSE</font>: `EXPOSE 8000` 表示的是容器内服务监听的端口号，即这个镜像会<font color="deeppink">预期</font>通过容器内部的 TCP 8000 端口提供服务。但是 EXPOSE 只是声明和说明，不会自动把端口开放到宿主机。
真正的端口监听和映射由两部分组成：
- `CMD` 指定的 `--port 参数`，它决定了docker 容器内部在监听哪个端口。
- `compose.yml` 中会有一个配置项，`ports -“8000: 8000”`, 会宿主机端口，映射到容器端口。格式就是`宿主机端口: 容器端口`。 
访问流程是
```
客户端访问 localhost:8000
          ↓
宿主机 8000
          ↓ Docker 端口映射
容器内部 8000
          ↓
vLLM OpenAI API
```
<font color="deppink">EXPOSE 只是用于说明预期使用哪个端口，而并不是说容器会监听这个端口并开放给宿主机。真实监听的端口，再CMD中配置。</font>

<font color="orange">STOPSIGNAL</font>: `STOPSIGNAL SIGTERM`. SIGTERM 是 Linux/Unix 的“请求进程终止" 信号，编号通常是15。Dockerfile 中的 `STOPSIGNAL SIGTERM`, 表示 Docker 容器停止时，先向容器主进程发送 `SIGTERM`, 让程序有机会优雅完整的退出。
```
Docker 发送 SIGTERM
        ↓
vLLM 开始停止服务
        ↓
拒绝或结束请求
        ↓
关闭 PP/TP worker
        ↓
释放 NCCL、共享内存和 GPU 资源
        ↓
容器正常退出
```
我们在 compose.yml 中，还可以配置
```compose.yml
stop_grace_period: 2m
```
表示发送 `SIGTREM` 信号之后，最多等2分钟，如果容器进程仍未退出，Docker就会发送`SIGKILL`信号。
