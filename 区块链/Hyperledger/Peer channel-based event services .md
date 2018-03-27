# 简介

在以前的Fabric版本中，peer event service 被称为event hub。 无论block关联哪个Channel，该服务都会在任何时候，将新block添加到peer Ledger时发送event。并且只有运行event peer的组织的成员才可以访问该event。


从v1.1开始，有两个提供event的新服务。 这些服务使用完全不同的设计来按每个Channel提供事件。 这意味着event的注册发生在channel的层面而不是peer层面，允许对peer数据的访问进行细粒度上的控制。 请求接受来自peer组织外部的身份的event（由Channel配置定义）。 这也提供了更高的可靠性和接收可能错过的event（无论是由于连接问题，还是由于peer正在加入已经运行的网络）。


# Available services
* `Deliver`

该服务发送已提交到Ledger的整个block。 如果event由Chaincode设置，则可以在block的`ChaincodeActionPayload`中找到这些event。

* `DeliverFiltered`

该服务发送“filtered” block，以及有关已提交到ledger的块的最小信息集。 它旨在用于网络中，peer的所有者,希望外部client主要接收有关其transaction和这些transaction状态的信息。 如果任何event由Chaincode设置，则可以在filtered block的`FilteredChaincodeAction`中找到这些event。

# 怎样注册事件（register for events）

通过向peer发送，包含deliver seek info message的信封,来完成来自任一服务的事件的注册。该信封包含期望的开始和停止位置，查找行为（如果未准备就绪，则阻塞直到就绪或失败）。 有帮助变量`SeekOldest`和`SeekNewest`可用于指示ledger中最早的（即第一个）块或最新的（即最后一个）块。 要使服务无限期地发送event，`SeekInfo`消息应该包含一个停止位置的`MAXINT64`。

默认情况下，这两个服务都使用Channel读取器策略,来确定是否为请求客户端授权event。

# deliver response 消息概览

event services发回`DeliverResponse`消息。

每个消息都包含以下内容之一：

* status - HTTP状态码。 如果发生任何故障，两种服务都将返回相应的故障代码; 否则，一旦服务完成发送`SeekInfo`消息请求的所有信息，它将返回`200` - SUCCESS。
* block - 仅由`Deliver` service返回。
* filtered block - 仅由`DeliverFiltered`service返回。

一个filtered block 包含:

* channel ID.

* number (如 block number)

* filtered transactions数组

* transaction ID
    type (如`ENDORSER_TRANSACTION`, `CONFIG`)
    transaction validation code.
    
* filtered transaction actions
filtered chaincode actions数组。
transaction的chaincode event。


欢迎加入区块链技术交流QQ群 694125199

更多区块链知识：
https://github.com/xiaofateng/knowledge-without-end/tree/master/区块链/Hyperledger

本文参考：
http://hyperledger-fabric.readthedocs.io/en/latest/peer_event_services.html


