## 1.堆 栈 RAII
* 堆：在内存管理的语境下，指的是动态分配内存的区域。
* 栈：函数调用过程中产生的本地变量和调用数据的区域。
* RAII：依托栈和析构函数，来对所有的资源——包括堆内存在内——进行管理。

## 2.右值和移动 
一个 lvalue 是通常可以放在等号左边的表达式，左值

一个 rvalue 是通常只能放在等号右边的表达式，右值

一个 glvalue 是 generalized lvalue，广义左值

一个 xvalue 是 expiring value，将亡值

一个 prvalue 是 pure rvalue，纯右值

> 我们可以把 std::move(ptr1) 看作是一个有名字的右值。为了跟无名的纯右值 prvalue 相区别，C++ 里目前就把这种表达式叫做 xvalue。跟左值 lvalue 不同，xvalue 仍然是不能取地址的——这点上，xvalue 和 prvalue 相同。所以，xvalue 和 prvalue 都被归为右值 rvalue。

### 生命周期和表达式类型

一个临时对象会在包含这个临时对象的完整表达式估值完成后、按生成顺序的逆序被销毁，除非有生命周期延长发生。

` process_shape(circle(), triangle())`

如果一个 prvalue 被绑定到一个引用上，它的生命周期则会延长到跟这个引用变量一样长。

`result&& r = process_shape(circle(), triangle());`

需要万分注意的是，这条生命期延长规则只对 prvalue 有效，而对 xvalue 无效。如果由于某种原因，prvalue 在绑定到引用以前已经变成了 xvalue，那生命期就不会延长。不注意这点的话，代码就可能会产生隐秘的 bug。比如，我们如果这样改一下代码，结果就不对了：

`result&& r = std::move(process_shape(circle(), triangle()))`

仍然有一个有效的变量 r，但它指向的对象已经不存在了，对 r 的解引用是一个未定义行为。

>你可以把一个没有虚析构函数的子类对象绑定到基类的引用变量上，这个子类对象的析构仍然是完全正常的——这是因为这条规则只是延后了临时对象的析构而已，不是利用引用计数等复杂的方法，因而只要引用绑定成功，其类型并没有什么影响。

### 移动
一句话总结，移动语义使得在 C++ 里返回大对象（如容器）的函数和运算符成为现实，因而可以提高代码的简洁性和可读性，提高程序员的生产率。

### 引用坍缩和完美转发
对于 template \<typename T> foo(T&&) 这样的代码，如果传递过去的参数是左值，T 的推导结果是左值引用；如果传递过去的参数是右值，T 的推导结果是参数的类型本身。

如果 T 是左值引用，那 T&& 的结果仍然是左值引用——即 type& && 坍缩成了 type&。

如果 T 是一个实际类型，那 T&& 的结果自然就是一个右值引用。

### 请记住：
* `std::move`执行到右值的无条件的转换，但就自身而言，它不移动任何东西。

* `std::forward`只有当它的参数被绑定到一个右值时，才将参数转换为右值。

* `std::move`和`std::forward`在运行期什么也不做。

## 3.易用性改进
### auto
auto 实际使用的规则类似于函数模板参数的推导规则

auto a = expr; 意味着用 expr 去匹配一个假想的 template \<typename T> f(T) 函数模板，结果为值类型。

const auto& a = expr; 意味着用 expr 去匹配一个假想的 template \<typename T> f(const T&) 函数模板，结果为常左值引用类型。

auto&& a = expr; 意味着用 expr 去匹配一个假想的 template \<typename T> f(T&&) 函数模板，根据[\[第 3 讲\]]() 中我们讨论过的转发引用和引用坍缩规则，结果是一个跟 expr 值类别相同的引用类型。

### decltype
decltype(变量名) 可以获得变量的精确类型。

decltype(表达式) （表达式不是变量名，但包括 decltype((变量名)) 的情况）可以获得表达式的引用类型；除非表达式的结果是个纯右值（prvalue），此时结果仍然是值类型。

如果我们有 int a;，那么：

decltype(a) 会获得 int（因为 a 是 int）。

