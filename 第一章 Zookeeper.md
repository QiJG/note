# 第一章 Zookeeper的理论基础

## 1.1 Zookeeper 简介

Zookeeper 是一个分布式应用程序协调服务器，其为分布式系统提供一致性服务。其一致性是通过基于Paxos算法的ZAB协议完成。

其主要功能包括：配置维护，域名服务，分布式同步，集群管理

## 1.2 一致性

* 顺序一致性
* 原子性
* 单一视图
* 可靠性
* 最终一致性

## 1.3 Paxos算法

Paxos算法是一种基于消息传递的，具有高容错性的一致性算法，Paxos算法的前提是不存在拜占庭将军问题（在不可靠的信道上试图通过消息传递的方式达到一致性是不可能的）

### 1.3.1 算法描述

#### （1）三种角色

* Proposer：提案者
* Acceptor：表决者
* Learner：同步者

#### （2）算法过程描述

Paxos算法的执行过程分为两个阶段：准备阶段 prepare与接受阶段

![image-20221019125021078](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019125021078.png)

**A、Prepare阶段**

1. 提案者(Proposer)准备提交一个编号为N的提议，于是向所有表决者(Acceptor) 发出Prepare(N)的请求，用于试探集群是否支持该编号的提议。
2. 每个表决者(Acceptor)都保存着自己曾经accept过得提议中最大的编号maxN。当一个表决者接收到其他主机发送过来的prepare(N)请求时，会比较N与maxN的值。
   1. 若 N < maxN 则说明该提议已经过时，当前表决者采取不回应或回应Error的方式来prepare请求
   2. 若N > maxN，则说明提议是可以接受的，当前表决者会将该N记录下来，并将其曾经已经accept的编号最大的天Proposer(myid,maxN,value)，反馈给提案者，其中 myid表示该天的提案者表示id，第二个参数表示其曾接受的最大的提案编号maxN，第三个参数表示提案的真正内容。如果当前表决者还未accept过任何提议，则会将 Proposal(myid,null,null)反馈给提案者

![image-20221019131408819](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019131408819.png)

**B、Accept阶段**

1. 当提案者发出prepare(N)，若收到了超过半数表决者的返回，那么该提案者就会将真正的提案Proposer(myid,N,value)发送给所有的表决者
2. 当表决者接收到提案者发送的Proposal(myid,N,value)提案后，会再次拿出自己曾经accept过得提议中最大的编号maxN，或曾经记录下的prepare的最大编号，让N与其比较，若N小于这两个编号，则采取不回应或回应Error的方式拒绝该提议
3. 若提案者没有接收到超过半数的表决者的accept反馈，则可能产生两个结果 一是 放弃该提案，二是重新进入prepare阶段，递增提案编号
4. 若提案者接收的反馈超过了半数，则会向外广播两类消息
   1. 向曾accept其提案的表决者发送"可执行的数据同步信号"，即让其执行其接收过得提案
   2. 向未曾向其发送accept反馈的表决者发送 “提案+ 可执行数据同步序号”，让其接收提案后马上执行

![image-20221019132754633](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019132754633.png)

![image-20221019132827397](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019132827397.png)

> 问题：
>
> 1.在Prepare阶段已经比较过了，并且已经通过了，为什么在Accept阶段还需要进行比较？
>
> 可能在accept阶段，该提案对应的表决者又接收到了新的提案，因此需要进行比较
>
> 2.在Prepare阶段与Accept阶段都进行比较，为什么在COMMIT阶段无需比较
>
> Paxos算法存在机会主义问题

#### 1.3.2 活锁问题

Paxos算法存在活锁问题，Fast Paxos算法进行了改进，只允许一个进程提提案，即该进程具有对N 的唯一操作权，该方式解决了活锁问题

## 1.4 ZAB协议

### 1.4.1 ZAB协议简介

ZAB，Zookeeper Atomic Broadcast，zk原子消息广播协议，是专为Zookeeper设计的一种支持崩溃恢复的原子广播协议。

![image-20221019141159215](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019141159215.png)

Zookeeper使用单一的主进程来接受客户端的所有事务清酒，即写请求，当服务器的数据状态发生变更后，集群采用ZAB原子广播协议，以事务提案Proposal的形式广播到所有的副本进程中。

### 1.4.2 ZAB协议与Paxos算法的关系

ZAB协议时Paxos算法的一种工业实现算法。ZAB协议主要用于构建一个高可用的分布式数据主从系统，即Follower是Leader的从机，Leader挂了，马上就可以选举出一个新的Leader。而Fast Paxos算法则是用于构建一个分布式一致性状态机的系统

### 1.4.3 三类角色

