---
title: C++对象内存模型
date: 2024-06-05 00:00:00 +0800
categories: [C/C++]
tags: [编程]
pin: false


toc: true
comments: true

math: false
mermaid: true

typora-root-url: ../../../xdooing.github.io
---





> 参考链接：
>
> https://www.miaoerduo.com/2023/01/19/cpp-object-model/
>
> https://www.cnblogs.com/pandamohist/p/13882020.html



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
        
        // 调用析构函数，调用第二个
        typedef void(*destructor_func_type)(void*);
        ((destructor_func_type)destructor_func2_ptr)((void*)ptr);

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
destructor: 0x602000000010

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
destructor: 0x602000000030
```

以上代码有几点需要说明：

1. 测试环境是64位的，因此此处直接将指针转换为uint64_t类型，打印地址的时候转换为void*

2. 为什么print函数通过vtable调用的时候，需要有参数，实际上是因为所有类的成员函数在编译期会被编译器重构成非成员函数，即将`this`指针作为函数的第一个参数，这样在函数中通过`this`指针就能找到属于该对象的其他数据成员了，因此此处传入的参数是`(void*)ptr`

   > 第二点存疑，目前不是特别清楚，暂且先这样理解，以后再搞清楚



使用GCC查看内存布局

```shell
g++ -O0 -std=c++11 -fdump-lang-class -fsanitize=address test.cpp
```

- `-O0`：表示不做编译器优化
- `-fdump-lang-class`: 会dump出内存布局。GCC7.x及更早的版本中为 `-fdump-class-hierarchy`，在GCC8.0中被删除
- `-std=c++11`: 使用C++11标准
- `-fsanitize=address`:开启内存检查

来看看dump出来的结果：

```
Vtable for Base
Base::_ZTV4Base: 5 entries
0     (int (*)(...))0
8     (int (*)(...))(& _ZTI4Base)
16    (int (*)(...))Base::~Base
24    (int (*)(...))Base::~Base
32    (int (*)(...))Base::print

Class Base
   size=12 align=1
   base size=12 base align=1
Base (0x0x7f7315fa41e0) 0
    vptr=((& Base::_ZTV4Base) + 16)
```

可以看出来，Base类大小为12，按照1字节对齐。vptr指向了虚表首地址+16的位置。并且有两个`Base:~Base`的虚函数。

**总结：**

1. 两个Instance本身的地址和vptr/data的地址均不同，说明这部分数据确实是存放在实例本身的。
2. 虚表和虚函数的地址都不变，说明被所有实例共享。
3. 类的成员函数本质上也是普通函数，只是默认有了个this指针，通过vtable的直接调用也可以证实。
4. 虚析构函数会生成两个虚函数，前者是对象析构但不调用`delete()`，相当于手动调用析构函数`obj->~Base()`，后者是析构且调用`delete()`，相当于`delete obj`。将案例中的析构改为调用第一个的话，就会报内存泄露的错误了。[参考 CXX API](https://itanium-cxx-abi.github.io/cxx-abi/abi.html#vtable-components)



## 继承

### 1. 单继承

来看代码，B继承A，C继承B

```c++
#include <iostream>

#pragma pack(push, 1)
class A {
public:
    A(int a) : data_(a) {}
    virtual ~A() { std::cout << "[A] destructor: " << this << std::endl; }
    virtual void print() {
        std::cout << "[A] address: " << this
                  << " a: " << &(data_) << " " << data_
                  << std::endl;
    }
public:
    int data_;
};

class B : public A {
public:
    B(int a, int b) : A(a), data_(b) {}
    virtual ~B() { std::cout << "[B] destructor: " << this << std::endl; }
    virtual void printB() {
        std::cout << "[B] address: " << this
                  << " a: " << &(A::data_) << " " << A::data_
                  << " b: " << &(data_) << " " << data_
                  << std::endl;
    }
public:
    int data_;
};

class C : public B {
public:
    C(int a, int b, int c) : B(a, b), data_(c) {}
    virtual ~C() { std::cout << "[C] destructor: " << this << std::endl; }
    virtual void print() {
        std::cout << "[C] address: " << this
                  << " a: " << &(A::data_) << " " << A::data_
                  << " b: " << &(B::data_) << " " << B::data_
                  << " c: " << &(data_) << " " << data_
                  << std::endl;
    }
public:
    int data_;
};
#pragma pack(pop)

