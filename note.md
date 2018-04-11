## 1 
(http://bit1129.iteye.com/blog/2182146)
(http://blog.csdn.net/qq_20641565/article/details/76216417)
cache和persist的区别了：cache只有一个默认的缓存级别MEMORY_ONLY ，而persist可以根据情况设置其它的缓存级别。
cache()与persist()：
会被重复使用的(但是)不能太大的RDD需要cache。cache 只使用 memory，写磁盘的话那就叫 checkpoint 了。 哪些 RDD 需要 checkpoint？
运算时间很长或运算量太大才能得到的 RDD，computing chain 过长或依赖其他 RDD 很多的 RDD。 实际上，
将 ShuffleMapTask 的输出结果存放到本地磁盘也算是 checkpoint，只不过这个 checkpoint 的主要目的是去 partition 输出数据。
cache 机制是每计算出一个要 cache 的 partition 就直接将其 cache 到内存了。但 checkpoint 没有使用这种第一次计算得到就存储的方法，
而是等到 job 结束后另外启动专门的 job 去完成 checkpoint 。 也就是说需要 checkpoint 的 RDD 会被计算两次。
因此，在使用 rdd.checkpoint() 的时候，建议加上 rdd.cache()， 这样第二次运行的 job 就不用再去计算该 rdd 了，直接读取 cache 写磁盘。
cache 与 checkpoint 的区别:
关于这个问题，Tathagata Das 有一段回答:
> There is a significant difference between cache and checkpoint. Cache materializes the RDD and keeps it in memory and/or disk（其实只有 memory）. But the lineage（也就是 computing chain） of RDD (that is, seq of operations that generated the RDD) will be remembered, 
so that if there are node failures and parts of the cached RDDs are lost, they can be regenerated. However, checkpoint saves the RDD to an HDFS file and actually forgets the lineage completely. 
This is allows long lineages to be truncated and the data to be saved reliably in HDFS (which is naturally fault tolerant by replication). 
>persist()与checkpoint() 深入一点讨论，rdd.persist(StorageLevel.DISK_ONLY) 与 checkpoint 也有区别。前者虽然可以将 RDD 的 partition 持久化到磁盘，但该 partition 由 blockManager 管理。一旦 driver program 执行结束，也就是 executor 所在进程 CoarseGrainedExecutorBackend stop，blockManager 也会 stop，被 cache 到磁盘上的 RDD 也会被清空（整个 blockManager 使用的 local 文件夹被删除）。而 checkpoint 将 RDD 持久化到 HDFS 或本地文件夹，如果不被手动 remove 掉（ 话说怎么 remove checkpoint 过的 RDD？ ），是一直存在的，也就是说可以被下一个 driver program 使用，
而 cached RDD 不能被其他 dirver program 使用。
'''object StorageLevel {
  val NONE = new StorageLevel(false, false, false, false)
  val DISK_ONLY = new StorageLevel(true, false, false, false)
  val DISK_ONLY_2 = new StorageLevel(true, false, false, false, 2)
  val MEMORY_ONLY = new StorageLevel(false, true, false, true)
  val MEMORY_ONLY_2 = new StorageLevel(false, true, false, true, 2)
  val MEMORY_ONLY_SER = new StorageLevel(false, true, false, false)
  val MEMORY_ONLY_SER_2 = new StorageLevel(false, true, false, false, 2)
  val MEMORY_AND_DISK = new StorageLevel(true, true, false, true)
  val MEMORY_AND_DISK_2 = new StorageLevel(true, true, false, true, 2)
  val MEMORY_AND_DISK_SER = new StorageLevel(true, true, false, false)
  val MEMORY_AND_DISK_SER_2 = new StorageLevel(true, true, false, false, 2)
  val OFF_HEAP = new StorageLevel(false, false, true, false)
  ......
}
'''
## 2 dataframetable不宜做太复杂的操作，计算很慢.
使用迭代
"""
df.select(*[col expression for col in col_list])
"""

## 3如何优雅地终止正在运行的Spark Streaming程序 
(https://www.iteblog.com/archives/1890.html)

## 4.定时任务
通过SparkContext的setJobGroup方法为每个Job设置不同的JobGroup，自己维护一个列表，记录这些JobGroup，并记录启动时间。
 写个定时器，定期扫描已经JobGroup列表，对于>=5min的任务调用SparkContext的cancelJobGroup方法kill掉就可以了（
如果任务已经执行完了，调用改方法也没啥副作用），最后将超过5min的JobGroup信息从列表移除就可以了。

## 5.面对选择条件时，sql小trick
http://blog.csdn.net/u010454030/article/details/78925143
使用spark sql访问hive的表，然后根据一批id把需要的数据过滤出来，本来是非常简单的需求直接使用下面的伪SQL即可：
select * from table where  id in (id1,id2,id3,id4,idn)
但现在遇到的问题是id条件比较多，大概有几万个，这样量级的in是肯定会出错的，看网上文章hive的in查询超过3000个就报错了。
如何解决？
主要有两种解决方法：
（一）分批执行，就是把几万个id，按3000一组查询一次，最后把所有的查询结果在汇合起来。
（二）使用join，把几万个id创建成一张hive表，然后两表关联，可以一次性把结果给获取到。
这里倾向于第二种解决办法，比较灵活和方便扩展，尽量不要把数据集分散，一旦分散意味着客户端需要做更多的工作来合并结果集，
比如随便一个sum或者dinstict，如果是第一种则需要在最终的结果集再次sum或者distinct。
下面看看如何使用第二种解决：
由于我们id列表是动态的，每个任务的id列表都有可能变换，所以要满足第二种方法，就得把他们变成一张临时表存储在内存中，
当spark任务停止时，就自动销毁，因为他们不需要持久化到硬盘上。


## 5.配置CDH

'''
export http_proxy=http://
export https_proxy=http://
export ftp_proxy=http://

export JAVA_HOME=/usr/java/jdk1.8.0_73
export CDH_HOME=/opt/cloudera/parcels/CDH
export SPARK2_HOME=/opt/cloudera/parcels/SPARK2
export KYLIN_HOME=/hadoop/kylin/apache-kylin-1.6.0-cdh5.7-bin
export HBASE_HOME=/opt/cloudera/parcels/CDH/lib/hbase
export HADOOP_HOME=/opt/cloudera/parcels/CDH/lib/hadoop
export HIVE_HOME=/opt/cloudera/parcels/CDH/lib/hive
export HADOOP_CMD=/opt/cloudera/parcels/CDH/lib/opt/bin/hadoop
export HCAT_HOME=/opt/cloudera/parcels/CDH/lib/hive-hcatalog
export JAVA_LIBRARY_PATH=$JAVA_LIBRARY_PATH:$HADOOP_HOME/lib/native
export SPARK_YARN_USER_ENV="JAVA_LIBRARY_PATH=$JAVA_LIBRARY_PATH,LD_LIBRARY_PATH=$LD_LIBRARY_PATH"
export hive_dependency=$HIVE_HOME/conf:$HIVE_HOME/lib/*:/opt/cloudera/parcels/CDH/lib/hive-hcatalog/share/hcatalog/hive-hcatalog-core-1.1.0-cdh5.13.0.jar
export PATH=$JAVA_HOME/bin:$KYLIN_HOME/bin:$HADOOP_HOME/bin:$HBASE_HOME/bin:$HIVE_HOME/bin:$SCALA_HOME/bin:$PATH
export YARN_HOME=$HADOOP_HOME
export YARN_CONF_DIR=${YARN_HOME}/etc/hadoop
export CLASS_PATH=.:/usr/share/cmf/lib:$JAVA_HOME/lib:$SPARK_HOME/jars:$HBASE_HOME:$HBASE_HOME/lib:$HADOOP_HOME/share/hadoop/common/lib:$CLASS_PATH
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HADOOP_HOME/lib:$HBASE_HOME/lib:$SPARK_HOME/jars:$SCALA_HOME/lib:$HADOOP_HOME/lib/native

export SPARK_DIST_CLASSPATH=$(${HADOOP_HOME}/bin/hadoop classpath):$HBASE_HOME/lib/*:/hadoop/spark_python_deps/*
'''

###6 select
"""
sqlContext.sql(s"SELECT count from mytable WHERE id=$id")
"""


###6 spark和hadoop生态系统的调优
"""
现象：
通过spark2-submit提交的程序，运行到tasks到199/200的时候，就一直等到，不再输出日志。
	spark2-submit  --master yarn  --num-executors 3  --executor-cores 2  --jars /home/fastjson-1.2.47.jar --class com.klclear.data.hbase.KLServer /home/kldata-_2.11-0.1.jar

检查：
通过netstat -antp | grep 2181 查看发现很多等待zookeeper的连接


通过日志发现有如下错误：
18/04/08 14:50:50 INFO zookeeper.ClientCnxn: Socket connection established, initiating session, client: /172.16.222.228:54948, server: test1/172.16.222.228:2181
18/04/08 14:50:50 INFO zookeeper.ClientCnxn: Unable to read additional data from server sessionid 0x0, likely server has closed socket, closing socket connection and attempting reconnect
18/04/08 14:50:51 INFO zookeeper.ClientCnxn: Opening socket connection to server test3/172.16.222.230:2181. Will not attempt to authenticate using SASL (unknown error)
18/04/08 14:50:51 INFO zookeeper.ClientCnxn: Socket connection established, initiating session, client: /172.16.222.228:39384, server: test3/172.16.222.230:2181


18/04/08 14:56:54 WARN zookeeper.ClientCnxn: Session 0x0 for server test1/172.16.222.228:2181, unexpected error, closing socket connection and attempting reconnect
java.io.IOException: Connection reset by peer
at sun.nio.ch.FileDispatcherImpl.read0(Native Method)
at sun.nio.ch.SocketDispatcher.read(SocketDispatcher.java:39)
at sun.nio.ch.IOUtil.readIntoNativeBuffer(IOUtil.java:223)
at sun.nio.ch.IOUtil.read(IOUtil.java:192)
at sun.nio.ch.SocketChannelImpl.read(SocketChannelImpl.java:380)
at org.apache.zookeeper.ClientCnxnSocketNIO.doIO(ClientCnxnSocketNIO.java:68)
at org.apache.zookeeper.ClientCnxnSocketNIO.doTransport(ClientCnxnSocketNIO.java:355)
at org.apache.zookeeper.ClientCnxn$SendThread.run(ClientCnxn.java:1081)

综合分析zookeeper连接达到最大值之后， spark的程序一直等待zookeeper的连接，通过增加zookeeper的maxClientCnxns来更改。
更改之后，spark程序运行的速度提高，对应的task不再pending。

Spark环境组件很多，涉及各日志很多，检查日志可以帮助定位问题的。
Spark的环境参数，会很大影响程序运行效率，Spark调优需要深入研究。


参考：
https://blog.csdn.net/hengyunabc/article/details/41450003
http://812893004-qq-com.iteye.com/blog/2323574

"""