* Leader：事务请求的唯一处理者，也可以处理读请求

* Follower：可以直接处理客户端的读请求，并向客户端响应；但不会处理事务请求，只会将事务请求转发给Leader来处理；对Leader发起的事务提案有表决权；同步Leader中的事务处理结果；Leader选举过程中的参与者，具有选举权和被选举权

* Observer：不参与选举的工作；只响应读请求；也没有表决权

  Observer一般设置为Follower的数量

### 1.4.4 三个数据

* zxid：其为64位的Long类型数据，高32位表示epoch，低32位表示xid
* epoch：每个Leader选举结束后都会生成一个新的epoch，并会通知到其他所有Server,baokuo Follower与Observer。
* xid：事务ID，是一个流水号

### 1.4.5 三种模式：

* 恢复模式：在集群启动过程中，或Leader崩溃后，系统都会进入恢复模式，其包含两个重要阶段：Leader选举与初始化同步
* 广播模式：其分为初始化广播与更新广播
* 同步模式：其分为两类：初始化同步与更新同步

### 1.4.6 四种状态

* LOOKING：选举状态
* FOLLOWING：Follower的正常工作状态
* OBSERVING：Observer的正常工作状态
* LEADING：Leader的正常工作模式

### 1.4.7 同步模式与广播模式

#### (1) 初始化广播

当Leader选举后，此时Leader还是一个准Leader，其要通过初始化同步后才能变为真正的Leader

![image-20221019143927389](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019143927389.png)

具体过程如下：

1. 为了保证Leader向Learner发送的提案有序，Leader会为每一个Learner服务器准备一个队列

2. Leader将哪些没有被各个Learner同步的事务封装为Proposal

3. Leader将这些Proposal逐条发送给Learner，并在每一个Proposal后紧跟一个COMMIT消息，表示该事务已经被提交，Learner可以直接接收并执行

4. Learner接收到Leader的Proposal，并将其更新到本地

5. Learner更新成功后，会向准Leader发送ACK信息

6. Leader服务器接收到来自Learner的ACK后会将其加入到真正可用的Follower列表或Observer列表中；

   没有假如的列表后续通过心跳可以继续同步

   > 在初始化同步过程中还需要 prepare和 accept阶段吗？

#### (2) 消息广播算法

![image-20221019144439253](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019144439253.png)

当集群中的Learner完成了初始化同步，那么整个zk集群就进入了正常工作模式了。

如果集群中的Learner节点收到了客户端的事务请求，那么会将请求转发到Leader服务器，然后再执行以下操作

1. Leader接收到事务请求后，为事务服务一个全局唯一的64位自增ID，即zxid，通过zxid大小比较即可实现事务的有序管理，然后将事务封装为Proposal

2. Leader 根据Follower列表获取所有的Follower，然后将Proposal通过这些Follower的队列将提案发送给各个Follower

3. 当Follower接收到提案后会将提案的zxid与本地记录的事务日志中的最大zxid进行比较。若当前天的zxid大于最大的zxid，则会将提案记录到本地的事务日志中，并向Leader 返回一个ACK。

   > 既然只有一个Leader发送信息：那么为什么还需要比较呢？
   >
   > 可能会由于网络的原因 先发的信息后到，发生ABA的问题

4. 当Leader接收到过半的ACK 后，Leader就会向所有的Follower的队列发送COMMIT消息，向所有的Observer队列发送Proposal。
5. 当Follower收到COMMIT消息后，就会将日志中的事务正式更新到本地。当Observer收到Proposal后，会直接将事务更新到本地
6. 无论是Follower还是Observer在同步完成后都需要向Leader发送ACK。

### 1.4.8 恢复模式的三个原则

#### (1) Leader主动出让原则

当集群中的Leader收到Follower心跳的数量没有过半，此时Leader会自认为自己与集群的连接出现了问题，会主动修改自己的状态位LOOKING，去查找新的Leader，为了防止出现脑裂

而其他的server由于过半的主机认为自己丢失了Leader，所以会发起新的Leader选举

#### (2) 已被处理过得消息不能丢原则

当集群中的Follower 部分同步了最新的消息时(N)，Leader突然挂掉了，这就导致部分Server同步了数据，还有部分Server未接收到COMMOT消息。当新的Leader被选举出来，经过恢复模式后需要保证所有的Server上都执行了哪些已经被部分Server执行过得事务

#### (3) 被丢弃的消息不能再现原则

当在Leader新事务已经通过，并将其更新到了本地，但所有Follower还没有收到COMMIT之前，Leader宕机了，那么当新的Leader选举出来之后，并不知道有Proposal保留在原来的主机，那么该Proposal应该被丢弃

