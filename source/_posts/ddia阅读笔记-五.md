---
title: ddia阅读笔记(五)
date: 2018-11-01 17:04:38
tags: [基础,概念,杂谈]
---

# 写在前面的话

本章主要是ddia第八章，分布式系统的麻烦

这一章概述性的讲解了分布式系统的几种类型的麻烦，主要集中在网络问题、时钟问题以及节点的正确性问题

网络问题这个确实是个老生常谈的问题了，其实这个无所谓分布式系统，在普通的b/s或者c/s架构的服务端都会面临这个问题，延迟，重试等等

时钟问题是个很蛋疼的问题，在BTC中使用了时钟服务器的方式进行同步，容忍大概11个区块时间(大概2小时)的时间偏差，偏差更多就无视掉，这个大概算是一种同步时钟的方式，ETH中使用了TD(total difficult)来确定谁更晚，这个更像是一种单调钟的理念，个人其实更喜欢单调钟这种方式，不过还是看具体业务的影响

节点的正确性问题就很尴尬了，在区块链中的共识算法其实就是来解决这个问题的，无论是POW还是BFT，只要能构建出一个不容易被攻击的多数(算力，无法被轻易复制的节点数量)，就能保证一个网络的健壮和正确性。

本章的整体内容来说偏概念性，没有太多直接具体的例子和立足，阅读起来也比较快速和轻松。


# 分布式系统的麻烦

## 故障与部分失效

计算机设计中的重要选择
> 如果发生内部错误，我们宁愿电脑完全崩溃，而不是返回错误的结果，因为错误的结果很难处理

部分失效
> 尽管系统的其他部分工作正常，但系统的某些部分可能会以某种不可预知的方式被破坏

部分失效的难点是不确定的：它有时可能会工作，有时会出现不可预知的失败，你甚至不知道是否成功了，因为消息通过网络传播的时间也是不确定的!

这种不确定，导致分布式系统难以工作。

## 云计算与超级计算机

- 超级计算机：具有数千个CPU的超级计算机通常用于计算密集型科学计算任务
- 云计算：通常与多租户数据中心，连接IP网络的商品计算机(通常是以太网)，弹性/按需资源分配以及计量计费等相关联
- 传统企业数据中心：位于这两个极端之间


处理故障方式的不同：
- 超级计算机：如果一个节点出现故障，通常的解决方案是简单地停止整个集群的工 作负载。故障节点修复后，计算从上一个检查点重新开始，更像是一个**单节点计算机**
    - 工作可以停止，重新启动影响很小 
    - 专用硬件，每个节点相当可靠
    - 专门的网络拓扑结构，具有更好的性能
    - 所有节点靠近一起，通信通过内网进行
- 云计算：实现互联网服务
    - 应用程序都是在线的，服务不可用是不可接受的
    - 商品机器构建而成，有较高的故障率
    - 基于IP和以太网，以闭合拓扑排列
    - 可以容忍发生故障的节点，并继续保持整体的工作状态
    - 如果是地理分散部署，通信很可能通过互联网进行，这种通信缓慢而不可靠

## 网络的不可靠

发出请求并期待响应，很多地方可能出错：

- 请求可能已经丢失
- 请求正在排队，等待交付
- 远程节点可能失效
- 远程节点可能暂时停止了响应，但是稍后会再次响应
- 远程节点可能处理了请求，但是网络上的响应已经丢失
- 远程节点可能处理了请求，但是响应被延迟，并且稍后将会被传递

通常方法：超时——在一段时间后放弃等待，并且认为响应不会到达

### 真实世界的网络故障

- 当你的网络遇到问题时，简单地向用户显示一条错误信息
- 需要知道您的软件如何应对网络问题，并确保系统能够从中恢复
- 有意识地触发网络问题并测试系统响应


### 检测故障

自动检测故障节点，例如：
- 负载平衡器需要停止向已死亡的节点转发请求(移出轮询列表)
- 在单主复制功能的分布式数据库中，如果主库失效，则需要将从库之一升级为新主库

特定的情况下，可能会被明确告知某些事情没有成功

### 超时与无穷的延迟

没有确定的答案处理超时应该等待多久

- 短暂的超时可以更快检测故障，但是极可能误报
- 长时间的超时意味着长时间等待，直到一个节点被宣告死亡

一个相对比较合适的处理：

- 网络传递的最长时间为`d`
- 非故障时间处理请求的最长时间为`r`
- 那么超时设置为`2d+r`


### 网络拥塞和排队

网络上数据包延迟通常是因为排队

- 多个节点同时将数据包发向同一目的地，网络交换机必须排队将他们逐一送入目标网络
- 当数据包到达目标机器时，如果机器繁忙，网络传入的请求将被操作系统排队直到应用程序准备处理
- 虚拟化环境中会有虚拟机被排队缓冲的情况
- TCP执行流量控制

