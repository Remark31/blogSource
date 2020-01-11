---
title: JAVA并发编程(十三)
date: 2020-01-11 15:13:42
tags: [Java,并发]
---

# 写在前面的话

时隔很久的一篇文章，感觉都要淡忘了。
本章主要在分析synchronizer的实现，读得很晦涩，感觉近期不太适合读这种文章... 

# 构建自定义的同步工具

# 管理状态依赖性

通常的可阻塞状态的代码如下所示

```java
void blockingAction() throws InterruptedException{
    acquire lock
    while(codition is not hold){
        release lock
        wait until condition hold
        option failed
        reacquire lock
    }
    perform hold
    reacquire lock
}
```

## 示例

有限缓存的条件队列

```java
public synchronized void put(V v) throw InterruptException{
    while (isFull()){
        wait();
    }
    doPut(v);
    notifyAll();
} 
```

此处还可以使用一个时限版本的的wait，用于超时返回

# 使用条件队列

## 识别条件谓词

条件谓词是先验条件的第一站，它在一个操作与状态之间建立起依赖关系。

例如，在有限缓存中，只有缓存不为空，才能执行take操作

## 过早的唤醒

在被唤醒后，还需要再检查一下条件谓词

- 永远设置一个条件谓词
- 永远在调用wait前检查条件谓词，并在wait返回后再次测试
- 永远在循环中调用wait
- 确保构成条件谓词的状态变量被锁所保护
- 当调用wait,notify或者notifyAll时，要持有与条件队列相关的锁，并且在条件谓词检查与完成任务之间不要释放锁

## 丢失的信号

严格按照规范，不会出现丢失信号

## 通知

> 当执行wait时，一定要确保有人会notify

只有满足下述条件时，notify才会优于notifyAll

- 相同的等待者
- 只有一个条件谓词与条件队列相关
- 每个线程从wait后返回相同的逻辑
- 一进一出
- 一个对条件变量的通知，至多只执行一个线程


一种对notifyAll的优化是依据条件的通知

```java
public synchronized void put(V v) throws InterruptedException{
    while(isFull()){
        wait();
    }
    bool wasEmpty = isEmpty();
    doPut(v);
    if(wasEmpty){
        notifyAll();
    }
} 
```

## 示例：阀门类

一个可重关闭的ThreadGate

```java
@ThreadSafe
public class ThreadGate {
    @GuardedBy("this")
    private boolean isOpen;

    @GuardedBy("this")
    private int generation;

    public synchronized void close() {
        isOpen = false;
    }

    public synchronized void open() {
        generation++;
        isOpen = true;
        notifyAll();
    }

    public synchronized void await() throws InterruptedException {
        int arrivalGenerataion = generation;
        while (!isOpen && arrivalGenerataion == generation) {
            wait();

        }
    }
}
```

## 子类的安全问题

> 一个依赖于状态的类，要么完全将文档和等待通知文档化给子类，要么设置为final禁止子类化

## 封装条件队列

通常，最好是将条件队列封装起来，这样在使用它时不会被非法使用。

## 入口协议和出口协议

> 入口协议：操作的条件谓词

> 出口协议：检查任何被操作改变的条件状态变量，确认它是否引起其他一些条件谓词变为真，如果是，通知相关的条件队列。

# 显式的condition对象

```java
public interface Condition{
    void await() throws InterruptedException;
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    boolean awaitUntil(Date deadline) throws InterruptedException;
    
    void signal();
    void signalAll();
}
```

# 剖析Synchronizer

一个重要的基类 AbstractQueuedSynchronizer（AQS），构建锁和同步的一个框架





