第六部分 K8s高级篇-service

[Service-K8s 官网参考](https://kubernetes.io/zh/docs/concepts/services-networking/service/)

# 1 Service

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

  在大规模集群（例如10,000个服务）中，iptables 操作会显著降低速度。IPVS 专为负载平衡而设计，并基于内核内哈希表。因此，你可以通过 IPVS 的 kube-proxy 在大量服务中实现性能一致性。同时，基于 IPVS 的 kube-proxy 具有更复杂的负载平衡算法（最小连接、局部性）

## 2.2 ClusterIP

### 2.2.1 使用镜像

### 2.2.2 部署service

### 2.2.3 运行service

## 2.3 NodePort

### 2.3.1 使用镜像

### 2.3.2 部署service

### 2.3.3 运行service

## 2.4 LoadBalancer

## 2.5 ExternalName

# 3 ingress网络

# 4 ingress网络实验一

# 5 ingress网络实验二