设置超时时间：
- 通过实验方式选择超时
- 系统不使用配置的常量超时，而是连续测量响应时间及其变化，根据观察到的响应时间分布自动调整超时


### 同步网络和异步网络

完全同步的网络：即使数据经过多个路由器，也不会受到排队的影响，网络的最大端到端延迟是固定的。称之为有限延迟

#### 预测网络延迟

TCP连接与电话网络中的电路不太一样

- 电路是固定数量的预留带宽：电路交换网络
- TCP连接会机会性的使用任何可用的网络带宽：分组交换网络


分组交换网络针对突发流量进行了优化

- 不需要猜测带宽分配
- 不会使网络传输不必要的缓慢

目前有尝试去混合支持电路交换和分组交换的混合网络

然而在互联网或者公共云上，并没有这种服务，因此超时时间没有*正确*的值，必须通过实验来确定。

## 不可靠的时钟

时钟和时间非常重要，然而在分布式系统中时间是一件棘手的事情，通信是不即时的。这件事情很难确定涉及多台机器时发生事情的顺序。

NTP：网络时间协议，允许根据一组服务器报告的时间来调整计算机时钟，在一定程度上同步时钟。

### 单调钟与时钟

#### 时钟

> 根据某个日历返回当前日期和时间，如`linux`上的`clock_gettime(CLOCK_REALTIME)`

时钟通常用NTP同步，由于同步时的跳跃和重置，时钟不能用于测量经过时间。

#### 单调钟

> 单调钟适用于测量持续时间(时间间隔)，如`Linux`上的 `clock_gettime(CLOCK_MONOTONIC)`

可以通过单调钟的差值来确定经过的时间。

### 时钟同步和准确性

- 单调钟不需要同步
- 时钟需要根据NTP服务器或其他外部时间源来设置才能有用


然而外部时钟和硬件时钟并不精确，会导致如下的问题：

- 如果计算机的时钟与NTP服务器的时钟差别太大，可能会拒绝同步，或者本地时钟将被强制重置
- 如果某个节点被NTP服务器意外阻塞，可能会在一段时间内忽略错误配置
- 在拥有可变数据包延迟的拥塞网络上时， NTP同步的准确性会受到限制
- 一些NTP服务器错误或配置错误，报告时间已经过去了几个小时
- 闰秒导致59分钟或61秒长的分钟，这混淆了未设计闰秒的系统中的时序假设
- 在虚拟机中，硬件时钟被虚拟化，当一个CPU核心在虚拟机之间共享时，每个虚拟机都会暂停几十毫秒，而另一个虚拟机正在运行。从应用程序的角度来看，这种停顿表现为时钟突然向前跳跃
- 在未完全控制的设备上运行软件，则可能完全不信任该设备的硬件时钟

可能的解决方案

- GPS接收机
- 精确时间协议(PTP)
- 仔细的部署和监测


### 依赖同步时钟

如果你使用需要同步时钟的软件，必须仔细监控所有机器之间的时钟偏移。时钟偏离其他时钟太远的节点应当被宣告死亡，并从集群中移除。这样的监控可以确保你在损失发生之前注意到破损的时钟。


### 有序事件的时间戳

如果两个客户端写入分布式数据库，谁先到达? 哪一个更近?

![时间戳顺序](/imgs/ddia_timestamporder.png)

最后写入为准的解决冲突策略的基本问题：

- 数据库写入可能会神秘地消失:具有滞后时钟的节点无法用快速时钟覆盖之前由节点写入的值，直到节点之间的时钟偏差过去
- 无法区分高频顺序写入和真正并发写入，需要额外的因果关系跟踪(版本向量)
- 两个节点可以独立生成具有相同时间戳的写入，特别是在时钟仅具有毫秒分辨率的情况下，可能需要额外的决胜值。


逻辑时钟是基于递增计数器而不是振荡石英晶体，对于排序事件来说是更安全的选择

### 时钟读数的置信区间

时钟读数更像是一段时间范围。

- Spanner中的Google TrueTime API，它明确地报告了本地时钟的置信区间


### 全局快照的同步时钟

数据库分布在多台机器上时，全局单调递增的事务ID可能很难生成。

可以考虑使用同步时钟的时间戳作为事务ID。

举例：

- Spanner，确保时间戳反映因果关系，提交读写事务时会故意等待置信区间长度的时间，确保事务的置信区间不会重叠。


### 暂停进程

在分布式系统中，一个节点如何知道它仍然是领导者？

> 领导者从其它节点获得一个租约，任一时刻只有一个节点能拥有租约，节点必须周期性的在租约过期前续约。

请求处理循环大概如下所示：

``` golang

while(true){
    request=getIncomingRequest();
    // 确保租约还剩下至少10秒
    if (lease.expiryTimeMillis - System.currentTimeMillis() < 10000){
        lease = lease.renew();
    }
    
    if(lease.isValid()){
    // 暂停
        process(request);
}} }
 
```

