---
title: go中的观察者模式
date: 2019-01-02 23:52:08
tags: [golang,设计模式]
---


# 前言

在做baas时，监控等页面都是使用的websocket进行数据通信的，因此需要后端在捕获到fabric端数据变动时主动向已经连接上的websocket推送消息，这个正好就是观察者模式所做的事情。

# 简述

观察者模式
> 一个目标物件管理所有相依于它的观察者物件，并且在它本身的状态改变时主动发出通知。这通常透过呼叫各观察者所提供的方法来实现。此种模式通常被用来实现事件处理系统。
> https://baike.baidu.com/item/%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F/5881786?fr=aladdin

简单的说，当被观察者(fabric中的数据)发生变化时，观察者(websocket)能独立的看到这些变化

# 基本思路

当初刚学了channel，所以第一个想法是用channel去传递信息，但是channel存在两个问题：

- channel中的消息在消费前会阻塞
- channel中的消息只能被消费一次，无法同时被多个消费者所消费


因此只能作罢，但是channel的特性用于传递消息真的是太好了，在网上瞎看时看到了有使用channel.close做信号量通知时，突然茅塞顿开，有一点思路了，再看了些网上的代码，有了如下的思路

- 使用类似链表来存储消息源的数据
- 新消息来临时使用channel.close来通知观察者来阅读新消息
- 观察者阅读完后，进入下一个消息数据


# 基础数据设计

根据上述思路，我们能得到基础的节点结构设计

``` golang
type state struct {
	value interface{}  // 数据
	next  *state
	done  chan struct{}   // 通知
}

func newState(value interface{}) *state {
	return &state{
		value: value,
		done:  make(chan struct{}),
	}
}

func (s *state) update(value interface{}) *state {
	s.next = newState(value)
	close(s.done)
	return s.next
}

```

接下来构建这个数据源链表，并且抽象出一个interface来供其它可能的形式使用

``` golang

type stream struct{
    state *state
}

type Stream interface{
    Value() interface{}
    Changes() chan struct{} // 数据源变化
    Next() interface{}
}

func (s *stream) Value() interface{}{
    return s.state.value
}

func (s *stream) Changes() chan struct{}{
    return s.state.done
}

func (s *stream) Next() interface{}{
    return s.state.next
}

```

OK，如上所示，就构建起来了基础的数据链，接下来就可以构建最核心的使用接口了，首先还是抽象一下

```golang
type Property interface {
	Value() interface{} //获取值

	Update(value interface{}) // 数据更新

	Observe() Stream //开始观察
}
```

接下来开始具体实现了，因为这里有多协程来操作，所以需要考虑锁的问题

``` golang

type property struct {
    sync.RWMutex
    state *state
}

func (p *property) Value() interface{}{
    p.RLock()
	defer p.RUnlock()
    return p.state.value
}

func (p *property) 	Update(value interface{}){
    p.Lock()
	defer p.Unlock()
	p.state = p.state.update(value)
}

func (p *property) Observe() Stream{
    p.RLock()
	defer p.RUnlock()
	return &stream{state: p.state}
}

```

这样，我们就实现了观察者模式的代码，所有的观察者都是从获取的一个stream的引用，golang有自己的垃圾回收机制，当某个stream上的节点完全没有人观察时，就会被回收掉。整体上内存开销也算OK的


# 使用

简单写了个小test测试了下，produce产生数据，consume消费展示数据，代码如下

```golang

import (
	"fmt"
	"testing"
	"time"
)

var pp Property

func init() {
	pp = NewProperty(0)
}

func produce(t *testing.T) {
	tick := time.NewTicker(20 * time.Second)
	i := 0
	for i < 100 {
		select {
		case <-tick.C:
			return
		default:
			time.Sleep(1 * time.Second)
			i += 1
			pp.Update(i)
		}
	}
}

func consume(s string, t *testing.T) {
	a := pp.Observe()
	if s == "C" {
		time.Sleep(3 * time.Second)
	}
	for {
		select {
		case <-a.Changes():
			a.Next()
			v := a.Value()
			fmt.Println("name", s, "value", v)
		}
	}
}

func TestConsume(t *testing.T) {

	go produce(t)

	go consume("A", t)

	time.Sleep(3 * time.Second)
	go consume("B", t)
	go consume("C", t)

	for i := 0; i < 1000; i++ {
		time.Sleep(1 * time.Second)
	}
}

```


最终输出如下

```
name A value 1
name A value 2
name A value 3
name B value 3
name A value 4
name B value 4
name A value 5
name B value 5
name C value 3
name C value 4
name C value 5
name A value 6
name C value 6
name B value 6
name A value 7
name B value 7
name C value 7
name A value 8
```

可以看到，B是在3秒处开始输出，C从3秒处开始拷贝了，但是由于进入后协程暂停，所以过了几秒才输出，但是输出的点仍然是3秒处的数据

