# 准备工作

## 安装go
需要版本1.6以上

## 安装gPRC

```c
go get -u google.golang.org/grpc
```

## 安装 Protocol Buffers v3

* 安装 protoc 编译器，用来生成 gRPC service 代码。最简单的方式是下载你的平台对应的 pre-compiled binaries (protoc-<version>-<platform>.zip)，下载地址 https://github.com/google/protobuf/releases
* 解压文件。
* 更新环境变量PATH，加入 protoc binary file
* 然后，为Go安装 protoc 插件。

```c
go get -u github.com/golang/protobuf/protoc-gen-go
```
compiler 插件, protoc-gen-go, 会安装在 $GOPATH/bin目录

# Example演示

example例子位置

```c
$GOPATH/src/google.golang.org/grpc/examples
```

到example目录下

```c
cd $GOPATH/src/google.golang.org/grpc/examples/helloworld
```

protoc编译器，根据.proto文件，生成相应的 .pb.go文件

本例中，helloworld.pb.go 文件已经存在，它包含:

* 生成的 client 和 server 代码。
* 填充, 序列化, 检索 HelloRequest 和 HelloReply message types的代码。

## 运行
编译和运行server 和 client 代码, 使用go run命令。在examples helloworld目录下:

```c
go run greeter_server/main.go
```
另外一个终端:

```c
go run greeter_client/main.go
```
在client端会看到 Hello world。说明运行gRPC服务成功了。

# Go开发
主要介绍内容如下：
* 在 .proto 文件中，定义service。
* 使用protocol buffer compiler 生成server 和 client代码。
* 使用Go gRPC API 写一个简单的client 和 server。

## 定义 service

首先，使用protocol buffers，定义gRPC service 和 method request、 response types。  文件目录`examples/route_guide/routeguide/route_guide.proto`

在 .proto 文件中定义service:

```c
service RouteGuide {
   ...
}
```

## 定义rpc methods
在 service 定义中 定义rpc methods, 指定request 和 response types。gRPC 允许定义四种类型的 service method, 都在RouteGuide service中:

A simple RPC

```c
// Obtains the feature at a given position.
rpc GetFeature(Point) returns (Feature) {}
```

A server-side streaming RPC 

```c
//获得给定矩形内的可用特征。 结果是流式传输而不是立即返回
//（例如，在一个响应消息中使用重复字段），
//因为矩形可能会覆盖大领域并包含大量的功能。
rpc ListFeatures(Rectangle) returns (stream Feature) {}
```

A client-side streaming RPC 

```c
// Accepts a stream of Points on a route being traversed, returning a
// RouteSummary when traversal is completed.
rpc RecordRoute(stream Point) returns (RouteSummary) {}
```

A bidirectional streaming RPC

```c
// Accepts a stream of RouteNotes sent while a route is being traversed,
// while receiving other RouteNotes (e.g. from other users).
rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
```

.proto 同时包含了 protocol buffer message type 定义 Point message type:

```c
// Points 代表经度-纬度 ，以 E7 形式。
// (degrees multiplied by 10**7 and rounded to the nearest integer).
// Latitudes should be in the range +/- 90 degrees and longitude should be in
// the range +/- 180 degrees (inclusive).
message Point {
  int32 latitude = 1;
  int32 longitude = 2;
}
```

## 生成 client 和 server 代码

下一步是从.proto service 定义，生成 gRPC client 和 server 接口 。 
route_guide example目录下运行 :

```c
 protoc -I routeguide/ routeguide/route_guide.proto --go_out=plugins=grpc:routeguide
```
执行完，在routeguide目录中生成route_guide.pb.go文件。
这个文件中包含下面信息：

* 所有的填充、序列化、检索request and response message types的protocol buffer代码。
* clients调用 RouteGuide service中定义的方法的interface type (或者 stub存根)。 
* servers 要实现的interface type, 定义在 RouteGuide service中的方法。

## 创建server

有两部分:
* 实现从service定义生成的service interface : 做真正的工作。
* 运行 gRPC server 监听 client端的requests， 并分发到正确的service实现.

### 实现 RouteGuide
server有一个routeGuideServer struct type， 它实现生成的RouteGuideServer接口:

```c
type routeGuideServer struct {
        ...
}
...

func (s *routeGuideServer) GetFeature(ctx context.Context, point *pb.Point) (*pb.Feature, error) {
        ...
}
...

func (s *routeGuideServer) ListFeatures(rect *pb.Rectangle, stream pb.RouteGuide_ListFeaturesServer) error {
        ...
}
...

func (s *routeGuideServer) RecordRoute(stream pb.RouteGuide_RecordRouteServer) error {
        ...
}
...

func (s *routeGuideServer) RouteChat(stream pb.RouteGuide_RouteChatServer) error {
        ...
}
...
```

#### Simple RPC
routeGuideServer 实现所有的service methods。先看最简单的, GetFeature, 只从client获取一个 Point ，从它的Feature数据库中，返回相应的feature信息。

```c
func (s *routeGuideServer) GetFeature(ctx context.Context, point *pb.Point) (*pb.Feature, error) {
	for _, feature := range s.savedFeatures {
		if proto.Equal(feature.Location, point) {
			return feature, nil
		}
	}
	// No feature was found, return an unnamed feature
	return &pb.Feature{"", point}, nil
}
```

## 启动 server

指定我们想监听client requests 的端口
使用grpc.NewServer()创建一个gRPC server实例。
把我们的service 实现 注册到 gRPC server.
调用 Serve()， 会阻塞直到killed 或 Stop()。

```c
flag.Parse()
lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
if err != nil {
        log.Fatalf("failed to listen: %v", err)
}
grpcServer := grpc.NewServer()
pb.RegisterRouteGuideServer(grpcServer, &routeGuideServer{})
... // determine whether to use TLS
grpcServer.Serve(lis)
```
## 创建client

现在，为我们的RouteGuide service创建客户端. 
完整代码是 `grpc-go/examples/route_guide/client/client.go`.

### 创建 stub
**为了调用后service methods,我们首先创建一个 gRPC channel 来和 server通信。**
我们通过传递server地址 和 端口号到grpc.Dial()，来创建。

```c
conn, err := grpc.Dial(*serverAddr)
if err != nil {
    ...
}
defer conn.Close()
```

可以使用DialOptions来设置 auth credentials (例如 TLS, GCE credentials, JWT credentials等)。

一旦gRPC channel创建了, 我们需要一个 client stub 来执行 RPCs. 我们使用pb包中提供的 NewRouteGuideClient 方法（从.proto文件中生成的）

`client := pb.NewRouteGuideClient(conn)`

### 调用service methods

在 gRPC-Go中, RPCs是 blocking/synchronous 模式，即， RPC 调用阻塞等待server响应。

Simple RPC
调用simple RPC GetFeature 就像调用一个本地方法一样。

```c
feature, err := client.GetFeature(context.Background(), &pb.Point{409146138, -746188906})
if err != nil {
        ...
}
```
如果不报错，可以读取response的消息
`log.Println(feature)`

##运行
在
```c
$GOPATH/src/google.golang.org/grpc/examples/route_guide
```
目录下执行
`go run server/server.go`
在另一个终端执行
`go run client/client.go`

