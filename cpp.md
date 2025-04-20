## 1.堆 栈 RAII
* 堆：在内存管理的语境下，指的是动态分配内存的区域。
* 栈：函数调用过程中产生的本地变量和调用数据的区域。
* RAII：依托栈和析构函数，来对所有的资源——包括堆内存在内——进行管理。

### 不用RAII什么情况容易发生内存泄漏的情况
* 当代码中抛出*异常*时，如果没有恰当的异常处理机制，可能会跳过资源释放的代码，进而造成内存泄漏。
* 提前return没有正确释放资源，就可能造成内存泄漏。

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
### const 左值引用

绑定对象范围广
- 可绑定普通左值对象，即有名字、有固定内存地址的对象。
- 能绑定 `const` 左值对象，保证不会修改该对象。
- 还能绑定右值对象，这是非 `const` 左值引用所不具备的能力。

延长右值生命周期
- 当 `const` 左值引用绑定到右值时，右值的生命周期会延长至与引用的生命周期一致，避免因右值提前销毁而导致访问错误。

优化函数传参
- 作为函数参数使用时，可避免对象的拷贝，提升程序性能。
- 既能接收左值参数，也能接收右值参数，增强函数的通用性。

### auto
auto 实际使用的规则类似于函数模板参数的推导规则

auto a = expr; 意味着用 expr 去匹配一个假想的 template \<typename T> f(T) 函数模板，结果为值类型。

const auto& a = expr; 意味着用 expr 去匹配一个假想的 template \<typename T> f(const T&) 函数模板，结果为常左值引用类型。

auto&& a = expr; 意味着用 expr 去匹配一个假想的 template \<typename T> f(T&&) 函数模板，根据[\[第 3 讲\]]() 中我们讨论过的转发引用和引用坍缩规则，结果是一个跟 expr 值类别相同的引用类型。

### decltype
decltype(变量名) 可以获得变量的精确类型。
> 如果实参是没有括号的标识表达式或没有括号的类成员访问表达式，那么 decltype 产生该表达式指名的实体的类型

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

cppref:
```
const int& getRef(const int* p) { return *p; }
static_assert(std::is_same_v<decltype(getRef), const int&(const int*)>);
auto getRefFwdBad(const int* p) { return getRef(p); }
static_assert(std::is_same_v<decltype(getRefFwdBad), int(const int*)>,
    "仅返回 auto 并不能完美转发。");
decltype(auto) getRefFwdGood(const int* p) { return getRef(p); }
static_assert(std::is_same_v<decltype(getRefFwdGood), const int&(const int*)>,
    "返回 decltype(auto) 完美转发返回类型。");
```

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
#include <stdio.h>

struct Test {

typedef int foo;

};

template <typename T>

void f(typename T::foo)

{

puts("1");

}

template <typename T>

void f(T)

{

puts("2");

}

int main()

{

f<Test>(10);

f<int>(10);

}

输出为：

1

