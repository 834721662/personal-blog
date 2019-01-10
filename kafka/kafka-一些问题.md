
一、目的
记录问题列表，并且给出相应的原理性的答疑解惑，方便后续维护、运维、参数优化配置等。

二、问题解惑
1）Kafka broker下线后，重新再上线，会发生什么事情，对数据写入、读取是否有影响？

上下线机制：

每台 Broker 在上线时，都会与 ZK /brokers/ids 下注册一个节点，节点名字就是 broker id，这个节点是临时节点。Controller 会监听 /brokers/ids 这个路径下的所有子节点，如果有新的节点出现，那么就代表有新的 Broker 上线，如果有节点消失，就代表有 broker 下线，Controller 会进行相应的处理。

1.1上线：

（1）先在 ControllerChannelManager 中添加该 broker（即建立与该 Broker 的连接、初始化相应的发送线程和请求队列）

（2）Controller 调用 onBrokerStartup() 上线该 Broker：

向当前集群所有存活的 Broker 发送 Update Metadata 请求，这样的话其他的节点就会知道当前的 Broker 已经上线；
获取当前节点分配的所有的 Replica 列表，并将其状态转移为 OnlineReplica 状态
触发 PartitionStateMachine 的 triggerOnlinePartitionStateChange() 方法，为所有处于 NewPartition/OfflinePartition 状态的 Partition 进行 leader 选举，如果 leader 选举成功，那么该 Partition 的状态就会转移到 OnlinePartition 状态，否则状态转移失败；
如果副本迁移中有新的 Replica 落在这台新上线的节点上，那么开始执行副本迁移操作
如果之前由于这个 Topic 设置为删除标志，但是由于其中有 Replica 掉线而导致无法删除，这里在节点启动后，尝试重新执行删除操作。


1.2掉线：

（1）先在 ControllerChannelManager 中移除该 broker（即关闭与 Broker 的连接、关闭相应的发送线程和清空请求队列）；

（2）Controller 调用 onBrokerFailure() 下线该 Broker:

首先找到 Leader 在该 Broker 上所有 Partition 列表，然后将这些 Partition 的状态全部转移为 OfflinePartition 状态；
触发 PartitionStateMachine 的 triggerOnlinePartitionStateChange() 方法，为所有处于 NewPartition/OfflinePartition 状态的 Partition 进行 Leader 选举，如果 Leader 选举成功，那么该 Partition 的状态就会迁移到 OnlinePartition 状态，否则状态转移失败（Broker 上线/掉线、Controller 初始化时都会触发这个方法）；
获取在该 Broker 上的所有 Replica 列表，将其状态转移成 OfflineReplica 状态；
过滤出设置为删除、并且有副本在该节点上的 Topic 列表，先将该 Replica 的转移成 ReplicaDeletionIneligible 状态，然后再将该 Topic 标记为非法删除，即因为有 Replica 掉线导致该 Topic 无法删除；
如果 leader 在该 Broker 上所有 Partition 列表不为空，证明有 Partition 的 leader 需要选举，在最后一步会触发全局 metadata 信息的更新。


broker下线分优雅下线和非优雅下线，区别是：优雅下线会主动通知controller

获取副本在该节点上的所有 Partition 的列表集合；
遍历上述 Partition 列表进行处理：如果该 Partition 的 leader 是要下线的节点，那么通过 PartitionStateMachine 进行状态转移（OnlinePartition –> OnlinePartition）触发 leader 选举，使用的 leader 选举方法是 ControlledShutdownLeaderSelector，它会选举 isr 中第一个没有正在关闭的 Replica 作为 leader，否则抛出 StateChangeFailedException 异常；
否则的话，即要下线的节点不是 leader，那么就向要下线的节点发送 StopReplica 请求停止副本同步，并将该副本设置为 OfflineReplica 状态，这里对 Replica 进行处理的原因是为了让要下线的机器关闭副本同步流程，这样 Kafka 服务才能正常关闭。
1.3对数据写入、读取的影响

broker下线对写入的影响：

如果leader replica 在要下线的broker，会重新选举新的leader，对应partition如果提交数据，会发生metadata失效。
对于leader replica 不在要下线的broker，如果是ISR节点，下线后在没有新的节点加入ISR情况下，如果ISR数量小于minisr,会导致数据写入失败。


broker上线对写入的影响

如果发生replica均衡操作，leader转移到新节点上，producer客户端会收到metadata过期消息，连接新节点进行写入。
如果新节点的replica加入ISR，并超过配置的minISR,使该partition能重新写入。
broker上下线对读取的影响：

