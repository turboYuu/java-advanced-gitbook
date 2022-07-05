> 第六部分 Tomcat 对 Https 的支持及 Tomcat 性能优化策略

# 1 Tomcat 对 HTTPS 的支持

Https 是用来加强数据传输安全的

## 1.1 HTTPS 简介

![image-20220705175212806](assest/image-20220705175212806.png)

**HTTPS 和 HTTP 的主要区别**

- HTTPS 协议使用时需要到电子商务认证授权机构（CA）申请 SSL  证书
- HTTP 默认使用 8080 端口，HTTPS 默认使用 8443 端口
- HTTPS 则是具有 SSL 加密的安全性传输协议，对数据的传输进行加密，效果上相当于 HTTP 的升级版
- HTTP 的链接是无状态的，不安全；HTTPS 协议是由 SSL+HTTP 协议构建的可进行加密传输、身份认证的网络协议，比 HTTP 协议安全

**HTTP工作原理**

说明：Https 在传输数据之前需要客户端与服务器之间进行一次握手，在握手过程中将确定双方加密传输的密码信息



![image-20220705182239806](assest/image-20220705182239806.png)

## 1.2 Tomcat 对 HTTPS 的支持

1. 使用 JDK 中的 keytool 工具生成免费的密钥库文件（证书）

   ```bash
   keytool -genkey -alias turbo -keyalg RSA -keystore turbo.keystore
   ```

2. 配置 conf/server.xml

   ```xml
   
   ```

3. 使用 https 协议访问 8443 端口（）

   

# 2 Tomcat 性能优化策略

## 2.1 虚拟机运行优化（参数调整）

## 2.2 tomcat 配置优化













