title: 《深入理解计算机系统》读书笔记——Chapter 1
author: 天渊
tags:
  - csapp
categories:
  - 读书笔记
date: 2019-09-01 08:23:00
---
深入理解计算机系统 —— 第一章：

本书第一章是从一个基本的c语言hello world程序是如何被计算机执行的作为线索，简单介绍了计算机系统中的基本组成和一些核心概念，这些核心概念会贯穿整本书的讲解

<!--more-->

#### 目录

- 何为信息？bit + 上下文
- 程序的编译，如何让机器读懂你的程序
- 了解一下编译原理
- 处理器是如何处理内存指令的，包含系统硬件组成简介
- 高速缓存为加速处理器读写起到关键作用
- 存储设备的层次结构
- 操作系统是如何运作在系统硬件上的（线程，进程，虚拟内存，文件系统等）
- 重要概念总结（Amdahl定律，并发与并行，计算机系统中的抽象）



#### 信息就是bit+上下文

信息在计算机系统中是以何种形式存在的？计算机系统中所有信息（内存中的数据，用户编写的程序，磁盘中的文件，网络中传输的信号和文件等）都是以一串bit表示的

如何区分这些bit串要表达的不同含义？这时就需要判断这些bit串的上下文，不同的bit串在不同的上下文中可能表示一个整数，一个字符串或者一个特殊的cpu指令

> C语言的前世今生：
>
> C语言是一种高级语言，诞生于为Unix操作系统编写各种应用程序的需求，由于其小而简单并易于移植到不同系统的特性而广受欢迎；不过由于它的指针特性等不易于程序员掌握的特性以及缺乏一些抽象的显示支持如类，对象以及异常等，因此不太适合用于编写大型的企业级应用，往往更适合于编写偏底层的小型应用

#### 如何让机器读懂你的程序？程序的编译

程序员编写好一个c语言源文件，是不能直接运行在系统上的，因为机器读不懂，cpu无法直接执行源文件

只有将源文件编译为cpu能够执行的一系列的机器语言指令，然后将这些指令打包为**可执行文件**，才能被cpu执行

##### c语言源文件的编译过程

c语言源文件`hello.c`编译为cpu能够执行的可执行文件需要以下过程：

1. 预处理阶段

> 执行单位：预处理器cpp
>
> 生成：修改后的源文件`hello.i`
>
> 目的：根据`#`字符开头的命令，读取其他程序包到源文件中比如`stdio.h`，形成一个新的源文件

2. 汇编阶段

这个阶段又包括两个过程，一个是编译器编译为汇编文本文件，然后由汇编器编译为二进制格式的可重定位目标程序

> 执行单位：编译器ccl + 汇编器as
>
> 生成：汇编文本文件`hello.s` ， 二进制的可重定位目标程序`hello.o`
>
> 目的：源文件编译后的汇编程序已经非常接近机器能够执行的机器指令了，不同的高级语言编译后的汇编语言指令都是同一种

3. 链接阶段

由于`hello.c`中会使用c语言标准库的printf函数，这个阶段会将已经编译好的printf函数链接到我们的`hello.o`文件中，生成最终的可执行文件，执行的时候加载到内存中由cpu读取指令进执行

#### 了解一下编译原理

虽然源代码的编译过程由系统编译器自行完成，但了解一些编译原理也是有必要的：

- 有助于编写性能良好的程序，熟悉编译原理的话就知道自己写的程序将会编译成什么样的机器指令去执行，什么样的执行过程对cpu执行来说才是最优的
- 理解链接时出现的错误，比如如何区分静态变量和全局变量，为什么直到运行时才发现链接错误
- 避免缓冲区溢出等安全隐患，理解数据和信息存储在栈上的方法会引起的后果

#### 处理器是如何处理内存指令的

在理解我们的`hello`可执行文件是如何被cpu执行前，需要先了解系统硬件的组成和协作过程

##### 系统硬件组成介绍

一个基本的冯诺依曼体系的计算机系统由`处理器`，`主存`，`I/O设备`和`总线`组成

- **处理器**CPU：系统的大脑，负责执行程序机器指令，由`算术逻辑单元ALU`，`程序计数器PC`，`寄存器`组成，PC指向主存中某条需要执行的机器指令，由ALU读取该指令并执行该指令的操作，再更新PC使其指向下一条指令
- **主存**：包括RAM临时存储和ROM永久存储，主存是系统cpu执行指令时用来临时存放程序指令和数据的地方，每个主存的存储字节都有其唯一的地址
- **I/O设备**：包括显示器，鼠标键盘，网卡和磁盘等用于系统和外界交互的硬件设备，每个I/O设备都需要适配器与I/O总线相连
- **总线**：一个携带信息字节负责在各个部件间传输信息的电子管道，总线传输宽度由`字长`确定，现代操作系统一般为32位字长或者64位字长

##### hello程序的执行过程

在键盘上输入命令执行`hello`文件或者鼠标在屏幕上点击该可执行文件后，依次发生了以下事件：

