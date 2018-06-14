---
title: golang中的context
date: 2018-06-14 18:48:49
tags: [golang]
---

# 动机

在学习ipfs代码的过程中大量看到了context包的使用，以前只是在使用beego框架时通过context在request的body中获取一些数据和信息，没有对整个context的使用有个完整的认识和了解，借此机会重新来看看context包具体是如何使用的

# context的接口

```golang
type Context interface {
    
    Deadline() (deadline time.Time, ok bool)

   
    Done() <-chan struct{}

   
    Err() error

    
    Value(key interface{}) interface{}
}
```

从godoc中看context的接口，可以看到context是由如下4个方法构成的

- Deadline() (deadline time.Time, ok bool)
    - 返回context结束/超时的时间
- Done() <-chan struct{}
    - 当context结束/超时时，返回一个closed的channel信号
- Err() error
    - 在channel被结束/超时时，返回原因 
- Value(key interface{}) interface{}
    - 返回与key对应的value值 


在context包中，提供了两个具体实现，分别是

- func Background() Context
    - 一个非nil的空context，不能被取消，没有value，没有deadline，主要是被main，初始化以及测试用例所使用，作为最底层的context等待请求的到来 
- func TODO() Context
    - 推荐作为一个静态分析工具去检测context在程序上正确的被执行


# 基础的使用原则

- 不要把 Context 存在一个结构体当中，显式地传入函数。Context 变量需要作为第一个参数使用，一般命名为ctx
- 即使方法允许，也**不要传入一个 nil 的 Context**，如果你不确定你要用什么 Context 的时候传一个context.TODO
- 使用 context 的 Value 相关方法只应该用于在程序和接口中传递的和请求相关的元数据，**不要用它来传递一些可选的参数**
- 同样的 Context 可以用来传递到不同的 goroutine 中，Context 在多个goroutine 中是**安全的**

# 常见方法

- func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
    - 返回了一个父context的拷贝和一个新的Done channel，当返回的cancel方法被调用时，这个Done channel会被close掉

这里用个代码例子


```golang
package main

import (
	"context"
	"log"
	"os"
	"time"
)

var logg *log.Logger

func someHandler() {
	ctx, cancel := context.WithCancel(context.Background())
	go doStuff(ctx)

	time.Sleep(5 * time.Second)
	cancel()

}

func doStuff(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			logg.Printf("done")
			return
		default:
			logg.Printf("work")
			time.Sleep(1 * time.Second)
		}

	}

}

func main() {
	logg = log.New(os.Stdout, "", log.Ltime)
	someHandler()
	time.Sleep(2 * time.Second)
	logg.Printf("down")
}

```
可以看到，当cancel被触发时，doStuff的select{}中ctx.Done的逻辑被触发，然后return

- func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
    - 提供了一个父context的copy和一个cancel的返回，如果父context的deadline比d更早，那么会等同于父context，简单的说就是到时间出发context周期的结束，同样也能主动触发


``` golang
package main

import (
	"context"
	"log"
	"os"
	"time"
)

var logg *log.Logger

func someHandler() {
	d := time.Now().Add(3 * time.Second)
	ctx, cancel := context.WithDeadline(context.Background(), d)
	go doStuff(ctx)

	time.Sleep(5 * time.Second)
	cancel()

}

func doStuff(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			logg.Printf("done")
			return
		default:
			logg.Printf("work")
			time.Sleep(1 * time.Second)
		}

	}

}

func main() {
	logg = log.New(os.Stdout, "", log.Ltime)
	someHandler()
	time.Sleep(2 * time.Second)
	logg.Printf("down")
}

```

可以看到，设置deadline的时间早于cancel调用的时间，因此到了deadline的时间就优先触发了完成状态


- func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
    - 这个方法等同于WithDeadline(parent, time.Now().Add(timeout))，也即是在当前时间上timeout为deadline


- func WithValue(parent Context, key, val interface{}) Context
    - 返回父context的拷贝并且将key,val联系到一起


``` golang
package main

import (
	"context"
	"fmt"
)

func main() {

	type favContextKey string

	f := func(ctx context.Context, k favContextKey) {
		if v := ctx.Value(k); v != nil {
			fmt.Println("found value:", v)
			return
		}
		fmt.Println("key not found:", k)
	}

	k := favContextKey("language")
	ctx := context.WithValue(context.Background(), k, "Go")

	f(ctx, k)
	f(ctx, favContextKey("color"))

}

```

可以看到输出为
``` shell
found value: Go
key not found: color
```

因此看上去可以通过withValue管理很多的子context的内容

# 其他需要关注的点

如果父context被取消，结束了，相关的子context同样会被结束，取消

``` golang
package main

import (
	"context"
	// "fmt"
	"log"
	"os"
	"time"
)

var logg *log.Logger

func func1(ctx context.Context, a string) {
	for {
		select {
		case <-ctx.Done():
			logg.Printf("END, %s", a)
			return
		}
		time.Sleep(1 * time.Second)
	}
}

func main() {
	logg = log.New(os.Stdout, "", log.Ltime)
	rctx, rcancel := context.WithCancel(context.Background())
	cctx, _ := context.WithCancel(rctx)
	go func1(cctx, "func1")
	go func1(rctx, "func2")

	time.Sleep(5 * time.Second)
	// ccancel()
	// logg.Printf("child End")

	rcancel()
	logg.Printf("root End")
	time.Sleep(2 * time.Second)
}

```

输出为

``` shell
18:46:50 END, func1
18:46:50 END, func2
18:46:50 root End
```

可以看到，rctx的cancel被触发了，rctx和cctx都会被取消掉



