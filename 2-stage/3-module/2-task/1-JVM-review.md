第一章 JVM回顾

# 1 什么是JVM

> 什么是JVM

JVM是Java Virtual Machine（Java虚拟机）的缩写，JVM是一种用于计算设备的规范，它是一个虚构出来的计算机，是通过再实际的计算机上仿真模拟各种计算机功能来实现的。

> 主流虚拟机

| 虚拟机名称  | 介绍                                                         |
| ----------- | ------------------------------------------------------------ |
| HotSpot     | Oracle/Sun JDK和Open JDK都使用HotSpot VM的相同核心           |
| J9          | J9是IBM开发的高度模块化的JVM                                 |
| **JRockit** | JRockit与HotSpot同属于Oracle，目前位置Oracle一直在推进HotSpot与JRockit<br>两款各有优势的虚拟机进行融合互补 |
| Zing        | 由Azul Systems根据HotSpot为基础改进的高性能低延迟的JVM       |
| Dalvik      | Android上的Dalvik 虽然名字不叫JVM，但骨子里就是不折不扣的JVM |

# 2 JVM与操作系统

> 为什么要在程序和操作系统中间添加一个JVM

Java是一门抽象程度特别高的语言，提供了自动内存管理等一系列的特性。这些特性直接在操作系统上实现是不太可能的，所以就需要JVM进行一番转换。

![image-20211007001822553](assest/image-20211007001822553.png)

从图中可以看到，有了JVM这个抽象层之后，Java就可以实现跨平台了。JVM只需要保证能够正确执行.class文件，就可以运行在诸如Linux、Windows、MacOS等平台上了。

而Java跨平台的意义在于一次编译，处处运行，能够做到这一点JVM功不可没。比如在Maven仓库下载同一版本的jar包就可以到处运行，不需要再每个平台上在编译一次。

下载乃的一些JVM的扩展语言，比如Clojure、JRuby、Groovy等，编译到最后都是.class文件，Java语言的维护者，只需要控制好JVM这个解析器，就可以将这些扩展语言无缝的运行在JVM之上了。



> 应用程序、JVM、操作系统之间的关系

![image-20211007002403026](assest/image-20211007002403026.png)

用一句话概括JVM与操作系统之间的关系：JVM上承开发语言，下接操作系统，它的中间接口就是字节码。

# 3 JVM、JRE、JDK的关系

![image-20211007002523118](assest/image-20211007002523118.png)

JVM是Java程序能够运行的核心。但是需要注意，JVM自己什么也干不了，你需要给他提供生产原料（.class文件）。

仅仅是JVM，是无法完成一次编译，处处运行的。它需要一个基本的类库，比如怎么操作文件，怎么连接网络等。而Java体系很慷慨，会一次性将JVM运行所需的类库都传递给它。JVM标准加上实现的一大堆基础类库，就组成了Java的运行时环境，也就是常说的JRE（Java Runtime Environment）。

对于JDK来说，就更庞大了一些。除了JRE，JDK还提供了一些非常好用的小工具，比如javac、java、jar等。它是Java开发的核心，让外行也可以炼剑！

可以看下JDK的全拼，Java Development Kit。我们非常怕kit（装备）这个单词，它就像一个无底洞，预示着你永无休止的对它进行研究。JVM、JRE、JDK它们三者之间的关系，可以用一个包含关系表示。

![image-20211007003945593](assest/image-20211007003945593.png)

# 4 Java虚拟机规范和Java语言规范的关系

![image-20211007003958060](assest/image-20211007003958060.png)