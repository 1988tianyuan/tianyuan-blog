title: java线程池源码分析 --- submit()的过程
author: 天渊hyominnLover
tags:

  - Java
  - 多线程
  - Java并发包
categories: [基础知识]
date: 2019-01-23 13:49:00
---
在jdk线程池中，`submit()`是`ExecutorService`的基础api，用于提交新任务给线程池进行执行，现对`submit()`执行过程一探究竟，这里主要对`AbstractExecutorService`及其子类`ThreadPoolExecutor`的实现进行讨论。
<!-- more -->
### submit()

jdk 8中，submit()有三个重载，分别是：

```JAVA
<T> Future<T> submit(Callable<T> task);
<T> Future<T> submit(Runnable task, T result);
Future<?> submit(Runnable task);
```

三者大同小异，最终都会返回`Future`对象来获取异步执行结果，即便传进来的是Runnable对象，也会包装为Callable进行执行，下面仅探讨第一个重载，源码如下：

```JAVA
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```

任务提交进来后，调用`newTaskFor`方法构建了一个`RunnableFuture`对象，最终会调用`execute`方法提交这个RunnableFuture，其实`RunnableFuture`是一个同时继承了`Runnable`和`Future`的接口，同时具有这两者的功能，在`submit()`中构建的是其实现类：`FutureTask`：

```JAVA
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}
```

这个对象是线程池的主要操作对象，是客户提交的任务的执行载体，其中封装了客户提交的callable (Runnable)任务

任务提交进来后会统一交给`execute()`方法进行执行，这个方法`AbstractExecutorService`交给了子类去实现

### execute()

在`ThreadPoolExecutor`中，源码如下：

```JAVA
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    // 获取ctl值，ctl对同时对线程池的两个状态进行控制：
    // 1. 当前线程池状态 2. 存活的工作线程总数（即worker数）
    int c = ctl.get();
    // 通过ctl获取当前工作线程数目
    if (workerCountOf(c) < corePoolSize) {
        // 如果小于核心线程数则增加worker，增加并提交任务成功则直接返回
        // 本次使用核心线程数来判断worker能否增加成功
        if (addWorker(command, true))
            return;
        // 增加失败，继续获取当前ctl值
        c = ctl.get();
    }
    // 检查当前线程池状态是否为RUNNING，并向workQueue缓存当前任务
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 如果当前状态不为RUNNING，尝试从workQueue移除本次任务
        // 移除成功后执行拒绝策略
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 如果当前存活worker总数为0则继续尝试增加worker
        else if (workerCountOf(recheck) == 0)
            // 第二个参数为false，说明本次使用最大线程数来判断worker能否增加成功
            addWorker(null, false);
    }
    // 若当前状态不为RUNNING或者向workQueue缓存当前任务失败，则尝试增加worker
    // 若增加worker失败（通常为已达到最大线程数）
    else if (!addWorker(command, false))
        reject(command);
}
```

整个execute()的过程很复杂，涉及到线程池中各组件比较复杂的交互过程，参考官方注释的说法，整个过程分为三步：

1. 如果worker数量少于**核心线程数**，则尝试增加worker并把当前任务作为新worker的firstTask并执行
2. 如果以上路线走不通，则尝试向`workqueue`缓存任务，待空闲的worker取任务，在这个过程中对ctl进行双重检查，防止ctl出现不一致（因为以上过程中并没有做同步处理），如果ctl状态不为RUNNING则将刚才的任务弹出workqueue并执行拒绝策略；若成功缓存任务后，且当前worker数为0，则尝试继续增加worker，用**最大线程数**来判断worker能否增加成功
3. 如果第2步走不通（比如状态非RUNNING或者缓存任务失败），尝试继续增加worker，用**最大线程数**来判断worker能否增加成功，如果这一步都走不通，那直接进行拒绝策略，整个过程结束

可以看出，整个过程非常依赖`addWorker`这个方法，主要用于新建worker并且提交firstTask，该方法执行成功与否直接关系到整个流程的走向，以下情况会导致增加worker失败：

`状态为Stop、Tidying或者Terminate`

`状态为Shutdown，提交任务为null并且workqueue为空`

`达到核心线程数或者最大线程数，或最大容量限制(2的29次方减1)`

`创建新worker时出现其他异常`

### addWorker

`addWorker`是整个任务提交过程中最重要的方法，以下是源码：

