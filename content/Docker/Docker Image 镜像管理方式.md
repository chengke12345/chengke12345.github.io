
`docker build`  产生的镜像文件，dockerd 会把它放在本地的一个目录里，Linux默认是 `/var/lib/docker/` , 这个目录通常叫做<font color="orange"> local image store 或 本地镜像缓存</font>, 远程镜像仓库(Docker Hub, 或者配置的私有远程仓库)叫做 registry。

运行 `docker run` 时，我们给定一个镜像名，docker 查找这个镜像的方式是：
1. 先查本地 image store，命中就直接用
2. 没命中，按镜像名解析 registry 地址（不带前缀的默认走 Docker Hub）
3. 从 registry 拉取（`docker pull`），存到本地 image store，再启动容器

<font color="orange">要把自己本地生成的镜像放到远程服务器上执行，我们通常是使用 registry 中转，这也是生产的标准做法。</font>

最规范、最常用的方式。流程：

```bash
# 本地：给镜像打 tag，指向目标 registry
docker tag myapp:latest registry.example.com/myteam/myapp:v1.0

# 本地：推到 registry
docker login registry.example.com
docker push registry.example.com/myteam/myapp:v1.0

# 远程服务器：拉下来运行
docker login registry.example.com
docker pull registry.example.com/myteam/myapp:v1.0
docker run -d registry.example.com/myteam/myapp:v1.0
```

`docker push` 将镜像推送至哪里，主要通过解析镜像名中的 Registry 地址。所以通常都要给本地的镜像打 tag，然后再推送到远程镜像仓库。例如:
```bash
docker push registry.example.com/myteam/myapp:v1.0
```
docker 就会解析：
- `registry.example.com`, 远程 registry地址, 如果没有写，默认就是 Docker Hub地址
- `myteam/myapp`, 仓库路径/镜像名
- `:v1.0` :  tag 版本
所以，我们要对本地镜像打 tag, tag只是让本地镜像多一个名字，但是如果是要 `docker push` 推送到远程某个 registry 中，这个tag名就不能随便取，镜像名必须符合目标 Registry 的命名格式。如上 `registry.example.com/myteam/myapp:v1.0`。
如果只是执行 `docker push myapp:latest`, 它默认会被推送到 Docker Hub 上，即推送到 `docker.io/用户名/myapp:latest` 。所以上面要先 login。

Registry 的选择：
- **Docker Hub**：免费但公开（私有 repo 数量有限制）
- **阿里云 ACR、腾讯云 TCR、华为云 SWR**：国内速度快，国内云厂商基本都提供
- **GitHub Container Registry (ghcr.io)**：和 GitHub 仓库绑定，方便
- **私有自建**：用官方 `registry:2` 镜像跑一个，或者 Harbor（更完整的企业级方案）