title: 《深入理解计算机系统》读书笔记——Chapter 11(1)
author: 天渊
tags:
  - csapp
categories:
  - 读书笔记
date: 2019-12-11 15:12:00
---
本章主要对计算机系统种的网络编程模型进行讲解，主要基于Linux网络api
<!--more-->
### 初始C-S模型

计算机系统中典型的网络应用都是基于C-S架构即`客户端-服务器模型`，一个典型的网络任务的基本流程是由客户端进程和服务器进程协同完成以下操作：

1. 客户端向服务器发起`请求（request）`
2. 服务器收到请求，解析这个请求知道客户端需要什么资源，开始操作这个请求并获取资源
3. 服务器操作完成后发起`响应（response）`，将客户端需要的资源发送给客户端，并等待下一个请求
4. 客户端收到响应并处理服务端返还的资源

### LAN和WAN

`LAN（Local Area Network）`即局域网，将一片区域内的计算机通过集线器和网桥进行连接，每台计算机都可以向局域网内的任意一台计算机发送网路请求，通过网桥来判断该向局域网内的哪台计算机路由该请求

`WAN（Wide Area Network）`即广域网，各个LAN网络通过路由器与其他局域网进行互联，形成一个WAN网络，如下图所示：

![](http://img.mantian.site/201912101005_792.png)

该方式的缺陷在于，每个局域网都有着不同的各不兼容的网络技术实现，怎么样让这些主机与互联网上的任意一台目的主机进行互联？这时候就需要一种统一的网络层协议来消除不同网络之间的差异，通过这一层协议控制主机和路由器协同工作实现数据传输，这种协议需要具备两种最重要的能力：

1. `命名机制`：每台WAN上的主机都会被分配至少一个互联网络地址，网络层协议通过定义这种一致的主机地址消除了不同的局域网技术之间主机地址格式的差异
2. `传送机制`：网络层协议通过定义`包（Package）`的形式把主机发送的数据打包成不连续的包，再将数据进行传输，消除了不同的联网技术中封装编码位和数据传输帧的方式的差异，一个包的header必须包括包大小，源主机地址以及目标主机地址

目前运用得最广泛的网络层协议就是IP协议