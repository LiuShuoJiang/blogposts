---
title: Professional C++ 面向对象笔记之二
date: 2024-05-06 12:05:01
updated: 2024-05-06 20:10:23
tags: [C++]
categories:
  - [Modern C++]
keywords:
description: 本文对Professional C++(6th Edition) 第九章内容进行了总结整理。
top_img:
highlight_shrink: true
---

本文对Professional C++ (6th Edition) 第九章的一些重点内容进行了摘抄和总结。

## `Spreadsheet`类初步

### 初步知识点整理

#### 浅拷贝和深拷贝

如果自己不编写复制构造函数和复制赋值操作符，C++会为我们编写它们。这些编译器生成的成员函数会递归调用对象数据成员的复制构造函数或复制赋值操作符。不过，对于`int`、`double`和指针等基元，它们提供的是浅层或按位复制或赋值 (shallow or bitwise copying or assignment)：它们只是将源对象中的数据成员直接复制或赋值给目标对象。这就给在对象中动态分配内存带来了问题 (例如：**悬垂指针** (dangling pointer))。

拷贝构造函数和拷贝赋值操作符必须进行 **深度拷贝**；也就是说，它们不能仅仅拷贝指针数据成员，还必须拷贝这些指针指向的实际数据。

#### Copy-Swap Idiom

实现异常安全的 **拷贝和交换惯用法** ( **copy and swap idiom** )的一个要求是`swap()`不会抛出任何异常，因此它被标记为`noexcept`。有了copy and swap idiom，就不需要像"`if (this == &rhs) { return *this; }`"这样的自赋值测试了。

> The copy-and-swap idiom can be used for more than just assignment operators. It can be used for any operation that takes multiple steps and that you want to turn into an ***all-or-nothing operation***: first, make a copy; then, do all the modifications on the copy; and finally, if there are no errors, perform a non-throwing swap operation.

### 初步代码

#### `SpreadsheetCell.cppm`

```C++
export module spreadsheet_cell;

import std;

export class SpreadsheetCell
{
public:
	SpreadsheetCell() = default;
	SpreadsheetCell(double initialValue);
	SpreadsheetCell(std::string_view initialValue);

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

#### `SpreadsheetCell.cpp`

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

#### `Spreadsheet.cppm`

```C++
export module spreadsheet;

export import spreadsheet_cell;

import std;

export class Spreadsheet
{
public:
	Spreadsheet(std::size_t width, std::size_t height);
	Spreadsheet(const Spreadsheet& src);
	~Spreadsheet();

	Spreadsheet& operator=(const Spreadsheet& rhs);

	void setCellAt(std::size_t x, std::size_t y, const SpreadsheetCell& cell);
	SpreadsheetCell& getCellAt(std::size_t x, std::size_t y);

	void swap(Spreadsheet& other) noexcept;

private:
	void verifyCoordinate(std::size_t x, std::size_t y) const;

	std::size_t m_width{ 0 };
	std::size_t m_height{ 0 };
	SpreadsheetCell** m_cells{ nullptr };
};

export void swap(Spreadsheet& first, Spreadsheet& second) noexcept;
```

#### `Spreadsheet.cpp`

```C++
module spreadsheet;

import std;

Spreadsheet::Spreadsheet(std::size_t width, std::size_t height)
	: m_width{ width }, m_height{ height }
{
	m_cells = new SpreadsheetCell * [m_width];
	for (std::size_t i{ 0 }; i < m_width; ++i)
	{
		m_cells[i] = new SpreadsheetCell[m_height];
	}
}

Spreadsheet::~Spreadsheet()
{
	for (std::size_t i{ 0 }; i < m_width; ++i)
	{
		delete[] m_cells[i];
	}
	delete[] m_cells;
	m_cells = nullptr;
}

Spreadsheet::Spreadsheet(const Spreadsheet& src)
	: Spreadsheet{ src.m_width, src.m_height }
{
	// 该构造函数的ctor-initializer首先委托非拷贝构造函数分配适当数量的内存

	// 接下来才拷贝数据
	for (std::size_t i{ 0 }; i < m_width; ++i)
	{
		for (size_t j{ 0 }; j < m_height; ++j)
		{
			m_cells[i][j] = src.m_cells[i][j];
		}
	}
}

void Spreadsheet::verifyCoordinate(std::size_t x, std::size_t y) const
{
	if (x >= m_width)
	{
		throw std::out_of_range{ std::format("x ({}) must be less than width ({}).", x, m_width) };
	}
	if (y >= m_height) 
	{
		throw std::out_of_range{ std::format("y ({}) must be less than height ({}).", y, m_height) };
	}
}

void Spreadsheet::setCellAt(std::size_t x, std::size_t y, const SpreadsheetCell& cell)
{
	verifyCoordinate(x, y);
	m_cells[x][y] = cell;
}

SpreadsheetCell& Spreadsheet::getCellAt(std::size_t x, std::size_t y)
{
	verifyCoordinate(x, y);
	return m_cells[x][y];
}

void Spreadsheet::swap(Spreadsheet& other) noexcept
{
	std::swap(m_width, other.m_width);
	std::swap(m_height, other.m_height);
	std::swap(m_cells, other.m_cells);
}

void swap(Spreadsheet& first, Spreadsheet& second) noexcept
{
	first.swap(second);
}

