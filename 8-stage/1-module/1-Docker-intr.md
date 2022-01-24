第一部分 Docker快速入门

# 1 初识 Docker

## 1.1 docker 官网

https://www.docker.com/

docker 官方文档地址：https://docs.docker.com/

## 1.2 github 地址

https://github.com/docker/docker-ce

## 1.3 Docker的历史

2010年，几个年轻人，就在美国的旧金山成立的一家公司 dotcloud。做一些Paas平台的创业公司！从事LXC（Linux Container 容器）有关的容器技术！Linux Container 容器是一种内核虚拟化技术，可以提供轻量级的虚拟化，以便隔离进程和资源。它们将自己的技术（容器化技术）命名为docker。docker刚刚诞生的时候，没有引起行业的注意，虽然获得来创业孵化器（Y Combinator）的支持、也获得过一些融资，但随着IT巨头们 也进入 Paas，dotCloud举步维艰。

2013年，dotCloud的创始人，28岁的Solo Hykes做了一个艰难的决定，将dotCloud的核心引擎开源，这项核心引擎技术能够将Linux容器中的应用程序、代码打包、轻松的在服务器之间进行迁移。这个基于LXC技术的核心管理引擎开源后，让全世界的技术人员感到惊艳。感叹这一切太方便了！越来越多的人发现 docker 的优点。之后，Docker每个月都会更新一个版本！2014年6月9日，Docker 1.0 发布！1.0 版本的发布，标志着docker频台已经足够成熟稳定，并可以被应用到生产环境。

docker为什么火。因为docker十分轻巧，在容器技术出来之前，我们都是使用虚拟机技术！比较笨重，Docker容器技术，也是一种虚拟化技术！

## 1.4 docker 版本

docker 从 17.03 版本之后分为 CE（Community Edition：社区版）和 EE（Enterprise Edition：企业版），我们使用社区版（CE）

## 1.5 DevOps（开发、运维）

一款产品：开发-上线 两套环境！应用环境，应用配置！开发-运维。

- 开发环境Windows，最后发布到 Linux！
- 应用更快速的交付和部署
- 更便捷的升级和扩缩容
- 更简单的系统运维
- 更高效的计算资源利用

## 1.6 什么是docker 

当人们说“Docker”时，他们通常是指 Docker Engine，它是一个客户端-服务器应用程序，由 Docker 守护进程，一个 REST API 指定与守护进程交互的接口，和一个命令行接口（CLI）与守护进程通信（通过封装 REST API）。Docker Engine 从 CLI 中接受 docker 命令，例如 docker run、docker ps 来列出正在运行的容器、docker images 列出镜像，等等。

- docker 是一个软件，可以运行在window、linux、mac等各种操作系统上
- docker是一个开源的应用容器引擎，基于Go语言开发并遵从 Apache 2.0 协议开源，项目代码托管在 github 上进行维护
- docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的Linux 机器上。
- 容器是完全使用沙箱机制，相互之间不会有任何接口，更重要的是容器性能开销极低。

## 1.7 docker 基本组成

- docker主机（Host）：安装了Docker程序的机器（Docker直接安装在操作系统之上）；
- docker仓库（Registry）：用来保存各种打包好的软件镜像；仓库分为公有仓库和私有仓库。（很类似 maven）
- docker镜像（Images）：软件打包好的镜像；放在docker仓库中；
- docker容器（Container）：镜像启动后的示例称为一个容器；容器是独立运行的一个或一组应用

## 1.8 docker 与 操作系统比较

docker是一种轻量级的虚拟化技术，与传统操作系统技术的特性比较如下表：

| 特性     | 容器               | 虚拟机     |
| -------- | ------------------ | ---------- |
| 启动速度 | 秒级               | 分钟级     |
| 性能     | 接近原生           | 较弱       |
| 内存代价 | 很小               | 较多       |
| 硬盘使用 | 一般为MB           | 一般为GB   |
| 运行密度 | 单机支持上千个容器 | 一般几十个 |
| 隔离性   | 安全隔离           | 完全隔离   |
| 迁移性   | 优秀               | 一般       |

