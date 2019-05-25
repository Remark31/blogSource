---
title: Java并发编程(九)
date: 2019-05-25 20:41:13
tags: [java,并发]
---

# 写在前面的话

上周缺文的原因是加班. 这周参加了培训，感觉巨累无比

本章的内容主要是死锁，核心解决方案其实就两个，1.保证锁的调用顺序相同；2.使用开放性调用，让加锁的部分变小

# 避免活跃度危险

> 滥用锁会导致锁顺序死锁

# 死锁

> 死锁：一个线程永远占有一个锁，而其它线程尝试去获得这个锁，那么它们将永远被阻塞。

## 锁顺序死锁

> 如果所有的线程都以同样的顺序获得锁，程序就不会出现锁顺序死锁的问题

## 动态的锁顺序死锁

```java
public void transferMoney(Account fromAccount, Account toAccount, DollarAmount amount) throws InsufficientFundsException {
    synchronized(fromAccount){
        synchronized(toAccount){
            if(fromAccount.getBalance().compareTo(amount) < 0) throw new InsufficientFundsException();
            else{
                fromAccount.debit(amount);
                toAccount.credit(amount);
            }
        }
    }
    
}
```

如上代码锁的顺序依赖于传参的顺序，极易发生死锁

为了解决这种问题，必须制定锁的顺序

- 使用对象的hash来判断顺序
- 如果hash相同，则使用一个额外的锁来做"加时赛"

## 协作对象间的死锁

> 持有锁时调用外部方法会增加活跃度问题。

## 开放调用

> 当调用的方法不需要持有锁时，被称为开放调用

使用更小的synchronized块来替代synchronized方法

## 资源死锁

- 当线程的等待目标变为资源时，就可能出现类似的死锁
- 线程饥饿死锁(如上一章提到的长期等待i/o的线程池)


# 避免和诊断死锁

- 首先识别什么地方会获取多个锁
- 确保锁的顺序一致
- 尽可能使用开放调用

## 尝试定时的锁

使用显式Lock类中的定时tryLock特性(可以定义超时时间)


## 通过线程转储来分析死锁

- 在Unix中使用 kill -3 <pid>
- https://www.cnblogs.com/huangfox/p/3442746.html


# 其它的活跃度危险

## 饥饿

> 当线程访问它需要的资源时被永久拒绝，以至于不能继续运行，这就发生了饥饿

尽量不要使用线程优先级，这个容易导致饥饿

## 弱响应性

锁被长时间占用，其它的程序就可能出现弱响应性

## 活锁

> 尽管没有被阻塞，但是线程不能被继续，因为重复相同的操作，但是总是失败

解决活锁方案是对重试机制尝试引入一定的随机性


