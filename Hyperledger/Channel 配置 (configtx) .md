# Channel 配置(configtx)

**Hyperledger Fabric区块链网络的共享配置，存储在collection configuration transactions中，每个Channel一个。** 每个configuration transaction 通常使用一个较短的名称`configtx`。

Channel配置具有以下重要属性：

* Versioned（版本化）：配置文件中的所有元素，都有一个关联的版本，每次修改都会更新版本号。 此外，每个提交的配置都会收到一个序列号。

* Permissioned（权限）：配置文件中的每个元素，都有一个关联的策略，用于控制是否允许对该元素进行修改。 任何拥有之前configtx副本（并且没有其他额外的信息）的人，都可以根据这些策略验证新配置的有效性。

* Hierarchical（分层）：根配置组包含子组，并且每个层次组都有关联的值和策略。 这些政策可以利用层次结构，从较低层次的政策中生成较高层次的政策（These policies can take advantage of the hierarchy to derive policies at one level from policies of lower levels）。


## 详解一个配置

配置作为`HeaderType_CONFIG`类型的transaction，存储在没有其它transaction的block（块）中。 这些块被称为Configuration Blocks（配置块），其中第一块被称为Genesis Block（创世区块）。

配置的原型结构(proto structures)存储在`fabric/protos/common /configtx.proto`中。` HeaderType_CONFIG`类型的Envelope(信封、包、封装)将`ConfigEnvelope`消息编码为`Payload`数据字段。 `ConfigEnvelope`的原型定义如下：

```c
message ConfigEnvelope {
    Config config = 1;
    Envelope last_update = 2;
}
```

`last_update`字段在下面的 “Updates to configuration(更新到配置)” 部分中定义，它仅在验证配置时需要。 相反，当前提交的配置，存储在`config`字段中，其中包含`Config`消息。

```c
message Config {
    uint64 sequence = 1;
    ConfigGroup channel_group = 2;
}
```

对于每个已提交的配置，序号(sequence numbe)都加1。 `channel_group`字段是包含配置的根group。 `ConfigGroup`结构是递归定义的，并构建一个group的树，其中每个group都包含值和策略。 它被定义如下：

```c
message ConfigGroup {
    uint64 version = 1;
    map<string,ConfigGroup> groups = 2;
    map<string,ConfigValue> values = 3;
    map<string,ConfigPolicy> policies = 4;
    string mod_policy = 5;
}
```

由于`ConfigGroup`是一个递归结构，它具有分层结构。 下面的例子是为了使golang符号更清晰。

```c
// 假设下面的 groups 如下定义
//var root, child1, child2, grandChild1, grandChild2, grandChild3 *ConfigGroup

// 设置下面的 values
root.Groups["child1"] = child1
root.Groups["child2"] = child2
child1.Groups["grandChild1"] = grandChild1
child2.Groups["grandChild2"] = grandChild2
child2.Groups["grandChild3"] = grandChild3

// groups的config structure 结果像下面：
// root:
//     child1:
//         grandChild1
//     child2:
//         grandChild2
//         grandChild3
```

每个group都在配置层次结构中定义一个级别，每个group都有一组关联的value（由字符串key索引）和Policies（策略，也由字符串key索引）。

values由以下定义：

```c
message ConfigValue {
    uint64 version = 1;
    bytes value = 2;
    string mod_policy = 3;
}
```
Policies 由以下定义:

```c
message ConfigPolicy {
    uint64 version = 1;
    Policy policy = 2;
    string mod_policy = 3;
}
```

请注意，Values, Policies, 和 Groups都有一个版本和一个`mod_policy`。 每次元素被修改时，元素的版本都会增加。   `mod_policy`用于管理修改该元素所需的签名(signatures)。 

对于Group，修改是向Values, Policies, or Groups maps添加或删除元素（或更改`mod_policy`）。 

对于Values and Policies，修改是分别更改`Value` 和 `Policy`字段（或更改`mod_policy`）。 每个元素的`mod_policy`都是在当前级别config的上下文中进行评估的。 

