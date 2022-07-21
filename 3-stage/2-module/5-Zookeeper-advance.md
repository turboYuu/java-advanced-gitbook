> 第五部分 Zookeeper深入进阶

# 1 ZAB 协议

## 1.1 概念

zookeeper 并没有完全采用 paxos 算法，而是使用了一种称为 Zookeeper Atomic Broadcast（ZAB，Zookeeper 原子消息广播协议）的协议作为其数据一致性的核心算法。

ZAB 协议并不像 Paxos 算法那样是一种通用的分布式一致性算法，它是一种特别为 zookeeper 专门设计的一种**支持崩溃恢复的原子广播协议**

在 Zookeeper 中，主要就是依赖 ZAB 协议来实现分布式数据的一致性，基于该协议，Zookeeper 实现了一种主备模式的系统架构来保持集群中各副本之间数据的一致性，表现形式就是 使用一个单一的主进程来接收并处理客户端的所有事务请求，并采用ZAB 的原子广播协议，将服务器数据的状态变更以事务 Proposal 的形式广播到所有的副本进程中，ZAB 协议的主备模型架构保证了同一时刻集群中只能够有一个主进程来广播服务器的状态变更，因此能够很好的处理客户端大量的并发请求。但是，也要考虑到主进程在任何时候都有可能出现崩溃退出 或 重启现象，因此，ZAB 协议还需要做到当前主进程出现上述异常情况的时候，依旧能正常工作。



![ZooKeeper Service](assest/zkservice.jpg)

## 1.2 ZAB 核心

ZAB 协议的核心是定义了对于那些会改变 Zookeeper 服务器数据状态的事务请求的处理方式。

> 即：所有的事务请求必须由一个全局唯一的服务器来协调处理，这样的服务器被称为 Leader 服务器，余下的服务器则称为 Follower 服务器。
>
> Leader 服务器负责将一个客户端事务请求转化为一个事务 Proposal （提议），并将该 Proposal 分发给集群中所有的 Follower 服务器，之后 Leader 服务器需要等待所有 Follower 服务器的反馈，一旦超过半数的 Follower 服务器进行了正确的反馈后，那么 Leader 就会再次向所有的 Follower 服务器分发 Commit 消息，要求其将前一个 Proposal 进行提交。

![image-20220719172818430](assest/image-20220719172818430.png)



## 1.3 ZAB 协议介绍

ZAB 协议包括两种基本模式：**崩溃恢复** 和 **消息广播**

进入崩溃恢复模式：

当整个服务框架启过程中，或者是 Leader 服务器出现网络中断、崩溃退出 或 重启等异常情况时，ZAB 协议就会进入崩溃恢复模式，同时选举产生新的 Leader 服务器。当选举产生了新的 Leader 服务器，同时集群中已经有过半的机器与该 Leader 服务器完成了状态同步之后，ZAB 协议就会退出恢复模式，其中，所谓的状态同步 就是指数据同步，用来保证集群中过半的机器能够和 Leader 服务器的数据状态保持一致。

进入消息广播模式：

当集群中已经有过半的Follower服务器完成了 和 Leader 服务器的状态同步，那么整个服务框架就可以进入**消息广播模式**，当一台同样遵守 ZAB 协议的服务器启动后加入到集群中，如果此时集群中已经存在一个 Leader 服务器在负责进行消息广播，那么加入的服务器就会自觉地进入**数据恢复模式**：**找到Leader所在的服务器，并与其进行数据同步，然后一起参与到消息广播流程中去**。Zookeeper 只允许唯一的一个Leader服务器来进行事务请求的处理，Leader服务器在接收到客户端的事务请求后，会生成对应的事务提议并发起一轮广播，而如果集群中的其他机器收到客户都的事务请求后，那么这些非 Leader 服务器会首先将这个事务转发给 Leader 服务器。

接下来重点讲解 ZAB 协议的消息广播过程和崩溃恢复过程。

### 1.3.1 消息广播

ZAB 协议的消息广播过程使用原子广播协议，类似于一个二阶段提交过程，针对客户端的事务请求，Leader 服务器会为其生成对应的事务 Proposal，并将其发送给集群中其余所有的机器，然后再分别收集各自的选票，最后进行事务提交。

