---
title: Java并发编程(二)
date: 2019-03-24 17:20:07
tags: [Java,并发]
---

# 写在前面的话
本章是Java并发编程的第二章，线程安全，核心介绍了两个内容`Atomic`与`synchronized`

整体来说内容还是都接触过的，比较简单，还是吐槽一下，Java的线程启动并没有go的协程那么方便

不过直接对于对象的锁用起来还是挺爽的，但是go里面直接在struct里填一个lock其实也很方便

整体来说还没有什么醍醐灌顶的感觉，都属于已经知道的东西，希望未来会有更多新东西。

# 线程安全

> 编写线程安全的代码，本质上就是管理对状态的访问，而且通常都是共享的，可变的状态

> 首先让代码正确，然后让它跑得更快


# 什么是线程安全

> 当多个线程访问一个类时，如果不考虑线程在运行时环境下的调度和交替运行，并且不需要额外的同步及调用方代码不用做其它的协调，这个类仍然行为正确，就被称为线程安全

无状态的对象永远线程安全

# 原子性

a ++ 并非是原子操作

## 竞争条件

检测后再运行的问题，检测时的条件到运行时已经不再成立，导致运行时错误

### 使用已有的线程安全类

`AtomicLong`：原子递增

# 锁

虽然每个操作都是原子性的，但是组合起来并不能保证整体是线程安全的。

> 为了保护状态的一致性，要在单一的原子操作中更新相互关联的状态变量

## 内部锁

### synchronized

- synchronized方法的锁，是该方法所在变量的本身
- 静态synchronized方法的锁来自于Class对象(唯一)

内部锁是互斥的(mutex)

### 可重入

内部锁是可重入的。

实现方法：通过为每个锁关联一个请求计数和占用的线程，当线程请求锁时，JVM将会记录线程并且记录次数，如果该线程再次请求，计数+1，释放时计数-1，当计数为0时整个锁释放

例如

``` java

package javaconcurrency;

public class SyncDemo {

    private int a;

    public synchronized void increaseA() {
        this.a = a + 1;
    }

    public synchronized void increaseA(int num) {
        this.a = a + num;
        this.increaseA();
    }


    public void getA(){
        System.out.println("this a = " + a);
    }

    public static void main(String[] args) throws InterruptedException {
        int num = 10000;
        Thread[] threads = new Thread[num];

        SyncDemo syncDemo = new SyncDemo();

        for (int i = 0 ; i < num ; i ++){
            threads[i] = new Thread(){
                public void run(){
                    syncDemo.increaseA(10);
                }
            };
            threads[i].start();

        }


        for (int i = 0 ; i < num ; i ++){
            threads[i].join();
        }


        syncDemo.getA();
    }

}

```

如果内部锁是不可重入的，那么就会造成死锁

## 用锁来保护状态

> 并非只有在写入时才需要同步

**每个共享变量都需要用由唯一一个确定的锁来保护**

**对于每一个涉及多个变量的不变约束，需要同一个锁保护所有变量**

## 活跃度与性能

![弱并发](/imgs/javacon_sickconcurrency.png)

如果同步策略是同步整个方法，那么并发的方式会如上所示。

会导致整个系统的吞吐量降低。

决定synchronized的大小需要权衡各种设计，包括安全，简单和性能

线程长时间占用锁会有问题

```
耗时操作，例如网络I/O等，这些操作期间不要占用锁
```


