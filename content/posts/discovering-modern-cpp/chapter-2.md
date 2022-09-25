---
title: "《现代C++探秘》笔记 Chapter 2 - Classes"
subtitle: ""
date: 2022-09-10T16:33:54+08:00
author: "teralem"
authorLink: ""
authorEmail: ""
description: ""
keywords: ""
license: ""
comment: true
weight: 0

tags:

categories:


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

## 2.2 Members
### 2.2.2 Accessibility
使用`friend`来允许其他函数或类访问此类的`private`/`protected`成员。
```cpp
class ClassA {
    friend int func(const ClassA &);
    friend class ClassB;
    int data;
};
class ClassB {
    int get_data(const ClassA &a) {
        return a.data;
    }
};
int func(const ClassA &a) {
    return a.data;
}
```

## 2.3 Setting Values: Constructors and Assignments
### 2.3.1 Constructors
构造函数用于初始化一个对象。

类的构造函数可以附加一个**成员初始化列表（Member Initialization List）**，或称初始化列表。初始化列表可以看作一系列对成员变量的构造函数的调用（对基本变量来说是赋值）。对于没有在初始化列表中出现过的成员变量，若为基本类型则不初始化，若为类类型则进行默认初始化（如果存在的话）。

对于基本类型而言，在初始化列表里初始化还是在构造函数体里赋值并没有很大的影响；但对于类类型而言，不在初始化列表里初始化而是在构造函数体里赋值会造成额外的默认构造函数（可能还有析构函数）的调用，可能造成额外开销，更不用说有可能没有默认构造函数可用（引用也是这样的情况）。

成员变量按照定义的顺序初始化，如果成员变量在初始化列表里的顺序与定义的顺序不同，编译器会给出警告。
```cpp
class ClassA {
  public:
    ClassA() : b(1), a(2) {} // warning!
  private:
    int a, b;
};

```
作用域的隐藏机制允许我们使用相同的名称命名成员变量和构造函数参数。
```cpp
class ClassA {
  public:
    ClassA(int a, int b) : a(a), a(b) {}
  private:
    int a, b;
};
```

C++11开始支持**委托构造函数**，也就是用对其他构造函数的重载的调用来代替初始化列表（不能同时存在委托和初始化列表）。
```cpp
class ClassA {
  public:
    int data;
    ClassA(int data) : data(data) {}
    ClassA() : ClassA(0) {}
};
```

C++11开始支持为成员变量设置默认值（此前只能对静态成员变量这么做），这提供了一种更为简明的初始化成员变量的方法。如果某个构造函数的初始化列表里也初始化了这个成员变量，那么覆盖原来的默认值。

```cpp
class ClassA {
  public:
    int data = 1;
    ClassA(int data) : data(data){}
};
int main(){
	ClassA a(2);
	std::cout << a.data << '\n'; // 输出 2
	return 0;
}
```

有一个很容易错认的地方是下面的代码
```cpp
ClassA a();
```
声明了一个返回值为`ClassA`的函数而不是定义了一个类型为`ClassA`的变量并默认初始化。

没有参数（或者所有参数均为默认）的构造函数称为**默认构造函数**，没有提供参数的初始化均调用默认构造函数。如果没有编写任何构造函数，且所有成员变量都有构造默认构造函数，则编译器会自动生成一个默认构造函数。

以自身类型为唯一参数（或其后的参数均为默认）的构造函数称为**复制构造函数**。复制构造函数可以使用引用或const引用传递参数（绝大多数情况下应该使用const引用），但不能按值传递，否则就形成了循环调用。在用同类型对象初始化、赋值、按值传参等情况下都会调用复制构造函数。如果没有编写复制构造函数，且所有成员变量都有复制构造函数，则编译器会自动生成使用const引用的复制构造函数，其行为是调用所有成员变量的复制构造函数（若为基本类型则为赋值）。如果类中包含外部资源，就有必要手动实现复制构造函数。
```cpp
class ClassA {
  public:
    int data;

    ClassA(int data) : data(data) {}

    ClassA(const ClassA &a) : data(a.data) {}
    ClassA(ClassA &a) : data(a.data) {}
    ClassA(ClassA &&a) : data(a.data) {}
    ClassA(ClassA a) : data(a.data) {} // Error
};
```
以非自身类型为唯一参数（或其后的参数均为默认）的构造函数称为**转换构造函数**，能够将其他类型转换为该类型。转换构造函数可以使用按值、引用、const引用传参。定义不带`explicit`的转换构造函数后，就可以在赋值和传参时进行相应的隐式类型转换。若定义中加入了`explicit`，则禁用隐式类型转换，只能使用C风格强制类型转换或者`static_cast`进行转换。

