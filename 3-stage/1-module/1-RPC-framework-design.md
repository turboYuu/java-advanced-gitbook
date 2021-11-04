第一部分 RPC框架设计

# 1 Socket回顾与I/O模型

## 1.1 Socket网络编程回顾

### 1.1.1 Socket概述

Socket，套接字就是两台主机之间逻辑连接的端点。TCP/IP协议是**传输层**协议，主要解决数据如何在网络中传输；而HTTP是**应用层**协议，主要解决如何包装数据。Socket是通信的基石，是支持TCP/IP协议的网络通信的基本操作单元。它是网络通信过程中端点的抽象表示，包含进行网络通信必须的五种信息：连接使用的协议、本地主机的IP地址、本地进程的协议端口、远程主机的IP地址、远程进程的协议端口。

![osi](assest/20201112090821783.png)

### 1.1.2 Socket整体流程

Socket编程主要涉及客户端和服务端两个方面，首先是在服务器端创建一个服务器套接字（ServerSocket），并把它附加到一个端口上，服务器从这个端口监听连接。端口号的范围是0~65536，但是0~1024是为特权服务保留的端口号，可以选择任意一个当前没有被其他进程使用的端口。

客户端请求与服务器进行连接的时候，根据服务器的域名或者IP地址，加上端口号，打开一个套接字。当服务器接受连接后，服务器和客户端之间的通信就像输入输出流一样进行操作。

![image-20211104150354554](assest/image-20211104150354554.png)



### 1.1.3 代码实现

https://gitee.com/turboYuu/rpc-3-1/tree/master/lab/socket

1. 服务端代码

   ```java
   package com.turbo.server;
   
   import java.io.IOException;
   import java.io.InputStream;
   import java.io.OutputStream;
   import java.net.ServerSocket;
   import java.net.Socket;
   import java.util.concurrent.ExecutorService;
   import java.util.concurrent.Executors;
   
   public class ServerDemo {
   
       public static void main(String[] args) throws Exception {
           //1.创建一个线程池,如果有客户端连接就创建一个线程, 与之通信
           ExecutorService executorService = Executors.newCachedThreadPool();
           //2.创建 ServerSocket 对象
           ServerSocket serverSocket = new ServerSocket(9999);
           System.out.println("服务器已启动");
           while (true) {
               //3.监听客户端
               final Socket socket = serverSocket.accept();
               System.out.println("有客户端连接");
               //4.开启新的线程处理
               executorService.execute(new Runnable() {
                   @Override
                   public void run() {
                       handle(socket);
                   }
               });
           }
       }
   
       public static void handle(Socket socket) {
           try {
               System.out.println("线程ID:" + Thread.currentThread().getId()
                       + "   线程名称:" + Thread.currentThread().getName());
               //从连接中取出输入流来接收消息
               InputStream is = socket.getInputStream();
               byte[] b = new byte[1024];
               int read = is.read(b);
               System.out.println("客户端:" + new String(b, 0, read));
               //连接中取出输出流并回话
               OutputStream os = socket.getOutputStream();
               os.write("没钱".getBytes());
           } catch (Exception e) {
               e.printStackTrace();
           } finally {
               try {
                   //关闭连接
                   socket.close();
               } catch (IOException e) {
                   e.printStackTrace();
               }
           }
       }
   }
   ```

2. 客户端代码

   ```java
   package com.turbo.client;
   
   import java.io.InputStream;
   import java.io.OutputStream;
   import java.net.Socket;
   import java.util.Scanner;
   
   public class ClientDemo {
       public static void main(String[] args) throws Exception {
           while (true) {
               //1.创建 Socket 对象
               Socket s = new Socket("127.0.0.1", 9999);
               //2.从连接中取出输出流并发消息
               OutputStream os = s.getOutputStream();
               System.out.println("请输入:");
               Scanner sc = new Scanner(System.in);
               String msg = sc.nextLine();
               os.write(msg.getBytes());
               //3.从连接中取出输入流并接收回话
               InputStream is = s.getInputStream();
               byte[] b = new byte[1024];
               int read = is.read(b);
               System.out.println("老板说:" + new String(b, 0, read).trim());
               //4.关闭
               s.close();
           }
       }
   }
   ```

   

# 2 NIO编程

## 2.1 I/O模型说明

1. I/O模型简单的理解：就是用什么样的通道进行数据的发送和接受，很大程度上决定了通信的性能
2. Java共支持3中网络编程模型 I/O模式：BIO（同步并阻塞）、NIO（同步非阻塞）、AIO（异步非阻塞）

# 3 Netty核心原理

# 4 Netty高级应用

# 5 Netty核心源码剖析

# 6 自定义RPC框架