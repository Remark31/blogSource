---
title: Java并发编程(七)
date: 2019-04-27 21:21:20
tags: [Java,并发]
---

# 写在前面的话

本章主要描述的是线程的中途退出和关闭的问题，普通中断，阻塞中断以及基于线程的服务的停止是主要的讨论点，最后的JVM关闭对于现在我的知识储备来说还是晦涩了一点，等看完JVM那本书之后再回头来看可能会有更好的体验。

线程的中途退出的基本理念还是清楚的，只是很多阻塞操作会导致只能使用异常来处理中断。

尽量还是多使用future和executor来处理中断，会让代码逻辑变简单很多。



# 取消和关闭

Java没有提供任何机制强迫线程安全的停止工作。提供了**中断**，使一个线程能够让另一个线程停止当前工作。

停止的流程：
- 清除当前进程中的工作
- 然后再终止

# 任务取消

> 当外部代码能够在活动自然完成前，改变活动的状态，那么这个活动被称为可取消的。

一种协作机制是设置标志位，周期性查看标志位的状态，当标志位状态改变，就停止。

```java
public class PrimeGenerator implements Runnable {
    @GuardedBy("this")
    private final List<BigInteger> primes = new ArrayList<>();

    // cancelled一定要是volatile的，不然可能不可见
    private volatile boolean cancelled;

    @Override
    public void run() {
        BigInteger p = BigInteger.ONE;
        while (!cancelled) {
            p = p.nextProbablePrime();
            synchronized (this) {
                primes.add(p);
            }
        }
    }

    public void cancel() {
        this.cancelled = true;
    }


    public synchronized List<BigInteger> getPrimes() {
        return new ArrayList<BigInteger>(primes);
    }


    public static void main(String[] args) throws InterruptedException {
        PrimeGenerator primeGenerator = new PrimeGenerator();
        new Thread(primeGenerator).start();

        try {
            TimeUnit.SECONDS.sleep(2);
        } finally {
            primeGenerator.cancel();
        }

        System.out.println(primeGenerator.getPrimes());

    }
}
```


一个可取消的任务一定要具有取消策略
- 其它代码如何请求取消该任务
- 任务在什么时候检查取消的请求是否到达
- 响应取消请求的任务中应有的行为。

对于上面的例子，取消策略如下：
- 使用cancelled来请求取消任务
- 通过每次生成前检查请求来确定到达
- 任务取消的行为为直接停止

## 中断

如果任务调用了阻塞方法，有可能会导致任务永远检查不到取消标志，导致不能停止

此时应该使用特定阻塞库方法支持中断。

```java
public class Thread{
    public void interrupt(){}
    public boolean isInterrupted(){}
    public static boolean interrupted(){} // 小心使用，会清除并发程序的中断状态
}
```

> 中断方法并不意味停止了线程目前的工作，只是传递了一个请求中断的消息

对于阻塞方法，应该使用如下方式来中断

```java
public class PrimeGeneratorBlocking extends Thread {
    private final BlockingDeque<BigInteger> queue;
    public PrimeGeneratorBlocking(BlockingDeque<BigInteger> queue){
        this.queue = queue;
    }

    public void run(){
        try{
            BigInteger p = BigInteger.ONE;
            while (!Thread.currentThread().isInterrupted()){
                queue.put(p = p.nextProbablePrime());
            }
        } catch (InterruptedException e){

        }
    }

    public void cancell(){
        this.interrupt();
    }
}
```

## 中断策略

区分任何和线程对中断的反应是很重要的。一个单一的中断请求可能有一个或一个以上的预期接收者。

> 因为每个线程都有自己的中断策略，所以不应该中断线程，除非你明确。

## 响应中断

- 传递异常
- 保持中断状态，让上层调用的代码能够调用

```java
public Task getNextTask(BlockingQueue<Task> queue){
    boolean interrupted = false;
    try{
        while(true){
            try{
                return queue.take();
            } catch (InterruptedException e){
                interrupted = true;
            }
        }
        
    } finnal {
        if (interrupted){
            Thread.currentThread().interrupt();
        }
    }
}
```

## 通过Future取消

计时任务，通过定时的Future来获得结果，如果get终止于一个TimeoutException，任务就通过Future来取消。

```java
public class TimedRun {
    private static final ScheduledExecutorService taskExec = Executors.newScheduledThreadPool(100);

    public static void timedRun(Runnable r, long timeout, TimeUnit unit) throws InterruptedException {
        Future<?> task = taskExec.submit(r);
        try {
            task.get(timeout, unit);
        } catch (TimeoutException e){

        } catch (ExecutionException e){
            throw launderThrowable(e.getCause());
        } finally {
            task.cancel(true);
        }
    }
}
```

## 处理不可中断阻塞

- java.io中的同步Scoket I/O：可以通过关闭底层Socket，让read/write所阻塞的线程抛出一个SocketException
- java.nio中的同步I/O：关闭一个InterruptibleChannel可以导致多个阻塞在链路上的线程抛出AsynchronousCloseException
- Selector的异步I/O：close方法可以抛出ClosedSelectorException
- 获得锁：显示锁提供了lockInterruptibly方法，允许等待获得锁并且响应中断。


可以在线程中覆写interrupt来封装非标准取消。

# 停止基于线程的服务

> 对于线程持有的服务，只要服务的存在时间大于创建线程方法的存在时间，那么就应该提供生命周期的方法。

## 日志系统的关闭

```java
public class LogService {
    private final ExecutorService exec = newSingleThreadExecutor();

    public void start() {

    }

    public void stop() throws InterruptedException {
        try {
            exec.shutdown();
            exec.awaitTermination(TIMEOUT, UNIT);
        } finally {
            writer.close();
        }
    }

    public void log(String msg) {
        try{
            exec.execute(new WriterTask(msg));

        } catch (RejectedExecutionException e){

        }
    }
}
```

## 使用致命药丸

> 一个可识别的对象，放在队列中，收到时停止一切工作。 生产者在发送致命药丸后不再放置任何活动，消费者收到后停止工作。


## 只执行一次的服务

使用局部Executor来简化服务的生命周期

```java
public boolean checkMail(Set<String> hosts, long timeout, TimeUnit unit) throws InterruptedException {
        ExecutorService exec = Executors.newCachedThreadPool();
        final AtomicBoolean hasNewMail = new AtomicBoolean(false);
        try{
            for (final String host: hosts){
                exec.execute(new Runnable() {
                    @Override
                    public void run() {
                        if (checkMail(host)){
                            hasNewMail.set(true);
                        }
                    }
                });
            }
        } finally {
            exec.shutdown();
            exec.awaitTermination(timeout,unit);
        }
        return hasNewMail.get();
    }
```

# 处理反常的线程终止

> 在一个长时间运行程序中，所有的线程都要给未捕获异常设置一个处理器，这个处理器至少要异常信息写入日志中。


# JVM关闭

## 关闭钩子

关闭钩子是使用Runtime.addShutdownHook注册的尚未开始线程。

## 守护线程(daemon thread)

JVM启动时创建的线程，除了主线程以外，都是守护线程，主线程创建的线程都是普通线程。

差异：
- 当一个线程退出时，如果剩下的只有守护线程，那么就会正常退出，剩下的守护线程都会被抛弃


> 守护线程不能替代服务对生命周期的良好管理。

## Finalizer
> 避免使用