decltype((a)) 会获得 int&（因为 a 是 lvalue）。

decltype(a + a) 会获得 int（因为 a + a 是 prvalue）。

### decltype(auto)
但这儿有个限制，你需要在写下 auto 时就决定你写下的是个引用类型还是值类型。根据类型推导规则，auto 是值类型，auto& 是左值引用类型，auto&& 是转发引用（可以是左值引用，也可以是右值引用）。使用 auto 不能通用地根据表达式类型来决定返回值的类型。

`decltype(expr) a = expr;`

这种写法明显不能让人满意，特别是表达式很长的情况（而且，任何代码重复都是潜在的问题）。为此，C++14 引入了 decltype(auto) 语法。对于上面的情况，我们只需要像下面这样写就行了。

`decltype(auto) a = expr;`

## 4.接口返回对象
《C++ 核心指南》的 F.20 这一条款是这么说的
>在函数输出数值时，尽量使用返回值而非输出参数

一个用来返回的对象，通常应当是可移动构造 / 赋值的，一般也同时是可拷贝构造 / 赋值的。如果这样一个对象同时又可以默认构造，我们就称其为一个半正则（semiregular）的对象。如果可能的话，我们应当尽量让我们的类满足半正则这个要求。
```
matrix operator*(const matrix& lhs,
                 const matrix& rhs)
{
  if (lhs.cols() != rhs.rows()) {
    throw runtime_error(
      "sizes mismatch");
  }
  matrix result(lhs.rows(),
                rhs.cols());
  // 具体计算过程
  return result;
}
```

返回非引用类型的表达式结果是个纯右值（prvalue）。在执行 auto r = … 的时候，编译器会认为我们实际是在构造 matrix r(…)，而“…”部分是一个纯右值。因此编译器会首先试图匹配 matrix(matrix&&)，在没有时则试图匹配 matrix(const matrix&)；也就是说，有移动支持时使用移动，没有移动支持时则拷贝。

## 5.返回值优化（拷贝消除）

我们再来看一个能显示生命期过程的对象的例子：

```
\#include \<iostream>

using namespace std;

// Can copy and move

class A {

public:

A() { cout << "Create A\n"; }

\~A() { cout << "Destroy A\n"; }

A(const A&) { cout << "Copy A\n"; }

A(A&&) { cout << "Move A\n"; }

};

A getA\_unnamed()

{

return A();

}

int main()

{

auto a = getA\_unnamed();

}
```

如果你认为执行结果里应当有一行“Copy A”或“Move A”的话，你就忽视了返回值优化的威力了。即使完全关闭优化，三种主流编译器（GCC、Clang 和 MSVC）都只输出两行：

```
Create A

Destroy A

我们把代码稍稍改一下：

A getA\_named()

{

A a;

return a;

}

int main()

{

auto a = getA\_named();

}
```

这回结果有了一点点小变化。虽然 GCC 和 Clang 的结果完全不变，但 MSVC 在非优化编译的情况下产生了不同的输出（优化编译——使用命令行参数 /O1、/O2 或 /Ox——则不变）：
```
Create A

Move A

Destroy A

Destroy A
```

## 6.编译期多态：泛型编程和模板入门
### 实例化模板

不管是类模板还是函数模板，编译器在看到其定义时只能做最基本的语法检查，真正的类型检查要在实例化（instantiation）的时候才能做。一般而言，这也是编译器会报错的时候。

实例化失败的话，编译当然就出错退出了。如果成功的话，模板的实例就产生了。在整个的编译过程中，可能产生多个这样的（相同）实例，但最后链接时，会只剩下一个实例。
模板还可以显式实例化和外部实例化。如果我们在调用 my\_gcd 之前进行显式实例化——即，使用 template 关键字并给出完整的类型来声明函数：

类似的，当我们在使用 vector<int> 这样的表达式时，我们就在隐式地实例化 vector<int>。我们同样也可以选择用 template class vector<int>; 来显式实例化，或使用 extern template class vector<int>;

