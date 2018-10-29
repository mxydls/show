## 分布式一致性
### CAP
对于一个分布式系统，不可能同时兼顾三点：
一致性（Consistency）
可用性（Availability）
分区容错性（Partition Tolerance）

一般来说做到两点最优已经是极限

### 弱一致性（高可用性和分区容错性）
 - DNS
 - Cassandra

### 强一致性（高一致性和分区容错性）
 - Master Slave(Redis): 高一致性、低可用性
 - Paxos
 - Raft
 - ZAB



## Zookeeper

***分布式协调服务框架***
***本质是分布式小文件存储系统 + 通知机制***

**特性:**
>全局数据一致
可靠
顺序性
原子性
实时性

Leader：唯一。事务性请求的调度者，处理者
Follower：多个。处理非事务性请求；转发事务性请求给leader

***zk的文件系统：***
znode：
兼有文件和目录两种特性
具有原子性操作
每个节点存储大小有严格限制，一般都是几KB
必须绝对路径引用

znode创建时可以指定有三种类型：永久、临时、序列化


***zk watcher事件***
单次的，触发一次后不会再触发
用WatchedEvent来封装事件
异步发送


### ZK的典型应用场景
1. **数据发布与订阅**
主要针对一些服务的配置场景。
比如发布者作为数据库配置的管理。订阅者是所有使用配置的服务。
可以使用ZK作为发布者
服务在启动时可以从ZK获取需要的配置，设置监听事件。

2. **命名服务**
利用ZK的全局文件一致性

3. **分布式锁**
*保持独占*
创建同名的节点作为锁。创建成功就获得了锁并操作某个文件
其他应用可以监听目录是否存在的事件
*控制时序*
根据ZK创建节点的序列化特性，根据创建节点的先后来访问文件

4. **集群管理**
每个集群服务器都创建一个临时节点，并监听父节点的子节点变化消息
master的选举也可以采用时序节点，选最小的那个即可

###zookeeper原理
####Paxos算法
>在一个分布式数据库系统中，如果各节点的初始状态一致，每个节点都执行相同的操作序列，那么他们最后能得到一个一致的状态。

Paxos算法解决的什么问题呢，解决的就是保证每个节点执行相同的操作序列。好吧，这还不简单，master维护一个全局写队列，所有写操作都必须 放入这个队列编号，那么无论我们写多少个节点，只要写操作是按编号来的，就能保证一致性。没错，就是这样，可是如果master挂了呢。

Paxos算法通过投票来对**写操作进行全局编号**，同一时刻，只有一个写操作被批准，同时并发的写操作要去争取选票，只有获得过半数选票的写操作才会被 批准（所以永远只会有一个写操作得到批准），其他的写操作竞争失败只好再发起一轮投票，就这样，在日复一日年复一年的投票中，所有写操作都被严格编号排 序。**编号严格递增**，当一个节点接受了一个编号为100的写操作，之后又接受到编号为99的写操作（因为网络延迟等很多不可预见原因），它马上能意识到自己 数据不一致了，自动停止对外服务并重启同步过程。任何一个节点挂掉都不会影响整个集群的数据一致性（总2n+1台，除非挂掉大于n台）。

####事务的顺序一致
为了保证事务的顺序一致性，zookeeper采用了递增的事务id号（zxid）来标识事务。所有的提议（proposal）都在被提出的时候加上 了zxid。实现中zxid是一个64位的数字，它高32位是epoch用来标识leader关系是否改变，每次一个leader被选出来，它都会有一个 新的epoch，标识当前属于那个leader的统治时期。低32位用于递增计数。

https://blog.csdn.net/gs80140/article/details/51496925



## Storm
**实时计算要解决的几个问题**
 - 低延迟
 - 高性能
 - 分布式
 - 可扩展
 - 容错性

**storm的几个特性**
 - 消息不会丢失
 - 消息保证有序
 - 使用ZeroMQ作为底层消息队列，保证消息得到快速处理

