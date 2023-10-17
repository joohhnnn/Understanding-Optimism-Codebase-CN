# optimism中区块的传递

区块的传递是整个optimism rollup系统中较为重要的概念，在这一章节，我们将从介绍optimism中多种sync方式的原理，来揭开整个系统里区块的传递过程。

## 区块类型

在进行进一步深入前，让我们了解一些基本的概念。

- **Unsafe L2 Block (不安全的 L2 区块)**:
    - 这是指 L1 链上最高的 L2 区块，其 L1 起源是规范 L1 链的 *可能* 扩展（如 op-node 所知）。这意味着，尽管该区块链接到 L1 链，但其完整性和正确性尚未得到充分验证。

- **Safe L2 Block (安全的 L2 区块)**:
    - 这是指 L1 链上最高的 L2 区块，其 epoch 的序列窗口在规范的 L1 链中是完整的（如 op-node 所知）。这意味着该区块的所有前提条件都已在 L1 链上得到验证，因此它被认为是安全的。

- **Finalized L2 Block (定稿的 L2 区块)**:
    - 这是指已知完全源自定稿 L1 区块数据的 L2 区块。这意味着该区块不仅安全，而且已根据 L1 链的数据完全确认，不会再发生更改。


## sync类型

1. **op-node p2p gossip 同步**：
   - op-node 通过 p2p gossip 协议接收最新的不安全区块，由 sequencer 推送的。

2. **op-node 基于libp2p的请求-响应的逆向区块头同步**：
   - 通过此同步方式，op-node 可以填补不安全区块的任何缺口。

3. **执行层（EL，又名 engine sync）同步**：
   - 在 op-node 中有两个标志，允许来自 gossip 的不安全区块触发引擎中向这些区块的长范围同步。相关的标志是 `--l2.engine-sync` 和 `--l2.skip-sync-start-check`（用于处理非常旧的安全区块）。然后，如果为此设置了 EL，它可以执行任何同步，例如 snap-sync（需要 op-geth p2p 连接等，并且需要从某些节点进行同步）。

4. **op-node RPC 同步**：
   - 这是一种基于可信 RPC 方法的同步，当 L1 出现问题时，这种同步方式相对简单。

## op-node p2p gossip 同步

这种同步的场景处于：当l2的块新产生的时候，即在上一节我们讨论的sequencer模式下是如何产生新的区块的。

