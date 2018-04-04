# 环境准备
本文，根据byfn例子，介绍手动向Channel中添加一个Org的详细步骤。
## 日志配置

将`cli` 和 `Org3cli` containers的日志级别`CORE_LOGGING_LEVEL`修改为 `DEBUG`

修改`first-network`目录中的`docker-compose-cli.yaml`文件，

```c
cli:
  container_name: cli
  image: hyperledger/fabric-tools:$IMAGE_TAG
  tty: true
  stdin_open: true
  environment:
    - GOPATH=/opt/gopath
    - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
    #- CORE_LOGGING_LEVEL=INFO
    - CORE_LOGGING_LEVEL=DEBUG
```

修改相同目录下的`docker-compose-org3.yaml`文件

```c
Org3cli:
  container_name: Org3cli
  image: hyperledger/fabric-tools:$IMAGE_TAG
  tty: true
  stdin_open: true
  environment:
    - GOPATH=/opt/gopath
    - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
    #- CORE_LOGGING_LEVEL=INFO
    - CORE_LOGGING_LEVEL=DEBUG
```

## 清理环境
如果启动过`eyfn`
`./eyfn.sh down`

停掉container等
`./byfn.sh -m down`

生成默认的 BYFN artifacts:
`./byfn.sh -m generate`

启动网络
`./byfn.sh -m up`

# 生成 Org3 Crypto Material

从`first-network`切换到`org3-artifacts`子目录。

`cd org3-artifacts`

这里有两个有趣的yaml文件：`org3-crypto.yaml`和`configtx.yaml`。首先，为`Org3`生成加密资料：

```c
../../../bin/cryptogen generate --config=./org3-crypto.yaml
```

该命令读入我们新的加密yaml文件 -`·org3-crypto.yaml` - 并利用`cryptogen`为`Org3` CA,以及与此新Org绑定的两个peer,生成密钥和证书。与BYFN实现一样，这个加密资料被放入当前工作目录（在我们的例子中为`org3-artifacts`）内新生成的`crypto-config`文件夹中。

现在使用`configtxgen`工具在JSON中打印出特定于`Org3`的配置材料。我们告诉工具在当前目录中查找它需要引用的`configtx.yaml`。

```c
export FABRIC_CFG_PATH=$PWD && ../../../bin/configtxgen -printOrg Org3MSP > ../channel-artifacts/org3.json
```
上述命令创建一个JSON文件 - `org3.json`，并将其输出到`first-network`目录下的`channel-artifacts`子目录中。该文件包含Org3的策略定义，以及以base 64格式提供的三个重要证书：admin用户证书（稍后将用作Org3的管理员），CA根证书和TLS根证书。**在后面的步骤中，我们将把这个JSON文件附加到Channel配置中。**

**最后一项工作是将`Orderer Org`的`MSP资料`移到Org3 `crypto-config`目录中。**特别是，我们关注Orderer的TLS根证书，这将允许Org3实体和网络的ordering节点之间的安全通信。

```c
cd ../ && cp -r crypto-config/ordererOrganizations org3-artifacts/crypto-config/
```

现在我们准备更新Channel配置...

# 准备CLI环境

更新过程中，需要使用配置转换器工具 - `configtxlator`。该工具，提供独立于SDK的无状态`REST API`。此外，它还提供CLI，以简化Fabric网络中的配置任务。**该工具允许，在不同的等效数据表示/格式（本例中的情况，是protobufs和JSON之间）之间轻松转换。另外，该工具可以根据两个channel配置之间的差异来计算配置更新transaction。**

首先，exec进入CLI容器。回想一下，这个容器已经安装了`BYFN`加密配置库，让我们可以访问两个原始peer组织和`Orderer Org`的MSP资料。bootstrapped identity 是Org1管理员用户，这意味着我们想要充当Org2的任何步骤，都需要导出MSP特定的环境变量。
命令如下：
`docker exec -it cli bash`

现在将jq工具安装到容器中。该工具允许,脚本与由`configtxlator`工具返回的JSON文件进行交互：

`apt update && apt install -y jq`

导出`ORDERER_CA`和`CHANNEL_NAME`变量：

```c
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem  && export CHANNEL_NAME=mychannel
```

检查以确保变量已正确设置：

`echo $ORDERER_CA && echo $CHANNEL_NAME`

# Fetch the Configuration

现在,我们有一个带有两个关键环境变量的CLI容器 - `ORDERER_CA`和`CHANNEL_NAME`。让我们去获取channel的最新配置块 - mychannel。

我们不得不,拉取配置的最新版本的原因,是因为channel配置元素是版本化的。版本管理很重要，原因有几个:

