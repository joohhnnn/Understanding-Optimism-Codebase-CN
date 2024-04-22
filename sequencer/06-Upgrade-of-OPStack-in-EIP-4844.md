# OPStack在EIP-4844中的升级

Ethereum的EIP-4844对layer2来说是一次巨大的变革，它显著降低了layer2在使用L1作为DA（数据可用性）的费用。本文不详细解析EIP-4844的具体内容，只简要介绍，作为我们了解OP-Stack更新的背景。

## EIP-4844

TL;DR:
EIP-4844引入了一种称为“blob”的数据格式，这种格式的数据不参与EVM执行，而是存储在共识层，每个数据块的生命周期为4096个epoch（约18天）。blob存在于l1主网上，由新的type3 transaction携带，每个区块最多能容纳6个blob，每个transaction最多可以携带6个blob。

欲了解更多详情，请参考以下文章：

[Ethereum Evolved: Dencun Upgrade Part 5, EIP-4844](https://consensys.io/blog/ethereum-evolved-dencun-upgrade-part-5-eip-4844)

[EIP-4844, Blobs, and Blob Gas: What you need to know](https://www.blocknative.com/blog/eip-4844-blobs-and-blob-gas-what-you-need-to-know)

[Proto-Danksharding](https://notes.ethereum.org/@vbuterin/proto_danksharding_faq#Proto-Danksharding-FAQ)

## OP-Stack的应用

OP-Stack在采用BLOB替换之前的CALLDATA作为rollup的数据存储方式后，费率直线下降
![image](https://hackmd.io/_uploads/BJCSEyngA.png)

**在OP-Stack的此次更新中，主要的业务逻辑变更涉及将原先通过calldata发送的数据转换为blob格式，并通过blob类型的交易发送到L1。此外，还涉及到从L1获取发送到rollup的数据时对blob的解析，以下是参与此次升级的主要组件：**

1. **submitter** —— 负责将rollup数据发送到L1的组件
2. **fetcher**  —— 将L1的数据（旧rollup数据/deposit交易等）同步到L2中
3. **blob相关定义与实现** —— 如何获取和结构blob数据等内容等
4. **其他相关设计部分** —— 如客户端支持blob类型交易的签名、与fault proof相关的设计等

---

⚠️⚠️⚠️请注意，本文中所有涉及的代码均基于最初的PR设计，可能与实际生产环境中运行的代码存在差异。

---

### Blob相关定义与编解码实现

[Pull Request(8131) blob 定义](https://github.com/ethereum-optimism/optimism/pull/8131/files#diff-30107b16d72d6e958093d83b5d736522a7994cab064187562605c82174400cd5)

[Pull Request(8767) encoding & decoding](https://github.com/ethereum-optimism/optimism/commit/78ecdf523026d0afa45c519524a15b83cbe162c8#diff-30107b16d72d6e958093d83b5d736522a7994cab064187562605c82174400cd5R86)

#### 定义blob

```
BlobSize        = 4096 * 32

type Blob [BlobSize]byte
```

#### blob encoding

[**official specs about blob encoding**](https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/derivation.md#blob-encoding)

需要注意的是，此specs对应的是最新版本的代码，而下方PR截取的代码则为最初的简化版本。主要区别在于：Blob类型被分为4096个字段元素，每个字段元素的最大大小受限于特定模的大小，即math.log2(BLS_MODULUS) = 254.8570894...，这意味着每个字段元素的大小不会超过254位，即31.75字节。最初的演示代码只使用了31字节，放弃了0.75字节的空间。而在最新版本的代码中，通过四个字段元素的联合作用，充分利用了每个字段元素的0.75字节空间，从而提高了数据的使用效率。
以下为Pull Request(8767)的部分截取代码
通过4096次循环，它读取总共31*4096字节的数据，这些数据随后被加入到blob中。

```
func (b *Blob) FromData(data Data) error {
	if len(data) > MaxBlobDataSize {
		return fmt.Errorf("data is too large for blob. len=%v", len(data))
	}
	b.Clear()
	// encode 4-byte little-endian length value into topmost 4 bytes (out of 31) of first field
	// element
	binary.LittleEndian.PutUint32(b[1:5], uint32(len(data)))
	// encode first 27 bytes of input data into remaining bytes of first field element
	offset := copy(b[5:32], data)
	// encode (up to) 31 bytes of remaining input data at a time into the subsequent field element
	for i := 1; i < 4096; i++ {
		offset += copy(b[i*32+1:i*32+32], data[offset:])
		if offset == len(data) {
			break
		}
	}
	if offset < len(data) {
		return fmt.Errorf("failed to fit all data into blob. bytes remaining: %v", len(data)-offset)
	}
	return nil
}
```

#### blob decoding

blob数据的解码，原理同上述的数据编码

```
func (b *Blob) ToData() (Data, error) {
	data := make(Data, 4096*32)
	for i := 0; i < 4096; i++ {
		if b[i*32] != 0 {
			return nil, fmt.Errorf("invalid blob, found non-zero high order byte %x of field element %d", b[i*32], i)
		}
		copy(data[i*31:i*31+31], b[i*32+1:i*32+32])
	}
	// extract the length prefix & trim the output accordingly
	dataLen := binary.LittleEndian.Uint32(data[:4])
	data = data[4:]
	if dataLen > uint32(len(data)) {
		return nil, fmt.Errorf("invalid blob, length prefix out of range: %d", dataLen)
	}
	data = data[:dataLen]
	return data, nil
}
```

### Submiter

[Pull Request(8769)](https://github.com/ethereum-optimism/optimism/pull/8769)

#### flag配置

```
switch c.DataAvailabilityType {
case flags.CalldataType:
case flags.BlobsType:
default:
    return fmt.Errorf("unknown data availability type: %v", c.DataAvailabilityType)
}
```

#### BatchSubmitter

BatchSubmitter的功能从之前仅发送calldata数据扩展为根据情况发送calldata或blob类型的数据。Blob类型的数据通过之前提到的FromData（blob-encode）函数在blobTxCandidate内部进行编码

```
func (l *BatchSubmitter) sendTransaction(txdata txData, queue *txmgr.Queue[txData], receiptsCh chan txmgr.TxReceipt[txData]) error {
	// Do the gas estimation offline. A value of 0 will cause the [txmgr] to estimate the gas limit.
	data := txdata.Bytes()

	var candidate *txmgr.TxCandidate
	if l.Config.UseBlobs {
		var err error
		if candidate, err = l.blobTxCandidate(data); err != nil {
			// We could potentially fall through and try a calldata tx instead, but this would
			// likely result in the chain spending more in gas fees than it is tuned for, so best
			// to just fail. We do not expect this error to trigger unless there is a serious bug
			// or configuration issue.
			return fmt.Errorf("could not create blob tx candidate: %w", err)
		}
	} else {
		candidate = l.calldataTxCandidate(data)
	}

	intrinsicGas, err := core.IntrinsicGas(candidate.TxData, nil, false, true, true, false)
	if err != nil {
		// we log instead of return an error here because txmgr can do its own gas estimation
		l.Log.Error("Failed to calculate intrinsic gas", "err", err)
	} else {
		candidate.GasLimit = intrinsicGas
	}

	queue.Send(txdata, *candidate, receiptsCh)
	return nil
}

func (l *BatchSubmitter) blobTxCandidate(data []byte) (*txmgr.TxCandidate, error) {
	var b eth.Blob
	if err := b.FromData(data); err != nil {
		return nil, fmt.Errorf("data could not be converted to blob: %w", err)
	}
	return &txmgr.TxCandidate{
		To:    &l.RollupConfig.BatchInboxAddress,
		Blobs: []*eth.Blob{&b},
	}, nil
}
```

### Fetcher

[Pull Request(9098)](https://github.com/ethereum-optimism/optimism/pull/9098/files#diff-1fd8727490dbd2214b6d0c247eb222f2ac4098d259b4a45e9c0caea7fb2d3e08)

#### GetBlob

GetBlob负责获取blob数据，其主要逻辑包括利用4096个字段元素构建完整的blob，并通过commitment验证构建的blob的正确性。
同时，GetBlob也参与了上层[L1Retrieval中的逻辑流程](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/blob/main/sequencer/04-how-derivation-works.md)。

```
func (p *PreimageOracle) GetBlob(ref eth.L1BlockRef, blobHash eth.IndexedBlobHash) *eth.Blob {
	// Send a hint for the blob commitment & blob field elements.
	blobReqMeta := make([]byte, 16)
	binary.BigEndian.PutUint64(blobReqMeta[0:8], blobHash.Index)
	binary.BigEndian.PutUint64(blobReqMeta[8:16], ref.Time)
	p.hint.Hint(BlobHint(append(blobHash.Hash[:], blobReqMeta...)))

	commitment := p.oracle.Get(preimage.Sha256Key(blobHash.Hash))

	// Reconstruct the full blob from the 4096 field elements.
	blob := eth.Blob{}
	fieldElemKey := make([]byte, 80)
	copy(fieldElemKey[:48], commitment)
	for i := 0; i < params.BlobTxFieldElementsPerBlob; i++ {
		binary.BigEndian.PutUint64(fieldElemKey[72:], uint64(i))
		fieldElement := p.oracle.Get(preimage.BlobKey(crypto.Keccak256(fieldElemKey)))

		copy(blob[i<<5:(i+1)<<5], fieldElement[:])
	}

	blobCommitment, err := blob.ComputeKZGCommitment()
	if err != nil || !bytes.Equal(blobCommitment[:], commitment[:]) {
		panic(fmt.Errorf("invalid blob commitment: %w", err))
	}

	return &blob
}
```

### 其他杂项

除了以上几个主要模块外，还包含例如负责签署type3类型transaction的client sign模块，和fault proof相关涉及的模块，fault proof会在下一章节进行详细描述，这里就不过多赘述了

[Pull Request(5452), fault proof相关](https://github.com/ethereum-optimism/optimism/commit/4739b0f8bfe2f3848af3f1a5661a038c5d602b2f#diff-790daa91002e5c07497fdc2d7c2149b551d77ccec1b1906cc70f575b7c7bad65)

[Pull Request(9182), client sign相关](https://github.com/ethereum-optimism/optimism/pull/9185/files#diff-8046655b02fcced5322724e2cd61ece649a9d79ba09405093f9ed70b2087e47d)