// copy and swap idiom
Spreadsheet& Spreadsheet::operator=(const Spreadsheet& rhs)
{
	Spreadsheet temp{ rhs };
	swap(temp);
	return *this;
}
```

实现使用了copy and swap idiom。首先，创建右侧对象的一个副本，称为`temp`。然后，将当前对象与这个副本交换。这种模式是实现赋值运算符的推荐方式，因为它保证了强异常安全性，意味着如果发生任何异常，当前的`Spreadsheet`对象的状态将保持不变。这种惯用法分为三个阶段实现：

1. 第一阶段制作一个临时副本。这不会修改当前`Spreadsheet`对象的状态，因此在这个阶段抛出异常时不会有问题。
2. 第二阶段使用`swap()`函数交换创建的临时副本与当前对象。`swap()`函数应当不会抛出异常。
3. 第三阶段是销毁临时对象，此时临时对象包含了原始对象（因为交换），以清理任何内存。

## `Spreadsheet`类进阶一

### 进阶一知识点整理

#### 移动语义

类的移动语义需要一个**移动构造函数** (move constructor) 和一个 **移动赋值运算符** (move assignment operator)。当源对象是一个临时对象，操作完成后将被销毁时，或者当显式使用`std::move()`时，编译器可以使用这些。移动操作将内存和其他资源的所有权从一个对象转移到另一个对象。它基本上是对数据成员进行 ***浅拷贝***，并切换分配的内存和其他资源的所有权，以防止悬垂的指针或资源和防止内存泄漏。

移动构造函数和移动赋值运算符都将数据成员从源对象移动到新对象，使源对象处于某种有效但不确定的状态。通常，源对象的数据成员会被重置为“空”值，但这不是严格要求。然而，程序员应确保在移动操作后源对象处于一个明确定义的空状态。为了安全起见， ***永远不要使用已经移动过的对象，因为这可能会触发未定义的行为***。来自标准库的一些显著例外是`std::unique_ptr`和`shared_ptr`。标准库明确指出，这些智能指针在移动后必须将其内部指针重置为`nullptr`，这使得在移动操作后重用这样的智能指针变得安全。

显然，移动语义只有在知道已经不再需要源对象时才有用。

#### 右值和左值

如果一个函数通过 **值** 返回内容，调用该函数的结果是一个 **右值**（rvalue），一个临时对象。如果函数返回一个 **非常量引用**（reference-to-non-const），则调用该函数的结果是一个 **左值**（lvalue），因为你可以在赋值的左侧使用该结果。**右值引用** (rvalue reference) 是对右值的引用。特别地，当右值是一个临时对象或使用`std::move()`显式移动的对象时，会应用这一概念。

右值引用的目的是使得在涉及右值时可以选择特定的函数重载。这允许某些通常涉及复制较为“笨重“的值的操作改为复制指向那些值的指针。一个函数可以通过使用`&&`作为参数规范的一部分来指定一个右值引用参数，例如，`type&& name`。通常，一个临时对象会被看作是`const type&`，但当有一个使用右值引用的函数重载时，一个临时对象可以被解析到那个重载。

详见以下测试程序：

```C++
import std;

void helper(std::string&& message)
{
	std::println("print message in helper function: {}", message);
}

// lvalue reference parameter
void handleMessage(std::string& message)
{
	std::println("handleMessage with lvalue reference: {}", message);
}

// rvalue reference parameter
void handleMessage(std::string&& message)
{
	std::println("handleMessage with rvalue reference: {}", message);
	helper(std::move(message));
}

int main()
{
	std::string a{ "Hello " };
	std::string b{ "World" };

	// Handle a named variable
	handleMessage(a);             // Calls handleMessage(string& value)

	// Handle an expression
	handleMessage(a + b);         // Calls handleMessage(string&& value)

	// Handle a literal
	handleMessage("Hello World"); // Calls handleMessage(string&& value)

	// Handle a named variable and force to use rvalue reference function
	handleMessage(std::move(b));  // Calls handleMessage(string&& value)
}
```

可以通过使用`std::move()`强制编译器调用`handleMessage()`的右值引用重载。`move()`唯一的作用是将左值转换为右值引用；也就是说，它实际上并不执行任何移动操作。然而，通过返回一个右值引用，它允许编译器找到接受右值引用的`handleMessage()`的重载，这样就可以执行移动操作。注意：***一个具名的右值引用，比如一个右值引用参数，本身是一个左值，因为它有一个名称！***

#### Rule of Five和Rule of Zero

编译器仅当类没有用户声明的拷贝构造函数、拷贝赋值运算符、移动赋值运算符或析构函数时，才会自动生成默认的移动构造函数。如果类没有用户声明的拷贝构造函数、移动构造函数、拷贝赋值运算符或析构函数，那么编译器将为该类生成默认的移动赋值运算符。

> 当声明一个或多个特殊成员函数（ **析构函数**、**拷贝构造函数**、**移动构造函数**、**拷贝赋值运算符** 和 **移动赋值运算符** ）时，推荐声明 ***所有*** 这些函数。这被称为 **五法则（rule of five）**。可以为它们提供明确的实现，或者显式地将它们设为默认（`=default`）或删除（`=delete`）。

注：在现代C++中，应当使用零法则：**零法则 (rule of zero)** 规定，在设计类时，不需要这五个特殊成员函数中的任何一个。如何做到这一点呢？对于非多态类型，可以避免使用旧式的动态分配内存或其他资源。取而代之的是使用标准库容器和智能指针等现代构造。五法则应仅限于自定义资源获取初始化（resource acquisition is initialization, RAII）类。RAII类拥有资源的所有权，并在适当的时候处理资源的deallocation。一些多态类型也需要遵循五法则。

### 进阶一代码

`SpreadsheetCell`类同前。

#### `Spreadsheet.cppm`修改一

```C++
export module spreadsheet;