显式实例化和外部实例化通常在大型项目中可以用来集中模板的实例化，从而加速编译过程——不需要在每个用到模板的地方都进行实例化了——但这种方式有额外的管理开销，如果实例化了不必要实例化的模板的话，反而会导致可执行文件变大。因而，显式实例化和外部实例化应当谨慎使用。

### 特化模板
我们需要使用的模板参数类型，不能完全满足模板的要求，应该怎么办？

我们实际上有好几个选择：

添加代码，让那个类型支持所需要的操作（对成员函数无效）。

对于函数模板，可以直接针对那个类型进行重载。

对于类模板和函数模板，可以针对那个类型进行特化。

对于 cln::cl_I 不支持 % 运算符这种情况，恰好上面的三种方法我们都可以用。
1. 添加 operator% 的实现：
```
cln::cl_I
operator%(const cln::cl_I& lhs,
          const cln::cl_I& rhs)
{
  return mod(lhs, rhs);
}
```

2. 重载
为通用起见，我不直接使用 cl_I 的 mod 函数，而用 my_mod 把 my_gcd 改造如下：
```
cln::cl_I
my_mod(const cln::cl_I& lhs,
       const cln::cl_I& rhs)
{
  return mod(lhs, rhs);
}
```

3. 针对 cl_I 进行特化：
同二类似，但我们提供的不是一个重载，而是特化（specialization）：
```
template <>
cln::cl_I my_mod<cln::cl_I>(
  const cln::cl_I& lhs,
  const cln::cl_I& rhs)
{
  return mod(lhs, rhs);
}
```

## 7.编译器能做什么

### 编译器计算

首先，我们给出一个已经被证明的结论：C++ 模板是图灵完全的 [1]。这句话的意思是，使用 C++ 模板，你可以在编译期间模拟一个完整的图灵机，也就是说，可以完成任何的计算任务。

### 编译期类型推导
为了方便地在值和类型之间转换，标准库定义了一些经常需要用到的工具类。上面描述的 integral_constant 就是其中一个（我的定义有所简化）。为了方便使用，针对布尔值有两个额外的类型定义：
```
typedef std::integral_constant<
  bool, true> true_type;
typedef std::integral_constant<
  bool, false> false_type;
```

这两个标准类型 true_type 和 false_type 经常可以在函数重载中见到。有一个工具函数常常会写成下面这个样子：
```
template <typename T>
class SomeContainer {
public:
  …
  static void destroy(T* ptr)
  {
    _destroy(ptr,
      is_trivially_destructible<
        T>());
  }
private:
  static void _destroy(T* ptr,
                       true_type)
  {}
  static void _destroy(T* ptr,
                       false_type)
  {
    ptr->~T();
  }
};
```
像 is_trivially_destructible 这样的 trait 类有很多，可以用来在模板里决定所需的特殊行为：
is_array
is_enum
is_function
is_pointer
is_reference
is_const
has_virtual_destructor
…

## 8.替换失败非错 SFINAE
我们之前已经讨论了不少模板特化。我们今天来着重看一个函数模板的情况。当一个函数名称和某个函数模板名称匹配时，重载决议过程大致如下：

根据名称找出所有适用的函数和函数模板

对于适用的函数模板，要根据实际情况对模板形参进行替换；替换过程中如果发生错误，这个模板会被丢弃

在上面两步生成的可行函数集合中，编译器会寻找一个最佳匹配，产生对该函数的调用

如果没有找到最佳匹配，或者找到多个匹配程度相当的函数，则编译器需要报错

```
\#include \<stdio.h>

struct Test {

typedef int foo;

};

template \<typename T>

void f(typename T::foo)

{

puts("1");

}

template \<typename T>

void f(T)

{

puts("2");

}

int main()

{

f\<Test>(10);

f\<int>(10);

}

输出为：

1

2
```

我们来分析一下。首先看 f\<Test>(10); 的情况：

我们有两个模板符合名字 f

替换结果为 f(Test::foo) 和 f(Test)

使用参数 10 去匹配，只有前者参数可以匹配，因而第一个模板被选择

再看一下 f\<int>(10) 的情况：

