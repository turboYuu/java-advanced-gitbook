​	第二部分 RabbitMQ

# 1 RabbitMQ架构与实战

## 1.1 RabbitMQ介绍、概念、基本架构

### 1.1.1 RabbitMQ介绍

RabbitMQ，是目前非常热门的一款开源消息中间件。

- 高可靠性，易扩展、高可用、功能丰富等
- 支持大多数（甚至冷门）的编程语言客户端
- RabbitMQ遵循AMQP协议，自身采用Erlang编写。
- RabbitMQ也支持MQTT等其他协议。

RabbitMQ具有强大的插件扩展能力，官方和社区提供了丰富的插件可供选择：

https://www.rabbitmq.com/community-plugins.html

### 1.1.2  RabbitMQ整体逻辑框架

![image-20210831154427834](assest/image-20210831154427834.png)

### 1.1.3 RabbitMQ Exchange类型

RabbitMQ常用的交换器类型有：`fanout`、`direct`、`topic`、`headers`四种。

**fanout**

会把所有发送到该交换器的消息路由到所有与该交换器绑定的队列中，

![image-20210831155754330](assest/image-20210831155754330.png)

**direct**

direct类型的交换器规则很简单，它会把消息路由到那些BindingKey和RouteingKey完全匹配的队列中：

![image-20210831155811570](assest/image-20210831155811570.png)

**topic**

topic类型的交换器在direct匹配规则上进行了扩展，也是讲消息路由到Bindingkey和RoutingKey相匹配的队列中，这里的匹配规则不同，约定：BindingKey和RoutingKey一样都是由“.”分隔的字符串；BindingKey中可以存在两种特殊字符"*"和"#"，用于模糊匹配，其中 ” *“用于匹配一个单词，”#“用于匹配多个单词（可以是0个）。

![image-20210831160734535](assest/image-20210831160734535.png)

**headers**

headers类型的交换器不依赖于路由键的匹配规则来路由信息，而是根据发送的消息内容中的headers属性进行匹配。在绑定队列和交换器时指定一组键值对，当发送的消息到交换器时，RabbitMQ会获取到该消息的headers，对比其中的键值对是否完全匹配队列和交换器绑定时指定的键值对，如果匹配，消息就会路由到该队列。headers类型的交换器性能很差，不实用。

### 1.1.4 RabbitMQ数据存储

#### 1.1.4.1 存储机制

RabbitMQ消息有两种类型：

- 持久化消息
- 非持久化消息

这两种消息都会被写入磁盘。

持久化消息在达到队列时写入磁盘，同时会在内存中保存一份备份，当内存吃紧时，消息从内存中清除，这会提高一定的性能。

非持久化消息一般只存在于内存中，当内存压力大时数据刷盘处理，节省内存空间。

**RabbitMQ存储层包含两个部分**，**队列索引**和**消息存储**。

![image-20210831162019149](assest/image-20210831162019149.png)

#### 1.1.4.2 队列索引 Rabbit_queue_index

索引维护队列的落盘消息的信息，如存储地点，是否已被消费者接收，是否已被消费者ack等。每个队列都有相对应的索引。

索引使用顺序的段文件来存储，后缀为.idx，文件名从0开始累加，每个段文件中包含固定的`segment_entry_count`条记录，默认值**16384**。**每个index从磁盘中读取消息的时候，至少要在内存中维护一个段文件**，所以设置`queue_index_embed_msgs_below`值时要格外谨慎，一点点增大可能会引起内存爆炸式增长。

![image-20210831163013600](assest/image-20210831163013600.png)

#### 1.1.4.3 消息存储 rabbit_msg_store

消息以键值对的形式存储到文件中，一个虚拟机上的所有队列使用同一块存储，每个节点只有一个。存储分为持久化存储（msg_store_persistent）和短暂存储（msg_store_transient）。持久化存储的内容在broker重启后不会丢失，短暂存储的内容在broker重启后丢失。

store使用文件来存储，后缀为.rdq，经过store处理的所有消息都会以追加的方式写入到该文件中，当文件的大小超过指定限制（file_size_limit）后，将会关闭该文件并创建一个新的文件供新消息写入。文件名从0开始进行累加，在进行消息存储时，RabbitMQ会在ETS（Erlang Term Storage）表中记录消息在文件中的位置映射和文件的相关信息。

消息（包括消息头、消息体、属性）可以直接存储在index中，也可以存储在store中。最佳的方式是较小的消息存储在index中，而较大的消息存储在store中。这个消息大小的界定可以通过`queue_index_embed_msgs_below`来配置，默认值为4096B，当一个消息小于设定的大小阈值时，就可以存储在index中，这样性能上可以得到优化。一个完整的消息小于这个值就放索引中，否则就放到持久化消息文件中。

读取消息时，先跟及消息ID（msg_id）找到对应存储的文件，如果文件存在并且没有锁住，则直接打开文件，从执行位置读取消息内容。如果文件不存在或者被锁住了，则发送请求由store进行处理。

删除消息时，只是从ETS表中删除指定消息的相关信息，同时更新消息对应的存储文件和相关信息。在执行消息删除操作时，并不立即对文件中的消息进行删除，也就是说信息依然在文件中，仅仅是标记为垃圾数据而已。当一个文件中都是垃圾数据时可以删除这个文件。当检测到前后两个文件中的有效数据可以合并成一个文件，并且所有的垃圾数据的大小和所有文件（至少有3个文件存在的情况下）的数据大小的比值超过设置的阈值garbage_fraction（默认值0.5）时，才会触发垃圾回收，将这两个文件合并，执行合并的两个文件一定是逻辑上相邻的两个文件，合并逻辑：

- 锁定这两个文件
- 先整理起那面文件的有效数据，再整理后面文件的有效数据
- 将后面文件的有效数据写入到前面的文件中
- 更新消息在ETS表中的记录
- 删除后面的文件

![image-20210831171213507](assest/image-20210831171213507.png)

#### 1.1.4.4 队列结构

通常队列由rabbit_amqqueue_process和backing_queue这两部分组成，rabbit_amqqueue_process负责协议相关的消息处理，即接收生产者发布的消息、向消费者较复消息、处理消息的确认（包括生产段的confirm和消费端的ack）等。backing_queue是消息存储的具体形式和引擎，并向rabbit_amqqueue_process提供相关的接口以供调用。

![image-20210831180418442](assest/image-20210831180418442.png)

如果消息投递的目的队列是空的，并且有消息消费者订阅了这个队列，那么该消息会直接发送给消费者，不会经过队列这一步。当消息无法直接投递给消费者时，需要暂时将消息存入队列，以便重新投递。

`rabbit_variable_queue.erl`源码中定义了RabbitMQ队列的**4种状态**：

- alpha：消息索引和消息内容**都存内存**，最消耗内存，很少消耗CPU
- beta：消息索引存内存，消息内容存磁盘
- gama：消息索引在内存和磁盘都有，消息内容存磁盘
- delta：消息索引和内容都存磁盘，基本不消耗内存，消耗更多CPU和IO操作

消息存入队列后，不是固定不变的，它会随着系统的负载在队列中不断流动，消息的状态会不断发生变化。

持久化的消息，索引和内容都必须先保存在磁盘上，才会处于上述状态中的一种。gama状态只有持久化消息才会有的状态。

在运行时，RabbitMQ会根据消息传递的速度定期计算一个当前内存中能够保存的最大消息数量（target_tam_count），如果alpha状态的消息数据大于此值，则会引起消息的状态转换，对于的消息可能转换到beta、gama、delta状态。区分这四种状态的主要作用是满足不同内存和CPU需求。

对于普通没有设置优先级和镜像的队列来说，backing_queue的默认实现是rabbit_variable_queue，其内部通过5个子队列Q1，Q2，delta，Q3，Q4来体现消息的各种状态。

![image-20210831183429680](assest/image-20210831183429680.png)

消费者获取消息也会引起消息的状态转换。

当消费者获取消息时

> 1.首先会从Q4中获取消息，如果获取成功则返回。
>
> 2.如果Q4为空，则尝试从Q3中获取消息，系统首先会判断Q3是否为空，如果为空则返回队列为空，即此时队列中无消息。
>
> 3.如果Q3不为空，则取出Q3中的消息；进而再判断此时Q3和Delta中的长度，如果都为空，则可以认为Q2，Delta，Q3，Q4全部为空，此时将Q1中的消息直接转移至Q4，下次直接从Q4中获取消息。
>
> 4.如果Q3为空，Delta不为空，则将Delta的消息转移至Q3中，下次可以直接从Q3中获取消息，在将消息从Delta转移到Q3的过程中，是按照索引分段读取的，首先读取某一段，然后判断读取的消息个数与Delta中的消息个数是否相等，如果相等，则可以判断此时Delta中已无消息，则直接将Q2和刚读取的消息一并放入到Q3中，如果不相等，仅将此次读取的消息转移到Q3。

