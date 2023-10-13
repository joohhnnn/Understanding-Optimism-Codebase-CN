# optimism中的libp2p应用

在本节中，主要用于讲解optimism是如何使用libp2p来完成op-node中的p2p网络建立的。 
p2p网络主要是用于在不同的node中传递信息，例如sequencer完成unsafe的区块构建后，通过p2p的gossiphub的pub/sub进行传播。libp2p还处理了其他，例如网络，寻址等在p2p网络中的基础件层。

## 了解libp2p

libp2p（简称来自“库对等”或“library peer-to-peer”）是一个面向对等（P2P）网络的框架，能够帮助开发P2P应用程序。它包含了一套协议、规范和库，使网络参与者（也称为“对等体”或“peers”）之间的P2P通信变得更为简便 ([source](https://docs.libp2p.io/introduction/what-is-libp2p))。libp2p最初是作为IPFS（InterPlanetary File System，星际文件系统）项目的一部分，后来它演变成了一个独立的项目，成为了分布式网络的模块化网络堆栈 ([source](https://proto.school/introduction-to-libp2p))。

libp2p是IPFS社区的一个开源项目，欢迎广泛社区的贡献，包括帮助编写规范、编码实现以及创建示例和教程 ([source](https://libp2p.io/))。libp2p是由多个构建模块组成的，每个模块都有非常明确、有文档记录且经过测试的接口，使得它们可组合、可替换，因此可升级 ([source](https://medium.com/@mycoralhealth/understanding-ipfs-in-depth-5-6-what-is-libp2p-fd42ed27e656))。libp2p的模块化特性使得开发人员可以选择并使用仅对他们的应用程序必要的组件，从而在构建P2P网络应用程序时促进了灵活性和效率。

### 相关资源
- [libp2p的官方文档](https://docs.libp2p.io/)
- [libp2p的GitHub仓库](https://github.com/libp2p)
- [在ProtoSchool上的libp2p简介](https://proto.school/introduction-to-libp2p)

libp2p的模块化架构和开源特性为开发强大、可扩展和灵活的P2P应用程序提供了良好的环境，使其成为分布式网络和网络应用程序开发领域的重要参与者。


### libp2p实现方式

在使用libp2p时，你会需要实现和配置一些核心组件以构建你的P2P网络。以下是libp2p在应用中的一些主要实现方面：

#### 1. **节点创建与配置**:
   - 创建和配置libp2p节点是最基本的步骤，这包括设置节点的网络地址、身份和其他基本参数。
   关键使用代码:
   ```go
   libp2p.New()
   ```

#### 2. **传输协议**:
   - 选择和配置你的传输协议（例如TCP、WebSockets等）以确保节点之间的通信。
   关键使用代码：
   ```go
   tcpTransport := tcp.NewTCPTransport()
   ```

#### 3. **多路复用和流控制**:
   - 实现多路复用来允许在单一的连接上处理多个并发的数据流。
   - 实现流量控制来管理数据的传输速率和处理速率。
   关键使用代码：
   ```go
   yamuxTransport := yamux.New()
   ```

#### 4. **安全和加密**:
   - 配置安全传输层以确保通信的安全性和隐私。
   - 实现加密和身份验证机制以保护数据和验证通信方。
   关键使用代码：
   ```go
   tlsTransport := tls.New()
   ```

#### 5. **协议和消息处理**:
   - 定义和实现自定义协议来处理特定的网络操作和消息交换。
   - 处理接收到的消息并根据需要发送响应。
   关键使用代码：
   ```go
   host.SetStreamHandler("/my-protocol/1.0.0", myProtocolHandler)
   ```

#### 6. **发现和路由**:
   - 实现节点发现机制来找到网络中的其他节点。
   - 实现路由逻辑以确定如何将消息路由到网络中的正确节点。
   关键使用代码：
   ```go
   dht := kaddht.NewDHT(ctx, host, datastore.NewMapDatastore())
   ```

#### 7. **网络行为和策略**:
   - 定义和实现网络的行为和策略，例如连接管理、错误处理和重试逻辑。
   关键使用代码：
   ```go
   connManager := connmgr.NewConnManager(lowWater, highWater, gracePeriod)
   ```

#### 8. **状态管理和存储**:
   - 管理节点和网络的状态，包括连接状态、节点列表和数据存储。
   关键使用代码：
   ```go
   peerstore := pstoremem.NewPeerstore()
   ```

#### 9. **测试和调试**:
   - 为你的libp2p应用编写测试以确保其正确性和可靠性。
   - 使用调试工具和日志来诊断和解决网络问题。
   关键使用代码：
   ```go
   logging.SetLogLevel("libp2p", "DEBUG")
   ```

#### 10. **文档和社区支持**:
    - 查阅libp2p的文档以了解其各种组件和API。
    - 与libp2p社区交流以获取支持和解决问题。
}


以上是使用libp2p时需要考虑和实现的一些主要方面。每个项目的具体实现可能会有所不同，但这些基本方面是构建和运行libp2p应用所必需的。在实现这些功能时，可以参考libp2p的[官方文档](https://docs.libp2p.io/)和[GitHub仓库](https://github.com/libp2p/go-libp2p/tree/master/examples)中的示例代码和教程。


## 在OP-node中libp2p的使用

为了弄清楚op-node和libp2p的关系，我们必须弄清楚几个问题

    - 为什么选择libp2p？为什么不选择devp2p（geth使用devp2p）
    - OP-node有哪些数据或者流程和p2p网络紧密相关
    - 这些功能是如何在代码层实现的 

### op-node需要libp2p网络的原因

**首先我们要了解为什么optimism需要p2p网络**
libp2p是一个模块化的网络协议，允许开发人员构建去中心化的点对点应用，适用于多种用例 ([source](https://ethereum.stackexchange.com/questions/12290/what-is-the-distinction-between-libp2p-devp2p-and-rlpx))([source](https://www.geeksforgeeks.org/what-is-the-difference-between-libp2p-devp2p-and-rlpx/))。而devp2p主要用于以太坊生态系统，专为以太坊应用定制 ([source](https://docs.libp2p.io/concepts/similar-projects/devp2p/))。libp2p的灵活性和广泛适用性可能使其成为开发人员的首选。

### op-node主要使用libp2p的功能点

    - 用于sequencer将产生的unsafe的block传递到其他非sequencer节点
    - 用于非sequencer模式下的其他节点当出现gap时进行快速同步（反向链同步）
    - 用于采用积分声誉系统来规范整体节点的良好环境

### 代码实现

#### host自定义初始化

host可以理解为是p2p的节点，当开启这个节点的时候，需要针对自己的项目进行一些特殊的初始化配置

现在让我们看一下 `op-node/p2p/host.go`文件中的`Host`方法，

该函数主要用于设置 libp2p 主机并进行各种配置。以下是该函数的关键部分以及各部分的简单中文描述：

1. **检查是否禁用 P2P**  
   如果 P2P 被禁用，函数会直接返回。

2. **从公钥获取 Peer ID**  
   使用配置中的公钥来生成 Peer ID。

3. **初始化 Peerstore**  
   创建一个基础的 Peerstore 存储。

4. **初始化扩展 Peerstore**  
   在基础 Peerstore 的基础上，创建一个扩展的 Peerstore。

5. **将私钥和公钥添加到 Peerstore**  
   在 Peerstore 中存储 Peer 的私钥和公钥。

6. **初始化连接控制器（Connection Gater）**  
   用于控制网络连接。

7. **初始化连接管理器（Connection Manager）**  
   用于管理网络连接。

8. **设置传输和监听地址**  
   设置网络传输协议和主机的监听地址。

9. **创建 libp2p 主机**  
   使用前面的所有设置来创建一个新的 libp2p 主机。

10. **初始化静态 Peer**  
    如果有配置静态 Peer，进行初始化。

11. **返回主机**  
    最后，函数返回创建好的 libp2p 主机。

这些关键部分负责 libp2p 主机的初始化和设置，每个部分都负责主机配置的一个特定方面。


```go
    func (conf *Config) Host(log log.Logger, reporter metrics.Reporter, metrics HostMetrics) (host.Host, error) {
        if conf.DisableP2P {
            return nil, nil
        }
        pub := conf.Priv.GetPublic()
        pid, err := peer.IDFromPublicKey(pub)
        if err != nil {
            return nil, fmt.Errorf("failed to derive pubkey from network priv key: %w", err)
        }

        basePs, err := pstoreds.NewPeerstore(context.Background(), conf.Store, pstoreds.DefaultOpts())
        if err != nil {
            return nil, fmt.Errorf("failed to open peerstore: %w", err)
        }

        peerScoreParams := conf.PeerScoringParams()
        var scoreRetention time.Duration
        if peerScoreParams != nil {
            // Use the same retention period as gossip will if available
            scoreRetention = peerScoreParams.PeerScoring.RetainScore
        } else {
            // Disable score GC if peer scoring is disabled
            scoreRetention = 0
        }
        ps, err := store.NewExtendedPeerstore(context.Background(), log, clock.SystemClock, basePs, conf.Store, scoreRetention)
        if err != nil {
            return nil, fmt.Errorf("failed to open extended peerstore: %w", err)
        }

        if err := ps.AddPrivKey(pid, conf.Priv); err != nil {
            return nil, fmt.Errorf("failed to set up peerstore with priv key: %w", err)
        }
        if err := ps.AddPubKey(pid, pub); err != nil {
            return nil, fmt.Errorf("failed to set up peerstore with pub key: %w", err)
        }

        var connGtr gating.BlockingConnectionGater
        connGtr, err = gating.NewBlockingConnectionGater(conf.Store)
        if err != nil {
            return nil, fmt.Errorf("failed to open connection gater: %w", err)
        }
        connGtr = gating.AddBanExpiry(connGtr, ps, log, clock.SystemClock, metrics)
        connGtr = gating.AddMetering(connGtr, metrics)

        connMngr, err := DefaultConnManager(conf)
        if err != nil {
            return nil, fmt.Errorf("failed to open connection manager: %w", err)
        }

        listenAddr, err := addrFromIPAndPort(conf.ListenIP, conf.ListenTCPPort)
        if err != nil {
            return nil, fmt.Errorf("failed to make listen addr: %w", err)
        }
        tcpTransport := libp2p.Transport(
            tcp.NewTCPTransport,
            tcp.WithConnectionTimeout(time.Minute*60)) // break unused connections
        // TODO: technically we can also run the node on websocket and QUIC transports. Maybe in the future?

        var nat lconf.NATManagerC // disabled if nil
        if conf.NAT {
            nat = basichost.NewNATManager
        }

        opts := []libp2p.Option{
            libp2p.Identity(conf.Priv),
            // Explicitly set the user-agent, so we can differentiate from other Go libp2p users.
            libp2p.UserAgent(conf.UserAgent),
            tcpTransport,
            libp2p.WithDialTimeout(conf.TimeoutDial),
            // No relay services, direct connections between peers only.
            libp2p.DisableRelay(),
            // host will start and listen to network directly after construction from config.
            libp2p.ListenAddrs(listenAddr),
            libp2p.ConnectionGater(connGtr),
            libp2p.ConnectionManager(connMngr),
            //libp2p.ResourceManager(nil), // TODO use resource manager interface to manage resources per peer better.
            libp2p.NATManager(nat),
            libp2p.Peerstore(ps),
            libp2p.BandwidthReporter(reporter), // may be nil if disabled
            libp2p.MultiaddrResolver(madns.DefaultResolver),
            // Ping is a small built-in libp2p protocol that helps us check/debug latency between peers.
            libp2p.Ping(true),
            // Help peers with their NAT reachability status, but throttle to avoid too much work.
            libp2p.EnableNATService(),
            libp2p.AutoNATServiceRateLimit(10, 5, time.Second*60),
        }
        opts = append(opts, conf.HostMux...)
        if conf.NoTransportSecurity {
            opts = append(opts, libp2p.Security(insecure.ID, insecure.NewWithIdentity))
        } else {
            opts = append(opts, conf.HostSecurity...)
        }
        h, err := libp2p.New(opts...)
        if err != nil {
            return nil, err
        }

        staticPeers := make([]*peer.AddrInfo, len(conf.StaticPeers))
        for i, peerAddr := range conf.StaticPeers {
            addr, err := peer.AddrInfoFromP2pAddr(peerAddr)
            if err != nil {
                return nil, fmt.Errorf("bad peer address: %w", err)
            }
            staticPeers[i] = addr
        }

        out := &extraHost{
            Host:        h,
            connMgr:     connMngr,
            log:         log,
            staticPeers: staticPeers,
            quitC:       make(chan struct{}),
        }
        out.initStaticPeers()
        if len(conf.StaticPeers) > 0 {
            go out.monitorStaticPeers()
        }

        out.gater = connGtr
        return out, nil
    }
```

#### gossip下的区块传播

gossip在分布式系统中用于确保数据一致性，并修复由多播引起的问题。它是一种通信协议，其中信息从一个或多个节点发送到网络中的其他节点集，当网络中的一组客户端同时需要相同的数据时，这会很有用。当sequencer产生出unsafe状态的区块的时候，就是通过gossip网络传递给其他节点的。

首先让我们来看看节点是在哪里加入gossip网络的，
`op-node/p2p/node.go`中的`init`方法，在节点初始化的时候，调用JoinGossip方法加入了gossip网络

```go
    func (n *NodeP2P) init(resourcesCtx context.Context, rollupCfg *rollup.Config, log log.Logger, setup SetupP2P, gossipIn GossipIn, l2Chain L2Chain, runCfg GossipRuntimeConfig, metrics metrics.Metricer) error {
            …
            // note: the IDDelta functionality was removed from libP2P, and no longer needs to be explicitly disabled.
            n.gs, err = NewGossipSub(resourcesCtx, n.host, rollupCfg, setup, n.scorer, metrics, log)
            if err != nil {
                return fmt.Errorf("failed to start gossipsub router: %w", err)
            }
            n.gsOut, err = JoinGossip(resourcesCtx, n.host.ID(), n.gs, log, rollupCfg, runCfg, gossipIn)
            …
    }
```

接下来来到`op-node/p2p/gossip.go`中

以下是 `JoinGossip` 函数中执行的主要操作的简单概述：

1. **验证器创建**：
   - `val` 被赋予 `guardGossipValidator` 函数调用的结果，目的是为八卦消息创建验证器，该验证器检查网络中传播的区块的有效性。

2. **区块主题名称生成**：
   - 使用 `blocksTopicV1` 函数生成 `blocksTopicName`，该函数根据配置（`cfg`）中的 `L2ChainID` 格式化字符串。格式化的字符串遵循特定的结构：`/optimism/{L2ChainID}/0/blocks`。

3. **主题验证器注册**：
   - 调用 `ps` 的 `RegisterTopicValidator` 方法，以将 `val` 注册为区块主题的验证器。还指定了一些验证器的配置选项，例如3秒的超时和4的并发级别。

4. **加入主题**：
   - 函数通过调用 `ps.Join(blocksTopicName)` 尝试加入区块八卦主题。如果出现错误，它将返回一个错误消息，指示无法加入主题。

5. **事件处理器创建**：
   - 通过调用 `blocksTopic.EventHandler()` 尝试为区块主题创建事件处理器。如果出现错误，它将返回一个错误消息，指示无法创建处理器。

6. **记录主题事件**：
   - 生成了一个新的goroutine来使用 `LogTopicEvents` 函数记录主题事件。

7. **主题订阅**：
   - 函数通过调用 `blocksTopic.Subscribe()` 尝试订阅区块八卦主题。如果出现错误，它将返回一个错误消息，指示无法订阅。

8. **订阅者创建**：
   - 使用 `MakeSubscriber` 函数创建了一个 `subscriber`，该函数封装了一个 `BlocksHandler`，该处理器处理来自 `gossipIn` 的 `OnUnsafeL2Payload` 事件。生成了一个新的goroutine来运行提供的 `subscription`。

9. **创建并返回发布者**：
   - 创建了一个 `publisher` 实例并返回，该实例配置为使用提供的配置和区块主题。

```go
    func JoinGossip(p2pCtx context.Context, self peer.ID, ps *pubsub.PubSub, log log.Logger, cfg *rollup.Config, runCfg GossipRuntimeConfig, gossipIn GossipIn) (GossipOut, error) {
        val := guardGossipValidator(log, logValidationResult(self, "validated block", log, BuildBlocksValidator(log, cfg, runCfg)))
        blocksTopicName := blocksTopicV1(cfg) // return fmt.Sprintf("/optimism/%s/0/blocks", cfg.L2ChainID.String())
        err := ps.RegisterTopicValidator(blocksTopicName,
            val,
            pubsub.WithValidatorTimeout(3*time.Second),
            pubsub.WithValidatorConcurrency(4))
        if err != nil {	
            return nil, fmt.Errorf("failed to register blocks gossip topic: %w", err)
        }
        blocksTopic, err := ps.Join(blocksTopicName)
        if err != nil {
            return nil, fmt.Errorf("failed to join blocks gossip topic: %w", err)
        }
        blocksTopicEvents, err := blocksTopic.EventHandler()
        if err != nil {
            return nil, fmt.Errorf("failed to create blocks gossip topic handler: %w", err)
        }
        go LogTopicEvents(p2pCtx, log.New("topic", "blocks"), blocksTopicEvents)

        subscription, err := blocksTopic.Subscribe()
        if err != nil {
            return nil, fmt.Errorf("failed to subscribe to blocks gossip topic: %w", err)
        }

        subscriber := MakeSubscriber(log, BlocksHandler(gossipIn.OnUnsafeL2Payload))
        go subscriber(p2pCtx, subscription)

        return &publisher{log: log, cfg: cfg, blocksTopic: blocksTopic, runCfg: runCfg}, nil
    }
```

这样，一个非sequencer节点的订阅就已经建立了，接下来让我们把目光移到sequencer模式的节点当中，然后看看他是如果将区块广播出去的。

`op-node/rollup/driver/state.go`

在eventloop中通过循环来等待sequencer模式中新的payload的产生（unsafe区块），然后将这个payload通过PublishL2Payload传播到gossip网络中

    func (s *Driver) eventLoop() {
        …
        for(){
            …
            select {
            case <-sequencerCh:
                payload, err := s.sequencer.RunNextSequencerAction(ctx)
                if err != nil {
                    s.log.Error("Sequencer critical error", "err", err)
                    return
                }
                if s.network != nil && payload != nil {
                    // Publishing of unsafe data via p2p is optional.
                    // Errors are not severe enough to change/halt sequencing but should be logged and metered.
                    if err := s.network.PublishL2Payload(ctx, payload); err != nil {
                        s.log.Warn("failed to publish newly created block", "id", payload.ID(), "err", err)
                        s.metrics.RecordPublishingError()
                    }
                }
                planSequencerAction() // schedule the next sequencer action to keep the sequencing looping
                …
                }
        …
        }
        …
    }

这样，一个新的payload的就进入到gossip网络中了。

在libp2p的pubsub系统中，节点首先从其他节点接收消息，然后检查消息的有效性。如果消息有效并且符合节点的订阅标准，节点会考虑将其转发给其他节点。基于某些策略，如网络拓扑和节点的订阅情况，节点会决定是否将消息转发给其它节点。如果决定转发，节点会将消息发送给与其连接并订阅了相同主题的所有节点。在转发过程中，为防止消息在网络中无限循环，通常会有机制来跟踪已转发的消息，并确保不会多次转发同一消息。同时，消息可能具有“生存时间”（TTL）属性，定义了消息可以在网络中转发的次数或时间，每当消息被转发时，TTL值都会递减，直到消息不再被转发为止。在验证方面，消息通常会通过一些验证过程，例如检查消息的签名和格式，以确保消息的完整性和真实性。在libp2p的pubsub模型中，这个过程确保了消息能够广泛传播到网络中的许多节点，同时避免了无限循环和网络拥塞，实现了有效的消息传递和处理。


#### 当存在缺失区块，通过p2p快速同步

当节点因为特殊情况，比如宕机后重新链接，可能会产生一些没有同步上的区块（gaps），当遇到这种情况时，可以通过p2p网络的反向链的方式快速同步。

我们来看一下`op-node/rollup/driver/state.go`中的`checkForGapInUnsafeQueue`函数

该代码段定义了一个名为 `checkForGapInUnsafeQueue` 的方法，属于 `Driver` 结构体。它的目的是检查一个名为 "unsafe queue" 的队列中是否存在数据缺口，并尝试通过一个名为 `altSync` 的备用同步方法来检索缺失的负载。这里的关键点是，该方法是为了确保数据的连续性，并在检测到数据缺失时尝试从其他同步方法中检索缺失的数据。以下是函数的主要步骤：

1. 函数首先从 `s.derivation` 中获取 `UnsafeL2Head` 和 `UnsafeL2SyncTarget` 作为检查范围的起始和结束点。
2. 函数检查在 `start` 和 `end` 之间是否存在缺失的数据块，这是通过比较 `end` 和 `start` 的 `Number` 值来完成的。
3. 如果检测到数据缺口，函数会通过调用 `s.altSync.RequestL2Range(ctx, start, end)` 来请求缺失的数据范围。如果 `end` 是一个空引用（即 `eth.L2BlockRef{}`），函数将请求一个开放结束范围的同步，从 `start` 开始。
4. 在请求数据时，函数会记录一个调试日志，说明它正在请求哪个范围的数据。
5. 函数最终返回一个错误值。如果没有错误，它会返回 `nil`

```go
    // checkForGapInUnsafeQueue checks if there is a gap in the unsafe queue and attempts to retrieve the missing payloads from an alt-sync method.
    // WARNING: This is only an outgoing signal, the blocks are not guaranteed to be retrieved.
    // Results are received through OnUnsafeL2Payload.
    func (s *Driver) checkForGapInUnsafeQueue(ctx context.Context) error {
        start := s.derivation.UnsafeL2Head()
        end := s.derivation.UnsafeL2SyncTarget()
        // Check if we have missing blocks between the start and end. Request them if we do.
        if end == (eth.L2BlockRef{}) {
            s.log.Debug("requesting sync with open-end range", "start", start)
            return s.altSync.RequestL2Range(ctx, start, eth.L2BlockRef{})
        } else if end.Number > start.Number+1 {
            s.log.Debug("requesting missing unsafe L2 block range", "start", start, "end", end, "size", end.Number-start.Number)
            return s.altSync.RequestL2Range(ctx, start, end)
        }
        return nil
    }
```

`RequestL2Range`函数向`requests`通道里传递请求区块的开始和结束信号。

然后通过`onRangeRequest`方法来对请求向`peerRequests`通道分发，`peerRequests`通道会被多个peer开启的loop所等待，即每一次分发都只有一个peer会去处理这个request。
```go
    func (s *SyncClient) onRangeRequest(ctx context.Context, req rangeRequest) {
            …
            for i := uint64(0); ; i++ {
            num := req.end.Number - 1 - i
            if num <= req.start {
                return
            }
            // check if we have something in quarantine already
            if h, ok := s.quarantineByNum[num]; ok {
                if s.trusted.Contains(h) { // if we trust it, try to promote it.
                    s.tryPromote(h)
                }
                // Don't fetch things that we have a candidate for already.
                // We'll evict it from quarantine by finding a conflict, or if we sync enough other blocks
                continue
            }

            if _, ok := s.inFlight[num]; ok {
                log.Debug("request still in-flight, not rescheduling sync request", "num", num)
                continue // request still in flight
            }
            pr := peerRequest{num: num, complete: new(atomic.Bool)}

            log.Debug("Scheduling P2P block request", "num", num)
            // schedule number
            select {
            case s.peerRequests <- pr:
                s.inFlight[num] = pr.complete
            case <-ctx.Done():
                log.Info("did not schedule full P2P sync range", "current", num, "err", ctx.Err())
                return
            default: // peers may all be busy processing requests already
                log.Info("no peers ready to handle block requests for more P2P requests for L2 block history", "current", num)
                return
            }
        }
    }
```

接下来我们看看，当peer收到这个request的时候会怎么处理。

首先我们要知道的是，peer和请求节点之间的链接，或者消息传递是通过libp2p的stream来传递的。stream的处理方法由接收peer节点实现，stream的创建由发送节点来开启。

我们可以在之前的init函数中看到这样的代码，这里MakeStreamHandler返回了一个处理函数，SetStreamHandler将协议id和这个处理函数绑定，因此，每当发送节点创建并使用这个stream的时候，都会触发返回的处理函数。
    
```go
    n.syncSrv = NewReqRespServer(rollupCfg, l2Chain, metrics)
    // register the sync protocol with libp2p host
    payloadByNumber := MakeStreamHandler(resourcesCtx, log.New("serve", "payloads_by_number"), n.syncSrv.HandleSyncRequest)
    n.host.SetStreamHandler(PayloadByNumberProtocolID(rollupCfg.L2ChainID), payloadByNumber)
```

接下来让我们看看处理函数里面是如何处理的
函数首先进行全局和个人的速率限制检查，以控制处理请求的速度。然后，它读取并验证了请求的区块号，确保它在合理的范围内。之后，函数从 L2 层获取请求的区块负载，并将其写入到响应流中。在写入响应数据时，它设置了写入截止时间，以避免在写入过程中被慢速的 peer 连接阻塞。最终，函数返回请求的区块号和可能的错误。

```go
    func (srv *ReqRespServer) handleSyncRequest(ctx context.Context, stream network.Stream) (uint64, error) {
        peerId := stream.Conn().RemotePeer()

        // take a token from the global rate-limiter,
        // to make sure there's not too much concurrent server work between different peers.
        if err := srv.globalRequestsRL.Wait(ctx); err != nil {
            return 0, fmt.Errorf("timed out waiting for global sync rate limit: %w", err)
        }

        // find rate limiting data of peer, or add otherwise
        srv.peerStatsLock.Lock()
        ps, _ := srv.peerRateLimits.Get(peerId)
        if ps == nil {
            ps = &peerStat{
                Requests: rate.NewLimiter(peerServerBlocksRateLimit, peerServerBlocksBurst),
            }
            srv.peerRateLimits.Add(peerId, ps)
            ps.Requests.Reserve() // count the hit, but make it delay the next request rather than immediately waiting
        } else {
            // Only wait if it's an existing peer, otherwise the instant rate-limit Wait call always errors.

            // If the requester thinks we're taking too long, then it's their problem and they can disconnect.
            // We'll disconnect ourselves only when failing to read/write,
            // if the work is invalid (range validation), or when individual sub tasks timeout.
            if err := ps.Requests.Wait(ctx); err != nil {
                return 0, fmt.Errorf("timed out waiting for global sync rate limit: %w", err)
            }
        }
        srv.peerStatsLock.Unlock()

        // Set read deadline, if available
        _ = stream.SetReadDeadline(time.Now().Add(serverReadRequestTimeout))

        // Read the request
        var req uint64
        if err := binary.Read(stream, binary.LittleEndian, &req); err != nil {
            return 0, fmt.Errorf("failed to read requested block number: %w", err)
        }
        if err := stream.CloseRead(); err != nil {
            return req, fmt.Errorf("failed to close reading-side of a P2P sync request call: %w", err)
        }

        // Check the request is within the expected range of blocks
        if req < srv.cfg.Genesis.L2.Number {
            return req, fmt.Errorf("cannot serve request for L2 block %d before genesis %d: %w", req, srv.cfg.Genesis.L2.Number, invalidRequestErr)
        }
        max, err := srv.cfg.TargetBlockNumber(uint64(time.Now().Unix()))
        if err != nil {
            return req, fmt.Errorf("cannot determine max target block number to verify request: %w", invalidRequestErr)
        }
        if req > max {
            return req, fmt.Errorf("cannot serve request for L2 block %d after max expected block (%v): %w", req, max, invalidRequestErr)
        }

        payload, err := srv.l2.PayloadByNumber(ctx, req)
        if err != nil {
            if errors.Is(err, ethereum.NotFound) {
                return req, fmt.Errorf("peer requested unknown block by number: %w", err)
            } else {
                return req, fmt.Errorf("failed to retrieve payload to serve to peer: %w", err)
            }
        }

        // We set write deadline, if available, to safely write without blocking on a throttling peer connection
        _ = stream.SetWriteDeadline(time.Now().Add(serverWriteChunkTimeout))

        // 0 - resultCode: success = 0
        // 1:5 - version: 0
        var tmp [5]byte
        if _, err := stream.Write(tmp[:]); err != nil {
            return req, fmt.Errorf("failed to write response header data: %w", err)
        }
        w := snappy.NewBufferedWriter(stream)
        if _, err := payload.MarshalSSZ(w); err != nil {
            return req, fmt.Errorf("failed to write payload to sync response: %w", err)
        }
        if err := w.Close(); err != nil {
            return req, fmt.Errorf("failed to finishing writing payload to sync response: %w", err)
        }
        return req, nil
    }
```

至此，反向链同步请求和处理的大致流程已经讲解完毕

#### p2p节点中的积分声誉系统

为了防止某些节点进行恶意的请求与响应来破坏整个网络的安全性，optimism还使用了一套积分系统。

例如在`op-node/p2p/app_scores.go` 中存在一系列函数对peer的分数进行设置

```go
    func (s *peerApplicationScorer) onValidResponse(id peer.ID) {
        _, err := s.scorebook.SetScore(id, store.IncrementValidResponses{Cap: s.params.ValidResponseCap})
        if err != nil {
            s.log.Error("Unable to update peer score", "peer", id, "err", err)
            return
        }
    }

    func (s *peerApplicationScorer) onResponseError(id peer.ID) {
        _, err := s.scorebook.SetScore(id, store.IncrementErrorResponses{Cap: s.params.ErrorResponseCap})
        if err != nil {
            s.log.Error("Unable to update peer score", "peer", id, "err", err)
            return
        }
    }

    func (s *peerApplicationScorer) onRejectedPayload(id peer.ID) {
        _, err := s.scorebook.SetScore(id, store.IncrementRejectedPayloads{Cap: s.params.RejectedPayloadCap})
        if err != nil {
            s.log.Error("Unable to update peer score", "peer", id, "err", err)
            return
        }
    }
```

然后在添加新的节点前会检查其积分情况

```go
    func AddScoring(gater BlockingConnectionGater, scores Scores, minScore float64) *ScoringConnectionGater {
        return &ScoringConnectionGater{BlockingConnectionGater: gater, scores: scores, minScore: minScore}
    }

    func (g *ScoringConnectionGater) checkScore(p peer.ID) (allow bool) {
        score, err := g.scores.GetPeerScore(p)
        if err != nil {
            return false
        }
        return score >= g.minScore
    }

    func (g *ScoringConnectionGater) InterceptPeerDial(p peer.ID) (allow bool) {
        return g.BlockingConnectionGater.InterceptPeerDial(p) && g.checkScore(p)
    }

    func (g *ScoringConnectionGater) InterceptAddrDial(id peer.ID, ma multiaddr.Multiaddr) (allow bool) {
        return g.BlockingConnectionGater.InterceptAddrDial(id, ma) && g.checkScore(id)
    }

    func (g *ScoringConnectionGater) InterceptSecured(dir network.Direction, id peer.ID, mas network.ConnMultiaddrs) (allow bool) {
        return g.BlockingConnectionGater.InterceptSecured(dir, id, mas) && g.checkScore(id)
    }
```

### 总结
libp2p的高度可配置性使得整个项目的p2p具有高度的可自定义化和模块话，以上是optimsim对libp2p进行个性化实现的主要逻辑，还有其他细节可以在p2p目录下通过阅读源码的方式来详细学习。