
## 技巧卡
- 技巧：当你决定使用ZooKeeper来设计应用时，最好**将应用数据和协同数据独立开**。
- 例子：邮箱服务的例子：用户只对邮箱中的内容感兴趣，并不关心由哪台服务器来处理特定邮箱的请求。
	- 邮箱内容就是**应用数据**
	- 从邮箱到某一台邮箱服务器之间的映射关系就是**协同数据（也称元数据）**
	- ZooKeeper服务所管理的正是后者。

## 术语卡
- 术语：ZooKeeper的使命
- 印象：一句话总结就是**它可以在分布式系统中协作多个任务**。
	- 协作任务：一个协作任务是指一个包含多个进程的任务。这个任务既可以是为了协作，也可以是为了竞争。
		- 协作意味着多个进程相互配合才能完成一件事，比如典型的主从工作模式中，从节点空闲的时候会通知主节点，于是主节点就会给从节点分配任务。
		- 竞争则意味着多个进程不能同时处理工作，一个进程必须等待另一个进程。同样的在主从模式中，我们想要得到一个主节点，其他节点也想成为主节点，这时候就存在竞争关系，因此我们需要**互斥排他锁**。获取主节点控制锁的进程就是主节点进程。
	- 在典型的不共享环境下，不同的计算机之间不共享除了网络之外的其他任何信息。虽然许多消息传递算法可以实现同步原语，但是使用一个提供某种有序共享存储的组件往往更加简便，这正是ZooKeeper所采用的方式。
- 例子：ZooKeeper的使用实例
	- Apache HBase: 数据存储仓库，ZooKeeper用于选举一个集群内的主节点，以便跟踪可用的服务器，并保存集群的元数据。
	- Apache Kafka: 一个基于发布-订阅（pub-sub）模型的消息系统。其中ZooKeeper用于检测崩溃，实现主题（topic）的发现，并保持主题的生产和消费状态。
	- Apache Solr: 一个企业级的搜索平台，使用ZooKeeper来存储集群的元数据，并协作更新这些元数据。
	- Facebook Messages: 通信应用。该应用将ZooKeeper作为控制器，用来实现数据分片、故障恢复和服务发现等功能。

## 术语卡
- 术语：ZooKeeper不适用的场景
- 印象：ZooKeeper用于管理应用协作的关键数据，它不适合用作海量数据的存储。
- 例子：
	- 海量数据的存储一般选择**数据库**或**分布式文件系统**。ZooKeeper用于存储轻量级的元数据。所以最佳实践是将应用数据和协同数据独立开。
	- ZooKeeper还提供了一些工具，利用这些工具开发人员可以实现主节点选举、进程存活与否的功能跟踪等协同任务。

## 术语卡
- 术语：分布式系统
- 印象：分布式系统是同时跨越多个物理主机，独立运行的多个软件组件所组成的系统。分布式系统中的进程通信有两种选择：
	- 直接通过网络进行信息交换
	- 读写某些共享存储
- 例子：ZooKeeper使用共享存储模型来实现应用间的协作和同步原语。对于共享存储本身，又需要在进程和存储间进行网络通信。

## 实例卡
- 实例：一般的主-从应用
- 印象：主节点负责跟踪从节点的状态和任务的有效性，并分配任务给从节点。要实现主-从模式的系统，需要解决三个关键问题：
	- 主节点崩溃：无法分配新任务或重新分配执行失败的任务。
	- 从节点崩溃：已分配的任务无法完成。
	- 通信故障：主-从节点之间无法进行信息交换，任务也无法分配。
- 例子：
	- 主节点失效：
		- 需要增加一个**备份主节点**，当主要主节点崩溃时，备份主节点接管相关工作，但这并不是简单的处理新接入的任务，而是要将主节点的状态恢复到旧的主要主节点崩溃时的状态。我们不能依靠已经崩溃的主节点来获取这些信息，而是需要从ZooKeeper获取。
		- 除了状态恢复，还要注意另一个问题。有时候主节点并未崩溃，但由于负载过高，导致消息延迟，使得备份主节点以为主要主节点崩溃了，然后接管主要主节点的角色，成为第二个主要主节点。更糟糕的是，有时候由于网络分区错误等原因，导致一部分从节点无法与主要主节点通信，而选择与备份主节点建立主-从关系。这种场景导致的问题我们一般称之为**脑裂（split-brain）: 系统中两个或者多个部分开始独立工作，导致整体行为的不一致性**。
	- 从节点失效：
		- 正常流程：客户端向主节点提交任务，主节点将任务派发给有效的从节点，从节点接收到派发的任务，执行任务，执行完后向主节点报告执行状态，主节点会将执行结果通知给客户端。
		- 如果从节点崩溃了，那么所有派发给该从节点且尚未完成的任务都需要重新分配，所以主节点需要有检测从节点是否崩溃的能力，并且能够确定哪些从节点是否有效以便给分配崩溃节点未完成的任务。
	- 通信故障：
		- 如果出现主-从节点之间通信失败，重新分配一个任务有可能导致两个从节点执行相同的任务，如果一个任务不允许多次执行，那么在分配任务时不需要验证前面的从节点是否执行过此任务。**对任务加锁，并不能保证一个任务执行多次**。
		- 通信故障的另一个重要问题是对锁等同步原语的影响。
- 总结：根据以上描述，可以得到主-从架构的需求：
	- 主节点选举
	- 崩溃检测
	- 组成员关系管理
	- 元数据管理


