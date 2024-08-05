---
title: 父类引用指向子类对象
date: 2024-08-05 00:00:00 +0800
categories: [C/C++]
tags: [引用]
pin: false


toc: true
comments: true

math: false
mermaid: true

typora-root-url: ../../xdooing.github.io
---











最近在进行开发的时候，看到了一个之前没有见过的写法，即父类的引用指向子类对象，指针指向子类对象之前经常见，引用确实少见，因此记录一下：

例如现在有如下代码示意：

```c++
Base tmp;
Son son;
Base& ref = son;
ref = tmp;
```

当执行`ref = tmp`之后，son对象会有什么变化呢？

在 C++ 中，赋值操作 `ref = tmp;` 会调用 `Base` 类的赋值运算符 `operator=`。这意味着赋值是发生在 `Base` 类的层面，而不是 `Son` 类。由于 `ref` 是对 `Son` 对象 `son` 的引用，这个赋值操作将 `tmp` 对象的值赋给了 `son` 对象的 `Base` 部分，但不会改变 `son` 对象中 `Base` 部分之外的数据。如果 `Son` 类有其他成员数据或状态，这些不会被 `base` 对象所影响。

测试：

```c++
struct Base {
    int a;
    const char* s;
};

struct Son : public Base {
    int t;
};

int main() {
    Base tmp;
    tmp.a = 20;
    tmp.s = "tmp";
    
    Son son;
    son.a = 111;
    son.s = "sss";
    son.t = 222;
    printf("before ref: [%d %s %d]\n", son.a, son.s, son.t);
    
    Base& ref = son;
    ref = tmp;
    printf("after ref: [%d %s %d]\n", son.a, son.s, son.t);
    
    return 0;
}
```

output:

```
before ref: [111 sss 222]
after ref: [20 tmp 222]
```

赋值操作对子类Son独有的成员变量t没有影响，只覆盖了从Base类继承过来的成员变量。
