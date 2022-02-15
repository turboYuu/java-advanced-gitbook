第七部分 K8s高级篇-volume(存储)

## 1 准备镜像

k8s集群每个node节点需要下载镜像：

```bash
docker pull mariadb:10.5.2
```



## 2 安装mariaDB

## 2.1 部署service

maria/mariadb.yml

```yaml

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



## 3 secret



## 4 configmap

## 5 configmap 升级 mariadb

## 6 label操作

## 7 volume

## 8 PV&&PVC

## 9 NFS存储卷