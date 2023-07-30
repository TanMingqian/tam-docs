# envoy流量转发

利用istio注入可以为每个pod添加istio-init容器和istio-proxy的容器（常说的sidecar容器），istio-proxy容器中有两个进程，一个是`piolot-agent`，一个是`envoy，`并且可以实现流量的调度



大致的步骤

1、istio-init容器，负责初始化pod的iptables规则，执行完成后退出

2、通过修改后的iptables，将进出pod的网络流量都拦截发送到istio-proxy容器

3、istio-proxy容器实现流量管控、安全认证、负载均衡、监控等功能

