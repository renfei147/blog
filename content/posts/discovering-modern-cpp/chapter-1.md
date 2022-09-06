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
从C++11开始，使用大括号初始化数值类型能够起到防止隐式转换导致精度损失的作用，如：
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
} catch (e1_type &e1) {
} catch (e2_type &e2) {
} catch (...) {
  // 在这里捕获所有异常
}
```

## 1.7 IO
### 1.7.4 Ceneric Stream Concept
对`istream``ostream``iostream`这些基类进行操作可以应用到各种具体的流上。
### 1.7.5 Formatting
C++流里控制输出格式的函数（或者叫IO manipulators）还是很丰富的，但好像没有展开说的必要了。
### 1.7.6 Dealing with I/O Errors
默认情况下C++的IO发生错误后是静默的，确保安全的话需要每次操作后都用good检查，否则在错误状态下会产生未定义行为。可以手动让某个流在出错的时候抛出异常，然而显然也不是个好做法。

~~好做法就是别用标准库的IO了~~