还是两个模板符合名字 f

替换结果为 f(int::foo) 和 f(int)；显然前者不是个合法的类型，被抛弃

使用参数 10 去匹配 f(int)，没有问题，那就使用这个模板实例了

在这儿，体现的是 SFINAE 设计的最初用法：如果模板实例化中发生了失败，没有理由编译就此出错终止，因为还是可能有其他可用的函数重载的。

这儿的失败仅指函数模板的原型声明，即参数和返回值。函数体内的失败不考虑在内。如果重载决议选择了某个函数模板，而函数体在实例化的过程中出错，那我们仍然会得到一个编译错误。

### 编译期成员检测

enable_if  
C++11 开始，标准库里有了一个叫 enable_if 的模板（定义在 <type_traits> 里），可以用它来选择性地启用某个函数的重载。

```
template <typename C, typename T>
enable_if_t<has_reserve<C>::value,
            void>
append(C& container, T* ptr,
       size_t size)
{
  container.reserve(
    container.size() + size);
  for (size_t i = 0; i < size;
       ++i) {
    container.push_back(ptr[i]);
  }
}
template <typename C, typename T>
enable_if_t<!has_reserve<C>::value,
            void>
append(C& container, T* ptr,
       size_t size)
{
  for (size_t i = 0; i < size;
       ++i) {
    container.push_back(ptr[i]);
  }
}
```

对于某个 type trait，添加 _t 的后缀等价于其 type 成员类型。因而，我们可以用 enable_if_t 来取到结果的类型。enable_if_t<has_reserve<C>::value, void> 的意思可以理解成：如果类型 C 有 reserve 成员的话，那我们启用下面的成员函数，它的返回类型为 void。

### decltype 返回值
```
template <typename C, typename T>
auto append(C& container, T* ptr,
            size_t size)
  -> decltype(
    declval<C&>().reserve(1U),
    void())
{
  container.reserve(
    container.size() + size);
  for (size_t i = 0; i < size;
       ++i) {
    container.push_back(ptr[i]);
  }
}
```

这是我们第一次用到 declval [4]，需要简单介绍一下。这个模板用来声明一个某个类型的参数，但这个参数只是用来参加模板的匹配，不允许实际使用。使用这个模板，我们可以在某类型没有默认构造函数的情况下，假想出一个该类的对象来进行类型推导。declval<C&>().reserve(1U) 用来测试 C& 类型的对象是不是可以拿 1U 作为参数来调用 reserve 成员函数。此外，我们需要记得，C++ 里的逗号表达式的意思是按顺序逐个估值，并返回最后一项。所以，上面这个函数的返回值类型是 void。

## 9.constexpr

一个 constexpr 变量是一个编译时完全确定的常数。一个 constexpr 函数至少对于某一组实参可以在编译期间产生一个编译期常数。

注意一个 constexpr 函数不保证在所有情况下都会产生一个编译期常数（因而也是可以作为普通函数来使用的）。编译器也没法通用地检查这点。编译器唯一强制的是：
constexpr 变量必须立即初始化
初始化只能使用字面量或常量表达式，后者不允许调用任何非 constexpr 函数

```
#include <array>
constexpr int sqr(int n)
{
  return n * n;
}
int main()
{
  constexpr int n = sqr(3);
  std::array<int, n> a;
  int b[n];
}
```

### constexpr 和编译期计算
```
constexpr int factorial(int n)
{
  if (n == 0) {
    return 1;
  } else {
    return n * factorial(n - 1);
  }
}

int main()
{
  constexpr int n = factorial(10);
  printf("%d\n", n);
}
```

### constexpr 和 const

注意 const 在类型声明的不同位置会产生不同的结果。对于常见的 const char* 这样的类型声明，意义和 char const* 相同，是指向常字符的指针，指针指向的内容不可更改；但和 char * const 不同，那代表指向字符的常指针，指针本身不可更改。本质上，const 用来表示一个运行时常量。

