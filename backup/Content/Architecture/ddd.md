- [一、DDD的基础概念](#一ddd的基础概念)
  - [1.1 什么是 DDD](#11-什么是-ddd)
  - [1.2 DDD 主要解决什么问题](#12-ddd-主要解决什么问题)
  - [1.3 DDD 与微服务的关系](#13-ddd-与微服务的关系)
  - [1.4 领域、核心域、通用域、支撑域](#14-领域核心域通用域支撑域)
  - [1.5 通用语言和限界上下文](#15-通用语言和限界上下文)
  - [1.6 **防腐层（Anti-corruption layer）**](#16-防腐层anti-corruption-layer)
  - [1.7 贫血模型、充血模型](#17-贫血模型充血模型)
  - [1.8 实体和对象](#18-实体和对象)
  - [1.9 聚合和聚合根](#19-聚合和聚合根)
  - [1.10 领域事件](#110-领域事件)
  - [1.11 PO、DO、DTO、VO](#111-pododtovo)
  - [1.12 CQRS](#112-cqrs)
  - [1.13 DDD的战略设计和战术设计](#113-ddd的战略设计和战术设计)
  - [1.14 从DDD设计到代码落地的大致流程](#114-从ddd设计到代码落地的大致流程)
  - [1.15 如何划定领域模型和微服务的边界](#115-如何划定领域模型和微服务的边界)
- [二、DDD 战略设计流程](#二ddd-战略设计流程)
  - [2.1 产品愿景](#21-产品愿景)
  - [2.2 场景分析](#22-场景分析)
  - [2.3 领域建模](#23-领域建模)
    - [**第一步：找出实体和值对象等领域对象**](#第一步找出实体和值对象等领域对象)
    - [第二步：定义聚合](#第二步定义聚合)
    - [第三步：定义限界上下文](#第三步定义限界上下文)
  - [2.4 微服务的拆分](#24-微服务的拆分)
- [三、DDD的战术设计流程](#三ddd的战术设计流程)
  - [3.1 分析微服务领域对象](#31-分析微服务领域对象)
  - [3.2 DDD 微服务的四层架构](#32-ddd-微服务的四层架构)
    - [**Interfaces（用户接口层）**](#interfaces用户接口层)
    - [**Application（应用层）**](#application应用层)
    - [**Domain（领域层）**](#domain领域层)
    - [**Infrastructure（基础层）**](#infrastructure基础层)
- [四、DDD常见的一些误区](#四ddd常见的一些误区)
  - [4.1 所有的领域都用 DDD](#41-所有的领域都用-ddd)
  - [4.2 全部采用 DDD 战术设计方法](#42-全部采用-ddd-战术设计方法)
  - [4.3 重战术设计而轻战略设计](#43-重战术设计而轻战略设计)
- [参考资料](#参考资料)

## 一、DDD的基础概念

### 1.1 什么是 DDD

> 2004 年埃里克·埃文斯（Eric Evans）发表了《领域驱动设计》（Domain-Driven Design –Tackling Complexity in the Heart of Software）这本书，从此领域驱动设计（Domain Driven Design，简称 DDD）诞生。**DDD 核心思想是通过领域驱动设计方法定义领域模型，从而确定业务和应用边界，保证业务模型与代码模型的一致性**。
> 领域驱动设计，主要是用来指导**如何解耦业务系统，划分业务模块**，定义业务领域模型及其交互。领域驱动设计这个概念并不新颖，早在 2004 年就被提出了，到现在已经有十几年的历史了。不过，它被大众熟知，还是基于另一个概念的兴起，那就是微服务。
> 不过，我个人觉得，领域驱动设计有点儿类似敏捷开发、SOA、PAAS 等概念，听起来很高大上，但实际上只值“五分钱”。即便你没有听说过领域驱动设计，对这个概念一无所知，只要你是在开发业务系统，也或多或少都在使用它。做好领域驱动设计的关键是，看你对自己所做业务的熟悉程度，而并不是对领域驱动设计这个概念本身的掌握程度。即便你对领域驱动搞得再清楚，但是对业务不熟悉，也并不一定能做出合理的领域设计。所以，**不要把领域驱动设计当银弹，不要花太多的时间去过度地研究它。**

引自：https://time.geekbang.org/column/article/169600

### 1.2 DDD 主要解决什么问题

DDD主要解决微服务边界划分困难的问题。

### 1.3 DDD 与微服务的关系

> DDD 是一种**架构设计方法/思想**，微服务是一种架构风格，两者从本质上都是为了追求高响应力，而从业务视角去分离应用系统建设复杂度的手段。两者都强调从业务出发，其核心要义是强调根据业务发展，合理划分领域边界，持续调整现有架构，优化现有代码，以保持架构和代码的生命力，也就是我们常说的演进式架构。
> **DDD 主要关注**：从业务领域视角划分领域边界，构建通用语言进行高效沟通，通过业务抽象，建立领域模型，维持业务和代码的逻辑一致性。
> **微服务主要关注**：运行时的进程间通信、容错和故障隔离，实现去中心化数据管理和去中心化服务治理，关注微服务的独立开发、测试、构建和部署。

### 1.4 领域、核心域、通用域、支撑域

> 领域就是用来确定范围的，范围即边界，这也是 DDD 在设计中不断强调边界的原因。
> 在领域不断划分的过程中，领域会细分为不同的子域，子域可以根据自身重要性和功能属性划分为三类子域，它们分别是：核心域、通用域和支撑域。
> 核心域：产品的核心的模块，能够给产品提供核心竞争力。
> 通用域：有一定通用性，比如认证、权限、邮件发送服务，不是业务的核心，但是没有他们业务也无法运转。
> 支撑域：则具有企业特性，但不具有通用性，例如数据代码类的数据字典等系统

![image](https://upload-images.jianshu.io/upload_images/12321605-0023cee42b88a41a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image](https://upload-images.jianshu.io/upload_images/12321605-40cc56f2d8c55f61?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1.5 通用语言和限界上下文

> 在事件风暴过程中，**通过团队交流达成共识的，能够简单、清晰、准确描述业务涵义和规则的语言就是通用语言**。也就是说，通用语言是团队统一的语言，不管你在团队中承担什么角色，在同一个领域的软件生命周期里都**使用统一的语言进行交流**。
> 我们可以将限界上下文拆解为两个词：限界和上下文。限界就是领域的边界，而上下文则是语义环境。通过领域的限界上下文，我们就可以在统一的领域边界内用统一的语言进行交流。
> **理论上限界上下文就是微服务的边界**。我们将限界上下文内的领域模型映射到微服务，就完成了从问题域到软件的解决方案。
![image](https://upload-images.jianshu.io/upload_images/12321605-94dda6eba5345bb7?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1.6 **防腐层（Anti-corruption layer）**

> 大多数应用程序依赖于其他系统的某些数据或功能**。** 例如，旧版应用程序迁移到新式系统时，可能仍需要现有的旧的资源。 新功能必须能够调用旧系统。 逐步迁移尤其如此，随着时间推移，较大型应用程序的不同功能迁移到新式系统中。
> 这些旧系统通常会出现质量问题，如复杂的数据架构或过时的 API。 旧系统使用的功能和技术可能与新式系统中的功能和技术有很大差异。 若要与旧系统进行互操作，新应用程序可能需要支持过时的基础结构、协议、数据模型、API、或其他不会引入新式应用程序的功能。
> 保持新旧系统之间的访问可以**强制新系统至少支持某些旧系统的 API 或其他语义**。 这些旧的功能出现质量问题时，支持它们“损坏”可能会是完全设计的新式应用程序。
> 不仅仅是旧系统，不受开发团队控制的**任何外部系统(第三方系统)**都可能出现类似的问题。
> **解决方案**
> **在不同的子系统之间放置防损层以将其隔离**。 此层转换两个系统之间的通信，在一个系统保持不变的情况下，使另一个系统可以避免破坏其设计和技术方法。

![image](https://upload-images.jianshu.io/upload_images/12321605-6ec64f901f97f1cb?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1.7 贫血模型、充血模型

> 贫血模型（Anemic Domain Model）：像 VirtualWalletBo 这样，只包含数据，不包含业务逻辑的类，就叫作贫血模型。在贫血模型中，数据和业务逻辑被分割到不同的类中， 例如 MVC架构。

```
public class VirtualWalletBo {
  private Long id;
  private Long createTime;
  private BigDecimal balance;
}

```

> 充血模型（Rich Domain Model）：充血模型正好相反，数据和对应的业务逻辑被封装到同一个类中。因此，这种充血模型满足面向对象的封装特性，是典型的面向对象编程风格，例如DDD架构，领域模型。

```
public class VirtualWallet { // Domain领域模型(充血模型)
  private Long id;
  private Long createTime = System.currentTimeMillis();;
  private BigDecimal balance = BigDecimal.ZERO;

  public void debit(BigDecimal amount) {
    if (this.balance.compareTo(amount) < 0) {
      throw new InsufficientBalanceException(...);
    }
    this.balance = this.balance.subtract(amount);
  }

  public void credit(BigDecimal amount) {
    if (amount.compareTo(BigDecimal.ZERO) < 0) {
      throw new InvalidAmountException(...);
    }
    this.balance = this.balance.add(amount);
  }
}

```

### 1.8 实体和对象

**实体**

> 在 DDD 不同的设计过程中，实体的形态是不同的。在战略设计时，实体是领域模型的一个重要对象。领域模型中的实体是多个属性、操作或行为的载体。在事件风暴中，我们可以根据命令、操作或者事件，找出产生这些行为的业务实体对象，进而按照一定的业务规则将依存度高和业务关联紧密的多个实体对象和值对象进行聚类，形成聚合。你可以这么理解，实体和值对象是组成领域模型的基础单元。
> 在代码模型中，实体的表现形式是实体类，这个类包含了实体的属性和方法，通过这些方法实现实体自身的业务逻辑。在 DDD 里，**这些实体类通常采用充血模型**，与这个实体相关的所有业务逻辑都在实体类的方法中实现，跨多个实体的领域逻辑则在领域服务中实现。

**值对象**

> 值对象是 DDD 领域模型中的一个基础对象，它跟实体一样都来源于事件风暴所构建的领域模型，都包含了若干个属性，它与实体一起构成聚合。
> 当一个概念缺乏明显身份时，就基本可以断定它是模型中的一个值对象。比如Content、Extra就是一个值对象，值对象一般都是贫血对象。

```
type Message struct {
        ID                    int64                              
        Type                  MessageConstants.MessageType 
        CreateTime            int64                              
        FromID                *int64                             
        Content               *Content                           
        Extra                 *Extra                             
        RootID                int64                              
        .......               
}

```

### 1.9 聚合和聚合根

**聚合**

> 领域模型内的实体和值对象就好比个体，**而能让实体和值对象协同工作的组织就是聚合**，它用来确保这些领域对象在实现共同的业务逻辑时，能保证数据的一致性。
> 聚合就是由业务和逻辑紧密关联的实体和值对象组合而成的，**聚合是数据修改和持久化的基本单元，每一个聚合对应一个仓储，实现数据的持久化。**
> 聚合在 DDD 分层架构里属于领域层，领域层包含了多个聚合，共同实现核心业务逻辑。聚合内实体以充血模型实现个体业务能力，以及业务逻辑的高内聚。跨多个实体的业务逻辑通过领域服务来实现，跨多个聚合的业务逻辑通过应用服务来实现。比如有的业务场景需要同一个聚合的 A 和 B 两个实体来共同完成，我们就可以将这段业务逻辑用领域服务来实现；而有的业务逻辑需要聚合 C 和聚合 D 中的两个服务共同完成，这时你就可以用应用服务来组合这两个服务。
> **聚合的特点：高内聚、低耦合，它是领域模型中最底层的边界，可以作为拆分微服务的最小单位。**

![image](https://upload-images.jianshu.io/upload_images/12321605-531efebb998de361?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**聚合根**

> 如果把聚合比作组织，那聚合根就是这个组织的负责人。聚合根也称为根实体，**它不仅是实体，还是聚合的管理者**。

### 1.10 领域事件

**微服务内的领域事件**

> 当领域事件发生在微服务内的聚合之间，领域事件发生后完成事件实体构建和事件数据持久化，发布方聚合将事件发布到事件总线，订阅方接收事件数据完成后续业务操作。

**微服务之间的领域事件**

> 跨微服务的领域事件会在不同的限界上下文或领域模型之间实现业务协作，其主要目的是实现微服务解耦，减轻微服务之间实时服务访问的压力。

![image](https://upload-images.jianshu.io/upload_images/12321605-d9ca14e82fa79c5c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1.11 PO、DO、DTO、VO

> PO：数据持久化对象 (Persistent Object， PO)，与数据库结构一一映射，它是数据持久化过程中的数据载体。
> DO：领域对象（ Domain Object， DO），微服务运行时核心业务对象的载体， DO 一般包括实体或值对象。
> DTO：数据传输对象（ Data Transfer Object， DTO），用于前端应用与微服务应用层或者微服务之间的数据组装和传输，是应用之间数据传输的载体。
> VO：视图对象（View Object， VO），用于封装展示层指定页面或组件的数据。
> DTO 是用于数据传输的对象，我们可以把kite生成的Request、Response当做DTO对象，在Http服务中可以把用户传输的Json对象作为DTO。
> VO 是视图对象，处于MVC架构的Logic层，多用于UI组件的数据的封装。

![image](https://upload-images.jianshu.io/upload_images/12321605-c2e2fdaeffd4b619?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1.12 CQRS

> CQRS — Command Query Responsibility Segregation。CQRS 将系统中的操作分为两类，即「命令」(Command) 与「查询」(Query)。命令则是对会引起数据发生变化操作的总称，即我们常说的新增，更新，删除这些操作，都是命令。而查询则和字面意思一样，即不会对数据产生变化的操作，只是按照某些条件查找数据。
> CQRS 的核心思想是将这两类不同的操作进行分离，然后在两个独立的「服务」中实现。这里的「服务」一般是指两个独立部署的应用。在某些特殊情况下，也可以部署在同一个应用内的不同接口上。
> CQRS 在 DDD 中是一种常常被提及的模式，它的用途在于将领域模型与查询功能进行分离，让一些复杂的查询摆脱领域模型的限制，以更为简单的 DTO 形式展现查询结果。同时分离了不同的数据存储结构，让开发者按照查询的功能与要求更加自由的选择数据存储引擎。

### 1.13 DDD的战略设计和战术设计
 
**战略设计**

主要从业务视角出发，建立业务领域模型，划分领域边界，建立通用语言的限界上下文，限界上下文可以作为微服务设计的参考边界。

**战术设计**

则从技术视角出发，侧重于领域模型的技术实现，完成软件开发和落地，包括：聚合根、实体、值对象、领域服务、应用服务和资源库等代码逻辑的设计和实现。

### 1.14 从DDD设计到代码落地的大致流程

![image](https://upload-images.jianshu.io/upload_images/12321605-30195b6cad75f228?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

事件风暴 -> 领域故事分析 -> 提取领域对象 -> 领域对象与代码模型映射 -> 代码落地

在事件风暴的过程中，领域专家会和设计、开发人员一起建立领域模型，在领域建模的过程中会形成通用的业务术语和用户故事。

事件风暴也是一个项目团队统一语言的过程。通过用户故事分析会形成一个个的领域对象，这些领域对象对应领域模型的业务对象，每一个业务对象和领域对象都有通用的名词术语，并且一一映射。

微服务代码模型来源于领域模型，每个代码模型的代码对象跟领域对象一一对应。

### 1.15 如何划定领域模型和微服务的边界

* 第一步：在事件风暴中梳理业务过程中的用户操作、事件以及外部依赖关系等，根据这些要素梳理出领域实体等领域对象。
* 第二步：根据领域实体之间的业务关联性，将业务紧密相关的实体进行组合形成聚合，同时确定聚合中的聚合根、值对象和实体。
* 第三步：根据业务及语义边界等因素，将一个或者多个聚合划定在一个限界上下文内，形成领域模型。在这个图里，限界上下文之间的边界是第二层边界，这一层边界可能就是未来微服务的边界，不同限界上下文内的领域逻辑被隔离在不同的微服务实例中运行，物理上相互隔离，所以是物理边界，边界之间用实线来表示。

首先，领域可以拆分为多个子领域。一个领域相当于一个问题域，领域拆分为子域的过程就是大问题拆分为小问题的过程。

## 二、DDD 战略设计流程

**战略设计是根据用户旅程分析，找出领域对象和聚合根，对实体和值对象进行聚类组成聚合，划分限界上下文，建立领域模型的过程**。

战略设计采用的方法是事件风暴，包括：产品愿景、场景分析、领域建模和微服务拆分等几个主要过程。

战略设计阶段建议参与人员：领域专家、业务需求方、产品经理、架构师、项目经理、开发经理和测试经理。

假设我们项目基本信息**项目的目标**是实现在线请假和考勤管理功能描述如下：

*   请假人填写请假单提交审批，根据请假人身份、请假类型和请假天数进行校验，根据审批规则逐级递交上级审批，逐级核批通过则完成审批，否则审批不通过退回申请人。
*   根据考勤规则，核销请假数据后，对考勤数据进行校验，输出考勤统计。

### 2.1 产品愿景

产品愿景是对产品顶层价值设计，对产品目标用户、核心价值、差异化竞争点等信息达成一致，避免产品偏离方向。

事件风暴时，所有参与者针对每一个要点，在贴纸上写出自己的意见，贴到白板上。事件风暴主持者会对每个贴纸，讨论并对发散的意见进行收敛和统一，形成下面的产品愿景图。

![image](https://upload-images.jianshu.io/upload_images/12321605-c5172e8bed5f09f8?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.2 场景分析

场景分析是从用户视角出发，探索业务领域中的典型场景，产出领域中需要支撑的场景分类、用例操作以及不同子域之间的依赖关系，用以支撑领域建模。

项目团队成员一起用事件风暴分析请假和考勤的用户旅程。根据不同角色的旅程和场景分析，尽可能全面地梳理从前端操作到后端业务逻辑发生的所有操作、命令、领域事件以及外部依赖关系等信息。

![image](https://upload-images.jianshu.io/upload_images/12321605-1be5705b3210533c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.3 领域建模

领域建模是通过对业务和问题域进行分析，建立领域模型。向上通过限界上下文指导微服务边界设计，向下通过聚合指导实体对象设计。

领域建模是一个收敛的过程，分三步：

#### **第一步：找出实体和值对象等领域对象**

根据场景分析，分析并找出发起或产生这些命令或领域事件的实体和值对象，将与实体或值对象有关的命令和事件聚集到实体。

下面这个图是分析后的实体与命令的关系。通过分析，我们找到了：请假单、审批意见、审批规则、人员、组织关系、刷卡明细、考勤明细以及考勤统计等实体和值对象。

![image](https://upload-images.jianshu.io/upload_images/12321605-4f2bb757b11f09ca?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 第二步：定义聚合

定义聚合前，先找出聚合根。从上面的实体中，我们可以找出“请假单”和“人员”两个聚合根。然后找出与聚合根紧密依赖的实体和值对象。我们发现审批意见、审批规则和请假单紧密关联，组织关系和人员紧密关联。

刷卡明细、考勤明细和考勤统计这几个实体，它们之间相互独立，找不出聚合根，不是富领域模型，但它们一起完成考勤业务逻辑，具有很高的业务内聚性。我们将这几个业务关联紧密的实体，放在一个考勤聚合内。在微服务设计时，我们依然采用 DDD 的设计和分析方法。由于没有聚合根来管理聚合内的实体，我们可以用传统的方法来管理实体。

经过分析，我们建立了请假、人员组织关系和考勤三个聚合。其中请假聚合有请假单、审批意见实体和审批规则等值对象。人员组织关系聚合有人员和组织关系等实体。考勤聚合有刷卡明细、考勤明细和考勤统计等实体。

![image](https://upload-images.jianshu.io/upload_images/12321605-03e1c6c1dcd90557?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 第三步：定义限界上下文

由于人员组织关系聚合与请假聚合，共同完成请假的业务功能，两者在请假的限界上下文内。考勤聚合则单独构成考勤统计限界上下文。因此我们为业务划分请假和考勤统计两个限界上下文，建立请假和考勤两个领域模型。

### 2.4 微服务的拆分

理论上一个限界上下文就可以设计为一个微服务，但还需要综合考虑多种外部因素，比如：职责单一性、敏态与稳态业务分离、非功能性需求（如弹性伸缩、版本发布频率和安全等要求）、软件包大小、团队沟通效率和技术异构等非业务要素。

![image](https://upload-images.jianshu.io/upload_images/12321605-0c66711c35455f78?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 三、DDD的战术设计流程

则从技术视角出发，**侧重于领域模型的技术实现，完成软件开发和落地**，包括：聚合根、实体、值对象、领域服务、应用服务和资源库等代码逻辑的设计和实现。

### 3.1 分析微服务领域对象

领域模型有很多领域对象，但是这些对象带有比较重的业务属性。要完成从领域模型到微服务的落地，还需要进一步的分析和设计。在事件风暴基础上，我们进一步细化领域对象以及它们的关系，补充事件风暴可能遗漏的业务和技术细节。

我们分析微服务内应该有哪些服务？服务的分层？应用服务由哪些服务组合和编排完成？领域服务包括哪些实体和实体方法？哪个实体是聚合根？实体有哪些属性和方法？哪些对象应该设计为值对象等。

**服务的识别和设计**

事件风暴的命令是外部的一些操作和业务行为，也是微服务对外提供的能力。它往往与微服务的应用服务或者领域服务对应。我们可以将命令作为服务识别和设计的起点。具体步骤如下：

*   根据命令设计应用服务，确定应用服务的功能，服务集合，组合和编排方式。服务集合中的服务包括领域服务或其它微服务的应用服务。
*   根据应用服务功能要求设计领域服务，定义领域服务。这里需要注意：应用服务可能是由多个聚合的领域服务组合而成的。
*   根据领域服务的功能，确定领域服务内的实体以及功能。
*   设计实体基本属性和方法。

### 3.2 DDD 微服务的四层架构

![image](https://upload-images.jianshu.io/upload_images/12321605-9039d51b27a23a73?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### **Interfaces（用户接口层）**

> 它主要存放**用户接口层与前端交互、展现数据相关的代码**。前端应用通过这一层的接口，向应用服务获取展现所需的数据。这一层主要用来处理用户发送的 RESTful 请求，解析用户输入的配置文件，并将数据传递给 Application 层。数据的组装、数据传输格式以及 Facade 接口等代码都会放在这一层目录里。

这一层主要对应kite生成的handle.go，handle会调用Application Service接口。

```
func (s *MessageServiceImpl) PushMessage(ctx context.Context, request *message.PushMessageRequest) (*message.PushMessageResponse, error) {
   return service.PushMessage(ctx, request)
}

```

#### **Application（应用层）**

> 它主要存放应用层**服务组合和编排相关**的代码。应用服务向下基于微服务内的领域服务或外部微服务的应用服务完成服务的编排和组合，向上为用户接口层提供各种应用数据展现支持服务。应用服务和事件等代码会放在这一层目录里。

Application层应该是很薄的一层，实现服务组合和编排，适应业务流程快速变化的需求。

```
func PushMessage(ctx context.Context, request *message.PushMessageRequest) (*message.PushMessageResponse, error) {
   if request.Message == nil {
      return nil, errors.New("message is nil")
   } else if request.Message.ChannelID == nil {
      return nil, errors.New("message channel id is nil")
   }

   pushMessageDO := assembler.PushMsgDTO.ToDO(request)
   err := DomainService.NewPushDomainService().PushMessage(ctx, pushMessageDO)
   if err != nil {
         log.WithError(err).Errorf("Push.PushMessage fail")
         return nil, err
   }

   return resp, nil
}

```

#### **Domain（领域层）**

> 它主要存放领域层核心业务逻辑相关的代码。领域层可以包含多个聚合代码包，它们共同实现领域模型的核心业务逻辑。聚合以及聚合内的实体、方法、领域服务和事件等代码会放在这一层目录里。

实现核心业务逻辑。这一层聚集了领域模型的聚合、聚合根、实体、值对象、领域服务和事件等领域对象，以及它们组合所形成的业务能力。

```
type PushDomainService interface {//暴露给Application 层的 interface
   PushMessage(ctx context.Context, pushMessage *entity.PushMessagesDO) error
   SaveChannelLastPushMessagePosition(ctx context.Context, channelID int64, position int32) error
}

func NewPushDomainService() PushDomainService {
   return &pushDomainServiceImpl{}
}

type pushDomainServiceImpl struct {
}

func (p *pushDomainServiceImpl) PushMessage(ctx context.Context, pushMessage *entity.PushMessagesDO) error {
   return nil
}

```

**Domain Object （充血模型）**

```
type Message struct {
   *MessageEntity.Message
}

// IsDelete 此处表明 此消息已撤回.
func (m *Message) IsDeleted() bool {
   if !m.IsValidated() {
      return false
   }

   return ture
}

.....

```

**Repository 定义，一般一个聚合对应一个Repository**

```
type PushRepository interface { //定义repository接口，infra层会实现这个接口
   // 保存最后一个 push message 的 position
   SaveChannelLastPushMessagePosition(ctx context.Context, channelID int64, position int32) (bool, error)
}

func NewPushRepo() PushRepository {
   return persistence.PackRepoImpl{} // 这里也可以用依赖注入思想
}

```

#### **Infrastructure（基础层）**

> 它主要存放基础资源服务相关的代码，为其它各层提供的通用技术能力、三方软件包、数据库服务、配置和基础资源服务的代码都会放在这一层目录里。

基础层贯穿所有层，为各层提供基础资源服务。这一层聚集了各种底层资源相关的服务和能力。

```
// PushRepositoryImpl 可以包含mysql、abase、cache、MQ相关的实现
type PushRepositoryImpl struct{}

func (p PushRepositoryImpl) SaveChannelLastPushMessagePosition(ctx context.Context, channelID int64, position int32) (bool, error) {
   return abase.SaveChannelLastPushMessagePosition(ctx, channelID, position)

}

```

**调用关系如下图**

![image](https://upload-images.jianshu.io/upload_images/12321605-4741d33605fdb613?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image](https://upload-images.jianshu.io/upload_images/12321605-fc6b67727aea94f7?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 四、DDD常见的一些误区

### 4.1 所有的领域都用 DDD

很多人在学会 DDD 后，可能会将其用在所有业务域，即全部使用 DDD 来设计。DDD 从战略设计到战术设计，是一个相对复杂的过程，首先企业内要培养 DDD 的文化，其次对团队成员的设计和技术能力要求相对比较高。在资源有限的情况下，应聚焦核心域，建议你先从富领域模型的核心域开始，而不必一下就在全业务域推开。

### 4.2 全部采用 DDD 战术设计方法

不同的设计方法有它的适用环境，我们应选择它最擅长的场景。DDD 有很多的概念和战术设计方法，比如聚合根和值对象等。聚合根利用仓储管理聚合内实体数据之间的一致性，这种方法对于管理新建和修改数据非常有效，比如在修改订单数据时，它可以保证订单总金额与所有商品明细金额的一致，但它并不擅长较大数据量的查询处理，甚至有延迟加载进而影响效率的问题。

而传统的设计方法，可能一条简单的 SQL 语句就可以很快地解决问题。而很多贫领域模型的业务，比如数据统计和分析，DDD 很多方法可能都用不上，或用得并不顺手，而传统的方法很容易就解决了。

因此，在遵守领域边界和微服务分层等大原则下，在进行战术层面设计时，我们应该选择最适合的方法，不只是 DDD 设计方法，当然还应该包括传统的设计方法。这里要以快速、高效解决实际问题为最佳，不要为做 DDD 而做 DDD。

### 4.3 重战术设计而轻战略设计

很多 DDD 初学者，学习 DDD 的主要目的，可能是为了开发微服务，因此更看重 DDD 的战术设计实现。殊不知 DDD 是一种从领域建模到微服务落地的全方位的解决方案。

战略设计时构建的领域模型，是微服务设计和开发的输入，它确定了微服务的边界、聚合、代码对象以及服务等关键领域对象。领域模型边界划分得清不清晰，领域对象定义得明不明确，会决定微服务的设计和开发质量。没有领域模型的输入，基于 DDD 的微服务的设计和开发将无从谈起。因此我们不仅要重视战术设计，更要重视战略设计。

## 参考资料

https://docs.microsoft.com/zh-cn/azure/architecture/patterns/cqrs

https://www.infoq.cn/article/s_LFUlU6ZQODd030RbH9

https://time.geekbang.org/column/article/169600

https://mp.weixin.qq.com/s/Wt2Fssm8kd8b8evVD9aujA

https://insights.thoughtworks.cn/path-to-ddd/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io

https://blog.csdn.net/fly910905/article/details/104145292

