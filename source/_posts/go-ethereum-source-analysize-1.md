title: go-ethereum 源码分析 - 区块结构(1)
tags:
  - Ethereum
  - 以太坊
  - 区块链
  - blockchain
categories:
  - Ethereum(以太坊)
  - blockchain(区块链)
author: Steven Hu
date: 2018-03-17 23:58:00
---
今天抽时间浏览了 Ethereum 的区块结构的相关源码，更加深入地了解区块的数据结构。

> 分析的主要是 go-ethereum 项目的源码。版本取的是 Frost(v1.8.2)。

## 1. 以太坊的基本结构

先大致了解了以太坊的基本结构，如下图（来自网络）:

{% asset_img ethereum-overall-arch-1.jpg %}

其中除了区块链管理（Blockchain Management）和挖矿模块（Miner）之外还包含了：

- 基础的分布式数据库结构（Swarm）；
- 智能合约（Solidity、Serpent、LLL等），及执行环境 EVM；
- 共识机制（PoW-Ethash，PoS）；
- 账户管理；
- 加密算法模块（SHA-3、RLP等）；
- P2P 网络（Whisper）；
- 应用模块（DApp、浏览器钱包-Mist、桌面钱包-EtherWallet、JS框架-Web3js）；

<!--more-->

## 2. go-ethereum 源码目录结构

go-ethereum 的源码层次结构也还是很清晰的，如下为 go-ethereum 项目的源码目录结构：

```shell
➜  go-ethereum :
.
├── accounts        // 以太坊的钱包和账户管理
    ├── abi         // 以太坊合约的ABI代码
    ├── keystore    // 支持 keystore 模式的钱包
    └── usbwallet   // 支持 USB 模式的钱包
├── bmt             // 二进制的 Merkle 树的实现
├── build           // 编译与构建的一些脚本和配置
├── cmd             // 命令行工具集
    ├── abigen      // ABI 生成器
    ├── bootnode    // 启动一个仅仅实现网络发现的节点
    ├── ethkey
    ├── evm         // 以太坊虚拟机的开发工具
    ├── faucet
    ├── geth        // geth 命令行工具
    ├── p2psim      // 提供了一个工具来模拟 HTTP 的 API
    ├── puppeth     // 创建一个新的以太坊网络的向导
    ├── rlpdump     // RLP 数据的格式化输出
    ├── swarm       // swarm 网络的接入点
    ├── utils       // 公共工具
    └── wnode       // 一个简单的 whisper 节点，可以用作独立的引导节点。此外，可以用于不同的测试和诊断目的
├── common          // 提供了一些公共通用的工具类
├── compression     // 压缩
├── consensus       // 提供了以太坊的一些共识算法，比如：ethhash
├── console         // 控制台
├── containers      // 支持 docker 和 vagrant 等容器
├── contracts       // 合约管理
├── core            // 核心数据结构和算法（EVM，state，Blockchain，布隆过滤器等）
├── crypto          // 加密相关
├── dashboard       // 
├── eth             // 在其中实现了以太坊的协议
├── ethclient       // geth 客户端入口
├── ethdb           // eth 的数据库（包括生产环境的leveldb和供测试用的内存数据库）
├── ethstats        // 提供以太坊网络状态的报告
├── event           // 用于处理实时事件
├── les             // 以太坊的轻量级协议子集(Light Ethereum Subprotocol)
├── light           // 实现为以太坊轻量级客户端提供按需检索的功能
├── log             // 日志模块
├── metrics         // 度量和检测
├── miner           // 提供以太坊的区块创建和挖矿
├── mobile          // 移动端的一些 wrapper
├── node            // 以太坊的多种类型的节点
├── p2p             // P2P 网络协议
├── params          // 参数管理
├── rlp             // 以太坊的序列化处理（rlp 递归长度前缀编码）
├── rpc             // rpc 远程方法调用
├── swarm           // swarm 网络存储和处理
├── trie            // 以太坊中的重要数据结构，Merkle Patricia Tries
└── whisper         // whisper 节点协议
```

## 3. Block 区块结构

