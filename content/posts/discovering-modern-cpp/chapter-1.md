---
title: "《现代C++探秘》笔记 Chapter 1 - C++ Basics"
subtitle: ""
date: 2022-09-06T22:12:13+08:00
author: "teralem"
authorLink: ""
authorEmail: ""
description: ""
keywords: ""
license: ""
comment: true
weight: 0

tags:
- C++
categories:
- 《现代C++探秘》笔记

hiddenFromHomePage: false
hiddenFromSearch: false

summary: ""
resources:
- name: featured-image
  src: featured-image.jpg
- name: featured-image-preview
  src: featured-image-preview.jpg

toc:
  enable: true
math:
  enable: true
lightgallery: false
seo:
  images: []

# See details front matter: /theme-documentation-content/#front-matter
---

<!--more-->
## 前言
这一系列笔记记录阅读本书过程中遇到的不熟悉的内容。
本书的“现代C++”指C++14，C++17、C++20内容的系统学习要先放到后面了。
## 1.2 Variables
### 1.2.3 Non-narrowing Initialization
从C++11开始，使用大括号初始化（统一初始化）数值类型能够起到防止隐式转换导致精度损失的作用，如：
```cpp
int i1 = 3.14; // 正确
int i2 = {3.14}; // 错误
```
如果值为常量表达式（字面量或constexpr），那么编译器将会检查这一特定值能否用目标类型精确表示（浮点数类型即使是整数也认为不能精确转化为整数类型），否则要求原类型的所有可能取值都能用目标类型精确表示。
```cpp
constexpr int i1 = {3};
int i2 = {3};
unsigned u1 = {3}; // 正确
unsigned u2 = {i1}; // 正确
unsigned u3 = {i2}; // 错误
```
比较有趣的一点是`double`类型的常量表达式是可以用这种方式赋给`float`变量的。大概是因为像`3.14`这种字面量不管用什么二进制浮点类型都不能精确表示，这一转化过程本来就存在误差，就不认为是隐式转换带来的精度损失了。
```cpp
constexpr double d1 = {3.14};
double d2 = {3.14};
float u1 = {3.14}; // 正确
float u2 = {d1}; // 正确
float u3 = {d2}; // 错误
```
需要注意的是实测上面的许多例子在g++中并不会报error，而只会报warning；而clang对标准的实现比较严格，均会报error。

## 1.5 Functions
### 1.5.1 Arguments
非const引用只能引用左值，const引用可以应用右值。这在语法上不是非常自然，但可以给编程带来很大的便利。
### 1.5.4 Overloading
重载决议的细节非常复杂，但总的来说就是寻找最为确切的函数。如果根据规则无法找出可行的函数或者有多个函数都是“最为确切的”，那么失败。

一个类型及其引用类型被在函数签名中被视为不同的类型，下面的三个函数可以同时存在
```cpp
void f(int x) {}
void f(int &x) {}
void f(const int& x) {}
```
然而在这种情况下这些函数几乎无法被调用，因为无论是传入值、传入引用还是传入const引用，都有两个函数可以匹配。但是只要删掉第一个函数，剩下两个函数就完全可以正常使用了。
```cpp
void f(int &x) {}
void f(const int& x) {}
```

*以上内容留待日后完善。*

## 1.6 Error Handling
### 1.6.1 Assertions
多多使用assert断言来测试程序。
```cpp
#define NDEBUG // 定义NDEBUG后可以忽略assert提高性能
#include <cassert>
```
### 1.6.2 Execeptions
`catch`捕获的异常最好使用引用传递，使用...来捕获所有类型的异常。
```cpp
try {
} catch (e1_type& e1) {
} catch (e2_type& e2) {
} catch (...) {
    // 在这里捕获所有异常
}
```

## 1.7 IO
### 1.7.4 Generic Stream Concept
对`istream` `ostream` `iostream`这些基类进行操作可以应用到各种具体的流上。
### 1.7.5 Formatting
C++流里控制输出格式的函数（或者叫IO manipulators）还是很丰富的，但好像没有展开说的必要了。
### 1.7.6 Dealing with I/O Errors
默认情况下C++的IO发生错误后是静默的，确保安全的话需要每次操作后都用good检查，否则在错误状态下会产生未定义行为。可以手动让某个流在出错的时候抛出异常，然而显然也不是个好做法。

~~好做法就是别用标准库的IO了。~~

## 1.8 Arrays, Pointers, and References
### 1.8.3 Smart Pointers
从C++11开始提供了三种智能指针：`unique_ptr`、`shared_ptr`和`weak_ptr` ，均定义在头文件`<memory>`中。

#### `unique_ptr`

`unique_ptr`提供对一个对象的唯一所有权，通过语法设施保证同一时刻只有一个`unique_ptr`持有指针的值，并在析构时`delete`/`delete[]`。

`unique_ptr`必须用一个裸指针（而且需要是`new`/`new[]`出来的）或者`unique_ptr`的右值初始化，此后无法被复制，只能被移动，移动后原来的`unique_ptr`变为0。

可以使用`get`方法获取裸指针。
```cpp
std::unique_ptr<int> a{new int};
std::unique_ptr<int> b = a; // 错误，无法复制
std::unique_ptr<int> c = std::move(a); // 正确
std::cout << a.get() << '\n'; // 输出0
// 离开c的作用域时执行delete
```

为了避免接触裸指针，标准提供并推荐使用`make_unique`函数。
```cpp
auto a = std::make_unique<int>();
```

#### `shared_ptr`
`shared_ptr`实现了引用计数，支持多个指针引用同一个对象，并在最后一个指针析构时执行`delete`/`delete[]`。

`shared_ptr`可以使用裸指针初始化，但同样标准推荐使用`make_shared`函数，此后`shared_ptr`可以自由复制，内部会维护引用计数（引用计数和用户对象放在一起）。

可以使用`get`方法获取裸指针，使用`use_count`方法获取引用计数。
```cpp
auto a = std::make_shared<int>();
auto b = a;
std::cout << a.get() << " " << a.use_count();
```

#### `weak_ptr`
众所周知引用计数不能解决循环引用的问题，这时就需要`weak_ptr`，在此没有详细介绍。

*以上内容留待日后完善。*

### 1.8.7 Containers for Arrays
`valarray`定义在头文件`<valarray>`中，是对数组的封装，特点是提供了各种逐元素操作，包括`sin`等函数。
```cpp
 std::valarray<double> v = {1, 2, 3}, w = {2, 3, 4};
 v = 2.0 * v + w;
 v = sin(v);
```
大小不同的`valarray`之间的运算是未定义行为。

可以使用`resize`方法调整`valarray`的大小，但没有像`vector`一样的扩容机制，每次都会重新分配新的内存。
