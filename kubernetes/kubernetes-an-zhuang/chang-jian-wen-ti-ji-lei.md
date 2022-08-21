# 常见问题积累

### k8s命名空间termintaing

删除terminating状态命名空间&#x20;

1、kubectl get ns vela-system -o json > vela-system.json&#x20;

2、vim vela-system.json 删除spec{}内容&#x20;

3、curl -k -H "Content-Type:application/json" -X PUT --data-binary @vela-system.json https://192.168.0.111:6443/api/v1/namespaces/vela-system/finalize

```shell
##删除terminating状态命名空间
kubectl get ns vela-system -o json > vela-system.json 
```

```shell
##删除spec{}内容 
vim vela-system.json 
```

```
##调用k8s api 接口删除命名空间
##ip按照实际apiserver ip修改
curl -k -H "Content-Type:application/json" -X PUT --data-binary @vela-system.json https://192.168.0.111:6443/api/v1/namespaces/vela-system/finalize
```

### kubeadm纳管节点失败问题

```
// Some code
error execution phase preflight: couldn't validate the identity of the API Server: could not find a JWS signature in the cluster-info ConfigMap for token ID "narjhw"
```

原因： master没有可用的token

解决：

```shell
 kubeadm token create --print-join-command --ttl=0
```

其中 --ttl=0 表示生成的 token 永不失效. 如果不带 --ttl 参数, 那么默认有效时间为24小时. 在24小时内, 可以无数量限制添加 worker.

### K8s 挂载nfs

#### 在各个节点安装nfs service

<pre class="language-shell"><code class="lang-shell"><strong>##1、yum下载nfs-utils
</strong>Yum install nfs-utils -y</code></pre>

```shell
##2、启动nfs服务
Service nfs-server start
```

挂载问题&#x20;

报错： mount failed: exit status 32

&#x20;原因：服务端主机开启防火墙

<pre class="language-shell"><code class="lang-shell"><strong>##3、关闭防火墙
</strong>Service firewalld stop</code></pre>

报错： mount.nfs: access denied by server while mounting

```
##4、修改/etc/exports
vim /etc/exports
/mountPath     xxx.xxx.xxx.xxx(rw,no_root_squash,async,insecure)     
#e.g.
#/data    192.168.0.*(rw,no_root_squash,async,insecure)
```

注：将/mountPath这个目录共享给192.168.0.\*这些客户机，括号中的参数设置意义为：

ro 该主机对该共享目录有只读权限&#x20;

rw 该主机对该共享目录有读写权限&#x20;

root\_squash 客户机用root用户访问该共享文件夹时，将root用户映射成匿名用户&#x20;

no\_root\_squash 客户机用root访问该共享文件夹时，不映射root用户&#x20;

all\_squash 客户机上的任何用户访问该共享目录时都映射成匿名用户&#x20;

anonuid 将客户机上的用户映射成指定的本地用户ID的用户&#x20;

anongid 将客户机上的用户映射成属于指定的本地用户组ID&#x20;

sync 资料同步写入到内存与硬盘中 async 资料会先暂存于内存中，而非直接写入硬盘 insecure 允许从这台机器过来的非授权访问

### 配置docker代理

```shell
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo vim /etc/systemd/system/docker.service.d/proxy.conf
```

```
##编辑proxy.conf
Environment="HTTP_PROXY=http://192.168.1.6:7890/"
Environment="HTTPS_PROXY=http://192.168.1.6:7890/"
Environment="NO_PROXY=localhost,127.0.0.1,.example.com"
```

```shell
sudo systemctl daemon-reload
sudo systemctl restart docker
```



