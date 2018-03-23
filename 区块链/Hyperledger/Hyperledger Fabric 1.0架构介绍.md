

#  简介
本文主要介绍了Fabric1.0中的重大变化和架构。
Fabric1.0版本中，把节点分为peers节点（维护state、ledger）和orderers节点（负责对ledger中的transactions达成共识）。在Fabric0.6和之前的版本中，没有这一概念。

介绍了Endorsing peers，它作为一类特殊的peers，负责同时执行chaincode和endorsing transactions。
# 新架构的优点
新架构有下面的优点：
*  Chaincode信任的灵活性。

该架构将Chaincode的信任假设和ordering的信任假设分离开。
也就是说，ordering service可以被一组节点提供，并且容忍其中的一些节点fail或者出现异常；同时对于每一个Chaincode，endorsers都可以是不同的。

联系实际的使用场景，多个组织加入了同一个区块链网络中，并且各自提供了自己机器作为网络中的各种节点，这些节点可以配置成peers节点，也可以配置为orderers节点，是分散的。

如果网络中的所有peers和为orderers节点都在同一个组织中，如果这个组织的集群出现故障，那么会影响整个区块链网络，或者这个组织作弊，那么其它节点很难发现。

*  可扩展性

**当不同的Chaincodes指定了不关联的endorsers，可以在endorsers间对Chaincode进行分区，并允许Chaincode的并行执行。**
同时，Chaincode的执行可能消耗很大，把它从ordering service这个关键路径中移除是明智的。

* 保密性

该体系结构有助于部署，对于其交易的内容和状态更新具有保密要求的Chaincode。

* 共识机制模块化

该架构师模块化的，允许可插拔的共识机制实现（例如ordering service）

下面要介绍的内容，一部分是1.0版本中的，一部分是1.0版本以后的。

# 目录
Part I: Hyperledger Fabric v1架构相关的概念

* 系统架构

* Transaction endorsement的基本工作流

* Endorsement policies

Part II: 1.0版本后续架构的概念

* Ledger checkpointing (pruning)

## 1.系统架构

Blockchain运行的程序叫Chaincode。Chaincode是核心元素，因为transactions是对Chaincode上的操作的调用。
Transations 必须被“endorsed”，只有endorsed的transactions（被认可的交易）才能被提交。
会存在一个或多个特殊的Chaincode，来管理函数和参数，统称为system Chaincode。

## 1.1. Transactions
Transactions 有以下两种类型:

部署 transactions：创建新的chaincode，并把程序作为参数。 当一个部署 transaction 执行成功后, chaincode就被安装在了blockchain上。

调用 transactions：在上一步部署好的Chaincode上执行操作。Chaincode执行特定的函数，这个函数可能调用对相应状态的修改，并返回结果。

后面会介绍到，部署transactions是调用transactions的特殊情况，创建新Chaincode的部署transactions，对应于System Chaincode上的调用transaction。