> 对于持久化消息，一定会进入gama状态，在开启publisher confirm机制时，只有到了gama状态时才会确认该消息已被接收，若消息消费速度足够快，内存也充足，这些消息不会继续走到下一个状态。

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

![image-20210901143623608](assest/image-20210901143623608.png)

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

查看rabbitmq手册 `man rabbitmq-server`，如果配置了配置文件，在`rabbitmqctl status`时会看到。

![image-20210905181539755](assest/image-20210905181539755.png)

![image-20210831230535830](assest/image-20210831230535830.png)

erlang port mapper daemon 端口管理，负责通信 epmd 4369 端口号

```shell
# 前台启动Erlang VM和RabbitMQ
rabbitmq-server

# 后台启动
rabbitmq-server -detached

# 停止RabbitMQ和Erlang VM
rabbitmqctl stop

#查看所有队列
rabbitmqctl list_queues
systemctl start rabbitmq-server

# 查看所有虚拟主机
rabbitmqctl list_vhosts
rabbitmqctl list_vhosts --formatter pretty_table

# 在Erlang VM运行的情况下启动/关闭RabbitMQ应用
rabbitmqctl start_app
rabbitmqctl stop_app

#查看节点状态
rabbitmqctl status

Interface: [::], port: 25672, protocol: clustering, purpose: inter-node and CLI tool communication
Interface: [::], port: 5672, protocol: amqp, purpose: AMQP 0-9-1 and AMQP 1.0
Interface: [::], port: 15672, protocol: http, purpose: HTTP API

```

```shell
[root@localhost plugins]# pwd
/usr/lib/rabbitmq/lib/rabbitmq_server-3.8.5/plugins 	#rabbit插件地址

#查看启动插件和关联启动插件
[root@localhost plugins]# rabbitmq-plugins list --implicitly-enabled 
Listing plugins with pattern ".*" ...
 Configured: E = explicitly enabled; e = implicitly enabled
 | Status: * = running on rabbit@localhost
 |/
[E*] rabbitmq_management       3.8.5
[e*] rabbitmq_management_agent 3.8.5
[e*] rabbitmq_web_dispatch     3.8.5

# 查看插件 使用正则
[root@localhost plugins]# rabbitmq-plugins list "discovery"
Listing plugins with pattern "discovery" ...
 Configured: E = explicitly enabled; e = implicitly enabled
 | Status: * = running on rabbit@localhost
 |/
[  ] rabbitmq_peer_discovery_aws    3.8.5
[  ] rabbitmq_peer_discovery_common 3.8.5
[  ] rabbitmq_peer_discovery_consul 3.8.5
[  ] rabbitmq_peer_discovery_etcd   3.8.5
[  ] rabbitmq_peer_discovery_k8s    3.8.5
[root@localhost plugins]# rabbitmq-plugins list "examples$"
Listing plugins with pattern "examples$" ...
 Configured: E = explicitly enabled; e = implicitly enabled
 | Status: * = running on rabbit@localhost
 |/
[  ] rabbitmq_web_mqtt_examples  3.8.5
[  ] rabbitmq_web_stomp_examples 3.8.5

# 启用插件
rabbitmqctl-plugins enable <plugin-name>
# 停用插件
rabbitmqctl-plugins disable <plugin-name>
```

![image-20210901145447113](assest/image-20210901145447113.png)

```shell
# 添加用户
[root@localhost plugins]# rabbitmqctl add_user zhangsan 123456
Adding user "zhangsan" ...

# 添加用户标签
[root@localhost plugins]# rabbitmqctl set_user_tags zhangsan administrator
Setting tags for user "zhangsan" to [administrator] ...

# 设置用户权限
[root@localhost plugins]# rabbitmqctl set_permissions --vhost / zhangsan "^$" ".*" ".*"
Setting permissions for user "zhangsan" in vhost "/" ...

# 列出用户权限
[root@localhost plugins]# rabbitmqctl list_user_permissions zhangsan
Listing permissions for user "zhangsan" ...
vhost	configure	write	read
/	^$	.*	.*

#清空用户权限
[root@localhost plugins]# rabbitmqctl clear_permissions zhangsan
Clearing permissions for user "zhangsan" in vhost "/" ...

[root@localhost plugins]# rabbitmqctl list_user_permissions zhangsan
Listing permissions for user "zhangsan" ...

# 列出虚拟机上的权限
[root@localhost plugins]# rabbitmqctl list_permissions
Listing permissions for vhost "/" ...
user	configure	write	read
root	.*	.*	.*
guest	.*	.*	.*

# 清除用户标签
[root@localhost plugins]# rabbitmqctl set_user_tags zhangsan ""
Setting tags for user "zhangsan" to [] ...

# 列出用户
[root@localhost plugins]# rabbitmqctl list_users
Listing users ...
user	tags
guest	[administrator]
zhangsan	[]
root	[administrator]

# 删除用户
[root@localhost plugins]# rabbitmqctl delete_user zhangsan
Deleting user "zhangsan" ...

# 列出用户
[root@localhost plugins]# rabbitmqctl list_users
Listing users ...
user	tags
guest	[administrator]
root	[administrator]

# 修改密码
[root@localhost plugins]# rabbitmqctl change_password root 111111
Changing password for user "root" ...

```



```shell
# 创建虚拟主机
[root@localhost plugins]# rabbitmqctl add_vhost vh1
Adding vhost "vh1" ...

# 列出所有虚拟主机
[root@localhost plugins]# rabbitmqctl list_vhosts
Listing vhosts ...
name
vh1
/

# 删除虚拟主机
[root@localhost plugins]# rabbitmqctl delete_vhost vh1
Deleting vhost "vh1" ...
[root@localhost plugins]# rabbitmqctl list_vhosts
Listing vhosts ...
name
/


[root@localhost plugins]# rabbitmqctl stop_app
Stopping rabbit application on node rabbit@localhost ...
# 移除所有数据之前，要在rabbitmqctl stop_app之后
[root@localhost plugins]# rabbitmqctl reset
Resetting node rabbit@localhost ...
[root@localhost plugins]# rabbitmqctl start_app
Starting node rabbit@localhost ...
[root@localhost plugins]# rabbitmqctl list_vhosts
Listing vhosts ...
name
/
```



## 1.4 RabbitMQ工作流程详解

###  1.4.1 生产者发送消息的流程

1.生产者连接RabbitMQ，建立TCP连接（Connection），开启通道（Channel）

2.生产者声明一个Exchange（交换器），并设置相关属性，比如交换器类型，是否持久化等

3.生产者声明一个队列并设置相关属性，比如是否排他、是否持久化、是否自动删除等

4.生产者通过`bindingkey`（绑定key）将交换器和队列绑定（`binding`）起来

5.生产者发送消息至RabbitMQ Broker，其中包含`routingKey`（路由键）、交换器等信息

6.相应的交换器根据接受到的`routingkey`查找相匹配的队列。

7.如果找到，则将从生产者发送过来的信息存入相应的队列中。

8.如果没有找到，则根据生产者配置的属性选择丢弃还是回退给生产者

9.关闭信道

10.关闭连接

### 1.4.2 消费者接受消息的过程

1.消费者连接到RabbitMQ Broker，建立一个连接（Connection），开启一个信道（Channel）。

2.消费者向RabbitMQ Broker请求消费相应队列中的消息，可能会设置相应的回调函数，以及做一些准备工作。

3.等待RabbitMQ Broker回应并投递相应队列种的消息，消费者接收消息。

4.消费者确认（ack）接收到的消息。

5.RabbitMQ从队列中删除相应已经被确认的消息。

6.关闭信道。

7.关闭连接。

### 1.4.3 案例

![image-20210903111444636](assest/image-20210903111444636.png)



一对一简单模式，生产者直接发送消息给RabbitMQ，另一端消费。未定义和指定Exchange的情况下，使用的是AMQP default这个内置的Exchange。

代码地址：https://gitee.com/turboYuu/rabbit-mq-6-1/tree/master/lab/rabbit-demo/demo_02_rabbitmq

*helloProducer*

