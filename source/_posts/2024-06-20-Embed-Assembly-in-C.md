---
title: 在C中嵌入汇编
date: 2024-06-20 22:29:33
updated: 2024-06-20 22:29:33
tags: [C, RISC-V]
categories:
  - [System]
keywords:
description: GCC在C语言中嵌入汇编的简要语法
top_img:
---

## 语法(GCC)

```C
asm [volatile] (
    "汇编指令"
    : 输出操作数列表(可选)
    : 输入操作数列表(可选)
    : 可能影响的寄存器或者存储器(可选)
);
```

- 汇编指令用双引号括起来，多条指令之间用`";"`或者`"\n"`分隔；
- "输出操作数列表"和"输入操作数列表"用于将需要操作的C变量和汇编指令的操作数对应起来，多个操作数之间用`","`分隔；
- "可能影响的寄存器或者存储器"用于告知编译器当前嵌入的汇编语句可能修改的寄存器或者内存，方便编译器执行优化。

## 示例(RISC-V)

```C
int foo(int a, int b) {
    int c;

    asm volatile (
        "add %[sum], %[add1], %[add2]"
        : [sum]"=r"(c)
        : [add1]"r"(a), [add2]"r"(b)
    );

    return c;
}
```

另一种写法：

```C
int foo(int a, int b) {
    int c;

    asm volatile (
        "add %0, %1, %2"
        : "=r"(c)
        : "r"(a), "r"(b)
    );

    return c;
}
```

[xv6](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/riscv.h)中的示例：

```C++
// Supervisor Trap-Vector Base Address
// low two bits are mode.
static inline void w_stvec(uint64 x) {
    asm volatile (
        "csrw stvec, %0"
        :
        : "r" (x)
    );
}

static inline uint64 r_stvec() {
    uint64 x;

    asm volatile (
        "csrr %0, stvec"
        : "=r" (x)
    );

    return x;
}
```
