# 介绍
用户编写的Chaincode在container中运行（本文中称为`“用户chaincode”`），并通过网络与peer进行通信。 这些Chaincode可以执行的代码有限制的。 例如，他们只能通过`“ChaincodeStub”`接口（如`GetState`或`PutState`）与peer进行交互。 Chaincode需要放宽这些限制，这样的Chaincode被广义地称为`“System Chaincode”`。

System Chaincode的目的是为了定制Fabric的某些行为，如timer service 或 naming service。

设计System Chaincode最简单的方法就是和Peer一样运行它。 下文详细介绍下。

## In-process chaincode
该方法考虑作为hosting peer，运行Chaincode所需的更改和通用化。

要求

* 与 用户Chaincode 共存
* 坚持与 用户Chaincode 一样的生命周期 - 除了可能的“部署”（**这里根据官方文档，它与用户Chaincode的生命周期是不一样的**）
* 易于安装


## Chaincode环境的重要考虑事项
对于任何Chaincode环境（容器，进程等）有两个主要考虑因素：

* Runtime 注意事项
* Lifecycle 注意事项

### Runtime 注意事项
Runtime考虑可以大致分为Transport（传输） 和 Execution机制。 Fabric为这些机制提供的抽象，是扩展系统，以便以直接的方式支持与现有Chaincode runtimes 一致的能力。

### Transport 机制
peer 和Chaincode使用`ChaincodeMessages`进行通信。

* 该协议通过`ChaincodeMessage`完全封装
* 目前的gRPC streaming 可以很好地映射到 Go Channels

上述内容，允许抽象处理Chaincode传输的一小部分代码，这使得大部分代码在两种环境中都是通用的。 这是runtime environments（执行环境）统一处理的关键。

### Execution机制
fabric为`“vm”`提供了一个基本interface。 Docker容器是在这个抽象之上实现的。 `“vm”`（vm接口）也可用于为进程运行时提供execution environment。 这对透明化的实现所有执行访问（部署，启动，停止等）进行封装有好处。

## Lifecycle 注意事项

| Life cycle<span class="Apple-tab-span" style="white-space:pre"></span><span class="Apple-tab-span" style="white-space:pre"></span> | Container model | In-process Model |
| --- | --- | --- |
| Deploy |  通过外部request。参与共识 | 两种方法 (1) 和peer一起自动启动,不参与共识(2) 用户通过 request启动，参与共识 |

# Deploying System Chaincodes

System chaincodes提出了一个系统。System chaincodes将在`core.yaml`中定义。 至少，Chaincode定义将包含以下信息`{名称，路径}`。 每个Chaincode定义还可以指定 依赖的Chaincode，以便Chaincode仅在其相关Chaincode存在时才会被执行。


欢迎加入区块链技术交流QQ群 694125199

更多区块链知识：
https://github.com/xiaofateng/knowledge-without-end/tree/master/区块链/Hyperledger


本文主要参考
https://github.com/hyperledger/fabric/blob/release-1.1/proposals/r1/System-Chaincode-Specification.md#execution-mechanism


