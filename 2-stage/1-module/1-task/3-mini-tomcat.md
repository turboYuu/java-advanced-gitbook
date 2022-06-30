> 第三部分 手写实现迷你版 Tomcat

名称：Minicat

Minicat 要做的事情：作为一个服务器软件提供服务，也即可以通过浏览器客户端发送 http 请求，Minicat 可以接收到请求进行处理，处理之后的结果可以返回浏览器客户端。

1. 提供服务，接收请求（Socket 通信）
2. 请求信息封装成 Request 对象（Response 对象）
3. 客户端请求资源，资源分成静态资源（html）和 动态资源（Servlet）
4. 资源返回给客户端浏览器

我们递进式完成以上需求，提出 v1.0、v2.0、v3.0 版本的需求：

- v1.0 需求：浏览器请求 http://localhost:8080，返回一个固定的字符串到页面”Hello Minicat!“
- v2.0需求：封装Request 和 Response 对象，返回 html 静态资源文件
- v3.0需求：可以请求动态资源（Servlet）



# 1 Minicat v1.0

首先创建一个 普通的 Maven Java 项目

1. pom中添加 Apache Maven 编译器插件

   ```xml
   <build>
       <plugins>
           <!--Apache Maven 编译器插件-->
           <plugin>
               <groupId>org.apache.maven.plugins</groupId>
               <artifactId>maven-compiler-plugin</artifactId>
               <version>3.1</version>
               <configuration>
                   <source>11</source>
                   <target>11</target>
                   <encoding>utf-8</encoding>
               </configuration>
           </plugin>
       </plugins>
   </build>
   ```

2. Minicat 的主类

   ```java
   package server;
   
   import java.io.IOException;
   import java.io.OutputStream;
   import java.net.ServerSocket;
   import java.net.Socket;
   
   /**
    * Minicat 的主类
    */
   public class Bootstrap {
   
       /**定义socket监听的端口号**/
       private int port = 8080;
   
       public int getPort() {
           return port;
       }
   
       public void setPort(int port) {
           this.port = port;
       }
   
       /**
        * Minicat 启动需要初始化展开的一些操作
        */
       public void start() throws IOException {
           /**
               完成 Minicat 1.0 版本
              需求：（浏览器请求 http://localhost:8080,返回固定的字符串到页面 "Hello Minicat!"）
            */
           ServerSocket serverSocket = new ServerSocket(port);
           System.out.println("====>Minicat start on port: "+port);
           // 阻塞式监听端口
           while (true){
               Socket socket = serverSocket.accept();
               // 有了 socket, 接收到请求,获取输出流
               OutputStream outputStream = socket.getOutputStream();
               String data = "Hello Minicat!";
               String responseText = HttpProtocolUtil.getHttpHeader200("Hello Minicat!".getBytes().length) + data;
               outputStream.write(responseText.getBytes());
               socket.close();
           }
   
       }
   
       /**
        * Minicat 的程序启动入口
        * @param args
        */
       public static void main(String[] args) {
           Bootstrap bootstrap = new Bootstrap();
           try {
               // 启动 Minicat
               bootstrap.start();
           } catch (IOException e) {
               e.printStackTrace();
           }
       }
   }
   ```

3. 响应头信息

   http 协议需要响应头

   ```java
   package server;
   
   /**
    * http 协议工具类，主要是提供响应头信息，这里只提供200和404的情况
    */
   public class HttpProtocolUtil {
   
       /**
        * 为响应码 200 提供请求头信息
        * @return
        */
       public static String getHttpHeader200(long contextLength){
           return "HTTP/1.1 200 OK \n" +
                   "Content-Type: text/html; charset=utf-8 \n" +
                   "Content-Length: "+contextLength +" \n" +
                   "\r\n";
       }
   
   
       /**
        * 为响应码 404 提供请求头信息(此处也包含的数据内容)
        * @return
        */
       public static String getHttpHeader404(){
           String str404 = "<h1>404 not found</h1>";
           return "HTTP/1.1 404 NOT Found \n" +
                   "Content-Type: text/html; charset=utf-8 \n" +
                   "Content-Length: "+str404.getBytes().length +" \n" +
                   "\r\n" + str404;
       }
   }
   ```



启动主类中的main方法，查看效果：

![image-20220630174414206](assest/image-20220630174414206.png)

# 2 Minicat v2.0



# 3 Minicat v3.0



