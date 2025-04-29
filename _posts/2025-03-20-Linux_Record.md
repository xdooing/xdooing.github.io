---
title: Linux常用记录
date: 2025-03-20 00:00:00 +0800
categories: [Linux]
tags: [Linux]
pin: false


toc: true
comments: true

math: false
mermaid: true

typora-root-url: ../../xdooing.github.io
---





> 记录一下工作中常用的Linux操作



## 1. Linux常用记录

```shell
#在当前目录下的所有文件中的test_run下的icpower.log文件中查找 Merge 关键字  ---注意grep可以在文件中查找
grep Merge */test_run/icpower.log

#-r是递归的意思，这里的意思是在当前文件中的所有项中查找Merge，无论是文件还是文件夹
grep Merge -r * 

#统计目录的总大小，并以可读的方式（以K、M或G为单位）进行表示
du -sh foldername

#在整个系统中查找文件夹folder，-type d表示只搜索文件夹
find / -type d -name "folder"

#查找当前环境变量TMPDIR的值
echo $TMPDIR

#查看自己的端口号
vi ~/ .vnc/ 

#查看当前文件中有多少个文件，find . 会把文件一行行列出来，而 wc -l 这条命令是查看行数，例如 wc -l file1
find . | wc -l

#命令行清空一个文件的内容
truncate -s 0 filename
cp /dev/null filename
echo -n > filename  #echo一个空字符串并重定向到filename，-n是echo参数，表示不要加换行符

#终端里面直接用命令打开文件图形界面
nautilus filename   #ps -ef | grep -i desktop 查看是哪种桌面环境

#scp --两个主机之间发送接收文件
scp xiedong@10.0.1.3:/home/xiedong/test.cpp .  #从远程主机拷贝文件到当前目录
scp text.cpp xiedong@10.0.1.3:/home/xiedong/   #将本地文件拷贝到远程主机

#通过其他server登录当前server -- ssh
ssh xiedong@10.0.1.4

#查看Linux的版本
lsb_release -a

#处理僵尸进程
ps aux | grep 'Z'(或直接用top) 查看僵尸进程号child_id
ps -o ppid= -p <child_id> 查看父进程parent_id
kill -9 parent_id
```



## 2. Vim 常用记录

```shell
# :%! 
	# 该指令可以把 buffer 作为 stdin 输入给一个程序，再用那个程序输出的内容替换 buffer
	# % 表示整个文件的所有行（即当前缓冲区 buffer 的内容），当然这里也能写其他方式，例如$ 最后行，1,100之类的
	# ! 表示将内容传递给外部程序
	# 例如 :%!sort 用vim内置的sort对所有行进行排序，:2，5!sort对[2, 5]行进行排序
```







