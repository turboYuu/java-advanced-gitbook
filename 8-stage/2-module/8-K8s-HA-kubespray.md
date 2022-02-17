第八部分 k8s高可用-kubespray

# 1 官网地址

## 1.1 不推荐安装使用的理由

1. 国内特殊的网络环境导致使用 kubespray 困难重重，部分镜像需要从 gcr.io 拉取，部分二进制文件需要从 github 下载。
2. 额外安装大量的软件。造成不必要的学习成本
3. 二进制安装文件和镜像体积庞大

```html
https://kubespray.io/ 

github地址
https://github.com/kubernetes-sigs/kubespray
```



# 2 centos 系统设置

## 2.1 hosts文件

可以忽略不执行，kubespray 会再次更改 hosts 文件内容。

```
192.168.31.181 k8s-master01
192.168.31.182 k8s-node01
192.168.31.183 k8s-node02
```

## 2.2 上传二进制文件

```bash
mkdir -p /tmp/releases 
cd /tmp/releases/

将下边三个文件上传到/tmp/releases/目录中
kubeadm-v1.17.6-amd64
kubectl-v1.17.6-amd64
kubelet-v1.17.6-amd64
```

## 2.3 上传镜像

```bash
cd /tmp/releases/

docker load < calico3.13.2.tar 
docker load < k8sv1.17.6.tar

rm -rf calico3.13.2.tar 
rm -rf k8sv1.17.6.tar
```

## 2.4 免密登录

```bash
需要对所有K8S节点免密登录，master节点本身也需要免密登录。在master节点执行免密命令。
ssh-keygen

ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.31.181
ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.31.182
ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.31.183
```



# 3 安装准备

## 3.1 安装ansible软件包

### 3.1.1 老版本安装方式

```bash
# 安装ansible
yum install -y ansible 
# 安装Python 3.6
yum install –y python36

pip3 install -i https://mirrors.aliyun.com/pypi/simple/ --upgrade pip Jinja2
```

### 3.1.2 新版本安装方式

```bash
yum install -y epel-release python3-pip
```

## 3.2 配置集群环境

如果是下载安装包进行安装，一下配置可以忽略不进行配置。

### 3.2.1 main 配置文件

如果有自己的harbor私服镜像地址，可以修改默认配置文件中的镜像地址。

```bash
cd /opt/kubespray/roles/download/defaults/main.yml 
修改镜像地址：
kube_image_repo: "harbor.turbo.com/k8s.gcr.io" 
docker_image_repo: "harbor.turbo.com/docker.io" 
quay_image_repo: "harbor.turbo.com/quay.io"
```



### 3.2.2 k8s-cluster 配置文件

如果有自己的harbor私服镜像地址，可以修改默认配置文件中的镜像地址。

```bash
cd /opt/kubespray/inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml

确认k8s版本
kube_version: v1.17.6 

修改k8s镜像下载地址：
kube_image_repo: "harbor.turbo.com/k8s.gcr.io"

修改k8s网络配置：默认为calico，推荐大家使用calico。也可以选择其他网络模式
kube_network_plugin: calico

kube_service_addresses: 172.25.0.0/16
kube_pods_subnet: 172.26.0.0/16
kube_network_node_prefix: 16
```



# 4 安装集群

```bash
cd /opt/kubespray

ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml
```

