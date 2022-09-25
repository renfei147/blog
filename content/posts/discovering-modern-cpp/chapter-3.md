---
title: "《现代C++探秘》笔记 Chapter 1 - Generic Programming"
subtitle: ""
date: 2022-09-25T16:51:17+08:00
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

## 3.1 Function Templates
### 3.1.2 Parameter Type Deduction
在使用函数模板时，函数参数类型不一定和类型参数类型相同，如：
```cpp
template <typename T>
void func(const T &p);
```
可以把这样的情况抽象为
```cpp
template <typename TPara>
void func(FPara p);
```
在没有用尖括号显式指明类型的情况下，我们知道的是传入的`FPara`的类型，要从中推导出`TPara`的类型。这一过程是如何完成的？

首先是最简单的情况：
```cpp
template <typename TPara>
void func(TPara p) {
    if (std::is_same<decltype(p), int>::value) std::cout << "int\n";
    if (std::is_same<decltype(p), const int>::value) std::cout << "const int\n";
    if (std::is_same<decltype(p), int &>::value) std::cout << "int &\n";
    if (std::is_same<decltype(p), const int &>::value) std::cout << "const int &\n";
    if (std::is_same<decltype(p), int &&>::value) std::cout << "int &&\n";
    if (std::is_same<decltype(p), const int &&>::value) std::cout << "int &&\n";
}
int main() {
    int a = 1;
    const int b = 1;
    func(1);
    func(a);
    func(b);
    return 0;
}
```
输出
```
int
int
int
```
当函数参数为值类型时，`TPara`都会推导为值类型（剥离传入类型的cv修饰），从而`FPara`也为值类型，这保证了函数参数的按值传递的意图。

当函数参数为const引用时
```cpp
template <typename TPara>
void func(const TPara &p) {
    if (std::is_same<decltype(p), int>::value) std::cout << "int\n";
    if (std::is_same<decltype(p), const int>::value) std::cout << "const int\n";
    if (std::is_same<decltype(p), int &>::value) std::cout << "int &\n";
    if (std::is_same<decltype(p), const int &>::value) std::cout << "const int &\n";
    if (std::is_same<decltype(p), int &&>::value) std::cout << "int &&\n";
    if (std::is_same<decltype(p), const int &&>::value) std::cout << "const int &&\n";
}
int main() {
    int a = 1;
    const int b = 1;
    func(1);
    func(a);
    func(b);
    return 0;
}
```
输出
```
const int &
const int &
const int &
```
`TPara`仍为剥离传入类型的引用和cv修饰的值类型，`FPara`变为`const TPara &`，维持了const引用传递的意图。

当函数参数为非const引用时
```cpp
template <typename TPara>
void func(TPara &p) {
    if (std::is_same<decltype(p), int>::value) std::cout << "int\n";
    if (std::is_same<decltype(p), const int>::value) std::cout << "const int\n";
    if (std::is_same<decltype(p), int &>::value) std::cout << "int &\n";
    if (std::is_same<decltype(p), const int &>::value) std::cout << "const int &\n";
    if (std::is_same<decltype(p), int &&>::value) std::cout << "int &&\n";
    if (std::is_same<decltype(p), const int &&>::value) std::cout << "const int &&\n";
}
int main() {
    int a = 1;
    const int b = 1;
    // func(1); doesn't work
    func(a);
    func(b);
    return 0;
}
```
输出
```
int &
const int &
```

这时函数不能接受右值。当传入左值时，cv约束会在`TPara`中保留。

在模板函数参数中使用右值引用可以同时接受左值和右值，这被称为**万能引用**或 **转发引用**。为了理解这一点，我们需要引入**引用折叠**。

通过模板或者typedef/using形成引用的引用时，会根据下面的表格进行引用折叠。
|     |  &  | &&  |
| --- | --- | --- |
| T&  | T&  | T&  |
| T&& | T&  | T&& |

简而言之，只有右值引用的右值引用才为右值引用，其他情况下均为左值引用。

