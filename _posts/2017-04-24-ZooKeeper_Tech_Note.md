---
layout: post
title: 上手Hadoop-Zookeeper技术内幕篇
date: 2017-04-24 16:34
categories: Tech-Hadoop
tags: hadoop zookeeper backend
---

* content
{:toc}

之前的两篇文章已经大致的介绍来ZooKeeper从搭建开发环境到上手写代码的一些知识，相信读过之前的介绍，各位看官对ZooKeeper已经有了一定的认识，在下也在最近这几天会完善之前的文章，并且将代码做一定的更新，更全面的写一些demo以及应用场景来分享给各位，当然，如果有时间，也确实应该把英文版写了……毕竟是自己挖的坑。



## 基础知识
### 数据模型
- ZNode  
  首先Zookeeper的最基本数据结构是ZNode，顾名思义，就是节点，zookeeper会从一个根节点树形向下延伸，非常类似Unix的文件目录结构，如果你尝试这跑了前几篇博客的Sample，你可能会在你之前配置的Data目录下看到这样的目录结构  
  ![zk-file](/image/posted/zookeeper/zk-file.png)
  就如标准的文件结构一样，zookeeper也提供了这样的树形目录结构，每个ZNode上可存储少量数据(默认是1M, 可修改配置, 但是并不不建议在ZNode上存储大量的数据)，除此之外，ZNode最重要的是储存acl信息，Acl见下文。另外如之前博客所讲，ZNode的节点分为如下两类
  - Regular ZNode: 常规型ZNode, 用户需要显式的创建、删除
  - Ephemeral ZNode: 临时型ZNode, 用户创建它之后，可以显式的删除，也可以在创建它的Session结束后，由ZooKeeper Server自动删除
  另外加上zookeeper的Sequential特性（按编号递增创建)，就有了我们之前看到clientApi中的四种zookeeper节点创建方式

- Acl  
  既是Access Control的简称，这是一个在传统文件系统中就存在的概念，一般用来表示两个事情：所属组以及权限，自文件自上而下继承父目录。  
  然而在zookeeper中Acl的概念是略有不同的，首先zk的Acl不存在任何继承关系，也就是每个ZNode的Acl彼此独立，其次Zookeeper的要表示三件事情：
  1. scheme：Acl的管理方案，这是一个可扩展的概念，也可以使用zookeeper的省却配置
    - world: 它下面只有一个id, 叫anyone, world:anyone代表任何人，zookeeper中对所有人有权限的结点就是属于world:anyone的
    - auth: 它不需要id, 只要是通过authentication的user都有权限（zookeeper支持通过kerberos来进行authencation, 也支持username/password形式的authentication)
digest: 它对应的id为username:BASE64(SHA1(password))，它需要先通过username:password形式的authentication
    - ip: 它对应的id为客户机的IP地址，设置的时候可以设置一个ip段，比如ip:192.168.1.0/16, 表示匹配前16个bit的IP段
    - super: 在这种scheme情况下，对应的id拥有超级权限，可以做任何事情(cdrwa)
    - sasl: sasl的对应的id，是一个通过sasl authentication用户的id，zookeeper-3.4.4中的sasl authentication是通过kerberos来实现的，也就是说用户只有通过了kerberos认证，才能访问它有权限的node.
  2. id：使用该scheme的用户id，可以是anyone
  3. permissions：可以操作的权限
    - CREATE(c): 创建权限，可以在在当前node下创建child node
    - DELETE(d): 删除权限，可以删除当前的node
    - READ(r): 读权限，可以获取当前node的数据，可以list当前node所有的child nodes
    - WRITE(w): 写权限，可以向当前node写数据
    - ADMIN(a): 管理权限，可以设置当前node的permission
  > 尽管之前的sample这个参数很多时候都传入的是null，但是在实际项目中acl还是一个非常重要的应用。建议有时间详细读一下官方文档

