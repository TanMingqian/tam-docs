# Jenkins安装

**环境：kubernetes 1.20**

&#x20;            **nfs挂载 /data/nfs/jenkins**

```yaml
##创建pv pvc
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins
  namespace: jenkins
  labels:
    app: jenkins
spec:
  capacity:          
    storage: 10Gi
  accessModes:       
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain  
  nfs:            #NFS设置
    path: /data/nfs/jenkins   
    server: 192.168.0.112
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: jenkins
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi                     
  selector:
    matchLabels:
      app: jenkins
```

```yaml
##创建SA ClusterRoleBinding
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-admin       #ServiceAccount名
  namespace: jenkins     #指定namespace，一定要修改成你自己的namespace
  labels:
    name: jenkins
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: jenkins-admin
  labels:
    name: jenkins
subjects:
  - kind: ServiceAccount
    name: jenkins-admin
    namespace: jenkins
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

```yaml
##创建svc  两个端口 8080 50000
apiVersion: v1
kind: Service
metadata:
  name: jenkins-http
  namespace: jenkins
  labels:
    app: jenkins
spec:
  type: NodePort
  ports:
  - name: http
    port: 8080          #服务端口
    targetPort: 8080 
    nodePort: 30003          #NodePort方式暴露 Jenkins 端口
  selector:
    app: jenkins
---
apiVersion: v1
kind: Service
metadata:
 name: jenkins
 namespace: jenkins
 labels:
  app: jenkins
spec:
 type: ClusterIP
 ports:
 - name: http
   port: 8080
   targetPort: 8080
 - name: agent
   port: 50000
   targetPort: 50000
 selector:
  app: jenkins
```

```yaml
##创建deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
  labels:
    app: jenkins
spec:
  selector:
    matchLabels:
      app: jenkins
  replicas: 
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins-admin
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts-jdk11
        securityContext:                     
          runAsUser:        #设置以ROOT用户运行容器
          privileged: true   #拥有特权
        ports:
        - name: http
          containerPort: 8080 
        - name: agent
          containerPort: 50000
        resources:
          limits:
            memory: Gi
            cpu: "1000m"
          requests:
            memory: Gi
            cpu: "500m"
        env:
        - name: "JAVA_OPTS"  #设置变量，指定时区和 jenkins slave 执行者设置
          value: " -Xmx1024m
                   -XshowSettings:vm
                   -Dhudson.slaves.NodeProvisioner.initialDelay=0
                   -Dhudson.slaves.NodeProvisioner.MARGIN=50
                   -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85
                   -Duser.timezone=Asia/Shanghai
                 "    
        - name: "JENKINS_OPTS"
          value: "--prefix=/jenkins"         #设置路径前缀加上 Jenkins
        volumeMounts:                        #设置要挂在的目录
        - name: data
          mountPath: /var/jenkins_home
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: jenkins      #设置PVC
```

```shell
##获取token
kubectl logs $(kubectl get pods -n jenkins | awk '{print $1}' | grep jenkins) -n jenkins
```

