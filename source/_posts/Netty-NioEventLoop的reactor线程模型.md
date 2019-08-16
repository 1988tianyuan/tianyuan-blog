title: Netty-NioEventLoop的reactor线程模型
author: 天渊
tags:
  - netty
categories:
  - 基础知识
date: 2019-08-16 19:00:00
---
### Netty-NioEventLoop的reactor线程模型

Netty中reactor的核心逻辑主要由`NioEventLoop`及其父类`SingleThreadEventExecutor`和`SingleThreadEventLoop`这几个类进行实现，其中`NioEventLoop`的线程就是主要负责IO事件轮询工作的线程
<!--more-->

#### NioEventLoop的run方法

每一个`NioEventLoop`都会封装一个核心执行器`ThreadPerTaskExecutor`，`NioEventLoop`依靠这个执行器来启动它自己的核心线程`FastThreadLocalThread`，即eventLoop的reactor线程，由reactor线程来执行`NioEventLoop`的核心逻辑，即`run()`方法，包括NIO的select过程，处理IO事件并执行任务，以下是`NioEventLoop`的`run()`方法的源码：

```java
protected void run() {
    for (;;) {
        try {
            switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.CONTINUE:
                    continue;
                case SelectStrategy.SELECT:
                    select(wakenUp.getAndSet(false));
                    if (wakenUp.get()) {
                        selector.wakeup();
                    }
                default:
                    // fallthrough
            }
            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            if (ioRatio == 100) {
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    runAllTasks();
                }
            } else {
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    final long ioTime = System.nanoTime() - ioStartTime;
                    runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
        // Always handle shutdown even if the loop processing threw an exception.
        try {
            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    return;
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}
```

这个`run()`方法是一个死循环，单次循环的执行逻辑顺序大致如下：

> 1. 对注册到当前reactor线程selector的所有channel事件进行轮询
> 2. 处理产生的IO事件
> 3. 处理eventLoop任务队列中的任务

##### channel事件轮询

这个过程主要是基于jdk nio提供的`Selector`来实现，是eventLoop最核心的操作，对注册的channel IO事件进行轮询并取出已经产生的IO事件供后续的处理，这个操作基于操作系统底层的`epoll`等模型实现：

```java
select(wakenUp.getAndSet(false));
if (wakenUp.get()) {
    selector.wakeup();
}
```

`selector.wakeup()`这个操作能够解除阻塞在`Selector.select()`或者`select(long)`上的线程并立即返回，如果`Selector`当前没有阻塞在select操作上，则本次wakeup调用将作用于下一次select

什么时候需要唤醒？

> 1. 注册了新的channel或者原有channel注册了新的事件
> 2. channel关闭，取消注册
> 3. 优先级更高的事件触发，需要及时处理（如急需处理的普通任务或定时任务）

`NioEventLoop`的`select(boolean oldWakenUp)`方法内部是一个死循环，在对`Selector`进行轮询select时，基本上遵循以下顺序：

> 首先，如果eventLoop中有定时任务快到期了则立刻退出轮询，执行一次非阻塞select操作并停止select

```java
long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
if (timeoutMillis <= 0) {
    if (selectCnt == 0) {
        selector.selectNow();
        selectCnt = 1;
    }
    break;
}
```

> 然后，检测任务队列中如果有待执行的任务，则执行一次非阻塞select操作并立刻退出，需要将wakeup状态设置为true

```java
if (hasTasks() && wakenUp.compareAndSet(false, true)) {
    selector.selectNow();
    selectCnt = 1;
    break;
}
```

> 随后，执行select操作，该操作在超时时间到期前会一直阻塞，超时时间是截至到最早的那个定时任务到期
>
> 超时后会进行一系列判断，比如selectedKey的数量，wakeup状态，是否有待执行任务等等，如果满足条件则马上退出轮询

```java
int selectedKeys = selector.select(timeoutMillis);
selectCnt ++;
if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
    break;
}
```

> 最后，由于jdk nio有一个bug会使`Selector`无法成功地阻塞规定的时间，造成无限空轮询，进而导致CPU 100%，netty使用了某种措施规避了这个bug，在一次成功的select操作后，如果仍然无需退出轮询，则判断如果selectCnt（已轮询的次数）大于某个阈值（默认512）后，直接重建`Selector`

```java
long time = System.nanoTime();
if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
    // select持续时间大于规定的超时时间timeoutMillis，说明本次select有效，重置selectCnt并进行下一次轮询
    selectCnt = 1;
} else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
        selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
    // selectCnt已经达到空轮询阈值，立即重建Selector
    rebuildSelector();
    selector = this.selector;
    // 进行一次立即返回的select并重置selectCnt
    selector.selectNow();
    selectCnt = 1;
    break;
}
```



##### 处理产生的IO事件

进行完一次成功的select操作后，需要处理产生的IO事件，其中，`ioRatio`是处理IO事件和处理非IO任务（即用户自定义handler的任务）的百分比，默认是50，即IO事件和非IO事件所占用时间一半一半；如果`ioRatio`为100，则对两种事件没有时间限制，代码如下，

