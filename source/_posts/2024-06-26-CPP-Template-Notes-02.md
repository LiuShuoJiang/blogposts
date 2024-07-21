---
title: C++ Template笔记第二章：类模板
date: 2024-06-23 16:28:04
updated: 2024-06-26 16:28:04
tags: [C++, Template]
categories:
  - [Modern C++]
keywords:
description: 本文是对C++ Template第二版第二章类模板内容的学习笔记。
top_img:
---

## 模板类初步

以下以`Stack`类举例。每当在声明中使用此类的类型时，都必须使用`Stack<T>`，除非模板参数可以推断。但是，在类模板内部，使用类名(后面不跟模板参数)表示以模板参数作为参数的类。

```C++
#include <cassert>
#include <vector>

template <typename T> class Stack {
private:
    std::vector<T> elems;

public:
    void push(T const &elem);
    void pop();
    T const &top() const;
    bool empty() const { return elems.empty(); }
};

template <typename T> void Stack<T>::push(T const &elem) {
    elems.push_back(elem);
}

template <typename T> void Stack<T>::pop() {
    assert(!elems.empty());
    elems.pop_back();
}

template <typename T> T const &Stack<T>::top() const {
    assert(!elems.empty());
    return elems.back();
}
```

注意以上实现并不确保异常安全。

代码仅对调用的模板(成员)函数进行实例化。对于类模板，成员函数仅在使用时才进行实例化。这当然可以节省时间和空间，并允许仅部分使用类模板。

类模板通常会对其实例化的模板参数应用多项操作(包括构造和析构)。这可能会让人觉得这些模板参数必须提供类模板所有成员函数所需的所有操作。但事实并非如此：模板参数只需提供所需的所有必要操作(而不是可能需要的操作)。我们如何知道模板需要哪些操作才能实例化？术语 **concept** (C++20支持) 通常用于表示模板库中重复需要的一组约束。例如，C++标准库依赖于随机访问迭代器和默认构造等概念。

## 友元

对于`Stack`类，`operator<<`必须作为非成员函数实现。

```C++
template<typename T> class Stack {
    // ...
    void printOn() (std::ostream& strm) const {
    // ...
    }

    friend std::ostream& operator<< (std::ostream& strm, Stack<T> const& s) {
        s.printOn(strm);
        return strm;
    }
};
```

请注意，上述实现意味着类`Stack<>`的`operator<<`不是一个函数模板，而是一个在需要时用类模板实例化的 ***普通函数*** ( **templated entity** )。

但是，当尝试声明友元函数并在之后才定义它时，事情会变得更加复杂。我们有如下两个选择：

第一种选择是：我们可以隐式声明一个新的函数模板，它必须使用不同的模板参数，例如`U`：

```C++
template<typename T> class Stack {
    // ...
    template<typename U>
    friend std::ostream& operator<< (std::ostream&, Stack<U> const&);
};
```

再次使用`T`或跳过模板参数声明都是错误的(要么内部`T`隐藏外部`T`，要么我们在命名空间范围内声明一个非模板函数)。

第二种选择是：我们可以将`Stack<T>`的输出运算符前向声明为模板，但这意味着我们必须首先前向声明`Stack<T>`：

```C++
template<typename T> class Stack;

template<typename T> std::ostream& operator<< (std::ostream&, Stack<T> const&);
```

之后，我们可以以友元声明该函数：

```C++
template<typename T> class Stack {
    // ...
    friend std::ostream& operator<< <T> (std::ostream&, Stack<T> const&);
};
```

请注意"函数名"`operator<<`后面的`<T>`。因此，我们将非成员函数模板的特化声明为友元。如果没有`<T>`，我们将声明一个新的非模板函数！

## 类模板的特化 (Specialization)

可以针对某些模板参数对类模板进行特化。与函数模板的重载类似，对类模板进行特化允许针对某些类型优化实现，或针对类模板的实例化修复某些类型的错误行为。但是，如果特化了一个类模板，还必须特化所有成员函数。

要特化类模板，必须使用前导模板`<>`和类模板特化类型的规范来声明类。类型用作模板参数，必须在类名称后直接指定。

```C++
#include "stack1.hpp"  // 之前的Stack实现
#include <cassert>
#include <deque>
#include <string>

template <> class Stack<std::string> {
private:
    std::deque<std::string> elems; // elements

public:
    void push(std::string const &); // push element
    void pop();                     // pop element
    std::string const &top() const; // return top element
    bool empty() const {            // return whether the stack is empty
        return elems.empty();
    }
};

void Stack<std::string>::push(std::string const &elem) {
    elems.push_back(elem); // append copy of passed elem
}

void Stack<std::string>::pop() {
    assert(!elems.empty());
    elems.pop_back(); // remove last element
}

std::string const &Stack<std::string>::top() const {
    assert(!elems.empty());
    return elems.back(); // return copy of last element
}
```

在这个例子中，specialization使用引用语义将字符串参数传递给`push()`，这对于这个特定类型来说更有意义(尽管我们最好传递一个forwarding reference)。另一个区别是使用`deque`而不是`vector`来管理`Stack`内的元素。虽然这在这里没有什么特别的好处，但它确实表明特化的实现可能看起来与主模板的实现非常不同。

## 部分特化 (Partial Specialization)

类模板可以部分特化。可以为特定情况提供特殊实现，但某些模板参数仍必须由用户定义。

