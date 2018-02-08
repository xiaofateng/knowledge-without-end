# 使用Scala+IntelliJ IDEA+Sbt搭建开发环境

提示

搭建开发环境常遇到的问题：

1.网络问题，导致sbt插件下载失败，解决方法，找到一个好的网络环境，

或者预先从我提供的网盘中下载jar(链接：http://pan.baidu.com/s/1qWFSTze 密码：lszc)

将下载的.ivy2压缩文件，解压后，放到你的用户目录下。


2.版本匹配问题，版本不匹配会遇到各种问题，解决方法，按照如下版本搭建，

scala(2.10.3)，sbt(0.13)，sbt-assembly(0.11.2)，spark(1.2.0)

3.如果按照本教程 搭建仍不成功，推荐看www.bigdatastudy.cn上我录制的课程Spark开发环境搭建(免费的)

安装Scala

scala下载地址：http://www.scala-lang.org/download/2.10.3.html

本教程中，相关软件下载网盘地址(链接：http://pan.baidu.com/s/1qWFSTze 密码：lszc)

默认安装选项会自动配置环境变量。

如果没有自动配置，进行环境变量配置

SCALA_HOME: C:\Program Files (x86)\scala\

Path后面加上 ;%SCALA_HOME%\bin

 

IntelliJ IDEA的下载，安装

下载地址：https://www.jetbrains.com/idea/

激活码如下：

key:tommy
value:49164-YPNVL-OXUZL-XIWM4-Z9OHC-LF053

key:itey
value:91758-T1CLA-C64F3-T7X5R-A7YDO-CRSN1

 

IntelliJ IDEA常用的设置

在IntellJ/bin/idea64.exe.vmoptions(64位，物理内存大，建议增大)，加大IDEA的启动内存：
-Xms512m
-Xmx1024m
-XX:MaxPermSize=512m

主题和颜色：
Settings – IDE Settings – Appearance – Theme:Darcula
然后把下面override font选项勾上，选择Yahei 14号字体。

编辑器界面字体设置：
可以在Editor – Colors&Fonts – Fonts另存为一个新的主题，并在这个新主题中修改配置。

光标所在行背景颜色：
Editor – Colors&Fonts – General – Caret row，选择蓝色背景，以便具有较大色差。

 

为每个项目指定不同版本的JDK：

IDEA可以为每个项目指定不同版本的JDK，并且需要开发者手动配置项目的所使用的JDK版本。配置方法如下：
单击File | Project Structure菜单项，打开ProjectStructure对话框；
在左侧列表框中，选择SDKs列表项，进入SDK配置页面；
若中间的SDK列表框没有选项，则单击“+”号创建一个JDK列表项；
选择JDK列表项，在SDK ’JDK’选项卡页面中，单击JDK home path项目的浏览按钮，定位JDK安装路径并保存

 

插件安装
File>Settings>Plugins，搜索Scala直接安装，安装完后会提示重新启动。这个插件中有scala和sbt。
无需再单独下载sbt插件。

 
build.sbt文件
使用bt 0.13进行编译

build.sbt文件内容如下：
//导入支持编译成jar包的一些函数(这些函数是sbt-assembly插件中的)

```
import AssemblyKeys._
name := “SparkApp”
version := “1.0”
scalaVersion := “2.10.3”
libraryDependencies ++= Seq(// Spark dependency
“org.apache.spark” % “spark-core_2.10″ % “1.2.0” % “provided”,
“net.sf.jopt-simple” % “jopt-simple” % “4.3”,
“joda-time” % “joda-time” % “2.0”)

//该声明包括assembly plug-in功能
assemblySettings
// 使用 assembly plug-in配置jar
jarName in assembly := “my-project-assembly.jar”
// 从我们的assembly JAR中排除Scala, 因为Spark已经绑定了Scala
assemblyOption in assembly :=(assemblyOption in assembly).value.copy(includeScala = false)

```

进一步配置

要使sbt-assembly插件生效，在project/目录下新建一个文件，列出这个插件的依赖。

新建project/assembly.sbt 增加如下的配置：

addSbtPlugin(“com.eed3si9n” % “sbt-assembly” % “0.11.2”)


自此，环境搭建完毕。

spark的安装，请参考Spark入门三部曲之第一步Spark的安装

Spark程序的开发和运行，请参考Spark入门三部曲之第三步Spark程序的开发和运行

