---
title: Java并发编程(十一)
date: 2019-06-16 14:41:13
tags: [Java,并发]
---
# 写在前面的话

本章讲解的主要内容是如何测试并发程序，整体来看是如下几个步骤：
1. 先考虑无并发的顺序性执行的正确性
2. 对阻塞操作进行测试
3. 引入随机进行并发测试其安全性
4. 资源管理方面的测试，防止内存不合理占用
5. 性能测试

接下来提出了测试中需要关注的一些问题，例如GC，代码动态编译等等。
最末提出了一些补充测试的方法。整体述求还是如何写出正确的并发代码。

# 测试并发程序

- 安全性测试：验证不变约束
- 活跃度测试：应该“执行”和不应该“执行”
- 性能测试：
    - 吞吐量
    - 响应性
    - 可伸缩性

# 测试正确性

首先看下测试的基准程序，一个基于数组的定长序列

```java
package javaconcurrency.javatest;


import net.jcip.annotations.GuardedBy;
import net.jcip.annotations.ThreadSafe;

import java.util.concurrent.Semaphore;

@ThreadSafe
public class BoundedBuffer<E> {
    private final Semaphore availableItems, availableSpace;

    @GuardedBy("this")
    private final E[] items;

    @GuardedBy("this")
    private int putPosition, takePosition;

    public BoundedBuffer(int capcatiy){
        availableItems = new Semaphore(0);
        availableSpace = new Semaphore(capcatiy);
        items = (E[]) new Object[capcatiy];
    }

    public boolean isEmpty(){
        return availableItems.availablePermits() == 0;
    }

    public boolean isFull(){
        return availableSpace.availablePermits() == 0;
    }

    public void put(E v) throws InterruptedException{
        availableSpace.acquire();
        doInsert(v);
        availableSpace.release();
    }

    public E take() throws InterruptedException{
        availableItems.acquire();
        E item = doExtract();
        availableItems.release();
        return item;
    }

    private synchronized void doInsert(E x){
        int i = putPosition;
        items[i] = x;
        putPosition = (++i == items.length ) ? 0: i;
    }

    private synchronized E doExtract(){
        int i = takePosition;
        E x = items[i];
        items[i] = null;
        takePosition = (++i == items.length) ? 0 : i;
        return x;
    }

}

```


接下来开始各种测试

## 基本的单元测试

首先简单的验证下空和满的问题

```java
package javaconcurrency.javatest;

import org.junit.Test;

public class BounderBufferTest{


    @Test
    public void testIsEmptyWhenConstructed(){
        BoundedBuffer<Integer> boundedBuffer = new BoundedBuffer<>(10);
        assert (boundedBuffer.isEmpty() == true);
        assert (boundedBuffer.isFull() == false);
    }

    @Test
    public void testFullAfterPut() throws InterruptedException {
        BoundedBuffer<Integer> boundedBuffer = new BoundedBuffer<>(10);
        for(int i = 0 ; i < 10 ; i ++){
            boundedBuffer.put(i);
        }

        assert (boundedBuffer.isFull() == true);
        assert (boundedBuffer.isEmpty() == false);
    }
}

```

这种验证是验证顺序化的基本逻辑是否正确。


## 测试阻塞操作

```java
@Test                                                                             
public void tableBlockWhenEmpty() {                                               
    final BoundedBuffer<Integer> boundedBuffer = new BoundedBuffer<>(10);         
                                                                                  
    Thread taker = new Thread() {                                                 
        public void run(){                                                        
          try{                                                                    
              int unused = boundedBuffer.take();                                  
              fail();                                                             
          } catch (InterruptedException e){                                       
                                                                                  
          };                                                                      
        }                                                                         
    };                                                                            
                                                                                  
    try{                                                                          
        taker.start();                                                            
        Thread.sleep(1000);                                                       
        taker.interrupt();                                                        
        taker.join(1000);                                                         
        assert (taker.isAlive() == false);                                        
    } catch (Exception unexcepted){                                               
        fail();                                                                   
    }                                                                             
                                                                                  
}                                                                                 
```

上述的测试用例测试了阻塞和恢复的情况。先从空的队列中取，这时候就阻塞了操作，最后再恢复。

## 测试安全性

要构建一个并发类在不可知的并发条件下能否正确执行，只能安排多个线程运行put和take操作


> 测试应该在多处理器上进行，并且让测试中的线程数大于CPU数。

## 测试资源管理

需要测试资源用尽的情况。当生产者远多于消费者时就会出现。

对内存不合理的占用，可以通过很多的堆工具测试出来。

## 使用回调

回调用户提供的代码，有助于创建测试用例。

向线程池提交一些长耗时的任务，可以测试线程池是否如期的增长

## 产生更多的交替操作

使用thread.yeild或者thread.sleep，产生更多的交替操作。


# 测试性能

在性能测试中包含一些基本的功能测试，这样可以确保做性能测试的代码都是正确的。

## 测试响应性

## 对比多个缓存方案

# 避免性能测试的陷阱

## 垃圾回收

垃圾回收是可能对测试造成误差的。

- 确保在整个运行期间都不进行垃圾回收(-verbose:gc)
- 确保垃圾回收再整个运行期间被执行多次

## 动态编译

抛弃第一组数据

## 代码路径非真实取样

## 不切实际的竞争程度

## 死代码的消除

# 测试方式补遗

## 代码审查

## 静态工具分析

## 面向方面的测试

## 统计与剖析工具






