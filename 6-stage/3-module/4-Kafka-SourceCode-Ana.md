第四部分 Kafka源码剖析

# 1 源码阅读环境搭建

首先下载源码：https://archive.apache.org/dist/kafka/1.0.2/kafka-1.0.2-src.tgz

gradle-4.8.1 下载地址：https://services.gradle.org/distributions/gradle-4.8.1-bin.zip

Scala-2.12.12 下载地址：https://downloads.lightbend.com/scala/2.12.12/scala-2.12.12.msi

## 1.1 安装Gradle

![image-20211215134711958](assest/image-20211215134711958.png)

![image-20211215134757138](assest/image-20211215134757138.png)

![image-20211215134851449](assest/image-20211215134851449.png)

进入GRADLE_USER_HOME目录，添加init.gradle，配置gradle的源：

init.gradle内容

```json
allprojects {
    repositories {
        maven { url 'https://maven.aliyun.com/repository/public/' }
        maven { url 'https://maven.aliyun.com/nexus/content/repositories/google' }
        maven { url 'https://maven.aliyun.com/nexus/content/groups/public/' }
        maven { url 'https://maven.aliyun.com/nexus/content/repositories/jcenter'}

        all { ArtifactRepository repo ->
            if (repo instanceof MavenArtifactRepository) {
                def url = repo.url.toString()

                if (url.startsWith('https://repo.maven.apache.org/maven2/') || url.startsWith('https://repo.maven.org/maven2') || url.startsWith('https://repo1.maven.org/maven2') || url.startsWith('https://jcenter.bintray.com/')) {
                    //project.logger.lifecycle "Repository ${repo.url} replaced by $REPOSITORY_URL."
                    remove repo
                }
            }
        }
    }

    buildscript {

        repositories {

            maven { url 'https://maven.aliyun.com/repository/public/'}
            maven { url 'https://maven.aliyun.com/nexus/content/repositories/google' }
            maven { url 'https://maven.aliyun.com/nexus/content/groups/public/' }
            maven { url 'https://maven.aliyun.com/nexus/content/repositories/jcenter'}
            all { ArtifactRepository repo ->
                if (repo instanceof MavenArtifactRepository) {
                    def url = repo.url.toString()
                    if (url.startsWith('https://repo1.maven.org/maven2') || url.startsWith('https://jcenter.bintray.com/')) {
                        //project.logger.lifecycle "Repository ${repo.url} replaced by $REPOSITORY_URL."
                        remove repo
                    }
                }
            }
        }
    }
}
```

保存并退出，打开cmd，运行：

![image-20211215135356914](assest/image-20211215135356914.png)

设置成功。



## 1.2 Scala安装和配置

双击安装

![image-20211215135440537](assest/image-20211215135440537.png)

![image-20211215135723903](assest/image-20211215135723903.png)

![image-20211215135746718](assest/image-20211215135746718.png)

![image-20211215135849395](assest/image-20211215135849395.png)

![image-20211215135927038](assest/image-20211215135927038.png)

![image-20211215140022055](assest/image-20211215140022055.png)

![image-20211215140221777](assest/image-20211215140221777.png)

![image-20211215140354376](assest/image-20211215140354376.png)

![image-20211215140529307](assest/image-20211215140529307.png)

## 1.3 Idea配置

![image-20211215142015876](assest/image-20211215142015876.png)

## 1.4 源码操作

解压源码

打开cmd ，进入源码根目录，执行：gradle

![image-20211215142412360](assest/image-20211215142412360.png)

结束后，执行 gradle idea（注意不要使用生成的gradlew.bat执行操作）。

![image-20211215143121837](assest/image-20211215143121837.png)

idea导入源码：

![image-20211215143516186](assest/image-20211215143516186.png)

![image-20211215143612610](assest/image-20211215143612610.png)



# 2 Broker启动流程



# 3 Topic创建流程



# 4 Producer生产者流程



# 5 Consumer消费者流程



# 6 消息存储机制



# 7 SocketServer



# 8 KafkaRequestHandlerPool



# 9 LogManager



# 10 ReplicaManager





