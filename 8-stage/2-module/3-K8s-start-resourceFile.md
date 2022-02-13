第三部分 K8s快速入门之资源文件

# 1 idea安装k8s插件

## 1.1 idea插件官网地址

```html
https://plugins.jetbrains.com/ 

kubernetes地址：
https://plugins.jetbrains.com/plugin/10485-kubernetes
```

## 1.2 查找对应自己idea版本的k8s插件信息

```html
help->about->查看idea内部版本信息
一定要注意版本信息，否则无法安装
```



### 1.3 离线安装k8s插件

```html
因国外网站网速较慢，在线安装有安装失败的危险。推荐大家下载idea对应版本的插件后，进行离线安装

193.5662.65
settings->plugins->Install Plugin from Disk->插件安装目录 

安装完成后重启idea开发工具
```



# 2 idea配置SSH客户端

目标：在idea中打开终端操作 k8s 集群 master 节点。

下图是 idea 2019.1.3 版本的操作：

![image-20220213170356196](assest/image-20220213170356196.png)

![image-20220213170414555](assest/image-20220213170414555.png)

![image-20220213170455852](assest/image-20220213170455852.png)



## 2.1 idea配置

```
settings->Tools->SSH Configurations->新建
```

## 2.2 使用SSH客户端

```html
Tools->Start SSH session->选择我们刚刚配置的ssh客户端名称
```

## 2.3 新建yml类型文件

idea默认没有yml文件类型。可以通过 new -> file -> 手工输入 *.yml 创建 yml 类型文件。也可以通过配置增加 yml 类型文件。

```

```

![image-20220213160208093](assest/image-20220213160208093.png)

# 3 Remote Host

目标：将idea 工程中的文件上传 k8s 集群 master 节点

## 3.1 idea 配置

```
Tools->Deployment->Configurations->配置Remote Host
```

## 3.2 使用 Remote Host

```html
可以将本工程中的文件上传k8s集群
```



# 4 NameSpace

## 4.1 创建 NameSpace

```html
操作指南：
settings->Editor->Live Template->Kubernetes->查看自动生成的模板信息内容
```

### 4.1.1 turbonamespace.yml

```yaml
apiVersion: v1 
kind: Namespace 
metadata:
  name: turbo
```

通过idea的Remote Host快速将 yml文件上传 k8s 集群进行测试

```bash
mkdir -p /data/namespaces 
cd /data/namespaces

kubectl apply -f turbonamespace.yml
```

## 4.2 删除NameSpace

```bash
kubectl delete -f turbonamespace.yml
```



# 5 pod

## 5.1 创建pod

在idea工程 resource/pod/tomcatpod.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tomcat-pod
  labels:
    app: tomcat-pod
spec:
  containers:
    - name: tomcat-pod
      image: tomcat:9.0.20-jre8-alpine
      imagePullPolicy: IfNotPresent
  restartPolicy: Always
```

## 5.2 镜像下载策略、重启策略

```bash
imagePullPolicy:
   Always:总是拉取pull
   IfNotPresent:如果本地有镜像，使用本地，如果本地没有镜像，下载镜像。
   Never:只使用本地镜像，从不拉取
```



```bash
restartPolicy:
   Always：只要退出就重启。
   OnFailure：失败退出时（exit code不为0）才重启
   Never：永远不重启
```



## 5.2 运行 pod

```bash
kubectl apply -f tomcatpod.yml
```

## 5.4 测试 pod

```bash

```

## 5.5 删除pod

```bash
kubectl delete -f tomcatpod.yml
```



# 6 deployment

## 6.1 创建deployment

在idea工程 resource/deployment/tomcatdeployment.yml

```yaml

```



matchLabels

```

```



## 6.2 运行deployment

```bash

```



## 6.3 控制器类型

| 控制器类型 | 作用 |
| ---------- | ---- |
|            |      |
|            |      |
|            |      |
|            |      |
|            |      |
|            |      |

## 6.4 Deployment控制器介绍



## 6.5 删除 Deployment 

```bash

```



# 7 service

## 7.1 创建service

在idea工程创建

```yaml

```

### 7.1.1 service 的 selector

### 7.1.2 service类型

### 7.1.3 service参数

## 7.2 运行service

## 7.3 删除service























