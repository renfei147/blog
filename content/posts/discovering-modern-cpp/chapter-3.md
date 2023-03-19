---
title: "《现代C++探秘》笔记 Chapter 3 - Generic Programming"
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
`TPara`为剥离传入类型的引用和volatile修饰的值类型，`FPara`变为`const TPara &`，维持了const引用传递的意图。

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

在模板函数参数中使用右值引用可以同时接受左值和右值，这被称为**万能引用**或**转发引用**。为了理解这一点，我们需要引入**引用折叠**。

通过模板或者typedef/using形成引用的引用时，会根据下面的表格进行引用折叠。
|     | &   | &&  |
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
从C++14开始可以用`auto`自动推导函数的返回值类型（根据return的类型），这在函数模板和普通函数中均适用，不过在函数模板中更加实用。

## 3.2 Namespaces and Function Lookup

### 3.2.1 Namespaces
内层命名空间的名称可以隐藏外层命名空间的名称。需要外层命名空间的名称时可以加上外层命名空间的名称来指代。在开头加上`::`表示从全局开始查找。
```cpp
void func() {
    std::cout << "::func\n";
}
namespace c1 {
    void func() {
        std::cout << "::c1::func\n";
    }
    namespace c2 {
        namespace c1 {}
        void func() {
            std::cout << "::c1::c2::func\n";
        }
        void test() {
            func();
            c1::func(); // error
            ::c1::func();
            ::func();
        }
    }
}
int main() {
    c1::c2::test();
    return 0;
}
```

可以使用`using`在函数或命名空间（但不能在类中）中导入某个命名空间中的某个名称或所有名称。
```cpp
using std::cin, std::cout;
using namespace std;
```

可以使用命名空间别名来更加方便地引用其他命名空间，尤其是嵌套命名空间。
```cpp
namespace a_long_namespace_name {
    namespace another_long_namespace_name {
        void func(){
            std::cout <<"hello, world\n";
        }
    }
}
namespace a = a_long_namespace_name::another_long_namespace_name;
```

### 3.2.2 Argument-Dependent Lookup
Argument-Dependent Lookup (ADL) 是C++在查找函数时的一个特殊特性：查找函数时会自动包括函数参数类型所在的命名空间，但不包括所在命名空间的上层命名空间，也不包括用`using`导入的名称。
```cpp
namespace c1 {
    namespace c2{
        class ClassA{
          public:
            int data;
        };
        void func1(ClassA a){
            std::cout << "func1: "<<a.data << '\n';
        }
        void func2(){
            std::cout << "func2: "<< '\n';
        }
    }
    void func3(c2::ClassA a){
        std::cout << "func3: "<<a.data << '\n';
    }
}
int main() {
    c1::c2::ClassA a{5};
    func1(a);
    func2(); // error
    func3(a); // error
    return 0;
}
```
这一特性的实用性主要体现在运算符重载（否则就要`using`或者用`ns::operator+`这样的方法访问了）。

除此之外还可以利用这一特性控制函数模板的匹配（这也是这一内容出现在第三章的原因）。
```cpp
namespace mat {
    struct sparse_matrix {};
    struct dense_matrix {};
    struct uber_matrix {};

    template <typename Matrix>
    double one_norm(const Matrix &A) { ... }
}
namespace vec {
    struct sparse_vector {};
    struct dense_vector {};
    struct uber_vector {};

    template <typename Vector>
    double one_norm(const Vector &A) { ... }
}
```

### 3.2.3 Namespace Qualification or ADL
假设我们想在一个函数模板中调用`std::swap`，但某一个类型提供了自己的`swap`函数，这时可以用下面的方法：
```cpp
class ClassA {
  public:
    ClassA(int data) : data(data) {}
    friend void swap(ClassA &a, ClassA &b) {
        std::cout << "called custom swap\n";
        std::swap(a.data, b.data);
    }
  private:
    int data;
};
template <typename T>
void my_swap(T &a, T &b) {
    using std::swap;
    swap(a, b);
}
int main() {
    ClassA a(1), b(2);
    swap(a, b);
    return 0;
}
```
使用了`using std::swap;`后，`std::swap`和类型提供的`swap`（如果在某个命名空间里那么ADL会起作用）会同时成为候选对象，由于类型提供的`swap`参数类型更加精确，最后调用的是类型提供的`swap`。如果类型没有`swap`，那么就会调用`std::swap`。

## 3.3 Class Templates
在类模板中常用空结构体来做标记，如下面的`vector`模板标记了向量是行向量还是列向量。
```cpp
struct row_major {};
struct col_major {};
template <typename T = double, typename Orientation = col_major>
class vector {
    ...
};
```
上面的代码同样展示了模板参数可以设置默认值，但即使所有模板均为默认值，也不能省略尖括号，即不能使用`vector`而应该使用`vector<>`。

