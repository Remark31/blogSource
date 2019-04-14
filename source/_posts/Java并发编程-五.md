---
title: Java并发编程(五)
date: 2019-04-14 18:17:53
tags: [Java,并发]
---

# 写在前面的话

本章的内容很有意思，主要讨论了如下问题：

- 同步容器：这个就是之前在golang中最常用的使用lock来做的容器，所有操作都持有一个锁。
- 并发容器：使用的分离锁，允许更高的并发访问，也允许同步修改，但是如果一个容器需要独占，则不能胜任
- 生产者消费者模式：这个在golang中非常常见了
- 阻塞和可中断方法：提供了两种对于阻塞和可中断的处理，本质上都是抛给上层或者让上层感知到，去处理
- 调节系统线程流，用于让线程步骤更加一致，和之前golang中的sync.waitgroup的感觉一致。

最后给出了一个高速缓存的例子，让人直观的感受了并发的构建，内容还是不少，比之前golang中的并发更加系统多了。

# 构建块

## 同步容器

- Vector和Hashtable
- 使用Collections.synchronizedxxx工厂方法创建的容器

### 同步容器中的问题

同步容器是线程安全的，但是对于复合操作，就需要有客户端锁来保护

例如对同一个容器的删除和获取操作

```java
public static Object getLast(Vector list){
    int lastIndex = list.size() - 1;
    return list.get(lastIndex);
}

public static void deleteLast(Vector list){
    int lastIndex = list.size() - 1;
    list.remove(lastIndex);
}
```

可能出现get时被删除了，导致的list越界

最合适的办法是客户端加锁

```java
public static Object getLast(Vector list){
    synchronized(list) {
        int lastIndex = list.size() - 1;
        return list.get(lastIndex);
    }
    
}

public static void deleteLast(Vector list){
    synchronized(list) {
        int lastIndex = list.size() - 1;
        list.remove(lastIndex);
    }
}
```

### 迭代器和ConcurrentModificationException

迭代器在察觉到容器被修改时，会抛出一个ConcurrentModificationException的错误。这是通过将修改计数器和容器关联起来来实现的(有点乐观锁的感觉)


迭代期间对容器加锁可能导致等待时间变长，甚至死锁。

一种替代方式是复制容器。


### 隐藏迭代器

有些迭代器是隐藏的发生的，例如字符串的拼接以及容器的toString()方法。

最好的做法是将最初的对象封装成同步的。这样能避免这种隐藏的问题。


## 并发容器

### ConcurrentHashMap

同步类容器在每个操作执行时都持有一个锁，某些较长的操作会导致持有锁的时间很长。

ConcurrentHashMap使用的是分离锁。可以允许更深层次的共享访问。

ConcurrentHashMap返回的迭代器具有弱一致性，容许并发修改，可以感应到在迭代器被创建后对容器的修改。

只有当程序需要独占访问ConcurrentHashMap时，才不能胜任。

### Map附加的原子操作

- 缺少即加入
- 相等便移除
- 相等便替换


### CopyOnWriteArrayList

是同步list的一个替代品，提供了更好的并发性，并且避免了在迭代期间对容器加锁。

“写入时复制”容器线程安全性来自于“只要有效不可变对象被安全发布，那么访问它不需要更多同步”。每次访问时会创建并发布一个新的容器拷贝，以此来实现可变性。

复制基础数组需要开销。当对容器修改频率远低于操作频率时，可以使用写入时复制的容器。

## 阻塞队列和生产者消费者模式

阻塞队列提供了可阻塞的get和take方法。

生产者只负责生产数据，不关心数据的使用，消费者只关心使用数据，不关心数据的生产。


### 双端队列和窃取工作

Deque和BlockingDeque

双端队列适用于窃取者模式：

每个消费者都消费自己的双端队列，如果完成了自己的，可以去从别人的队尾获取任务执行。


## 阻塞和可中断方法

线程可能被多种原因阻塞或暂停：
- 等待i/o操作
- 等待一个锁
- 等待sleep

中断是一种协作机制，线程A要求线程B在某个方便的点，停止正在做的事情。

相应中断通常有两种选择：
- 传递：抛给上一层来处理
- 恢复：不能抛出时，就使用中断恢复机制，让上层知道。


```java
public class TaskRunnable implements Runnable {
    BlockingQueue<Task> queue;
    
    public void run(){
        try{
            processTask(queue.take());
        } catch(InterruptException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```


## Synchronizer

Synchronizer是一个对象，根据本身的状态调节线程的控制流。

常见的有：
- 信号量(semaphore)
- 关卡(barrier)
- 闭锁(latch)


### 闭锁

可以延迟线程的进度到线程终止。一旦闭锁达到终点状态，就不能改变状态了。常见的使用场景：

