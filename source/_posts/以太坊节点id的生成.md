---
title: 以太坊节点id的生成
date: 2018-08-08 15:30:50
tags: [区块链,p2p网络,以太坊]
---


# 前言
继续之前的节点id生成的故事，这次我们讨论的是以太坊的节点id生成

# 节点id表现形式

首先我们看一下一个完整的以太坊id的表现形式：

```
enode://<hex node id>@10.3.58.6:30303?discport=30301
```

以太坊的地址是由4个部分组成

- 前缀: endoe://
- 节点id: <hex node id>
- 节点ip: 10.3.58.6
- 端口：这里分为tcp端口与udp端口，分别是30303和30301

接下来我们关心一下id的生成方式

# 节点id生成

``` golang
func PubkeyID(pub *ecdsa.PublicKey) NodeID {
	var id NodeID
	pbytes := elliptic.Marshal(pub.Curve, pub.X, pub.Y)
	if len(pbytes)-1 != len(id) {
		panic(fmt.Errorf("need %d bit pubkey, got %d bits", (len(id)+1)*8, len(pbytes)))
	}
	copy(id[:], pbytes[1:])
	return id
}
```

核心的id生成方法如上所示，通过ecdsa的公钥获取曲线，X，Y生成byte[1]，然后根据这个产生id，再用hex进行编码就可以得到可读的字符串版本的nodeid了。






# 备注
```
[1] Marshal converts a point into the uncompressed form specified in section 4.3.6 of ANSI X9.62.
```