```java
package com.turbo.rabbitmq.demo;

import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class HelloProducer {

    public static void main(String[] args) throws IOException, TimeoutException {
        // 1.连接工厂
        ConnectionFactory factory = new ConnectionFactory();
        // 设置主机名
        factory.setHost("node1");
        // 设置虚拟主机名称 /在url中的转义字符 %2f
        factory.setVirtualHost("/");
        // 用户名
        factory.setUsername("root");
        // 密码
        factory.setPassword("123456");
        // amqp的端口号
        factory.setPort(5672);
        // 建立tcp连接
        Connection connection = factory.newConnection();
        // 获取通道
        Channel channel = connection.createChannel();

        // 声明消息队列
            // 消息队列名称
            // 是否是持久化的
            // 是否是排他的
            // 是否是自动删除的
            // 消息队列属性信息，使用默认值
        channel.queueDeclare("queue.biz",false,false,true,null);

        // 声明一个交换器
            // 交换器名称
            // 交换器类型
            // 是否是持久化的
            // 是否是自动删除的
            // 交换器的属性map集合
        channel.exchangeDeclare("ex.biz", BuiltinExchangeType.DIRECT,false,false,null);

        // 较交换器和消息队列绑定，并指定路由键
        channel.queueBind("queue.biz","ex.biz","hello.world");

        // 发送消息
            // 交换器名字
            // 该消息的路由键
            // 该消息的属性BasicProperties对象
            // 消息字节数组
        channel.basicPublish("ex.biz","hello.world",null,"hello world 2".getBytes());

        // 关闭通道
        channel.close();
        // 关闭连接
        connection.close();
    }
}

```

*HelloConsumeConsumer*

```java
package com.turbo.rabbitmq.demo;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.net.URISyntaxException;
import java.security.KeyManagementException;
import java.security.NoSuchAlgorithmException;
import java.util.concurrent.TimeoutException;

public class HelloConsumeConsumer {

    public static void main(String[] args) throws NoSuchAlgorithmException, KeyManagementException, URISyntaxException, IOException, TimeoutException {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setUri("amqp://root:123456@node1:5672/%2f");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        // 确保MQ中有该队列，如果没有则创建
        channel.queueDeclare("queue.biz",false,false,true,null);

        // 监听消息 一旦有消息推送过来，就调用第一个Lambda
        // 消息推送方式 
        channel.basicConsume("queue.biz", (consumerTag, message) -> {
            //消息推送的回调函数
            System.out.println(new String(message.getBody()));
        }, (consumerTag)->{
            // 客户端忽略该消息的回调方法
        });
    }
}

```

*HelloGetConsumer* 拉消息模式

```java
package com.turbo.rabbitmq.demo;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.net.URISyntaxException;
import java.security.KeyManagementException;
import java.security.NoSuchAlgorithmException;
import java.util.concurrent.TimeoutException;

public class HelloGetConsumer {

    public static void main(String[] args) throws NoSuchAlgorithmException, KeyManagementException, URISyntaxException, IOException, TimeoutException {

        ConnectionFactory factory = new ConnectionFactory();
        // 指定协议 amqp:
        // 指定 用户名   root
        // 指定密码    123456
        // 指定host   node1
        // 指定端口号    5672
        // 指定虚拟主机   %2f
        factory.setUri("amqp://root:123456@node1:5672/%2f");

        Connection connection = factory.newConnection();

        Channel channel = connection.createChannel();

        // 拉消息模式
        // 指定从哪个消费者消费
        // 指定是否自动确认消息   true表示自动确认
        GetResponse getResponse = channel.basicGet("queue.biz", true);

        // 获取消息体
        byte[] body = getResponse.getBody();
        System.out.println(new String(body));

        //AMQP.BasicProperties props = getResponse.getProps();

        channel.close();
        connection.close();
    }
}
```



### 1.4.4 Connection和Channel关系

生产者和消费者，需要与RabbitMQ Broker建立TCP连接，也就是Connection。一旦TCP连接建立，客户端紧接着创建一个AMQP信道（Channel），每一个信道都会指派唯一的ID。信道是建立在Connection之上的虚拟连接，RabbitMQ处理的每条AMQP指令都是通过信道完成的。

![image-20210908155034017](assest/image-20210908155034017.png)

> 为什么不直接使用TCP连接，而是使用信道？
>
> 答：RabbitMQ采用类似NIO的做法，复用TCP连接，减少性能开销，便于管理。当每个信道的流量不是很大时，复用单一的Connection可以在产生性能瓶颈的情况下节省TCP连接资源。
>
> 当信道本身的流量很大时，一个Connection就会产生性能瓶颈，流量被限制。需要建立多个Connection，分摊信道。具体的调优看业务需要。

信道在AMQP中是一个很重要的概念，大多数操作都是在信道这个层面进行的。

```java
channel.exchangeDeclare
channel.queueDeclare
channel.basicPublish
channel.basicConsume

```

RabbitMQ相关的API与AMQP紧密相连，比如channel.basicPublish对应AMQP的Basic.Publish命令。

## 1.5 RabbitMQ工作模式详解

https://www.rabbitmq.com/getstarted.html

### 1.5.1 Work Queue

生产者发布消息，启动多个消费者实例来消费，每个消费者仅消费部分信息，可达到负载均衡的效果。

代码：https://gitee.com/turboYuu/rabbit-mq-6-1/tree/master/lab/rabbit-demo/demo_03_wq

![image-20210908160312932](assest/image-20210908160312932.png)

*Producer*

```java
package com.turbo.rabbitmq.demo;

import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class Producer {

    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setUri("amqp://root:123456@node1:5672/%2f");
        Connection connection = factory.newConnection();

        Channel channel = connection.createChannel();
        // 声明一个消息队列 在rabbitmq中有就使用（属性不一样的话 报错），没有就创建，
        channel.queueDeclare("queue.wq",true,false,false,null);

        // 声明direct交换器
        channel.exchangeDeclare("ex.wq", BuiltinExchangeType.DIRECT,true,false,null);

        // 将消息队列绑定到指定的交换器，并指定绑定键
        channel.queueBind("queue.wq","ex.wq","key.wq");

        for (int i = 0; i < 15; i++) {
            channel.basicPublish("ex.wq","key.wq",null,
                    ("工作队列"+i).getBytes("utf-8"));
        }
        channel.close();
        connection.close();
    }
}
```

*Consumer*

```java
package com.turbo.rabbitmq.demo;

import com.rabbitmq.client.*;

import java.io.IOException;

public class Consumer {

    public static void main(String[] args) throws Exception{
        ConnectionFactory factory = new ConnectionFactory();
        factory.setUri("amqp://root:123456@node1:5672/%2f");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
        
        channel.queueDeclare("queue.wq",true,false,false,null);

        channel.basicConsume("queue.wq", new DeliverCallback() {
            public void handle(String consumerTag, Delivery message) throws IOException {
                System.out.println("推送来的消息"+ new String(message.getBody(),"utf-8"));
            }
        }, new CancelCallback() {
            public void handle(String consumerTag) throws IOException {
                System.out.println("Cancel:"+consumerTag);
            }
        });
    }
}
```



### 1.5.2 发布订阅模式

使用fanout类型交换器，routingKey忽略。每个消费者定义一个队列并绑定到同一个Exchange，每个消费者都可以消费到完整的消息。

消息广播给所有订阅该消息的消费者。

在RabbitMQ中，生产者不是讲消息直接发送给消息队列，实际上生产者根本不知道一个消息被发送到哪个队列。

生产者将消息发送给交换器。交换器非常简单，从生产者接收消息，将消息推送给消息队列。交换器必须清除的知道怎么处理接收到的消息，应该是追加到一个指定的队列，还是追加到多个队列，还是丢弃。规则就是交换器类型。

![image-20210908162651941](assest/image-20210908162651941.png)

发布订阅，交换器使用fanout类型。

代码：https://gitee.com/turboYuu/rabbit-mq-6-1/tree/master/lab/rabbit-demo/demo_04_fanout

消费者可以使用临时队列 ，将临时队列和fanout类型的交换器绑定（此时不需要路由键），接收消息

```shell
生成的临时队列名称为：amq.gen-2_X5vNhI6_ZihUJ3taFpYQ
```



#### 1.5.2.1 默认(未命名)交换器

代码：https://gitee.com/turboYuu/rabbit-mq-6-1/tree/master/lab/rabbit-demo/demo_05_default_ex

不指定交换器，依然可以向队列中发送消息，因为使用了默认交换器，默认交换器的类型是diret。

```java
// 在发送消息时，没有指定交换器名字，此时使用的是默认交换器，默认交换器就没有名字
// 路由键 就是 目的地 消息队列的名字
channel.basicPublish("","queue.default.ex",null,"hello turbo".getBytes());
```



```shell
#查看RabbitMQ中的交换器和对应类型，名称为空是默认（未命名）的交换器
rabbitmqctl list_exchanges --formatter pretty_table
```

![image-20210904141917484](assest/image-20210904141917484.png)

