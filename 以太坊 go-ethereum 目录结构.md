﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿# 以太坊 go-ethereum 目录结构
| 目录 | 介绍|
|:--|:--|
| accounts | 实现的是以太坊账户管理 |
| bmt | 二进制的默克尔树的实现 |
| build | 实现编译和构建相关脚本与配置 |
| cmd | 命令行工具 |
| common | 提供公共的工具类 |
| consensus | 以太坊共识算法 |
| core | 以太坊核心的数据结构和算法(区块，虚拟机，区块链...) |
| crypto | 以太坊加密算法的实现 |
| eth | 以太坊协议 |
| ethClient | 以太坊 RPC 客户端 |
| ethdb | 以太坊数据库：包含持久化数据库与测试使用的内存数据库 |
| ethStatus | 以太坊网络状态的报告 |
| event | 以太坊事件 |
| les | 以太坊轻量级协议 |
| light | 为以太坊轻量级客户端提供索引功能 |
| log | 提供日志功能 |
| miner | 以太坊挖矿 |
| mobile | 移动端相关管理 |
| node | 以太坊中各种类型的节点 |
| p2p | 以太坊主要的网络协议 |
| rlp | 以太坊序列化处理 |
| swarm | 以太坊 swarm 网络处理 |
| tests | 测试 |
| trie | 以太坊主要数据结构 默克尔-帕特里夏树 的实现 |
| whisper | 提供 whisper 节点协议，主要用于以太坊DAPP之间的通信 |

# 区块与区块链
## 区块(Block)
core/types/block.go
 1. 所有与账户相关的活动都会以交易的格式存储到 Block 中，每个 block 中都会有一个交易列表
 2.  交易执行结构，日志记录
 3. 不同的区块通过 ParentHash 进行链接


```go
// Block represents an entire block in the Ethereum blockchain.
// 区块结构
type Block struct {
	header       *Header
	uncles       []*Header
	transactions Transactions

	// caches
	// 哈希与size的缓存
	// atomic:原子操作
	hash atomic.Value
	// Block的hash缓存上一次Header计算出的哈希值，避免不必要的计算
	size atomic.Value

	// Td is used by package core to store the total difficulty
	// of the chain up to and including the block.
	// 挖矿总难度
	td *big.Int

	// These fields are used by package eth to track
	// inter-peer block relay.
	ReceivedAt   time.Time
	ReceivedFrom interface{}
}

// Header represents a block header in the Ethereum blockchain.
// 区块头
type Header struct {
	ParentHash  common.Hash    `json:"parentHash"       gencodec:"required"`
	UncleHash   common.Hash    `json:"sha3Uncles"       gencodec:"required"`
	Coinbase    common.Address `json:"miner"            gencodec:"required"`
	// Merkel根节点的哈希值
	Root        common.Hash    `json:"stateRoot"        gencodec:"required"`
	TxHash      common.Hash    `json:"transactionsRoot" gencodec:"required"`
	ReceiptHash common.Hash    `json:"receiptsRoot"     gencodec:"required"`
	// 过滤器，快速判断一个log对象是否在一组已知的log集合中
	Bloom       Bloom          `json:"logsBloom"        gencodec:"required"`
	Difficulty  *big.Int       `json:"difficulty"       gencodec:"required"`
	Number      *big.Int       `json:"number"           gencodec:"required"`
	GasLimit    *big.Int       `json:"gasLimit"         gencodec:"required"`
	GasUsed     *big.Int       `json:"gasUsed"          gencodec:"required"`
	Time        *big.Int       `json:"timestamp"        gencodec:"required"`
	Extra       []byte         `json:"extraData"        gencodec:"required"`
	// 以太坊共识算法ethash与比特币共识POW所不同的一个关键变量
	// mixDigest 是在挖矿过程中计算出来，它的作用就是作为矿工在进行消耗内存进行挖矿的证明
	MixDigest   common.Hash    `json:"mixHash"          gencodec:"required"`
	Nonce       BlockNonce     `json:"nonce"            gencodec:"required"`
}

// Body is a simple (mutable, non-safe) data container for storing and moving
// a block's data contents (transactions and uncles) together.
// 存储以太坊区块链的交易信息
type Body struct {
	Transactions []*Transaction
	// uncle设计的目的就是为了抵消整个以太坊网络中能力太强的节点对
	// 区块的产生影响太大。防止这些节点破坏区块链的去中心化原则
	Uncles       []*Header
}

// Hash returns the keccak256 hash of b's header.
// The hash is computed on the first call and cached thereafter.
// 以太坊区块唯一标识生成函数
func (b *Block) Hash() common.Hash {
	if hash := b.hash.Load(); hash != nil {
		return hash.(common.Hash)
	}
	v := b.header.Hash() // 调用Header的hash
	b.hash.Store(v)
	return v
}

// Hash returns the block hash of the header, which is simply the keccak256 hash of its
// RLP encoding.
// 生成区块头的哈希
func (h *Header) Hash() common.Hash {
	return rlpHash(h)
}

// 区块头的RLP hash值，rlp是一种编码规则
func rlpHash(x interface{}) (h common.Hash) {
	hw := sha3.NewKeccak256()
	rlp.Encode(hw, x)
	hw.Sum(h[:0])
	return h
}
```