int main() {
    std::cout << "sizeof A: " << sizeof(A) 
              << " B: " << sizeof(B)
              << " C: " << sizeof(C) << std::endl;
    
    A* a = new A(100);
    A* b = new B(100, 200);
    A* c = new C(100, 200, 300);

    a->print();         // A print
    b->print();         // A print
    ((B*)b)->printB();  // B print
    c->print();         // C print
    ((B*)c)->printB();  // B print

    delete a;
    delete b;
    delete c;

    return 0;
}
```

输出结果为：

```
sizeof A: 12 B: 16 C: 20
[A] address: 0x602000000010 a: 0x602000000018 100
[A] address: 0x602000000030 a: 0x602000000038 100
[B] address: 0x602000000030 a: 0x602000000038 100 b: 0x60200000003c 200
[C] address: 0x603000000010 a: 0x603000000018 100 b: 0x60300000001c 200 c: 0x603000000020 300
[B] address: 0x603000000010 a: 0x603000000018 100 b: 0x60300000001c 200
[A] destructor: 0x602000000010
[B] destructor: 0x602000000030
[A] destructor: 0x602000000030
[C] destructor: 0x603000000010
[B] destructor: 0x603000000010
[A] destructor: 0x603000000010
```

GCC查看一下内存布局：

```
Vtable for A
A::_ZTV1A: 5 entries
0     (int (*)(...))0
8     (int (*)(...))(& _ZTI1A)
16    (int (*)(...))A::~A
24    (int (*)(...))A::~A
32    (int (*)(...))A::print

Class A
   size=12 align=1
   base size=12 base align=1
A (0x0x7f0fa4c3a1e0) 0
    vptr=((& A::_ZTV1A) + 16)
    
Vtable for B
B::_ZTV1B: 6 entries
0     (int (*)(...))0
8     (int (*)(...))(& _ZTI1B)
16    (int (*)(...))B::~B
24    (int (*)(...))B::~B
32    (int (*)(...))A::print
40    (int (*)(...))B::printB

Class B
   size=16 align=1
   base size=16 base align=1
B (0x0x7f0fa4c48d00) 0
    vptr=((& B::_ZTV1B) + 16)
  A (0x0x7f0fa4c3af00) 0
      primary-for B (0x0x7f0fa4c48d00)

Vtable for C
C::_ZTV1C: 6 entries
0     (int (*)(...))0
8     (int (*)(...))(& _ZTI1C)
16    (int (*)(...))C::~C
24    (int (*)(...))C::~C
32    (int (*)(...))C::print
40    (int (*)(...))B::printB

Class C
   size=20 align=1
   base size=20 base align=1
C (0x0x7f0fa4c48ea0) 0
    vptr=((& C::_ZTV1C) + 16)
  B (0x0x7f0fa4c48f08) 0
      primary-for C (0x0x7f0fa4c48ea0)
    A (0x0x7f0fa4c87480) 0
        primary-for B (0x0x7f0fa4c48f08)
```

**总结**

1. A和之前的Base一样，没有啥好说的。大小`12 = vptr+int`。
2. B继承A。同时B也定义了自己的成员变量（虽然和A的相同，但二者不是同一个变量，可以通过obj->A::data_来访问父类的对象）。因此大小是`16 = vptr + A::int + B::int`。
3. C继承B。也定义了自己的成员变量。因此大小是`20 = vptr + A::int + B::int + C::int`。
4. 通过每个`print`和`printB`的打印结果可以看出，派生类先存放了自己的基类的数据，之后才存放自己的数据

**关于虚表**

每个类都有且只有一个虚表对象。

1. A和Base一样就不解释了。
2. B继承了A的`print`方法，同时自己又定义了`printB`方法，因此B复制了A的虚表结构，除了改了析构函数的地址外，还新增了`printB`的指针。
3. C继承了B，同时覆盖了`print`方法。因此C复制了B的虚表，修改了析构函数，并修改了`print`函数的指针。
4. 可以总结个规律：**单继承下，派生类有且只有一个虚表，相当于直接将基类的虚表复制一次，替换掉自己的覆盖的虚函数，并追加自己新增的虚函数。**

### 2. 多继承

多继承比单继承复杂了很多。而且多继承一致被很多人诟病，像Java就直接不支持多继承。这里我们不考虑基类重名等情况。

来看代码，C继承A和B

```c++
#include <iostream>

