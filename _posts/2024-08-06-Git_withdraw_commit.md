---
title: git撤回某次提交
date: 2024-08-06 00:00:00 +0800
categories: [Git]
tags: [Git]
pin: false


toc: true
comments: true

math: false
mermaid: true

typora-root-url: ../../xdooing.github.io
---



开发时push到远端之后，如果想撤回某次提交，主要方式有以下几种：



## 1. git revert

`git revert` 命令会创建一个新的提交，用于撤销最近一次的提交。该命令会生成一个新的提交，其中包含了对之前提交的更改进行撤销的代码。这种方式相对安全，因为它会生成一个新的提交，保留了之前的提交历史。

git revert  commit-hash

将 `commit-hash` 替换为想要撤销的提交的哈希值（使用git log查看）。

例如要撤回最近一次的提交，可以使用`git revert HEAD`



## 2. git reset

`git reset` 命令可以用来撤销之前的提交。这有几种模式，包括 `soft`、`mixed`（默认）和 `hard`。

- `soft`: 只移动HEAD指针，不改变工作区和暂存区。

  > 撤销git commit，不撤销git add

- `mixed`（默认）: 移动HEAD指针，并重置暂存区，但不改变工作区。

  > 撤销git commit，撤销git add，但是工作区的修改不会变动

- `hard`: 移动HEAD指针，重置暂存区，并改变工作区以匹配HEAD。

  > 使用`--hard`选项会使**工作目录中的文件**恢复到这个父提交的状态；
  >
  > 这意味着所有自上次提交以来的未提交的修改都将被删除；
  >
  > 如果想保留这些修改，可以使用`git stash`命令来保存它们，然后在需要的时候再应用这些修改。



撤销到某次提交时：

`git reset --<mode> <commit-hash>`

替换mode与hash即可。

如果要撤回最近一次的提交，可以使用 `HEAD~1` 或 `HEAD^`。

`HEAD^`的意思是上一个版本，也可以写成 `HEAD~1`，如果进行了 **2** 次 **commit**，都想撤回，可以使用 `HEAD~2`



## 3. 注意

与`git revert`不同，`git reset`命令会改变提交历史。因此，如果你已经将代码推送到远程仓库，最好不要使用`git reset`来撤销提交，因为会导致分支历史的改变，可能会引起问题。

如果一定需要使用`git reset`来撤销提交，并且已经将代码推送到远程仓库，可以使用`git push –-force`命令强制推送到远程仓库。但请注意，在团队协作开发时，应与团队成员商定好再执行这样的操作，以免造成代码丢失或冲突。

如果需要强制推送，但又想确保不会覆盖其他协作者的工作，可以使用 `--force-with-lease` 选项。这个选项比 `--force` 更安全，它会在推送前检查远程分支的状态是否发生了改变。