## 新建区块
core/types/block.go
```go
// NewBlock creates a new block. The input data is copied,
// changes to header and to the field values will not affect the
// block.
//
// The values of TxHash, UncleHash, ReceiptHash and Bloom in header
// are ignored and set to values derived from the given txs, uncles
// and receipts.
// 新建区块
func NewBlock(header *Header, txs []*Transaction, uncles []*Header, receipts []*Receipt) *Block {
	b := &Block{header: CopyHeader(header), td: new(big.Int)}

	// TODO: panic if len(txs) != len(receipts)
	if len(txs) == 0 {
		b.header.TxHash = EmptyRootHash
	} else {
		b.header.TxHash = DeriveSha(Transactions(txs))
		b.transactions = make(Transactions, len(txs))
		copy(b.transactions, txs)
	}

	if len(receipts) == 0 {
		b.header.ReceiptHash = EmptyRootHash
	} else {
		b.header.ReceiptHash = DeriveSha(Receipts(receipts))
		b.header.Bloom = CreateBloom(receipts)
	}

	if len(uncles) == 0 {
		b.header.UncleHash = EmptyUncleHash
	} else {
		b.header.UncleHash = CalcUncleHash(uncles)
		b.uncles = make([]*Header, len(uncles))
		for i := range uncles {
			b.uncles[i] = CopyHeader(uncles[i])
		}
	}

	return b
}
```

## 区块链
blockchain.go:主要是对区块链状态进行维护，包括区块的验证、插入以及状态查询

