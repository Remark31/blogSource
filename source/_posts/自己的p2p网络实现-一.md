---
title: 自己的p2p网络实现(一)
date: 2018-06-28 11:54:00
tags: [p2p]
---

# 基础数据结构

``` golang
type IpfsDHT struct {
	host      host.Host        // 提供了基础的网络服务接口
	self      peer.ID          // 自己的peer
	peerstore pstore.Peerstore // 线程安全的方法去存储peer以及相关信息

	datastore ds.Datastore // 本地数据

	routingTable *kb.RoutingTable // 不同距离的节点的路由表集合
	providers    *providers.ProviderManager

	birth time.Time // peer启动时间

	Validator record.Validator

	ctx  context.Context
	proc goprocess.Process

	strmap map[peer.ID]*messageSender
	smlk   sync.Mutex

	plk sync.Mutex

	protocols []protocol.ID // DHT协议
}
```

这里简单的回顾一下host.Host与pstore.Peerstore提供的方法

这是host提供的方法，主要还是网络连接，协议处理相关的内容

``` golang

type Host interface {
	// 返回了连接上这个Host的本地peer的ID
	ID() peer.ID

	// 返回了与peer相关的一些信息，例如地址以及公私钥
	Peerstore() pstore.Peerstore

	// 返回这个host目前监听的地址
	Addrs() []ma.Multiaddr

	// 这个Host目前的网络接口
	Network() inet.Network

	// 到协议处理的多路流
	Mux() *msmux.MultistreamMuxer

    // 保证了目前有一条连接，从当前host到目标peer，如果当前连接失效，会继续连接或者返回错误
	Connect(ctx context.Context, pi pstore.PeerInfo) error

	// 在host的mux上设置一个协议处理器(线程安全)
	SetStreamHandler(pid protocol.ID, handler inet.StreamHandler)

    // 在Host上设置一个合适的协议处理器通过一个协议选择的方法
	SetStreamHandlerMatch(protocol.ID, func(string) bool, inet.StreamHandler)

	// 移除一个协议流处理
	RemoveStreamHandler(pid protocol.ID)

	// 使用一个指定协议与指定peer相连
	NewStream(ctx context.Context, p peer.ID, pids ...protocol.ID) (inet.Stream, error)

	// 关闭网络服务
	Close() error

	// 返回这个Host的连接管理器
	ConnManager() ifconnmgr.ConnManager
}

```

peerstore提供的方法就简单很多，主要还是返回一些Peer相关的基础信息

``` golang
type Peerstore interface {
	// 地址管理器
	AddrBook
	
	// 管理peer的公钥/私钥
	KeyBook
	
	// 追踪peer的性能数据
	Metrics

	// 返回存储到目前peerstore中的peerID
	Peers() []peer.ID

	// 返回当前peerID对应的peerInfo结构体
	PeerInfo(peer.ID) PeerInfo

	// 提供了一个简单的存储其他的与Peer相关的键值对的地方
	Get(id peer.ID, key string) (interface{}, error)
	Put(id peer.ID, key string, val interface{}) error

	GetProtocols(peer.ID) ([]string, error)
	AddProtocols(peer.ID, ...string) error
	SetProtocols(peer.ID, ...string) error
	SupportsProtocols(peer.ID, ...string) ([]string, error)
}
```

# 基本方法

接下来我们从KAD网络的基础消息类型入手分析

## PING

此处暂时没有发现libp2p中的实现

## STORE

ipfs中的store不是简单的在一个节点中进行store操作，它会找到最临近的k个节点，然后执行这个操作


``` golang
func (dht *IpfsDHT) PutValue(ctx context.Context, key string, value []byte, opts ...ropts.Option) (err error) {
	eip := log.EventBegin(ctx, "PutValue")
	defer func() {
		eip.Append(loggableKey(key))
		if err != nil {
			eip.SetError(err)
		}
		eip.Done()
	}()
	log.Debugf("PutValue %s", key)

    // 构建key-value的结构体
	rec := record.MakePutRecord(key, value)
	rec.TimeReceived = proto.String(u.FormatRFC3339(time.Now()))
	err = dht.putLocal(key, rec)
	if err != nil {
		return err
	}
    
    
    // 返回网络中当前距离指定key最近的k个节点
	pchan, err := dht.GetClosestPeers(ctx, key)
	if err != nil {
		return err
	}
    
    
    // 将这最近的k个节点分别设置上键值对
	wg := sync.WaitGroup{}
	for p := range pchan {
		wg.Add(1)
		go func(p peer.ID) {
			ctx, cancel := context.WithCancel(ctx)
			defer cancel()
			defer wg.Done()
			notif.PublishQueryEvent(ctx, &notif.QueryEvent{
				Type: notif.Value,
				ID:   p,
			})

			err := dht.putValueToPeer(ctx, p, key, rec)
			if err != nil {
				log.Debugf("failed putting value to peer: %s", err)
			}
		}(p)
	}
	wg.Wait()
	return nil
}
```

