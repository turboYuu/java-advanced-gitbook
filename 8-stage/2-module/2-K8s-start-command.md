第二部分 K8s快速入门之命令行

# 1 NameSpace介绍

可以认为namespace是你kubernetes集群中的虚拟化集群。在一个Kubernetes集群中可以拥有多个命名空间，它们在逻辑上彼此隔离。可以为你提供组织，安全甚至性能上的帮助。



## 1.1 查看命名空间

```bash
kubectl get namespace
简写命令 
kubectl get ns

查看所有命名空间的pod资源
kubectl get pod --all-namespaces 
kubectl get pod -A
```



### 1.1.1 说明

## 1.2 创建NameSpace

```bash
kubectl create namespace turbo 
简写命令
kubectl create ns turbo
```



## 1.3 删除NameSpace

```bash
kubectl delete namespace turbo 
简写命令
kubectl delete ns turbo
```



# 2 Pod

## 2.1 Pod简介

### 2.1.1 Pod有两个必须知道的特点

### 2.1.2 Pod的工作方式

## 2.2 查看Pod

```bash
查看default命名空间下的pods 
kubectl get pods

查看kube-system命名空间下的pods 
kubectl get pods -n kube-system 

查看所有命名空间下的pods
kubectl get pod --all-namespaces 
kubectl get pod -A
```



## 2.3 创建Pod

### 2.3.1 下载镜像

```bash
K8S集群的每一个节点都需要下载镜像:选择不同的基础镜像，下载镜像的大小也不同。
docker pull tomcat:9.0.20-jre8-alpine        108MB
docker pull tomcat:9.0.37-jdk8-openjdk-slim  305MB
docker pull tomcat:9.0.37-jdk8               531MB

可以自行下载后进行备份。
docker save -o tomcat9.tar tomcat:9.0.20-jre8-alpine
docker load -i tomcat9.tar
```



### 2.3.2 运行pod

```bash
在default命名空间中创建一个pod副本的deployment
kubectl run tomcat9-test --image=tomcat:9.0.20-jre8-alpine --port=8080

kubectl get pod
kubectl get pod -o wide 
使用pod的IP访问容器 
crul ***:8080
```



## 2.4 扩容

```bash
将副本扩容至3个
kubectl scale --replicas=3 deployment/tomcat9-test
kubectl get deployment
kubectl get deployment -o wide
使用deployment的IP访问pod
```



## 2.5 创建服务

```bash
kubectl expose deployment tomcat9-test --name=tomcat9-svc --port=8888 --target-port=8080 --protocol=TCP --type=NodePort

kubectl get svc 
kubectl get svc -o wide 
访问服务端口
curl 10.105.225.0:8888 
访问集群外端口
http://192.168.31.61:30217
```

```
1.练习
pod deployment 扩容    service 案例

2.自行查找一些资料：k8s集群 NodePort默认的端口号范围。。。

3.扩展技能：如何修改k8s集群NodePort默认的端口号范围
```





## 2.6 kubectl常用命令

### 2.6.1 语法规则

### 2.6.2 get命令

### 2.6.3 describe命令

### 2.6.4 delete命令

### 2.6.5 进入容器命令

### 2.6.6 logs命令

### 2.6.7 格式化输出

### 2.6.8 强制删除

## 2.7 资源缩写

### 2.7.1 缩写汇总