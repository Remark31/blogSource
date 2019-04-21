---
title: Java并发编程(六)
date: 2019-04-21 14:37:33
tags: [Java,并发]
---

# 写在前面的话

本章主要的内容其实就是Executor。

这里是Java与Golang的区别，Golang是协程，创建和销毁开销很小，不需要特别关注，Java是真正的线程，这里的创建和销毁开销不少，并且线程的上限是远远比协程低的的。因此引入了线程池这个概念。

本章介绍了线程池的优势，各种用法，例如周期任务，各种不同的线程池，携带结果的线程池等等，理解和使用并不困难。

# 任务执行

## 在线程中执行任务

**确定任务的边界**
> 理想情况下，任务是独立的活动，它的工作不依赖于其它任务的状态、结果或者边界效应。

### 顺序的执行任务

```java
class SingleThreadWebServer{
    public static void main(String[] args) throws IOExceptions {
        ServerSocket socket = new ServerSocket(80);
        while(true){
            Socket connection = socket.accept();
            handleRequest();
        }
    }
}
```

顺序执行会让同一时刻只有一个任务在进程上执行，服务器利用率会较低。

P.S. 游戏服务器其实就是这样顺序的执行的。

### 显式的为任务创建线程

```java
class SingleThreadWebServer{
    public static void main(String[] args) throws IOExceptions {
        ServerSocket socket = new ServerSocket(80);
        while(true){
            Socket connection = socket.accept();
            Runnable task = new Runnable() {
                public void run(){
                    handleRequest(connection);
                }
            }
            new Thread(task).start();
        }
    }
}
```

只要请求的到达速度没有达到服务器处理速度的上限，这种方法可以带来更快的响应和更大的吞吐量

### 无限制创建线程的缺点

- 线程生命周期的开销：为每个请求创建线程会带来消耗大量的计算资源
- 资源消耗量：如果已经有了足够多线程保持所有CPU忙碌，那么创建更多的线程是有百害而无一利的。
- 稳定性：需要限制创建线程的数量，否则可能导致OutMemoryError.

> 在32位机上主要限制因素是地址空间，每个线程都维护两个执行栈。典型的JVM会产生一个组合栈，大小大概半MB。

## Executor框架

Executor基于生产者-消费者模式，提交任务的执行者是生产者，执行任务的线程是消费者。

### 示例：使用Executor实现简单的Web Server

```java
class TaskExecutionWebServer{
    private static finnal int NTHREAD = 100;
    private static finnal Executor exec = Executors.newFixedThreadPool(NTHREAD);
    
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while(true){
            finnal Socket connection = socket.accept();
            Runnable task = new Runnable(){
                public void run(){
                    handleRequest(connection);
                }
            };
            exec.execute(task);
        }
    }
}

// 为每个请求开辟一个线程
public class ThreadPerTaskExecutor implements Executor{
    public void execute(Runnable r){
        new Thread(r).start();
    }
}

// 让每个请求顺序执行
public class WithinThreadExecutor implements Executor{
    public void execute(Runnable r){
        r.run();
    }
}
```

## 执行策略

执行策略确定了任务的执行因素：
- 任务在什么线程中执行
- 任务以什么顺序执行
- 可以有多少个任务并发执行
- 可以有多少个任务进入等待执行队列
- 如果系统过载，需要放弃一个任务，应该挑选哪一个任务？然后如何通知系统？
- 在一个任务执行前和结束后，该执行什么操作

> 无论如何，当看到new Thread(Runnable).start();这种代码时，请将它们替换成Executor

## 线程池

在线程池中执行任务的优势：
- 重用存在的线程，可以抵消线程创建和消亡的过程
- 任务到来时线程已经存在，提高了响应性
- 适当的调整线程池的大小，还可以防止线程的竞争。

可以调用Executor中已经存在的方法来创建线程池
- newFixedThreadPool:创建定长线程池
- newCachedThreadPool:创建带缓存的线程池
- newSingleThreadExecutor:单线程化的Executor.
- newScheduledThreadPool:创建定长的线程池，支持定时的以及周期性的任务执行。

## Executor的生命周期

- 运行
- 关闭
- 终止(平缓关闭和强制关闭)
    - shutdown
    - shutdownnow

## 延迟的，具有周期性的任务

Timer本身的问题：
- 只使用唯一的线程来执行任务：如果其中一个任务很耗时，那么其它的任务会受到影响。
- 如果TimerTask抛出未受检查的异常，会产生无法预料的结果。(Timer被取消，无法恢复，已经被安排的任务也不会被执行。**线程泄露**)

此处使用newScheduledThreadPool作为替代品

# 寻找可强化的并行性

单一的客户请求内部仍然有进一步细化的并行性

## 可携带结果的任务 Callable和Future

- Callable：可以返回一些结果
- Future：描述了任务的周期，可以获取任务的结果，取消任务以及检验任务已经被完成或者是被取消。

将Runnable、Callable和Future提交到Executor可以创建一个安全发布。

## 并行运行异类任务的局限性

为了使划分任务是值得的，这一开销不能多于通过并行带来的性能提升。

> 大量相互独立且同类的任务进行并发处理。会将任务量分配到不同的任务中，这才能真正获得性能的提升。


## ExecutorCompletionService：Executor和BlockingQueue

向Executor提交了任务，然后希望完成时立即获得结果。

ExecutorCompletionService，完成时可以获得结果，当结果不可用时阻塞。

- take() 用于获取future任务
- poll() 用于获取下一个已经完成的任务，可以设置超时时间


# 总结

围绕任务的执行来构造应用程序，可以简化开发，便于同步。Executor可以有助于你在任务提交与任务的执行策略之间解耦，并且支持不同类型的执行策略。