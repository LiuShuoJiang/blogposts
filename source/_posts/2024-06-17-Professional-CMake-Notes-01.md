---
title: Professional CMake读书笔记之一
date: 2024-06-17 19:21:58
updated: 2024-06-18 22:21:58
tags: [CMake, C++]
categories:
  - [CMake]
keywords:
description: 本文是对Professional CMake第十八版第一部分内容的读书笔记。
top_img:
---

## 介绍

CMake的执行全过程：

![CMake Process](/blogposts/img/Professional-CMake/Cmake-Components.png)

## 设置项目

CMake项目的构成：

- **Source directory**
- **Build directory**

CMake构建的方式：

- **In-source builds** (不建议)
- **Out-of-source builds** (建议)

开发人员通过选择特定的 **Project file generator** 来选择要创建的项目文件类型，如UNIX Makefile、Ninja或Visual Studio。对于不支持多种配置的生成器，必须重新运行CMake以在Debug、Release等之间切换构建。

CMake生成项目文件的两个阶段：

- **Configuring phase**
- **Generating phase**

当CMake完成运行后，它将在构建目录中保存一个CMakeCache.txt文件。CMake使用此文件保存详细信息，以便在再次运行时可以重新使用第一次计算的信息并加快项目生成速度。

CMake可以代表开发人员调用构建工具：`cmake --build /pathTo/build --config Debug --target MyApp`。

## 最小的项目

在CMake中，命令(commands)与其他语言的函数调用类似，不同之处在于它们虽然支持参数，但不直接返回值。

CMake provides **policy** mechanisms which allow the project to say “Behave like CMake version X.Y.Z”. This allows CMake to fix bugs internally and introduce new features, but still maintain the expected behavior of any particular past release.

对`cmake_minimum_required`，除了指定最低的CMake版本，会强制执行策略设置(policy settings)，使CMake行为与指定版本相匹配。

```cmake
project(projectName
  [VERSION major[.minor[.patch[.tweak]]]]
  [LANGUAGES languageName ...]
)
```

如果没有提供`LANGUAGES`选项，CMake将默认使用`C`和`CXX`。`project()`命令的作用远不止填充几个变量。它的重要职责之一是检查每种启用语言的编译器，确保它们能够成功编译和链接。这样就能及早发现编译器和链接器设置方面的问题。一旦这些检查通过，CMake就会设置一系列变量和属性，以控制启用语言的编译。

```cmake
add_executable(targetName source1 [source2 ...])
```

通过多次调用`add_executable()`，并使用不同的target名称，也可以在一个CMakeLists.txt文件中定义多个可执行文件。如果在多个`add_executable()`命令中使用相同的目标名称，CMake会失败并高亮显示错误。

## 构建简单的Target

```cmake
add_executable(targetName [WIN32] [MACOSX_BUNDLE]
  [EXCLUDE_FROM_ALL]
  source1 [source2 ...]
)
```

如果使用`EXCLUDE_FROM_ALL`选项定义了可执行文件，则默认`ALL` target文件中将不包含该文件。只有在编译命令明确要求，或可执行文件是另一个默认`ALL` target文件的依赖文件时，才会编译该可执行文件。

```cmake
add_library(targetName [STATIC | SHARED | MODULE]
  [EXCLUDE_FROM_ALL]
  source1 [source2 ...]
)
```

可以省略定义要构建的库类型的关键字。除非项目特别需要特定类型的库，否则首选的做法是不指定，让开发者在构建项目时自行选择。在这种情况下，库将是`STATIC`或`SHARED`，由CMake变量`BUILD_SHARED_LIBS`的值决定。

选择构建`SHARED`库可以使用如下命令：`cmake -DBUILD_SHARED_LIBS=YES /path/to/source`，也可以通过`set(BUILD_SHARED_LIBS YES)`写在CMakeLists.txt文件中。

库之间可能存在几种不同类型的 **依赖关系**：

- **`PRIVATE`**：指定库A在其内部实现中使用库B。链接到A库的其他任何东西都不需要知道B，因为它是A的内部实现细节。
- **`PUBLIC`**：规定库A不仅在内部使用库B，还在其接口中使用B。例如，在库A中定义的函数至少有一个参数的类型是在库B中定义和实现的，因此如果不提供一个类型来自B的参数，代码就无法从A中调用该函数。
- **`INTERFACE`**：规定要使用库A，还必须使用库B的部分内容。这与公共依赖关系的不同之处在于，库A内部并不需要B，它只是在接口中使用了B。在处理使用`add_library()`的`INTERFACE`形式定义的library targets时，例如使用target来表示header-only库的依赖关系时，这就是一个有用的例子。

