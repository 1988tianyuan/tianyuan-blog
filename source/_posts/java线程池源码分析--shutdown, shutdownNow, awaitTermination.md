title: 'java线程池源码分析--shutdown, shutdownNow, awaitTermination'
tags:
  - 多线程
  - Java并发包
  - Java
categories:
  - 基础知识
date: 2019-01-21 10:31:00
---
谈到jdk线程池的生命周期就不得不说shutdown，shutdownNow和awaitTermination这三个方法，一般用来进行线程池的停止和资源的释放，以下例子主要讨论`ThreadPoolExecutor`的实现

<!-- more -->

### 程序演示

- shutdown

  该方法用于在结束线程池的任务提交后关闭线程池，让线程池拒绝后续的任务提交，以下提交5个任务到线程池，并调用`shutdown`方法，接着尝试再次提交任务：

  ```JAVA
  @Before
  public void setup() {
  	executorService = Executors.newFixedThreadPool(5);
  	runnable = new Runnable() {
  		public void run() {
  			System.out.println("Thread: {" + Thread.currentThread().getName() + "} start!");
  			try {
  				Thread.sleep(3000);
  			} catch (InterruptedException e) {
  				e.printStackTrace();
  			}
  			System.out.println("Thread: {" + Thread.currentThread().getName() + "} stop!");
  		}
  	};
  	for (int i = 0; i < 5; i++) {
  		executorService.submit(runnable);
  	}
  }
  
  @Test
  public void test2() throws Exception {
  	System.out.println("try to shutdown threadPool!");
  	executorService.shutdown();
  	// here, jump out RejectedExecutionException
  	executorService.submit(runnable);
  }
  ```

  输出结果：

  ![1547708979007](/blog/images/1547708979007.png)

  可以看出，调用`shutdown`后是无法再次提交任务的，资源会被回收，只能重新创建一个新的线程池重新提交任务

- shutdown配合awaitTermination

  通常情况下`shutdown`方法不会独立使用，因为调用这个方法后无法得知立即线程池是否已经停止了，可能还有任务未执行完，官方注释如下：

  ![1547709651604](/blog/images/1547709651604.png)

  > 该方法并不会等待之前提交的任务完成执行，如果想达到这个目的请使用awaitTermination

  这个解释我认为有一点歧义，因为`shutdown`方法是会等待之前提交的任务完成执行的，只不过调用了该方法后并不会留时间给用户一个反馈，而`awaitTermination`会等待一段时间并查看线程池是否已经停止了:

  ```JAVA
  @Test
  public void test1() throws Exception {
  	System.out.println("try to shutdown threadPool!");
  	// wait for the tasks which is in execution to finish their work
  	executorService.shutdown();
  	while (!executorService.awaitTermination(1, TimeUnit.SECONDS)) {
  		System.out.println("threadPool is not terminated!");
  	}
  	System.out.println("threadPool is terminated!");
  }
  ```

  结果：

  ![1547710015291](/blog/images/1547710015291.png)

  提交5个任务后马上调用`shutdown`, 并不会中止任务，让`awaitTermination`每隔一秒检查一下是否停止，以此达到确认线程池成功停止的目的



- shutdownNow

  `shutdown`会等到提交过的任务完成执行， 但`shutdownNow`会尝试停止当前所有正在执行的任务

  ```JAVA
  @Before
  public void setup() {
  	executorService = Executors.newFixedThreadPool(5);
  	runnable = new Runnable() {
  		public void run() {
  			System.out.println("Thread: {" + Thread.currentThread().getName() + "} start!");
  			boolean isInterrupted = false;
  			try {
  				Thread.sleep(3000);
  			} catch (InterruptedException e) {
  				isInterrupted = true;
  			}
  			if (!isInterrupted) {
  				System.out.println("Thread: {" + Thread.currentThread().getName() + "} stop!");
  			}
  		}
  	};
  	for (int i = 0; i < 5; i++) {
  		executorService.submit(runnable);
  	}
  }
  
  @Test
  public void test3() throws Exception {
  	System.out.println("try to shutdownNow threadPool!");
  	// if the Runnable or Callable can jump out InterruptedException
  	// the thread in ThreadPool can be terminated by this method
  	executorService.shutdownNow();
  }
  ```

  程序对比之前的例子，做了一些修改，当出现`InterruptedException`则中断线程，结果如下：

  ![1547712156480](/blog/images/1547712156480.png)

  官方文档指出，当调用`shutdownNow`后，每个正在执行任务的线程调用了`interrupt()`方法，但这种方式只有对能正常反馈`InterruptedException`的线程使用，否则正在运行的线程依然无法停止：

  ![1547712312524](/blog/images/1547712312524.png)



  `shutdownNow`最终会返回还未执行的任务集合，比如如下例子：

  ```JAVA
  @Test
  	public void test1() {
  		executorService = Executors.newFixedThreadPool(1);
  		runnable = new Runnable() {
  			public void run() {
  				System.out.println("Thread: {" + Thread.currentThread().getName() + "} start!");
  				boolean isInterrupted = false;
  				try {
  					Thread.sleep(3000);
  				} catch (InterruptedException e) {
  					isInterrupted = true;
  				}
  				if (!isInterrupted) {
  					System.out.println("Thread: {" + Thread.currentThread().getName() + "} stop!");
  				}
  			}
  		};
  		executorService.submit(runnable);
  		executorService.submit(runnable);
  		executorService.submit(runnable);
  		executorService.submit(runnable);
  		executorService.submit(runnable);
  		List<Runnable> noExecutedTasks = executorService.shutdownNow();
  		System.out.println(noExecutedTasks.size());
  	}
  ```

  结果如下：

  ![1547712873168](/blog/images/1547712873168.png)

  线程池最大线程数只有1个，提交5个任务后马上调用`shutdownNow`，正在执行的任务被终止，并导致剩下4个任务没有机会执行，则将这四个任务集合返还给用户



