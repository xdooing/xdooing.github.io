---
title: Linux backtrace()系列函数
date: 2024-07-08 00:00:00 +0800
categories: [Linux]
tags: [Linux]
pin: false


toc: true
comments: true

math: false
mermaid: true

typora-root-url: ../../xdooing.github.io
---









## 1. 系列函数

backtrace()系列函数有3个：backtrace，backtrace_symbols，backtrace_symbols_fd。主要用于应用程序反调试（self-debugging）。

参见man 3 BACKTRACE，3个函数原型：

```c
#include <execinfo.h>

int backtrace(void **buffer, int size);
char **backtrace_symbols(void *const *buffer, int size);
void backtrace_symbols_fd(void *const *buffer, int size, int fd);
```

backtrace，backtrace_symbols，backtrace_symbols_fd在glibc 2.1以后就提供了。

3个函数是GNU 扩展（GNU extensions），因此只能用于GNU gcc/g++系列编译器。



## 2. backtrace()

backtrace() 返回调用程序的回溯（跟踪）信息，存储在由buffer指向的数组中。对于特定程序，backtrace就是一系列当前激活的函数调用（active function call）。

- 参数
  buffer 由buffer指向的数组，每一项都是void*类型，存储的是相应（调用函数的）栈帧的返回地址。
  size 指定存储在buffer中的地址最大数量。
- 返回值
  返回buffer中实际地址的数量，应当<=size。如果返回值 < size，那么完整的回溯信息被存储；如果返回值 = size，那么它可能被截断，最旧的栈帧可能没有返回。

## 3. backtrace_symbols()

backtrace() 返回一组地址，backtrace_symbols()象征性地翻译这些地址为一个描述地址的字符串数组。

- 参数
  buffer 一个字符串数组，由backtrace()返回的buffer，每项代表一个函数地址。backtrace_symbols()会用字符串描述每个函数地址，字符串包括：函数名称，一个16进制偏移（offset），实际的返回地址（16进制）。
  size 表明buffer中的地址个数。
- 返回值
  成功时，返回一个指向由malloc(3)分配的array；失败时，返回NULL。arrary是一个二维数组，该数组的每个元素 指向一个代表backtrace()返回的函数地址的符号信息的字符串，数组由函数内部调用malloc分配空间，必须由调用者free。
  **注意：指向字符串的指针的数组，不必释放，而且不应该释放。应该释放的是返回的二维数组指针。**

## 4. backtrace_symbols_fd()

backtrace_symbols_fd()的参数buffer、size同backtrace_symbos()，不同之处在于，backtrace_symbols_fd()并不会返回一个字符串数组给调用者，而是将字符串写入fd对应文件。backtrace_symbols_fd()也不会调用malloc分配二维数组空间，因此可应用于malloc可能会失败的情形。



## 5. 应用示例

用backtrace和backtrace_symbols，打印函数的调用栈信息。

注意：

- backtrace的实现依赖于栈指针（fp寄存器），编译时，任何非0优化等级（-On），或加入栈指针优化`-fomit-frame-pointer`参数后，将不能得到正确的程序调用栈信息。
- backtrace_symbols的实现需要符号名称的支持，编译时，需要加上-rdynamic选项（否则会无法获取func_name+offset）。



```c++
// backtrace.c
#include <stdio.h>
#include <execinfo.h>
#include <stdlib.h>
#include <unistd.h>

using namespace std;

void myfunc3()
{
    int j, nptrs;
#define SIZE 128
    void *buffer[100];
    char **strings;

    nptrs = backtrace(buffer, SIZE);
    printf("backtrace() return %d address\n", nptrs);

    strings = backtrace_symbols(buffer, nptrs);
    if (strings == NULL) {
        perror("backtrace_symbols");
        exit(EXIT_FAILURE);
    }

    for (j = 0; j < nptrs; j++)
        printf("%s\n", strings[j]);

    free(strings);
}

static void myfunc2()
{
    myfunc3();
}

void myfunc(int ncalls)
{
    if (ncalls > 1)
        myfunc(ncalls - 1);
    else
        myfunc2();
}

int main(int argc, char* argv[])
{
    if (argc != 2) {
        fprintf(stderr, "%s num-calls\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    myfunc(atoi(argv[1]));

/*
    printf("main address: %p\n", main);
    printf("myfunc address: %p\n", myfunc);
    printf("myfunc2 address: %p\n", myfunc2);
    printf("myfunc3 address: %p\n", myfunc3);
*/
    exit(EXIT_SUCCESS);
    return 0;
}
```

这里我们用g++编译器编译（当然也可以用gcc编译器）。

```shell
$ g++ -rdynamic -std=c++11 backtrace.c -o backtrace
```

结果为：

```
$ ./backtrace 2
backtrace() return 7 address
./backtrace(_Z7myfunc3v+0x1f) [0x400a8c]
./backtrace() [0x400b45]
./backtrace(_Z6myfunci+0x25) [0x400b6c]
./backtrace(_Z6myfunci+0x1e) [0x400b65]
./backtrace(main+0x59) [0x400bc7]
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xf5) [0x7f7170ed1f45]
./backtrace() [0x4009a9]
```

可以看到一共返回了7个地址，从上到下，可以推测出调用栈对应函数依次为：myfunc3、无名函数、myfunc2、myfunc1、main、__libc_start_main、无名函数。
而从`$ ./backtrace 2`，我们可以知道调用函数顺序为：main、myfunc、myfunc、myfunc2、myfunc3。与推测的函数栈调用顺序基本一致。