- 确保一个计算不会被执行，直到所需要的资源被初始化
- 确保一个服务不会开始，直到所依赖的服务都已经开始
- 等待，直到活动的所有部分都为继续处理做好准备。


```java
public class TestHarness {
    public static long timeTasks(int nThreads, final Runnable task) throws InterruptedException {
        final CountDownLatch startGate = new CountDownLatch(1);
        final CountDownLatch endGate = new CountDownLatch(nThreads);

        for (int i = 0 ; i < nThreads; i ++){
            Thread t = new Thread(){
                public void run(){
                    try{
                        startGate.await();
                        try {
                            task.run();
                        } finally {
                            endGate.countDown();
                        }

                    } catch (InterruptedException e) {

                    }
                }
            };
            t.start();
        }

        long start = System.nanoTime();
        startGate.countDown();
        endGate.await();
        long end = System.nanoTime();
        return end - start;

    }
}
```

可以看到，创建了一个开始阀门和结束阀门，当所有线程初始化完成时一起使用开始阀门，所有结束时使用结束阀门，这样能计算所有线程的使用时间。

### FutureTask

FutureTask等价于一个可携带结果的Runnable，并且有3个状态：等待、运行和完成。完成包括所有计算以任意方式结束，包括正常结束，取消和异常。

### 信号量

用来控制能够访问某特定资源的活动的数量，或者同时执行某一给定操作的数量。

release用来释放一个许可，acquire用来获得一个许可。

可以使用 信号量来控制一个容器的边界

```java
public class BoundSet<T> {
    private final Set<T> set;
    private final Semaphore semaphore;

    public BoundSet(int bound){
        this.set = Collections.synchronizedSet(new HashSet<T>());
        this.semaphore = new Semaphore(bound);
    }

    public boolean addSet(T o ) throws InterruptedException {
        semaphore.acquire();
        boolean wasAdded = false;
        try{
            wasAdded = set.add(o);
            return wasAdded;
        } finally {
            if(!wasAdded){
                semaphore.release();
            }
        }
    }

    public boolean removeSet(Object o){
        boolean wasRemoved = set.remove(o);
        if(wasRemoved){
            semaphore.release();
        }
        return wasRemoved;
    }
}
```

### 关卡

关卡类似于闭锁，都能阻塞线程，关卡等待的是其它线程。

关卡通常用来模拟如下情况：一个步骤的计算可以并行完成，但是要求完成所有相关的工作才能进行下一步。

CyclicBarrier是一种具体的实现。

## 为计算结果建立高效，可伸缩的高速缓存

```java
public class Memozier<A,V> implements Computable<A,V>  {
    private final ConcurrentHashMap<A, Future<V>> cache = new ConcurrentHashMap<>();
    private final Computable<A,V> compute;

    public Memozier(Computable<A,V> c) {
        this.compute = c;
    }

    @Override
    public V compute(A arg) throws InterruptedException {
        while(true){
            Future<V> future = cache.get(arg);
            if(future == null) {
                Callable<V> eval = new Callable<V>() {
                    @Override
                    public V call() throws InterruptedException {
                        return compute.compute(arg);
                    }
                };
                FutureTask<V> ft = new FutureTask<>(eval);
                future = cache.putIfAbsent(arg, ft);
                if (future == null) {
                    future = ft;
                    ft.run();
                }
            }
            try{
                return future.get();
            } catch (CancellationException e) {
                cache.remove(arg);
            } catch (ExecutionException e) {
                throw launderThrowable(e.getCause());
            }
        }
    }

    public static RuntimeException launderThrowable(Throwable t) {
        if (t instanceof RuntimeException) {
            return (RuntimeException) t;
        } else if(t instanceof Error){
            throw  (Error) t;
        } else{
            throw new IllegalStateException("Not unchecked" , t);
        }
    }


}
```

- 使用ConcurrentHashMap增加并发性，提升效率
- 使用future避免重复计算
- 使用putIfAbsent避免两者同时建立计算缓存
- 使用cache.remove避免缓存污染


# 第一部分小结

- 所有并发问题都可以归为如何协调访问并发状态，可变状态越少，保证线程安全就更容易
- 尽量将域声明为final，除非是可变的
- 不可变对象天生线程安全
- 封装使管理复杂度变得更可行
- 用锁来守护每一个可变对象
- 对同一约束的所有变量都使用相同的锁
- 在运行复合操作期间持有锁
- 非同步的多线程情况下，访问可变变量的程序是具有隐患的
- 不要依赖于可以需要同步的小聪明
- 在设计过程中就考虑线程安全，或者在文档中明确说明是线程不安全的
- 文档化同步策略
