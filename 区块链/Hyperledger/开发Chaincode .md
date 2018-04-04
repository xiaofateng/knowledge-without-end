## Chaincode简介

chaincode通常处理由网络成员赞同的业务逻辑，因此它类似于“智能合约”。 可以调用chaincode来更新或查询提案交易中的ledger。 

如果有适当的许可，chaincode可以调用另一个chaincode，以访问其状态，无论是在同一个Channel还是在不同的Channel中。 请注意，如果被调用的chaincode与调用chaincode位于不同的通道上，则只允许读取查询。


## Chaincode API
每个chaincode程序必须实现`Chaincode interface`,其方法被调用以回应收到的交易。

特别是当Chaincode接收实例化或升级transaction时，将调用Init方法，以便Chaincode可以执行任何必要的初始化，包括应用程序状态的初始化。 调用Invoke方法是为了响应接收调用transaction来处理transaction提议。

chaincode “shim”API中的另一个接口是`ChaincodeStubInterface`,用于访问和修改ledger，并在chaincode之间进行调用。

在本文中，我们将通过实现一个管理“资产”的简单chaincode应用程序来演示如何使用这些API。

## 简单的资产Chaincode
我们的应用程序是一个基本样本chaincode，用于在ledger上创建资产（key-value对）。

### 选择代码的位置

现在，需要为chaincode应用程序创建一个目录作为`$GOPATH/src/`的子目录。

为了简单起见，我们使用下面的命令：

```c
mkdir -p $GOPATH/src/sacc && cd $GOPATH/src/sacc
```

现在，创建将用代码填充的源文件：

`touch sacc.go`


### Housekeeping

首先，我们从一些Housekeeping开始。**与每个chaincode一样，它实现了Chaincode接口，即`Init`和`Invoke`函数。** 因此，让我们添加go import语句以获取chaincode的必要依赖关系。 我们将导入chaincode shim包和peer protobuf包。 接下来，让我们添加一个结构`SimpleAsset`作为Chaincode shim函数的接收器。

```c
package main

import (
    "fmt"

    "github.com/hyperledger/fabric/core/chaincode/shim"
    "github.com/hyperledger/fabric/protos/peer"
)

// SimpleAsset 实现管理asset的一个简单chaincode
type SimpleAsset struct {
}
```

### Initializing the Chaincode

下面，我们实现 `Init` 函数.

```c
// chaincode 实例化的时候，调用Init来初始化数据
func (t *SimpleAsset) Init(stub shim.ChaincodeStubInterface) peer.Response {

}
```

请注意，chaincode升级也会调用此函数。在编写将升级现有chaincode的代码时，请确保适当地修改`Init`函数。 特别是，如果没有“migration”(迁移)，或没有任何东西需要作为升级的一部分进行初始化时，请提供一个空的`“Init”`方法。

接下来，我们将使用`ChaincodeStubInterface.GetStringArgs`函数检索`Init`调用的参数，并检查其有效性。 在我们的例子中，我们期待一个key-value对。

```c
func (t *SimpleAsset) Init(stub shim.ChaincodeStubInterface) peer.Response {
  // 从 transaction proposal 获取参数
  args := stub.GetStringArgs()
  if len(args) != 2 {
    return shim.Error("Incorrect arguments. Expecting a key and a value")
  }
}
```

接下来，已经确定该调用是有效的，我们将把初始状态存储在ledger中。 要做到这一点，我们将调用`ChaincodeStubInterface.PutState`，并将key和value作为参数传入。 假设一切顺利，会返回一个指示初始化成功的`peer.Response`对象。