#### 1.5.2.2 临时队列

任何时候连接RabbitMQ，都需要一个新的空的队列，可以使用随机的名字创建，也可以让服务器生成随机的消费队列的名字。一旦断开消费者的连接，该队列应该自动删除。

```java
// 生成一个非持久化,排他的，自动删除的队列，并且名字是服务器随机生成的。
String queueName = channel.queueDeclare().getQueue();
//生成的临时队列名称为：amq.gen-2_X5vNhI6_ZihUJ3taFpYQ
```

**绑定**

![image-20210908165407143](assest/image-20210908165407143.png)

在创建了消息队列和`fanout`类型的交换器之后，需要将两者进行绑定，让交换器将消息发送给该队列。

![image-20210908165730868](assest/image-20210908165730868.png)



```shell
#查看RabbitMQ中的绑定
rabbitmqctl list_bindings --formatter pretty_table
```

![image-20210904142106438](assest/image-20210904142106438.png)



#### 1.5.2.3 消息的推拉

实现RabbitMQ的消费者有两种模式，推模式（Pull）和拉模式（Pull）。实现**推模式**的方式是继承DefaultConsumer基类，也可以使用Spring AMQP的SimpleMessgaeListenerContainer。推模式是最常用的，但是有些情况下推模式并不适用。需要批量拉取消息进行处理时，实现**拉模式**RabbitMQ的Channel提供了basicGet方法用于拉取消息。



### 1.5.3 路由模式

使用`direct`类型的交换器

代码：https://gitee.com/turboYuu/rabbit-mq-6-1/tree/master/lab/rabbit-demo/demo_06_routingmode



### 1.5.4 direct交换器



### 1.5.5 主题模式

代码：https://gitee.com/turboYuu/rabbit-mq-6-1/tree/master/lab/rabbit-demo/demo_07_topic_mode

使用`topic`类型的交换器，队列绑定到交换器，bindingKey使用通配符，交换器将消息路由到具体队列时会根据消息`routingKey`模糊匹配，比较灵活。

```java
// Producer部分代码，此处不声明队列,不做绑定
channel.exchangeDeclare("ex.topic", BuiltinExchangeType.TOPIC,true,false,null);
String area,level,biz;
String routingKey,message;
for (int i = 0; i < 100; i++) {
    area = LOG_AREA[RANDOM.nextInt(LOG_AREA.length)];
    level = LOG_LEVEL[RANDOM.nextInt(LOG_LEVEL.length)];
    biz = LOG_BIZ[RANDOM.nextInt(LOG_BIZ.length)];
    // routingkey包含三个维度
    routingKey = area+"."+biz+"."+level;

    message = "log:["+level+"] :这是 ["+area+"] 地区,"+biz+"服务发来的消息 Msg_seq:"+i;

    channel.basicPublish("ex.topic",routingKey,null,message.getBytes("utf-8"));
}
```



```java
// consumer部分代码
// 临时队列 返回值是服务器为该队列生成的名字
String queue = channel.queueDeclare().getQueue();
channel.exchangeDeclare("ex.topic", BuiltinExchangeType.TOPIC,true,false,null);

// beijing.biz-online.error
channel.queueBind(queue,"ex.topic","#.error");
```



![image-20210908172011555](assest/image-20210908172011555.png)



![image-20210904162608763](assest/image-20210904162608763.png)

## 1.6 Spring整合RabbitMQ

**spring-amqp**是对AMQP的一些概念的一些抽象，**spring-rabbit**是对RabbitMQ操作的封装实现。

主要有几个核心类`RabbitAdmin`、`RabbitTemplate`、`SimpleMessageListenerContainer`等。

- `RabbitAdmin`主要是完成对Exchange，Queue，Binding的操作，在容器中管理了`RabbitAdmin`类的时候，可以对Exchange、Queue、Binding进行自动声明。
- `RabbitTemplate`类是发送和接收消息的工具类。
- `SimpleMessageListenerContainer`是消费消息的容器。

目前比较新的一些项目都会选择基于注解方式，老项目可能还是基于配置文件。

### 1.6.1 基于配置文件的整合

https://gitee.com/turboYuu/rabbit-mq-6-1/tree/master/lab/rabbit-demo/demo_08_spring_rabbit_xml

1 创建Maven项目

2 配置pom.xml，添加rabbit的spring依赖

```xml
<dependency>
    <groupId>org.springframework.amqp</groupId>
    <artifactId>spring-rabbit</artifactId>
    <version>2.2.9.RELEASE</version>
</dependency>
```

3 rabbit-context.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:rabbit="http://www.springframework.org/schema/rabbit"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/rabbit
        http://www.springframework.org/schema/rabbit/spring-rabbit.xsd">

    <!--创建连接工厂-->
    <rabbit:connection-factory id="connectionFactory"
                               host="node1"
                               virtual-host="/"
                               username="root"
                               password="123456"
                               port="5672" />


    <!--用于自动向RabbitMQ声明队列，交换器，绑定等操作的工具类-->
    <rabbit:admin id="rabbitAdmin" connection-factory="connectionFactory" />

    <!--用于简化操作的模板类-->
    <rabbit:template id="rabbitTemplate" connection-factory="connectionFactory" />

    <!--声明一个消息队列-->
    <rabbit:queue id="q1" name="queue.q1" durable="false" exclusive="false" auto-delete="false" />

    <!--声明交换器-->
    <rabbit:direct-exchange name="ex.direct" durable="false" auto-delete="false" id="directExchange" >
        <rabbit:exchange-arguments>
            <entry key="" value=""/>
        </rabbit:exchange-arguments>
        <rabbit:bindings>
            <!-- key表示绑定键 -->
            <!--queue表示将交换器 绑定到哪个消息队列 ,不要使用队列的名字，要使用bean的id-->
            <!--exchange表示将交换器绑定到哪个交换器-->
            <!--<rabbit:binding queue="" key="" exchange=""></rabbit:binding>-->
            <rabbit:binding queue="q1" key="routing.q1"/>
        </rabbit:bindings>
    </rabbit:direct-exchange>
</beans>
```

4 Application.java

```java
package com.turbo.rabbitmq.demo;

import org.springframework.amqp.core.Message;
import org.springframework.amqp.core.MessageBuilder;
import org.springframework.amqp.core.MessageProperties;
import org.springframework.amqp.core.MessagePropertiesBuilder;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.context.support.AbstractApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import java.io.UnsupportedEncodingException;

public class ProducerApp {

    public static void main(String[] args) throws UnsupportedEncodingException {
        AbstractApplicationContext context = new ClassPathXmlApplicationContext("spring-rabbit.xml");

        RabbitTemplate template = context.getBean(RabbitTemplate.class);

        Message msg;
        MessagePropertiesBuilder builder = MessagePropertiesBuilder.newInstance();
        builder.setContentEncoding("gbk");
        builder.setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN);

        for (int i = 0; i < 1000; i++) {
            msg = MessageBuilder.withBody(("你好，世界"+i).getBytes("gbk"))
                    .andProperties(builder.build())
                    .build();
            template.send("ex.direct","routing.q1",msg);
        }
        context.close();
    }
}
```

启动RabbitMQ后，直接运行即可。

消息的消息

拉取模式：https://gitee.com/turboYuu/rabbit-mq-6-1/tree/master/lab/rabbit-demo/demo_09_spring_rabbit_xml_consumer

推送模式（监听）：https://gitee.com/turboYuu/rabbit-mq-6-1/tree/master/lab/rabbit-demo/demo_10_spring_rabbit_xml_listener



### 1.6.2 基于注解的整合

https://gitee.com/turboYuu/rabbit-mq-6-1/tree/master/lab/rabbit-demo/demo_11_spring_rabbit_anno_producer

1 创建Maven项目

2 配置pom.xml，添加rabbit的spring依赖

```java
<dependency>
    <groupId>org.springframework.amqp</groupId>
    <artifactId>spring-rabbit</artifactId>
    <version>2.2.9.RELEASE</version>
</dependency>
```

3 添加配置类RabbitConfiguration.java

```java
package com.turbo.rabbitmq.demo;

import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitAdmin;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.net.URI;

@Configuration
public class rabbitConfig {

    // 连接工厂
    @Bean
    public ConnectionFactory connectionFactory(){
        ConnectionFactory factory = new 
            CachingConnectionFactory(URI.create("amqp://root:123456@node1:5672/%2f"));
        return  factory;
    }

    // RabbitTemplate
    @Bean
    @Autowired
    public RabbitTemplate rabbitTemplate(ConnectionFactory factory){
        RabbitTemplate rabbitTemplate = new RabbitTemplate(factory);
        return rabbitTemplate;
    }