```cpp
class ClassA {
  public:
    int data;
    ClassA() {}
    ClassA(int data) : data(data) {}
};
class ClassB {
  public:
    int data;
    ClassB() {}
    explicit ClassB(int data) : data(data) {}
};
int main() {
    ClassA a;
    ClassB b;
    a = 1;
    b = 1; // Error
    b = static_cast<ClassB>(1);
    return 0;
}
```

在定义基本类型的时候，我们经常会这样初始化：
```cpp
int a = 1;
```
虽然看着像赋值，但这并不是赋值，而是初始化，我们称其为**复制初始化**。

不带`explicit`的复制构造函数和转换构造函数可以使用复制初始化调用（这并不会造成额外开销），但若加了`explicit`就不能使用。
```cpp
class ClassA {
  public:
    int data;
    ClassA(int data) : data(data) {}
    ClassA(const ClassA &a) {data = a.data;}
};
class ClassB {
  public:
    int data;
    explicit ClassB(int data) : data(data) {}
    explicit ClassB(const ClassA &a) {data = a.data;}
};
int main() {
    ClassA a = 1;
    ClassA b = a;
	ClassB c = 1; // Error
    ClassA d = c; // Error
    return 0;
}
```

### 2.3.2 Assignment
下面是一个典型的赋值运算符的实现
```cpp
class ClassA {
  public:
    int data;
    ClassA& operator=(const ClassA& src) {
        data = src.data;
        return *this;
    }
};
```
尽管不是强制的，但为了和C++中基本类型的赋值运算符的特性保持一致，赋值运算符通常返回对自身的引用。

上面的赋值运算符的参数类型是自身类型，被称为**复制赋值函数**。如果没有编写复制赋值函数，且所有成员变量都有复制赋值函数，则编译器会自动生成复制赋值函数，复制所有成员变量。如果类中包含外部资源，就有必要手动实现复制赋值函数。

上面的赋值运算符的参数类型是也可以不是自身类型，这时就可以把其他类型赋值给此类型。这看上去很像隐式类型转换，但是隐式类型转换会多生成一个临时对象。如下面的代码：
```cpp
#include <iostream>
#include <memory>
class ClassA {
  public:
    int data;
    ClassA(int data)
        : data(data) {
        std::cout << "construct with " << data << '\n';
    }
    ~ClassA() {
        std::cout << "destruct with " << data << '\n';
    }
    ClassA &operator=(const ClassA &src) {
        std::cout << "copy assign with " << src.data << '\n';
        data = src.data;
        return *this;
    }
    ClassA &operator=(int d) {
        std::cout << "conversion assign with " << d << '\n';
        data = d;
        return *this;
    }
};
int main() {
    ClassA a = 1;
    a = 2;
    return 0;
}
```

输出
```
construct with 1
conversion assign with 2  
destruct with 2
```

如果删去转换赋值函数，输出
```
construct with 1
construct with 2
copy assign with 2
destruct with 2
destruct with 2
```

### 2.3.3 Initializer Lists
C++11引入了**初始化列表（Initializer List）**，为避免与成员初始化列表（Member Initialization List）混淆，在此使用`initializer_list`进行表述。`std::initializer_list`定义在头文件`<initializer_list>`中，尽管在多数时候这都已经被其他头文件顺带包含了。

我们可以用下面的方式对数组进行初始化。
```cpp
int v[] = {1, 2, 3};
```
`initializer_list`对这一方法进行了泛化，使之能用于自定义类型，其使用也不局限于初始化。（不过下面就会发现对原生数组本身的初始化用的反倒不是`initializer_list`而是聚合初始化）

