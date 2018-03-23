

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
#### 1.3.1. Client
The client represents the entity that acts on behalf of an end-user. It must connect to a peer for communicating with the blockchain. The client may connect to any peer of its choice. Clients create and thereby invoke transactions.

As detailed in Section 2, clients communicate with both peers and the ordering service.


本文主要参考
https://github.com/hyperledger/fabric/blob/release-1.1/proposals/r1/Next-Consensus-Architecture-Proposal.md