export import spreadsheet_cell;

import std;

export class Spreadsheet
{
public:
	Spreadsheet(std::size_t width, std::size_t height);
	Spreadsheet(const Spreadsheet& src);
	Spreadsheet(Spreadsheet&& src) noexcept;  // Move constructor
	~Spreadsheet();

	Spreadsheet& operator=(const Spreadsheet& rhs);
	Spreadsheet& operator=(Spreadsheet&& rhs) noexcept;  // Move assignment

	void setCellAt(std::size_t x, std::size_t y, const SpreadsheetCell& cell);
	SpreadsheetCell& getCellAt(std::size_t x, std::size_t y);

	void swap(Spreadsheet& other) noexcept;

private:
	void verifyCoordinate(std::size_t x, std::size_t y) const;

	void cleanup() noexcept;
	void moveFrom(Spreadsheet& src) noexcept;

	std::size_t m_width{ 0 };
	std::size_t m_height{ 0 };
	SpreadsheetCell** m_cells{ nullptr };
};

export void swap(Spreadsheet& first, Spreadsheet& second) noexcept;
```

#### `Spreadsheet.cpp`修改一

```C++
module spreadsheet;

import std;

Spreadsheet::Spreadsheet(std::size_t width, std::size_t height)
	: m_width{ width }, m_height{ height }
{
	std::println("Normal constructor");

	m_cells = new SpreadsheetCell * [m_width];
	for (std::size_t i{ 0 }; i < m_width; ++i)
	{
		m_cells[i] = new SpreadsheetCell[m_height];
	}
}

Spreadsheet::~Spreadsheet()
{
	cleanup();
}

Spreadsheet::Spreadsheet(const Spreadsheet& src)
	: Spreadsheet{ src.m_width, src.m_height }
{
	std::println("copy constructor");
	// 该构造函数的ctor-initializer首先委托非拷贝构造函数分配适当数量的内存

	// 接下来才拷贝数据
	for (std::size_t i{ 0 }; i < m_width; ++i)
	{
		for (size_t j{ 0 }; j < m_height; ++j)
		{
			m_cells[i][j] = src.m_cells[i][j];
		}
	}
}

void Spreadsheet::cleanup() noexcept
{
	for (std::size_t i{ 0 }; i < m_width; ++i)
	{
		delete[] m_cells[i];
	}
	delete[] m_cells;
	m_cells = nullptr;

	m_width = m_height = 0;
}


void Spreadsheet::verifyCoordinate(std::size_t x, std::size_t y) const
{
	if (x >= m_width)
	{
		throw std::out_of_range{ std::format("x ({}) must be less than width ({}).", x, m_width) };
	}
	if (y >= m_height) 
	{
		throw std::out_of_range{ std::format("y ({}) must be less than height ({}).", y, m_height) };
	}
}

void Spreadsheet::setCellAt(std::size_t x, std::size_t y, const SpreadsheetCell& cell)
{
	verifyCoordinate(x, y);
	m_cells[x][y] = cell;
}

SpreadsheetCell& Spreadsheet::getCellAt(std::size_t x, std::size_t y)
{
	verifyCoordinate(x, y);
	return m_cells[x][y];
}

void Spreadsheet::swap(Spreadsheet& other) noexcept
{
	std::swap(m_width, other.m_width);
	std::swap(m_height, other.m_height);
	std::swap(m_cells, other.m_cells);
}

void swap(Spreadsheet& first, Spreadsheet& second) noexcept
{
	first.swap(second);
}

// copy and swap idiom
Spreadsheet& Spreadsheet::operator=(const Spreadsheet& rhs)
{
	std::println("Copy assignment operator");

	Spreadsheet temp{ rhs };
	swap(temp);
	return *this;
}

void Spreadsheet::moveFrom(Spreadsheet& src) noexcept
{
	//// Shallow copy of data!!!
	//m_width = src.m_width;
	//m_height = src.m_height;
	//m_cells = src.m_cells;

	//// Reset the source object, because ownership has been moved!
	//src.m_width = 0;
	//src.m_height = 0;
	//src.m_cells = nullptr;

	// 更好的写法
	m_width = std::exchange(src.m_width, 0);
	m_height = std::exchange(src.m_height, 0);
	m_cells = std::exchange(src.m_cells, nullptr);
}

// Move constructor
Spreadsheet::Spreadsheet(Spreadsheet&& src) noexcept
{
	std::println("Move constructor");

	moveFrom(src);
}

