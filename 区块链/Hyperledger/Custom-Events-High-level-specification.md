# 介绍
事件框架支持发出2种类型的event(事件)，block和自定义/chaincode event（在`events.proto`中定义的`ChaincodeEvent`类型）的能力。基本思想是，client（event consumers\事件消费者）将注册event类型（当前为`“block”`或`“chaincode”`）。并且在chaincode的情况下，它们可以指定附加的注册标准，即`chaincodeID`和`eventname`。 

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


