第七部分 K8s高级篇-volume(存储)

# 1 准备镜像

k8s集群每个node节点需要下载镜像：

```bash
docker pull mariadb:10.5.2
```



# 2 安装mariaDB

## 2.1 部署service

maria/mariadb.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb-deploy
  labels:
    app: mariadb-deploy
spec:
  replicas: 1
  template:
    metadata:
      name: mariadb-deploy
      labels:
        app: mariadb-deploy
    spec:
      containers:
        - name: mariadb-deploy
          image: mariadb:10.5.2
          imagePullPolicy: IfNotPresent
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: admin #这是 mysql root 用户的密码
            - name: TZ
              value: Asia/Shanghai
          args:
            - "--character-set-server=utf8mb4"
            - "--collation-server=utf8mb4_unicode_ci"
          ports:
            - containerPort: 3306
      restartPolicy: Always
  selector:
    matchLabels:
      app: mariadb-deploy
---
apiVersion: v1
kind: Service
metadata:
  name: mariadb-svc
spec:
  selector:
    app: mariadb-deploy
  ports:
    - port: 3306
      targetPort: 3306
      nodePort: 30036
  type: NodePort
```



## 2.2 运行服务

```bash
kubectl apply -f .

kubectl get pod -o wide
```



## 2.3 客户端测试

```bash
IP:192.168.31.61
username:root 
password:admin 
prot: 30036
```



## 2.4 删除 service

```bash
kubectl delete -f mariadb.yml
```



# 3 secret

Secret 解决了密码、token、密钥等敏感数据的配置问题，而不需要把这些敏感数暴露到镜像或者 Pod Spec 中。Secret 可以以 Volume 或者环境变量的方式使用。

Secret 有三种类型：

- Service Account：用来访问 Kubernetes API，由 Kubernetes 自动创建，并且会自动挂载到Pod 的 /run/secrets/kubernetes.io/serviceaccount 目录中。
- Opaque：base64编码格式的Secret，用来存储密码、密钥等
- kubernetes.io/dockerconfigjson：用来存储私有 docker registry 的认证信息

## 3.1 Service Account

Service Account 简称 sa，Service Account 用来访问 Kubernetes API，由 Kubernetes自动创建，并且会自动挂载到 Pod 的 /run/secrets/kubernetes.io/serviceaccount 目录中

```bash
查询命名空间为kube-system的pod信息 
kubectl get pod -n kube-system

进入pod:kube-proxy-48bz4
kubectl exec -it kube-proxy-48bz4 -n kube-system sh 

cd /run/secrets/kubernetes.io/serviceaccount
ls

cat ca.crt
cat namespace
cat token
```



## 3.2 Opaque Secret

Opaque 类型的数据是一个 map 类型，要求 value 是 base64 编码格式。

### 3.2.1 加密解密

使用命令行方式对需要加密的字符串进行加密。例如：mysql数据库的密码。

```bash
对admin字符串进行base64加密:获得admin的加密字符串"YWRtaW4=" 
echo -n "admin" | base64

base64解密：对加密后的字符串进行解密 
echo -n "YWRtaW4=" | base64 -d
```



### 3.2.2 资源文件方式创建

对mariadb数据库密码进行加密

```yaml

```



### 3.2.3 升级mariadb的service

```yaml

```



### 3.2.4 全部资源文件清单

secret/mariadbsecret.yml

```yaml

```

secret/mariadb.yml

```yaml

```



### 3.2.5 运行service

```bash
kubectl apply -f .

kubectl get secret
kubectl get svc
```



### 3.2.6 客户端测试

```bash
IP:192.168.31.61
username:root 
password:admin 
prot: 30036
```



### 3.2.7 删除service、secret

## 3.3 安装harbor私服

## 3.4 注册私服

## 3.5 secret升级mariadb

## 3.6 客户端测试

# 4 configmap

# 5 configmap 升级 mariadb

# 6 label操作

# 7 volume

# 8 PV&&PVC

# 9 NFS存储卷





















