---
title: Golang 学习之 grpc 的使用
slug: go-grpc-usage
categories:
  - Golang
tags:
  - Development language
halo:
  site: https://blog.silentmo.cn
  name: c45b26cd-fbf3-4701-b661-f9c5876c874a
  publish: true
---
# Golang 学习之 grpc 的使用

本文记录了笔者对 gRPC 的学习，主要是为了加深记忆，代码解析那一部分写的不是很好，希望大佬们评论区帮忙补充一下

## Protocol Buffers (协议缓冲区) 介绍
协议缓冲区提供一种语言中立、平台中立、可扩展的机制，用于以前后兼容的方式序列化和结构化数据。

协议缓冲区是定义语言 (在 .proto 文件创建) 组合、数据交互代码通过 proto 编译器生成、特定的语言运行时、以及写入文件 (或通过网络连接发送) 的数据序列化格式

**协议缓冲区优势**
- 紧凑的数据存储
- 快速解析
- 多编程语言的可用性
- 跨语言兼容性
- 自动生成类优化
  
**缓冲区处理数据的流程:**
![protocol-buffers-concepts](https://gallery-lsky.silentmo.cn/i_blog/2025/01//protocol-buffers-concepts.png)

## gRPC 介绍
gPRC是一个现代开源的高性能远程过程调用 (RPC) 框架，可以在任何环境中运行。它可以通过对负载平衡、跟踪、健康检查和身份验证的可插拔支持有效地连接数据中心内和跨数据中心的服务。它也适用于分布式计算的最后一英里，将设备、移动应用程序和浏览器连接到后端服务。

**gRPC 特性**
- 简单的服务定义：使用 Protocl Buffers (二进制序列号工具集和语言) 定义服务
- 快速启动并扩展：一行简单的代码即可安装运行和开发环境，借助框架扩展到每秒数百万个RPC
- 跨语言和平台：gRPC 客户端和服务器可以在各种环境中运行和相互通信；例如，可以使用 Go、Python 或 Ruby 中的客户端轻松地在 Java 中创建 gRPC 服务器
- 双向流和集成身份验证：双向流式传输和完全集成的可插拔身份验证以及基于 HTTP/2 的传输
  
**gRPC 工作流程**
![gRPC工作流程](https://gallery-lsky.silentmo.cn/i_blog/2025/01//gRPC-model.png)
1. 客户端 (gRPC Sub) 调用方法，发起 RPC 调用
2. Protobuf 对请求信息进行对象序列号压缩 (IDL)
3. 服务端 (gRPC Server) 接收到请求，解码请求体，进行业务逻辑处理并返回
4. Protobuf 对响应结果进行对象序列化压缩 (IDL)
5. 客户端接收到服务端响应，解码请求体。回调被调用的方法，唤醒正在等待响应 (阻塞) 的客户端调用并返回结果 

**gRPC 定义服务的四种方法**
- 一元 RPC，其中客户端向服务器发送单个请求并获得单个响应，就像正常的函数调用一样
  ```go
    rpc SayHello(HelloRequest) returns (HelloResponse);
  ```
- 服务器流式 RPC，其中客户端向服务器发送请求并获取流以读回一系列消息。客户端从返回的流中读取，直到没有更多消息为止。gRPC 保证单个 RPC 调用中的消息顺序
  ```go
    rpc LotsOfReplies(HelloRequest) returns (stream HelloResponse);
  ```
- 客户端流式 RPC，其中客户端写入一系列消息并将它们发送到服务器，再次使用提供的流。一旦客户端完成了消息的写入，它就会等待服务器读取它们并返回它的响应。gRPC 再次保证了单个 RPC 调用中的消息顺序
  ```go
    rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse);
  ```
- 双向流式 RPC，双方使用读写流发送一系列消息。这两个流独立运行，因此客户端和服务器可以按照他们喜欢的任何顺序读取和写入：例如，服务器可以在写入响应之前等待接收所有客户端消息，或者它可以交替读取消息然后写入消息，或其他一些读取和写入的组合。保留每个流中消息的顺序
  ```go
    rpc BidiHello(stream HelloRequest) returns (stream HelloResponse);
  ```

## Golang gRPC 代码示例
下载对应的库:
```go
go get google.golang.org/grpc
go get -u github.com/golang/protobuf/{proto,protoc-gen-go}
```
### 定义 Protobuf 文件
helloworld.proto
```go
syntax = "proto3";

option go_package = "google.golang.org/grpc/examples/helloworld/helloworld";

package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```
这个文件定义了一个 `Greeter` 服务，有一个 `SayHello` 的方法，方法接收一个 `Request`，返回一个 `Reply`

编译生成服务器和客户端的 `stub`
```go
protoc -I protos protos/helloworld.proto --go_out=plugins=grpc:src/greeter

// 在 Proto文件目录下执行
protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative *.proto
```
在 helloworld 目录下生成了两个文件，主要看一下 `helloworld_grpc.pb.go` 这个文件的下列代码
```go
type UnimplementedGreeterServer struct {
}

func (UnimplementedGreeterServer) SayHello(context.Context, *HelloRequest) (*HelloReply, error) {
	return nil, status.Errorf(codes.Unimplemented, "method SayHello not implemented")
}
```
该文件定义了一个 `UnimplementedGreeterServer` 结构体，该结构体有一个 `SayHello` 方法，这个方法就是 `.proto` 文件定义的方法

`SayHello` 方法中有两个参数，一个是 `context.Context` 类型 (请求的上下文)，一个是 `*HelloRequest`；两个返回值 `*HelloReply` 和 `error`

### 服务端代码
1. 建立 TCP 连接监听
2. 创建一个 gRPC 服务
3. 注册 gRPC 服务
4. 运行服务
```go 
package main

import (
	"context"
	"flag"
	"fmt"
	"log"
	"net"

	"google.golang.org/grpc"
	pb "google.golang.org/grpc/examples/helloworld/helloworld"
)

var (
	port = flag.Int("port", 50051, "The server port")
)

// server is used to implement helloworld.GreeterServer.
type server struct {
	pb.UnimplementedGreeterServer
}

// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	log.Printf("Received: %v", in.GetName())
	return &pb.HelloReply{Message: "Hello " + in.GetName()}, nil
}

func main() {
    // flag 读取命令行参数
	flag.Parse()
    // 建立 TCP 连接
	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
    // 创建 gRPC 服务
	s := grpc.NewServer()
    // 注册服务，两个参数，将 server 结构体的方法进行注册
	pb.RegisterGreeterServer(s, &server{})
	log.Printf("server listening at %v", lis.Addr())
    // 运行服务
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```
在这段代码中，最主要的是服务注册
```go
pb.RegisterGreeterServer(s, &server{})

// RegisterGreeterServer func
func RegisterGreeterServer(s grpc.ServiceRegistrar, srv GreeterServer) {
	s.RegisterService(&Greeter_ServiceDesc, srv)
}
```
其中 `RegisterGreeterServer` 函数在 `helloworld_grpc.pb.go`，是自动生成的注册服务函数。这个函数很简单，调用了`grpc.ServiceRegistrar` 这个类型的 `RegisterService` 方法。`RegisterService` 方法在 `grpc-go` 的 `server.go` 下被定义，如下：
```go
// 删除了一些判断代码，精简
func (s *Server) RegisterService(sd *ServiceDesc, ss interface{}) {
	s.register(sd, ss)
}

func (s *Server) register(sd *ServiceDesc, ss interface{}) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.printf("RegisterService(%q)", sd.ServiceName)
	if s.serve {
		logger.Fatalf("grpc: Server.RegisterService after Server.Serve for %q", sd.ServiceName)
	}
	if _, ok := s.services[sd.ServiceName]; ok {
		logger.Fatalf("grpc: Server.RegisterService found duplicate service registration for %q", sd.ServiceName)
	}
	info := &serviceInfo{
		serviceImpl: ss,
		methods:     make(map[string]*MethodDesc),
		streams:     make(map[string]*StreamDesc),
		mdata:       sd.Metadata,
	}
	for i := range sd.Methods {
		d := &sd.Methods[i]
		info.methods[d.MethodName] = d
	}
	for i := range sd.Streams {
		d := &sd.Streams[i]
		info.streams[d.StreamName] = d
	}
	s.services[sd.ServiceName] = info
}
```
`RegisterService` 方法又调用了 `register` 方法，看一下 `register` 方法。该方法其实就是加锁的 RPC 对象注册，该方法封装了对象名和对象方法，使得客户端能够直接调用方式并且获得服务端执行方法的返回

### 客户端代码
客户端代码主要是调用一下 `SayHello` 方法
```go
package main

import (
	"context"
	"flag"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	pb "google.golang.org/grpc/examples/helloworld/helloworld"
)

const (
	defaultName = "world"
)

var (
	addr = flag.String("addr", "localhost:50051", "the address to connect to")
	name = flag.String("name", defaultName, "Name to greet")
)

func main() {
	flag.Parse()
	// Set up a connection to the server.
	conn, err := grpc.Dial(*addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewGreeterClient(conn)

	// Contact the server and print out its response.
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	r, err := c.SayHello(ctx, &pb.HelloRequest{Name: *name})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
	log.Printf("Greeting: %s", r.GetMessage())
}
```
`Dial` 构建连接后，通过 `pb.NewGreeterClient(conn)` 建立新连接。而 `pb.NewGreeterClient(conn)` 这个函数返回值是 `GreeterClient`。这是一个接口 (`interface`) 类型，里面是 `SayHello` 方法。这里可以看出，这个就是一个继承，通过接口类型，来继承 `SayHello` 方法
```go
type GreeterClient interface {
	// Sends a greeting
	SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error)
}

type greeterClient struct {
	cc grpc.ClientConnInterface
}

func NewGreeterClient(cc grpc.ClientConnInterface) GreeterClient {
	return &greeterClient{cc}
}

func (c *greeterClient) SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error) {
	out := new(HelloReply)
	err := c.cc.Invoke(ctx, "/helloworld.Greeter/SayHello", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}

```

## 参考
[1] [Protocol buffers Overview](https://www.grpc.io/docs/what-is-grpc/introduction/)
[2] [Introduction to gRPC](https://www.grpc.io/docs/what-is-grpc/introduction/)