> 可以在 etherchain 上查看一些真实[区块示例](https://www.etherchain.org/block/041c3d9b9696b767938adc6fe3b0d31951ba4e167157cb5f6853c5b55f1526fa)

Block 区块的数据结构图：

{% asset_img blockchain-structure.png %}
.
从上图可以看到在区块链上的每个区块主要包括区块头（Header）和区块体（Body）两部分。

如下为 Block 结构定义：

```go
// source: core/types/block.go

// Block represents an entire block in the Ethereum blockchain.
type Block struct {
    header       *Header
    uncles       []*Header
    transactions Transactions

    // caches
    hash atomic.Value
    size atomic.Value

    // Td is used by package core to store the total difficulty
    // of the chain up to and including the block.
    td *big.Int

    // These fields are used by package eth to track
    // inter-peer block relay.
    ReceivedAt   time.Time
    ReceivedFrom interface{}
}
```

- **header**: 指向区块头的指针，类型为 **Header**，见下面；
- **uncles**: uncles 链表头，每一个都为 Header 结构；
- **transactions**: 交易列表，**Transactions** 定义为存储 **Transaction** 的指针切片（Slice）；
- **td**: total difficulty，总体复杂度；
- **ReceivedAt**: 接收块的时间；
- **ReceivedFrom**:

> **Transaction** 类型定义在 core/types/transaction.go 中。

这里定义没有看到 **Body**，其实 **Block** 提供了一个获取（动态生成）**Body** 的方法:

```go
// source: core/types/block.go

// Body returns the non-header content of the block.
func (b *Block) Body() *Body { return &Body{b.transactions, b.uncles} }
```

下面分别看下区块头和区块体的结构定义。

### 3.1 区块头（Header）

```go
// source: core/types/block.go

// Header represents a block header in the Ethereum blockchain.
type Header struct {
    ParentHash  common.Hash    `json:"parentHash"       gencodec:"required"`
    UncleHash   common.Hash    `json:"sha3Uncles"       gencodec:"required"`
    Coinbase    common.Address `json:"miner"            gencodec:"required"`
    Root        common.Hash    `json:"stateRoot"        gencodec:"required"`
    TxHash      common.Hash    `json:"transactionsRoot" gencodec:"required"`
    ReceiptHash common.Hash    `json:"receiptsRoot"     gencodec:"required"`
    Bloom       Bloom          `json:"logsBloom"        gencodec:"required"`
    Difficulty  *big.Int       `json:"difficulty"       gencodec:"required"`
    Number      *big.Int       `json:"number"           gencodec:"required"`
    GasLimit    uint64         `json:"gasLimit"         gencodec:"required"`
    GasUsed     uint64         `json:"gasUsed"          gencodec:"required"`
    Time        *big.Int       `json:"timestamp"        gencodec:"required"`
    Extra       []byte         `json:"extraData"        gencodec:"required"`
    MixDigest   common.Hash    `json:"mixHash"          gencodec:"required"`
    Nonce       BlockNonce     `json:"nonce"            gencodec:"required"`
}
```

- **ParentHash**: 对应前一个区块（父区块）的 Hash；
- **UncleHash**: 当前区块的 uncle 链的 Hash；
- **Coinbase**: 矿工地址；
- **Root**: Transaction Root Hash，交易字典树根结点hash，交易记录构成的 Merkle 树的根，在交易字典树上含有区块中的所有交易列表；
- **TxHash**: 交易 Hash
- **ReceiptHash**: 接收方的字典树根节点 hash；
- **Difficulty**: 当前区块工作量证明（PoW）算法的复杂度，表示当前区块的难度水平，这一个值根据前一个区块的难度水平和时间戳计算得到；
- **Number**: 区块体中的交易数目；
- **GasLimit**: Gas 限制，表示目前每个区块的 Gas 消耗上限；
- **GasUsed**: Gas 使用量，当前区块的所有交易使用的 Gas 之和；
- **Time**: 时间戳（区块初始化时的Unix时间戳）；
- **Extra**: 额外的信息，字节数组；
- **MixDigest**: 签名
- **Nonce**: PoW 中使用的随机数；

> 其中 common.Hash 类型定义在 common/types.go 中，定义为长度为 32 个字节的数组，存储的为 Keccak256 hash 值（SHA-3算法）。
> SHA-3 算法参考: https://zh.wikipedia.org/wiki/SHA-3

如下为一笔典型的区块头结构示例：

| 字段 | 取值 |
|-----|------|
| **Hash** | 0xb321e1b949169eb8a7465d3d9cbad991c2bb77f02aa76df85ebbee7a9b784013 |
| **Difficulty** | 2,944,102,190,231,396 |
| **Miner** | ✔ Nanopool (0x52bc...) (Mined in 33s) |
| **Reward** | 3.26744 ETH = $2,869.17 (Block Reward: 3 ETH + Fee Reward: 0.17369 ETH + Uncle Inclusion Reward: 0.09375 ETH) |
| **Tx Fees** | 0.17369 ETH = $152.52 (5.32% of the total block reward) |
| **Gas Limit** | 8,000,029 |
| **Gas Usage** | 99.7 % (7,978,750 of 8,000,029) |
| **Lowest Gas Price** | 1 GWei |
| **Time** | 2018-02-27T07:47:48+00:00 (2 minutes ago) |
| **Size** | 31,867 bytes |
| **Extra** | nanopool.org (Raw: 0x6e616e6f706f6f6c2e6f7267) |
| **UncleHash** | 0x5b43ac12d745836430feca6e7c7179d6f34e5eca5d9d5eb8a35bb6ce2e400676 |
| **Receipts Root Hash** | 0xf8c094df87434623bb21ed5828e99a71afb351a8ea06f4df27dfb3c06c7464d3 |
| **State Root Hash** | 0x41261f7694610dc68c8704ad026b15c009b139a0206f90b4366346d60f6d2b1b |
| **Transactions Root Hash** | 0xec1804b0217bfd6e6e696b1fbb8edad9ec00077f9c5cb013ff5b97791ff9d310 |
| **PoW Mix Hash** | 0x669bfda843d619df8befa18920fcc0964cd62e6f43d836dd056dd483f03f81a3 |
| **Bloom Data**| 0x0623e2... |

我们可以到 etherchain 上查询到该[区块信息](https://www.etherchain.org/block/5164216)，如下图:

{% asset_img block-header.jpg %}

### 3.2 区块体（Body）

```go
// source: core/types/block.go

// Body is a simple (mutable, non-safe) data container for storing and moving
// a block's data contents (transactions and uncles) together.
type Body struct {
    Transactions []*Transaction
    Uncles       []*Header
}
```

主要包括了 **交易记录列表** 和 **Uncles列表** 两部分数据。

### 3.3 创世区块

创世区块（genesis block）是以太坊网络中创建的第一个区块，是整个以太坊区块链的共同祖先，被静态写入地写入在所有以太坊客户端代码中，以确保创世区块不会被改变，可以在 etherscan.io 中查看创世区块的信息，如下图：

{% asset_img ethereum-genesis-block-1.jpg %}

对应于 mainnet 中的默认 genesis block 如下：

```go
// source: core/genesis.go
// 
// DefaultGenesisBlock returns the Ethereum main net genesis block.
func DefaultGenesisBlock() *Genesis {
    return &Genesis{
        Config:     params.MainnetChainConfig,
        Nonce:      66,
        ExtraData:  hexutil.MustDecode("0x11bbe8db4e347b4e8c937c1c8370e4b5ed33adb3db69cbdb7a38e1e50b1b82fa"),
        GasLimit:   5000,
        Difficulty: big.NewInt(17179869184),
        Alloc:      decodePrealloc(mainnetAllocData),
    }
}
```

基于以太坊创建私有链时可以通过配置指定创世区块的配置。

### 3.4 创建区块

其余的区块由矿工创建：

```go
// source: core/types/block.go
//
// NewBlock creates a new block. The input data is copied,
// changes to header and to the field values will not affect the
// block.
//
// The values of TxHash, UncleHash, ReceiptHash and Bloom in header
// are ignored and set to values derived from the given txs, uncles
// and receipts.
func NewBlock(header *Header, txs []*Transaction, uncles []*Header, receipts []*Receipt) *Block {
    b := &Block{header: CopyHeader(header), td: new(big.Int)}

    // TODO: panic if len(txs) != len(receipts)
    if len(txs) == 0 {
        // 无交易记录，生成 EmptyRootHash
        b.header.TxHash = EmptyRootHash
    } else {
        // 生成 Transaction Root Hash
        // Merkle Patricia Tree
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

该方法主要由 **miner** 中的 **worker** 在 pending 状态时调用此方法，如下:

```go
// source: miner/miner.go
//
// PendingBlock returns the currently pending block.
//
// Note, to access both the pending block and the pending state
// simultaneously, please use Pending(), as the pending state can
// change between multiple method calls
func (self *Miner) PendingBlock() *types.Block {
    return self.worker.pendingBlock()
}

// source: miner/worker.go
func (self *worker) pendingBlock() *types.Block {
    self.currentMu.Lock()
    defer self.currentMu.Unlock()

    if atomic.LoadInt32(&self.mining) == 0 {
        // 创建新区块
        return types.NewBlock(
            self.current.header,
            self.current.txs,
            nil,
            self.current.receipts,
        )
    }
    return self.current.Block
}
```

## 4. Blockchain

Blockchain 的主要功能是维护整个区块链的状态，包括区块的验证、插入、状态查询等。

```go
// 两个阶段校验(Validator)：区块的处理、状态的校验。
type BlockChain struct {
    chainConfig *params.ChainConfig // Chain & network configuration
    cacheConfig *CacheConfig        // Cache configuration for pruning

    // 底层数据库，用于将区块链数据持久化存储
    db     ethdb.Database // Low level persistent database to store final content in

    triegc *prque.Prque   // Priority queue mapping block numbers to tries to gc
    gcproc time.Duration  // Accumulates canonical block processing for trie dumping

    hc            *HeaderChain  // 只包含区块头的区块链（如在 light syncmode）

    // 下面几个是涉及消息通知的组件
    rmLogsFeed    event.Feed
    chainFeed     event.Feed
    chainSideFeed event.Feed
    chainHeadFeed event.Feed
    logsFeed      event.Feed
    scope         event.SubscriptionScope

    // 指向创世区块
    genesisBlock  *types.Block

    mu      sync.RWMutex // global mutex for locking chain operations
    chainmu sync.RWMutex // blockchain insertion lock
    procmu  sync.RWMutex // block processor lock

    checkpoint       int          // checkpoint counts towards the new checkpoint
    currentBlock     atomic.Value // Current head of the block chain
    currentFastBlock atomic.Value // Current head of the fast-sync chain (may be above the block chain!)

    stateCache   state.Database // State database to reuse between imports (contains state cache)
    bodyCache    *lru.Cache     // Cache for the most recent block bodies
    bodyRLPCache *lru.Cache     // Cache for the most recent block bodies in RLP encoded format
    blockCache   *lru.Cache     // Cache for the most recent entire blocks
    futureBlocks *lru.Cache     // future blocks are blocks added for later processing

    quit    chan struct{} // blockchain quit channel
    running int32         // running must be called atomically
    // procInterrupt must be atomically called
    procInterrupt int32          // interrupt signaler for block processing
    wg            sync.WaitGroup // chain processing wait group for shutting down

    // 共识算法的引擎
    engine    consensus.Engine

    // 当需要导入 blocks 时需要根据两个阶段的 Validator 的规则集进行校验
    // 分别为 block validator(区块验证器) 和 state validator(状态验证器)
    processor Processor // block processor interface
    validator Validator // block and state validator interface

    // VM 的配置参数
    vmConfig  vm.Config

    badBlocks *lru.Cache // Bad block cache
}
```

## 5. HeaderChain

上面的 Blockchain 是管理所有的 block，而 Headerchain 则是管理所有的 Header 形成一个单向链表。

```go
// source: core/headerchain.go

// HeaderChain implements the basic block header chain logic that is shared by
// core.BlockChain and light.LightChain. It is not usable in itself, only as
// a part of either structure.
// It is not thread safe either, the encapsulating chain structures should do
// the necessary mutex locking/unlocking.
type HeaderChain struct {
    config *params.ChainConfig

    chainDb       ethdb.Database  // 用于将区块信息写入本地的 leveldb 实例
    genesisHeader *types.Header   // 创世区块头

    // Current head of the header chain (may be above the block chain!)
    currentHeader     atomic.Value // 当前区块头

    // Hash of the current head of the header chain (prevent recomputing all the time)
    currentHeaderHash common.Hash // 当前区块头的 hash（避免重新计算）

    headerCache *lru.Cache // Cache for the most recent block headers
    tdCache     *lru.Cache // Cache for the most recent block total difficulties
    numberCache *lru.Cache // Cache for the most recent block numbers

    procInterrupt func() bool

    rand   *mrand.Rand
    engine consensus.Engine  // 共识算法引擎
}
```

## 6. Miner

下面我们主要通过源码跟踪一个 block 是如何被“挖”出来的。

如果需要自己的节点启动挖矿，可以在 geth 的 控制台下调用 miner.start() 方法来简单启动挖矿流程，创建一个新的 miner 实例，如下代码：

```go
func New(eth Backend, config *params.ChainConfig, mux *event.TypeMux, engine consensus.Engine) *Miner {
    miner := &Miner{
        eth:      eth,
        mux:      mux,
        engine:   engine,
        worker:   newWorker(config, engine, common.Address{}, eth, mux),
        canStart: 1,
    }
    miner.Register(NewCpuAgent(eth.BlockChain(), engine))
    go miner.update()

    return miner
}
```

这里主要执行了如下三个步骤：

1) 首先初始化一个 **Miner** 实例 **miner**，传入了 eth、engine、以及一个 worker 实例；



2) 将 CpuAgent 实例注册进 **miner** 中，表示后续的挖矿主要为消耗 CPU 类型的 agent；
3) 启动 miner.update() 协程；