```C++
#include "stack1.hpp"

template <typename T> class Stack<T *> {
private:
    std::vector<T *> elems;

public:
    void push(T *);
    T *pop();
    T *top() const;
    bool empty() const { return elems.empty(); }
};

template <typename T> void Stack<T *>::push(T *elem) { elems.push_back(elem); }

template <typename T> T *Stack<T *>::pop() {
    assert(!elems.empty());
    T *p = elems.back();
    elems.pop_back(); // remove last element
    return p;         // and return it (unlike in the general case)
}

template <typename T> T *Stack<T *>::top() const {
    assert(!elems.empty());
    return elems.back(); // return copy of last element
}
```

使用方法如下：

```C++
Stack<int *> ptrStack;
ptrStack.push(new int{42});
std::cout << *ptrStack.top() << '\n';
delete ptrStack.top();
```

## 默认类模板参数 (Default Class Template Arguments)

对于函数模板，可以为类模板参数定义默认值。例如，在类`Stack<>`中，可以将用于管理元素的容器定义为第二个模板参数，并使用`std::vector<>`作为默认值：

```C++
#include <cassert>
#include <vector>

template <typename T, typename Cont = std::vector<T>> class Stack {
private:
    Cont elems; // elements

public:
    void push(T const &elem); // push element
    void pop();               // pop element
    T const &top() const;     // return top element
    bool empty() const {      // return whether the stack is empty
        return elems.empty();
    }
};

template <typename T, typename Cont> void Stack<T, Cont>::push(T const &elem) {
    elems.push_back(elem); // append copy of passed elem
}

template <typename T, typename Cont> void Stack<T, Cont>::pop() {
    assert(!elems.empty());
    elems.pop_back(); // remove last element
}

template <typename T, typename Cont> T const &Stack<T, Cont>::top() const {
    assert(!elems.empty());
    return elems.back(); // return copy of last element
}
```

使用方法如下：

```C++
template <typename T> using DequeStack = Stack<T, std::deque<T>>;

Stack<int> intStack;

// Stack<double, std::deque<double>> dblStack;
DequeStack<double> dblStack;

intStack.push(7);
std::cout << intStack.top() << '\n';
intStack.pop();

dblStack.push(42.42);
std::cout << dblStack.top() << '\n';
dblStack.pop();
```

## 类型别名 (Type Aliases)

要简单地为完整类型定义一个新名称，有两种方法可以做到：

- **Typedef name**：使用`typedef`；
- **Alias declaration** (建议使用)：使用`using`。

与`typedef`不同，alas declaration可以模板化，以便为类型系列提供方便的名称。这在C++11之后可用，称为 **alias template**。

自C++14起，标准库使用Type Traits Suffix `_t`技术为标准库中产生类型的所有type traits定义快捷方式。例如，可以编写：

```C++
std::add_const_t<T> // 自 C++14 起
```

而不是

```C++
typename std::add_const<T>::type // 自 C++11 起
```

标准库定义：

```C++
namespace std { 
    template<typename T> using add_const_t = typename add_const<T>::type; 
}
```

## 类模板参数推导 (Class Template Argument Deduction)

在C++17之前，总是必须将所有模板参数类型传递给类模板(除非它们具有默认值)。从C++17开始，总是必须显式指定模板参数的约束被放宽了。相反，如果构造函数能够推断出所有模板参数(没有默认值)，则可以跳过显式定义模板参数。

通过提供传递一些初始参数的构造函数，可以支持推断`Stack`的元素类型。例如，我们可以提供一个可由单个元素初始化的`Stack`：

```C++
template<typename T>
class Stack {
private:
    std::vector<T> elems; // elements
public:
    Stack () = default;
    Stack (T elem) // initialize stack with one element
        : elems({std::move(elem)}) {}
    // ...
};
```

这使得可以按如下方式声明一个`Stack`：

```C++
Stack intStack = 0; // Stack<int>
```

通过用整数`0`初始化堆栈，模板参数`T`被推导为`int`，从而实例化`Stack<int>`。

请注意由于`int`构造函数的定义，必须请求默认构造函数以其默认行为可用，因为只有在未定义其他构造函数时，默认构造函数才可用。

注意，与函数模板不同，类模板参数不能仅部分推断 (通过仅明确指定 *部分* 模板参数)。

可以定义特定的 **deduction guides** 来提供额外的或修复现有的类模板参数推导。例如，可以定义每当传递字符串文字或C字符串时，都会为`std::string`实例化`Stack`：

```C++
Stack(char const*) -> Stack<std::string>;
```

此引导必须出现在与类定义相同的范围(命名空间)中。通常，它在类定义之后。我们将遵循`->`的类型称为deduction guide的 **guided type**。

```C++
Stack stringStack{"bottom"}; // OK: Stack<std::string> deduced since C++17
// Stack stringStack = "bottom"; // Stack<std::string> deduced, but still not valid
```

## 模板化聚合 (Templatized Aggregates)

**聚合类 (Aggregate classes)** (没有用户提供的、显式的或继承的构造函数、没有私有的或受保护的非静态数据成员、没有虚函数、也没有虚的、私有的或受保护的基类的类/结构体) 也可以是模板。例如：

```C++
template<typename T>
struct ValueWithComment {
    T value;
    std::string comment;
};
```

定义一个聚合类，该聚合类针对其所持有的值`value`的类型进行参数化。可以像声明任何其他类模板一样声明对象，并且仍将其用作聚合类：

```C++
ValueWithComment<int> vc;
vc.value = 42;
vc.comment = "initial value";
```

从C++17开始，可以为聚合类模板定义deduction guide：

```C++
ValueWithComment(char const*, char const*) -> ValueWithComment<std::string>;

ValueWithComment vc2 = {"hello", "initial value"};
```

如果没有deduction guide，初始化就不可能实现，因为`ValueWithComment`没有构造函数来执行推导。


