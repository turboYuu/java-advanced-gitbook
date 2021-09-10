第一部分 RocketMQ架构与实战

# 1 RocketMQ的前世今生

![image-20210910182151286](assest/image-20210910182151286.png)

# 2 RocketMQ的使用场景

- 应用解耦
- 流量削峰
- 数据分发

# 3 RocketMQ部署架构

RocketMQ的角色介绍

- Producer：消息的发送者，
- Consumer：消息接收者
- Broker：暂存和传输消息
- NameServer：管理Broker
- Topic：区分消息的种类，一个发送者可以发送消息给一个或多个Topic；一个消费的接收者可以订阅一个或多额Topic消息
- MessageQueue：相当于Topic的分区；用于并行发送和接收消息

# 4 RocketMQ特性

# 5 消费模式Push or Pull

# 6 RocketMQ中的角色及相关术语

# 7 RocketMQ环境搭建

## 7.1 软件准备

RocketMQ版本：4.5.1

https://www.apache.org/dyn/closer.cgi?path=rocketmq/4.5.1/rocketmq-all-4.5.1-bin-release.zip

## 7.2 环境要求

- JDK 11.0.5
- Linux64位操作系统（Centos Linux release 7.7 1908）
- 源码安装需要安装maven 3.2.x
- 最少4G存储空间

## 7.3 安装及启动

1. 下载rockermq

   ```
   wget https://archive.apache.org/dist/rocketmq/4.5.1/rocketmq-all-4.5.1-bin-release.zip
   ```

2. 解压

   ```
   #解压到/opt目录下
   unzip rocketmq-all-4.5.1-bin-release.zip -d /opt
   mv rocketmq-all-4.5.1-bin-release/ rocket
   ```

3. 修改环境变量

   安装JDK 11.0.5 ：rpm -ivh jdk-11.0.5_linux-x64_bin.rpm

   配置环境变量 vim /etc/profile

   ```shell
   export JAVA_HOME=/usr/java/jdk-11.0.5
   export PATH=$PATH:$JAVA_HOME/bin
   
   export ROCKET_HOME=/opt/rocket
   export PATH=$PATH:$ROCKET_HOME/bin
   
   #生效环境变量
   . /etc/profile
   ```

   输入mq,tab有提示，说明RocketMQ安装成功。

   ![image-20210910142958724](assest/image-20210910142958724.png)

   4.启动前要修改脚本，因为RocketMQ是用JDK8编译开发的，而现在使用JDK11。

   vim /opt/rocket/bin/runserver.sh

   vim /opt/rocket/bin/runbroker.sh

   vim /opt/rocket/bin/tools.sh





**修改vim /opt/rocket/bin/runserver.sh**

```sh
#!/bin/sh

# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

#===========================================================================================
# Java Environment Setting
#===========================================================================================
error_exit () 
{
echo "ERROR: $1 !!" 
exit 1
}

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"

export JAVA_HOME
export JAVA="$JAVA_HOME/bin/java"
export BASE_DIR=$(dirname $0)/..
export CLASSPATH=.:${BASE_DIR}/conf:${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib/*

#===========================================================================================
# JVM Configuration
#===========================================================================================
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m -XX:MetaspaceSize=64m -XX:MaxMetaspaceSize=160m"
JAVA_OPT="${JAVA_OPT} -XX:CMSInitiatingOccupancyFraction=70 -XX:+CMSParallelRemarkEnabled -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+CMSClassUnloadingEnabled -XX:SurvivorRatio=8"
JAVA_OPT="${JAVA_OPT} -verbose:gc -Xlog:gc:/dev/shm/rmq_srv_gc.log -XX:+PrintGCDetails"
JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
JAVA_OPT="${JAVA_OPT}  -XX:-UseLargePages"
#JAVA_OPT="${JAVA_OPT} -Djava.ext.dirs=${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib" 
#JAVA_OPT="${JAVA_OPT} -Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n" 
JAVA_OPT="${JAVA_OPT} ${JAVA_OPT_EXT}"
JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"

$JAVA ${JAVA_OPT} $@
```

启动namesrv：mqnamesrv

![image-20210909204354060](assest/image-20210909204354060.png)



**vim bin/runbroker.sh**

