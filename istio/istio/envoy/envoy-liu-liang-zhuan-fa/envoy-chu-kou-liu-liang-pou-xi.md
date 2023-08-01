# envoy出口流量剖析

<figure><img src="../../../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

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

<pre><code><strong>istioctl pc listener integration-86924652-bfsfs -n istio-demo --port 9000 -ojson
</strong></code></pre>

会发现envoy为服务网格内所有端口为9000的service建立一个listener

```

...
{
       "name": 184.29.159.225_9000",
       "address": {
           "socketAddress": {
               "address": "184.29.159.225",
               "portValue": 9000
           }
       },
       "filterChains": [
           {
               "filterChainMatch": {
                   "applicationProtocols": [
                       "http/1.0",
                       "http/1.1",
                       "h2c"
                   ]
               },
               "filters": [
                   {
                       "name": "envoy.filters.network.http_connection_manager",
                       "typedConfig": {
                           "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                           "statPrefix": "outbound_184.29.159.225_9000",
                           "rds": {
                               "configSource": {
                                   "ads": {},
                                   "resourceApiVersion": "V3"
                               },
                               "routeConfigName": "publictest.istio-demo.svc.cluster.local:9000" #重点是这个
                           },
​
...

```

envoy的15001端口收到请求后，直接转到了`184.29.159.225_9000`，进而转到了`"routeConfigName": publictest.istio-demo.svc.cluster.local:9000"`，即`publictest.istio-demo.svc.cluster.local:9000`这个route中。



使用命令查看`publictest.istio-demo.svc.cluster.local:9000`

<pre><code>// Some code
istioctl pc route  integration-86924652-bfsfs.istio-demo publictest.istio-demo.svc.cluster.local:9000 -ojson

NAME                                             DOMAINS          MATCH     VIRTUAL SERVICE
publictest.istio-demo.svc.cluster.local:9000     publictest       /*        publictest.istio-demo


[
    {
        "name": " publictest.istio-demo.svc.cluster.local:9000",
        "virtualHosts": [
            {
                "name": "allow_any",
                "domains": [
                    "*"
                ],
                "routes": [
                    {
                        "name": "allow_any",
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "PassthroughCluster",
                            "timeout": "0s",
                            "maxGrpcTimeout": "0s"
                        }
                    }
                ],
                "includeRequestAttemptCount": true
            },
            {
                "name": " publictest.istio-demo.svc.cluster.local:9000",
                "domains": [
                    "publictest.istio-demo.svc.cluster.local",
                    "publictest.istio-demo.svc.cluster.local:9000",
                    "publictest",
                    "publictest:9999",
                    "publictest.istio-demo.svc.cluster",
                    "publictest.istio-demo.svc.cluster:9000",
                    "publictest.istio-demo.svc",
                    "publictest.istio-demo.svc:9000",
                    "publictest.istio-demo",
                    "publictest.istio-demo:9000",
                    "10.1.122.241",
                    "10.1.122.241:9000"
                ],
                "routes": [
                    {
                        "name": "publictest-route",
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "weightedClusters": {
                                <a data-footnote-ref href="#user-content-fn-1">"clusters":</a> [
                                    {
                                        "name": "outbound|9000|v1|publictest.istio-demo.svc.cluster.local",
                                        "weight": 90
                                    },
                                    {
                                        "name": "outbound|9000|v2|publictest.istio-demo.svc.cluster.local",
                                        "weight": 10
                                    }
                                ]
                            },
​
...
</code></pre>

满足访问domains列表的会优先匹配到，访问10.111.219.247:9000或者publictest:9999时就会匹配到下面的规则，因此匹配publictest.istio-demo.svc.cluster.local:9000这组虚拟hosts，进而使用到基于weight的集群配置&#x20;

流量按照预期的配置进行了转发

```
90% -> outbound|9000|v1|publictest.istio-demo.svc.cluster.local
10% -> outbound|9000|v2|publictest.istio-demo.svc.cluster.local
```

查看cluster配置&#x20;

```
// Some code
istioctl pc cluster integration-86924652-bfsfs.istio-demo --fqdn outbound|9000|v1|publictest.istio-demo.svc.cluster.local -ojson

        "name": "outbound|9000|v1|publictest.istio-demo.svc.cluster.local",
        "type": "EDS",
        "edsClusterConfig": {
            "edsConfig": {
                "ads": {},
                "resourceApiVersion": "V3"
            },
            "serviceName": "outbound|9000|v1|publictest.istio-demo.svc.cluster.local"
        },

```

endpoint列表是通过eds获取的，因此，查看endpoint信息：

```
// Some code
istioctl pc endpoint integration-86924652-bfsfs.istio-demo  --cluster 'outbound|9000|v1|publictest.istio-demo.svc.cluster.local' -ojson


    {
        "name": "outbound|9000|v1|publictest.istio-demo.svc.cluster.local",
        "addedViaApi": true,
        "hostStatuses": [
            {
                "address": {
                    "socketAddress": {
                        "address": "10.244.0.53",
                        "portValue": 9000
                    }
                },

```

经过envoy的规则，流量从integration的pod中知道要发往`10.244.0.53:9000` 这个pod地址。也就是说实现了请求到目的pod的9000端口的过程。现在相当于在integration的istio-proxy容器里面curl 10.244.0.53:9000地址





[^1]: 
