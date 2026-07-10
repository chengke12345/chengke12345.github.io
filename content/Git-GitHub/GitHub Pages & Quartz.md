# 1. GitHub Pages

Github本身仓库，默认是源码视图，即显示Markdown原始文本。可以通过 Preview 视图，就是渲染后的预览。但是如果用obsidian编写文档，即便是Preview 视图，图片也未必显示，原因是Obsidian的图片语法GitHub不认。

事实上，现在很多文档软件是基于 Markdown 核心语法而扩展，比如Notion，Obisidian等。虽然它们仍然是 Markdown 文档，但是 GitHub 只支持最原始的 Markdown 语法的渲染，而不支持比如 Obsidian 的 wikilink 风格。

为了解决对文档渲染显示的需求，GitHub 就推出了GitHub Pages 的仓库，它可以做技术博客，可以渲染显示 HTML。

我们可以把生成好的HTML文件推送到 GitHub Pages 仓库，让 GitHub Pages 的 GitHub Actions 自动把仓库中的 HTML 文件，部署成一个博客网站。

所以 Github Pages 是一个独立的页面仓库服务，它本身就是页面仓库，由云端 GitHub Actions 组件自动完成部署。记住，它只是一个独立的页面仓库服务。

# 2. Quartz

Quartz 是个静态网站生成器，专门把 obsidian vault 编译成网站。Github只是一个托管平台，不是必须要使用它。但它们搭配起来最方便。

Quartz做的事，就是输入 Obsidian 的 vault (一堆.md文件+他们使用图片)，输出就是一个完整的静态网站(一堆 .html + CSS + JS)。生成出来的网站长这样：左侧文件树、右侧反向链接、底部图谱，跟 Obsidian 阅读体验很接近。可以参考 Quartz 作者自己的站：`https://quartz.jzhao.xyz`

它本身能理解Obsidian语法，将 obsidian 的 markdown 文件，转化成与Obsidan本身阅读体验接近的静态网站。

# 3. Github Pages 与 Quartz 关系

**Quartz 本身和 GitHub 没绑定**，它只是个生成工具。生成出的静态网页文件可以放在任何地方：
- 自己服务器
- Netlify / Vercel / Cloudflare Pages
- **GitHub Pages**（最常见）

**GitHub Pages 是 GitHub 提供的免费静态网站托管服务**。你的仓库里有 HTML 文件，开启 Pages 功能后，即部署完成后，GitHub 自动给你一个网址（默认 `https://yourname.github.io/repo-name/`），访问就能看到网站。

总结：Quartz 是把obsidian vault 变成 HTML 的工具。GitHub Pages 是免费的网站托管平台。两者结合可以把写笔记变成公开博客。
# 4. 典型工作流程

```
你在 Obsidian 写笔记
   ↓
git push 到 GitHub
   ↓
GitHub Actions 自动跑 Quartz，编译生成 HTML
   ↓
GitHub Pages 自动发布，公网可访问
   ↓
访问 https://chengke12345.github.io/xxx-repo/ 看到网站
```

最后生成的是一个独立的静态网站，有独立URL，公网任何人都可以访问。默认的URL形式为：`https://chengke12345.github.io/xxx-repo/`

# 5. 基本原理

<u><b>整体思路</b></u> ：把 Quartz 的代码看作一个新的独立项目，让它读取 vault 中的笔记，编译(生成静态网站)后发布到 GitHub Pages上。

<u><b>关键点</b></u>：Quartz是一个独立项目它会把笔记复制到自己的content目录里编译。

<u><b>最佳方案</b></u>：同时用两个独立的仓库。一个托管笔记文件，一个托管笔记编译后的HTML文件。普通仓库作为数据库(数据源)，Pages仓库作为展示面。

```
方案：两个独立仓库(这里是两个本地仓库，分别对应 Github  和 GithubPages 两个托管仓库)
~/Documents/Tech-Notes/         ← vault 仓库
~/Documents/quartz/             ← 新建的 Quartz 项目（新建）
```

<u><b>发布流程</b></u>：从 vault 复制 `.md` 文件到 `quartz/content/`，然后 push quartz 仓库，GitHub Actions 自动构建发布。


