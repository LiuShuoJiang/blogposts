---
title: RISC-V Cheatsheet之一
date: 2024-05-09 21:17:21
updated: 2024-05-09 21:17:21
tags: [RISC-V]
categories:
  - [System]
keywords:
description: 本文记录一些RISC-V的知识点。
top_img:
---

本文参考自[RVOS](https://github.com/plctlab/riscv-operating-system-mooc)的讲义。

## 函数调用中有关寄存器的编程约定

| 寄存器名 | ABI(Application Binary Interface)名(编程用名) | 用途约定 | 谁负责在函数调用过程中维护这些寄存器 |
| :---: | :---: | --- | :---: |
| `x0` | `zero` | 读取时总为`0`， 写入时不起任何效果 | 不适用 |
| `x1` | `ra` | 存放 **函数返回地址** (return address) | Caller |
| `x2` | `sp` | 存放 **栈指针** (stack pointer) | Callee |
| `x5~x7, x28~x31` | `t0~t2, t3~t6` | **临时(temporary)寄存器**，Callee可能会使用这些寄存器，所以Callee不保证这些寄存器中的值在函数调用过程中保持不变，这意味着对于Caller来说，如果需要的话，Caller需要自己在调用Callee之前保存临时寄存器中的值 | Caller |
| `x8~x9, x18~x27` | `s0~s1, s2~s11` | **保存(saved)寄存器**，Callee需要保证这些寄存器的值在函数返回后仍然维持函数调用之前的原值，所以一旦Callee在自己的函数中会用到这些寄存器则需要在栈中备份并在退出函数时进行恢复 | Callee |
| `x10, x11` | `a0, a1` | **参数(argument)寄存器**，用于在函数调用过程中保存第一个和第二个参数， *以及在函数返回时传递返回值* | Caller |
| `x12~x17` | `a2~a7` | **参数(argument)寄存器**， 如果函数调用时需要传递更多的参数，则可以用这些寄存器，但注意用于传递参数的寄存器最多只有8个(`a0~a7`)，如果还有更多的参数则要利用栈 | Caller |

## 函数调用过程的函数跳转和返回指令

| 伪指令 | 等价指令 | 描述 | 示例 |
| :---: | :---: | :---: | :---: |
| `jal offset` | `jal x1, offset` | 跳转到`offset`指定位置，返回地址保存在`x1(ra)` | `jal foo` |
| `jalr rs` | `jalr x1, 0(rs)` | 跳转到`rs`中的值所指定的位置，返回地址保存在`x1(ra)` | `jalr s1` |
| `j offset` | `jal x0, offset` | 跳转到`offset`指定位置， *不保存返回地址* | `j loop` |
| `jr rs` | `jalr x0, 0(rs)` | 跳转到`rs`中值所指定的位置， *不保存返回地址* | `jr, s1` |
| **`call offset`** | `auipc x1, offset[31 : 12] + offset[11]` <br> `jalr x1, offset[11:0](x1)` | 长跳转调用函数 | `call foo` |
| `tail offset` | `auipc x6, offset[31 : 12] + offset[11]` <br> `jalr x0, offset[11:0](x6)` | 长跳转 *尾调用* | `tail foo` |
| **`ret`** | `jalr x0, 0(x1)` | 从Callee返回 | `ret` |

## 函数调用的常见程序

函数调用的一些标准操作(由编译器实现)：

1. **函数起始部分** (Prologue)：
   1. 减少`sp`的值，根据本函数中使用saved寄存器的情况以及local变量的多少开辟栈空间；
   2. 将saved寄存器的值保存到栈中；
   3. 如果函数中还会调用其他的函数，则将`ra`寄存器的值保存到栈中。
2. ***函数执行体***；
3. **函数退出部分** (Epilogue)：
   1. 从栈中恢复saved寄存器；
   2. 如果需要的话，从栈中恢复`ra`寄存器；
   3. 增加`sp`的值，恢复到进入本函数之前的状态；
   4. 调用`ret`返回。

示例一：

```nasm
# Calling Convention
# Demo to create a leaf routine
#
# void _start()
# {
#     // calling leaf routine
#     square(3);
# }
#
# int square(int num)
# {
#     return num * num;
# }

	.text			# Define beginning of text section
	.global	_start		# Define entry _start

_start:
	# 注意：栈由高地址向低地址增长
	la sp, stack_end	# prepare stack for calling functions

	li a0, 3
	call square

	# the time return here, a0 should stores the result
stop:
	j stop			# Infinite loop to stop execution

# int square(int num)
square:
	# prologue
	addi sp, sp, -8
	sw s0, 0(sp)
	sw s1, 4(sp)

	# "mul a0, a0, a0" should be fine,
	# programing as below just to demo we can contine use the stack
	mv s0, a0
	mul s1, s0, s0
	mv a0, s1  # 通过a0传返回值

	# epilogue
	lw s0, 0(sp)
	lw s1, 4(sp)
	addi sp, sp, 8

	ret

	# add nop here just for demo in gdb
	nop

	# allocate stack space
stack_start:
	.rept 12
	.word 0
	.endr
stack_end:

	.end			# End of file
```

示例二(嵌套函数)：

```nasm
# Calling Convention
# Demo how to write nested routines
#
# void _start()
# {
#     // calling nested routine
#     aa_bb(3, 4);
# }
#
# int aa_bb(int a, int b)
# {
#     return square(a) + square(b);
# }
#
# int square(int num)
# {
#     return num * num;
# }

	.text			# Define beginning of text section
	.global	_start		# Define entry _start

_start:
	la sp, stack_end	# prepare stack for calling functions

	# aa_bb(3, 4);
	li a0, 3
	li a1, 4
	call aa_bb

stop:
	j stop			# Infinite loop to stop execution

# int aa_bb(int a, int b)
# return a^2 + b^2
aa_bb:
	# prologue
	addi sp, sp, -16
	sw s0, 0(sp)
	sw s1, 4(sp)
	sw s2, 8(sp)
	sw ra, 12(sp)  # 嵌套函数要保存ra!!!

	# cp and store the input params
	mv s0, a0
	mv s1, a1

	# sum will be stored in s2 and is initialized as zero
	li s2, 0

	mv a0, s0
	jal square
	add s2, s2, a0

	mv a0, s1
	jal square
	add s2, s2, a0

	mv a0, s2

	# epilogue
	lw s0, 0(sp)
	lw s1, 4(sp)
	lw s2, 8(sp)
	lw ra, 12(sp)
	addi sp, sp, 16
	ret

# int square(int num)
square:
	# prologue
	addi sp, sp, -8
	sw s0, 0(sp)
	sw s1, 4(sp)

	# "mul a0, a0, a0" should be fine,
	# programing as below just to demo we can contine use the stack
	mv s0, a0
	mul s1, s0, s0
	mv a0, s1

	# epilogue
	lw s0, 0(sp)
	lw s1, 4(sp)
	addi sp, sp, 8

	ret

	# add nop here just for demo in gdb
	nop

	# allocate stack space
stack_start:
	.rept 12
	.word 0
	.endr
stack_end:

	.end			# End of file
```
