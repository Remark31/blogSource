---
title: POW共识初探
date: 2018-06-13 11:35:54
tags: [区块链,以太坊,共识]
---

此处，我们对以太坊的POW共识的实现进行一些简单的梳理，大概了解下POW究竟是如何运作的。


# 区块结构

首先了解下以太坊的区块结构

``` golang
// /core/type/block.go
// 区块结构
type Block struct {
	header       *Header       //区块头
	uncles       []*Header     //叔伯区块头
	transactions Transactions  //交易列表

	// caches
	hash atomic.Value
	size atomic.Value

	td *big.Int // 整体复杂度，在该链中包含该区块的所有区块的复杂度的和

	ReceivedAt   time.Time
	ReceivedFrom interface{}
}
```

再看表头结构

``` golang
type Header struct {
	ParentHash  common.Hash    `json:"parentHash"       gencodec:"required"`  // 父区块hash
	UncleHash   common.Hash    `json:"sha3Uncles"       gencodec:"required"`
	// 叔伯区块hash
	Coinbase    common.Address `json:"miner"            gencodec:"required"`
	// 矿工地址
	Root        common.Hash    `json:"stateRoot"        gencodec:"required"`  // state Trie根节点rlp hash
	TxHash      common.Hash    `json:"transactionsRoot" gencodec:"required"`  // tx Trie根节点rlp hash
	ReceiptHash common.Hash    `json:"receiptsRoot"     gencodec:"required"`   // receipt Trie根节点rlp hash
	Bloom       Bloom          `json:"logsBloom"        gencodec:"required"`
	Difficulty  *big.Int       `json:"difficulty"       gencodec:"required"`  // 难度值
	Number      *big.Int       `json:"number"           gencodec:"required"`  // 区块编号
	GasLimit    uint64         `json:"gasLimit"         gencodec:"required"`   // 区块内所有Gas消耗的上限
	GasUsed     uint64         `json:"gasUsed"          gencodec:"required"`   // 区块内所有交易消耗的gas
	Time        *big.Int       `json:"timestamp"        gencodec:"required"`
	Extra       []byte         `json:"extraData"        gencodec:"required"`
	MixDigest   common.Hash    `json:"mixHash"          gencodec:"required"`
	Nonce       BlockNonce     `json:"nonce"            gencodec:"required"`
	// POW的Nonce
}

```

在表头结构中，与POW算法相关的是区块hash，Nonce以及Difficulty

# POW 算法

POW简单的描述可以表示为如下公式
```
RAND(hash, Nonce) <= M / Difficulty
```

M是一个极大的整数，在hash和M确定的情况下，增大Difficulty会使满足条件的Nonce更少，暴力算出的概率会降低，反之亦然

整体的POW算法就是在得到了需要打包的交易之后开始计算Nonce，在算出来后开始打包生成区块，接下来看在以太坊中的具体实现


``` golang

func (ethash *Ethash) mine(block *types.Block, id int, seed uint64, abort chan struct{}, found chan *types.Block) {
	// 从block.header中获取部分数据，初始化
	var (
		header  = block.Header()
		hash    = header.HashNoNonce().Bytes()
		target  = new(big.Int).Div(maxUint256, header.Difficulty)
		number  = header.Number.Uint64()
		dataset = ethash.dataset(number)
	)
	// 开始生成随机数nounce，直到生成后退出
	var (
		attempts = int64(0)
		nonce    = seed
	)
	logger := log.New("miner", id)
	logger.Trace("Started ethash search for new nonces", "seed", seed)
search:
	for {
		select {
		case <-abort:
			// 挖矿停止，更新状态并且退出
			logger.Trace("Ethash nonce search aborted", "attempts", nonce-seed)
			ethash.hashrate.Mark(attempts)
			break search

		default:
			// 
			attempts++
			if (attempts % (1 << 15)) == 0 {
				ethash.hashrate.Mark(attempts)
				attempts = 0
			}
			// 计算POW的nounce值
			digest, result := hashimotoFull(dataset.dataset, hash, nonce)
			if new(big.Int).SetBytes(result).Cmp(target) <= 0 {
				// 发现了正确的nounce，使用它创建新的block header
				header = types.CopyHeader(header)
				header.Nonce = types.EncodeNonce(nonce)
				header.MixDigest = common.BytesToHash(digest)

				// 返回区块，并且退出
				select {
				case found <- block.WithSeal(header):
					logger.Trace("Ethash nonce found and reported", "attempts", nonce-seed, "nonce", nonce)
				case <-abort:
					logger.Trace("Ethash nonce found but discarded", "attempts", nonce-seed, "nonce", nonce)
				}
				break search
			}
			// 如果没有找到则增加nonce的值
			nonce++
		}
	}
	// 保持dataset数据不被回收
	runtime.KeepAlive(dataset)
}

```

可以看到整个挖矿方法，在首先获取了Block的header数据之后开始了组装，然后开始根据随机的种子`ethash.rand.Int63()`开始遍历的计算`hashimotoFull`的结果是否满足`new(big.Int).SetBytes(result).Cmp(target) <= 0`，如果满足，则说明找到了正确的nonce，就可以开始打包区块。

