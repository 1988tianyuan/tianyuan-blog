title: Netty概览
author: 天渊
tags:
  - netty
categories:
  - 基础知识
date: 2019-08-12 16:06:00
---
#### Netty是什么

Netty是一款用于快速开发高性能网络应用的Java框架，封装了网络编程的复杂性，使网络编程和web技术的最新进展能够比以往更广泛的让开发人员接触到。
<!--more-->

#### 为什么要用Netty

Netty是一个网络通信框架，目的就是屏蔽底层复杂的网络编程细节，提供更便捷的编程模型。

##### 开箱即用的网络组件

有了Netty，你可以实现自己的HTTP服务器，FTP服务器，UDP服务器，RPC服务器，WebSocket服务器，Redis的Proxy服务器，MySQL的Proxy服务器等等。

反过来看看，不使用netty，直接基于BIO或者NIO编写网络程序，你需要做什么：

1. 创建一个ServerSocket，监听并绑定一个端口
2. 一系列客户端来请求这个端口
3. 服务器使用Accept，获得一个来自客户端的Socket连接对象
4. 启动一个新线程处理连接

   - 读Socket，得到字节流

   - 解码协议，得到反序列化后的请求对象

   - 处理请求对象，得到一个结果，封装成一个返回对象

   - 编码协议，将结果序列化字节流

   - 写Socket，将字节流发给客户端
5. 继续循环步骤3

Netty并不需要你针对这些基础的IO过程编写大量的代码，已经封装好了成熟的IO库，开发人员只需要关注逻辑处理部分就可以。

##### 高性能并发机制

对于高性能网络组件，还得关注它的并发性能，这就涉及到多线程并发编程，Netty提供了便捷的开箱即用多线程框架，保证了成熟的异步回调和事件驱动机制

##### 高性能长连接支持

因为TCP连接的特性，我们还要使用连接池来进行管理：

1. 对于频繁的TCP通讯，很多时候需要保持长连接，保持连接效果更好
2. 对于并发请求，可能需要建立多个连接
3. 维护多个连接后，每次通讯，需要选择某一可用连接
4. 连接超时和关闭机制

Netty能够支持高性能长连接机制，因此在即时通讯和物联网等领域有很大的用武之地

##### 众多开源项目鼎力支持

一个优秀的开源项目离不开高质量的社区，Netty作为基础组件被众多开源项目和大型互联网企业采用，社区质量也是非常之高，如Apple，Twitter，Google和阿里等大型企业，还有Akka, Vert.x，Hadoop，ElasticSearch, Cassandra等优秀的开源项目，都在给Netty源源不断地贡献代码

{% qnimg 1564638670965.png %}

### Java NIO

介绍Netty基本特性之前，需要对NIO (non-blocking IO, 也叫做new IO)有一定的了解

Java NIO最早于Jdk 1.4版本引入，跟传统IO（BIO，也叫NIO）相比，极大的缓解了线程池处理海量连接的瓶颈，提高了IO密集型应用的处理效率

