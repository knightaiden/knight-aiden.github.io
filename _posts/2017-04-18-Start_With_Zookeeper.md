---
layout: post
title: 上手Hadoop-Zookeeper入门篇
date: 2017-04-18 17:00
categories: Tech-Hadoop
tags: hadoop, zookeeper, backend
---

> 最近由于公司项目需要使用Hadoop相关的内容，于是作为一个后台的Java开发人员，决定开一个Hadoop的坑，说起来还真是个有趣的设定。。。  
> Hadoop作为一个老牌的大数据工具，包含了大数据分析处理的方方面面，尽管不得不说随着技术的发展，越来越多的新技术层出不穷，Hadoop作为一个一名长者，被无数人崇敬着，使用者，吐槽着，放弃着，然而在互联网技术横行的如今，这位长者依然在很多企业的很多系统中扮演者至关重要的作用，在下作为一个后台系统的开发人员，只能说对整套体系略知皮毛，借着写博客的契机，总结学习，同时也站在一个非大数据开发人员的角度去分享我自己的一些体会，如有勘误或是任何写的不好的地方，还请各位发邮件给我进行交流。  

* content
{:toc}

首先先要感谢公司里和我一起工作的年轻同事，对于Hadoop平台，其实我并没有很深的接触，各方面给我的指导和帮助确实帮助我提升了很多。尽管多数人建议开篇写一写关于整个Hadoop的体系架构以及从HDFS上逐渐深入，但是经过一番思考，还是决定先从Zookeeper入手，原因很简单，因为在微服务和分布式系统成为主流趋势的现今年代，Zookeeper作为注册中心，分布式信息管理等方面被大部分开发人员广泛的使用着，在我的理解中，其实它所处的角色，于Hadoop其他产品有着一定的应用独立性，相信也有很多开发人员同我一样，最早接触的Hadoop产品不是Hbase，HDFS以及Map Reduce这类，而是将自己的RPC应用注册到了Zookeeper开始了服务化的脚步，所以废了好多话，还是想说，对于Zk，我还是有着一些特♂殊♂的♂感♂情的（捂脸～）




## 装一个本地环境
我相信很多人和我一样，当你想熟悉一个工具的时候，首先先想办法在手里搞一个玩，我这里主要还是以Mac环境为事例来做说明，其实其他平台都是类似的。
- 下载zookeeper安装包，这里我是直接在zk的官网下载的安装包,然后在本地进行解压，当然你也可以直接使用homebrew来安装```$ brew install zookeeper```
- 配置zookeeper  
  当你解压好你的zk之后，进入目录 ./conf 会发现有个叫做zoo_sample.cfg的文件，复制这份文件并重命名为zoo.cfg，然后用你喜欢的文本编辑器打开  
```
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just
# example sakes.
# 这里要说明下，dataDir是你实际存放数据的目录，dataLogDir是你节点日志的位置
dataDir=/Users/zhangzhe/dev/data/zk-node-1
dataLogDir=/Users/zhangzhe/dev/log/zk-node1-log
# the port at which the clients will connect
clientPort=2181
```  
  稍微说明一下的是，我的配置是zk的standalone模式，说白了就是本地一个练习的单机环境，真是的zk部署一定是多记多节点的集群，至于集群中的配置，会在未来的博客中更新
- 保存后进入 ./bin 目录执行命令
```
# 启动zk
$ zkServer start
ZooKeeper JMX enabled by default
Using config: /usr/local/etc/zookeeper/zoo.cfg
Starting zookeeper ... STARTED
# 查看zk运行状态
$ zkServer status
ZooKeeper JMX enabled by default
Using config: /usr/local/etc/zookeeper/zoo.cfg
Mode: standalone
# 停掉zk服务
$ zkServer stop
ZooKeeper JMX enabled by default
Using config: /usr/local/etc/zookeeper/zoo.cfg
Stopping zookeeper ... STOPPED
```

## 使用Zookeeper的API
任何一个强大的工具，都会有一套简单的API，在开始正式的代码之前，先大致了解下java环境中Zookeeper相关的常用API
- org.apache.zookeeper.ZooKeeper
  这是zookeeper客户端最基础的类，通过这个类作为基础对zookeeper进行“增删改查”的操作，详细的用法建议直接参照 [官方文档](http://zookeeper.apache.org/doc/r3.4.5/api/org/apache/zookeeper/ZooKeeper.html)
- org.apache.zookeeper.Watcher(接口)
  ZooKeepe 在运行过程中使用 Watcher来与实际调用zk的程序进行通讯，实现process()方法，即可让主线程获取zk运行过程中感兴趣的信息，在zk的构造函数中我们可以看到在一个zk连接产生时，即可定义watcher所关注的内容。
- ZooKeeper.States(Enum) zk的相关状态常量，[API](http://zookeeper.apache.org/doc/r3.4.5/api/org/apache/zookeeper/ZooKeeper.States.html)
- org.apache.zookeeper.CreateMode
  这个要特别说明一下，因为这个常量涉及到zk结点的不同模式
  - PERSISTENT：持久化目录节点，存储的数据不会丢失。
  - PERSISTENT_SEQUENTIAL：顺序自动编号的持久化目录节点，存储的数据不会丢失，并且根据当前已近存在的节点数自动加 1，然后返回给客户端已经成功创建的目录节点名。
  - EPHEMERAL：临时目录节点，一旦创建这个节点的客户端与服务器端口也就是session 超时，这种节点会被自动删除。
  - EPHEMERAL_SEQUENTIAL：临时自动编号节点，一旦创建这个节点的客户端与服务器端口也就是session 超时，这种节点会被自动删除，并且根据当前已近存在的节点数自动加 1，然后返回给客户端已经成功创建的目录节点名。

## 开始你的表演
好了，现在你有环境，熟悉了大致的API，是时候have a try了～，首先如果你使用maven来构建项目，在pom中定义如下代码：
```xml
<dependency>
  <groupId>org.apache.zookeeper</groupId>
  <artifactId>zookeeper</artifactId>
  <version>3.4.6</version>
</dependency>
```
然后是具体的sample
```java
import org.apache.zookeeper.*;

import java.io.IOException;

/**
 * Created by zhangzhe on 2017/4/17.
 */
public class ZookeeperTester {

    public ZooKeeper zk;

    public Watcher watcher;

    public ZookeeperTester() throws IOException {
        //initwatcher与Zookeeper instance
        watcher = (event) -> System.out.println("monitor event ："+ event.toString());
        zk= new ZooKeeper("localhost:2181", 60*60,this.watcher);
    }

    public void createZKInstance() throws KeeperException, InterruptedException {
        String data = "see you!";
        zk.create("/test", data.getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        String reslut = new String(zk.getData("/test", watcher, null));
        System.out.println(reslut);
        zk.delete("/test",-1);

    }

    //close the Zookeeper instance
    public void ZKclose() throws InterruptedException{
        zk.close();
    }

    public static void main(String[] args) throws IOException, InterruptedException, KeeperException {
        ZookeeperTester zookeeperTester = new ZookeeperTester();
        zookeeperTester.createZKInstance();
        zookeeperTester.ZKclose();

    }

}
```
好啦，到此为止，你已经跑了你的第一个zk应用了！～ Hello World！
