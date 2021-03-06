title: 《深入理解计算机系统》读书笔记——Chapter 3(2)
author: 天渊
tags:
  - csapp
categories:
  - 读书笔记
date: 2019-10-01 18:25:00
---
第三章：`程序的机器级表示`

第三章读书笔记第二部分，本部分主要了解：

1. 计算机中算术和逻辑操作在机器层面的具体实现
2. 计算机中控制语句（条件和循环，还有switch）的具体实现
3. 计算机中函数执行过程的具体实现，包括运行时栈的实现，局部变量存储以及递归的实现
<!--more-->

### 算术和逻辑运算

算术和逻辑操作包括加减乘除还有移位等操作，这些操作分为`加载有效地址`，`一元操作`（只有一个操作数），`二元操作`（有两个操作数）和`移位操作`，一共四大类，基本上囊括了计算机系统中所有基本操作指令

#### 加载有效地址：leaq

`leaq (load effective address)`操作是一种特殊的运算操作，该操作是`mov`操作的一个变种，但是对于`mov`，像下面这种操作是引用了内存地址，`%rdi`寄存器保存了某个变量的内存地址，`mov`根据这个地址去内存找相应的值然后拷贝到`%rax`寄存器上：

```assembly
movq	(%rdi), %rax
```

而对于`leaq`操作，则是直接将某个内存的地址值拷贝到寄存器或者其他内存，那么目标寄存器或者内存就会保存另外一个内存的指针，比如下面这个操作，`leaq`是直接把`%rdi`保存的值拷贝到`%rax`寄存器，那么`%rax`和`rdi`同时都保存了某个内存地址：

```assembly
leaq	(%rdi), %rax
```

该操作就是C语言中`int a = &b`这个操作的具体实现，将某个变量的指针取出来存到另一个变量中，因此`leaq`操作就叫做`加载有效地址`，需要注意的是`leaq`没有其他不同大小的操作数，在x86-64中只加载8字节数，因为内存地址的大小就是8字节

##### leaq执行有限加法和乘法

由于`leaq`计算内存地址的功能，因此可以基于内存地址计算过程来进行一些简单的加法和乘法操作，某些简单的C语言加减乘操作会被编译为多个`leaq`的汇编指令，比如如下操作：

```c
long t = x + 4 * y + 12 * z
```

会被gcc编译器编译为如下的汇编指令：

```assembly
x in %rdi, y in %rsi, z in %rdx

leaq	(%rdi, %rsi, 4), %rax	// x + 4 * y
leaq	(%rdx, %rdx, 2), %rdx	// z + 2 * z = 3 * z
leaq	(%rax, %rdx, 4), %rax	// (x + 4 * y) + 4 * (3 * z) = x + 4 * y + 12 * z
```

#### 一元操作

一元操作只有一个操作数，既是源又是目的地，操作数可以是寄存器，也可以是内存地址，比如自增操作或者取反操作，是直接改变的当前位置的数据，没有拷贝操作

内存操作，`%rax`存储值为0x100，内存0x110处存储的值为0x13：

```assembly
incq	16(%rax)	//结果为0x13 + 0x1 = 0x14
```

寄存器操作，`%rcx`存储值为0x1：

```assembly
decq	%rcx	//结果为0x0
```

#### 二元操作

像加减乘和异或这样的操作就属于二元操作，二元操作相对来说要复杂一些，不仅有两个操作数，而且操作过程也比较反直觉

在二元操作中，第二个操作数即是源又是目的，比如对于操作`subq %rax,%rdx`，操作过程是将`%rdx`减去`%rax`的值，然后再把结果存入`%rdx`，有点类似于`x-=y`这样的操作，x就是`%rdx`，y就是`%rax`

二元操作中，第一个操作数可以是立即数，第二个操作数不能是立即数，只能是寄存器或者内存地址

内存和寄存器的二元操作，其中`%rax`的值为0x100，`%rdx`的值为0x3，内存0x108处的值为0xAB，求以下操作的结果

```assembly
subq	%rdx, 8(%rax)	//先找到源操作数8(%rax)也就是0x108位置的数0xAB，然后再减去0x3
						//最后结果即为0xA8，将结果再存回0x108
```

#### 移位操作

移位操作中，第一个操作数是移位量，可以是立即数或者单字节寄存器`%c1`

第二个操作数是要移位的数，即是源也是目的地，可以是寄存器也可以是内存地址

移位操作大致上分为两种：`左移位`，`右移位`

具体还能分位`算术移位`或者`逻辑移位`，其中`算术左移`和`逻辑左移`没有区别，空出来的位都填充为0

`算术右移`和`逻辑右移`的区别在于：`算术右移`空出来的高位全部填充符号位，而`逻辑右移`空出来的高位全部填充为0

### 控制语句的实现

程序中的条件，循环和分支语句，需要有条件的执行，此时就不能按照传统方式将指令一条接一条地执行，而是需要根据条件从当前指令跳转到其他指令进行执行

计算机提供了两种机制来实现指令的有条件跳转：

1. 测试数据值
2. 根据测试结果改变控制流或数据流

#### 条件码寄存器

CPU维护了一种单个位`条件码`寄存器，这种寄存器只有0和1两种值，保存最近的运算操作产生结果的状态，通过检测这些寄存器来执行条件分支的跳转指令：

1. CF：进位标志，最近的操作是否使最高位产生了进位（无符号溢出）
2. ZF：零标志，最近的操作结果是否为0
3. SF：符号标志，最近的操作结果是否为负数
4. OF：溢出标志，与CF的区别是OF是有符号的补码溢出

之前描述的所有算术和逻辑运算指令在计算结果后，都会同时设置`条件码`

除此之外还有两个指令是专门用于设置`条件码`寄存器，而不会改变操作数：

1. CMP指令：CMP指令的操作和`SUB`指令都是一样的，都是做减法，不过CMP指令只修改条件码寄存器（ZF或者SF），用以比较两个数的大小，而不用修改任何一个操作数
2. TEST指令：TEST指令的操作和`AND`指令是一样的，都是作`&`操作，可以用来判断某数的符号

#### 如何访问条件码

条件码设置好后，需要读取条件码的状态来决定如何跳转