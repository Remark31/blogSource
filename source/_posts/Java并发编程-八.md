---
title: Java并发编程(八)
date: 2019-05-11 15:44:05
tags: [Java,并发]
---


# 写在前面的话

本章是Java并发编程第八章。整体在讲线程池的一些应用，例如调整线程池的配置，线程池的线程创建，队列管理，饱和策略，线程池的before，after的钩子。最末讲解了并行递归算法。

整体来说还是比较简单，并行递归之后应该找个更有趣的例子来尝试实现一下。

身体是革命的本钱，本文拖欠了一周主要原因是因为上周肺炎发作。好好歇息。

# 应用线程池

# 任务与执行策略间的隐性耦合

有不少任务是明确需要不同的执行策略的

- 依赖性任务：任务依赖于其它任务，可能导致活跃度的问题
- 采用线程限制的任务：需要单线程化的执行策略，如果是线程池就会有问题
- 对响应时间敏感的任务：多个长时间执行的任务放在了少线程的线程池中
- 使用ThreadLocal的任务：在线程池中，不应该用ThreadLocal传递任务间的数值

当任务都是同类的，独立的，线程池才能发挥最大作用

## 线程饥饿死锁

> 线程池饥饿死锁：只要池任务开始了无限期的阻塞，其目的是等待一些资源或条件，此时只有另一个池任务的活动才能使那些条件成立，如果池子不够大，就会发生线程池饥饿死锁。

## 耗时操作

**限定等待资源的时间**:无论任务是成功还是失败，该方法保证了任务总会向前发展

# 定制线程池的大小

池的长度很少会硬编码，都是通过某些配置或者利用`Runtime.availableProcessors`的结果，进行动态计算

只需要避免过大或者过小这两种极端的情况


# 配置ThreadPoolExecutor

```java
public ThreadPoolExecutorDemo(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);
    }
```

## 线程的创建与销毁

- corePoolSize：核心池大小
- maximumPoolSize：最大池大小
- keepAliveTime：存活时间

## 管理队列任务

ThreadPoolExecutor允许提供一个BlockingQueue来持有等待执行的任务，任务排队有三种方式：
- 无限队列
- 有限队列
- 同步移交

newFixedThreadPool、newSingleTheadExecutor默认采用无限队列

稳妥管理使用有限队列：ArrayBlockingQueue或者有限的LinkedBlockingQueue及PriorityBlockingQueue

庞大或者无限池，可以使用SynchronousQueue

## 饱和策略

当一个有限队列被充满时，饱和策略将会生效，可以通过`setRejectedExecutionHandler`来修改，JDK提供了几种默认的饱和策略

- AbortPolicy：抛出未检查的异常，可以处理异常
- DiscardPolicy：默认放弃新提交的任务
- DiscardOldestPolicy：丢弃目前下一个将要执行的任务(不能和优先级队列混合使用)
- CallerRunsPolicy：把任务推回给调用者执行，以减缓任务流

## 线程工厂

线程池创建线程都是通过线程工厂来实现

## 构造后再定制ThreadPoolExecutor

ThreadPoolExecutor在构造函数中所需要使用的参数，都可以使用setters进行调整

```java
ExecutorService exec = Executors.newFixedThreadPool(10);
if (exec instanceof ThreadPoolExecutor){
    ((ThreadPoolExecutor) exec).setCorePoolSize(100);

} else{
    throw new AssertionError("Oops, bad assumptions ");
}
```

# 扩展 ThreadPoolExecutor

ThreadPoolExecutor 提供了一些钩子用来扩展行为，例如beforeExecutor,afterExecutor和terminate

# 并行递归算法

顺序递归
```java
 public<T> void sequentialRecursive(List<Node<T>> nodes, Collection<T> results){
        for (Node<T> n : nodes){
            results.add(n.compute());

            sequentialRecursive(n.getChild(), results);
        }
    }

```

并行递归
```java
public<T>  void parralelRecursive(final Executor exec, List<Node<T>> nodes, Collection<T> results){
        for (Node<T> n : nodes){
            exec.execute(new Runnable() {
                @Override
                public void run() {
                    results.add(n.compute());
                }
            });

            parralelRecursive(exec, n.getChildren(), results);
        }

    }
```



