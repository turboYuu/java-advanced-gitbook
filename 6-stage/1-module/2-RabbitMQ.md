第二部分 RabbitMQ

# 1 RabbitMQ架构与实战

## 1.1 RabbitMQ介绍、概念、基本架构

## 1.2 安装和配置RabbitMQ

安装环境：

1.虚拟主机软件 ： VirtualBox  5.2.34 r133893 (Qt5.6.2)

2.操作系统：CentOS Linux release 7.7.1908

3.Erlang：erlang-23.0.2-1.el7.x86_64

4.RabbitMQ：rabbitmq-server-3.8.5-1.el7.noarch



RabbitMQ的安装需要首先安装Erlang，因为它是基于Erlang的VM运行的。

RabbitMQ需要的依赖：socat和logrotate，logrotate操作系统中已经存在了，只需要安装socat就可以。

RabbitMQ与Erlang的兼容关系详见：https://www.rabbitmq.com/which-erlang.html

![image-20210831005111069](assest/image-20210831005111069.png)

1.安装依赖

```
yum install socat -y
```

2.安装Erlang

erlang-23.0.2-1.el7.x86_64下载地址：https://github.com/rabbitmq/erlang-rpm/releases/download/v23.0.2/erlang-23.0.2-1.el7.x86_64.rpm

首先将erlang-23.0.2-1.el7.x86_64.rpm上传至服务器，然后执行下述命令：

```
rpm -ivh erlang-23.0.2-1.el7.x86_64.rpm
```

3.安装RabbitMQ

rabbitmq-server-3.8.5-1.el7.noarch.rpm下载地址：http://github.com/rabbitmq/rabbitmq-server/releases/download/v3.8.5/rabbitmq-server-3.8.5-1.el7.noarch.rpm

将rabbitmq-server-3.8.5-1.el7.noarch.rpm上转至服务器，然后执行：

```
rpm -ivh rabbitmq-server-3.8.5-1.el7.noarch.rpm
```

查看rabbitmq服务

```shell
[root@localhost ~]# systemctl list-unit-files | grep rabbitmq-server
rabbitmq-server.service                       disabled

```

```shell
#rabbitmq安装路径
[root@localhost rabbitmq]# pwd
/usr/lib/rabbitmq
```

4.启用RabbitMQ的管理插件

```
rabbitmq-plugins enable rabbitmq_management
```



![image-20210831002608641](assest/image-20210831002608641.png)

![image-20210831002639888](assest/image-20210831002639888.png)

5.开启RabbitMQ

```shell
#rabbitmq 后台启动
systemctl start rabbitmq-server
#或 前端启动
rabbitmq-server
```

后台启动

```shell
rabbitmq-server -detached
```

6.添加用户

```
[root@localhost ~]# rabbitmqctl add_user root 123456
Adding user "root" ...
[root@localhost ~]# rabbitmqctl list_users
Listing users ...
user	tags
guest	[administrator]
root	[]
rabbitmqctl add_user root 123456


```

7.给用户添加标签

```shell
[root@localhost ~]# rabbitmqctl set_user_tags root administrator
Setting tags for user "root" to [administrator] ...
[root@localhost ~]# rabbitmqctl list_users
Listing users ...
user	tags
guest	[administrator]
root	[administrator]

```

8.给用户添加权限

```shell
[root@localhost ~]# rabbitmqctl set_permissions --vhost / root ".*" ".*" ".*"
Setting permissions for user "root" in vhost "/" ...

```

用户的标签和权限：

| Tag           | Capabilities                                                 |
| ------------- | ------------------------------------------------------------ |
| None          | 没有访问management插件的权限                                 |
| management    | 可以使用消息协议做任何操作的权限，加上：<br/>1. 可以使用AMQP协议登录的虚拟主机的权限<br/>2. 查看它们能登录的所有虚拟主机中所有队列、交换器和绑定的权限<br/>3. 查看和关闭它们自己的通道和连接的权限<br/>4. 查看它们能访问的虚拟主机中的全局统计信息，包括其他用户的活动 |
| policymaker   | 所有management标签可以做的，加上：<br/>1. 在它们能通过AMQP协议登录的虚拟主机上，查看、创建和删除策略以及虚 拟主机参数的权限 |
| monitoring    | 所有management能做的，加上：<br/>1. 列出所有的虚拟主机，包括列出不能使用消息协议访问的虚拟主机的权限<br/>2. 查看其他用户连接和通道的权限<br/>3. 查看节点级别的数据如内存使用和集群的权限<br/>4. 查看真正的全局所有虚拟主机统计数据的权限 |
| administrator | 所有policymaker和monitoring能做的，加上：<br/>1. 创建删除虚拟主机的权限<br/>2. 查看、创建和删除用户的权限<br/>3. 查看、创建和删除权限的权限<br/>4. 关闭其他用户连接的权限 |

9.访问 http://ip:15672

![image-20210831011234965](assest/image-20210831011234965.png)

![image-20210831011318262](assest/image-20210831011318262.png)

## 1.3 RabbitMQ常用操作命令

## 1.4 RabbitMQ工作流程详解

## 1.5 RabbitMQ工作模式详解

## 1.6 Spring整合RabbitMQ

## 1.7 SpringBoot整合RabbitMQ

# 2 RabbitMQ高级特性解析

# 3 RabbitMQ集群与运维

# 4 RabbitMQ源码剖析