`initializer_list`本身就是一个存储数组的容器，其特殊之处在于编译器可以自动把大括号列表解析为`initializer_list`。下面是一个可以用`initializer_list`初始化和赋值的定长数组封装实现。
```cpp
class vector {
  public:
    vector(std::initializer_list<int> values)
        : size(values.size()), data(new int[size]) {
        std::copy(values.begin(), values.end(), data.get());
    }
    vector &operator=(std::initializer_list<int> values) {
        data.reset(new int[values.size()]);
        std::copy(values.begin(), values.end(), data.get());
    }
  private:
    int size;
    std::unique_ptr<int[]> data;
};
```

### 2.3.4 Uniform Initialization
C++11 引入了**统一初始化**，统一用大括号列表表达各种初始化方式，可用于变量定义、临时变量、`new`等情形。

数组类型和满足一定条件（如所有成员变量均为`public`，无自定义构造函数，无基类，无虚函数）的类类型称为**聚合类型**。聚合类型可以使用**聚合初始化**，只需在大括号列表中依次指定各元素的值。
```cpp
struct Person {
    int id;
    std::string name;
};
int main() {
    int a[] = {1,3,4};
    Person a{1, "Alice"};
    Person b = {2, "Blice"};
    return 0;
}
```

对于非聚合类型，大括号列表可以如同小括号一般调用该类型的构造函数。在这里使用统一初始化的一点好处是可以避免前文提到的`ClassA a();`看似用默认构造函数初始化了一个变量实则是一个函数前面的问题。如果构造函数为`explicit`，则不能使用复制初始化的格式。
```cpp
struct Person {
  public:
    Person(int id, std::string name)
        : id(id), name(name) {}

  private:
    int id;
    std::string name;
};
int main() {
    Person a{1, "Alice"};
    Person b = {2, "Blice"}; // 仅当构造函数未声明为explicit时
    return 0;
}
```

如果要初始化一个构造函数的参数类型为`initializer_list`的类型，理论上应该这样写：
```cpp
std::vector<int> a = {{1, 2, 3}};
```

但C++提供了**括号消除**，允许我们去掉一层大括号，将上面的代码简化为：

```cpp
std::vector<int> a = {1, 2, 3};
```

从而实现了上一节中`initializer_list`的用法。

但括号消除也会导致有时难以识别大括号中是构造函数的参数还是`initializer_list`，在两者均可的情况下优先认为是`initializer_list`。

最后，正如在第一章开头提到的，统一初始化中禁止数值类型的窄化（丢失精度）隐式类型转换。在成员初始化列表中也可以用大括号代替小括号来避免窄化隐式类型转换。

### 2.3.5 Move Semantics
C++11引入的右值引用等语言设施实现了移动语义。

这里先抛开书详谈一下**值类别**和**右值引用**

#### 值类别（value category）
每个C++表达式都由两种独立的特性加以辨别：类型和值类别。 
在C++14中，值类别有如下定义（引自cppreference）

    随着移动语义引入到 C++11 之中，值类别被重新进行了定义，以区别表达式的两种独立的性质：
    * 拥有身份 (identity)：可以确定表达式是否与另一表达式指代同一实体，例如通过比较它们所标识的对象或函数的（直接或间接获得的）地址；
    * 可被移动：移动构造函数、移动赋值运算符或实现了移动语义的其他函数重载能够绑定于这个表达式。

    * 拥有身份且不可被移动的表达式被称作左值 (lvalue) 表达式；
    * 拥有身份且可被移动的表达式被称作亡值 (xvalue) 表达式；
    * 不拥有身份且可被移动的表达式被称作纯右值 (prvalue) 表达式；
    * 不拥有身份且不可被移动的表达式无法使用。

    拥有身份的表达式被称作“泛左值 (glvalue) 表达式”。左值和亡值都是泛左值表达式。
    可被移动的表达式被称作“右值 (rvalue) 表达式”。纯右值和亡值都是右值表达式。

在C++17中，值类别的定义进行了修改，这里暂不考虑。

