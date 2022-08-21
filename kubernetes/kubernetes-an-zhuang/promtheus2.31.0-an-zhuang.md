# Promtheus2.31.0安装

**环境：kubernetes 1.20**

使用阿里云镜像仓库的prometheus镜像&#x20;

https://cr.console.aliyun.com/cn-hangzhou/instances/repositories&#x20;

镜像标签 --- registry.cn-hangzhou.aliyuncs.com/google\_containers/prometheus:v2.31.0



<pre class="language-shell"><code class="lang-shell">##1、创建kube-ops命名空间 
<strong>Kubectl create ns kube-ops</strong></code></pre>

```yaml
##2、创建rbac
apiVersion: v1
kind: ServiceAccount
metadata:
 name: prometheus
 namespace: kube-ops
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
 name: prometheus
rules:
 - apiGroups:
   - ""
   resources:
   - nodes
   - services
   - endpoints
   - pods
   - nodes/proxy
   verbs:
   - get
   - list
   - watch
 - apiGroups:
   - ""
   resources:
   - configmaps
   - nodes/metrics
   verbs:
   - get
 - nonResourceURLs:
   - /metrics
   verbs:
   - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: kube-ops
```

<pre class="language-yaml"><code class="lang-yaml"><strong>##3、创建configmap，用于配置普罗监控策略
</strong>apiVersion: v1
kind: ConfigMap
metadata:
 name: prometheus-config
 namespace: kube-ops
data:
 prometheus.yml: |
         global:
           scrape_interval:     15s
           external_labels:
            monitor: 'codelab-monitor'
         scrape_configs:
           - job_name: 'prometheus'
             scrape_interval: 5s
             static_configs:
              - targets: ['localhost:9090']

</code></pre>

```yaml
##4、创建pv pvc, 用作数据持久化,使用nfs类型挂载
apiVersion: v1
kind: PersistentVolume
metadata:
 name: prometheus
spec:
 capacity:
  storage: 10Gi
 accessModes:
 - ReadWriteOnce
 persistentVolumeReclaimPolicy: Recycle
 nfs:
  server: 192.168.0.111
  path: /data/k8s/prometheus
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: prometheus
 namespace: kube-ops
spec:
 accessModes:
 - ReadWriteOnce
 resources:
  requests:
   storage: 10Gi
```

```yaml
##5、创建deployment
apiVersion: apps/v1
kind: Deployment
metadata:
 name: prometheus
 namespace: kube-ops
 labels:
  app: prometheus
spec:
 selector:
  matchLabels:
   app: prometheus
 template:
  metadata:
   labels:
    app: prometheus
  spec:
   serviceAccountName: prometheus
   containers:
   - image: registry.cn-hangzhou.aliyuncs.com/google_containers/prometheus:v2.31.0
     name: prometheus
     command:
     - "/bin/prometheus"
     args:
     - "--config.file=/etc/prometheus/prometheus.yml"
     - "--storage.tsdb.path=/prometheus"
     - "--storage.tsdb.retention=24h"
     ports:
     - containerPort: 9090
       protocol: TCP
       name: http
     volumeMounts:
     - mountPath: "/prometheus"
       subPath: prometheus
       name: data
     - mountPath: "/etc/prometheus"
       name: config-volume
     resources:
      requests:
       cpu: 100m
       memory: 512Mi
      limits:
       cpu: 500m
       memory: 2048Mi
    securityContext:
     runAsUser: 0
    volumes:
    - name: data
      persistentVolumeClaim:
       claimName: prometheus
    - configMap:
       name: prometheus-config
      name: config-volume
```

浏览器访问https://NodeIP:Port