1. shell程序会启动一系列指令将二进制可执行文件从磁盘通过I/O总线加载到主存中（通过`DMA`技术直接将数据拷贝到主存而不经过cpu的加载和存储等操作）
2. hello文件加载到主存后，cpu通过内存总线读取主存中的机器指令，这些指令会将`hello world`字符串从主存复制到cpu寄存器，再从寄存器通过I/O总线输出给显卡适配器
3. 显卡适配器获取到具体需要打印的`hello world`字符串，将信息输出到屏幕上

#### 高速缓存为加速处理器读写起到关键作用

以上hello程序执行的过程中，需要先把信息从磁盘复制到主存，再由cpu根据指令从主存复制字符串信息到寄存器，然后再输出给I/O，这一来一去的复制过程大大降低的程序的执行速度，因为cpu从寄存器中读数据速度远大于从主存中读取数据，复制过程就成了瓶颈

`高速缓存`利用了空间和时间局部性原理，存放cpu近期可能会用到的信息能够大大降低cpu从主存读取数据带来的运行瓶颈

主流cpu采用`SRAM`技术，一共将高速缓存分为`L1`,`L2`和`L3`三个层级，其中最靠近cpu的`L1`高速缓存存取速度最快，容量也最低

#### 存储设备的层次结构

计算机系统中的存储设备组织成一个存储器层次结构，从小往上，存储容量越小，访问速度越快：

![1566833029236](http://img.mantian.site/1566833029236.png)

#### 操作系统是如何运作在系统硬件上的

用户的应用程序，比如一个shell脚本程序，或者一个c程序，是无法直接和系统硬件打交道的，取而代之的是操作系统，充当了用户程序和系统底层硬件的中间层，提供了统一的系统调用让用户程序间接访问系统硬件，这么做有两个目的：

- 防止用户程序失控滥用系统硬件

- 向用户程序提供简单一致的机制来对付复杂的底层硬件

通过`进程`,`虚拟内存`和`文件系统`这几种机制来实现以上两个目的

##### 进程

进程是现代操作系统中最重要的一个概念，操作系统使用进程这个概念来使得某一个应用程序与本机上其他应用程序进行资源隔离，并且让这个应用程序以为本台机器上只有它自己一个程序在运行

`进程上下文切换`：

- 单个CPU核心同一时间只能运行一个程序，如何让多个程序运行在这个CPU核心上？CPU通过**上下文切换**来并发地处理多个进程；

- 进程的上下文包含该进程在PC，寄存器和主存中暂存的数据

- CPU执行进程的上下文切换时需要切换到内核态

##### 线程

现代操作系统中，一个进程可以由多个线程组成，每个线程相互之间共享进程的代码和全局数据，可以把进程看作轻量级的进程，CPU也是并发地处理多个线程的请求，同样也会存在上下文切换的情况

##### 虚拟内存

操作系统为了让进程更方便地管理自己的内存空间，使用虚拟内存这一概念为每个进程提供一个抽象，让进程认为自己独占了整个内存空间，因此进程操作的内存地址空间都是操作系统创建出来的虚拟内存地址空间，底层真实内存地址空间是不提供给进程直接访问的

虚拟内存地址空间的组成如下：

- `只读代码和数据区`：包含程序代码数据和全局变量，进程初始化时就被分配
- `运行时堆`：堆可以动态缩/扩容，通过malloc和free等函数动态分配的内存
- `共享库`：存放c标准库等需要共享的代码和数据，与动态链接有关
- `栈`：编译器使用栈这种数据结构来实现函数调用，用户栈在运行时也可以动态缩/扩容
- `内核虚拟内存`：为内核调用保留的地址空间

##### 文件系统

在类Unix系统中，万物皆文件，包括磁盘，网络以及键盘鼠标显示器等IO设备，都看作文件

操作系统使用文件系统这一概念将底层IO设备进行了抽象，应用程序无需关注硬件设备复杂的实现技术，只需要关注操作系统提供的文件系统接口就行了



#### 重要概念总结

##### Amdahl定律

Amdahl定律的概念简要概况：系统中某个部分组件进行性能提升的比率，小于对系统整体带来性能提升的比率，也就是说如果要对系统整体进行性能提升，只优化某部分的性能是不够的，必须提升系统绝大多数组件的性能

##### 并发和并行

- 并发：指一个系统同时有多个活动在同时进行
- 并行：使用并发的方式让系统运行得更快

`线程级并发`：一个进程中运行多个线程就是线程级并发技术，并且现代CPU使用超线程技术让单个CPU核心运行不止一个线程

`指令级并行`：现代CPU可以在一个时钟周期内执行多条指令，称为指令级并行，这种处理器也称为`超标量处理器`

`单指令多数据并行`：现代CPU允许一条指令产生多个可以并行执行的操作，称为单指令多数据并行（SIMD）

##### 抽象

现代计算机科学中最重要的概念之一就是`抽象`，计算机系统中无时无刻不存在抽象这个概念的运用，提供不同层次的抽象来隐藏底层实现的复杂性