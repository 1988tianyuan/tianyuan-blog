title: 《深入理解计算机系统》读书笔记——Chapter 2(1)
author: 天渊
tags:
  - csapp
categories:
  - 读书笔记
date: 2019-09-05 10:55:00
---
第二章`信息的表示和处理`

本章主要是对信息在计算机系统上的表示和存储进行了详细的介绍，包括二进制位的相关知识以及整数和浮点数的存储和计算等知识

<!--more-->
目录：

- 信息存储
- 整数表示
- 整数运算
- 浮点


### 1. 信息的存储

**字节**：大多数计算机都使用`字节`(byte)作为系统中最小内存单位，内存就可以看作一个巨大的byte数组

**虚拟地址空间**：操作系统为程序提供`虚拟内存`(virtual memory)这一概念屏蔽了底层存储系统的复杂性，其中虚拟内存中每个字节都有自己的地址，也就是`虚拟地址`(virtual address)，虚拟内存中所有可能地址的集合就是`虚拟地址空间`(virtual address space)，C语言中某个指针的值就是某个存储块第一个字节的虚拟地址

> 指针：C语言的重要特性，有`值`和`类型`两个属性，`值`是某个对象的虚拟地址，`类型`是表示那个位置上存储对象的类型

#### 十六进制

一个字节是8位，表示成二进制是`00000000 ~ 11111111`，表示成十进制的范围是`0 ~ 255`，表示成十六进制是`00 ~ FF`，很明显十六进制的表示方式更加简洁易用，计算机系统中通常以`0x`开头表示一个以十六进制展现的内存地址

#### 字长

所谓`字长`，就是指针携带的地址值的标称大小，32位字长的机器其虚拟地址范围就是0 ~ 2^32^（最大虚拟地址值是2^32^-1，也就是说32位字长的系统虚拟地址空间大小就是4GB），64位字长的机器其虚拟地址范围就是0 ~ 2^64^（16EB）

> 向后兼容：64位机器可以运行32位机器编译的程序，但32位机器无法运行64位机器编译的程序

#### 寻址

一个对象在内存中是以连续的字节序列的形式存储的，对象的地址就是这段字节序列中最低位置的地址值

对象字节序列中，开始位置的字节称为`最低有效位`，结束位置称为`最高有效位`，比如对于`0x01234567`这个int，最高有效位字节就是01，最低有效位字节是67

##### 字节顺序

对象在内存中的字节序列是由一定的顺序进行保存的，保存对象字节序列的顺序就是字节顺序，有`大端法`和`小端法`两种：

- 大端法：从最高有效位到最低有效位的顺序存储对象
- 小端法：从最低有效位到最高有效位的顺序进行存储

