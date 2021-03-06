---
title: POS初探
date: 2018-05-26 23:47:43
tags: [区块链,以太坊,共识]
---

> 本文是阅读 https://github.com/ethereum/wiki/wiki/Proof-of-Stake-FAQ 以及 https://ethfans.org/posts/Proof-of-Stake-FAQ-new-2018-3-15 后自己总结的一些小点，更多的细节需要继续参照原文


## 什么是权益证明
> 权益证明（PoS）是一类应用于公共区块链的共识算法，取决于验证者在网络中的经济权益。

在基于权益证明的公共区块链（如以太坊即将实现的Casper协议）中，一组验证者轮流提议并票决下一个区块，而每位验证者的投票权重取决于其保证金额的大小（即权益）。权益证明的重要优势包括保障安全性、降低中心化危险以及提升能源效率。

一个通常的权益证明如下：
- 区块链会追踪一个**验证者集**
- 持有币的人可以通过一种发送一种**将币锁定为保证金的特殊交易**成为**验证者**
- 创造并认可新区块的过程可通过当前所有验证者均可参与的共识算法来完成


权益证明分类
- 基于区块链的权益证明
- 拜占庭容错（BFT）型权益证明


### 基于区块链的权益证明

- 伪随机选择一个验证者
- 验证者出块
- 新创造的区块必须跟在之前的某个区块（通常是位于最长链的末端的区块）后面


### 拜占庭容错（BFT）型权益证明

- 随机选择出块验证者
- 多轮过程投票
- 决定区块是否添加到链上


## 权益证明相对于工作量证明的优点

- 不需要为了保护区块链而消耗大量电力
- 不需要为了保持参与者积极性发型很多新代币，可以烧掉
- 有助于抑制中心化卡特尔式机构的形成
- 降低中心化风险，不需要投资大量矿机
- 能够采用经济处罚，这让发动各种形式的51%攻击所要付出的代价比在工作量证明中高出许多


```
卡特尔(Cartel)是指生产同类商品的企业，为了获取高额利润，在划分市场、规定商品产量、确定商品价格等一个或几个方面达成协议而形成的垄断性联合。

辛迪加(Syndicat)是同一生产部门的企业为了获取高额垄断利润，通过签订协议，共同采购原料和销售商品，而形成的垄断性联合。

托拉斯(Trust)是垄断组织的一种高级形式，通常指生产同类商品或在生产上有密切联系的企业，为了获取高额利润，从生产到销售全面合并，而形成的垄断联合。

康采恩(Konzem)是分属于不同部门的企业，以实力最为雄厚的企业为核心而结成的垄断联合，是一种高级而复杂的垄断组织。
```

## 权益证明将如何适应传统拜占庭容错研究

- CAP定理
```
如果网络出现分区，你必须在一致性和可用性中选择一个，不能二者兼得
```

- FLP不可能定理
```
在异步环境中（即使在正常运行的节点之间，也无法控制网络延时上限），不可能创造出一种算法，在出现单个故障或不诚实的节点之时，确保能在任何特定的有限时间内达成共识。
```


- 容错范围

    - 在部分同步的网络模型（即网络延时虽有上限，却不知上限值）中运行的协议可以容忍超过1/3的任意错误
    - 在异步模式（即对网络延时没有范围限制）下的决定性协议不能容忍错误（虽然文章中没有提及随机化算法的容错率高达1/3）
    - 在同步模式（即网络延时保证会低于某个已知数）下的协议竟然能够实现100%的容错，不过在出错节点大于等于1/2时有一些限制条件。
    
### POW的容错

- 与网络延迟有关系，不存在网络延迟的话容错率在50%
- 实际上以太坊容错率为46%，比特币是49.5%
- 网络延迟等于区块生成时间，容错率下降至33%

### POS的容错

- 同步网络模型：“基于区块链的”权益证明算法
- 部分异步网络模型：将部分同步网络中的传统拜占庭容错共识与权益证明联系起来


## “无利害关系”问题是什么？如何解决？

### 问题
> 在许多早期（基于区块链的）权益证明算法（包括 Peercoin）中，只为创造区块提供奖励，且没有惩罚措施。这就造成了不幸的结果，在出现多条区块链相互竞争的情况下，会激励验证者在每条链上都创造区块，以确保获得奖励

简单的说即是在POS算法中，参与者可以对于自己收到的链都进行下注，获得更多的期望收益，而在POW中，如果分开下注会导致算力下降，期望收益降低。

- POS的攻击者只需要比无私节点(只在正确链上下注)更多的算力，即可
- POW的攻击者一定要比无私节点和理性节点都要多的算力，即可