如果leader发生变化，consumer需要重新连接到新的leader进行消费。


2）在Kafka发生partition leader rebalance时，producer端配置了retries=2，但还会有少量数据发送失败，是什么原因导致的，leader rebalance时至少要重试多少次才能发送成功？

主要应该是retry的频率太快导致的。

partition leader rebalance 本身是需要时间的。一个消息批次在发送失败后，如果需要重试，就会重新放入队列中等待重发，等待的时间间隔由producer的配置retry.backoff.ms决定，这个配置的默认值是100ms。也就是说，默认情况下，消息发送失败后过100ms会尝试重发，如果在100ms内partition leader rebalance还没完成，则这条重发的消息还是会失败。

所以，可以观察一下partition leader rebalance的一个大概时间，然后调整retry.backoff.ms配置的值。

另外，需要注意一点，如果retries的次数配置成1，那么retry.backoff.ms的时间配置的再长，也可能导致下一次重发的消息也发送失败。这是因为在消息发送失败后，如果发现失败原因和元数据有关系（比如partition leader变了）,kafka会立即去更新元数据缓存起来，但是如果更新元数据的时候partition leader rebalance 还未完成，那么拿到的元数据还是不对，等我们下次重发消息时还是会因为元数据问题导致数据发送再次失败。

这个问题目前看来好像没办法避免，只能调大retries次数来避免。



3）做reassign topic/partition时，数据怎么同步，什么时候标识同步完成，以及如果reassign的partition正好是leader，什么时候切换leader，并且对数据读写的影响是怎样的？是否有流量控制？

RAR = Reassigned replicas，目标要分配的副本情况
OAR = Original list of replicas for partition，原先的副本分配情况
AR = current assigned replicas，当前的副本分配情况

更新zk处的partition副本配置：AR=RAR+OAR
向所有RAR+OAR的副本发送元数据更新请求
将新增的那部分的副本状态设置为NewReplica。也就是 RAR-OAR 那部分副本
等待所有的副本和leader保持同步。也就是抱着RAR+OAR的副本都在isr中了
将所有在RAR中的副本状态都设置为OnlineReplica
在内存中先将AR=RAR
如果leader不在RAR中，就需要重新竞选leader。采用ReassignedPartitionLeaderSelector选举
将所有准备移除的副本状态设置为OfflineReplica。也就是OAR-RAR的那部分副本。这时partition的isr会收缩
将所有准备移除的副本状态设置为NonExistentReplica。这时所在的分区副本数据会被删除。
将内存中的AR更新到zk
更新zk的/admin/reassign_partitions路径，移除这个partition
发送新的元数据到各个broker上
假设当前有OAR = {1, 2, 3}， RAR = {4,5,6}，在进行partition reaassigned的过程中会发生如下变化

AR
leader/isr
步骤
{1,2,3}	1/{1,2,3}	初始状态
{1,2,3,4,5,6}	1/{1,2,3,4,5,6}	步骤2
{1,2,3,4,5,6}	1/{1,2,3,4,5,6}	步骤4
{1,2,3,4,5,6}	4/{1,2,3,4,5,6}	步骤7
{1,2,3,4,5,6}	4/{1,2,3,4,5,6}	步骤8
{4,5,6}	4/{4,5,6}	步骤10


4）多磁盘下，选择磁盘存储机制？怎么保证磁盘存储均衡？
```java
//LogManager 
private def nextLogDir(): File = {
  if(logDirs.size == 1) {
    logDirs(0)
  } else {
    // count the number of logs in each parent directory (including 0 for empty directories
    val logCounts = allLogs.groupBy(_.dir.getParent).mapValues(_.size)
    val zeros = logDirs.map(dir => (dir.getPath, 0)).toMap
    var dirCounts = (zeros ++ logCounts).toBuffer

    // choose the directory with the least logs in it
    val dirCountsMap = new util.HashMap[String, Integer]()
    dirCounts.foreach(tuple2 => dirCountsMap.put(tuple2._1, tuple2._2))
    val leastLoaded = new DiskPartitioner().partitioner(dirCountsMap)
    new File(leastLoaded)
  }
}


@Override
public String partitioner(Map<String, Integer> dirCount) {
   long template = 0;
   String result = null;
   File file;
   for (String fileName : dirCount.keySet()) {
      file = new File(fileName);
      if (template < file.getFreeSpace()) {
         template = file.getFreeSpace();
         result = fileName;
      }
   }
   return result;
}

@Override
public String getType() {
   return "disk";
}
```

