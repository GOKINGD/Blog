---
title: 深入理解cpp-强制转换
date: 2024-07-07 19:54:29
tags: [C++]
categories: C++
---

# 摘要
在C++中，有4种显示的强制类型转换，分别为：static_cast,const_cast,reinterpret_cast和dynamic_cast, 本文将介绍这四种强制类型转换。

# static_cast
static_cast使用于任何具有明确定义的类型转换。但是不包括底层cosnt，底层const由const_cast进行转换。同时也不支持不想关的两个类型进行转换。
例如：
```c++
int i = 1;
double k = 22.3;
double j = static_cast<double>(i) / k;
double u = static_cast<char *>(i); //非法
```

# const_cast
const_cast用于去除const属性，但是只能用于去除底层const，例如：
```c++
int b = 1;
const int *i = &b;
int *j = const_cast<int *>(i);
*j = 2;
std::cout<< *i <<std::endl; // *i = 2;
std::cout<< *j <<std::endl; // *j = 2;
```
有一种特殊情况为：
```c++
const int i = 1;
int *j = const_cast<int *>(&i);
*j = 2;
std::cout<< i <<std::endl; // *i = 1;
std::cout<< *j <<std::endl; // *j = 2;
```
原因在于编译器的优化。由于i是const类型的，所以编译器会将i的值存入寄存器，每次都从寄存器中取值。
而通过j修改了内存中的值，所以输出i的时候，仍然会输出1.
只要在const前加入volatile，则可以告诉编译器从内存中取值。
```c++
volatile const int i = 1;
int *j = const_cast<int *>(&i);
*j = 2;
std::cout<< i <<std::endl; // *i = 2;
std::cout<< *j <<std::endl; // *j = 2;
```
const_cast常常用于有函数重载的上下文，通过const_cast来消除const属性来执行不同的重载过的函数。
## volatile
题外话，简单介绍volatile关键字
正如上面的例子，如果一个变量在程序的运行过程中容易被意想不到的改变，最常见的是多线程场景下，还有如硬件中断等场景下，但是你又希望你的值每次都从内存中读取，而不是从编译器优化的缓存中读取，这个时候就可以使用volatile关键字。

在普通场景中，两次读取变量的值时，没有对变量做修改，或者如上面的例子中，一个变量是const类型时，编译器就会对代码进行优化，将值存入寄存器，来加速访问。 这样就造成了不确定情况的发生，例如上面提到的中断，多线程，而寄存器缓存又来不及更改，就需要volatile来确保每次都从内存中读取。

# reinterpret_cast
reinterpret_cast将变量从位模式上提供较低层面的重新解释，换句话说，static_cast不能转换的不相关了两个类型，reinterpret_cast能转换，只不过可能有些转换没有意义，但是编译器不会提示，例如：
```c++
double d = 3.14;
int i = static_cast<int>(d); // i = 3;
int *p = static_cast<int *>(&d) //非法的
int *p = reinterpret_cast<int *>(&d); //*p = 3
```

reinterpret_cast的另一种用法是，将带参带返回值的函数指针转换成无参无返回值的函数指针，例如：
```c++
typedef void (*func)();
int fun(int i){
    std::cout<<123<<std::endl;
    return 0;
}

func f = reinterpret_cast<func>(fun);
f(); //通过这种方式调用
```
这种调用方式就可以忽视其中的传入参数和返回值，这种情况非常少见，也不建议这样使用。

# dynamic_cast
dynamic_cast用于父类向子类的强制转换。
首先，子类转换为父类，即向上转换，是一件自然的事情，这种转换是安全的，用来实现多态。
而将父类转换为子类，就需要使用dynamic_cast。
dynamic_cast的使用场景有3种，分别是指针，引用，和右值引用。
```c++
dynamic_cast<int *>
dynamic_cast<int &> //必须是左值
dynamic_cast<int &&> //必须是右值
```
在进行父类对子类的转换时，dynamic_cast会对类型进行检查，如果转换前本身就是子类，那么这样转换是安全的，但是如果转换前是父类，那么转换失败，返回一个空指针。
例如：
```c++
class A{

};

class B: public A{

}
void func(A* a){
    B* b = dynamic_cast<B*>(a); //a可能是父类A，也可能是子类B
}

```

其实在实际开发中，各类强制转换的使用场景非常少，至少在业务开发场景中很少，毕竟强制转换尽管你可能非常了解，但是还是容易带来一些意想不到的错误，但是基本的用法还是需要一定的了解。