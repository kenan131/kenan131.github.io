---
title: 定时线程池抛异常后任务丢失&原理分析
date: 2024-06-05 17:03:01
tags: 踩坑日记
categories: 踩坑日记
---

### 背景

#### 问题场景

在我的网关项目中，有个功能需求是：需要定期拉取nacos上的所有服务实例信息，如果有新出现的服务实例，则保存到本地缓存中并监听该服务的上下线情况。伪代码如下：

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(1);
    scheduledThreadPool.scheduleWithFixedDelay(new Runnable() {
        @Override
        public void run() {
            discovery();
        }
    },0,10,TimeUnit.SECONDS);
}

static public void discovery(){
    //1、获取所有服务
    //2、获取已监听的服务
    //3、遍历所有服务，查询出未注册监听的服务
    //4、注册监听
    //5、保存服务信息到本地缓存中
}
```

#### 问题发现

这代码其实是项目初期就写完了，一直没有机会测试。等到我写完路由过滤器之后，要测路由功能是否正确时才发现的bug。测试工具用postman调用网关地址，并且请求路径能映射到服务提供者的地址，请求路径和配置文件反复确认是没问题的。这时postman工具中显示请求失败，错误信息为”配置文件有误，根据请求路径未找到实例名“。此刻直接用debug模式运行项目，准备战斗。

经过一段调试后，最后将问题定位到服务本地缓存模块中，此时本地缓存竟然没有一个服务实例，而我nacos的控制台上是有服务实例的。这时候基本确定了是我定时获取nacos服务并监听出了问题。

此刻查看了线程池中核心线程的执行状态，为阻塞状态，心想此刻还没到10s钟触发周期，等等看。等了1分钟都没看到状态变更？？难道我任务被吃了？没添加进去？

又一通debug后，发现提交给定时线程池的任务确实是执行了，但是代码有bug抛异常了。之后线程就一直是waiting状态。此刻有点疑惑，线程池不是会每隔10s就往阻塞队列里丢一个任务吗？然后线程去阻塞队列中获取任务，怎么任务没了呢？难道跟我想象的不一样吗？于是我又写了段测试代码，代码如下：

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(1);
    scheduledThreadPool.scheduleWithFixedDelay(new Runnable() {
        int index = 1;
        @Override
        public void run() {
            System.out.println(index ++);
            if(index == 10){
                int a = 1/0;
            }
        }
    },0,2,TimeUnit.SECONDS);
}
```

代码每隔两秒打印了一个数字，共10个，从1-10。之后线程就一直是waiting状态了，且控制台都没有打印出信息。确实是任务丢了，阻塞队列中没有，线程就一直是waiting状态。问题复现

#### 问题处理

此刻其实只要将异常捕获掉并处理，不要抛出去就能解决上面的问题。代码如下

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(1);
    scheduledThreadPool.scheduleWithFixedDelay(new Runnable() {
        int index = 1;
        @Override
        public void run() {
            System.out.println(index ++);
            try{
                if(index == 5){
                    int a = 1/0;
                }
            }catch (Exception e){
                System.out.println("触发异常:" + e.getMessage());
            }
        }
    },0,1,TimeUnit.SECONDS);
}
```

运行结果如下：

```txt
1
2
3
4
触发异常:/ by zero
5
6
7
8
9
··· 一直递增打印下去	
```

问题其实就解决了。但是心中还是有疑惑，怎么代码抛出异常任务就直接丢失掉了呢。

#### 扩展

那么问题来了，定时线程池有任务丢失情况，那么普通线程池呢。测试代码如下：

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(1, 1, 60, TimeUnit.SECONDS, new ArrayBlockingQueue<>(10));
    threadPoolExecutor.execute(new Runnable() {
        int index = 1;
        @Override
        public void run() {
            while(true){
                System.out.println(index ++);
                if(index == 5){
                    int a = 1/0;
                }
                try{
                    Thread.sleep(1000); // 模拟定时执行
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        }
    });
}
```

代码运行结果：

```
1
2
3
4
Exception in thread "pool-1-thread-1" java.lang.ArithmeticException: / by zero
	at test.testAA$1.run(testAA.java:21)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
```

此刻呢，线程会一直处于waiting状态。但此刻和定时线程池不一样的时，此刻的线程名更改了，也就时原先哪个线程被清掉了，换了一个线程。此刻可以确定的是运行任务是和线程绑定的，因为任务是死循环，如果不抛异常就会一直执行，如果抛异常则会把任务和线程一并丢弃。