![image-20220719172818430](assest/image-20220719172818430.png)

在 ZAB 的二阶段提交过程中，移除了中断逻辑，所有的 Follower的服务要么正常反馈 Leader 提出的事务 Proposal，要么就抛弃 Leader 服务器。同时，ZAB 协议将二阶段提交中的中断逻辑移除意味着我们可以在过半的 Follower服务器已经反馈 Ack 之后就开始提交事务 Proposal 了，而不需要等待集群中所有的 Follower 服务器都反馈响应。

但是，在这种简化的二阶段提交模型下，无法处理因 Leader 服务器崩溃退出而带来的数据不一致问题，因此 ZAB 采用了崩溃恢复模式来解决问题。另外，整个消息广播协议是基于具有 FIFO 特性的 TCP 协议来进行网络通信的，因此 能够很容易保证消息广播过程中消息接收与发送的顺序性。

在整个消息广播过程中，Leader 服务器会为每个请求生成对应的 Proposal 来进行广播，并且在广播事务 Proposal 之前，Leader 服务器会首先为这个事务 Proposal 分配一个全局单调递增的唯一 ID，称为之 事务ID（**Zxid**），由于 ZAB 协议需要保证每个消息严格的因果关系，因此必须将每个事务 Proposal 按照其 Zxid 的先后顺序进行排序和处理。

具体的过程：在消息广播过程中，Leader 服务器会为每一个 Follower 服务器都各自分配一个单独的队列，然后将需要广播的事务 Proposal 一次放入这些队列中，并且根据 FIFO 策略进行消息发送。每一个 Follower 服务器在接收到这个事务 Proposal 之后，都会首先以事务日志的形式写入到本地磁盘中，并且在成功写入后反馈给 Leader 服务器一个 Ack 响应。当 Leader 服务器接收到超过半数 Follower 的 Ack 响应后，就会广播一个 Commit 消息给所有的 Follower 服务器以通知其进行事务提交。同时 Leader 自身也会完成对事务的提交，而每一个 Follower 服务器在接收到 Commit 消息后，也会完成对事务的提交。

### 1.3.2 崩溃恢复

ZAB 协议的这个基于原子广播协议的消息广播过程，在正常情况下运行非常良好，但是一旦在 Leader 服务器出现崩溃，或者由于网络原因导致 Leader 服务器失去了与过半 Follower 的联系，那么就会进入崩溃恢复模式。

在 ZAB 协议中，为了保证程序的正确运行，整个恢复过程结束后需要选举出一个新的 Leader 服务器，因此，ZAB 协议需要一个高效且可靠的 Leader 选举算法，从而保证能够快速的选举出新的 Leader。同时，Leader 选举算法不仅仅需要让 Leader 自身知道已经被选举为 Leader，同时还需要让集群中的所有其他机器也能够快速的感知到选举产生出来的新 Leader 服务器。

### 1.3.3 基本特性

根据上面的内容，了解到，ZAB 协议规定了如果一个事务 Proposal 在一台机器上被处理成功，那么应该在所有的机器上都被处理成功，哪怕机器出现故障崩溃。接下来看看在崩溃恢复过程中，可能会出现的两个数据不一致的隐患以及针对这些情况 ZAB 协议需要保证的特性。

**ZAB 协议需要确保那些已经在 Leader 服务器上提交的事务最终被所有服务器提交**

ZAB 协议必须设计成这样一个 Leader 选举算法：能够确保已经被 Leader 提交的事务 Proposal，同时丢弃已经被跳过的事务 Proposal。针对这个要求，如果让 Leader 选举算法能够保证新选举出来的 Leader 服务器拥有集群中所有机器最高编号 （ZXID最大）的事务 Proposal，那么就可以保证这个新选举出来的 Leader 一定具有所有已经提交的提案。更为重要的是，如果让具有更高编号事务 Proposal 的机器成为 Leader，就可以省去 Leader 服务器检查 Proposal 的提交和丢弃工作这一步操作了。

### 1.3.4 数据同步

