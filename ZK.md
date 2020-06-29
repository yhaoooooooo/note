---
typora-root-url: img
typora-copy-images-to: img
---

# Zookeeper

## Getting start

```
 bin/zkCli.sh -server 127.0.0.1:2181
```

**怎么避免脑裂的**

过半机制，，超多一半多的机器才能选举出leader





## ZAB协议（zookeeper atomic Broadcast) 

zookeeper原子广播

Zookeeper 客户端会随机的链接到 zookeeper 集群中的一个节点，如果是读请求，就直接从当前节点中读取数据；**如果是写请求，那么节点就会向 Leader 提交事务，Leader 接收到事务提交，会广播该事务，只要超过半数节点写入成功，该事务就会被提交**。





**Zab 协议的特性**：
 1）Zab 协议需要确保那些**已经在 Leader 服务器上提交（Commit）的事务最终被所有的服务器提交**。
 2）Zab 协议需要确保**丢弃那些只在 Leader 上被提出而没有被提交的事务**。



当leader挂掉之后或者**过半follower无法与其通信**，集群进入 **崩溃恢复模式**，选举新的leader， 过半机器和leader进行同步之后，集群状态从崩溃模式转换成 **消息广播模式**，如果新的机器加入进来的话，自动进入恢复模式，找到leader进行数据同步，同步完成之后作为新的follower进行消息广播流程中。

**保证消息有序**

在整个消息广播中，Leader会将每一个事务请求转换成对应的 proposal 来进行广播，并且在广播 事务Proposal 之前，Leader服务器会首先为这个事务Proposal分配一个全局单递增的唯一ID，称之为事务ID（即zxid），由于Zab协议需要保证每一个消息的严格的顺序关系，因此必须将每一个proposal按照其zxid的先后顺序进行排序和处理。