这里可以看到libp2p在执行操作的时候会去调用

```
notif.PublishQueryEvent(ctx, &notif.QueryEvent{
	Type: notif.Value,
	ID:   p,
})
```

目前暂时还没看出来这个QueryEvent的作用和目的。这个是接下来需要关注的问题和点了。



## FIND_NODE

此处的目标是在当前p2p网络中找到指定id的peer

``` golang
func (dht *IpfsDHT) FindPeer(ctx context.Context, id peer.ID) (_ pstore.PeerInfo, err error) {
	eip := log.EventBegin(ctx, "FindPeer", id)
	defer func() {
		if err != nil {
			eip.SetError(err)
		}
		eip.Done()
	}()

	// 检查下是否已经连接上了
	if pi := dht.FindLocal(id); pi.ID != "" {
		return pi, nil
	}

    // 返回当前桶中
	peers := dht.routingTable.NearestPeers(kb.ConvertPeerID(id), AlphaValue)
	if len(peers) == 0 {
		return pstore.PeerInfo{}, kb.ErrLookupFailure
	}

	// 运气好，桶中内容直接发现了
	for _, p := range peers {
		if p == id {
			log.Debug("found target peer in list of closest peers...")
			return dht.peerstore.PeerInfo(p), nil
		}
	}

	// 开始查询
	parent := ctx
	query := dht.newQuery(string(id), func(ctx context.Context, p peer.ID) (*dhtQueryResult, error) {
		notif.PublishQueryEvent(parent, &notif.QueryEvent{
			Type: notif.SendingQuery,
			ID:   p,
		})

		pmes, err := dht.findPeerSingle(ctx, p, id)
		if err != nil {
			return nil, err
		}

		closer := pmes.GetCloserPeers()
		clpeerInfos := pb.PBPeersToPeerInfos(closer)

		// 如果发现了Peer，设置结果success为true
		for _, npi := range clpeerInfos {
			if npi.ID == id {
				return &dhtQueryResult{
					peer:    npi,
					success: true,
				}, nil
			}
		}

		notif.PublishQueryEvent(parent, &notif.QueryEvent{
			Type:      notif.PeerResponse,
			ID:        p,
			Responses: clpeerInfos,
		})

		return &dhtQueryResult{closerPeers: clpeerInfos}, nil
	})

	// 循环的调用该方法，直到找不到peer或者找到最后的值
	result, err := query.Run(ctx, peers)
	if err != nil {
		return pstore.PeerInfo{}, err
	}

	log.Debugf("FindPeer %v %v", id, result.success)
	if result.peer.ID == "" {
		return pstore.PeerInfo{}, routing.ErrNotFound
	}

	return *result.peer, nil
}
```

这里的query.Run的迭代查找的实现非常有意思，最终的效果是能在迭代的peer桶中找到所需要的peer列表

## FIND_VALUE

此处的目标是在p2p网络中找到指定的value