> [!NOTE] Quartz核心逻辑
> 整个发布的核心逻辑是，首先我们把Quartz项目 clone 到本地。Quartz 项目本身是一个框架。我们将 vault中 的笔记文件以及笔记文件使用的资源(图片等)，复制到 Quartz 项目的 content 目录下。
> 然后，我们要到Github上新建一个仓库，这个仓库。我们将装好笔记内容的 Quartz 项目推送到Github上的新仓库。
> 然后 Github Actions会根据新仓库中的 `deploy.yml` 让 Quartz 把 contents中的文件渲染。再按照部署文件中的部署要求，部署成网站

所以 Quartz 是一个框架，在 content中装好笔记内容后，推送到Github. 由 Github Actions 自动调用 Quartz, 生成静态网站，然后发布。

# 6. 部署流程

## Step 1: 本地准备

首先确认是否安装Node.js, 如果没安装或者安装版本太低，就重新安装

```bash
brew install node
```

然后 clone Quartz 项目：

```bash
cd ~/Documents
git clone https://github.com/jackyzha0/quartz.git
cd quartz
npm install
```

下载完成后，quartz目录下，就是项目的本地仓库。在quartz目录下，安装npm, 这是node.js的包管理器，使用 quartz 编译笔记的时候，需要使用它管理包依赖。
npm在quartz目录下安装，并不会污染分支，影响后续push。因为，它依赖的包下载到本地的 `node_modules/` 目录里。Quartz 项目根目录有个 `.gitignore` 文件，里面已经写了 node_modules/ , 意思是 `node_modules/` 目录被 git 完全忽略，不会被追踪、不会进入暂存区、不会被 commit、不会被 push. 装完依赖后跑这条命令，`node_modules/` 不会出现在输出里，git 当它不存在。

然后我们可以初始化Quartz:
```bash
npx quartz create
```
会问几个问题，建议这样选：
- `Choose how to initialize the content`: **Empty Quartz**（之后手动复制笔记进来）
- `Choose how Quartz should resolve links`: **Treat links as shortest path**（最接近 Obsidian 行为）
执行完后，`quartz/content/` 目录就是放笔记的地方。
执行完成后，项目推送到Github上之后，就可以被GitHub Actions 顺利调用了。

## Step 2: 把笔记复制进 Quartz

比如我们要让 quartz 编译 `Flash Attention.md` 这个笔记
```bash
cd ~/Documents/quartz
cp ~/Documents/Tech-Notes/Tech-Notes/"Flash Attention.md" content/
cp -r "~/Documents/Tech-Notes/Tech-Notes/Chats_Flash Attention" content/
```
将笔记文件(Flash Attention.md)， 和笔记依赖的资源(Chats_Flash Attention, 笔记用到的图片) 一起拷贝到 content 下面。

## Step 3: 本地预览

```shell
cd ~/Documents/quartz
npx quartz build --serve
```

我们可以在本地，直接用quartz构建项目，生成一个静态网站，以服务进程方式启动起来。打开浏览器访问 `http://localhost:8080`，能看到你的笔记被渲染成网站了。图片、wikilink 都能正常显示。确认没问题, 按 `Ctrl+C` 停掉服务。

## Step 4: 发布到 Github Pages

#### 一、 在 Github 新建一个仓库