### ZooKeeper中的角色
说起ZooKeeper的原理，首先要提到的就是ZK的三大角色，Leader，Learner，Client，其中learner又分为Follower和Observer，同时之所以存在这三大角色，都是为了ZK中的选举，听上去好似"民主"国家的政治一样，那zookeeper为什么需要选举，而选举中这三大角色的指责又是什么呢？且耐心往下看  
- 选举  
  说到底，ZooKeeper是一个为分布式系统服务的中间件，而ZooKeeper本身在实际部署中也是一个集群环境，或庞大或精简，但是都是面临一个多server多节点的状态，而这种状态必须要保持数据一致性，至少是对外的一致性，那么在这样一个环境中总得有个说了算的人，以及一大帮跟着混的人，而如何选出这个说了算的人就依赖于ZooKeeper本身的一致性算法Paxos，而这就是一个选举投票的算法，稍后我会着重说明下这个算法，在下觉得，这个算法确实挺复杂的，我也是晕了很久才逐渐明白。
- Leader（领导者) zookeeper在运行过程中就像是一个紧密有序的组织，而Leader就是这个组织中负责发起投票和决议，以及最终更新系统状态的角色，同样Zookeeper也是一个节点无差别的系统，也即是说这个leader可以挂掉，也可以更换成其他节点来继任
- Learner（学习者） 按大白话说就是广大的负责具体工作的劳动人民，比如码农，产品狗等等，在leader的领导下尊从系统运行规则，并且时刻准备着变成leader，而这个learner细分了两种角色
  - Follower 接受client请求，并相应client请求，在选主的过程中要参与投票
  - Observer 接受client的连接请求，并且向leader发送client的写请求，同步leader状态。与Follower最大的区别是，Observer并不参与投票，因为Observer的设计目的就是为了能够快速扩展系统从而提高读取的速度。
- Client（客户端) 说白了就是zookeeper的消费者，正如其名，他们是最终zk的客户，就像之前在下写的那些demo，他们向zk发起请求满足自己无穷无尽的欲望。
总的来说ZooKeeper的逻辑如下图所示  
![zk-arch](/image/posted/zookeeper/ZK-Arch.jpg)

## Zookeeper的工作原理
### 原子广播
  原子广播是zookeeper工作的核心机制，这个机制保证了各个server间的同步，名曰：ZAB协议。Zab协议又分为两种状态：
  - 恢复模式（选主模式） 恢复模式是在leader挂掉之后，选出新的leader的过程，当新的leader产生，大多数server和leader同步之后，即退出恢复模式
  - 广播模式 即各个follower向leader通讯的过程，稍后详细讲解。

### 递增的事务id号（zxid）
  所有的提议（proposal）都在被提出的时候加上了zxid。实现中zxid是一个64位的数字，它高32位是epoch用来标识leader关系是否改变，每次一个leader被选出来，它都会有一个新的epoch，标识当前属于那个leader的统治时期。低32位用于递增计数。  
  每个Server在工作过程中有三种状态：
  - LOOKING：当前Server不知道leader是谁，正在搜寻
  - LEADING：当前Server即为选举出来的leader
  - FOLLOWING：leader已经选举出来，当前Server与之同步

### 同步流程
  在leader存在的情况下，zookeeper主要的同步流程如下
  1. leader等待server连接；
  2. Follower连接leader，将最大的zxid发送给leader；
  3. Leader根据follower的zxid确定同步点；
  4. 完成同步后通知follower 已经成为uptodate状态；
  5. Follower收到uptodate消息后，又可以重新接受client的请求进行服务了。

### 选主流程
zookeeper的选主流程主要基于一致性算法paxos实现的，分为两种basic paxos和fast paxos（默认)，首先先大致介绍下基于算法的zookeeper实现流程，未来我会抽出一篇博客的时间来专门讲解paxos的细节以及背后的故事与思想。

