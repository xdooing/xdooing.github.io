---
title: linux firefox提示“firefox is already running”的解决方法
date: 2024-10-31 00:00:00 +0800
categories: [Linux]
tags: [firefox]
pin: false


toc: true
comments: true

math: false
mermaid: true

typora-root-url: ../../../xdooing.github.io
---





> Linux环境下，通常会有这种情况：
>
> 1. 多用户通过vnc访问指定IP，通常是公司服务器。不同用户执行firefox，firefox进程存在且仅能存在一个。
> 2. 在不同的服务器上，home目录通过网络文件系统 NFS 等方式共享，因此导致Firefox 的配置文件也被共享，最终造成在一台服务器上打开firefox之后，其他服务器无法打开。



**解决办法：**

firefox每次启动，会关联一个profile，默认在~/.mozilla/firefox/xxxx.default目录里。

linux firefox提示“firefox is already running”的时候，将目录拷贝一份：

1. `cp -r ~/.mozilla/firefox/xxxx.default ~/.mozilla/firefox/xxxx.default-1`
2. `firefox –profile ~/.mozilla/firefox/xxxx.default-1`

问题即可解决！