完成 Leader 选举之后，在正式开始工作（即接收客户端的事务请求，然后提出新的提案）之前，Leader 服务器会首先确认事务日志中的所有 Proposal 是否都已经被集群中过半的机器提交了，即是否完成数据同步。下面看一下 ZAB 协议的数据同步过程。

所有正常运行的服务器，要么成为 Leader，要么成为 Follower 并和 Leader 保持同步。<br>Leader 服务器需要确保所有的 Follower 服务器能够接收到每一条事务 Proposal，并且能够正确地将所有已经提交的事务 Proposal 应用到内存数据库中。<br>具体的，Leader 服务器会为每一个 Follower 服务器都准备一个队列，并将那些没有被各 Follower 服务器同步的事务以 Proposal 消息的形式逐个发送给 Follower 服务器，并在每一个 Proposal 消息后面紧接着再发送一个 Commit 消息，以表示该事务已经被提交。<br>等到 Follower 服务器将所有其尚未同步的事务 Proposal 都从 Leader 服务器上同步过来并成功应用到本地数据库中后，Leader 服务器就会将该 Follower 服务器加入真正的可用 Follower列表中，并开始之后的其他流程。

## 1.4 运行时状态分析

在 ZAB 协议的设计中，每个进程都有可能处于如下三种状态之一：

- LOOKING：Leader 选举阶段
- FOLLOWING：Follower服务器和 Leader 服务器保持同步状态
- LEADING：Leader 服务器作为主进程领导状态

所有进程初始状态都是 LOOKING 状态，此时不存在 Leader，接下来，进程会试图选举出一个新的 Leader，之后，如果进程发现已经选举出新的 Leader 了，那么它就会切换到 FOLLOWING 状态，并开始和 Leader 保持同步，处于 FOLLOWING 状态的进程称为 Follower，LEADING 状态的进程称为 Leader，当 Leader 崩溃或放弃领导地位时，其余的 Follower 进程就会转换到 LOOKING 状态开始新一轮的 Leader 选举。

一个 Follower 只能和一个 Leader 保持同步，Leader 进程和所有的 Follower 进程之间都通过心跳检测机制来感知彼此的情况。若 Leader 能够在超时时间内正常收到心跳检测，那么 Follower 就会一直与该 Leader 保持连接；而如果在指定时间内 Leader 无法从过半的 Follower 进程那里接收到心跳检测，或者 TCP 连接断开，那么 Leader 就会放弃当前周期的领导，并转换到 LOOKING 状态，其他的 Follower也会选择放弃这个 Leader，同时转换到 LOOKING 状态，之后会进行新一轮的 Leader 选举。

## 1.5 ZAB 与 Paxos 的联系和区别

联系：

1. 都存在一个类似于 Leader 进程的角色，由其负责协调多个 Follower 进程的运行
2. Leader 进程都会等待超过半数的 Follower 做出正确的反馈后，才会将一个提议进行提交。
3. 在 ZAB 协议中，每个 Proposal 中都包含了一个 epoch 值，用来代表当前的 Leader 周期，在 Paxos 算法中，同样存在这样的一个标识，名字为 Ballot。

区别：

Paxos 算法中，新选举产生的主进程会进行两个节点的工作，第一阶段称为读阶段，新的主进程和其他进程通信来收集主进程提出的提议，并将它们提交。第二阶段称为写阶段，当前主进程开始提出自己的提议。

ZAB 协议在 Paxos 基础上添加了同步阶段，此时，新的 Leader 会确保存在过半的 Follower 已经提交了之前的 Leader 周期中所有事务 Proposal。这一同步阶段的引入，能够有效的保证 Leader 在新的周期中提出事务 Proposal 之前，所有的进程都已经完成了对之前所有事务 Proposal 的提交。

总的来说，ZAB 协议 和 Paxos 算法的本质区别在于，两者的设计目标不太一样，Z**AB 协议主要用于构建一个高可用的分布式数据主备系统，而 Paxos 算法则用于构建一个分布式的一致性状态机系统。**

# 2 服务器角色

## 2.1 Leader

Leader 服务器是 Zookeeper 集群工作的核心，其主要工作有以下两个：

1. 事务请求的唯一调度和处理者，保证集群事务处理的顺序性。
2. 集群内部各服务器的调度者。

