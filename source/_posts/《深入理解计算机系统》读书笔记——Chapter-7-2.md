title: 《深入理解计算机系统》读书笔记——Chapter 8(2)
author: 天渊
tags:
  - csapp
categories:
  - 读书笔记
date: 2019-11-03 21:50:00
---
第七章`异常`的第二部分读书笔记


### 进程

`进程(process)`这个概念，为一个程序提供了一个虚拟的隔离空间，会造成一种假象，让程序以为自己是系统中运行的唯一一个应用
<!--more-->

**进程上下文**：即`context`，操作系统为每个进程都分配了上下文，每个程序运行在自己的上下文context中；这些context代表程序正确运行的状态，包括代码和数据，分配的调用栈，寄存器内容，程序计数器，环境变量以及打开的文件描述符

操作系统提供了两个关键的抽象用以实现进程的特性：`独立逻辑控制流`，`私有地址空间`

#### 独立逻辑控制流

每个进程为当前运行的这个程序构造了一个假象，即它是独立运行在操作系统中的，实现这种功能的关键就是`程序计数器(PC)`，每个进程独占一个PC，它的值与程序指令唯一对应，PC的一系列值组成的执行序列就叫做`逻辑控制流`

实际运行过程中，在单个核心的CPU上，同一时间有多个进程在一起运行，在外界看来这些进程在交错着运行，当某个进程运行了一段时间后即被挂起，CPU去执行其他进程

![](http://img.mantian.site/201910291132_70.png)

如上图，同一时间单个核心CPU上只能有一个进程运行，但在程序看来它的逻辑控制流却是连贯不间断的

**并发**：两个逻辑控制流执行时间有重叠，称为`并发流`，如上图的进程A和C，进程A运行过程中，进程C开始运行

并发流中的多个程序并不限制是在单个核或者多个核上运行，如果严格要求在不同核心上同时运行的程序，称之为`并行流`

#### 私有地址空间

每个进程为当前的程序提供了另一种重要的假象就是，让程序以为自己独占整个系统的地址空间，这种技术就是`虚拟内存`技术

每个进程的私有地址空间都具有同样的结构：

![](http://img.mantian.site/201910292116_973.png)

每个虚拟的私有地址的空间都有内核区域和用户区域，最下层的代码段地址总是从0x0040000开始

#### 用户模式和内核模式

操作系统使用`用户模式`和`内核模式`来隔离和区分用户程序和内核程序，这种机制限制了用户程序只能访问操作系统的一部分资源，例如`用户模式`下的程序只能访问一部分内存区域和系统调用，而`内核模式`下的程序拥有访问所有资源的权限，所以`内核模式`又称为`超级用户模式`，典型的与计算机硬件打交道的操作就必须运行在内核模式下，这样最大程度保证计算机系统运行的稳定性，不至于被用户错误的程序所影响

**模式位**：CPU通过`模式位`寄存器来限定当前程序是工作在哪种模式下

**/proc文件系统**：这种文件系统为用户模式下的程序提供了一种访问内核数据结构的途径，比如像CPU具体信息就可以访问这个文件系统；在Linux 2.6后引入了`/sys`文件系统，能够访问系统总线和设备的一些信息

#### 上下文切换

操作系统使用`上下文切换`的机制来实现多进程并发地运行在单个核心的CPU上，由于每个进程都有自己独立的逻辑控制流，对应一个特定的状态即`上下文(context)`，内核调度器在进行进程（或者线程）调度（scheduling）时，为了让各个进程平衡地运行，做了以下工作：

1. 内核选择一个新的进程运行，开始上下文切换，准备抢占当前运行进程
2. 内核调度器保存当前进程的上下文，恢复要运行的进程被保存的上下文
3. 将控制传递给新进程，CPU开始根据PC执行新进程还没执行的指令

哪几种情况会导致上下文切换呢？

1. 进程一些耗时的系统调用（例如磁盘IO操作）时，调度器可能会产生一次上下文切换，让当前进程暂时休眠，运行其他进程，待系统调用完成后再恢复之前休眠的进程
2. 当前进程主动执行sleep系统调用进行休眠；类似的像Java抢占锁失败也会让调度器执行上下文切换，休眠当前线程
3. 中断也会引发上下文切换

### 进程控制

##### 如何获取PID

`PID`即`Process Id`，每个进程在操作系统种运行时都有这么一个唯一的整数Id，用以标识某个进程的身份，在类Unix系统上用C语言获取当前进程PID和父进程PID的方法如下：

```c
int pid = getpid();
int parent_pid = getppid();
printf("pid: %d\n", pid);
printf("parent_pid: %d\n", parent_pid);
```

#### 进程的生命周期

在计算机系统中，进程一共有三种状态：

- `运行`：处理`运行`状态的进程是可以被调度器调度的进程，要么正在执行要么等待执行
- `停止`：进程处于挂起状态，一般是收到中断类信号后进入这个状态，在收到中断恢复信号后可以重新进入`运行`状态
- `终止`：进程停止，一般是收到一个终止信号，程序本身调用了exit函数或者直接从主程序返回

**父子进程 & fork** ：系统使用`fork`函数从父进程中创建一个新的运行的子进程，fork函数会返回子进程的PID，操作系统会为子进程产生一份父进程context的副本，包括一模一样的虚拟地址空间以及同样的文件描述符的副本，除了PID不同，子进程和父进程基本上一样；类Unix系统中，所有新启动的进程的父进程PID都是1