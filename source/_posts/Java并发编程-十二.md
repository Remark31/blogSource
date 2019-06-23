---
title: Java并发编程(十二)
date: 2019-06-23 13:12:05
tags: [Java,并发]
---
# 写在前面的话

本章挺简单的，有golang的经历要理解这个lock非常的容易。这完全就是golang中的sync.Lock()。不过在Java中还是应该更可能的去使用synchronzied。

# 显式锁

# Lock接口

```java
public interface Lock{
    void lock();
    void lockInterruptiby() throws InterruptionException;
    boolean tryLock();
    boolean tryLock(long timeout, TimeUnit unit) throw InterruptedException;
    void unlock();
    Condition newCondition();
}
```

ReentranLock 实现了Lock接口，并且提供了域synchronized相同的保证

## 可轮询和可定时的锁请求

使用tryLock实现的。

## 可中断的锁的获取

```java
public boolean sendOnSharedLine(String message) throws InterruptedException{
    lock.lockInterruptibly();
    try{
        return cancellableSendOnSharedLine(message);
    } finally{
        lock.unlock();
    }
    
}

private boolean cancellableSendOnSharedLine(String message) throws InterruptedException{
    ...
}
```

注意，lockInterruptibly需要有两层try。

## 非结构化的锁

在内部锁中，加锁和释放都是在同样的块结构中，有时候需要更灵活的使用，就需要使用Lock

# 对性能的考量

内部锁和 ReentranLock 在性能上几乎一致。

# 公平性

ReentranLock 提供了公平锁和非公平锁

在大多数情况下，非公平锁的性能高于公平锁

原因在于：

挂起的线程在被唤醒时和真正运行时之间有严重的延时。

但是当持有锁的时间较长，或者请求锁的时间间隔较长时，使用公平锁是比较好的。

# synchronzied 和 ReentranLock 的比较

> 当内部锁不满足需求时，才应该使用 ReentranLock，例如如下的高级需求：可定时的，可轮询的与可中断的锁获取操作，公平队列或者非块结构的锁。

内部锁的另一个优点：线程转储能够显示哪个调用框架获取了锁，并且是基于JVM的，未来可能更优化。


# 读写锁

```java
public interface ReadWriteLock(){
    Lock readLock();
    Lock writeLock();
}
```

读写锁由 ReentransLock实现下需要注意的一些要点：

- 释放优先：选择权交给等待最长的线程。
- 读者闯入：如果有写者要获取锁，就不允许新的读者获取锁。
- 重进入：是可重入的。
- 降级：允许写者降级为读者。
- 升级：不允许读者升级为写者。
- 公平性：是公平锁。