现在我们来看在模板函数参数中使用右值引用的行为。
```cpp
template <typename TPara>
void func(TPara &&p) {
    if (std::is_same<decltype(p), int>::value) std::cout << "int\n";
    if (std::is_same<decltype(p), const int>::value) std::cout << "const int\n";
    if (std::is_same<decltype(p), int &>::value) std::cout << "int &\n";
    if (std::is_same<decltype(p), const int &>::value) std::cout << "const int &\n";
    if (std::is_same<decltype(p), int &&>::value) std::cout << "int &&\n";
    if (std::is_same<decltype(p), const int &&>::value) std::cout << "const int &&\n";
}
int f(int &&a) {
}
int main() {
    int a = 1;
    const int b = 1;
    func(1);
    func(static_cast<const int &&>(1));
    func(a);
    func(b);
    return 0;
}
```
输出
```
int &&
const int &&
int &
const int &
```
结果非常奇妙，如果传入的是右值，那么FPara就为右值引用，如果传入的是左值，那么FPara就为左值引用，且cv修饰得到保留。

再看看上面这些情况下TPara是什么（将上面代码的`decltype(p)`改成`TPara`即可）：
```
int
const int
int &
const int &
```
结合引用折叠就能明白转发引用的原理了。传入右值时，`TPara`推导为值类型，加上`&&`后变成右值引用。传入左值时，编译器考虑什么类型加上`&&`后会变成左值引用，由于引用折叠编译器发现左值引用加上`&&`还是左值引用，因此`TPara`推导为左值引用。

（可以发现上面的论述其实都不大严谨，实际上有一套相当复杂的规则控制着模板实参推导，但是对于一般使用来说这样的认识足够了。）

有了转发引用，我们就能用一个函数模板正确接收左值和右值了，但是当我们想把这一值传递到其他函数（也就是“转发”）时又会遇到问题：
```cpp
void f(int &a) {
    std::cout << "int &\n";
}
void f(const int &a) {
    std::cout << "const int &\n";
}
void f(int &&a) {
    std::cout << "int &&\n";
}
void f(const int &&a) {
    std::cout << "const int &&\n";
}
template <typename TPara>
void func(TPara &&p) {
    f(p);
}
int main() {
    int a = 1;
    const int b = 1;
    func(1);
    func(static_cast<const int &&>(1));
    func(a);
    func(b);
    return 0;
}
```
输出
```
int &
const int &
int &
const int &
```
由于右值引用本身是左值，将右值引用直接传递给函数只会作为左值传入。这就导致上面的代码中无论向`func`中传入什么，转发的都是左值。而如果使用`std::move`，传入的左值也会被转换成右值。这时候就需要“完美转发”——`std::forward`出场了。`std::forward`可以把右值引用转换为右值，而将其他值保持为左值。`std::forward`必须显式指定模板参数。
```cpp
void f(int &a) {
    std::cout << "int &\n";
}
void f(const int &a) {
    std::cout << "const int &\n";
}
void f(int &&a) {
    std::cout << "int &&\n";
}
void f(const int &&a) {
    std::cout << "const int &&\n";
}
template <typename TPara>
void func(TPara &&p) {
    f(std::forward<TPara>(p));
}
int main() {
    int a = 1;
    const int b = 1;
    func(1);
    func(static_cast<const int &&>(1));
    func(a);
    func(b);
    return 0;
}
```
输出
```
int &&
const int &&
int &
const int &
```

### 3.1.4 Mixing Types
在一个函数模板的多个模板参数中传入不同类型的参数会发生什么？
```cpp
std::max(1, 1.0);
```
上面的代码编译失败，这是因为**隐式调用模板函数时模板参数不会进行隐式类型转换**。但如果我们显式调用模板函数，就可以进行隐式类型转换了。下面的代码可以编译。
```cpp
std::max<int>(1, 1.0);
```

### 3.1.6 Automatic return Type
从C++14开始可以用`auto`自动推导函数的返回值类型（根据return的类型），这在函数模板和普通函数中均使用，不过在函数模板中更加实用。