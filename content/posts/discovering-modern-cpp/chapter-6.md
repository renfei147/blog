---
title: "《现代C++探秘》笔记 Chapter 6 - Object-Oriented Programming"
subtitle: ""
date: 2023-03-19T15:40:33+08:00
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
# 6.1 Basic Principles

## 6.1.1 Base and Derived Classes
在C++中继承分为`public`继承、`protected`继承和`private`继承。
* `public`继承保持基类所有成员的可访问性不变
* `protected`继承将基类的`public`成员改为`protected`，其他成员的可访问性不变
* `private`继承将基类的所有成员的可访问性改为`private`

`class`默认使用`private`继承，而`struct`默认使用`public`继承。

子类中的成员会隐藏基类的同名成员，但可以通过添加名称限定（如`base::a`的方式访问）。
```cpp
struct base {
    int a;
};

struct derived : base {
    int a;
    int get1() {
        return a;
    }
    int get2() {
        return base::a;
    }
};

int main() {
    derived t;
    t.a = 1;
    std::cout << t.get1() << "\n";
    std::cout << t.get2() << "\n";
}
```

对于函数而言，隐藏是基于名称而非签名的，因此子类会隐藏基类中同名函数的全部重载。可以使用`using`使基类中被隐藏的重载函数在子类中可见。
```cpp
class base {
  public:
    void func(int x) {
        std::cout << "base func(int) " << x << "\n";
    }
    void func(std::string x) {
        std::cout << "base func(std::string) " << x << "\n";
    }
};

class derived1 : public base {
  public:
    void func(std::string x) {
        std::cout << "derived1 func(std::string) " << x << "\n";
    }
};

class derived2 : public base {
  public:
    using base::func;
    void func(std::string x) {
        std::cout << "derived2 func(std::string) " << x << "\n";
    }
};

int main() {
    derived1 d1;
    derived2 d2;
    d1.func("abc");
    d1.func(1); // error
    d2.func(1); // ok
}
```

所有需要某个基类的场合都可以用子类代替。不过在上面这样没有使用虚函数的情况下，调用子类还是基类的函数取决于我们是以基类还是子类的类型调用。
```cpp
class base {
  public:
    void func(int x) {
        std::cout << "base func(int) " << x << "\n";
    }
    void func(std::string x) {
        std::cout << "base func(std::string) " << x << "\n";
    }
};

class derived : public base {
  public:
    void func(std::string x) {
        std::cout << "derived func(std::string) " << x << "\n";
    }
};

int main() {
    derived d;
    base b = d;
    d.func("abc"); // output: derived func(std::string) abc
    b.func("abc"); // output: base func(std::string) abc
}
```

如果基类有默认构造函数，那么子类的构造函数会自动加入对基类的默认构造函数的调用。但是如果基类没有默认构造函数或需要调用其他构造函数，那么就需要在子类的构造函数的成员初始化列表中写明。显然，子类必须要调用基类的某个构造函数
```cpp
class person {
  public:
    person(const std::string &name)
        : name(name) {
    }
    void all_info() const {
        std::cout << "name: " << name << "\n";
    }

  private:
    std::string name;
};

class student : public person {
  public:
    student(const std::string &name, const std::string &id)
        : person(name), id(id) {
    }
    void all_info() const {
        person::all_info();
        std::cout << "id: " << id << "\n";
    }

  private:
    std::string id;
};

int main() {
    student s("John", "666");
    s.all_info();
}
```

## 6.1.2 Inheriting Constructors
子类的默认构造函数（如果有的话）会自动调用基类的构造函数。然而其他的构造函数不会自动从基类继承。但是，可以使用`using`使基类的构造函数在子类中可见。
```cpp
class base {
  public:
    explicit base(int data)
        : data(data) {
        std::cout << "base(int)\n";
    }

  private:
    int data;
};

class derived : public base {
  public:
    using base::base;
};

int main() {
    derived d(1);
}
```

此外，可以在子类的构造函数的初始化列表中

## 6.1.3 Virtual Functions and Polymorphic Classes

虚函数开启了面向对象最重要的特性：**覆盖（override）**，也就是运行时多态。与此前不使用虚函数时的**隐藏**基类函数不同，**覆盖**会在使用基类的引用或指针调用的时候动态地调用其实际类型对应的函数。具有虚函数的类被称为**多态类型**。

