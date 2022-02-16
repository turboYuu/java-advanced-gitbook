第七部分 K8s高级篇-volume(存储)

[卷-K8s-官网参考](https://kubernetes.io/zh/docs/concepts/storage/volumes/)

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
apiVersion: v1
kind: Secret
metadata:
  name: mariadbsecret
type: Opaque
data:
  password: YWRtaW4=
  # 
  username: cm9vdA==
```



### 3.2.3 升级mariadb的service

```yaml
          env:
            - name: MYSQL_ROOT_PASSWORD
              #这是 mysql root 用户的密码
              valueFrom:
                secretKeyRef:
                  key: password
                  name: mariadbsecret
            - name: TZ
              value: Asia/Shanghai
```



### 3.2.4 全部资源文件清单

secret/mariadbsecret.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadbsecret
type: Opaque
data:
  password: YWRtaW4=
  # 
  username: cm9vdA==
```

secret/mariadb.yml

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
              #这是 mysql root 用户的密码
              valueFrom:
                secretKeyRef:
                  key: password
                  name: mariadbsecret
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

```bash
kubectl delete -f .

kubectl get secret
kubectl get svc
```



## 3.3 安装harbor私服

[参考docker部分的内容](https://github.com/turboYuu/java-advanced-gitbook/blob/master/8-stage/1-module/2-Docker-core-theory.md#53-%E4%BC%81%E4%B8%9A%E7%A7%81%E6%9C%8D)，使用192.168.31.82

### 3.3.1 harbor官网地址

```html
harbor官网地址： 
https://goharbor.io/

github官网地址：
https://github.com/goharbor/harbor
```

### 3.3.2 docker-compose

[参考docker部分的笔记](https://github.com/turboYuu/java-advanced-gitbook/blob/master/8-stage/1-module/2-Docker-core-theory.md#45-docker-compose%E5%AE%89%E8%A3%85)

```bahs
验证docker-compose
docker-compose -v
```

### 3.3.3 安装harbor

```bash
1.解压软件 
cd /data
tar zxf harbor-offline-installer-v1.9.4.tgz

2.进入安装目录 
cd harbor

3.修改配置文件 
vi harbor.yml
  3.1修改私服镜像地址 
  hostname: 192.168.31.82
  3.2修改镜像地址访问端口号 
  port: 5000
  3.3harbor管理员登录系统密码
  harbor_admin_password: Harbor12345
  3.4修改harbor映射卷目录 
  data_volume: /data/harbor
  
4.安装harbor
  4.1执行启动脚本,经过下述3个步骤后，成功安装harbor私服 
  ./install.sh
  4.2准备安装环境：检查docker版本和docker-compose版本
  4.3加载harbor需要的镜像
  4.4准备编译环境
  4.5启动harbor。通过docker-compose方式启动服务
  4.6google浏览器访问harbor私服 
  http://192.168.31.82:5000    
  username: admin
  password: Harbor12345
  
5.关闭harbor服务
docker-compose down [-v]
```

### 3.3.4 新建项目

```bash
在harbor中新建公共项目： 
turbine
```

### 3.3.5 配置私服

```bash
k8s集群master节点配置docker私服：master节点用于上传镜像。其余工作节点暂时不要配置私服地址。

vi /etc/docker/daemon.json
"insecure-registries":["192.168.31.82:5000"]

重启docker服务：
systemctl daemon-reload 
systemctl restart docker
```

### 3.3.6 登录私服

```bash
docker login -u admin -p Harbor12345 192.168.31.82:5000 

退出私服
docker logout 192.168.198.155:5000
```

### 3.3.7 上传mariadb镜像

```bash
docker tag mariadb:10.5.2 192.168.31.82:5000/turbine/mariadb:10.5.2

docker push 192.168.31.82:5000/turbine/mariadb:10.5.2

docker rmi -f 192.168.31.82:5000/turbine/mariadb:10.5.2
```

### 3.3.8 修改mariadb镜像地址

修改 secret/maraiadb.yml 文件，将image地址修改为harbor私服地址

```yaml
image: 192.168.31.82:5000/turbine/mariadb:10.5.2
```

运行服务

```bash
kubectl apply -f .

查看pod信息：发现镜像拉取失败，STATUS显示信息为"ImagePullBackOff" 
kubectl get pods

查看pod详细信息:拉取harbor私服镜像失败。 
kubectl describe pod mariadb-7b6f895b5b-mc5xp

删除服务：
kubectl delete -f .
```



## 3.4 注册私服

使用 kubectl 创建 docker registry 认证的

```bash
语法规则：
kubectl create secret docker-registry myregistrykey --docker-server=REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
例子：
kubectl create secret docker-registry turboharbor --docker-server=192.168.31.82:5000 --docker-username=admin --docker-password=Harbor12345 --docker-email=harbor@turbo.com

k8s集群其余工作节点配置docker私服地址： 
vi /etc/docker/daemon.json
"insecure-registries":["192.168.31.82:5000"]

重启docker服务： 
systemctl daemon-reload 
systemctl restart docker
```



## 3.5 secret升级mariadb

将mariadb镜像修改为harbor私服地址。在创建 Pod 的时候，通过 imagesPullSecrets 引用刚创建的 `myregistrykey`

```yaml

```

### 3.5.1 全部资源文件清单

mariadbsecret.yml

```yaml

```



mariadb.yml

```yaml

```



## 3.6 客户端测试

```bash
IP:192.168.31.61
username:root 
password:admin 
prot: 30036
```



# 4 configmap

# 5 configmap 升级 mariadb

# 6 label操作

# 7 volume

# 8 PV&&PVC

# 9 NFS存储卷





















