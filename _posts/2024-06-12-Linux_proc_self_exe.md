---
title: Linux readlink与/proc/self/exe
date: 2024-06-12 00:00:00 +0800
categories: [Linux]
tags: [Linux]
pin: false


toc: true
comments: true

math: false
mermaid: true

typora-root-url: ../../xdooing.github.io
---







`ssize_t readlink(const char *path, char *buf, size_t bufsiz);`

`/proc/self/exe`是一个符号链接，代表当前程序的绝对路径
用readlink读取/proc/self/exe可以获取当前程序的绝对路径

```c++
int GetexeFilePath()
{
	char szFilePath [255];
	memset(szFilePath, 0, sizeof(szFilePath));
#ifdef _WIN__32
	::GetModuleFileName (NULL, szFilePath, 255);
	MY_LOG("_WIN__32 defined szFilePath=[%s]", szFilePath);
#else
	readlink ("/proc/self/exe", szFilePath, 255);
	MY_LOG("_WIN__32 not defined szFilePath=[%s]", szFilePath);
#endif
 
	return 0;
}
```

**/proc/self/exe的意义**

我们都知道可以通过/proc/$pid/来获取指定进程的信息，例如内存映射、CPU绑定信息等等。如果某个进程想要获取本进程的系统信息，就可以通过进程的pid来访问/proc/$pid/目录。但是这个方法还需要获取进程pid，在fork、daemon等情况下pid还可能发生变化。为了更方便的获取本进程的信息，linux提供了/proc/self/目录，这个目录比较独特，不同的进程访问该目录时获得的信息是不同的，内容等价于/proc/本进程pid/。进程可以通过访问/proc/self/目录来获取自己的系统信息，而不用每次都获取pid。
