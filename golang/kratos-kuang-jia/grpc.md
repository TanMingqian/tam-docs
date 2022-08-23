# grpc

## grpc使用

### grpc建连(直连

```go
conn, err := transgrpc.DialInsecure(
		context.Background(),
		transgrpc.WithEndpoint("127.0.0.1:9000"),
		transgrpc.WithMiddleware(
			recovery.Recovery(),
		),
	)
```

### grpc建连(服务发现

```go
zkconn, _, err := zk.Connect([]string{"127.0.0.1:2181"}, time.Duration(time.Second * 10))
if err != nil {
		panic(err)
}
r := zookeeper.New(zkconn)
conn, err := grpc.DialInsecure(
		context.Background(),
		grpc.WithEndpoint("discovery:///helloworld"),
		grpc.WithDiscovery(r),
	)
```

经过抓包验证，执行上述代码grpc.DialInsecure() 后，程序会通过zookeeper获取所有服务端实例，并分别与所有实例建立长连接

### 初始化grpc client

```go
client := helloworld.NewGreeterClient(conn)
```

### 调用客户端存根

```go
reply, err := client.SayHello(context.Background(), &helloworld.HelloRequest{Name: "kratos"})
```





