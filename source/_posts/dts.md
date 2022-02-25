---
title: 数据传输系统落地和思考
tags:
  - Backend
  - Golang
categories:
  - Golang
date: 2022-02-26 22:00:00
updated: 2023.10.09 12:50:20
---



## 一、背景


我们的产品需要支持 [Multi-Geo](https://docs.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-multi-geo?view=o365-worldwide) 功能。 


什么是`Multi-Geo`？简单的说就是：“将一个租户下不同用户/设备/组织等数据，分散存储在不同的地理位置的能力”，在同一个租下管理员可以配置任意用户的数据驻留地（`Preferred Data Location`简称`PDL`）。

该功能主要是解决跨国企业，数据合规存放的问题。支持同一个企业下，不用国家的用户，数据存放在不同的国家的机房。



`Multi-Geo`的功能涉及到几点核心能力。

1. **数据的路由能力**。比如，我们服务在`CN`收到一个`User`数据查询需求，首先我们要知道这个`User`是归属于`CN`还是`i18n`（国外）的`Unit`，然后再把请求转发给相应的`Unit`的服务。
2. **数据的定位能力**，管理员更新用户的`PDL`时候，我们需要把用户所有的数据（存量和增量）找出来，然后发送到新的数据驻留地。
3. **数据传输的能力**，数据传输主要包括存量数据和增量数据的传输过程，存量和增量的`overlap`怎么处理，业务上是否需要停写，什么时间点修改数据的`Unit`信息，让后续数据的增删改查请求转发到更新以后的`Unit`。


## 二、数据路由

为了支持数据路由的能力，我们引入了`Global Meta` 和 `Edge Proxy`两个组件。这个是之前做`Unit`互通就已经有了的组件，不是本文讨论重点。简单说下大概流程如下：

假设一个用户在`CN`的发起请求，拉取一个`i18n`的用户的资料，大概链路应该是这样

1. 客户端`Http`调用查询`GetUserInfo`接口查询用户资料。
2. `HttpGateway`收到用户的`Http`请求，转成`Thrift`请求，`RPC`调用`User`服务的`GetUserInfo`接口。
3. `User`服务收到请求以后，会有一个通用的`Cross-MiddleWare`，它会提出`IDL`中的打了`Cross`标记的`Tag`，当前上下这个`Tag`是`UserID`字段，所以`Cross-MiddleWare` 会提取出 `UserID`，然后去`Global Meta`查询这个`UserID`的归属于哪个`Unit`。
4. `Cross-MiddleWare`查出这个`UserID`不属于当前`Unit`以后，它会设置`Dst-Unit`和`Dst-Service`，然后当前请求转发给`EdgeProxy`。
5. `EdgeProxy`取出`Dst-Unit`，然后把这个请求发送`Dst-Unit`对应的`EdgeProxy`。
6. `Dst-Unit`的`EdgeProxy` 会取出`Dst-Service`，然后把请求转发给`Dst-Service`，这里就是转发给`i18n`的`User`服务。
7. `i18n`的`User`服务发现当前请求是`Cross`过来的，不会再去请求`Global Meta`，会直接走本地的查询逻辑，查出`User`的数据返回给`EdgeProxy`
8. `EdgeProxy`拿到`Response`以后，会走原路返回给客户端。

![image.png](https://upload-images.jianshu.io/upload_images/12321605-b94e52b2371bb8db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 三、数据定位

数据定位能力是指，我们能快速在`MySQL`、`Redis`、`Abase`这些存储组件中找到归属于`User`所有的数据的能力。常用的解决方案有两种：正向查询定位、数据打标定位。

### 3.1 正向查询

一是正向查询，比如我们有一个`UserID`，我们可以查到`User`下面所有的`Chat`，再查到归属于`Chat`所有的`Message`，然后查到`Message`下面的`Reaction`和`File`等等资源，依次类推可以查到所有`User`相关的信息。

![image.png](https://upload-images.jianshu.io/upload_images/12321605-b56c83b3c7878d46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

正向迁移的方案有几个弊端

1. 并不是所有的场景都能正向查询，就比如文件转发场景，有两个表`file_info`和`file_ref`,每次转发的时候，会新产生一个`file_key`，并把`file_key`和`parent_key`关系记录到`file_ref`中。`file_ref`表索引只有`file_key`，如果想通过`parent_key`拉取改文件所有被转发的记录，是不支持的。
2. 部分数据还需要解析内容才能得到，比如`Message`和`File`关系，`Message`的`Content`是加密以后存在数据库中的，如果要拿到`Message`中的`file_key`，我们必须把内容取出来，解密，然后反序列化为`PB`的`Struct`。这样对业务耦合太重，会对系统的扩展性和可维护性造成一定困难。

这个方案不够`General`，站在架构侧我们希望能够提供更`General`的方案，而不是去过多的关注业务的数据结构、数据层级。

### 3.2 数据打标

数据打标，就是有一个专门的数据打标系统，会对系统里面每条产生的数据打上标记（有个前提需要基础组件支持`binlog`），可以按自己的需求打上`User`标、`Chat`标等等，查询的时候，能够按自己想要的方式快速查询出来。打标方式如下：

![image.png](https://upload-images.jianshu.io/upload_images/12321605-88e519cf13ce5193.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 3.2.1 MySQL Schema 打标规则描述

字段  | 描述
------------- | -------------
repo_name  | 数据库名
table_name  | 数据表名
index_keys  | 唯一索引列，用于存量数据的获取，比如`Message`表唯一索引是["chat_id","message_id"]，消费到`Message`表`binlog`时候，我们会记录下这两个列的值。方便后面需要迁移数据的时候，我们能够快速定位到相关数据
need-replace-pk  | 是否需要替换PK字段，如果表的`id`字段是用的`MySQL`的自增`id`，所以插入时候可能会冲突，需要插入前替换掉
entity-type  | 实体类型，比如`Message`、`Chat`、`File`、`Reaction`，每个数据都可以归属到一个实体类型，实体类型之间也有层级归属关系。
entity-id-field  | 哪一列是实体数据，比如`chat`表的`id`字段表示`Chat`这个实体数据

`MySQL`的数据打标匹配比较简单，收到`MySQL`的`Binlog`，用`DB`+`Table`就能`Match`到`binlog`对应的`Schema`，然后可以找到`Schema`里面的`IndexKeys`和`Entity`数据，就能知道这个`binlog`表示的`DataEntity`数据。

#### 3.2.2 NoSQL Schema 打标规则描述

字段  | 描述
------------- | -------------
repo_name  | 数据库名
pattern  | key的格式，如 `{{env}}:chat_last_msg_id:{{message_id}}`
entity-type  | 实体类型，比如`Message`
entity-id-field  | 哪一列是实体数据，比如上面`pattern`中的`message_id`


`NoSQL`的匹配相对要复杂一点，首先需要根据`Repo`的纬度构建压缩字典树`radix_tree`，然后通过`key`来匹配，找到`key`对应的`Schema`。有点类似`Http`请求的`Path`匹配，有点不同的是`Path`都是“`/`”结束，截取变量的话相对简单一些。

做`NoSQL`的`Key`匹配的时候，可能比`Path`要复杂一点，比如`{{table_id}}_{{rev}}_{{rec_id}}`和`{{table_id}}_{{rev}}_{{rec_id}}_{{level}` 用`tableStr_1000_abccc_1`key是可以同时匹配上面两个的`pattern`的。

这个时候只能通过增加变量的限制条件，比如`rec_id`必须包含`xx`字符串，不包含`xx`字符串，变量必须是数字、变量长度固定是多少位的方式来做匹配。



#### 3.2.3 打标流程如下


![image.png](https://upload-images.jianshu.io/upload_images/12321605-53278902baada353.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### 3.2.4 打标库选型


![image.png](https://upload-images.jianshu.io/upload_images/12321605-4893819dee218052.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

存储选型前，需要明确自己的数据量和需求。

1. 存量数据数在`百亿~千亿`级别，数据总大小概在`几百TB`左右。
2. 增量`Binlog`的`QPS`在`200W`左右， 需要打标的增量数据，预期有`10W`左右`TPS`左右。
3. 写多读少，主要是打标迁移的时候才会查，还会有少量的删除操作。
4. 对事务没有要求。
5. 有一定的一致性要求，写入打标数据以后，需要能再预期的时间内查到。

数据库类型	  | 常见数据库
------------- | -------------
关系型  | MySQL、Oracle、DB2、SQLServer 等。
非关系型 | Hbase、Redis、MongodDB 等。
行式存储 | MySQL、Oracle、DB2、SQLServer 等。
列式存储 | Hbase、ClickHouse 等。
分布式存储 | Cassandra、Hbase、MongodDB 等。
键值存储 | Memcached、Redis、MemcacheDB 等。
图形存储 | Neo4J、TigerGraph 等。
文档存储 | MongoDB、CouchDB 等

分别调研了公司内部几个代表性的存储

1. 关系型，`NDB`，对标业界最流行的`cloud native`的`RDS AWS Aurora/Alibaba  PolarDB`，100%兼容`MySQL`，计算存储分离，独立扩缩计算/存储，成本较低。 `DBA`给的数据是单台机器`20`核 `128G`内存，最大写入`TPS`可以到 `15~20K`左右。
2. 非关系型，`Abase`，基于`RocksDB`的分布式`KV`存储，支持`Redis`协议、极致高可用、低延迟、大容量的在线存储 & 缓存服务，一个小集群能支持几十万的写入。单库支持`PB`级别的数据存储。
3. 列式存储，`ClickHouse`，适用于大批量的数据插入、基本无需修改现有数据、拥有许多列的大宽表、在指定的列进行聚合计算操作、快速执行的 `SELECT` 查询等场景。目前只支持从`Kafka`导入数据，且导入的数据，有一定的延迟（10分钟以内）才能查到。



#### 3.2.4 打标方案总结

打标方案的好处是 方案更`General`一些，业务方只需要按照我们提供的`Schema`规则，来我们系统里面注册就好了。

缺点就是，增加了资源的开销，需要额外的存储。

还有一个就是，打标方案有一个假设，就是一定能从下往上查，比如可以从`Message`查到`Chat`，再从`Chat`查到归属的`User`，但是实际中还有少量表的数据是没办法这样往上查的。

所以这一部分逻辑，我们会在迁移的某个表的数据时候，会正向查出这个数据关联的子表数据。然后一起迁移走。


## 四、数据传输


### 4.1 业务实体

![image.png](https://upload-images.jianshu.io/upload_images/12321605-5bcb732e5d40a5a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


* `Biz Entity` : 业务实体类型，是一个逻辑概念，比如一个用户是一个`Biz Entity`，或者一个租户是一个`Biz Entity`，暴露给管理员的最小迁移单元。
* `Locating Entity`: 简称`LE`，用于判断数据归属`Unit`以及数据迁移的最小单元。本身可包含子业务实体。比如一个会话、一篇文档都可以认为是一个`LE`
* `Sub Entity`: 简称`SE`，`Sub Enitty`的归属`Unit`继承于`Locating Entity`并随`Locating Entity`一起迁移。


### 4.2 数据传输方案


#### 4.2.1 存量+停写+增量

**一、Locating Entity迁移状态修改**

这一步主要是标记`Locating Entity`状态为迁移中，开始对`Locating Entity`存量数据扫描，与此同时记录所有`Locating Entity`相关的增量`binlog`数据。

**二、存量数据同步**

在打标库中，查出`Locating Entity`所有的打标数据，然后根据`Schema`信息，去业务`Repo`中查出所有数据，生成`binlog`发送到对端。

**三、增量数据同步**

存量数据同步完成以后，我们再开始同步“`步骤一`”开始记录的增量数据。

**四、实时同步状态**

发送完增量数据的瞬间，我们需要对这个`Locating Entity`加锁，然后修改当前`Locating Entity`的状态为`Syncing`状态，后续所有的增量`binlog`可以实时发送到`Dst Unit`。

**五、停写**

在状态达到`Syncing`以后，且时间到了我们配置的某个“`时间点`”，我们会先判断这个`LE`能否开始停写（检查`binlog`消费是否有延迟等等），如果可以停写，则设置`Locating Entity`状态为“`停写`”。

停写即是暂停对一个`Locating Entity`的所有数据的写入和修改，主要为了规避以下问题：

- 防止修改`Locating Entity`的归属`Unit`时，各服务读取的`Locating Entity Unit` 有短暂的不一致，这样造成数据会写入地不一致的问题
- 保证剩余的增量数据全部同步到目标机房。防止迁移结束后，仍后剩余数据未迁移，亦或是当业务写入数据较快，导致迁移任务无法结束

停写可以发生在“数据层”和“业务层”。我们这里选取“业务层”停写的方案。

优点：性能好，可以实现“`fail-fast`机制”，出错以后可以尽快返回，能减少无用`RPC`和数据写入请求。

缺点：需要各业务方识别出所有的“写”接口并引入停写机制，比较难统一处理和维护。

停写的纬度是 `Locating Entity`。业务方需要在自己服务中接入停写的`MiddleWare`，停写的 `MiddleWare`会根据`Locating Entity`的停写标记来判断是否要返回停写错误。

**六、发送结束及标记给对端**

设置了`Locating Entity`状态为停写以后`10s`，我们可以假设后续不会再产生`Locating Entity`的数据了，这个时候我们发送一个`Last Locating Entity Data` 给`Dst Unit`，告知对端当前的`Locating Entity`数据已经发送完成。

PS： `Locating Entity`下的所有迁移数据是需要有序发送，有序消费。

**七、Dst Unit 回写完所有数据以后 Ack**

`Dst Unit`收到了`Last Locating Entity Data`消息以后，知道这个`Locating Entity`之前的所有数据都回写到业务`DB`成功了，这个时候会回一个`Last Data Ack`给`Source Unit`

**八、修改 Locating Entity Unit 信息**

收到`Last Data Ack` 以后表示所有数据对端都回写成功了，我们可以修改`Locating Entity`的`Unit`信息为`Dst Unit`。

**九、关闭停写**

等待所有服务`Unit`信息同步完成以后，然后关闭`Locating Entity`的停写标记。至此一个`Locating Entity`的迁移过程就完成了。

后续所有`Locating Entity`的读写操作都会写到新的`Unit`


![image.png](https://upload-images.jianshu.io/upload_images/12321605-5f41d0eb4ed6df22.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/12321605-9327a6583a577818.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### 4.2.1 停写+全量同步

改方案相当于上面方案的精简版，主要流程就是 `停写` -> `数据迁移` -> `修改Unit` -> `回复停写` -> `结束`。

PS：该方案适用于数据量不大场景，数据量大的场景，会导致停写时间过长，对用户体验不友好。


#### 4.2.3 双写方案

双写方案，就是 `存量数据迁移` -> `增量数据迁移` -> `实时同步 binlog 数据` -> `修改Unit` -> `结束`
 
双写的优点是，不用停写，业务对数据迁移过程无感知，存量数据同步完成以后，可以直接修改Locating Entity 的 Unit 信息，实现无缝切换。

双写的缺点是，数据可能会有一致性的问题。不太适用于对数据有强一致要求的业务。

### 4.3 跨Unit数据传输通道

这个是用的基建提供的一个`Mirror`服务。

`Mirror`是一个分布式的消息同步服务，目前支持`kafka→kafka`、`rmq->rmq`之间的数据同步。`Mirror`集群部署在目标端，跨`region`消费数据后再写入同`region`相应的消息队列中。

简单说，就是在`Source Unit`写入一个 `MQ Message`，`Mirror`会自动把这个数据同步到`Dst Unit`。可以在`Dst Unit`直接消费这个消息。

### 4.4 Binloger 模块

`binloger` 模块，主要负责`binlog`生成和回写。

![image.png](https://upload-images.jianshu.io/upload_images/12321605-4fc574f723cc4c4e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 4.5 MySQL 数据迁移时区问题

因为国内和国外时区不一样，MySQL库里面时间都是字符串格式存储，所以传输过程中可能有些问题，详见 [MySQL DateTime和Timestamp时区问题](https://fanlv.wiki/2021/11/28/mysql-time/)。


## 五、架构总览

![image.png](https://upload-images.jianshu.io/upload_images/12321605-d61d2a70c79726c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 六、名词解释

* `Unit`：一个功能自封闭的部署单元，可以为用户提供完整的产品功能。`Unit`之间的数据存储是隔离的。一个`Unit`可以包含多个`IDC`。在本文上下文中可以假设有`CN`和`i18n`（国际化）两个`Unit`。
* `Tenant`：租户，可以认为一个公司就是一个租户。
* `Global Meta`：记录所有实体（`Entity`）归属的一个服务吗，还会记录实体一些其他`Meta`信息，比如`是否处于停写状态`。
* `EdgeProxy` ：边缘代理，负责转发`Unit`之间的请求。
* `Binlog`：描述数据实体内容的数据结构，当前上下文中就是指的一个`PB`的`Struct`。