考虑在`Channel.Groups [“Application”]`处定义的以下示例mod policies（mod策略）（这里，我们使用golang map参考语法，因此`Channel.Groups [“Application”].Policies [“policy1”]`引用基本`Channel` group的`Application ` group的`Policies` 的 `policy1` policy。）

* `policy1` maps to `Channel.Groups["Application"].Policies["policy1"]`
* `Org1/policy2` maps to `Channel.Groups["Application"].Groups["Org1"].Policies["policy2"]`
* `/Channel/policy3` maps to `Channel.Policies["policy3"]`

注意，如果 mod_policy 引用一个不存在的 policy , 条目不会被修改。

## Configuration 更新

Configuration更新，作为`HeaderType_CONFIG_UPDATE`类型的`Envelope`(信封）消息被提交。 transaction的`Payload`数据是一个编组的`ConfigUpdateEnvelope`。 `ConfigUpdateEnvelope`定义如下：

```c
message ConfigUpdateEnvelope {
    bytes config_update = 1;
    repeated ConfigSignature signatures = 2;
}
```

`signatures`字段，包含授权配置更新的一组signatures（签名）。 其消息定义是：

```c
message ConfigSignature {
    bytes signature_header = 1;
    bytes signature = 2;
}
```

`signature_header`与标准transactions定义的一样，但signature位于`signature_header`字节和`ConfigUpdateEnvelope`消息的`config_update`字节的连接之上。

`ConfigUpdateEnvelope` `config_update`字节是封装的（marshaled）`ConfigUpdate`消息，其定义如下：

```c
message ConfigUpdate {
    string channel_id = 1;
    ConfigGroup read_set = 2;
    ConfigGroup write_set = 3;
}
```

channel_id是，更新需要绑定的Channel ID。

* `read_set`指定现有配置的一个子集，指定了，只有版本字段，且没有其它字段必须填充的稀疏部分。`read_set`中不应该设置特定的`ConfigValue`值或`ConfigPolicy`策略字段。 
`ConfigGroup`可能会填充其映射字段的子集，以便引用配置树中更深的元素。 例如，要将`Application` group 包含在`read_set`中，它的父级（`Channel` group）也必须包含在`read_set`中，但`Channel` group不需要填充所有keys，例如`Orderer` group key， 或任何`values` 或 `policies `keys。

* `write_set`指定修改的配置片段。 由于配置的层次性，对层次结构中深层元素的写入操作,也必须在其`write_set`中包含更高级别的元素。 但是，对于`read_set`中同样版本中指定的`write_set`中的任何元素，该元素应该被指定为稀疏，就像在`read_set`中一样。例如，给定配置：

```c
Channel: (version 0)
    Orderer (version 0)
    Appplication (version 3)
       Org1 (version 2)
```

提交一个修改 `Org1` 的configuration update, `read_set` 是:

```c
Channel: (version 0)
    Application: (version 3)
```
`write_set`是：

```c
Channel: (version 0)
    Application: (version 3)
        Org1 (version 3)
```

当收到`CONFIG_UPDATE`时，orderer通过执行以下操作来计算生成的`CONFIG`：

1. 验证`channel_id`和`read_set`。 `read_set`中的所有元素都必须存在于给定的版本中。
2. 通过收集`write_set`中所有不在read_set中显示为相同版本的元素来计算`update set`。
3. 验证`update set`中的每个元素，通过加1更新元素的版本号。
4. 验证附加到`ConfigUpdateEnvelope`的`signature set`，满足`update set`中每个元素的`mod_policy`。
5. 通过将`update set`应用于当前配置，来计算新的完整配置版本。
6. 将新配置写入一个`ConfigEnvelope`，其中包括`CONFIG_UPDATE`作为`last_update`字段，以及配置字段中编码的新配置，以及递增的序列值。
7. 将新的`ConfigEnvelope`写入`CONFIG`类型的`Envelope`中，并最终将其作为单个事务写入新configuration block中。

当peer（或任何其它`Deliver`的receiver）接收到此configuration block时，它应该通过，将`last_update`消息应用于当前配置，并验证`orderer-computed` `config`字段包含正确的新配置，来验证配置是否有效。


## Permitted configuration groups and values(允许的配置组和值)

任何有效的配置都是以下配置的一个子集。 这里我们使用记号`peer.<MSG>`来定义一个`ConfigValue`，其值域是在`fabric/protos/peer/configuration.proto`中定义的名为<MSG>的封装的proto消息。

 `common.<MSG>` `msp.<MSG>`和`orderer.<MSG>`与上面的类似，但是它们的消息在
`fabric/protos / common/configuration.proto`，
`fabric/protos/msp/ mspconfig.proto`，
`fabric/protos/orderer/ configuration.proto`中相应定义。

请注意，关键字`{{org_name}}`和`{{consortium_name}}`表示任意名称，并表示可能以不同名称重复的元素。

```c
&ConfigGroup{
    Groups: map<string, *ConfigGroup> {
        "Application":&ConfigGroup{
            Groups:map<String, *ConfigGroup> {
                {{org_name}}:&ConfigGroup{
                    Values:map<string, *ConfigValue>{
                        "MSP":msp.MSPConfig,
                        "AnchorPeers":peer.AnchorPeers,
                    },
                },
            },
        },
        "Orderer":&ConfigGroup{
            Groups:map<String, *ConfigGroup> {
                {{org_name}}:&ConfigGroup{
                    Values:map<string, *ConfigValue>{
                        "MSP":msp.MSPConfig,
                    },
                },
            },

            Values:map<string, *ConfigValue> {
                "ConsensusType":orderer.ConsensusType,
                "BatchSize":orderer.BatchSize,
                "BatchTimeout":orderer.BatchTimeout,
                "KafkaBrokers":orderer.KafkaBrokers,
            },
        },
        "Consortiums":&ConfigGroup{
            Groups:map<String, *ConfigGroup> {
                {{consortium_name}}:&ConfigGroup{
                    Groups:map<string, *ConfigGroup> {
                        {{org_name}}:&ConfigGroup{
                            Values:map<string, *ConfigValue>{
                                "MSP":msp.MSPConfig,
                            },
                        },
                    },
                    Values:map<string, *ConfigValue> {
                        "ChannelCreationPolicy":common.Policy,
                    }
                },
            },
        },
    },

    Values: map<string, *ConfigValue> {
        "HashingAlgorithm":common.HashingAlgorithm,
        "BlockHashingDataStructure":common.BlockDataHashingStructure,
        "Consortium":common.Consortium,
        "OrdererAddresses":common.OrdererAddresses,
    },
}
```

## Orderer system channel configuration

ordering system channel需要定义ordering参数, 和用于创建channel的联盟。 一个 ordering service必须有一个ordering system channel，并且它是第一个要创建的Channel（或更精确地说是bootstrapped）。 建议不要在ordering system channel生成配置中定义Application部分，但在测试时候，可以添加。 请注意，任何对ordering system channel具有读权限的成员，都可以看到所有Channel创建，因此应该限制Channel的访问权限。

ordering 参数被定义为config的以下子集：

```c
&ConfigGroup{
    Groups: map<string, *ConfigGroup> {
        "Orderer":&ConfigGroup{
            Groups:map<String, *ConfigGroup> {
                {{org_name}}:&ConfigGroup{
                    Values:map<string, *ConfigValue>{
                        "MSP":msp.MSPConfig,
                    },
                },
            },

            Values:map<string, *ConfigValue> {
                "ConsensusType":orderer.ConsensusType,
                "BatchSize":orderer.BatchSize,
                "BatchTimeout":orderer.BatchTimeout,
                "KafkaBrokers":orderer.KafkaBrokers,
            },
        },
    },
```

参与ordering的每个organization都在Orderer组下有一个group元素。 该group定义了单个参数MSP，其中包含该organization的加密身份信息。 Orderer group的值决定了ordering节点的功能。 它们存在于每个channel中，因此orderer.BatchTimeout可以在不同的Channel上有不同的值。

在启动时，orderer面对的是包含许多Channel信息的文件系统。 orderer通过识别定义consortiums group的Channel，来识别system channel。 consortiums group具有以下结构。

```c
&ConfigGroup{
    Groups: map<string, *ConfigGroup> {
        "Consortiums":&ConfigGroup{
            Groups:map<String, *ConfigGroup> {
                {{consortium_name}}:&ConfigGroup{
                    Groups:map<string, *ConfigGroup> {
                        {{org_name}}:&ConfigGroup{
                            Values:map<string, *ConfigValue>{
                                "MSP":msp.MSPConfig,
                            },
                        },
                    },
                    Values:map<string, *ConfigValue> {
                        "ChannelCreationPolicy":common.Policy,
                    }
                },
            },
        },
    },
},
```

请注意，每个consortium（联盟）都会定义一组members（成员），就像ordering orgs的organizational members一样。 每个联盟还定义一个`ChannelCreationPolicy`，这是一个应用于授权Channel创建请求的策略。 通常，此值将设置为`ImplicitMetaPolicy`，要求Channel的新成员签名，以授权Channel的创建。 有关Channel创建的更多细节将在本文后面介绍。

## Application channel configuration
Application配置，适用于为Application类型的transactions设计的Channel。它被定义如下：

```c
&ConfigGroup{
    Groups: map<string, *ConfigGroup> {
        "Application":&ConfigGroup{
            Groups:map<String, *ConfigGroup> {
                {{org_name}}:&ConfigGroup{
                    Values:map<string, *ConfigValue>{
                        "MSP":msp.MSPConfig,
                        "AnchorPeers":peer.AnchorPeers,
                    },
                },
            },
        },
    },
}
```

就像Orderer部分一样，每个organization都被编码为一个group。 然而，不是仅对MSP身份信息进行编码，而是每个org还编码一列 `AnchorPeers`。 该列表允许不同org的peer彼此联系，以进行peer gossip networking。

application channel对orderer orgs和consensus options的副本，进行编码以允许确定性地更新这些参数，因此包含了与orderer system channel configuration相同的Orderer部分。 但从应用角度来看，这可能在很大程度上被忽略。


## Channel creation

当orderer收到一个不存在的Channel的CONFIG_UPDATE时，orderer就认为这是channel creation请求并执行以下操作。

1. orderer识别要执行channel creation请求的`Consortium`。它通过查看上级group 的Consortium value来完成。
2. orderer验证包含在`Application` group 中的organizations，是相应consortium中包含的organizations的子集，并且`ApplicationGroup`被设置为版本1。
3. orderer验证，如果consortium拥有members，新Channel也有application members（创建没有member的consortium和Channel 仅对测试有用）。
4. orderer通过ordering system channel中获取Orderer group，创建一个模板configuration。使用新指定的members，创建`Application` group。为`ChannelCreationPolicy`指定`mod_policy`，如consortium config中指定的那样。请注意，policy是在新配置的上下文中进行评估的，因此需要所有members的策略，需要来自所有新Channel成员的签名，而不是所有consortium成员的签名。
5. orderer然后将`CONFIG_UPDATE`作为此模板配置的更新应用。由于`CONFIG_UPDATE`对Application group（其版本为1）进行了修改，配置代码根据`ChannelCreationPolicy`验证这些更新。如果Channel创建包含任何其它修改，例如对单个org的anchor peers，则将调用该元素的相应mod policy。
6. 具有新Channel配置的新`CONFIG` transaction 将被打包并发送，以便在ordering system channel上进行ordering。ordering后，channel就创建了。


欢迎加入区块链技术交流QQ群 694125199

更多区块链知识：
https://github.com/xiaofateng/knowledge-without-end/tree/master/区块链/Hyperledger

本文参考
http://hyperledger-fabric.readthedocs.io/en/latest/configtx.html