```sh
#!/bin/sh

# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with 
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

#===========================================================================================
# Java Environment Setting
#===========================================================================================
error_exit () 
{
   echo "ERROR: $1 !!"    
   exit 1
}

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"

export JAVA_HOME
export JAVA="$JAVA_HOME/bin/java"
export BASE_DIR=$(dirname $0)/..
export CLASSPATH=.${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib/*:${BASE_DIR}/conf:${CLASSPATH}

#===========================================================================================
# JVM Configuration
#===========================================================================================
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m"
JAVA_OPT="${JAVA_OPT} -XX:+UseG1GC -XX:G1HeapRegionSize=16m -XX:G1ReservePercent=25 -XX:InitiatingHeapOccupancyPercent=30 -XX:SoftRefLRUPolicyMSPerMB=0"
JAVA_OPT="${JAVA_OPT} -verbose:gc -Xloggc:/dev/shm/mq_gc_%p.log -XX:+PrintGCDetails"
JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
JAVA_OPT="${JAVA_OPT} -XX:+AlwaysPreTouch"
JAVA_OPT="${JAVA_OPT} -XX:MaxDirectMemorySize=15g"
JAVA_OPT="${JAVA_OPT} -XX:-UseLargePages -XX:-UseBiasedLocking"
#JAVA_OPT="${JAVA_OPT} -Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n" 
JAVA_OPT="${JAVA_OPT} ${JAVA_OPT_EXT}"
JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"

numactl --interleave=all pwd > /dev/null 2>&1
if [ $? -eq 0 ]
then
    if [ -z "$RMQ_NUMA_NODE" ] ; then
        numactl --interleave=all $JAVA ${JAVA_OPT} $@
    else
        numactl --cpunodebind=$RMQ_NUMA_NODE --membind=$RMQ_NUMA_NODE $JAVA ${JAVA_OPT} $@
    fi
else
    $JAVA ${JAVA_OPT} --add-exports=java.base/jdk.internal.ref=ALL-UNNAMED $@
fi
```

启动broker： mqbroker -n localhost:9876

![image-20210909210414060](assest/image-20210909210414060.png)



日志

![image-20210909210805231](assest/image-20210909210805231.png)



**vim bin/tools.sh**

```sh
#!/bin/sh

# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

#===========================================================================================
# Java Environment Setting
#===========================================================================================
error_exit () 
{
    echo "ERROR: $1 !!"
    exit 1
}

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"

export JAVA_HOME
export JAVA="$JAVA_HOME/bin/java"
export BASE_DIR=$(dirname $0)/..
#export CLASSPATH=.:${BASE_DIR}/conf:${CLASSPATH}
export CLASSPATH=.${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib/*:${BASE_DIR}/conf:${CLASSPATH}
#===========================================================================================
# JVM Configuration
#===========================================================================================
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn256m -XX:PermSize=128m -XX:MaxPermSize=128m"
# JAVA_OPT="${JAVA_OPT} -Djava.ext.dirs=${BASE_DIR}/lib:${JAVA_HOME}/jre/lib/ext" 
JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"

$JAVA ${JAVA_OPT} $@
```



![image-20210909211908712](assest/image-20210909211908712.png)

**先启动namesrv，在启动broker**

```shell
# 1.启动NameServer
mqnamesrv
# 2.查看启动日志
tail -f ~/logs/rocketmqlogs/namesrv.log
```

```shell
# 1.启动Broker
mqbroker -n localhost:9876 
# 2.查看启动日志
tail -f ~/logs/rocketmqlogs/broker.log
```

**先关闭broker，再关闭namesrv**

```shell
#关闭broker
mqshutdown broker
#关闭namesrv
mqshutdown namesrv
```



# 8 RocketMQ环境测试

建议先启动消费者

## 8.1 接收消息

```shell
export NAMESRV_ADDR=localhost:9876 && tools.sh org.apache.rocketmq.example.quickstart.Consumer
```

![image-20210909212559540](assest/image-20210909212559540.png)

## 8.2 生产消息

```shell
# 1.设置环境变量
export NAMESRV_ADDR=localhost:9876 
# 2.使用安装包的Demo发送消息
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
```

![image-20210909213030624](assest/image-20210909213030624.png)

# 9 RocketMQ相关API使用

使用前记得关闭linux的防火墙。



```
SendResult [
	sendStatus=SEND_OK,
    msgId=C0A81F8830601F89AB8331BAA5870000, 
    offsetMsgId=C0A81F6500002A9F0000000000057D64, 
    messageQueue=MessageQueue[topic=tp_demo_01, brokerName=node1, queueId=1], 
    queueOffset=0
    ]
```



# 10 RocketMQ和Spring整合

## 10.1 消息生产者

## 10.2 消息消费者