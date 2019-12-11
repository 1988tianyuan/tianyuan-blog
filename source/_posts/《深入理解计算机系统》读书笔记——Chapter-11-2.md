title: 《深入理解计算机系统》读书笔记——Chapter 11(2)
author: 天渊
tags:
  - csapp
categories:
  - 读书笔记
date: 2019-12-11 15:13:00
---

### IP协议 & 全球IP因特网
<!--more-->
一个典型的因特网架构模型就是如下图所示：

![](http://img.mantian.site/201912101106_404.png)

因特网上的每台主机都运行着实现了TCP/IP协议族的软件，这些软件在很多计算机中都是以`套接字接口（Socket）`和各种I/O函数（例如Unix I/O）暴露给用户进行使用

**IP协议**：IP协议（Internet Protocol）实现了通用的基本的网络包传送机制，这种网络传输包也叫`数据报（datagram）`，IP层的协议就负责将数据报从一台主机传送给另一台主机，但是IP层的传输是不可靠的，数据发送出去后并不会管对方主机是否收到或者是否在传输过程中丢失或者出错

**UDP协议**：UDP协议（Unreliable Datagram Protocol），不可靠的数据报协议，从名字上就可以看出，UDP协议在传输可靠性方面与IP协议并没有区别，只不过在IP层上再封装了一层，可以实现基于套接字的进程之间数据传输

**TCP协议**：TCP协议（Transmission Control Protocol），传输控制协议是，顾名思义该协议在IP协议的基础上实现了可靠进程间全双工的连接和传输

以下内容是对IP协议的一些细节进行的讲解

#### IP地址

IP地址（传统的IPv4地址）是一个32位无符号整数，通常用点分十进制来表示，每个点将一个IP地址分割为4个字节

使用`hostname -i`命令查看Linux环境下主机ip：

```shell
[root@localhost ~]$ hostname -i
10.250.140.27
```

#### 域名

使用IP地址标记主机地址的方式不太容易让人记住，因此推出了`域名（domain name）`，将ip地址和域名进行映射便于寻址，该映射关系可以保存在主机本地的`hosts`文件，但目前运用最广泛的还是遍布世界的`DNS服务（Domain Name Service）`来进行管理，DNS服务由成百上千万的`主机条目结构（Host Entry Structure）`组成，每一个条目定义了一组从域名到IP地址的映射

Linux环境下可以通过`nslookup`命令来查看一个域名和IP地址之间的映射关系

#### 网络连接 & 套接字接口

在OSI七层模型中，传输层的TCP协议是基于连接的，通过点对点的方式，两台主机进行全双工的通信，是一种可靠传输协议，TCP连接的两个端点就是`套接字Socket`，相应的套接字地址就是`address:port`的形式

##### 套接字接口

`套接字接口（Socket Interface）`是一组函数，是对底层网络协议的封装，现代操作系统上大多都实现了套接字接口，下图描绘了如何用套接字接口提供的函数来进行网络连接和通信：

![](http://img.mantian.site/201912101426_857.png)

从Linux的角度看，一个Socket就是一个有相应描述符的打开的文件

以下C程序是在win10上进行socket编程的例子：

```c
/*服务端*/
#include <stdio.h>
#include <inaddr.h>
#include "fcntl.h"
#include "winsock2.h"
#pragma comment(lib, "ws2_32.lib")

int main(int argc, char* argv[]) {
    // 初始化WSA
    WORD socket_version = MAKEWORD(2, 2);
    WSADATA was_data;
    if (WSAStartup(socket_version, &was_data) != 0) {
        return 0;
    }
	// 创建socket描述符
    SOCKET server_socket = socket(AF_INET, SOCK_STREAM, 0);
    // 初始化sockaddr_in
    struct sockaddr_in sin;
    sin.sin_family = AF_INET;
    sin.sin_port = htons(8080);
    sin.sin_addr.S_un.S_addr = INADDR_ANY;
	// 将socket绑定到相应地址和端口上
    if (bind(server_socket, (LPSOCKADDR)&sin, sizeof(sin)) == SOCKET_ERROR) {
        printf("bind error !");
        return 1;
    }
	// socket监听该地址和端口
    if (listen(server_socket, 1024) == SOCKET_ERROR) {
        printf("listen error !");
        return 1;
    }

    SOCKET client_socket;
    struct sockaddr_in client_addr;
    int remote_addr_len = sizeof(client_addr);
    char recv_data[256];
    int count = 0;
    while (count < 10) {
        printf("等待连接...\n");
        // 服务端socket接受一个从客户端发来的tcp请求，建立连接，创建客户端socket描述符
        client_socket = accept(server_socket, (SOCKADDR *)&client_addr, &remote_addr_len);
        if (client_socket == INVALID_SOCKET) {
            continue;
        }
        printf("接收到连接：%s...\n", inet_ntoa(client_addr.sin_addr));
        // 接收客户端数据
        int ret = recv(client_socket, recv_data, 255, 0);
        if (ret > 0) {
            recv_data[ret] = 0x00;
            printf("%s", recv_data);
        }

        char* data = "hello client!";
        // 向客户端发送数据
        send(client_socket, data, strlen(data), 0);
        // 关闭客户端socket
        closesocket(client_socket);
        count++;
    }
	// 关闭服务端socket
    closesocket(server_socket);
    WSACleanup();
    return 0;
}
```

客户端socket和服务端类似，只不过需要调用`connect`函数向服务端发送连接请求：

```c
/*客户端*/
#include <stdio.h>
#include "fcntl.h"
#include "winsock2.h"
#pragma comment(lib, "ws2_32.lib")

int main() {
    WORD socket_version = MAKEWORD(2, 2);
    WSADATA was_data;
    if (WSAStartup(socket_version, &was_data) != 0) {
        return 0;
    }

    SOCKET client_socket = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in sin;
    sin.sin_family = AF_INET;
    sin.sin_port = htons(8080);
    sin.sin_addr.S_un.S_addr = inet_addr("127.0.0.1");

    if (connect(client_socket, (LPSOCKADDR)&sin, sizeof(sin)) == SOCKET_ERROR) {
        printf("connect error.");
        closesocket(client_socket);
        return 1;
    }
    printf("到%s的连接建立...\n", inet_ntoa(sin.sin_addr));
    char data[256];
    send(client_socket, data, strlen(data), 0);

    char recv_data[256];
    int ret = recv(client_socket, recv_data, 255, 0);
    if (ret > 0) {
        recv_data[ret] = 0x00;
        printf(recv_data);
    }

    closesocket(client_socket);
    WSACleanup();
    return 0;
}
```

### Web服务器

目前运用最广泛的Web服务使用的应用层协议是`Http协议`