![](http://img.mantian.site/201909041347_779.png)

如图，箭头方向就是地址顺序，左起是地址起始位置，大端法中最左侧是最高有效位字节，反之小端法中最左侧是最低有效位字节

大多数Intel兼容机都是小端模式，IBM和Oracle的服务器使用大端模式，不过现在很多微处理器都是`双端模式`，兼容两种模式

> 实战：使用c程序来展现不同的数据类型在内存中的字节表示

```c
#include <stdio.h>
//定义一个指向unsigned char的指针类型byte_pointer
typedef unsigned char *byte_pointer;
//将某个unsigned char按照内存中的字节顺序打印出来
void show_bytes(byte_pointer start, size_t len) {
    size_t i;
    for (i = 0; i < len; i++) {
        printf("%.2x", start[i]);
    }
    printf("\n");
}
//将某个int按照内存中的字节顺序打印出来
void show_int(int x) {
    //使用强制类型转换将int类指针转换为unsigned char指针
    show_bytes((byte_pointer)&x, sizeof(int));
}
//将某个float按照内存中的字节顺序打印出来
void show_float(float x) {
    show_bytes((byte_pointer)&x, sizeof(float));
}
//将某个指针数据按照内存中的字节顺序打印出来
void show_pointer(void *x) {
    show_bytes((byte_pointer)&x, sizeof(void *));
}
int main() {
    char c = 'h';
    byte_pointer start = &c;
    show_bytes(start, sizeof(char));
    show_int(1000);
    show_float(19.88f);
    show_pointer(&c);
    return 0;
}
```

按照内存中的顺序将各个数据类型按照字节排列顺序进行打印，可以发现：

show_bytes打印h字符结果为68，恰好就是Ascii码中h字符的字节表示

show_int打印1000结果为e8030000，因为当前windows环境下是**小端**表示，高位在后，低位在前，忽略掉高位0后其值为3e8，也就是1000的16进制表示

show_pointer打印char c指针的结果为47fe610000000000，忽略掉高位的0后就是0x61fe47，也说明64位环境下对象地址大小为8字节

> 使用数组方式引用指针：
>
> 上述例子中`start[i]`表示从start这个指针指向的内存起始位置，以数组的形式获取数据

###### 字符串的字节顺序

字符串类型的字节顺序与其他类型稍有不同，在任意平台上字符串都是正序排列，如下：

```c
#include <stdio.h>
#include <string.h>
void show_string(char *str) {
    show_bytes((byte_pointer)str, strlen(str));
}
int main() {
    char str[] = "hello";
    show_string(&str);
    return 0;
}
```

结果为`68656c6c6f`，也就是"hello"的正序Ascii码序列，说明字符串类型具有良好的跨平台通用性

#### 布尔代数

布尔运算在计算机系统中占有很重要的地位，在数值位运算中运用尤为广泛

基本的布尔运算逻辑有四种`&（与）`，`|（或）`，`~（非）`，`^（异或）`，四种布尔运算逻辑表格如下：

![](http://img.mantian.site/201909041553_177.png)

##### 位运算

布尔代数在计算机科学中一个很重要的运用就是位运算，C和Java等高级语言都支持位运算

有以下程序，可以将x和y各自的地址相互对调：

```c
#include <stdio.h>
void inplace_swap(int *x, int *y) {
    *y = *x ^ *y;
    *x = *x ^ *y;
    *y = *x ^ *y;
}
int main() {
    int x = 10;
    int y = 9;
    inplace_swap(&x, &y);
    printf("%d\n", x);
    printf("%d", y);
    return 0;
}
```

打印结果显示x=9，y=10，因为我们通过三次异或运算将x和y的值进行了对调，由此产生了一些结论：

1. 异或运算中，任何值与0的异或结果还是它本身
2. 异或运算中，任何值与它自己的异或结果为0
3. 异或运算支持结合律

通过以下函数可以将一个数组前后颠倒：

```c
void reverse_array(int a[], int count) {
    int first, last;
    for (first = 0, last = count -1; first < last; first++, last--) {
        inplace_swap(&a[first], &a[last]);
    }
}
int main() {
    int a[] = {1, 2, 3, 4, 5, 6};
    int len = sizeof(a) / sizeof(a[0]);
    printf("size of array is %d\n", len);
    reverse_array(a, len);
    for (int i = 0; i < len; i++) {
        printf("%d\n", a[i]);
    }
    return 0;
}
```

打印结果是"6, 5, 4, 3, 2, 1"

##### 逻辑运算符

逻辑运算符和位运算符功能相似，不过逻辑运算符针对Boolean类型，而位运算符仅仅对数字做二进制位运算

##### 移位运算

移位运算通常用于对数字的二进制位进行移动计算，具体的移位运算还包括`逻辑移位`和`算术移位`：

![](http://img.mantian.site/201909051029_959.png)

与`逻辑右移位`不同的是，`算术右移位`会在空出来的高位补足移位个数的最高有效位的值

与C语言不同的是，Java中`>>`代表`算术右移位`，`>>>`代表`逻辑右移位`

**注意**：移位运算符的优先级低于加减法运算符

### 2. 整数表示