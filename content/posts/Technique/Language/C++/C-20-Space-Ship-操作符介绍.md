---
title: "C++20 <=> Space-Ship 操作符介绍"
date: 2019-08-09T13:32:36+08:00
draft: false
tags:
  - c++
---

C++20 计划引入 `<=>` 操作符，学名是 Three-Way Comparison Operator，俗称 Space Ship Operator —— 这个运算符的引入从根本上重新定义了 C++ 如何进行比较，并且为比较系统带来了极大的灵活性，可能会大量缩减重复代码。这篇博客就简单介绍一下 `<=>` 操作符的用法。

<!--more-->

## 为何引入 <=> 运算符

在 C++ 中，所有的操作符都是同级的，不存在谁依赖于谁的关系。但是我们在为自定义类型重载比较运算符是却并不总是如此 —— 我们往往习惯先定义 `==` 与 `<` 运算符，然后剩余的 `!=, <=, >, >=` 运算符都能在这两个*基础*运算符之上定义。也就是说在语义（semantic）上，我们的比较运算符是分层级的，存在着某系基础比较运算符与在其之上衍生定义的次级运算符。

但是使用 `==` 和 `<` 作为**基础比较运算符**并不总是正确的。这种旧的比较运算符分层结构依赖于一个重要的假设：所有比较运算都符合三分法（trichotomy），即对于任意两个操作数 `a` 和 `b`，`a == b, a < b, a > b` 三者中总有一者是成立的；因为只有这样，`a >= b` 与 `!(a < b)` 才能是恒等价的。

但事实并非总是如此，在部分**偏序关系**（partial order）中（准确地说是所有非全序关系（total order）的偏序关系），存在两个操作数 `a` 和 `b` 使得 `a == b, a < b, a > b` 三者中任意一者都是不成立的。例如包括 `NaN` 的实数类型中，`NaN` 不大于、不小于也不等于任何一个实数；又例如自然数集上的整除关系，我们有 2 < 6、10 > 5、1 == 1，但是 3 和 5 互不整除，因此也不满足 `==, <, >` 三者中任意的一者。

除了解决偏序关系带来的比较问题，`<=>` 还解决了**对称异构比较**（symmetric heterogeneous comparison）的问题。在 C++17 即以前，如果我们实现了一个自己的整数类 Integer 而要将其与原始 int 类型进行比较，我们将不得不为每个运算符实现两个以上的的运算符重载函数，使得 `Integer @ int` 与 `int @ Integer` 都能正常工作（这里 `@` 代表任意比较运算符）。但是使用 `<=>` 运算符，你只需要实现一个 `operator<=>` 函数就足够了 —— 这大大减少了 C++ 编程者的重复代码。

还值得注意的是，所有的比较运算符号都有其**默认值**（default）。例如对于没有重载 `>=` 运算符的某种类型，编译系统会将 `a >= b` 自动转换为 `(a <=> b) >= 0`；当然你也可以显示声明 `bool operator>=(Typename&) = default;`。这就使得只要你正确地实现了 `<=>` 运算符，你就基本上不再需要为其他次级比较运算符编写重复而无意义的代码 —— 基础比较运输符的威力就在于此。

## 更复杂的返回值

旧的运算符重载的一大问题就在于其返回值是 boolean 类型，只可能有 `true/false` 两种可能；而 `<=>` 运算符的返回值通常有三种可能：`greater/equivalent/less`，这使得 `<=>` 运算符的运算结果有了更强大的表达能力，从而能正确地处理更复杂的情况。

`<=>` 运算符的返回值有五种类型之多，它们分别是 `std::strong_ordering`， `std::weak_ordering`， `std::partial_ordering`， `std::strong_equality`， `std::weak_equality`，每种类型都有 2~4 种取值可能，下面我就来简单介绍一下。

> 截至 2019-08-09，以下所有代码均只在 Visual Studio 2019 上测试过。MSVC 19.20+ 支持 <=> 运算符，Clang-8 部分支持（但我没有做测试），GCC/G++ 尚不支持。详细编译器支持请参见 https://en.cppreference.com/w/cpp/compiler_support。