去 [https://github.com/new](https://github.com/new) 建一个仓库，**必须命名为** `chengke12345.github.io`（用你的实际 GitHub 用户名）。这是 GitHub Pages 用户站点的特殊命名规则，会发布到 `https://chengke12345.github.io`。

也可以叫别的名字（比如 `tech-notes-site`），那样网址会是 `https://chengke12345.github.io/tech-notes-site/`。

设为 Public（GitHub Pages 在免费账号下要求 Public）。
#### 二、关联到远程仓库

把Quartz项目clone下来之后，进入quartz目录，就进入了Quartz 项目的本地仓库。此时我们默认在项目的v4分支上，v4是Quartz项目采用的主分支名字，Quartz是以项目版本号作为分支名的。但是此时我们连接的远程仓库是Quartz的项目仓库，我们需要把连接仓库变成在 GitHub 上新建的 GitHub Pages 仓库。

```shell
cd ~/Documents/quartz
git remote set-url origin git@github.com:chengke12345/              chengke12345.github.io.git
```
注意是 `set-url` 不是 `add`——因为 clone 下来时已经有了 origin 指向 Quartz 官方仓库，要改成你自己的。
#### 三、启动 GitHub Pages

去 GitHub 仓库页面 → Settings → Pages： 
Source: 选 **GitHub Actions**（不是 Deploy from a branch）
选择这一步后，GitHub才会在仓库有更新的时候，自动触发 GitHub Actions, 调用 Quartz 完成相关工作。

#### 四、 添加 GitHub Actions 工作流

在本地的quartz仓库中，新建 GitHub 工作流。GitHub Actions 是根据这个工作流的配置，来自动完成调用 Quartz，并部署到Github Pages服务器上，成为静态网站的。

新建文件： `.github/workflows/deploy.yml` 内容为：
```yml
name: Deploy Quartz site to GitHub Pages

on:
  push:
    branches:
      - v5

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v6
        with:
          node-version: 24

      - name: Cache dependencies
        uses: actions/cache@v5
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-

      - name: Cache Quartz plugins
        uses: actions/cache@v5
        with:
          path: .quartz/plugins
          key: ${{ runner.os }}-plugins-${{ hashFiles('quartz.lock.json') }}
          restore-keys: |
            ${{ runner.os }}-plugins-

      - name: Install Dependencies
        run: npm ci

      - name: Install Quartz plugins
        run: npx quartz plugin install

      - name: Build Quartz
        run: npx quartz build

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: public

  deploy:
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```
这个 YAML 文件主要就是给 GitHub Actions看的，是 GitHub Actions 的操作指南。该文件的具体解析，看后面。
我们推送项目到远程仓库的时候，`.github/workflows/deploy.yml`文件也会被推送到 GitHub 服务器上，GitHub Actions 捕捉到这个工作流配置文件的时候，就知道在该仓库更新的时候，该采取哪些行为了。
#### 五、推送 

```shell
git add .
git commit -m "init: 部署 Quartz 站点"
git push origin v5
```
注意分支是v5, 不是 main -- Quartz 用 v4 作为主分支。

## Step 5. 等待构建 

去 GitHub 仓库的 **Actions** 标签页，能看到工作流在跑。大约 1-2 分钟完成。
在`https://github.com/chengke12345/chengke12345.github.io/actions` 中可以查看Actions是否在跑，以及自动构建的结果。应该能看到 "Deploy Quartz site to GitHub Pages" 工作流正在运行或已完成。

成功后访问 `https://chengke12345.github.io` 就能看到你的网站。

## Step 6: 后续更新流程 

```shell
# 1. 在 Obsidian 写完笔记，照常 push 到 Tech-Notes 仓库
cd ~/Documents/Tech-Notes/Tech-Notes
git add . && git commit -m "新笔记" && git push

# 2. 同步到 Quartz 项目
cp -r ~/Documents/Tech-Notes/Tech-Notes/* ~/Documents/quartz/content/

# 3. 将quartz本地仓库推送到远程 Github Pages仓库
cd ~/Documents/quartz
git add . && git commit -m "update content" && git push origin v4

# 4. 等 Actions 自动构建发布，1-2 分钟后网站更新
```

## 几点说明

<u><b>GitHub 远程仓库(Repository)</b></u> 和 <u><b>Github Pages</b></u> 是 GitHub 的两种服务。

<u><b>Repository</b></u>  是存代码/文件的地方，**会渲染 markdown**，但用的是 GitHub 自己的渲染器，不认 Obsidian 扩展语法。

<u><b>GitHub Pages</b></u> 把仓库里的 HTML 文件发布成一个网站。它只负责"把文件挂出来给人访问"，**自己不做 markdown 渲染**。如果你只是把 `.md` 文件直接放到 Pages，访问到的是一个 raw 文件下载，根本看不到渲染。它只能渲染 HTML 文档。

Pages 期待的是 `.html`。所以 Pages **必须配合一个"生成器"** 使用：
- **Quartz** —— 把 Obsidian vault 编译成 HTML（懂 wikilink、双链）
- **Hugo / Jekyll** —— 通用静态站生成器（不懂 Obsidian 语法）
- **MkDocs** —— 文档型站点生成器
是这些**生成器**完成"Obsidian markdown → HTML"的转换，Pages 只是托管最终的 HTML。

Quartz总结：**Quartz 不是工具，是项目模板。你 clone 它就是把"装好了 Quartz 的骨架项目"拿到本地，往里加内容，整个推到 GitHub，云端构建发布。** 因为是同一个项目，所以分支名也沿用 Quartz 原项目的 `v4`。

```
本地 quartz/ 项目（v4 分支）
   ↓ git push origin v4
你的 GitHub 仓库（v4 分支）
   ↓ 触发 GitHub Actions
云端构建 HTML
   ↓ 自动部署
GitHub Pages 发布
   ↓
公网访问 https://chengke12345.github.io
```

## 构建笔记的基本架构

同时使用 Repositoty 和 GitHub Pages 两个仓库, 两个仓库的角色彻底分开了：

**Tech-Notes 仓库**：你的笔记**数据源**
- 存原始 `.md` 文件和图片
- Obsidian Git 插件自动同步
- 你写笔记的"源头"
- 不对外，主要是版本管理 + 备份 + 多设备同步

**chengke12345.github.io 仓库**：发布**项目**
- 内容主要是 Quartz 框架代码
- `content/` 目录是从 Tech-Notes 复制过来的笔记副本
- 触发 GitHub Actions 构建网站
- 对外展示

这是典型的 **"源 / 发布"分离架构**，在内容工程领域很常见。优势：
- 源数据干净（只有笔记本身），可以随便重组、迁移到别的发布系统（哪天换 Hugo、MkDocs 都行）
- 发布项目可以独立迭代（换主题、改样式、加插件）不影响源数据
- 你可以选择性发布——Tech-Notes 里有 100 篇，只复制 30 篇到 content/，其余的留在数据库里不公开

要做"复制"这一步同步。这一步可以手动（`cp`），也可以自动化（git submodule、GitHub Actions 跨仓库拉取、本地脚本）。这个后面再说。

这套架构想清楚后，未来你做技术博客的整个发布管线就是稳定的：

```
Obsidian 写 → Tech-Notes 存 → 复制到 Quartz → 自动构建 → Pages 发布。
```

## GitHub Actions & deploy.yml 配置

GitHub Actions 是 GitHub 提供的云端自动化服务。每个 GitHub 仓库都自带这个能力，免费账号也有（公开仓库无限额度，私有仓库每月 2000 分钟）。

它的工作方式：你在仓库的 `.github/workflows/` 目录下放 YAML 文件，描述"什么时候触发、做什么事"。GitHub 监听仓库事件，符合条件时自动在云端起一台虚拟机执行你定义的步骤。

#### deploy.yml 解析 

逐段拆解前面那份 deploy.yml 配置：

```yaml
on:
  push:
    branches:
      - v5
```

**触发条件**：当 v5 分支收到 push 时，启动这个工作流。

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
```

作业 jobs 有两个任务，一个是构建 build, 一个是部署 deploy
**在哪运行**：在 GitHub 起一台 最新的 ubuntu-latest 虚拟机来跑。

```yaml
    steps: # 省略了 -name: xxxx
      - uses: actions/checkout@v6        # 把你的仓库代码拉到这台机器上
      - uses: actions/setup-node@v6      # 装 Node.js
      - run: npm ci                      # 装 Quartz 依赖
      - run: npx quartz plugin install   # 安装           
      - run: npx quartz build            # 跑 Quartz 编译，构建生成 HTML
      - uses: actions/upload-pages-artifact@v3   # 把生成的 HTML 打包
```

**做什么事**：checkout 代码 → 装 Node → 装依赖 → 装插件 → 编译 → 打包产物。
这其实是本地做的流程，推送到GitHub Pages仓库中，让GitHub Actions 在云端再做一次。
做完这步，就相当于静态网站已经生成，我们已经做到了本地的网页预览环节。

接下来就要把它部署到 GitHub Pages 上去了。

```yaml
  deploy:
    needs: build
    steps:
      - uses: actions/deploy-pages@v4     # 把打包的 HTML 部署到 Pages
```

**部署**：把构建产物推到 GitHub Pages。

#### 完整链路

```
你 git push origin v5
   ↓
GitHub 收到 push 事件
   ↓
GitHub 检查 .github/workflows/ 下的 yml 文件
   ↓
deploy.yml 的触发条件匹配（push 到 v5）
   ↓
GitHub 起一台 ubuntu-latest 虚拟机
   ↓
按 deploy.yml 描述的步骤依次执行
   ↓
最后一步把 HTML 发布到 GitHub Pages
   ↓
你的网站更新
```

整个过程发生在 GitHub 服务器上，你的本地电脑啥也不用干。可以去仓库的 **Actions** 标签页实时看执行日志。