#### basic paxos
basic paxos机制作为Zookeeper对paxos算法最基本的实现，虽然没有被zk用作默认选主算法，但是却很明确的说明了paxos算法的核心意思，首先paxos是一种“民主的算法”，所以对于任意节点，都是有机会成为leader的，所以不管是basic paxos还是fast paxos算法，首先大家在第一轮都先选自己，然后对于basic paxos算法，每个server对集群内所有的server（包括他自己）发起一次询问：“你们选谁呢？！”，这个时候其他节点会把自己的结果反映给他，而这个节点会根据zxid判断是不是自己发的提案，只要不是自己的提案，都会把这个提案的（id,zxid）存在自己的投票表内进行收集，当投票结束后，会整理所有结果，将zxid最大的server作为自己认为的leader，这个时候再去检查投票列表中自己推选的leader是否有超过半数的其他server支持，如果有，那么该serve被推举为leader，更改server状态，如果没有半数以上的支持，那么继续回到之前的步骤，告知其他server自己选举的leader在询问其他server。  
听到这里不知道你是否能够看得明白，我觉得最好还是分成步骤再详细介绍一遍。
1. 第一轮投票，所有人都选自己  
2. 发起提议，选我！  
3. 询问其他server，并接收其他server的选主提案  
4. 如果接收的是自己发起的提案，则直接丢弃/如果不是自己的发起的提案，获取对方的leader信息并存在自己的提案信息表中（id,zxid）。  
5. 结束本轮投票后，统计投票表中的结果，将zxid最大的server作为自己新推举的leader。  
6. 检查投票表中自己新推举的leader是否有超过半数的其他server也推举（假设2N+1个server，则需要大于等于N+1个server）。  
7. 如果超过半数的server提起相同的leader议案，则该leader被选出，更改server状态/如果没有超过半数的server，则继续回到步骤3.直至leader产生。  
到此我相信你已经对basic算法有了一个初步的了解。

#### fast paxos
对于basic paxos算法，虽然最终能够选出server但是却面对两种问题，第一是速度不够快（总结为数学收敛，收敛速度较慢），第二就是所谓的“活锁问题”，简单来说就是一个“竞争”问题，假如提议者（Proposer）大于等于三，很难收到半数以上相同leader提案，这个时候就会不断的重复basic paxos中的步骤3.这时候就不是一个很好的状态了。  
fast paxos算法，首先一个重要的概念是逻辑时钟，这个概念其实很大，为了便于理解，可以先理解为每次发起选举，server都会对这个逻辑时钟+1，意思就是逻辑时钟越大，选举的信息越新。fast paxos算法和basic paxos算法最核心的区别就在于接受消息的处理，具体流程如下：
1. 准备发起投票，更新逻辑时钟+1  
2. 发起提议，选自己  
3. 询问其他server，并接收其他server的选主提案  
4. 依次判断接受消息中的逻辑时钟（Epoch）->zxid->serverId，只要满足大于当前推举leader的值就推举这个较大的为leader  
5. 发送新推举leader的信息  
6. 如果自己推举的leader超过半数（同basic），则确定该server为leader,更改server状态。  
便是fast paxos算法中的流程  

#### 同步流程
当选主流程完成后就进入了同步流程，这个相对简单很多  
1. leader等待server连接；  
2. Follower连接leader，将最大的zxid发送给leader；  
3. Leader根据follower的zxid确定同步点；  
4. 完成同步后通知follower 已经成为uptodate状态；  
5. Follower收到uptodate消息后，又可以重新接受client的请求进行服务了。  


## Zookeeper的运行流程

### leader工作流程
首先leader一个很重要的用途是与各个learner维持心跳，同时等待接收来自learner的请求，而learner会有四种请求内容，leader会根据不同类型的内容做不同的操作
- PING 即使心跳请求，得知这个server还活着，另外leader会判断此时是否有超过半数follower还活着，如果没有就进行选举
- REQUEST 即follower发起的提议，leader会放在处理队列中，后续会向各个follow询问是否接受（接受即返回下面的ACK）
- ACK 即Follower的对提议的回复，超过半数的Follower通过，则commit该提议使其生效
- REVALIDATE 是用来延长SESSION有效时间。

### follower工作流程
follower主要会做三件事情
- 向leader发起请求，即上面的四种消息
- 循环接收leader的消息
	- PING消息： 心跳消息；
	- PROPOSAL消息：Leader发起的提案，要求Follower投票；
	- COMMIT消息：服务器端最新一次提案的信息；
	- UPTODATE消息：表明同步完成；
	- REVALIDATE消息：根据Leader的REVALIDATE结果，关闭待revalidate的session还是允许其接受消息；
	- SYNC消息：返回SYNC结果到客户端，这个消息最初由客户端发起，用来强制得到最新的更新。
- 接收Client的请求
	- 读请求，直接回复
	- 写请求，向leader发起请求

> 好了，到此为止，相信你对zookeeper的运行流程已经有个大致的了解了，对于这篇博客，也许还有很多细节需要补充，我会在之后的更新中不断完善，如果您有任何想要交流的内容，欢迎给我写邮件，另外这个版本的博客我加入了评论功能，希望能有回复~
