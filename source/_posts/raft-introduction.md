---
title: 《Raft-分布式共识算法》- 笔记
tags:
  - Backend
  - Note
  - Distribution
categories:
  - Distribution
date: 2022-03-08 23:00:00
updated: 2022-03-08 23:00:00
---


 
## 一、背景

在分布式系统中，一致性算法至关重要。在所有一致性算法中，`Paxos`最负盛名，它由莱斯利·兰伯特（Leslie Lamport）于`1990`年提出，是一种基于消息传递的一致性算法，被认为是类似算法中最有效的。

`Paxos`算法虽然很有效，但复杂的原理使它实现起来非常困难，截止目前，实现`Paxos`算法的开源软件很少，比较出名的有`Chubby`、`LibPaxos`。此外，`Zookeeper`采用的 `ZAB（Zookeeper Atomic Broadcast）`协议也是基于`Paxos`算法实现的，不过`ZAB`对`Paxos`进行了很多改进与优化，两者的设计目标也存在差异——`ZAB`协议主要用于构建一个高可用的分布式数据主备系统，而`Paxos` 算法则是用于构建一个分布式的一致性状态机系统。

由于`Paxos`算法过于复杂、实现困难，极大地制约了其应用，而分布式系统领域又亟需一种高效而易于实现的分布式一致性算法，在此背景下，`Raft`算法应运而生。

`Raft`算法在斯坦福`Diego Ongaro` 和`John Ousterhout`于`2013`年发表的`《In Search of an Understandable Consensus Algorithm》`中提出。相较于`Paxos`，`Raft`通过逻辑分离使其更容易理解和实现，目前，已经有十多种语言的`Raft`算法实现框架，较为出名的有`etcd`、`Consul` 。


`Raft`是在可信环境的算法，每个节点应该按照“预期”方式运行，非拜占庭，即没有叛徒，有没有欺骗，相互信任。

`Raft` 主要解决以下几个问题：

1. 如何在主从上同步数据。  | 日志负责
2. 如何在异常中选择性的主节点。 | 领导选主
3. 如何保证异常状态中数据安全。  | 数据安全性

