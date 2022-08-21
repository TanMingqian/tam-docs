# Kubernetes dashboard 安装

```shell
##1. 下载应用官方的DashBoard模板
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
```

<pre class="language-shell"><code class="lang-shell"><strong>##2、修改DashBoard的Service端口暴露模式为NodePort
</strong>kubectl edit service kubernetes-dashboard -n kubernetes-dashboard</code></pre>

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  selector:
    k8s-app: kubernetes-dashboard
```

```shell
##3、创建Service Account 及 ClusterRoleBinding
vim auth.yaml
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

```shell
kubectl apply -f auth.yaml
```

<pre class="language-shell"><code class="lang-shell"><strong>##4、获取访问 Kubernetes Dashboard所需的 Token
</strong>kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')</code></pre>

访问dashboard ui&#x20;

浏览器访问https://NodeIP:Port，并输入Token
