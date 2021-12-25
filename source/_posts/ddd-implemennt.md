---
title: DDD在Message服务的实践
tags:
  - Architecture
  - Note
categories:
  - Golang
date: 2021-06-25 21:23:15
updated: 2021-11-25 21:23:15
---

## 一、背景

我们`message`服务经过`4`年的迭代，项目的可维护性、代码的可读性、功能的可扩展性越来越差，经常改一个小功能牵一发而动全身，新来的同学对`message`服务不能快速上手，而且因为代码可读性不高，导致新同学修代码经常会出现`bug`。还有一部分原因是大多数人都是第一次用`Golang`，对项目的分层也没有一个统一的规范，导致跌倒最后成了一个四不像的框架。

然后我们今年的两个主要目标，一是项目的稳定性、二是为了支持`KA`私有划部署战略，我们需要合并微服务数量，降低运维和部署成本。


基于上背景，老板让我牵头对`Message`服务基于`DDD`做一次大规模重构，重构目标很明确：

1. 对项目有个明确的分层，提高项目代码可读性、可维护性、可扩展性。降低新人上手成本。
2. 升级`RPC`框架，合并`Message`相关服务，提高优化项目资源利用率。
3. 合理分层，方便后续扩展，比如做`存储`的高可用。
4. 产出一个最佳实践规范，作为整个Messenger项目指导规范。


## 二、架构方案

