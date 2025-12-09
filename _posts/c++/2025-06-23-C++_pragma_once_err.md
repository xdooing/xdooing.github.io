---
title: pragma_once删除注释编译报错
date: 2025-06-23 00:00:00 +0800
categories: [C/C++]
tags: [pragma once]
pin: false


toc: true
comments: true

math: false
mermaid: true

typora-root-url: ../../../xdooing.github.io
---







偶尔刷到了一个C中，如何实现删除注释造成程序无法运行的帖子，自己想到的办法是 `#define true (__LINE__%2==0)`，这样应该能实现所要的效果。结果看到了一个关于pragma once bug的帖子，觉得比较有趣，记录一下。

[原文链接](https://www.zhihu.com/question/646854372/answer/1918463227303539528)



## 复现

要复现这个例子，我们首先需要分别在两个子目录中创建四个文件：

```bash
mkdir foo
mkdir bar
touch foo/impl.inc
touch foo/foo.h
touch bar/impl.inc
touch bar/bar.h
```

然后我们在头文件 `foo/foo.h` 中填入以下代码（注意第一行的注释也是要保留的）：

```c
// foo.h
#pragma once
#include "impl.inc"
```

在头文件 `bar/bar.h` 中填入以下代码（同样第一行的注释也需要保留）：

```c
// bar.h
#pragma once
#include "impl.inc"
```

两个 `impl.inc` 文件中的内容对于本例来说不重要，为了得到一个最小复现，我们在 `foo/impl.inc` 文件中填入以下代码：

```c
void foo(void) {}
```

在 `bar/impl.inc` 文件中填入以下代码：

```c
void bar(void) {}
```

然后我们建立一个 `main.c` 源文件并填入以下代码：

```c
#include "foo/foo.h"
#include "bar/bar.h"

int main(void) {
  foo();
  bar();
}
```

如果此时我们尝试在 Linux 平台上将 `main.c` 使用gcc编译为可执行文件，我们可以顺利编译通过并顺利运行。但如果现在我们通过执行下面这条指令删除 `foo/foo.h` 以及 `bar/bar.h` 文件中的那两行注释：

```bash
sed -i '/^\/\//d' foo/foo.h bar/bar.h
```

然后使用 gcc 再次尝试编译 `main.c` ，我们将得到编译错误，提示我们 `bar` 函数未声明：

![](/assets/blog_res/assets/20250623-144146.jpg)

结果就是删除两行注释，结果程序跑不起来了~~



## 原因

造成这个结果的原因是 gcc 处理 `#pragma once` 的方式。众所周知，`#pragma once` 是一个虽然不是标准但是仍被广泛应用的防止同一个头文件被多次包含的预处理指令。那么编译器如何识别一个新的头文件是否已经被包含过？

大部分编译器的做法是根据头文件在文件系统中的路径来进行识别，当两个头文件的路径不一致时则认为他们不是同一个头文件。然而 gcc 除此之外还有他自己的想法，当两个头文件的路径不一致时，gcc 会进一步尝试通过匹配头文件的内容以及头文件的最后修改时间戳进行重复包含检测。在匹配时间戳时，gcc 也只会使用秒级别的时间戳，不会使用更精确的时间戳。即，如果两个路径不同的头文件都只使用 `#pragma once` 进行保护，并且这两个头文件的内容和秒级别的时间戳都一致，那么 gcc 会认为他们是同一个头文件。

理解了 gcc 对 `#pragma once` 的处理方式，本例中出现编译错误的原因就很容易理解了。`foo/foo.h` 以及 `bar/bar.h` 完美符合刚才提到的要求：

- 他们都只使用 `#pragma once` 进行保护； 
- 他们的内容在删除注释后完全一致；
- 由于我们是通过 `sed` 工具批量对这两个文件中的注释进行删除的，因此删除注释后这两个文件的时间戳在秒级别的精度上大概率也是一致的。

因此 gcc 会认为他们是同一个头文件，在编译 `main.c` 时实际也就只会包含 `foo/foo.h` ，进而导致 `bar` 未声明错误。



------

这个例子的触发条件看起来非常离谱，那么为什么会在实际的场景中遇到呢？这是因为，在真实的场景中，`foo` 和 `bar` 目录下的代码都是由代码生成器自动生成的。代码生成器恰好生成了两份内容完全一致且时间戳也相同的头文件，当下游尝试同时包含这两个头文件时便产生了意想不到的编译错误。

还有一种情况：除了sed能造成时间戳一样以外，git checkout也能。于是刚写好code能编译，提交后自己切了分支又切回来，或者别人pull就挂。



**解决方法**：直接使用 `#ifndef... #define... #endif` 即可

