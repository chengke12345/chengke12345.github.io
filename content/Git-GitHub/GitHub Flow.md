业界标准的工作流程是这样：
# Step 1：首先将项目克隆到本地

```shell
git clone <远程仓库地址>
```

克隆的方式和 `git clone` = `git init` + `git add remote` + `git fetch` + `git checkout main`效果一样。但是基本上都直接用 clone 就可以了。 

# Step 2: pull 拉新

切换到本地 main分支，用 pull 拉main分支最新的状态，确保目前本地的main是和远程仓库同步的最新状态。`git pull = git fetch + git merge`

```shell
git checkout main
git pull                    # 确保 main 是最新的
```

# Step 3: 在main上创建自己的分支

```shell
git checkout -b myfeature # 从最新的 main 开新分支 myfeature 并切换到myfeature上去。
```

# Step 4: 在自己的分支上开发，多次提交

```shell
# 多次 commit
git push origin myfeature  # 第一次推送，分支出现在远程
```

在自己的分支 myfeature 上开发，开发过程中，会有多次 commit, 并推送至远程仓库的myfeature分支进行更新。

# Step 5 : 开发完成后，再次同步一下远程的main

以免远程的main在开发期间发生了变化。更新的时候，先拉远程的 main 到本地，让本地的main先完成merge。再切换到 my-feature 分支，让它merge 本地的 main。这是为了避免冲突. 然后再推送到远程。如果是第一次提交，远程仓库中就会多一个 my-feature 分支

```bash
git checkout main
git pull
git checkout my-feature
git merge main              # 或 git rebase main
git push origin my-feature
```

# Step 6: 发起 PR 

在 GitHub 网页上点击发起 PR，等 review。Review 过程中，如果 reviewer 提了修改意见，你在本地改完 commit，再 push 到同一个分支，PR 会自动更新。

# Step 7: Merge完成，删除分支

**Merge 之后**， 即维护者合并 PR 后，你这个分支的使命就完成了。本地和远程的 my-feature 分支都可以删掉：

```bash
git checkout main
git pull                    # 把合并结果拉下来
git branch -d my-feature    # 删本地分支
git push origin --delete my-feature  # 删远程分支
```

然后开始下一个功能时，重新从 main 拉新分支。

# 总结

> [!NOTE] 标准GitHub Flow
> <font color="lightblue">首先git clone项目到本地(第一次)。git pull 拉分支保持更新(第二次开始)。开一个自己的分支，在自己的分支上往后开发。开发完成后，为了避免在此期间远程仓库中的main发生了变化，再git pull 一次更新本地main，再在自己的分支上 merge 本地 main，让自己的分支也处于最新状态. 推送自己的分支到远程仓库，确保没问题后，发起一个PR。等待维护者审核，审核通过后，远程仓库的主分支merge我们提交的分支。然后，本地和远程的我们的分支都可以删除了。下一个功能，重新从main拉一个新分支来开发。</font>

我们多个人都拉同一个仓库里面的分支，然后开发，提交推送给这个仓库的分支，本质上就是多个人在同一个分支上往后开发。而把仓库分支拉下来，创建自己的分支，在自己分支上开发，然后推送到远程，并请求PR的方式，就是每个人有一个自己的开发分支。这是两种不同的方式。

> [!NOTE] 最佳实践
> 一般1～3人的小项目，或者是自己的项目，就采用直接在main上开发，pull/push，这样效率最高。但是更大的团队或者开源项目，就采用feature分支 + PR 的方式，更加安全可控
