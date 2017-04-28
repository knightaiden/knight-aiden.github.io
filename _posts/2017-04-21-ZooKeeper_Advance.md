---
layout: post
title: 上手Hadoop-Zookeeper实战篇
date: 2017-04-22 22:00
categories: Tech-Hadoop
tags: hadoop zookeeper backend
---

* content
{:toc}

上一篇博客大致介绍了下怎么在本地部署，这一篇文章，我会举一个最常见的栗子来说明下Zookeeper的实际用法，那就是分布式锁，也借这个topic大致介绍下Curator的用法，希望能帮助到你。




## 分布式锁
### Zookeeper原生分布式锁
> 在程序设计中，避免不了各种资源锁的应用，锁的分类很多，比如可重入锁，非可重入锁，读写锁，互斥锁，悲观锁，乐观锁，公平锁，非公平锁等等等等，如果有机会，在下想有一天开一个新坑专门介绍Java中各种锁的机制，讲真，其实好多锁我的知识体系也并不系统，也希望借机自己做个总结。  

扯远了，现如今分布式系统已经是非常稀松平常的技术来，在分布式系统中，实现资源锁是一项非常重要的技术，其实zookeeper也只是实现分布式锁的其中一种方式，redis，memcache，同样是非常好的方式，再以后的博客中我会找机会来介绍，这一篇，我们着重聊聊zookeeper。  
其实在zookeeper的代码中有个recipes包，这里面包含来zookeeper官方给出的各种应用的实现方式，为了这个事情我在网上看了很多文章，最终觉得还是官方给出的代码写的更加准确一些，所以在下也建了一个项目，不仅无脑复制来zk官方的一部分代码，也同样为各位编写了较为简单的单元测试，帮助理解zk在运行过程中的方式

