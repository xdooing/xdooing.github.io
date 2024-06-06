---
title: C++对象内存模型
date: 2024-06-05 00:00:00 +0800
categories: [C/C++]
tags: [编程]
pin: true


toc: true
comments: true

math: false
mermaid: true

typora-root-url: ../../xdooing.github.io
---















> 参考链接：https://www.miaoerduo.com/2023/01/19/cpp-object-model/



## 前言

> C++的三大特性是封装、继承和多态。
>
> 本节内容将详细介绍单一继承、多重继承、重复继承、虚拟继承等不同的继承方式的对象内存模型。

首先需要了解的是，编译器是会进行内存对齐优化的，例如：

```c++
class withAlign {
    int a;
    char c;
};

#pragma pack(push, 1)
class withoutAlign {
    int a;
    char c;
};
#pragma pack(pop)

int main() {
    std::cout << sizeof(withAlign) << std::endl;
    std::cout << sizeof(withoutAlign) << std::endl;
    return 0;
}
// output
// 8
// 5
```

内存对齐的类大小为8（按int 4字节对齐），未对齐的为5（int + char），以下内容为了更容易理解，全部默认使用1字节对齐。

测试环境：

```shell
# lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 20.04.6 LTS
Release:        20.04
Codename:       focal
# gcc --version
gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0
```



## 数据存放

在C++类中，变量有两种

1. static：也称为类变量、类静态变量，由同一个类的所有实例共享。
2. non-static，也称为成员变量，每个类的实例均持有自己的一份。

类成员函数由三种：

1. static：静态函数，属于该类，不需要实例也可以调用。
2. non-static：成员函数，一般需要实例才可以调用。
3. virtual：虚函数，用于实现多态。

看代码：

```c++
#include <iostream>

#pragma pack(push, 1)
class Base {
public:
    Base(int d) : data_(d) { static_num_++; }
    virtual ~Base() {}
    static int getStaticNum() { return static_num_; }
    int getData() { return data_; }
    virtual void print() {
        std::cout << "[Base] address: " << this
                  << " data: " << this->data_
                  << std::endl;
    }
private:
    int data_;
    static int static_num_;
};
int Base::static_num_ = 0;
#pragma pack(pop)


int main() {
    Base a(100);
    Base b(200);

    std::cout << "Base size: " << sizeof(Base) << std::endl;
    a.print();
    b.print();

    return 0;
}

// output
// Base size: 12
// [Base] address: 0x7ffdb1751af0 data: 100
// [Base] address: 0x7ffdb1751afc data: 200
```

问题来了，为什么Base类的大小为12个字节？先来看张图：

<img src="/assets/blog_res/assets/objectmodel.png" alt="C++ object model" style="zoom: 67%;" />

Base类中的成员与函数，在内存中的存放方式为：

1. static data：单独存放，存放在**静态存储区**，不计入class的size中。
2. non-static data：在实例存放，计入class的size。
3. static function：单独存放
4. non-static function：单独存放
5. virtual function：单独存放，一个具体类对应的虚函数会整合进一个表中，表中存放了虚函数的指针等信息，实例存放一到多个指向虚表的指针。

这里可以看到Base类的size为12，其实就是存放了`vptr`和`int data_`这两个数据导致的。而我们把析构函数与print函数前面的virtual关键字去掉之后，Base类的大小就变为了4，少了8个字节的vptr指针。

用代码具体看看Base的内存布局：