```JAVA
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
        // 以下条件判断能否增加worker
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;
		// 内嵌循环，
        // 每一次循环都要重新判断worker数目，worker达到数量限制则直接返回false
        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // cas方式增加worker数目，成功后直接退出外层循环
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            // 若内嵌循环过程中状态改变，则推出内嵌循环开始外层循环
            if (runStateOf(c) != rs)
                continue retry;
            // 如果仅仅是因为worker数目改变导致cas失败，则仅进行内嵌循环
            // 不需要进行外层循环重新获取ctl状态
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 新建worker对象并将任务作为其firstTask
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            // 同步操作
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // 重新获取ctl状态
                int rs = runStateOf(ctl.get());
				// 仅当状态为RUNNING或者为SHUTDOWN时提交的任务是null，才继续执行
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // 若该worker线程已经启动则抛出异常
                    if (t.isAlive())
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    // 增加largestPoolSize，仅作为统计用处
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                // 若增加worker成功则启动其线程，执行的是Worker对象的run方法
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

该方法第一部分的for循环略微有些绕，总的说来就是对线程池状态有可能随时变化作出的双重保障，内层循环服务于增加worker数目的cas操作，外层循环在此基础上加上了对ctl状态的重新获取及判断。

可以看出，整个submit过程离不开对线程池ctl状态的多次核查，保证了线程池的顺利运行，接下来对worker启动后做的工作进行简要分析

### Worker

Worker类继承了Runnable以及`AQS(AbstractQueuedSynchronizer)`，在他的run方法中调用了线程池对象的`runWorker`方法：

```JAVA
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    // 这个task变量很重要，是worker本次执行中的主要执行对象
    // 首先将worker的firstTask赋值给他
    // 赋值完后将worker的firstTask置为null
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        // 进入循环，执行任务，如果任务为null则从workqueue里面取
        while (task != null || (task = getTask()) != null) {
            // 对当前worker执行同步
            w.lock();
            // 如果当前worker线程未被打断，且状态为STOP及其以上（Tyding或者terminated），
            // 则将当前worker线程中断
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                // 执行前预处理，留给子类定制，通常用来对资源进行初始化，或者打印日志
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 真正执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    // 跟beforeExecute类似，也是执行资源释放或打印错误日志
                    afterExecute(task, thrown);
                }
            } finally {
                // 将task重置为null
                task = null;
                // 统计当前worker的执行任务数目
                w.completedTasks++;
                // 释放worker同步
                w.unlock();
            }
        }
        // 如果推出了该循环，则将completedAbruptly参数置为false
        completedAbruptly = false;
    } finally {
        // 执行worker退出操作
        processWorkerExit(w, completedAbruptly);
    }
}
```

整个过程大致分为以下几个步骤

1. 初始化，将firstTask作为初始任务
2. worker执行unlock()，调整AQS状态使其可以被打断
3. 进入循环，执行任务，如果任务为null则从workqueue里面取：`task = getTask()`
4. 判断是否需要将当前worker打断，满足条件则interrupt该worker的线程
5. 执行任务
6. worker循环执行任务，直到无任务可以执行，则正常退出循环，将completedAbruptly置为false；又或者执行了打断线程操作等原因抛出了异常，属于非正常推出循环，这时候completedAbruptly仍为true
7. 执行worker退出操作

其中最重要的两项操作分别是`getTask()`和`processWorkerExit(w, completedAbruptly)`，worker会持续尝试从`workQueue`中拿任务，worker拿不到任务或者非正常退出时则会执行退出操作，退出操作也比较重要，直接决定接下来线程池中是否保留以及保留多少个worker，现在对`getTask()`进行分析

### getTask()

源码如下：

```JAVA
private Runnable getTask() {
    boolean timedOut = false; // 判断poll()获取任务的过程是否超时
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
        // 若状态不为Running，并且workQueue为空或者状态为Stop，表明已经不需要执行任何任务了
        // 这时会直接减少workerCount并直接返回null，本次getTask提前结束
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
        // 重新计算workerCount
        int wc = workerCountOf(c);
        // Are workers subject to culling?
        // 官方注释的意思是，用timer参数标记当前worker是否需要保留，timed为true则不需要保留
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        // 如果workQueue为空或者workerCount大于1，有两种情况当前worker不需要保留：
        // 1. workerCount已经超出了最大线程数
        // 2. 获取任务超时并且不需要保留为核心线程
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            // cas方式减少workerCount，如果cas失败则循环重试
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }
        try {
            // 从workQueue取任务，根据timed不同又分为两种情况
            // 1. timed为true，当前worker在keepAliveTime时间内拿不到任务则会被抛弃
            // 2. timed为false，则当前worker作为核心线程保留下来并尝试拿任务
            // 由于workQueue是BlockingQueue，所以执行take()拿不到任务的话会阻塞直到队列中有任务可用
            // take()和poll()的过程都是可以被interrupt的
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
            workQueue.take();
            // 如果拿到任务后会返回，拿不到任务则将timedOut标记为true
            if (r != null)
                return r;
            // 拿不到任务，说明poll()超时了
            timedOut = true;
        } catch (InterruptedException retry) {
            // 说明拿任务的过程被interrupt了，将timedOut标记为false
            // 表明并不是因为poll()超时而获取不了任务
            timedOut = false;
        }
    }
}
```

`getTask()`成功与否直接关系到该worker是否会被抛弃，其中，`timed`这个boolean变量对worker是否需要保留为核心线程进行标记，还涉及到`allowCoreThreadTimeOut`这个属性，分为两种情况：

`allowCoreThreadTimeOut为false`：默认情况，线程池种会保留`corePoolSize`数量的线程作为核心线程，从上述代码种可以看出，只要当前workerCount不大于corePoolSize，那该worker就可以作为核心线程保留下来，取任务时调用`workQueue.take()`，持续阻塞直到有任务可以执行

`allowCoreThreadTimeOut为true`：需要手动调用`allowCoreThreadTimeOut(boolean value)`方法进行设置，这种情况下线程池不会保留核心线程，所有worker在取任务时均会调用`workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)`方法，在`keepAliveTime`时间后若还未取到任务则会被抛弃

以下几种情况会导致getTask方法返回null，即该worker无任务可以执行，将被抛弃：

1. 线程池状态不为Running，并且workQueue为空
2. 线程池状态为Stop
3. 线程池状态为Running，workQueue为空，并且workerCount已经超出了最大线程数
4. 线程池状态为Running，workQueue为空，获取任务超时并且当前worker不需要保留为核心线程

整个流程走下来，以上4种情况下该worker会被抛弃，进行下面的退出操作`processWorkerExit`，这种情况worker均为正常退出，`completedAbruptly`为false

### processWorkerExit

processWorkerExit源码如下：

```JAVA
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // 如果worker是非正常退出任务执行循环，则减少workerCount
    // 若是正常退出，则worker在getTask获取任务失败退出后已经减少了workerCount，可以正常移除该worker了
    if (completedAbruptly)
        decrementWorkerCount();
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        // 移除worker
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
    // 尝试执行终止操作
    tryTerminate();
    int c = ctl.get();
    // 如果当前状态为Running或者Shutdown，则执行以下流程
    if (runStateLessThan(c, STOP)) {
        // 若worker为正常退出任务执行循环，则需要额外判断是否需要新增worker
        // 分两种情况：
        // 1. 若需要将核心线程在一定闲置时间后被移除，则当前worker最多保留一个
        // 2. 如果不需要将核心线程闲置一段时间后移除，则可以保留不超过核心线程数的worker
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            // 如果worker已经够用了就不用addWorker了
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        // 执行以上判断后依然需要增加worker的话就调用addWorker，不传入任何task
        addWorker(null, false);
    }
}
```

逻辑相对比较复杂，总结来说，根据`completedAbruptly`参数将退出操作分为两条路线：

`正常退出任务执行循环：`

1. 不需要减少worker数目
2. 将当前worker移除，并尝试执行终止操作
3. 如果当前状态为`Running`或者`Shutdown`，表示如果`workQueue`里面还有任务要执行的话，是需要继续执行的，那么接下来尝试新增worker
4. 计算当前需要保留的worker数目（min变量），如果`workerCount`已经满足需求则不额外增加worker了（这里依然使用`allowCoreThreadTimeOut`判断是否保留一定数量的核心线程，如果为true，则worker最多保留一个），直接退出
5. 如果`workerCount`数目不满足需求，则新增一个worker然后让他去`workQueue`里面取任务执行

`非正常退出任务执行循环`

1. 减少worker数目
2. 移除当前worker并尝试执行终止操作，如果当前状态为`Running`或者`Shutdown`，则直接新增worker

至于为什么将worker退出操作分为正常和非正常，我是这么理解的：

`正常退出`：说明worker调用`getTask()`没有成功取到任务，将被抛弃，`getTask`方法已经对`workCount`进行了扣减，这里就不需要对`workerCount`作任何变动，此外需要判断当前`workerCount`数目够不够

`非正常退出`：这种情况下需要对`workerCount`进行扣减并立即补充一个worker，当然如果当前状态为`Stop`或者`Tyding`甚至`Terminated`的话就没必要补充了