// Move assignment operator
Spreadsheet& Spreadsheet::operator=(Spreadsheet&& rhs) noexcept
{
	std::println("Move assignment operator");

	// check for self-assignment
	if (this == &rhs)
	{
		return *this;
	}

	// Free the old memory and move ownership
	cleanup();

	moveFrom(rhs);

	return *this;
}
```

`moveFrom()`成员函数对三个数据成员进行直接赋值，因为它们都是原始类型。如果对象包含其他对象作为数据成员，则应使用`std::move()`移动这些对象。假设`Spreadsheet`类还有一个名为`m_name`的`std::string`数据成员。那么`moveFrom()`成员函数的实现过程如下：

```C++
void Spreadsheet::moveFrom(Spreadsheet& src) noexcept
{
	m_name = std::move(src.m_name);
	m_width = std::exchange(src.m_width, 0);
	m_height = std::exchange(src.m_height, 0);
	m_cells = std::exchange(src.m_cells, nullptr);
}
```

## `Spreadsheet`类进阶二

### 进阶二知识点整理

#### Move-Swap Idiom

移动构造函数可以简单地将默认构造的`*this`与给定的源对象交换。移动赋值运算符使用 **移动-交换惯用法** (move-swap idiom)，这与之前讨论的拷贝-交换惯用法类似。

用`swap()`来实现移动构造函数和移动赋值运算符需要更少的代码。当添加数据成员时，引入错误的可能性也较低，因为只需要更新`swap()`实现以包括那些新的数据成员。

但是对于移动赋值运算符而言，这样做并不能保证`this`的内容立即被清理。相反，`this`的内容会通过`rhs`逃逸到移动赋值运算符的调用者那里，因此其存活时间可能比预期的要长。

#### 什么时候按值传递？

自C++17起，编译器不允许对形式为 `return object;` 的语句执行任何对象的复制或移动，其中`object`是一个无名临时对象。这被称为 ***复制/移动操作的强制省略*** (mandatory elision of copy/move operations)，意味着通过值返回对象根本没有性能损失。如果`object`是一个非函数参数的局部变量，允许 ***非强制性省略复制/移动操作*** (non-mandatory elision of copy/move operations)，这种优化也被称为 **命名返回值优化**（named return value optimization, NRVO）。这种优化并不被标准所保证。一些编译器只在发布构建中执行此优化，而不在调试构建中执行。通过强制性和非强制性省略，编译器可以避免复制从函数返回的对象。这导致 ***零拷贝的按值传递语义*** (zero-copy pass-by-value semantics)。

当从函数返回一个局部变量或无名临时对象时，简单地写 `return object;`，不要使用`std::move()`。

到目前为止，建议始终使用`const`引用参数传递对象，以避免任何不必要的复制，但现在我们建议使用按值传递：对于不需要复制的参数，使用`const`引用传递仍然是正确的方式。按值传递的建议只适用于函数本来就会复制的参数。在这种情况下，使用按值传递语义，代码对于左值和右值都是最优的。如果传入一个左值，它只会被复制一次，就像使用`const`引用参数一样。而如果传入一个右值，则不会进行复制，就像使用右值引用参数一样。

对于函数本质上会复制的参数，推荐按值传递，但仅当参数类型支持移动语义且不需要参数的多态行为时。否则，使用`const`引用参数。按值传递多态类型可能会导致切片 (slicing)。

例如下列代码：

```C++
import std;
using namespace std;

class DataHolder
{
public:
	//void setData(const vector<int>& data) { m_data = data; }
	//void setData(vector<int>&& data) { m_data = move(data); }
	void setData(vector<int> data) { m_data = move(data); }

private:
	vector<int> m_data;
};

int main()
{
	DataHolder wrapper;
	vector myData{ 11, 22, 33 };
	wrapper.setData(myData);
	wrapper.setData({ 22, 33, 44 });
}
```

不能声明静态成员函数为`const`，因为这是多余的。静态成员函数不作用于类的特定实例，因此不可能改变内部值。可以在非`const`对象上调用`const`和非`const`成员函数。但是，只能在常量对象上调用常量成员函数。

### 进阶二代码

`SpreadsheetCell`类同前。

#### `Spreadsheet.cppm`修改二

```C++
export module spreadsheet;

export import spreadsheet_cell;

import std;

export class Spreadsheet
{
public:
	Spreadsheet(std::size_t width, std::size_t height);
	Spreadsheet(const Spreadsheet& src);
	Spreadsheet(Spreadsheet&& src) noexcept;  // Move constructor
	~Spreadsheet();

	Spreadsheet& operator=(const Spreadsheet& rhs);
	Spreadsheet& operator=(Spreadsheet&& rhs) noexcept;  // Move assignment

	void setCellAt(std::size_t x, std::size_t y, const SpreadsheetCell& cell);
	SpreadsheetCell& getCellAt(std::size_t x, std::size_t y);

	void swap(Spreadsheet& other) noexcept;

private:
	void verifyCoordinate(std::size_t x, std::size_t y) const;

	std::size_t m_width{ 0 };
	std::size_t m_height{ 0 };
	SpreadsheetCell** m_cells{ nullptr };
};

export void swap(Spreadsheet& first, Spreadsheet& second) noexcept;
```

#### `Spreadsheet.cpp`修改二

```C++
module spreadsheet;

import std;

Spreadsheet::Spreadsheet(std::size_t width, std::size_t height)
	: m_width{ width }, m_height{ height }
{
	std::println("Normal constructor");

	m_cells = new SpreadsheetCell * [m_width];
	for (std::size_t i{ 0 }; i < m_width; ++i)
	{
		m_cells[i] = new SpreadsheetCell[m_height];
	}
}

Spreadsheet::~Spreadsheet()
{
	for (std::size_t i{ 0 }; i < m_width; ++i)
	{
		delete[] m_cells[i];
	}
	delete[] m_cells;
	m_cells = nullptr;
}

Spreadsheet::Spreadsheet(const Spreadsheet& src)
	: Spreadsheet{ src.m_width, src.m_height }
{
	std::println("copy constructor");
	// 该构造函数的ctor-initializer首先委托非拷贝构造函数分配适当数量的内存

	// 接下来才拷贝数据
	for (std::size_t i{ 0 }; i < m_width; ++i)
	{
		for (size_t j{ 0 }; j < m_height; ++j)
		{
			m_cells[i][j] = src.m_cells[i][j];
		}
	}
}


