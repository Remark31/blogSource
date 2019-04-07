---
title: Java并发编程(四)
date: 2019-04-07 16:30:43
tags: [Java,并发]
---

# 写在前面的话

本文是java并发编程的第四章，这一章的感觉就是教你如何去实现一个线程安全的类。

首先介绍了一些基本的设计思路，接下来提供了一些常用的套路，例如：

- 实例限制
- 委托安全
- 对已有的线程安全类的添加方法

之前在go中使用的方法大部分是客户端加锁或者是实例限制，监视器模式也是很常见的。

主要是golang之前是没有什么线程安全的map的(1.9后才添加，而且使用起来很蛋疼)

整体方法其实基本之前都用过，这里只是将方法更加理论化了一些，然后了解了一些Collection带的方法

# 组合对象
# 设计线程安全的类

设计线程安全类的基本要素
- 确定对象状态由哪些变量构成
- 确定限制状态变量的不变约束
- 制定一个管理并发访问对象状态的策略


## 收集同步需求

- 明确类的状态空间(可能处于的状态的范围)
- 确定状态转换的合法性
- 多变量的不变约束需要原子性

## 状态依赖的操作

> 一个状态存在基于状态的先验条件，则称为状态依赖的

## 状态所有权

- 大部分情况下，所有权和封装性总是一起出现的。对象封装并且拥有它的状态
- 容器类通常所有权分离

# 实例限制

> 将数据封装在对象内部，把对数据的访问限制在对象的方法上，更容易确保线程在访问数据时总能获得正确的锁

``` java
@ThreadSafe
public class PersionSet{
    @GuardedBy("this")
    private final Set<Person> mySet = new HashSet<Person>();
    
    public synchronized void addPerson(Person p){
        mySet.add(p);
    }
    
    public synchronized boolean containsPerson(Person p) {
        return mySet.contains(p);
    }
    
}
```

实例限制是构建线程安全类的最简单方式之一。

使用装饰器模式可以让非线程安全的结构的每个接触方法处理为同步方法，包装器对象就是线程安全的。

发布受限状态会影响其限制性

## Java 监视器模式

``` java
public class PrivateLock{
    private final Object mylock = new Object();
    @GuardedBy("mylock") Widget widgt;
    
    void SomeMethod(){
        synchronized(mylock) {
            
        }
    }
}
```

使用私有锁对象可以封装锁，其它人并不能得到他。

# 委托线程安全

当所有类组件都是线程安全时，还需要一个额外的线程安全层么？“看情况”

当类的状态就是线程安全组件时，可以说线程安全委托给了安全组件

TIPS.

> Collections.unmodifiableMap：返回一个不可修改的Map，但是如果value是对象，那么这个对象的内容还是可以修改的。

## 非状态依赖

也可以委托到多个隐含的状态变量上，只要这些状态变量是相互独立的，安全的。

## 委托无法胜任


> 如果一个类由多个彼此独立的线程安全状态变量组成，并且类的操作不包含任何无效状态变换时，可以将线程安全委托给这些状态变量

然而如果有多个，还是用一个公开或者私有的锁吧

## 发布底层的状态变量

> 如果一个状态变量是线程安全的，没有任何不变约束去限制它的值，并且没有任何状态转换限制它的操作，那么它可以被安全发布。

# 向已有的线程安全类添加功能

- 最安全方法：直接在原始类中进行修改，然而可能性比较低，可能原始类中是不能修改的
- 继承这个类：如果这个类设计是可扩展的。


## 客户端加锁

TIPS.
Collections.synchronizedList 将线程不安全的list转换为线程安全的list：源码就是使用了一个mutex的object作为锁

一个常见的错误

``` java
public class ListHelper<E>{
    public List<E> list = Collections.synchronizedList(new ArrayList<>());
    
    public synchronized boolean putifAbsent(E x) {
        boolean absent = !list.contains(x);
        if (absent){
            list.add(x);
        }
        return absent;
    }
}
```

此处synchronized锁的对象是ListHelper，并不是需要的list。因此这里正确的写法应该是：

``` java
public class ListHelper<E>{
    public List<E> list = Collections.synchronizedList(new ArrayList<>());
    
    public boolean putifAbsent(E x) {
    synchronized(list){
        boolean absent = !list.contains(x);
        if (absent){
            list.add(x);
        }
        return absent;
        }
        
    }
}
```

## 组合

``` java 
public class ImprovedList<T> implements List<T> {
    private finnal List<T> list;
    
    public ImprovedList(List<T> list) {
        this.list = list;
    }
    
    public synchronized void clear(){
        this.list.clear();
    }
    
    ...
}
```

完全构造一个新的锁层，不关心原有的底层结构是如何实现的。

# 同步策略的文档化

> 为类的用户编写线程安全性担保的文档，为类的维护者编写类的同步策略文档，最佳的时间是设计的时间。

> 如果你从未考虑过这些，我们倒是钦佩你的乐观