### 1.4.9 Leader 选举

#### （1） Leader选举中的概念

* myid：也称为ServerId这是zk集群中服务器的唯一标识，例如三个zk服务器，那么编号分别为1,2,3
* 逻辑时钟： logicalclock，是一个整型数，该概念在选举时称为logicalclock，选举结束后称为epoch

####    (2)  Leader 的选举算法

**集群启动中的Leader选举**

![image-20221019152117459](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019152117459.png)

在集群初始化阶段，当第一台服务器Serve1启动时会给自己投票，然后发布自己的投票结果，投票中包含myid和ZXID ，此时 Server1的投票为(1,0) 由于其他机器还没有启动所以Server1的状态一直属于Looking即属于非服务状态。

当第二台Server2启动时，此时两台计算机可以相互通信

**宕机后的Leader选举**

![image-20221019153331811](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019153331811.png)



## 1.5 高可用的容灾

### 1.5.1 服务器数量的基数与偶数

6台的容灾能力与5台的容灾能力是一致的，因此建议采用基数

### 1.5.2 容灾设计方案

在生产环境中，三机房部署是最常见的，容灾性最好的部署方案

## 1.6 CAP 定理

### 1.6.1 简介

CAP定理指的是在一个分布式系统中 Consistency(一致性) Availabity(可用性) Partition tolerance(分区容错性)

三者不可兼得

### 1.6.2 BASE理论

BASE 是 Basically Avaliable(基本可用) Soft state(软状态) Eventually consistent(最终一致性)

### 1.6.3 zk 与cp

ZK 遵循的是CP原则，保证了一致性，牺牲了可用性。

当Leader宕机后，zk集群马上进行新的Leader选举，但是选举时长一般在200毫秒，最长不超过60s，整个选举期间zk集群是不接受客户端的读写操作的

## 1.7 zk可能会存在脑裂问题

zk可能会引发脑裂，是在多机房部署时，若出现了网络连接问题，形成多个分区，则可能会出现脑裂问题

# 第二章 Leader 的选举机制

# 第三章 Zookeeper安装与集群搭建



# 第四章 Zookeeper的技术内幕

## 4.1 重要理论

### 4.1.1 数据模型

![image-20221019185835491](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019185835491.png)

#### （1）节点类型

* 持久节点：zk中最常见的节点，节点一旦被创建，只要不删除，其就会一致存在于zk中
* 持久顺序节点：一个父节点可以为其直接子节点维护一份顺序，用于记录子节点的先后顺序。序号由10位数字组成，从0开始计数
* 临时节点：临时节点的生命周期与客户端的会话绑定在一起，会话丢失该节点也会消失。临时节点不能存在子节点
* 临时顺序节点

#### （2）节点状态

* czxid：表示当前znode被创建时事务ID

* ctime：Created Time，表示当前znode被创建的时间

* mZxid：Modified Zxid，表示当前znode最后一次被修改时的事务ID

* mtime：Modified Time，表示当前znode最后一次被修改时的时间

* pZxid：表示当前znode的子节点列表最后一次被修改时的事务ID。注意，只能是其子节点列表变更了才会引起pZxid的变更，子节点内容的修改不会影响pZxid。

* cversion：hildren Version，表示子节点的版本号。该版本号用于充当乐观锁。

* dataVersion：表示当前znode数据的版本号。该版本号用于充当乐观锁。

* aclVersion：表示当前znode的权限ACL的版本号。该版本号用于充当乐观锁。

* ephemeralOwner：若当前znode是持久节点，则其值为0；若为临时节点，则其值为创

  建该节点的会话的SessionID。当会话消失后，会根据SessionID来查找与该会话相关的

  临时节点进行删除。

* dataLength：当前znode中存放的数据的长度。

* numChildren：当前znode所包含的子节点的个数。

### 4.1.2 会话



# 第五章 Zookeeper的典型应用场景

## 5.1 配置维护

![image-20221019222151228](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019222151228.png)

## 5.2 命名服务

![image-20221019222211971](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019222211971.png)

## 5.3 DNS服务

![image-20221019222244125](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019222244125.png)

![image-20221019222258557](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019222258557.png)

![image-20221019222316215](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019222316215.png)

## 5.4 Master 选举

![image-20221019222341715](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019222341715.png)

## 5.5 分布式同步

![image-20221019222409250](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019222409250.png)

## 5.6 集群管理

![image-20221019222436631](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019222436631.png)

## 5.7 分布式锁

![image-20221019222508378](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019222508378.png)

## 5.8 分布式队列

![image-20221019222532940](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221019222532940.png)