#pragma pack(push, 1)
class A {
public:
    A(int a) : data_(a) {}
    virtual ~A() { std::cout << "[A] destructor: " << this << std::endl; }
    virtual void printA1() {
        std::cout << "[A1] address: " << this
                  << " " << &(data_) << " " << data_
                  << std::endl;
    }
    virtual void printA2() {
        std::cout << "[A2] address: " << this
                  << " " << &(data_) << " " << data_
                  << std::endl;
    }
public:
    int data_;
};

class B {
public:
    B(int a) : data_(a) {}
    virtual ~B() { std::cout << "[B] destructor: " << this << std::endl; }
    virtual void printB1() {
        std::cout << "[B1] address: " << this
                  << " " << &(data_) << " " << data_
                  << std::endl;
    }
    virtual void printB2() {
        std::cout << "[B2] address: " << this
                  << " " << &(data_) << " " << data_
                  << std::endl;
    }
public:
    int data_;
};

class C : public A, public B {
public:
    C(int a, int b, int c) : A(a), B(b), data_(c) {}
    virtual ~C() { std::cout << "[C] destructor: " << this << std::endl; }
    virtual void printB2() {
        std::cout << "[C B2] address: " << this
                  << " a: " << &(A::data_) << " " << A::data_ << " "
                  << " b: " << &(B::data_) << " " << B::data_ << " "
                  << " c: " << &data_     << " " << data_ << std::endl;
    }
    virtual void printC() {
        std::cout << "[C] address: " << this
                  << " a: " << &(A::data_) << " " << A::data_ << " "
                  << " b: " << &(B::data_) << " " << B::data_ << " "
                  << " c: " << &data_     << " " << data_ << std::endl;
    }

public:
    int data_;
};

#pragma pack(pop)


int main() {
    std::cout << "sizeof A: " << sizeof(A) 
              << " B: " << sizeof(B)
              << " C: " << sizeof(C) << std::endl;
    
    C* c = new C(100, 200, 300);

    std::cout << "\n" << "call with C*" << std::endl;
    c->printA1();
    c->printA2();
    c->printB1();
    c->printB2();
    c->printC();

    // for A
    std::cout << "\n" << "call with dynamic_cast<A*>: " << dynamic_cast<A*>(c) << std::endl;
    dynamic_cast<A*>(c)->printA1();
    dynamic_cast<A*>(c)->printA2();
    std::cout << "\n" << "call with static_cast<A*>: " << static_cast<A*>(c) << std::endl;
    static_cast<A*>(c)->printA1();
    static_cast<A*>(c)->printA2();
    std::cout << "\n" << "call with reinterpret_cast<A*>: " << reinterpret_cast<A*>(c) << std::endl;
    reinterpret_cast<A*>(c)->printA1();
    reinterpret_cast<A*>(c)->printA2();

    // for B
    std::cout << "\n" << "call with dynamic_cast<B*>: " << dynamic_cast<B*>(c) << std::endl;
    dynamic_cast<B*>(c)->printB1();
    dynamic_cast<B*>(c)->printB2();
    std::cout << "\n" << "call with static_cast<B*>: " << static_cast<B*>(c) << std::endl;
    static_cast<B*>(c)->printB1();
    static_cast<B*>(c)->printB2();
    std::cout << "\n" << "call with reinterpret_cast<B*>: " << reinterpret_cast<B*>(c) << std::endl;
    reinterpret_cast<B*>(c)->printB1();
    reinterpret_cast<B*>(c)->printB2();

    std::cout << std::endl;

    delete c;

    return 0;
}
```

输出结果：

```
sizeof A: 12 B: 12 C: 28

call with C*
[A1] address: 0x556c941792c0 0x556c941792c8 100
[A2] address: 0x556c941792c0 0x556c941792c8 100
[B1] address: 0x556c941792cc 0x556c941792d4 200
[C B2] address: 0x556c941792c0 a: 0x556c941792c8 100  b: 0x556c941792d4 200  c: 0x556c941792d8 300
[C] address: 0x556c941792c0 a: 0x556c941792c8 100  b: 0x556c941792d4 200  c: 0x556c941792d8 300