```c
func (t *SimpleAsset) Init(stub shim.ChaincodeStubInterface) peer.Response {
  // 从 transaction proposal 获取参数
  args := stub.GetStringArgs()
  if len(args) != 2 {
    return shim.Error("Incorrect arguments. Expecting a key and a value")
  }

  // 通过调用 stub.PutState()，设置任何 variables或assets 
// 我们把 key 和 value 存储在ledger上
  err := stub.PutState(args[0], []byte(args[1]))
  if err != nil {
    return shim.Error(fmt.Sprintf("Failed to create asset: %s", args[0]))
  }
  return shim.Success(nil)
}
```

### Invoking the Chaincode

首先，我们加入`Invoke`函数的signature（签名）。

```c
//每个transaction都调用`Invoke`。每个 transactio是'get' 
//或者 'set' asset 。 'set'方法可以，
//通过提供一个新的key-value对，创建一个新的asset。
func (t *SimpleAsset) Invoke(stub shim.ChaincodeStubInterface) peer.Response {

}
```

与上面的`Init`函数一样，我们需要从`ChaincodeStubInterface`中提取参数。 `Invoke`函数的参数，将是要调用的chaincode应用程序函数的名称。 

在我们的例子中，我们的应用程序只有两个函数：`set`和`get`，它们允许设置资产的value或检索当前状态。 我们首先调用`ChaincodeStubInterface.GetFunctionAndParameters`来提取该chaincode应用函数的函数名称和参数。

```c
func (t *SimpleAsset) Invoke(stub shim.ChaincodeStubInterface) peer.Response {
    // 从transaction proposal中抽取 function和 args 
    fn, args := stub.GetFunctionAndParameters()

}
```

接下来，我们将验证函数名称是`set`还是`get`，并调用这些chaincode应用函数，通过shim返回相应的响应。shim.Success或shim.Error函数将响应序列化为gRPC protobuf消息。

```c
func (t *SimpleAsset) Invoke(stub shim.ChaincodeStubInterface) peer.Response {
    // 从transaction proposal中抽取 function和 args 
    fn, args := stub.GetFunctionAndParameters()

    var result string
    var err error
    if fn == "set" {
            result, err = set(stub, args)
    } else {
            result, err = get(stub, args)
    }
    if err != nil {
            return shim.Error(err.Error())
    }

    return shim.Success([]byte(result))
}
```

### 实现Chaincode应用程序

如前所述，我们的chaincode应用程序实现了，两个可以通过`Invoke`函数调用的函数。 现在我们来实现这些功能。 

请注意，正如我们上面提到的，为了访问ledger的状态，我们将利用`chaincode shim API`的`ChaincodeStubInterface.PutState`和`ChaincodeStubInterface.GetState`函数。

```c
//  在ledger上，存储 asset (both key and value)，如果key 存在,会用新的value覆盖.
func set(stub shim.ChaincodeStubInterface, args []string) (string, error) {
    if len(args) != 2 {
            return "", fmt.Errorf("Incorrect arguments. Expecting a key and a value")
    }

    err := stub.PutState(args[0], []byte(args[1]))
    if err != nil {
            return "", fmt.Errorf("Failed to set asset: %s", args[0])
    }
    return args[1], nil
}

// 获取指定asset key的value
func get(stub shim.ChaincodeStubInterface, args []string) (string, error) {
    if len(args) != 1 {
            return "", fmt.Errorf("Incorrect arguments. Expecting a key")
    }

    value, err := stub.GetState(args[0])
    if err != nil {
            return "", fmt.Errorf("Failed to get asset: %s with error: %s", args[0], err)
    }
    if value == nil {
            return "", fmt.Errorf("Asset not found: %s", args[0])
    }
    return string(value), nil
}
```
### 完整代码

最后, 加入 `main` 函数, 它调用 `shim.Start` 函数。

