---
title: "《现代C++探秘》笔记 Chapter 4 - Libraries"
subtitle: ""
date: 2023-03-21T20:41:57+08:00
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

# 4.1 Standard Template Library

注意`const_iterator`和`const iterator`的差别。例如`std::vector<int>::const_iterator`表示不能修改指向的元素的迭代器，而`const std::vector<int>::iterator`表示自身不能修改的迭代器。`const_iterator`可以通过被`const`修饰的容器的`begin`/`end`得到，C++11后还可通过`cbegin`/`cend`得到。

C++11引入了`begin`/`end`的自由函数版本。使用自由函数可以给没有实现`begin`/`end`成员函数的类型添加`begin`/`end`（包括C风格数组），因此下面的写法可以带来最大的通用性。
```cpp
template <typename T, typename U>
int sum(const T &container, U init) {
    using std::begin, std::end;
    for (auto it = begin(container), e = end(container); it != e; it++) {
        init += *it;
    }
    return init;
}

int main() {
    int a[] = {1, 2, 3};
    auto s = sum(a, 0);
    std::cout << s <<'\n';
}
```
注意我们没有直接写`std::begin`和`std::end`，而是使用了`using std::begin, std::end;`这让自定义类型定制的`begin`和`end`函数能够通过ADL被查找到。

从C++14开始`cbegin`/`cend`也有对应自由函数版本。

头文件`<iterator>`中的`advance`和`distance`可以便于移动非随机访问迭代器和计算非随机访问迭代器的距离。

C++11中引入的`emplace`系列函数接收构造函数参数并将对象直接构造在容器中，避免了复制或移动的开销。`emplace`系列函数甚至可以往容器中放入既不能复制也不能移动的类型。然而由于`vector`具有重新分配空间的扩容机制，既不能复制也不能移动的类型是不能放入`vector`的，但`deque`可以保证元素放入后不被复制或移动（除非一定要往中间插入元素）。

`map`的`operator[]`会在被查找的key不存在时自动插入该key，这一行为使得`const`修饰的`map`不提供`operator[]`。要想避免意外地插入元素，可以使用`find`或`at`成员函数。`find`在key不存在时返回end迭代器，而`at`在key不存在时抛出`std::out_of_range`异常。

头文件`numeric`中有一些有趣的函数，如
* `iota` 用从起始值开始连续递增的值填充一个范围
* `accumulate` 对一个范围内的元素求和
* `inner_product` 计算两个范围的元素的内积
* `adjacent_difference` 计算范围内各相邻元素之间的差（差分）
* `partial_sum` 计算计算范围内元素的部分和（前缀和）

更多内容见cppreference。

流迭代器也是一个非常有趣的东西，下面只放一个例子。
```cpp
int main() {
    std::istringstream str("0.1 0.2 0.3 0.4");
    std::copy(std::istream_iterator<double>(str),
                     std::istream_iterator<double>(),
                     std::ostream_iterator<double>(std::cout, ", "));
}
```
输出
```
0.1, 0.3, 0.6, 1, 
```

# 4.2 Numerics
C++11提供了一个相当复杂的随机数库。下面是一个最为简单的封装使用。
```cpp
std::default_random_engine &global_urng() {
    static std::default_random_engine u{};
    return u;
}

void randomize() {
    static std::random_device rd{};
    global_urng().seed(rd());
}

// [from, thru]
int pick_int(int from, int thru) {
    static std::uniform_int_distribution<> d{}; // 模板参数默认为int，故留空
    using param_t = decltype(d)::param_type;
    return d(global_urng(), param_t{from, thru});
}

// [from, upto)
int pick_double(double from, double upto) {
    static std::uniform_real_distribution<> d{}; // 模板参数默认为double，故留空
    using param_t = decltype(d)::param_type;
    return d(global_urng(), param_t{from, upto});
}
```

# 4.3 Meta-programming
## 4.3.1 Limits
`std::numeric_limits`提供了查询各种算术类型属性的标准化方式，如`std::numeric_limits<long>::min()`可以获取`long`类型的最小值。然而一个比较怪异的行为是对于浮点数，如`std::numeric_limits<double>::min()`表示的是`double`可表示的最小的正数。为了解决这一不一致性，C++11引入了`lowest`来将行为统一为类型可表示的最低有限值。

`std::numeric_limits`还可以提供很多属性，详见cppreference。

## 4.3.2 Type Traits
`<type_traits>`头文件中的各种获取类型特征的类为元编程提供了有力工具。其具体内容不宜在此详述。

`__cplusplus`宏会被展开为编译时使用的C++标准的发布日期（注意msvc出于兼容性有一些特殊处理）。这个宏常用于根据当前是C语言还是C++或者当前的C++标准版本进行选择性编译。

# 4.4 Utilities
# 4.4.1 Tuple
C++11引入了`tuple`。可以使用`get<0>(t)`、`get<1>(t)`来获取`t`的第一、第二个元素。从C++14开始在没有歧义的情况下也可以用`get<T>(t)`获取`t`中类型为`T`的元素。

使用`std::tie`可以在一条语句中将`tuple`的多个元素取出。C++17引入的结构化绑定进一步简化了这一写法。
```cpp
int main()
{
    auto t = std::make_tuple(4, 7);
    int a, b;
    std::tie(a, b) = t;
}
```

在函数返回时最好在`make_tuple`的参数中加上`move`来提高效率。
```cpp
auto func() {
    std::string a;
    std::vector<int> b;
    return std::make_tuple(std::move(a), std::move(b));
}
```

## 4.4.2 function
C++11中加入的`std::function`可以包装任何可调用目标，具有极大的灵活性。当`std::function`没有指定可调用目标时为空，调用空的`std::function`会引发`std::bad_function_call`异常。

`std::bind`可以用于将全部或部分参数绑定到指定的可调用目标上（若为非静态成员函数，则首参数为其类型的引用或指针）。如果只绑定部分参数，那么未绑定的参数需要用`std::placeholders::_1, _2, ... _N`占位。
```cpp
class A {
  public:
    A(int id)
        : id(id) {}
    auto get_say_call() const {
        return std::bind(say, this, std::placeholders::_1);
    }
    void say(const std::string &msg) const {
        std::cout << id << " say: " << msg << "\n";
    }

  private:
    int id;
};

int main() {
    A a(123);
    auto f = a.get_say_call();
    f("hello");
}
```

实际上，`std::bind`往往不如lambda表达式好用。上面的例子可以改写成：
```cpp
auto get_say_call() const {
    return [this](auto msg) { say(msg); };
}
```

## 4.4.3 Reference Warpper
C++11提供了`reference_warpper`来包装引用，使其可以被放入容器中。
两个辅助函数`ref`和`cref`返回对传入对象的`reference_warpper`以及`const`版本。
```cpp
int main() {
    int a = 1;
    auto rw = std::ref(a);  // std::reference_wrapper<int>
    auto crw = std::cref(a); // std::reference_wrapper<const int>
}
```