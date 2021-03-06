> Nginx

# 1 Nginx 基础回顾

![image-20220706183958183](assest/image-20220706183958183.png)

- Nginx 到底是什么？

  Nginx 是一个高性能的 HTTP 和 反向代理 web 服务器，核心特点是占有内存少，并发能力强。

- Nginx 又能做什么事情（应用场景）

  - Http 服务器（Web 服务器）

    性能非常高，非常注重效率，能够经受高负载的考验

    支持 50,000 个并发连接数，不仅如此，CPU 和内存的占用也非常低，10,000 个没有活动的连接才占用 2.5 M的内存。

  - 反向代理服务器

    - 正向代理

      在浏览器中配置代理服务器的相关信息，通过代理服务器访问目标网站，代理服务器收到目标网站的响应之后，会把响应信息返回给我们自己的浏览器客户端。

    - 反向代理

      浏览器客户端发送请求到反向代理服务器（比如 Nginx），由反向代理服务器选择原始服务器提供服务获取结果响应，最终再返回给客户端浏览器。

  - 负载均衡服务器

    负载均衡，当一个请求到来的时候（结合上图），Nginx 反向代理服务器根据请求去找到一个原始服务器来处理当前请求，那么这叫反向代理。那么如果目标服务器有多台，找哪一个目标服务器来处理当前请求呢，这样一个寻找确定的过程叫做负载均衡。

    负载均衡就是为了解决高负载的问题。

  - 动静分离



## 1.1 Nginx的安装

1. 上传nginx安装包到linux服务器，安装包（.tar）文件下载地址：https://nginx.org/en/download.html，使用 1.17.8 版本

2. 安装 Nginx 依赖，pcre、openssl、gcc、zlib （推荐使用yum 源自动安装）

   ```bash
   yum -y install gcc zlib zlib-devel pcre-devel openssl openssl-devel
   ```

3. 加压 nginx 安装包

   ```bash
   tar -xvf nginx-1.17.8.tar
   ```

4. 进入解压之后的目录 nginx-1.17.8

   ```bash
   cd nginx-1.17.8
   ```

5. 命令执行 

   ```bash
   ./configure
   make
   make install
   ```

   之后会在 /usr/local/下会产生一个 nginx 目录

   ![image-20220706192007109](assest/image-20220706192007109.png)

6. 进入 sbin 目录中，执行启动命令 

   ```bash
   cd nginx/sbin
   ./nginx
   ```

   然后访问服务器 80 端口（nginx 默认监听 80 端口）

   ![image-20220706191922795](assest/image-20220706191922795.png)

## 1.2 Nginx 主要命令

- `./nginx` 启动 nginx
- `./nginx -s stop` 终止nginx（也可以 kill -9 nginx进程号）
- `./nginx -s reload`  重新加载 nginx.conf 配置文件

# 2 Nginx 核心配置文件解读

Nginx 的核心配置文件 conf/nginx.conf 中包含三块内容：全局快、events 块、http 块

- 全局块

  从配置文件开始到 events 块之间的内容，此处的配置影响 nginx 服务器整体的运行，比如 worker 进程的数量、错误日志的位置等。

  ![image-20220707092749993](assest/image-20220707092749993.png)

- events 块

  events 块主要影响 nginx 服务器与用户的网络连接，比如 worker_connections 1024 ，标识每个worker process 支持的最大连接数为 1024。

  ![image-20220707094707593](assest/image-20220707094707593.png)

- http块

  http块 是配置最频繁的部分，虚拟主机的配置，监听端口的配置，请求转发、反向代理、负载均衡等。

  ![image-20220707095506426](assest/image-20220707095506426.png)

  ![image-20220707095611629](assest/image-20220707095611629.png)

  ![image-20220707095650457](assest/image-20220707095650457.png)

# 3 Nginx 应用场景之反向代理

需求：

![image-20220707101126970](assest/image-20220707101126970.png)

![image-20220707101155495](assest/image-20220707101155495.png)

## 3.1 需求一

1. 部署 tomcat，保持默认监听 8080 端口

2. 修改 nginx 配置，并重新加载

   修改 nginx 中 http 块的配置

   ![image-20220707102352490](assest/image-20220707102352490.png)

   重新加载 nginx 配置 `./nginx -s reload`

3. 测试，访问 http://192.168.3.135:9003，

   ![image-20220707102538150](assest/image-20220707102538150.png)

## 3.2 需求二

1. 在部署一台 tomcat，保持默认监听 8081 端口

2. 修改 nginx 配置，并重新加载

   ![image-20220707103053340](assest/image-20220707103053340.png)

3. 测试

   访问：http://192.168.3.135:9003/abc

   ![image-20220707104333262](assest/image-20220707104333262.png)

   访问：http://192.168.3.135:9003/def

   ![image-20220707104406517](assest/image-20220707104406517.png)

   



这里主要是 **多localhost的使用，这里的nginx中的 http/server/location 就好比 tomcat 中的 Server/Service/Engine/Host/Context**

