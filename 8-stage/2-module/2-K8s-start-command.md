第二部分 K8s快速入门之命令行

# 1 NameSpace介绍

可以认为namespace是你kubernetes集群中的虚拟化集群。在一个Kubernetes集群中可以拥有多个命名空间，它们在逻辑上彼此隔离。可以为你提供组织，安全甚至性能上的帮助。

NameSpace是对一组资源和对象的抽象集合，比如可以用来将系统内部的对象划分为不同的项目组或用户组。常见的pods，services，replication 和 deployments 等都是属于某一个 namespace 的（默认是default），而 node，persistentVolumes等则不属于任何namespace。

大多数的Kubernetes中的集群默认会有一个叫default的namespace。实际上，应该是4个：

- default：你的资源默认被创建于default命名空间。
- kube-system：Kubernetes系统组件使用。
- kebu-node-lease：Kubernetes集群节点租约状态，v1.13加入。
- kube-public：公共资源使用。但实际上现在并不常用。

这个默认（default）的namespace并没有什么特别，但你不能删除它。这很适合刚刚开始使用kubernetes 和一些小的产品系统。单不建议应用于大型生产系统。因为，这种复杂系统中，团队会非常容易意外地或无意识地重写或者中断其他服务service。相反，请创建多个命名空间来把你的service（服务）分割成更容易管理的块。

作用：多租户情况下，实现资源隔离<br>属于逻辑隔离<br>属于管理边界<br>不属于网络边界<br>可以针对每个namespace做资源配额

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

```
default 用户创建的pod默认在此命名空间
kube-public 所有用户均可以访问，包括未认证用户
kube-node-lease kubernetes集群节点租约状态,v1.13加入
kube-system kubernetes集群在使用
```



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

Pod是Kubernetes集群能够调度的最小单元。Pod是容器的封装。

在Kubernetes集群中，Pod是所有业务类型的基础，也是K8S管理的最小单位级，它是一个或多个容器的组合。这些容器共享存储、网络 和 命名空间，以及如何运行的规范。在Pod中，所有容器被统一安排和调度，并运行在共享的上下文中。对具体应用而言，Pod是它们的逻辑主机，Pod包含业务相关的多个应用容器。

### 2.1.1 Pod有两个必须知道的特点

**网络**：每一个Pod都会被指派一个唯一的ip地址，在Pod中的每一个容器共享网络命名空间，包括ip地址和网络端口。在同一个Pod中的容器可以和 localhost进行相互通信。当Pod中的容器需要与 Pod 外的实体进行通信时，则需要通过端口等共享网络资源。

**存储**：Pod能够被指定共享存储卷的集合，在Pod中所有的容器能够访问共享存储卷，允许这些容器共享数据。存储卷也允许在一个 Pod 持久化数据，以防止其中的容器被重启。

### 2.1.2 Pod的工作方式

k8s一般不直接创建Pod，而是通过控制器和摹本配置来管理和调度

- Pod模板

- Pod重启

  在Pod中的容器可能会由于异常等原因导致其终止退出，Kubernetes提供了重启策略以重启容器。重启策略对同一个 Pod 的所有容器起作用，容器的重启由 Node上的 kubelet 执行。Pod 支持三种重启策略，在配置文件中通过 *restartPolicy* 字段设置重启策略：

  1. Always：只要退出就会重启
  2. OnFailure：只有在失败退出（exit code不等于0）时，才会重启
  3. Never：只要退出，就不再重启

  注意，这里的重启是指在Pod的宿主Node上进行本地重启，而不是调度到其他Node上。

- 资源限制

  Kubernetes通过cgroup限制容器的CPU和内存等计算资源，包括 request（请求，调度器保证调度到资源充足的Node上）和 limits（上限）等。

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