- [github 仓库地址，点击即可](https://github.com/knightaiden/HadoopSimulator)
- module : ZookeeperSimulator
- package : org.aiden.lab.sample.hadoop.zookeeper.lock
- 类功能概括（由于搬运的是原版代码，就不再代码里面写注释了，当然原本的英文注释还是很给力的）
    - LockListener.java  lock监听器接口，用于定制监听到获取锁和释放锁的事件
    - ProtocolSupport.java 用于实现Zk一些基本功能，主要包括包括：
      - retry一个ZooKeeperOperation
      - 查询一个路径是否存在
    - WriteLock.java	NodeName.java 分布式锁的实际实现逻辑
    - ZNodeName.java 用于辅助节点大小顺序排序的类
    - ZooKeeperOperation.java Zookeeper操作接口，用于定制Zk的实际操作

  我们来实际看一下WriteLock里面的流程，这里面实现了lock和unlock方法，其中lock中加锁的主要动作在内部类LockZooKeeperOperation(implements ZooKeeperOperation)中的execute方法，下面我们来详细说明一下这一段代码  

1.首先execute()方法一进来这个判断，获取idName,其中主要方法是findPrefixInChildren  

```java
if (id == null) {
    long sessionId = zookeeper.getSessionId();
    String prefix = "x-" + sessionId + "-";
    // lets try look up the current ID if we failed
    // in the middle of creating the znode
    findPrefixInChildren(prefix, zookeeper, dir);
    idName = new ZNodeName(id);
}
```

2.在方法中如果存在子节点，那么走for循环，假如子节点的名称符合锁节点前缀规则，那么将当前的id设置成已存在的该节点的名称，之后的流程在来判断这个节点是否会阻止当前进程得到锁。当为找到子节点的情况下，id依然为空，进入if代码块，使用前缀名称来创建一个EPHEMERAL_SEQUENTIAL递增序号的节点，而这个节点就是锁节点，id为当前创建节点的名称  

```java
private void findPrefixInChildren(String prefix, ZooKeeper zookeeper, String dir)
        throws KeeperException, InterruptedException {
    List<String> names = zookeeper.getChildren(dir, false);
    for (String name : names) {
        if (name.startsWith(prefix)) {
            id = name;
            if (LOG.isDebugEnabled()) {
                LOG.debug("Found id created last time: " + id);
            }
            break;
        }
    }
    if (id == null) {
        id = zookeeper.create(dir + "/" + prefix, data,
                getAcl(), EPHEMERAL_SEQUENTIAL);
        if (LOG.isDebugEnabled()) {
            LOG.debug("Created id: " + id);
        }
    }
}
```

3.获取到id后(当前新建的节点id或者已存在节点id)同样再次获取锁路径所有的子节点，并将节点的名称存在一个SortedSet中，用于排序；然后从SortedSet中取到最小节点名称，设置成ownerId(拥有锁的id)，紧接着我们可以看到lessThanMe这个新的SortedSet中尝试获取当前idName的前一个节点，这里我们看到这个if-else便是如果lessThanMe是非空的，那么说明当前节点前还有未删除的锁节点，如果是空的，那么走else，再次判断当前节点idName是否是锁的拥有者ownerId(isOwner()方法)，如果确定还是owner，则判断可以获取到锁。  

```java
if (id != null) {
    List<String> names = zookeeper.getChildren(dir, false);
    if (names.isEmpty()) {
        //...异常流程，没找到子节点
    } else {
        // lets sort them explicitly (though they do seem to come back in order ususally :)
        SortedSet<ZNodeName> sortedNames = new TreeSet<ZNodeName>();
        for (String name : names) {
            sortedNames.add(new ZNodeName(dir + "/" + name));
        }
        ownerId = sortedNames.first().getName();
        SortedSet<ZNodeName> lessThanMe = sortedNames.headSet(idName);
        if (!lessThanMe.isEmpty()) {
          //无法获取锁的流程，见下一点4.说明
        } else {
            if (isOwner()) {
                if (callback != null) {
                    callback.lockAcquired();
                }
                return Boolean.TRUE;
            }
        }
    }
}
```

4.接3.中的内容，还是看lessThanMe，假如这个SortedSet中存在比当前节点还小的未被删除的锁节点，那么说明这个锁还被占用，lastChildName 获取lessThanMe 中的最后一个判断这个节点是否还存在，如果确定还存在，直接告知，无法获取到锁，如果没有状态信息，继续下一轮判断  

```java
if (!lessThanMe.isEmpty()) {
    ZNodeName lastChildName = lessThanMe.last();
    lastChildId = lastChildName.getName();
    if (LOG.isDebugEnabled()) {
        LOG.debug("watching less than me node: " + lastChildId);
    }
    Stat stat = zookeeper.exists(lastChildId, new LockWatcher());
    if (stat != null) {
        return Boolean.FALSE;
    } else {
        LOG.warn("Could not find the" +
                " stats for less than me: " + lastChildName.getName());
    }
}
```

5.之后就是unlock的代码，原理就很简单了，直接删除当前锁节点

```java
public synchronized void unlock() throws RuntimeException {

    if (!isClosed() && id != null) {
        // we don't need to retry this operation in the case of failure
        // as ZK will remove ephemeral files and we don't wanna hang
        // this process when closing if we cannot reconnect to ZK
        try {

            ZooKeeperOperation zopdel = new ZooKeeperOperation() {
                public boolean execute() throws KeeperException,
                        InterruptedException {
                    zookeeper.delete(id, -1);
                    return Boolean.TRUE;
                }
            };
            zopdel.execute();
        } catch (InterruptedException e) {
            LOG.warn("Caught: " + e, e);
            //set that we have been interrupted.
            Thread.currentThread().interrupt();
        } catch (KeeperException.NoNodeException e) {
            // do nothing
        } catch (KeeperException e) {
            LOG.warn("Caught: " + e, e);
            throw (RuntimeException) new RuntimeException(e.getMessage()).
                    initCause(e);
        }
        finally {
            if (callback != null) {
                callback.lockReleased();
            }
            id = null;
        }
    }
}
```


好了，到此为止，主要的锁流程就完成了，如果你想看清流程，建议直接运行单元测试中的SimpleWriteLockTest.java，在方法入口打好断点，全流程清晰明了～  
然鹅！尽管这个实现看上去很令人满意，但是依然很多人吐槽直接使用zk原生的这个客户端非常麻烦，而且还有各种各样的小问题，如果用到自己项目中，还需要修改很多地方，所以这个时候Curator应运而生，他最大限度的封装了zk的使用流程以及实现了各种zk所需的components非常强大，即便如此，我们不得不说，zookeeper原版的recipes的代码依然用最简明的方式说明了分布式锁的实现原理。  

### Curator 分布式锁
> Curator是在线影业租赁届的大佬Netflix公司开源的产品，又一个Apache顶级项目，Curator的抽象层次更高，简化了Zookeeper客户端的开发量。 并且实现了分布式锁，消息队列(然而并不推荐使用)，监听器，并且在微服务架构中是服务的注册发现的首选工具，博主写这篇博客的时候，也在一家视频公司搬砖，而我们组所使用的zk客户端和服务注册发现，正是用的Curator

博主依然在github的项目中编写了一套Curator测试代码，详见[代码仓库（直接点击）](https://github.com/knightaiden/HadoopSimulator)，在此要感谢另一位博主所写的文章，[这篇文章（直接点击）](http://www.cnblogs.com/LiZhiW/p/4931577.html)实现了目前能想到的所有分布式锁的实现，并且代码简单明了，给了我很大的启发,我也在此对他的文章进行一些扩充，详细的说明下Curator中的api
- CuratorFramework Curator的zk客户端，并且使用Fluent风格创建客户端(以及其他的操作)，把创建和启动过程分离，示例代码如下

```java
CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", new ExponentialBackoffRetry(1000, 3));
client.start();
```

- 重试策略  
  - ExponentialBackoffRetry:重试指定的次数, 且每一次重试之间停顿的时间逐渐增加.
  - RetryNTimes:指定最大重试次数的重试策略
  - RetryOneTime:仅重试一次
  - RetryUntilElapsed:一直重试直到达到规定的时间

- 监听器
  - PathChildrenCacheListener 注册监听某个路径下节点和子节点的的更改，对应PathChildrenCache
  - NodeCacheListener 监听某个节点的更改，NodeCache会本地缓存该节点，对应NodeCache
  - TreeCacheListener 上面两种监听器的结合，对应TreeCache

- InterProcessMutex Curator“菜单”的主菜，分布式锁的基本实现  
  通过 acquire() 和 release()方法来控制锁

- InterProcessMultiLock 多锁对象的实现类

除此之外还有很多很多，暂时不一一赘述，通过阅读我们可以发现，Curator真的实现类很多很多功能，如果你的项目需要使用，直接引入如下maven包
```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>2.7.1</version>
</dependency>
```

具体实现可参见代码仓库中的代码
- [github 仓库地址，点击即可](https://github.com/knightaiden/HadoopSimulator)
- module : ZookeeperSimulator
- package : org.aiden.lab.sample.hadoop.zookeeper.sample
- class : CuratorLockTestTask

## 结语
洋洋洒洒又写了一大堆，其实zk以及curator可以实现的功能还有很多，原本想在一篇博客中尽量多的去介绍zk以及curator的方方面面，写到这里才发现完全不可能，只好附上一句"详细的api文档可以直接查询官方说明",但是分布式锁其实是zk使用非常广泛的一个功能，在下也会在未来的日子里把sample写的更加完善，希望能够帮助到你～
