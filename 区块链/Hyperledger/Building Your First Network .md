# Building Your First Network 步骤详解介绍

## build your first network (BYFN) 包含的内容

第一个Hyperledger Fabric network由下面内容组成：

*  4 个peers，代表2个不同的organizations。
*  1 个orderer 节点。
*  启动1个 container， 执行脚本，将peers加入Channel，部署和实例化Chaincode，并根据部署的Chaincode，驱动transactions的执行。

## 准备工作
1. https://github.com/xiaofateng/knowledge-without-end/blob/master/区块链/Hyperledger/Hyperledger%20Fabric%20源码和镜像下载.md
2. 修改`fabric-samples/first-network`目录下的`docker-compose-cli.yaml`日志级别为debug

```c
cli:
  container_name: cli
  image: hyperledger/fabric-tools:$IMAGE_TAG
  tty: true
  stdin_open: true
  environment:
    - GOPATH=/opt/gopath
    - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
    - CORE_LOGGING_LEVEL=DEBUG
    #- CORE_LOGGING_LEVEL=INFO
```

## Crypto Generator

我们将使用`cryptogen`工具（这个工具只是在测试的时候使用，线上一般使用Fabric CA等）为我们的各种network entitie（网络实体）生成加密资料（x509证书和签名密钥）。 这些`certificates`（证书）是身份的代表，它们允许在我们的实体，进行通信和transact时进行签名/验证身份。

`Cryptogen`使用包含网络拓扑的文件 - `crypto-config.yaml`，并允许我们为Organizations和属于这些Organizations的组件生成一组`certificates` and `keys`（证书和密钥）。

每个Organizations都配备了一个独特的根证书（`ca-cert`），将特定的组件（peers and orderers）绑定到该Org。 通过为每个Org分配一个唯一的CA证书，我们正在模拟一个典型的网络，其中一个Member将使用自己的Certificate Authority（证书授权）。 Hyperledger Fabric中的Transactions和通信由实体的private key（私钥，存储的文件名是 `keystore`）签名，然后通过public key（公钥，存储的文件名是 `signcerts`）进行验证。

你会注意到这个文件中的 `count` 变量。我们用它来指定每个Organization的peers数量; 在我们的案例中，每个Organization有两个peers。

在运行该工具之前，让我们快速浏览一下`crypto-config.yaml`的代码片段。 特别注意OrdererOrgs header下的`“Name”`, `“Domain”` 和 `“Specs”`：

```c
OrdererOrgs:
#---------------------------------------------------------
# Orderer
# --------------------------------------------------------
- Name: Orderer
  Domain: example.com
  CA:
      Country: US
      Province: California
      Locality: San Francisco
  #   OrganizationalUnit: Hyperledger Fabric
  #   StreetAddress: address for org # default nil
  #   PostalCode: postalCode for org # default nil
  # ------------------------------------------------------
  # "Specs" - See PeerOrgs below for complete description
# -----------------------------------------------------
  Specs:
    - Hostname: orderer
# -------------------------------------------------------
# "PeerOrgs" - Definition of organizations managing peer nodes
 # ------------------------------------------------------
PeerOrgs:
# -----------------------------------------------------
# Org1
# ----------------------------------------------------
- Name: Org1
  Domain: org1.example.com
  EnableNodeOUs: true
```

网络实体的命名约定如下 -  `“{{.Hostname}}.{{.Domain}}”`。 因此，使用我们的ordering node作为参考点，我们看到一个名为 - `orderer.example.com`的ordering node，它与`Orderer`的`MSP ID`绑定。 

运行`cryptogen`工具后，生成的证书和密钥将被保存到名为`crypto-config`的文件夹中。

下面执行命令(假设当前路径是`hyperledger/fabric-samples/first-network`)：

`../../bin/cryptogen generate --config=./crypto-config.yaml`


执行命令后，为各个节点生成的证书如下：

目录`/hyperledger/fabric-samples/first-network/crypto-config/ordererOrganizations/example.com`

![](media/crypto-config.png)






