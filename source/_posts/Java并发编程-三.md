---
title: Java并发编程(三)
date: 2019-03-31 17:35:57
tags: [Java,并发]
---

# 写在前面的话

本章主要讨论的是共享变量，涉及到的知识点在于volatile、finnal、构造函数的逸出和安全发布。

volatile这个是java特有的玩意，以前没接触过，今天看了之后理解的七七八八，估计还要找点资料看看。

finnal这个之前就看过了，关键点在于对象实际是可变的。

构造函数的逸出知道个概念了，但是没有吃过坑，所以还不能很深入的理解，目前暂时理解成不能在构造函数中启动线程。

安全发布这个是在之前被忽略的玩意。没太考虑过这么多。这样看来Spring的Bean其实做到的就是个安全发布。


# 共享对象

不仅希望避免一个线程修改其它线程正在使用的对象的状态，同样也希望一个线程更改状态后，其它的线程能看到这个状态的改变

# 可见性

> 重排序现象：在单个线程中，只要重排序不会对结果造成影响，那么就不能保证其中的操作一定按照写定的顺序执行


只要数据需要被线程共享，就需要恰当的同步。

## 过期数据

除非每一次访问变量都是同步的，不然有可能会看到过期的值。

## 非原子的64位操作

> 最低限的安全性：一个线程在没有同步的情况下读取变量，它可能得到一个过期值，但是它至少可以看到一个线程在那里设定的一个真实数值，而不是一个凭空而来的值

例外：没有声明成为`volatile`的64位数据变量。
> JVM对于64位的读写允许分为两个32位的操作，因此可能出现一个值得高32位和另一个值得低32位。

## 锁和可见性

> 锁不仅仅是关于同步和互斥，也是关于内存可见的，为了保证所有线程都能看到共享的、可变变量的最新值，读取和写入线程都必须使用公共的锁进行同步

## Volatie变量

> 对voliatie变量不会被重排序，读一个volatie变量，总会返回某一线程所写入的最新值

访问volatie变量不会加锁

> 通常被当做标识完成、中断、状态的标记使用

> 加锁可以保证可见性和原子性，volatie只能保证可见性

使用volatie变量的标准
- 写入变量时并不依赖于变量的当前值；或者能够确保只有单一的线程能够修改变量的值
- 变量不需要与其它的状态变量共同参与不变约束
- 访问变量时，没有其它原因需要加锁


# 发布和逸出

> 发布对象：使它能够被当前范围之外的代码所使用
> 逸出：一个对象在尚未准备好时就将它发布

一种常见的发布方式：将对象和变量存储到公共静态域中。


``` java
public class ThisEscap{
    public ThisEscap(){
        source.registerListener(new EventListener(){
            public void onEvent(Event e) {
                doSomething(e);
            }
        });
    }
}
```

**疑问：为什么内部类的实例会包含对封装实例的逸出？**

此处暂时理解为构造函数中不允许启动线程，启动线程可能导致this可以被线程所调用。



## 安全构建的实践

> 对象只有通过构造函数返回后，才处于稳定的，可预言的状态。

常见错误：在构造函数中启动一个线程。(可以创建线程，但是不要立即启动它)

如果想要在构造函数中注册启动线程，可以将它私有化，如下所示：

``` java
public class SafeListener{
    private final EventListener listener;
    
    private SafeListener(){
        listener = new EventListenr(){
            public void onEvent(Event e){
                doSomething(e);
            }
        }
    }
    
    public static SafeListener newInstance(Event source){
        SafeListener safe = new SafeListener();
        source.registerListener(safe.listener);
        return safe;
    }
    
}
```

# 线程封闭

一种避免共享数据的方式就是不共享数据。

## Ad-hoc 线程限制

> Ad-hoc：非正式的，这里指未经过设计而得到的线程封闭行为。

最好使用一种线程限制的强形式来取代它(栈限制或者Thread Local) 

## 栈限制

栈限制是线程限制中的一种特例，在栈限制中，只有通过本地变量才能触碰到对象。本地变量被限制在执行栈中，其它线程无法访问这个执行栈。