### 解决策略

#### 方案一：惩罚证明
> 惩罚证明：两个冲突的已签名区块头

简单的说即是发现对多条链同时出块的验证着，会将惩罚证明纳入区块链，再扣除有恶意行为(同时向多条链出块)的验证者的保证金

##### 前提条件
- 提前很久确定验证集
    - 验证集不同会导致向两边同时出块 
- 节点保持联网状态
- 存在中程验证者窜谋的风险
    - 连续30名验证者中有25人联合起来，提前同意在之前的19个区块上发动51%攻击


#### 方案二：惩罚在错误链上出块的证明者

简单的说即是对在错误链上出块的验证者进行罚金惩罚

```
这是一种和POW的惩罚很类似的做法，POW在错误链上出块的节点会浪费电费，POS就会直接损失权益，这会导致验证者的风险加大，但是好处在于不需要提前知道验证着是谁
```

## 拜占庭容错型权益证明算法目前是如何运作的？

允许验证者通过发送一种或者多种类型的签名信息对区块进行“投票”

### 规则

- 确定条件
    - 什么样的情况下一个特定的哈希可以被认为是确定值
- 惩罚条件
    - 什么样的情况下一个特定的验证着可以被认定位恶意行为者


### 举例

- 如果 `MESSAGES` 包含 `["COMMIT", HASH1, view]` 和 `["COMMIT", HASH2, view]` 形式的信息，其中由同一个验证者签署的 `view` 是相同的，但 `HASH1` 和 `HASH2` 是不同的，那么该验证者就会受到惩罚。
```
同时向两个块签名投票
```

- 如果`MESSAGES`包含 `["COMMIT", HASH, view1]` 形式的信息，那么除非`view1` = -1或是同时包含某个特定 `view2` 的 `["PREPARE", HASH, view1, view2]` 形式的信息，且 `view2` < `view1` ，由 2/3 的验证者签署，那么下达COMMIT命令的验证者就会受到惩罚。

```
只要有2/3以上的验证者就可以产生确定值，否则会受到惩罚
```

### 总结

- 可问责安全性
    - 如果互相冲突的`HASH1` 和 `HASH2` （也就是说 `HASH1` 和 `HASH2` 是不同的，且互不为衍生值）是确定值，那么至少有1/3的验证者违反了某个惩罚条件。
- 似乎合理的活性
    - 除非有1/3以上的验证者违反了惩罚条件，只要有2/3以上的验证者就可以产生确定值。


## “经济确定性”的一般概念是什么？

> 经济确定性的意思是一旦某个区块确定了下来，或者更普遍地说，一旦已经签署了足够的特定消息，在未来的任意时刻，想要让合法的历史记录包含冲突区块，只有在很多人愿意为此消耗大笔金钱的情况下才能实现。如果一个节点认为某个区块满足了该条件，就会有强大的经济保障来支持这个区块将成为所有人都认同的合法区块历史的一部分。

### 达到经济确定性的方式

- 如果足够数量的验证者已经签署了“我同意在所有不包含区块B的链中损失X”这种形式的加密经济声明，就可以从经济上确定一个区块。
- 如果足够数量的验证者已经签署了支持区块B的信息，就可以从经济上确定一个区块。

### 总结

实现确定性方法源自于非利害关系问题的两个解决方案：
- 惩罚`验证错误区块`
- 惩罚`同时验证冲突区块`


## 经济确定性与拜占庭容错理论的关联

### 传统的拜占庭容错理论

- 假设大体会具有相似的安全性（safety）和活跃度（liveness）特点
- 只需要在2/3的验证者是诚实的情况下就可以实现安全性
- 试图证明的是“如果机制M出现安全故障，会有至少1/3的节点出了错”

### 经济确定性模型理论

- 证明的是“如果机制M出现安全故障，会有至少1/3的节点出了错，即使你在发生故障时离线了，也能知道是那些节点发生了故障”。
- 从活跃度角度来看，该模型更加容易，因为不需要证明网络将达成共识，只需要证明网络不会阻塞。
- 问责型（即如果你犯了可被抓住的那类错误，我们就可以为之制定惩罚条件）
- 会与延迟混淆型（要注意即使是信息发送过早这样的错误也会与延迟混淆，通过调快所有人的时间并让没有过早发送的信息出现更高的延迟可以模拟出这一情况）


### 弱主观性

```
此处需要再次调研
```

#### 问题

- 假设保证金被锁定时间是4个月
- 6个月后发动了51%攻击，重新导入之前的区块
- 恶意验证着可以从主链上取回保证金，不受惩罚