**请求处理链**

使用责任链来处理每个客户端的请求是 Zookeeper 的特色，Leader 服务器的请求处理链如下：

![image-20220721103137034](assest/image-20220721103137034.png)

可以看到，从 prepRequestProcessor 到 FinalRequestProcessor 前后一共 7 个请求处理器组成了 Leader 服务器的请求处理链。

1. PrepRequestProcessor

   请求预处理器，也是 Leader 服务器中的第一个请求处理器。在 Zookeeper 中，那么会改变服务器状态的请求称为事务请求（创建节点、更新数据、删除节点、创建会话等），PrepRequestProcessor 能够识别出当前客户端请求是否是事务请求。对于事务请求，PrepRequestProcessor 处理器会对其进行一系列预处理，如创建请求事务头、事务体、会话检查、ACL 检查 和 版本检查等。

2. ProposalRequestProcessor

   事务投票处理器。也是 Leader 服务器处理流程的发起者，对于非事务性请求，ProposalRequestProcessor 会直接将请求转发到 CommitProcessor处理器，不再做任何处理，而对于事务性请求，处理将请求转发到 CommitProcessor 外，还会根据请求类型创建对应的 Proposal 提议，并发送给所有的 Follower 服务器来发起一次集群内的事务投票。同时 ProposalRequestProcessor 还会将事务请求交付给 SyncRequestProcessor 进行事务日志的记录。

3. SyncRequestProcessor

   事务日志记录处理器。用来将事务请求记录到事务日志文件中，同时会触发 Zookeeper 进行数据快照。

4. AckRequestProcessor

   负责在 SyncRequestProcessor 完成事务日志记录后，向 Proposal 的投票收集器发送 Ack 反馈，以通知投票收集器已经完成了对该 Proposal 的事务日志记录。

5. CommitProcessor

   事务提交处理器。对于非事务请求，该处理器会直接将其交付给下一级处理器处理；对于事务请求，其会等待集群内针对Proposal的投票直到该 Proposal可被提交，利用 CommitProcessor ，每个服务器都可以很好地控制对事务请求的顺序处理。

6. ToBeCommitProcessor

   该处理器有一个 toBeApplied 队列，用来存储那些已经被 CommitProcessor 处理过的可被提交的 Proposal。其会将这些请求交付给 FinalRequestProcessor 处理器处理，待其处理完后，再将其从 toBeApplied 队列中移除。

7. FinalRequestProcessor

   用来进行客户端请求返回之前的操作，包括创建客户端请求的响应，针对事务请求，该处理器还会负责将事务应用到内存数据库中。

## 2.2 Follower

Follower 服务器是 Zookeeper 集群状态中的跟随着，其主要工作有以下三个：

1. 处理客户端非事务性请求（读数据），转发事务请求给 Leader 服务器
2. 参与事务请求 Proposal 的投票
3. 参与 Leader 选举投票

和 Leader 一样，Follower 也采用了责任链模式组装的请求处理链来处理每一个客户端请求，由于不需要对事务请求的投票处理，因此 Follower 的请求处理链相对简单，其处理链如下：

![image-20220721112937584](assest/image-20220721112937584.png)

和 Leader 服务器的请求处理链最大的不同点在于，Follower 服务器的第一个处理器换成了 FollowerRequestProcessor 处理器，同时由于不需要处理事务请求的投票，因此也没有了 ProposalRequestProcessor 处理器。

1. FollowerRequestProcessor

   起作用识别当前请求是否是事务请求，若是，那么 Follower 就会将该请求转发给 Leader 服务器，Leader 服务器在接收到这个事务请求后，就会将其提交到请求处理链，按照正常事务请求进行处理。

2. SendAckRequestProcessor

   其承担了事务日记记录反馈的角色，在完成事务日志记录后，会向 Leader 服务器发送 ACK 消息以表明 自身完成了事务日志的记录工作。

## 2.3 Observer

# 3 服务器启动

## 3.1 服务端整体架构图

## 3.2 单机版服务器启动

## 3.3 集群服务器启动

# 4 Leader选举

## 4.1 服务器启动时期的 Leader 选举

## 4.2 服务器运行时期的 Leader 选举