[Raft算法演示地址](http://thesecretlivesofdata.com/raft/)


## 二、Raft 核心算法


### 2.1 基本概念

`Raft`将系统中的角色分为领导者（`Leader`）、跟从者（`Follower`）和候选人（`Candidate`）：

* `Leader`：接受客户端请求，并向`Follower`同步请求日志，当日志同步到大多数节点上后告诉`Follower`提交日志。
* `Follower`：接受并持久化`Leader`同步的日志，在`Leader`告之日志可以提交之后，提交日志。
* `Candidate`：`Leader`选举过程中的临时角色。


任期`Term`

* `Raft`的时间被切分为多个任期
* 当切换`Leader`时，首先会进行选举，同事也开启一个新的任期
* `Raft`每个任期只能产生一名`Leader`
* 每一个节点都会保存当前`Leader`的最大任期


![image.png](https://upload-images.jianshu.io/upload_images/12321605-a0904d409c2e4999.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.2 领导人选举

**一次正常请求的处理流程：**

1. 主节点收到请求，追加日志，将数据同步给所有从节点。
2. 从节点收到数据以后，返回`ACK`给主节点
3. 主节点收到了`1/2`以上的节点`ACK`后，确认数据安全，提交数据。

* `Raft`保证只要数据提交了，那么半数以上的节点都会有一份数据备份。
* `Raft`保证集群中**只要半数以上的节点有效，则整个集群能提供正常服务**。


#### 2.2.1 我们如何检查服务是否可用？

* 从节点会监控主节点心跳是否超时
* 任何节点只要发现主节点心跳超时，就可以认为主节点已经失效

![image.png](https://upload-images.jianshu.io/upload_images/12321605-ce4f99bfb3fb82e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 2.2.2 如何选出新的主节点？

* 某个从节点发现心跳超时时，会将自己的任期`Term`加一，并发起新一轮选举
* 任意一个节点收到一轮新任期的选举是，都会进行投票
* 当一个候选人收到半数以上的选票时，赢得此次任期
* 新的主节点开始向所有节点发送心跳


#### 2.2.3 从节点如投票？
  
* 选举的任期比当前任期大
* 一个任期只会投一次票。
* 候选人的数据必须比自己新。 
  
  
#### 2.2.4 如何保证主节点数据是有效的？

* 数据被提交前，至少需要超过半数的`ACK`。即一半以上的节点有已经提交的数据
* 如果要赢的选举，要比半数以上的节点数据新。
 
结论：赢得选举的节点，必然包含最新已经提交的新数据。
 
 
#### 2.2.5 有个没有可能出现多个主节点？

不会
 
 * 新的任期开始后，所有节点会屏蔽掉比当前任期小的请求和心跳。
 * 由于超过半数的节点已经进入新一轮任期，旧`Leader`不再可能获得半数以上的`ACK`。
 * 旧`Leader`一旦收到`Term`更高的心跳，则直接降级为从节点。
 
 
#### 2.2.6 有没有可能出现无法选出合适的主节点？

 * 有可能有平票。
 * 通过随机超时时间，避免下一次选举冲突
 * 当候选再次超时，会把任期+1 ，发起新一轮选举。
 * **任何节点收到高任期的心跳，都会退化为从节点**。

#### 2.2.7 有没有可能出现脑裂？

由于一个任期需要**半数以上**节点投同意票，因此不会出现脑裂



#### 2.2.8 预选举
 
 * 当网络发生异常, 但是节点没有发生异常时。可能会导致某些节点任期无限增加。
 * Raft 采取 “预选举（preVote）”方式避免。
 * 节点在发起选举前，会先发起一轮预选举，当其发现在预选举中能活的半数的支持时，才会真的发起选举


![image.png](https://upload-images.jianshu.io/upload_images/12321605-15513b43e2b82cb8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### 2.2.9 选举总结

![image.png](https://upload-images.jianshu.io/upload_images/12321605-3a55cdd489accc61.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 2.3 成员变更

* 在生产环境中，有时候需要改变集群配置，比如更换坏掉的节点、增加冗余。
* 需要在保证安全性的前提下完成成员变更，不能在同一`term`有多个`leader`
* 同时也喜欢升级不停机，能对外提供服务。
* 如果贸然加入多个节点，势必会导致多个`Leader`节点情况

![image.png](https://upload-images.jianshu.io/upload_images/12321605-c7f83800d50bc8a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 2.3.1 一次只变化一个节点

加入集群流程（挂了一个节点，集群就不可用了。）

1. 先`Leader`申请，`Leader`同步所有申请信息给所有`Follower`
2. 超过半数同意后，新节点加入集群。
3. 之后可以开启新的一轮添加节点。
4. 新增节点由于没有任何日志，无法直接参与新日志追加，会导致新集群可用性变差。
5. 可以引入`Learner`身份，在没有投票权的情况下，先从`Leader`节点获取一段时间日志。

![image.png](https://upload-images.jianshu.io/upload_images/12321605-41afa1170a64b3cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 2.3.1 一次添加多个节点

![image.png](https://upload-images.jianshu.io/upload_images/12321605-9aa937cbe402c949.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/12321605-e2b0595f11882b92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 2.4 日志复制

#### 2.4.1 基本概念

* `Raft`数据包含日志序和数据状态机
* 日志本质上就是一个数组，内部存了一条条日志。
* 任意一个节点“`按序执行`”日志里面的操作，都可以还原相同的状态机结果。
* `Leader`产生日志，同步到`Follower`节点中，`Follower`按序追加到自己的日志队列中执行

由于一个`Term`只会有一个`Leader`、一个`Leader`只会在一个位置放一次日志。
因此索引+任期，能确认唯一一个数据


![image.png](https://upload-images.jianshu.io/upload_images/12321605-1bb78ec190181fab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### 2.4.2 一次正常的日志同步流程

* 主节点在收到新的请求以后，先将日志追加到自己的日志中，这个时候日志还未提交（`uncommit`）
* `Master`将日志提交搞所有从节点，从节点日志也保存到未提交队列中。
* 当`Master`确认半数以上节点获取到日志后，将日志提交


#### 2.4.3 如何处理日志缺失?

* `Master`节点中维护了所有冲节点的下一个预期日志(`next index`)
* 知己诶单只会接受当前`max index`后的下一个日志，其他的日志全部拒绝.
* 子节点会先`Master`汇报自己当前的`max index`

![image.png](https://upload-images.jianshu.io/upload_images/12321605-109ee56898ebc158.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 2.4.4 从节点日志冲突 

* 当主节点宕机时,没有来及吧日志同步给半数以上的节点,就会出现数据冲突
* 从节点收到的日志请求时,**会判断未提交的日志是否发生冲突,如果发生冲突则直接截断覆盖**.


#### 2.4.4 不同节点索引号和任期号是相同，数据一定相同吗？ 
1. 获得一个任期，必须得到超过半数的成员投票。
2. 一个成员永久的只会给一个任期投一次票。
3. 鸽巢原理。

可以推出 ： 一个任期只会有一个`Leader`、`Leader`只会在一个索引处提交一次日志。新`Leader`一定有全新的已经`commit`的日志。


进而可以退出 **不同节点，两个条目拥有相同的索引号和任期号是相同的，那么他们之前所有的数据都是相同的。**


#### 2.4.5 如何快速确定主从节点的日志是否冲突/相同？

如果在不同的节点中的两个条目拥有`相同`的**索引号和任期号**，那么他们之前所有的日志条目也全部相同。

从节点收到一条新的数据时候，还会收到上一条的`Term+Index`，只有和自己的上一条数据完全相同才会追加。否则向前追溯，**不符合条件的全部截断**。

#### 2.4.6 如何判断数据的新旧？

 比较最新日志的任期，更大的、更长的 新



#### 2.4.7  新Leader如何处理前任未提交的数据 ？

1. 新`Leader`只会追加自己的日志，不会删除或覆盖自己的日志（无论是否已被`Commit`）
2. 不主动提交非自己任期的日志。
3. 只在新日志请求来到以后顺便提交。

![image.png](https://upload-images.jianshu.io/upload_images/12321605-27751ef85a94560b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 2.4.8 日志压缩 ？

1. 不定期将日志合并为一张快照，可以缩短日志长度，节约空间。
2. 快照保存了当时的状态机，同时也保存了最后一条

![image.png](https://upload-images.jianshu.io/upload_images/12321605-ea90856c6b23b376.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



 
### 2.5 安全性

* **选举安全性（Election Safety）**：一个任期（`term`）内最多允许有一个领导人被选上。
* **领导人只增加原则（Leader Append-Only）**：领导人永远不会覆盖或者删除自己的日志，他只会增加条目
* **日志匹配原则（Log Matching）**：如果两个日志在相同的索引位置上的任期号相同，那么我们就认为这个日志从头到这个索引位置的之间的条目完全相同。
* **领导人完全原则（Leader Completeness）**：如果一个日志条目在一个给的任期内被提交，那么这个条目一定会出现在所有任期号更大的领导人中。
* **状态机安全原则（State Machine Safety）**：如果一个服务器已经将给定索引位置的日志条目应用到状态机中，则所有其他服务器不会在该索引位置应用不同的条目。


## 三、总结

[https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf](https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf)

[https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md](https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md)

[https://github.com/etcd-io/etcd/tree/main/raft](https://github.com/etcd-io/etcd/tree/main/raft)

[https://ramcloud.atlassian.net/wiki/download/attachments/6586375/raft.pdf](https://ramcloud.atlassian.net/wiki/download/attachments/6586375/raft.pdf)

[https://raft.github.io/](https://raft.github.io/)

[别再怀疑自己的智商了，Raft协议本来就不好理解](https://juejin.cn/post/6844903602918522888)

[etcd Raft库解析](https://www.codedump.info/post/20180922-etcd-raft/)


 
 
 
  
 
  