#### 右值引用
右值引用只能引用右值。`T &&a = rvalue;`定义一个右值引用`a`引用右值`rvalue`。需要注意的是**右值引用本身是泛左值**，即右值引用可以是左值或亡值（使用`std::move`等情况），但不会是纯右值。引用/右值引用属于类型系统，而左值/亡值/纯右值属于值类别系统，两者需要区分。

```cpp
class ClassA {
  public:
    ClassA(int data) : data(data) {
        std::cout << "construct with " << data << '\n';
    }
    ~ClassA() {
        std::cout << "destruct with " << data << '\n';
    }
    int data;
};
int main() {
    {
        ClassA &&a = ClassA(1);
        a.data = 2; // 若改为const &，则这一句报错
    }
    std::cout << "end of main\n";
    return 0;
}
```

输出

```
construct with 1
destruct with 2
end of main
```

从上面的代码中可以发现如果用右值引用引用一个右值，那么和用const引用引用一个右值一样，这一右值的生存期被延长到匹配该引用的生命期。但不同于const引用的是可以用右值引用修改原对象。

const右值引用同样是存在的，只能引用右值且不能修改原对象，不过这玩意儿的应用场景比较少。

到现在我们有了引用、const引用、右值引用、const右值引用四种引用类型，看起来非常混乱。我们通过下面的代码来研究一下这些类型在重载中的行为。

```cpp
void func(std::string &a) {
    std::cout << a << " -> lvalue ref\n";
}
void func(const std::string &a) {
    std::cout << a << " -> const lvalue ref\n";
}
void func(std::string &&a) {
    std::cout << a << " -> rvalue ref\n";
}
void func(const std::string &&a) {
    std::cout << a << " -> const rvalue ref\n";
}
std::string &&like_move(std::string &a) {
    return static_cast<std::string &&>(a);
}
const std::string &&like_move_const(std::string &a) {
    return static_cast<const std::string &&>(a);
}
int main() {
    std::string obj = "non-ref lvalue";
    std::string ref_obj = "lvalue ref";
    std::string &ref = ref_obj;
    const std::string &const_ref = "const lvalue ref";
    std::string &&rvalue_ref = "rvalue ref";
    const std::string &&const_rvalue_ref = "const rvalue ref";
    std::string move_obj1 = "rvalue ref as xvalue";
    std::string move_obj2 = "const rvalue ref as xvalue";

    func(obj);
    func(ref);
    func(const_ref);
    func(rvalue_ref);
    func(const_rvalue_ref);
    func("pure rvalue");
    func(like_move(move_obj1));
    func(like_move_const(move_obj2));
    return 0;
}
```

输出
```
non-ref lvalue -> lvalue ref
lvalue ref -> lvalue ref
const lvalue ref -> const lvalue ref
rvalue ref -> lvalue ref
const rvalue ref -> const lvalue ref
pure rvalue -> rvalue ref
rvalue ref as xvalue -> rvalue ref
const rvalue ref as xvalue -> const rvalue ref
```

左值引用本身均为左值，前三个结果无需更多解释。使用名称访问的右值引用本身是左值，因此右值引用调用了左值引用重载，const右值引用调用了const左值引用重载。作为函数返回值的右值引用本身是亡值，因此最后三个是纯右值和亡值，均属于右值，因此调用右值引用重载。在这里也可以看出对右值来说右值引用的重载优先级高于const左值引用（先看右值引用，再看const右值引用，最后看const左值引用）。

（关于这个示例代码的细节：`"pure rvalue"`这样的字符串字面量实际上是左值，但由此构造出的`std::string`对象是纯右值。）

#### 移动语义
回到书本。
移动语义的意图是将一个不再需要的对象中的资源“偷”到另一个对象中来避免复制资源的开销，为了实现这一点，我们需要**移动构造函数**和**移动赋值函数**。下面是一个典型的移动构造函数和移动赋值函数的实现。（注意移动赋值函数中利用`std::swap`的实现技巧，我们把释放自身资源的工作交给了被移动对象的析构函数，这会导致资源要到原对象的生命期结束才会被释放，如果要立即释放则需要在移动赋值函数编写释放代码。）
```cpp
class ClassA {
  public:
    // ...
    ClassA(ClassA &&a)
        : size(a.size), data(a.data) {
        a.size = 0;
        a.data = nullptr;
    }
    ClassA &operator=(ClassA &&a) {
        std::swap(size, a.size);
        std::swap(data, a.data);
        return *this;
    }
  private:
    int size;
    int *data;
};
```

