---
title: 万能引用与完美转发实现剖析
date: 2025-06-26 00:00:00 +0800
categories: [C/C++]
tags: [万能引用与完美转发]
pin: false


toc: true
comments: true

math: false
mermaid: true

typora-root-url: ../../../xdooing.github.io
---







## 左值引用(&)与右值引用(&&)

在c++11中提出了右值引用，作用是为了和左值引用区分开来，其作用是: **右值引用限制了其只能接收右值，可以利用这个特性从而提供重载**，这是右值引用有且唯一的特性，限制了接收参数必为右值, 这点常用在move construct中，告诉别人这是一个即将消失的对象的引用，可以瓜分我的对象东西，除此之外，右值引用就没有别的特性了。

```c++
class Base{
public:
      Base(const Base& b){...} //copy construct 
      Base(Base&& b){...}      //move construct
};
```

需要注意的是，一个右值引用变量在开始使用了之后，他就会变成左值，并且不再携带其是右引用这样的信息，只是一个左值，这就是引用在c++中特殊而且复杂的一点：

**引用在c++中是一个特别的类型，因为它的值类型和变量类型不一样, 左值/右值引用变量的值类型都是左值, 而不是左值引用或者右值引用。**

```c++
int val = 0;
int& val_left_ref = val;      
int&& val_right_ref = 0;

val_left_ref = 0;      // val_left_ref此时是int，而不是int&
val_right_ref = 0;     // val_right_ref此时是int， 而不是int&&
```



## 万能引用(&&)

模板中的`&&`不代表右值引用，而是万能引用，其既能接收左值又能接收右值。

```c++
template<typename T>
void emplace_back(T&& arg){

}

Class Base{
};

int main(){
    Base a;
    emplace_back(a);      // ok
    emplace_back(Base()); // also ok
	return 0;
}
```

这种特性常用在容器元素的增加上，利用传参是左值还是右值进而在生成元素的时候调用copy construct还是move construct，比如说vector的emplace_back。
但需要注意的是，不是所有的模板引用都是万能引用，万能引用只发生在推导时

```c++
template<typename T>
void f(T&& param);               // deduced parameter type ⇒ type deduction;
                                 // && ≡ universal reference
 
template<typename T>
class Widget {
    ...
    Widget(Widget&& rhs);        // fully specified parameter type ⇒ no type deduction;
    ...                          // && ≡ rvalue reference
};
 
template<typename T1>
class Gadget {
    ...
    template<typename T2>
    Gadget(T2&& rhs);            // deduced parameter type ⇒ type deduction;
    ...                          // && ≡ universal reference
};
 
void f(Widget&& param);          // fully specified parameter type ⇒ no type deduction;
                                 // && ≡ rvalue reference
```



## 为什么需要std::forward

模板的万能引用只是提供了能够接收同时接收左值引用和右值引用的能力，但是引用类型的唯一作用就是限制了接收的类型，后续使用中都退化成了左值，我们希望能够在传递过程中保持它的左值或者右值的属性, 如果不使用forward，直接按照下面的方式写就会导致问题。

```c++
void RFn(int&& arg){

}

template<typename T>
void ProxyFn(T&& arg){
      RFn(arg);
}

void main(){
     ProxyFn(1);
}
```

会发现右值版本不能传过去, [int]无法到[int&&]，就导致参数不匹配。

![](/assets/blog_res/assets/941153-20201005120853902-1066461089.png)

为了解决这个问题，引入了std::forward, 将模板函数改成如下形式就可以了, forward被称为完美转发

> 语义：数据是左值就转发成左值，右值就转发成右值，哪怕在万能引用中也是如此。

```c++
template<typename T>
void ProxyFn(T&& arg){
    RFn(std::forward<T>(arg));
}
```



## std::forward 实现原理与细节

左值和右值引用在完成了参数传递之后，再使用时已经完全退化成了左值了，那么forward是如何实现完美转发的呢，举个例子:

```c++
Class Base{
public:
      Base() = default;
      Base(const Base& b){
            // copy construct
      }
      Base(Base&& b){
            //move construct
      }
};

template<typename T>
void ProxyFn(T&& arg){
      Base(std::forward<T>(arg));
}

void main(){
     Base b;
     ProxyFn(b);
}
```

整个推导转发的过程如下图，为了叙述方便，把forward的源码也拷了过来：

![](/assets/blog_res/assets/941153-20201005215906198-2082271165.png)

图中所说的`T会被推导成Base&`，是因为**`在万能引用中，编译器有一个规则，如果传入的是左值，则模板类型会被推导成左值引用类型； 传入的是右值，则模板类型就是值的类型`**。

