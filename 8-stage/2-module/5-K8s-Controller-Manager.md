第五部分 K8s-pod控制器进阶

# 1 简介

Controller Manager 由 kube-controller-manager 和 cloud-controller-manager 组成，是Kubernetes 的大脑，它通过 apiserver 监控整个集群的状态，并确保集群处于预期的工作状态。

**kube-controller-manager 由一些列的控制器组成**

```bash
1 Replication Controller
2 Node Controller
3 CronJob Controller
4 DaemonSet Controller
5 Deployment Controller
6 Endpoint Controller
7 Garbage Collector
8 Namespace Controller
9 Job Controller
10 Pod AutoScaler
11 RelicaSet
12 Service Controller
13 ServiceAccount Controller
14 StatefulSet Controller
15 Volume Controller
16 Resource quota Controller
```

**cloud-controller-manager 在 Kubernetes 启用 Cloud Provider 的时候才需要，用来配合云服务提供商的控制，也包括一些列的控制器**

```bash
1 Node Controller
2 Route Controller
3 Service Controller
```

从 v1.6 开始，cloud provider已经经历了几次重大重构，以便在不修改 Kubernetes 核心代码的同时构建自定义的云服务商支持。

# 2 常见 Pod 控制器及其含义

1. ReplicaSet：适合无状态的服务部署

   用户创建指定数量的pod副本数量，确保 pod 副本数量符合预期状体，并且支持滚动式自动扩容和缩容功能。<br>ReplicaSet 主要由三个组件组成：

   - 用户期望的 pod 副本数量
   - 标签选择器，判断哪个 pod 归自己管理
   - 当现存的 pod 数量不足，会根据 pod 资源模板进行新建
   
   帮助用户管理无状态的 pod 资源，精确反映用户自定义的目标数量，**但是 ReplicaSet 不是直接使用的控制器，而是使用Deployment**。
   
2. deployment：适合无状态的服务部署

   工作在ReplicaSet之上，用于管理无状态应用，目前来说最好的控制器，支持滚动更新和回滚功能，还提供声明式配置。

3. StatefullSet：适合有状态的服务部署。需要学完存储卷后进行系统学习

4. Daemon：一次部署，所有的node节点都会部署，例如一些典型的应用场景：

   - 运行集群存储 daemon，例如在每个Node上运行 glusterd、ceph
   - 在每个Node上运行日志收集daemon，例如fluentd、logstash
   - 在每个Node上运行监控 daemon，例如 Prometheus Node Exporter

   用于确保集群中的每一个节点只运行特定的 pod 副本，通常用于实现系统级后台任务，比如ELK 服务

   特性：服务是无状态的，服务必须是守护进程

5. Job：一次性的执行任务。只要完成就立即退出，不需要重启或重建

6. Cronjob：周期性的执行任务。周期性任务控制，不需要持续后台运行。

# 3 使用镜像

```bash
演示pod的控制器升级使用：
docker pull nginx:1.17.10-alpine
docker pull nginx:1.18.0-alpine
docker pull nginx:1.19.2-alpine
```



# 4 replication Controller 控制器

replication controller 简称 RC，是 Kubernetes 系统中的核心概念之一，简单说，他其实定义了一个期望的场景，即声明某种pod的副本数量在任意时刻都符合某个预期值，所以 RC 的定义包含以下部分：

- pod 期待的副本数量
- 用于筛选目标 pod 的 Label Selector
- 当 pod 的副本数量小于期望值时，用于创建新的 pod 的 pod模板（template）

# 5 ReplicaSet

Replication Controller 用来确保容器应用额副本数始终保持在用户定义的副本数，即如果有容器异常退出，会自动创建新的Pod来替代；而如果异常多出来的容器也会自定回收。

在新版本的 Kubernetes 中建议使用 ReplicaSet 来取代 ReplicationController。ReplicaSet 和 ReplicationController没有本质的不同，只是名字不一样，并且 ReplicaSet 支持集合式的 selector。

虽然ReplicaSet可以独立使用，但一般还是建议使用 Deployment 来自动管理 ReplicaSet，这样就无需担心跟其他机制的不兼容问题（比如 ReplicaSet不支持 rolling-update，但Deployment支持）

## 5.1 ReplicaSet模板说明

```yaml
apiVersion: apps/v1 # api版本定义
kind: ReplicaSet   # 定义资源类型为ReplicaSet
metadata: # 元数据定义
  name: myapp
  namespace: default
  labels:
    app: myapp
spec: # ReplicaSet的规格定义
  replicas: 2 # 定义副本数量为2
  template: # pod的模板定义
    metadata: # pod的元数据定义
      name: myapp # 自定义pod的名称
      labels: # 定义pod的标签，需要和上面定义的标签一致，也可以多出其他标签
        app: myapp
        release: canary
        environment: qa
    spec: # pod 的规格定义
      containers: # 容器定义
        - name: myapp # 容器名称
          image: nginx:1.17.10-alpine # 容器镜像
          imagePullPolicy: IfNotPresent
          ports: # 暴露端口
            - containerPort: 80
              name: http
      restartPolicy: Always
  selector:  # 标签选择器，定义匹配 pod 的标签
    matchLabels:
      app: myapp
      release: canary
```



可以通过kubectl命令行方式获取更加详细信息

```bash
kubectl explain rs
kubectl explain rs.spec
kubectl explain rs.spec.template.spec
```



## 5.2 部署ReplicaSet

controller/replicasetdemo.yml

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replicasetdemo
  labels:
    app: replicasetdemo
spec:
  replicas: 3
  template:
    metadata:
      name: replicasetdemo
      labels:
        app: replicasetdemo
    spec:
      containers:
        - name: replicasetdemo
          image: nginx:1.17.10-alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
      restartPolicy: Always
  selector:
    matchLabels:
      app: replicasetdemo
```



## 5.3 运行ReplicaSet

```bash
运行ReplicaSet
kubectl apply -f replicasetdemo.yml 

查看rs控制器
kubectl get rs 

查看pod信息
kubectl get pod 

查看pod详细信息
kubectl describe pod replicasetdemo-7fdd7b5f67-5gzfg

测试controller控制器下的pod删除、重新被controller控制器拉起
kubectl delete pod --all 
kubectl get pod

修改pod的副本数量:通过命令行方式
kubectl scale replicaset replicasetdemo --replicas=8 
kubectl get rs

修改pod的副本数量:通过资源清单方式
kubectl edit replicasets.apps replicasetdemo 
kubectl get rs

显示pod的标签
kubectl get pod --show-labels 
修改pod标签(label)
kubectl label pod replicasetdemo-652lc app=lagou --overwrite=True
再次显示pod的标签：发现多了一个pod，原来的rs中又重新拉起一个pod，说明rs是通过label去管理pod
kubectl get pod --show-labels 

删除rs
kubectl delete rs replicasetdemo
```



## 5.4 总结

kubectl命令行工具适用于RC的绝大部分命令同样适用于ReplicaSet，此外，我们当前很多好单独使用 ReplicaSet，它主要被Deployment这个更高层的资源对象所使用，从而形成一整套 Pod 创建，删除，更新的编排机制，我们在使用 Deployment 时无需

# 6 Deployment

# 7 DaemonSet

# 8 Job

# 9 StatefulSet