传统的虚拟机方式提供的是相对封闭的隔离。Docker利用Linux系统上的多种防护实现了严格的隔离可靠性，并且可以整合众多安全工具。从 1.3.0 版本开始，docker重点改善了容器的安全控制和镜像的安全机制，极大提高了使用docker的安全性。



# 2 Docker 安装

[Install Docker Engine on CentOS](https://docs.docker.com/engine/install/centos/)

## 2.1 安装 docker 前置条件

当我们安装 Docker的时候，会涉及两个主要组件：

- Docker CLI：客户端
- Docker daemon：有时也称为 “服务端” 或者 “引擎”

### 2.1.1 硬件安装要求

| 序号 |    硬件    |                    要求                    |
| :--: | :--------: | :----------------------------------------: |
|  1   |    CPU     |                 推荐 2 核                  |
|  2   |    内存    |                  至少 2G                   |
|  3   |    硬盘    |                  至少 50G                  |
|  4   | centos 7.7 | docker及K8S集群推荐<br>使用centos 7.8 版本 |



### 2.1.2 节点信息

服务器用户名：root，服务器密码：123456。及时做好系统快照。

| 主机名     | IP地址        | 说明       |
| ---------- | ------------- | ---------- |
| docker-100 | 192.168.31.81 | docker主机 |



### 2.1.3 centos下载

推荐使用centos 7.8 版本，版本号：CentOS-7-x86_64-Minimal-2003.iso

编写文档时，7.7 和 7.8 版本已经停用，下载链接为 7.9 版本，本版号：[CentOS-7-x86_64-Minimal-2009.iso](http://mirrors.aliyun.com/centos/7.9.2009/isos/x86_64/CentOS-7-x86_64-Minimal-2009.iso)

http://mirrors.aliyun.com/centos/7.9.2009/isos/x86_64/

https://mirrors.aliyun.com/centos-vault/7.8.2003/isos/x86_64/

实际使用的 7.7版本，版本号：CentOS-7-x86_64-Minimal-1908.iso

### 2.1.4 centos 配置

#### 2.1.4.1 查看centos系统版本命令

```
cat /etc/centos-release
```

#### 2.1.4.2 配置阿里云 yum 源

```shell
1. 下载安装wget
yum install -y wget

2. 备份默认的yum
mv /etc/yum.repos.d /etc/yum.repos.d.backup

3. 设置新的yum目录
mkdir -p /etc/yum.repos.d

4. 下载阿里yum配置到该目录中，选择对应版本
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

5. 更新epel源为 阿里云epel源
mv /etc/yum.repos.d/epel.repo /etc/yum.repos.d/epel.repo.backup
mv /etc/yum.repos.d/epel-testing.repo /etc/yum.repos.d/epel-testing.repo.backup
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo

6. 重建缓存
yum clean all
yum makecache

7. 看一下 yum 仓库有多少包
yum repolist
```



#### 2.1.4.3 升级系统内核

```shell
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
yum --enablerepo=elrepo-kernel install -y kernel-lt
grep initrd16 /boot/grub2/grub.cfg
grub2-set-default 0

reboot
```



#### 2.1.4.4 查看 centos系统内核命令

```shell
uname -r 
uname -a
```

#### 2.1.4.5 查看CPU命令

```shell
lscpu
```

#### 2.1.4.6 查看内存命令

```shell
free
free -h
```



#### 2.1.4.7 查看硬盘信息

```shell
fdisk -l
```



#### 2.1.4.8 关闭防火墙

```shell
systemctl status firewalld
systemctl stop firewalld 
systemctl disable firewalld
```



#### 2.1.4.9 关闭 selinux

#### 2.1.4.10  网桥过滤

#### 2.1.4.11 命令补全

#### 2.1.4.12 上传文件

## 2.2 安装 docker



# 3 Docker 的使用

## 3.1 docker命令分类

## 3.2 docker镜像（image）

## 3.3 docker镜像常用命令

## 3.4 docker容器（container）

## 3.5 docker容器常用命令

## 3.5 docker常用命令汇总

## 3.6 安装 nginx

## 3.7 安装 mysql

## 3.8 安装zookeeper

## 3.9 安装activeMQ