2
```

我们来分析一下。首先看 f<Test>(10); 的情况：

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

## 14.四种cast

### 什么情况下用const_cast是ub什么时候不是ub
const_cast 可以用于去除一个变量的 const 限定，或者把一个指针转换成另外一个类型。

当 const_cast 被用于移除一个原本就是 const 对象的 const 限定符，并尝试修改该对象的值时，就会产生未定义行为。这是因为 const 对象在设计上就是不允许被修改的，绕过这个限制会破坏程序的安全性和可预测性

如果一个对象本身不是 const 的，只是通过 const 引用或 const 指针来访问，那么使用 const_cast 移除 const 限定符并修改对象的值是合法的，不会产生未定义行为。

### 1. static_cast
static_cast 是最常用的类型转换运算符，它可以在编译时进行各种类型转换，包括基本数据类型转换、类层次结构中的上行和下行转换等。
 基本数据类型转换 类层次结构中的转换
注意 ： static_cast 进行下行转换时没有运行时检查，可能会导致未定义行为。

### 2. reinterpret_cast
reinterpret_cast 用于进行底层的、危险的类型转换，它可以将一个指针转换为任意其他类型的指针，也可以将指针转换为整数类型，反之亦然。

注意 ： reinterpret_cast 非常危险，因为它绕过了类型系统的检查，可能会导致未定义行为，应谨慎使用。

### reinterpret_cast,用int强转double去用可能有什么问题吗

int 和 double 在内存中的存储格式大不相同。 int 一般以补码形式存储整数，而 double 依据 IEEE 754 标准存储浮点数，该标准把浮点数拆分为符号位、指数位和尾数位。

当运用 reinterpret_cast 时，它只是简单地把 int 类型的内存比特模式原封不动地解释成 double 类型，并不会按照浮点数的规则对整数进行转换。

> 正确的转换方式
若要把 int 转换为 double ，应该使用隐式转换或者 static_cast ，这样编译器会按照正确的规则进行数值转换。

### 3. dynamic_cast
dynamic_cast 主要用于类层次结构中的安全下行转换，它会在运行时进行类型检查，确保转换的安全性。如果转换失败，对于指针类型会返回 nullptr ，对于引用类型会抛出 std::bad_cast 异常。

注意 ： dynamic_cast 只能用于含有虚函数的类层次结构，因为它依赖于运行时类型信息（RTTI）。

### 4. const_cast
const_cast 主要用于移除变量的 const 或 volatile 限定符。它不能用于其他类型的转换，只能改变对象的常量性或易变性。

注意 ：如果原始对象是 const 的，使用 const_cast 移除 const 限定符并修改对象的值会导致未定义行为。

### 总结 类型转换运算符 用途 安全性 static_cast 
| 类型转换运算符 | 用途 | 安全性 |
| ---- | ---- | ---- |
| `static_cast` | 基本数据类型转换、类层次结构中的上行和下行转换 | 部分转换不安全 |
| `reinterpret_cast` | 底层的、危险的类型转换 | 不安全 |
| `dynamic_cast` | 类层次结构中的安全下行转换 | 安全 |
| `const_cast` | 移除变量的 `const` 或 `volatile` 限定符 | 不当使用会导致未定义行为 |

### dynamic_cast怎么知道转换到子类的

dynamic_cast 依赖于运行时类型信息（Run-Time Type Information，RTTI）。当类中包含至少一个虚函数时，编译器会为该类生成 RTTI 信息，这些信息会记录类的层次结构和类型关系。在运行时， dynamic_cast 会根据这些信息来判断转换是否合法。

> 运行时类型信息（RTTI）
dynamic_cast 依赖于运行时类型信息（Run-Time Type Information，RTTI）。RTTI 是 C++ 提供的一种机制，用于在运行时确定对象的类型。当类中至少有一个虚函数时，编译器会为该类及其派生类生成 RTTI 信息。
 虚函数表（VTable）
在 C++ 里，含有虚函数的类会有一个虚函数表（VTable）。每个该类的对象都会包含一个指向虚函数表的指针（VPTR）。虚函数表是一个存储类的虚函数地址的数组，同时也会存储该类的 RTTI 信息。

下面简单示意含有虚函数的类的对象布局：

```plaintext
+-------------------+
|       VPTR        |  -> 指向虚函数表
+-------------------+
|  其他成员变量     |
+-------------------+
 ``` 

#### 虚函数表中的 RTTI 信息
虚函数表中除了存储虚函数的地址，还会有一个指向该类 RTTI 信息的指针。RTTI 信息通常是一个 type_info 对象，它包含了类的名称、类的层次结构等信息。

#### dynamic_cast 转换过程 指针转换
当使用 dynamic_cast 进行指针转换时，例如从基类指针转换到派生类指针，其过程如下：  
1. 获取 RTTI 信息 ：通过基类指针找到对象的 VPTR，再通过 VPTR 找到虚函数表，进而获取到基类的 RTTI 信息。
2. 类型检查 ：根据 RTTI 信息检查基类指针实际指向的对象是否是目标派生类的对象或者是其派生类的对象。这需要遍历类的继承层次结构，查看是否存在合法的转换路径。
3. 返回结果 ：如果类型检查通过，计算派生类对象在内存中的起始地址，并返回指向该派生类对象的指针；如果检查不通过，返回 nullptr 。

## 15.异常
### 如果没有异常，例如c的代码
```
errcode = matrix_multiply(c, a, b);
  if (errcode != MATRIX_SUCCESS) {
    goto error_exit;
  }