``` golang

func (dht *IpfsDHT) GetValue(ctx context.Context, key string, opts ...ropts.Option) (_ []byte, err error) {
	eip := log.EventBegin(ctx, "GetValue")
	defer func() {
		eip.Append(loggableKey(key))
		if err != nil {
			eip.SetError(err)
		}
		eip.Done()
	}()
	
	
	// 设置超时时间
	ctx, cancel := context.WithTimeout(ctx, time.Minute)
	defer cancel()

	var cfg ropts.Options
	if err := cfg.Apply(opts...); err != nil {
		return nil, err
	}

	responsesNeeded := 0
	if !cfg.Offline {
		responsesNeeded = getQuorum(&cfg)
	}
    
    // 迭代的获取Values的值
	vals, err := dht.GetValues(ctx, key, responsesNeeded)
	if err != nil {
		return nil, err
	}

	recs := make([][]byte, 0, len(vals))
	for _, v := range vals {
		if v.Val != nil {
			recs = append(recs, v.Val)
		}
	}
	if len(recs) == 0 {
		return nil, routing.ErrNotFound
	}

	i, err := dht.Validator.Select(key, recs)
	if err != nil {
		return nil, err
	}

	best := recs[i]
	log.Debugf("GetValue %v %v", key, best)
	if best == nil {
		log.Errorf("GetValue yielded correct record with nil value.")
		return nil, routing.ErrNotFound
	}

	fixupRec := record.MakePutRecord(key, best)
	for _, v := range vals {
		// 如果有人发来的值不是正确的值，就会更新
		if !bytes.Equal(v.Val, best) {
			go func(v RecvdVal) {
				if v.From == dht.self {
					err := dht.putLocal(key, fixupRec)
					if err != nil {
						log.Error("Error correcting local dht entry:", err)
					}
					return
				}
				ctx, cancel := context.WithTimeout(dht.Context(), time.Second*30)
				defer cancel()
				err := dht.putValueToPeer(ctx, v.From, key, fixupRec)
				if err != nil {
					log.Error("Error correcting DHT entry: ", err)
				}
			}(v)
		}
	}

	return best, nil
}

```

这里`VALUE`的更新只会对于以前存储了这个`VALUE`的节点。这个是挺有趣的故事，按照论文的说法会将其它临近的节点缓存上，这里有些存疑。

> 大部分的操作都是基于上述的查询过程实现的。为了存储一个<key, value>对，一个参与者定位出k个最接近key的节点，然后向它们发送STORERPC请求。另外，每个节点在需要保活的时候，都会重新发布<key, value>对，这个会在之后说。这种做法会保证<key, value>对有很大可能持久化。对于现在的Kademlia应用（比如文件共享），我们还会要求<key, value>对的原始发布者每隔24小时再重新发布一次。因为，为了限制系统中不可用的索引信息不至于太多，一个<key, value>对，在发布24小时后，默认就会被设为过期。对于其它的应用，比如数字证书或用于值映射的加密哈希，更长的过期时间也许会更合适。   
> 要找到一个<key, value>对，一个节点先开始找到最接近这个key的k个节点。但是，值查找用的是FIND_VALUE而不是FIND_NODERPC。并且，只要有一个节点返回了value，查找过程就立即停止。为了缓存，一旦一个查找成功，请求节点将会在它发现的，那些最接近key的，但又没有存储这个<key, value>对的节点上存储这个键值对。


这里在关注完成kad网络的基本操作后，其他的操作暂时先不再赘述，我们开始考虑如何在使用kad-dht包来实现一个p2p发现网络。


# 实现思路

整体来说我们先考虑验证，再考虑实际的实现

## demo模型

整体要实现的demo的场景很简单，就是一个3个节点的发现网络

## 验证

这里要解决的是如何kad网络中才能正常

这里我们需要参考以太坊的devp2p的使用方法来实现一个基础的kad发现网络


1. 启动3个KAD节点
2. 通过KAD节点来进行发现和通信

下面是我们最终实现的代码