### `std::strong_ordering`

`std::strong_ordering` 有三种取值：`greater/equal/less`，你可以简单地认为 `strong_ordering` 基本上就是我们平时最常见的比较类型，它满足全序关系。这里的 "strong" 表明这种比较关系中的等价关系是具有完全的可替换性（substitutability）的，也就是说对于任意函数 `f`，若 `a == b`，则恒有 `f(a) == f(b)` 成立。下面是一个简单的例子：

```C++
#include <compare>
#include <iomanip>
#include <iostream>

class Integer {
 public:
  int value;
  Integer(int value = 0) : value{value} {}
  bool operator==(const Integer& operand) { return value == operand.value; }
  std::strong_ordering operator<=>(const Integer& operand) {
    return value <=> operand.value;
  }
  std::strong_ordering operator<=>(const int operand) {
    return value <=> operand;
  }
};

int main() {
  Integer x{-1}, y{}, z{1};
  std::cout << std::boolalpha;
  // secondary comparison operators are defaulted
  std::cout << (x < y) << std::endl
            << (y < z) << std::endl
            << (x < z)
            << std::endl
            // symmetric heterogeneous comparisons
            << (1 <= x) << std::endl
            << (z >= 0) << std::endl
            << (-1 > y) << std::endl;
  return 0;
}
```

你还需要知道的是，`<=>` 运输符也是有默认值的。默认情况下，`<=>` 会对每一个成员应用 `<=>` 运算符，先被声明的成员具有更高的优先级 —— 这就像比较两个位数相同的整数，我们会先比较高位，若高位不同就直接返回比较结果，若高位相同再比较下一位。例如：

```c++
#include <compare>
#include <iomanip>
#include <iostream>

class Vector {
 public:
  int x, y;
  Vector(int x, int y) : x{x}, y{y} {}
  std::strong_ordering operator<=>(const Vector&) = default;
};

int main() {
  Vector v1{1, 2}, v2{2, 6}, v3{3, 4};
  std::cout << std::boolalpha;
  std::cout << (v1 < v2) << std::endl << (v2 < v3) << std::endl;
  return 0;
}
```

这个例子会输出 "true true"。但如果我们调换 x，y 的声明顺序（写成 `int y, x;`）就会输出 "true false"，因为按照声明顺序变量 y 具有更高的优先级。

### `std::weak_ordering`

`std::weak_ordering` 同样也是全序关系，其取值有三种可能：`greater/equivalent/less`。之所以使用 "weak" 来修饰，是因为这里的等价只代表了一种等价类（equivalent class）关系 —— 也就是说，一定存在某个函数 `f`，使得存在某一对 `a == b` 满足 `f(a) != f(b)`。

> 注意：`std::strong_ordering::equivalent` 同样存在，且与 `std::strong_ordering::equal` 是完全等价的。通常推荐在泛型编程（generic type programming）中使用 `equivalent` 关键字，因为 `<=>` 操作符的五种返回类型都支持；而 `equal` 则只被 `strong_ordering` 与 `strong_equality` 支持。

一个简单的等价类例子：

```c++
#include <compare>
#include <iomanip>
#include <iostream>

class IntMod {
 public:
  int value;
  IntMod(int value) : value{value} {}
  bool operator==(const IntMod& operand) const {
    return (value % 3) == (operand.value % 3);
  }
  std::weak_ordering operator<=>(const IntMod& operand) const {
    return (value % 3) <=> (operand.value % 3);
  }
};

int f(IntMod x) { return x.value; }

int main() {
  IntMod a{1}, b{4}, c{2};
  std::cout << std::boolalpha;
  std::cout << (a == b) << std::endl
            << (b < c) << std::endl
            << (f(a) == f(b)) << std::endl;
  return 0;
}
```