可见移动构造函数和移动赋值函数就是复制构造函数和复制赋值函数的参数为右值引用的重载，当传入的参数为右值时调用，并将传入对象的资源偷过来。一个对象在以右值方式传入函数后就被视为无效（expired），其数据处于不确定状态，唯一的要求是析构函数必须正常执行，例如必须将指针置为`nullptr`以使`delete`/`delete[]`意识到资源已经被移动，无需释放。

传入纯右值的情况很好理解：一个临时对象被移动给了另一个对象。不过在移动构造函数中传入纯右值实际上并没有必要，因为我们完全可以把对象直接构造在最后定义出的变量上（见下文复制消除）。这一点在在C++17中从优化建议变成了标准强制，也因此在C++17中值类型的定义进行了修改，不再允许纯右值移动，而是在需要时通过临时量实质化转换为亡值进行移动。

如果要移动一个原本就存在的左值，我们需要通过`std::move`将其转化为亡值:

```cpp
class ClassA {
  public:
    ClassA(int size)
        : size(size), data(new int[size]) {
        std::cout << "construct " << size << '\n';
    }
    ~ClassA() {
        std::cout << "destruct " << size << '\n';
        delete[] data;
    }
    ClassA(ClassA &&a)
        : size(a.size), data(a.data) {
        std::cout << "move construct " << size << '\n';
        a.size = 0;
        a.data = nullptr;
    }
    ClassA &operator=(ClassA &&a) {
        std::cout << "move assign " << size << '\n';
        std::swap(size, a.size);
        std::swap(data, a.data);
        return *this;
    }

  private:
    int size;
    int *data;
};
int main() {
    ClassA a = ClassA(1);
    ClassA b = ClassA(2);
    b = std::move(a);
    return 0;
}
```

输出
```
construct 1
construct 2
move assign 2
destruct 1
destruct 2
```

对象被移动后要到生存期结束才会被析构，在此期间对象处于合法但不确定的状态，进行与此对象状态有关的操作是未定义行为。

#### 复制消除

许多情况下的复制/移动操作是没有必要的，这时编译器可以将这些复制/移动操作优化掉，注意这会造成复制/移动构造函数以及析构函数中的副作用发生改变，这是C++中优化可以影响程序副作用的一个特例，因此程序不应该依赖于复制/移动构造函数以及析构函数中的副作用。从C++17开始， return与函数返回类型相同的类类型的纯右值（即RVO，返回值优化），以及在对象的初始化中，当初始化器表达式是一个与变量类型相同的类类型的纯右值（见下代码）时进行复制/移动消除变成强制要求而不再是优化建议。
```cpp
ClassA a = ClassA(1); // 这不会造成复制或移动
```

## 2.4 Destructors

### 2.4.1 Implementation Rules
1. 在析构函数中绝不能抛出异常。
2. 如果类中有虚函数，那么析构函数必须也为虚函数。

### 2.4.2 Dealing with Resources Properly
在C++中，我们使用**RAII(Resource Acquisition Is Initialization)**管理资源，其关键在于**在析构函数中释放对象使用的资源**（与名字不同的是构造函数并没有那么重要，因为对象完全可以不在构造函数而在其后某一时刻获取资源）。

如果在构造函数中抛出异常，析构函数将不会被执行，构造函数需要自行确保此前获取的资源全部被释放。

有时我们可以通过自定义`deleter`把智能指针用于管理内存以外的资源。例如书中例子：
```cpp
struct environment_deleter {
    void operator()(Environment* env) {
        terminateEnvironment(env);
    }
}
std::shared_ptr<Environment> environment(createEnvironment(), environment_deleter{});
```
可以在`deleter`中保存用于删除该资源需要的额外信息。

