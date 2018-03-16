# Hyperledger Fabric 源码和镜像下载
在这之前需要下载安装Docker和Go环境，安装简单，不再介绍。

Hyperledger Fabric的源码和镜像下载非常简单，只需要执行下面一个命令：

```curl -sSL https://goo.gl/6wtTN5 | bash -s 1.1.0-rc1
```
上面的命令，是执行下面的脚本
https://github.com/hyperledger/fabric/blob/master/scripts/bootstrap.sh

上面脚本里面的内容就是下载Fabric源码：
*         cryptogen
*         configtxgen
*         configtxlator
*         peer
*         orderer
*         fabric-ca-client
上面的代码，将放到当前目录的bin目录下面。

并且下载下面的镜像：
` hyperledger/fabric-ca          latest              8a6c8c2e2ebf        13 days ago         283MB
hyperledger/fabric-ca          x86_64-1.1.0-rc1    8a6c8c2e2ebf        13 days ago         283MB
hyperledger/fabric-tools       latest              006c689ec08e        13 days ago         1.46GB
hyperledger/fabric-tools       x86_64-1.1.0-rc1    006c689ec08e        13 days ago         1.46GB
hyperledger/fabric-orderer     latest              10afc128d402        13 days ago         180MB
hyperledger/fabric-orderer     x86_64-1.1.0-rc1    10afc128d402        13 days ago         180MB
hyperledger/fabric-peer        latest              6b44b1d021cb        13 days ago         187MB
hyperledger/fabric-peer        x86_64-1.1.0-rc1    6b44b1d021cb        13 days ago         187MB
hyperledger/fabric-javaenv     latest              ea263125afb1        13 days ago         1.52GB
hyperledger/fabric-javaenv     x86_64-1.1.0-rc1    ea263125afb1        13 days ago         1.52GB
hyperledger/fabric-ccenv       latest              65c951b9681f        13 days ago         1.39GB
hyperledger/fabric-ccenv       x86_64-1.1.0-rc1    65c951b9681f        13 days ago         1.39GB
hyperledger/fabric-zookeeper   latest              92cbb952b6f8        3 weeks ago         1.39GB
hyperledger/fabric-zookeeper   x86_64-0.4.6        92cbb952b6f8        3 weeks ago         1.39GB
hyperledger/fa。bric-kafka       latest              554c591b86a8        3 weeks ago         1.4GB
hyperledger/fabric-kafka       x86_64-0.4.6        554c591b86a8        3 weeks ago         1.4GB
hyperledger/fabric-couchdb     latest              7e73c828fc5b        3 weeks ago         1.56GB
hyperledger/fabric-couchdb     x86_64-0.4.6        7e73c828fc5b        3 weeks ago         1.56GB
 `
由上面的列表可以看出，会下载Fabric所依赖的Zookeeper、Kafka、Couchdb等docker镜像。并且镜像文件还挺大。

最后可以把 
` export PATH=<path to download location>/bin:$PATH `
加入到环境变量。

**会遇到最大的问题是，国内连接国外的网络，非常慢，会导致各种问题，例如下载到一半时候网络超时，下载失败等等。最好、最直接的方式是VPN下安装，科学上网的方法推荐使用外国VPN，稳定一些。**

**只要解决了网络问题，本文中的步骤非常简单，而且不会出现各种问题**

更多内容 https://github.com/xiaofateng/knowledge-without-end

