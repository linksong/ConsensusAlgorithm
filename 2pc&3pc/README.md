# 分布式一致性算法2PC和3PC

为了解决分布式一致性问题，产生了不少经典的分布式一致性算法，本文将介绍其中的2PC和3PC。
2PC即Two-Phase Commit，译为二阶段提交协议。
3PC即Three-Phase Commit，译为三阶段提交协议。

### 分布式系统和分布式一致性问题

分布式系统，即运行在多台不同的网络计算机上的软硬件系统，并且仅通过消息传递来进行通信和协调。
分布式一致性问题，即相互独立的节点之间如何就一项决议达成一致的问题。

### 2PC

2PC，二阶段提交协议，即将事务的提交过程分为两个阶段来进行处理：准备阶段和提交阶段。
事务的发起者称协调者，事务的执行者称参与者。

##### 阶段1：准备阶段

* 1、协调者向所有参与者发送事务内容，询问是否可以提交事务，并等待所有参与者答复。
* 2、各参与者执行事务操作，将Undo和Redo信息记入事务日志中（但不提交事务）。
* 3、如参与者执行成功，给协调者反馈YES，即可以提交；如执行失败，给协调者反馈NO，即不可提交。

##### 阶段2：提交阶段

此阶段分两种情况：所有参与者均反馈YES、或任何一个参与者反馈NO。
所有参与者均反馈YES时，即提交事务。
任何一个参与者反馈NO时，即中断事务。

提交事务：（所有参与者均反馈YES）
* 1、协调者向所有参与者发出正式提交事务的请求（即Commit请求）。
* 2、参与者执行Commit请求，并释放整个事务期间占用的资源。
* 3、各参与者向协调者反馈Ack完成的消息。
* 4、协调者收到所有参与者反馈的Ack消息后，即完成事务提交。

附如下示意图：

![](Commit.png)

中断事务：（任何一个参与者反馈NO）
* 1、协调者向所有参与者发出回滚请求（即Rollback请求）。
* 2、参与者使用阶段1中的Undo信息执行回滚操作，并释放整个事务期间占用的资源。
* 3、各参与者向协调者反馈Ack完成的消息。
* 4、协调者收到所有参与者反馈的Ack消息后，即完成事务中断。

附如下示意图：

![](Rollback.png)

### 2PC的缺陷

* 1、同步阻塞：最大的问题即同步阻塞，即：所有参与事务的逻辑均处于阻塞状态。
* 2、单点：协调者存在单点问题，如果协调者出现故障，参与者将一直处于锁定状态。
* 3、脑裂：在阶段2中，如果只有部分参与者接收并执行了Commit请求，会导致节点数据不一致。

由于2PC存在如上同步阻塞、单点、脑裂问题，因此又出现了2PC的改进方案，即3PC。

### 3PC

3PC，三阶段提交协议，是2PC的改进版本，即将事务的提交过程分为CanCommit、PreCommit、do Commit三个阶段来进行处理。

##### 阶段1：CanCommit

* 1、协调者向所有参与者发出包含事务内容的CanCommit请求，询问是否可以提交事务，并等待所有参与者答复。
* 2、参与者收到CanCommit请求后，如果认为可以执行事务操作，则反馈YES并进入预备状态，否则反馈NO。

##### 阶段2：PreCommit

此阶段分两种情况：
* 1、所有参与者均反馈YES，即执行事务预提交。
* 2、任何一个参与者反馈NO，或者等待超时后协调者尚无法收到所有参与者的反馈，即中断事务。

事务预提交：（所有参与者均反馈YES时）
* 1、协调者向所有参与者发出PreCommit请求，进入准备阶段。
* 2、参与者收到PreCommit请求后，执行事务操作，将Undo和Redo信息记入事务日志中（但不提交事务）。
* 3、各参与者向协调者反馈Ack响应或No响应，并等待最终指令。

中断事务：（任何一个参与者反馈NO，或者等待超时后协调者尚无法收到所有参与者的反馈时）
* 1、协调者向所有参与者发出abort请求。
* 2、无论收到协调者发出的abort请求，或者在等待协调者请求过程中出现超时，参与者均会中断事务。

##### 阶段3：do Commit

此阶段也存在两种情况：
* 1、所有参与者均反馈Ack响应，即执行真正的事务提交。
* 2、任何一个参与者反馈NO，或者等待超时后协调者尚无法收到所有参与者的反馈，即中断事务。

提交事务：（所有参与者均反馈Ack响应时）
* 1、如果协调者处于工作状态，则向所有参与者发出do Commit请求。
* 2、参与者收到do Commit请求后，会正式执行事务提交，并释放整个事务期间占用的资源。
* 3、各参与者向协调者反馈Ack完成的消息。
* 4、协调者收到所有参与者反馈的Ack消息后，即完成事务提交。

中断事务：（任何一个参与者反馈NO，或者等待超时后协调者尚无法收到所有参与者的反馈时）
* 1、如果协调者处于工作状态，向所有参与者发出abort请求。
* 2、参与者使用阶段1中的Undo信息执行回滚操作，并释放整个事务期间占用的资源。
* 3、各参与者向协调者反馈Ack完成的消息。
* 4、协调者收到所有参与者反馈的Ack消息后，即完成事务中断。

注意：进入阶段三后，无论协调者出现问题，或者协调者与参与者网络出现问题，都会导致参与者无法接收到协调者发出的do Commit请求或abort请求。
此时，参与者都会在等待超时之后，继续执行事务提交。

附示意图如下：

![](3PC.png)

### 3PC的优点和缺陷

优点：降低了阻塞范围，在等待超时后协调者或参与者会中断事务。避免了协调者单点问题，阶段3中协调者出现问题时，参与者会继续提交事务。

缺陷：脑裂问题依然存在，即在参与者收到PreCommit请求后等待最终指令，如果此时协调者无法与参与者正常通信，会导致参与者继续提交事务，造成数据不一致。

### 后记

无论2PC或3PC，均无法彻底解决分布式一致性问题。
解决一致性问题，唯有Paxos，后续将单独总结。