接下来我们来看下如何的计算`hashimotoFull`的

``` golang
func hashimoto(hash []byte, nonce uint64, size uint64, lookup func(index uint32) []uint32) ([]byte, []byte) {
	// Calculate the number of theoretical rows (we use one buffer nonetheless)
	rows := uint32(size / mixBytes)

	// Combine header+nonce into a 64 byte seed
	seed := make([]byte, 40)
	copy(seed, hash)
	binary.LittleEndian.PutUint64(seed[32:], nonce)

	seed = crypto.Keccak512(seed)
	seedHead := binary.LittleEndian.Uint32(seed)

	// Start the mix with replicated seed
	mix := make([]uint32, mixBytes/4)
	for i := 0; i < len(mix); i++ {
		mix[i] = binary.LittleEndian.Uint32(seed[i%16*4:])
	}
	// Mix in random dataset nodes
	temp := make([]uint32, len(mix))

	for i := 0; i < loopAccesses; i++ {
		parent := fnv(uint32(i)^seedHead, mix[i%len(mix)]) % rows
		for j := uint32(0); j < mixBytes/hashBytes; j++ {
			copy(temp[j*hashWords:], lookup(2*parent+j))
		}
		fnvHash(mix, temp)
	}
	// Compress mix
	for i := 0; i < len(mix); i += 4 {
		mix[i/4] = fnv(fnv(fnv(mix[i], mix[i+1]), mix[i+2]), mix[i+3])
	}
	mix = mix[:len(mix)/4]

	digest := make([]byte, common.HashLength)
	for i, val := range mix {
		binary.LittleEndian.PutUint32(digest[i*4:], val)
	}
	return digest, crypto.Keccak256(append(seed, digest...))
}
```

这一部分代码就很头疼了。没有太多的逻辑，完全是符合公式的计算。

最后，还有一个关键点就是如何计算Difficulty


``` golang
//  /consensus/ethash/consensus.go
func CalcDifficulty(config *params.ChainConfig, time uint64, parent *types.Header) *big.Int {
	next := new(big.Int).Add(parent.Number, big1)
	switch {
	case config.IsByzantium(next):
		return calcDifficultyByzantium(time, parent)
	case config.IsHomestead(next):
		return calcDifficultyHomestead(time, parent)
	default:
		return calcDifficultyFrontier(time, parent)
	}
}
```

这里根据不同的配置计算不同情况下的难度值，这里简单的以拜占庭为例

``` golang

// 此处在计算时主要考虑的是其父区块的复杂度以及父区块的生成时间，和自己的生成时间
func calcDifficultyByzantium(time uint64, parent *types.Header) *big.Int {
	// https://github.com/ethereum/EIPs/issues/100.
	// algorithm:
	// diff = (parent_diff +
	//         (parent_diff / 2048 * max((2 if len(parent.uncles) else 1) - ((timestamp - parent.timestamp) // 9), -99))
	//        ) + 2^(periodCount - 2)

	bigTime := new(big.Int).SetUint64(time)
	bigParentTime := new(big.Int).Set(parent.Time)

	// holds intermediate values to make the algo easier to read & audit
	x := new(big.Int)
	y := new(big.Int)

	// (2 if len(parent_uncles) else 1) - (block_timestamp - parent_timestamp) // 9
	x.Sub(bigTime, bigParentTime)
	x.Div(x, big9)
	if parent.UncleHash == types.EmptyUncleHash {
		x.Sub(big1, x)
	} else {
		x.Sub(big2, x)
	}
	// max((2 if len(parent_uncles) else 1) - (block_timestamp - parent_timestamp) // 9, -99)
	if x.Cmp(bigMinus99) < 0 {
		x.Set(bigMinus99)
	}
	// parent_diff + (parent_diff / 2048 * max((2 if len(parent.uncles) else 1) - ((timestamp - parent.timestamp) // 9), -99))
	y.Div(parent.Difficulty, params.DifficultyBoundDivisor)
	x.Mul(y, x)
	x.Add(parent.Difficulty, x)

	// minimum difficulty can ever be (before exponential factor)
	if x.Cmp(params.MinimumDifficulty) < 0 {
		x.Set(params.MinimumDifficulty)
	}
	// calculate a fake block number for the ice-age delay:
	//   https://github.com/ethereum/EIPs/pull/669
	//   fake_block_number = min(0, block.number - 3_000_000
	fakeBlockNumber := new(big.Int)
	if parent.Number.Cmp(big2999999) >= 0 {
		fakeBlockNumber = fakeBlockNumber.Sub(parent.Number, big2999999) // Note, parent is 1 less than the actual block number
	}
	// for the exponential factor
	periodCount := fakeBlockNumber
	periodCount.Div(periodCount, expDiffPeriod)

	// the exponential factor, commonly referred to as "the bomb"
	// diff = diff + 2^(periodCount - 2)
	if periodCount.Cmp(big1) > 0 {
		y.Sub(periodCount, big2)
		y.Exp(big2, y, nil)
		x.Add(x, y)
	}
	return x
}
```

至此，POW共识的实现简单的梳理完成。
