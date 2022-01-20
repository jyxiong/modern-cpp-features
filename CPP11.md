# C++11

## 概述
这些说明和例子参考了许多资源(详见 [致谢](#致谢) 部分)自己总结的。

C++11 包含下列新的语言功能
- [移动语意](#移动语意)
- [右值引用](#右值引用)
- [转发引用](#转发引用)
- [移动语意下的特殊成员函数](#移动语意下的特殊成员函数)

C++11 包含下列新的库功能
- [`std::move`](#stdmove)
- [`std::forward`](#stdforward)

## C++11 语言功能

### 移动语意
移动语意是指将一个对象管理的资源的所有权转移给另一个对象。

移动语意(move semantics)的第一个好处是性能的优化。当一个对象即将到达生命周期的尾声时, 无论是因为它是一个临时对象, 还是通过显示地调用 `std::move`, 移动语意通常是一种更便宜的资源转移方式。例如, 移动一个 `std::vector` 只要将指针和内部状态拷贝到新的 vector, 而复制则必须拷贝 vector 包含的每个完整元素。在即将销毁原始 vector 的情况下, 复制是费力不讨好的方式。

移动语意还能让不可复制的类型, 如 `std::unique_ptr` ([智能指针](#智能指针)), 在语言层面上确保只管理一个资源实例的同时, 还能在作用域内转移资源实例。

参考其它部分: [右值引用](#右值引用), [转发引用](#转发引用), [移动语意的特殊成员函数](#移动语意的特殊成员函数), [`std::move`](#stdmove), [`std::forward`](#stdforward)。

### 右值引用
C++11引入了一种新的引用类型: _右值引用_ (rvalue reference)。用 `T&&` 表示一个非模板类型参数 `T` (non-template type parameter, 例如 `int` 类型, 或者自定义类型)的右值引用。右值引用只能绑定右值。

左值和右值的类型推断:
```c++
int x = 0; // `x` 是 `int` 类型的左值
int& xl = x; // `xl` 是 `int&` 类型的左值
int&& xr = x; // 编译错误 -- `x` 是左值, 右值引用不能绑定到左值
int&& xr2 = 0; // `xr2` 是 `int&&` 类型的左值 -- 绑定了临时的右值 `0`

void f(int& x) {}
void f(int&& x) {}

f(x);  // 调用 f(int&)
f(xl); // 调用 f(int&)
f(3);  // 调用 f(int&&), `3` 是临时的右值
f(std::move(x)) // 调用 f(int&&), 调用 `std::move` 生成右值

f(xr2);           // ** 调用 f(int&)
f(std::move(xr2)) // ** 调用 f(int&& x), 调用 `std::move` 生成右值
```

### 转发引用
转发引用(forwarding references)又叫 _万能引用_ (universal references, 非官方)。转发引用可以用 `T&&` 表示, 其中 `T` 是模板类型参数(template type parameter)。也可以直接使用 `auto&&` 表示。转发引用实现了 _完美转发_ (perfect forwarding): 传递参数时保持参数的值类型(value category, 包括左右值, mutable, const, voliate等)不变。例如, 左值还是左值, 临时变量还是右值。

转发引用可以根据类型分别绑定到左值或者右值。转发引用遵循 _引用折叠_ (reference collapsing)规则:
* `T& &` -> `T&`
* `T& &&` -> `T&`
* `T&& &` -> `T&`
* `T&& &&` -> `T&&`

左值/右值的 `auto` 类型推断:
```c++
int x = 0; // `x` 是 `int` 类型的左值
auto&& al = x; // `al` 是 `int&` 类型的左值 -- 绑定到左值 `x`
auto&& ar = 0; // `ar` 是 `int&&` 类型的左值 -- 绑定到临时右值 `0`
```

左值/右值的模板类型参数推断:
```c++
// C++14 及以上:
void f(auto&& t) {
  // ...
}

// C++11 及以上:
template <typename T>
void f(T&& t) {
  // ...
}

int x = 0;
f(0); // T 是 int 类型的右值, 推断为 f(int &&) => f(int&&)
f(x); // T 是 int& 类型的左值, 推断为 f(int& &&) => f(int&)

int& y = x;
f(y); // T 是 int& 类型的左值, 推断为 f(int& &&) => f(int&)

int&& z = 0; // 注意: `z` 是 `int&&` 类型的左值.
f(z); // T 是 int& 类型的左值, 推断为 f(int& &&) => f(int&)
f(std::move(z)); // T 是 int 类型的右值, 推断为 f(int &&) => f(int&&)
```

参照: [右值引用](#右值引用), [`std::move`](#stdmove), [`std::forward`](#stdforward)。

### 移动语意下的特殊成员函数
当发生拷贝操作时, 拷贝构造函数和拷贝赋值操作符被调用。随着 C++11 引入了移动语意, 可以采用移动构造函数和移动赋值操作符来转移所有权。
```c++
struct A {
  std::string s;
  A() : s{"test"} {} // 默认构造
  A(const A& o) : s{o.s} {} // 拷贝构造
  A(A&& o) : s{std::move(o.s)} {} // 移动构造函数
  A& operator=(A&& o) { // 移动赋值操作符
   s = std::move(o.s);
   return *this;
  }
};

A f(A a) {
  return a;
}

A a1 = f(A{}); // 使用临时右值进行移动构造
A a2 = std::move(a1); // 使用 std::move 进行移动构造
A a3 = A{}; // 默认构造
a2 = std::move(a3); // 使用 std::move 进行移动赋值
a1 = f(A{}); // 使用临时右值进行移动赋值
```

## C++11 库功能

### `std::move`
`std::move` 表示作为参数传入的对象转移了它的资源所有权。应当非常小心地使用已经移动过的对象, 因为不能确认它们的状态(参考: [What can I do with a moved-from object?](http://stackoverflow.com/questions/7027523/what-can-i-do-with-a-moved-from-object))。

`std::move` 的定义(move 操作就是转换成右值引用):
```c++
template <typename T>
typename remove_reference<T>::type&& move(T&& arg) {
  return static_cast<typename remove_reference<T>::type&&>(arg);
}
```

转移 `std::unique_ptr`:
```c++
std::unique_ptr<int> p1 {new int{0}};  // 实践中请使用 std::make_unique<int>(new int(0))
std::unique_ptr<int> p2 = p1; // 错误 -- 不能复制 unique_ptr
std::unique_ptr<int> p3 = std::move(p1); // 将 `p1` 移动到 `p3`
                                         // 现在使用 `p1` 指向的对象是不安全的
```

### `std::forward`
返回传入参数的同时保持它们的值类型和 cv-qualifiers 不变(包括左右值, mutable, const, voliate等)。有利于范型编程和工厂模式(useful for generic code and factories)。与[转发引用](#转发引用)结合使用。

`std::forward` 的定义:
```c++
template <typename T>
T&& forward(typename remove_reference<T>::type& arg) {
  return static_cast<T&&>(arg);
}
```

`wrapper` 函数的示例, 通过转发 `A` 的实例对象作为拷贝构造函数和移动构造函数的参数:
```c++
struct A {
  A() = default;
  A(const A& o) { std::cout << "copied" << std::endl; }
  A(A&& o) { std::cout << "moved" << std::endl; }
};

template <typename T>
A wrapper(T&& arg) {
  return A{std::forward<T>(arg)};
}

wrapper(A{}); // 传入临时右值, 经过转发, 调用移动构造函数
A a;
wrapper(a); // 传入左值, 经过转发, 调用拷贝构造函数
wrapper(std::move(a)); // 传入移动生成的右值, 经过转发, 调用移动构造函数
```

### 智能指针

## 致谢
* [cppreference](http://en.cppreference.com/w/cpp) - 示例和文档尤其有用。
* [C++ Rvalue References Explained](http://thbecker.net/articles/rvalue_references/section_01.html) - 一文理解右值引用, 完美转发和移动语意。
* [clang](http://clang.llvm.org/cxx_status.html) 和 [gcc](https://gcc.gnu.org/projects/cxx-status.html) 支持的C++标准查询网页。包含了语言/库功能的建议, 可以查看每个版本修复/改进的内容和示例。
* [Compiler explorer](https://godbolt.org/)
* [Scott Meyers' Effective Modern C++](https://www.amazon.com/Effective-Modern-Specific-Ways-Improve/dp/1491903996) - 非常推荐的一本书!
* [Jason Turner's C++ Weekly](https://www.youtube.com/channel/UCxHAlbZQNFU2LgEtiqd2Maw) - 一些不错的C++相关的视频。
* [What can I do with a moved-from object?](http://stackoverflow.com/questions/7027523/what-can-i-do-with-a-moved-from-object)
* [What are some uses of decltype(auto)?](http://stackoverflow.com/questions/24109737/what-are-some-uses-of-decltypeauto)
* 还有更多的被遗忘的帖子...

## 作者
原文: Anthony Calandra

翻译: 宇半仙

## 内容贡献者
原文: https://github.com/AnthonyCalandra/modern-cpp-features/graphs/contributors

翻译: https://github.com/jyxiong/modern-cpp-features/graphs/contributors

## License
MIT