输出为 "true true false"。需要注意的是 `operator<=>(int, int)` 的默认返回类型是 `std::strong_ordering`，不过这里被隐式转换成 `std::weak_ordering`。

### `std::partial_ordering`

这种返回类型代表我们无法保证满足三分法；一个典型的例子就是 NaN，因为 NaN 不大于、小于或等于任何数，包括它自身。下面这个例子简单地实现一个 `IntNan` 类型：

```c++
#include <compare>
#include <iomanip>
#include <iostream>
#include <optional>

class IntNan {
 public:
  std::optional<int> val = std::nullopt;
  IntNan() = default;
  IntNan(const int value) { val = value; }
  bool operator==(const IntNan& operand) const {
    if (!val || !operand.val) return false;
    return *val == *operand.val;
  }
  std::partial_ordering operator<=>(const IntNan& operand) const {
    if (!val || !operand.val) return std::partial_ordering::unordered;
    return *val <=> *operand.val;
  }
};

int main() {
  IntNan a{}, b{1}, c{1}, d{2};
  std::cout << std::boolalpha;
  std::cout << (a == b) << std::endl
            << (b == c) << std::endl
            << (a == d) << std::endl
            << (b == d) << std::endl;
  return 0;
}
```

输出为 "false true false false"。

这段代码实现的特殊之处在于 `partial_ordering::unordered` 这个特殊取值。通常来说，`<=>` 的返回值有三种可能：`*_ordering::greater > 0`， `*_ordering::equal == 0` 或 `*_ordering::equivalent == 0` 和 `*_ordering:less < 0` —— 这也是三路比较运算符（Three-Way Comparison Operator）这个名字的由来。但是 `partial_ordering` 还有第四种可能取值 `unordered` ，且 `unordered == 0` `unordered > 0` `unordered < 0` 均不成立 —— 正是 `unordered` 的这一特殊性质使得我们能够实现不满足三分法的偏序关系。

### `std::strong_equality` 与 `std::weak_equality`

`*_equality` 返回类型与 `*_ordering` 返回类型的主要区别就在于是否支持 < 比较：前者只存在相等或不相等的关系，后者则提供了更为具体的大小比较。类似的，这里 `strong` 与 `weak` 地区别就在于：`strong` 表示了明确的相等关系，而 `weak` 则引入了等价类的概念。

然而 MSVC 编译器目前对这两种类型的支持似乎与 [open-std paper](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0515r3.pdf) 的描述还有出入，我无法使代码在 Visual Studio 2019 上成功地运行起来（也可能是我没能正确地理解 Paper）。总之这个问题就留到 2020 年再谈吧 >_<！

## 注意事项

- 上述五种类型的返回值均能与整数 0 作比较，也*只能* 与整数 0 作比较。一般的，我们有 `*::greater > 0`，`*::less < 0` 和 `*::equivalent == 0`；对于特殊的 `partial_ordering::unordered`，`>,==,<` 三者均不成立。

- 这五种类型之间存在着隐式类型转换，转换关系如下：

  - `strong_ordering` 能转换为 `weak_ordering`，`partial_ordering`，`strong_equality` 与 `weak_equality`
  - `weak_ordering` 能转换为 `partial_ordering` 与 `weak_equality`
  - `partial_ordering` 与 `strong_equality` 均只能转换为 `weak_equality`
  - `weak_equality` 不能转换为其他任意四种类型

  这种隐式转换对应着一种严格的 "IS-A" 关系，标准库一般也是通过类继承来实现这一关系的。

- 设存在 `a == b`，如果不存在任何函数 `f` 使得 `f(a) != f(b)`，那么这种关系是 `strong_*`；反之则为 `weak_*`

- 如果比较关系支持 < 运算，那么这种关系为 `*_ordering`；反之则为 `*_equality`

---

**参考文献**：

- [Barry Revzin's Blog: Comparisons in C++20](https://brevzin.github.io/c++/2019/07/28/comparisons-cpp20/)
- [Open-Std Paper: P0515](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0515r3.pdf)