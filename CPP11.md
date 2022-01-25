# C++11

## 概述
这些说明和例子参考了许多资源(详见 [致谢](#致谢) 部分)自己总结的。

C++11 包含下列新的语言功能
- [移动语意](#移动语意)
- [变参模板](#变参模板)
- [右值引用](#右值引用)
- [转发引用](#转发引用)
- [初始化列表](#初始化列表)
- [静态断言](#静态断言)
- [auto](#auto)
- [lambda 表达式](#lambda-表达式)
- [decltype](#decltype)
- [unsing](#unsing)
- [nullptr](#nullptr)
- [强类型枚举](#强类型枚举)
- [属性](#属性)
- [constexpr](#constexpr)
- [委托构造函数](#委托构造函数)
- [用户定义字面量](#用户定义字面量)
- [override](#override)
- [final](#final)
- [default](#default)
- [delete](#deleted)
- [基于范围的 for 循环](#基于范围的-for-循环)
- [移动语意下的特殊成员函数](#移动语意下的特殊成员函数)

C++11 包含下列新的库功能
- [`std::move`](#stdmove)
- [`std::forward`](#stdforward)
- [智能指针](#智能指针)
- [`std::make_shared`](#stdmake_shared)

## C++11 语言功能

### 移动语意
移动语意是指将一个对象管理的资源的所有权转移给另一个对象。

移动语意(move semantics)的第一个好处是性能的优化。当一个对象即将到达生命周期的尾声时, 无论是因为它是一个临时对象, 还是通过显示地调用 `std::move`, 移动语意通常是一种更便宜的资源转移方式。例如, 移动一个 `std::vector` 只要将指针和内部状态拷贝到新的 vector, 而复制则必须拷贝 vector 包含的每个完整元素。在即将销毁原始 vector 的情况下, 复制是费力不讨好的方式。

移动语意还能让不可复制的类型, 如 `std::unique_ptr` ([智能指针](#智能指针)), 在语言层面上确保只管理一个资源实例的同时, 还能在作用域内转移资源实例。

参考其它部分: [右值引用](#右值引用), [转发引用](#转发引用), [移动语意的特殊成员函数](#移动语意的特殊成员函数), [std::move](#stdmove), [std::forward](#stdforward)。

### 变参模板
`...`创建或展开一个 _形参包_。模板 _形参包_ 是接受零个或多个模板实参(非类型, 类型或模板)的模板形参。至少有一个形参包的模板被称作 _变参模板_。
```c++
template <typename... T>
struct arity {
  constexpr static int value = sizeof...(T);
};
static_assert(arity<>::value == 0); // 0 个实参
static_assert(arity<char, short, int>::value == 3); // 3 个实参
```

一个有趣的用法是, 从 _形参包_ 创建一个 _初始化列表_ 来迭代变参函数的实参。
```c++
template <typename First, typename... Args>
auto sum(const First first, const Args... args) -> decltype(first) {
  const auto values = {first, args...}; // 展开
  return std::accumulate(values.begin(), values.end(), First{0});
}

sum(1, 2, 3, 4, 5); // 15
sum(1, 2, 3);       // 6
sum(1.5, 2.0, 3.7); // 7.2
```

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

参照: [右值引用](#右值引用), [std::move](#stdmove), [std::forward](#stdforward)。

### 初始化列表
初始化列表是一种轻量的类似数组的元素容器, 通过花括号`{}`创建。例如, `{ 1, 2, 3 }` 创建了一个 `std::initializer_list<int>` 类型的整数序列。作为替代 `vector` 传递给函数很有用。
```c++
int sum(const std::initializer_list<int>& list) {
  int total = 0;
  for (auto& e : list) {
    total += e;
  }

  return total;
}

auto list = {1, 2, 3};
sum(list); // == 6
sum({1, 2, 3}); // == 6
sum({}); // == 0
```

### 静态断言
静态断言是编译期断言
```c++
constexpr int x = 0;
constexpr int y = 1;
static_assert(x == y, "x != y");
```

### auto
`auto` 类型的变量由编译器根据它们的初始化来推导。
```c++
auto a = 3.14; // double
auto b = 1; // int
auto& c = b; // int&
auto d = { 0 }; // std::initializer_list<int>
auto&& e = 1; // int&&
auto&& f = b; // int&
auto g = new auto(123); // int*
const auto h = 1; // const int
auto i = 1, j = 2, k = 3; // int, int, int
auto l = 1, m = true, n = 1.61; // 错误 -- `l` 被推断为 int, 而 `m` 是 bool
auto o; // 错误 -- `o` 需要初始化
```

能极大地增加可读性, 尤其是复杂的类型:
```c++
std::vector<int> v = ...;
std::vector<int>::const_iterator cit = v.cbegin();
// vs.
auto cit = v.cbegin();
```

函数也能使用 `auto` 推断返回类型。在 C++11, 要么明确指定返回类型, 要么像这样使用 `decltype`:
```c++
template <typename X, typename Y>
auto add(X x, Y y) -> decltype(x + y) {
  return x + y;
}
add(1, 2); // == 3
add(1, 2.0); // == 3.0
add(1.5, 1.5); // == 3.0
```
上述示例中, 尾返回类型(trailing return type)就是表达式 `x + y` 的 _声明类型_(详见[`decltype`](#decltype))。例如, 如果 `x` 是整数 而 `y` 是双精度浮点数, `decltype(x + y)` 则是双精度浮点数。由此可知, 上面的函数将会根据表达式 `x + y` 生成的类型来推断类型。注意尾返回类型已经能获取它的形参, and `this` when appropriate。

### lambda 表达式
`lambda` 是一种能够捕捉作用域内变量的匿名函数对象。它的特点是: _捕获列表_, 可选的形参集合, 可选的尾返回类型, 以及函数体。捕获列表的示例:
* `[]` - 没有捕获
* `[=]` - 值捕获作用域内的局部对象(局部变量, 形参)
* `[&]` - 引用捕获作用域内的局部对象(l局部变量, 形参)
* `[this]` - 引用捕获 `this`
* `[a, &b]` - 值捕获 `a`, 引用捕获 `b`

```c++
int x = 1;

auto getX = [=] { return x; };
getX(); // == 1

auto addX = [=](int y) { return x + y; };
addX(1); // == 2

auto getXRef = [&]() -> int& { return x; };
getXRef(); // int& to `x`
```
值捕获不能在 lambda 内表达式内修改, 因为编译器生成的方法被标记为 `const`。关键字 `mutable` 允许修改值捕获的变量。该关键字必须放在形参列表后面(即使形参列表为空也要列出来)。
```c++
int x = 1;

auto f1 = [&x] { x = 2; }; // 正确: x 是引用, 并且改变了原始值

auto f2 = [x] { x = 2; }; // 错误: lambda 的值捕获只能是 const 的
// vs.
auto f3 = [x]() mutable { x = 2; }; // 正确: 此时 lambda 的值捕获可以随意操作了
```

### decltype
`decltype` 是返回表达式传入的 _声明类型_ 的操作符。表达式里的 const, voliate 和引用类型会保持不变。`decltype` 的例子:
```c++
int a = 1; // `a` 被声明为 `int` 类型
decltype(a) b = a; // `decltype(a)` 是 `int` 类型
const int& c = a; // `c` 被声明为 `const int&` 类型
decltype(c) d = a; // `decltype(c)` 是 `const int&` 类型
decltype(123) e = 123; // `decltype(123)` 是 `int` 类型
int&& f = 1; // `f` 被声明为 `int&&` 类型
decltype(f) g = 1; // `decltype(f) 是 `int&&` 类型
decltype((a)) h = g; // `decltype((a))` 是 int& 类型
```
```c++
template <typename X, typename Y>
auto add(X x, Y y) -> decltype(x + y) {
  return x + y;
}
add(1, 2.0); // `decltype(x + y)` => `decltype(3.0)` => `double`
```

还可以参照: [`decltype(auto) (C++14)`](CPP4.md#decltypeauto).

### unsing
使用 `using` 的类型别名在语意上与 `typedef` 相近, 可读性上更强，甚至还支持模板。
```c++
template <typename T>
using Vec = std::vector<T>;
Vec<int> v; // std::vector<int>

using String = std::string;
String s {"foo"};
```

### nullptr
C++11 引入了新的空指针, 用来替代 C语言的宏 `NULL`。`nullptr` 本身是 `std::nullptr_t` 类型的,  可以隐式转换到其他指针类型, 而 `NULL`不能转换到除了 `bool` 以外的整型。
```c++
void foo(int);
void foo(char*);
foo(NULL); // 错误 -- 有歧义
foo(nullptr); // 调用 foo(char*)
```

### 强类型枚举
类型安全的枚举可以解决许多 C 风格枚举的问题: 隐式转换, 不能指定基础类型, 作用域污染
```c++
// 指定基础类型为 `unsigned int`
enum class Color : unsigned int { Red = 0xff0000, Green = 0xff00, Blue = 0xff };
// `Alert` 里的 `Red`/`Green` 不会与 `Color` 里的冲突
enum class Alert : bool { Red, Green };
Color c = Color::Red;
```

### 属性
属性提供了一种通用语法, 如 `__attribute__(...)`, `__declspc`等。
```c++
// `noreturn` 属性表明函数 `f` 没有返回值
[[ noreturn ]] void f() {
  throw "error";
}
```

### constexpr
常量表达式是指编译器在编译时计算的表达式。常量表达式只能运行非复数计算。使用 `constexpr` 说明符表明变量, 函数等事常量表达式。
```c++
constexpr int square(int x) {
  return x * x;
}

int square2(int x) {
  return x * x;
}

int a = square(2);  // mov DWORD PTR [rbp-4], 4

int b = square2(2); // mov edi, 2
                    // call square2(int)
                    // mov DWORD PTR [rbp-8], eax
```

`constexpr` 的值是编译器可以在编译时计算的:
```c++
const int x = 123;
constexpr const int& y = x; // 错误 -- constexpr 变量 `y` 必须由常量表达式初始化
```

常量表达式与类:
```c++
struct Complex {
  constexpr Complex(double r, double i) : re{r}, im{i} { }
  constexpr double real() { return re; }
  constexpr double imag() { return im; }

private:
  double re;
  double im;
};

constexpr Complex I(0, 1);
```

### 委托构造函数
类的构造函数可以通过初始化列表调用其他构造函数。
```c++
struct Foo {
  int foo;
  Foo(int foo) : foo{foo} {}
  Foo() : Foo(0) {}
};

Foo foo;
foo.foo; // == 0
```

### 用户定义字面量
用户定义字面量允许拓展语言, 添加语法。创建一个字面量, 需要定义一个返回类型 `T` 的, 名为 `X` 的函数 `T operator "" X(...) { ... }`。注意, 函数的名称决定了字面量的名称。任何不以下划线开头的字面量名称都被保留且不予调用。根据字面量需要的类型, 用户定义字面量接收的形参需要遵守一些规则。

摄氏度转为华氏度:
```c++
// `unsigned long long` 类型的形参需要整数字面量
long long operator "" _celsius(unsigned long long tempCelsius) {
  return std::llround(tempCelsius * 1.8 + 32);
}
24_celsius; // == 75
```

字符串转为整数:
```c++
// `const char*` 和 `std::size_t` 是需要的形参
int operator "" _int(const char* str, std::size_t) {
  return std::stoi(str);
}

"123"_int; // == 123, with type `int`
```

### override
明确指出派生类的虚函数是重写的
```c++
struct A {
  virtual void foo();
  void bar();
};

struct B : A {
  void foo() override; // correct -- B::foo overrides A::foo
  void bar() override; // error -- A::bar is not virtual
  void baz() override; // error -- B::baz does not override A::baz
};
```

### final
表明虚函数不能被派生类覆写, 或者类不能被继承。
```c++
struct A {
  virtual void foo();
};

struct B : A {
  virtual void foo() final;
};

struct C : B {
  virtual void foo(); // 错误 -- 函数声明 'foo' 覆写了 'final' 函数
};
```

Class cannot be inherited from.
```c++
struct A final {};
struct B : A {}; // 错误 -- 基类 'A' 被标记为 'final'
```

### default
一种更优雅, 更高效的提供默认函数实现的方式, 比如默认构造函数。
```c++
struct A {
  A() = default;
  A(int x) : x{x} {}
  int x {1};
};
A a; // a.x == 1
A a2 {123}; // a.x == 123
```

继承:
```c++
struct B {
  B() : x{1} {}
  int x;
};

struct C : B {
  // 默认调用基类的构造函数 B::B()
  C() = default;
};

C c; // c.x == 1
```

### delete
一种更优雅, 更高效的提供禁用函数的方式。阻止对象复制很有用。
```c++
class A {
  int x;

public:
  A(int x) : x{x} {};
  A(const A&) = delete;
  A& operator=(const A&) = delete;
};

A x {123};
A y = x; // 错误 -- 调用了禁用的拷贝构造函数
y = x; // 错误 -- 拷贝赋值函数被禁用
```

### 基于范围的 for 循环
迭代容器元素的语法糖。
```c++
std::array<int, 5> a {1, 2, 3, 4, 5};
for (int& x : a) x *= 2;
// a == { 2, 4, 6, 8, 10 }
```

注意使用 `int` 和 `int&` 之间的区别:
```c++
std::array<int, 5> a {1, 2, 3, 4, 5};
for (int x : a) x *= 2;
// a == { 1, 2, 3, 4, 5 }
```

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
C++11 引入了新的智能指针: `std::unique_ptr`, `std::shared_ptr`, `std::weak_ptr`。`std::auto_ptr` 已经被弃用(deprecated), 并且在 C++17 中移除。

`std::unique_ptr` 是一种不能复制, 只能转移的智能指针。它持有并管理堆上分配的内存。**注意: 最好使用 `std::make_X` 函数而不是构造函数来创建智能指针。参考 [std::make_unique]() 和 [std::make_sheared]()。**
```c++
std::unique_ptr<Foo> p1 { new Foo{} };  // `p1` 持有 `Foo` 对象的所有权
if (p1) {
  p1->bar();
}

{
  std::unique_ptr<Foo> p2 {std::move(p1)};  // 现在 `p2` 持有 `Foo` 对象的所有权
  f(*p2);

  p1 = std::move(p2);  // 所有权回到 `p1` -- `p2` 被销毁
}

if (p1) {
  p1->bar();
}
// 当 `p1` 离开作用域时, `Foo` 对象被销毁
```

`std::shared_ptr` 是一种可以共享管理资源的智能指针。`std::shared_ptr` 有 _控制块_(control block), 包括被管理的对象和引用计数等。控制块的访问是线程安全的, 然而被管理的对象本身的操作 *不是* 线程安全的。
```c++
void foo(std::shared_ptr<T> t) {
  // Do something with `t`...
}

void bar(std::shared_ptr<T> t) {
  // Do something with `t`...
}

void baz(std::shared_ptr<T> t) {
  // Do something with `t`...
}

std::shared_ptr<T> p1 {new T{}};
// 以下函数可能在其他线程
foo(p1);
bar(p1);
baz(p1);
```

### std::make_shared
推荐使用 `std::make_shared` 来创建 `std::shared_ptr`, 原因如下:
* 避免使用 `new` 操作符
* 当指定指针持有的类型时避免代码重复
* 异常安全。如果像这样调用 `foo` 函数:
```c++
foo(std::shared_ptr<T>{new T{}}, function_that_throws(), std::shared_ptr<T>{new T{}});
```
编译器按顺序调用 `new T{}`, 然后调用 `function_that_throws()`。因为我们在最先构造 `T` 时已经在堆上分配了内存, 会导致内存泄漏。使用 `std::make_shared` 则是异常安全的:
```c++
foo(std::make_shared<T>(), function_that_throws(), std::make_shared<T>());
```
* 避免两次内存分配。当调用 `std::shared_ptr{ new T{} }` 时, 先给 `T` 分配内存, 然后在智能指针里还要给控制块分配内存。

参考 [智能指针](#智能指针) 部分获取更多关于 `std::unique_ptr` 和 `std::shared_ptr` 的信息。

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