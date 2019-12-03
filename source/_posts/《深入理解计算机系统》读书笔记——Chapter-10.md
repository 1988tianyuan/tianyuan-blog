title: 《深入理解计算机系统》读书笔记——Chapter 10
author: 天渊
tags:
  - csapp
categories:
  - 读书笔记
date: 2019-12-03 15:11:00
---
I/O是计算机系统中不可或缺的一部分，本章将了解I/O和其他系统之间的依赖和交互关系，以及通过C程序调用内核提供的I/O函数来实现一些通用的I/O功能，重点介绍了Unix I/O的一些概念和使用
<!--more-->

### Unix I/O和文件系统

在类Unix系统中，I/O设备都被定义为文件，输入和输出都是对该文件的读操作和写操作，所有的文件操作都抽象出了一个通用的应用接口即`Unix I/O`

**文件描述符（File Descriptor）**：一个进程打开一个文件时，内核都会给它返回一个非负整数即文件描述符，后续该进程对这个文件的所有操作都基于这个描述符，每个进程默认都有三个打开的文件即`标准输入`，`标准输出`和`标准错误`，对应的描述符分别为1，2和3

**文件类型**：每个Linux文件都有一个`类型（type）`来定义该文件在系统中的角色，常用的文件类型有三种：`普通文件`，`目录`和`套接字（Socket）`，每个文件在Linux文件树中都有自己对应的位置

#### 对文件的操作

##### 打开/关闭文件

使用`open`函数打开文件，内核会返回一个文件描述符，还可以指定对该文件的操作权限

使用`close`传入对应的描述符关闭该文件

##### 读写文件

通过`read`函数从文件中读取n个字节到指定的buffer内存位置中，该函数返回当前读取的字节数，返回0代表当前操作触发了EOF，返回-1则表示错误

通过`write`函数将指定buffer内存位置中的n个字节写入到指定文件中

##### 读取文件元数据

通过`fstat`函数来将某个文件的元数据读入到一个`stat`结构体中：

```c
struct stat status;
fstat(fd, &status);
if (S_ISREG(status.st_mode)) {
    printf("the type is regular");
    printf(", and st_size is %ld", status.st_size);
} else if (S_ISDIR(status.st_mode)) {
    printf("the type is dir");
} else {
    printf("the type is other");
}
```

这个`stat`结构体包含了一个文件所能拥有的所有状态，包括文件类型，所属inode节点和设备，以及所属的user和group，以及文件大小和创建时间等

##### 读取目录文件

通过`opendir`函数打开一个目录，该函数返回一个指针类型`DIR`，再调用`readdir`函数对这个`DIR`指针进行遍历，可返回一个结构体指针`dirent`，该结构体包含了该目录项的名称，inode号以及名称长度：

```c
DIR *dir = opendir("D:\\code learning\\csapp-chapter11-code");
struct dirent *dire;
while ((dire = readdir(dir)) != NULL) {
    printf("found file: %s\n", dire->d_name);
}
closedir(dir);
```

它将`csapp-chapter11-code`目录下的所有文件以及目录项打印了出来（不过没有进行递归遍历）

#### 共享文件

操作系统内核使用以下三个概念来管理一个进程打开的文件：

`描述符表（descriptor table）`：每个进程打开的文件所对应的文件描述符表，表中每一项都指向了下述文件表中的某一项

`文件表（file table）`：所有进程共享这张文件表，该表中的每一项包含该文件位置（即读写指针），引用计数（被上述某个进程的描述符表项引用一次就加1，如果引用计数为0，该项就被会删除），还有一个指向下述v-node表中的某一项

`v-node表`：所有进程共享该表，表中每一项都包含了对应文件的`stat`结构体中的大部分信息，例如`st_mode`信息和`st_size`信息

上述三张表对应关系如下：

![](http://img.mantian.site/201912031128_991.png)

需要注意的是，每个描述符表项与文件表项**一一对应**，如果某个文件被打开了两次，生成了2个描述符，那就有两个文件表项，但这两个文件表项都指向了v-node表中的统一项

**父子进程打开文件的情况**：父进程调用fork生成子进程后，子进程拥有和父进程描述符表相同的一份副本，其中每一个描述符项都与父进行指向同一个文件表项

每一个文件表项都拥有自己的文件位置，因此两个进程持有相同的文件描述符，其对该文件的读写操作都会对对方造成影响，而如果持有相同文件的不同文件描述符，相互之间的文件位置，也就是读写操作就是独立的

#### I/O重定向

在Linux系统中，通常在执行一个命令式，需要将输出结果由标准输出流转移到某个文件中，这时候就需要使用`I/O重定向`，一般是使用`>`符号：

在系统层面，调用`dup2`函数使某一个文件的I/O流重定向到其他的文件中去：

```c
int fd1 = open("D:\\code learning\\csapp-chapter11-code\\test.txt", O_RDWR, S_IRUSR);
int fd2 = open("D:\\code learning\\csapp-chapter11-code\\test2.txt", O_RDWR, S_IRUSR);
char c;
read(fd2, &c, 1);
dup2(fd2, fd1);
read(fd1, &c, 1);
printf("c=%c", c);
```

如上所示，一开始fd1和fd2是不同的文件描述符，对应不同的文件表项，首先从fd2中读一个字符，随后将fd1重定向到fd2，此时再读fd1，读出来的就已经是fd2对应的文件信息了，fd1此时指向的文件表项也跟fd2相同，此时对fd1的任何操作都与fd2完全一致

如下图：

![](http://img.mantian.site/201912031414_946.png)

上面是调用了`dup2(fd4, fd1)`的结果，将fd1重定向到fd4，重定向完成后fd1原来的文件表项引用计数变为0，被系统回收，相应的v-node项也被删掉，此时fd1和fd4都指向了同一个文件表项B，引用计数变为2

#### 标准I/O库

像`open`和`read`这些函数都是低级别I/O函数，C语言还额外封装了一组高级别的I/O函数称为`标准I/O库`，包括针对文件读写的`fread`和`fwrite`函数，读写字符串的`fgets`和`fputs`函数，以及常见针对标准输入输出流的`scanf`和`printf`函数

##### 该如何选择I/O库

除了读取元数据使用的`stat`函数外，其他情况下都建议使用标准I/O库，不使用低级别的I/O库









