> 第一部分 Tomcat 系统架构与原理剖析

B/S （浏览器/服务器模式）浏览器是客户端（发送http请求） ---> 服务器端

# 1 浏览器访问服务器的流程

http 请求的处理过程

![image-20220629140840487](assest/image-20220629140840487.png)



![Figure 4 Protocol layers](assest/img004.png)

注意：浏览器访问服务器使用的是 HTTP 协议，HTTP是应用层协议，用于定义数据通信的格式，具体的数据传输使用的是 TCP/IP 协议。

![image-20220629142802249](assest/image-20220629142802249.png)