void Spreadsheet::verifyCoordinate(std::size_t x, std::size_t y) const
{
	if (x >= m_width)
	{
		throw std::out_of_range{ std::format("x ({}) must be less than width ({}).", x, m_width) };
	}
	if (y >= m_height) 
	{
		throw std::out_of_range{ std::format("y ({}) must be less than height ({}).", y, m_height) };
	}
}

void Spreadsheet::setCellAt(std::size_t x, std::size_t y, const SpreadsheetCell& cell)
{
	verifyCoordinate(x, y);
	m_cells[x][y] = cell;
}

SpreadsheetCell& Spreadsheet::getCellAt(std::size_t x, std::size_t y)
{
	verifyCoordinate(x, y);
	return m_cells[x][y];
}

void Spreadsheet::swap(Spreadsheet& other) noexcept
{
	std::swap(m_width, other.m_width);
	std::swap(m_height, other.m_height);
	std::swap(m_cells, other.m_cells);
}

void swap(Spreadsheet& first, Spreadsheet& second) noexcept
{
	first.swap(second);
}

// copy and swap idiom
Spreadsheet& Spreadsheet::operator=(const Spreadsheet& rhs)
{
	std::println("Copy assignment operator");

	Spreadsheet temp{ rhs };
	swap(temp);
	return *this;
}

// Move constructor
Spreadsheet::Spreadsheet(Spreadsheet&& src) noexcept
{
	std::println("Move constructor");

	swap(src);
}

// Move assignment operator
Spreadsheet& Spreadsheet::operator=(Spreadsheet&& rhs) noexcept
{
	std::println("Move assignment operator");
	
	auto moved{ std::move(rhs) };  // Move rhs into moved (noexcept)
	swap(moved);  // Commit the work with only non-throwing operations

	return *this;
}
```

#### 进阶二的测试程序代码

```C++
import spreadsheet;
import std;

Spreadsheet createObject()
{
	return Spreadsheet{ 3, 2 };
}

int main()
{
	std::vector<Spreadsheet> vec;
	for (std::size_t i{ 0 }; i < 4; ++i)
	{
		std::println("Iteration: {}", i);
		vec.push_back(Spreadsheet{ 100, 100 });
		std::println("");
	}

	std::println("{:=>40}", "");

	Spreadsheet s{ 2, 3 };
	s = createObject();

	std::println("");

	Spreadsheet s2{ 5, 6 };
	s2 = s;

	return 0;
}
```

输出：

```
Iteration: 0
Normal constructor
Move constructor

Iteration: 1
Normal constructor
Move constructor
Move constructor

Iteration: 2
Normal constructor
Move constructor
Move constructor
Move constructor

Iteration: 3
Normal constructor
Move constructor
Move constructor
Move constructor
Move constructor

========================================
Normal constructor
Normal constructor
Move assignment operator
Move constructor