注：本文没有介绍: 
a) query (read-only) transactions 的优化 (包含在1.0版本中)。
b) 支持cross-chaincode的transactions (1.0后续版本中的特性。

## 1.2. Blockchain 数据结构
### 1.2.1. State（状态）
blockchain的状态是版本化的 key/value store (KVS), keys 是名字，values是任意的 blobs，版本号标识这条记录的版本。 这些数据项，由Chaincode通过put和get KVS-操作来管理. state是持久化存储的，对state的更新是被log记录的。
更具体点，state的模型是一个mapping K -> (V X N), :

* K 是一组keys
* V 是一组values
* N is 一组无限的，有序的版本号. Injective函数 `next: N -> N` 根据N 返回下一个版本号。
`V` 和 `N` 都包含一个特殊的元素 `\bot`, 这在N是最低元素的情况下
. 初始情况下，所有的 keys 都映射为 `(\bot,\bot)`。对于` s(k)=(v,ver) ` 我们使用`s(k).value`表示 `v`, 使用` s(k).version`表示 `ver`。

KVS 操作具体如下:

* `put(k,v)`, 把 blockchain 的状态` s` 变成` s'`，像下面这样 `s'(k)=(v,next(s(k).version))` 
* `get(k)` 返回 `s(k)`

State 是被peers管理的（state就是存储在peer上的）, 而不是被orderers 和 clients。

**State partitioning**
 KVS 中的Keys，可以从它们的名字，识别出属于属于哪一个特定的chaincode, 只有特定Chaincode的transaction，才能修改属于它的keys。 原则上, 任何 chaincode 都能够读取属于其它 chaincodes的keys。cross-chaincode的transactions, 即修改属于其它chaincodes的state是v1后续版本的功能。

### 1.2.2 Ledger
Ledger 提供所有成功的state改变，和不成功的尝试改变的历史。

**Ledger 被ordering service (see Sec 1.3.3) 构建成一个完全有序的，（有效或无效）transaction 块组成的hash chain。** Hashchain 强加了ledger中blocks的完全有序性。每个block 又包含一个完全有序的transactions的数组， 这又整体上强加了所有transaction的完全有序性。

**Ledger存储在所有的peers上, 也可以选择存储在几个orderers上**。 在orderer的上下文中，我们将OrdererLedger统称为Ledger, 在 peer 的上下文中，我们将PeerLedger统称为Ledger。 PeerLedger 与 the OrdererLedger 不同，peers 本地维护着一个 bitmask 来区分有效transactions和无效transaction。(XX章节会详细介绍).

Peers 可能会修剪（去掉？） PeerLedger（ XX章节描述了这点，它是v1后续版本的特性). Orderers维护OrdererLedger，可以增加整个系统的容错性和可用性，并且可以随时决定修剪它，只要维护ordering service的属性即可。

Ledger允许重做所有transactions的历史记录，并且重建state。

### 1.3. Nodes（节点）
Nodes是blockchain的通信实体。一个"node" 仅仅是一个逻辑上的功能，多个不同类型的node可以运行在同一个物理server上。重要的是node如何分组在“信任域（trust domains）”中并与控制它们的逻辑实体相关联。

有三种类型的nodes:

* Client or submitting-client: 

提交真正的transaction-invocation 到 endorsers, **广播 transaction-proposals 到 ordering service**。

* Peer: 

提交transactions ，维护state 和 ledger的拷贝（Ledger在每个peer上都有，且是同步的，某一个peer上的ledger，可能是从leader peer上同步过来的）。 除此之外，peer还有特殊的 endorser 角色。

* Ordering-service-node or orderer:

 运行实现了分发保证的通信服务，例如atomic 或者 total order 的广播.
#### 1.3.1. Client（客户端）
客户端为了和blockchain通信，必须和peer连接。 The client may connect to any peer of its choice. 客户端创建，并从而invoke(调用) transactions.

**第二章节会详细介绍, 客户端和 peers、 ordering service都会通信。**

#### 1.3.2. Peer
一个 peer 从ordering service那里，以block的形式，接收有序的state更新。并且维护state和ledger。

Peers 可以同时担当一个特殊的角色endorser。endorsing peer 的特殊功能是针对特定的chaincode，包含在一个transaction被提交前，对它背书、认可。 **每个chaincode都可以指定endorsement policy（背书策略）** （背书策略是引用一组endorsing peers的）。 

策略定义了一个有效transaction endorsement(一般是一组endorsers的签名)的充分必要条件，二三章节会详细介绍这点。 **在install Chaincode的部署transactions的特殊情况下，（部署）背书策略被指定为System Chaincode的认可策略。**

#### 1.3.3. Ordering service nodes (Orderers)
Orderers 组成了 ordering service, 它提供可靠的通信网络。 Ordering service 可以用不同的方式实现: 中心化的服务(主要用户开发和测试) 、分布式的方式（用来应对不同的网络和节点的容错性）。

