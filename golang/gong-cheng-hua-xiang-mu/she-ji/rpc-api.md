# RPC API

**RPC（Remote Procedure Call）**，即**远程过程调用**，是一个计算机通信协议。该协议允许运行于一台计算机的程序调用另一台计算机的子程序，而程序员不用额外地为这个交互作用编程。

服务端实现了一个函数，客户端使用 RPC 框架提供的接口，像调用本地函数一样调用这个函数，并获取返回值。

RPC **屏蔽了底层的网络通信细节**，使得开发人员无需关注网络编程的细节，可以将更多的时间和精力放在业务逻辑本身的实现上，从而提高开发效率。

![](<../../../.gitbook/assets/image (1).png>)

## RPC 调用具体流程

• Client 通过本地调用，调用 Client Stub。

• Client Stub 将参数打包（也叫 Marshalling）成一个消息，然后发送这个消息。&#x20;

• Client 所在的 OS 将消息发送给 Server。&#x20;

• Server 端接收到消息后，将消息传递给 Server Stub。&#x20;

• Server Stub 将消息解包（也叫 Unmarshalling）得到参数。&#x20;

• Server Stub 调用服务端的子程序（函数），处理完后，将最终结果按照相反的步骤返回给 Client。



## gRPC

gRPC 是由 Google 开发的高性能、开源、跨多种编程语言的通用 RPC 框架，基于 HTTP 2.0 协议开发，默认采用 Protocol Buffers 数据序列化协议。

### gRPC 特性

• 支持多种语言，例如 Go、Java、C、C++、C#、Node.js、PHP、Python、Ruby 等。&#x20;

• 基于 IDL（Interface Definition Language）文件定义服务，通过 proto3 工具生成指定语言的数据结构、服务端接口以及客户端 Stub。通过这种方式，也可以将服务端和客户端解耦，使客户端和服务端可以并行开发。&#x20;

• 通信协议基于标准的 HTTP/2 设计，支持双向流、消息头压缩、单 TCP 的多路复用、服务端推送等特性。&#x20;

• 支持 Protobuf 和 JSON 序列化数据格式。Protobuf 是一种语言无关的高性能序列化框架，可以减少网络传输流量，提高通信效率。

![](<../../../.gitbook/assets/image (16).png>)

### grpc服务方法类型

gRPC 支持定义 4 种类型的服务方法，分别是**简单模式、服务端数据流模式、客户端数据流模式和双向数据流模式**。&#x20;

**简单模式（Simple RPC）**：是最简单的 gRPC 模式。客户端发起一次请求，服务端响应一个数据。定义格式为 rpc SayHello (HelloRequest) returns (HelloReply) {}。&#x20;

**服务端数据流模式（Server-side streaming RPC）**：客户端发送一个请求，服务器返回数据流响应，客户端从流中读取数据直到为空。定义格式为 rpc SayHello (HelloRequest) returns (stream HelloReply) {}。&#x20;

**客户端数据流模式（Client-side streaming RPC）**：客户端将消息以流的方式发送给服务器，服务器全部处理完成之后返回一次响应。定义格式为 rpc SayHello (stream HelloRequest) returns (HelloReply) {}。&#x20;

**双向数据流模式（Bidirectional streaming RPC）**：客户端和服务端都可以向对方发送数据流，这个时候双方的数据可以同时互相发送，也就是可以实现实时交互 RPC 框架原理。定义格式为 rpc SayHello (stream HelloRequest) returns (stream HelloReply) {}。

### Protocol Buffers

Protocol Buffers（ProtocolBuffer/ protobuf）是 Google 开发的一套对数据结构进行序列化的方法，可用作（数据）通信协议、数据存储格式等

Protocol Buffers 的主要特性有下面这几个&#x20;

**更快的数据传输速度**：protobuf 在传输时，会将数据序列化为二进制数据，和 XML、JSON 的文本传输格式相比，这可以节省大量的 IO 操作，从而提高数据传输速度。跨平台多语言：protobuf 自带的编译工具 protoc 可以基于 protobuf 定义文件，编译出不同语言的客户端或者服务端，供程序直接调用，因此可以满足多语言需求的场景。&#x20;

**具有非常好的扩展性和兼容性**，可以更新已有的数据结构，而不破坏和影响原有的程序。&#x20;

**基于 IDL 文件定义服务**，通过 proto3 工具生成指定语言的数据结构、服务端和客户端接口。

### RESTful VS gRPC

![](<../../../.gitbook/assets/image (20).png>)

![](<../../../.gitbook/assets/image (12).png>)