Normal constructor
Copy assignment operator
Normal constructor
copy constructor
```

## `Spreadsheet`类进阶三

### 进阶三知识点整理

#### 特殊的成员函数

可以根据`const`重载成员函数。也就是说，可以编写两个具有相同名称和参数的成员函数，其中一个声明为`const`，另一个声明为非`const`。如果有一个`const`对象，编译器就会调用`const`成员函数；如果有一个非`const`对象，编译器就会调用非`const`重载。

可以明确指定某个成员函数可以在哪些实例上调用，无论是临时实例还是非临时实例。这是通过在成员函数后添加 **引用限定符**（ref-qualifier）来实现的。如果一个成员函数只能在非临时实例上调用，则在成员函数头部后添加`&`限定符。类似地，如果一个成员函数只能在临时实例上调用，则添加`&&`限定符。

内联(`inline`)函数的定义应该在每个调用它们的源文件中可用。这是有道理的：如果编译器看不到函数定义，它如何替换函数体呢？因此，如果你编写内联成员函数，应该将这些成员函数的定义放在与成员函数所属的类的定义相同的文件中。高级C++编译器不要求你将内联成员函数的定义放在与类定义相同的文件中。例如，Microsoft Visual C++支持链接时代码生成（LTCG），它可以自动内联小函数体，即使它们没有被声明为内联，即使它们没有在与类定义相同的文件中定义。GCC和Clang也有类似的功能。

在C++ module之外，如果成员函数的定义直接放在类定义中，那么该成员函数隐式地被标记为内联，即使没有使用`inline`关键字。对于从模块导出的类(classes exported from modules)来说，情况并非如此。如果希望这些成员函数是内联的，需要用`inline`关键字标记它们。

为所有参数设置了默认值的构造函数可以作为默认构造函数使用。也就是说这时可以在不指定任何参数的情况下构造一个该类的对象。如果同时声明缺省构造函数和为所有参数设置了缺省值的多参数构造函数，编译器会报错，因为如果不指定任何参数，它就不知道该调用哪个构造函数。

#### `constexpr`

声明一个函数为`constexpr`会对该函数可以执行的操作施加限制，因为编译器需要能够在编译时评估该函数。例如，`constexpr`函数不允许有任何副作用，也不能让任何异常逃离函数。在函数内部的`try`块中抛出异常并捕获它们是允许的。`constexpr`函数允许无条件地调用其他`constexpr`函数。它也允许调用非`constexpr`函数，但只有当这些调用是在运行时评估期间 (evaluation at runtime) 触发的，而不是在常量评估 (constant evaluation) 期间。

`constexpr`关键字指定一个函数可以在编译时执行，但它不保证编译时执行。如果真的想要保证一个函数总是在编译时评估，则需要使用`consteval`关键字将一个函数变为 **立即函数** (immediate function)。立即函数只能在常量评估期间被调用。这种立即函数可以从`constexpr`函数中被调用，但仅当 `constexpr`函数是在常量评估期间执行时。

编译器生成的成员函数（无论是隐式生成的还是使用`=default`显式指定的），如默认构造函数、析构函数、赋值运算符等，会自动被视为`constexpr`，除非类包含的数据成员使这些成员函数无法成为`constexpr`。`constexpr`和`consteval`成员函数的定义必须对编译器可用，以便它们能在编译时进行评估。这意味着，如果类在模块(module)中定义，这些成员函数必须在模块接口文件(module interface file)中定义，而不是在模块实现文件(module implementation file)中。这种要求确保所有相关的编译时评估都能在编译过程中正确地处理，符合C++模块的设计原则，即清晰地区分接口和实现，同时确保接口中提供所有必要的信息以支持编译时的各种操作。

#### 静态成员

与普通变量和数据成员不同，静态数据成员默认初始化为`0`。静态指针的初始化值为`nullptr`。可以将静态数据成员声明为内联(`inline`)，这样做的好处是无需在源文件中为它们分配空间。

类中的数据成员可以声明为`const`或`constexpr`，这意味着它们在创建和初始化后不能更改。当常量只适用于类 (也称为 *类常量* (class constants) )时，应使用`static constexpr`（或`constexpr static`）数据成员来代替全局常量。

#### 运算符重载

在C++中，不能更改运算符的优先级。例如，`*`和`/`总是在`+`和`-`之前执行。用户定义的运算符唯一能做的就是在确定了运算的优先级后指定执行方式。C++也不允许发明新的运算符符号或更改运算符的参数数量。

从C++20开始，在类中添加对比较运算符的支持简化了很多。首先，现在建议将`operator==`作为类的成员函数而不是全局函数来实现。还要注意的是，最好添加`[[nodiscard]]`属性，这样运算符的结果就不会被忽略。接下来，要实现对全套比较运算符的支持，只需实现一个额外的重载运算符`operator<=>`。一旦类中有了`operator==`和`<=>`的重载，编译器就会自动提供对所有六个比较运算符(`>, <, >=, <=, ==, !=`)的支持。

在实现`SpreadsheetCell`的 `operator==` 和 `<=>` 时，它们只是简单地比较所有数据成员。在这种情况下，可以进一步减少所需的代码行数，因为从C++20（及以后版本）开始，编译器可以为我们编写这些运算符。就像可以显式默认拷贝构造函数一样，`operator==`和`<=>`也可以被默认，这种情况下编译器将为我们编写它们，并通过按照类定义中声明的顺序依次比较每个数据成员来实现它们，这也被称为 **成员逐个字典式比较** (member-wise lexicographical comparison)。

此外，如果仅显式默认`operator<=>`，编译器还会自动包括一个默认的`operator==`。因此，对于没有显式`operator==`和`<=>`用于双精度浮点数的 SpreadsheetCell 版本，我们只需要写一行代码就可以为比较两个`SpreadsheetCell`添加对所有六个比较运算符的完全支持。

```C++
[[nodiscard]] auto operator<=>(const SpreadsheetCell&) const = default;
```

如果类的某些数据成员没有可访问的`operator==`，则该类的默认`operator==`将被隐式删除。如果类有不支持`operator<=>`的数据成员，一个默认的 `operator<=>` 会退回使用这些数据成员的 `operator<` 和 `==`。在这种情况下，返回类型推导将不起作用，需要显式指定返回类型为 `strong_ordering`、`partial_ordering` 或 `weak_ordering`。如果数据成员甚至没有可访问的`operator<`和`==`，那么默认的`operator<=>`也会被隐式删除。

总结一下，为了使编译器能够写一个默认的`<=>`运算符，类的所有数据成员都需要支持`operator<=>`，在这种情况下返回类型可以是`auto`；或者支持`operator<`和`==`，在这种情况下返回类型不能是`auto`。由于`SpreadsheetCell`有一个单一的`double`作为数据成员，编译器推导返回类型为 `partial_ordering`。

#### `SpreadsheetCell.cppm`修改一

```C++
export module spreadsheet_cell;

import std;

export class SpreadsheetCell
{
public:
	SpreadsheetCell() = default;
	SpreadsheetCell(double initialValue);
	SpreadsheetCell(std::string_view initialValue);

	void set(double value);
	void set(std::string_view value);

	double getValue() const;
	std::string getString() const;

	SpreadsheetCell& operator+=(const SpreadsheetCell& rhs);
	SpreadsheetCell& operator-=(const SpreadsheetCell& rhs);
	SpreadsheetCell& operator*=(const SpreadsheetCell& rhs);
	SpreadsheetCell& operator/=(const SpreadsheetCell& rhs);

	/*[[nodiscard]] bool operator==(const SpreadsheetCell& rhs) const;
	[[nodiscard]] std::partial_ordering operator<=>(const SpreadsheetCell& rhs) const;

	[[nodiscard]] bool operator==(double rhs) const;
	[[nodiscard]] std::partial_ordering operator<=>(double rhs) const;*/

	[[nodiscard]] auto operator<=>(const SpreadsheetCell&) const = default;

private:
	static std::string doubleToString(double value);
	static double stringToDouble(std::string_view value);