5）kafka默认每7天触发一次log compaction，log compaction会导致长时间stop the world,是什么原因导致的？



6）已知broker配置一致，新增broker节点，并增加partition 备份到新broker节点，当备份节点同步进度跟上leader后，发现备份节点的日志量比旧机器日志量大，为什么？

　  0.10.0.0版本kafka按日志文件最后修改时间删除日志，新的备份节点的日志文件最后修改时间为日志同步完成的时间，而不是消息生成的时间。清理策略不能及时删除新备份节点上的数据导致备份节点的日志量比旧机器日志量大。



7）ack=1 时leader节点有没有把日志落盘再返回response?

     日志的落盘是靠kafka-log-flusher任务来定时的刷盘

8）kafka 日志文件的稀疏索引是怎么做的 ？


```java
Log.scala
val segment = new LogSegment(···
                             indexIntervalBytes = config.indexInterval, //获取的信息来自 index.interval.bytes
                             ···)

LogSegment.scala
@nonthreadsafe
def append(firstOffset: Long, largestOffset: Long, largestTimestamp: Long, shallowOffsetOfMaxTimestamp: Long, records: MemoryRecords) {
  	·····
    // append an entry to the index (if needed)
    if(bytesSinceLastIndexEntry > indexIntervalBytes) { //判断距离上次写入OffSetinde和timeIndex的后产生的增量字节是否大于索引的步长 (0.10.2和0.10.0 只是多了一个timeIndex)
      index.append(firstOffset, physicalPosition)
      timeIndex.maybeAppend(maxTimestampSoFar, offsetOfMaxTimestamp)
      bytesSinceLastIndexEntry = 0
    }
    bytesSinceLastIndexEntry += records.sizeInBytes
  }

```

9）delete策略是怎么清理日志的？

     0.10.0.0版本按日志文件最后修改时间删除，0.11.0.0后按日志内容里最大的Message timestamp删除。


10）delete策略清理数据时的最小粒度是文件还是消息？（原问题：delete策略是整个文件删除还是可以部分删除）

   1.delete的策略有两种 根据时间删除LogSegment文件 和 根据文件大小来删除文件  文件的删除是一个一个删除 不会有存在删除部分的情况
11）描述使用log.retention.bytes时kafka的日志清理策略（原问题：跟据文件大小删除时是怎么删的？）

根据文件的大小计算出溢出大小 然后在遍历所有的LogSegment 
```java
private def cleanupSegmentsToMaintainSize(log: Log): Int = {
    if(log.config.retentionSize < 0 || log.size < log.config.retentionSize)
      return 0
    var diff = log.size - log.config.retentionSize
    def shouldDelete(segment: LogSegment) = {
      //判断删除容量是否已经大于溢出大小
      if(diff - segment.size >= 0) {
        diff -= segment.size
        true
      } else {
        false
      }
    }
    log.deleteOldSegments(shouldDelete)
  }
```

12）日志解压发生在consumer端,还是broker端，怎么获取消息的压缩类型的?

1.broker会解压 压缩类型是producer上传的 2.consumer也会解压 原理看下面代码
```java
 public static MemoryRecordsBuilder builderWithEntries(TimestampType timestampType,
                                                      CompressionType compressionType,
                                                      long logAppendTime,
                                                      List<LogEntry> entries) {
    //准备写入数据是首先会把compressionType的entries长度数据和写入buffer中 estimatedSize(compressionType, entries) 得到的结果是高16为是长度 低八位是压缩类型
    ByteBuffer buffer = ByteBuffer.allocate(estimatedSize(compressionType, entries));
    return builderWithEntries(buffer, timestampType, compressionType, logAppendTime, entries);
}
```
13）旧版的consumer api有没有类似新版api的ConsumerCoordiantor存活检查功能? （原问题：旧的consumer api有没有用ConsumerCoordiantor? 是否会给broker发送心跳？怎么判断consumer group 里的consumer线程/进程已经死了？）



14）broker的compression.type配置:如果在producer端已经压缩过了，当broker的压缩类型和producer配置的类型不一致,会在broker解压缩再压缩一次么?

     1.压缩的类型已broker server的配置为主   producer压缩数据传输到broker server 都会解压 然后通过broker配置的压缩方式在进行压缩

15）client id有什么用？

       1.为了日志调试 2.为了区别内置的client id
16）log.retention.bytes只对log.cleanup.policy＝delete时生效?

      只对log.cleanup.policy＝delete生效

17）min.insync.replicas配置和isr列表有什么关系？

isr数量小于min.insync.replicas，日志存储会抛异常。






