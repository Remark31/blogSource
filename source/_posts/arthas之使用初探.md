---
title: arthas之使用初探
date: 2021-02-17 22:20:31
tags: [jvm,工具]
---

# 写在前面的话

Arthas是阿里开源的一款非常好用的java诊断工具

本质上原理是通过jvm提供的attach机制提供了进程间通信的能力，将arthas的进程能与目标进程进行通信,再通过java的agent机制将arthas的代理注入到目标进程中实现一种类似于AOP的效果，从而查看到目标进程中想要的信息，本文就简单介绍一下这个巫妖王

# 什么是Arthas

> https://github.com/alibaba/arthas

# 使用Arthas

> https://arthas.gitee.io/quick-start.html 
> 手把手指南

- java -jar arthas-boot.jar 启动arthas
- 选择要attach的java应用，等待attach成功
- dashboard: 查看当前进程信息
- thread ID: 打印线程ID的栈
- jad package.class: 反编译
- watch package.class function: 查看function当时的返回值


## 延伸点：火焰图

![火焰图](/imgs/jvm_arthas_fire.png)

x轴为调用顺序，y轴为栈深，线条颜色无实际意义，线条长度代表cpu执行该方法所花费的时间占比；调用顺序为自下而上。最上方的则为最底层的方法。
所以通常寻找的是最上方的，最宽的方法

# Arthas原理

## 基础背景： Attach

> http://lovestblog.cn/blog/2014/06/18/jvm-attach/

是jvm提供的一种进程间通信的能力，能让一个进程传命令给另一个进程，并让它执行内部的一些操作

使用了 linux下文件的socket通信


## 基础背景：Instrumentation && Agent

> http://lovestblog.cn/blog/2015/09/14/javaagent/

通过agent
- 可以在加载class文件之前做拦截把字节码做修改
- 可以在运行期将已经加载的类的字节码做变更
- 获取所有已经被加载过的类
- 获取所有已经被初始化过了的类（执行过了clinit方法，是上面的一个子集）
- 获取某个对象的大小
- 将某个jar加入到bootstrapclasspath里作为高优先级被bootstrapClassloader加载
- 将某个jar加入到classpath里供AppClassloard去加载
设置某些native方法的前缀，主要在查找native方法的时候做规则匹配


## Arthas 执行流程

![arthas流程](/imgs/jvm_arthas.png)



