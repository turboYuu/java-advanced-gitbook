> 第六部分 附录

# 附录一 乱码问题解决

- Post请求乱码，web.xml 中加入过滤器

  ```xml
  <!--springmvc 提供的针对 post 请求的编码过滤器-->
  <filter>
      <filter-name>encoding</filter-name>
      <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
      <init-param>
          <param-name>encoding</param-name>
          <param-value>utf-8</param-value>
      </init-param>
  </filter>
  <filter-mapping>
      <filter-name>encoding</filter-name>
      <url-pattern>/*</url-pattern>
  </filter-mapping>
  ```

- Get 请求乱码（Get 请求乱码需要修改 tomcat 下 server.xml 的配置）

  ```xml
  <Connector URIEncoding="utf-8" connectionTimeout="20000" port="8080" protocol="HTTP/1.1" redirectPort="8443"/>
  ```



# 附录二  玩转 SpringMVC 必备设计模式



