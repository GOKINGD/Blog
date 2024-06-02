---
title: 深入理解cpp - const
date: 2024-05-27 23:47:56
tags: [C++,const]
categories: C++
---


# 摘要
本文将介绍C++中的常见关键字const。本文将从变量地址和引用开始进行介绍，逐步介绍以对const有一个深入理解。

# 变量、地址、引用
开始之前，需要了解变量、地址、引用和指针的关系。
![label](/images/cpp_const-image1.png)
```c++
int b = 1;
int& a = b;
int* ptr = b;
```
如上图所示，开辟了一块内存区域，其中的值存储为1，并赋值给变量b，此时通过b即可访问内存区域内存储的值。
ptr是一个指针，存储了变量b所对应的内存区域的地址，通过指针，也能够访问该内存区域。
而变量a定义为一个引用，引用可以理解为别名，即a是b内存区域的另一个名字。
实际上，引用不是一个变量，不占内存空间，所以一旦一个引用绑定了一块内存区域，那么就不可绑定其他内存区域。
```c++
a = 4; // 这种方式是给引用a对应的内存区域赋值，此时b也是4
```

## 指针和引用的区别
1. 引用不可以为空，指针可以为空，所以引用在定义时就必须初始化
2. 引用一旦绑定不可更改，而指针可以随意改变指向
3. 引用本质上不是一个变量，说引用的大小是只引用的那个对象的大小，而指针的大小就是4个字节，因为其存储的是指向对象的地址

# const与const引用
const限定符本质上就是权限管理，从可读可写变成只读的权限。
注意，const仅在文件内有效。
```c++
int a = 1; //读写权限
const int b = 2; //只读权限
```
正常情况下，int a=1 访问的权限就是可读可写的，但是加了const之后，如const int b=2 就理解为， 对b的访问权限是只读的，此时就不能更改b的值。

## const引用
```c++
int a = 1;
int* c = &a;
const int& b = a;
```
值得注意的是，const引用可以理解为仅仅作用于引用变量的本身，而非引用的内存空间。即此时b不能修改值，但是可以通过指针c修改值，或者直接通过a修改值。（此处是底层const）,可以理解为，无法通过b修改其所指向的内容。

## 类型转换
在进行拷贝操作时，两边的对象必须有相同的底层const，而顶层const可以忽略。
例如：
```c++
const int a = 1;
int b = a;
int c = 1;
const int d = c;
```
由于const int是顶层const，可以忽略，所以不论是如何赋值，都是合法的。但是const引用是底层const，赋值时不可忽略。例如：
```c++
int a = 1;
const int& b = a; //合法
int& c = b; //非法，底层const不可忽略
```
由于：
const类型可以绑定非const类型，这是因为可读可写权限转换为了只读权限，权限变小，是安全的。
而非const类型不能绑定const类型，这是因为只读权限扩展成了可读可写权限，权限变大，不安全。
所以，a赋值给b时，合法，但是b是const引用，即底层const，赋值给c时，权限缩小，是非法的。


**类型转换问题：**
```c++
double a = 3.14;
int &b = a; //非法
const int& b = a; //合法
```
这种问题是因为在赋值的过程中会先转换成一个临时量，例如:
```c++
double a = 3.14;
const int tmp = a;
const int& b = tmp;
```
所以引用绑定时，类型不匹配时，需要加上const

# 顶层const与底层const
顶层const指的是变量本身是不可修改的，而底层const指的是指针或者引用所指向的内容是不可修改的。
值得注意的是，const引用都是底层const，引用本身不是一个对象，并且引用本身就不可更改指向，所以没有顶层const一说。
不论是底层const还是顶层const，都应该理解为，无法通过该变量对其所指代的对象进行修改，例如：
```c++
int a=1;
int *c = &a;
const int*b  = &a;
*b = 2; //非法，b所引用的是const类型的，无法通过b修改
*c = 2; //合法
int *const d = &a;
d = c; //非法，顶层const不能修改指向
*d = 2;
```
顶层const和底层const举例如下：
```c++
int a = 1;
int* const p = &a; //顶层const
const int* ptr = &a; //底层const
const int* const pptr = &a; //左边底层，右边顶层
```
c++对修饰符的读法应该要从右往左。即上述例子中，const修饰p，而ptr中，const修的是所指的对象，即ptr指向一个const int* 类型的对象。
# const与函数
## 函数返回const
函数返回const一般分为两种,直接返回const，和返回const的指针或者引用
```c++
const int get(){
    return 1;
}
const A& get(){

}
```
直接返回const意义不大，而返回const的指针或者引用目的就是为了函数调用者不修改类的内容，只读取内容。

## const成员函数
类中的this指针，被隐式声明为顶层const，例如类A中的this指针实际上是A* const this, 这很好理解，因为this总是要指向自己的类，而不能更改。

在类的成员函数中，this指针是隐式传入的。如果希望一个成员函数不修改类中的成员，即可以讲this所指的内容也设置为const，例如，类A中的this应该为const A* const this. 在C++中，在函数的参数列表后添加const，即代表改成员函数为const成员函数，不能修改类中的成员，其本质还是在this指针所指的对象上加了const。例如：
```c++
class A{
    void func(int a) const{
        this->b = 1;//合法，mutable可修改
    }

private:
    mutable int b;
};
```
const成员函数仅一种情况可修改成员变量，即改成员变量为mutable类型。
## 形参const
在函数形参中加入const，为底层const，是为了防止修改传入的参数。
```c++
void func(const int& v) {
	//v += 10;
	std::cout << v << endl;
}
```
而形参使用const时，顶层const会被忽略，例如下面的例子，传入const和非const都可以被func接收，并且无法重载非const参数的相同函数。
```c++
void func(const int a){

}
void func(int a){ //非法

}
```

在进行函数重载的类型匹配时，非精确匹配中，const总是被最先考虑的。
![label](/images/cpp_const-image2.png)

## const修饰类对象与mutable
const对象只能调用常函数。这是因为const对象显然不能修改类的成员，其this指针是const的，而只有常函数的this指针是底层const的，否则会报错。
const对象与常函数中不能修改类中的成员，但是mutable类型的除外。
例如：
```c++
class A{
public:
    mutable int b;
    int c;
}

const A a;
a.b = 2; //合法
a.c = 3; //非法
```
# 总结
本质上，const就是给变量加上权限。const分为顶层const和底层const，简单来看，从右往左读，右边的是顶层const，左边的是底层const。