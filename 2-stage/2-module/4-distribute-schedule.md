> 第四部分 分布式调度问题

调度（定时任务），分布式调度（在分布式集群环境下的定时任务）

[elastic-job](https://gitee.com/elasticjob/elastic-job) 当当网开源的分布式调度框架

# 1 定时任务的场景

定时任务形式：每隔一定时间/特定某一时刻执行。

# 2 什么是分布式调度

什么是分布式任务调度？有两层含义

1. 运行在分布式集群环境下的调度任务（同一个定时任务程序部署多分，只应该有一个定时任务在执行）
2. 分布式调度 ----> 定时任务的分布式 ----> 定时任务的拆分（即为把一个大的作业任务拆分为多个小的作业任务，同时执行）



# 3 定时任务与消息队列的区别

- 共同点

  - 异步处理

    比如注册、下单事件

  - 应用解耦

    不管定时任务作业还是 MQ 都可以作为两个应用之间的齿轮实现应用解耦，这个齿轮可以中转数据，当然单体服务不需要考虑这些，服务拆分的时候往往都会考虑。

  - 流量削峰

- 本质不同

  **定时任务作业是时间驱动，而 MQ 是事件驱动；**

  时间驱动是不可替代的，比如金融系统每日的利息结算，不是说利息来一条（利息到来事件）就算一下，而往往是通过定时任务批量计算；

  所以，定时任务作业更倾向于批处理，MQ倾向于逐条处理；

  

# 4 定时任务的实现方式

定时任务的实现方式有多种。早期没有定时任务框架的时候，我们会使用 JDK 中的 Timer 机制的多线程机制（Runnable + 线程休眠）来实现定时 或者 间隔一段时间 执行某一段程序；后来有了定时任务框架，比如大名鼎鼎的 Quartz 任务调度框架，使用时间表达式（包括：秒、分、时、日、周、年）配置某一个任务什么时间去执行：

任务调度框架Quartz回顾示意（主要内容不是 Quartz）

- 引入 jar

  ```xml
  <!--任务调度框架quartz-->
  <dependency>
      <groupId>org.quartz-scheduler</groupId>
      <artifactId>quartz</artifactId>
      <version>2.3.2</version>
  </dependency>
  ```

- 定时任务作业主调度程序

  ```java
  package quartz;
  
  import org.quartz.*;
  import org.quartz.impl.StdSchedulerFactory;
  
  public class QuartzMan {
  
      /**
       * 1、创建任务调度器
       * @return
       * @throws SchedulerException
       */
      public static Scheduler createScheduler() throws SchedulerException {
          SchedulerFactory schedulerFactory = new StdSchedulerFactory();
          Scheduler scheduler = schedulerFactory.getScheduler();
          return scheduler;
      }
  
      /**
       * 2、创建一个任务
       * @return
       */
      public static JobDetail createJob(){
          // 这个地方可以传入任务类
          JobBuilder jobBuilder = JobBuilder.newJob(DemoJob.class); //TODO 自定义任务类
          jobBuilder.withIdentity("jobName","myJob");
          JobDetail jobDetail = jobBuilder.build();
          return jobDetail;
      }
  
  
      /**
       * 3、任务的时间触发器
       * 创建作业任务时间触发器（类似于公交⻋出⻋时间表） * cron表达式由七个位置组成，空格分隔
       * 1、Seconds（秒）  0~59
       * 2、Minutes（分）  0~59
       * 3、Hours（⼩时）  0~23
       * 4、Day of Month（天）1~31,注意有的⽉份不⾜31天
       * 5、Month（⽉） 0~11,或者 JAN,FEB,MAR,APR,MAY,JUN,JUL,AUG,SEP,OCT,NOV,DEC
       * 6、Day of Week(周)  1~7,1=SUN或者  SUN,MON,TUE,WEB,THU,FRI,SAT
       * 7、Year（年）1970~2099  可选项
       *   示例：
       *      0 0 11 * * ? 每天的11点触发执⾏⼀次
       *      0 30 10 1 * ? 每⽉1号上午10点半触发执⾏⼀次 */
      public static Trigger createTrigger(){
          // 创建时间触发器
          CronTrigger cronTrigger = TriggerBuilder.newTrigger()
                  .withIdentity("triggerName","myTrigger")
                  .startNow()
                  .withSchedule(CronScheduleBuilder.cronSchedule("*/2 * * * * ?"))
                  .build();
          // 创建触发器，按简单间隔调度
          /*SimpleTrigger cronTrigger = TriggerBuilder.newTrigger().withIdentity("triggerName", "myTrigger")
                  .startNow()
                  .withSchedule(SimpleScheduleBuilder
                          .simpleSchedule()
                          .withIntervalInSeconds(3)
                          .repeatForever())
                  .build();*/
          return cronTrigger;
  
      }
  
  
      /**
       * main函数中开启定时任务
       * @param args
       */
      public static void main(String[] args) throws SchedulerException {
          // 1、创建任务调度器
          Scheduler scheduler = QuartzMan.createScheduler();
  
          // 2、创建一个任务
          JobDetail job = QuartzMan.createJob();
          // 3、任务的时间触发器
          Trigger trigger = QuartzMan.createTrigger();
          // 4、使用任务调度器根据时间触发器执行我们的任务
          scheduler.scheduleJob(job,trigger);
          scheduler.start();
      }
  }
  ```

- 定义一个 job，需要实现 Job 接口

  ```java
  package quartz;
  
  import org.quartz.Job;
  import org.quartz.JobExecutionContext;
  import org.quartz.JobExecutionException;
  
  import java.text.SimpleDateFormat;
  import java.util.Date;
  
  public class DemoJob implements Job {
  
      @Override
      public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
          SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
  
          System.out.println("我是一个定时任务执行逻辑 : " + simpleDateFormat.format(new Date()));
      }
  }
  ```

以上，是回顾一下任务调度框架 Quartz 的大致用法，那么在分布式架构环境中使用 Quartz 已经不能更好的满足我们需求，可以使用专业的分布式调度框架，这里推荐使用 Elastic-Job。

# 5 分布式调度框架 Elastic-Job

## 5.1 Elastic-Job 介绍

Elastic-Job是当当网开源的一个分布式调度解决方案，基于 Quartz二次开发，有两个相互独立的子项目 Elastic-Job-Lite 和 Elastic-Job-Cloud 组成。我们主要学习的是 Elastic-Job-Lite，它定位为轻量级无中心化解决方案，使用Jar包的形式提供分布式任务的协调服务；而 Elastic-Job-Cloud 子项目需要结合 Mesos 以及 Docker 在云环境下使用。

Elastic-Job 的 github 地址：https://github.com/elasticjob

**主要功能介绍**

- 分布式调度协调

  在分布式环境中，任务能够按指定的调度策略执行，并且能够避免同一任务多实例重复执行

- 丰富的调度策略。

  基于成熟的定时任务作业框架 Quartz cron 表达式 执行定时任务

- 弹性扩容缩容。

  当集群中增加某一个实例，它应当也能够被选举并执行任务；当集群减少一个实例时，它所执行的任务能被转移到别的实例来执行。

- 失效转移。

  某实例在任务执行失败后，会被转移到其他实例执行

- 错过执行作业重触发。

  若因为某种原因导致作业错过执行，自动记录错过执行的作业，并在上次作业完成后自动自动触发。

- 支持并行调度，支持任务分片。

  任务分片是指将一个任务分为多个小任务在多个实例同时执行。

- 作业分片一致性。

  当任务被分片后，保证同一分片在分布式环境中仅一个执行实例

## 5.2 Elastic-Job-Lite 应用

jar 包（API）+ 安装 zk 软件

Elastic-Job 依赖于 Zookeeper 进行分布式协调，所以需要安装 Zookeeper 软件（3.4.6 版本以上）。此处需要明白 Zookeeper 的半只功能：存储 + 通知。

### 5.2.1 安装 Zookeeper（此处单例配置）

zookeeper-3.4.10 下载地址： http://archive.apache.org/dist/zookeeper/zookeeper-3.4.10/

![image-20220710161216670](assest/image-20220710161216670.png)

1. 上传 zookeeper-3.4.10.tar.gz 到 linux，并解压

2. 进入 conf 目录，cp zoo_sample.cfg zoo.cfg

3. 进入 bin 目录，启动 zk 服务

   ```bash
   # 启动
   ./zkServer.sh start
   # 停止
   ./zkServer.sh stop
   # 查看状态
   ./zkServer.sh status
   ```

   ![image-20220710175154573](assest/image-20220710175154573.png)



**Zookeeper的树形节点结构图**

![image-20220710175955819](assest/image-20220710175955819.png)

### 5.2.2 代码

#### 5.2.2.1 引入 Jar

```xml
<!-- https://mvnrepository.com/artifact/com.dangdang/elastic-job-lite-core -->
<!--elastic-job-lite核心包-->
<dependency>
    <groupId>com.dangdang</groupId>
    <artifactId>elastic-job-lite-core</artifactId>
    <version>2.1.5</version>
</dependency>
```

#### 5.2.2.2 定时任务实例

- 需求：每隔两秒执行一次定时任务（resume表中未归档的数据归档到 resume_bak 表中，每次归档一条记录）

  - resume_bak 和 resume 表结构完全一致
  - resume 表中数据归档之后不删除，只将 state 置为 “已归档”

- 数据表结构

  ```sql
  -- ---------------------------- 
  -- Table structure for resume
  -- ---------------------------- 
  DROP TABLE IF EXISTS `resume`;
  
  CREATE TABLE `resume` (
  	`id` BIGINT (20) NOT NULL AUTO_INCREMENT,
  	`name` VARCHAR (255) DEFAULT NULL,
  	`sex` VARCHAR (255) DEFAULT NULL,
  	`phone` VARCHAR (255) DEFAULT NULL,
  	`address` VARCHAR (255) DEFAULT NULL,
  	`education` VARCHAR (255) DEFAULT NULL,
  	`state` VARCHAR (255) DEFAULT NULL,
  	PRIMARY KEY (`id`)
  ) ENGINE = INNODB AUTO_INCREMENT = 1001 DEFAULT CHARSET = utf8;
  
  
  SET FOREIGN_KEY_CHECKS = 1;
  ```

#### 5.2.2.3 程序开发



## 5.3 Elastic-Job-Lite 轻量级去中心化的特点

## 5.4 任务分片

## 5.5 弹性扩容

