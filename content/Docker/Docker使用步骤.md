## <font color="grape">使用Docker</font>

按照容器化的三个步骤：首先我们编写 Dockerfile。

### <font color="#b48ff4">Step1: 创建 Dockerfile</font>

一般我们会在项目根目录下，创建一个Dockerfile文件
```dockerfile
FROM python: 3.8-slim-buster
WORKDIR /app
COPY . .
RUN pip3 install -r requrirements.txt
```

我们在第一行用FROM命令指定一个基础镜像(base image)。这可以帮我们节省很多软件安装时间。

在 Docker Hub 上有很多高质量的操作系统镜像，比如 ubuntu, debian, fedora, alpine等。不同的操作系统提供不同的包管理工具，比如 unbuntu上的apt，fedora上的dnf。DockerHub 还有方便某一种语言，某一种框架开发的镜像，比如 nginx，redis, node, python, tomcat等。

上面我们就使用DockerHub上的python镜像，这样免去了python的安装步骤。这里的 `python` 是官方镜像的名字，冒号后的 `3.8-slim-buster` 是版本号，也是标签 Tag。我们可以到DockerHub上，找到python镜像的页面，里面可以找到所有支持的标签，这里就是用的3.8版本。运行在debian buster的发行版本上。

`WORKDIR /app`：指定了这个命令之后所有的 Docker 命令的工作路径(working directory)。如果这个路径不存在，Docker会自动创建。这样可以避免使用绝对路径或手动cd切换路径。

`COPY . .` : 调用 COPY 命令将所有程序拷贝到 Docker 镜像中， 命令为 `COPY<本地路径><目标路径>`，第一个 `.` 代表根目录下的所有文件，也就是项目根目录下，即整个项目。第二个参数代表Docker镜像中的路径。这里的 `.` 代表当前工作路径，也就是之前指定的 `/app` 目录。

`RUN` 运行我们在创建镜像时，运行任意的 shell 命令。比如这里我们使用了 pip install 来安装 python程序的所有关联。

通过以上命令，我们就可以完成一个docker镜像的创建。

`CMD` ：在 Dockerfile 最后，我们会用 CMD 来指定当Docker 容器运行起来以后，要执行的命令。

注意区分 `RUN`和 `CMD`，<font color="orange">RUN是创建镜像时使用的，而CMD是运行容器时使用的</font>。

### <font color="#b48ff4">Step2: 创建镜像image</font>

我们可以使用 docker build 来创建一个镜像。我们在项目根目录下，运行`docker build -t my-proj .` ,  -t 表示 tag/标签，指定了创建镜像的名字，最后面的 `.` 告诉 docker 应该在当前目录下寻找 Dockerfile 文件。

第一次调用 `docker build` 会比较慢，因为会下载必要的镜像文件。再次调用就会快很多，因为它会缓存操作。

### <font color="#b48ff4">Step3: 启动容器 Container</font>

创建好了镜像之后，我们就可以使用 `docker run` 来启动一个容器。执行
```shell
docker run -p 80:5000 -d my-proj
```

`-p` 参数， 他会将容器上的一个端口映射到本地主机上，这样我们才能从主机上访问容器中的Web服务。前面的 80 是我们本地主机端口，后面的5000是容器上的端口。

`-d` 参数，--detached, 让容器在后台运行，这样容器的输出就不会直接显示在控制台。

`my-proj`: 表示要启动容器，使用的镜像名称。

我们通过 Docker Desktop 图形界面，可以查看应用在后台的所有输出。这对调试很方便，同时也可以看到当前容器的各种信息状态等。在图形界面中可以直接进行一些操作，它们对应的命令行工具如下：
![[Pasted image 20260528072233.png\|500]]

当我们删除一个容器的时候，之前所做的修改，新添加的数据会全部消失。如果希望保留容器中的数据，可以使用 Docker 提供的 volume 数据卷功能。可以把它当作是在本地主机和不同容器中共享的文件夹。比如下面容器1中修改了数据，可能反应到容器3中。
![[Pasted image 20260528072632.png\|500]]

我们可以通过 docker volume create 来创建一个数据卷， `docker volume create my-vlume-data`. 在启动容器的时候
```shell
docker run -dp 80:5000 -v my-volume-data:/etc/finance mt-volume-data 
```
通过 `-v`参数指定，将这个数据卷挂载(mount)到容器中哪个路径上。这里将 my-volume-data 挂载到了 /etc/finance 这个路径下，向这个路径写入的任何数据，都将被永久保存在这个数据卷中。