### 源码分析

先说总结，定时线程池ScheduledExecutorService其实并不是每次都隔一段时间往阻塞队列中丢任务，而是每次执行任务**成功**后都将任务重新投递到队列中，如果失败则不会重新投递任务到队列中的。此刻就是任务丢失的问题所在。

首先看提交任务的方法scheduleWithFixedDelay。主要的目的会把用户传入的Runnable对象封装到一个ScheduledFutureTask对象中。

```java
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                 long initialDelay,
                                                 long delay,
                                                 TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    if (delay <= 0)
        throw new IllegalArgumentException();
    ScheduledFutureTask<Void> sft =
        new ScheduledFutureTask<Void>(command,
                                      null,
                                      triggerTime(initialDelay, unit),
                                      unit.toNanos(-delay));
    // RunnableScheduledFuture 是ScheduledFutureTask的父类，这个decorateTask方法会直接返回sft
    RunnableScheduledFuture<Void> t = decorateTask(command, sft); 
    sft.outerTask = t;
    delayedExecute(t);//该方法会将RunnableScheduledFuture对象add到阻塞队列中
    return t;
}

ScheduledFutureTask(Runnable r, V result, long ns, long period) {
    super(r, result);//调用父类构造方法
    this.time = ns;
    this.period = period;
    this.sequenceNumber = sequencer.getAndIncrement();
}

public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result); //用户封装的Runnable对象被包装成Callable对象了。
    this.state = NEW;       // ensure visibility of callable
}
```

该方法delayedExecute主要的作用是将RunnableScheduledFuture对象add到阻塞队列中。

```java
private void delayedExecute(RunnableScheduledFuture<?> task) {
    if (isShutdown())
        reject(task);
    else {
        super.getQueue().add(task);
        if (isShutdown() &&
            !canRunInCurrentRunState(task.isPeriodic()) &&
            remove(task))
            task.cancel(false);
        else
            ensurePrestart();//这里是创建线程。
    }
}
```

当任务被放入队列后，线程创建完毕就会从任务队列中取出RunnableScheduledFuture对象，调用RunnableScheduledFuture对像的run方法。

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        // 主要就是这个getTask方法，获取刚刚add到阻塞队列中的RunnableScheduledFuture对象
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();// 调用RunnableScheduledFuture对象的run方法
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

先来看看ScheduledFutureTask对象内部的run方法。

```java
public void run() {
    boolean periodic = isPeriodic();
    if (!canRunInCurrentRunState(periodic))
        cancel(false);
    else if (!periodic)
        ScheduledFutureTask.super.run();
    else if (ScheduledFutureTask.super.runAndReset()) { // 这个方法会调用用户封装好的代码逻辑。
        // 这里if代码块中，只有runAndReset方法运行成功，即不抛异常则会返回true，抛异常则会返回默认值false
        setNextRunTime();// 设置下一次运行时间。入十秒的定时任务，则在当前时间戳的基础上加十秒
        reExecutePeriodic(outerTask); // 重点，这里会将任务重新add新阻塞队列中去
    }
}
```

先来看看定时任务内部实现的阻塞队列，是如何获取任务的。

```java
public RunnableScheduledFuture<?> take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            //拿到队列中的第一个对象
            RunnableScheduledFuture<?> first = queue[0];
            if (first == null)
                available.await();//如果为空，则等待
            else {
                long delay = first.getDelay(NANOSECONDS);//距离下一次执行时间还剩多少时间戳。
                if (delay <= 0)
                    return finishPoll(first);//如果小于0则直接pop出阻塞队列中。
                first = null; // don't retain ref while waiting
                if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay);//将线程休眠，等待下一次触发时间到来。
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && queue[0] != null)
            available.signal();
        lock.unlock();
    }
}
```

ScheduledExecutorService是基于内部的阻塞队列DelayedWorkQueue实现的定时任务，在任务执行结束后，会将任务重新丢入阻塞队列中并计算下一次的执行时间戳，如果在获取任务的时候未到达下一次触发任务的时间戳，则休眠线程等待下一次可执行的时间戳到来。

**建议：**在使用只有一个线程的普通线程池去循环处理任务或者使用定时线程池的时候，一定要确保异常情况的处理，如果异常未处理，则很有可能会出现任务丢失。从而造成bug事故。