	double m_value{ 0 };
};

export SpreadsheetCell operator+(const SpreadsheetCell& lhs, const SpreadsheetCell& rhs);
export SpreadsheetCell operator-(const SpreadsheetCell& lhs, const SpreadsheetCell& rhs);
export SpreadsheetCell operator*(const SpreadsheetCell& lhs, const SpreadsheetCell& rhs);
export SpreadsheetCell operator/(const SpreadsheetCell& lhs, const SpreadsheetCell& rhs);
```

#### `SpreadsheetCell.cpp`修改一

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

void SpreadsheetCell::set(double value)
{
	m_value = value;
}

double SpreadsheetCell::getValue() const
{
	return m_value;
}

void SpreadsheetCell::set(std::string_view value)
{
	m_value = stringToDouble(value);
}

std::string SpreadsheetCell::getString() const
{
	return doubleToString(m_value);
}

std::string SpreadsheetCell::doubleToString(double value)
{
	return std::to_string(value);
}

double SpreadsheetCell::stringToDouble(std::string_view value)
{
	double number{ 0 };
	std::from_chars(value.data(), value.data() + value.size(), number);
	return number;
}


SpreadsheetCell& SpreadsheetCell::operator+=(const SpreadsheetCell& rhs)
{
	set(getValue() + rhs.getValue());
	return *this;
}

SpreadsheetCell& SpreadsheetCell::operator-=(const SpreadsheetCell& rhs)
{
	set(getValue() - rhs.getValue());
	return *this;
}

SpreadsheetCell& SpreadsheetCell::operator*=(const SpreadsheetCell& rhs)
{
	set(getValue() * rhs.getValue());
	return *this;
}

SpreadsheetCell& SpreadsheetCell::operator/=(const SpreadsheetCell& rhs)
{
	if (rhs.getValue() == 0)
	{
		throw std::invalid_argument{ "Divide by zero!!" };
	}
	set(getValue() / rhs.getValue());
	return *this;
}


SpreadsheetCell operator+(const SpreadsheetCell& lhs, const SpreadsheetCell& rhs)
{
	auto result{ lhs };
	result += rhs;
	return result;
}

SpreadsheetCell operator-(const SpreadsheetCell& lhs, const SpreadsheetCell& rhs)
{
	auto result{ lhs };
	result -= rhs;
	return result;
}

SpreadsheetCell operator*(const SpreadsheetCell& lhs, const SpreadsheetCell& rhs)
{
	auto result{ lhs };
	result *= rhs;
	return result;
}

SpreadsheetCell operator/(const SpreadsheetCell& lhs, const SpreadsheetCell& rhs)
{
	auto result{ lhs };
	result /= rhs;
	return result;
}


//bool SpreadsheetCell::operator==(const SpreadsheetCell& rhs) const
//{
//	return (getValue() == rhs.getValue());
//}
//
//std::partial_ordering SpreadsheetCell::operator<=>(const SpreadsheetCell& rhs) const
//{
//	return getValue() <=> rhs.getValue();
//}
//
//bool SpreadsheetCell::operator==(double rhs) const
//{
//	return getValue() == rhs;
//}
//
//std::partial_ordering SpreadsheetCell::operator<=>(double rhs) const
//{
//	return getValue() <=> rhs;
//}
```

#### `Spreadsheet.cppm`修改三

```C++
export module spreadsheet;

export import spreadsheet_cell;

import std;

export class Spreadsheet
{
public:
	Spreadsheet(std::size_t width, std::size_t height);
	Spreadsheet(const Spreadsheet& src);
	Spreadsheet(Spreadsheet&& src) noexcept;  // Move constructor
	~Spreadsheet();

	Spreadsheet& operator=(const Spreadsheet& rhs);
	Spreadsheet& operator=(Spreadsheet&& rhs) noexcept;  // Move assignment

	void setCellAt(std::size_t x, std::size_t y, const SpreadsheetCell& cell);
	SpreadsheetCell& getCellAt(std::size_t x, std::size_t y);
	const SpreadsheetCell& getCellAt(std::size_t x, std::size_t y) const;

	void swap(Spreadsheet& other) noexcept;

	std::size_t getId() const;

	static constexpr std::size_t MaxHeight{ 100 };
	static constexpr std::size_t MaxWidth{ 100 };

private:
	void verifyCoordinate(std::size_t x, std::size_t y) const;

	const std::size_t m_id{ 0 };

	std::size_t m_width{ 0 };
	std::size_t m_height{ 0 };
	SpreadsheetCell** m_cells{ nullptr };

	static inline std::size_t ms_counter{ 0 };
};

export void swap(Spreadsheet& first, Spreadsheet& second) noexcept;
```

#### `Spreadsheet.cpp`修改三