将T的实际类型代入后，会发现出现了`Base& && arg`这种类型，编译器会将`Base& &&`转成`Base&`，这个过程称之为**`引用折叠`**， 引用折叠的规则如下, 简而言之，`只有左右两个引用都为右值引用时才会折叠成右值引用`。

```c++
Base&& && -> Base&&
Base& &&  -> Base&
Base&& &  -> Base&
Base& &   -> Base& 
```

当然这个arg的类型我们实际上是不关心的，因为它的类型没有什么作用了，在后面的使用下已经退化成左值了。

真正需要关注的是**`std::forward<T>(arg)`**，其是实现完美转发的关键，这个forward中的`T`把`Base&` 给传递了过去, 然后在forward中**`_Ty&&`**进行折叠推导后，就变成了Base&，这就使得`static_cast`以及返回的类型都是**`Base&`**。

这里有两个很重要的点，返回的类型虽然是*`Base&`*，但前文不是说引用只是用来对接收参数的类型起限制作用，后续使用的时候就完全退化成了左值了吗？ forward的完美转发对于返回的是*`Base&`*类型还好，但加入推导返回的是*`Base&&`*，传递到Base去构造的时候，不还是传一个左值吗，匹配到还是copy construct，无法达到完美转发需求吗？

这一块就牵涉到c++标准中的对左值右值的规定，其中规定了**`返回值中, 左值引用的值类型是左值，右值引用的值类型是右值`，`所以无论是forward也好，还是move也好，都是通过函数返回值来实现左值化和右值化的`**，一切都是为了这一步，最后上面的例子返回的是*`Base&`*， 因为它的值是左值，所以会匹配到copy construct。

其次，forward其实有两个版本，但是上面的例子中只给出了一个，因为我们是在万能引用的场景中使用std::forwared<>，因为传递的是左值，所以优先匹配是forward左值引用的版本, 正如其注释所言，`将一个左值转发成一个左值或者右值`，上面分析过这是由模板类型_Ty所决定的最终转发成什么类型；当传递给forward的是一个右值的时候，才会去匹配第二个万能引用版本。

```c++
// FUNCTION TEMPLATE forward
template <class _Ty>
_NODISCARD constexpr _Ty&& forward(
    remove_reference_t<_Ty>& _Arg) noexcept { // forward an lvalue as either an lvalue or an rvalue
    return static_cast<_Ty&&>(_Arg);
}

template <class _Ty>
_NODISCARD constexpr _Ty&& forward(remove_reference_t<_Ty>&& _Arg) noexcept { // forward an rvalue as an rvalue
    static_assert(!is_lvalue_reference_v<_Ty>, "bad forward call");
    return static_cast<_Ty&&>(_Arg);
}
```

下面的例子是forward配合万能引用转发右值的过程:

![](/assets/blog_res/assets/941153-20201005225431096-765191610.png)



## 总结

c++ 搞出了左值引用和右值引用是为了提供能够根据接收左值/右值类型不同而产生重载，整个系统设计的也还算优雅，但是支整套引用系统的隐藏了不少细节，只是理解的时候需要了解这些东西，理解完后，抽象出引用的目的和核心就好了：

1. 左值引用只能接收左值，右值引用只能接收右值

2. 引用变量的`值的类型`是左值，而不是左值引用或右值引用。

   ```c++
   template<typename T>
   void fn(T arg){..}
   
   main(){
      int temp;
      int&& a1 = 0;
      int& a2 = temp;    // 
      fn(a1);            // a1为int, 
      fn(a2);            // a2为int,
   }
   ```

3. 右值引用有且只有一个特性：限制了接收的参数必为右值，可利用其提供函数重载； 这之后，变量的使用上就退换成左值。

4. forward的语义为： 数据是左值就转发成左值，右值就转发成右值，哪怕在万能引用中也是如此; 使用场景为: 配合万能引用实现完美转发。

5. forward和move的原理是: c++编译器规定了函数返回的左值引用是左值，返回的右值引用是右值，通过这个特性配合static_cast的转换，返回了左值/右值。

   ```c++
   int& fn(){...} //返回类型为左值引用，但是返回值为左值
   int&& fn(){...} //返回值为右值引用，但是返回值为右值
   ```

6. 模板的万能引用是通过引用折叠实现，而且，`左值传递到万能引用上，模板类型会先被推导成左值引用以支持引用折叠推导, 右值不做处理，引用折叠推导后刚好是右值引用`。

   ```c++
   template<typename T>
   void fn(T&& arg){..}
   
   main(){
      int a = 0;
      fn(a);             // T为int&
      fn(std::move(a));  // T为int
   }
   ```