- isShutdown和isTerminated

  线程池生命周期方法中的`isShutdown`和`isTerminated`也比较重要，前者在线程池调用`shutdown`后返回true，后者在线程池中任务全部执行结束后才返回true：

  ```JAVA
  @Test
  public void test4() throws Exception {
  	System.out.println("try to shutdownNow threadPool!");
  	executorService.shutdown();
  	// still RejectedExecutionException
  	System.out.println(executorService.isShutdown());
  	System.out.println(executorService.isTerminated());
  }
  ```

  ![1547713322805](/blog/images/1547713322805.png)

  调用`isShutdown`后，`isShutdown`为true，但`isTerminated`为false

  ```JAVA
  @Test
  public void test5() throws Exception {
  	System.out.println("try to shutdown threadPool!");
  	// wait for the tasks which is in execution to finish their work
  	executorService.shutdown();
  	System.out.println(executorService.isShutdown());
  	System.out.println(executorService.isTerminated());
  	while (!executorService.awaitTermination(1, TimeUnit.SECONDS)) {
  		System.out.println("threadPool is not terminated!");
  	}
  	System.out.println("threadPool is terminated!");
  	System.out.println(executorService.isTerminated());
  }
  ```

  ![1547713402558](/blog/images/1547713402558.png)

  第一次调用`isTerminated`，返回false，最后等所有任务都执行完了，该方法便返回true



### 源码浅析

调用上述几种线程池生命周期方法后，线程池内部做的怎样的实现呢？jdk线程池内部实现机理还是挺复杂的，现在从`isShutdown`入手来一探究竟

- shutdown()

  ```JAVA
  public void shutdown() {
      final ReentrantLock mainLock = this.mainLock;
      mainLock.lock();
      try {
          checkShutdownAccess();
          advanceRunState(SHUTDOWN);
          interruptIdleWorkers();
          onShutdown(); // hook for ScheduledThreadPoolExecutor
      } finally {
          mainLock.unlock();
      }
      tryTerminate();
  }
  ```

  `mainLock`用于将线程的停止操作进行同步处理，然后调用`checkShutdownAccess`对用户是否有线程访问权限

- advanceRunState(SHUTDOWN)

  `advanceRunState`方法用于切换线程池状态，调用该方法将线程池状态设置为`SHUTDOWN`

  ```JAVA
      private void advanceRunState(int targetState) {
          for (;;) {
              int c = ctl.get();
              if (runStateAtLeast(c, targetState) ||
                  ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))))
                  break;
          }
      }
  ```

  内部使用了CAS原子操作来进行状态切换

- interruptIdleWorkers()

  interruptIdleWorkers()方法用来给每个空闲的`worker`线程打上interrupt标记:

  ```JAVA
      private void interruptIdleWorkers(boolean onlyOne) {
          final ReentrantLock mainLock = this.mainLock;
          mainLock.lock();
          try {
              for (Worker w : workers) {
                  Thread t = w.thread;
                  if (!t.isInterrupted() && w.tryLock()) {
                      try {
                          t.interrupt();
                      } catch (SecurityException ignore) {
                      } finally {
                          w.unlock();
                      }
                  }
                  if (onlyOne)
                      break;
              }
          } finally {
              mainLock.unlock();
          }
      }
  ```

  `!t.isInterrupted() && w.tryLock() `这个判断很重要，首先判断当前worker的执行线程是否已经interrupt，然后判断worker是否能成功获取锁，如果返回true则说明当前worker没有执行任务（），最后执行interrupt()方法并释放work锁和mainLock同步锁

- onShutdown()

- tryTerminate()

  interruptIdleWorkers方法只能保证将空闲的worker线程置为interruptted，但正在工作的worker还是会继续执行任务，这时候需要启用`tryTerminate`方法，进行终止操作：

  ```JAVA
      final void tryTerminate() {
          for (;;) {
              int c = ctl.get();
              // 以下三种状态不进行终止操作：
              // 1.RUNNING状态 2.TIDING或者TERMINATED状态 3.SHUTDOWN状态并且workQueue不为空
              if (isRunning(c) ||
                  runStateAtLeast(c, TIDYING) ||
                  (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                  return;
              // 尝试interrupt空闲的worker，只中断一个
              if (workerCountOf(c) != 0) { // Eligible to terminate
                  interruptIdleWorkers(ONLY_ONE);
                  return;
              }
  			
              final ReentrantLock mainLock = this.mainLock;
              mainLock.lock();
              try {
                  // 真正开始终止操作，通过CAS将当前状态置为TIDYNG，即高于STOP的一种状态
                  if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                      try {
                          //终止操作，在ThreadPoolExecutor中啥也没做，留给子类用作额外的资源回收
                          terminated();
                      } finally {
                          //将状态置为TERMINATED
                          ctl.set(ctlOf(TERMINATED, 0));
                          //将调用了awaitTermination()的线程唤醒
                          termination.signalAll();
                      }
                      return;
                  }
              } finally {
                  mainLock.unlock();
              }
              // else retry on failed CAS
          }
      }
  ```