[kubernetes文档-参考-kubectl](https://kubernetes.io/zh/docs/reference/kubectl/)

### 2.6.1 语法规则

```
kubectl [command] [TYPE] [NAME] [flags]
```

其中`command`、`TYPE`、`NAME` 和 `flags`分别是：

- `command`：指定要对一个或多个资源执行的操作，例如`create`、`get`、`describe`、`delete`。

- `TYPE`：指定[资源类型](https://kubernetes.io/zh/docs/reference/kubectl/overview/#%E8%B5%84%E6%BA%90%E7%B1%BB%E5%9E%8B)。资源类型不区分大小写，可以指定单数、复数或缩写形式。例如，以下命令输出相同的结果：

  ```
  kubectl get pod pod1
  kubectl get pods pod1
  kubectl get po pod1
  ```

- `NAME`：指定资源的名称。名称区分大小写。如果省略名称，则显示所有资源的详细信息 `get pods`。

  在对多个资源进行操作时，你可以按类型和名称指定每个资源，或指定一个或多个文件：

  - 按类型和名称指定资源：

    - 要对所有类型相同的资源进行分组，请执行以下操作：`TYPE1 name1 name2 name<#>`

      例子：`kubectl get pod example-pod1 example-pod2`

    - 分别指定多个资源类型：`TYPE1/name1 TYPE2/name2 TYPE3/name3 TYPE<#>/name<#>`。

      例子：`kubectl get pod/example-pod1 replicationcontroller/example-rc-1`

  - 用一个或多个文件指定资源：`-f file -f file2 -f file<#>`

    - [使用 YAML 而不是 JSON ](https://kubernetes.io/zh/docs/concepts/configuration/overview/#general-configuration-tips)因为YAML更容易使用，特别是用于配置文件时。

      例子：`kubectl get -f ./pod.yaml`

- `flags`：指定可选参数。例如，可以使用 -s 或 -server 参数指定 Kubernetes API 服务器的地址和端口。

> **注意**：
>
> 从命令行指定的参数会覆盖默认值和任何相应的环境变量。

如果需要帮助，从终端窗口运行 `kubectl help`。

### 2.6.2 get命令

`kubectl get` 列出一个或多个资源。

```bash
# 查看集群状态信息
kubectl cluster-info            

# 查看集群状态
kubectl get cs 

# 查看集群节点信息
kubectl get nodes 

# 查看集群命名空间
kubectl get ns                          

# 查看指定命名空间的服务
kubectl get svc -n kube-system          

# 以纯文本输出格式列出所有pod。
kubectl get pods

# 以纯文本输出格式列出所有pod，并包含附加信息(如节点名)。
kubectl get pods -o wide

# 以纯文本输出格式列出具有指定名称的副本控制器。提示：您可以使用别名'rc' 缩短和替换'replicationcontroller'资源类型。
kubectl get replicationcontroller <rc-name> 

# 以纯文本输出格式列出所有副本控制器和服务。
kubectl get rc,services

# 以纯文本输出格式列出所有守护程序集，包括未初始化的守护程序集。 
kubectl get ds --include-uninitialized

# 列出在节点server01上运行的所有pod
kubectl get pods --field-selector=spec.nodeName=server01
```



### 2.6.3 describe命令

`kubectl describe` 显示一个或多个资源的详细状态，默认情况下包括未初始化的资源。

```bash
# 显示名称为<node-name>的节点的详细信息。 
kubectl describe nodes <node-name>

# 显示名为<pod-name>的pod的详细信息。 
kubectl describe pods/<pod-name>

# 显示由名为<rc-name>的副本控制器管理的所有pod 的详细信息。 
# 记住：副本控制器创建的任何pod 都以复制控制器的名称为前缀。 
kubectl describe pods <rc-name>

# 描述所有的pod，不包括未初始化的pod
kubectl describe pods --include-uninitialized=false
```

> **说明**：<br>`kubectl get`  命令通常用于检索同一资源类型的一个或多个资源，它具有丰富的参数，允许你使用 `-o` 或 `--output` 参数自定义输出格式。你可以指定 `-w` 或 `--watch` 参数以开始观察特定对象的更新。`kubectl describe` 命令更侧重于描述指定资源的许多相关方面。它可以调用对 `API 服务器` 的多个API调用来为用户构建视图。例如，该`kubectl describe node`命令不仅检索有关节点的信息，还检索在其上运行的pod的摘要，为节点生成的事件等。

### 2.6.4 delete命令

`kubectl delete` 从文件、stdin 或指定标签选择器、名称、资源选择器或资源中删除资源。

```bash
# 使用pod.yaml 文件中指定的类型和名称删除pod。 
kubectl delete -f pod.yaml

# 删除标签名= <label-name> 的所有pod 和服务。 
kubectl delete pods,services -l name=<label-name>

# 删除所有具有标签名称= <label-name> 的pod 和服务，包括未初始化的那些。
kubectl delete pods,services -l name=<label-name> --include-uninitialized 

# 删除所有pod，包括未初始化的pod。
kubectl delete pods --all
```



### 2.6.5 进入容器命令

`kubectl exec`  对 pod 中的容器执行命令。与 docker 的exec命令非常类似

```bash
# 从pod <pod-name>中获取运行'date'的输出。默认情况下，输出来自第一个容器。
kubectl exec <pod-name> date

# 运行输出'date' 获取在容器的<container-name>中pod <pod-name> 的输出。
kubectl exec <pod-name> -c <container-name> date

# 获取一个交互TTY并运行 /bin/bash <pod-name >。默认情况下，输出来自第一个容器。
kubectl exec -ti <pod-name> /bin/bash
```



### 2.6.6 logs命令

`kubectl logs` 打印 Pod中容器的日志

```bash
# 从 pod 返回日志快照。
kubectl logs <pod-name>

# 从 pod <pod-name> 开始流式传输日志。这类似于 'tail -f' Linux 命令。
kubectl logs -f <pod-name>
```



### 2.6.7 格式化输出

[格式化输出](https://kubernetes.io/zh/docs/reference/kubectl/overview/#%E6%A0%BC%E5%BC%8F%E5%8C%96%E8%BE%93%E5%87%BA)

**语法**

```bash
kubectl [command] [TYPE] [NAME] -o=<output_format>
```

**例子**

```bash
将pod信息格式化输出到一个yaml文件
kubectl get pod web-pod-13je7 -o yaml
```



### 2.6.8 强制删除

```bash
强制删除一个pod
--force --grace-period=0
```



## 2.7 资源缩写

### 2.7.1 缩写汇总

[资源类型](https://kubernetes.io/zh/docs/reference/kubectl/overview/#%E8%B5%84%E6%BA%90%E7%B1%BB%E5%9E%8B)

(以下输出可以通过 `kubectl api-resources` 获取，内容以 Kubernetes 1.19.1 版本为准。)

| 资源名                            | 缩写名     | API 分组                     | 按命名空间 | 资源类型                       |
| --------------------------------- | ---------- | ---------------------------- | ---------- | ------------------------------ |
| `bindings`                        |            |                              | true       | Binding                        |
| `componentstatuses`               | `cs`       |                              | false      | ComponentStatus                |
| `configmaps`                      | `cm`       |                              | true       | ConfigMap                      |
| `endpoints`                       | `ep`       |                              | true       | Endpoints                      |
| `events`                          | `ev`       |                              | true       | Event                          |
| `limitranges`                     | `limits`   |                              | true       | LimitRange                     |
| `namespaces`                      | `ns`       |                              | false      | Namespace                      |
| `nodes`                           | `no`       |                              | false      | Node                           |
| `persistentvolumeclaims`          | `pvc`      |                              | true       | PersistentVolumeClaim          |
| `persistentvolumes`               | `pv`       |                              | false      | PersistentVolume               |
| `pods`                            | `po`       |                              | true       | Pod                            |
| `podtemplates`                    |            |                              | true       | PodTemplate                    |
| `replicationcontrollers`          | `rc`       |                              | true       | ReplicationController          |
| `resourcequotas`                  | `quota`    |                              | true       | ResourceQuota                  |
| `secrets`                         |            |                              | true       | Secret                         |
| `serviceaccounts`                 | `sa`       |                              | true       | ServiceAccount                 |
| `services`                        | `svc`      |                              | true       | Service                        |
| `mutatingwebhookconfigurations`   |            | admissionregistration.k8s.io | false      | MutatingWebhookConfiguration   |
| `validatingwebhookconfigurations` |            | admissionregistration.k8s.io | false      | ValidatingWebhookConfiguration |
| `customresourcedefinitions`       | `crd,crds` | apiextensions.k8s.io         | false      | CustomResourceDefinition       |
| `apiservices`                     |            | apiregistration.k8s.io       | false      | APIService                     |
| `controllerrevisions`             |            | apps                         | true       | ControllerRevision             |
| `daemonsets`                      | `ds`       | apps                         | true       | DaemonSet                      |
| `deployments`                     | `deploy`   | apps                         | true       | Deployment                     |
| `replicasets`                     | `rs`       | apps                         | true       | ReplicaSet                     |
| `statefulsets`                    | `sts`      | apps                         | true       | StatefulSet                    |
| `tokenreviews`                    |            | authentication.k8s.io        | false      | TokenReview                    |
| `localsubjectaccessreviews`       |            | authorization.k8s.io         | true       | LocalSubjectAccessReview       |
| `selfsubjectaccessreviews`        |            | authorization.k8s.io         | false      | SelfSubjectAccessReview        |
| `selfsubjectrulesreviews`         |            | authorization.k8s.io         | false      | SelfSubjectRulesReview         |
| `subjectaccessreviews`            |            | authorization.k8s.io         | false      | SubjectAccessReview            |
| `horizontalpodautoscalers`        | `hpa`      | autoscaling                  | true       | HorizontalPodAutoscaler        |
| `cronjobs`                        | `cj`       | batch                        | true       | CronJob                        |
| `jobs`                            |            | batch                        | true       | Job                            |
| `certificatesigningrequests`      | `csr`      | certificates.k8s.io          | false      | CertificateSigningRequest      |
| `leases`                          |            | coordination.k8s.io          | true       | Lease                          |
| `endpointslices`                  |            | discovery.k8s.io             | true       | EndpointSlice                  |
| `events`                          | `ev`       | events.k8s.io                | true       | Event                          |
| `ingresses`                       | `ing`      | extensions                   | true       | Ingress                        |
| `flowschemas`                     |            | flowcontrol.apiserver.k8s.io | false      | FlowSchema                     |
| `prioritylevelconfigurations`     |            | flowcontrol.apiserver.k8s.io | false      | PriorityLevelConfiguration     |
| `ingressclasses`                  |            | networking.k8s.io            | false      | IngressClass                   |
| `ingresses`                       | `ing`      | networking.k8s.io            | true       | Ingress                        |
| `networkpolicies`                 | `netpol`   | networking.k8s.io            | true       | NetworkPolicy                  |
| `runtimeclasses`                  |            | node.k8s.io                  | false      | RuntimeClass                   |
| `poddisruptionbudgets`            | `pdb`      | policy                       | true       | PodDisruptionBudget            |
| `podsecuritypolicies`             | `psp`      | policy                       | false      | PodSecurityPolicy              |
| `clusterrolebindings`             |            | rbac.authorization.k8s.io    | false      | ClusterRoleBinding             |
| `clusterroles`                    |            | rbac.authorization.k8s.io    | false      | ClusterRole                    |
| `rolebindings`                    |            | rbac.authorization.k8s.io    | true       | RoleBinding                    |
| `roles`                           |            | rbac.authorization.k8s.io    | true       | Role                           |
| `priorityclasses`                 | `pc`       | scheduling.k8s.io            | false      | PriorityClass                  |
| `csidrivers`                      |            | storage.k8s.io               | false      | CSIDriver                      |
| `csinodes`                        |            | storage.k8s.io               | false      | CSINode                        |
| `storageclasses`                  | `sc`       | storage.k8s.io               | false      | StorageClass                   |
| `volumeattachments`               |            | storage.k8s.io               | false      | VolumeAttachment               |

