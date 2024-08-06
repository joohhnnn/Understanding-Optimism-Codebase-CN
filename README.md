# Understanding Optimism Codebase (CN)

此文档为对Optimism的codebase进行全面讲解，意在帮助新来到Optimism的开发者快速上手，和真正了解在codebase的代码流中是怎么工作的。

## 项目目录

### 已完成：

- [**00-sequencer如何产生一个新的L2区块**](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/blob/main/sequencer/00-how-sequencer-generates-L2-blocks.md)
- [**01-区块是如何同步的**](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/blob/main/sequencer/01-how-block-sync.md)
- [**02-libp2p在op-stack中的使用**](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/blob/main/sequencer/02-how-optimism-use-libp2p.md)
- [**03-op-batcher工作原理**](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/blob/main/sequencer/03-how-batcher-works.md)
- [**04-L2派生（derivation）原理**](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/blob/main/sequencer/04-how-derivation-works.md)
- [**05-op-proposer工作原理**](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/blob/main/sequencer/05-how-proposer-works.md)
- [**06-OP-Stack在EIP-4844中的升级**](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/blob/main/sequencer/06-Upgrade-of-OPStack-in-EIP-4844.md)
- [**07-什么是fault-proof**](https://github.com/joohhnnn/The-book-of-optimism-fault-proof-CN/blob/main/01-what-is-fault-proof.md)
- [**08-Fault-Dispute-Game**](https://github.com/joohhnnn/The-book-of-optimism-fault-proof-CN/blob/main/02-fault-dispute-game.md)
- [**09-Cannon**](https://github.com/joohhnnn/The-book-of-optimism-fault-proof-CN/blob/main/03-cannon.md)
- [**10-op-program**](https://github.com/joohhnnn/The-book-of-optimism-fault-proof-CN/blob/main/04-op-program.md)
- [**11-op-challenger**](https://github.com/joohhnnn/The-book-of-optimism-fault-proof-CN/blob/main/05-op-challenger.md)

---

### [The-book-of-optimism-fault-proof](https://github.com/joohhnnn/The-book-of-optimism-fault-proof)

---

### 待开始：

- [op-e2e](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-e2e): 使用Go进行所有Bedrock组件的端到端测试
- [op-heartbeat](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-heartbeat): 心跳监控服务
- [op-service](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-service): 通用代码库实用程序
- [op-wheel](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-wheel): 数据库实用程序

## 联系信息

若您有任何疑惑，或需在开发公共产品方面获得协助，请随时通过电子邮件 [joohhnnn8@gmail.com](mailto:joohhnnn8@gmail.com) 与我联系。如我有空闲时间，将十分乐意提供帮助。

