---
title: git将仓库1的commit转移到仓库2
date: 2024-06-24 00:00:00 +0800
categories: [Git]
tags: [Git]
pin: false


toc: true
comments: true

math: false
mermaid: true

typora-root-url: ../../xdooing.github.io
---







最近在进行开发的时候，遇到了一个棘手的事情（当然是对于git菜鸟的我来讲），公司之前的代码都在dev分支，最近开发了分布式之后，在dp分支开发了一段时间，因此我自己目前做的功能也往远程分支commit了几次，但是还没有merge到dp分支，然而分布式测试完成之后，要将dp分支替换用作接下来项目的主分支。

这样一来，问题就出现了，相当于我本地代码的远程仓库被删除了，或者说仓库名与地址都改变了，现在我想将之前commit的代码以及正在开发的代码转移到新的仓库中继续开发，怎么做到呢？



## 1. git cherry-pick

> 参考链接：https://blog.csdn.net/weixin_44799217/article/details/128279250

通常开发时分两种情况：

- 需要将某一个分支的所有代码变动，那么就采用合并（git merge）
- 只需要某一个分支的部分代码变动（某几个提交），这时可以采用 Cherry pick

cherry-pick 和它的名称翻译一样，精心挑选，挑选一个我们需要的 commit 进行操作。它可以将在其他分支上的 commit 修改，移植到当前的分支。git cherry-pick命令的作用，就是将指定的提交（commit）应用于其他分支。



## 2. 不同仓库之间移植commit

假如要将repo1中的某次commit移植到repo2中，主要分为以下步骤：

1. 获取commit的哈希值

   首先，在 repo1 中找到你想要 cherry-pick 的 commit，并记下它的哈希值。（git log）

2. 在 repo2 中添加 repo1 作为远程仓库

   切换到 repo2 并添加 repo1 作为一个远程仓库。这样你就可以从 repo1 中获取特定的 commit

   ```shell
   cd path/to/repo2
   git remote add repo1 ../path/to/repo1  #git remote add old ../../icp-dp/icpower
   ```

3. 获取 repo1 中的更改

   ```shell
   git fetch repo1  #git fetch old
   ```

4. cherry-pick 特定的 commit

   ```shell
   git cherry-pick abc123
   ```

5. 处理冲突（如果有）

   如果有冲突，Git 会提示你解决这些冲突。你需要手动编辑冲突文件以解决问题。

   ```shell
   git add <resolved_file>
   git cherry-pick --continue
   # 如果你想放弃 cherry-pick 操作，可以使用以下命令：
   git cherry-pick --abort
   ```

6. 移除临时远程仓库（可选）

   ```shell
   # 如果你不再需要 repo1 作为远程仓库，可以将其删除：
   git remote remove repo1
   ```

一般来说，在第5步自己解决完冲突之后，就可以直接add + commit + push了。

以上方法实测有效，避免在新的仓库下cv之前代码。

