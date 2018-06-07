---
title: libp2p节点id的生成
date: 2018-06-07 12:58:54
tags: [区块链,p2p网络,ipfs]
---

最近对各种区块链的p2p网络的节点id的生成过程产生了兴趣，最近从ipfs的libp2p开始分析。

# 节点的表现形式

我们首先从go-libp2p提供的example里面中的host.go入手：

```golang
package main

import (
	"context"
	"strings"
	"fmt"

	libp2p "github.com/libp2p/go-libp2p"
	crypto "gx/ipfs/QmaPbCnUMBohSGo3KnxEa2bHqyJVVeEEcwtqJAYxerieBo/go-libp2p-crypto"
)

func main() {
	
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// 完全使用默认配置，生成一个libp2p节点
	h, err := libp2p.New(ctx)
	if err != nil {
		panic(err)
	}

	fmt.Printf("Hello World, my hosts ID is %s\n", h.ID())

    // 创建自己的私钥，并且指定监听模式以及端口生成libp2p节点
    a := strings.NewReader("aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa")
    
	priv, _, err := crypto.GenerateEd25519Key(a)
	if err != nil {
		panic(err)
	}

	h2, err := libp2p.New(ctx,
		//使用私钥
		libp2p.Identity(priv),

		//指定监听端口
		libp2p.ListenAddrStrings("/ip4/0.0.0.0/tcp/9000"),
	)
	if err != nil {
		panic(err)
	}

	fmt.Printf("Hello World, my second hosts ID is %s\n", h2.ID())
}

```

在如上的范例中生成了两个libp2p的节点，我们运行两次后看下生成的ID分别是啥

![libp2p_host测试图](/imgs/libp2p_host_test.jpeg)

从这里可以看到，节点ID的结果与私钥是一一对应的，接下来我们就开始探寻libp2p中节点id到底是如何生成的

# 节点id生成的方法

对libp2p.New代码进行一阵分析，可以看到peer.ID的生成是通过私钥得到对应公钥，然后再通过公钥获得的，核心方法如下所示：

```golang

type ID string

func IDFromPublicKey(pk ic.PubKey) (ID, error) {
	b, err := pk.Bytes()
	if err != nil {
		return "", err
	}
	var alg uint64 = mh.SHA2_256
	if len(b) <= MaxInlineKeyLength {
		alg = mh.ID
	}
	hash, _ := mh.Sum(b, alg, -1)
	return ID(hash), nil
}
```

其中，mh是libp2p自己的hash库 `mh "github.com/multiformats/go-multihash"`，其实就是单纯的将公钥的hash计算出作为libp2p的节点hash，是个简单易懂的策略，和以太坊的非常类似。