```go
// NewBlockChain returns a fully initialised block chain using information
// available in the database. It initialises the default Ethereum Validator and
// Processor.
// 新建区块链
func NewBlockChain(chainDb ethdb.Database, config *params.ChainConfig, engine consensus.Engine, vmConfig vm.Config) (*BlockChain, error) {
	bodyCache, _ := lru.New(bodyCacheLimit)
	bodyRLPCache, _ := lru.New(bodyCacheLimit)
	blockCache, _ := lru.New(blockCacheLimit)
	futureBlocks, _ := lru.New(maxFutureBlocks)
	badBlocks, _ := lru.New(badBlockLimit)

	bc := &BlockChain{
		config:       config,
		chainDb:      chainDb,
		stateCache:   state.NewDatabase(chainDb),
		quit:         make(chan struct{}),
		bodyCache:    bodyCache,
		bodyRLPCache: bodyRLPCache,
		blockCache:   blockCache,
		futureBlocks: futureBlocks,
		engine:       engine,
		vmConfig:     vmConfig,
		badBlocks:    badBlocks,
	}
	bc.SetValidator(NewBlockValidator(config, bc, engine))
	bc.SetProcessor(NewStateProcessor(config, bc, engine))

	var err error
	bc.hc, err = NewHeaderChain(chainDb, config, engine, bc.getProcInterrupt)
	if err != nil {
		return nil, err
	}
	// 获取创世区块
	bc.genesisBlock = bc.GetBlockByNumber(0)
	if bc.genesisBlock == nil {
		return nil, ErrNoGenesis
	}
	// 获取最新状态
	if err := bc.loadLastState(); err != nil {
		return nil, err
	}
	// Check the current state of the block hashes and make sure that we do not have any of the bad blocks in our chain
	// 检测是否有坏区块
	for hash := range BadHashes {
		if header := bc.GetHeaderByHash(hash); header != nil {
			// get the canonical block corresponding to the offending header's number
			headerByNumber := bc.GetHeaderByNumber(header.Number.Uint64())
			// make sure the headerByNumber (if present) is in our current canonical chain
			if headerByNumber != nil && headerByNumber.Hash() == header.Hash() {
				log.Error("Found bad hash, rewinding chain", "number", header.Number, "hash", header.ParentHash)
				bc.SetHead(header.Number.Uint64() - 1)
				log.Error("Chain rewind was successful, resuming normal operation")
			}
		}
	}
	// Take ownership of this particular state
	go bc.update()
	return bc, nil
}
```
## 创世区块生成分析
## 生成创世区块之后，整个区块链配置完成
# 挖矿
## 以太坊中的共识-ethash
1. eth 和 bitcoin 一样，采用的都是基于工作量证明的 POW 共识来产生新的区块，与 bitcoin 不同的是 eth 采用的是可以抵御 ASIC 对挖矿工作的垄断，就是 ethash。
2. ASIC：专门为某种特点的用途设计的电子电路芯片，在比特币中就叫矿机芯片。
3. 与 CPU、GPU 想比，ASIC 算力能够高出数万倍。
4. 在比特币中，单从挖矿来说，已经不是一个去中心化思想的区块链了。
5. ethash：专门针对比特币中算力集中的情况所设计的一种算法。
6. ethash 在 bitcoin 的工作量证明的基础上增加了内存消耗的步骤。
7. 在 ethash 中，除了进行比特币中的工作量证明，还需要进行添加了历史交易数据的混淆运算。矿工节点在运算该算法的时候还需要去访问内存的历史交易信息(内存消耗的来源)。
8. 在以太坊中，会有一个以特定算法生成的大小 1GB 左右的数据集合，矿工节点需要在挖矿时需要把这 1GB 的数据全部加载到内存中去。
# 共识
概念：区块链中各个节点对下一个区块的内容形成一致的认识。
源码：
- 测试网络采用用 clique
- 公网采用 ethash

ethash解析

# 数据持久化