**Ordering service为clients 和 peers 提供一个共享的通信 channel， 提供一个包含transaction的消息的广播服务。** Clients 连接到 channel，并且可以在该Channel上广播消息，然后将消息传递给所有peers。 Channel 支持所有消息的原子分发（有序分发）。 换而言之, channel 对所有连接到它的peers，发送相同的、商业逻辑上有序的消息。 **这种原子性的通信保证，也叫做全序广播, 原子广播, 或者分布式系统下的共识机制（我们通常都叫做共识机制）
。**
通信的消息，就是会包含在blockchain state中的候选transaction。

**Partitioning (ordering service channels)**
 Ordering service 可以支持多个channels ，就像发布-订阅消息系统一样。 Clients 可以连接到一个指定的channel，然后发送消息和获取到达的消息。 Channels可以被看做 partitions - 连接到一个Channel的Client不知道其他Channel的存在，但client可能连接到多个Channel。
 
尽管Fabric v1中的ordering service 的实现支持多个channels, 为了简化介绍，本文后面的内容，我们假设 ordering service 由一个单独的channel/topic组成。

**Ordering service API** 
Peers通过ordering service提供的接口，连接到ordering service提供的Channel。Ordering service API 由两个基本操作组成(更通用的异步事件):

* `broadcast(blob)`: client 调用它来广播一条任意的消息块，在Channel上传播。
* `deliver(seqno, prevhash, blob)`: Ordering service 在peer上调用这个接口，使用指定的非负整数序列（seqno）和最近交付的的blob（可以理解为操作transaction）的hash（prevhash）来发送消息blob。换而言之, 它是ordering service的输出event。deliver() 有时在pub-sub 系统中也叫做 notify() ，或者在 BFT 系统中叫做commit() 。


**通常情况下，为了效率的提升, ordering service 会对 blobs 分组（分批次），然后在一个单独的deliver事件中输出blocks，而不是输出每个独立的transactions（也就是blobs）。**在这种情况下，ordering service必须在每个block中，强加和传达一个确定的blobs的顺序。每个block中的blobs的个数可以被ordering service 动态指定。

在下文中，为了便于表达，我们定义订ordering service的属性（本小节的其余部分），并解释transaction endorsement的工作流程（第2节）（假设每个deliver事件有一个blob的情况下，解释的工作流）。 根据上面提到的一个block的blob的确定性排序，工作流的解释很容易扩展到Block。

**Ordering service 属性**

Ordering service 有如下的保证:

* 安全性（一致性保证）。尽管peers会断开连接或者宕机，但是它们会重新连接和启动。除非某些客client（或者peer）实际调用广播（blob），否则不发送 blob，并且最好每个Channel的blob只发送一次。此外, `deliver()` 事件包含上一次`deliver()` 事件（`prevhash`）数据的加密hash。`prevhash`是 序列号为`seqno-1`的`deliver()` 事件参数的加密hash。 这将在`deliver（`）事件之间建立hash chain，用于帮助验证ordering service输出的完整性，后面的第4节和第5节中会介绍。第一个`deliver()` 事件的特殊情况下, `prevhash` 有一个默认值。

* Liveness (delivery 保证)。 Liveness保证是由具体的ordering service实现达到的。准确的保证需要依赖网络和节点的容错性。
## 2. Transaction endorsement的基本工作流
下面我们概要介绍一下transaction的high-level请求。

注意：下面的协议没有假设所有的transactions都是确定的, 也就是说，它允许不确定的transactions。

### 2.1. client 创建一个 transaction 并且发送到它选择的endorsing peers
为了调用transaction, client 发送一个 PROPOSE 消息到它选择的一组endorsing peers(可能并不是同时 - 见 2.1.2. 和 2.3.节)。**对于指定chaincodeID的endorsing peers组，client通过peer得到，对方又通过endorsement policy发现endorsing peers(详见3章节)。** 例如, transaction 可以发送给一个指定chaincodeID的所有的endorsers。 这就是说, 一些endorsers 可以是offline的, 其它的可能反对并选择不支持transaction 。 发出Submitting 的client 会去满足可用的endorsers的政策表达 。

