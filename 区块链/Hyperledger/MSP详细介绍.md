# 简介

MSP的作用，不仅仅在于列出谁是网络参与者或Channel成员。 MSP可以确定，成员在MSP所代表的Org（trust domain）（例如，MSP管理员，组织细分成员）中扮演的特定角色。 它将MSP的配置通告给，相应组织的成员参与的所有Channel（以MSP Channel的形式）。 Peers, orderers 和 clients还维护本地MSP实例（也称为 Local MSP），以在channel环境之外验证其组织成员的消息。 此外，MSP可以识别已被吊销的身份列表。

也就是MSP可以分为：local 和 channel MSPs

**organizations 最重要的是，他们在单独的一个MSP下管理其成员。** 请注意，这与在X.509证书中定义的organization概念不同，我们稍后会讨论。


organization与MSP之间的专一性关系，使得在organization之后命名MSP是明智的，在大多数policy配置中，您会发现这一个惯例。 例如，organization `ORG1`将有一个名为 `ORG1-MSP` 的MSP。 在某些情况下，organization可能需要多个成员资格组 - 例如，不同的Channel用于在organisations之间执行完全不同的业务职能， 在这些情况下，拥有多个MSP并相应地命名它们是有意义的，例如`ORG2-MSP-NATIONAL` 和 `ORG2-MSP-GOVERNMENT`，反映了与政府监管Channel相比，国家销售Channel中ORG2中不同的成员信任root。详细见下图例子：

![](media/MSPOrgRelation.png)

第一种配置显示了MSP和Org之间的典型关系 - 单个MSP定义Org成员的列表。 
在第二种配置中，不同的MSP用于代表具有国家，国际和政府关系的不同组织团体。

# Organizational Units（企业中各管理部门） and MSPs

一个组织经常被分成多个组织单位（organizational units 、OUs)，每个组织单位都有一定的责任。 例如，`ORG1`组织可能同时具有`ORG1-MANUFACTURING`和`ORG1-DISTRIBUTION` OUS，以反映这些单独的业务线。 当CA颁发X.509证书时，证书中的`OU`字段将指定该identity（身份）所属的业务线。

我们稍后会看到`OUs`如何有助于控制，区块链网络成员的组织部分。 例如，只有来自`ORG1-MANUFACTURING` OU的身份可能能够访问某个Channel，而`ORG1-DISTRIBUTION`则不能。

# Local and Channel MSPs

MSP出现在区块链网络中的两个地方：Channel配置（Channel MSP），以及本地（local MSP）。localMSP，是为节点（peer 或 orderer）和用户（使用CLI或使用SDK的客户端应用程序的管理员）定义的。**每个节点和用户都必须定义一个localMSP，因为它定义了谁在该级别和Channel的上下文之外，具有管理或参与权限（例如，谁是peer所在组织的管理员）。**

相反，channel MSP在channel层面定义管理和参与权。参与Channel的每个组织，都必须为其定义MSP。Channel上的Peers 和 orderers将在Channel MSP上共享相同的视图，并且此后将能够正确认证Channel参与者。**这意味着如果一个组织希望加入该Channel，那么需要在Channel配置中，加入一个包含该组织成员的信任链的MSP。否则来自该组织身份的交易将被拒绝。**

local和channel MSP之间的主要区别不在于它们的功能，而在于它们的范围。

![](media/localAndchannelMSPS.png)


Local 和 channel MSP。 **每个peer的信任域（例如，组织）由peer 的Local MSP（例如`ORG1`或`ORG2`）定义。** 组织在Channel中的表示，通过将组织的MSP包含在渠道中来实现。 例如，上图的Channel由`ORG1`和`ORG2`管理。 类似的原则适用于网络，orderers和用户，但为简单起见，这里没有显示。


Local MSP仅在其应用的 节点或用户 的文件系统上定义。 因此，在物理上和逻辑上，每个节点或用户只有一个Local MSP。
 
但是，由于Channel MSP可用于Channel中的所有节点，因此它们在其配置中的Channel中逻辑定义一次。 **但是，Channel MSP在channel中的每个节点的文件系统上,实例化并且通过共识保持同步。** 因此，虽然每个节点的本地文件系统上存在每个channel MSP的副本，但逻辑上，channel MSP驻留在channel或网络上并由其维护。

管理员B使用`RCA1`颁发的，并存储在其local MSP中的身份连接到peer（用户、管理员也有local MSP）。 当B尝试在peer上安装智能合同时，peer检查其local MSP `ORG1-MSP`，以验证B的身份确实是`ORG1`的成员。 成功验证将允许安装命令成功完成。 随后，B希望实例化该channel上的智能合约。 由于这是Channel操作，Channel中的所有组织都必须同意。 因此，peer必须先检查Channel的MSP，然后才能成功提交该命令。 （其他事情也必须发生，但现在要专注于上述内容。）


# MSP Levels

MSP级别
channel 和 local MSP之间的分割反映了，组织管理其本地资源（例如peer or orderer节点）及其Channel资源（例如在Channel或网络级别，运营的ledgers，智能合约和联盟）的需求。 将这些MSP视为处于不同级别是有好处的，其中较高级别的MSP，处理与网络管理有关的问题，而较低级别的MSP，处理私有资源管理的身份。 MSP在每个管理级别都是必须的 - 它们必须为网络，Channel，peer, orderer 和 users 定义。


![](media/MSPLevel.png)

peer 和 orderer的MSP是本地的，而Channel的MSP**（包括网络配置Channel）**在该Channel的所有参与者之间共享。 在上图中，网络配置Channel由ORG1管理，但另一个应用程序Channel可由ORG1和ORG2管理。 

