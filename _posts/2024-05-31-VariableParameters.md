---
title: C/C++可变参数
date: 2024-05-31 00:00:00 +0800
categories: [C/C++]
tags: [编程]
pin: false


toc: true
comments: true

math: false
mermaid: true

typora-root-url: ../../xdooing.github.io
---





## 1. C语言可变参数

c中典型的变参函数是我们已经用得非常熟练的printf和scanf函数，它们都可以接受不确定个数的参数，它们的函数声明形式如下：

```c
int printf(const char *format, ...);
int scanf(const char *format, ...);
```

两个函数后面的`...`是占位符，并不是参数，而是告诉编译器，该函数是变参函数，不管该函数使用时参数有多少个，都对其一一做压栈处理，这就实现了变参函数。

变参函数的实现与**栈**密切相关，栈是一种数据结构，是一种只能在一端进行插入和删除的特殊线性表。栈的存储规律是先进后出（First In Last Out），先入栈的数据被压入栈底，最后入栈的数据在栈顶，需要读数据时则从栈顶开始弹出数据（最后一个数据被第一个弹出来）。

一个栈中，栈底是固定的，而栈顶是浮动的，对栈的插入和删除操作，不需要改变栈底的位置。

在计算机系统中，栈则是一个具有以上属性的动态内存区域。程序可以将数据压入栈中，也可以将数据从栈顶弹出。在i386机器中，栈底位于高地址，栈顶位于低地址，压栈（PUSH）使得栈顶地址变小，弹栈（POP）（也可以称为退栈）使栈顶地址变大。

栈在程序的运行中有着举足轻重的作用。栈可以用来在函数调用时存储断点信息，做递归时要用到栈。最重要的是栈保存了一个函数调用时所需要的维护信息，通常称为堆栈帧，保存函数的返回地址和参数，以及函数的局部变量。

下面我们来分析变参函数参数的压栈过程。一般来说，**函数参数的入栈顺序是从右向左的，意味着，最右边的参数最先入栈**，位于高地址处，而最左边的参数最后入栈，位于低地址。下面通过一个具体的例子来看函数的压栈操作。

```c
#include <stdio.h>
void myPrint(int n, ...) {
    int *p, i;
    p = &n + 1;
    for (i = 0; i < n; i++)
    {
        printf("%d ", *(p + i));
    }
    printf("\n");
    return;
}
 
int main() {
    myPrint(4, 1, 2, 3, 4); // 1 2 3 4
    return 0;
}
```

以上代码，首先在myPrint函数中使用占位符`...`，因此该函数在编译时被当成变参函数来处理，对该函数调用中的参数将一一进行压栈。上述代码定义了一个int型指针p，由于函数参数的压栈顺序是从右向左，参数的存储的地址由高地址到低地址，所以`p = &n + 1`得到的是指向第一个可变参数的地址，接下来通过循环一一取出函数中的参数。

在使用变参函数时，`...`前面至少要有一个普通的参数。必须知道参数什么时候结束，如果没有给出变参函数的个数，直接结出第一个参数，则必须约定一个参数作为结束标志。

但是这里需要注意的是，上面的代码只在32位系统下能够运行成功，64位系统下无法顺序输出，因此需要安装32位的库和运行环境：

```shell
# Linux查看系统架构（结果为 x86_64）
uname -m
# 安装32位的库和运行环境
apt-get install gcc-multilib
```

安装环境完成后，gcc增加`-m32`参数，表示编译32位架构下的可执行程序，`gcc myPrint.cpp -m32 -o myPrint`

而在64位系统中，要实现以上功能，需要借助 <stdarg.h> 头文件中的 va_list 类型以及相关宏函数 va_start()、va_arg() 和 va_end() 来处理可变参数。

```c
#include<cstdarg>  // C中是<stdarg.h>

// va_list是一种数据类型，args用于持有可变参数。
va_list args;

// 调用va_start并传入两个参数：第一个参数为va_list类型的变量
// 第二个参数为"..."前最后一个参数名
// 将args初始化为指向第一个参数（可变参数列表）
va_start(args, paramN);

// 检索参数，va_arg的第一个参数是va_list变量，第二个参数指定返回值的类型
// 每一次调用va_arg会获取当前的参数，并自动更新指向下一个可变参数。
va_arg(args,type);

// 释放va_list变量
va_end(args);
```

接下来实现一下myPrint函数：

```c
void myPrint(int n, ...) {
    int i, arg;
	va_list args;
    va_start(args, n);
    for(i = 0; i < n; ++i) {
        arg = va_arg(args, int);
        printf("%d ", arg);
    }
    printf("\n");
    va_end(args);
}
```



上述函数可能实现方式为：

```c
typedef char *va_list;
#define _INTSIZEOF(n) ((sizeof(n) + sizeof(int) - 1) & ~(sizeof(int) - 1))
#define va_start(ap, v) (ap = (va_list)&v + _INTSIZEOF(v))
#define va_arg(ap, t) (*(t *)((ap += _INTSIZEOF(t)) - _INTSIZEOF(t)))
#define va_end(ap) (ap = (va_list)0)
```

一个一个来解释：