    // RabbitAmin
    @Bean
    @Autowired
    public RabbitAdmin rabbitAdmin(ConnectionFactory factory){
        RabbitAdmin rabbitAdmin = new RabbitAdmin(factory);
        return rabbitAdmin;
    }

    // Queue
    @Bean
    public Queue queue(){
        Queue queue = QueueBuilder.nonDurable("queue.anno").build();
        return queue;
    }

    // Exchange
    @Bean
    public Exchange exchange(){
        FanoutExchange fanoutExchange = new FanoutExchange("ex.anno.fanout", false, false, null);
        return fanoutExchange;
    }

    // Binding
    @Bean
    @Autowired
    public Binding binding(Queue queue,Exchange exchange){
        // 创建一个绑定，不指定绑定的参数
        Binding binding = BindingBuilder.bind(queue).to(exchange).with("key.anno").noargs();
        return binding;
    }
}
```

4 主入口类

```
package com.turbo.rabbitmq.demo;

import org.springframework.amqp.core.Message;
import org.springframework.amqp.core.MessageBuilder;
import org.springframework.amqp.core.MessageProperties;
import org.springframework.amqp.core.MessagePropertiesBuilder;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.support.AbstractApplicationContext;
import java.io.UnsupportedEncodingException;

public class ProducerApp {

    public static void main(String[] args) throws UnsupportedEncodingException {
        AbstractApplicationContext context = new AnnotationConfigApplicationContext(rabbitConfig.class);

        RabbitTemplate rabbitTemplate = context.getBean(RabbitTemplate.class);

        MessageProperties messageProperties = MessagePropertiesBuilder.newInstance()
                .setContentEncoding("gbk")
                .setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN)
                .setHeader("myKey","myValue")
                .build();

        for (int i = 0; i < 1000; i++) {
            Message message = MessageBuilder.withBody(("你好，turbo:"+i).getBytes("gbk"))
                    .andProperties(messageProperties)
                    .build();
            rabbitTemplate.send("ex.anno.fanout","key.anno",message);
        }
        context.close();
    }
}

```

和配置文件方式基本一致，只是spring上下文的生成方式有区别

消息拉取模式：https://gitee.com/turboYuu/rabbit-mq-6-1/tree/master/lab/rabbit-demo/demo_12_spring_rabbit_anno_consumer

消息推模式：https://gitee.com/turboYuu/rabbit-mq-6-1/tree/master/lab/rabbit-demo/demo_13_spring_rabbit_anno_listener

## 1.7 SpringBoot整合RabbitMQ

https://gitee.com/turboYuu/rabbit-mq-6-1/tree/master/lab/rabbit-demo/demo_14_springboot_rabbitmq

1 添加starter依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

2 application.properties中添加连接信息

```properties
spring.application.name=springboot_rabbitmq_consumer
spring.rabbitmq.host=node1
spring.rabbitmq.virtual-host=/
spring.rabbitmq.username=root
spring.rabbitmq.password=123456
spring.rabbitmq.port=5672
```

3 主入口类

```
@SpringBootApplication
public class Demo15SpringbootRabbitmqConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(Demo15SpringbootRabbitmqConsumerApplication.class, args);
    }
}
```

4 RabbitConfig类

```java
package com.turbo.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.Exchange;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.TopicExchange;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitConfig {

    @Bean
    public Queue queue(){
        return new Queue("queue.boot",
                false,
                false,
                false,
                null);
    }

    @Bean
    public Exchange exchange(){
        // 交换器类型（交换器名称，是否是持久化的，是否自动删除，交换器属性 Map集合）
        return new TopicExchange("ex.boot",
                false,
                false,
                null);
    }

    @Bean
    public Binding binding(){
        // 绑定了交换器ex.boot到队列queue.boot，路由key是key.boot
        return new Binding("queue.boot",
                Binding.DestinationType.QUEUE,
                "ex.boot",
                "key.boot",
                null);
    }
}
```

5 使用RestController发送消息

```java
package com.turbo.controller;

import org.springframework.amqp.core.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import java.io.UnsupportedEncodingException;

@RestController
public class MessageController {

    @Autowired
    private AmqpTemplate rabbitTemplate;

    @RequestMapping("/rabbit/{message}")
    public String receive(@PathVariable String message) throws UnsupportedEncodingException {

        MessageProperties properties = MessagePropertiesBuilder.newInstance()
        		.setContentEncoding("utf-8")
                .setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN)
                .setHeader("hello", "world").build();


        Message msg = MessageBuilder.withBody(message.getBytes("utf-8"))
                .andProperties(properties)
//                .setContentEncoding("utf-8")
//                .setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN)
                .build();

        rabbitTemplate.send("ex.boot","key.boot",msg);
        return  "ok";
    }
}

```

6 使用监听，用于推消息

```java
package com.turbo.consumer;

import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;

@Component
public class MyMessageListener {

    @RabbitListener(queues = "queue.boot")
    public void getMyMessage(@Payload String message, @Header(name = "hello") String value){
        System.out.println(message);
        System.out.println("hello = "+value);
    }
}
```

7 拉取消息

```java
rabbitTemplate.execute(new ChannelCallback<String>() {
    @Override
    public String doInRabbit(Channel channel) throws Exception {
        final GetResponse getResponse = channel.basicGet("queue.biz", false);
        if(null != getResponse){
            System.out.println(new String(getResponse.getBody(),
                                          getResponse.getProps().getContentEncoding()));
            channel.basicAck(getResponse.getEnvelope().getDeliveryTag(),false);
        }
        return null;
    }
});
```



# 2 RabbitMQ高级特性解析

创建一个用于RabbitMQ重置，并添加用户root，配置标签和权限的文件

```shell
[root@node1 /]# cat /usr/local/bin/resetrabbit 
#!/bin/bash
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl start_app
rabbitmqctl add_user root 123456
rabbitmqctl set_user_tags root administrator
rabbitmqctl set_permissions --vhost / root ".*" ".*" ".*"
```

要改为可执行文件

`chmod +x resetrabbit`

## 2.1 消息可靠性

支付平台如何保证转账、付款不出问题？

![image-20210908181647150](assest/image-20210908181647150.png)

支付平台必须保证数据正确性，保证数据并发安全性，保证数据最终一致性。

支付平台通过如下几种方式**保证数据一致性**：

- 分布式锁

  比较容易理解，就是在操作某条数据式先锁定，可以使用Redis或Zookeeper等常用框架来实现。如在修改账单时，先锁定该账单，如果该账单有并发操作，后面的操作只能等待上一个操作的锁释放后再依次执行。

  **优点：**能够保证数据强一致性。**缺点**：高并发场景下有性能问题。

- 消息队列

  消息队列是为了保证最终一致性，需要确保消息队列有ack机制，客户端收到消息并消费处理完成后，客户端发送ack消息给消息中间件，如果消息中间件超过指定时间还没有收到ack，则定时重发消息。

  **优点：**异步，高并发。**缺点：**有一定延时，数据弱一致性，并且必须确保该业务操作，肯定能够成功完成，不能失败。

可以从以下几个方面来**保证消息的可靠性**：

- 客户端代码中的异常捕获机制，包括生产者和消费者
- AMQP/RabbitMQ的事务机制
- 发送端确认机制
- 消息持久化机制
- Broker端的高可用集群
- 消费者确认机制
- 消费端限流
- 消息幂等性

### 2.1.1 异常捕获机制

先执行业务操作，业务操作成功后执行消息发送，消息发送过程中通过try cache方式捕获异常，在异常处理的代码块中执行回滚操作或者重发操作。这是一种最大努力确保的方式，并无法保证100%绝对可靠，因为这里没有异常不代表就一定投递成功。

![image-20210908183930746](assest/image-20210908183930746.png)

可以通过`spring.rabbitmq.template.retry.enabled=true`配置开启发送端的重试。

### 2.1.2 AMQP/RabbitMQ的事务机制

一直到事务提交后都没有异常，确实说明消息是投递成功了。但是，这种方式在性能方面的**开销比较大**，一般不推荐使用。

![image-20210908185527286](assest/image-20210908185527286.png)

### 2.1.3 发送端确认机制

RabbitMQ后来引入了一种轻量级的方式，叫**发送方确认**（publisher confirm）机制。生产者将信道设置成confirm（确认）模式，所有在该信道上面发布的消息都会被指派一个唯一的ID（从1开始），一旦消息被投递到所有匹配的队列之后（如果消息和队列是持久化的，那么确认消息会在消息持久化后发出），RabbitMQ就会发送一个确认（Basic.Ack）给生产者（包含消息的唯一ID），这样生产者就知道消息已经正确送达了。

![image-20210908190209033](assest/image-20210908190209033.png)

RabbirMQ回传给生产者的确认消息中的deliveryTag字段，包含了确认消息的序号，另外，通过设置channel.basicAck方法中的multiple参数（表示到这个序号之前的所有消息是否都已经得到了处理）。生产者投递消息后并不需要一致阻塞着，可以继续投递下一条消息并通过回调方式处理ACK响应。如果RabbitMQ因为自身内部错误导致消息丢失等异常情况发生，就会响应一条nack（Basic.Nack）命令，生产者应用程序同样可以在回调方法中处理该nack命令。

代码：https://gitee.com/turboYuu/rabbit-mq-6-1/tree/master/lab/rabbit-demo/demo_20_publishconfirms_01

*同步的方式等待RabbitMQ的确认消息*

```java
package com.turbo.rabbitmq.demo;