### 内联变量
C++17 引入了内联（inline）变量的概念，允许在头文件中定义内联变量，然后像内联函数一样，只要所有的定义都相同，那变量的定义出现多次也没有关系。对于类的静态数据成员，const 缺省是不内联的，而 constexpr 缺省就是内联的。这种区别在你用 & 去取一个 const int 值的地址、或将其传到一个形参类型为 const int& 的函数去的时候（这在 C++ 文档里的行话叫 ODR-use），就会体现出来。

```

#include <iostream>
#include <vector>
struct magic {
  static const int number = 42;
};
int main()
{
  std::vector<int> v;
  // 调用 push_back(const T&)
  v.push_back(magic::number);
  std::cout << v[0] << std::endl;
}
```
程序在链接时就会报错了，说找不到 magic::number

修正这个问题的简单方法是把 magic 里的 static const 改成 static constexpr 或 static inline const。前者可行的原因是，类的静态 constexpr 成员变量默认就是内联的。const 常量和类外面的 constexpr 变量不默认内联，需要手工加 inline 关键字才会变成内联。

### constexpr 变量模板
```
template <class T>
inline constexpr bool
  is_trivially_destructible_v =
    is_trivially_destructible<
      T>::value;
```

了解了变量也可以是模板之后，上面这个代码就很容易看懂了吧？这只是一个小小的语法糖，允许我们把 is_trivially_destructible<T>::value 写成 is_trivially_destructible_v<T>。

### if constexpr
```
template <typename C, typename T>
void append(C& container, T* ptr,
            size_t size)
{
  if (has_reserve<C>::value) {
    container.reserve(
      container.size() + size);
  }
  for (size_t i = 0; i < size;
       ++i) {
    container.push_back(ptr[i]);
  }
}
```

我们只要在 if 后面加上 constexpr，代码就能工作了 [2]。当然，它要求括号里的条件是个编译期常量。满足这个条件后，标签分发、enable_if 那些技巧就不那么有用了。显然，使用 if constexpr 能比使用其他那些方式，写出更可读的代码

## 10.函数对象和lambda

### C++98 的函数对象
```
struct adder {
  adder(int n) : n_(n) {}
  int operator()(int x) const
  {
    return x + n_;
  }
private:
  int n_;
};
```

### lambda
```
transform(v.begin(), v.end(),
          v.begin(),
          [](int x) {
            return x + 2;
          });


auto seven = adder(2)(5);
```


一个 lambda 表达式除了没有名字之外，还有一个特点是你可以立即进行求值。这就使得我们可以把一段独立的代码封装起来，达到更干净、表意的效果。

`[](int x) { return x * x; }(3)`

只要能满足 constexpr 函数的条件，一个 lambda 表达式默认就是 constexpr 函数。

另外一种用途是解决多重初始化路径的问题。假设你有这样的代码：
```

Obj obj;
switch (init_mode) {
case init_mode1:
  obj = Obj(…);
  break;
case init_mode2;
  obj = Obj(…);
  break;
…
}
```

实际上是调用了默认构造函数、带参数的构造函数和（移动）赋值函数：既可能有性能损失，也对 Obj 提出了有默认构造函数的额外要求。
```
auto obj = [init_mode]() {
  switch (init_mode) {
  case init_mode1:
    return Obj(…);
    break;
  case init_mode2:
    return Obj(…);
    break;
  …
  }
}();
```

### 泛型 lambda 表达式
```
template <typename T1,
          typename T2>
auto sum(T1 x, T2 y)
{
  return x + y;
}

auto sum = [](auto x, auto y)
{
  return x + y;
}
```
两者等价

### function
```
map<string, function<int(int, int)>>
  op_dict{
    {"+",
     [](int x, int y) {
       return x + y;
     }},
    {"-",
     [](int x, int y) {
       return x - y;
     }},
    {"*",
     [](int x, int y) {
       return x * y;
     }},
    {"/",
     [](int x, int y) {
       return x / y;
     }},
  };
```

## 12.可变模板
可变模板 是 C++11 引入的一项新功能，使我们可以在模板参数里表达不定个数和类型的参数。从实际的角度，它有两个明显的用途： 
1. 用于在通用工具模板中转发参数到另外一个函数
2. 用于在递归的模板中表达通用的情况（另外会有至少一个模板特化来表达边界情况）

