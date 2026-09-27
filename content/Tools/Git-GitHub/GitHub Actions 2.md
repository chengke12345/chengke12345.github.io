GitHub Actions 是 GitHub 自带的自动化平台，最常见用途是做 **CI/CD**：自动测试、构建、发布、部署项目。简单说：当代码仓库发生某些事件时，GitHub Actions 会自动执行你定义好的脚本。
比如：
- 提交代码后自动跑测试
- 合并到 `main` 后自动部署网站
- 创建 tag 后自动发布 npm / Docker / GitHub Release
- 定时执行任务，比如每天同步数据
- PR 时自动检查格式、类型、单元测试
**核心概念**
- **Workflow**：工作流，一个自动化流程，写在 `.github/workflows/*.yml`
- **Event**：触发条件，比如 `push`、`pull_request`、`schedule`
- **Job**：工作流里的任务，可以并行或串行执行
- **Step**：Job 里的具体步骤
- **Action**：可复用的动作，比如 `actions/checkout`
- **Runner**：执行任务的机器，GitHub 提供的 Linux/macOS/Windows 虚拟机，也可以自托管
- **Secrets**：保存敏感信息，比如部署密钥、API Token

# 0. 启用 Github Pages

GitHub Pages 是一个仓库级别的功能或者选项。任何一个普通仓库都可以选择启用 Pages 功能

```
Settings -> Pages -> Build and deployment
```

只要在这里配置了发布源比如：

```bash
Source: Deploy from a branch
Branch: main
Folder: /docs
# 或者(上下两者二选一). 关掉发布源就是在 Deploy from a branch下，Branch 设为 None
# Deploy from a branch. Branch: None， 这是默认设置。
Source: GitHub Actions
```

这个仓库就被 GitHub 当作一个 GitHub Pages 站点来处理。
# 1. 启动 GitHub Actions

GitHub Actions 对应的操作域是仓库级别。默认情况下 GitHub Actions 是不启动的，我们需要在 GitHub Pages 页面下手动选择开启。
![image](Pasted%20image%2020260708163638.png)

选中之后，就表示 GitHub Actions 功能开启到了。

# 3. Github Actions 事件触发 

GitHub Actions 功能启动之后，每次收到 push 过来的分支，GitHub Actions 都会自动扫描提交分支的 .github/workflows/\*.yml 文件，所有的 yaml 文件，GitHub Actions 都会读取，但是不是所有的 yaml 都会被执行。

**GitHub 的触发逻辑如下**：
1. 本仓库发生一个事件，比如 `push`。
2. GitHub 根据这个事件关联的 `commit SHA` / `Git ref`，去该版本里找 `.github/workflows` 下的 workflow 文件， 即 `*.yml`  或者`*.yaml`文件。如果这个分支是真的推到了**本仓库**，例如 `refs/heads/foo`，那就是本仓库的 `push` 事件。
3. 只有 `on:` 配置匹配该事件的 workflow，才会被创建运行。如果该分支里的 `.github/workflows/xxx.yml` 或 `.yaml` 有：
```yaml
on: push   
```
表示 on: 匹配事件。即，如果触发了 push 事件，那么这个 yaml 就会创建执行。触发事件必须与on: 的事件匹配才行。比如 `push`、`pull_request`、`workflow_dispatch` 等.

4. 除了事件匹配，还需要该 yaml 或 yml 要满足其他配置的过滤条件，没有被过滤掉才能真正执行。比如设置branches 等过滤条件，例如：
```yaml
on: 
	push:
		branches:
			- main
```
就表示，push事件发生，并且这个push必须是推送到 main 分支上(过滤条件)时，这个yaml 才会被创建执行。
5. 触发的条件除了 `push` 之外，还有可能有 `pull_request`, `workflow_dispatch` 等。

# 4. GitHub Actions 执行工作流

以 quartz 项目为例说明。当 GitHub Actions 扫描到 .github/workflows/deploy.yml 文件时，就会按照 yaml 文件中的配置执行工作流。目前最新的 quartz v5 的工作流配置文件 deploy.yml 如下:

```yaml
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

GitHub Actions 在分支推送的时候，就会读取分支下的 .github/workflows/deploy.yml，然后根据配置，会执行如下行为：
1. 这个工作流 workflow 的名字叫做 Deploy Quartz site to Github Pages. 在 v5 这个分支被推送push的时候，触发这个工作流。
2. 工作流设置的三个权限，contents 是读，pages 和 id_tokens 是写。读取 contents 中的内容生成pages页面。
3. 工作流中的作业 jobs 有两个任务，一个是 build， 另一个是 deploy，按顺序执行。即首先要调用quartz项目构建出Pages页面，然后把页面部署到 GitHub Pages 上。
4. 构建和部署的过程，GitHub 都会启动一个 Ubuntu 虚拟机来执行(两个任务都配置了 runs-on ubuntu-latest)。
5. 构建和配置细节，不做赘述。只是 v5 与 v4 相比，除了要通过 npm ci 安装依赖之外，还要安装quartz 的插件。

>总结：Github Actions 会扫描读取工作流文件 deploy.yml。 当工作流文件中的触发条件满足时，v5 分支上的 push。GitHub Actions就会创建并执行这个工作流。执行工作流的过程，GitHub 主要是通过创建一个 Linux 虚拟机来执行。执行两个任务，调用 quartz 项目构建出页面，把页面部署在 Github Pages。执行完之后，虚拟机释放。然后就可以访问 GitHub 服务器的页面了。

这就是 GitHub Actions 标准触发工作流，自动化构建和部署页面的过程。