1. 目录结构

 	1. database.go:封装了对 levelDB 的操作代码
 	2. interface.go:数据库接口
 	3. memory_database.go:提供一个测试使用的内存数据库
 	4. database_test.go:测试案例
  2. levelDB
     1. google 开发开源 k-v 存储源码数据库
     2. 源码: https://github.com/syndtr/goleveldb
     3. 特点
        1. leveldb 是一个持久化存储的 k-v 系统，与 redis 相比， leveldb 是将大部分数据存储在磁盘中。而 redis 是一个内存型的 k-v 存储系统，会吃内存。
        2. leveldb 在存储数据时，是有序存储的，也就是相邻的 key 值在存储文件中按照顺序存储。
        3. 与其它 k-v 系统一样，levelDB 操作接口简单，基本操作也只包括增、删、改、查。也支持批量操作。
        4. leverDB 支持数据快照(snapshot)功能，可以使得读取操作不受到写操作的影响。
        5. levelDB 支持数据压缩，可以很好的减少存储空间，提高 IO 效率。
       4. 限制
            1. 非关系型数据库，不支持 sql 查询，不支持索引。
            2. 一次只允许一个进程访问一个特定的数据库。
  3. 源码详解
        1. interface.go
               1. 对 leveldb 的数据库操作的封装
               2. 单独处理时并发安全
               3. 批处理时不能并发操作
        2. database.go
               1. 新建 ldb 对象
               2. 对 interface 中接口函数的实现
                      1. 单条数据操作
                      2. 批量数据操作
               3. 对 eth 服务的监听以及数据统计
               4. 引用
                   			1. 初始化创世区块
                   			2. 从指定的区块链数据库中创建本地区块链
               5. Metircs
                      			1. 概念：系统性能度量框架，如果我们需要为某个系统或者服务做监控、统计等，就可以用到它。通常有 5 种类型。
                      			2. Meters：监控一系列事件发生的速率，在以太坊最大的作用就是监控 TPS。Meters 会统计最近 1min，5min，15min以及全部时间的速率。
                      			3. gauges：最简单的度量指标，统计瞬时状态，只有一个简单的返回值。
                      			4. Histograms：统计数据的分布情况。比如最小值，最大值和中间值，中位数。
                      			5. Times：和 meters 类似，它是 meters 和 histograms 结合，histograms  统计耗时，meter 统计 TPS。 
                      			6. counter：计数器。
        3. memory_database.go: 内存数据库，主要用于测试。
               1. 关于内存数据库的相关结构定义
                      1. 单条数据操作
               2. 批处理

# 以太坊 geth 命令操作
1. 概念：geth 是 go-ethereum 中最主要的一个命令行工具，也是各种网络的接入点，支持全节点和轻节点模式，其它程序也可以通过暴露的 JSON RPC 接口调用访问以太坊网络。
2. 主要引用第三方的 cli 包实现
	1. 源码地址：https://gopkg.in/urfave/cli.v1
	2. 概念: 一个基于 go 开发的用于再 go 里面构建命令行的应用程序
3. geth 启动流程分析
	1. 通过 init 做一个 geth 整体初始化
	2.  
		```go
		// geth 整体初始化
		func init() {
			// Initialize the CLI app and start Geth
			// 命令/行为，如果用户没有其它子命令，就调用这个字段指向的函数
			app.Action = geth
			app.HideVersion = true // we have a command to print the version
			app.Copyright = "Copyright 2013-2017 The go-ethereum Authors"
			// 所有支持的子命令
			app.Commands = []cli.Command{
				// See chaincmd.go:
				initCommand,
				...
			}
			// 通过 sort 函数为 cli 所有子命令基于首字母进行排序
			sort.Sort(cli.CommandsByName(app.Commands))
		
			// 所有能够解析的 options
			app.Flags = append(app.Flags, nodeFlags...)
			...
		
			// 在所有命令执行之前调用的函数
			app.Before = func(ctx *cli.Context) error {
				// 设置用于当前程序的 CPU 上限
				runtime.GOMAXPROCS(runtime.NumCPU())
				if err := debug.Setup(ctx); err != nil {
					return err
				}
				// Start system runtime metrics collection
				// 启动专门监控协程收集正在运行的进程的各种指标， 3s 一次
				go metrics.CollectProcessMetrics(3 * time.Second)
		
				// 配置指定的网络(主网或者其它的测试网络)
				utils.SetupNetwork(ctx)
				return nil
			}
		
			app.After = func(ctx *cli.Context) error {
				debug.Exit()
				// 充值终端
				console.Stdin.Close() // Resets terminal mode.
				return nil
			}
		}
		```

4. 通过 geth 函数默认启动(在没有调用其它的子命令的情况下默认启动 geth)
	1. 	
		```go
		// 如果没有指定其它子命令，那么 geth 就是默认的系统入口
		// 主要是根据提供的参数创建一个默认的节点
		// 以阻塞的方式来运行节点，直到节点被种植
		func geth(ctx *cli.Context) error {
			// 创建一个节点
			node := makeFullNode(ctx)
			// 启动节点
			startNode(ctx, node)
			// 等待，知道节点停止
			node.Wait()
			return nil
		}
		```