* 它可以防止重复或重放配置更改（例如，使用旧CRL恢复到channel配置会带来安全风险）。
* 此外，它有助于确保并发性（如果您希望从您的channel中移除组织，例如，在添加新组织后，版本控制将防止你误删除两个组织。仅仅会删除，你想要删除的组织。）

```c
peer channel fetch config config_block.pb -o orderer.example.com:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA
```
该命令将二进制protobuf channel配置块保存到`config_block.pb`（pb是protobuf的简写）。请注意，名称和文件扩展名的选择是任意的。但是，建议遵循一个约定,来标识所表示的对象的类型及其编码（protobuf或JSON 编码格式）。

当您发布`peer channel fetch`命令时，终端中的输出量很大。日志中的最后一行很有用：

```c
2017-11-07 17:17:57.383 UTC [channelCmd] readBlock -> DEBU 011 Received block: 2
```

这告诉我们，mychannel的最新配置块实际上是块2，而不是起始块。默认情况下，`peer channel fetch`命令返回目标channel的最新配置块，在本例中为第三个块(块从0开始计数)。这是因为BYFN脚本在两个单独的Channel更新transaction中，为我们的两个组织（Org1和Org2）定义了Anchor peers。

因此，我们有以下配置顺序：

```c
block 0: genesis block
block 1: Org1 anchor peer update
block 2: Org2 anchor peer update
```


# 将配置转换为JSON并Trim It Down(修剪)

现在我们将使用`configtxlator`工具，将此Channel配置块 解码为JSON格式（这种json格式，可由人读取和修改）。我们还必须删除，所有与我们想要改变无关的header，metadata, creator signatures等等。我们通过jq工具来实现这一点：

```c
configtxlator proto_decode --input config_block.pb --type common.Block | jq .data.data[0].payload.data.config > config.json
```

这给我们留下了一个修剪后的JSON对象--`config.json`，位于`first-network`中`fabric-samples`文件夹中（在cli container中路径是 `/opt/gopath/src/github.com/hyperledger/fabric/peer`，至于`fabric-samples`中没找到），它将作为我们配置更新的baseline。

阅读此文件，值得研究，因为它揭示了底层的配置结构，和可以进行的其他类型的Channel更新。

# Add the Org3 Crypto Material

无论您尝试进行哪种配置更新，您采取的步骤都将几乎相同。 
我们选择在本教程中添加一个组织，因为它是您可以尝试的最复杂的channel配置更新。

我们将再次使用`jq`工具,将`Org3`配置定义（`org3.json`）附加到channel的应用程序groups字段，并将输出命名为`modified_config.json`。

```c
jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"Org3MSP":.[1]}}}}}' config.json ./channel-artifacts/org3.json > modified_config.json
```
现在，在CLI容器中，我们有两个感兴趣的JSON文件--`config.json`和`modified_config.json`。初始文件仅包含`Org1`和`Org2`资料，而“修改”文件包含全部三个Orgs。此时，只需重新编码这两个JSON文件并计算增量即可。

首先，将`config.json`翻译成名为`config.pb`的protobuf：

```c
configtxlator proto_encode --input config.json --type common.Config --output config.pb
```

接下来，将`modified_config.json`编码为`modified_config.pb`：

```c
configtxlator proto_encode --input modified_config.json --type common.Config --output modified_config.pb
```

现在使用`configtxlator`来计算这两个配置`protobufs`之间的增量。该命令将输出一个名为`org3_update.pb`的新protobuf二进制文件：

```c
configtxlator compute_update --channel_id $ CHANNEL_NAME --original config.pb --updated modified_config.pb --output org3_update.pb
```

这个新的原型 - `org3_update.pb` - 包含Org3定义和指向Org1和Org2资料的高级指针。我们可以放弃针对Org1和Org2的广泛的MSP资料和修改政策信息，因为这些数据已经存在于Channel的genesis block中。因此，我们只需要两种配置之间的增量（delta）。

在提交Channel更新之前，我们需要执行一些最后步骤。首先，让我们将此对象，解码为可编辑的JSON格式，并将其称为`org3_update.json`：

```c
configtxlator proto_decode --input org3_update.pb --type common.ConfigUpdate | jq . > org3_update.json
```

现在，我们有一个解码的更新文件 - `org3_update.json` - 我们需要将其封装在信封消息中。这一步将，给回我们之前剥去的header 字段。我们将这个文件命名为`org3_update_in_envelope.json`：

```c
echo '{"payload":{"header":{"channel_header":{"channel_id":"mychannel", "type":2}},"data":{"config_update":'$(cat org3_update.json)'}}}' | jq . > org3_update_in_envelope.json
```

使用我们正确构建的JSON - `org3_update_in_envelope.json `- 我们将最后一次利用`configtxlator`工具并将其转换为Fabric所需的完全成熟的protobuf格式。我们将命名我们的最终更新对象`org3_update_in_envelope.pb`：