```cpp
class base {
  public:
    virtual void func() {
        std::cout << "base\n";
    }
};

class derived : public base {
  public:
    // 这里的virtual不是必须的，即使不写也是虚函数
    virtual void func() {
        std::cout << "derived\n";
    }
};

int main() {
    derived d;
    base b1 = d;
    base &b2 = d;
    base *b3 = &d;
    b1.func();
    b2.func();
    b3->func();
}
```

输出
```
base
derived
derived
```

可以看到使用基类的引用或指针调用时都调用了子类的函数（上面的`d2`和`d3`）。然而如果将子类对象值传递（赋值、初始化、按值传参）给基类对象（上面的`d1`），会发生**对象切片（Object Slicing）**，即只保留基类部分，所有子类中新增的数据和函数都会被“切除”，这样虚函数也就无法发挥作用了。对象切片是值语义的面向对象语言中的特有问题。

虚函数覆盖要求基类和子类的函数签名完全相同，稍有不注意就有可能产生意外。例如下面的代码仅仅向基类的函数中加了一个`const`，就导致基类和子类的函数签名不同从而没有实现覆盖，导致最后调用的都是基类的函数。

```cpp
class base {
  public:
    virtual void func() const {
        std::cout << "base\n";
    }
};

class derived : public base {
  public:
    virtual void func() {
        std::cout << "derived\n";
    }
};

int main() {
    derived d;
    base b1 = d;
    base &b2 = d;
    base *b3 = &d;
    b1.func();
    b2.func();
    b3->func();
}
```

C++11加入了关键字`override`来解决这一问题。用`override`修饰的函数必须覆盖基类的虚函数。将子类的代码如下改写，现在编译器会在没有正确覆盖的情况下报错。
```cpp
class derived : public base {
  public:
    virtual void func() override {
        std::cout << "derived\n";
    }
};
```

C++11还加入了关键字`final`，表示一个虚函数或一整个类不能被覆盖。

带有纯说明符`= 0`的虚函数称为**纯虚函数**，带有纯虚函数的类称为**抽象类**，不能创建抽象类的对象。
```cpp
class base {
  public:
    virtual void func() = 0;
};
```

## 6.1.4 Functors via Inheritance
此前我们使用模板（编译时多态）的方式来让一个函数接受多种仿函数。现在我们也可以用继承（运行时多态）的方式。也就是接收一个仿函数的基类（往往是抽象类），然后将各种具体的仿函数实现为基类的子类。然而使用模板不仅运行效率更高，而且允许传入传统函数指针，因此要常用得多。

`std::function`的实现结合了编译时多态和运行时多态，实现了最大的灵活性，代价是需要一定的运行时开销。

# 6.3 Multiple Inheritance

# 6.3.1 Multiple Parents
进行多继承时，如果多个基类中有同名成员，而且子类中没有用同名成员隐藏或覆盖，那么尝试访问时会产生歧义。
```cpp
class base1 {
  public:
    void func() {
        std::cout << "base1\n";
    }
};

class base2 {
  public:
    void func() {
        std::cout << "base2\n";
    }
};

class derived1 : public base1, public base2 {
};

class derived2 : public base1, public base2 {
  public:
    void func() {
        std::cout << "derived2\n";
    }
};

int main() {
    derived1{}.func(); // error
    derived2{}.func(); // output: derived2
}
```

当然，如果在类的内部使用，那么可以在成员名称前面加上基类限定符来进行区分。那么能否通过给不同的基类设置不同的继承访问权限（如对一个基类使用`public`继承，对另一个基类使用`private`继承）来使得在类外访问时也能区分呢？事实证明这是不可行的。**`public`、`protected`和`private`修改的是可访问性而不是可见性**。

