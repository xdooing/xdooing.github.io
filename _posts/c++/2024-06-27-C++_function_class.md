---
title: 使用std::function传参类成员函数时，不使用bind
date: 2024-06-27 00:00:00 +0800
categories: [C/C++]
tags: [C/C++]
pin: false


toc: true
comments: true

math: false
mermaid: true

typora-root-url: ../../../xdooing.github.io
---









## 1. 关于std::bind

std::bind是函数模板（是一个函数）。

使用std::bind可以将可调用对象和参数一起绑定，绑定后的结果使用std::function进行保存，并在我们需要的任何时候调用。

std::bind返回一个基于f的函数对象，其参数被绑定到args上。f的参数要么被绑定到值，要么被绑定到placeholders（占位符，如_1, _2, ..., _n）。即，可将std::bind函数看作一个通用的函数适配器，它接受一个可调用对象，生成一个新的可调用对象来“适应”原对象的参数列表。

std::bind通常有两大作用：

1. 将可调用对象与参数一起绑定为另一个std::function供调用

2. 将n元可调用对象转成m(m < n)元可调用对象，绑定一部分参数。(即减少可调用对象传入的参数)。这里需要使用std::placeholders

例如：

```c++
int TestFunc(int a, char c, float f) {
    cout << a << endl;
    cout << c << endl;
    cout << f << endl;
    return a;
}
auto bindFunc1 = bind(TestFunc, placeholders::_1, 'A', 100.1);
bindFunc1(10);
// or
auto bindFunc3 = bind(TestFunc, placeholders::_2, placeholders::_3, placeholders::_1);
bindFunc3(100.1, 30, 'C');
```



## 2. std::bind绑定类成员函数

bind绑定类成员函数时，第一个参数表示对象的成员函数的指针，第二个参数表示对象的地址。必须显示的指定&Foo::memFunc，因为编译器不会将对象的成员函数隐式转换成函数指针，所以必须在Foo::memFunc前添加&；使用对象成员函数的指针时，必须要知道该指针属于哪个对象，因此第二个参数为对象的地址 &foo；

```c++
struct Foo {
    void print_sum(int n1, int n2)
    {
        std::cout << n1+n2 << '\n';
    }
    int data = 10;
};
int main() 
{
    Foo foo;
    auto f = std::bind(&Foo::print_sum, &foo, 95, std::placeholders::_1);
    f(5); // 100
}
```



## 3. 成员对象还未创建

但是实际开发的时候，有时会有这样的情况出现，我们需要将成员函数传给std::function，正常情况下需要使用std::bind绑定成员对象与成员函数，但如果此时成员对象还未创建该怎么办呢？

```c++
class myClass {
public:
    myClass(int a) : data(a) {}
    void print(int a) {
		std::cout << "a = " << a << " " 
                  << "data = " << data 
                  << std::endl;
    }
private:
    int data;
};

int main() {
    auto lamFunc = [](const std::function<void(myClass*, int)>& func, int d) {
		myClass* obj = new myClass(t);
        func(obj, 100);
        delete obj;
    }
    std::function<void(myClass*, int)> func = &myClass::print;
    
    lamFunc(func, 1);
    lamFunc(func, 2);
    lamFunc(func, 3);

	return 0;
}
// output
// a = 100 data = 1
// a = 100 data = 2
// a = 100 data = 3
```

可以看到，当对象还未创建的时候，如果要将类成员函数传递给std::function，就可以在参数中显示的指定对象指针，并在合适的时候创建对象，调用成员函数时，也需要将对象指针放在第一个参数位置。原因是：

所有类的成员函数在编译期会被编译器重构成非成员函数，即将`this`指针作为函数的第一个参数，这样在函数中通过`this`指针就能找到属于该对象的其他数据成员了。

