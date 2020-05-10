---
layout: post
title:  "分布式事务的几种解决方案"
date:   2020-02-10 19:02:00
categories: article
tags: java
author: "sxzhou"
---
分布式事务是是一个很复杂的问题，但同时也是后端开发无法回避的一个问题。如这篇文章[[A Guide to Transactions Across Microservices]([https://www.baeldung.com/transactions-across-microservices)]开头所说，保证分布式事务很难，因此其实最好的方法是避免使用它，但是，在交易、支付等系统必须要保证，同时互联网公司的高并发场景也使得问题更复杂，工作经历了几家头部互联网公司，也看到一些不同的解决方案。

## 1. 一些概念  
**分布式事务**就是指事务的参与者、支持事务的服务器、资源服务器以及事务管理器分别位于不同的分布式系统的不同节点之上。
### 1.1 ACID
* `Atomicity`（原子性）  
  事务中的所有操作，要么全部完成，要么全部不做，不能在某个中间点停止。
* `Consistency`（一致性）  
  必须使数据从一个一致性状态转换为另一个一致性状态。
* `Isolation`（隔离性）  
  一个事务的的执行不收到另一个事务干扰。  
* `Durability`（持久性）  
  事务成功后，事务所做修改持久保存在数据库中。  

单机事务可以保证ACID，可以称作`刚性事务`，分布式环境下完全遵守ACID很困难，因此又出现一些新的理论，可以称为`柔性事务`。  

### 1.2 CAP  
对于一个分布式系统，不可能同时满足以下三点：  
* `Consistency` (一致性)  
所有节点访问同一份最新的数据副本。  

* `Availability` (可用性)  
每次请求都能获取到非错的响应——但是不保证获取的数据为最新数据。  

* `Partition tolerance` (分区容错性)  
以实际效果而言，分区相当于对通信的时限要求。系统如果不能在时限内达成数据一致性，就意味着发生了分区的情况，必须就当前操作在C和A之间做出选择。  

如何理解必须三选二？  
根据维基百科：理解CAP理论的最简单方式是想象两个节点分处分区两侧。允许至少一个节点更新状态会导致数据不一致，即丧失了C性质。如果为了保证数据一致性，将分区一侧的节点设置为不可用，那么又丧失了A性质。除非两个节点可以互相通信，才能既保证C又保证A，这又会导致丧失P性质。  
CP系统：如果不要求保证可用A但必须一致C，也就是节点之间数据变更必须保证同步，而P会导致同步时间无限延长。  
AP系统：如果不要求保证一致C但必须可用A，那么如果发生分区，每个节点使用本地数据提供服务，就会导致数据不一致。

### 1.3 BASE
* `BA`（BASIC Availability） 基本可用
* `S`（Soft State） 软状态
* `E`（Eventual consistency） 最终一致  
其核心思想是即使无法做到强一致性，但每个应用都可以根据自身业务特点，才用适当的方式来使系统打到最终一致性。为了可用性和性能的需求，降低了对一致性和隔离性的要求，做到基本可用最终一致。  

### 1.4 2PC  
为了使基于分布式系统架构下的所有节点在进行事务提交时保持一致性而设计的一种算法。  
[二阶段提交](https://zh.wikipedia.org/wiki/%E4%BA%8C%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4) 

## 2. 一常见解决方案  
有了完备的理论基础，在实际生产中还需要考虑一些现实的问题，主要包括在高并发环境下的性能、对业务的侵入以及改造成本等，要是不能权衡以上各个问题，很难在实际生产环境推广。   
### 2.1 XA  
[X/Open XA](https://zh.wikipedia.org/wiki/X/Open_XA)  
XA是一种比较完备的分布式事务规范，对**业务无侵入**，但是需要数据库层面提供协议支持，而且资源锁定周期长，性能可能无法达到高并发环境的需求。目前主流互联网公司使用较少。  

### 2.2 TCC 
TCC是业务上常用的一种解决方案，是Try-Confirm-Cancel的简称，从名字便可知道大致流程。首先上图，来自我司CTO程立(鲁肃)在08年的一次演讲内容。  
![https://s1.ax1x.com/2020/05/10/Y37DZ6.png](https://s1.ax1x.com/2020/05/10/Y37DZ6.png)
* Try  
  完成所有业务检查，预留业务资源。
* Confirm  
  确认执行业务操作，不做任何业务检查， 只使用Try阶段预留的业务资源。
* Cancel  
  取消Try阶段预留的业务资源。  

相比XA，TCC在生产中使用更广泛，XA是**数据库的分布式事务**，强一致性，在整个过程中，数据一张锁住状态，即从prepare到commit、rollback的整个过程中，TM一直把持折数据库的锁，如果有其他人要修改数据库的该条数据，就必须等待锁的释放，存在**长事务风险**。TCC是**业务层**的分布式事务，最终一致，不会出现长事务的锁风险，try是本地事务，锁定资源后就提交事务，confirm／cancel也是本地事务，可以直接提交事务，所以多个短事务不会出现长事务的风险。  

更多TCC的概念介绍：[https://houbb.github.io/2018/09/02/sql-distribute-transaction-tcc](https://houbb.github.io/2018/09/02/sql-distribute-transaction-tcc)
### 2.3 本地消息异步确保  
![https://s1.ax1x.com/2020/05/10/Y3qfy9.png](https://s1.ax1x.com/2020/05/10/Y3qfy9.png)  
业务活动的主动方，在完成业务处理的
同一个本地事务中，记录消息数据。业务处理事务提交后、通过实时消息服
务通知业务被动方，实时通知成功后删除消息数据。消息恢复系统定期找到未成功发送的消息，交给实时消息服务补发送。  
前公司使用使用这种方式处理分布式事务，是一种实现成本比较低的方案，让业务数据提交和写入本地消息在同一个本地事务，但是需要**消息中间件的可靠性**，保证一定能够推送被动方成功。同时，**被动方的执行结果不会影响主动方**，如果被动方后续流程失败，那么需要补偿措施通知主动方在业务层面实现回滚。
### 2.4 事务消息  
部分消息中间件实现了事务消息功能，保证消息发送成功一定完成下游消费，对比上面本地消息方案，业务层面开发成本更低。  
待续