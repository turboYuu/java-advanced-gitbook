第六部分 K8s高级篇-service

# 1 Service

[Service-K8s 官网参考](https://kubernetes.io/zh/docs/concepts/services-networking/service/)

## 1.1 简介

通过以前的学习，已经能够通过Deployment来创建一组 Pod 来提供具有高可用性的服务。虽然每个 Pod 都会分配一个单独的 Pod IP，然而却存在如下两个问题：

- Pod IP 仅仅是集群内可见的虚拟 IP，外部无法访问
- Pod IP 会随着 Pod 的销毁而消失，当Deployment 对 Pod 进行动态伸缩时，Pod IP 可能随时随地都会变化，这样对于我们访问这个服务带来了难度

Service 能提供负载均衡的能力，但是在使用上有以下限制：只提供4 层负载均衡能力，而没有 7 层功能，但有时我们可能需要很多的匹配规则来转发请求，这点上 4 层负载均衡是不支持的。

因此，Kubernetes 中的 service 对象就是解决以上问题的，实现服务发现核心关键。

## 1.2 Service 在 K8s 中有以下四种类型

[服务类型-K8s-官网参考](https://kubernetes.io/zh/docs/concepts/services-networking/service/#publishing-services-service-types)

- ClusterIP：默认类型，自动分配一个仅 Cluster 内部可以访问的虚拟 IP。
- NodePort：在 ClusterIP 基础上为 Service 在每台机器上绑定一个端口，这样就可以通过 `:NodePort`来访问该服务。
- LoadBalancer：在NodePort 的基础上，借助 cloud provider 创建一个外部负载均衡器，并将请求转发到 NodePort，是付费服务，而且价格不菲。
- ExternalName：把集群外部的服务引入到集群内部来，在集群内部直接使用。没有任何类型代理被创建，这只有 Kubernetes 1.7 或更高版本 kube-dns 才支持。

# 2 详解 4 种 Service 类型

## 2.1 Service 和 Pods

Kubernetes 的 Pods 是有生命周期的。它们可以被创建，而且销毁不会再启动。如果你使用 Deployment 来运行你的应用程序，则它可以动态创建和销毁 Pod。

一个Kubernetes 的 Service是一种抽象，它定义了一组 Pods 的逻辑集合和一个用于访问它们的策略 - 有时候被称之为 微服务。一个 Service 的目标 Pod 集合通常是由 Label Selector 来决定的。

举个例子，想象一个处理图片的后端运行了三个副本。这些副本都是可以替代的 - 前端不关心它们使用的是哪一个后端。尽管实际组成后端集合的Pod可能会变化，前端的客户端却不需要知道这个变化，也不需要自己有一个列表来记录这些后端服务。Service 抽象能让你达到这种解耦。

不像 Pod 的 IP 地址，它实际路由到一个固定的目的地，Service 的 IP 实际上不能通过单个主机来进行应答。相反，我们使用 iptables （Linux中的数据包处理逻辑）来定义一个虚拟 IP 地址（VIP），它可以根据需要透明地进行重定向。当客户端连接到 VIP 时，它们的流量会自动地传输到一个合适的 Endpoint。环境变量和DNS，实际上会根据 Service 的 VIP 和端口来进行填充。

kube-proxy 支持三种代理模式：用户空间，iptables 和 IPVS；它们各自的操作略有不同。

- ***Userspace代理模式***

  Client Pod 要访问 Server Pod 时，它先将请求发给本机内核空间中的 service 规则，由它再将请求转给监听在指定套接字上的 kube-proxy，kube-proxy处理完请求，并分发请求到指定 Server Pod 后，再将请求递交给内核空间中的 service，由 service 将请求转发给指定的Server Pod。由于其需要来回在用户空间和内核空间交互通信，因此效率很差。

  当一个客户端连接到一个 VIP，iptables 规则开始起作用，它会重定向该数据包到 Service 代理的端口。Service 代理选择一个 backend，并将客户端的流量代理到 backend 上。

  这意味着 Service 的所有者能够选择任何它们想使用的端口，而不存在冲突的风险。客户端可以简单地连接到一个 IP 和 端口，而不需要直到实际访问了哪些 Pod。

- ***iptables代理模式***

  当一个客户端连接到一个 VIP ，iptables 规则开始起作用。一个 backend会被选择（或者根据会话亲和性，或者随机），数据包被重定向到这个 backend。不像 userspace 代理，数据包从来不拷贝到用户空间，kube-proxy不是必须为该 VIP 工作而运行，并且客户端 IP是不可更改的。当流量打到 Node 的端口上，或通过负载均衡器，会执行相同的基本流程，但是在那些案例中客户端 IP 是可以更改的

- ***IPVS代理模式***

  在大规模集群（例如10,000个服务）中，iptables 操作会显著降低速度。IPVS 专为负载平衡而设计，并基于内核内哈希表。因此，你可以通过 IPVS 的 kube-proxy 在大量服务中实现性能一致性。同时，基于 IPVS 的 kube-proxy 具有更复杂的负载平衡算法（最小连接、局部性，加权，持久性）。

  在Kubernetes 集群中，每个 Node 运行一个 kube-proxy 进程，kube-proxy 负责为  Service 实现了一种 VIP（虚拟IP）的形式，而不是 ExternalName 的形式。在 Kubernetes v1.0 版本，代理完全在 userspace。在 Kubernetes v1.1 版本，新增了 iptables 代理，但并不是默认的运行模式。从 Kubernetes v1.2 起，默认就是 iptables 代理。在 Kubernetes v1.8.0-beta.0 中，添加了 ipvs 代理，在Kubernetes 1.14 版本开始默认使用 ipvs 代理。在Kubernetes v1.0 版本，Service 是 “4层”（TCP/UDP over IP）概念。在 Kubernetes v1.1版本，新增了Ingress API（beta 版），用来表示 “7层”（HTTP）服务。

  这种模式，kube-proxy 会监视 Kubernetes Service 对象和 Endpoints，调用 netlink 接口以相应地创建 ipvs 规则并定期与 Kubernetes Service 对象和Endpoints 对象同步 ipvs 规则，以确保 ipvs 状态与期望一致。访问服务时，流量将被重定向到其中一个后端 Pod 与 iptables 类似。ipvs 于 netfilter 的 hook 公功能，但使用哈希表作为底层数据结构并在内核空间中工作，这意味着 ipvs 可以更快地重定向流量，并且在同步代理规则时就有更好的性能。此外，ipvs 为负载均衡算法提供了更多选项。

## 2.2 ClusterIP

类型为 ClusterIP 的 service，这个 service 有一个 Cluster-IP，其实就是一个 VIP。具体原理依靠 kube-proxy 组件，通过iptables或是ipvs实现。

ClusterIP 主要在每个 node 节点使用 iptables，将发向 clusterIP 对应端口的数据，转发到 kube-proxy中。然后 kube-proxy 自己内部实现有负载均衡的算法，并可以查询到这个 service 下对应 pod 的地址和端口，进而把数据转发给对应的 pod 的地址和端口。

***这种类型的 service 只能在集群内访问***

### 2.2.1 使用镜像

```bash
docker pull tomcat:9.0.20-jre8-alpine
```



### 2.2.2 部署service

service/cluseripdemo.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clusteripdemo
  labels:
    app: clusteripdemo
spec:
  replicas: 1
  template:
    metadata:
      name: clusteripdemo
      labels:
        app: clusteripdemo
    spec:
      containers:
        - name: clusteripdemo
          image: tomcat:9.0.20-jre8-alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
      restartPolicy: Always
  selector:
    matchLabels:
      app: clusteripdemo
---
apiVersion: v1
kind: Service
metadata:
  name: clusterip-svc
spec:
  selector:
    app: clusteripdemo
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP
```



### 2.2.3 运行service

```bash
运行服务
kubectl apply -f clusteripdemo.yml 

查看服务
kubectl get svc 

访问服务
curl 10.1.15.24:8080 

删除服务
kubectl delete -f clusteripdemo.yml
```



## 2.3 NodePort

我们的场景不全是集群内访问，也需要集群外业务访问。那么 ClusterIP 就满足不了。NodePort 当然是其中的一种实现方案。nodePort的原理在于在 node 上开了一个端口，将流向该端口的流量导入到 kube-proxy，然后由 kube-proxy 进一步给到对应的 pod。

### 2.3.1 使用镜像

```bash
docker pull tomcat:9.0.20-jre8-alpine
```



### 2.3.2 部署service

service/nodeportdemo.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodeportdemo
  labels:
    app: nodeportdemo
spec:
  replicas: 1
  template:
    metadata:
      name: nodeportdemo
      labels:
        app: nodeportdemo
    spec:
      containers:
        - name: nodeportdemo
          image: tomcat:9.0.20-jre8-alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
      restartPolicy: Always
  selector:
    matchLabels:
      app: nodeportdemo
---
apiVersion: v1
kind: Service
metadata:
  name: nodeport-svc
spec:
  selector:
    app: nodeportdemo
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30088
  type: NodePort
```



### 2.3.3 运行service

```bash
运行服务
kubectl apply -f nodeportdemo.yml 

查看服务
kubectl get svc 

访问服务
curl 10.1.61.126:8080

浏览器访问服务
http://192.168.31.61:30088 

删除服务
kubectl delete -f nodeportdemo.yml
```



## 2.4 LoadBalancer

LoadBalancer类型的 service 是可以实现集群外不访问服务的另外一种解决方案。不过并不是所有的K8s集群都会支持，大多是在公有云托管集群中会支持该类型。负载均衡器是异步创建的，关于被提供的负载均衡器的信息将会通过 Service 的 status.loadBalancer字段被发布出去。

创建LoadBalancer service 的 yaml ：

```yaml
apiVersion: v1
kind: Service 
metadata:
  name: service-lagou 
spec:
  ports:
  - port: 3000
    protocol: TCP    
    targetPort: 443    
    nodePort: 30080
  selector:
    run: pod-turbo 
  type: LoadBalancer
```



## 2.5 ExternalName

类型为 ExternalName 的service 将服务映射到 DNS 名称，而不是典型的选择器，例如my-service或者Cassandra。你可以使用spec.externalName参数指定这些服务。

这种类型的Service 通过返回 CNAME 和 它的值，可以将服务映射到 externalName 字段的内容（例如：hub.turbo.com）。ExternalName Service 是 Service 的特例，它没有 selector，也没有定义任何的端口和Endponit。相反，对于运行在集群外部的服务，它通过返回该外部服务的 cname （别名）这种方式来提供服务 。

创建 ExternalName 类型的服务的 yaml：

```yaml
kind: Service 
apiVersion: v1 
metadata:
  name: service-lagou 
spec:
  ports:
  - port: 3000
    protocol: TCP    
    targetPort: 443
  type: ExternalName
  externalName: www.turbo.com
```



# 3 ingress网络

## 3.1 什么是Ingress？

k8s 集群对外暴露服务的方式目前只有三种：LoadBalancer、NodePort、Ingress。前两种熟悉起来比较快，而且使用起来也比较方便。

ingress 由两部分组成：ingress controller 和 ingress 服务。

其中 ingress controller 目前主要有两种：基于nginx服务的ingress controller 和 基于 taefik 的 ingress controller。两者都目前支持 http 和 https 协议，由于对 nginx 比较熟悉，而且需要使用 TCP 负载，所以在此我们选择的是基于 nginx服务的 ingress controller。

在Kubernetes集群中，我们知道 service 和 pod 的 ip 仅在集群内部访问。如果外部应用要访问集群内的服务，集群外部的请求需要通过负载均衡转发到service 在Node上暴露的NodePort上，然后再由 kube-proxy组件将其转发给相关的 pod。

而 Ingress 就是为进入集群的请求提供路由规则的集合，通俗点就是提供外部访问集群的入口，将外部的HTTP或者HTTPS请求转发到集群内部 service上。

## 3.2 Ingress-nginx组成

Ingress-nginx 一般由三个组件组成：

- 反向代理负载均衡器：通常以service的port方式运行，接收并按照ingress定义的规则进行转发，常用的有 nginx，Haproxy，Traefik等，本次实验中使用的就是nginx。
- Ingress Controller：监听APIServer，根据用户编写的**ingress规则**（编写ingress的yaml文件），动态的去更改 nginx 服务的配置文件，并且 reload 重载使其生效，此过程是自动化的（通过lua脚本来实现）。
- Ingress：将nginx的配置抽象成一个Ingress对象，当用户每添加一个新的服务，只需要编写一个新的ingress的yaml文件即可。

## 3.3 Ingress-nginx的工作原理

1. ingress controller 通过和 Kubernetes api 交互，动态的去感知集群中 ingress 规则变化。然后读取它，按照自定义的规则，规则就是写明了那个域名对应哪个service，生成一段 nginx 配置。
2. 再写到 nginx-ingress-controller 的 pod 里，这个 Ingress controller 的 pod 里运行着一个 nginx 服务，控制器会把生成的 nginx 配置写入 /etc/nginx.conf 文件中。然后 reload 一下 使配置生效，以此达到分配和动态更新问题。

## 3.4 官网地址

**基于nginx服务的ingress controller 根据不同的开发公司，又分为 k8s 社区的 ingress-nginx 和 nginx 公司的 nginx-ingress**。

```html
Ingress-Nginx github 地址：
https://github.com/kubernetes/ingress-nginx 

Ingress-Nginx官方网站：
https://kubernetes.github.io/ingress-nginx/
```



## 3.5 下载资源文件

**根据github上的活跃度和关注人数，我们选择的是 k8s 社区的 ingress-nginx**。

### 3.5.1 下载 ingress-controller

```html
打开github官网：选择某一个具体版本后进入deploy/static/目录中，复制mandatory.yaml文件内容。
https://github.com/kubernetes/ingress-nginx/tree/nginx-0.30.0/deploy/static/mandatory.yaml
```



### 3.5.2 下载ingress服务

```html
https://github.com/kubernetes/ingress-nginx/blob/nginx-0.30.0/deploy/static/provider/baremetal/service-nodeport.yaml
```



## 3.6 下载镜像

需要将镜像pull到 k8s 集群各个 node 节点

```bash
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:0.30.0

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:0.30.0 quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0

docker rmi -f registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:0.30.0
```



## 3.7 ingress 与 ingress-controller



# 4 ingress网络实验一

## 4.1 使用镜像

```bash
docker pull tomcat:9.0.20-jre8-alpine
docker pull quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0
```



## 4.2 运行 ingress-controller

```bash
在mandatory.yaml文件的Deployment资源中增加属性sepc.template.sepc.hostNetWork 
hostNetwork: true
hostNetwork网络，这是一种直接定义Pod网络的方式。
如果在Pod中使用hostNetwork:true配置网络，那么Pod中运行的应用程序可以直接使用node节点的端口

运行ingress/mandatory.yaml文件
kubectl apply -f mandatory.yaml
```



## 4.3 运行ingress服务

```bash
运行ingress/service-nodeport.yaml文件
kubectl apply -f service-nodeport.yml
```



## 4.4 部署tomcat服务

ingress/tomcat-service.yml

```yaml

```



## 4.5 运行tomcat-service

```bash
kubectl apply -f tomcat-service.yml
```



## 4.6 部署ingress规则文件

ingress/ingress-tomcat.yml

```yaml

```



## 4.7 运行ingress规则

```bash
kubectl apply -f ingress-tomcat.yml 

查看ingress
kubectl get ingress

查看ingress服务:查看service的部署端口号 
kubectl get svc -n ingress-nginx

查看ingress-controller运行在那个node节点
kubectl get pod -n ingress-nginx -o wide
```

通过ingress访问tomcat

```html

```



# 5 ingress网络实验二

上边案例的部署方式只能通过 ingress-controller 部署的节点访问。集群内其他节点无法访问 ingress规则。本章节通过修改 mandatory.yaml文件的控制类类型，让集群内每一个节点都可以正常访问 ingress 规则。

## 5.1 ingress-controller

ingress/mandatory.yaml

```yaml
修改mandatory.yaml配置文件

1.将Deployment类型控制器修改为：DaemonSet
2.属性：replicas: 1  # 删除这行
```



## 5.2 service-nodeport 固定端口

ingress/service-nodeport.yml

```yaml

```



## 5.3 域名访问 ingress 规则

ingress/ingerss-tomcat.yml

```yaml

```



## 5.4 修改宿主机 hosts 文件

C:\Windows\System32\drivers\etc\hosts

```html
增加ingress-tomcat.turbo.com 域名配置： 

192.168.31.61 ingress-turbo.lagou.com
```



## 5.5 部署服务

```
kubectl apply -f .
```



## 5.6 浏览器测试

## 5.7 nginx-controller原理

```bash
查看ingress-nginx 命名空间下的pod
kubectl get pods -n ingress-nginx

进入ingress-nginx 的pod
kubectl exec -it nginx-ingress-controller-5gt4l -n ingress-nginx sh 

查看nginx反向代理域名ingress-tomcat.lagou.com
cat nginx.conf
```

