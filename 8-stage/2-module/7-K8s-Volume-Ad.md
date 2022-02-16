第七部分 K8s高级篇-volume(存储)

[卷-K8s-官网参考](https://kubernetes.io/zh/docs/concepts/storage/volumes/)

# 1 准备镜像

k8s集群每个node节点需要下载镜像：

```bash
docker pull mariadb:10.5.2
```



# 2 安装mariaDB

## 2.1 部署service

maria/mariadb.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb-deploy
  labels:
    app: mariadb-deploy
spec:
  replicas: 1
  template:
    metadata:
      name: mariadb-deploy
      labels:
        app: mariadb-deploy
    spec:
      containers:
        - name: mariadb-deploy
          image: mariadb:10.5.2
          imagePullPolicy: IfNotPresent
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: admin #这是 mysql root 用户的密码
            - name: TZ
              value: Asia/Shanghai
          args:
            - "--character-set-server=utf8mb4"
            - "--collation-server=utf8mb4_unicode_ci"
          ports:
            - containerPort: 3306
      restartPolicy: Always
  selector:
    matchLabels:
      app: mariadb-deploy
---
apiVersion: v1
kind: Service
metadata:
  name: mariadb-svc
spec:
  selector:
    app: mariadb-deploy
  ports:
    - port: 3306
      targetPort: 3306
      nodePort: 30036
  type: NodePort
```



## 2.2 运行服务

```bash
kubectl apply -f .

kubectl get pod -o wide
```



## 2.3 客户端测试

```bash
IP:192.168.31.61
username:root 
password:admin 
prot: 30036
```



## 2.4 删除 service

```bash
kubectl delete -f mariadb.yml
```



# 3 secret

[secret-K8s-官网参考](https://kubernetes.io/zh/docs/concepts/configuration/secret/)

Secret 解决了密码、token、密钥等敏感数据的配置问题，而不需要把这些敏感数暴露到镜像或者 Pod Spec 中。Secret 可以以 Volume 或者环境变量的方式使用。

Secret 有三种类型：

- Service Account：用来访问 Kubernetes API，由 Kubernetes 自动创建，并且会自动挂载到Pod 的 /run/secrets/kubernetes.io/serviceaccount 目录中。
- Opaque：base64编码格式的Secret，用来存储密码、密钥等
- kubernetes.io/dockerconfigjson：用来存储私有 docker registry 的认证信息

## 3.1 Service Account

Service Account 简称 sa，Service Account 用来访问 Kubernetes API，由 Kubernetes自动创建，并且会自动挂载到 Pod 的 /run/secrets/kubernetes.io/serviceaccount 目录中

```bash
查询命名空间为kube-system的pod信息 
kubectl get pod -n kube-system

进入pod:kube-proxy-48bz4
kubectl exec -it kube-proxy-48bz4 -n kube-system sh 

cd /run/secrets/kubernetes.io/serviceaccount
ls

cat ca.crt
cat namespace
cat token
```



## 3.2 Opaque Secret

Opaque 类型的数据是一个 map 类型，要求 value 是 base64 编码格式。

### 3.2.1 加密解密

使用命令行方式对需要加密的字符串进行加密。例如：mysql数据库的密码。

```bash
对admin字符串进行base64加密:获得admin的加密字符串"YWRtaW4=" 
echo -n "admin" | base64

base64解密：对加密后的字符串进行解密 
echo -n "YWRtaW4=" | base64 -d
```



### 3.2.2 资源文件方式创建

对mariadb数据库密码进行加密

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadbsecret
type: Opaque
data:
  password: YWRtaW4=
  # 
  username: cm9vdA==
```



### 3.2.3 升级mariadb的service

```yaml
          env:
            - name: MYSQL_ROOT_PASSWORD
              #这是 mysql root 用户的密码
              valueFrom:
                secretKeyRef:
                  key: password
                  name: mariadbsecret
            - name: TZ
              value: Asia/Shanghai
```



### 3.2.4 全部资源文件清单

secret/mariadbsecret.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadbsecret
type: Opaque
data:
  password: YWRtaW4=
  # 
  username: cm9vdA==
```

secret/mariadb.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb-deploy
  labels:
    app: mariadb-deploy
spec:
  replicas: 1
  template:
    metadata:
      name: mariadb-deploy
      labels:
        app: mariadb-deploy
    spec:
      containers:
        - name: mariadb-deploy
          image: mariadb:10.5.2
          imagePullPolicy: IfNotPresent
          env:
            - name: MYSQL_ROOT_PASSWORD
              #这是 mysql root 用户的密码
              valueFrom:
                secretKeyRef:
                  key: password
                  name: mariadbsecret
            - name: TZ
              value: Asia/Shanghai
          args:
            - "--character-set-server=utf8mb4"
            - "--collation-server=utf8mb4_unicode_ci"
          ports:
            - containerPort: 3306
      restartPolicy: Always
  selector:
    matchLabels:
      app: mariadb-deploy
---
apiVersion: v1
kind: Service
metadata:
  name: mariadb-svc
spec:
  selector:
    app: mariadb-deploy
  ports:
    - port: 3306
      targetPort: 3306
      nodePort: 30036
  type: NodePort
```



### 3.2.5 运行service

```bash
kubectl apply -f .

kubectl get secret
kubectl get svc
```



### 3.2.6 客户端测试

```bash
IP:192.168.31.61
username:root 
password:admin 
prot: 30036
```



### 3.2.7 删除service、secret

```bash
kubectl delete -f .

kubectl get secret
kubectl get svc
```



## 3.3 安装harbor私服

[参考docker部分的内容](https://github.com/turboYuu/java-advanced-gitbook/blob/master/8-stage/1-module/2-Docker-core-theory.md#53-%E4%BC%81%E4%B8%9A%E7%A7%81%E6%9C%8D)，使用192.168.31.82

### 3.3.1 harbor官网地址

```html
harbor官网地址： 
https://goharbor.io/

github官网地址：
https://github.com/goharbor/harbor
```

### 3.3.2 docker-compose

[参考docker部分的笔记](https://github.com/turboYuu/java-advanced-gitbook/blob/master/8-stage/1-module/2-Docker-core-theory.md#45-docker-compose%E5%AE%89%E8%A3%85)

```bahs
验证docker-compose
docker-compose -v
```

### 3.3.3 安装harbor

```bash
1.解压软件 
cd /data
tar zxf harbor-offline-installer-v1.9.4.tgz

2.进入安装目录 
cd harbor

3.修改配置文件 
vi harbor.yml
  3.1修改私服镜像地址 
  hostname: 192.168.31.82
  3.2修改镜像地址访问端口号 
  port: 5000
  3.3harbor管理员登录系统密码
  harbor_admin_password: Harbor12345
  3.4修改harbor映射卷目录 
  data_volume: /data/harbor
  
4.安装harbor
  4.1执行启动脚本,经过下述3个步骤后，成功安装harbor私服 
  ./install.sh
  4.2准备安装环境：检查docker版本和docker-compose版本
  4.3加载harbor需要的镜像
  4.4准备编译环境
  4.5启动harbor。通过docker-compose方式启动服务
  4.6google浏览器访问harbor私服 
  http://192.168.31.82:5000    
  username: admin
  password: Harbor12345
  
5.关闭harbor服务
docker-compose down [-v]
```

### 3.3.4 新建项目

```bash
在harbor中新建公共项目： 
turbine
```

### 3.3.5 配置私服

```bash
k8s集群master节点配置docker私服：master节点用于上传镜像。其余工作节点暂时不要配置私服地址。

vi /etc/docker/daemon.json
"insecure-registries":["192.168.31.82:5000"]

重启docker服务：
systemctl daemon-reload 
systemctl restart docker
```

### 3.3.6 登录私服

```bash
docker login -u admin -p Harbor12345 192.168.31.82:5000 

退出私服
docker logout 192.168.31.82:5000
```

### 3.3.7 上传mariadb镜像

```bash
docker tag mariadb:10.5.2 192.168.31.82:5000/turbine/mariadb:10.5.2

docker push 192.168.31.82:5000/turbine/mariadb:10.5.2

docker rmi -f 192.168.31.82:5000/turbine/mariadb:10.5.2
```

### 3.3.8 修改mariadb镜像地址

修改 secret/maraiadb.yml 文件，将image地址修改为harbor私服地址

```yaml
image: 192.168.31.82:5000/turbine/mariadb:10.5.2
```

运行服务

```bash
kubectl apply -f .

查看pod信息：发现镜像拉取失败，STATUS显示信息为"ImagePullBackOff" 
kubectl get pods

查看pod详细信息:拉取harbor私服镜像失败。 
kubectl describe pod mariadb-7b6f895b5b-mc5xp

删除服务：
kubectl delete -f .
```



## 3.4 注册私服

使用 kubectl 创建 docker registry 认证的

```bash
语法规则：
kubectl create secret docker-registry myregistrykey --docker-server=REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
例子：
kubectl create secret docker-registry turboharbor --docker-server=192.168.31.82:5000 --docker-username=admin --docker-password=Harbor12345 --docker-email=harbor@turbo.com

k8s集群其余工作节点配置docker私服地址： 
vi /etc/docker/daemon.json
"insecure-registries":["192.168.31.82:5000"]

重启docker服务： 
systemctl daemon-reload
systemctl restart docker
```



## 3.5 secret升级mariadb

将mariadb镜像修改为harbor私服地址。在创建 Pod 的时候，通过 imagesPullSecrets 引用刚创建的 `myregistrykey`

```yaml
    spec:
      imagePullSecrets:
        - name: turboharbor
      containers:
        - name: mariadb-deploy
          image: 192.168.31.82:5000/turbine/mariadb:10.5.2
```

### 3.5.1 全部资源文件清单

mariadbsecret.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadbsecret
type: Opaque
data:
  password: YWRtaW4=
  # 
  username: cm9vdA==
```



mariadb.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb-deploy
  labels:
    app: mariadb-deploy
spec:
  replicas: 1
  template:
    metadata:
      name: mariadb-deploy
      labels:
        app: mariadb-deploy
    spec:
      imagePullSecrets:
        - name: turboharbor
      containers:
        - name: mariadb-deploy
          image: 192.168.31.82:5000/turbine/mariadb:10.5.2
          imagePullPolicy: IfNotPresent
          env:
            - name: MYSQL_ROOT_PASSWORD
              #这是 mysql root 用户的密码
              valueFrom:
                secretKeyRef:
                  key: password
                  name: mariadbsecret
            - name: TZ
              value: Asia/Shanghai
          args:
            - "--character-set-server=utf8mb4"
            - "--collation-server=utf8mb4_unicode_ci"
          ports:
            - containerPort: 3306
      restartPolicy: Always
  selector:
    matchLabels:
      app: mariadb-deploy
---
apiVersion: v1
kind: Service
metadata:
  name: mariadb-svc
spec:
  selector:
    app: mariadb-deploy
  ports:
    - port: 3306
      targetPort: 3306
      nodePort: 30036
  type: NodePort
```



## 3.6 客户端测试

```bash
IP:192.168.31.61
username:root 
password:admin 
prot: 30036
```



# 4 configmap

[ConfigMap-K8s-官网参考](https://kubernetes.io/zh/docs/concepts/configuration/configmap/)

ConfigMap 顾名思义，用于保存配置数据的键值对，可以用来保存单个属性，也可以保存配置文件。

ConfigMaps 允许你将配置构件 与 映射内容解耦，以保持容器化应用程序的可移植性。configmap 可以从文件、目录 或者 key-value 字符串创建等 创建 configmap。也可以通过 kubectl create -f 从描述文件创建。可以使用 kubectl create 创建命令。创建 ConfigMap 的方式有 4 种：

1. 直接在命令行中指定 configmap 参数创建，即 `--from-literal`。
2. 指定文件创建，即将一个配置文件创建为一个 ConfigMap，--from-file=<文件>
3. 指定目录创建，即将一个目录下的所有配置文件创建为一个 ConfigMap，--from-file=<目录>
4. 先写好标准的configmap的yaml文件，然后kubectl create -f 创建

## 4.1 命令行方式

从 key-value 字符串创建，官方翻译是从字面值中创建 ConfigMap。

### 4.1.1 语法规则

```bash
kubectl create configmap map的名字 --from-literal=key=value 

如果有多个key，value。可以继续在后边写
kubectl create configmap map的名字
  --from-literal=key1=value1
  --from-literal=key2=value2
  --from-literal=key3=value4
```



### 4.1.2 案例

```bash
创建configmap
kubectl create configmap helloconfigmap --from-literal=turbo.hello=world

查看configmap
kubectl get configmap helloconfigmap -o go-template='{{.data}}'

删除configmap
kubectl delete configmaps helloconfigmap
```

```html
在使用kubectl get获取资源信息的时候,可以通过-o(--output简写形式)指定信息输出的格式,
如果指定的是yaml或者json输出的是资源的完整信息,

实际工作中,输出内容过少则得不到我们想要的信息,输出内容过于详细又不利于快速定位的我们想要找到的内容,
其实-o输出格式可以指定为go-template 然后指定一个template,这样我们就可以通过go-template获取我们想要的内容.
go-template与 kubernetes无关,它是go语言内置的一种模板引擎.这里不对go-template做过多解释,
仅介绍在 kubernetes中获取资源常用的语法,想要获取更多内容,大家可以参考相关资料获取帮助。
大家记住是固定语法即可。
```



## 4.2 配置文件方式

### 4.2.1 语法规则

```bash
语法规则如下：当 --from-file指向一文件，key的名称是文件名称，value的值是这个文件的内容。
kubectl create configmap cumulx-test --from-file=xxxx
```



### 4.2.2 案例

创建一个配置文件：jdbc.properties

```properties
vi jdbc.properties

jdbc.driverclass=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/test
jdbc.username=root
jdbc.password=admin
```



```bash
创建configmap换用其他方式查看信息
kubectl create configmap myjdbcmap --from-file=jdbc.properties 

查看configmap详细信息
kubectl describe configmaps myjdbcmap
```



## 4.3 目录方式

### 4.3.1 语法规则

```bash
语法规则如下：当 --from-file指向一个目录，每个目录中的文件直接用于填充ConfigMap中的key,key的名称是文件名称，value的值是这个文件的内容。下面的命令读取/data目录下的所有文件
kubectl create configmap cumulx-test --from-file=/data/
```



### 4.3.2 案例

```bash
kubectl create configmap myjdbcconfigmap --from-file=/data/jdbc.properties 

查看configmap详细信息
kubectl describe configmaps myjdbcmap
```



## 4.4 资源文件方式

### 4.4.1 yml文件

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myjdbcconfigmap
data:
  jdbc-driverclass: com.mysql.jdbc.Driver
  jdbc-url: jdbc:mysql://localhost:3306/test
  jdbc-username: root
  jdbc-password: admin
```



### 4.4.2 运行yml文件

```bash
kubectl apply -f mariadb-configmap.yml 

查看configmap详细信息
kubectl describe configmaps mariadbconfigmap
```



# 5 configmap 升级 mariadb

## 5.1 获得 my.cnf 文件

```bash
运行测试镜像：
docker run --name some-mariadb -e MYSQL_ROOT_PASSWORD=my-secret-pw -d 192.168.31.82:5000/turbine/mariadb:10.5.2

将my.cnf文件从容器中复制到/data目录
docker cp some-mariadb:/etc/mysql/my.cnf /data 

停止容器
docker stop some-mariadb 

删除容器
docker rm some-mariadb 

删除镜像
docker rmi -f 192.168.31.82:5000/turbine/mariadb:10.5.2

更改my.cnf文件中的端口号为：3307
```



## 5.2 生成 configmap 文件

data/mariadbconfigmap.yml

```bash
将my.cnf文件中mariadb数据库启动端口号修改为3307，
将my.cnf文件上传master节点，首先通过命令行方式生成configmap资源。
再通过命令行方式将configmap反向生成yaml文件。
将生成文件copy回 idea开发工具。修改yaml文件备用。

kubectl create configmap mysqlini --from-file=my.cnf 
kubectl get configmap mysqlini -o yaml > mariadbconfigmap.yml
```



## 5.3 修改 mariadb.yml

```yaml
          volumeMounts:
            - mountPath: /etc/mysql/mariadb.conf.d/ # 容器内的挂载目录
              name: turbo-mariadb # 随便给一个名字,这个名字必须与volumes.name 一致
          ports:
            - containerPort: 3307
      restartPolicy: Always
      volumes:
        - name: turbo-mariadb
          configMap:
            name: mariadbconfigmap
```



## 5.4 全部资源文件清单

### 5.4.1 configmap/mariadbsecret.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadbsecret
type: Opaque
data:
  password: YWRtaW4=
  # mariadb的用户名root加密，用于演示，无实际效果
  username: cm9vdA==
```

### 5.4.2 configmap/mariadb.yml

**需要再次调整，加入私服镜像信息**

注意修改 pod 的端口号为 3307，service的targetPort端口号为 3307

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb-deploy
  labels:
    app: mariadb-deploy
spec:
  replicas: 1
  template:
    metadata:
      name: mariadb-deploy
      labels:
        app: mariadb-deploy
    spec:
      imagePullSecrets:
        - name: turboharbor
      containers:
        - name: mariadb-deploy
          image: 192.168.31.82:5000/turbine/mariadb:10.5.2
          imagePullPolicy: IfNotPresent
          env:
            - name: MYSQL_ROOT_PASSWORD
              #这是 mysql root 用户的密码
              valueFrom:
                secretKeyRef:
                  key: password
                  name: mariadbsecret
            - name: TZ
              value: Asia/Shanghai
          args:
            - "--character-set-server=utf8mb4"
            - "--collation-server=utf8mb4_unicode_ci"
          volumeMounts:
            - mountPath: /etc/mysql/mariadb.conf.d/ # 容器内的挂载目录
              name: turbo-mariadb # 随便给一个名字,这个名字必须与volumes.name 一致
          ports:
            - containerPort: 3307
      restartPolicy: Always
      volumes:
        - name: turbo-mariadb
          configMap:
            name: mariadbconfigmap
  selector:
    matchLabels:
      app: mariadb-deploy
---
apiVersion: v1
kind: Service
metadata:
  name: mariadb-svc
spec:
  selector:
    app: mariadb-deploy
  ports:
    - port: 3307
      targetPort: 3307
      nodePort: 30036
  type: NodePort
```

### 5.4.3 configmap/mariadbconfigmap.yml

```yaml
apiVersion: v1
data:
  my.cnf: "# MariaDB database server configuration file.\n#\n# You can copy this file
    to one of:\n# - \"/etc/mysql/my.cnf\" to set global options,\n# - \"~/.my.cnf\"
    to set user-specific options.\n# \n# One can use all long options that the program
    supports.\n# Run program with --help to get a list of available options and with\n#
    --print-defaults to see which it would actually understand and use.\n#\n# For
    explanations see\n# http://dev.mysql.com/doc/mysql/en/server-system-variables.html\n\n#
    This will be passed to all mysql clients\n# It has been reported that passwords
    should be enclosed with ticks/quotes\n# escpecially if they contain \"#\" chars...\n#
    Remember to edit /etc/mysql/debian.cnf when changing the socket location.\n[client]\nport\t\t=
    3307\nsocket\t\t= /var/run/mysqld/mysqld.sock\n\n# Here is entries for some specific
    programs\n# The following values assume you have at least 32M ram\n\n# This was
    formally known as [safe_mysqld]. Both versions are currently parsed.\n[mysqld_safe]\nsocket\t\t=
    /var/run/mysqld/mysqld.sock\nnice\t\t= 0\n\n[mysqld]\n#\n# * Basic Settings\n#\n#user\t\t=
    mysql\npid-file\t= /var/run/mysqld/mysqld.pid\nsocket\t\t= /var/run/mysqld/mysqld.sock\nport\t\t=
    3307\nbasedir\t\t= /usr\ndatadir\t\t= /var/lib/mysql\ntmpdir\t\t= /tmp\nlc_messages_dir\t=
    /usr/share/mysql\nlc_messages\t= en_US\nskip-external-locking\n#\n# Instead of
    skip-networking the default is now to listen only on\n# localhost which is more
    compatible and is not less secure.\n#bind-address\t\t= 127.0.0.1\n#\n# * Fine
    Tuning\n#\nmax_connections\t\t= 100\nconnect_timeout\t\t= 5\nwait_timeout\t\t=
    600\nmax_allowed_packet\t= 16M\nthread_cache_size       = 128\nsort_buffer_size\t=
    4M\nbulk_insert_buffer_size\t= 16M\ntmp_table_size\t\t= 32M\nmax_heap_table_size\t=
    32M\n#\n# * MyISAM\n#\n# This replaces the startup script and checks MyISAM tables
    if needed\n# the first time they are touched. On error, make copy and try a repair.\nmyisam_recover_options
    = BACKUP\nkey_buffer_size\t\t= 128M\n#open-files-limit\t= 2000\ntable_open_cache\t=
    400\nmyisam_sort_buffer_size\t= 512M\nconcurrent_insert\t= 2\nread_buffer_size\t=
    2M\nread_rnd_buffer_size\t= 1M\n#\n# * Query Cache Configuration\n#\n# Cache only
    tiny result sets, so we can fit more in the query cache.\nquery_cache_limit\t\t=
    128K\nquery_cache_size\t\t= 64M\n# for more write intensive setups, set to DEMAND
    or OFF\n#query_cache_type\t\t= DEMAND\n#\n# * Logging and Replication\n#\n# Both
    location gets rotated by the cronjob.\n# Be aware that this log type is a performance
    killer.\n# As of 5.1 you can enable the log at runtime!\n#general_log_file        =
    /var/log/mysql/mysql.log\n#general_log             = 1\n#\n# Error logging goes
    to syslog due to /etc/mysql/conf.d/mysqld_safe_syslog.cnf.\n#\n# we do want to
    know about network errors and such\n#log_warnings\t\t= 2\n#\n# Enable the slow
    query log to see queries with especially long duration\n#slow_query_log[={0|1}]\nslow_query_log_file\t=
    /var/log/mysql/mariadb-slow.log\nlong_query_time = 10\n#log_slow_rate_limit\t=
    1000\n#log_slow_verbosity\t= query_plan\n\n#log-queries-not-using-indexes\n#log_slow_admin_statements\n#\n#
    The following can be used as easy to replay backup logs or for replication.\n#
    note: if you are setting up a replication slave, see README.Debian about\n#       other
    settings you may need to change.\n#server-id\t\t= 1\n#report_host\t\t= master1\n#auto_increment_increment
    = 2\n#auto_increment_offset\t= 1\n#log_bin\t\t\t= /var/log/mysql/mariadb-bin\n#log_bin_index\t\t=
    /var/log/mysql/mariadb-bin.index\n# not fab for performance, but safer\n#sync_binlog\t\t=
    1\nexpire_logs_days\t= 10\nmax_binlog_size         = 100M\n# slaves\n#relay_log\t\t=
    /var/log/mysql/relay-bin\n#relay_log_index\t= /var/log/mysql/relay-bin.index\n#relay_log_info_file\t=
    /var/log/mysql/relay-bin.info\n#log_slave_updates\n#read_only\n#\n# If applications
    support it, this stricter sql_mode prevents some\n# mistakes like inserting invalid
    dates etc.\n#sql_mode\t\t= NO_ENGINE_SUBSTITUTION,TRADITIONAL\n#\n# * InnoDB\n#\n#
    InnoDB is enabled by default with a 10MB datafile in /var/lib/mysql/.\n# Read
    the manual for more InnoDB related options. There are many!\ndefault_storage_engine\t=
    InnoDB\ninnodb_buffer_pool_size\t= 256M\ninnodb_log_buffer_size\t= 8M\ninnodb_file_per_table\t=
    1\ninnodb_open_files\t= 400\ninnodb_io_capacity\t= 400\ninnodb_flush_method\t=
    O_DIRECT\n#\n# * Security Features\n#\n# Read the manual, too, if you want chroot!\n#
    chroot = /var/lib/mysql/\n#\n# For generating SSL certificates I recommend the
    OpenSSL GUI \"tinyca\".\n#\n# ssl-ca=/etc/mysql/cacert.pem\n# ssl-cert=/etc/mysql/server-cert.pem\n#
    ssl-key=/etc/mysql/server-key.pem\n\n#\n# * Galera-related settings\n#\n[galera]\n#
    Mandatory settings\n#wsrep_on=ON\n#wsrep_provider=\n#wsrep_cluster_address=\n#binlog_format=row\n#default_storage_engine=InnoDB\n#innodb_autoinc_lock_mode=2\n#\n#
    Allow server to accept connections on all interfaces.\n#\n#bind-address=0.0.0.0\n#\n#
    Optional setting\n#wsrep_slave_threads=1\n#innodb_flush_log_at_trx_commit=0\n\n[mysqldump]\nquick\nquote-names\nmax_allowed_packet\t=
    16M\n\n[mysql]\n#no-auto-rehash\t# faster start of mysql but no tab completion\n\n[isamchk]\nkey_buffer\t\t=
    16M\n\n#\n# * IMPORTANT: Additional settings that can override those from this
    file!\n#   The files must end with '.cnf', otherwise they'll be ignored.\n#\n!include
    /etc/mysql/mariadb.cnf\n!includedir /etc/mysql/conf.d/\n"
kind: ConfigMap
metadata:
  name: mariadbconfigmap
```



## 5.5 运行服务

```bash
部署服务
kubectl apply -f .

查看相关信息
kubectl get configmaps 
kubectl get pods
```



## 5.6 查看 mariadb容器日志

```bash
查看mariadb容器日志：观察mariadb启动port是否为3307 
kubectl logs -f mariadb-5d99587d4b-xwthc
```



## 5.7 客户端测试

```bash
IP:192.168.31.61 
username:root 
password:admin 
prot: 30036
```



# 6 label操作

前面学习了如何给 pod 打标签及[修改pod的标签值](https://github.com/turboYuu/java-advanced-gitbook/blob/master/8-stage/2-module/5-K8s-Controller-Manager.md#53-%E8%BF%90%E8%A1%8Creplicaset)。在某些特殊情况下，需要将某些服务固定在一台宿主机上，k8s可以使用 label 给node节点打赏标签来满足这种需求。



## 6.1 添加 label操作

```bash
kubectl label nodes <node-name> <label-key>=<label-value> 

为k8s-node01节点打标签
kubectl label nodes k8s-node01 mariadb=mariadb 

查看node节点label值
kubectl get nodes --show-labels
```



## 6.2 修改Label的值

**语法**：**需要加上`--overwrite`参数**

```bash
kubectl label nodes <node-name> <label-key>=<label-value> --overwrite 

修改k8s-node01节点label值
kubectl label nodes k8s-node01 mariadb=mariadb10.5 --overwrite 

查看node节点label值
kubectl get nodes --show-labels
```



## 6.3 删除label语法

```bash
注意事项：label的可以后边要增加"-"
kubectl label nodes <node-name> <label-key>- 

删除k8s-node01节点mariadb的label
kubectl label nodes k8s-node01 mariadb- 

查看node节点label值
kubectl get nodes --show-labels
```



## 6.4 mariaDB 部署

通过指定 node 节点 lable，将mariaDB部署到指定节点。方便演示 volume的各种方式。

### 6.4.1 指定node

```bash
kubectl label nodes k8s-node01 mariadb=mariadb 

查看node节点label值
kubectl get nodes --show-labels
```



### 6.4.2 service部署

```bash
在spec.template.spec属性下增加nodeSelector属性。 
spec:
  nodeSelector: #根据label设置，配置节点选择器       
    mariadb: mariadb  #语法规则: key: value     
  containers:
```



## 6.5 全部资源文件清单

labels/mariadbsecret.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadbsecret
type: Opaque
data:
  password: YWRtaW4=
  # mariadb的用户名root加密，用于演示，无实际效果
  username: cm9vdA==
```



labels/mariadb.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb-deploy
  labels:
    app: mariadb-deploy
spec:
  replicas: 1
  template:
    metadata:
      name: mariadb-deploy
      labels:
        app: mariadb-deploy
    spec:
      nodeSelector:
        mariadb: mariadb
      imagePullSecrets:
        - name: turboharbor
      containers:
        - name: mariadb-deploy
          image: 192.168.31.82:5000/turbine/mariadb:10.5.2
          imagePullPolicy: IfNotPresent
          env:
            - name: MYSQL_ROOT_PASSWORD
              #这是 mysql root 用户的密码
              valueFrom:
                secretKeyRef:
                  key: password
                  name: mariadbsecret
            - name: TZ
              value: Asia/Shanghai
          args:
            - "--character-set-server=utf8mb4"
            - "--collation-server=utf8mb4_unicode_ci"
          volumeMounts:
            - mountPath: /etc/mysql/mariadb.conf.d/ # 容器内的挂载目录
              name: turbo-mariadb # 随便给一个名字,这个名字必须与volumes.name 一致
          ports:
            - containerPort: 3307
      restartPolicy: Always
      volumes:
        - name: turbo-mariadb
          configMap:
            name: mariadbconfigmap
  selector:
    matchLabels:
      app: mariadb-deploy
---
apiVersion: v1
kind: Service
metadata:
  name: mariadb-svc
spec:
  selector:
    app: mariadb-deploy
  ports:
    - port: 3307
      targetPort: 3307
      nodePort: 30036
  type: NodePort
```



labels/mariadbconfigmap.yml

```yaml
apiVersion: v1
data:
  my.cnf: "省略中间数据部分“
kind: ConfigMap
metadata:
  name: mariadbconfigmap
```



## 6.6 客户端测试

```
IP:192.168.31.62
username:root 
password:admin 
prot: 30036
```



# 7 volume

## 7.1 hostPath（卷的入门）

hostPath类型的存储卷是指将工作节点上某文件系统的目录或文件挂载于 Pod 中的一种存储卷。把宿主机上的目录挂载到容器，但是在每个节点上都要有，因为不确定容器会分配到哪个节点。也是把存储从宿主机挂载到 K8s 集群上，但它有许多限制，例如只支持单节点（Node），而且只支持 ”ReadWriteOnce“ 模式。

### 7.1.1 指定node节点

```bash
kubectl label nodes k8s-node01 mariadb=mariadb 

查看node节点label值
kubectl get nodes --show-labels
```



### 7.1.2 挂载卷

```yaml
语法：
1. volumeMounts为containers下级key，containers.volumeMounts。volumes与 containers平级。
2. containers.volumeMounts.name与volumes.name值一致。
3. containers.volumeMounts.mountPath是容器内目录
4. volumes.hostPath.path是宿主机挂载目录
5. volumes.hostPath.type值必须为"Directory"

containers 
  volumeMounts:
    - mountPath: /var/lib/mysql      
    name: mariadb-volume
    ....... 
volumes:
  - name: mariadb-volume     
  hostPath:
    path: /data/mariadb 
    type: Directory
    
    
例如： 
  volumeMounts:
    - mountPath: /var/lib/mysql      
    name: mariadb-volume 
restartPolicy: Always
volumes:
  - name: mariadb-volume    
  hostPath:
    path: /data/mariadb      
    type: Directory
```



### 7.1.3 全部资源文件清单

hostPath/mariadbsecret.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadbsecret
type: Opaque
data:
  password: YWRtaW4=
  # mariadb的用户名root加密，用于演示，无实际效果
  username: cm9vdA==
```



hostPath/mariadb.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb-deploy
  labels:
    app: mariadb-deploy
spec:
  replicas: 1
  template:
    metadata:
      name: mariadb-deploy
      labels:
        app: mariadb-deploy
    spec:
      nodeSelector:
        mariadb: mariadb
      imagePullSecrets:
        - name: turboharbor
      containers:
        - name: mariadb-deploy
          image: 192.168.31.82:5000/turbine/mariadb:10.5.2
          imagePullPolicy: IfNotPresent
          env:
            - name: MYSQL_ROOT_PASSWORD
              #这是 mysql root 用户的密码
              valueFrom:
                secretKeyRef:
                  key: password
                  name: mariadbsecret
            - name: TZ
              value: Asia/Shanghai
          args:
            - "--character-set-server=utf8mb4"
            - "--collation-server=utf8mb4_unicode_ci"
          volumeMounts:
            - mountPath: /etc/mysql/mariadb.conf.d/ # 容器内的挂载目录
              name: turbo-mariadb # 随便给一个名字,这个名字必须与volumes.name 一致
            - mountPath: /var/lib/mysql # 容器内的挂载目录
              name: turbo-volume-mariadb
          ports:
            - containerPort: 3307
      restartPolicy: Always
      volumes:
        - name: turbo-mariadb
          configMap:
            name: mariadbconfigmap
        - name: turbo-volume-mariadb
          hostPath:
            path: /data/mariadb
            type: Directory
  selector:
    matchLabels:
      app: mariadb-deploy
---
apiVersion: v1
kind: Service
metadata:
  name: mariadb-svc
spec:
  selector:
    app: mariadb-deploy
  ports:
    - port: 3307
      targetPort: 3307
      nodePort: 30036
  type: NodePort
```



hostPath/mariadbconfigmap.yml

```yaml
apiVersion: v1
data:
  my.cnf: "省略..."
kind: ConfigMap
metadata:
  name: mariadbconfigmap
```

### 7.1.4 客户端测试

```bash
IP:192.168.31.63
username:root 
password:admin 
prot: 30036
```



## 7.2 emptyDir

emptyDir存储卷是Pod生命周期中的一个临时目录你，在Pod对象被移除时会被一并删除，用得少。例如：同一 pod内的多个容器间文件共享，或者作为容器数据的临时存储目录用于数据缓存系统等。可以自己查资料学习。

# 8 PV&&PVC

# 9 NFS存储卷





















