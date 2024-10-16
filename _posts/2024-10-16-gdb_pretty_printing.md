---
title: VS Code中开启gdb的pretty-printer功能
date: 2024-10-16 00:00:00 +0800
categories: [gdb]
tags: [vscode]
pin: false


toc: true
comments: true

math: false
mermaid: true

typora-root-url: ../../xdooing.github.io
---







## 1. pretty-printer

> C++的STL容器的实现并不直观，直接使用gdb之类的debugger查看内存是需要周转多次才能看到具体的内容的。
>
> 在Visual Studio之类的IDE中内置了一些脚本，用来较为友好的显示容器内的元素。
>
> GDB的pretty-printer脚本提供了类似的功能。

然而虽然pretty-printer是GDB官方提供的，但是并没有默认安装/开启，所以需要手动安装。

- 新建文件 ~/.gdb

- 访问https://gcc.gnu.org/git/?p=gcc.git;a=tree;f=libstdc%2B%2B-v3/python/libstdcxx;hb=HEAD 并下载v6目录下的printers.py脚本到~/.gdb

- 配置~/.gdbinit文件（就是一个bash脚本，每次执行gdb操作的时候都会自动执行这个文件），然后往文件中写入：

  ```python
  python
  import sys
  sys.path.insert(0, '/home/xxx/share/gcc/python') # 按实际情况修改目录
  from libstdcxx.v6.printers import register_libstdcxx_printers
  register_libstdcxx_printers (None)
  end
  ```



## 2. 配置vscode launch.json

在launch.json中的`”MIMode“：“gdb”`同一级添加setupCommands：

```
"setupCommands": [
    { 
        "text": "-enable-pretty-printing", 
        "description": "enable pretty printing",
        "ignoreFailures": true
    }
]
```

如果以上操作没有问题，调试的时候就可以看到stl容器能够被正确显示了。



## 3. 其他情况

如果确认自己的服务器上已经有v6/printers.py文件，那么有一种比较简单的方法，直接修改launch.json：

```
"setupCommands": [
    { 
        "text": "-enable-pretty-printing", 
        "description": "enable pretty printing",
        "ignoreFailures": true
    }，
    {
    	"description": "print stl",
    	"text": "python \
		import sys; \
		sys.path.insert(0, '/home/xxx/share/gcc/python'); \
		from libstdcxx.v6.printers import register_libstdcxx_printers; \
		register_libstdcxx_printers (None)", 
        "ignoreFailures": true
    }
]
```

当然，path要替换成自己v6文件夹所在的目录，也就是printers.py脚本所在的python目录。
