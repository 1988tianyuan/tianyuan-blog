title: netty-ByteBuf浅析
author: 天渊
tags:
  - Netty
  - Java
categories:
  - 基础知识
date: 2019-02-12 17:59:00
---
netty使用`ByteBuf`来提代jdk自带的`ByteBuffer`作为nio的数据传输载体，相比于jdk原生实现，功能更加丰富，灵活性更强<!-- more -->，具有以下优点：

- 扩展性好，用户可自定义所需要的缓冲区实现
- 内置复合缓冲区实现了零拷贝功能
- 容量按需增长
- 读数据和写数据有独立的index，互相隔离，互不干扰
- 支持引用计数和池化

在netty中`ByteBuf`有三种实现：`heapBuffer`，`directBuffer`，`compositeBuffer`，通常情况下使用directBuffer：

1. **heapBuffer**：即将数据存储通过java Byte数组的方式（称为支撑数组）存储在jvm heap中，使用以下方式快速创建一个heapBuffer，但java进行io读写时仍然需要将堆内内存的数据拷贝到堆外并传递给底层的C库:

   ```java
   ByteBuf buffer = ByteBufAllocator.DEFAULT.heapBuffer();
   // 可以直接将所需Byte数组拿出来
   if (buffer.hasArray()) {
   	byte[] bufferArray = buffer.array();
   	int offset = buffer.arrayOffset() + buffer.readerIndex();
   	int length = buffer.readableBytes();
       // 通过读指针和可读长度获取所需的数据
   	byte[] neededData = Arrays.copyOfRange(bufferArray, offset, offset + length);
   }
   ```

2. **directBuffer**：使用堆外内存存储数据，直接使用堆外内存进行io操作，好处是比`heapBuffer`少一次内存拷贝且在io操作频繁的时候大大降低了gc压力，缺点是需要手动释放内存空间：

   ```java
   ByteBuf buffer = ByteBufAllocator.DEFAULT.directBuffer();
   ```

   `directBuffer`没有支撑数组，因此不能直接提取Byte数组，需要通过读写指针取数据

3. **compositeBuffer**：复合buffer，其中可同时包含堆内数据和堆外数据，其实现是`ByteBuf`的子类：`CompositeByteBuf`，通过以下方式组装一个复合buffer，访问复合buffer的方式也类似于`directBuffer`，不能直接访问其支撑数组：

   ```java
   CompositeByteBuf compBuf = ByteBufAllocator.DEFAULT.compositeBuffer();
   compBuf.addComponents(buffer, heapBuffer);
   ```

   复合buffer广泛运用于需要组合多种不同数据源的buffer，在对不同数据源的数据进行整合后提供统一的ByteBuf API供用户使用



### ByteBuf字节操作

以下操作均以`directBuffer`为例

