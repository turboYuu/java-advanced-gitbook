第十部分 K8s运维篇-常用软件安装

# 1 dashboard

在kubeadm方式安装的k8s集群中安装 Dashboard。

## 1.1 Dashboard简介

K8s的可视化工具

## 1.2 官网地址

```html
https://github.com/kubernetes/dashboard

下载配置文件
https://github.com/kubernetes/dashboard/blob/v2.0.3/aio/deploy/recommended.yaml
```

## 1.3 安装镜像

```
kubernetsui 下边的镜像不需要科学上网

所有节点
docker pull kubernetesui/dashboard:v2.0.3
docker pull kubernetesui/metrics-scraper:v1.0.4
```

## 1.4 修改配置文件

### 1.4.1 控制器部分

```bash
179行左右 修改下载策略
     containers:
       - name: kubernetes-dashboard
         image: kubernetesui/dashboard:v2.0.3          
         imagePullPolicy: IfNotPresent
         
262行左右。新增下载策略           
     containers:
       - name: dashboard-metrics-scraper
         image: kubernetesui/metrics-scraper:v1.0.4          
         imagePullPolicy: IfNotPresent
```



### 1.4.2 service部分

默认Dashboard只能集群内访问，修改Service 为 NodePort 类型，暴露到外部访问。找到Service配置。在配置文件上边。增加 type:NodePort 和 nodePort:30100 端口

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30100
  type: NodePort
  selector:
    k8s-app: kubernetes-dashboard
```



# 2 管理 Service Accounts

# 3 RBAC

使用RBAC鉴权。基于角色（Role）的访问控制

# 4 Dashboard新增用户

## 4.1 使用资源文件方式新增用户

在配置文件下边增加用户及给用户授予集群管理员角色

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: dashboard-admin
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin-cluster-role
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: dashboard-admin
    namespace: kubernetes-dashboard
```



## 4.2 使用命令行方式新增用户

## 4.3 部署dashboard

部署完dashboard服务，可以选在使用token认证方式登录或者kubeconfig认证方式登录dashboard。

```bash
kubectl apply -f .

kubectl get pods -n kubernetes-dashboard -o wide

kubectl get svc -n kubernetes-dashboard

kubectl delete -f .
```





![image-20220218173554069](assest/image-20220218173554069.png)

## 4.4 token认证方式

### 4.4.1 分步查看 token 信息

```bash
1.根据命名空间找到我们创建的用户
kubectl get sa -n kubernetes-dashboard

2.查看我们创建用户的详细信息。找到token属性对应的secret值
kubectl describe sa dashboard-admin -n kubernetes-dashboard
kubectl describe secrets dashboard-admin-token-7rwn4 -n kubernetes-dashboard

3.或者是根据命名空间查找secrets。获得dashboard-admin用户的secret。 
kubectl get secrets -n kubernetes-dashboard
kubectl describe secrets dashboard-admin-token-7rwn4 -n kubernetes-dashboard
```



### 4.4.2 快速查看 token 信息

```bash
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep dashboard-admin | awk '{print $1}')
```

## 4.5 浏览器访问

```html
注意：是https方式访问
https://192.168.31.61:30100/
```

![image-20220218175206861](assest/image-20220218175206861.png)

## 4.6 kubeConfig认证方式

了解即可

```bash
以下命令可以一起执行。可以更改dashboard-admin.conf的生成目录。关键点还是要首先或者 dashboard-admin用户的secret值。

DASH_TOCKEN=$(kubectl get secret -n kubernetes-dashboard dashboard-admin-token-7rwn4 -o jsonpath={.data.token}|base64 -d)

kubectl config set-cluster kubernetes --server=192.168.31.61:6443 --kubeconfig=/root/dashboard-admin.conf

kubectl config set-credentials dashboard-admin --token=$DASH_TOCKEN --kubeconfig=/root/dashboard-admin.conf

kubectl config set-context dashboard-admin@kubernetes --cluster=kubernetes --user=dashboard-admin --kubeconfig=/root/dashboard-admin.conf

kubectl config use-context dashboard-admin@kubernetes --kubeconfig=/root/dashboard-admin.conf

将生成的dashboard-admin.conf上传到windows系统中。浏览器选择dashboard-admin.conf文件即可用于登录dashboard
```



# 5 使用statefulSet创建Zookeeper集群

# 6 statefulSet

# 7 动态PV

# 8 动态PV案例一

# 9 动态PV案例二