### 转发用法
```
template <typename T,
          typename... Args>
inline unique_ptr<T>
make_unique(Args&&... args)
{
  return unique_ptr<T>(
    new T(forward<Args>(args)...));
}


// make_unique<vector<int>>(100, 1)

模板实例化之后，会得到相当于下面的代码：
template <>
inline unique_ptr<vector<int>>
make_unique(int&& arg1, int&& arg2)
{
  return unique_ptr<vector<int>>(
    new vector<int>(
      forward<int>(arg1),
      forward<int>(arg2)));
}

```

### 递归用法

```
template <typename T>
constexpr auto sum(T x)
{
  return x;
}
template <typename T1, typename T2,
          typename... Targ>
constexpr auto sum(T1 x, T2 y,
                   Targ... args)
{
  return sum(x + y, args...);
}
```

在上面的定义里，如果 sum 得到的参数只有一个，会走到上面那个重载。如果有两个或更多参数，编译器就会选择下面那个重载，执行一次加法，随后你的参数数量就少了一个，因而递归总会终止到上面那个重载，结束计算。

### 数值计算
我们希望快速地计算一串二进制数中 1 比特的数量
```
constexpr int
count_bits(unsigned char value)
{
  if (value == 0) {
    return 0;
  } else {
    return (value & 1) +
           count_bits(value >> 1);
  }
}


template <size_t... V>
constexpr bit_count_t<V...>
get_bit_count(index_sequence<V...>)
{
  return bit_count_t<V...>();
}
auto bit_count = get_bit_count(
  make_index_sequence<256>());

```

## 13.thread 和 future

### 互斥量
互斥量的基本语义是，一个互斥量只能被一个线程锁定，用来保护某个代码块在同一时间只能被一个线程执行。

主要提供的方法是：

lock：锁定，锁已经被其他线程获得时则阻塞执行

try\_lock：尝试锁定，获得锁返回 true，在锁被其他线程获得时返回 false

unlock：解除锁定（只允许在已获得锁时调用）

### future

```
#include <chrono>
#include <future>
#include <iostream>
#include <thread>
using namespace std;
int work()
{
  // 假装我们计算了很久
  this_thread::sleep_for(2s);
  return 42;
}
int main()
{
  auto fut = async(launch::async, work);
  // 干一些其他事
  cout << "I am waiting now\n";
  cout << "Answer: " << fut.get()
       << '\n';
}
```
work 函数现在不需要考虑条件变量之类的实现细节了，专心干好自己的计算活、老老实实返回结果就可以了。  
调用 async 可以获得一个未来量，launch::async 是运行策略，告诉函数模板 async 应当在新线程里异步调用目标函数。在一些老版本的 GCC 里，不指定运行策略，默认不会起新线程。  
async 函数模板可以根据参数来推导出返回类型，在我们的例子里，返回类型是 future<int>。  
在未来量上调用 get 成员函数可以获得其结果。这个结果可以是返回值，也可以是异常，即，如果 work 抛出了异常，那 main 里在执行 fut.get() 时也会得到同样的异常，需要有相应的异常处理代码程序才能正常工作。

### promise
```
#include <chrono>
#include <future>
#include <iostream>
#include <thread>
#include <utility>
using namespace std;
class scoped_thread {
  … // 定义同上，略
};
void work(promise<int> prom)
{
  // 假装我们计算了很久
  this_thread::sleep_for(2s);
  prom.set_value(42);
}
int main()
{
  promise<int> prom;
  auto fut = prom.get_future();
  scoped_thread th{work,
                   move(prom)};
  // 干一些其他事
  cout << "I am waiting now\n";
  cout << "Answer: " << fut.get()
       << '\n';
}
```
就这个例子而言，使用 promise 没有 async 方便，但可以看到，这是一种非常灵活的方式，你不需要在一个函数结束的时候才去设置 future 的值。仍然需要注意的是，一组 promise 和 future 只能使用一次，既不能重复设，也不能重复取。