import com.rabbitmq.client.*;
import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class PublisherConfirmProducer {

    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setUri("amqp://root:123456@node1:5672/%2f");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
        // 向RabbitMQ服务器发送AMQP命令，将当前通道标记为发送方确认通道
        AMQP.Confirm.SelectOk selectOk = channel.confirmSelect();

        channel.queueDeclare("queue.pc",true,false,false,null);
        channel.exchangeDeclare("ex.pc", BuiltinExchangeType.DIRECT,true,false,null);
        channel.queueBind("queue.pc","ex.pc","key.pc");

        // 发送消息
        channel.basicPublish("ex.pc","key.pc",null,"hello world".getBytes());
        try {
            // 同步的方式等待RabbitMQ的确认消息 5000ms
            channel.waitForConfirmsOrDie(5_000);
            System.out.println("消息已经得到确认");
        }catch (IOException ex){
            System.out.println("消息被拒收");
        }catch (IllegalStateException ex){
            System.out.println("发送消息的通道不是publisherConfirms通道");
        }catch (TimeoutException ex){
            System.out.println("等待消息确认超时");
        }
        channel.close();
        connection.close();
    }
}
```

waitForConfirm方法有个重载的，可以自定义timeout超时时间，超时后会抛出TimeoutException。类似的有几个waitForConfirmsOrDie方法，Broker端在返回nack（Basic.Nack）之后该方法会抛出java.io.IOException。需要根据异常类型来做区别处理，TImeoutException超时属于第三状态（无法确定成功还是失败），而返回Basic.Nack抛出IOException这种是明确的失败。以上代码还是同步阻塞模式，性能不是太好。



实际上，也可以通过**批处理**的方式来改善整体的性能（即批量发送消息后，仅调用一次waitForConfirms方法）。正常情况下这种批量处理的方式效率高很多，但是如果发生超时或者nack（失败）后，就需要批量重发消息或者通知上游业务批量回滚。所以，批量重发消息肯定会造成部分消息重复。可以**通过异步回调的方式来处理Broker的响应**。addConfirmListener方法可以添加ConfirmListener这个回调接口，这个ConfirmListener家口包含两个方法：handleAck和handleNack，分别用来处理RabbitMQ回传Basic.Ack和Basic.Nack。

*原生API批量确认*

```java
package com.turbo.rabbitmq.demo;

import com.rabbitmq.client.*;
import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class PublisherConfirmProducer {

    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setUri("amqp://root:123456@node1:5672/%2f");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        // 向RabbitMQ服务器发送AMQP命令，将当前通道标记为发送方确认通道
        AMQP.Confirm.SelectOk selectOk = channel.confirmSelect();
        channel.queueDeclare("queue.pc",true,false,false,null);
        channel.exchangeDeclare("ex.pc", BuiltinExchangeType.DIRECT,true,false,null);
        channel.queueBind("queue.pc","ex.pc","key.pc");
        // 发送消息
        channel.basicPublish("ex.pc","key.pc",null,"hello world".getBytes());
        try {
            // 同步的方式等待RabbitMQ的确认消息
            channel.waitForConfirmsOrDie(5_000);
            System.out.println("消息已经得到确认");
        }catch (IOException ex){
            System.out.println("消息被拒收");
        }catch (IllegalStateException ex){
            System.out.println("发送消息的通道不是publisherConfirms通道");
        }catch (TimeoutException ex){
            System.out.println("等待消息确认超时");
        }
        channel.close();
        connection.close();
    }
}
```

*回调方法确认*

````java
package com.turbo.rabbitmq.demo;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.ConcurrentNavigableMap;
import java.util.concurrent.ConcurrentSkipListMap;

public class PublisherConfirmsProducer3 {
    public static void main(String[] args) throws Exception {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setUri("amqp://root:123456@node1:5672/%2f");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        // 向RabbitMQ服务器发送AMQP命令，将当前通道标记为发送方确认通道
        AMQP.Confirm.SelectOk selectOk = channel.confirmSelect();
        channel.queueDeclare("queue.pc",true,false,false,null);
        channel.exchangeDeclare("ex.pc", BuiltinExchangeType.DIRECT,true,false,null);
        channel.queueBind("queue.pc","ex.pc","key.pc");

        ConcurrentNavigableMap<Long,String> outstandingConfirms = new ConcurrentSkipListMap<>();
        ConfirmCallback clearOutstandingConfirms = ( deliveryTag,  multiple)->{
            if(multiple){
                System.out.println("编号小于等于："+deliveryTag+" 的消息都已经被确认了");
                ConcurrentNavigableMap<Long, String> headMap
                        = outstandingConfirms.headMap(deliveryTag, true);
                // 清空outstandingConfirms中已经被确认的消息信息
                headMap.clear();
            }else {
                // 移除已经被确认的消息
                outstandingConfirms.remove(deliveryTag);
                System.out.println("编号为："+deliveryTag+" 的消息被确认了");
            }
        };

        // 设置channel的监听器，处理确认的消息和不确认的消息
        channel.addConfirmListener(clearOutstandingConfirms,(deliveryTag, multiple) -> {
            if(multiple){
                // 将没有确认的消息记录到一个集合中...
                System.out.println("消息编号小于等于："+deliveryTag+" 的消息不确认");
            }else {
                System.out.println("消息编号为："+deliveryTag+" 的消息不确认");
            }
        });

        String message = "hello - ";
        for (int i = 0; i < 1000; i++) {
            // 获取下一条即将发送消息的编号
            long nextPublishSeqNo = channel.getNextPublishSeqNo();
            channel.basicPublish("ex.pc","key.pc",null,(message+i).getBytes());

            System.out.println("编号为："+nextPublishSeqNo +" 的消息已经发送成功，尚未确认");
            outstandingConfirms.put(nextPublishSeqNo,(message+i));
        }
        Thread.sleep(10000);
        channel.close();
        connection.close();
    }
}
````

#### 2.1.3.1 springboot案例



### 2.1.4 持久化存储机制

持久化是提高RabbitMQ可靠性的基础，否则当RabbitMQ遇到异常时，数据将会丢失，主要从以下几个方面来保证消息的持久性：

- Exchange的持久化，通过定义时设置durable参数为true，保证Exchange相关的元数据不丢失
- Queue的持久化，通过定义时设置durable参数为true，保证Queue相关的元数据不丢失
- 消息的持久化，通过将消息的投递模式（BasicProperties中的deliveryMode属性）设置为2，即可实现消息的持久化，保证消息自身不丢失。

代码：https://gitee.com/turboYuu/rabbit-mq-6-1/tree/master/lab/rabbit-demo/demo_21_persistent



![image-20210908200738596](assest/image-20210908200738596.png)

RabbitMQ中的持久化信息都需要写入磁盘（当系统内存不足时，非持久化的消息也会被刷盘处理），这些处理动作都是在“持久层”中完成的。吃就成是一个逻辑上的概念，实际包括两个部分：

- 队列索引（rabbit_queue_index），rabbit_queue_index负载维护Queue中消息的信息，包括消息的存储位置，是否已经交给消费者，是否已经被消费及Ack确认等，每个Queue都有与之对应的rabbit_queue_index。
- 消息存储（rabbit_msg_store），rabbit_msg_store以键值对的形式存储消息，它被所有队列共享，在每个节点中有且只有一个。



下图是RabbitMQ的实际存储消息的位置，其中queues目录中保存着rabbit_queue_index相关数据，msg_store_persistent保存着持久化消息数据，msg_store_transient保存着非持久化的数据。

另外，RabbitMQ通过配置`queue_index_embed_msgs_below`可以根据消息大小决定存储位置，默认`queue_index_embed_msgs_below`是4096字节（包含消息体，属性及headers），小于该值的消息存在rabbit_queue_index中。

![image-20210905162445181](assest/image-20210905162445181.png)

![image-20210905162947803](assest/image-20210905162947803.png)

### 2.1.5 Consumer ACK

RabbitMQ在消费端有Ack机制，即消费端消费消息后需要发送Ack确认报文给Broker端，告知自己是否已消费完成，否则可能会一直重发知道消息过期（AUTO模式）。

这也是“最终一致性”、“可恢复性”的基础。

有如下处理手段：

- 采用NONE模式，消费的过程中自行捕获异常。
- AUTO（自动Ack）模式，不主动捕获异常，当消费过程中出现异常时会将消息放回Queue中，然后消息会被分配到其他消费节点（如果没有，则还是选择当前节点）重新被消费，默认会一直重发消息并指导消费完成返回Ack回执一直到过期。
- MANUAL（手动模式），消费者自行控制流程并手动调用channel的相关方法返回Ack



在消费端直接指定ackMode

```java
/**
    * NONE模式，则只要收到消息后就立即确认（消息出列，标记已消费），有丢失数据的风险     
    * AUTO模式，看情况确认，如果此时消费者抛出异常则消息会返回到队列中
    * MANUAL模式，需要显式的调用当前channel的basicAck方法     
    * @param channel
    * @param deliveryTag     
    * @param message
    */
   @RabbitListener(queues = "lagou.topic.queue", ackMode = "AUTO")    
   public void handleMessageTopic(Channel channel,
		@Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag, @Payload byte[] message) {
       
       System.out.println("RabbitListener消费消息，消息内容：" + new String((message)));
       try {
			// 手动ack，deliveryTag表示消息的唯一标志，multiple表示是否是批量确认           	 			
			channel.basicAck(deliveryTag, false);
           	// 手动nack，告诉broker消费者处理失败，最后一个参数表示是否需要将消息重新 入列
           	channel.basicNack(deliveryTag, false, true);           
           	// 手动拒绝消息。第二个参数表示是否重新入列
           	channel.basicReject(deliveryTag, true);        
		} catch (IOException e) {
			e.printStackTrace();      
		}
 	}
