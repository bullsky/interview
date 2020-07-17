# 基于raft共识算法的kv存储

## 是什么
基于raft共识算法的kv存储服务，提供GET/PUT/APPEND操作
## 为什么
- raft是目前应用比较广泛的共识算法，也相对易于理解和实现，从这个点切入去了解共识算法是一个合理的选择。
- fabric当前使用raft算法来做共识，项目需要去实现自己的共识算法，需要熟悉raft共识算法，熟悉最好的办法就是自己实现。
## 怎么做

### raft共识算法实现
raft算法包含leader选举，日志复制，存储快照
#### leader的选举
**leader选举基本流程**
1. 所有节点以follower启动
2. follower的选举时钟超时，转为candidate
   1. 将term加1
   2. 给自己投票
3. candidate向其他节点发送投票请求，如果收到过半节点的投票，则成为leader
   1. 如果没有选出leader，candidate重新进入candidate状态，开启新的一轮选举
4. leader周期性向其他节点发送心跳包以维持权威
![raft状态转移图](raft-状态转移.png)

**重置选举时钟**
重置选举时钟的3种情形：
1. 从leader处收到appendentriesRPC调用（如果leader的term过期则不用重置）
2. 开始新的一轮选举
3. 给其他节点投票（收到requestVoteRPC调用）

**关于逻辑时钟term**
1. 任何状态下的节点只要发现更高的term都要转为follwer,同时将term更新为最新
2. 对于过期term的消息不予处理。

**什么样的日志是最新的？**
1. term大为最新
2. term相同，log长的为最新的

#### 日志的复制
leader中维护了两个列表nextindex和matchindex来协助日志的复制。
nextindex维护了对于需要发送给每个follower的下一个日志条目的索引值（初始化为领导人最后索引值加一）
matchindex维护了于每个follwer已经复制的日志的最高索引值。

**日志冲突处理**
日志冲突指指定index的元素的term不一致。论文中日志冲突follower只会返回失败，而不是具体的冲突信息。leader对于冲突的处理也是简单的线性递减探查，此种办法效率低下。经过优化后：
1. follower: 定位冲突信息（conflictTerm、conflictIndex），并将其返回给leader。其中conflictTerm是follwer的冲突term，conflictIndex是该conflictTerm下的第一个元素。
2. leader: 根据conflictTerm、conflictIndex，找到在log中conflictTerm的最后一个元素。并将nextindex设置为该值。

**提交状态机**
- 一个守护线程，批量处理需要提交的元素

### kv存储实现

**client**
- 需要客户端id和随机消息id来区分每一条seq，其中消息id递增，防止重复消息
- 发现请求的不是leader需要及时切换leader

**server**
- 需要记录已经提交到状态机每个client的最新序列号的seq，任何小于seq的消息都不予处理。
- 需要一个channel map 来获取每个操作对应的返回结果
- client所有请求操作通过调用raft接口，利用raft模块做共识。在提交之前需要设置好对应的消息传递channel，之后等待返回消息。
  - 此处需要用select来同时等待消息以及等待时钟的信号
- server会有一个守护线程，接受raft模块提交的操作，并根据相应的操作执行kvDB的操作。并将相应的操作结果通过channel返回给接收端
- 将处理结果返回给client

## 结果
实现了功能

## 成长
熟悉raft共识算法，对分布式系统有了更多的了解, 深刻认识到分布式系统开发的复杂性,培养了能够从无到有实现系统的能力。

## Q&A

leader负载均衡

高并发处理

存储扩展