```c++
#include <iostream>

#pragma pack(push, 1)
class Base {
public:
    Base(int d) : data_(d) { static_num_++; }
    virtual ~Base() {std::cout << "destructor: " << this << std::endl;}
    static int getStaticNum() { return static_num_; }
    int getData() { return data_; }
    virtual void print() {
        std::cout << "[Base] address: " << this
                  << " data: " << this->data_
                  << std::endl;
    }
private:
    int data_;
    static int static_num_;
};
int Base::static_num_ = 0;
#pragma pack(pop)


int main() {
    Base* a = new Base(100);
    Base* b = new Base(200);

    uint64_t ptr_list[2] = {(uint64_t)a, (uint64_t)b};

    for(int i = 0; i < 2; ++i) {
        uint64_t ptr = ptr_list[i];
        std::cout << "base " << i << " address: " << (void*)ptr << std::endl;

        // 指向vptr的地址，对象的前8个字节存放的是vptr
        uint64_t point2vptr = ptr;
        // 指向data的地址
        uint64_t point2data = ptr + 8;
        // 获取vptr具体数据，转换为(uint64_t*)是因为vptr中存放的是指针
        uint64_t vptr = *(uint64_t*)point2vptr;
        // 获取data具体数据
        int data = *(int*)point2data;

        std::cout << "  vptr address: " << (void*)point2vptr << " vptr: " << (void*)vptr << std::endl;
        std::cout << "  data address: " << (void*)point2data << " data: " << data << std::endl;

        // 关于虚函数表
        // 1. 虚表中存放了多个指针，顺序是 offset，type_info，virtual_func1，virtual_func2...
        // 2. 虚函数在虚表中的顺序与声明的顺序一致
        // 3. 实例的vptr指向的是第一个virtual_func，而不是vtable的真正起始位置
        // 4. 对于虚析构函数，GCC会生成两个虚函数
        uint64_t* vtable = (uint64_t*)vptr;         // 严格意义上来讲应该是 vptr-16
        uint64_t offset_ptr = vtable[-2];
        uint64_t type_info_ptr = vtable[-1];
        uint64_t destructor_func1_ptr = vtable[0];  // 析构函数，不调用delete()
        uint64_t destructor_func2_ptr = vtable[1];  // 析构函数，调用delete()
        uint64_t print_func_ptr = vtable[2];

        std::cout << "vtable address: " << vtable << std::endl;
        std::cout << "  offset address: " << (void*)offset_ptr << std::endl;
        std::cout << "  type_info address: " << (void*)type_info_ptr << std::endl;
        std::cout << "  destructor1 address: " << (void*)destructor_func1_ptr << std::endl;
        std::cout << "  destructor2 address: " << (void*)destructor_func2_ptr << std::endl;
        std::cout << "  print_func address: " << (void*)print_func_ptr << std::endl;

        // print函数的函数指针，参数是一个指针，返回值为void
        typedef void(*print_func_type)(void*);
        // 执行print函数
        std::cout << "call with object" << std::endl;
        ((Base*)ptr)->print();                          // 1. 对象调用
        std::cout << "call with vtable" << std::endl;
        ((print_func_type)print_func_ptr)((void*)ptr);  // 2. vtable调用

    }

    return 0;
}
```

输出结果：

```
base 0 address: 0x557876f30eb0
  vptr address: 0x557876f30eb0 vptr: 0x557875f04d40
  data address: 0x557876f30eb8 data: 100
vtable address: 0x557875f04d40
  offset address: 0
  type_info address: 0x557875f04d58
  destructor1 address: 0x557875f0271e
  destructor2 address: 0x557875f0277a
  print_func address: 0x557875f027aa
call with object
[Base] address: 0x557876f30eb0 data: 100
call with vtable
[Base] address: 0x557876f30eb0 data: 100

base 1 address: 0x557876f30ed0
  vptr address: 0x557876f30ed0 vptr: 0x557875f04d40
  data address: 0x557876f30ed8 data: 200
vtable address: 0x557875f04d40
  offset address: 0
  type_info address: 0x557875f04d58
  destructor1 address: 0x557875f0271e
  destructor2 address: 0x557875f0277a
  print_func address: 0x557875f027aa
call with object
[Base] address: 0x557876f30ed0 data: 200
call with vtable
[Base] address: 0x557876f30ed0 data: 200
```

以上代码有几点需要说明：

1. 测试环境是64位的，因此此处直接将指针转换为uint64_t类型，打印地址的时候转换为void*

2. 为什么print函数通过vtable调用的时候，需要有参数，实际上是因为所有类的成员函数在编译期会被编译器重构成非成员函数，即将`this`指针作为函数的第一个参数，这样在函数中通过`this`指针就能找到属于该对象的其他数据成员了，因此此处传入的参数是`(void*)ptr`

   > 第二点存疑，目前不是特别清楚，暂且先这样理解，以后再搞清楚



使用GCC查看内存布局

```shell
g++ -O0 -std=c++11 -fdump-class-hierarchy -fsanitize=address test.cpp
```

- `-O0`：表示不做编译器优化
- `-fdump-class-hierarchy`: 会dump出内存布局
- `-std=c++11`: 使用C++11标准
- `-fsanitize=address`:开启内存检查