不同于函数的默认参数，类的默认参数可以使用前面的参数，如：
```cpp
template <typename T, typename U = T>
struct pair {
    T first;
    U second;
}
```
上面的`pair`在没有指定`second`的类型时`second`的类型和`first`相同。


在函数模板中可以用传入数组的长度来推导非类型模板参数。需要注意，由于存在数组到指针的隐式转换，直接传入数组的话会被转换为指针而丢失长度信息，应该传入数组的引用。（也可以传入数组的指针，但显然比较麻烦。）
```cpp
template <typename T, size_t N>
T sum(const T (&array)[N]) {
    T sum(0);
    for (size_t i = 0; i < N; ++i) {
        sum += array[i];
    }
    return sum;
}
```

# 3.4 Type Deduction and Definition

## 3.4.1 Automatic Variable Type

用`auto`在一个语句内声明多个变量时所有变量必须为同一类型。

使用cv或引用限定`auto`时的行为和函数模板参数的推导如出一辙（也就是根据`FPara`推导`TPara`）。

```cpp
int main() {
    const int &t = 1;
    auto a = t; // 值类型，不保留cv
    auto &b = t; // 只接受左值
    const auto &c = 1; // 强制const
    auto &&d = 1; // 万能引用
    const auto &&e = 1;// 只接受右值，强制const
}

```

## 3.4.2 Type of an Expression

`decltype`可用于获取一个表达式的类型。

```cpp
int main() {
    decltype('a' + 'b') a;
}
```
如上面的代码中a的类型为`int`，因为两个根据整型提升两个`char`相加为`int`类型。

在C++11中还没有引入返回类型推导，必须使用使用`decltype`进行手动推导，例如下面的（错误）代码。
```cpp
template <typename T, typename U>
decltype(a + b) add(const T &a, const U &b) {
    return a + b;
}
```

问题在于返回值写在参数前面，因此无法用参数运算得到的类型来推导返回值类型。C++11引入了**后置返回值类型（trailing return type）**来解决这一问题。
```cpp
template <typename T, typename U>
auto add(const T &a, const U &b) -> decltype(a + b) {
    return a + b;
}
```

不过在C++14引入返回类型推导后直接写下面的代码即可。
```cpp
template <typename T, typename U>
auto add(const T &a, const U &b) {
    return a + b;
}
```

和`auto`不同，`decltype`不使用类似函数模板参数的推导，而是严格反映表达式的原类型。
```cpp
int main() {
    const int &t = 1;
    auto a = t;        // int
    decltype(t) b = t; // const int &

    int p = 1;
    auto &&c = p;        // 正确，万能引用
    decltype(p) &&d = p; // 错误，无法将右值引用绑定到左值
}
```

## 3.4.3 `decltype(auto)`
上面的写法形如`decltype(expr) a = expr;`，需要将`expr`写两遍。C++14引入了`decltype(auto)`来简化这一写法，可以写成`decltype(auto) a = expr;`。`decltype(auto)`也可用于函数返回值。


下面是一个需要利用`decltype(auto)`保留cv限定和引用的例子。
```cpp
// 一个对列表的引用，用该引用访问时自动将下标偏移offset
template <typename T>
class shifted_list_ref {
  public:
    shifted_list_ref(T &list, int offset)
        : list(list), offset(offset) {}

    decltype(auto) operator[](int i) {
        return list[i + offset];
    }

  private:
    T &list;
    int offset;
};

template <typename T>
auto make_shifted_list_ref(T &list, int offset) {
    return shifted_list_ref<T>(list, offset);
}

int main() {
    int list[5] = {0, 1, 2, 3, 4};
    auto ref = make_shifted_list_ref(list, 2);
    ref[1] = 66; // ok
    std::cout << ref[1] << '\n';

    const int clist[5] = {0, 1, 2, 3, 4};
    auto cref = make_shifted_list_ref(clist, 2);
    cref[1] = 66; // error
    std::cout << cref[1] << '\n';
}
```

## 3.4.4 Defining Types
除了兼容性，否则在C++11后没有理由使用`typedef`，而应该使用`using`。`using`的一大优点是可以实现模板别名。
```cpp
template <std::size_t N>
using int_array = std::array<int, N>
```

# 3.6 Template Specialization

## 3.6.1 Specializing a Class for One Type
模板的特化必须在原模版的后面出现。即使所有的模板参数均被特化，也要在前面加上一个`template<>`。
```cpp
template <typename T, typename U>
class sometype {
};

template <typename T>
class sometype<T, int> {
};

template <>
class sometype<int, int> {
};
```

