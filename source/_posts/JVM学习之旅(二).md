---
title: JVM学习之旅(二)
date: 2020-03-28 18:41:33
tags: [Java]
---

# 写在前面的话

终于开始Java虚拟机的实战了，感觉活还是挺多的，希望之后能按时搞定相关内容

本章主要讲的是Java内存区域与内存溢出异常，介绍了Java运行时有哪些数据，以及内存溢出异常的原因，算是对Java内存管理的一点点初步了解

# Java内存区域与内存溢出异常

## 概述
内存动态分配与垃圾回收就是一堵高墙，墙里的人想出来，墙外的人想进来

## 运行时数据区域

![jvm结果](/imgs/learnjvm_jvm.png)

### 程序计数器

> 是一小块内存空间，可以看作当前线程所执行的字节码的行号指示器，JAVA虚拟机规范中唯一没有规定任何OOM情况的区域

- 确定的时刻，每个处理器只会执行一条线程，每条线程会有一个独立的程序计数器，这类内存被称为线程私有

- 线程如果执行JAVA方法，计数器记录正在执行的虚拟机字节码指令地址
- 如果执行Native方法，计数器值为空


### JAVA虚拟机栈

- 线程私有
- 生命周期与线程相同
- 描述JAVA方法执行的内存模型：用于存储局部变量表，操作数栈，动态链接，方法出口等信息
- 异常状况：
    - StackOverflowError:线程请求栈深度大于虚拟机所允许的深度
    - OutOfMemoryError:动态扩展无法申请到足够的内存


### 本地方法栈

- 与虚拟机栈相似，区别在于虚拟机执行JAVA方法服务，本地方法栈执行Native服务
- 异常状况：
    - StackOverflowError
    - OutOfMemoryError

### JAVA堆

- 被所有线程共享的一块内存区域，在虚拟机启动时创建
- 用于存放对象实例
- GC的主要区域
- 可以处于物理不连续区域，只要逻辑上连续即可，按照可扩展实现(-Xmx和-Xms控制)
- 异常情况：
     - OutOfMemoryError

### 方法区
- 所有线程共享的内存区域，存储已被虚拟机加载的类信息，常量，静态变量，即时编译器编译后的代码等数据。
- 方法区可以选择不实现垃圾回收
- 异常情况：
    - OutOfMemoryError

### 运行时常量池
- 是方法区的一部分
- 用于存放编译期生成的各种字面量和符号引用
- 具备动态性，运行时可以将常量放入池中
- 异常情况：
    - OutOfMemoryError   

### 直接内存
- JKD1.4开始可以基于Native函数库直接分配堆外内存，通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。
- 异常情况：
    - OutOfMemoryError

## HotSpot虚拟机对象探秘

### 对象的创建

- 进行类加载检查：
    - 是否能在常量池中定位到一个类的符号引用
    - 检查这个符号引用代表类是否被加载，解析和初始化
- 为新生对象分配内存
    - 规整内存：指针碰撞
    - 非规整内存：空闲列表
- 解决并发分配内存：
    - 分配内存空间动作进行同步处理，CAS配上失败重试保证原子性
    - 把内存分配的动作按照线程划分在不同的空间之中进行(本地线程分配缓冲 TLAB)

- 初始化内存：（可以提前到TLAB分配时）
- 对对象进行必要的设置
    - 对象属于哪个类的实例，如何找到类的元数据信息等

### 对象的内存布局

在HotSpot虚拟机中，对象可以分为3块区域：
- 对象头
    - 存储对象自身的运行时数据：如哈希，GC分代年龄，锁状态标志灯(Mark Word)
    - 类型指针
- 实例数据：对象真正存储的有效信息
    - 相同宽度的字段被分配在一起
    - 父类定义的变量会出现在子类之前
- 对齐填充：仅起占位符的作用，要求对象大小必须是8字节的整数倍


### 对象的访问定位

- 句柄访问：
    - reference指向句柄池
    - Java堆中分出一块内存作为句柄池；句柄中包含对象实例数据和类型数据；
    - 对象被移动只会改变句柄中的指针，不会修改reference
- 直接指针访问：
    - reference指向对象地址  
    - 堆对象中放置方位类型数据相关信息
    - 速度更快


## OOMEroor异常




### Java堆溢出
JVM参数配置如下
```
-Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails
```
代码如下：

```java
package Demo;

import java.util.ArrayList;
import java.util.List;

public class Hello {


    static class OOMObject{

    }

    public static void main(String[] args){
        List<OOMObject> list = new ArrayList<>();
        while (true){
            list.add(new OOMObject());
        }

    }

}
```

**运行结果**
```
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```

**排查思路**

- 内存分析工具进行分析
- 查看泄露的引用链
- 检查参数，分析是否可以增加内存，抑或是减小内存开销

### 虚拟机栈和本地方法溢出

JVM配置如下：
```
-Xss128k
```

代码如下：
```java
package Demo;

public class JVMStackSOF {
    private int stackLength = 1;

    public void stackLeak(){
        stackLength ++;
        stackLeak();
    }

    public static void main(String[] args) throws Throwable{
        JVMStackSOF oom = new JVMStackSOF();
        try{
           oom.stackLeak();
        } catch (Throwable e){
            System.out.println("stack length:" + oom.stackLength);
            throw e;
        }


    }

}
```

运行结果如下
```
stack length:11757
Exception in thread "main" java.lang.StackOverflowError
```

P.S. 多线程开发时出现内存溢出时，在不能减少线程数或者更换64位虚拟机时只能减少对大堆或者栈容量来换取更多的线程


### 本机直接内存溢出

配置
```
-Xmx20M -XX:MaxDirectMemorySize=1M
```


```java
package Demo;

import sun.misc.Unsafe;

import java.lang.reflect.Field;

public class DirectOOM {
    private static final int _1MB = 1024*1024;

    public static void main(String[] args) throws Exception{
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];
        unsafeField.setAccessible(true);
        Unsafe unsafe = (Unsafe) unsafeField.get(null);
        while (true){
            unsafe.allocateMemory(_1MB);
        }
    }

}

```

P.S. DirectMemory导致的内存溢出，很明显的特征是在Heap Dump文件中不会看见明显的异常