```
可以看到，我们有大量需要判断错误的代码，零散分布在代码各处。

C++ 的构造函数是不能返回错误码的，所以你根本不能用构造函数来做可能出错的事情。你不得不定义一个只能清零的构造函数，再使用一个 init 函数来做真正的构造操作。

### 异常安全的代码，可以没有任何 try 和 catch。
如果你不确定什么是“异常安全”，我们先来温习一下概念：异常安全是指当异常发生时，既不会发生资源泄漏，系统也不会处于一个不一致的状态。

我们看看可能会出现错误 / 异常的地方：

首先是内存分配。如果 new 出错，按照 C++ 的规则，一般会得到异常 bad_alloc，对象的构造也就失败了。这种情况下，在 catch 捕捉到这个异常之前，所有的栈上对象会全部被析构，资源全部被自动清理。

如果是矩阵的长宽不合适不能做乘法呢？我们同样会得到一个异常，这样，在使用乘法的地方，对象 c 根本不会被构造出来。

如果在乘法函数里内存分配失败呢？一样，result 对象根本没有构造出来，也就没有 c 对象了。还是一切正常。

如果 a、b 是本地变量，然后乘法失败了呢？析构函数会自动释放其空间，我们同样不会有任何资源泄漏。

总而言之，只要我们适当地组织好代码、利用好 RAII，实现矩阵的代码和使用矩阵的代码都可以更短、更清晰。我们可以统一在外层某个地方处理异常——通常会记日志、或在界面上向用户报告错误了。

### 异常的问题
1. 异常违反了“你不用就不需要付出代价”的 C++ 原则。只要开启了异常，即使不使用异常你编译出的二进制代码通常也会膨胀。

2. 异常比较隐蔽，不容易看出来哪些地方会发生异常和发生什么异常。

### 避免异常带来的问题有几点建议：
1. 写异常安全的代码，尤其在模板里。可能的话，提供强异常安全保证 ，在任何第三方代码发生异常的情况下，不改变对象的内容，也不产生任何资源泄漏。

2. 如果你的代码可能抛出异常的话，在文档里明确声明可能发生的异常类型和发生条件。确保使用你的代码的人，能在不检查你的实现的情况下，了解需要准备处理哪些异常。

3. 对于肯定不会抛出异常的代码，将其标为 noexcept。尤其是，移动构造函数、移动赋值运算符和 swap 函数一般需要保证不抛异常并标为 noexcept（析构函数通常不抛异常且自动默认为 noexcept，不需要标）。

### 错误码有什么好处或者弊端
- 直观易懂 ：错误码通常是一个整数或者枚举值，开发人员可以通过查阅文档或者代码注释，快速了解错误的类型和原因。例如在 C 语言中， fopen 函数打开文件失败时会返回 NULL ，并通过 errno 全局变量设置错误码，开发者可以根据不同的错误码判断是权限问题、文件不存在等。
- 性能开销小 ：错误码的处理只是简单的数值比较，不会涉及到栈展开等复杂操作，因此性能开销相对较小。在对性能要求较高的场景中，使用错误码更为合适。
- 兼容性好 ：错误码在各种编程语言和系统中都有广泛的应用，不同模块之间可以方便地传递和处理错误信息，具有较好的兼容性。
- 错误处理代码繁琐 ：在使用错误码时，每个可能出错的函数调用都需要进行错误检查，这会导致代码中充斥着大量的错误处理逻辑，降低代码的可读性和可维护性。

## 16.CRTP
CRTP是Curiously Recurring Template Pattern的缩写，中文可以翻成奇异递归模板，它是通过将子类类型作为模板参数传给基类的一种模板的使用技巧，类似下面代码形式：
```
template<typename T>
class Base {};
​
class Derived : public Base<Derived> {};
```

### 静态多态
通过CRTP这种编程技巧可以在C++中实现静态多态，也可以叫编译期多态，这是相对运行时多态发明的名字，下面通过一个具体的例子来理解静态多态。
```
template<typename T> 
class Base {
public:
  void foo() { static_cast<T *>(this)->internal_foo(); }
  void bar() { static_cast<T *>(this)->internal_bar(); }
};
​
class Derived1 : public Base<Derived1> {
public:
  void internal_foo() { cout << "Derived1 foo" << endl; }
  void internal_bar() { cout << "Derived1 bar" << endl; }
};