## 3.6.2 Specializing and Overloading Functions
函数模板不参与重载决议，在和重载函数混用的时候很容易出现意外情况，参见[此文](https://zhuanlan.zhihu.com/p/509671916)。因此，最佳做法是**不要使用函数模板特化**，仅使用重载函数来代替。除了减少困惑，另一个好处在于函数模板只能全特化而不能偏特化，但用重载函数却可以方便地模拟出类似偏特化的效果。

```cpp
template <typename Base, typename Exponent>
Base power(const Base &x, const Exponent &y);

template <typename Base>
Base power(const Base &x, int y);

template <typename Exponent>
double power(double x, const Exponent &y);
```

调用上面这些函数时遵循复杂的重载决议规则。如果调用`power(3.0, 2)`，那么后两个重载根据规则具有相同的确切程度，因此调用失败。

## 3.6.3 Partial Specialization
最为常见的类模板的偏特化就是直接指定某个模板参数，但也有较为复杂的形式。

一个例子是我们想要给自己编写的`vector`模板中所有模板参数为`std::complex<T>`的类型进行特化，可以这样写：
```cpp
template <typename T>
class point {
    ...
};

template <typename T>
class point<std::complex<T>> {
    ...
};
```

还有一些更为复杂的例子，从cppreference中摘录一些：
```cpp
template<class T1, class T2, int I>
class A {};             // 主模板
 
template<class T, int I>
class A<T, T*, I> {};   // #1：部分特化，其中 T2 是指向 T1 的指针
 
template<class T, class T2, int I>
class A<T*, T2, I> {};  // #2：部分特化，其中 T1 是指针
 
template<class T>
class A<int, T*, 5> {}; // #3：部分特化，其中 T1 是 int，I 是 5，T2 是指针
 
template<class X, class T, int I>
class A<X, T*, I> {};   // #4：部分特化，其中 T2 是指针
```

类似重载决议，编译器会寻找最为确切的特化。

## 3.6.4 Partially Specializing Functions
如上文所述，语法上函数模板不能偏特化，但用重载函数可以实现类似的效果。
```cpp
template <typename T>
inline T abs(const T &x) {
    return x < T(0) : -x : x;
}

template <typename T>
inline T abs(const std::complex<T> &x) {
    return sqrt(real(x) * real(x) + imag(x) * imag(x));
}
```

由于函数调用的复杂规则，上面的做法很容易出问题。为了让函数的特化调用更加稳定、清晰，我们可以把不同版本的函数包装在类模板中，并利用类模板的特化，也就是使用**仿函数（functor）**
```cpp
template <typename T>
struct abs_functor {
    T operator()(const T &x) {
        return x < T(0) : -x : x;
    }
};

template <typename T>
struct abs_functor<std::complex<T>> {
    T operator()(const T &x) {
        return sqrt(real(x) * real(x) + imag(x) * imag(x));
    }
};

template <typename T>
decltype(auto) abs(const T &x) {
    return abs_functor<T>()(x);
}
```

# 3.7 Non-Type Parameters for Templates
模板的非类型参数只能是整型而不是浮点型。（事实上从C++20起支持浮点型了！）（事实上还可以用左值引用、指针等类型，详见cppreference。）

# 3.8 Functors
相比于函数指针，仿函数提供了大得多的灵活性，还可以存储内部状态以及进行编译器内联优化。不过一个函数要想接受各种仿函数（以及函数指针），就必须写成模板的形式。本节给出了一个利用仿函数实现求近似n阶导数的例子。
```cpp
template <typename F, typename T, unsigned N>
class nth_derivative {
    using prev_derivative = nth_derivative<F, T, N - 1>;

  public:
    nth_derivative(const F &f, const T &h)
        : fp(f, h), h(h) {}
    T operator()(const T &x) const {
        return (fp(x + h) - fp(x)) / h;
    }

  private:
    prev_derivative fp;
    T h;
};

template <typename F, typename T>
class nth_derivative<F, T, 1> {
  public:
    nth_derivative(const F &f, const T &h)
        : f(f), h(h) {}
    T operator()(const T &x) const {
        return (f(x + h) - f(x)) / h;
    }

  private:
    const F &f;
    T h;
};

template <unsigned N, typename F, typename T>
nth_derivative<F, T, N> make_nth_derivative(const F &f, const T &h) {
    return nth_derivative<F, T, N>(f, h);
}

double func(double x) {
    return sin(x) + cos(x);
}

int main() {
    auto d3 = make_nth_derivative<3>(func, 0.001);
    std::cout << d3(1);
}
```

将仿函数模板从类模板改写成函数模板可以利用函数模板的自动推导避免手写类型。
```cpp
template <typename T>
struct add1 {
    T operator()(const T &x, const T &y) const {
        return x + y;
    }
};

struct add2 {
    template <typename T>
    T operator()(const T &x, const T &y) const {
        return x + y;
    }
};

int main() {
    std::cout << add1<int>{}(1, 2) << "\n";
    std::cout << add2{}(1, 2) << "\n";
}
```

# 3.9 Lambda

通常编译器可以自动推导出lambda的返回值类型，但在特殊情况下（有多个`return`语句）需要用后置返回值类型手动指定。

```cpp
auto f = [](double x) -> double { return x * x; };
```

## 3.9.2 Capture by Value
在捕获列表中加入局部变量名来按值捕获局部变量。
```cpp
int main() {
    double k = 1.2, b = 3.5;
    auto f = [k, b](double x) { return k * x + b; };
}
```

这相当于
```cpp
struct lambda_f {
    lambda_f(double k, double b)
        : k(k), b(b) {}
    double operator()(double x) const {
        return k * x + b;
    }
    const double k, b;
};
int main() {
    double k = 1.2, b = 3.5;
    auto f = lambda_f(k, b);
}
```

注意捕获的变量在lambda内部是const的，如果要去除const限定，那么要在小括号后加上`mutable`。

```cpp
int main() {
    double k = 1.2, b = 3.5;
    auto f = [k, b](double x) mutable { k += 0.1;return k * x + b; };
}
```

## 3.9.3 Capture by Reference
在捕获列表中的局部变量名前加上`&`来按引用捕获局部变量。
```cpp
int main() {
    double k = 1.2, b = 3.5;
    auto f = [&k, &b](double x) { return k * x + b; };
}
```
现在在`f`创建之后对`k`和`b`的修改也会影响`f`。上面的代码相当于

```cpp
struct lambda_f {
    lambda_f(double &k, double &b)
        : k(k), b(b) {}
    double operator()(double x) const {
        return k * x + b;
    }
    double &k, &b;
};
int main() {
    double k = 1.2, b = 3.5;
    auto f = lambda_f(k, b);
}
```

注意不同于按值捕获，按引用捕获的变量在lambda内始终是可修改的（除非被捕获的变量本身就是`const`修饰的）。

捕获列表有一些简写：

* `[=]`：全部按值捕获
* `[&]`：全部按引用捕获
* `[=, &a, &b, &b]`：除了a，b，c按引用捕获，其他全部按值捕获
* `[&, a, b, b]`：除了a，b，c按值捕获，其他全部按引用捕获

## 3.9.4 Generalized Capture
C++14引入了**广义捕获（Generalized Capture）**，可以在捕获列表中用`var = expr`的格式定义新变量，注意`var`处于lambda的内部作用域，而`expr`处于外部作用域，因此下面的代码是正确的。
```cpp
int main() {
    int x = 4;
    auto y = [&r = x, x = x + 1]() {
        r += 2;
        return x + 2;
    }(); // y = 7
}
```

广义捕获可用于将变量所有权移动到lambda内部。

```cpp
int main() {
    auto p = std::make_unique<int>(2);
    auto f = [p = std::move(p)]() { return *p; };
}
```

## 3.9.5 Generic Lambdas
C++14开始可以简单地用`auto`作为lambda的参数类型来编写泛型形式的lambda表达式。
```cpp
int main() {
    auto f = [](auto x, auto y) { return x + y; };
    std::cout << f(1, 2) << "\n";
    std::cout << f(1.2, 2.3) << "\n";
}
```

# 3.10 Variadic Templates
C++11引入了**可变参数模板（Variadic Templates）**。可以使用`...`操作**参数包**，将`...`放在左侧表示打包，放在右侧表示解包。具体来说：
* `typename ...P` 将多个类型模板参数打包到类型包`P`中
* `<P...>` 在特化类或函数模板时解包类型包`P`
* `P ...p` 将多个函数参数打包到变量包p中
* `sum(p...)` 将变量包`p`解包并用这些参数调用函数
* `f(p)` 将变量包`p`中的每个变量分别传入函数`f`并得到新的变量包，若`f`为函数模板还可以写`f<P>(p)`
* `sizeof...(P)` 求参数包`P`中的参数个数

一个利用可变参数模板求和的例子：
```cpp
template <typename T>
T sum(T t) {
    return t;
}

template <typename T, typename... P>
T sum(T t, P... p) {
    return t + sum(p...);
}

int main() {
    std::cout << sum(1, 2, 3, 4) << "\n";
}
```