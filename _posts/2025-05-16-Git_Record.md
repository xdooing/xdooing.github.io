---
title: git相关记录
date: 2025-05-16 00:00:00 +0800
categories: [Git]
tags: [Git]
pin: false


toc: true
comments: true

math: false
mermaid: true

typora-root-url: ../../xdooing.github.io
---





记录开发中遇到的git相关的问题



## **使用vpn代理**

> 开了VPN之后，git clone/push 仍然会失败，访问不到链接，是由于 git 默认不使用代理导致，配置 git 代理后可提升速度。

### 1 查看vpn本地代理端口

不同 vpn 软件或安装的随机性导致每台机器的端口号并不一致，以显示为准。

![](/assets/blog_res/2024-05-29-GitVPN.assets/screenshot-20240529-114154.png)

当前显示 http 端口为：2802；socks5 端口为：2801；

或者：

![](/assets/blog_res/2024-05-29-GitVPN.assets/screenshot-20250509-144436.png)

显示端口号为 7890

### 2. 配置git代理

**windows上使用git**

如果是在windows上面使用git，则只需要设置本地回环地址即可，端口就是代理客户端的端口。

```shell
git config --global http.proxy http://127.0.0.1:{port}
git config --global https.proxy http://127.0.0.1:{port}
或者
git config --global http.proxy socks5://127.0.0.1:{port}
git config --global https.proxy socks5://127.0.0.1:{port}
```

例如我可以配置为下面这样，或者端口号改为7890

```shell
git config --global http.proxy http://127.0.0.1:2802
git config --global https.proxy http://127.0.0.1:2802

git config --global http.proxy socks5://127.0.0.1:2801
git config --global https.proxy socks5://127.0.0.1:2801
```

查看是否配置成功：

```shell
git config --global --list
```

取消Git配置：

```shell
git config --global --unset http.proxy
git config --global --unset https.proxy
```

**Linux上使用git**

如果是在Linux上使用git，而代理服务运行在 Windows，则需将 Linux 的代理地址改为 **Windows 的局域网 IP**

ip查看方式：windows terminal 输入`ipconfig`

找到正在使用的网络适配器，例如无线局域网适配器WLAN或以太网适配器Ethernet，下面的ipv4地址就是自己的局域网地址。

![](/assets/blog_res/2024-05-29-GitVPN.assets/ss_20250509203833.png)

接着设置代理客户端为 **允许局域网连接**，这一点很重要，clash可以进行设置。

最后在Linux home目录下创建配置文件：

```shell
# ~/.http_proxy
export http_proxy=http://192.168.31.43:7890
export https_proxy=http://192.168.31.43:7890
```

之后使用git或需要代理的时候，仅需  `source ~/.http_proxy `即可，很方便。



## **将仓库1的commit转移到仓库2**

最近在进行开发的时候，遇到了一个棘手的事情（当然是对于git菜鸟的我来讲），公司之前的代码都在dev分支，最近开发了分布式之后，在dp分支开发了一段时间，因此我自己目前做的功能也往远程分支commit了几次，但是还没有merge到dp分支，然而分布式测试完成之后，要将dp分支替换用作接下来项目的主分支。

这样一来，问题就出现了，相当于我本地代码的远程仓库被删除了，或者说仓库名与地址都改变了，现在我想将之前commit的代码以及正在开发的代码转移到新的仓库中继续开发，怎么做到呢？

### 1. git cherry-pick

[git cherry-pick]: https://blog.csdn.net/weixin_44799217/article/details/128279250

通常开发时分两种情况：

- 需要将某一个分支的所有代码变动，那么就采用合并（git merge）
- 只需要某一个分支的部分代码变动（某几个提交），这时可以采用 Cherry pick

cherry-pick 和它的名称翻译一样，精心挑选，挑选一个我们需要的 commit 进行操作。它可以将在其他分支上的 commit 修改，移植到当前的分支。git cherry-pick命令的作用，就是将指定的提交（commit）应用于其他分支。

### 2. 不同仓库之间移植commit

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



## **撤回某次提交**

开发时push到远端之后，如果想撤回某次提交，主要方式有以下几种：

### 1. git revert

`git revert` 命令会创建一个新的提交，用于撤销最近一次的提交。该命令会生成一个新的提交，其中包含了对之前提交的更改进行撤销的代码。这种方式相对安全，因为它会生成一个新的提交，保留了之前的提交历史。

git revert  commit-hash

将 `commit-hash` 替换为想要撤销的提交的哈希值（使用git log查看）。

例如要撤回最近一次的提交，可以使用`git revert HEAD`

### 2. git reset

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

### 3. 注意

与`git revert`不同，`git reset`命令会改变提交历史。因此，如果你已经将代码推送到远程仓库，最好不要使用`git reset`来撤销提交，因为会导致分支历史的改变，可能会引起问题。

如果一定需要使用`git reset`来撤销提交，并且已经将代码推送到远程仓库，可以使用`git push –-force`命令强制推送到远程仓库。但请注意，在团队协作开发时，应与团队成员商定好再执行这样的操作，以免造成代码丢失或冲突。

如果需要强制推送，但又想确保不会覆盖其他协作者的工作，可以使用 `--force-with-lease` 选项。这个选项比 `--force` 更安全，它会在推送前检查远程分支的状态是否发生了改变。



## **shallow clone导致的push rejected**

push到远程分支的时候显示 `remote rejected new_dev -> new_dev (shallow update not allowed)`

### 1. 原因

- **浅克隆的局限性**：浅克隆通过 `--depth` 参数限制拉取的提交历史深度（例如仅拉取最新的一次提交）。虽然节省了时间和空间，但此类仓库无法正常推送新分支或更新引用，因为缺少必要的历史提交信息。
- **Git 服务器的保护机制**：为了防止仓库损坏，Git 服务器会拒绝来自浅克隆的推送操作（`shallow update not allowed`）。

### 2. 解决方法

检查是否是shallow clone，是的话返回true

```bash
git rev-parse --is-shallow-repository
```

拉取仓库完整历史，这会从远程仓库下载所有缺失的提交历史，将你的本地仓库转换为完整克隆。

```bash
git fetch --unshallow
```

### 3. 注意

如果远程已有 `new_dev` 分支：

- 若远程分支存在但本地未关联，可先拉取远程分支：`git pull origin new_dev`。
- 如果存在冲突，需要合并或使用 `git push --force`（强制推送需谨慎，避免覆盖他人工作）。

因此如果需要推送代码，clone时不要用--depth参数