可能的问题：
- 依赖于同步时钟，如果时钟不同步，那么代码将不可控
- 假设在`暂停`处进行进程暂停一段时间，那么处理时租约已经过期，然而进程自己并不知道，代码会并不知道租约到期。


**分布式系统中的节点，必须假定其执行可能在任意时刻暂停相当长的时间**

### 响应时间保证

硬实时系统：
> 软件必须有一个特定的截止时间，如果截止时间不满足，可能会导致整个系统的故障。如飞机主控计算机等。

然而，对于大多数服务器端数据处理系统来说，实时保证是不经济或不合适的。因此，这些系统必须承受在非实时环境中运行的暂停和时钟不稳定性。


### 限制垃圾回收的影响

将GC暂停视为一个节点的短暂计划中断，并让其他节点处理来自客户端的请求，同时一个节点正在收集其垃圾

- 运行时可以警告应用程序一个节点很快需要GC暂停
- 应用程序可以停止向该节点发送新的请求
- 节点在没有请求正在进行时执行GC

只用垃圾收集器来处理短命对象(这些对象要快速收集)，并定期在积累大量长寿对象(因此需要完整GC)之前重新启动进程


## 知识、真相与谎言

### 真理由多数所定义

**节点不一定能相信自己对于情况的判断**

决策需要来自多个节点的最小投票数，以减少对某个特定节点的依赖

### 领导者与锁定

通常情况下，在一个系统中，一些东西只能有一个，如：

- 分区领导者只能有一个节点，以避免脑裂
- 特定资源的锁或对象只允许一个事务持有，以防止同时写入和损坏
- 一个特定的用户名只能被一个用户所注册

在分布式系统中这里需要注意：

**即使一个节点认为自己是“天选者”，但并不意味着系统中其它节点同意**

### 防护令牌

![防护令牌](/imgs/ddia_fencing.png)

如上图所示，客户端1以33的令牌获取的租约，在挂起超时后无法再以33的令牌写入，保证了存储顺序不被破坏。

在zookeeper中可以使用事务标识`zxid`或节点版本`cversion`用作屏蔽令牌

### 拜占庭故障

拜占庭故障
> 节点出现撒谎，如声称有然而实际没有收到任何消息，这种行为被称为拜占庭故障

拜占庭容错
> 一个系统在部分节点发生故障、不遵守协议、甚至恶意攻击、扰乱网络时仍然能继续正确工作，称之为拜占庭容错

大多数拜占庭容错算法需要保证总共有`3f+1`个节点时，至少有`2f+1`个节点是正确的。

### 弱谎言形式

- 应用程序级协议中的校验和：防止由于硬件、操作系统、驱动程序或路由器导致的网络数据包的错误
- 可公开访问的应用程序必须仔细检查来自用户的任何输入：如检查值是否在合理范围内，限制字符串大小以防止大内存拒绝服务
- NTP客户端配置多个服务器，使用多个服务器使NTP更加健壮


### 系统模式与现实

对于定时假设，如下三种模型是常用的：

#### 同步模型

假设网络延迟，进程暂停，时钟误差都是有界限的

#### 部分同步模型

一个系统在大多数情况下像一个同步系统一样运行，但有时候会超出网络延迟，进程暂停和时钟漂移的界限

#### 异步模型

一个算法不允许对时机做任何假设


另外，常见的节点系统模型如下：

#### 崩溃-停止故障

算法会假设一个节点只能以崩溃的方式失效，这意味节点可能在任意时间突然停止响应，此后该节点突然停止响应

#### 崩溃-恢复故障

假设节点可能会在任何时候崩溃，但也许会在未知的时间之后再次开始响应，节点的稳定存储会在崩溃中保留，内存中的状态会丢失

#### 拜占庭故障

节点可以做任何事情，包括视图戏弄和欺骗其他节点


### 算法的正确性

- 唯一性：没有两个屏蔽令牌请求返回相同的值
- 单调序列：如果请求`x`返回了令牌`t_x`,`y`返回了令牌`t_y`，并且`x`在`y`开始之前已经完成，那么`t_x` < `t_y`
- 可用性：请求防护令牌并且不会崩溃的节点，最终会收到响应


### 安全性和活性

通俗定义：
- 安全性：没有坏事发生
- 活性：最终好事发生

严肃定义：
- 安全性：如果被违反，可以指向一个特定的时间点。违规行为不能撤销，损失已经发生
- 活性：在某个时间点，它可能不成立，但总是希望在未来


### 系统模型映射到现实

现实往往比理论系统更加复杂，比如：

- 故障恢复模型中假定稳定存储数据不会崩溃，然而实际中磁盘上的数据可能被破坏
- 法定人数算法中依赖节点存储某些需要存储的数据，然而如果某个节点上存储的数据全部丢失，可能会打破法定条件，破坏算法正确性。


**证明算法正确并不意味着它在真实系统上的实现必然总是正确的**