当产生新的区块后，sequencer通过基于libp2p的P2P网络的pub/sub（广播/订阅）模块，向’新unsafe区块‘ topic 发出广播。所有订阅了此topic的节点都会直接或间接的收到这一广播消息。详情可以查看[link](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/blob/main/sequencer/02-how-optimism-use-libp2p.md#gossip%E4%B8%8B%E7%9A%84%E5%8C%BA%E5%9D%97%E4%BC%A0%E6%92%AD)

## op-node 基于libp2p的请求-响应的逆向区块头同步

这种同步的场景处于：当节点因为特殊情况，比如宕机后重新链接，可能会产生一些没有同步上的区块（gaps）

当这种情况出现的时候，可以通过p2p网络的反向链的方式快速同步，即通过使用libp2p原生的stream流来和其他p2p节点建立链接，同时发送同步请求。详情可以查看[link](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/blob/main/sequencer/02-how-optimism-use-libp2p.md#%E5%BD%93%E5%AD%98%E5%9C%A8%E7%BC%BA%E5%A4%B1%E5%8C%BA%E5%9D%97%E9%80%9A%E8%BF%87p2p%E5%BF%AB%E9%80%9F%E5%90%8C%E6%AD%A5)

## 执行层（EL，又名 engine sync）同步

这种同步的场景处于：当有较多区块，一个大范围区块需要同步的时候，从l1慢慢派生比较慢，想要快速同步。

使用`--l2.engine-sync` 和 `--l2.skip-sync-start-check`去启动op-node，发送的payload来达到发送长范围同步请求的目的。

### 代码层讲解

首先我们来看一下这两个标志的定义

在 `op-node/flags/flags.go` 中定义并解释了这两个flag的作用

- **L2EngineSyncEnabled Flag (`l2.engine-sync`)**:
    - 该标志用于启用或禁用执行引擎的 P2P 同步功能。当设置为 `true` 时，它允许执行引擎通过 P2P 网络与其他节点同步区块数据。它的默认值是 `false`，意味着在默认情况下，该 P2P 同步功能是禁用的。

- **SkipSyncStartCheck Flag (`l2.skip-sync-start-check`)**:
    - 该标志用于在确定同步起始点时，跳过对不安全 L2 区块的 L1 起源一致性的合理性检查。当设置为 `true` 时，它会推迟 L1 起源的验证。如果你正在使用 `l2.engine-sync`，建议启用此标志来跳过初始的一致性检查。它的默认值是 `false`，意味着在默认情况下，该合理性检查是启用的。

```go
	L2EngineSyncEnabled = &cli.BoolFlag{
		Name:     "l2.engine-sync",
		Usage:    "Enables or disables execution engine P2P sync",
		EnvVars:  prefixEnvVars("L2_ENGINE_SYNC_ENABLED"),
		Required: false,
		Value:    false,
	}
	SkipSyncStartCheck = &cli.BoolFlag{
		Name: "l2.skip-sync-start-check",
		Usage: "Skip sanity check of consistency of L1 origins of the unsafe L2 blocks when determining the sync-starting point. " +
			"This defers the L1-origin verification, and is recommended to use in when utilizing l2.engine-sync",
		EnvVars:  prefixEnvVars("L2_SKIP_SYNC_START_CHECK"),
		Required: false,
		Value:    false,
	}
```
#### L2EngineSyncEnabled

L2EngineSyncEnabled标志用于在op-node接收到新的unsafe的payload（区块）后，发送给op-geth进一步验证时，触发op-geth的p2p之间sync，在sync期间所有的unsafe区块都会被视为通过验证，并进行下一个unsafe的流程。op-geth内部的p2p sync比较适用于长范围的unsafe区块的获取。其实在op-geth内部，不管L2EngineSyncEnabled标志有没有启用，在遇到parent区块不存在的时候，都会开启sync去同步数据。

让我们深入代码层面看一下
首先是 `op-node/rollup/derive/engine_queue.go`

`EngineSync`为L2EngineSyncEnabled标志的具体表达。在这里嵌套在两个检查函数当中。

```go
   // checkNewPayloadStatus checks returned status of engine_newPayloadV1 request for next unsafe payload.
   // It returns true if the status is acceptable.
   func (eq *EngineQueue) checkNewPayloadStatus(status eth.ExecutePayloadStatus) bool {
      if eq.syncCfg.EngineSync {
         // Allow SYNCING and ACCEPTED if engine P2P sync is enabled
         return status == eth.ExecutionValid || status == eth.ExecutionSyncing || status == eth.ExecutionAccepted
      }
      return status == eth.ExecutionValid
   }

   // checkForkchoiceUpdatedStatus checks returned status of engine_forkchoiceUpdatedV1 request for next unsafe payload.
   // It returns true if the status is acceptable.
   func (eq *EngineQueue) checkForkchoiceUpdatedStatus(status eth.ExecutePayloadStatus) bool {
      if eq.syncCfg.EngineSync {
         // Allow SYNCING if engine P2P sync is enabled
         return status == eth.ExecutionValid || status == eth.ExecutionSyncing
      }
      return status == eth.ExecutionValid
   }
```

让我们把视角转到op-geth的 `eth/catalyst/api.go`当中，当parent区块缺失后，触发sync，并且返回SYNCING Status

```go
   func (api *ConsensusAPI) newPayload(params engine.ExecutableData) (engine.PayloadStatusV1, error) {
      …
      // If the parent is missing, we - in theory - could trigger a sync, but that
      // would also entail a reorg. That is problematic if multiple sibling blocks
      // are being fed to us, and even more so, if some semi-distant uncle shortens
      // our live chain. As such, payload execution will not permit reorgs and thus
      // will not trigger a sync cycle. That is fine though, if we get a fork choice
      // update after legit payload executions.
      parent := api.eth.BlockChain().GetBlock(block.ParentHash(), block.NumberU64()-1)
      if parent == nil {
         return api.delayPayloadImport(block)
      }
      …
   }
```

```go
   func (api *ConsensusAPI) delayPayloadImport(block *types.Block) (engine.PayloadStatusV1, error) {
      …
      if err := api.eth.Downloader().BeaconExtend(api.eth.SyncMode(), block.Header()); err == nil {
         log.Debug("Payload accepted for sync extension", "number", block.NumberU64(), "hash", block.Hash())
         return engine.PayloadStatusV1{Status: engine.SYNCING}, nil
      }
      …
   }
```
#### SkipSyncStartCheck 
SkipSyncStartCheck这个标识符主要是帮助在选择sync模式下，优化性能和减少不必要的检查。在已确认找到一个符合条件的L2块后，代码会跳过进一步的健全性检查，以加速同步或其他后续处理。这是一种优化手段，用于在确定性高的情况下快速地进行操作。

在`op-node/rollup/sync/start.go`目录中

FindL2Heads函数通过从给定的“开始”（start）点（即之前的不安全L2区块）开始逐步回溯，来查找这三种类型的区块。在回溯过程中，该函数会检查各个L2区块的L1源是否与已知的L1规范链匹配，以及是否符合其他一些条件和检查。这允许函数更快地确定L2的“安全”头部，从而可能加速整个同步过程。

```go
   func FindL2Heads(ctx context.Context, cfg *rollup.Config, l1 L1Chain, l2 L2Chain, lgr log.Logger, syncCfg *Config) (result *FindHeadsResult, err error) {
      …
      for {

         …

         if syncCfg.SkipSyncStartCheck && highestL2WithCanonicalL1Origin.Hash == n.Hash {
            lgr.Info("Found highest L2 block with canonical L1 origin. Skip further sanity check and jump to the safe head")
            n = result.Safe
            continue
         }
         // Pull L2 parent for next iteration
         parent, err := l2.L2BlockRefByHash(ctx, n.ParentHash)
         if err != nil {
            return nil, fmt.Errorf("failed to fetch L2 block by hash %v: %w", n.ParentHash, err)
         }

         // Check the L1 origin relation
         if parent.L1Origin != n.L1Origin {
            // sanity check that the L1 origin block number is coherent
            if parent.L1Origin.Number+1 != n.L1Origin.Number {
               return nil, fmt.Errorf("l2 parent %s of %s has L1 origin %s that is not before %s", parent, n, parent.L1Origin, n.L1Origin)
            }
            // sanity check that the later sequence number is 0, if it changed between the L2 blocks
            if n.SequenceNumber != 0 {
               return nil, fmt.Errorf("l2 block %s has parent %s with different L1 origin %s, but non-zero sequence number %d", n, parent, parent.L1Origin, n.SequenceNumber)
            }
            // if the L1 origin is known to be canonical, then the parent must be too
            if l1Block.Hash == n.L1Origin.Hash && l1Block.ParentHash != parent.L1Origin.Hash {
               return nil, fmt.Errorf("parent L2 block %s has origin %s but expected %s", parent, parent.L1Origin, l1Block.ParentHash)
            }
         } else {
            if parent.SequenceNumber+1 != n.SequenceNumber {
               return nil, fmt.Errorf("sequence number inconsistency %d <> %d between l2 blocks %s and %s", parent.SequenceNumber, n.SequenceNumber, parent, n)
            }
         }

         n = parent

         // once we found the block at seq nr 0 that is more than a full seq window behind the common chain post-reorg, then use the parent block as safe head.
         if ready {
            result.Safe = n
            return result, nil
         }
      }
   }
```


### op-node RPC 同步

这种同步场景处于： 当你有信任的l2 rpc节点的时候，我们可以直接和rpc通信，发送较短范围的同步请求，和2类似。如果设置，在反向链同步中会优先使用RPC而不是P2P同步。

#### 关键代码

`op-node/node/node.go`

初始化rpcSync，如果rpcSyncClient设置，赋值给rpcSync

```go
   func (n *OpNode) initRPCSync(ctx context.Context, cfg *Config) error {
      rpcSyncClient, rpcCfg, err := cfg.L2Sync.Setup(ctx, n.log, &cfg.Rollup)
      if err != nil {
         return fmt.Errorf("failed to setup L2 execution-engine RPC client for backup sync: %w", err)
      }
      if rpcSyncClient == nil { // if no RPC client is configured to sync from, then don't add the RPC sync client
         return nil
      }
      syncClient, err := sources.NewSyncClient(n.OnUnsafeL2Payload, rpcSyncClient, n.log, n.metrics.L2SourceCache, rpcCfg)
      if err != nil {
         return fmt.Errorf("failed to create sync client: %w", err)
      }
      n.rpcSync = syncClient
      return nil
   }
```

启动node，如果rpcSync非空，开启rpcSync eventloop

```go
   func (n *OpNode) Start(ctx context.Context) error {
      n.log.Info("Starting execution engine driver")

      // start driving engine: sync blocks by deriving them from L1 and driving them into the engine
      if err := n.l2Driver.Start(); err != nil {
         n.log.Error("Could not start a rollup node", "err", err)
         return err
      }

      // If the backup unsafe sync client is enabled, start its event loop
      if n.rpcSync != nil {
         if err := n.rpcSync.Start(); err != nil {
            n.log.Error("Could not start the backup sync client", "err", err)
            return err
         }
         n.log.Info("Started L2-RPC sync service")
      }

      return nil
   }
```

`op-node/sources/sync_client.go`

一旦接收到s.requests通道里的信号后（区块号），调用fetchUnsafeBlockFromRpc函数从RPC节点中获取相应的区块信息。

```go
   // eventLoop is the main event loop for the sync client.
   func (s *SyncClient) eventLoop() {
      defer s.wg.Done()
      s.log.Info("Starting sync client event loop")

      backoffStrategy := &retry.ExponentialStrategy{
         Min:       1000 * time.Millisecond,
         Max:       20_000 * time.Millisecond,
         MaxJitter: 250 * time.Millisecond,
      }

      for {
         select {
         case <-s.resCtx.Done():
            s.log.Debug("Shutting down RPC sync worker")
            return
         case reqNum := <-s.requests:
            _, err := retry.Do(s.resCtx, 5, backoffStrategy, func() (interface{}, error) {
               // Limit the maximum time for fetching payloads
               ctx, cancel := context.WithTimeout(s.resCtx, time.Second*10)
               defer cancel()
               // We are only fetching one block at a time here.
               return nil, s.fetchUnsafeBlockFromRpc(ctx, reqNum)
            })
            if err != nil {
               if err == s.resCtx.Err() {
                  return
               }
               s.log.Error("failed syncing L2 block via RPC", "err", err, "num", reqNum)
               // Reschedule at end of queue
               select {
               case s.requests <- reqNum:
               default:
                  // drop syncing job if we are too busy with sync jobs already.
               }
            }
         }
      }
   }
```

接下来我们来看看从哪里往`s.requests`通道发送信号的呢？
同文件下的`RequestL2Range`函数，此函数介绍一个需要同步的区块范围，然后将任务通过for循环，分别发送出去。
```go
   func (s *SyncClient) RequestL2Range(ctx context.Context, start, end eth.L2BlockRef) error {
      // Drain previous requests now that we have new information
      for len(s.requests) > 0 {
         select { // in case requests is being read at the same time, don't block on draining it.
         case <-s.requests:
         default:
            break
         }
      }

      endNum := end.Number
      if end == (eth.L2BlockRef{}) {
         n, err := s.rollupCfg.TargetBlockNumber(uint64(time.Now().Unix()))
         if err != nil {
            return err
         }
         if n <= start.Number {
            return nil
         }
         endNum = n
      }

      // TODO(CLI-3635): optimize the by-range fetching with the Engine API payloads-by-range method.

      s.log.Info("Scheduling to fetch trailing missing payloads from backup RPC", "start", start, "end", endNum, "size", endNum-start.Number-1)

      for i := start.Number + 1; i < endNum; i++ {
         select {
         case s.requests <- i:
         case <-ctx.Done():
            return ctx.Err()
         }
      }
      return nil
   }
```

在外层的OpNode类型的RequestL2Range实现方法里。可以清楚的看到rpcSync类型的反向链同步是优先选择的。

```go
   func (n *OpNode) RequestL2Range(ctx context.Context, start, end eth.L2BlockRef) error {
      if n.rpcSync != nil {
         return n.rpcSync.RequestL2Range(ctx, start, end)
      }
      if n.p2pNode != nil && n.p2pNode.AltSyncEnabled() {
         if unixTimeStale(start.Time, 12*time.Hour) {
            n.log.Debug("ignoring request to sync L2 range, timestamp is too old for p2p", "start", start, "end", end, "start_time", start.Time)
            return nil
         }
         return n.p2pNode.RequestL2Range(ctx, start, end)
      }
      n.log.Debug("ignoring request to sync L2 range, no sync method available", "start", start, "end", end)
      return nil
   }
```
## 总结
理解了这些同步方式后，我们知道了unsafe的payload（区块）究竟是怎么进行传递的。不同的sync模块对应着在不同场景下的区块数据传递。那么整个网络中如何一步步的将unsafe的区块变成safe区块，然后再进行finalized的呢？这些内容会在其他章节进行讲解。