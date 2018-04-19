# ConfigEnvelope

ConfigEnvelope被设计为，不依赖之前的configuration transactions的，包含一个chain的_all_配置。

它用以下方案生成：
1.  检索现有配置
2.  note要修改的配置属性（ConfigValue，ConfigPolicy，ConfigGroup）
3.  向ConfigUpdate.read_set（稀疏）添加任何中间ConfigGroups
4.  向ConfigUpdate.read_set（稀疏）添加其它所需的依赖项
5.  修改配置属性，将每个版本递增1，并将它们设置在ConfigUpdate.write_set中。注意：任何没有修改但指定的元素，应该已经在read_set中，因此可以指定为稀疏
6.  创建ConfigUpdate消息，并将其编组到ConfigUpdateEnvelope.update。更新和编码所需的签名
*每个签名都是ConfigSignature类型
*ConfigSignature签名位于signature_header和ConfigUpdate字节（它包含一个ChainHeader）的连接之上，
9.   提交新的Config，以在由提交者签名的Envelope中进行排序
* Envelope Payload将数据，设置为编组的ConfigEnvelope
* Envelope Payload有一个类型为Header.Type.CONFIG_UPDATE的header

configuration manager（配置管理器）将验证：
 1. read_set中的所有项都存在于读取版本中
 2. write_set中与read_set不同版本的所有项，或者不read_set中的项，已根据其mod_policy进行了适当的签名
 3.新的configuration满足ConfigSchema


