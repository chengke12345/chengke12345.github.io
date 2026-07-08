
# Git 简介

Git 就像一个可以无限次存档的游戏和时光机。写代码就如同打游戏，Git 能随时记录你的进度，创建不通分支，去尝试新玩法，还能随时回档，以及合并不同分支的成果。

-  `git init` : 创建一个仓库，相当于游戏文件夹。打开终端，进入项目文件夹，执行`git init`, 该目录下就会创建一个 `.git` 的文件夹 (隐藏文件/文件夹用 . 开头)。这样，该文件夹成为了本地仓库。它的标志就是该文件夹下包含 `.git 文件夹`。该项目的所有版本信息都存放在里面。`git init` 之后，就默认“创建并进入main分支”，但是此时main分支并不是真实存在，`git branch` 什么都看不到。要等到第一次 commit 的时候，main分支才会真实被创建。

- `git commit` : 比如写了一个程序，[hello.py](http://hello.py)，然后告诉 Git 要跟踪这个文件，我们要执行 `git add hello.py`。这个操作把文件放进暂存区，类似于购物车。然后再 git commit 写上备注。比如“第一次提交，实现了 hello”。 `git commit -m "第一次提交，实现了hello"`. 这就是正式存档。把这个文件提交到本地仓库，.git 文件夹中会保存版本信息，这样就算这个文件/改动已经加入了本地仓库。

> add vs commit : 为什么要分 add 和 commit 两个步骤呢？add 是挑选哪些改动要保存，可以只选几个文件，commit 是真正生成一个版本快照。就像拍照前，可以先让几个人站进镜头，那是add，按下快门，就是commit。也可以用 `git add .` 把当前所有改动都加进去。

- `git status` : 可以查看 git 的当前状态。看哪些文件改了，哪些文件在暂存区，哪些还没跟踪。

- `git log --oneline` : 简洁看提交历史，每一个提交有一个40位哈希，前几位就能辨认出来。 比如a1b2c3d 这样的 ID，这就是版本的唯一编号。可以用它来回滚，或比较差异。

- `git branch`: 分支, 分支就是平行宇宙。默认分支叫 master 或 main。假如我们希望添加新功能，又害怕破坏现有代码。就可以创建一个新的分支，用 `git branch bname` 。单纯要列出本地所有分支，直接用 `git branch` 。

- `git checkout`: 切换分支 ，分支创建完成后，可以使用 checkout 切换到 bname 分支，用 `git checkout bname` 切换过去。
  也可以 `git checkout -b bname`, 一条命令执行创建新的分支和切换到这个分支，两个操作。

- `git merge`: 合并, 要把某分支合并到主分支，就是先切换回主分支main，然后用 `git merge bname` 把 bname 这个分支合并到主分支中来。

<u><b>合并 merge</b></u>：就是把另一分支的修改嫁接过来。如果两个分支修改了同一个文件的同一行，就会产生冲突，需要手动解决。例如，我们在main分支，改了文件第一行是 hello, 在 feature 分支改的同一行是 hi。合并时，git 会停下来，文件里会显示冲突标记，

```shell
<<<<<<Head
hello
========
hi
>>>>>>feature // 分支名
```

这种情况下，我们手动编辑成想要的样子，然后 add commit 就可以了。冲突不是错误，只是需要我们做个决定。关于 git merge 合并，之后会做详细说明。

# Github 简介

刚才我们的操作都是在本地的 git 仓库上，即我们将某个目录设置为 git 的仓库目录。我们如何实现多人合作呢？这就要使用远程仓库， 比如 Github 和 Gittee 等等。以 Github 为例。

首先我们需要在 Github 上新建空仓库，这一步需要到 Github 的网页上去执行。然后，在本地Git仓库的目录下，执行下面两行命令

```bash
git remote add origin <https://github.com/你的用户名/项目名.git>
git push -u origin main # -u 记下默认关联，以后可以直接 git push
```

它是不会创建出任何新的东西。它只是给某个远程的URL取了一个名字，以后用该名字代替长的URL。origin 是我们为远程的仓库取的一个别名。注意⚠️，这个别名的作用域就是当前仓库。

第一次推送主分支，使用 `git push -u origin main`, 把本地的main分支，推送到远程origin仓库。它把 origin 和 main 设置为默认关联，以后直接 `git push` 就可以推送了。
##  1. git push

`git push` 就是把当前分支，推送到远程仓库中。Github 的机制下，远程仓库会认为push过来的分支，是对这个分支的一次提交。所以，它要求本地仓库在本次修改之前的状态，与仓库的最新状态保持一致。注意⚠️，Push 是一次提交，而不是请求合并，这个要区分开。

我们创建一个分支feature，在第一次执行 `git push` 提交的时候，远程并没有这个分支。执行之后应该会报错

```bash
fatal: The current branch feature has no upstream branch.
To push the current branch and set the remote as upstream, use
    git push --set-upstream origin feature
```
加 `-u`（即 `--set-upstream`）才会建立跟踪关系：

```bash
git push -u origin feature
```

这条命令做了两件事：
1. 在远程创建 feature 分支，指向你 push 的 commit
2. 建立本地 feature → `origin/feature` 的跟踪关系
之后再 push/pull 就不用带参数了。

我们在建立新项目的时候，在远程创建好一个新仓库后，在本地仓库目录下 `git init` , `git remote add orign ...` 后，就会执行一次 `git push origin main` 进行空推送，目的就是为了建立跟踪关系。

## 远程跟踪分支 与 跟踪关系

<u><b>远程跟踪的分支(remote-tracking branch)</b></u>:  是本地的一个只读指针，比如 `origin/main`. 它记录"我上次 fetch 时远程 main 的位置"。`git fetch` 时会自动创建和更新它们。

<u><b>跟踪关系（tracking / upstream</b></u>:  是本地分支和 远程跟踪分支之间的关联配置。比如配置"本地 main 跟踪 `origin/main`"。配置后，`git pull` 和 `git push` 不用指定参数，Git 知道默认对应哪个远程分支。

配置跟踪关系是在 checkout/switch 出本地分支时建立的（或手动用 `git branch --set-upstream-to` 配置, `-u`参数） checkout 自动建立跟踪关系，push 不自动建立。
- **checkout 场景**：远程已有 `origin/feature`，本地基于它创建分支，**因果明确**——新分支显然就是要跟踪这个远程分支。
- **push 场景**：本地先有 feature，远程还没有，Git 不确定你是否真的想把它公开为同名远程分支，所以要你明确确认。
- 
## 2. git clone

其他的人，可以通过

```bash
git clone <远程仓库地址>
git pull origin main # 相当于 fetch + merge
```

git clone 是获取远程仓库地址中的项目，将项目文件，包括.git仓库，拷贝到本地我们执行git clone命令的目录下。它会新建一个项目文件夹，默认以远程仓库的项目名命名。这个文件夹下，就是拷贝远程仓库到本地的完整.git仓库和工作区文件。

克隆后，git 会自动把克隆来源的远程仓库命名为 origin。并设置 master/main 分支跟踪远程。克隆只是把远程仓库的项目，拷贝到本地，我们可以在本地玩，项目就变成 “单机版” 了。

我们采用 git clone 将项目克隆到本地的某个目录下的时候，该目录自动就成为一个本地 Git 仓库，因为我们克隆了 .git 文件夹。
`git clone` 做的事：
>1. 在本地创建目录 `/myproj/`
>2. 在里面创建 `.git/` 文件夹，存放完整的 Git 仓库数据（所有 commit、tree、blob、分支指针等）
>3. 把远程的最新文件 checkout 到工作目录
>4. 自动配置远程地址（`origin` 指向你 clone 的 URL）

**`.git/` 目录的存在 = 这就是一个 Git 仓库**。Git 识别仓库的唯一标准就是这个目录。

- ##### 项目目录

	当我们进入项目目录的时候，我们实际进入了项目本地仓库，进入了和远程仓库中相同的分支，默认是主分支(main)。我们往后的开发，是在这个分支上往后开发，并不是开了一个新分支。提交的时候，提交的是主分支往后的状态。

- ##### 自己的分支

	我们可以使用 `git branch bname` 创建一个自己的分支。这个分支是从本地仓库中的主分支分出来的，我们可以在上面开发。开发完成后，我们可以直接推送自己的分支到远程仓库，远程仓库中就会保存一个我们自己的分支。比如 Github 上可能有 main 分支, feature A 分支， feature B 分支，feature C 分支等等。

我们自己分支与远程分支之间的关系和管理方式，与本地的main与远程仓库中main的关系和管理方式相同。main其实没有任何特殊地位，它在 Git 里就是一个普通分支只是约定俗成，成为主线，所有分支的 push / pull 的机制完全一致。

## 3. git fork

fork 是将别人远程仓库中的项目，在我们自己账号下，完整拷贝一个别人的项目仓库的副本。由于这个仓库的副本，在我们自己的账号下，所以我们对这个仓库副本具备完全的权限。我们提交的操作，包括 `git push` 都是提交到fork的副本仓库上。fork出来的项目是放在远程服务器上，我们自己的账号下，相当于创建了一个独立的项目实例，专门供我们开发使用。

## 4. Pull Request, PR

注意⚠️，对于开源项目这样的公开资源，原始的项目仓库，通常我们是没有权限进行任何提交的，比如 `git push` 等。我们只能提交到 fork 仓库，然后再提交一次 PR(Pull Request)。

PR主要是由项目维护者，审核你的合并请求，通过测试，代码审查，benchmark等一系列审查之后，就将你 fork 仓库中的分支，merge 到项目仓库中。

注意，PR的提交，必须通过 fork 仓库作为“中转”，项目维护者，是通过fork仓库的分支去查看审核，如果通过，就从fork拉分支去merge. 我们不能直接对原始项目仓库提交 push，会被直接拒绝。

## 5. git pull

我们可以通过 `git pull origin main` 将远程代码拉下来更新，这个命令是将 origin 这个分支拉下来merge到本地main上。这个命令相当于是 fetch + merge。

## 6. git fetch

`git fetch` 只是从远程下载最新的提交，但是不合并到当前分支，相当于将远程的更新下载下来放在本地缓存，可以先检查一下更新，再决定要不要 merge。 团队协作时，常常用 fetch 再加上自己 merge。避免意外。

fetch主要做的工作分为下面两部分
1. 真实下载数据对象，把远程有，本地没有的commit, tree, blob 全部下载下来存到本地的 `.git/objects/` 目录下.
2. 更新远程跟踪指针，把`origin/main`, `origin/feature` 等指针更新到远程仓库分支的当前位置。

> [!NOTE] 注意⚠️
> `git fetch`是确实会真实下载数据。并且它会更新远程仓库分支的指针 `/origin/main`, `origin/feature`等等。但是，它在本地不创建分支，如果分支本地已经存在，也不会merge。它只做下载数据，修改跟踪指针这两件事。

## 7. git reset 与 git revert

提交了错误代码

改错了但是还没有add, `git checkout --<文件名>` 可以撤销工作区修改。已经add但是还没有commit, 用 `git reset HEAD <文件名>` 移出暂存区。已经 commit 但是还没有 push, 可以用 `git reset soft` 回到上一次保留修改，或者 reset hard，这个慎用，会永久丢失修改。如果已经 push 到远程，用 `git revert <提交哈希>` 生成反操作新提交。

git reset 与 git revert

reset 是往回移动分支指针，改写历史。revert是保留历史，在分支上叠加一次反向提交。比如，往前走一步，reset 是撤销这一步向前，revert 是下一步是往回退一步。

团队合作时，严禁对已经 push 的公共历史乱用 reset。这会搞乱别人的仓库。

## 8.  git stash

如果改了一半的代码，想要切换到另一个分支修bug，但是又不想提交这些半成品，这时就可以用 `git stash` , 它会把当前修改暂存起来，工作区变干净，然后 checkout 到 bugfix 分支。修完再切回来，用 `git stash pop` 恢复。stash就类似一个临时抽屉，可以 stash list 查看多个。

## 9.  git tag

可以使用 `git tag v1.0` 做轻量级标签，或用 `git tag -a v1.0 -m “发布1.0版本”` , 再用 `push origin v1.0<标签名>` 将tag的分支推送到远程仓库。 以后可以随时checkout 标签，回到此版本。

## 10. git rebase

rebase 是另外一种整理历史的方式。比如，我们在 feature 分支提交了两次，在main上也有新提交。在feature上，执行 `git rebase main` 会把 feature 上的提交，挪到main最新的提交之后，历史就更直，优点是清晰，缺点是改写哈希。不要对已经 push 的公共分支乱用。

个人分支或本地整理，可以用 rebase, 团队共享的主干用merge。不要rebase别人依赖的分支。

## 11. git diff

我们要查看代码做了哪些修改。

`git diff` 默认是工作区对暂存区，暂存区是 git add 的临时区域。本质上是对比，我上次add之后，又改动了哪些地方。
`git diff --cached` 是暂存区对最近一次提交的对比。即 add 对上次 commit 的差异。
`git diff HEAD` 是工作区，对最近一次 commit
`git diff 分支1 分支2` 比较两个分支之间的差异。

## 12. git checkout

`git checkout` 是个**多功能命令**，这是 Git 历史遗留的设计问题。checkout 的几种用法如下：

```bash
git checkout main              # 切换到已存在的 main 分支
git checkout -b new-branch     # 创建并切换到新分支
git checkout file.txt          # 撤销 file.txt 的修改（恢复到上次 commit 状态）
git checkout <commit-hash>     # 切换到某个 commit（detached HEAD）
```

一个命令干了好几件事。`git checkout main` 能"创建"本地分支
当你执行 `git checkout main` 时，Git 的逻辑是：

1. 本地有 main 分支吗？有 → 切换过去
2. 没有？检查是否有**唯一一个**远程跟踪分支叫 `origin/main`
3. 有 → **自动**基于 `origin/main` 创建本地 main，并设置追踪关系，然后切换过去
这是 Git 的便利特性，叫 "DWIM"（Do What I Mean）。所以同一条命令 `git checkout main`，第一次执行时是"创建+切换"，之后执行是"切换"。


### .gitignore 临时文件/密码配置不想提交

在项目根目录创建 `.gitignore` 文件, 写要忽略的规则和文件。这样，在 `git add .` 时会自动跳过它们.

### 典型 Github flow

典型 github flow, 首先从main上开一个新的 feature 分支，在上面开发新功能，开发过程中多次推远程 git push，新功能开发完成后，提交 Pull Request，同事 review, 合并回main，删除特性分支，再在main上打标签发布。然后开发下一个功能，从新开一个新的分支，再重复上述步骤。
关于更详细的 GitHub flow, 参考 [标准 GitHub Flow](Git&GitHub入门原理/标准 GitHub Flow)
更复杂的Github flow还有develop(逐渐Deprecated)，release, hotfix，本质还是分支加合并。

### 常用命令

基本命令：`init` `clone` `add` `commit` `status` `log` `diff`
分支相关：`branch` `checkout` `merge` `rebase`
远程推送：`pull` `push` `remote`
其他命令：`stash` `reset` `revert` `tag`

不用怕犯错，Git几乎都能撤销。 add是样片选择，commit 是实际拍照， push是发朋友圈。