```



SpringBoot项目中支持如下的一些配置：

```properties
#最大重试次数
spring.rabbitmq.listener.simple.retry.max-attempts=5

#是否开启消费者重试（false-关闭消费者重试，意思不是"不重试",而是一直收消息知道jack确认或者一直到超时）
spring.rabbitmq.listener.simple.retry.enabled=true

#重试时间间隔（ms）
spring.rabbitmq.listener.simple.retry.initial-interval=500

#重试超过最大次数后是否拒绝
spring.rabbitmq.listener.simple.default-requeue-rejected=false

#ack模式
spring.rabbitmq.listener.simple.acknowledge-mode=manual
```

本小节的内容总结起来，本质上就是“请求/应答”确认模式

#### 2.1.5.1 springboot案例



### 2.1.6 消费端限流

当消费投递速度远快于消费速度时，随着时间积累就会出现消息积压。消息中间件本身具备一定的缓冲能力，但这个能力是**有容量限制**的，如果长期运行并没有任何处理，最终会导致Broker崩溃，而分布式系统的故障往往会发生在上下游传递，连锁反应就悲剧了。

从多个角度介绍Qos与限流，防止悲剧发生：

1. RabbitMQ可以对**内存和磁盘使用量**设置阈值，**当到达阈值后，生产者将被阻塞（block）**，直到对应指标恢复正常。全局上可以防止超大流量，消息积压等导致的Broker被压垮。当内存受限或磁盘可用空间受限的时候，服务器都会暂时阻止连接，服务器将暂停从发布消息的已连接客户端的套接字读取数据。连心跳监视也将被禁用。所有网络连接将在rabbitmqctl和管理插件中显示为“以阻止”，这意味着它们已发布，现在已暂停。兼容的客户端被阻止时将收到通知。

在/etc/rabbitmq/rabbitmq.conf中配置磁盘可用空间大小：

![image-20210908213635781](assest/image-20210908213635781.png)![image-20210908213646907](assest/image-20210908213646907.png)

2. RabbitMQ还默认提供了一种基于credit flow的流控机制，面向每一个连接进行流控。当单个队列达到最大流速时，或者多个队列达到总流速时，都会触发流控。触发单个连接的流控可能是因为connection、channel、queue的某一个过程处于flow状态，这些状态都可以从监控平台看到。

![image-20210909113244741](assest/image-20210909113244741.png)

![image-20210909113333368](assest/image-20210909113333368.png)

![image-20210909113436778](assest/image-20210909113436778.png)

![image-20210909113530675](assest/image-20210909113530675.png)

![image-20210909113627283](assest/image-20210909113627283.png)

3. RabbitMQ中有一种QoS保证机制，可以限制Channel上接收的未被Ack的消息数量，如果超过这个数量限制，RabbitMQ将不会再往消费端推送消息。这是一种流控手段，可以防止大量消息瞬时从Broker送达消费端造成消息端巨大压力（甚至压垮消费端）。比较值得注意的是QoS机制仅对消费端推模式有效，对拉模式无效。而且不支持NONE Ack模式。执行`channel.basicConsume`方法之前通过`channel.basicQoS`方法可以设置该数量。消息的发送是异步的，消息的确认也是异步的。再消费者消费慢的时候，可以设置QoS的prefetchCount，它表示broker在向消费者发送消息的时候，一旦发送了prefetchCount个消息而没有一个消息确认的时候，就停止发送。消费者确认一个，broker就发送一个。换句话说，消费者确认多少，broker就发送多少，消费者等待处理的个数永远限制在prefetchCount个。

   如果对于每个消息都发送确认，增加了网络流量，此时可以批量确认消息。如果设置了multiple为true，消费者在确认的时候，比如id是8的消息确认了，则在8之前的所有消息都确认了。

   代码：https://gitee.com/turboYuu/rabbit-mq-6-1/tree/master/lab/rabbit-demo/demo_23_consumerQos

   ```java
   package com.turbo.rabbitmq.demo;
   
   import com.rabbitmq.client.*;
   import java.io.IOException;
   
   public class MyConsumer {
   
       public static void main(String[] args) throws Exception {
           ConnectionFactory factory = new ConnectionFactory();
           factory.setUri("amqp://root:123456/node1:5672/%2f");
           Connection connection = factory.newConnection();
           Channel channel = connection.createChannel();
   
           channel.queueDeclare("queue.qos",false,false,false,null);
           /**
            * 使用basic做限流 ，进队消息推送模式生效。
            */
           // 表示Qos是10个消息，最多有10个消息等待确认
           channel.basicQos(10);
           // 表示最多10个消息等待确认。
           // 如果global为true表示只要使用当前的channel的consumer,该设置都生效
           // false表示 仅限于当前consumer
           channel.basicQos(10,false);
           // 第一个参数表示未确认消息的大小，没有实现，不用管。
           channel.basicQos(1000,10,false);
   
           channel.basicConsume("queue.qos",false,new DefaultConsumer(channel){
               @Override
               public void handleDelivery(String consumerTag,
                                          Envelope envelope,
                                          AMQP.BasicProperties properties,
                                          byte[] body) throws IOException {
                   // some code going on
                   // 可以批量确认消息，减少每个消息都发送确认带来的网络流量负载。
                   channel.basicAck(envelope.getDeliveryTag(),true);
               }
           });
           channel.close();
           connection.close();
       }
   }
   ```

   

生产者往往希望自己生产的消息能快速投递出去，而当消息投递太快且超过了下游的消费速度时就容易出现消息积压/堆积，所以，从上游来讲应该在生产段应用程序中也加入限流，应急开关等控制手段，避免超过Broker端的极限承受能力或者压垮下游消费者。

希望下游消费者能尽快消费完信息，而且还要防止瞬时大量消息压垮消费端（推模式），希望消费端处理速度时最快的，最稳定而亲还相对均匀。

**提升下游应用的吞吐量和缩短消费过程的耗时**，优化主要以下几种方式：

1. 优化应用程序的性能，缩短响应时间
2. 增加消费节点实例（成本增加，而且底层数据库操作这些也可能是瓶颈）
3. 调整并发消费的线程数（线程数并非越大越好，需要大量压测调优至合理值）

![image-20210909131745069](assest/image-20210909131745069.png)

```java
@Bean
public RabbitListenerContainerFactory rabbitListenerContainerFactory(ConnectionFactory connectionFactory) {
	// SimpleRabbitListenerContainerFactory发现消息中有content_type有text 就会默认将其
	// 转换为String类型的，没有content_type都按byte[]类型        
	SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
	factory.setConnectionFactory(connectionFactory);	// 设置并发线程数
	factory.setConcurrentConsumers(10);					// 设置最大并发线程数
	factory.setMaxConcurrentConsumers(20);        
    return factory;
}
```



### 2.1.7 消费可靠性保障

在讲高级特性的时候都会涉及到

1. 消息传输保障
2. 各种限流，应急手段
3. 业务层面的一些容错、补偿、异常重试等手段

**消息可靠传输**一般是业务系统接入消息中间件时**首要考虑的问题**，一般消息中间件的消息传出保障分为三个层次：

1. At most once：最多一次。消息可能会丢失，但绝**不会重复**传输，**基本不会使用**。
2. At Least once：最少一次。消息绝不会丢失，但**可能会重复**传输
3. Exactly once：恰好一次，每条消息肯定被**传输一次且仅传输一次**

RabbitMQ其中“最多一次”和“最少一次”。



其中**“最少一次”**投递需要考虑以下这几个方面的内容：

1. 消费生产者需要开启事务机制或者publisher confirm机制，已确保消息可以可靠得传输到RabbitMQ中。
2. 消息生产者需要配合使用`mandatory`参数或者备份交换器来保证消息能够从交换器路由到队列中，进而能够保证保存下来而不会被丢弃。
3. 消息和队列都需进行持久化处理，以确保RabbitMQ服务器在遇到异常情况时不会造成消息丢失
4. 消费者在消费的同时需要将autoAck设置为false，然后通过手动确认的方式去确认已经正确消费的消息，以避免在消费端引起不必要的消息丢失。

**“恰好一次”是RabbitMQ目前无法保障**的

考虑到这样一种情况，消费者在消费完一条消息之后向RabbitMQ发送确认Basic.Ack命令，此时由于网络断开或者其他原因造成RabbitMQ并没有收到这个确认命令，那么RabbitMQ不会将此条消息标记删除。在重新建立连接之后，消费还是会消费到这一条消息，这就造成了重复消费。

在考虑一种情况，生产者在使用publisher confirm机制的时候，发送完一条消息等待RabbitMQ返回确认通知，此时网络断开，生产者捕获到异常情况，为了确保消息的可靠性选择重新发送，这样RabbitMQ中就有两条同样的消息，在消费的时候消费者就会重复消息。

### 2.1.8 消费幂等性处理

追求性能就无法保证消息的顺序，而追求可靠性那么就可能产生重复消息，从而导致重复消费。**做架构就是权衡取舍**

RabbitMQ层面有实现**去重机制**来保证**恰好一次**吗？并没有，这个在目前主流的消息中间件都没有实现。

借用淘宝沈洵的一句话：最好的解决颁发就是不解决。当为了在基础的分布式中间件中实现某种相对不太通用的功能，需要牺牲到性能、可靠性、扩展性时，并且会额外增加很多复杂度，最简单的颁发就是交给业务自己去处理。事实证明，很多业务场景下是可以容忍重复消费的。

一般解决重复消费的办法是，在**消费端**让我们消费消息的操作具备**幂等性。**



## 2.2 可靠性分析

开启Firehose命令

```shell
rabbitmqctl trace_on [-p vhost]
```

其中[-p vhost]是可选参数，用来指定虚拟主机vhost。

对应的关闭命令

```
rabbitmqctl trace_off [-p vhost]
```

![image-20210905191733551](assest/image-20210905191733551.png)



![image-20210905195549407](assest/image-20210905195549407.png)

![image-20210905200437481](assest/image-20210905200437481.png)

![image-20210905200502404](assest/image-20210905200502404.png)



## 2.3 TTL机制



```
rabbitmqctl set_policy q.ttl ".*" '{"message-ttl":20000}' --apply-to queues
rabbitmqctl set_policy q.ttl ".*" '{"message-ttl":20000,"expires":10000}' --apply-to queues
```

![image-20210905204823473](assest/image-20210905204823473.png)

![image-20210905204648123](assest/image-20210905204648123.png)

## 2.4 死信队列



## 2.5 延迟队列

https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases/tag/v3.8.0

![image-20210906110719127](assest/image-20210906110719127.png)



![image-20210906112214539](assest/image-20210906112214539.png)



![image-20210906112439362](assest/image-20210906112439362.png)

```
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```

![image-20210906112542811](assest/image-20210906112542811.png)

![image-20210906112901506](assest/image-20210906112901506.png)

# 3 RabbitMQ集群与运维

## 3.1 集群方案原理

### 3.1.1 业界实践

### 3.1.2 常用负载均衡算法

### 3.1.3 网络中的而经典问题

### 3.1.4 rabbitMQ分布式架构模式

#### 主备模式

#### Shovel铲子模式

#### RabbitMQ集群

##### 镜像队列模式

##### Federation联邦模式

## 3.2 单机多实例部署

此处在单机版本基础上，也就是一台Linux虚拟机上启动多个RabbitMQ实例，部署集群。

1 在单个Linux虚拟机上运行多个RabbitMQ实例

- 多个RabbitMQ使用的端口号不能冲突
- 多个RabbitMQ使用的磁盘存储路径不能冲突
- 多个RabbitMQ的配置文件不能冲突

在单个Linux虚拟机上运行多个RabbitMQ实例，涉及到RabbitMQ虚拟主机的名称不能重复，每个RabbitMQ使用的端口号不能重复。

`RABBITMQ_NODE_PORT`用于设置RabbitMQ的服务发现，对外发布的其他端口在这个端口基础上计算得来。

| 端口号     | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| 4369       | empd，RabbitMQ节点和CLI工具使用的对等发现服务                |
| 5672、5671 | 分别为不带TLS和带TLS的AMQP 0-9-1和1.0客户端使用              |
| 25672      | 用于节点键=间和CLI工具通信（Erlang分发服务端口），并从动态范围分配（默认情况下为单个端口，计算为AMQP端口+20000）。一般这些端口不应暴露出去。 |
|            |                                                              |
|            |                                                              |
|            |                                                              |
|            |                                                              |
|            |                                                              |
|            |                                                              |



在/opt/rabbitmqconf中创建三个配置文件：rabbit1.conf，rabbit2.conf，rabbit3.conf，其中要注明三个RabbitMQ实例使用的rabbitmq_management插件使用的端口号，以及开通guest远程登录系统的权限：

注意；修改权限

![image-20210907103928437](assest/image-20210907103928437.png)

| 文件路径                      | 内容                                                    |
| ----------------------------- | ------------------------------------------------------- |
| /opt/rabbitmqconf/rabbit1.cof | loopback_users.guest=fasle<br>management.tcp.port=6001  |
| /opt/rabbitmqconf/rabbit2.cof | loopback_users.guest=fasle<br/>management.tcp.port=6002 |
| /opt/rabbitmqconf/rabbit3.cof | loopback_users.guest=fasle<br/>management.tcp.port=6003 |

启动命令

| 节点名称 | 命令                                                         |
| -------- | ------------------------------------------------------------ |
| rabbit1  | RABBITMQ_NODENAME=rabbit1 RABBIT_NODE_PORT=5001 RABBITMQ_CONFIG_FILE=/opt/rabitmqconf/rabbit1.conf rabbitmq-server |
| rabbit2  | RABBITMQ_NODENAME=rabbit1 RABBIT_NODE_PORT=5002 RABBITMQ_CONFIG_FILE=/opt/rabitmqconf/rabbit1.conf rabbitmq-server |
| rabbit3  | RABBITMQ_NODENAME=rabbit1 RABBIT_NODE_PORT=5003 RABBITMQ_CONFIG_FILE=/opt/rabitmqconf/rabbit1.conf rabbitmq-server |

![image-20210907110211338](assest/image-20210907110211338.png)

停止命令：

`rabbitmqctl help stop`  查看停止命令帮助，停止指定节点

| 节点名称 | 命令                                                         |
| -------- | ------------------------------------------------------------ |
| rabbit1  | rabbitmqctl -n rabbit1 stop<br>或<br>rabbitmqctl --node rabbit1 stop |
| rabbit2  | rabbitmqctl -n rabbit2 stop                                  |
| rabbit3  | rabbitmqctl -n rabbit3 stop                                  |

![image-20210907110826750](assest/image-20210907110826750.png)

## 3.3 集群管理



## 3.4 RabbitMQ镜像集群配置

## 3.5 负载均衡-HAProxy

## 3.6 监控



# 4 RabbitMQ源码剖析

## 4.1 队列

## 4.2 交换器

## 4.3 持久化



### 4.3.1 消息入队分析

### 4.3.2 消息出队源码分析

## 4.4 启动过程

## 4.5 消息的发送

## 4.6 消息的消费

### 4.6.1 两种方式：推拉

### 4.6.2 拉消息的代码实现

### 4.6.3 推消息的代码实现