location 语法如下：

```bash
location [=|^~|~*|~] /uri/ {...}
```

在 nginx 配置文件中，location 主要有这几种形式：

1. 正则匹配 `location ~/turbo{}`
2. 不区分大小写的正则匹配 `location ~*/turbo {}`
3. 匹配路径的前缀 `location ^~/turbo {}`
4. 精准匹配 `location =/turbo {}`
5. 普通路径前缀匹配 `location /turbo {}`

优先级：4>3>2>1>5

# 4 Nginx 应用场景之负载均衡

![image-20220707105818670](assest/image-20220707105818670.png)

Nginx 负载均衡策略

## 4.1 轮询

默认策略，每个请求按时间顺序逐一分配到不同的服务器，**如果某一个服务器下线，能自动剔除**

![image-20220707111806956](assest/image-20220707111806956.png)

## 4.2 weight

weight 代表权重，默认每一个负载的服务器都为1，去找你红越高那么被分配的请求越多（用于服务器性能不均衡的场景）

![image-20220707112102643](assest/image-20220707112102643.png)

## 4.3 ip_hash

每个请求按照 ip 的 hash 结果分配，每一个客户端的请求会固定分配到同一个目标服务器处理，可以解决 session 问题。

![image-20220707112522147](assest/image-20220707112522147.png)

# 5 Nginx 应用场景之动静分离

动静分离是将动态资源和静态资源的请求处理分配到不同的服务器上，比较经典的组合就是 Nginx + Tomcat 架构（Nginx 处理静态资源请求，Tomcat处理动态资源请求），那么其实之前的讲解中，Nginx反向代理目标服务器 tomcat，看到目标服务器 ROOT 项目的 index.jsp，这本身就是tomcat 在处理动态资源请求了。

所以，只需要配置静态资源访问即可。

![image-20220707132127239](assest/image-20220707132127239.png)

nginx 配置：

![image-20220707125734769](assest/image-20220707125734769.png)

![image-20220707125834919](assest/image-20220707125834919.png)

![image-20220707131655658](assest/image-20220707131655658.png)



或者

![image-20220707130551445](assest/image-20220707130551445.png)

![image-20220707130853101](assest/image-20220707130853101.png)

![image-20220707131940942](assest/image-20220707131940942.png)

![image-20220707131813108](assest/image-20220707131813108.png)

# 6 Nginx 底层进程机制剖析

Nginx启动后，以 daemon 多进程方式 在后台运行，包括一个Master进程和多个 Worker 进程，Master 进程是领导；Worker 进程是干活的小弟。

![image-20220707134432708](assest/image-20220707134432708.png)

- master 进程

  主要是管理 worker 进程，比如：

  - 接收外界信号向各 worker 进程发送信号（`./nginx -s reload`）
  - 监控 worker 进程的运行状态，当 worker 进程异常退出后，master 进程会自动重新启动新的 worker 进程 等。

- worker 进程

  worker 进程具体处理网络请求。多个 worker 进程之间是对等的，它们同等竞争来自客户端的请求，**各进程相互之间是独立的**。一个请求只可能在一个 worker 进程中处理，一个 worker 进程不可能处理其他进程的请求。worker 进程的个数是可以设置的，一般设置与机器 cpu 核数一致。

**Nginx 进程模式示意图如下**

![image-20220707141355864](assest/image-20220707141355864.png)

> 在计算 nginx 最大并发时候：(worker_processes * worker_connections)/4

**以  `./nginx -s reload` 来说明 nginx 信号处理部分**

1. master 进程对配置文件进行语法检查
2. 尝试配置（比如修改了监听端口，那就尝试分配新的监听端口）
3. 尝试成功，则使用新的配置，新建 worker 进程
4. 新建成功，给旧的 worker 进程发送关闭消息
5. 旧的 worker 进程收到信号后会继续服务，直到把当前进程接收到的请求处理完毕后关闭

所以reload 之后 worker 进程 pid 是发生了变化

![image-20220707142159504](assest/image-20220707142159504.png)



**worker 进程处理请求部分说明**

例如，监听 9003 端口，一个请求到来时，如果有多个 worker 进程，那么每个 worker 进程都有可能处理这个链接。

- master 进程创建之后，会建立好需要监听的 socket，然后从 master 进程在 fork 出多个 worker 进程。所以，所有worker 进程的监听描述符 listenfd 在新连接到来时都变得可读。
- nginx 使用互斥锁来保证只有一个 worker 进程能够处理请求，拿到互斥锁的那个进程注册 listenfd 读事件，在读事件里调用 accept 接收该连接，然后 解析、处理、返回客户端



**nginx 多进程模型好处**

- 每个 worker 进程都是独立的，不需要加锁，节省开销
- 每个 worker 进程都是独立的，互不影响，一个异常结束，其他的仍然可以提供服务
- 多进程模型为 reload 热部署机制提供了支撑