## 6.3.2 Common Grandparents
复杂的情况出现在多个基类有共同祖先的情况下。
```cpp
class person {
  public:
    person(const std::string &name)
        : name(name) {
        std::cout << "construct person\n";
    }
    void all_info() const {
        std::cout << "name:" << name << "\n";
    }

  private:
    std::string name;
};

class student : public person {
  public:
    student(const std::string &name, const std::string &id)
        : person(name), id(id) {
        std::cout << "construct student\n";
    }
    void all_info() const {
        person::all_info();
        std::cout << "id:" << id << "\n";
    }

  private:
    std::string id;
};

class mathematician : public person {
  public:
    mathematician(const std::string &name, const std::string &proved)
        : person(name), proved(proved) {
        std::cout << "construct mathematician\n";
    }
    void all_info() const {
        person::all_info();
        std::cout << "proved:" << proved << "\n";
    }

  private:
    std::string proved;
};

class math_student : public student, public mathematician {
  public:
    math_student(const std::string &name, const std::string &id, const std::string &proved)
        : student(name, id), mathematician(name, proved) {
        std::cout << "construct math_student\n";
    }
    void all_info() const {
        student::all_info();
        mathematician::all_info();
    }
};

int main() {
    math_student a("John", "666", "Fermat's Last Theorem");
    a.all_info();
}
```
输出
```
construct person
construct student
construct person
construct mathematician
construct math_student
name:John
id:666
name:John
proved:Fermat's Last Theorem
```
construct person
可以看到`person`被初始化了两次。事实上`math_student`中包含`person`的两个副本，`math_student`的构造函数调用`student`和`mathematician`的构造函数，它们再分别调用各自的`person`副本的构造函数。尽管在这个例子中`person`的两个副本具有相同的数据，但存储不同的数据也是完全可行的。如果想访问共同祖先的成员，那么会产生歧义。

此外，如果试图把子类转换成共同祖先，编译器也会因为不能确定选取祖先的哪个副本而给出歧义。可以使用下面的写法来明确要`student`中的那个`person`。
```cpp
person p = (student)a;
person &p = (student &)a;
```

我们更想要的可能是在`math_student`中只包含一份`person`，为此C++提供了虚继承。被虚继承的类只会在“最派生的类”中保留一份，且其初始化只由“最派生的类”负责，也就是继承层次中中间的类对被虚继承的类的构造函数的调用会被忽略。
我们只需要如下修改代码：
```cpp
class student : public virtual person {
    ...
};

class mathematician : public virtual person {
    ...
};
class math_student : public student, public mathematician {
  public:
    math_student(const std::string &name, const std::string &id, const std::string &proved)
        : person(name), student(name, id), mathematician(name, proved) {
        std::cout << "construct math_student\n";
    }
    void all_info() const {
        student::all_info();
        mathematician::all_info();
    }
};
```
输出
```
construct person
construct student
construct mathematician
construct math_student
name:John
id:666
name:John
proved:Fermat's Last Theorem
```
可以看到`person`只被初始化了一次。此外，由于现在`person`只有一份，就可以无歧义地直接转换成共同祖先了。

对于更为复杂的虚继承和非虚继承的混合情况，摘录cppreference中的例子。
```cpp
struct B { int n; };
class X : public virtual B {};
class Y : virtual public B {};
class Z : public B {};
 
// 每个 AA 类型对象拥有一个 X，一个 Y，一个 Z 和两个 B：
// 一个是 Z 的基类，另一个由 X 与 Y 所共享
struct AA : X, Y, Z
{
    void f()
    {
        X::n = 1; // 修改虚 B 子对象的成员
        Y::n = 2; // 修改同一虚 B 子对象的成员
        Z::n = 3; // 修改非虚 B 子对象的成员
 
        std::cout << X::n << Y::n << Z::n << '\n'; // 打印 223
    }
};
```

# 6.4 Dynamic Selection by Sub-typing
C++禁止虚函数模板，但可以在类模板中使用虚函数。

# 6.5 Conversion
除了C风格类型转换，C++提供了以下四种类型转换操作：
* `static_cast`
* `dynamic_cast`
* `const_cast`
* `reinterpret_cast`

C风格类型转换过于自由，可能一次转换类型的多个方面。而上面的类型转换操作一次只转换一个方面，具有更好的语义。因此推荐使用上面的类型转换操作代替C风格类型转换。

## 6.5.1 Casting between Base and Derived Classes
在继承层次中的向上转换可以隐式进行，关于对象切片和虚继承时可能出现的歧义已在前文讨论过。

向下转换显然只能发生在指针和引用上，并且只能向下转换到其实际的类型。向下转换有`static_cast`和`dynamic_cast`两种形式。

`static_cast`只进行编译期检查，我们需要自行保证转换到的类型是对象的实际类型，否则行为未定义。

