# 介绍
事件框架支持发出2种类型的event(事件)，`block`和`自定义/chaincode event`（在`events.proto`中定义的`ChaincodeEvent`类型）的能力。基本思想是，client（event consumers\事件消费者）将注册event类型（当前为`“block”`或`“chaincode”`）。并且在chaincode的情况下，它们可以指定附加的注册标准，即`chaincodeID`和`eventname`。 

`ChaincodeID`标识client想要查看event的特定Chaincode。
`eventname`是Chaincode开发人员，在调用Chaincode中的`SetEvent API`时嵌入的字符串。调用transaction是当前唯一可以发出event的操作，并且每个调用，在每个transaction中只能发出一个event。

# 设计: 使用 TransactionResult 存储 events
向`TransactionResult`加入一个event消息

``` c
/ chaincodeEvent - any event emitted by a transaction
type TransactionResult struct {
	Uuid           string          
	Result         []byte
	ErrorCode      uint32
	Error          string
	ChaincodeEvent *ChaincodeEvent
}
```
ChaincodeEvent是：

``` c
type ChaincodeEvent struct {
	ChaincodeID string
	TxID        string
	EventName   string
	Payload     []byte
}
```

`ChaincodeID`是与Chaincode关联的`uuid`，`TxID`是生成event的事transaction，`EventName`是由Chaincode给event赋予的名称（请参阅SetEvent），`Payload`是Chaincode附加到event的payload（有效载荷）。 Fabric不对有效载荷施加struct或限制，但最好不要使其尺寸太大。**Payload具体是什么？**

chaincode Shim上的 `SetEvent API`

```c
func (stub *ChaincodeStub) GetState(key string)
func (stub *ChaincodeStub) PutState(key string, value [\]byte)
func (stub *ChaincodeStub) DelState(key string)
…
…
func (stub *ChaincodeStub) SetEvent(name string, payload []byte)
…
…
```

当transaction完成时，`SetEvent`将event添加到`TransactionResult`并提交到ledger。 然后该event将由event framework（事件框架）发布到，注册该event的所有客户端。

## 一般Event类型及其与Chaincode Event的关系

Event与event类型相关联。 客户注册他们想要接收event的event类型。 event类型的生命周期由`“block”`event来说明

1.在启动peer时，在支持的event类型中添加`“block”`
2.client可以与peer（或多个peers）一起注册感兴趣的`“block”` event类型
3.创建`Block`的Peers，向所有注册client发布event
4.客户收到`“block”` event并处理`Block`中的事务

Chaincode event添加了额外的注册过滤级别。 Chaincode event不是注册给定event类型的所有event，而是允许client从特定Chaincode注册特定event。 对于目前的第一个版本，为了简单起见，没有在`eventname`上实现通配符或正则表达式匹配，但后续会做。 更多信息，参见下面的`“Client Interface”`。

## Client Interface

client 当前注册并接收event的接口是一个gRPC接口，带有在`protos / events.proto`中定义的`pb`消息。 `Register` message 是一系列(重复的) `Interest` messages。每个 `Interest` 代表一个单独的event注册.

```c
message Interest {
    EventType eventType = 1;
    //理想的情况下,对不同的Reg types ，我们应该只有下面的 oneof 并摆脱 EventType。
    //但它是这样的 API： 
    //改变 Additional Reg 类型，可能增加特定类型的 
    //messages 添加到the oneof
    oneof RegInfo {
        ChaincodeReg chaincodeRegInfo = 2;
    }
}

//当 EventType 是 CHAINCODE 时候，
//ChaincodeReg 用来注册 chaincode Interests
message ChaincodeReg {
    string chaincodeID = 1;
    string eventName = 2;
    ~~bool anyTransaction = 3;~~ //TO BE REMOVED
}

//consumers发出 Register 来注册events
//string type - "register"
message Register {
    repeated Interest events = 1;
}

```

如前面部分所述，client应该使用`ChaincodeReg`消息注册Chaincode event。 使用空字符串（或`“*”`）设置`eventName`将导致chaincode中的所有event发送到listener。

在`events.proto`中定义了一种服务，使用单一方法接收注册event的stream(流)，并返回client,当 event 发生时读取事件的stream(流)。

欢迎加入区块链技术交流QQ群 694125199

更多区块链知识：
https://github.com/xiaofateng/knowledge-without-end/tree/master/区块链/Hyperledger

本文参考
https://github.com/hyperledger/fabric/blob/release-1.1/proposals/r1/Custom-Events-High-level-specification.md