```java
final int ioRatio = this.ioRatio;
if (ioRatio == 100) {
    try {
        processSelectedKeys();
    } finally {
        // 保证一定会执行任务队列中的任务
        runAllTasks();
    }
} else {
    final long ioStartTime = System.nanoTime();
    try {
        processSelectedKeys();
    } finally {
        final long ioTime = System.nanoTime() - ioStartTime;
        // 非IO任务耗费的时间不能超过ioRatio的比率
        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
    }
}
```

select完成后得到的`SelectionKey`位于`selectedKeys`这个成员变量中，其是由netty自己基于`AbstractSet`实现的一个类：`SelectedSelectionKeySet`，在jdk nio `SelectorImpl`中`HashSet`的基础上做了一些优化，将保存`SelectionKey`的容器设置为数组，相比于`HashSet`从时间复杂度和内存占用上都有一定的优化

由`processSelectedKeysOptimized(SelectionKey[] selectedKeys)`方法处理获得的`selectedKeys`：

```java
private void processSelectedKeysOptimized(SelectionKey[] selectedKeys) {
    for (int i = 0;; i ++) {
        final SelectionKey k = selectedKeys[i];
        if (k == null) {
            break;
        }
        selectedKeys[i] = null;
        // 1. 取出attachment，可能是AbstractNioChannel或者NioTask
        final Object a = k.attachment();
        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }
        if (needsToSelectAgain) {
            for (;;) {
                i++;
                if (selectedKeys[i] == null) {
                    break;
                }
                selectedKeys[i] = null;
            }
            selectAgain();
            selectedKeys = this.selectedKeys.flip();
            i = -1;
        }
    }
}
```

遍历`selectedKeys`，每一次循环分为以下几步：

1. 取出`SelectionKey`，将`selectedKeys`数组的当前位置设置为null，在相关的`Channel`关闭后，gc会将这个`SelectionKey`及其与之关联的`Channel`回收掉，防止内存泄漏
2. 从`SelectionKey`中取出`Attachment`，可能是`AbstractNioChannel`或者`NioTask`，其中`AbstractNioChannel`相关的事件由Netty框架处理，`NioTask`相关的事件由用户自己注册并处理；主要关注`AbstractNioChannel`及其处理方法`processSelectedKey(k, (AbstractNioChannel) a)`
3. 当前`SelectionKey`处理完成后判断是否该再来一次select轮询，当`needsToSelectAgain`为true时则需要再来一次轮询

> 为何要在处理SelectionKey期间重新进行select轮询？
>
> 因为随时都有Channel断线，Channel断线后会执行cancel()方法取消对应的SelectionKey，为了保证现存的SelectionKey都是有效的，需要定期（默认256次cancel后）清理SelectionKey的集合，保证现存的SelectionKey时及时有效的

处理完`SelectionKey`后就是最后一步：处理eventLoop任务队列中的任务

##### 处理eventLoop任务队列中的任务

eventLoop任务队列是`taskQueue`，其实现默认是`jctools`包中的同步队列`MpscChunkedArrayQueue`，即多生产者单消费者队列，针对并发写的场景进行了一些优化

`taskQueue`任务队列中的任务主要来自三个途径：

1. 用户调用`eventLoop.execute()`执行的自定义任务，会直接向`taskQueue`添加任务
2. channel触发的各种事件，比如往channel中写数据会产生出站事件，会向`taskQueue`添加一个`WriteTask`或者`WriteAndFlushTask`
3. 用户自定义的定时任务，通过调用`eventLoop().schedule()`方法添加一个定时任务用于在一定时间后执行任务，保存定时任务的队列`scheduledTaskQueue`是一个优先队列`PriorityQueue`

###### 自定义任务

用户可以在任意一个handler通过以下方式获取eventLoop实例并调用execute()添加自定义任务，然后由eventLoop线程来执行任务:

```java
ctx.channel().eventLoop().execute(() -> {
    System.out.println("task");
});
```

execute()方法部分代码如下：

```java
public void execute(Runnable task) {
    //...
    boolean inEventLoop = inEventLoop();
    if (inEventLoop) {
        addTask(task);
    } else {
        startThread();
        addTask(task);
        if (isShutdown() && removeTask(task)) {
            reject();
        }
    }
    //...
}
```

首先通过inEventLoop()方法判断当前执行线程是否是eventLoop线程，如果不是的话则要先启动执行线程，保证执行任务的始终是唯一确定的eventLoop线程

> 在netty的线程模型中，每一个eventLoop对应一个唯一的执行器ThreadPerTaskExecutor，每个执行器都会提供一个FastThreadLocalThread线程实例，这个线程就是eventLoop的reactor线程，任务队列中的所有任务在执行前都会判断该任务是否由reactor线程来执行，保证了eventLoop的核心逻辑的单线程环境

###### channel事件任务







###### 自定义定时任务