class Derived2 : public Base<Derived2> {
public:
  void internal_foo() { cout << "Derived2 foo" << endl; }
  void internal_bar() { cout << "Derived2 bar" << endl; }
};
```

使用CRTP可以使得类具有类似virtual function的效果，同时还没有virtual function的调用开销，因为virtual function调用需要通过vptr来找到vtbl进而找到真正的函数指针进行调用，同时对象size相比使用virtual function也会减小，但是CRTP也有明显的缺点，最直接的就是代码可读性降低（模板代码的通病）

### 代码复用
如果一些不同的类有一些相同的操作或者相同功能的变量，可以使用CRTP来复用代码，不用每个类都去定义一次这些相同的操作和变量了

假如Derived1、Derived2都有同样的foo、bar操作，区别只是里面操作的数据类型不同，那么可以把foo、bar提到基类中，这三个类略加修改之后如下：
```
template<typename T> 
class Base {
public:
  void foo(const T& t) { ... }
  void bar(const T& t) { ... }
​
protected:
  T data_;
};
​
class Derived1 : public Base<Derived1> {
public:
  void self_func1() { ... }
  void self_func2() { ... }
};

class Derived2 : public Base<Derived2> {
public:
  void self_func1() { ... }
  void self_func2() { ... }
};
```

### 实例化多套基类静态变量和方法
如果一些不同的类有一些相同性质的静态变量或者方法，可以使用CRTP的方法来定义这些静态变量和方法，不用每个类都去定义一次了，这种应用场景和上一节代码复用的场景很相似，只是因为这里是静态变量和方法，所以单独拆开来说
```
template<typename T> 
class Base {
public:
  static int getObjCnt() { return cnt; }
​
protected:
  static int cnt;
};
template <typename T> int Base<T>::cnt = 0;

class Derived1 : public Base<Derived1> {
public:
  Derived1() { cnt++; }
};
​
class Derived2 : public Base<Derived2> {
public:
  Derived2() { cnt++; }
};
```

### 什么虚函数调用是可以被通过CRTP消除的
当虚函数用于实现静态多态，且派生类类型在编译时已知，不存在运行时动态类型变化的需求时，就可以用 CRTP 来消除虚函数调用，从而提升性能并减少运行时开销。

## 17.static, extern 和 inline 
### static
修饰的变量或函数具有内链接属性，不可被其他文件引用，好处即外部文件中函数或变量可以重名，static的变量存储在GVAR（global value）内存区（静态存储区 ".data"），所以如果static修饰函数内局部变量，即使函数执行完成，堆栈释放而static变量不会被释放，但需要注意的是即使变量在全局区，但是可见域并没有改变，仅函数内可见。

### extern
修饰的变量或内存具有外链接属性，外部文件通过声明就可以引用其他文件中定义的符号（函数或变量），一般情况下函数或全局变量默认为extern属性。

#### extern 解决的问题
- 跨文件访问变量和函数 ：在一个文件中定义的全局变量和函数，在其他文件中如果需要使用，可以通过 extern 进行声明，告知编译器这些变量和函数在其他地方已经定义，链接器会在链接时找到对应的定义。
- 引用外部库 ：在使用外部库时，通常需要使用 extern 声明库中提供的函数和变量，以便在自己的代码中调用这些库函数。

### inline
inline和链接属性没有直接的关系，且只用来修饰函数，inline修饰的函数是对编译器做调用点代码展开建议，但具体是否展开由编译器决定，如果函数复杂度较高inline可能不会生效。因为内联函数要在调用点展开，所以编译器必须所有调用点可见函数的定义，不然就成为了普通函数调用，所以将内联函数的定义放在头文件里实现是合适的，不然你需要为每个文件实现一次。如果你真的打算每个文件里都实现一次，那么最好保证每个定义都是一样的，否则行为是未定义的。

inline 函数有以下特点：
- 多个定义的处理 ：在不同的编译单元（源文件）中可以有相同 inline 函数的多个定义。链接器在链接时会忽略这些重复定义，只要这些定义完全相同，链接过程就不会报错。
- 代码整合 ：由于 inline 函数可能在多个地方被展开，链接器不需要像普通函数那样去解析函数的调用地址，减少了函数调用的开销，同时也避免了因重复定义导致的链接错误。

#### 如果不同文件的 foo 函数实现不一样 inline 会怎么样呢
如果不同文件中 inline 修饰的 foo 函数实现不一样，链接时会产生未定义行为。虽然 inline 函数允许在多个编译单元中有定义，但要求这些定义必须完全相同（函数签名、内部实现、修饰符以及命名空间等都要保持一致）。如果实现不同，链接器无法确定使用哪个实现，从而可能导致程序在运行时出现不可预期的结果。