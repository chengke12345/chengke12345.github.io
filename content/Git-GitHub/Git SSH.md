# 1. SSH使用 git 服务

ssh 除了作为远程 shell 之外，还可以使用远程 git 服务。
比如使用 github, `ssh -T git@github.com`.  用 git 用户身份连接远程 Github。
>① `-T` 表示不分配 PTY(伪终端)。
  ② 客户端请求的是 exec channel, 执行一条远程命令。
  ③ GitHub 服务器为 git 这个用户配置了一个特殊的shell(通常是 git-shell). 它只允许执行几个命令，`git-upload-pack`, `git-receive-pack`等几个命令。
  ④ 当 `git push` 执行推送时，本地git命令通过 SSH 在远端执行 `git-receive-pack`, 两端通过stdin/stdout 在这个 channel 上对话。

所以从 SSH 协议的角度看，它干的事情和你 `ssh user@host ls /tmp` 没有本质区别——都是 exec 一个远端命令，然后流式读写它的标准输入输出。

GitHub 的 SSH 端点（`git@github.com`，22 端口）只接受 `publickey` 认证，密码认证是关闭的。这就是为什么用 SSH 方式访问 GitHub 必须先把公钥上传到账号设置里。

# 2. 配置Git SSH

我们首先在本地生成一个 SSH Key. 然后将生成的公钥 public key 复制到 Github上面去，在 Github 的 settings 页面上完成注册。这样在Github仓库，在本机连接的时候，就能识别本机了。完成身份验证之后，就可以使用 git 命令完成各种远程仓库的操作了。具体配置方式如下：

1. 首先生成本地的 SSH Key
```bash
ssh-keygen -t ed25519 -C "chengke12345@gmail.com"
```
执行之后，ssh 默认在 `~/.ssh/` 目录生成两个文件，`id_e25519.pub` 是公钥文件，`id_ed25519 是私钥文件`。

2. 然后查看生成的公钥 public key
```bash
cat ~/.ssh/id_ed25519.pub
```

3. 把输出整段复制到 Github 上面
```bash
GitHub -> Stettings -> SSH and GPG Keys -> New SSH Key

ssh -T git@github.com #看到类似下面的提示就成功了：
Hi 用户名! You \'ve successfully authenticated...
```

4. 然后 remote 用 SSH：

```
git remote add origin git@github.com:用户名/仓库名.git
git push -u origin main
```

### 使用 HTTPS

如果你用 **HTTPS 地址**：

```
https://github.com/用户名/仓库名.git
```

就不需要 SSH key，但 push 时通常要用 GitHub 登录或 Personal Access Token。长期用的话，我更推荐 SSH，一次配置好后比较省心。

要查看目前使用的是什么协议，先看 remote：

```bash
git remote -v

#返回结果
origin  git@github.com:用户名/仓库名.git (push)     # 结果1
origin  https://github.com/用户名/仓库名.git (push) # 结果2
```

如果返回的是上面的结果1，那 git 用的是 SSH。如果返回的是结果2，那 git 用的就是Https。

注意⚠️： `git push` 本身只是“推送到远程仓库”，真正决定 SSH/HTTPS 的是 `origin` 的 URL。

可以这样改成 SSH：

```bash
git remote set-url origin git@github.com:用户名/仓库名.git
git push
```

这时的`git push`才是在通过 SSH 推送。

# 3. SSH 使用 publickey 连接 Github

执行 `ssh -T git@github.com` 时, 表面上没有传送 ssh key，但是 ssh 客户端在背后自动做了。
首先, ssh会自动查找本地的默认私钥，按顺序在 ～/.ssh/ 下查找这些文件
```shell
~/.ssh/id_ed25519
~/.ssh/id_rsa
~/.ssh/id_ecdsa
~/.ssh/id_dsa
```
`id_ed25519` 私钥文件对应的公钥文件是 `id_ed25519.pub`. 公钥会在申请连接的时候，一并发送过去，表明“我想用 `id_ed25519` 这个公钥登陆”

其次，验证身份时，它传送的不是Key 本身, 而是一次私钥签名。这个过程跟密码登录完全不同，大致流程如下：
```
客户端                              GitHub 服务器
  |                                      |
  |--- 我想用 id_ed25519.pub 这个公钥登录 -->|
  |                                      |
  |<-- 给你一段随机数，用私钥签名一下 ---|
  |                                      |
  | [用本地私钥对随机数做签名]            |
  |                                      |
  |--- 签名结果 ----------------------->|
  |                                      |
  |          [服务器用之前你上传的公钥]   |
  |          [验证这个签名是否成立]       |
  |                                      |
  |<-- 验证通过，认证成功 ---------------|
```

所以，ssh首先找到私钥，然后用它对应的公钥去登录服务器，服务器认证后，返回一个随机数，ssh 在本地用私钥对随机数签名，再发给服务器，服务器验证成功后，连接成功建立。

**私钥从头到尾没离开你的电脑**，只有"用私钥算出来的签名结果"被发出去了。这就是非对称加密的核心安全性——攻击者就算截获了全部通信，也拿不到私钥，也伪造不出新的签名。

# 4.  Github 验证用户身份原理

注意命令 `git@github.com` —— 这个 `git` 是 SSH 协议层面的**用户名**，跟你的 GitHub 账号无关。

**所有人**连接 GitHub 用的都是 `git@github.com`。我连也是 `git@github.com`，你连也是 `git@github.com`。GitHub 服务器侧只有一个 Linux 用户叫 `git`，负责接所有人的 SSH 请求。
那 GitHub 怎么区分到底是谁连进来？**靠公钥反查账号**。

当你之前在 GitHub 网页上把 `id_ed25519.pub` 的内容粘贴到 Settings → SSH keys 时，GitHub 把这个公钥和你的账号 `chengke12345` 在数据库里建立了一条绑定关系：

```
GitHub 数据库（示意）:
  公钥指纹 SHA256:abc123...  →  账号 chengke12345
  公钥指纹 SHA256:def456...  →  账号 alice
  公钥指纹 SHA256:ghi789...  →  账号 bob
  ...
```

认证流程实际是这样的：

```
1. 你的 ssh 客户端发起连接，说"我用这个公钥"
2. GitHub 拿到公钥，去数据库反查 → "哦，这个公钥属于 chengke12345"
3. GitHub 发挑战，要求用对应私钥签名
4. 你的客户端用本地私钥签名返回
5. GitHub 用数据库里存的同一个公钥验证签名
6. 验证通过 → 服务器已经知道这是 chengke12345，于是回复 "Hi chengke12345!"
```

**所以 `git@github.com` 里的 `git` 只是 SSH 传输层的用户，真正的"你是谁"完全由公钥决定。**

# 5. Git 推送 

`git push` 时也是同一套机制。`git@github.com:YourName/tech-notes.git` 这个 URL 里：
- `git@github.com` —— SSH 连接目标（人人相同）
- `YourName/tech-notes.git` —— 要操作的仓库路径

SSH 认证完 GitHub 知道你是 `chengke12345`，然后检查"`chengke12345` 有没有权限 push 到 `YourName/tech-notes`"。如果 `YourName` 就是 `chengke12345`，权限自然有；如果是别人的仓库，就看你有没有被加为 collaborator。

# 6. 总结

公钥就是你在 GitHub 的"身份证"，私钥是签名笔。`ssh -T` 命令背后 ssh 客户端要登陆，Github 端让你签名，客户端自动找到笔、签了字、把签名交出去；GitHub 拿身份证一查就知道你是谁。