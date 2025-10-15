# 1.Spark常见的算子
* Transformation：从旧的RDD得到新的RDD，但是懒运算
	* map
	* filter
	* flatMap
	* union
	* groupByKey
	* reduceByKey
	* join
* Action：与Transformation相比一定会触发运算，即一定不会产生新的RDD
	* reduce
	* collect
	* count
	* foreach
# 2.Spark的Job、Stage、Task模型
* Job: 每个Action会产生一个Job，会被DAGScheduler分解成Stage和Task
* Stage：每调用一次会产生shuffle动作的算子（宽依赖操作）就会生成一个Stage
* Task：不管在一个Stage阶段有多少个算子，DAGScheduler都会产生与它的**最后一个 RDD 分区数相同数量的 Task**
# 3.Spark的并行度取决于哪些因素
简单来说，**并行度 ≈ 集群中同时运行的Task数量**
* 数据源的分区数：即初始并行度
	* 读取HDFS文件时，初始并行度为这个文件的块数
	* 读取JDBC源时，需要配置并行度（即分区数）
* shuffle下游的分区数： `spark.sql.shuffle.partitions` shuffle
* 显式重分区（算子）
* 集群资源数量：**num-executors * executor-cores**
# 4.Spark的join有哪些类型
* BroadcastJoin：将小表发送到每个Executor上，即不产生shuffle
* ShuffleHashJoin：按join key将两个表重新分区，通常在每个Executor上都是先建小表的Hash表，然后把大表的数据和Hash表拼接
* SortMergeJoin：与ShuffleHashJoin相比，不使用Hash表合并，而是使用归并排序合并
* BroadcastNestedLoopJoin：BroadcastJoin的非等值join场景
* CartesianProduct + Filter：即求笛卡尔积之后对非等值条件进行过滤，代价极高
# 5.RDD、DataFrame、DataSet、DataStream区别
* RDD：即弹性分布式数据集，Spark的基础，只读的分区数据
* DataFrame：基于RDD构建的行列结构数据，与RDD相比，具有元数据，即列有名称和类型
* DataSet：DataFrame是DataSet的特例，简单解释就是DataFrame每一行存储的是**抽象的行**，行内通过列名检索值，而DataSet则每一行存储具体的对象，行内通过对象属性来访问检索值
* DataStream：即SparkStream的微批底层实现，本质是**多个连续的RDD序列**，与RDD相比封装了自己专属的API