---
title: Professional CMake读书笔记之二
date: 2024-06-18 21:25:48
updated: 2024-06-18 21:25:48
tags: [CMake, C++]
categories:
  - [CMake]
keywords:
description: 本文是对Professional CMake第十八版第二部分内容的读书笔记(之一)。
top_img:
---

## 变量基础

设置变量的常见命令：

```cmake
set(varName value... [PARENT_SCOPE])
```

与其他语言相比，CMake中的变量作用域更为灵活，目前可暂时将变量的作用域视为定义它的所在文件。

CMake将所有变量都视为 **字符串**。在不同的上下文中，变量可能被解释为不同的类型，但归根结底，它们只是字符串。在设置变量值时，CMake并不要求变量值必须加引号，除非变量值包含空格。如果给定了多个值，这些值将被连接在一起，每个值之间用分号隔开————由此产生的字符串就是CMake表示lists的方式。

变量的值可以用`${myVar}`来获取，它可以在任何需要字符串或变量的地方使用。CMake并不要求在使用变量前对其进行定义。使用未定义的变量只会导致空字符串被替换，这与Unix shell脚本的行为方式类似。

```cmake
set(foo ab) # foo = "ab"
set(bar ${foo}cd) # bar = "abcd"
set(baz ${foo} cd) # baz = "ab;cd"
set(myVar ba) # myVar = "ba"
set(big "${${myVar}r}ef") # big = "${bar}ef" = "abcdef"
set(${foo} xyz) # ab = "xyz"
set(bar ${notSetVar}) # bar = ""
```

如果使用CMake 3.0或更高版本，引号的另一种替代方法是使用受lua启发的括号语法，即内容的开始用`[=[`标记，结束用`]=]`标记。方括号之间可以出现任意数量的`=`字符，也可以一个都不出现，但开头和结尾必须使用相同数量的`=`字符。

```cmake
# Bracket syntax prevents unwanted substitution
set(shellScript [=[
#!/bin/bash
[[ -n "${USER}" ]] && echo "Have USER"
]=])

# Equivalent code without bracket syntax
set(shellScript
"#!/bin/bash
[[ -n \"\${USER}\" ]] && echo \"Have USER\"
")
```

设置环境变量的方式与设置CMake变量的方式类似，不同之处在于使用`ENV{varName}`而不是仅使用`varName`作为要设置的变量。例如：

```cmake
set(ENV{PATH} "$ENV{PATH}:/opt/myDir")
```

设置这样的环境变量只会影响当前正在运行的CMake实例。CMake运行完成后，对环境变量的更改将丢失，并且对环境变量的更改在build时将不可见。

CMake 还支持 **Cache变量**。普通变量的生命周期仅限于CMakeLists.txt文件的处理过程，而Cache变量则不同，它们存储在构建目录中名为CMakeCache.txt的特殊文件中，并在CMake运行之间持续存在。Cache变量一旦被设置，就会一直保留，直到被明确从缓存中移除。

`set()`命令也用于设置Cache变量，但需要额外的参数并且行为更复杂：

```cmake
set(varName value... CACHE type "docstring" [FORCE])
```

当出现`CACHE`关键字时，`set()`命令将应用于名为`varName`的Cache变量，而不是普通变量。普通变量和Cache变量之间的一个重要区别是，只有当存在`FORCE`关键字时，`set()`命令才会覆盖Cache变量。而对于普通变量而言，`set()`命令将始终覆盖已存在的值。

Cache变量比普通变量附加了更多信息，包括名义类型 ( **nominal type** ) 和文档字符串 ( **documentation string** )。设置Cache变量时必须提供这两项信息。


