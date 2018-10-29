## 分布式一致性
### CAP
对于一个分布式系统，不可能同时兼顾三点：
一致性（Consistency）
可用性（Availability）
分区容错性（Partition Tolerance）

一般来说做到两点最优已经是极限

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