``` golang


package main

import (
	"context"
	// "crypto/rand"
	"flag"

	libp2p "github.com/libp2p/go-libp2p"
	crypto "github.com/libp2p/go-libp2p-crypto"
	p2phost "github.com/libp2p/go-libp2p-host"
	dht "github.com/libp2p/go-libp2p-kad-dht"
	dhtopts "github.com/libp2p/go-libp2p-kad-dht/opts"
	net "github.com/libp2p/go-libp2p-net"
	peer "github.com/libp2p/go-libp2p-peer"
	peerstore "github.com/libp2p/go-libp2p-peerstore"
	ma "github.com/multiformats/go-multiaddr"

	"encoding/json"

	"fmt"
	"log"
	"os"
	"strconv"
	// "runtime/debug"
	"bufio"
	"strings"
	// "io"
	// "io/ioutil"
)

var logg *log.Logger

const PID = "/p2p/1.0.0"

const BootnodeCon = 1

func init() {
	logg = log.New(os.Stdout, "", log.Ltime)
}

type Msg struct {
	Code    uint64
	Size    uint32
	Payload []byte
}

// HostWrapper 主要用于streamHandler()方法可以操纵host对象
type HostWrapper struct {
	Host p2phost.Host
}

type blankValidator struct{}

func (blankValidator) Validate(_ string, _ []byte) error        { return nil }
func (blankValidator) Select(_ string, _ [][]byte) (int, error) { return 0, nil }

func constructDHT(ctx context.Context, listenstring string, key string) (*dht.IpfsDHT, p2phost.Host, error) {
	host, err := constructPeerHost(ctx, listenstring, key)
	if err != nil {
		logg.Printf("constructPeerHost: %s", err)
		return nil, nil, err
	}
	vdht, err := dht.New(
		ctx, host,
		dhtopts.NamespacedValidator("v", blankValidator{}),
	)

	hostAddr, _ := ma.NewMultiaddr(fmt.Sprintf("/ipfs/%s", host.ID().Pretty()))

	addr := host.Addrs()[0]
	fullAddr := addr.Encapsulate(hostAddr)
	logg.Printf("addr: %s", fullAddr)
	return vdht, host, err
}

func constructPeerHost(ctx context.Context, listenstring string, key string) (p2phost.Host, error) {
	var options []libp2p.Option
	if key != "" {
		a := strings.NewReader(key)
		pkey, _, err := crypto.GenerateEd25519Key(a)
		if err != nil {
			logg.Printf("%s", err)
			return nil, err
		}

		options = append(options, libp2p.ListenAddrStrings(listenstring), libp2p.Identity(pkey))

	} else {
		options = append(options, libp2p.ListenAddrStrings(listenstring))
	}

	return libp2p.New(ctx, options...)
}

func makePort(port int) string {
	return "/ip4/0.0.0.0/tcp/" + strconv.Itoa(port)
}

func getAddr(target string) (ma.Multiaddr, peer.ID, error) {
	ipfsaddr, err := ma.NewMultiaddr(target)
	if err != nil {
		logg.Printf("getAddr NewMultiaddr Err: %s", err)
		return nil, "", err
	}

	pid, err := ipfsaddr.ValueForProtocol(ma.P_IPFS)
	if err != nil {
		logg.Printf("getAddr ValueForProtocol Err: %s", err)
		return nil, "", err
	}

	peerid, err := peer.IDB58Decode(pid)
	if err != nil {
		logg.Printf("getAddr IDB58Decode Err: %s", err)
		return nil, "", err
	}

	// Decapsulate the /ipfs/<peerID> part from the target
	// /ip4/<a.b.c.d>/ipfs/<peer> becomes /ip4/<a.b.c.d>
	targetPeerAddr, _ := ma.NewMultiaddr(
		fmt.Sprintf("/ipfs/%s", peer.IDB58Encode(peerid)))
	targetAddr := ipfsaddr.Decapsulate(targetPeerAddr)

	return targetAddr, peerid, nil
}

func DecodePackage(data []byte) (*Msg, error) {
	msg := &Msg{}
	err := json.Unmarshal(data, msg)
	if err != nil {
		logg.Printf("DecodePackage Error: %s", err)
		return nil, err
	}
	return msg, nil
}

func WritePackage(rw *bufio.ReadWriter, code uint64, data []byte) ([]byte, error) {
	msg := &Msg{Code: code, Payload: data, Size: uint32(len(data))}
	// logg.Printf("Msg: %v", msg)
	return json.Marshal(msg)
}

func main() {

	// func NewDHT(ctx context.Context, h host.Host, dstore ds.Batching) *IpfsDHT
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// libp2p.ListenAddrStrings("/ip4/0.0.0.0/tcp/9000")
	key := flag.String("k", "", "private key")
	target := flag.String("b", "", "target peer to dial")
	findnode := flag.String("f", "", "target peer to find")
	listenport := flag.Int("p", 9000, "port to listen")
	flag.Parse()

	vdht, host, err := constructDHT(ctx, makePort(*listenport), *key)
	if err != nil {
		logg.Printf("constructDHT a: %s", err)
		return
	}

	// logg.Printf("dhta : %v", dhta)
	logg.Printf("自己的peerID = %v\n", host.ID())

	h := &HostWrapper{
		Host: host,
	}

	if *target == "" {
		logg.Printf("Listen for connect")
		logg.Printf("PID: %s Addr: %s", host.ID(), host.Addrs())

		content := host.Peerstore().Peers()
		for i := range content {
			logg.Printf("---> 当前PeerInfo[%d] = %v\n", i, host.Peerstore().PeerInfo(content[i]))
		}

		host.SetStreamHandler(PID, h.handleStream)
		//go listenPeerStoreChange(host)

	} else {

		host.SetStreamHandler(PID, h.handleStream)
		targetAddr, peerid, err := getAddr(*target)
		if err != nil {
			logg.Printf("target Error: %s", err)
			return
		}
		logg.Printf("muaddr : %s", targetAddr)

		host.Peerstore().AddAddrs(peerid, []ma.Multiaddr{targetAddr}, peerstore.PermanentAddrTTL)
		vdht.Update(ctx, peerid)

		content := host.Peerstore().Peers()
		for i := range content {
			logg.Printf("---> 当前PeerInfo[%d] = %v\n", i, host.Peerstore().PeerInfo(content[i]))
		}

		s, err := host.NewStream(ctx, peerid, PID)
		if err != nil {
			logg.Printf("NewStream Error: %s", err)
			return
		}

		rw := bufio.NewReadWriter(bufio.NewReader(s), bufio.NewWriter(s))

		// 封包，并且发送
		msg, err := WritePackage(rw, 1, []byte("yoyoyo\n"))
		if err != nil {
			logg.Printf("WritePackage Err: %s", err)
			return
		}
		logg.Printf("msg : %v", msg)
		_, err = rw.Write(msg)
		if err != nil {
			logg.Printf("Write Err: %s", err)
			return
		}
		rw.WriteByte('\n')
		rw.Flush()

	}

	if *findnode != "" {
		_, peerid, err := getAddr(*findnode)
		if err != nil {
			logg.Printf("getAddr error: %v\n", err)
			return
		}
		logg.Printf("find node: id =%v\n", peerid)
		targetPeerInfo, err := vdht.FindPeer(ctx, peerid)
		if err != nil {
			logg.Printf("findPeer error: %v\n", err)
			return
		}
		logg.Printf("find peerid = %v SUCCESS with info = %v\n", peerid, targetPeerInfo)
	}


	select {}

}



func (h *HostWrapper) handleStream(s net.Stream) {
	logg.Printf("new stream")
	// 对接收到的包进行处理
	rw := bufio.NewReadWriter(bufio.NewReader(s), bufio.NewWriter(s))

	data, err := rw.ReadBytes('\n')
	if err != nil {
		logg.Printf("handleStream ReadString Err: %s", err)
	}
	logg.Printf("data: %s", data)
	msg, err := DecodePackage(data)
	if err != nil {
		logg.Printf("handleStream DecodePackage Error: %s", err)
		return
	}
	switch msg.Code {
	case BootnodeCon:
		conn := s.Conn()
		rma := conn.RemoteMultiaddr()
		rpid := conn.RemotePeer()
		h.Host.Peerstore().AddAddrs(rpid, []ma.Multiaddr{rma}, peerstore.PermanentAddrTTL)
		logg.Printf("got handshake signal: %v", string(msg.Payload))
	default:
		logg.Printf("nothing to get")
	}

}


```

如上的demo实际上实现的就是节点的发现与通信的功能

整体实验进行如下：

1. 首先启动节点1
2. 接下来启动节点2，设置节点2的bootnode节点为节点1
3. 最后启动节点3，设置节点3的bootnode节点为节点1，并且在节点3上尝试findnode节点2


最终效果图如下

- 节点1输出   
   ![节点1输出](/imgs/host1.png)
- 节点2输出   
   ![节点2输出](/imgs/host2.png)
- 节点3输出   
   ![节点3输出](/imgs/host3.png)

可以看到，节点3通过k桶中的节点1的地址最终成功的发现到了节点2


# 总结

整个libp2p的kad网络的使用是非常有趣的，kad包其实本质上只去维护了k桶的关系以及实现了kad网络的基本操作，真正网络层的消息处理与地址维护还是在host层中，在我们的demo中虽然实现了一个小的p2p网络，可以节点之间相互发现，但是这个还不能算是一个真正的p2p网络，因为这个发现还是手动的，缺少一个任务调度层去自发的调度各种类型的任务。再之后的文章中我们会去讨论如何来调度这些任务。