## ThreadLocal

允许每个线程和持有数值的对象关联在一起。ThreadLocal提供了get与set访问器，为每个使用它的线程提供一份单独的拷贝。get总是返回当前set设置的最新值。

使用情况：
- JDBC
- 一个频繁的操作需要buffer这样的临时对象，又需要避免每次频繁分配。

概念上可以将ThreadLocal<T>看做 map<Thread,T>。与线程相关的值存储在线程对象自身中，线程中止后相关值会被GC掉。

# 不可变性

创建后不能修改的对象是天生线程安全的。

只有满足如下的状态，对象才是不可变的
- 它的状态不能在创建后再被修改
- 所有域都是final类型
- 它被正确创建

## final域

final域是不能修改的。(final域指向的对象是可变的)

## 使用volatie发布不可变对象

使用一个不可变对象来持有变量，这样就不会担心对象呗其它线程所改变。

``` java
class OneValueCache{
    private final BigInteger lastNumber;
    private final BigInteger[] lastFactors;
    
    class OneValueCache(BigInteger i, BigInteger[] factors){
        lastNumber = i;
        lastFactors = Array.copyOf(factors, factors.length);
    }
    
    public BigInteger[] getFactors(BigInteger i){
        if (lastNumber == null || lastNumer.equals(i) ) {
            return null;
        } else {
            return Arrays.copyOf(lastFactors, lastFactors.length);
        }
    }
}

```

再使用volatie进行发布，保证数据发布后被其它线程可见。

``` java
public class VolatileCache implements Servlet{
    private volatile OneValueCache cache = new OneValueCache(null, null);
    
    public void service(ServletRequest req, ServletResponse resp){
        BigInteger i = extractFromRequest(req);
        BigInteger factors = cache.getFactors(i);
        if (factors == null) {
            factors = factor(i);
            cache = new OneValueCache(i, factors);
        }
        
        encodeIntoResponse(resp, factors);
        
    }
}

```

# 安全发布

简单的讲对象放在公共区域中不代表已经安全发布。

## 不正确发布：当好对象变坏时

如果没有充足的同步，跨线程共享数据会出现很多奇怪的现象。

## 不可变对象与初始化安全性

> 不可变对象可以在没有额外同步的情况下，安全的用于任何线程，甚至发布它们时不用同步。

## 安全发布的模式

一个正确创建的对象可以被如下方式安全发布：
- 通过静态初始化器初始化对象的引用
- 将它的引用存储到volatile或者AtomicReference
- 将它的引用存储到正确创建的对象的final域中
- 将它的引用存储到由锁正确保护的区域中。

例如：
- 置入`Hashtable`、`synchronizedMap`、`ConcurrentMap`中的主键或者键值，会安全发布到可以从Map中获得它们的任意线程中。
- 置入`Vector`、`CopyOnWriteArrayList`、`CopyOnWriteArraySet`、`synchronized-List`或者`synchronizedSet`
- 置入BlockingQueue或者ConcurrentLinkedQueue的元素。

## 有效不可变对象

如果一个对象在技术上不是不可变的，但是它的状态不会在发布后被修改，这些对象称为有效不可变对象。

> 任何线程都可以在没有额外同步的情况下安全的使用一个安全发布的高效不可变对象。

## 可变对象

发布对象的必要条件依赖于对象的可变性
- 不可变对象可以通过任意机制发布
- 有效不可变对象必须要安全发布
- 可变对象必须要安全发布，同时必须要线程安全或者是被锁保护


## 安全共享变量

一些有效策略如下：
- 线程限制：一个线程限制的对象，被线程独占，只能被一个线程修改
- 共享只读：一个共享的只读对象，无人能修改
- 共享线程安全：一个线程安全的对象在内部进行同步，其它线程无需同步，可以通过公共接口访问。
- 被守护的：被特定的锁所保护，只能通过特定的锁来访问。

# 相关阅读

- Volatile变量 https://www.ibm.com/developerworks/cn/java/j-jtp06197.html