![image.png](https://upload-images.jianshu.io/upload_images/12321605-2cc2ee62447df8af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/12321605-1c2ee27646713bac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/12321605-ad25686d0d9d878f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 三、应用代码分层

### 3.1 基本架构思想

目前使用的是`整洁架构/洋葱架构`的思想，领域层是项目核心，里面存放业务的核心逻辑。

1. 对上`Domain Service`抽象自己的对外的提供的能力，定义好接口给`Application`层服务调用。
2. 对下`Domain Service`抽象依赖的接口，通过依赖反转方式让底层的`Infrastructure`去实现相关逻辑。
3. 注意内层对象的不能依赖外层对象。`Domain Service`之间也不能互相调用。

![image](https://upload-images.jianshu.io/upload_images/12321605-c63c554e4115bb48?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/12321605-a90b40c0686d827f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 3.2 项目分层

我们可以把 `Message`、`Read`、`Pin`、`Urgent`、`Reaction`几个模块看成`Message`服务域下的一个个聚合，聚合和聚合相互独立，聚合之间可以通过`EventBus`来相互通信，或者通过`Local Call`的方式来调用。

![image.png](https://upload-images.jianshu.io/upload_images/12321605-ef7a9c26e3aaf8bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/12321605-5b40b3f135527bbd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


####  Interfaces（用户接口层）

这一层主要做数据解析操作，如果是`RPC`服务，数据解析逻辑一般`RPC`框架都已经给我们做好了，没啥好说的，`Http`，这一层主要解析请求数据，然后转`DTO`。

#### Application（应用层）

1. 权限校验、参数校验。
2. 发送或订阅领域事件等。
3. `Domain Obejct` 组装（简单接口可以不用DO）
4. `Domain Service` 服务的组合、编排。 
5. `Response`的组装返回。

#### Domain（领域层）

`Domain`层，主要包括`Domain Service`和`Domain Object`。在基于充血模型的`DDD`开发模式中，大部分业务逻辑都在`Domain Object`中，`Domain Service`类变得很薄。

`Domain Service`类主要有下面这样几个职责：

1. `Domain Service`类负责与`Repository`交流。之所以让`Domain Service`与`Repository`打交道，而不是让领域模型与`Repository`打交道，那是因为我们想**保持领域模型的独立性**，不与任何其他层的代码（`Repository` 层的代码）或开发框架耦合在一起，将流程性的代码逻辑（比如从`DB`中取数据、映射数据）与领域模型的业务逻辑解耦，让领域模型更加可复用。
2. `Domain Service`负责一些非功能性及与三方系统交互的工作。比如幂等、事务、调用其他系统的`RPC`接口等，都可以放到`Service`类中。
3. `Domain Service`类负责跨领域模型的业务聚合功能。
4. `Domain Service` 应该相互独立，`Domain Service`里面不能直接调用其他`Domain Service`的服务，比如在`pushDomainService`调用`NewPackDomainService().Pack()`。 
5. `Domain Object`，充血模型，里面存放一些核心逻辑，比如`Pack`时候，`Domain Object`中会有`Thrift Message` 转 `Protobuf Message` 数据组装的若干逻辑。

#### Infrastructure（基础层）

基础层贯穿所有层，为各层提供基础资源服务

1. `Infrastructure/Repository` 提供存储相关能力。
2. `Infrastructure/Service` 提供比如消息加解密、`Risk`检查、部分带`LocalCache` 或者`CtxCacheRPC`服务接口。
3. `Infrastructure/RPC` 提供RPC能力，这个里面只做简单的`RPC`调用，参数解析返回，如果有`LocalCache`相关，需要封装到`Infrastructure/Service`中去（理论上`Domain`层不应该直接依赖`Infrastructure/RPC`包，都需要抽象为接口封装为`Infrastructure/Service`，考虑到`IDL`一般都是向前兼容的，为了开发方便所以`Application`层和`Domain`层都直接依赖了`RPC`包）。
4. `Infrastructure/Pkg` 提供一些基础工具包，比如`fg`、`metrics`、`idgen`、`errror`包相关的，这个包理论上不能有具体的业务逻辑。

### 3.3 关于各层之间数据传递

`DO`、`PO`、`VO` 存在的意义是什么？

1. 为了尽量减少每层之间的耦合，把职责边界划分明确，每层都会维护自己的数据对象，层与层之间通过接口交互。数据从下一层传递到上一层的时候，将下一层的数据对象转化成上一层的数据对象，再继续处理。虽然这样的设计稍微有些繁琐，每层都需要定义各自的数据对象，需要做数据对象之间的转化，但是分层清晰。对于非常大的项目来说，结构清晰是第一位的！
2. `DO`、`PO`、`VO` 并非完全一样。每一层各个对象还是有一些区别。
3. `DO`、`PO`、`VO`三个类虽然代码重复，但功能语义不重复，从职责上讲是不一样的。所以，也并不能算违背`DRY`原则。

不同分层之间的数据对象该如何互相转化呢？

当下一层的数据通过接口调用传递到上一层之后，我们需要将它转化成上一层对应的数据对象类型。比如，`Domain` 层从 `Repository`层获取的`Entity`之后，将其转化成`DTO`，再继续业务逻辑的处理。
具体可以参考 https://time.geekbang.org/column/article/183007
1. `IDL`生成的 `RPC Request/Response` 对象我们可以认为是`DTO`对象。
2. `Redis` 持久化的对象比如`pbpersistent.PersistentMessage`需要收敛在`Infrastructure`层，`Domain`和`Application`不应该感知到这个类型。
3. `PO`对象也不应该让`Domain`和`Application` 层感知到。比如 `MessageEntity`实体。
4. 不能通过`context`隐式传递自定义的参数，所有的数据传递必须显示传递。
5. `context`中的公共参数只能在`Application`层获取，到`Domain`层和`Infrastructure`层必须显示传递。
6. 不想重复定义`Domain Obeject` ，可以通过组合的方式组合`DTO`对象（包括自定义`DTO`对象和`Thrift`生成的对象）
7. 如果一个对象被多层用到了，这个对象可以放到`types/dto`文件夹下。注意`types/dto`应该都是贫血对象，理论上只能依赖`idl_gen`这种`dto`包，不应该依赖其他层的任何包。

### 3.4 服务内聚合之间如何通信

#### 3.4.1 基于事件

![image.png](https://upload-images.jianshu.io/upload_images/12321605-f7d32a97fa93b293.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 3.4.2 Local Call

`kitex`服务可以直接使用`kitex`生成的`Interface`，`kite`服务可以自己定义各个聚合需要暴露的`Interface`。`Localcall`只应该依赖`kitex/kite`生成的结构 。

优点：后续要把某个聚合再拆出去以后，只需要把`Localcall`包名改成`RPC`即可，不需要修改其他逻辑。

	
	package localcall
	
	import (
	   "context"
	
	   "git.byted.org/ee/go/kitex_gen/lark/im/message"
	)
	
	var localSVC message.MessageService
	
	func InitLocalSVC(svc message.MessageService) {
	   localSVC = svc
	}
	
	func Pack(ctx context.Context, request *message.PackRequest) (r *message.PackResponse, err error) {
	   return localSVC.Pack(ctx, request)
	}
	
	// 在main初始化
	func main() {
	   ....
	   localcall.InitLocalSVCImpl(new(MessageServiceImpl))
	   ...
	}
	  
## 四、编码规范

### 4.1 基本原则

1. 单一职责（`SRP`）：`Single Responsibility Principle`，一个类只负责完成一个职责或者功能。不要设计大而全的类，要设计粒度小、功能单一的类。单一职责原则是为了实现代码高内聚、低耦合，提高代码的复用性、可读性、可维护性。

2. 开闭原则（`OCP`）：`Open Closed Principle`，对扩展开放，对修改关闭。添加一个新的功能，应该是通过在已有代码基础上扩展代码（新增模块、类、方法、属性等），而非修改已有代码（修改模块、类、方法、属性等）的方式来完成。

3. 里式替换（`LSP`）：`Liskov Substitution Principle` 子类对象（`object of subtype/derived class`）能够替换程序（`program`）中父类对象（`object of base/parent class`）出现的任何地方，并且保证原来程序的逻辑行为（`behavior`）不变及正确性不被破坏。举例： 是拿父类的单元测试去验证子类的代码。如果某些单元测试运行失败，就有可能说明，子类的设计实现没有完全地遵守父类的约定，子类有可能违背了里式替换原则。

4. 接口隔离原则（`ISP`）：`Interface Segregation Principle` 调用方不应该被强迫依赖它不需要的接口。

5. 依赖反转原则（`DIP`）： `Dependency Inversion Principle` 高层模块（`high-level modules`）不要依赖低层模块（`low-level`）。高层模块和低层模块应该通过抽象（`abstractions`）来互相依赖。除此之外，抽象（`abstractions`）不要依赖具体实现细节（`details`），具体实现细节（`details`）依赖抽象（`abstractions`）。举例 `Domain` 层不依赖 `Infrastructure` 层具体实现，只依赖`Domain`自己抽象的 `Interface`


### 4.2 CodeReview 标准

1. 可维护性（`maintainability`），落实到编码开发，所谓的“维护”无外乎就是修改 bug、修改老的代码、添加新的代码之类的工作。所谓“代码易维护”就是指，在不破坏原有代码设计、不引入新的 `bug` 的情况下，能够快速地修改或者添加代码。所谓“代码不易维护”就是指，修改或者添加代码需要冒着极大的引入新 `bug` 的风险，并且需要花费很长的时间才能完成。
2. 可读性（`readability`），我们在编写代码的时候，时刻要考虑到代码是否易读、易理解。除此之外，代码的可读性在非常大程度上会影响代码的可维护性。毕竟，不管是修改 bug，还是修改添加功能代码，我们首先要做的事情就是读懂代码。代码读不大懂，就很有可能因为考虑不周全，而引入新的 bug。我们需要看代码是否符合编码规范、命名是否达意、注释是否详尽、函数是否长短合适、模块划分是否清晰、是否符合高内聚低耦合等等。你应该也能感觉到，从正面上，我们很难给出一个覆盖所有评价指标的列表。这也是我们无法量化可读性的原因。
3. 可扩展性（`extensibility`），代码的可扩展性表示，我们在不修改或少量修改原有代码的情况下，通过扩展的方式添加新的功能代码。说直白点就是，代码预留了一些功能扩展点，你可以把新功能代码，直接插到扩展点上，而不需要因为要添加一个功能而大动干戈，改动大量的原始代码。
4. 灵活性（`flexibility`），从刚刚举的场景来看，如果一段代码易扩展、易复用或者易用，我们都可以称这段代码写得比较灵活。所以，灵活这个词的含义非常宽泛，很多场景下都可以使用。
5. 简洁性（`simplicity`），有一条非常著名的设计原则，你一定听过，那就是 `KISS` 原则：“`Keep It Simple，Stupid`”。这个原则说的意思就是，尽量保持代码简单。代码简单、逻辑清晰，也就意味着易读、易维护。我们在编写代码的时候，往往也会把简单、清晰放到首位。不过，很多编程经验不足的程序员会觉得，简单的代码没有技术含量，喜欢在项目中引入一些复杂的设计模式，觉得这样才能体现自己的技术水平。实际上，思从深而行从简，真正的高手能云淡风轻地用最简单的方法解决最复杂的问题。这也是一个编程老手跟编程新手的本质区别之一。
6. 可复用性，代码的可复用性可以简单地理解为，尽量减少重复代码的编写，复用已有的代码。
7. 可测试性，相对于前面六个评价标准，代码的可测试性是一个相对较少被提及，但又非常重要的代码质量评价标准。代码可测试性的好坏，能从侧面上非常准确地反应代码质量的好坏。代码的可测试性差，比较难写单元测试，那基本上就能说明代码设计得有问题。

### 4.3 关于重构

总结一下重构的做法，其实就是“分段实施”，将要解决的问题根据优先级、重要性、实施难度等划分为不同的阶段，每个阶段聚焦于一个整体的目标，集中精力和资源解决一类问题。
这样做有几个好处：

1. 每个阶段都有明确目标，做完之后效果明显，团队信心足，后续推进更加容易。
2. 每个阶段的工作量不会太大，可以和业务并行。
3. 每个阶段的改动不会太大，降低了总体风险。

**优先级排序 -> 问题分类 -> 先易后难 -> 循序渐进**

**重构不是简单功能搬运，把老的项目功能迁移到新的项目就完事了，这样的重构没有任何收益。重构的时候更多的是需要考虑、可读性、可维护性、可扩展性等几个方面。**

需求开发的时候可能工期紧，或者没想好怎么实现相关功能。后续可以多思考有没有更好的方式去实现相关功能，持续优化老的功能，让项目的代码质量越来越高。


### 4.4 函数

1. 尽量最短小 `golint` 行数超过60行会有警告。
	- `BadCase` ： 一个函数400-500行，变量声明就10~20行。给阅读者极大的心理负担，需要花大量时间去理解。
2. 函数应该做一件事（单一原则）, 函数应该做一件事。做好这件事。只做这一件事。
3. 函数参数,函数请求参数和返回参数(除去`context`)，理论上不应该超过`3`个，超过三个可以考虑封装为类。如果参数过多，可以尝试用那个下面几种方式解决：
	1. 首先可以考虑是不是函数过于复杂，一个大函数能否拆成几个小函数。
	2. 如果函数不能拆，比如`new`一个对象的需要传递很多参数，这种情况可以考虑使用“构建者模式”来重构。
	3. 将参数封装为对象的方式，来处理参数过多的情况
4. 关于标示参数
	![image.png](https://upload-images.jianshu.io/upload_images/12321605-683cb911a4e6421e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
5. `If/else Switch`逻辑过于复杂
	* `BadCase `: 函数过长，每个分支逻辑过多。
	* `GoodCase`：用多态特性，把相关逻辑封装到各自的类中去

### 4.5 注释

注释不能美化糟糕的代码，写注释的时候，可以多想下能否用代码表达清楚，能用函数或变量时表达清楚，就别用注释。
注释一定要跟着代码变动一起修改，不然不如不写注释。

####  4.5.1 坏注释场景

1. 喃喃自语，如果只是因为你觉得应该或者因为过程需要就添加注释，那就是无谓之举。如果你决定写注释，就要花必要的时间确保写出最好的注释。
2. 多余的注释，读这段注释花的时间没准比读代码花的时间还要长。
3. 误导性注释，尽管初衷可嘉，程序员还是会写出不够精确的注释。
4. 废话注释，有时，你会看到纯然是废话的注释。它们对于显然之事喋喋不休，毫无新意。
5. 位置标记，有时，程序员喜欢在源代码中标记某个特别位置。例如，最近我在程序中看到这样一行：// Actions //////////////////////////////////把特定函数趸放在这种标记栏下面，多数时候实属无理。鸡零狗碎，理当删除—特别是尾部那一长串无用的斜杠。
6. 注释掉的代码，直接把代码注释掉是讨厌的做法。别这么干！其他人不敢删除注释掉的代码。他们会想，代码依然放在那儿，一定有其原因，而且这段代码很重要，不能删除。注释掉的代码堆积在一起，就像破酒瓶底的渣滓一般。
7. 短函数不需要太多描述。为只做一件事的短函数选个好名字，通常要比写函数头注释要好。
8. 代码里面的 长期没有清理掉的TODO 

####  4.5.2 好的注释场景

1. 用注释来提供基本信息也有其用处。
2. 对意图的解释,注释不仅提供了有关实现的有用信息，而且还提供了某个决定后面的意图。
3. 阐释, 注释把某些晦涩难明的参数或返回值的意义翻译为某种可读形式，也会是有用的。通常，更好的方法是尽量让参数或返回值自身就足够清楚；但如果参数或返回值是某个标准库的一部分，或是你不能修改的代码，帮助阐释其含义的代码就会有用。
4. 警示， 用于警告其他程序员会出现某种后果的注释也是有用的。
5. TODO 注释，有时，有理由用//TODO 形式在源代码中放置要做的工作列表。
6. 放大，注释可以用来放大某种看来不合理之物的重要性。

### 4.6 错误和日志

1. 所有错误`new`、`wrap`、收敛到`errno`包
2. 返回错误的场景，不需要打印错误，直接`wrap`一下`err`返回就行了，在中间件里面统一打`Error`。
3. 不返回错误的场景，吞掉的`errror` 可以打印一条`warn`日志。

### 4.7 关于命名

#### 4.7.1 名副其实
名副其实说起来简单。我们想要强调，这事很严肃。选个好名字要花时间，但省下来的时间比花掉的多。**注意命名，而且一旦发现有更好的名称，就换掉旧的**。这么做，读你代码的人（包括你自己）都会更开心。

变量、函数或类的名称应该已经答复了所有的大问题。它该告诉你，它为什么会存在，它做什么事，应该怎么用。如果名称需要注释来补充，那就不算是名副其实。

#### 4.7.2 名称长短

名称长短应与其作用域大小相对应。比如for循环的i，简单明了。如果名称作用域比较大，不推荐使用缩写。

#### 4.7.3 关于Constants类
1.  `Constants` 类拆解为功能更加单一的多个类，比如跟 `MySQL` 配置相关的常量，我们放到 `MysqlConstants` 类中；跟 `Redis` 配置相关的常量，我们放到 `RedisConstants` 类中。`FG`的配置放到`FG`包中、`Metric`点位放`Metric`包中。
2. 剩下的多个地方用到了，就放`Constants`包中。


#### 4.7.4 关于Utils类

1. 在`Infrastructure/utils`中添加函数的时候，需要想下能不能拆到`Infrastructure/pkg` 或者其他地方去，实在不知道放哪，再放`Infrastructure/utils`。
![image.png](https://upload-images.jianshu.io/upload_images/12321605-1e6ec6fff53d0e5e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 4.7.5 关于Common、Helper

1. 尽量不要用 common 和 helper 命名文件和类，不然最后这个类、文件会变成一个大杂烩，大家什么都往这里面放。

[avoid-package-names-like-base-util-or-common](https://dave.cheney.net/2019/01/08/avoid-package-names-like-base-util-or-common)

### 4.8  关于提交 MergeRequest

为方便进行代码的`Review`，一次提交尽量不要超过`200`行（除过完整的业务功能模块）

### 4.9 可测试性

代码可测试性的好坏，能从侧面上非常准确地反应代码质量的好坏。代码的可测试性差，比较难写单元测试，那基本上就能说明代码设计得有问题。

单元测试是保证服务稳定性的重要手段之一，时间允许的情况下，开发时间和编写单测时间应该可以达到1:1。在时间排期紧张的情况下，优先保证核心逻辑的测试覆盖率，`message`项目约定新的`Feature`开发，每次代码合入测试覆盖率不能低于 `60%`。

重构后的服务，必须要保证每个接口都是有回归测试`case`的，这个作为接口灰度的一个卡点交付物，没有回测`case`，就不允许接口灰度上线。


## 五、参考资料

[《整洁架构》 ](https://book.douban.com/subject/30333919/)

[《整洁代码》 ](https://book.douban.com/subject/5442024/)

[DDD 实战课](https://time.geekbang.org/column/intro/100037301)

[《设计模式之美》 ](https://time.geekbang.org/column/article/183007)