关于NIO的技术介绍可以看美团这篇文章：[Java NIO浅析](https://tech.meituan.com/2016/11/04/nio.html)，更进一步可以看Doug Lea的这篇论文：[Scalable IO in Java](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)

Netty对底层网络模型进行了一系列的封装和抽象，包括NIO和BIO，不过最常用的还是基于NIO的抽象，在Java NIO的`Selector`，`Channel`和`Buffer`基础上进行了丰富的抽象和封装，极大的简化了Java原生NIO组件复杂的编程模型，并在此基础上实现了reactor线程模型，能做到非常高效的并发处理

以下是一个简单的NIO网络模型：

![1564639452945](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1564639452945.png)

NIO网络模型最重要的组件就是`Selector`，底层基于操作系统的`epoll`或者`kqueue`等机制实现事件驱动式非阻塞IO操作，`Selector`在某些时候又称作`Reactor`（响应器，选择器，分派器......）

在Netty中，`Selector`(或者`Reactor`) 称作`EventLoop`，在Netty中采用的是`单bossEventLoop+多workEventLoop`的模式，由`bossEventLoop`负责响应client的连接请求，并建立连接，由多个`workEventLoop`负责维护客户端socket的数据交互和读写工作，每个`EventLoop`都会在一个独立的线程中执行

![1564642157798](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1564642157798.png)

### Netty核心特性浅析

#### Netty核心组件

摘自Netty官网的核心组件构成：

![](https://netty.io/images/components.png)

Netty核心功能由三部分组成：

##### Extensible Event Model

在我看来，Netty之所以这么优秀，除了对NIO网络模型进行了很好的抽象封装，另外一点就是其提供的方便高效的事件驱动的设计思想

###### 事件

Netty使用不同的事件来通知我们状态的改变或是操作的状态，我们能基于已经发生的事件来触发适当的动作，在Netty中事件类型分为两大类：`入站事件`，`出站事件`

> 入站事件：`Socket连接激活或连接失活`，`数据读取`，`用户事件`，`错误事件（Exception）`
>
> 出站事件：`打开/关闭远程连接`，`写数据到Socket`
>

###### Channel

事件的作用对象是`Channel`通道，`Channel`是Java NIO中表示连接的基本组件，代表到实体（硬件设备，文件描述符，socket网络套接字等）的开放连接，可以把它看作入站或出站的数据载体，对应`Channel`的事件就有读数据，写数据，开启或关闭等

事件的发起者即为`EventLoop`选择器，当检测到某个`Channel`状态发生变化（数据可读，可写等），即产生一个事件，并触发一系列的回调

Channel的生命周期：

![1564728511092](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1564728511092.png)

每一个阶段都会产生相应的事件并触发对应的回调，并且在`ChannelActive`和`ChannelInactive`两个状态之间还会产生读写事件，用户事件和错误事件

###### ChannelHandler

既然定义了事件，那就得有相应的事件回调处理器，在Netty中所有回调处理器均实现`ChannelHandler`这个接口，根据入站或出站事件又分为`ChannelInboundHandler`和`ChannelOutboundHandler`

![1564643610452](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1564643610452.png)

`ChannelHandler`由开发人员自己实现，开发人员可以根据不同的事件实现不同回调处理器的不同方法，例如某个handler需要捕获Channel激活的事件，可以像如下方式实现一个`ChannelInboundHandler`，服务端一旦检测到连接激活，则向客户端回复一条消息：

```java
public class FirstServerHandler extends ChannelInboundHandlerAdapter{
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ByteBuf buffer = ctx.alloc().buffer();
        byte[] bytes = "Connection successfully".getBytes(Charset.forName("UTF-8"));
        buffer.writeBytes(bytes);
        ctx.channel().writeAndFlush(buffer);
    }
}    
```

一个`ChannelHandler`可以实现多个回调方法，一个入站handler可以同时订阅激活事件，读事件，用户自定义事件以及错误事件

`ChannelHandler`编写完后注册到`ChannelPipeline`上，至于如何注册handler回调处理器，将在后续的sample中展示

###### ChannelPipeline和ChannelHandlerContext

`ChannelPipeline`是一个拦截流经`Channel`入站和出站事件的链条，所有`ChannelHandler`都需要挂载在`ChannelPipeline`上，每一个新创建的`Channel`都会分配一个新的`ChannelPipeline`

![1564731217542](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1564731217542.png)

上图是事件在每个`ChannelHandler`上的传播顺序

`ChannelHandlerContext`就是`ChannelHandler`和`ChannelPipeline`之间沟通的桥梁，每当一个新的`ChannelHandler`添加到pipeline中时，都会创建一个对应的`ChannelHandlerContext`，其主要功能时管理它所关联的`ChannelHandler`和在同一个pipeline中的其他`ChannelHandler`之间的数据交互

![1564732729637](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1564732729637.png)

如上图：

1. 事件（入站或者出站）传给`ChannelPipeline`的第一个handler
2. 通过与这个handler关联的`ChannelHandlerContext`将事件传递给下一个handler
3. 同2

##### Zero-Copy-Capable Rich Byte Buffer

netty使用`ByteBuf`来取代jdk自带的`ByteBuffer`作为nio的数据传输载体，相比于jdk原生的ByteBuffer实现，功能更加丰富，灵活性更强，具有以下优点：

- 扩展性好，用户可自定义所需要的缓冲区实现
- 内置复合缓冲区实现了零拷贝功能
- 容量按需增长
- 读数据和写数据有独立的index，互相隔离，互不干扰
- 支持引用计数和池化

在netty中`ByteBuf`有三种实现：`heapBuffer`，`directBuffer`，`compositeBuffer`，通常情况下使用directBuffer：

- heapBuffer：即将数据存储通过java Byte数组的方式（称为支撑数组）存储在jvm heap中，使用以下方式快速创建一个heapBuffer，但java进行io读写时仍然需要将堆内内存的数据拷贝到堆外并传递给底层的C库:

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

- directBuffer：使用堆外内存存储数据，直接使用堆外内存进行io操作，好处是比`heapBuffer`少一次内存拷贝且在io操作频繁的时候大大降低了gc压力，缺点是需要手动释放内存空间：

```java
ByteBuf buffer = ByteBufAllocator.DEFAULT.directBuffer();
```

`directBuffer`没有支撑数组，因此不能直接提取Byte数组，需要通过读写指针取数据

- compositeBuffer：复合buffer，其中可同时包含堆内数据和堆外数据，其实现是`ByteBuf`的子类：`CompositeByteBuf`，通过以下方式组装一个复合buffer，访问复合buffer的方式也类似于`directBuffer`，不能直接访问其支撑数组：

```java
CompositeByteBuf compBuf = ByteBufAllocator.DEFAULT.compositeBuffer();
compBuf.addComponents(buffer, heapBuffer);
```

复合buffer广泛运用于需要组合多种不同数据源的buffer，在对不同数据源的数据进行整合后提供统一的ByteBuf API供用户使用

Netty的零拷贝Buffer概念与操作系统层面的零拷贝不是一回事（Netty传输文件时已经通过Java NIO的DirectBuffer实现了基于DMA引擎的零拷贝），Netty的零拷贝描述的是组合buffer时不需要申请新的buffer内存，直接在原buffer的基础上通过`CompositeByteBuf`进行buffer的合并，而Java原生的ByteBuffer在这种情况下需要开辟新的buffer内存：

![1564645475015](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1564645475015.png)

##### Universal Communication API

统一的通讯API，Java原生的BIO和NIO使用了不同的API，而Netty则提供了统一的API(`org.jboss.netty.channel.Channel`)来封装这两种I/O模型。这部分代码在`org.jboss.netty.channel`包中

在核心功能之上，Netty还提供了很多开箱即用的API，为用户的协议解析，tcp层面的粘包拆包以及文件编码和安全认证等基础需求提供了诸多需求

#### Netty运行架构

Netty整体运行架构如下：

![1564648533067](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1564648533067.png)

   

### Netty基础使用 （sample）

Netty目前最新版本是`4.1.38.Final`，下列分析基本上都是基于4.x版本 （开发中的5.x版本因为某些原因作废了）

用Netty先实现一个最简单的tcp服务，发送一段简单的文本并获取相应

#### 启动Server端

启动一个能够运行的Netty服务端进程，大致有以下几步：

```java
1. 添加boss和work线程组
2. 指定io模型为nio方式
3. 指定server端启动时的初始化handler
4. 指定ChannelHandler，即具体的业务处理逻辑
5. 给NioServerSocketChannel指定attributes，后续可以通过channel.attr()取出这个属性
6. 给NioSocketChannel指定attributes
7. 给NioSocketChannel指定一些选项，比如是否开启TCP心跳机制或者Nagle算法等
8. 给NioServerSocketChannel指定一些选项，比如设置完成三次握手的请求的缓存队列大小00
```

代码如下：

```java
serverBootstrap.group(bossGroup, workerGroup)
    .channel(NioServerSocketChannel.class)
    .handler(new ChannelInitializer<NioServerSocketChannel>() {
        @Override
        protected void initChannel(NioServerSocketChannel channel){
            logger.debug("服务端启动中...");
        }
    })
    .childHandler(new ChannelInitializer<NioSocketChannel>() {
        protected void initChannel(NioSocketChannel nioSocketChannel){
            //（责任链模式）pipeline添加逻辑处理器，当接收到客户端数据时按顺序执行回调
            nioSocketChannel.pipeline()
                .addLast();
        }
    })
    .attr(AttributeKey.newInstance("serverName"), "nettyServer")
    .childAttr(AttributeKey.newInstance("clientKey"), "clientValue")
    .childOption(ChannelOption.SO_KEEPALIVE, true)
    .childOption(ChannelOption.TCP_NODELAY, true)
    .option(ChannelOption.SO_BACKLOG, 1024);

//绑定端口是一个异步过程，设置回调方法查看是否绑定成功
//默认绑定的ip地址是0.0.0.0
serverBootstrap.bind(8000).addListener(future -> {
    if(future.isSuccess()){
        logger.debug("8000端口绑定成功！");
    }else {
        logger.debug("8000端口绑定失败！");
    }
});
```

如上，`childHandler()`方法用于在建立连接的channel上绑定handler，一旦有事件触发， 事件会沿着添加的顺序进行传播，现在把`FirstServerHandler`这个handler绑定上:

```java
...
.addLast(new FirstServerHandler());
...
```

每一个成功建立连接的channel都会绑定一个`FirstServerHandler`，一旦channel激活，就会触发`channelActive`这个方法

#### 启动client端

启动一个Netty client，大致需要以下几步：

```java
1. 添加work线程组
2. 指定io模型为nio方式
3. 指定ChannelHandler，即具体的业务处理逻辑
4. 给NioSocketChannel添加attributes
5. 给NioSocketChannel指定一些选项，比如是否开启心跳以及设置连接超时时间，以及Nagle算法
```

代码如下：

```java
bootstrap.group(workerGroup)
	.channel(NioSocketChannel.class)
	.handler(new ChannelInitializer<NioSocketChannel>() {
		@Override
		protected void initChannel(NioSocketChannel nioSocketChannel) {
			//添加ClientHandler，连接上处理和服务器端的数据交互
			nioSocketChannel.pipeline().addLast(new FirstClientHandler());
		}
	}).attr(AttributeKey.newInstance("attrName"), "attrValue")
	.option(ChannelOption.SO_KEEPALIVE, true)
	.option(ChannelOption.TCP_NODELAY, true)
	.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000);

bootstrap.connect("127.0.0.1", 8000).addListener(future -> {
	if (future.isSuccess()){
		logger.debug("连接建立成功！");
		Channel channel = ((ChannelFuture) future).channel();
	} else {
		logger.error("连接建立失败！");
	}
});
```

client端实现一个`FirstClientHandler`来读取服务器发过来的信息：

```java
public class FirstClientHandler extends ChannelInboundHandlerAdapter {
	private static Logger logger = LoggerFactory.getLogger(FirstClientHandler.class);
	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
		ByteBuf buffer = (ByteBuf) msg;
		logger.debug("客户端读到数据：" + buffer.toString(Charset.forName("UTF-8")));
	}
}
```

以上就是一个最简单的Netty Server-Client demo

### Tips

使用Netty的过程中的一些知识点，小技巧和需要注意的地方

#### 调用ByteBuf.release()手动释放内存

由于netty默认使用的`ByteBuf`是`directBuffer`，不受gc影响，因此需要手动释放内存

> 对于入站消息，如果不调用`ctx.fireChannelRead(msg)`把消息往下传，则需要原地将消息释放
```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    ByteBuf buffer = null;
    try {
        buffer = (ByteBuf) msg;
        logger.debug("服务器端读到数据：" + buffer.toString(Charset.forName("UTF-8")));
    } finally {
        buffer.release();
    }
}
```

> 出站消息无需手动释放，netty最终会将其释放

netty提供了内存泄漏检测机制，日志中出现的`LEAK`字样需要格外引起注意

#### 异步Future

Netty中，进行IO操作如向channel写数据是异步操作，Netty提供了`ChannelFuture`，以异步的方式向channel写数据，保证不阻塞EventLoop线程，可以通过添加listenner获取异步发送的结果：

```java
ctx.channel().writeAndFlush(buffer).addListener(future -> {
    if (future.isSuccess()){
        logger.debug("Server数据发送成功");
    } else {
        logger.debug("Server数据发送失败");
    }
});
```

#### Attribute

Netty提供了`Attribute`类来实现属性绑定，使用`.childAttr()`方法在初始化阶段给每一个连上服务器的channel绑定属性：

```java
​```
.childAttr(AttributeKey.newInstance("clientKey"), "clientValue")
​```
```

属性跟随Channel整个生命周期存在，除非手动删除；运行阶段也可以动态绑定，修改或者删除属性值：

```java
// login
channel.attr("login").set(true);
//logout
channel.attr("login").set(false);
//check login status
Attribute<Boolean> loginAttr = channel.attr("login");
Boolean isLogin = loginAttr.get();
```



#### 自定义协议

Netty的网络组件只负责连接层（tcp或者udp）的数据解析和交互，剩下的应用层协议都需要用户自己实现

Netty已经提供了一些支持目前主流应用层协议的基础通信组件（http, websocket, mqtt, smtp, 还有redis协议等），除了这些开箱即用的应用层协议组件，大多数情况下都是基于Netty构建自定义协议来进行个性化开发

##### 设计协议

首先需要根据需要自己设计一个协议，便于解析二进制数据包满足业务需求，下面是一个经典的自定义二进制协议格式：

![1564725187585](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1564725187585.png)

#####  确定序列化算法

读取或者写入数据使用的载体是`ByteBuf`，需要某种序列化算法（json， protobuf，thrift等）使得Java对象和`ByteBuf`互相转化

##### 封装协议解析过程

以下是一个简单的例子：

```java
public class ProtocolCodeC {

    public static final int MAGIC_NUMBER = 0x12345678;

    //对象编码，返回
    public static ByteBuf encode(ByteBuf byteBuf, Packet packet){
        //序列化java对象
        byte[] bytes = Serializer.DEFAULT.serialize(packet);
        //写入数据
        byteBuf.writeInt(MAGIC_NUMBER);
        byteBuf.writeByte(packet.getVersion());
        byteBuf.writeByte(Serializer.DEFAULT.getSerializerAlgorithm());
        byteBuf.writeByte(packet.getCommand());
        byteBuf.writeInt(bytes.length);
        byteBuf.writeBytes(bytes);
        return byteBuf;
    }

    public static Packet decode(ByteBuf byteBuf){
        // 校验魔数
        byteBuf.skipBytes(4);
        // 校验版本
        byteBuf.skipBytes(1);
        // 解析序列化算法
        byte algorithmCode = byteBuf.readByte();
        // 解析指令
        byte command = byteBuf.readByte();
        // 解析消息长度
        int length = byteBuf.readInt();
        // 解析消息体
        byte[] data = new byte[length];
        byteBuf.readBytes(data);
        
        // 将消息体解析为具体的Java dto
        Class<? extends Packet> requireType = getRequireClass(command);
        Serializer serializer = getSerializer(algorithmCode);
        if(requireType != null && serializer != null){
            return serializer.deserialize(requireType, data);
        }
        return null;
    }
}
```

##### 注册Netty编解码器

将自定义协议组件封装为Netty提供的编解码器：

```java
public class PacketCodecHandler extends MessageToMessageCodec<ByteBuf, Packet> {
    public static final PacketCodecHandler INSTANCE = new PacketCodecHandler();
    private PacketCodecHandler(){}

    @Override
    protected void encode(ChannelHandlerContext ctx, Packet msg, List<Object> out) throws Exception {
        // 将ByteBuf序列化为Java对象（由出站事件触发）
        ByteBuf byteBuf = ctx.alloc().ioBuffer();
        PacketCodeC.encode(byteBuf, msg);
        out.add(byteBuf);
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf msg, List<Object> out) throws Exception {
        // 将Java对象反序列化为ByteBuf（由入站事件触发）
        Packet packet = PacketCodeC.decode(msg);
        out.add(packet);
    }
}
```

将编解码器注册到`ChannelPipeline`上，需要注意注册顺序：

```java
nioSocketChannel.pipeline()
    .addLast(new PacketCodecHandler())
    .addLast(...业务逻辑handler......)
```



#### 粘包/拆包

TCP连接过程中，粘包和拆包是经常发生的现象

> 拆包：由于TCP报文有长度限制，如果单体报文过长会拆包，将一个大包拆成几个小包，或者程序写入数据大小大于socket缓冲区，也会发生拆包
>
> 粘包：要发送的数据小于TCP发送缓冲区的大小，网卡将多次写入缓冲区的数据一次发送出去，将会发生粘包，或者接收端没有按时读取socket缓冲区的数据，导致一次性读取多个包的数据，也会发生粘包

如何解决这种粘包和拆包的情况？

> 1. 如果当前读取的数据不足以拼接成一个完整的业务数据包，那就保留该数据，继续从 TCP 缓冲区中读取，直到得到一个完整的数据包。
> 2. 如果当前读到的数据加上已经读取的数据足够拼接成一个数据包，那就将已经读取的数据拼接上本次读取的数据，构成一个完整的业务数据包传递到业务逻辑，多余的数据仍然保留，以便和下次读到的数据尝试拼接。

通常判断完整数据包的方法通常有以下几种：

> 1. 使用带消息头的协议、消息头存储消息开始标识及消息长度信息，服务端获取消息头的时候解析出消息长度，然后向后读取该长度的内容。
> 2. 设置定长消息，服务端每次读取既定长度的内容作为一条完整消息，当消息不够长时，空位补上固定字符。
> 3. 设置消息边界，服务端从网络流中读消息时通过'\n'等特殊字符判断消息边界来拆分消息

Netty对于粘包拆包的问题也提供了开箱即用的拆包合包器：

```java
1. 固定长度的拆包器 FixedLengthFrameDecoder
最简单的拆包器，Netty会把一个个长度为 100 的数据包 (ByteBuf) 传递到下一个ChannelHandler

2. 行拆包器 LineBasedFrameDecoder
发送端发送数据包的时候，每个数据包之间以换行符作为分隔，接收端通过 LineBasedFrameDecoder 将粘过的 ByteBuf 拆分成一个个完整的应用层数据包

3. 分隔符拆包器 DelimiterBasedFrameDecoder
DelimiterBasedFrameDecoder是行拆包器的通用版本，可以自定义分隔符。

4. 基于长度域拆包器 LengthFieldBasedFrameDecoder
只要自定义协议中包含长度域字段，均可以使用这个拆包器来实现应用层拆包
```

### Netty实战

#### Netty进阶——开发Http MVC框架

#### Netty进阶——开发RPC框架

#### Netty进阶——开发IM系统