# 实时计算概述
![enter image description here](http://static.oschina.net/uploads/img/201410/14080822_ia5p.png)

---

## 目录
 - 分布式系统和实时计算
 - Zookeeper
 - Storm
 - Kafka
 - 在业务中使用storm
 - 讨论

---

## 分布式系统
>***Storm是开源的、分布式、流式计算系统***

 - 分布式系统是把计算或存储任务放在多个主机上，提高扩展性

---

![谷歌三驾马车，提出分布式计算思想](https://images2015.cnblogs.com/blog/915691/201604/915691-20160408214808609-422199238.png)

---

Hadoop诞生

![enter image description here](https://images2015.cnblogs.com/blog/915691/201604/915691-20160408214958312-2124876577.png)

Hadoop和hive作为全量数据处理最著名的工具，具有吞吐量、自动容错等优点，在海量数据计算上具有很强的优势

---

## Hadoop的局限
>高延迟
>只能处理全量数据
>运维复杂

 - 广告点击
 - 电商推荐

---

## 实时计算
![enter image description here](https://images2015.cnblogs.com/blog/915691/201604/915691-20160408221451797-787695268.png)

---

## 实时计算要解决哪些问题
 - 低延迟
 - 高性能
 - 分布式
 - 可扩展
 - 容错性

---

# Zookeeper
![enter image description here](http://static.open-open.com/lib/uploadImg/20150321/20150321154620_278.jpg)

---

## zookeeper
 官方说辞：Zookeeper 分布式服务框架是Apache Hadoop 的一个子项目，它主要是用来解决分布式应用中经常遇到的一些数据管理问题，如：统一命名服务、状态同步服务、集群管理、分布式应用配置项的管理等。
***分布式协调服务框架***
***解决分布一致性问题***
***本质：分布式小文件存储系统 + 通知机制***

---

## 分布式一致性
### CAP
对于一个分布式系统，不可能同时兼顾三点：
 - 一致性（Consistency）
 - 可用性（Availability）
 - 分区容错性（Partition Tolerance）

---

### 弱一致性（高可用性和分区容错性）
 - DNS
 - Cassandra
### 强一致性（高一致性和分区容错性）
 - Master Slave(Redis): 高一致性、低可用性
 - Paxos
 - Raft
 - ZAB

---

## zookeeper提供了什么
### 特性
>全局数据一致
可靠性
顺序性
原子性
实时性

---

## zookeeper提供了什么
### 文件系统
znode：
 - 兼有文件和目录两种特性
 - 具有原子性操作
 - 每个节点存储大小有严格限制，一般都是几KB
 - 必须绝对路径引用

---

## zookeeper提供了什么
### 通知机制
 - 在客户端监听注册它关心的节点，一旦数据发生改变都会受到通知

---

## zookeeper的典型应用场景
>### 配置管理
在zk上管理服务的配置内容。订阅者是所有使用配置的服务
服务在启动时可以从ZK获取需要的配置，设置监听事件

---

>### 命名服务
利用zk的全局一致性，上下游的服务可以在zk约定好path相互探索发现

---

>### 分布式锁
***保持独占***
创建同名的节点作为锁。创建成功就获得了锁并操作某个文件
***控制时序***
根据ZK创建临时节点的序列化特性，根据创建节点的先后来访问文件

---

>### 集群管理
 - 机器的加入和退出
加入和退出时创建或删除相应的节点
 - master选举
类似于分布式独占锁

---

# Storm
![enter image description here](http://static.oschina.net/uploads/img/201410/14080822_ia5p.png)

---

## 
