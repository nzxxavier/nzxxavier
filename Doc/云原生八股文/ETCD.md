# 1.核心算法：Raft
* Leader：唯一决策者，由集群大多数节点选举出来，处理所有客户端请求（读写），负责数据同步
* Follower：存储数据，由Leader调度
* Candidate：Follower若长时间未收到Leader的心跳，则会变为Candidate并发起选举
# 2.存储机制
* WAL：用于**保证一致性、持久性和可恢复性**
* BoltDB：键值对存储，存储最终的**键值对数据**本身
* 快照机制：WAL文件太大时，etcd会将WAL转化为快照（即BoltDB的存储值），生成快照后，删除旧WAL并开始写新的WAL