#### 方案

```
区块回滚限制（revert limit）
```

- 节点必须拒绝回滚比保证金被锁定时间更久的区块（按照上例来说就是4个月）
- 节点可能会基于它们看到某些消息的时间而得出不同的结论(“主观的”)
- 进行链外验证

#### 权益证明链不需要与工作量证明链“锚定”

- 锚定机制本身不安全
- 弱主观性只是一个安全性假设的一个很小的附加部分，不需要外部来源信任来支持


### 权益证明中的经济上惩罚审查制

#### 问题

区块链不能直接分辨出
- 用户 A 尝试发送交易 X 但它被不公平地删掉了
- 用户 A 尝试发送交易 X，但因为交易费不够而始终没有被打包到区块中
- 用户 A 从未发送交易 X

三者的不同

#### 可用技术

- 利用停机问题来抵御审查
- 交易通过时间锁加密
    - 验证者将在不知道交易内容的情况下打包交易，并且交易只有在（被打包）之后内容才会自动显示
    - 
- 将审查识别包含在分叉选择规则中
    - 节点监视网络中的交易，如果他们观察到一笔交易在足够长的时间内包含足够高的费用，他们就会为不包含该交易的区块链打一个 “低分”。 
    - 包含了该区块的少数链会自动合并，所有诚实在线节点都将在其后挖矿


## 验证者选择机制的工作原理与“权益研磨(Grinding)”

### 权益研磨(Grinding)

> 验证者通过执行一些计算或者采取某些其他措施使得随机性更偏向他们

- 在`Peercoin(点点币)`中，验证者可以“研磨”许多参数组合 ，并找到最有利的参数来增加它们的币产生合法区块的可能性。
- 在一个现已不存在的实现中，第N+1个区块的随机性取决于第N个区块的签名。这就允许验证者反复产生新签名，直到找到允许他们产生下一个区块的签名，从而永远掌握系统的控制权。
- 在 NXT 中，第 N+1个区块的随机性取决于产生第N个区块的验证者。这就允许验证者通过简单的跳过一次产生区块的机会来操纵随机性。这里的机会成本等于一个区块奖励，但是有时候新的随机数种子将在接下来的几十个区块中使得验证者获取多于平均值区块产生权的区块数。


### 解决方案

- 要求验证者提前将他们的代币抵押
- 避免使用会被轻易操纵的数据信息作为随机性的源数据
- 使用基于秘密共享或确定性阈值签名的方案，并且让验证者合作产生随机数值
    - 33%-50% 的验证者勾结才可以干涉操作，导致协议必须假设有 67% 的活跃诚实节点）。    
- 使用加密经济方案
    - 验证者提前提交信息(先发布sha3(x))
    - 必须在区块中发布x
    - 将x添加到随机数池中

- 使用 Iddo Bentov的 “多数信号”
    - 使用其他信号产生的前 N 个随机数的 bit-majority （众数位）来生成一个随机数

```
如果大多数源数据的第一位是 1，那么结果的第一位是1，否则是0，如果大多数源数据的第二位是1，那么结果的第二位是1，否则是0，依此类推
```

对于加密经济方案的攻击方式
- 在提交阶段操纵 x 
    - 不切实际
- 选择性避免发布区块
    - 该攻击将消耗一个区块奖励的机会成本
    - 明确的轮空罚款来进一步惩罚


## 在 Casper 中的51% 攻击

### 确定性回滚
> 验证者已经确认区块 A 之后，又确认了另一竞争区块 A'；从而打破区块链的确定性保证

- 通过社区协调，使全社区专注于某一条分支挖矿

### 拒绝区块活性
> 一个算力大于等于34%的验证者联盟可以轻易拒绝添加更多的区块，而不必试图回滚区块

- 该协议可以包含一个自动轮换验证者集合的功能
- 规定在这种情况下，所有未参与共识过程的旧验证者，（系统）会对他们的存款收取大量的罚金。
- 使用硬分叉的方式添加新的验证者，并删除攻击者的余额。

### 屏蔽攻击
> 算力大于等于34%的验证者拒绝添加某个包含他们不喜欢的某些交易类型的区块，但区块链将继续运行，区块也会持续被添加到区块链中。这可能是一些温和的屏蔽攻击，只是用屏蔽的方法来干涉某些具体的应用


- 协议内添加功能使交易能够自动安排未来的事件
- 主动分叉选择规则
    - 通过尝试与给定的区块链交互或者验证其是否在审查你，来决定给定区块链是否是有效的
    - 节点重复发送预定存储他们的以太币的交易，并在最后时刻取消该笔交易

