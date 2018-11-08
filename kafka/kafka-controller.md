[TOC]

##Kafka Controller

###controller简介

in a Kafka cluster, one of the brokers serves as the controller, which is responsible for managing the states of partitions and replicas and for performing administrative tasks like reassigning partitions. 

在Kafka集群中存在多个broker，其中一个broker会选举成controller，负责整个分区管理分区和副本的状态，并执行管理任务，比如重新分配分区, partition的leader 副本故障，由controller 负责为该partition重新选举新的leader 副本；当检测到ISR列表发生变化，有controller通知集群中所有broker更新其MetadataCache信息；或者增加某个topic分区的时候也会由controller管理分区的重新分配工作。

---

###kafkaController选举

在每个broker启动的时候都会自动调用Kafka.startup方法，但是只有一个broker会成为controller  
作用: 
1. registerSessionExpirationListener方法: 用于在zk会话失效后重连时取消注册在zookeeper上的各种Listener
2. controllerElector.startup 启动选举 --> ZookeeperLeaderElector.elect
```java
    /**
     * Invoked when the controller module of a Kafka server is started up. This does not assume that the current broker
     * is the controller. It merely registers the session expiration listener and starts the controller leader
     * elector
     */
    def startup() = {
      inLock(controllerContext.controllerLock) {
        info("Controller starting up")
        //注册监听重新连接,实际上调用 KafkaHeathCheck.startup
        registerSessionExpirationListener()  
        isRunning = true
        //选举
        controllerElector.startup
        info("Controller startup complete")
      }
    }
```

关于zk如何通过抢占注册为一个controller
```java
    def startup {
      inLock(controllerContext.controllerLock) {
        controllerContext.zkUtils.zkClient.subscribeDataChanges(electionPath, leaderChangeListener)
        elect
      }
    }
    ...
    def elect: Boolean = {
      val timestamp = SystemTime.milliseconds.toString
      val electString = Json.encode(Map("version" -> 1, "brokerid" -> brokerId, "timestamp" -> timestamp))
     //判断是否已经存在controller
     leaderId = getControllerID 
      if(leaderId != -1) {
         debug("Broker %d has been elected as leader, so stopping the election process.".format(leaderId))
         return amILeader
      }
      //如果不存在controller 尝试创建znode
      try {
        val zkCheckedEphemeral = new ZKCheckedEphemeral(electionPath,
                                                        electString,
                                                        controllerContext.zkUtils.zkConnection.getZookeeper,
                                                        JaasUtils.isZkSecurityEnabled())
        //创建节点, 并且更新controller
        zkCheckedEphemeral.create()
        info(brokerId + " successfully elected as leader")
        leaderId = brokerId
        onBecomingLeader()
      } catch {
        case e: ZkNodeExistsException =>
          // If someone else has written the path, then
          leaderId = getControllerID 
          if (leaderId != -1)
            debug("Broker %d was elected as leader instead of broker %d".format(leaderId, brokerId))
          else
            warn("A leader has been elected but just resigned, this will result in another round of election")
        case e2: Throwable =>
          error("Error while electing or becoming leader on broker %d".format(brokerId), e2)
          resign()
      }
      amILeader
    }
```
ZookeeperLeaderElector.elect函数负责选举leader，KafkaController选举是直接通过zk实现的，就是在zk创建临时目录/controller并在目录下存放当前brokerId。
如果在zookeeper下创建路径没有抛出ZkNodeExistsException异常，则当前broker成功晋级为Controller。
除了调用elect外，controllerElector.startup还会在/controller/路径上注册Listener，监听dataChange事件和dataDelete事件，当/controller 数据发生变化时，表示Controller发生了变化；
而因为/controller/下的数据为临时znode，当Controller发生failover时，znode会被删除，触发dataDelete事件，同时触发重新选举新