```c
configtxlator proto_encode --input org3_update_in_envelope.json --type common.Envelope --output org3_update_in_envelope.pb
```


# Sign and Submit the Config Update(签署并提交配置更新)

现在在我们的CLI容器中,有一个protobuf二进制文件 - `org3_update_in_envelope.pb`。但是，在将配置写入ledger之前，我们需要必要的Admin用户签名。

我们channel Application group的修改政策（mod_policy）被设置为默认`“MAJORITY”`，这意味着我们需要大多数现有组织管理员对其进行签名。因为我们只有两个组织--`Org1`和`Org2` - 而其中大部分是两个，我们需要他们两个签名。如果没有两个签名，ordering service将拒绝,未完成该政策的交易。

首先，让我们以`Org1` Admin的身份签署这个更新协议。请记住，CLI容器是使用Org1 MSP资料引导的，所以我们只需发出peer channel `signconfigtx`命令：

```c
peer channel signconfigtx -f org3_update_in_envelope.pb
```

最后一步是切换CLI容器的身份以反映`Org2` Admin用户。我们通过导出特定于`Org2` MSP的四个环境变量来实现这一点。

注意：
在组织之间切换，以签署配置事务（或执行其他任何操作）并不是真实生产环境的Fabric操作。 一个容器永远不会与整个网络的加密资料一起安装。（本文中，只是测试环境。）相反，配置更新，需要安全地通过带外传递给Org2管理员进行检查和批准。

导出 `Org2` 环境变量:

```c
export CORE_PEER_LOCALMSPID="Org2MSP"

export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt

export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp

export CORE_PEER_ADDRESS=peer0.org2.example.com:7051
```


最后，我们将发出peer channel更新命令。 Org2管理员签名将附加到此调用中，因此不需要再次手动签署protobuf：

(注意:即将到来的ordering service更新调用,将接受一系列系统签名和政策检查。因此，您可能发现，流式传输并检查ordering节点的日志很有用。从另一个shell中，发出`docker logs -f orderer.example.com`命令以显示它们。)

发送更新调用：

```c
peer channel update -f org3_update_in_envelope.pb -c $CHANNEL_NAME -o orderer.example.com:7050 --tls --cafile $ORDERER_CA

```
如果您的更新已成功提交，您应该看到与以下内容类似的消息摘要：

```c
2018-02-24 18：56：33.499 UTC [msp / identity] Sign - > DEBU 00f Sign：digest：3207B24E40DE2FAB87A2E42BC004FEAA1E6FDCA42977CB78C64F05A88E556ABA
```

您还将看到我们的配置事务提交：

```c
2018-02-24 18:56:33.499 UTC [channelCmd] update -> INFO 010 Successfully submitted channel update
```

成功的channel更新调用，向Channel上的所有peer返回一个新的块 - 块5。如果你还记得，块0-2是初始Channel配置，而块3和块4是mycc chaincode的实例化和调用。同样，块5作为Channel上现在定义的Org3的最新Channel配置。

检查`peer0.org1.example.com`的日志：

`docker logs -f peer0.org1.example.com`

# Configuring Leader Election

注意:本部分作为通用参考，用于了解在初始channel配置完成后,向组织添加网络时的leader选举设置。 此示例默认为动态leader选举，这是为,peer-base.yaml中网络中的所有peer设置的。

新加入的peer由genesis块引导，其中不包含有关在channel配置更新中，添加的组织的信息。因此，新peer无法利用gossip，因为他们无法验证，自己组织中由其它peer转发的块，直到他们获得，将该组织添加到Channel的配置事务。因此，新添加的peer必须具有以下配置之一，以便他们接收来自ordering service的数据块：

1.要使用静态leader模式，请将peer配置为组织leader：

```c
CORE_PEER_GOSSIP_USELEADERELECTION=false
CORE_PEER_GOSSIP_ORGLEADER=true
```

注意:对于添加到channel的所有新peer，此配置必须相同。

2.为了利用动态leader选举，配置peer使用leader选举：

```c
CORE_PEER_GOSSIP_USELEADERELECTION=true
CORE_PEER_GOSSIP_ORGLEADER=false
```

注意:由于新添加的组织的peer，将无法形成成员资格视图，因此每个peer将开始宣称自己是leader，此选项与静态配置类似。但是，一旦他们获得了，将组织添加到Channel的配置事务的更新，组织将只有一个活动leader。因此，如果您最终希望组织的peer利用leader选举，建议您利用此选项。

# Join Org3 to the Channel
todo
# Upgrade and Invoke Chaincode
todo



欢迎加入区块链技术交流QQ群 694125199

更多区块链知识：
https://github.com/xiaofateng/knowledge-without-end/tree/master/区块链/Hyperledger

http://hyperledger-fabric.readthedocs.io/en/latest/channel_update_tutorial.html