5. 启动节点
6. 等待节点终止
7. 总结: 整个启动过程其实就是在不断解析参数，然后创建启动节点，再把服务注入到节点中。所有与以太坊相关的功能都是以服务的形式存在
# RLP 源码解析

1. 概念：RLP(Recursive Length Prefix--递归长度前缀): 是一个编码算法
2. 功能：主要用于编码任意嵌套结构的二进制数据。是以太坊中序列和反序列化的主要方法，所有的区块、交易等数据结构都会经过 RLP 编码之后再存储到区块链数据库中
3. 数据处理特性
   1. RLP 处理两类数据
      1. 字符串(一串二进制数据)
      2. 列表(不单是一个列表，可以是一个嵌套递归的结构，里面还可以包含字符串，列表)

4. RLP 编码规则
   1. 对于单个字节，如果其值范围是**(0x00,0x7f]**，它的 RLP 编码是其本身
   2. 如果不是单个字节，一个字符串的长度是 0 ~ 55 字节，它的 RLP 编码包含一个单字节的前缀，后面跟着字符串本身，这个前缀的值是 0x80 加上字符串的长度，由于被编码的字符串最大长度是  55 = 0x37 ，因此单字节的前缀最大值 0x80+0x37=0xb7，即编码的第一个字节取值范围是**(0x80,0xb7]**
   3. 如果字符串长度大于 55 个字节，它的 RLP 编码包含一个单字节的前缀，**然后后面跟着字符串的长度**，再后面跟着字符串本身。这个前缀的值是 0xb7 加上字符串的长度的**二进制形式**的字节长度
   4. 如果一个列表的总长度(列表总长度是它包含的项的数量加它包含的各项的长度之和)是 0 ~ 55 字节，它的 RLP 编码包含一个单字节的前缀，后面跟着列表中各项元素的 RLP 编码，这个前缀的值是 0xc0 加上列表的总长度，编码的第一个字节的取值范围是 **[0xc0,0xf7]**
   5. 如果一个列表的总长度大于 55 个字节，它的 RLP 编码包含一个单字节的前缀，后面跟着列表的总长度，再后面跟着列表的各项元素的 RLP 编码，这个前缀的值是 0xf7 加上**列表总长度二进制形式的字节长度**，编码的第一个字节范围是 **(0xf8,0xff]**

5. 编码实例

   1. 规则 1: "d" = "d"
   2. 规则 2: "dog" = [0x83, 'd', 'o', 'g']
   3. 规则 3: 如果一个字符串长度 1024，它的二进制就是 1000000000，该二进制长度为两个字节(一个字节 8 位)，则该字符串前缀应该是 0xb9，字符串长度 1024 = 0x400。编码的第一个字节范围是 [0xb8,0xbf]。
      1. [0xb9,0x04,0x00]

   4. 规则 4
      1. 空列表: [] = [0xc0]
      2. ["cat", "dog"] = [0xc8, 0x83, 'c', 'a', 't', '0x83', 'd', 'o', 'g']

   5. 规则 5: 以列表总长度为 1024 为例，它的二进制就是 1000000000，该二进制长度为两个字节(一个字节 8 位)，则该字符串前缀应该是 0xf9，列表总长度 0x0400，再跟上各项元素的总长度编码
      1. [...] = [0xf9, 0x04, 0x00, ...]