```cpp
struct connection_deleter {
    connection_deleter(std::shared_ptr<Environment> env) : env(env) {}
    void operator()(Connection* conn) {
        env->terminateEnvironment(conn);
    }
    shared_ptr<Environment> env;
}
std::shared_ptr<Connection> connection(environment->createConnection(...), connection_deleter{environment});
```

## 2.5 Method Generation Resume
编译器会自动生成如下函数：
* 默认构造函数
* 复制构造函数
* 移动构造函数
* 复制赋值函数
* 移动赋值函数
* 析构函数

这些函数的具体生成规则比前述的更为复杂，详见A.5。

## 2.6 Accessing Member Variables

### 2.6.1 Access Functions
在C++中经常会编写这样的方法来访问私有成员变量。
```cpp
class ClassA {
  public:
    int &data() {
        return d;
    }

  private:
    int d;
};
```
这返回了一个对私有成员变量的引用，这在使用临时对象时需要格外注意。
```cpp
class ClassA {
  public:
    ClassA(int d)
        : d(d) {
        std::cout << "construct " << d << '\n';
    }
    ~ClassA() {
        std::cout << "destruct " << d << '\n';
    }
    int &data() {
        return d;
    }

  private:
    int d;
};
int main() {
    std::cout << ClassA(1).data() << '\n';

    const int &r = ClassA(2).data();
    std::cout << r << '\n';
    return 0;
}
```
输出
```
construct 1
1
destruct 1
construct 2
destruct 2
2
```
可以发现第一部分是正确的，因为引用被使用时临时对象还在生命期内。但第二部分是错的，临时对象在第一条语句结束时生命期就已经结束，执行第二条语句时`r`已经为悬垂引用。这实际上是临时量的生存期的特例——`return`语句中绑定到函数返回值的临时量不会被延续。如果直接访问成员变量，就会正确地按照const引用和右值引用的规则，将临时变量的生存期延长到匹配该引用的生命期。
```cpp
class ClassA {
  public:
    ClassA(int d)
        : d(d) {
        std::cout << "construct " << d << '\n';
    }
    ~ClassA() {
        std::cout << "destruct " << d << '\n';
    }
    int &data() {
        return d;
    }
    int d;
};
int main() {
    const int &r = ClassA(2).d;
    std::cout << r << '\n';
    return 0;
}
```
输出
```
construct 2
2
destruct 2
```
可以发现没有悬垂引用。

### 2.6.3 Constant Member Functions
可以在成员函数声明的括号后加上`const`来限制此函数不能修改对象的状态。各种会导致对象状态被改变的操作都会被禁止，也不能调用没有用`const`修饰的成员函数。`const`对象只能调用其中用`const`修饰的成员函数。
```cpp
class ClassA {
  public:
    ClassA(int data)
        : data(data) {
    }
    void set(int d) const {
        data = d; // error
    }
    int &get1() const {
        return data; // error
    }
    const int &get2() const {
        func(); // error
        return data;
    }
    void func() {
    }

  private:
    int data;
};
```
`const`是函数签名的一部分，可以同时定义`const`版本和非`const`版本。
```cpp
class ClassA {
  public:
    ClassA(int d)
        : d(d) {
    }
    int &data() {
        return d;
    }
    const int &data() const {
        return d;
    }

  private:
    int d;
};
int main() {
    const ClassA a = ClassA(1);
    const int &b = a.data(); // 如果没有const版本的data，data将无法调用
    return 0;
}
```

可以用`mutable`修饰成员变量，使其即使在`const`成员函数或者在`const`对象中也能修改，这可以用于实现缓存等，但总之尽可能不用。

### 2.6.4 Reference-Qualified Members
C++11引入了引用限定符，可以指定成员函数只能用对象的左值或右值调用。
```cpp
class ClassA {
  public:
    void f() & {
        std::cout << "lvalue\n";
    }
    void f() && {
        std::cout << "rvalue\n";
    }
};

int main() {
    ClassA a;
    a.f();
    std::move(a).f();
    ClassA().f();
}
```
输出
```
lvalue
rvalue
rvalue
```
这可以用来明确一些操作在右值上没有意义，或在知道自己是右值的情况下把自己的资源送到别处。