![Alt text](./1533307255159.png)
![enter image description here](https://images2015.cnblogs.com/blog/915691/201604/915691-20160411182138738-1210708111.png)
![storm集群架构](https://7n.w3cschool.cn/attachments/tuploads/apache_storm/zookeeper_framework.jpg)

>**组件**
**Nimbus：**主节点，负责资源分配和任务调度
**supervisor：**工作节点。负责接受nimbus分配来的任务，管理自己的worker进程（每个工作节点的worker数可以在配置文件中配置，配置几个端口就对应几个worker）
**worker：**运行处理具体逻辑的进程
**task：**对应spout/bolt的线程，目前不和物理线程一一对应，多个task可能会在一个线程中，叫做**executor**

---

>**一些概念**
topology：拓扑结构，构成了一个在storm中运行的实例应用。拓扑描述了消息的流动形式
spout：在拓扑中充当数据源的角色。通常spout会从外部读取数据，然后转换为tuple数据形式，并输出。编程中实现nextTuple()函数
bolt：接收数据并执行处理。bolt可以执行过滤、函数操作、合并、写入数据库等任何操作。bolt是被动的角色，实现execute()函数


**分组方式**
![enter image description here](https://images2015.cnblogs.com/blog/915691/201604/915691-20160409214252734-740156118.png)


###Zookeeper在storm中的作用
![enter image description here](https://images2015.cnblogs.com/blog/163162/201607/163162-20160726175521091-2102927128.png)
客户端提交拓扑到nimbus。

Nimbus针对该拓扑建立本地的目录根据topology的配置计算task，分配task，在zookeeper上建立assignments节点存储task和supervisor机器节点中woker的对应关系；

在zookeeper上创建taskbeats节点来监控task的心跳；启动topology。

Supervisor去zookeeper上获取分配的tasks，启动多个woker进行，每个woker生成task，一个task一个线程；根据topology信息初始化建立task之间的连接;Task和Task之间是通过zeroMQ管理的；后整个拓扑运行起来。


###Storm的可靠性机制
####容错性
**如果一个worker意外死亡**，supervisor会重启它。如果它在启动时连续失败多次，那么会在一段时间内一直没有发送心跳给nimbus，这个时候nimbus会在另一台主机上重新分配worker。

**如果一个集群中的主机宕机了**，那么在这台主机上的任务都会停止，nimbus会重新分配这些任务。

**如果nimbus和supervisor守护进程不幸死亡**。storm将nimbus和supervisor设计成***快速失败***（碰到意外情况进程立刻毁灭）和无状态（状态在zk或本地文件上）的。我们最好使用demontools或monit工具监控运行nimbus和supervisor守护进程。这样一旦进程死亡，就可以重启，并从zk拿到信息像什么事都没发生一样继续工作。另外nimbus和supervisor的死亡并不影响worker（独立的进程，worker的心跳并不直接与supervisor和nimbus交互）。这也是zk在storm中的重要作用。

**某种意义上nimbus是单点故障（SPOF）的**。如果它宕机了，worker并不会直接受到影响，还会继续工作。如果某个worker死亡了，supervisor也会负责重启。但是如果碰到一些必须需要nimbus的事情（比如上面提到的nimbus在supervisor主机宕机时重新分配worker），就会受到影响。根据经验据说这在生产中影响不大。未来storm可能会设计成高可用的。

####什么是fully processed?
 >Storm的可靠性是指Storm会告知用户每一个消息单元是否在一个指定的时间(timeout)内被完全处理。完全处理的意思是该MessageId绑定的源Tuple以及由该源Tuple衍生的所有Tuple都经过了Topology中每一个应该到达的Bolt的处理。

注: timetout 可以通过Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS 来指定

Storm中的每一个Topology中都包含有一个***Acker组件***。Acker组件的任务就是跟踪从某个task中的Spout流出的每一个messageId所绑定的Tuple树中的所有Tuple的处理情况。
如果在用户设置的最大超时时间内这些Tuple没有被完全处理，那么Acker会告诉Spout该消息处理失败，相反则会告知Spout该消息处理成功,它会分别调用Spout中的**fail**和**ack**方法。

Storm允许用户在Spout中发射一个新的源Tuple时为其指定一个MessageId，这个MessageId可以是任意的Object对象。多个源Tuple可以共用同一个MessageId，表示这多个源Tuple对用户来说是同一个消息单元，它们会被放到同一棵tuple树中，如下图所示:
![Alt text](./1535369410157.png)

####storm的Acker机制
 storm里面有一类特殊的task称为acker（acker bolt）， 负责跟踪spout发出的每一个tuple的tuple树。当acker发现一个tuple树已经处理完成了。它会发送一个消息给产生这个tuple的那个task。你可以通过Config.TOPOLOGY_ACKERS来设置一个topology里面的acker的数量， 默认值是1。 如果你的topology里面的tuple比较多的话， 那么把acker的数量设置多一点，效率会高一点。

>**当一个tuple需要ack的时候，它到底选择哪个acker来发送这个信息呢？**
  storm使用一致性哈希来把一个spout-tuple-id对应到acker， 因为每一个tuple知道它所有的祖宗的tuple-id， 所以它自然可以算出要通知哪个acker来ack。
**acker是怎么知道每一个spout tuple应该交给哪个task来处理?**
当一个spout发射一个新的tuple， 它会简单的发一个消息给一个合适的acker，并且告诉acker它自己的id(taskid)， 这样storm就有了taskid-tupleid的对应关系。 当acker发现一个树完成处理了， 它知道给哪个task发送成功的消息。


![Alt text](./1535369891681.png)
msg1绑定了两个源tuple，它们的id分别为1001和1010.在经过Bolt1处理后新生成了tuple id为1110,新生成的tuple与传入的tuple 1001进行异或得到的值为0111，然后Bolt1通过spout-tuple-id映射到指定的Acker组件，向它发送消息，Acker组件将Bolt1传过来的值与ack_val异或，更新ack_val的值变为了0100。与此相同经过Bolt2处理后，ack_val的值变为0001。最后经Bolt3处理后ack_val的值变为了0，说明此时由msg1标识的Tuple处理成功，此时Acker组件会通过事先绑定的task id映射找到对应的Spout,然后调用该Spout的ack方法。

>**Storm如何进行异常处理**
1. 由于对应的task挂掉了，一个tuple没有被ack： storm的超时机制在超时之后会把这个tuple标记为失败，从而可以重新处理。
2. Acker挂掉了： 这种情况下由这个acker所跟踪的所有spout tuple都会超时，也就会被重新处理。
**注意，storm本身并不负责tuple的重新处理，需要我们手动在fail回调中加入重新emit的逻辑。可以保存msgid和tuple的对应关系**
3. spout挂掉了：如果spout挂掉，消息的重发将完全由消息来源负责，比如kafka、RabbitMQ等。

https://www.cnblogs.com/hd3013779515/p/6971875.html

####Storm的可靠性api
首先，要实现ack机制：
1，spout发射tuple的时候需要手动指定messageId
2，spout要重写BaseRichSpout的fail和ack方法
3，spout对发射的tuple进行缓存(否则spout的fail方法收到acker发来的messsageId，spout也无法获取到发送失败的数据进行重发)，看看系统提供的接口，只有msgId这个参数，这里的设计不合理，其实在系统里是有cache整个msg的，只给用户一个messageid，用户如何取得原来的msg貌似需要自己cache，然后用这个msgId去查询，太坑爹了
4,设置acker数至少大于0；Config.setNumAckers(conf, ackerParal);

**Storm的Bolt有BasicBolt和RichBolt:**
　　在BasicBolt中，BasicOutputCollector在emit数据的时候，会自动和输入的tuple相关联，而在execute方法结束的时候那个输入tuple会被自动ack。
　　使用RichBolt需要在emit数据的时候，显式指定该数据的源tuple要加上第一个参数anchor tuple，以保持tracker链路，即collector.emit(oldTuple, newTuple);并且需要在execute执行成功后调用OutputCollector.ack(tuple), 当失败处理时，执行OutputCollector.fail(tuple);

所以其实storm本身对可靠性的保障并不强，只是提供了acker机制来对tuple进行跟踪，其余的处理都需要用户自己保证。


### storm事务机制（Transactional Topology）
首先说明一点，原始的事务拓扑在storm中早已被弃用（也许是因为用起来比较复杂也容易出错），同样，使用事务拓扑实现的LinearDRPCTopology也已被弃用。现在storm推荐用封装更完善、更上层的Trident。这里介绍一些原理和我的理解，当然可能在Trident中有些原理是不同的，但是我觉得本质上上大同小异，所以还是有学习的必要的。

#### 设计机理
具体可以去看storm老的官方文档，使用这种分两阶段的事务性处理可以充分利用storm的并行优势，提升效率避免不必要的等待过程。

Processing阶段：多个batch可以并行计算
Commiting阶段：batch之间按照强顺序进行提交

#### Transactional Spout
The TransactionalSpout interface is completely different from a regular Spout interface. A TransactionalSpout implementation emits batches of tuples and must ensure that the same batch of tuples is always emitted for the same transaction id.

A transactional spout looks like this while a topology is executing:
![Alt text](./捕获.PNG)
其中coordinator是spout，emitter是bolt。

这里面有两种类型的tuple，一种是事务性的tuple，一种是真实batch中的tuple；

coordinator为事务性batch发射tuple，Emitter负责为每个batch实际发射tuple。

具体如下：

Here's how transactional spout works:

1. Transactional spout is a subtopology consisting of a coordinator spout and an emitter bolt
2. The coordinator is a regular spout with a parallelism of 1
3. The emitter is a bolt with a parallelism of P, connected to the coordinator's "batch" stream using an all grouping
4. When the coordinator determines it's time to enter the processing phase for a transaction, it emits a tuple containing the TransactionAttempt and the metadata for that transaction to the "batch" stream
5. Because of the all grouping, every single emitter task receives the notification that it's time to emit its portion of the tuples for that transaction attempt
6. Storm automatically manages the anchoring/acking necessary throughout the whole topology to determine when a transaction has completed the processing phase. The key here is that *the root tuple was created by the coordinator, so the coordinator will receive an "ack" if the processing phase succeeds, and a "fail" if it doesn't succeed for any reason (failure or timeout).
7. If the processing phase succeeds, and all prior transactions have successfully committed, the coordinator emits a tuple containing the TransactionAttempt to the "commit" stream.
8. All committing bolts subscribe to the commit stream using an all grouping, so that they will all receive a notification when the commit happens.
9. Like the processing phase, the coordinator uses the acking framework to determine whether the commit phase succeeded or not. If it receives an "ack", it marks that transaction as complete in zookeeper.

![Alt text](./捕获1.PNG)


利用ack机制，storm帮我们追踪了tuple流。当所有tuple都完成时，coordinator就可以收到通知，然后就意味着这个事务已经结束了。
commit bolt意味着我们希望这个bolt作为“最后一个”，在整个事务结束时它才会被调用finishBatch，也就是上面的最后一步。具体的调用逻辑在源码中找不到，因为已经被弃用太久，应该已经完全被移除出去了。也可能这部分逻辑在clojure中。但是应该就是在coordinator ack的时候用某种机制调用的。
但是还有一个问题是，非commit bolt的finishBatch是怎么调用的呢？既然它们的执行不依赖于整个事务的结束，而依赖于对自己已接收到前面bolt的所有tuple的确认，那就需要某种机制来进行追踪。也就是CoordinateBolt。

CoordinateBolt具体原理如下：

 - 真正执行计算的bolt外面封装了一个CoordinateBolt。真正执行任务的bolt我们称为real bolt。
 - 每个CoordinateBolt记录两个值：有哪些task给我发送了tuple（根据topology的grouping信息）；我要给哪些tuple发送信息（同样根据groping信息）
 - Real bolt发出一个tuple后，其外层的CoordinateBolt会记录下这个tuple发送给哪个task了。
 - 等所有的tuple都发送完了之后，CoordinateBolt通过另外一个特殊的stream以emitDirect的方式告诉所有它发送过 tuple的task，它发送了多少tuple给这个task。下游task会将这个数字和自己已经接收到的tuple数量做对比，如果相等，则说明处理 完了所有的tuple。
 - 下游CoordinateBolt会重复上面的步骤，通知其下游。

![Alt text](./捕获3.PNG)
在CoordinateBolt中找到的逻辑。其中finishId里会调用finishBatch。

![Alt text](./捕获4.PNG)
在LinearTopologyBuilder中，就是对所有bolt做了一个封装。componet.bolt是一个BasicBatchBoltExecutor。

事务API早已弃用，请使用Trident。


## Kafka
>Kafka主要设计目标如下：
 - 以时间复杂度为O(1)的方式提供消息持久化能力，即使对TB级以上数据也能保证常数时间的访问性能。
 - 高吞吐率。即使在非常廉价的商用机器上也能做到单机支持每秒100K条消息的传输。
 - 支持Kafka Server间的消息分区，及分布式消费，同时保证每个partition内的消息顺序传输。
 - 同时支持离线数据处理和实时数据处理。

>Kafka数据传输的事务特点


----------
![enter image description here](http://www.aboutyun.com/data/attachment/forum/201505/02/225851j2s4eq67aq9llaol.png)

![enter image description here](https://7n.w3cschool.cn/attachments/tuploads/apache_kafka/fundamentals.jpg)

![enter image description here](https://7n.w3cschool.cn/attachments/tuploads/apache_kafka/cluster_architecture.jpg)

####Kafka专用术语：
 - Broker：消息中间件处理结点，一个Kafka节点就是一个broker，多个broker可以组成一个Kafka集群。
 - Topic：一类消息，Kafka集群能够同时负责多个topic的分发。
 - Partition：topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列。
 - Segment：partition物理上由多个segment组成。
 - Offset：每个partition都由一系列有序的、不可变的消息组成，这些消息被连续的追加到partition中。partition中的每个消息都有一个连续的序列号叫做offset，用于partition唯一标识一条消息。
 - Producer：负责发布消息到Kafka broker。
 - Consumer：消息消费者，向Kafka broker读取消息的客户端。
 - Consumer Group：每个Consumer属于一个特定的Consumer Group。


####kafaka提供两种方式来发布消息
#####发布者-订阅者模型
 - 生产者定期向主题发送消息。
 - Kafka代理存储为该特定主题配置的分区中的所有消息。 它确保消息在分区之间平等共享。 如果生产者发送两个消息并且有两个分区，Kafka将在第一分区中存储一个消息，在第二分区中存储第二消息。
 - 消费者订阅特定主题。
 - 一旦消费者订阅主题，Kafka将向消费者提供主题的当前偏移，并且还将偏移保存在Zookeeper系综中。
 - 消费者将定期请求Kafka(如100 Ms)新消息。
 - 一旦Kafka收到来自生产者的消息，它将这些消息转发给消费者。
 - 消费者将收到消息并进行处理。
 - 一旦消息被处理，消费者将向Kafka代理发送确认。
 - 一旦Kafka收到确认，它将偏移更改为新值，并在Zookeeper中更新它。 由于偏移在Zookeeper中维护，消费者可以正确地读取下一封邮件，即使在服务器暴力期间。
 - 以上流程将重复，直到消费者停止请求。
 - 消费者可以随时回退/跳到所需的主题偏移量，并阅读所有后续消息。

#####队列消息/用户组的工作流
在队列消息传递系统而不是单个消费者中，具有相同组ID 的一组消费者将订阅主题。 简单来说，订阅具有相同 Group ID 的主题的消费者被认为是单个组，并且消息在它们之间共享。 让我们检查这个系统的实际工作流程。

 - 生产者以固定间隔向某个主题发送消息。
 - Kafka存储在为该特定主题配置的分区中的所有消息，类似于前面的方案。
 - 单个消费者订阅特定主题，假设 Topic-01 为 Group ID 为 Group-1 。
 - Kafka以与发布 - 订阅消息相同的方式与消费者交互，直到新消费者以相同的组ID 订阅相同主题 Topic-01  1 。
 - 一旦新消费者到达，Kafka将其操作切换到共享模式，并在两个消费者之间共享数据。 此共享将继续，直到用户数达到为该特定主题配置的分区数。
 - 一旦消费者的数量超过分区的数量，新消费者将不会接收任何进一步的消息，直到现有消费者取消订阅任何一个消费者。 出现这种情况是因为Kafka中的每个消费者将被分配至少一个分区，并且一旦所有分区被分配给现有消费者，新消费者将必须等待。
 - 此功能也称为使用者组。 同样，Kafka将以非常简单和高效的方式提供两个系统中最好的。

###kafka的副本机制

![Alt text](./1540470882493.png)

一个broker对应一个服务器，为了达到适当的负载均衡，多个partition和各自的多个副本会平摊到多个broker上。
如图

 - 分区的数量不能多于broker的数量
 - partition的数量可以多于broker的数量，不过这样的话负载不太均衡，不推荐

Producer在发布消息到某个Partition时，先通过ZooKeeper找到该Partition的Leader，然后无论该Topic的Replication Factor为多少，Producer只将该消息发送到该Partition的Leader。Leader会将该消息写入其本地Log。每个Follower都从Leader pull数据。这种方式上，Follower存储的数据顺序与Leader保持一致。Follower在收到该消息并写入其Log后，向Leader发送ACK。一旦Leader收到了ISR中的所有Replica的ACK，该消息就被认为已经commit了，Leader将增加HW并且向Producer发送ACK。

为了提高性能，每个Follower在接收到数据后就立马向Leader发送ACK，而非等到数据写入Log中。因此，对于已经commit的消息，Kafka只能保证它被存于多个Replica的内存中，而不能保证它们被持久化到磁盘中，也就不能完全保证异常发生后该条消息一定能被Consumer消费。

Consumer读消息也是从Leader读取，只有被commit过的消息才会暴露给Consumer。

###消息机制
![Alt text](./1534386712982.png)
![Alt text](./1534386732750.png)
>注意，kafka只会对每个partition内的消息进行offset记录，consumer也只记录partition的offset，所以分区内部消息是严格有序的，但是全局不能保证有序。


###consumer机制
同一Topic的一条消息只能被同一个Consumer Group内的一个Consumer消费，但多个Consumer Group可同时消费这一消息。

![Alt text](./1534383800900.png)

这是Kafka用来实现一个Topic消息的广播（发给所有的Consumer）和单播（发给某一个Consumer）的手段。一个Topic可以对应多个Consumer Group。如果需要实现广播，只要每个Consumer有一个独立的Group就可以了。要实现单播只要所有的Consumer在同一个Group里。用Consumer Group还可以将Consumer进行自由的分组而不需要多次发送消息到不同的Topic。


##Logstash + Kafka + Storm实时日志分析
### 1. logstash配置

logstash在1.51版本之后提供了对kafka的插件，建议使用最新版。
下载地址：

    https://artifacts.elastic.co/downloads/logstash/logstash-6.4.0.tar.gz

最简单的filter配置如下:

```
output {
	kafka {
		bootstrap_servers => "kafka_host:9092"
		topic_id => "your-topic-name"
	}
}

input {
	file {
		path => $your_log_file_paths
	}
}
```

日志来源根据需求来定

### 2. Kafka Spout
现在我们有producer了，而consumer就是我们的storm。storm和kafka同为Apache公司的开源组件，因此storm本身已经提供了针对kafka的编程接口。只是这个实现在多个storm和kafka版本迭代中一直在不断变化，不同的版本组合都可能要使用不同的api，写法上也有很大区别，因此建议使用上文中的kafka和storm版本。

在kafka 0.10.X 版本以后storm使用kafka-client jar 进行 Storm Apache Kafka 的集成，相较于老版本api有很大的改变，这是因为kafka在0.9.X 版本之后对offset的管理发生了改变。具体原理在这里不赘述。

#### 2.1 maven项目配置
在maven项目的pom.xml中加入如下依赖：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.netease.maxiaoyuan</groupId>
    <artifactId>pgxgamelogger</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>3.8.1</version>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.storm</groupId>
            <artifactId>storm-core</artifactId>
            <version>1.2.2</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka_2.11</artifactId>
            <version>2.0.0</version>

            <exclusions>
                <exclusion>
                    <groupId>org.apache.zookeeper</groupId>
                    <artifactId>zookeeper</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-log4j12</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>log4j</groupId>
                    <artifactId>log4j</artifactId>
                </exclusion>
            </exclusions>

            <scope>compile</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-clients</artifactId>
            <version>2.0.0</version>
        </dependency>

        <dependency>
            <groupId>org.apache.storm</groupId>
            <artifactId>storm-kafka-client</artifactId>
            <version>1.2.2</version>
            <scope>provided</scope>
        </dependency>


        <dependency>
            <groupId>org.apache.storm</groupId>
            <artifactId>storm-kafka</artifactId>
            <version>1.2.2</version>

        </dependency>
    </dependencies>

    <build>
        <plugins>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.1.0</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>PGXGameloggerAPP</mainClass>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```
实际项目开发时会添加更多依赖，这是最小的部分。

打包时执行命令：

```
mvn package -f pom.xml
```
生成的jar包在根目录的target目录下。


#### 2.2 KafkaSpout代码
在这里可以认为kafka是作为一类特定的spout，我们只需配置，不用管tuple生成的细节。

在main class中使用以下方式得到一个最简单的kafkaspout：
```
private static KafkaSpoutConfig getKafkaSpoutConfig(String bootstrapServer, String topic) {
    return KafkaSpoutConfig.builder(bootstrapServer, topic)
        .setProp(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true")
        .setFirstPollOffsetStrategy(KafkaSpoutConfig.FirstPollOffsetStrategy.LATEST)
        .setProp(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BROKER_LIST)
        .setProp(ConsumerConfig.GROUP_ID_CONFIG, GROUP_ID)
        .build();
}

KafkaSpoutConfig spoutConfig = getKafkaSpoutConfig(BOOTSTRAP_SERVER, TOPIC);

KafkaSpout kafkaSpout = new KafkaSpout(spoutConfig);
```
其中BOOTSTRAP_SERVER和TOPIC根据实际情况而定。

具体api请参官方文档：

    http://storm.apachecn.org/releases/cn/1.1.0/storm-kafka-client.html

我们知道，storm本身提供了acker机制来保障tuple被完全处理，有时候为了节省开销我们会忽略可靠性，关掉ack功能。kafkaspout被封装起来了，它也提供了可靠和不可靠两种方式。

在上面代码中我有一行：

```
.setProp(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true")
```
这一行表示让kafkaspout忽略可靠性保障，也就是不会去跟踪每一个tuple，这样的话只要spout接收到了kafka的消息，就会更新offset，结果就是如果我们之后因为某些错误想要重新读取旧的数据将无法实现，当然这是追求性能的牺牲。

#### 2.3 拓扑逻辑
kafkaspout会emit五个字段的tuple："topic","partition","offset","key","value"，一般来说获取"value"就是我们需要的日志内容，之后根据业务需求创建各种bolt处理即可。

