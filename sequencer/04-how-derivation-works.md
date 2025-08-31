# opstack是如何从Layer1中派生出来Layer2的
在阅读本文章之前，我强烈建议你先阅读一下来自`optimism/specs`中有关派生部分的介绍([source](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#deriving-payload-attributes))
如果你看完这篇文章，感到迷茫，这是正常的。但是还是请记住这份感觉，因为在看完我们这篇文章的分析之后，请你回过来头再看一遍，你就会发现这篇官方的文章真的很凝练，把所有要点和细节都精炼的阐述了一遍。

接下来让我们进入文章正题。我们都知道layer2的运行节点，是可以从DA层（layer1）中获取数据，并且构建出完整的区块数据的。今天我们就来讲解一下这个过程中是如何在`codebase`中实现的。

## 你需要有的问题

如果现在让你设计这样一套系统，你会怎么设计呢？你会有哪些问题？在这里我列出来了一些问题，带着这些问题去思考会帮助你更好的理解整篇文章
 - 当你启动一个新节点的时候，整个系统是如何运行的？
 - 你需要一个个去查询所有l1的区块数据吗？如何触发查询？
 - 当拿到l1区块的数据后，你需要哪些数据？
 - 派生过程中，区块的状态是怎么变化的？如何从`unsafe`变成`safe`再变成`finalized`？
 - 官方specs中晦涩的数据结构 `batch/channel/frame` 这些到底是干嘛的？（可以在上一章`03-how-batcher-works`章节中详细理解）

## 什么是派生（derivation）？

在理解`derivation`前，我们先来聊一聊optimism的基本rollup机制，这里我们简单以一笔l2上的transfer交易为例。

当你在optimism网络上发出一笔转账交易，这笔交易会被"转发"给`sequencer`节点，由`sequencer`进行排序，然后进行区块的封装并进行区块的广播，这里可以理解为出块。我们把这个包含你交易的区块称为`区块A`。这时的`区块A`状态为`unsafe`。接下来等`sequencer`达到一定的时间间隔了（比如4分钟），会由`sequencer`中的`batcher`的模块把这四分钟内所有收集到的交易（包括你这笔转账交易）通过一笔交易发送到l1上，并由l1产出区块X。这时的区块A状态仍然为`unsafe`。当任何一个节点执行`derivation`部分的程序后，此节点从l1中获取区块X的数据，并对`本地l2的unsafe区块A`进行更新。这时的`区块A`状态为`safe`。在经过`l1两个epoch（64个区块）`后，由l2节点将区块A标记为`finalized`区块。

而派生就是把角色带入到上述例子的l2节点当中，通过不断的并行执行`derivation`程序将获取的`unsafe`区块逐步变成`safe`区块，同时把已经是`safe`的区块逐步变成`finalized`状态的一个过程。

## 代码层深潜

hoho 船长，让我们深潜🤿

### 获取batcher发送的batch transactions的data

我们先来看看当我们知道一个新的l1的区块时，如何查看区块里面是否有`batch transactions`的数据
在这里我们先梳理一下所需要的模块，再针对这些模块进行查看
- 首先要确定下一个l1的区块块号是多少
- 将下一个区块的数据解析出来

#### 确定下一个区块的块号
`op-node/rollup/derive/l1_traversal.go`

通过查询当前`origin.Number + 1`的块高来获取最新的l1块，如果此块不存在，即`error`和`ethereum.NotFound`匹配，那么就代表当前块高即为最新的区块，下一个区块还未在l1上产生。如果获取成功，将最新的区块号记录在`l1t.block`中

> **Source Code**: [op-node/rollup/derive/l1_traversal.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/l1_traversal.go#L60)

```go
    func (l1t *L1Traversal) AdvanceL1Block(ctx context.Context) error {
        origin := l1t.block
        nextL1Origin, err := l1t.l1Blocks.L1BlockRefByNumber(ctx, origin.Number+1)
        if errors.Is(err, ethereum.NotFound) {
            l1t.log.Debug("can't find next L1 block info (yet)", "number", origin.Number+1, "origin", origin)
            return io.EOF
        } else if err != nil {
            return NewTemporaryError(fmt.Errorf("failed to find L1 block info by number, at origin %s next %d: %w", origin, origin.Number+1, err))
        }
        if l1t.block.Hash != nextL1Origin.ParentHash {
            return NewResetError(fmt.Errorf("detected L1 reorg from %s to %s with conflicting parent %s", l1t.block, nextL1Origin, nextL1Origin.ParentID()))
        }

        ……

        l1t.block = nextL1Origin
        l1t.done = false
        return nil
    }
```

#### 将区块的data解析出来
`op-node/rollup/derive/calldata_source.go`

首先先通过`InfoAndTxsByHash`将刚才获取的区块的所有`transactions`拿到，然后将`transactions`和我们的batcherAddr还有我们的config传入到`DataFromEVMTransactions`函数中，
为什么要传这些参数呢？因为我们在过滤这些交易的时候，需要保证`batcher`地址和接收地址的准确性（权威性）。在`DataFromEVMTransactions`接收到这些参数后，通过循环对每个交易进行地址的准确性过滤，找到正确的`batch transactions`。

> **Source Code**: [op-node/rollup/derive/calldata_source.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/calldata_source.go#L62)

```go
    func NewDataSource(ctx context.Context, log log.Logger, cfg *rollup.Config, fetcher L1TransactionFetcher, block eth.BlockID, batcherAddr common.Address) DataIter {
        _, txs, err := fetcher.InfoAndTxsByHash(ctx, block.Hash)
        if err != nil {
            return &DataSource{
                open:        false,
                id:          block,
                cfg:         cfg,
                fetcher:     fetcher,
                log:         log,
                batcherAddr: batcherAddr,
            }
        } else {
            return &DataSource{
                open: true,
                data: DataFromEVMTransactions(cfg, batcherAddr, txs, log.New("origin", block)),
            }
        }
    }
```

> **Source Code**: [op-node/rollup/derive/calldata_source.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/calldata_source.go#L107)

```go
    func DataFromEVMTransactions(config *rollup.Config, batcherAddr common.Address, txs types.Transactions, log log.Logger) []eth.Data {
        var out []eth.Data
        l1Signer := config.L1Signer()
        for j, tx := range txs {
            if to := tx.To(); to != nil && *to == config.BatchInboxAddress {
                seqDataSubmitter, err := l1Signer.Sender(tx) // optimization: only derive sender if To is correct
                if err != nil {
                    log.Warn("tx in inbox with invalid signature", "index", j, "err", err)
                    continue // bad signature, ignore
                }
                // some random L1 user might have sent a transaction to our batch inbox, ignore them
                if seqDataSubmitter != batcherAddr {
                    log.Warn("tx in inbox with unauthorized submitter", "index", j, "err", err)
                    continue // not an authorized batch submitter, ignore
                }
                out = append(out, tx.Data())
            }
        }
        return out
    }
```

### 从data到safeAttribute,使unsafe的区块safe化

在这一部分，首先会将上一步我们解析出来的`data`解析成`frame`并添加到`FrameQueue`的`frames`数组里面。然后从`frames`数组中提取一个`frame`，并将`frame`初始化进一个`channel`并添加到`channelbank`当中，等待该`channel`中的`frames`添加完毕后，从`channel`中提取`batch`信息，把`batch`添加到`BatchQueue`中，将`BatchQueue`中的`batch`添加到`AttributesQueue`中，用来构造`safeAttributes`，并把`enginequeue`里面的`safeblcok`更新，最终通过`ForkchoiceUpdate`函数的调用来完成EL层`safeblock`的更新

#### data -> frame
`op-node/rollup/derive/frame_queue.go`

此函数通过`NextData`函数获取上一步的data，然后将此data解析后添加到`FrameQueue`的`frames`数组里面，并返回在数组中第一个`frame`。

> **Source Code**: [op-node/rollup/derive/frame_queue.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/frame_queue.go#L36)

```go
    func (fq *FrameQueue) NextFrame(ctx context.Context) (Frame, error) {
        // Find more frames if we need to
        if len(fq.frames) == 0 {
            if data, err := fq.prev.NextData(ctx); err != nil {
                return Frame{}, err
            } else {
                if new, err := ParseFrames(data); err == nil {
                    fq.frames = append(fq.frames, new...)
                } else {
                    fq.log.Warn("Failed to parse frames", "origin", fq.prev.Origin(), "err", err)
                }
            }
        }
        // If we did not add more frames but still have more data, retry this function.
        if len(fq.frames) == 0 {
            return Frame{}, NotEnoughData
        }

        ret := fq.frames[0]
        fq.frames = fq.frames[1:]
        return ret, nil
    }
```

#### frame -> channel
`op-node/rollup/derive/channel_bank.go`

`NextData`函数负责从当前`channel bank`中读出第一个`channel`中的`raw data`并返回，同时负责调用`NextFrame`获取`frame`并装载到`channel`中

> **Source Code**: [op-node/rollup/derive/channel_bank.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/channel_bank.go#L158)

```go
    func (cb *ChannelBank) NextData(ctx context.Context) ([]byte, error) {
        // Do the read from the channel bank first
        data, err := cb.Read()
        if err == io.EOF {
            // continue - We will attempt to load data into the channel bank
        } else if err != nil {
            return nil, err
        } else {
            return data, nil
        }

        // Then load data into the channel bank
        if frame, err := cb.prev.NextFrame(ctx); err == io.EOF {
            return nil, io.EOF
        } else if err != nil {
            return nil, err
        } else {
            cb.IngestFrame(frame)
            return nil, NotEnoughData
        }
    }
```

#### channel -> batch
`op-node/rollup/derive/channel_in_reader.go`

`NextBatch`函数主要负责将刚才到`raw data` 解码成具有`batch`结构的数据并返回。其中`WriteChannel`函数的作用是提供一个函数并赋值给`nextBatchFn`，这个函数的目的是创建一个读取器，从读取器中解码`batch`结构的数据并返回。

> **Source Code**: [op-node/rollup/derive/channel_in_reader.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/channel_in_reader.go#L63)

```go
    func (cr *ChannelInReader) NextBatch(ctx context.Context) (*BatchData, error) {
        if cr.nextBatchFn == nil {
            if data, err := cr.prev.NextData(ctx); err == io.EOF {
                return nil, io.EOF
            } else if err != nil {
                return nil, err
            } else {
                if err := cr.WriteChannel(data); err != nil {
                    return nil, NewTemporaryError(err)
                }
            }
        }

        // TODO: can batch be non nil while err == io.EOF
        // This depends on the behavior of rlp.Stream
        batch, err := cr.nextBatchFn()
        if err == io.EOF {
            cr.NextChannel()
            return nil, NotEnoughData
        } else if err != nil {
            cr.log.Warn("failed to read batch from channel reader, skipping to next channel now", "err", err)
            cr.NextChannel()
            return nil, NotEnoughData
        }
        return batch.Batch, nil
    }
```

**注意❗️在这里NextBatch函数产生的batch并没有被直接使用，而是先加入了batchQueue当中，再统一管理和使用，并且这里的NextBatch实际由 op-node/rollup/derive/batch_queue.go 目录下的func (bq *BatchQueue) NextBatch()函数调用**
#### batch -> safeAttributes
**补充信息：**
1.在layer2区块中，区块中的交易中的第一个永远都是一个`锚定交易`，可以简单理解为包含了一些l1的信息，如果这个layer2区块同时还是epoch中第一个区块的话，那么还会包含来自layer1的`deposit`交易（[epoch中第一个区块示例](https://optimistic.etherscan.io/txs?block=110721915]）。
2.这里的batch不能理解为batcher发送的batch交易。例如，我们在这里将batcher发送的batch交易命名为batchA，而在我们这里使用和讨论的命名为batchB，batchA和batchB的关系为包含关系，即batchA中可能包含非常巨量的交易，这些交易可以构造为batchB，batchBB，batchBBB等。batchB对应一个layer2中区块的交易，而batchA对应大量layer2中区块的交易。

`op-node/rollup/derive/attributes_queue.go`
- `NextAttributes`函数传入当前l2的safe区块头后，将块头和我们上一步获取的batch传递到`createNextAttributes`函数中，构造`safeAttributes`。
- `createNextAttributes`中我们要注意的是，`createNextAttributes`函数内部调用的`PreparePayloadAttributes`函数，`PreparePayloadAttributes`函数主要负责，锚定交易和`deposit`交易的。最后再把`batch`的交易和`PreparePayloadAttributes`函数返回的交易拼接起来后返回

`createNextAttributes`函数在内部调用`PreparePayloadAttributes`

> **Source Code**: [op-node/rollup/derive/attributes_queue.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/attributes_queue.go#L51)

```go
    func (aq *AttributesQueue) NextAttributes(ctx context.Context, l2SafeHead eth.L2BlockRef) (*eth.PayloadAttributes, error) {
        // Get a batch if we need it
        if aq.batch == nil {
            batch, err := aq.prev.NextBatch(ctx, l2SafeHead)
            if err != nil {
                return nil, err
            }
            aq.batch = batch
        }

        // Actually generate the next attributes
        if attrs, err := aq.createNextAttributes(ctx, aq.batch, l2SafeHead); err != nil {
            return nil, err
        } else {
            // Clear out the local state once we will succeed
            aq.batch = nil
            return attrs, nil
        }

    }
```

> **Source Code**: [op-node/rollup/derive/attributes_queue.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/attributes_queue.go#L74)

```go
    func (aq *AttributesQueue) createNextAttributes(ctx context.Context, batch *BatchData, l2SafeHead eth.L2BlockRef) (*eth.PayloadAttributes, error) {
        
        ……
        attrs, err := aq.builder.PreparePayloadAttributes(fetchCtx, l2SafeHead, batch.Epoch())
        ……

        return attrs, nil
    }
```

```go
    func (aq *AttributesQueue) createNextAttributes(ctx context.Context, batch *BatchData, l2SafeHead eth.L2BlockRef) (*eth.PayloadAttributes, error) {
        // sanity check parent hash
        if batch.ParentHash != l2SafeHead.Hash {
            return nil, NewResetError(fmt.Errorf("valid batch has bad parent hash %s, expected %s", batch.ParentHash, l2SafeHead.Hash))
        }
        // sanity check timestamp
        if expected := l2SafeHead.Time + aq.config.BlockTime; expected != batch.Timestamp {
            return nil, NewResetError(fmt.Errorf("valid batch has bad timestamp %d, expected %d", batch.Timestamp, expected))
        }
        fetchCtx, cancel := context.WithTimeout(ctx, 20*time.Second)
        defer cancel()
        attrs, err := aq.builder.PreparePayloadAttributes(fetchCtx, l2SafeHead, batch.Epoch())
        if err != nil {
            return nil, err
        }

        // we are verifying, not sequencing, we've got all transactions and do not pull from the tx-pool
        // (that would make the block derivation non-deterministic)
        attrs.NoTxPool = true
        attrs.Transactions = append(attrs.Transactions, batch.Transactions...)

        aq.log.Info("generated attributes in payload queue", "txs", len(attrs.Transactions), "timestamp", batch.Timestamp)

        return attrs, nil
    }
```

#### safeAttributes -> safe block
在这一步，会先`engine queue`中的`safehead`设置为`safe`，但是这并不代表这个区块是`safe`的了，还必须通过`ForkchoiceUpdat`在EL中更新

`op-node/rollup/derive/engine_queue.go`

`tryNextSafeAttributes`函数在内部判断是否当前`safehead`和`unsafehead`的关系，如果一切正常，则触发`consolidateNextSafeAttributes`函数来把`engine queue`中的`safeHead` 设置为我们上一步拿到的`safeAttributes`构造出来的`safe`区块，并将`needForkchoiceUpdate`设置为`true`，触发后续的`ForkchoiceUpdate`来把EL中的区块状态改成`safe`而真正将`unsafe`区块转化成`safe`区块。最后的`postProcessSafeL2`函数是将`safehead`加入到`finalizedL1`队列中，以供后续`finalied`使用。

> **Source Code**: [op-node/rollup/derive/engine_queue.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/engine_queue.go#L548)

```go
    func (eq *EngineQueue) tryNextSafeAttributes(ctx context.Context) error {
        ……
        if eq.safeHead.Number < eq.unsafeHead.Number {
            return eq.consolidateNextSafeAttributes(ctx)
        } 
        ……
    }

    func (eq *EngineQueue) consolidateNextSafeAttributes(ctx context.Context) error {
        ……
        payload, err := eq.engine.PayloadByNumber(ctx, eq.safeHead.Number+1)
        ……
        ref, err := PayloadToBlockRef(payload, &eq.cfg.Genesis)
        ……
        eq.safeHead = ref
        eq.needForkchoiceUpdate = true
        eq.postProcessSafeL2()
        ……
        return nil
    }
```

### 将safe区块finalized化
safe区块并不是真的牢固安全的区块，他还需要进行进一步的最终化确定，即`finalized`化。当一个区块的状态转变为`safe`时，从此区块派生的来源`L1（batcher transaction）`开始计算，经过两个`L1 epoch（64个区块`后，此`safe`区块可以被更新成`finalzied`状态。

`op-node/rollup/derive/engine_queue.go`

`tryFinalizePastL2Blocks`函数在内部对`finalized队列`中区块进行64个区块的校验，如果通过校验，调用`tryFinalizeL2`来完成`engine queue`当中`finalized`的设置和标记`needForkchoiceUpdate`的更新。

> **Source Code**: [op-node/rollup/derive/engine_queue.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/engine_queue.go#L328)

```go
    func (eq *EngineQueue) tryFinalizePastL2Blocks(ctx context.Context) error {
        ……
        eq.log.Info("processing L1 finality information", "l1_finalized", eq.finalizedL1, "l1_origin", eq.origin, "previous", eq.triedFinalizeAt) //const finalityDelay untyped int = 64

        // Sanity check we are indeed on the finalizing chain, and not stuck on something else.
        // We assume that the block-by-number query is consistent with the previously received finalized chain signal
        ref, err := eq.l1Fetcher.L1BlockRefByNumber(ctx, eq.origin.Number)
        if err != nil {
            return NewTemporaryError(fmt.Errorf("failed to check if on finalizing L1 chain: %w", err))
        }
        if ref.Hash != eq.origin.Hash {
            return NewResetError(fmt.Errorf("need to reset, we are on %s, not on the finalizing L1 chain %s (towards %s)", eq.origin, ref, eq.finalizedL1))
        }
        eq.tryFinalizeL2()
        return nil
    }
```

> **Source Code**: [op-node/rollup/derive/engine_queue.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/engine_queue.go#L363)

```go
    func (eq *EngineQueue) tryFinalizeL2() {
        if eq.finalizedL1 == (eth.L1BlockRef{}) {
            return // if no L1 information is finalized yet, then skip this
        }
        eq.triedFinalizeAt = eq.origin
        // default to keep the same finalized block
        finalizedL2 := eq.finalized
        // go through the latest inclusion data, and find the last L2 block that was derived from a finalized L1 block
        for _, fd := range eq.finalityData {
            if fd.L2Block.Number > finalizedL2.Number && fd.L1Block.Number <= eq.finalizedL1.Number {
                finalizedL2 = fd.L2Block
                eq.needForkchoiceUpdate = true
            }
        }
        eq.finalized = finalizedL2
        eq.metrics.RecordL2Ref("l2_finalized", finalizedL2)
    }
```
### 循环触发
在`op-node/rollup/driver/state.go`中的`eventLoop`函数中负责触发整个循环过程中的执行入口。主要是间接执行了了`op-node/rollup/derive/engine_queue.go`中`Step`函数

> **Source Code**: [op-node/rollup/derive/engine_queue.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/engine_queue.go#L245)

```go
func (eq *EngineQueue) Step(ctx context.Context) error {
	if eq.needForkchoiceUpdate {
		return eq.tryUpdateEngine(ctx)
	}
	// Trying unsafe payload should be done before safe attributes
	// It allows the unsafe head can move forward while the long-range consolidation is in progress.
	if eq.unsafePayloads.Len() > 0 {
		if err := eq.tryNextUnsafePayload(ctx); err != io.EOF {
			return err
		}
		// EOF error means we can't process the next unsafe payload. Then we should process next safe attributes.
	}
	if eq.isEngineSyncing() {
		// Make pipeline first focus to sync unsafe blocks to engineSyncTarget
		return EngineP2PSyncing
	}
	if eq.safeAttributes != nil {
		return eq.tryNextSafeAttributes(ctx)
	}
	outOfData := false
	newOrigin := eq.prev.Origin()
	// Check if the L2 unsafe head origin is consistent with the new origin
	if err := eq.verifyNewL1Origin(ctx, newOrigin); err != nil {
		return err
	}
	eq.origin = newOrigin
	eq.postProcessSafeL2() // make sure we track the last L2 safe head for every new L1 block
	// try to finalize the L2 blocks we have synced so far (no-op if L1 finality is behind)
	if err := eq.tryFinalizePastL2Blocks(ctx); err != nil {
		return err
	}
	if next, err := eq.prev.NextAttributes(ctx, eq.safeHead); err == io.EOF {
		outOfData = true
	} else if err != nil {
		return err
	} else {
		eq.safeAttributes = &attributesWithParent{
			attributes: next,
			parent:     eq.safeHead,
		}
		eq.log.Debug("Adding next safe attributes", "safe_head", eq.safeHead, "next", next)
		return NotEnoughData
	}

	if outOfData {
		return io.EOF
	} else {
		return nil
	}
}
```
## 总结
整个`derivation`功能看似非常复杂，但是你如果将每个环节都拆解开的话，还是能够很好的掌握理解的，官方的那篇specs不好理解的原因在于，他的`batch`，`frame`，`channel`等概念很容易让人迷茫，因此，如果你在看完这篇文章后，仍然觉得还很迷惑，建议可以回过头去再看看我们的`03-how-batcher-works`。