下面, 我们首先详细介绍PROPOSE消息格式，然后讨论submitting client和endorsers之间可能的交互模式。
#### 2.1.1. PROPOSE 消息格式
一个PROPOSE的消息格式是：`<PROPOSE,tx,[anchor]>`, `tx` 是必须的 `anchor` 是可选的参数。下面详细介绍：

* `tx=<clientID,chaincodeID,txPayload,timestamp,clientSig>`

`clientID`是submitting client的`ID`,
`chaincodeID` 指的是transaction所属的 chaincode
`txPayload` 是包含 submitted transaction本身的有效负载。
`timestamp` 是由客户维护的单调递增（对于每个新transaction）整数。
`clientSig` 是client 端的签名。

txPayload的细节，在调用transaction和部署transaction之间有所不同。

对于调用transaction，txPayload将由2个字段组成：
`txPayload = <operation, metadata>`, 
`operation` 是指 chaincode 操作(函数) 和参数。
`metadata` 是指和调用相关的属性。

对于部署 transaction, txPayload 由3个字段组成：
`txPayload = <source, metadata, policies>`,
`source` 指的是chaincode的源码。
`metadata` 指的是与 chaincode 和 application相关的属性。
`policies` **包含与所有peer可访问的Chaincode有关的policies（政策），例如endorsement policies（背书政策）**。
 **注意endorsement policies 不是部署 transaction中的txPayload提供的, `txPayload` 包含的是 endorsement policy ID 和它的参数(详见章节3)**

* anchor 包含读取版本依赖关系，或更具体地说，key-version对（即，anchor是KxN的一个子集），它将`PROPOSE`请求绑定或“anchor”到KVS中指定版本的key（请参阅第1.2节）。 如果客户端指定anchor参数，则endorser仅在读取其本地KVS，并匹配anchor中的相应key的版本号时，才认可transaction（更多详细信息，请参见第2.2节）。

tx的加密hash被所有节点用作唯一transaction标识符tid（即，`tid = HASH（tx）`）。 客户端将tid存储在内存中，并等待来自endorsing peers的响应。

#####2.1.2. Message 模式

client定与endorsers的交互顺序。 例如，client通常会发送`<PROPOSE，tx>`（即没有anchor参数）给一个单独的endorser，然后client可以生成版本依赖关系（anchor），以便client稍后可以将其用作PROPOSE消息的参数给其它endorser。 作为另一个例子，client可以直接发送`<PROPOSE，tx>`（没有anchor）给所选的所有endorser。 不同的通信模式都可以，client可以自由选择这些模式（另见2.3节）。

### 2.2. Endorsing peer 模拟一个 transaction（交易） 并且 产生一个endorsement 签名

在接收到来自客户端的`<PROPOSE，tx，[anchor]>`消息时，endorsing peer epID首先验证client的签名clientSig，然后模拟transaction。 如果client指定了anchor，则只有在，从其本地KVS中的对应key，读取到的版本号（即，下面定义的read set），匹配由anchor指定的版本号时，endorsing peer才模拟transaction。

通过调用Chaincode和endorsing peer本地的state，模拟一个transaction。

执行的结果是,endorsing peer计算出读取版本依赖(readset) 和 state 更新 (writeset)。

回想一下，state由key/value（k / v）对组成。 所有k / v条目都是版本化的。也就是说，每个条目都包含有序的版本信息，每次更新时存储在key下的version（版本）都会增加。 执行transaction的peer记录Chaincode访问的所有、读取或写入的k / v对。但peer尚未真正的更新其state。 进一步来说：



本文中有些名术语是基本等价的，像transactions和blobs，之所以没有统一为一个术语，是为了匹配上下文环境。
本文主要参考
https://github.com/hyperledger/fabric/blob/release-1.1/proposals/r1/Next-Consensus-Architecture-Proposal.md