```C++
module spreadsheet;

import std;

Spreadsheet::Spreadsheet(std::size_t width, std::size_t height)
	: m_id{ ms_counter++ }, m_width { width }, m_height{ height }
{
	std::println("Normal constructor with id: {}", m_id);

	m_cells = new SpreadsheetCell * [m_width];
	for (std::size_t i{ 0 }; i < m_width; ++i)
	{
		m_cells[i] = new SpreadsheetCell[m_height];
	}
}

Spreadsheet::~Spreadsheet()
{
	for (std::size_t i{ 0 }; i < m_width; ++i)
	{
		delete[] m_cells[i];
	}
	delete[] m_cells;
	m_cells = nullptr;
}

Spreadsheet::Spreadsheet(const Spreadsheet& src)
	: Spreadsheet{ src.m_width, src.m_height }
{
	std::println("copy constructor");
	
	for (std::size_t i{ 0 }; i < m_width; ++i)
	{
		for (size_t j{ 0 }; j < m_height; ++j)
		{
			m_cells[i][j] = src.m_cells[i][j];
		}
	}
}


void Spreadsheet::verifyCoordinate(std::size_t x, std::size_t y) const
{
	if (x >= m_width)
	{
		throw std::out_of_range{ std::format("x ({}) must be less than width ({}).", x, m_width) };
	}
	if (y >= m_height) 
	{
		throw std::out_of_range{ std::format("y ({}) must be less than height ({}).", y, m_height) };
	}
}

void Spreadsheet::setCellAt(std::size_t x, std::size_t y, const SpreadsheetCell& cell)
{
	verifyCoordinate(x, y);
	m_cells[x][y] = cell;
}

const SpreadsheetCell& Spreadsheet::getCellAt(std::size_t x, std::size_t y) const
{
	std::println("constant overload");
	verifyCoordinate(x, y);
	return m_cells[x][y];
}


SpreadsheetCell& Spreadsheet::getCellAt(std::size_t x, std::size_t y)
{
	std::println("non-constant overload");
	return const_cast<SpreadsheetCell&>(std::as_const(*this).getCellAt(x, y));
}

void Spreadsheet::swap(Spreadsheet& other) noexcept
{
	std::swap(m_width, other.m_width);
	std::swap(m_height, other.m_height);
	std::swap(m_cells, other.m_cells);
}

void swap(Spreadsheet& first, Spreadsheet& second) noexcept
{
	first.swap(second);
}

// copy and swap idiom
Spreadsheet& Spreadsheet::operator=(const Spreadsheet& rhs)
{
	std::println("Copy assignment operator");

	Spreadsheet temp{ rhs };
	swap(temp);
	return *this;
}

std::size_t Spreadsheet::getId() const
{
	return m_id;
}

// Move constructor
Spreadsheet::Spreadsheet(Spreadsheet&& src) noexcept
{
	std::println("Move constructor");

	swap(src);
}

// Move assignment operator
Spreadsheet& Spreadsheet::operator=(Spreadsheet&& rhs) noexcept
{
	std::println("Move assignment operator");
	
	auto moved{ std::move(rhs) };  // Move rhs into moved (noexcept)
	swap(moved);  // Commit the work with only non-throwing operations

	return *this;
}
```

#### 进阶三的测试程序代码

```C++
import spreadsheet;
import std;

int main()
{
	Spreadsheet sheet1{ 5, 6 };
	SpreadsheetCell& cell1{ sheet1.getCellAt(1, 1) };

	const Spreadsheet sheet2{ 5, 6 };
	const SpreadsheetCell& cell2{ sheet2.getCellAt(1, 1) };

	{
		SpreadsheetCell myCell{ 4 }, anotherCell{ 5 };
		SpreadsheetCell aThirdCell{ myCell + anotherCell };

		std::string str{ "hello" };
		aThirdCell = myCell + std::string_view{ str };

		aThirdCell = myCell + 5.6;
		aThirdCell = myCell + 4;

		aThirdCell = 5.6 + myCell; // works fine
		aThirdCell = 4 + myCell;   // works fine
		aThirdCell = 4.5 + 5.5;

		aThirdCell = myCell - anotherCell;
		aThirdCell = myCell * anotherCell;
		aThirdCell = myCell / anotherCell;

		try 
		{
			aThirdCell = myCell / 0;
		}
		catch (const std::invalid_argument & e) 
		{
			std::println("Caught: {}", e.what());
		}

		aThirdCell -= myCell;
		aThirdCell += 5.4;
		aThirdCell *= myCell;
		aThirdCell /= myCell;


		if (myCell > aThirdCell || myCell < 10) 
		{
			std::println("{}", myCell.getValue());
		}

		if (myCell == 10) { std::println("myCell == 10"); }
		if (10 == myCell) { std::println("10 == myCell"); }

		if (myCell < aThirdCell) { std::println("myCell < aThirdCell"); }
		if (aThirdCell < myCell) { std::println("aThirdCell < myCell"); }

		if (myCell <= aThirdCell) { std::println("myCell <= aThirdCell"); }
		if (aThirdCell <= myCell) { std::println("aThirdCell <= myCell"); }

		if (myCell > aThirdCell) { std::println("myCell> aThirdCell"); }
		if (aThirdCell > myCell) { std::println("aThirdCell> myCell"); }

		if (myCell >= aThirdCell) { std::println("myCell>= aThirdCell"); }
		if (aThirdCell >= myCell) { std::println("aThirdCell>= myCell"); }

		if (myCell == aThirdCell) { std::println("myCell == aThirdCell"); }
		if (aThirdCell == myCell) { std::println("aThirdCell == myCell"); }

		if (myCell != aThirdCell) { std::println("myCell != aThirdCell"); }
		if (aThirdCell != myCell) { std::println("aThirdCell != myCell"); }

		if (myCell < 10) { std::println("myCell < 10"); }
		if (10 < myCell) { std::println("10 < myCell"); }
		if (10 != myCell) { std::println("10 != myCell"); }

		if (anotherCell == myCell) 
		{
			std::println("cells are equal");
		}
		else 
		{
			std::println("cells are not equal");
		}
	}

	return 0;
}
```