peer是ORG2的成员，并由ORG2管理，而ORG1管理orderer。 ORG1信任来自RCA1的身份，而ORG2信任来自RCA2的身份。 请注意，这些是管理身份，反映了谁可以管理这些组件。 所以当ORG1管理网络时，ORG2.MSP确实存在于网络定义中。


* Network MSP：网络配置通过定义参与者组织MSPs，来定义网络中的成员。同时定义这些成员中哪些成员，有权执行管理任务（例如，创建Channel）


* Channel MSP：Channel分开维护其成员的MSP非常重要。Channel提供了一组特定的组织之间的私人通信，这些组织又对其进行管理控制。**在该Channel的MSP上下文中的Channel policies定义谁能够参与Channel上的某些操作，例如添加组织或实例化chaincodes。**请注意，管理Channel的权限与管理网络配置Channel（或任何其他频道）的权限之间没有必要的关系。管理权限存在于正在管理的范围内（除非规则已另行编写 - 请参阅下面关于ROLE属性的讨论）。


* Peer MSP：此Local MSP在每个peer的文件系统上定义，并且每个peer都有一个MSP实例。从概念上讲，它执行的功能与Channel MSP完全相同，限制条件是它仅用于定义它的peer上。


* Orderer MSP：与peer MSP一样，orderer local MSP也在节点的文件系统上定义，并且仅用于该节点。与peer 节点相似，orderer也由单个组织拥有，因此只有一个MSP来列出其信任的参与者或节点。

# MSP Structure

到目前为止，已经看到MSP中最重要的两个要素，是用于确定相应组织中的参与者或节点成员资格的（root or intermediate）CA的规范。 但是，有更多的元素与这两个元素结合使用来协助membership功能。

![](media/MSPElemment.png)

下面详细介绍下：

## Root CAs
此文件夹包含，由此MSP代表的组织信任的Root CA的，自签名X.509证书列表。此MSP文件夹中必须至少有一个Root CA X.509证书。
这是最重要的文件夹，因为它标识了所有其它证书，必须从中派生出来的CA，以便被视为相应组织的成员。

## Intermediate CAs
此文件夹包含此组织信任的Intermediate CA的X.509证书列表。每个证书都必须由MSP中的一个Root CA签署，或者由 Intermediate CA 签署。

Intermediate CA可以表示组织的不同细分或组织本身（例如，如果商业CA用于组织的身份管理）。在后一种情况下，可以使用CA层次结构中，较低的其他Intermediate CA来表示组织细分。请注意，可能有一个没有任何中间CA的功能网络，在这种情况下，此文件夹将为空。

与Root CA文件夹一样，此文件夹定义了，必须从中颁发证书才能被视为组织成员的CA。

## Organizational Units (OUs)
可选的

## Administrators
该文件夹包含一个身份列表，用于定义具有该组织管理员角色的参与者。对于标准MSP类型，此列表中应该有一个或多个X.509证书。

值得注意的是，仅仅因为演员具有管理员的角色，并不意味着他们可以管理特定的资源！给定身份在管理系统方面的实际权力，取决于管理系统资源的策略。例如，Channel policy可能会指定`ORG1-MANUFACTURING`管理员有权将新组织添加到Channel，而`ORG1-DISTRIBUTION`管理员则没有此权限。

**即使X.509证书具有ROLE属性（例如，指定角色是管理员），这也是指角色在其组织中而不是区块链网络中的角色。**这与OU属性的用途类似 - 指的是参与者在组织中的位置。

如果该Channel的策略已经写入,允许来自组织（或某些组织）的任何管理员,执行某些channel功能（例如实例化chaincode）的权限，则ROLE属性,可用于在channel级别授予管理权限。**通过这种方式，组织角色可以赋予网络角色。这在概念上类似于美国佛罗里达州颁发的驾驶执照,是如何授权某人在美国的每个州开车的。**


## Revoked Certificates:
可选的

## Node Identity（节点身份）
**该文件夹包含节点的自身身份。与KeyStore内容相结合的密码材料，将允许节点，在发送给其channels和network的其他参与者的消息中，对自身进行身份验证。** 对于基于X.509的身份，此文件夹包含一个X.509证书。 例如，这是peer在交易提议响应中的证书，以表明peer已经认可了该证书 - 随后可以在验证时，根据所得到的交易的认可政策，来检查该证书。

该文件夹对于本地MSP是必需的，并且该节点必须只有一个X.509证书。 它不用于Channel MSP。


## KeyStore for Private Key（私钥KeyStore）
该文件夹为peer 或 orderer节点（或客户端的local MSP）的local MSP定义，并包含节点的signing key（签名密钥）。 **此密钥，与上面Node Identity 文件夹中的节点身份匹配，并用于签署数据 - 例如签署交易提议响应，作为认可阶段的一部分。**

该文件夹对Local MSP是必须的，并且必须包含一个私钥。 很明显，访问这个文件夹，只能由，对次peer有管理权限的用户。

Channel MSP的配置不包括此部分，因为Channel MSP旨在提供纯粹的身份验证功能，而不是签署能力。


## TLS Root CA
此文件夹包含，此组织为TLS通信所信任的Root CA的自签名X.509证书列表。 **TLS通信的一个例子是，peer需要连接到orderer以便它可以接收ledger更新。**

MSP TLS信息涉及网络内的节点，即对peers 和 the orderers，而不是那些使用网络的节点 - 应用程序和管理员。

此文件夹中必须至少有一个TLS Root CA X.509证书。

## TLS Intermediate CA
此文件夹包含由此MSP代表的，组织信任的用于TLS通信的Intermediate CA证书列表。当商业CA用于组织的TLS证书时，此文件夹特别有用。 它是可选的。