## 2.7 Operator Overloading Design

### 2.7.3 Member or Free Function
大多数重载运算符既可以作为成员函数也可以作为自由函数。下面讨论一下区别。

```cpp
class complex {
  public:
    complex(double r = 0.0, double i = 0.0)
        : r(r), i(i) {}
    complex operator+(const complex &c2) const {
        return complex(r + c2.r, i + c2.i);
    }

  private:
    double r, i;
};

int main() {
    complex a(1.0, 2.0);
    complex b = a + 1.0;
    complex c = 1.0 + a; // error
}
```
如上，如果二元重载运算符是成员函数，那么只有第二个参数能够被隐式类型转换，但如果使用自由函数，那么两个参数都可以被隐式类型转换。如果第一个参数是基本类型，那么就只能用自由函数了。（一元重载运算符同样，只有自由函数才能将参数隐式类型转换。）

改为自由函数实现：
```cpp
class complex {
  public:
    complex(double r = 0.0, double i = 0.0)
        : r(r), i(i) {}
    friend complex operator+(const complex &, const complex &);

  private:
    double r, i;
};
inline complex operator+(const complex &c1, const complex &c2) {
    return complex(c1.r + c2.r, c1.i + c2.i);
}
int main() {
    complex a(1.0, 2.0);
    complex b = a + 1.0;
    complex c = 1.0 + a;
}
```

这里使用了友元访问私有变量，当然也可以用`real()`这样的接口访问。

此外，使用自由函数的实现看上去也更加对称。

## A.4 Class Details
### A.4.1 Pointer to Member
C++支持**成员指针**，指向某个类中的某个类型的对象，使用**成员指针访问运算符** `.*` 与 `->*`来访问。
```cpp
class ClassA {
  public:
    int data;
};
int main() {
    int ClassA::*p = &ClassA::data; // 指向ClassA中某个int类型的成员
    ClassA a, *b = &a;
    a.*p = 2;
    std::cout << b->*p << '\n';
}
```

## A.5 Method Generation
编译器会自动生成如下函数：
* 默认构造函数
* 复制构造函数
* 移动构造函数
* 复制赋值函数
* 移动赋值函数
* 析构函数

### A.5.1 Controlling the Generation
C++11开始可以使用` = default`要求编译器生成具有默认行为的函数，使用` = delete`阻止编译器生成函数。下面的类只能被复制，不可被移动。
```cpp
class copy_only {
  public:
    copy_only() = default;
    copy_only(const copy_only &) = default;
    copy_only(copy_only &&) = delete;
    copy_only &operator=(const copy_only &) = default;
    copy_only &operator=(copy_only &&) = delete;
    ~copy_only() = default;
};
```
用` = default`或者` = delete`这声明的函数都会被认为是用户定义的（与写明函数体效果相同）。

如果调用函数时匹配到了声明为` = delete`的函数，会直接产生编译错误而非寻找其他可行的重载。如上面的代码，我们把移动操作写明为` = delete`，这时移动会导致编译错误。但如果不把移动操作写明为` = delete`，则传入右值时会自动重载到const引用，导致实际发生的是复制操作。

### A.5.2 Generation Rules
这六个函数是否自动生成的完整规则较为复杂，其中摘录较为重要的三点：
1. 如有任何用户定义的构造函数，默认构造函数都不会被生成
2. 如果没有用户定义的析构函数，那么析构函数必定会被生成
3. 如果用户定义了复制函数，那么对应的移动函数不会被生成；反之亦然

### A.5.3 Pitfalls and Design Guides
尽管规则复杂，但需要记住的是下面的设计规则：
1. Rule of Five: 除了默认构造函数外的五个函数要么全部声明要么全部不声明。
2. Rule of Zero: 如果类中用到的资源都已经以RAII的对象的形式使用（如`unique_ptr` `fstream`）了，那么五个函数都不用实现。
3. Rule of Six: 对于这六个函数，尽量多声明(` = default`, ` = delete`)少实现