```c
package main

import (
    "fmt"

    "github.com/hyperledger/fabric/core/chaincode/shim"
    "github.com/hyperledger/fabric/protos/peer"
)

type SimpleAsset struct {
}

func (t *SimpleAsset) Init(stub shim.ChaincodeStubInterface) peer.Response {
    args := stub.GetStringArgs()
    if len(args) != 2 {
            return shim.Error("Incorrect arguments. Expecting a key and a value")
    }

    err := stub.PutState(args[0], []byte(args[1]))
    if err != nil {
            return shim.Error(fmt.Sprintf("Failed to create asset: %s", args[0]))
    }
    return shim.Success(nil)
}


func (t *SimpleAsset) Invoke(stub shim.ChaincodeStubInterface) peer.Response {

    fn, args := stub.GetFunctionAndParameters()

    var result string
    var err error
    if fn == "set" {
            result, err = set(stub, args)
    } else {
            result, err = get(stub, args)
    }
    if err != nil {
            return shim.Error(err.Error())
    }
    return shim.Success([]byte(result))
}

func set(stub shim.ChaincodeStubInterface, args []string) (string, error) {
    if len(args) != 2 {
            return "", fmt.Errorf("Incorrect arguments. Expecting a key and a value")
    }

    err := stub.PutState(args[0], []byte(args[1]))
    if err != nil {
            return "", fmt.Errorf("Failed to set asset: %s", args[0])
    }
    return args[1], nil
}

func get(stub shim.ChaincodeStubInterface, args []string) (string, error) {
    if len(args) != 1 {
            return "", fmt.Errorf("Incorrect arguments. Expecting a key")
    }

    value, err := stub.GetState(args[0])
    if err != nil {
            return "", fmt.Errorf("Failed to get asset: %s with error: %s", args[0], err)
    }
    if value == nil {
            return "", fmt.Errorf("Asset not found: %s", args[0])
    }
    return string(value), nil
}

// main函数，启动container中的chaincode（在实例化的时候）。
main() {
    if err := shim.Start(new(SimpleAsset)); err != nil {
            fmt.Printf("Error starting SimpleAsset chaincode: %s", err)
    }
}
```

### Building Chaincode

现在让我们编译chaincode。

```c
go get -u --tags nopkcs11 github.com/hyperledger/fabric/core/chaincode/shim
go build --tags nopkcs11
```

假设没有错误，可以继续下一步，测试chaincode。

### 使用Dev模式测试

通常，chaincodes由peer启动和维护。 然而，在“Dev模式”下，链chaincodes由用户构建并启动。 在快速code/build/run/debug周期迭代的开发阶段，此模式非常有用。

我们通过利用，为样例dev网络，预先生成的orderer和channel artifacts，来开始“Dev模式”。 因此，用户可以立即跳到编辑chaincode和调用的过程中。

## 安装 Hyperledger Fabric Samples

如果之前没有做过，请安装Hyperledger Fabric Samples。

到`fabric-samples`目录的`chaincode-docker-devmode` 目录下:

`cd chaincode-docker-devmode`

## 下载 Docker images

我们需要四个Docker镜像，才能使“dev模式”按照提供的docker compose script运行。 
如果您安装了`fabric-samples` repo clone，并按照说明下载平台特定的二进制文件，那么你应该，已经在本地安装了必要的Docker镜像。

使用`docker images`命令，查看本地的Docker Registry。应该如下“

```c
docker images
REPOSITORY                     TAG                                  IMAGE ID            CREATED             SIZE
hyperledger/fabric-tools       latest                           b7bfddf508bc        About an hour ago   1.46GB
hyperledger/fabric-tools       x86_64-1.1.0                     b7bfddf508bc        About an hour ago   1.46GB
hyperledger/fabric-orderer     latest                           ce0c810df36a        About an hour ago   180MB
hyperledger/fabric-orderer     x86_64-1.1.0                     ce0c810df36a        About an hour ago   180MB
hyperledger/fabric-peer        latest                           b023f9be0771        About an hour ago   187MB
hyperledger/fabric-peer        x86_64-1.1.0                     b023f9be0771        About an hour ago   187MB
hyperledger/fabric-javaenv     latest                           82098abb1a17        About an hour ago   1.52GB
hyperledger/fabric-javaenv     x86_64-1.1.0                     82098abb1a17        About an hour ago   1.52GB
hyperledger/fabric-ccenv       latest                           c8b4909d8d46        About an hour ago   1.39GB
hyperledger/fabric-ccenv       x86_64-1.1.0                     c8b4909d8d46        About an hour ago   1.39GB
```

