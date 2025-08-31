# op-proposer介绍

在这一章节中，我们将探讨到底什么是`op-proposer` 🌊

首先分享下来自官方specs中的资源([source](https://github.com/ethereum-optimism/optimism/blob/develop/specs/proposals.md))

一句话概括性的描述`proposer`的作用：定期的将来自layer2上的状态根（state root）发送到layer1上，以便可以无需信任地直接在layer1层面上执行一些来自layer2的交易，如提款或者message通信。

在本章节中，将会以处理来自layer2的一笔layer1的提现交易为例子，讲解`op-proposer`在整个流程中的作用。

## 提现流程

在Optimism中，提现是从L2（如 OP Mainnet, OP Goerli）到L1（如 Ethereum mainnet, Goerli）的交易，可能附带或不附带资产。可以粗略的分为四笔交易：
- 用户在L2提交的提现发起交易；
- `proposer`将L2中的`state root` 通过发送交易的方式上传到L1当中，以供接下来步骤中用户中L1中使用
- 用户在L1提交的提现证明交易，基于`Merkle Patricia Trie`，证明提现的合法性；
- 错误挑战期过后，用户在L1提交的提现最终交易，实际运行L1交易，认领任何附加资产等；

具体详情可以查看官方对于这部分的描述([source](https://community.optimism.io/docs/protocol/withdrawal-flow/#))

## 什么是proposer
`proposer`是服务于在L1中需要用到L2部分数据时的连通器，通过`proposer`将这一部分L2的数据（state root）发送到L1的合约当中。L1的合约就可以通过合约调用的方式直接使用了。

注意⚠️：很多人认为`proposer`发送的`state root`后**才**代表这些设计的区块是`finalized`。这种理解是`错误`的。`safe`的区块在L1中经过**两个**`epoch（64个区块）`后即可认定为`finalized`。
`proposer`是将`finalized`的区块数据上传，而不是上传后才`finalized`。

### proposer和batcher的区别
在之前我们讲解了`batcher`部分，`batcher`也是负责把L2的数据发送到L1中。你可能会有疑问，`batcher`不都已经把数据搬运到L1当中了，为什么还需要一个`proposer`再进行一次搬运呢？

#### 区块状态不一致
`batcher`发送数据时，区块的状态还是`unsafe`状态，不能直接使用，且无法根据`batcher`的交易来判断区块的状态何时变成`finalized`状态。
`proposer`发送数据时，代表了相关区块已经达到了`finalized`阶段，可以最大程度的去相信并使用相关数据。

#### 传递的数据格式和大小不同
`batcher`是将几乎完整的交易信息，包括`gasprice，data`等详细信息存储在`layer1`当中。
`proposer`只是将区块的`state root`发送到l1当中。`state root`后续配合[merkle-tree](https://medium.com/techskill-brew/merkle-tree-in-blockchain-part-5-blockchain-basics-4e25b61179a2)的设计使用
`batcher`传递的数据是巨量，`proposer`是少量的。因此`batcher`的数据更适合放置在`calldate`中，便宜，但是不能直接被合约使用。`proposer`的数据存储在`合约的storage`当中，数据量少，成本不会很高，并且可以在合约交互中使用。

#### 在以太坊中，数据存储calldata当中和存储在合约的storage当中的区别
在以太坊中，`calldata`和`storage`的区别主要有三方面：

1. **持久性**：
   - `storage`：持久存储，数据永久保存。
   - `calldata`：临时存储，函数执行完毕后数据消失。

2. **成本**：
   - `storage`：较贵，需永久存储数据。
   - `calldata`：较便宜，临时存储。

3. **可访问性**：
   - `storage`：多个函数或事务中可访问。
   - `calldata`：仅当前函数执行期间可访问。

## 代码实现
在这部分我们会从代码层来进行深度的机制和实现原理的讲解

### 程序起点
`op-proposer/proposer/l2_output_submitter.go`

通过调用`Start`函数来启动`loop`循环，在`loop`的循环中，主要通过函数`FetchNextOutputInfo`负责查看下一个区块是否该发送`proposal`交易，如果需要发送，则直接调用`sendTransaction`函数发送到L1当作，如不需要发送，则进行下一次循环。

> **Source Code**: [op-proposer/proposer/l2_output_submitter.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-proposer/proposer/l2_output_submitter.go#L381)

```go
    func (l *L2OutputSubmitter) loop() {
        defer l.wg.Done()

        ctx := l.ctx

        ticker := time.NewTicker(l.pollInterval)
        defer ticker.Stop()
        for {
            select {
            case <-ticker.C:
                output, shouldPropose, err := l.FetchNextOutputInfo(ctx)
                if err != nil {
                    break
                }
                if !shouldPropose {
                    break
                }
                cCtx, cancel := context.WithTimeout(ctx, 10*time.Minute)
                if err := l.sendTransaction(cCtx, output); err != nil {
                    l.log.Error("Failed to send proposal transaction",
                        "err", err,
                        "l1blocknum", output.Status.CurrentL1.Number,
                        "l1blockhash", output.Status.CurrentL1.Hash,
                        "l1head", output.Status.HeadL1.Number)
                    cancel()
                    break
                }
                l.metr.RecordL2BlocksProposed(output.BlockRef)
                cancel()

            case <-l.done:
                return
            }
        }
    }
```

### 获取output
`op-proposer/proposer/l2_output_submitter.go`

`FetchNextOutputInfo`函数通过调用`l2ooContract`合约来获取下一次该发送`proposal`的区块数，再将该区块块号和当前L2度区块块号进行比较，来判断是否应该发送`proposal`交易。如果需要发送，则调用`fetchOutput`函数来生成`output`

> **Source Code**: [op-proposer/proposer/l2_output_submitter.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-proposer/proposer/l2_output_submitter.go#L241)

```go
    func (l *L2OutputSubmitter) FetchNextOutputInfo(ctx context.Context) (*eth.OutputResponse, bool, error) {
        cCtx, cancel := context.WithTimeout(ctx, l.networkTimeout)
        defer cancel()
        callOpts := &bind.CallOpts{
            From:    l.txMgr.From(),
            Context: cCtx,
        }
        nextCheckpointBlock, err := l.l2ooContract.NextBlockNumber(callOpts)
        if err != nil {
            l.log.Error("proposer unable to get next block number", "err", err)
            return nil, false, err
        }
        // Fetch the current L2 heads
        cCtx, cancel = context.WithTimeout(ctx, l.networkTimeout)
        defer cancel()
        status, err := l.rollupClient.SyncStatus(cCtx)
        if err != nil {
            l.log.Error("proposer unable to get sync status", "err", err)
            return nil, false, err
        }

        // Use either the finalized or safe head depending on the config. Finalized head is default & safer.
        var currentBlockNumber *big.Int
        if l.allowNonFinalized {
            currentBlockNumber = new(big.Int).SetUint64(status.SafeL2.Number)
        } else {
            currentBlockNumber = new(big.Int).SetUint64(status.FinalizedL2.Number)
        }
        // Ensure that we do not submit a block in the future
        if currentBlockNumber.Cmp(nextCheckpointBlock) < 0 {
            l.log.Debug("proposer submission interval has not elapsed", "currentBlockNumber", currentBlockNumber, "nextBlockNumber", nextCheckpointBlock)
            return nil, false, nil
        }

        return l.fetchOutput(ctx, nextCheckpointBlock)
    }
```

`fetchOutput`函数在内部间接通过`OutputV0AtBlock`函数来获取并处理`output`返回体

`op-service/sources/l2_client.go`

`OutputV0AtBlock`函数获取之前检索出来需要传递`proposal`的区块哈希来拿到区块头，再根据这个区块头派生`OutputV0`所需要的数据。其中通过`GetProof`函数获取的的`proof`中的`StorageHash（withdrawal_storage_root）`的作用是，如果只需要`L2ToL1MessagePasserAddr`相关的`state`的数据的话，`withdrawal_storage_root`可以大幅度减小整个默克尔树证明过程的大小。

> **Source Code**: [op-node/sources/l2_client.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/sources/l2_client.go#L170)

```go
    func (s *L2Client) OutputV0AtBlock(ctx context.Context, blockHash common.Hash) (*eth.OutputV0, error) {
        head, err := s.InfoByHash(ctx, blockHash)
        if err != nil {
            return nil, fmt.Errorf("failed to get L2 block by hash: %w", err)
        }
        if head == nil {
            return nil, ethereum.NotFound
        }

        proof, err := s.GetProof(ctx, predeploys.L2ToL1MessagePasserAddr, []common.Hash{}, blockHash.String())
        if err != nil {
            return nil, fmt.Errorf("failed to get contract proof at block %s: %w", blockHash, err)
        }
        if proof == nil {
            return nil, fmt.Errorf("proof %w", ethereum.NotFound)
        }
        // make sure that the proof (including storage hash) that we retrieved is correct by verifying it against the state-root
        if err := proof.Verify(head.Root()); err != nil {
            return nil, fmt.Errorf("invalid withdrawal root hash, state root was %s: %w", head.Root(), err)
        }
        stateRoot := head.Root()
        return &eth.OutputV0{
            StateRoot:                eth.Bytes32(stateRoot),
            MessagePasserStorageRoot: eth.Bytes32(proof.StorageHash),
            BlockHash:                blockHash,
        }, nil
    }
```

### 发送output

`op-proposer/proposer/l2_output_submitter.go`

在`sendTransaction`函数中会间接调用`proposeL2OutputTxData`函数去使用`L1链上合约的ABI`来将我们的`output`与合约函数的`入参格式`进行匹配。随后`sendTransaction`函数将包装好的数据发送到L1上，与`L2OutputOracle合约`交互。

> **Source Code**: [op-proposer/proposer/l2_output_submitter.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-proposer/proposer/l2_output_submitter.go#L313)

```go
    func proposeL2OutputTxData(abi *abi.ABI, output *eth.OutputResponse) ([]byte, error) {
        return abi.Pack(
            "proposeL2Output",
            output.OutputRoot,
            new(big.Int).SetUint64(output.BlockRef.Number),
            output.Status.CurrentL1.Hash,
            new(big.Int).SetUint64(output.Status.CurrentL1.Number))
    }
```

`packages/contracts-bedrock/src/L1/L2OutputOracle.sol`

`L2OutputOracle合约`通过将此来自`L2区块的state root`进行校验，并存入`合约的storage`当中。

> **Source Code**: [packages/contracts-bedrock/src/L1/L2OutputOracle.sol (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/packages/contracts-bedrock/src/L1/L2OutputOracle.sol#L178)

```solidity
    /// @notice Accepts an outputRoot and the timestamp of the corresponding L2 block.
    ///         The timestamp must be equal to the current value returned by `nextTimestamp()` in
    ///         order to be accepted. This function may only be called by the Proposer.
    /// @param _outputRoot    The L2 output of the checkpoint block.
    /// @param _l2BlockNumber The L2 block number that resulted in _outputRoot.
    /// @param _l1BlockHash   A block hash which must be included in the current chain.
    /// @param _l1BlockNumber The block number with the specified block hash.
    function proposeL2Output(
        bytes32 _outputRoot,
        uint256 _l2BlockNumber,
        bytes32 _l1BlockHash,
        uint256 _l1BlockNumber
    )
        external
        payable
    {
        require(msg.sender == proposer, "L2OutputOracle: only the proposer address can propose new outputs");

        require(
            _l2BlockNumber == nextBlockNumber(),
            "L2OutputOracle: block number must be equal to next expected block number"
        );

        require(
            computeL2Timestamp(_l2BlockNumber) < block.timestamp,
            "L2OutputOracle: cannot propose L2 output in the future"
        );

        require(_outputRoot != bytes32(0), "L2OutputOracle: L2 output proposal cannot be the zero hash");

        if (_l1BlockHash != bytes32(0)) {
            // This check allows the proposer to propose an output based on a given L1 block,
            // without fear that it will be reorged out.
            // It will also revert if the blockheight provided is more than 256 blocks behind the
            // chain tip (as the hash will return as zero). This does open the door to a griefing
            // attack in which the proposer's submission is censored until the block is no longer
            // retrievable, if the proposer is experiencing this attack it can simply leave out the
            // blockhash value, and delay submission until it is confident that the L1 block is
            // finalized.
            require(
                blockhash(_l1BlockNumber) == _l1BlockHash,
                "L2OutputOracle: block hash does not match the hash at the expected height"
            );
        }

        emit OutputProposed(_outputRoot, nextOutputIndex(), _l2BlockNumber, block.timestamp);

        l2Outputs.push(
            Types.OutputProposal({
                outputRoot: _outputRoot,
                timestamp: uint128(block.timestamp),
                l2BlockNumber: uint128(_l2BlockNumber)
            })
        );
    }
```

## 总结
`proposer`的总体实现思路与逻辑相对简单，即定期循环从L1中读取下次需要发送`proposal`的L2区块并与本地L2区块比较，并负责将数据处理并发送到L1当中。其他在提款过程中的其他交易流程大部分由`SDK`负责，可以详细阅读我们之前推送的官方对于提款过程部分的描述([source](https://community.optimism.io/docs/protocol/withdrawal-flow/#))。
如果想要查看在主网中`proposer`的实际行为，可以查看此[proposer address](https://etherscan.io/address/0x473300df21d047806a082244b417f96b32f13a33)