call with dynamic_cast<A*>: 0x556c941792c0
[A1] address: 0x556c941792c0 0x556c941792c8 100
[A2] address: 0x556c941792c0 0x556c941792c8 100

call with static_cast<A*>: 0x556c941792c0
[A1] address: 0x556c941792c0 0x556c941792c8 100
[A2] address: 0x556c941792c0 0x556c941792c8 100

call with reinterpret_cast<A*>: 0x556c941792c0
[A1] address: 0x556c941792c0 0x556c941792c8 100
[A2] address: 0x556c941792c0 0x556c941792c8 100

call with dynamic_cast<B*>: 0x556c941792cc
[B1] address: 0x556c941792cc 0x556c941792d4 200
[C B2] address: 0x556c941792c0 a: 0x556c941792c8 100  b: 0x556c941792d4 200  c: 0x556c941792d8 300

call with static_cast<B*>: 0x556c941792cc
[B1] address: 0x556c941792cc 0x556c941792d4 200
[C B2] address: 0x556c941792c0 a: 0x556c941792c8 100  b: 0x556c941792d4 200  c: 0x556c941792d8 300

call with reinterpret_cast<B*>: 0x556c941792c0
[A1] address: 0x556c941792c0 0x556c941792c8 100
[A2] address: 0x556c941792c0 0x556c941792c8 100

[C] destructor: 0x556c941792c0
[B] destructor: 0x556c941792cc
[A] destructor: 0x556c941792c0
```

dump一下内存布局：

```
Vtable for A
A::_ZTV1A: 6 entries
0     (int (*)(...))0
8     (int (*)(...))(& _ZTI1A)
16    (int (*)(...))A::~A
24    (int (*)(...))A::~A
32    (int (*)(...))A::printA1
40    (int (*)(...))A::printA2

Class A
   size=12 align=1
   base size=12 base align=1
A (0x0x7f37630261e0) 0
    vptr=((& A::_ZTV1A) + 16)

Vtable for B
B::_ZTV1B: 6 entries
0     (int (*)(...))0
8     (int (*)(...))(& _ZTI1B)
16    (int (*)(...))B::~B
24    (int (*)(...))B::~B
32    (int (*)(...))B::printB1
40    (int (*)(...))B::printB2

Class B
   size=12 align=1
   base size=12 base align=1
B (0x0x7f3763026e40) 0
    vptr=((& B::_ZTV1B) + 16)

Vtable for C
C::_ZTV1C: 14 entries
0     (int (*)(...))0
8     (int (*)(...))(& _ZTI1C)
16    (int (*)(...))C::~C
24    (int (*)(...))C::~C
32    (int (*)(...))A::printA1
40    (int (*)(...))A::printA2
48    (int (*)(...))C::printB2
56    (int (*)(...))C::printC
64    (int (*)(...))-12
72    (int (*)(...))(& _ZTI1C)
80    (int (*)(...))C::_ZThn12_N1CD1Ev
88    (int (*)(...))C::_ZThn12_N1CD0Ev
96    (int (*)(...))B::printB1
104   (int (*)(...))C::_ZThn12_N1C7printB2Ev

Class C
   size=28 align=1
   base size=28 base align=1
C (0x0x7f37630cf8c0) 0
    vptr=((& C::_ZTV1C) + 16)
  A (0x0x7f3763073420) 0
      primary-for C (0x0x7f37630cf8c0)
  B (0x0x7f3763073480) 12
      vptr=((& C::_ZTV1C) + 80)
```

**总结**

1. A和B一目了然，大小 `12 = data_ + vptr`
2. C继承A和B，顺序是先A再B。C中存放了A、B的数据，并且有两个虚指针（后续解释），因此大小为`28 = vptr + A::int + vptr + B::int + C::int`。
3. C的内存模型中有**2个**虚函数表。一个属于基类A，另一个属于基类B。同时派生类C的虚函数是**放在第一张虚函数表**中。按照先前的顺序：先基类，再派生，先声明，先存储。但是有虚函数的类要优先考虑。这里基类A和B还有派生类都含有虚函数。那么先看基类，按照先声明先存储的顺序，A基类相对B基类先声明，故基类A的虚函数表指针首先被存储，接着再是基类A的成员变量，然后是基类B的虚函数表指针，基类B的成员变量。最后是派生类。
