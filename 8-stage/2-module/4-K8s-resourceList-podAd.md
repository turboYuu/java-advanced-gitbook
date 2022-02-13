第四部分 资源清单-pod进阶

# 1 资源清单格式

## 1.1 简介

资源清单有5个顶级字段组成：`apiVersion`、`kind`、`metadata`、`spec`、`status`

```yaml
apiVersion: group/apiversion # 如果没有给定group 名称，那么默认为core，可以使用 kubectl apiversions # 获取当前k8s版本上所有的apiVersion版本信息(每个版本可能不同)
kind: #资源类别 
metadata: #资源元数据
  name
  namespace
  lables
  annotations # 主要目的是方便用户阅读查找 
spec: # 期望的状态（disired state）
status: # 当前状态，本字段由 Kubernetes自身维护，用户不能去定义
```

## 1.2 资源的 apiVersion 版本信息

使用kubectl命令可以查看 apiVersion 各个版本信息

```bash
kubectl api-versions
```

## 1.3 获取字段设置帮助文档

```bash
kubectl explain pod
```

![image-20220213190150129](assest/image-20220213190150129.png)

## 1.4 字段配置格式类型

资源清单中大致可以分为如下几种类型：

<map[String]string><[]string><[]Object>

```yaml
apiVersion <string> #表示字符串类型
metadata <Object> #表示需要嵌套多层字段
labels <map[string]string> #表示由k:v组成的映射 
finalizers <[]string> #表示字串列表 
ownerReferences <[]Object> #表示对象列表 
hostPID <boolean> #布尔类型
priority <integer> #整型
name <string> -required- #如果类型后面接 -required-，表示为必填字段
```



# 2 pod生命周期

## 2.1 案例

![image-20220213190434915](assest/image-20220213190434915.png)

实验需要准备镜像

```bash
docker pull busybox:1.32.0
docker pull nginx:1.17.10-alpine
```

### 2.1.1 initC 案例

initC 特点：

1. initC总是运行到成功完成为止。
2. 每个initC容器都必须在下一个initC启动之前成功完成。
3. 如果initC容器运行失败，K8S集群会不断的重启该pod，知道initC容器成功为止。
4. 如果pod对应的 restartPolicy 为never，它就不会重新启动。

pod/initcpod.yml 文件，***需要准备 busybox:1.32.0镜像***

```yaml

```



pod/initcservice1.yml

```yaml

```



pod/initcservice2.yml

```yaml

```



执行命令

```bash
先查看pod启动情况
kubectl get pods

详细查看pod启动情况
kubectl describe pod myapp-pod

查看myapp-pod中的第一个initContainer日志
kubectl logs myapp-pod -c init-myservice

运行init-myservice服务
kubectl apply -f initcservice1.yml

查看init-myservice服务运行情况
kubectl get svc

查看myapp-pod运行情况，需要耐心等一会，会发现pod的第一个init已经就绪
kubectl get pods

运行init-mydb服务
kubectl apply -f initcservice2.yml

查看init-myservice服务运行情况
kubectl get svc

查看myapp-pod运行情况，需要耐心等一会，会发现pod的两个init已经就绪，pod状态为ready
kubectl get pod -w
```

### 2.1.2 readinessProbe(就绪检测)

容器就绪检测案例，***需要准备 nginx:1.17.10-alpine镜像***

pod/readinessprobepod.yml

```yaml

```



执行命令

```bash
创建pod
kubectl apply -f readinessprobepod.yml

检查pod状态，虽然pod状态显示running但是ready显示0/1，因为就绪检查未通过
kubectl get pods

查看pod详细信息，文件最后一行显示readiness probe failed。。。。
kubectl describe pod readinessprobe-pod

进入pod内部，因为是alpine系统，需要使用sh命令
kubectl exec -it readinessprobe-pod sh

进入容器内目录
cd /usr/share/nginx/html/

追加一个index1.html文件
echo "welcome turbo" >> index1.html 

退出容器，再次查看pod状态，pod已经正常启动
exit
kubectl get pods
```



### 2.1.3 livenessProbe(存活检测)

容器存活检测，***需要准备busybox:1.32.0镜像*** 

pod/livenessprobepod.yml

```yaml

```



执行命令

```bash
创建pod
kubectl apply -f livenessprobepod.yml

监控pod状态变化,容器正常启动
kubectl get pod -w

等待30秒后，发现pod的RESTARTS值从0变为1.说明pod已经重启一次。
```



### 2.1.4 livenessprobe 案例二

容器存活检测案例，***需要准备 nginx:1.17.10-alpine镜像***

pod/livenessprobebeginxpod.yml

```yaml

```

执行命令

```bash
创建pod
kubectl apply -f livenessprobenginxpod.yml

查看pod状态
kubectl get pod

查看容器IP，访问index.html页面。index.html页面可以正常访问。
kubectl get pod -o wide
curl 10.81.58.199

进入容器内部
kubectl exec -it livenessprobenginx-pod sh

删除index.html文件,退出容器
rm -rf //usr/share/nginx/html/index.html
exit

再次监控pod状态，等待一段时间后，发现pod的RESTARTS值从0变为1.说明pod已经重启一次。
kubectl get pod -w

进入容器删除文件一条命令执行rm -rf命令后退出容器。
kubectl exec -it livenessprobenginx-pod --rm -rf /usr/share/nginx/html/index.html

再次监控pod状态，等待一段时间后，发现pod的RESTARTS值从1变为2.说明pod已经重启一次。
kubectl get pod -w

因为liveness监控index.html页面已经被删除，所以pod需要重新启动，重启后又重新创建nginx 镜像。nginx镜像中默认有index.html页面。
```

### 2.1.5 livenessprobe案例三

### 2.1.6 钩子函数案例

# 3 总结pod生命周期

## 3.1 pod 的相位

## 3.2 pod的创建过程

## 3.3 pod生命周期中重要行为

## 3.4 生命周期钩子函数

## 3.5 容器探测

## 3.6 容器的重启策略

## 3.7 pod的终止过程