现在打开3个终端，并切换到`chaincode-docker-devmode`目录


## Terminal 1 - Start the network

`docker-compose -f docker-compose-simple.yaml up`

上面的命令使用`SingleSampleMSPSolo`orderer profile启动网络，并以“dev模式”启动peer。 

它还启动了两个额外的容器 - 一个用于chaincode环境，一个用于与chaincode交互的CLI。 创建和加入Channel的命令被嵌入到CLI容器中，因此我们可以立即跳转到chaincode的调用。


## Terminal 2 - Build & start the chaincode

`docker exec -it chaincode bash`

应该看到如下信息：

```c
root@d2629980e76b:/opt/gopath/src/chaincode#
```

现在编译你的chaincode:

```c
cd sacc
go build
```
现在运行chaincode:

```c
CORE_PEER_ADDRESS=peer:7052 CORE_CHAINCODE_ID_NAME=mycc:0 ./sacc
```

chaincode，以peer和chaincode日志开始，表示与peer注册成功。 请注意，在此阶段chaincode不与任何Channel关联。与Channel的关联，是在后续步骤中使用实例化命令完成的。

## Terminal 3 - Use the chaincode

尽管你在 `--peer-chaincodedev` 模式中, 也需要install chaincode，这样 life-cycle system chaincode 可以正常通过它的检查。 当在 `--peer-chaincodedev` 模式中，这个需求可以在后续的版本中被移除掉。

启动 CLI container 来进行调用
`docker exec -it cli bash`

```c
peer chaincode install -p chaincodedev/chaincode/sacc -n mycc -v 0
peer chaincode instantiate -n mycc -v 0 -c '{"Args":["a","10"]}' -C myc
```

现在发起一个`invoke` 把 “a” 的值改成“20”。

```c
peer chaincode invoke -n mycc -c '{"Args":["set", "a", "20"]}' -C myc
```
再查询一下”a“的值

```c
peer chaincode query -n mycc -c '{"Args":["query","a"]}' -C myc
```


## Chaincode加密

在某些情况下，加密与key相关的value（全部加密或者部分加密）可能是有用的。例如，如果一个人的社会安全号码，或地址正在写入ledger，那么你可能不希望这些数据以明文形式出现。 Chaincode加密，通过利用实体扩展实现，该BCCSP包装，执行诸如加密和椭圆曲线数字签名的加密操作。例如，为了加密，Chaincode的调用者通过transient字段传递加密密钥。然后可以将相同的密钥用于随后的查询操作，从而允许对加密的状态值进行适当的解密。

有关更多信息和示例，请参阅`fabric/examples`目录中的`Encc`示例。特别注意`utils.go`帮手程序。此实用工具加载Chaincode shim API和实体扩展，并构建样本加密Chaincode，随后利用新类函数（例如，encryptAndPutState＆getStateAndDecrypt）。因此，chaincode现在可以结合Get和Put的基本shim API（填充API）以及Encrypt和Decrypt的附加功能。


## 管理用Go编写的chaincode的外部依赖关系

如果你的chaincode需要非Go标准库提供的软件包，则需要将这些软件包包含在您的chaincode中。 有许多工具可用于管理这些依赖关系。 以下演示如何使用govendor：

```c
govendor init
govendor add + external //添加所有外部包，或者
govendor add github.com/external/pkg //添加特定的外部软件包
```

这将外部依赖关系导入本地vendor目录。这样 peer chaincode包和peer chaincode安装操作，将会包含与chaincode包相关的代码。


