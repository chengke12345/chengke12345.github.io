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

