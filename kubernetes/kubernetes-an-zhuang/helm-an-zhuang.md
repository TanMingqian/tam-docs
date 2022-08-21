# Helm安装

<pre class="language-shell"><code class="lang-shell"><strong>##下载helm安装包
</strong><strong>wget https://get.helm.sh/helm-v3.7.2-linux-amd64.tar.gz</strong></code></pre>

解压后将可执⾏⽂件 helm 拷⻉到 /usr/local/bin ⽬录下

```shell
##验证安装是否成功
helm version
```

```
##更新国内chart源
helm repo add stable http://mirror.azure.cn/kubernetes/charts/
helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
```

```shell
##更新本地仓库
helm repo update
```

```shell
##使用helm安装
helm install  stable/xxxxx
```

