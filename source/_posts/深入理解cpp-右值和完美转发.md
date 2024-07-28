---
title: 深入理解cpp-右值和完美转发
date: 2024-07-28 17:21:34
tags: [C++]
categories: C++
---

# 前言
其实上一篇文章"深入理解cpp-拷贝和移动"就已经可以把右值和移动理解的非常透了，但是在度学堂里看了大佬的讲座，感觉讲的非常清晰明了，所以想再补充一点细节。


# std::move
```c++
  /**
   *  @brief  Convert a value to an rvalue.
   *  @param  __t  A thing of arbitrary type.
   *  @return The parameter cast to an rvalue-reference to allow moving it.
  */
  template<typename _Tp>
    _GLIBCXX_NODISCARD
    constexpr typename std::remove_reference<_Tp>::type&&
    move(_Tp&& __t) noexcept
    { return static_cast<typename std::remove_reference<_Tp>::type&&>(__t); }
```
直接看一下move源码，其实move的本质就是stat_cast转换，将传入的__t去掉引用后加上&& 转为右值引用，然后返回。

# RVO
RVO(Return Value Optimization, 返回值优化), 当函数返回值时，用于减少临时变量的创建和拷贝，编译器进行自动优化。
这要求函数在返回本地变量时，且类型和函数声明的类型一致时，编译器会自动优化。
例如下面的例子：
```c++
void caller() {
    Point point;
    point = foo();
}

Point foo() {
    Point p = {1,2};
    return p;
}
```
编译器优化可以理解为:
```c++
void caller() {
    Point point;
    foo(&point);
}

void foo(Point p) {
    p->x = 1;
    p->y = 2;
}

```

# 完美转发
现在假设这样一个场景，我们有一个foo函数，接受一个string参数，我们既想要拷贝，也想要移动，这个例子类似于上一篇文章最后的例子：
```c++
void foo(const string &);
void foo(string &&);
```
但是如果我们有2个参数，那么就需要写4个函数，来完成一个函数的拷贝与转发，函数的数量成了指数增长，如：
```c++
void foo(const string &, const string &);
void foo(const string &, string &&);
void foo(string &&, const string &);
void foo(string &&, string &&);
```

此时，可以使用完美转发来解决这个问题，完美转发的模板严格要求如下：
```c++
template <typename T>
void foo(T && value){
    bar(std::forward<T>value);
}

为了实现完美转发，c++实际提出了一系列的规则，其具体原理依靠的是引用折叠和编译器推理。
例如：
```c++
std::string str;
foo(str);

template <typename T>
void foo(T&& value){
    bar(std::forward<T>(value));
}
```
T&& 被称为转发引用(Forward Reference)
此时，编译器根据传入的参数str，推理出T的类型应该为string &, 那么value的类型就是string & &&类型。
而c++提出了引用折叠：
& & --> &
&& & --> &
& && --> &
&& && --> &&
所以此时value的类型是string &, 是一个左值，这样就实现了完成转发。
另一个例子：
```c++
std::string str;
foo(std::move(str));

template <typename T>
void foo(T&& value){
    bar(std::forward<T>(value));
}
```
这个例子中，T会被编译器推理为string,此时value类型就是string &&
所以bar的参数就是一个右值，实现了完成转发。

# 完美转发源码
先看一下完美转发的源码：
```c++
template<typename _Tp>
  constexpr _Tp&&
  forward(typename std::remove_reference<_Tp>::type&& __t) noexcept
  {
    static_assert(!std::is_lvalue_reference<_Tp>::value, "template argument"
      " substituting _Tp is an lvalue reference type");
    return static_cast<_Tp&&>(__t);
  }
```
实际上，上面说了那么多，看了源码就一目了然。不论是_Tp被推导为什么类型，forward的参数都会去除引用，于是就变成了T&& __t,一定是右值。
然后函数中首先需要判断不为左值。然后再通过static_cast转换。如果只有当_Tp推理为T&&时，才会转换为右值。

# 完美转发的问题
完美转发要求必须严格符合有推理+格式对这两条规则，例如：
如果没有模板，就肯定不是完美转发。
再如：
```c++
template <typename T>
class vector {
public:
    void push_back(T&& x);
}
```
上面这个例子中，push_back仍然不是完美转发，这是因为模板是类的，而不是这个函数的，即定义该类时，参数的类型就已经确定了，所以不存在推理，x自然是确定的类型，没有转发一说。
还有下面的两个例子，都不是完美转发:
```c++
template <typename T>
void foo(vector<T> &&v);

template <typename T>
void foo(const T&& v);
```

一个比较常见的问题是: 对完美转发的重载容易忽略其他的重载函数。
```c++
class widget{
public:
    widget(){

    }
    template <typename T>
    explicit widget (T&& value){ //--1

    }
    widget(const widget& other){ //--2
 
    }
};
widget w1;
const widget w2;
widget w3(w1); // --1
widget w4(w2); // --2
```
上面的例子，例如我们系统使用string直接初始化widget类型，这个时候应该执行函数1
而如果我们希望使用widget类来拷贝初始化，应该执行函数2。
可事实却是，w1会走函数1的完美转发，而不是函数2。因为函数1匹配更精确。
而使用w2的时候因为是const类型，所以会优先执行函数2。

为了解决这类问题，可以使用下面的方式:
```c++
template <typename T,
          typename enable_if<is_same<T, string>::value>::type* = nullptr>
explicit widget(T &&value){

}
```