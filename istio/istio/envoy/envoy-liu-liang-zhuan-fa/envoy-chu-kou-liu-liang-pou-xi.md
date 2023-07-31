# envoy出口流量剖析

从上文可以知道，envoy的15001端口负责处理出口流量

istio提供命令行工具istioctl用于查询配置

```
istioctl proxy-config -h
```

根据envoy架构，envoy会新建listerner去监听端口，所以必然存在15001端口的listener

```
istioctl pc listener integration-86924652-bfsfs -n istio-demo --port 15001 -ojson
ADDRESS PORT  MATCH DESTINATION
0.0.0.0 15001 ALL   PassthroughCluster
```

15001端口监听器为virtualOuntbound，virtualOutbound的监听不做请求处理, 直接转到原始的请求对应的监听器



查看端口为9000的listener

```
istioctl pc istener integration-86924652-bfsfs -n istio-demo --port 9000
```

会发现envoy为服务网格istener integration-86924652-bfsfs -n istio-demo --port 15001 -o内所有端口为9000的service建立一个listener











