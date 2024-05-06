---
title: Professional C++ 面向对象笔记之一
date: 2024-05-05 18:21:47
updated: 2024-05-06 11:20:23
tags: [C++]
categories: 
  - [Modern C++]
keywords:
description: 本文对Professional C++(6th Edition) 第八章内容进行了总结整理。
top_img:
---

本文对Professional C++ (6th Edition) 第八章的一些重点内容进行了摘抄和总结。

## 知识点整理

当使用默认构造函数在堆栈上创建对象时，请使用大括号(`{}`)作为统一初始化语法或省略任何括号，不应使用小括号(否则会和函数调用混淆)。

**如果没有指定任何构造函数，编译器会为我们编写一个不带任何参数的构造函数**。编译器生成的默认构造函数会调用该类所有对象成员的默认构造函数，但不会初始化`int`和`double`等语言基元 (language primitives)。不过，它允许你创建该类的对象。但是，如果自己声明了任何构造函数，编译器就不再会生成默认构造函数。C++提供了有关语法，可满足如下特殊需求：

- ***Explicitly Defaulted Default Constructors*** (`= default`): C++支持显式默认构造函数的概念。这样就可以不必为默认构造函数提供一个空的实现
- ***Explicitly Deleted Default Constructors*** (`= delete`): 例如，我们可以定义一个只有静态成员函数的类，不想为该类编写任何构造函数，也不想让编译器生成默认构造函数。

有几种数据类型必须在 **ctor-initializer** 或 **in-class initializer** 中初始化：

- `const` data members;
- Reference data members;
- 没有默认构造器的object data members: C++会尝试使用默认构造函数初始化成员对象。如果不存在默认构造函数，则无法初始化对象，必须明确告诉它调用哪个构造函数;
- 没有默认构造器的base classes.

ctor-initializers有一个重要的注意事项：它们按照类定义中出现的顺序初始化数据成员，而不是按照ctor-initializer中的顺序！

**拷贝构造函数** (copy initializer) 应该从源对象复制所有数据成员。**如果自己不编写拷贝构造函数，C++会为我们生成一个**，它会根据源对象中的对应数据成员初始化新对象的每个数据成员。对于对象数据成员，这种初始化意味着会调用它们的拷贝构造函数。因此，在大多数情况下，无需指定复制构造函数！拷贝构造函数也可以被explicitly defaulted or deleted。

> 请注意默认构造函数和拷贝构造函数之间缺乏对称性。只要不明确定义拷贝构造函数，编译器就会为我们创建一个。另一方面，只要定义了任何构造函数，编译器就会停止生成默认构造函数。