`dynamic_cast`会对类型进行运行时检查，运行时检查使用了**RTTI（Run-Time Type Information）**，为了避免额外开销，**`dynamic_cast`向下转换只能在多态类型（包含虚函数的类型）中使用**，如果运行时类型不匹配，`dynamic_cast`会抛出类型为`std::bad_cast`的异常。

为了运行下面的例子，需要之前的将`all_info`加上`virtual`。
```cpp
int main() {
    student a("John", "666");
    person &p = a;
    student &s1 = dynamic_cast<student &>(p);
    mathematician &s2 = dynamic_cast<mathematician &>(p); // throw std::bad_cast
}
```

`dynamic_cast`还可以进行cross-casting，也就是将实际类型为两个基类的共同子类的对象从一个基类转换到另一个基类，例如：
```cpp
int main() {
    math_student a("John", "666", "Fermat's Last Theorem");
    mathematician &b = a;
    student &c = dynamic_cast<student &>(b);
}
```

## 6.5.2 `const`-Cast
`const_cast`添加或移除类型的`const`与/或`volatile`限定。移除`const`或`volatile`通常只用于兼容性目的，且极易导致未定义行为。

## 6.5.3 Reinterpretation Cast
`reinterpret_cast`通过重新解释底层位模式在类型间转换。这同样是一个极易导致未定义行为的操作。其详细规则见cppreference。

## 6.5.4 Function-Style Conversion
我们可以利用构造函数或转型运算符来自定义类型转换，这样的类型转换可以以函数调用的形式编写（如果没有使用`explicit`也可以隐式进行）。

如果同时定义了A到B的构造函数和转型运算符，那么优先使用构造函数。

```cpp
struct B; // 为了声明operator B()的前向声明
struct A {
    A(double value)
        : value(value) {}
    operator B();
    double value;
};

struct B {
    B(double value)
        : value(value) {}
    B(const A &a)
        : value(a.value) {
        std::cout << "B(const A &a)\n";
    }
    double value;
};

// 在B类型完整后再编写函数体
A::operator B() {
    std::cout << "operator B()\n";
    return {value};
}
int main() {
    A a(1);
    B b = B(a);
}
```
输出
```
operator B()
```

一个不一致之处在于，对基本类型使用函数调用形式的类型转换的写法实际上是C风格类型转换，即`long(a)`实际上是`(long)a`。由于C风格类型转换过于自由，导致有可能意外地发生不希望的强制类型转换。
```cpp
double d = 3.0;
const double * const p = &d;
long l = long(dp); // 发生了意外的强制类型转换，隐含了const_cast和reinterpret_cast
```

统一初始化或`static_cast`提供了更加安全的写法。
```cpp
double d = 3.0;
const double * const p = &d;
long l1 = long{dp}; // error
long l2 = static_cast<long>{dp}; // error
```

## 6.5.5 Implicit Conversions
C++中的隐式转换详见cpprefrence。

任何数值类型都可以隐式互相转换，但在统一初始化中需要满足不丢失精度的要求。

# 6.6 CRTP
**CRTP（Curiously Recurring Template Pattern，奇特重现模板模式）**是C++的一种惯用手法。CRTP通过巧妙地结合模板和继承，避免了运行时多态的开销，同时相比单纯用模板实现的编译时多态减少了重复代码，常被用于给一个类添加特定的功能。

## 6.6.1 A Simple Example
实现了一个类型的`operator==`后通常也要实现该类型的`operator!=`。通常`operator!=`都只是`operator==`的结果取反，为了避免重复，我们可以使用下面的CRTP写法。
```cpp
template <typename T>
struct inequality {
    bool operator!=(const T &that) const {
        return !(static_cast<const T &>(*this) == that);
    }
};

struct point : public inequality<point> {
    point(int x, int y)
        : x(x), y(y) {}
    bool operator==(const point &that) const {
        return x == that.x && y == that.y;
    }
    int x, y;
};
```
可以看到，CRTP就是让一个类继承以其自身为模板参数的模板类。由于模板直到被调用才会被实例化，这样的双向依赖是可以正常编译的。

## 6.6.2 A Reusable Access Operator
由于模板的延迟编译，可以利用模板解决两个类相互依赖对方的完整类型的情况。