# 什么是Identity？
区块链网络中的不同参与者包括peer，orderers，client applications，administrators等等。
这些参与者中的每一个，都有封装在X.509数字证书中的Identity（身份）。**这些Identity真的很重要，因为它们确定了参与者，在区块链网络中拥有的资源权限。** 
Hyperledger Fabric使用参与者identity（身份）中的某些属性来确定权限，并为其提供特殊名称 - principal（主体）。principal就像用户ID或组ID一样，但更灵活一点，因为它们可以包含广泛的参与者的identity属性。
参与者identity的属性决定了它们的权限。这些属性通常是，参与者的organization，organizational unit，角色或参与者的特定identity。

最重要的是，identity必须是可验证的，因此它必须来自系统所信任的权威机构。MSP是在Hyperledger Fabric中实现此目标的手段。更具体地说，MSP是代表组织成员资格规则的组件，因此它定义了，管理org成员有效identity（身份）的规则。 Fabric中默认的MSP实现使用X.509证书作为identity，采用PKI分层模型。

**MSP把验证过的identities 转换成区块链网络中的members**

关于Identity的举例：
例如你去超市购物，结账的时候，只允许visa卡支付，尽管你的smart卡里面有钱，但是不能支付。
PKI提供identies的列表，MSP说出来，哪一个，是参与区块链网络的org的成员。