- `#define _INTSIZEOF(n) ((sizeof(n) + sizeof(int) - 1) & ~(sizeof(int) - 1))`

  > 函数的压栈是按照4字节对齐的，小于4字节的统统按4字节对齐来入栈，而这里的_INTSIZEOF宏，就是用于实现内存中的字节对齐操作。

- `#define va_start(ap, v) (ap = (va_list)&v + _INTSIZEOF(v))`

  > 作用是先得到变量v的地址，然后将其转化成char型指针，再加上v对齐之后所占用的内存大小，使指针指向下一个参数。注意此时的指针为char类型，所以接下来在使用va_arg(ap, t)时要将其强制转换为当前参数类型t的指针。

- `#define va_arg(ap, t) (*(t *)((ap += _INTSIZEOF(t)) - _INTSIZEOF(t)))`

  > ap += _INTSIZEOF(t)得到的是下一个参数的地址，再减去_INTSIZEOF(t)得到当前参数的地址。通过一个for循环就可以一一取出其中压栈的所有参数。

- `#define va_end(ap) (ap = (va_list)0)`

  > 清除指针，表示在接下来的部分不再使用该指针变量



上述可以用在一般函数上，但是无法使用在宏定义中，如果一定要在宏定义中使用，需要配合 ***\_\_VA_ARGS__***，示例如下：

```c
//#define CALC(fmt, ...) func(fmt, ...) //错误的使用
#define CALC(fmt, ...) func(fmt, __VA_ARGS__)
int main() {
    CALC("%d %f %s\n", 1, 2.0f, "hello world");
    return 0;
}
```



**实现类似printf函数：**

```c
#include<stdarg.h>
#include<cstdio>

void myPrintf(const char *fmt, ...) {
    char c;
    va_list list;
    va_start(list, fmt);
    while (*fmt != '\0') {
        c = *fmt;
        if (c != '%') {
            putchar(c);
        }else {
            fmt++;
            switch (*fmt) {
            case 'd':
                printf("%d", va_arg(list, int));        // #1
                break;
            case 'f':
                printf("%f", va_arg(list, double));
                break;
            case 's':
                printf("%s", va_arg(list, const char*));
                break;
            default:
                break;
            }
        }
        fmt++;
    }
    va_end(p);
}

int main() {
    int a = 5;
    double d = 3.14;
    const char* s = "test";
    myPrintf("a = %d d = %f s = %s\n", a, d, s); // a = 5 d = 3.140000 s = test
    
    return 0;
}
```

注意上述代码中#1处，64位与32位环境下都可以这样写，但是下面的写法只适用于32位环境：

```c
case 'd':
    printf("%d", *((int*)list));
	va_arg(list, int);
    break;
```

TODO：搞清楚原因。





## 2. C++可变参数



### 2.1 可变参数函数

#### 1. initializer_list形参

如果函数的实参数量未知但是全部实参的类型都相同，我们可以使用initializer_list类型的形参（C++11新标准）。和vector一样，initializer_list也是一种模板类型。

下面给出一个例子，需要注意的是，含有initializer_list形参的函数也可以同时拥有其他形参。另外，如果想给initializer_list形参传递一个实参的序列，必须把序列**放在一对花括号内**：

```c++
string func(initializer_list<string> li) {
	string str("");
	for(auto beg=li.begin(); beg!=li.end(); ++beg)
		str += *beg;
	return str;
}

int main() {
	cout << func({"This"," ","is"," ","C++"}) << endl;
	return 0;
}
```

#### 2. 占位符...形参

与c语言一致，见上文。

### 2.2 可变参数模板

C++11 中引入了新的功能，***可变参数模版***，语法如下：

```c++
template <typename T, typename ... Args>
void func(T t,Args ... args);
```

这里面，***Args*** 称之为模板参数包（template parameter pack），表示模板参数位置上的变长参数，

***args*** 称之为函数参数包（function parameter pack），表示函数参数位置上的变长参数

可以使用 ***sizeof...()*** 获取可变参数数目

先看一个示例：

```c++
template<typename... Args>
void print(Args... args) {
    int num = sizeof...(args);
    printf("%d\n", num);
}
int main() {
    print(1, 2, "123", 4); // 4
    return 0;
}
```

遍历可变参数的时候，有两种遍历方式：

#### 1. 递归遍历

可变参数一般使用递归的方式进行遍历，利用模板的推导机制，每次从可变参数中取出第一个元素，直到包为空

缺点：递归毕竟是使用栈内存，过多的递归层级容易导致爆栈的发生

```c++
template<typename T>
void print(const T& val) {
    cout << val << endl;
}
template<typename T, typename... Args>
void print(const T &value, Args... args) {
    cout << value << endl;
    print(args...);
}
int main() {
    print(1, 2, "333", 4);
    return 0;
}
```



#### 2. 非递归遍历

利用 ***std::initializer_list*** ，即初始化列表展开可变参数，这里是C++17引入的折叠表达式。

```c++
template<typename T>
void run(const T &t)
{
    cout << t << endl;
}
template<typename... Args>
void print(Args... args)
{
    std::initializer_list<int>{(run(args), 0)...};
}
int main()
{
    print(1, 2, "333as", 4);
    return 0;
}
```