```cmake
target_link_libraries(targetName
  <PRIVATE|PUBLIC|INTERFACE> item1 [item2 ...]
  [<PRIVATE|PUBLIC|INTERFACE> item3 [item4 ...]]
  ...
)
```

后面还将讨论如何使用更复杂的源代码目录层次结构。在这种情况下，如果使用的是CMake 3.12或更早版本，使用`target_link_libraries()`的`targetName`必须是在调用`target_link_libraries()`的同一目录中通过`add_executable()`或`add_library()`命令定义的(这一限制在CMake 3.13中被取消)。

以下内容也可以指定为`target_link_libraries()`命令中的item：

- **Full path to a library file**;
- **Plain library name**：如果只给出库的名称而没有路径，则linker命令将搜索该库(例如，`foo`变为`-lfoo`或`foo.lib`，具体取决于平台)；
- **Link flag**：作为一种特殊情况，除了`-l`或`-framework`之外，以连字号`-`开头的项目将被视为添加到链接器命令中的标志。CMake文档警告说，这些标志只能用于`PRIVATE`项目，因为如果将它们定义为`PUBLIC`或`INTERFACE`，它们就会被带到其他目标，而这并不总是安全的。

建议：

- 在为Library Target命名时，尽量不以`lib`开头或结尾；
- Project名称不必与Target名称相关。虽然它们有时相同，但两者是不同的概念。应该直接设置Project名称，而不是通过变量。根据Target的功能而不是其所属的Project来选择Target名称；
- 除非有充分的理由，否则应避免为库指定`STATIC`或`SHARED`关键字，直到知道需要它为止；
- 在调用`target_link_libraries()`命令时，应始终指定`PRIVATE`、`PUBLIC`和/或`INTERFACE`关键字，而不是使用旧式的CMake语法，即假定所有内容都是`PUBLIC`。

## 基本的测试与部署

例如：

```cmake
cmake_minimum_required(VERSION 3.19)
project(MyProj VERSION 4.7.2)

enable_testing()

add_executable(testSomething testSomething.cpp)

add_test(NAME SomethingWorks COMMAND testSomething)
add_test(NAME ExternalTool COMMAND /path/to/tool someArg moreArg)
```

`COMMAND`可以是任何可在shell或命令提示符下运行的任意命令。作为一种特殊情况，它也可以是项目定义的可执行Target的名称。

可使用`ctest --parallel 16`来并行执行测试。

```cmake
cmake_minimum_required(VERSION 3.14)
project(MyProj VERSION 4.7.2)

add_executable(MyApp ...)
add_library(AlgoRuntime SHARED ...)
add_library(AlgoSDK STATIC ...)

# This concise form requires CMake 3.14 or later
install(TARGETS MyApp AlgoRuntime AlgoSDK)
```

如果没有指定目标安装位置，CMake将使用默认位置，这与大多数Unix系统使用的习惯一致。CMake的默认布局会将可执行文件安装到基本安装位置下面的`bin`子目录，将库安装到`lib`子目录，将头文件安装到`include`子目录。在Windows系统中，DLL库将被安装到`bin`目录，而不是`lib`目录。

```cmake
install(FILES things.h algo.h DESTINATION include/myproj)
install(DIRECTORY headers/myproj DESTINATION include)
# 以下命令与上一行等效
# install(DIRECTORY headers/ DESTINATION include/myproj)
```

`install(FILES)`形式要求单独列出每个文件。当一个目录中只有部分文件需要安装时，这种方法非常有用。`install(DIRECTORY)`会递归地将指定目录复制到目的地。如果要复制目录的 *内容* 而不是目录本身，应当在目录名后加上`/`。

在CMake 3.23或更高版本中，一种更强大、更方便的头文件处理方法是使用 **file sets**。File sets可将头文件与target关联起来，头文件可在`install(TARGETS)`调用中与target一起安装。File sets由`target_sources()`命令定义。具体用法暂略。

以下是CMake的配置、编译和安装的常见命令行指令：

```bash
mkdir build
cd build
cmake -G Ninja -DCMAKE_BUILD_TYPE=Debug ../source
cmake --build .
cmake --install . --prefix /path/to/somewhere
```

CMake提供的 **cpack** 工具可以生成各种格式的二进制包。其中包括`.zip`、`.tar.gz`和`.7z`等简单压缩包，`RPM`、`DEB`和`MSI`等特定平台打包系统的包，甚至还有独立的图形安装包。这些都是基于使用`install()`命令和其他命令提供的信息来安装项目。cpack在原理上可以有效地执行一个或多个`cmake --install`命令，并将`--prefix`设置为临时暂存区域。然后，暂存区域的内容会被用来创建相关格式的软件包。使用`cpack -G "ZIP;WIX"`命令构建二进制包。具体用法暂略。


