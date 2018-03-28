# Hyperledger Fabric MSP和Fabric CA的区别

MSP是Membership Service Provider - 是可插拔的接口，它用于支持各种认证体系结构，为membership orchestration architecture提供抽象层。 MSP抽象提供：

* 具体的身份格式
* 用户证书验证
* 用户证书撤销
* 签名生成和验证

而 Fabric-CA 用于生成证书和密钥，以真正的初始化MSP。 Fabric-CA是用于身份管理的MSP接口的默认实现。

**即，MSP只是一个接口，Fabric-CA是MSP接口的一种实现。**


在官方的例子中，BYFN, 用命令`./byfn.sh -m up` 启动网络。
这时候网络中是没有CA containers的，那么MSP是怎么工作的呢?

在构建第一个网络示例时，使用了主要用于测试和演示场景的`cryptogen`工具，以快速设置初始化MSP所需的加密资料，基本上它会生成操作网络实体所需的certificates（证书）。 因此，实际上并不需要使用Fabric-CA。

使用Fabric-CA的示例在这里：
https://github.com/hyperledger/fabric-samples/tree/master/fabric-ca