注：C++11 *deprecated* the generation of a copy constructor if the class has a user-declared copy assignment operator or destructor. If you still need a compiler-generated copy constructor in such a case, you can explicitly `default` one. C++11 also *deprecated* the generation of a copy assignment operator if the class has a user-declared copy constructor or destructor. If you still need a compiler-generated copy assignment operator in such a case, you can explicitly `default` one. 详见[此页面](https://learn.microsoft.com/en-us/cpp/error-messages/compiler-warnings/c5267?view=msvc-170)。

Compiler-Generated Constructors总结如下：

![Compiler-Generated Constructors](/blogposts/img/Professional-CPP/compiler_generated_constructors.png)

**委托构造函数** (delegating constructor) 允许构造函数调用同一个类中的另一个构造函数。然而，这种调用不能放在构造函数体中；它必须位于ctor-initializer中，并且它必须是列表中唯一的成员初始化器。

单参数`double`和`string_view`构造函数可以用来将`double`或`string_view`转换成`SpreadsheetCell`。这样的构造函数被称为 **转换构造函数** (converting constructor)。编译器可以使用这样的构造函数为你执行隐式转换。这可能不总是我们想要的行为。可以通过将构造函数标记为`explicit`来阻止编译器进行这种隐式转换。`explicit`关键字只能在类定义中使用。当然，仅仅写`explicit(true)`等同于`explicit`，但在使用类型特征 (type traits) 的泛型模板代码的上下文中，这会更有用。通过类型特征，可以查询给定类型的某些属性，例如某个类型是否可以转换为另一个类型。这种类型特征的结果可以用作`explicit()`的参数。

建议至少将任何可以用单个参数调用的构造函数标记为`explicit`，以避免意外的隐式转换。如果有隐式转换的真实用例，可以使用`explicit(false)`来标记构造函数，以向用户说明隐式转换的可能性。

如果不声明析构函数，编译器就会为我们编写一个析构函数，执行 ***成员递归析构***，并允许删除对象。**栈中对象的销毁顺序与其声明（和构造）顺序相反**。这种排序也适用于作为其他对象的数据成员的对象：数据成员是按照它们在类中的声明顺序进行初始化的。因此，根据对象的销毁顺序与其构造顺序相反的规则， **数据成员对象的销毁顺序与其在类中的声明顺序相反**。

本文解释的赋值操作符有时也被称为 **拷贝赋值操作符** (copy assignment operator)，因为数据是从右侧对象复制到左侧对象的。还有另一种赋值操作符，即 **移动赋值操作符** (move assignment operator)，在这种操作符中，数据是移动而不是复制的，在某些使用情况下，移动赋值操作符可以提高性能。与往常一样，**如果不编写自己的赋值操作符，C++ 会为我们编写一个**，以允许对象之间相互赋值。C++的默认赋值行为与其默认拷贝行为几乎相同：它递归地将源对象中的每个数据成员赋值给目标对象。赋值操作符也可被explicitly defaulted or deleted。

## 代码

基于上述原则可写出如下示例代码 (这里的实现不是最好的，在后续的文章中会有更优的实现)。

这里的`SpreadsheetCell`复制构造函数只是为了演示。事实上，在这种情况下，复制构造函数可以省略，因为编译器默认生成的构造函数已经足够好了。类似的，显式的`SpreadsheetCell`赋值操作符也只是为了演示，它也可以省略，因为编译器默认生成的赋值操作符已经足够好；它可以对所有数据成员进行简单的成员赋值。但是，在某些条件下，编译器生成的赋值操作符并不足够。

### `SpreadsheetCell.cppm`

```C++
export module spreadsheet_cell;

import std;

export class SpreadsheetCell
{
public:
	SpreadsheetCell() = default;
	SpreadsheetCell(double initialValue);
	explicit SpreadsheetCell(std::string_view initialValue);
	SpreadsheetCell(const SpreadsheetCell& src);

	SpreadsheetCell& operator=(const SpreadsheetCell& rhs);

	void setValue(double value);
	double getValue() const;

	void setString(std::string_view value);
	std::string getString() const;

private:
	std::string doubleToString(double value) const;
	double stringToDouble(std::string_view value) const;

	double m_value{ 0 };
};
```

### `SpreadsheetCell.cpp`

```C++
module spreadsheet_cell;

import std;

SpreadsheetCell::SpreadsheetCell(double initialValue)
	: m_value{ initialValue }
{
}

SpreadsheetCell::SpreadsheetCell(std::string_view initialValue)
	: m_value{ stringToDouble(initialValue) }
{
}

SpreadsheetCell::SpreadsheetCell(const SpreadsheetCell& src)
	: m_value{ src.m_value }
{
	std::println("copy constructor", "");
}

SpreadsheetCell& SpreadsheetCell::operator=(const SpreadsheetCell& rhs)
{
	std::println("assignment operator", "");

	if (this == &rhs)
	{
		return *this;
	}

	m_value = rhs.m_value;
	return *this;
}

void SpreadsheetCell::setValue(double value)
{
	m_value = value;
}

double SpreadsheetCell::getValue() const
{
	return m_value;
}

void SpreadsheetCell::setString(std::string_view value)
{
	m_value = stringToDouble(value);
}

std::string SpreadsheetCell::getString() const
{
	return doubleToString(m_value);
}

std::string SpreadsheetCell::doubleToString(double value) const
{
	return std::to_string(value);
}

double SpreadsheetCell::stringToDouble(std::string_view value) const
{
	double number{ 0 };
	std::from_chars(value.data(), value.data() + value.size(), number);
	return number;
}
```

### `SpreadsheetTest.cpp`

测试程序：

```C++
import spreadsheet_cell;
import std;

using namespace std::literals;

int main()
{
	SpreadsheetCell myCell{ 5 };
	SpreadsheetCell anotherCell{ myCell };
	SpreadsheetCell aThirdCell = myCell;
	anotherCell = myCell;

	myCell = 10;
	// myCell = "20"sv;  // error!!!

	SpreadsheetCell myCell2{ 6 };
	std::string s1;
	s1 = myCell2.getString();

	SpreadsheetCell myCell3{ 7 };
	std::string s2 = myCell3.getString();

	auto myCellp{ std::make_unique<SpreadsheetCell>() };
	myCellp->setValue(3.8);
	std::println("Cell: {}, {}", myCellp->getValue(), myCellp->getString());

	return 0;
}
```
