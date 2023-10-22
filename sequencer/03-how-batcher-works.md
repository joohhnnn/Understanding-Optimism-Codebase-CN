# batcher工作原理
在这一章节中，我们将探讨到底什么是`batcher` ⚙️
官方specs中有batcher的介绍([source](https://github.com/ethereum-optimism/optimism/blob/develop/specs/batcher.md))

在进行之前，我们先提出几个问题，通过这两个问题来真正理解`batcher`的作用以及工作原理
- `batcher`是什么？它为什么叫做`batcher`
- `batcher`在代码中到底是怎么运行的？

## 前置知识
- 在rollup机制中，要想做到的去中心化特性，例如抗审查等。我们必须要把layer2上发生的数据（transactions）全部发送到layer1当中。这样就可以在利用layer1的安全性的同时，又可以完全从layer1中构建出来整个layer2的数据，使得layer2才真正的具有有效性。
- [Epochs and the Sequencing Window](https://github.com/ethereum-optimism/optimism/blob/develop/specs/overview.md#epochs-and-the-sequencing-window):`Epoch`可以简单理解为L1新的一个`区块（N+1）`生成的这段时间。`epoch`的编号等于L1`区块N`的编号，在L1区块`N -> N+1` 这段时间内产生的所有L2区块都属于`epoch N`。在上个概念中我们提到必须上传L2的数据到L1中，那么我们应该在什么范围内上传数据才是有效的呢，`Sequencing Window`的size给了我们答案，即区块N/epoch N的相关数据，必须在L1的第`N + size`之前已经上传到L1了。
- Batch/Batcher Transaction: `Batch`可以简单理解为每一个L2区块构建所需要的交易。`Batcher Transaction`为多个batch组合起来经过加工后发送到L1的那笔交易
- [Channe](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#channel): `channel`可以简单理解为是`batch`的组合，组合是为了获得更好的压缩率，从而降低数据可用性成本，以使`batcher`上传的成本进一步降低。
- [Frame](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#channel-frame): `frame`可以理解为，有时候为了更好的压缩率，可能会导致`channel`数据过大而不能直接被`batcher`将整个`channel`发送给L1，因此需要对`channel`进行切割，分多次进行发送。

## 什么是batcher
在rollup中，需要一个角色来传递L2信息到L1当中，同时每当有新的交易就马上发送是昂贵且不方便管理的。这时候我们将需要制定一种合理的批量上传策略。因此，为了解决这个问题，batcher出现了。batcher是唯一存在（sequencer当前掌管私钥），且和特定地址发送`Batcher Transaction`来传递L2信息的组件。

batcher通过对unsafe区块数据进行收集，来获取多个batch，在这里每个区块都对应一个batch。当收集足够的batch进行高效压缩后生成channel，并以frame的形式发送到L1来完成L2的信息上传。

## 代码实现
在这部分我们会从代码层来进行深度的机制和实现原理的讲解

### 程序起点 
`op-batcher/batcher/driver.go`

通过调用`Start`函数来启动`loop`循环，在`loop`的循环中，主要处理三件事
- 当定时器触发时，将所有新的还未加载的`L2block`加载进来，然后触发`publishStateToL1`函数向L1进行`state`发布
- 处理`receipts`，记录成功或者失败状态
- 处理关闭请求

```go
    func (l *BatchSubmitter) Start() error {
        l.log.Info("Starting Batch Submitter")

        l.mutex.Lock()
        defer l.mutex.Unlock()

        if l.running {
            return errors.New("batcher is already running")
        }
        l.running = true

        l.shutdownCtx, l.cancelShutdownCtx = context.WithCancel(context.Background())
        l.killCtx, l.cancelKillCtx = context.WithCancel(context.Background())
        l.state.Clear()
        l.lastStoredBlock = eth.BlockID{}

        l.wg.Add(1)
        go l.loop()

        l.log.Info("Batch Submitter started")

        return nil
    }
```

```go
    func (l *BatchSubmitter) loop() {
        defer l.wg.Done()

        ticker := time.NewTicker(l.PollInterval)
        defer ticker.Stop()

        receiptsCh := make(chan txmgr.TxReceipt[txData])
        queue := txmgr.NewQueue[txData](l.killCtx, l.txMgr, l.MaxPendingTransactions)

        for {
            select {
            case <-ticker.C:
                if err := l.loadBlocksIntoState(l.shutdownCtx); errors.Is(err, ErrReorg) {
                    err := l.state.Close()
                    if err != nil {
                        l.log.Error("error closing the channel manager to handle a L2 reorg", "err", err)
                    }
                    l.publishStateToL1(queue, receiptsCh, true)
                    l.state.Clear()
                    continue
                }
                l.publishStateToL1(queue, receiptsCh, false)
            case r := <-receiptsCh:
                l.handleReceipt(r)
            case <-l.shutdownCtx.Done():
                err := l.state.Close()
                if err != nil {
                    l.log.Error("error closing the channel manager", "err", err)
                }
                l.publishStateToL1(queue, receiptsCh, true)
                return
            }
        }
    }
```

### 加载最新区块数据
`op-batcher/batcher/driver.go`

`loadBlocksIntoState`函数调用`calculateL2BlockRangeToStore`来获取自上次发送`batch transaction`而派生的最新`safeblock`后新生成的`unsafeblock`范围。然后循环将这个范围中的每一个`unsafe`块调用`loadBlockIntoState`函数从L2里获取并通过`AddL2Block`函数加载到内部的`block队列`里。等待进一步处理。

```go
    func (l *BatchSubmitter) loadBlocksIntoState(ctx context.Context) error {
        start, end, err := l.calculateL2BlockRangeToStore(ctx)
        ……
        var latestBlock *types.Block
        // Add all blocks to "state"
        for i := start.Number + 1; i < end.Number+1; i++ {
            block, err := l.loadBlockIntoState(ctx, i)
            if errors.Is(err, ErrReorg) {
                l.log.Warn("Found L2 reorg", "block_number", i)
                l.lastStoredBlock = eth.BlockID{}
                return err
            } else if err != nil {
                l.log.Warn("failed to load block into state", "err", err)
                return err
            }
            l.lastStoredBlock = eth.ToBlockID(block)
            latestBlock = block
        }
        ……
    }
```

```go
    func (l *BatchSubmitter) loadBlockIntoState(ctx context.Context, blockNumber uint64) (*types.Block, error) {
        ……
        block, err := l.L2Client.BlockByNumber(ctx, new(big.Int).SetUint64(blockNumber))
        ……
        if err := l.state.AddL2Block(block); err != nil {
            return nil, fmt.Errorf("adding L2 block to state: %w", err)
        }
        ……
        return block, nil
    }
```
### 将加载的block数据处理，并发送到layer1
`op-batcher/batcher/driver.go`

`publishTxToL1`函数使用`TxData`函数对之前加载到数据进行处理，并调用`sendTransaction`函数发送到L1

```go
    func (l *BatchSubmitter) publishTxToL1(ctx context.Context, queue *txmgr.Queue[txData], receiptsCh chan txmgr.TxReceipt[txData]) error {
        // send all available transactions
        l1tip, err := l.l1Tip(ctx)
        if err != nil {
            l.log.Error("Failed to query L1 tip", "error", err)
            return err
        }
        l.recordL1Tip(l1tip)

        // Collect next transaction data
        txdata, err := l.state.TxData(l1tip.ID())
        if err == io.EOF {
            l.log.Trace("no transaction data available")
            return err
        } else if err != nil {
            l.log.Error("unable to get tx data", "err", err)
            return err
        }

        l.sendTransaction(txdata, queue, receiptsCh)
        return nil
    }
```

#### TxData详解
`op-batcher/batcher/channel_manager.go`

`TxData`函数主要负责两件事务
- 查找第一个含有`frame`的的`channel`，如果存在且通过检查后使用`nextTxData`获取数据并返回
- 如果没有这样的`channel`，我们需要现调用`ensureChannelWithSpace`检查`channel`还有剩余的空间，再使用`processBlocks`将之前加载到`block队列`中的数据构造到 `outchannel的composer`当中压缩
- `outputFrames`将`outchannel composer`当中的数据切割成适合大小的`frame`
- 最后再把刚构造到数据通过`nextTxData`函数返回出去。


`EnsureChannelWithSpace` 确保 `currentChannel` 填充有可容纳更多数据的空间的`channel`（即，`channel.IsFull` 返回 `false`）。 如果 `currentChannel` 为零或已满，则会创建一个新`channel`。

```go
    func (s *channelManager) TxData(l1Head eth.BlockID) (txData, error) {
        s.mu.Lock()
        defer s.mu.Unlock()
        var firstWithFrame *channel
        for _, ch := range s.channelQueue {
            if ch.HasFrame() {
                firstWithFrame = ch
                break
            }
        }

        dataPending := firstWithFrame != nil && firstWithFrame.HasFrame()
        s.log.Debug("Requested tx data", "l1Head", l1Head, "data_pending", dataPending, "blocks_pending", len(s.blocks))

        // Short circuit if there is a pending frame or the channel manager is closed.
        if dataPending || s.closed {
            return s.nextTxData(firstWithFrame)
        }

        // No pending frame, so we have to add new blocks to the channel

        // If we have no saved blocks, we will not be able to create valid frames
        if len(s.blocks) == 0 {
            return txData{}, io.EOF
        }

        if err := s.ensureChannelWithSpace(l1Head); err != nil {
            return txData{}, err
        }

        if err := s.processBlocks(); err != nil {
            return txData{}, err
        }

        // Register current L1 head only after all pending blocks have been
        // processed. Even if a timeout will be triggered now, it is better to have
        // all pending blocks be included in this channel for submission.
        s.registerL1Block(l1Head)

        if err := s.outputFrames(); err != nil {
            return txData{}, err
        }

        return s.nextTxData(s.currentChannel)
    }

```

`processBlocks`函数在内部通过`AddBlock`把`block队列`里的`block`加入到当前的`channel`当中

```go
    func (s *channelManager) processBlocks() error {
        var (
            blocksAdded int
            _chFullErr  *ChannelFullError // throw away, just for type checking
            latestL2ref eth.L2BlockRef
        )
        for i, block := range s.blocks {
            l1info, err := s.currentChannel.AddBlock(block)
            if errors.As(err, &_chFullErr) {
                // current block didn't get added because channel is already full
                break
            } else if err != nil {
                return fmt.Errorf("adding block[%d] to channel builder: %w", i, err)
            }
            s.log.Debug("Added block to channel", "channel", s.currentChannel.ID(), "block", block)

            blocksAdded += 1
            latestL2ref = l2BlockRefFromBlockAndL1Info(block, l1info)
            s.metr.RecordL2BlockInChannel(block)
            // current block got added but channel is now full
            if s.currentChannel.IsFull() {
                break
            }
        }
```

`AddBlock` 首先通过`BlockToBatch`把`batch`从`blcok`中获取出来，再通过`AddBatch`函数对数据进行压缩并存储。

```go
    func (c *channelBuilder) AddBlock(block *types.Block) (derive.L1BlockInfo, error) {
        if c.IsFull() {
            return derive.L1BlockInfo{}, c.FullErr()
        }

        batch, l1info, err := derive.BlockToBatch(block)
        if err != nil {
            return l1info, fmt.Errorf("converting block to batch: %w", err)
        }

        if _, err = c.co.AddBatch(batch); errors.Is(err, derive.ErrTooManyRLPBytes) || errors.Is(err, derive.CompressorFullErr) {
            c.setFullErr(err)
            return l1info, c.FullErr()
        } else if err != nil {
            return l1info, fmt.Errorf("adding block to channel out: %w", err)
        }
        c.blocks = append(c.blocks, block)
        c.updateSwTimeout(batch)

        if err = c.co.FullErr(); err != nil {
            c.setFullErr(err)
            // Adding this block still worked, so don't return error, just mark as full
        }

        return l1info, nil
    }
```

在`txdata`获取后，使用`sendTransaction`将整个数据发送到L1当中。

## 总结
在这一章节中，我们了解了什么是`batcher`并且了解了`batcher`的运行原理，你可以在这个 [address](https://etherscan.io/address/0x6887246668a3b87f54deb3b94ba47a6f63f32985)中查看当前`batcher`的行为。