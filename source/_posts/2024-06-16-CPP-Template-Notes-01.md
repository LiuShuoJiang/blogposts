---
title: C++ Template笔记第一章：函数模板
date: 2024-06-16 11:35:13
updated: 2024-06-19 12:11:24
tags: [C++, Template]
categories:
  - [Modern C++]
keywords:
description: 本文是对C++ Template第二版第一章函数模板内容的学习笔记。
top_img:
---

## 书中特殊的编程习惯

使用`int const N = 100;`而非`const int N = 100;`

再例如：

```C++
using CHARS = char*;
using CPTR = CHARS const;  // const pointer to chars
```

如果第二句写成`using CPTR = const CHARS;`则表示pointer to const chars，意义完全变了。

该习惯也适用于函数参数名：`void foo (int const& x);`

## 函数模板入门

求二值最大值：

```C++
template<typename T>
T max(T a, T b) {
    return b < a ? a : b;
}
```

注意不能写成`return a < b ? b : a;`(为什么？)

语法为：`template< comma-separated-list-of-parameters >`。`T`称为**type parameter**。

In `max()`, values of type `T` must also be *copyable* in order to be returned.

与类类型声明不同，声明类型参数时不能使用关键字`struct`代替`typename` (`class`可以但不建议)。

模板不会被编译成可以处理任何类型的单个实体。相反，对于使用模板的每种类型，都会从模板生成不同的实体。

The process of replacing template parameters by concrete types is called **instantiation**. It results in an **instance** of a template.

模板分两个阶段“编译”：

1. **Without instantiation** at ***definition time***, the template code itself is checked for correctness ignoring the template parameters.
2. At ***instantiation time***, the template code is checked (again) to ensure that all code is valid. That is, now especially, all parts that depend on template parameters are double-checked.

The fact that names are checked twice is called **two-phase lookup**.

两阶段翻译导致了实际处理模板时的一个重要问题：当函数模板的使用方式会触发其实例化时，编译器（在某些时候）需要查看该模板的定义(目前的解决方案是直接将模板实现在头文件中)。

## 模板参数推导

Type Conversions During Type Deduction(重要):

- When declaring call parameters ***by reference***,
  - Even trivial conversions do not apply to type deduction.
  - Two arguments declared with the same template parameter `T` must match exactly.
- When declaring call parameters ***by value***,
  - Only trivial conversions that decay are supported:
    - Qualifications with const or volatile are ignored,
    - References convert to the referenced type,
    - Raw arrays or functions convert to the corresponding pointer type.
  - For two arguments declared with the same template parameter `T` the *decayed types* must match.

还要注意，类型推断不适用于默认调用参数。为了支持这种情况，还必须为模板参数声明一个默认参数，例如：

```C++
template<typename T = std::string>
void f(T = "");

f();  // OK
```

## 多类型模板参数

函数返回类型取决于调用参数的顺序。解决方法依次为：

### 为返回类型引入第三个模板参数

```C++
template<typename T1, typename T2, typename RT>
RT max (T1 a, T2 b);
```

模板实参推导不考虑返回类型，而`RT`并不出现在函数调用参数的类型中。因此，无法推导`RT`。因此，必须明确指定模板参数列表。例如：`::max<int,double,double>(4, 7.2);`

另一种方法是只明确指定第一个参数，让推导过程推导出其余参数。一般来说，必须指定所有参数类型，直到无法隐式确定的最后一个参数类型。因此，如果改变我们示例中模板参数的顺序，调用者只需指定返回类型：

```C++
template<typename RT, typename T1, typename T2>
RT max(T1 a, T2 b);

::max<double>(4, 7.2) // OK: return type is double, T1 and T2 are deduced
```

### 让编译器找出返回类型

C++14之后：

```C++
template <typename T1, typename T2> auto max(T1 a, T2 b) {
    return b < a ? a : b;
}
```

事实上，在返回类型中使用`auto`而不使用相应的尾部返回类型 (trailing return type) (尾部返回类型应使用`->`引入)，表明实际的返回类型必须从函数体中的返回语句中推导出来。当然，从函数体中推导出返回类型必须是可能的。因此，代码必须可用，并且多个返回语句必须匹配。

注意返回类型可能是引用类型，因为在某些情况下`T`可能是引用。因此，应该返回`T`的decay类型：

```C++
#include <type_traits>

template <typename T1, typename T2>
auto max(T1 a, T2 b) -> typename std::decay<decltype(true ? a : b)>::type {
    return b < a ? a : b;
}
```

注意，`auto`类型的初始化总是会decay。当返回类型仅为`auto`(而没有别的修饰)时，这也适用于返回值。

### 将返回类型声明为两个参数类型的“公共类型”

```C++
#include <type_traits>

template <typename T1, typename T2> std::common_type_t<T1, T2> max(T1 a, T2 b) {
    return b < a ? a : b;
}
```

`std::common_type`是`<type_traits>`中定义的一个 **type trait**，它会产生一个structure，该structure有一个返回类型的`type` member。

注意以下两种用法等效：

```C++
typename std::common_type<T1, T2>::type // since C++11
std::common_type_t<T1, T2>              // equivalent since C++14
```

## 默认模板arguments

您还可以为模板参数定义默认值。这些值称为 **默认模板参数**，可用于任何类型的模板。

我们可以直接使用`operator?:`。但是，因为我们必须在声明调用参数`a`和`b`之前应用`operator?:`，所以我们只能使用它们的类型：

```C++
#include <type_traits>

// 注意此实现要求我们能够为传递的类型调用默认构造函数
template <typename T1, typename T2,
          typename RT = std::decay_t<decltype(true ? T1() : T2())>>
RT max(T1 a, T2 b) {
    return b < a ? a : b;
}
```

我们还可以使用`std::common_type<>`类型特征来指定返回类型的默认值：

```C++
#include <type_traits>

template <typename T1, typename T2, typename RT = std::common_type_t<T1, T2>>
RT max(T1 a, T2 b) {
    return b < a ? a : b;
}
```

## 函数模板的重载

非模板函数可以与具有相同名称且可以使用相同类型实例化的函数模板共存。在所有其他因素相同的情况下，重载解析过程优先选择非模板，而不是从模板生成的函数。

一般来说，在重载函数模板时，最好不要做超出必要的改动。应将更改限制在参数数量或明确指定模板参数上。否则，可能会产生意想不到的效果。例如，如果实现了通过引用传递参数的`max()`模板，并将其重载为通过值传递的两个C String，那么就不能使用如下三参数版本来计算三个C String的最大值：

```C++
#include <cstring>

// maximum of two values of any type (call-by-reference)
template <typename T> T const &max(T const &a, T const &b) {
    return b < a ? a : b;
}

// maximum of two C-strings (call-by-value)
char const *max(char const *a, char const *b) {
    return std::strcmp(b, a) < 0 ? a : b;
}

// maximum of three values of any type (call-by-reference)
template <typename T> T const &max(T const &a, T const &b, T const &c) {
    return max(max(a, b), c); // error if max(a,b) uses call-by-value
}

int main() {
    auto m1 = ::max(7, 42, 68); // OK

    char const *s1 = "frederic";
    char const *s2 = "anica";
    char const *s3 = "lucas";
    // auto m2 = ::max(s1, s2, s3); // run-time ERROR!!!
}
```

此外，要确保在调用函数之前声明函数的所有重载版本。这是因为在调用相应函数时，并非所有重载函数都是可见的，这一点可能很重要。


