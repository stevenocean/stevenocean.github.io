title: go-ethereum 源码分析 - ethdb(3)
tags:
  - Ethereum
  - 以太坊
  - 区块链
  - blockchain
  - LevelDB
categories:
  - Ethereum(以太坊)
  - blockchain(区块链)
author: Steven Hu
date: 2018-03-31 22:58:00
---

go-ethereum 中的所有区块数据都存储在 **leveldb** 中，并且 go-ethereum 又基于 leveldb 进行了一层简单的封装。

> [**leveldb**](https://zh.wikipedia.org/wiki/LevelDB) 是一个由 Google 开源（BSD）的 KV（Key/Value Pair）非关系型数据库，是基于 [**LSM(Log-Structured-Merge tree)**](http://www.benstopford.com/2015/02/14/log-structured-merge-trees/) 的典型实现。
> 
> 主要有如下几个[特性](https://github.com/google/leveldb)：
> 
> - Keys 和 Values 均为任意长度的字节数组；
> - Data(KV) 默认以 Key 字典序排序存储，也可以提供自定义的排序算法来重载排序；
> - 基本操作包括：Put(k,v)，Get(k)，Delete(k)；
> - 支持原子级的批量(Batch)操作；
> - 可以创建数据全景(transient)的 snapshot，并支持在 snapshot 中查找数据；
> - 支持前向和后向迭代遍历数据；
> - 数据自动采用 [**Snappy**](http://google.github.io/snappy/) 压缩算法进行压缩；
> - 可移植性；

源码目录如下，主要就下面 4 个源码文件：

```shell
➜  ethdb :
.
├── database.go        // leveldb 的封装代码
├── database_test.go   // 测试用例
├── interface.go       // 定义了 Database 的一些操作接口
└── memory_database.go // 供测试环境使用的基于内存的数据库
```

<!--more-->

我们主要分析下 **database.go** 和 **interface.go** 两个源码文件。

## database.go

这里主要封装了 leveldb 的接口，从如下 import 可以看出主要使用了 [**goleveldb**](https://github.com/syndtr/goleveldb/) 的封装。

```go
import (
  "strconv"
  "strings"
  "sync"
  "time"

  "github.com/ethereum/go-ethereum/log"
  "github.com/ethereum/go-ethereum/metrics"
  "github.com/syndtr/goleveldb/leveldb"
  "github.com/syndtr/goleveldb/leveldb/errors"
  "github.com/syndtr/goleveldb/leveldb/filter"
  "github.com/syndtr/goleveldb/leveldb/iterator"
  "github.com/syndtr/goleveldb/leveldb/opt"
  "github.com/syndtr/goleveldb/leveldb/util"
)
```

下面看下在 goleveldb 基础上进一步封装的 **LDBDatabase** 结构，如下，主要增加了很多维度的 Metrics 用于统计使用情况（几个 metrics 的注释很清晰）：

```go
type LDBDatabase struct {
  fn string      // filename for reporting
  db *leveldb.DB // LevelDB instance

  compTimeMeter  metrics.Meter // Meter for measuring the total time spent in database compaction
  compReadMeter  metrics.Meter // Meter for measuring the data read during compaction
  compWriteMeter metrics.Meter // Meter for measuring the data written during compaction
  diskReadMeter  metrics.Meter // Meter for measuring the effective amount of data read
  diskWriteMeter metrics.Meter // Meter for measuring the effective amount of data written

  quitLock sync.Mutex      // Mutex protecting the quit channel access
  quitChan chan chan error // Quit channel to stop the metrics collection before closing the database

  log log.Logger // Contextual logger tracking the database path
}
```

通过 NewLDBDatabase 方法来打开（或创建）并返回封装后的 LDBDatabase 实例，如下：

```go
// NewLDBDatabase returns a LevelDB wrapped object.
func NewLDBDatabase(file string, cache int, handles int) (*LDBDatabase, error) {
  // 跟踪对应 leveldb 文件的 logger
  logger := log.New("database", file)

  // Ensure we have some minimal caching and file guarantees
  if cache < 16 {
    cache = 16
  }
  if handles < 16 {
    handles = 16
  }
  logger.Info("Allocated cache and file handles", "cache", cache, "handles", handles)

  // 打开 leveldb 文件，并恢复潜在的冲突
  db, err := leveldb.OpenFile(file, &opt.Options{
    OpenFilesCacheCapacity: handles,
    BlockCacheCapacity:     cache / 2 * opt.MiB,
    WriteBuffer:            cache / 4 * opt.MiB, // Two of these are used internally
    Filter:                 filter.NewBloomFilter(10),
  })
  if _, corrupted := err.(*errors.ErrCorrupted); corrupted {
    // 如果有冲突，则需要修复文件
    db, err = leveldb.RecoverFile(file, nil)
  }

  // (Re)check for errors and abort if opening of the db failed
  if err != nil {
    return nil, err
  }

  // 返回 LDBDatabase 实例
  return &LDBDatabase{
    fn:  file,
    db:  db,
    log: logger,
  }, nil
}
```

此外，提供了一些基本操作方法，如 Put，Path，Get，Delete 等等用于操作 leveldb，这些基本操作都是直接调用 leveldb 的封装，如下为 Put 操作：

```go
// Put puts the given key / value to the queue
func (db *LDBDatabase) Put(key []byte, value []byte) error {
  // Generate the data to write to disk, update the meter and write
  //value = rle.Compress(value)

  return db.db.Put(key, value, nil)
}
```

### 创建本地 blockchain 数据库

下面看何时以及如何创建本地区块链DB：

```go
// source: eth/backend.go
// 
// New creates a new Ethereum object (including the
// initialisation of the common Ethereum object)
func New(ctx *node.ServiceContext, config *Config) (*Ethereum, error) {
  if config.SyncMode == downloader.LightSync {
    return nil, errors.New("can't run eth.Ethereum in light sync mode, use les.LightEthereum")
  }
  if !config.SyncMode.IsValid() {
    return nil, fmt.Errorf("invalid sync mode %d", config.SyncMode)
  }

  // 创建本地 blockchain 数据库
  // 返回的 chainDb
  chainDb, err := CreateDB(ctx, config, "chaindata")
  if err != nil {
    return nil, err
  }
  stopDbUpgrade := upgradeDeduplicateData(chainDb)

  // 设置创世区块，并写入 chainDb 中
  chainConfig, genesisHash, genesisErr := core.SetupGenesisBlock(chainDb, config.Genesis)
  if _, ok := genesisErr.(*params.ConfigCompatError); genesisErr != nil && !ok {
    return nil, genesisErr
  }
  log.Info("Initialised chain configuration", "config", chainConfig)

  eth := &Ethereum{
    config:         config,
    chainDb:        chainDb,
    chainConfig:    chainConfig,
    eventMux:       ctx.EventMux,
    accountManager: ctx.AccountManager,
    engine:         CreateConsensusEngine(ctx, &config.Ethash, chainConfig, chainDb),
    shutdownChan:   make(chan bool),
    stopDbUpgrade:  stopDbUpgrade,
    networkId:      config.NetworkId,
    gasPrice:       config.GasPrice,
    etherbase:      config.Etherbase,
    bloomRequests:  make(chan chan *bloombits.Retrieval),
    bloomIndexer:   NewBloomIndexer(chainDb, params.BloomBitsBlocks),
  }

  // ...
}
```

可以看到在创建和初始化 Ethereum 对象的时候会同时通过调用 **CreateDB** 方法来创建本地区块链数据库，并将 ethdb.LDBDatabase 实例 **chainDb** 作为 eth 实例的成员，在后续区块数据写入本地库时，都是通过该 chainDb 实例来操作写入的，如下代码：

```go
// source: eth/backend.go
// 
// CreateDB creates the chain database.
func CreateDB(ctx *node.ServiceContext, config *Config, name string) (ethdb.Database, error) {
  // 打开 name 数据库
  db, err := ctx.OpenDatabase(name, config.DatabaseCache, config.DatabaseHandles)
  if err != nil {
    return nil, err
  }
  if db, ok := db.(*ethdb.LDBDatabase); ok {
    db.Meter("eth/db/chaindata/")
  }
  return db, nil
}
```

这里打开 "chaindata" 数据库，并且启动 Metric（见下面），在 ctx.OpenDatabase 中创建 LDBDatabase，如下：

```go
// source: node/node.go
// 
// OpenDatabase opens an existing database with the given name (or creates one if no
// previous can be found) from within the node's instance directory. If the node is
// ephemeral, a memory database is returned.
func (n *Node) OpenDatabase(name string, cache, handles int) (ethdb.Database, error) {
  if n.config.DataDir == "" {
    // 没有指定 DataDir，则启动为内存数据库
    return ethdb.NewMemDatabase()
  }
  return ethdb.NewLDBDatabase(n.config.resolvePath(name), cache, handles)
}
```

### 将 block 写入到 leveldb

在 core/database_util.go 中，封装了 WriteHeader, WriteBody, WriteBlock 等方法用于将区块 header, body, block 写入 leveldb 中。

```go
// source: core/database_util.go
// 
// WriteHeader serializes a block header into the database.
func WriteHeader(db ethdb.Putter, header *types.Header) error {
  data, err := rlp.EncodeToBytes(header)
  if err != nil {
    return err
  }
  hash := header.Hash().Bytes()
  num := header.Number.Uint64()
  encNum := encodeBlockNumber(num)
  key := append(blockHashPrefix, hash...)
  if err := db.Put(key, encNum); err != nil {
    log.Crit("Failed to store hash to number mapping", "err", err)
  }
  key = append(append(headerPrefix, encNum...), hash...)
  if err := db.Put(key, data); err != nil {
    log.Crit("Failed to store header", "err", err)
  }
  return nil
}

// WriteBody serializes the body of a block into the database.
func WriteBody(db ethdb.Putter, hash common.Hash, number uint64, body *types.Body) error {
  data, err := rlp.EncodeToBytes(body)
  if err != nil {
    return err
  }
  return WriteBodyRLP(db, hash, number, data)
}

// WriteBodyRLP writes a serialized body of a block into the database.
func WriteBodyRLP(db ethdb.Putter, hash common.Hash, number uint64, rlp rlp.RawValue) error {
  key := append(append(bodyPrefix, encodeBlockNumber(number)...), hash.Bytes()...)
  if err := db.Put(key, rlp); err != nil {
    log.Crit("Failed to store block body", "err", err)
  }
  return nil
}

// WriteTd serializes the total difficulty of a block into the database.
func WriteTd(db ethdb.Putter, hash common.Hash, number uint64, td *big.Int) error {
  data, err := rlp.EncodeToBytes(td)
  if err != nil {
    return err
  }
  key := append(append(append(headerPrefix, encodeBlockNumber(number)...), hash.Bytes()...), tdSuffix...)
  if err := db.Put(key, data); err != nil {
    log.Crit("Failed to store block total difficulty", "err", err)
  }
  return nil
}

// WriteBlock serializes a block into the database, header and body separately.
func WriteBlock(db ethdb.Putter, block *types.Block) error {
  // Store the body first to retain database consistency
  if err := WriteBody(db, block.Hash(), block.NumberU64(), block.Body()); err != nil {
    return err
  }
  // Store the header too, signaling full block ownership
  if err := WriteHeader(db, block.Header()); err != nil {
    return err
  }
  return nil
}
```

其中都通过 db.Put 方法将区块信息写入 leveldb 中。

### Metrics

在 **Meter** 方法中创建了各种 **Metrics** 采集器，然后创建了 quitChan，最后启动一个协程调用了 db.meter 方法每隔 3 秒收集一次。

```go
// Meter configures the database metrics collectors and
func (db *LDBDatabase) Meter(prefix string) {
  // Short circuit metering if the metrics system is disabled
  if !metrics.Enabled {
    return
  }
  // Initialize all the metrics collector at the requested prefix
  db.compTimeMeter = metrics.NewRegisteredMeter(prefix+"compact/time", nil)
  db.compReadMeter = metrics.NewRegisteredMeter(prefix+"compact/input", nil)
  db.compWriteMeter = metrics.NewRegisteredMeter(prefix+"compact/output", nil)
  db.diskReadMeter = metrics.NewRegisteredMeter(prefix+"disk/read", nil)
  db.diskWriteMeter = metrics.NewRegisteredMeter(prefix+"disk/write", nil)

  // Create a quit channel for the periodic collector and run it
  db.quitLock.Lock()
  db.quitChan = make(chan chan error)
  db.quitLock.Unlock()

  go db.meter(3 * time.Second)
}
```

**metrics.Enabled** 开关默认为关闭(false)，通过命令行选项参数 **metrics** 来设置。

```go
// go-ethereum/metrics.go

// Enabled is checked by the constructor functions for all of the
// standard metrics.  If it is true, the metric returned is a stub.
//
// This global kill-switch helps quantify the observer effect and makes
// for less cluttered pprof profiles.
var Enabled bool = false

// MetricsEnabledFlag is the CLI flag name to use to enable metrics collections.
const MetricsEnabledFlag = "metrics"
const DashboardEnabledFlag = "dashboard"

// Init enables or disables the metrics system. Since we need this to run before
// any other code gets to create meters and timers, we'll actually do an ugly hack
// and peek into the command line args for the metrics flag.
func init() {
  for _, arg := range os.Args {
    if flag := strings.TrimLeft(arg, "-"); flag == MetricsEnabledFlag || flag == DashboardEnabledFlag {
      log.Info("Enabling metrics collection")
      Enabled = true
    }
  }
}
```

在启动 metrics 开关后，每隔 3 秒会从 leveldb 内部获取计数器，然后公布到 metrics 子系统，这里是一个无限循环，直到 quitChan 收到一个退出信号。

```go
// meter periodically retrieves internal leveldb counters and reports them to
// the metrics subsystem.
//
// This is how a stats table look like (currently):
//   Compactions
//    Level |   Tables   |    Size(MB)   |    Time(sec)  |    Read(MB)   |   Write(MB)
//   -------+------------+---------------+---------------+---------------+---------------
//      0   |          0 |       0.00000 |       1.27969 |       0.00000 |      12.31098
//      1   |         85 |     109.27913 |      28.09293 |     213.92493 |     214.26294
//      2   |        523 |    1000.37159 |       7.26059 |      66.86342 |      66.77884
//      3   |        570 |    1113.18458 |       0.00000 |       0.00000 |       0.00000
//
// This is how the iostats look like (currently):
// Read(MB):3895.04860 Write(MB):3654.64712
func (db *LDBDatabase) meter(refresh time.Duration) {
  // Create the counters to store current and previous compaction values
  // 创建一个计数器，用于存储当前和上一次的压缩值
  // 一个二维数组，第 1 维表示 当前 和 上一次，第 2 维表示统计值
  compactions := make([][]float64, 2)
  for i := 0; i < 2; i++ {
    compactions[i] = make([]float64, 3)
  }
  // Create storage for iostats.
  var iostats [2]float64
  // Iterate ad infinitum and collect the stats
  for i := 1; ; i++ {
    // Retrieve the database stats
    stats, err := db.db.GetProperty("leveldb.stats")
    if err != nil {
      db.log.Error("Failed to read database stats", "err", err)
      return
    }
    // Find the compaction table, skip the header
    lines := strings.Split(stats, "\n")
    for len(lines) > 0 && strings.TrimSpace(lines[0]) != "Compactions" {
      lines = lines[1:]
    }
    if len(lines) <= 3 {
      db.log.Error("Compaction table not found")
      return
    }
    lines = lines[3:]

    // Iterate over all the table rows, and accumulate the entries
    for j := 0; j < len(compactions[i%2]); j++ {
      compactions[i%2][j] = 0
    }
    for _, line := range lines {
      parts := strings.Split(line, "|")
      if len(parts) != 6 {
        break
      }
      for idx, counter := range parts[3:] {
        value, err := strconv.ParseFloat(strings.TrimSpace(counter), 64)
        if err != nil {
          db.log.Error("Compaction entry parsing failed", "err", err)
          return
        }
        compactions[i%2][idx] += value
      }
    }
    // Update all the requested meters
    if db.compTimeMeter != nil {
      db.compTimeMeter.Mark(int64((compactions[i%2][0] - compactions[(i-1)%2][0]) * 1000 * 1000 * 1000))
    }
    if db.compReadMeter != nil {
      db.compReadMeter.Mark(int64((compactions[i%2][1] - compactions[(i-1)%2][1]) * 1024 * 1024))
    }
    if db.compWriteMeter != nil {
      db.compWriteMeter.Mark(int64((compactions[i%2][2] - compactions[(i-1)%2][2]) * 1024 * 1024))
    }

    // Retrieve the database iostats.
    ioStats, err := db.db.GetProperty("leveldb.iostats")
    if err != nil {
      db.log.Error("Failed to read database iostats", "err", err)
      return
    }
    parts := strings.Split(ioStats, " ")
    if len(parts) < 2 {
      db.log.Error("Bad syntax of ioStats", "ioStats", ioStats)
      return
    }
    r := strings.Split(parts[0], ":")
    if len(r) < 2 {
      db.log.Error("Bad syntax of read entry", "entry", parts[0])
      return
    }
    read, err := strconv.ParseFloat(r[1], 64)
    if err != nil {
      db.log.Error("Read entry parsing failed", "err", err)
      return
    }
    w := strings.Split(parts[1], ":")
    if len(w) < 2 {
      db.log.Error("Bad syntax of write entry", "entry", parts[1])
      return
    }
    write, err := strconv.ParseFloat(w[1], 64)
    if err != nil {
      db.log.Error("Write entry parsing failed", "err", err)
      return
    }
    if db.diskReadMeter != nil {
      db.diskReadMeter.Mark(int64((read - iostats[0]) * 1024 * 1024))
    }
    if db.diskWriteMeter != nil {
      db.diskWriteMeter.Mark(int64((write - iostats[1]) * 1024 * 1024))
    }
    iostats[0] = read
    iostats[1] = write

    // Sleep a bit, then repeat the stats collection
    select {
    case errc := <-db.quitChan:
      // Quit requesting, stop hammering the database
      errc <- nil
      return

    case <-time.After(refresh):
      // Timeout, gather a new set of stats
    }
  }
}
```