6. RLP 解码规则

   1. 根据 RLP 编码规则和过程，RLP 解码的输入一律视为二进制字符数组，其过程如下：
      1. 根据输入首字节数据，解码数据类型、实际数据长度和位置；
      2. 根据类型和实际数据，解码不同类型的数据；
      3. 继续解码剩余的数据。

   2. 其中，解码数据类型、实际数据和位置的规则如下：
      1. 如果首字节(prefix)的值在[0x00, 0xf7]范围之间，那么该数据是字符串，且字符串就是首字节本身；
      2. 如果首字节的值在[0x80, 0xb7]范围之间，那么该数据是字符串，且字符串的长度等于首字节减去 0x80，且字符串位于首字节之后；(比如首字节占 0x87，那么长度就是 0x87 - 0x80 = 7)
      3. 如果首字节的值在[0xb8, 0xbf]范围之间，那么该数据是字符串，该字符串长度大于 55，且字符串的长度的**字节长度**等于首字节减去 0xb7，数据的长度位于首字节之后，且字符串位于数据的长度之后；
      4. 如果首字节的值在[0xc0, 0xf7]范围之间，那么该数据是列表，在这种情况下，需要对列表各项的数据进行递归解码。列表的总长度(列表各项编码后的长度之和)等于首字节减去 0xc0，且列表各项位于首字节之后；
      5. 如果首字节的值在[0xf8, 0xff]范围之间，那么该数据是列表，总长度大于 55，列表的总长度的字节长度等于首字节减去 0xf7，列表的总长度位于首字节之后，且列表各项位于列表的总长度之后。

7. 总结:
   1. RLP 编码主要和字符串或者列表的长度有关，在编码的过程中，采用相对应编码规则递推的方式进行
   2. 与其它的序列化方式相比，RLP 编码有点在于灵活使用长度前缀来表示数据的实际长度，并且使用递归的方式可以编码相当的数据
   3. 在接收到经过 RLP 编码的数据之后，根据第一个字节就可以推断出数据类型，长度，数据本身等信息。而其它的序列化方式，不能根据第一个字节获取这么多的信息

8. 目录结构

   ```
   decode.go			解码器，把 RLP 数据解码成 go 的数据结构
   decode_tail_test.go/decode_test.go	解码器测试代码
   encode.go			编码器，把 go 的数据结构转换为 RLP 的编码
   encode_test.go/encode_example_test.go	编码器的测试
   raw.go				原始的 RLP 数据
   raw_test.go			测试文件
   typecache.go		类型缓存，记录了类型->内容(编码器/解码器)
   ```

9. tpyecache.go: 根据给定的类型找到对应的编码器和解码器

   1. 在 C++ 或者 JAVA 等语言中，支持重载，可以通过不同的类型重载同一个函数名称来实现方法针对不同类型的实现，也可以通过泛型来实现函数的分派。

   2. ```c++
      string encode(int)
      string encode(log)
      string encode(struct test*)
      ```

   3. go 语言本身不支持重载，也没泛型，所以需要自己来实现函数的分派，typecache.go 就是通过自身的类型快速找到对应的编码器和解码器的函数。

   4. 总结
      1. 该文件定义了类型->编码器/解码器函数的核心数据结构
      2. 定义了编码器和解码器的函数
      3. 通过对应类型查找对应的编码器和解码器
      4. 通过给定的类型生成对应的编码器和解码器

10. encoder.go: 编码器函数，把数据结构转换为 RLP 编码
    1. 总结
       1. 定义编码器接口
       2. RLP 编码函数
       3. RLP 数据组装

11. decoder.go: 解码器函数，把 RLP 编码转换为对应的 golang 数据结构
    1. 总结
       1. 定义解码器接口
       2. RLP 解析函数

# 以太坊交易源码分析

1. 区块之间转账的基本概念以及流程
   1. 转账源地址，目标地址，金额
   2. 以太坊交易基本流程

<img src="pictures\以太坊交易基本流程.png" alt="以太坊交易基本流程" style="zoom:67%;" />

3. 完整流程
   1. 发起交易
   2. 交易签名
   3. 提交交易
   4. 广播交易: 通知 EVM 执行，同时把交易信息广播给其它节点

4. transaction.go
   1. 定义交易基本结构
   2. 新建交易