---
####附:zk存储信息
![zk](http://cf.meitu.com/confluence/download/attachments/63432943/zk.png?version=1&modificationDate=1532311639390&api=v2)
  
  /brokers/ids/[id] 记录集群中的broker id
  /brokers/topics/[topic]/partitions 记录了topic所有分区分配信息以及AR集合
  /brokers/topics/[topic]/partitions/[partition_id]/state记录了某partition的leader副本所在brokerId,leader_epoch, ISR集合,zk 版本信息
  /controller_epoch 记录了当前Controller Leader的年代信息
  /controller 记录了当前Controller Leader的id，也用于Controller Leader的选择
  /admin/reassign_partitions 记录了需要进行副本重新分配的分区,指导重新分配分区与副本的路径，通过命令修改分区和副本时会写入到这个路径下。
  /admin/preferred_replica_election：记录了需要进行"优先副本"选举的分区，分区需要重新选举第一个replica作为leader，即所谓的preferred replica。
  /admin/delete_topics 记录删除的topic
  /isr_change_notification 记录一段时间内ISR列表变化的分区信息
  /config 记录的一些配置信息

---
###kafkaController启动

  当一个broker成为controller之后执行的操作:
  1. 更新controller epoch,并把它写入zk，新的 epoch > 旧的 epoch
  2. 监听zk路径/admin/reassign_partitions  监听重新分配分区的信息
  3. 监听zk路径/admin/preferred_replica_election
  4. 注册partition状态机中的监听器，监听路径/brokers/topics的子目录变化，随时准备创建topic
  5. 注册replica状态机中的监听器，监听路径/brokers/ids/的子目录，以便在新的broker加入时能够感知到； 
  6. 初始化ControllerContext，主要是从zk中读取数据初始化context中的变量，比如liveBrokers，topics，分区和副本，LeadershipInfo等
  7. 初始化ReplicaStateMachine，将所有在活跃broker上的replica的状态变为OnlineReplica
  8. 初始化PartitionStateMachine，将所有leader在活跃broker上的partition的状态设置为Onlinepartition；其他的partition状态为OfflinePartition。Partition是否为Online的标识就是leader是否活着；之后还会触发OfflinePartition 和 NewPartition向OnlinePartition转变，因为OfflinePartition和NewPartition可能是选举leader不成功，所以没有成为OnlinePartition，在环境变化后需要重新触发
  9. 在所有的topic的zookeeper路径/brokers/topics/[topic]/上添加AddPartitionsListener，监听partition变化
```java
  def onControllerFailover() {
      if(isRunning) {
        info("Broker %d starting become controller state transition".format(config.brokerId))
        //从zk当中获取zk，并且更新它
        readControllerEpochFromZookeeper()
        incrementControllerEpoch(zkUtils.zkClient)
        //监听分区、副本、isr信息的状态
        registerReassignedPartitionsListener()
        registerIsrChangeNotificationListener()
        registerPreferredReplicaElectionListener()
        partitionStateMachine.registerListeners()
        replicaStateMachine.registerListeners()
        initializeControllerContext()
        //副本状态机
        replicaStateMachine.startup()
        //分区状态机
        partitionStateMachine.startup()
        // register the partition change listeners for all existing topics on failover
        controllerContext.allTopics.foreach(topic => partitionStateMachine.registerPartitionChangeListener(topic))
        info("Broker %d is ready to serve as the new controller with epoch %d".format(config.brokerId, epoch))
        brokerState.newState(RunningAsController)
        ...       
        deleteTopicManager.start()
      }
    }
```

存储topic的信息的ControllerContext长啥样
```java
  class ControllerContext(val zkUtils: ZkUtils,
                          val zkSessionTimeout: Int) {
    ...
    var epoch: Int = KafkaController.InitialControllerEpoch - 1
    var allTopics: Set[String] = Set.empty
    var partitionReplicaAssignment: mutable.Map[TopicAndPartition, Seq[Int]] = mutable.Map.empty
    var partitionLeadershipInfo: mutable.Map[TopicAndPartition, LeaderIsrAndControllerEpoch] = mutable.Map.empty
    val partitionsBeingReassigned: mutable.Map[TopicAndPartition, ReassignedPartitionsC ontext] = new mutable.HashMap
    val partitionsUndergoingPreferredReplicaElection: mutable.Set[TopicAndPartition] = new mutable.HashSet
    ...
  }
```

----

###kaffaController工作行为

####Create Topic

回顾一下创建topic的方法:
```java
    bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test 
```
这一步通过zk完成，在/brokers/topics/[topic]/路径下写入对应的主题，并且分配分区和副本。


controller 在启动的时候通过注册监听器，监听路径： /brokers/topics/,当zk创建一个新的[topic]znode之后，触发TopicChangeListener
接着controller获取到对应的信息开始执行以下工作: 
```java
    def onNewTopicCreation(topics: Set[String], newPartitions: Set[TopicAndPartition]) {
      info("New topic creation callback for %s".format(newPartitions.mkString(",")))
      // subscribe to partition changes
      //对set集合下的每个topic再次注册监听器, 分区状态修改的监听
      topics.foreach(topic => partitionStateMachine.registerPartitionChangeListener(topic))
      //创建新的分区信息
      onNewPartitionCreation(newPartitions)
    }
```

比较有意思的onNewPartitionCreation方法:
```java
    /**
     * This callback is invoked by the topic change callback with the list of failed brokers as input.
     * It does the following - 状态信息改变
     * 1. Move the newly created partitions to the NewPartition state
     * 2. Move the newly created partitions from NewPartition->OnlinePartition state
     */
    def onNewPartitionCreation(newPartitions: Set[TopicAndPartition]) {
      info("New partition creation callback for %s".format(newPartitions.mkString(",")))
      partitionStateMachine.handleStateChanges(newPartitions, NewPartition)
      replicaStateMachine.handleStateChanges(controllerContext.replicasForPartition(newPartitions), NewReplica)
      partitionStateMachine.handleStateChanges(newPartitions, OnlinePartition, offlinePartitionSelector)
      replicaStateMachine.handleStateChanges(controllerContext.replicasForPartition(newPartitions), OnlineReplica)
    }
```
对于当前topic执行一个状态信息改变:
    1. 创建一个新的分区 & 副本信息状态
      NonExistentPartition-> NewPartition:从zk中读取partition的replica 分配信息到ControllerContext中；
      NonExistentReplica -> NewReplica:在create topic的场景下，仅更新replica的状态为NewReplica；
    2. 修改状态为  Online
      NewPartition->OnlinePartition:选举Leader，这里的Leader选举方法很简单，直接选择活着的replica列表中的第一个作为Leader。
      得到Leader后，向Partition下所有活着的replica发送LeaderAndIsrRequest
      NewReplica->OnlineReplica:在create topic场景下，仅更新replica状态为NewReplica

用一个图总结一下create topic执行过程： 


---

####broker.startup

主要描述broker启动时候对分区和副本状态的影响
在broker启动后调用KafkaController.startup,实际上是创建一个KafkaHealthCheck对象,并调用KafkaHealthCheck->register方法
```java
  def startup() = {
    ...    
    //注册监听重新连接,实际上调用 KafkaHeathCheck.startup
    registerSessionExpirationListener()  
    ...
  }
  ...
  //KafkaHealthCheck
  def register() {
    //把broker的信息:host port 写入 /brokers/ids/[brokerId]/下
    zkUtils.registerBrokerInZk(brokerId, plaintextEndpoint.host, plaintextEndpoint.port, updatedEndpoints, jmxPort, rack,
      interBrokerProtocolVersion)
  }
```
KafkaHealthCheck->register方法将在 /brokers/ids/[brokerId]/路径下创建临时的znode，并且写入broker的必要配置信息，由于是临时的znode，那么在broker发生异常，不可用或者宕机的情况时，zk将会删除这个znode
由于监听的存在，在zk创建znode的时候会触发监听事件的产生BrokerChangeListener.handleChildChange,这个listener存在于ReplicaStateMachine
```java
  /**
   * This is the zookeeper listener that triggers all the state transitions for a replica
   * 触发副本的所有状态转换
   */
  class BrokerChangeListener() extends IZkChildListener with Logging {
    //在controller监听事件发生后产生,直接调用当前方法
    def handleChildChange(parentPath : String, currentBrokerList : java.util.List[String]) {
      inLock(controllerContext.controllerLock) {
        if (hasStarted.get) {
          ControllerStats.leaderElectionTimer.time {
            try {
              //获取当前的brokers列表
              val curBrokers = currentBrokerList.map(_.toInt).toSet.flatMap(zkUtils.getBrokerInfo)
              val curBrokerIds = curBrokers.map(_.id)
              val liveOrShuttingDownBrokerIds = controllerContext.liveOrShuttingDownBrokerIds
              //这一步应该是得到一个新增的broker列表
              val newBrokerIds = curBrokerIds -- liveOrShuttingDownBrokerIds
              //得到对应已经die的brokers
              val deadBrokerIds = liveOrShuttingDownBrokerIds -- curBrokerIds
              val newBrokers = curBrokers.filter(broker => newBrokerIds(broker.id))
              //在controllerContext当中更新liveBrokers，加入新的alive brokers
              controllerContext.liveBrokers = curBrokers
              val newBrokerIdsSorted = newBrokerIds.toSeq.sorted
              val deadBrokerIdsSorted = deadBrokerIds.toSeq.sorted
              val liveBrokerIdsSorted = curBrokerIds.toSeq.sorted
              //这里是添加request channel列表，用于发送request给broker
              newBrokers.foreach(controllerContext.controllerChannelManager.addBroker)
              deadBrokerIds.foreach(controllerContext.controllerChannelManager.removeBroker)
              if(newBrokerIds.size > 0)
                //对新的broker调用 onBrokerStartup
                controller.onBrokerStartup(newBrokerIdsSorted)
              if(deadBrokerIds.size > 0)
                //对死去的调用 onBrokerFailure
                controller.onBrokerFailure(deadBrokerIdsSorted)
            } catch {
              case e: Throwable => error("Error while handling broker changes", e)
            }
          }
        }
      }
    }
  }
```
来看一下 KafkaController.onBrokerStartup 到底干了啥:
```java
  def onBrokerStartup(newBrokers: Seq[Int]) {
    info("New broker startup callback for %s".format(newBrokers.mkString(",")))
    val newBrokersSet = newBrokers.toSet
    //发更新meta信息的Request给brokers，meta信息缓存在broker上，存放所有分区的状态和存活的brokerList
    sendUpdateMetadataRequest(controllerContext.liveOrShuttingDownBrokerIds.toSeq)
    //先broker发送它需要承载的分区和副本
    val allReplicasOnNewBrokers = controllerContext.replicasOnBrokers(newBrokersSet)
    replicaStateMachine.handleStateChanges(allReplicasOnNewBrokers, OnlineReplica)
    //出现新brokr时，触发选举:对象分区是所有新分区 or no leader partition
    partitionStateMachine.triggerOnlinePartitionStateChange()
    // 对于正在执行reassign的partition，调用onPartitionReassignment操作,即重新reassignment
    val partitionsWithReplicasOnNewBrokers = controllerContext.partitionsBeingReassigned.filter {
      case (topicAndPartition, reassignmentContext) => reassignmentContext.newReplicas.exists(newBrokersSet.contains(_))
    }
    partitionsWithReplicasOnNewBrokers.foreach(p => onPartitionReassignment(p._1, p._2))
    //判断新启动的broker上是否存在需要被删除的topic replica，如果存在，删除
    val replicasForTopicsToBeDeleted = allReplicasOnNewBrokers.filter(p => deleteTopicManager.isTopicQueuedUpForDeletion(p.topic))
    if(replicasForTopicsToBeDeleted.size > 0) {
      info(("Some replicas %s for topics scheduled for deletion %s are on the newly restarted brokers %s. " +
        "Signaling restart of topic deletion for these topics").format(replicasForTopicsToBeDeleted.mkString(","),
        deleteTopicManager.topicsToBeDeleted.mkString(","), newBrokers.mkString(",")))
      deleteTopicManager.resumeDeletionForTopics(replicasForTopicsToBeDeleted.map(_.topic))
    }
  }
```
总结
- 向newBrokers发送UpdateMetadataRequest，MetadataCache对象存在于每个broker上，存放集群中所有Partition的state信息和aliveBrokerList;
- 将newBrokers上的所有replica状态变为OnlineReplica，在broker下线时，相应的replica状态变为OfflineReplica。OfflineReplica->OnlineReplica过程中会向replica发送LeaderAndIsrRequest，收到消息后replica开始向leader同步log，在同步完成后成为新的ISR；
- 触发处于NewPartition和OfflinePartition状态的向OnlinePartition转变；NewPartition->OnlinePartition 和 OfflinePartition->OnlinePartition都有可能因为没有活着的broker而状态转换失败，有新的broker上线就要尝试上线；
- 对于正在执行reassign的partition，调用onPartitionReassignment操作；

---

####broker.failover

在broker宕机之后，zk当中的临时znode被删除，上文的handleChildChange -> onBrokerFailure方法
```java
  def onBrokerFailure(deadBrokers: Seq[Int]) {
    // trigger OfflinePartition state for all partitions whose current leader is one amongst the dead brokers
    //如果shutDown的broker是leader节点，把所有分区设置为 offlinePartition,这里是获取broker的分区
    val partitionsWithoutLeader = controllerContext.partitionLeadershipInfo.filter(partitionAndLeader =>
      deadBrokersSet.contains(partitionAndLeader._2.leaderAndIsr.leader) &&
        !deleteTopicManager.isTopicQueuedUpForDeletion(partitionAndLeader._1.topic)).keySet
    //触发分区下线规则:条件是leader下线
    partitionStateMachine.handleStateChanges(partitionsWithoutLeader, OfflinePartition)
    //这一步触发重新选举，重新选举 -> offlinePartition -> newPartition/onlinePartition
    partitionStateMachine.triggerOnlinePartitionStateChange()
    //过滤掉需要删除的主题于副本
    var allReplicasOnDeadBrokers = controllerContext.replicasOnBrokers(deadBrokersSet)
    val activeReplicasOnDeadBrokers = allReplicasOnDeadBrokers.filterNot(p => deleteTopicManager.isTopicQueuedUpForDeletion(p.topic))
    //对于已经shutDown的broker，执行一个副本下线操作，状态变更为offlineReplica
    replicaStateMachine.handleStateChanges(activeReplicasOnDeadBrokers, OfflineReplica)
    ...
  }
```
在上面的代码中，broker下线时执行的主要操作有： 
1. 对Leader副本在deadBrokers上的Partition执行下线操作，即Partition状态变为OfflinePartition。Partition是否上线的关键就是leader副本是否在线； 
2. 触发 NewPartition/OfflinePartition -> OnlinePartition，上一步Partition下线后，通过这个操作来重新选择Leader，新的Leader诞生后，向所有对应broker发送LeaderAndIsrRequest，无论是生产还是消费都可以无缝切换；
当然，选择Leader有可能失败，如没有replica在线的情况，这是Partition就一直处于OfflinePartition状态，直到有Broker上线； 
3. 将下线的broker上的所有replica切换为OfflineReplica状态，这个过程会向replica所在broker发送StopReplicaRequest消息，并将replica从相应partition的ISR中移除,如果移除的副本是leader,触发重新选举leader，最后再给partition剩下的replica发送LeaderAndIsrRequest；

---
###kaffaController PartStateMachine & ReplicaStateMachine 实现讲解
####PartitionStateMachine的状态转换机制过程
  注:更新分区的信息之后都需要在zk当中更新以及controllerContext更新
![分区状态机](http://cf.meitu.com/confluence/download/attachments/63432943/%E5%88%86%E5%8C%BA%E7%8A%B6%E6%80%81.png?version=2&modificationDate=1532433432371&api=v2)
*根据PartitionStateMachine -> handleStateChange方法总结而来*

---

####ReplicaStateMachine的状态转换机制过程

ReplicaStateMachine的七种状态(副本状态相当复杂):
    NewReplica: 在partition reassignment期间KafkaController创建New replica;
    OnlineReplica: 当一个replica变为一个parition的assingned replicas时, 其状态变为OnlineReplica, 即一个有效的OnlineReplica. Online状态的parition才能转变为leader或isr中的一员;
    OfflineReplica: 当一个broker down时, 上面的replica也不可用, 其状态转变为Onffline;
    ReplicaDeletionStarted: 当一个replica的删除操作开始时,其状态转变为ReplicaDeletionStarted;
    ReplicaDeletionSuccessful: Replica成功删除后,其状态转变为ReplicaDeletionSuccessful;
    ReplicaDeletionIneligible: Replica删除失败后,其状态转变为ReplicaDeletionIneligible;
    NonExistentReplica: Replica成功删除后, 从ReplicaDeletionSuccessful状态转变为NonExistentReplica状态.


*根据PartitionStateMachine -> handleStateChange方法总结而来*

leader 是针对 副本而言的
一个topic 存在多个分区，多个副本，同一个分区的数量，取决于副本的数量
leader副本 拥有全部的分区，其他的副本会向leader同步数据
leader副本下线，会触发一个重新选举的过程
