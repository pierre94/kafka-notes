# `__consumer_offsets`部分分区异常导致消费不到数据问题排查

> 记一次kafka消费异常问题的排查


## 一、问题描述

### 问题描述
部分消费组无法通过broker(new-consumer)正常消费数据,更改消费组名后恢复正常。

group名(可能涉及业务信息，group名非真实名):
- `group1-打马赛克`
- `group2-打马赛克`

kafka版本:
0.9.0.1

## 二、简单分析

### 1、describe对应消费组

describe对应消费组时抛如下异常:
```
Error while executing consumer group command The group coordinator is not available.
org.apache.kafka.common.errors.GroupCoordinatorNotAvailableException: The group coordinator is not available.
```

### 2、问题搜索

搜索到业界有类似问题,不过都没有解释清楚为什么出现这种问题，以及如何彻底解决(重启不算)!

- http://apache-flink-user-mailing-list-archive.2336050.n4.nabble.com/Problem-with-Kafka-0-9-Client-td4975.html

## 三、深入分析
日志是程序员的第一手分析资料。Kafka服务端因为现网有大量服务在运营，不适合开启debug日志，所以我们只能从客户端入手。

### 1、开启客户端debug日志
将客户端日志等级开成debug级别,发现持续循环地滚动如下日志:
```
19:52:41.785 TKD [main] DEBUG o.a.k.c.c.i.AbstractCoordinator - Issuing group metadata request to broker 43
19:52:41.788 TKD [main] DEBUG o.a.k.c.c.i.AbstractCoordinator - Group metadata response ClientResponse(receivedTimeMs=1587642761788, disconnected=false, request=ClientRequest(expectResponse=true, callback=org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient$RequestFutureCompletionHandler@1b68ddbd, request=RequestSend(header={api_key=10,api_version=0,correlation_id=30,client_id=consumer-1}, body={group_id=30cab231-05ed-43ef-96aa-a3ca1564baa3}), createdTimeMs=1587642761785, sendTimeMs=1587642761785), responseBody={error_code=15,coordinator={node_id=-1,host=,port=-1}})
19:52:41.875 TKD [main] DEBUG o.apache.kafka.clients.NetworkClient - Sending metadata request ClientRequest(expectResponse=true, callback=null, request=RequestSend(header={api_key=3,api_version=0,correlation_id=31,client_id=consumer-1}, body={topics=[topic打马赛克]}), isInitiatedByNetworkClient, createdTimeMs=1587642761875, sendTimeMs=0) to node 43
```
我们大致可以看出循环在做着几件事情(先后不一定准确):
- 从某个broker `Issuing group metadata request`
- 获取`Group metadata`
- 发起`metadata request`

我们聚焦到获取`Group metadata`的error关键字`responseBody={error_code=15,coordinator={node_id=-1,host=,port=-1}}`,大致得出是kafka服务端没有给出coordinator的node结点信息。

### 2、服务端如何响应请求

#### 请求对应的入口函数

首先我们需要查看```api_key=10```请求对应的服务端源码:

需要从`kafka.server.KafkaApis`中寻找对应的api接口函数

```java
  def handle(request: RequestChannel.Request) {
  ……
        case RequestKeys.GroupCoordinatorKey => handleGroupCoordinatorRequest(request)
  ……
    }
```
#### handleGroupCoordinatorRequest逻辑
```java
  def handleGroupCoordinatorRequest(request: RequestChannel.Request) {
    val groupCoordinatorRequest = request.body.asInstanceOf[GroupCoordinatorRequest]
    val responseHeader = new ResponseHeader(request.header.correlationId)

    if (!authorize(request.session, Describe, new Resource(Group, groupCoordinatorRequest.groupId))) {
      val responseBody = new GroupCoordinatorResponse(Errors.GROUP_AUTHORIZATION_FAILED.code, Node.noNode)
      requestChannel.sendResponse(new RequestChannel.Response(request, new ResponseSend(request.connectionId, responseHeader, responseBody)))
    } else {
      val partition = coordinator.partitionFor(groupCoordinatorRequest.groupId)

      // get metadata (and create the topic if necessary)
      val offsetsTopicMetadata = getOrCreateGroupMetadataTopic(request.securityProtocol)
        // 第一个可能存在的问题:offsetsTopicMetadata的errCode不为空
      val responseBody = if (offsetsTopicMetadata.errorCode != Errors.NONE.code) {
        new GroupCoordinatorResponse(Errors.GROUP_COORDINATOR_NOT_AVAILABLE.code, Node.noNode)
      } else {
        val coordinatorEndpoint = offsetsTopicMetadata.partitionsMetadata
          .find(_.partitionId == partition)
          .flatMap {
            partitionMetadata => partitionMetadata.leader
          }
        // 第二个可能存在的问题:coordinatorEndpoint为空
        coordinatorEndpoint match {
          case Some(endpoint) =>
            new GroupCoordinatorResponse(Errors.NONE.code, new Node(endpoint.id, endpoint.host, endpoint.port))
          case _ =>
            new GroupCoordinatorResponse(Errors.GROUP_COORDINATOR_NOT_AVAILABLE.code, Node.noNode)
        }
      }

      trace("Sending consumer metadata %s for correlation id %d to client %s."
        .format(responseBody, request.header.correlationId, request.header.clientId))
      requestChannel.sendResponse(new RequestChannel.Response(request, new ResponseSend(request.connectionId, responseHeader, responseBody)))
    }
  }
  ```
  其中`error_code=15`对应的是`Errors.GROUP_COORDINATOR_NOT_AVAILABLE.code`
  
  从源码不难看出，导致`Errors.GROUP_COORDINATOR_NOT_AVAILABLE.code`可能点有二:
  
 - 疑似问题点一:offsetsTopicMetadata的errCode不为空
      ```java
      offsetsTopicMetadata.errorCode != Errors.NONE.code
      ```
    offsetsTopicMetadata的errCode不为空,意味着整个`__consumer_offsets`元数据获取都有问题。但是现场只是部分group有问题,这里出问题的可能性不大。
  
 - 疑似问题点二:coordinatorEndpoint为空
      ```java
     val coordinatorEndpoint = offsetsTopicMetadata.partitionsMetadata
              .find(_.partitionId == partition)
              .flatMap {
                partitionMetadata => partitionMetadata.leader
              }
      ```
      从`offsetsTopicMetadata`获取到的元数据，过滤出`coordinator.partitionFor(groupCoordinatorRequest.groupId)`分区的leader。而`coordinator.partitionFor(groupCoordinatorRequest.groupId)`正是与group名相关!这里出问题的可能性极大!
  
#### partitionFor相关的逻辑:
```java
  def partitionFor(groupId: String): Int = Utils.abs(groupId.hashCode) % groupMetadataTopicPartitionCount
```
即取group名的正hashCode模`groupMetadataTopicPartitionCount`(即`__consumer_offsets`对应的分区数)。 

注:可能涉及业务信息，group名非真实名。而结果是正式group名算出的结果。

```java
scala> "group1-打马赛克".hashCode % 50
res2: Int = 43

scala> "group2-打马赛克".hashCode % 50
res3: Int = 43
```
我们发现2个异常的消费组,其`partitionFor`后的值均为43,我们初步判断分区可能与`__consumer_offsets`的43分区相关! 
接下来我们就要看下`offsetsTopicMetadata`相关的逻辑,来确认异常。

#### offsetsTopicMetadata的逻辑
```java
val offsetsTopicMetadata = getOrCreateGroupMetadataTopic(request.securityProtocol)
```
`getOrCreateGroupMetadataTopic` -> `metadataCache.getTopicMetadata` -> `getPartitionMetadata`

```java
  private def getPartitionMetadata(topic: String, protocol: SecurityProtocol): Option[Iterable[PartitionMetadata]] = {
    cache.get(topic).map { partitions =>
      partitions.map { case (partitionId, partitionState) =>
        val topicPartition = TopicAndPartition(topic, partitionId)

        val leaderAndIsr = partitionState.leaderIsrAndControllerEpoch.leaderAndIsr
        val maybeLeader = aliveBrokers.get(leaderAndIsr.leader)

        val replicas = partitionState.allReplicas
        val replicaInfo = getAliveEndpoints(replicas, protocol)

        maybeLeader match {
          case None =>
            debug("Error while fetching metadata for %s: leader not available".format(topicPartition))
            new PartitionMetadata(partitionId, None, replicaInfo, Seq.empty[BrokerEndPoint],
              Errors.LEADER_NOT_AVAILABLE.code)

          case Some(leader) =>
            val isr = leaderAndIsr.isr
            val isrInfo = getAliveEndpoints(isr, protocol)

            if (replicaInfo.size < replicas.size) {
              debug("Error while fetching metadata for %s: replica information not available for following brokers %s"
                .format(topicPartition, replicas.filterNot(replicaInfo.map(_.id).contains).mkString(",")))

              new PartitionMetadata(partitionId, Some(leader.getBrokerEndPoint(protocol)), replicaInfo, isrInfo, Errors.REPLICA_NOT_AVAILABLE.code)
            } else if (isrInfo.size < isr.size) {
              debug("Error while fetching metadata for %s: in sync replica information not available for following brokers %s"
                .format(topicPartition, isr.filterNot(isrInfo.map(_.id).contains).mkString(",")))
              new PartitionMetadata(partitionId, Some(leader.getBrokerEndPoint(protocol)), replicaInfo, isrInfo, Errors.REPLICA_NOT_AVAILABLE.code)
            } else {
              new PartitionMetadata(partitionId, Some(leader.getBrokerEndPoint(protocol)), replicaInfo, isrInfo, Errors.NONE.code)
            }
        }
      }
    }
  }
```
offsetsTopicMetadata即对于topic下所有leader、replicaInfo、isr正常分区的元数据信息,所以我们判断`__consumer_offsets` 43分区leader、replicaInfo、isr等可能存在异常,导致`find(_.partitionId == partition)`时找不到根据hashCode取模后对应的分区。

## 四、回到现网  
### 1、`__consumer_offsets`分区信息验证
```shell
Topic:__consumer_offsets        PartitionCount:50       ReplicationFactor:3     Configs:segment.bytes=104857600,cleanup.policy=compact,compression.type=uncompressed
        Topic: __consumer_offsets       Partition: 0    Leader: 18      Replicas: 18,2,17       Isr: 18
        Topic: __consumer_offsets       Partition: 1    Leader: 19      Replicas: 19,17,18      Isr: 19,18
        Topic: __consumer_offsets       Partition: 2    Leader: 20      Replicas: 20,18,19      Isr: 19,20,18
        Topic: __consumer_offsets       Partition: 3    Leader: 21      Replicas: 21,19,20      Isr: 19,20,21
        Topic: __consumer_offsets       Partition: 4    Leader: 22      Replicas: 22,20,21      Isr: 20,21,22
        Topic: __consumer_offsets       Partition: 5    Leader: 23      Replicas: 23,21,22      Isr: 23,21,22
        Topic: __consumer_offsets       Partition: 6    Leader: 24      Replicas: 24,22,23      Isr: 23,24,22
        Topic: __consumer_offsets       Partition: 7    Leader: 25      Replicas: 25,23,24      Isr: 23,25,24
        Topic: __consumer_offsets       Partition: 8    Leader: 26      Replicas: 26,24,25      Isr: 26,25,24
        Topic: __consumer_offsets       Partition: 9    Leader: 27      Replicas: 27,25,26      Isr: 27,26,25
        Topic: __consumer_offsets       Partition: 10   Leader: 28      Replicas: 28,26,27      Isr: 27,26,28
        Topic: __consumer_offsets       Partition: 11   Leader: 27      Replicas: 0,27,28       Isr: 27,28
        Topic: __consumer_offsets       Partition: 12   Leader: 28      Replicas: 1,28,0        Isr: 28
        Topic: __consumer_offsets       Partition: 13   Leader: -1      Replicas: 2,0,1 Isr: 
        Topic: __consumer_offsets       Partition: 14   Leader: -1      Replicas: 3,1,2 Isr: 
        Topic: __consumer_offsets       Partition: 15   Leader: -1      Replicas: 4,2,3 Isr: 
        Topic: __consumer_offsets       Partition: 16   Leader: -1      Replicas: 5,3,4 Isr: 
        Topic: __consumer_offsets       Partition: 17   Leader: -1      Replicas: 6,4,5 Isr: 
        Topic: __consumer_offsets       Partition: 18   Leader: -1      Replicas: 7,5,6 Isr: 
        Topic: __consumer_offsets       Partition: 19   Leader: 8       Replicas: 8,6,7 Isr: 8
        Topic: __consumer_offsets       Partition: 20   Leader: 9       Replicas: 9,7,8 Isr: 9,8
        Topic: __consumer_offsets       Partition: 21   Leader: 10      Replicas: 10,8,9        Isr: 10,8,9
        Topic: __consumer_offsets       Partition: 22   Leader: 11      Replicas: 11,9,10       Isr: 11,10,9
        Topic: __consumer_offsets       Partition: 23   Leader: 12      Replicas: 12,10,11      Isr: 11,12,10
        Topic: __consumer_offsets       Partition: 24   Leader: 13      Replicas: 13,11,12      Isr: 13,11,12
        Topic: __consumer_offsets       Partition: 25   Leader: 14      Replicas: 14,12,13      Isr: 13,12,14
        Topic: __consumer_offsets       Partition: 26   Leader: 15      Replicas: 15,13,14      Isr: 13,14,15
        Topic: __consumer_offsets       Partition: 27   Leader: 16      Replicas: 16,14,15      Isr: 14,16,15
        Topic: __consumer_offsets       Partition: 28   Leader: 42      Replicas: 17,15,42      Isr: 42,15
        Topic: __consumer_offsets       Partition: 29   Leader: 18      Replicas: 18,17,19      Isr: 19,18
        Topic: __consumer_offsets       Partition: 30   Leader: 19      Replicas: 19,18,20      Isr: 19,20,18
        Topic: __consumer_offsets       Partition: 31   Leader: 20      Replicas: 20,19,21      Isr: 19,20,21
        Topic: __consumer_offsets       Partition: 32   Leader: 21      Replicas: 21,20,22      Isr: 20,21,22
        Topic: __consumer_offsets       Partition: 33   Leader: 22      Replicas: 22,21,23      Isr: 23,21,22
        Topic: __consumer_offsets       Partition: 34   Leader: 23      Replicas: 23,22,24      Isr: 23,24,22
        Topic: __consumer_offsets       Partition: 35   Leader: 24      Replicas: 24,23,25      Isr: 23,25,24
        Topic: __consumer_offsets       Partition: 36   Leader: 25      Replicas: 25,24,26      Isr: 26,25,24
        Topic: __consumer_offsets       Partition: 37   Leader: 26      Replicas: 26,25,27      Isr: 27,26,25
        Topic: __consumer_offsets       Partition: 38   Leader: 27      Replicas: 27,26,28      Isr: 27,26,28
        Topic: __consumer_offsets       Partition: 39   Leader: 28      Replicas: 28,27,0       Isr: 27,28
        Topic: __consumer_offsets       Partition: 40   Leader: 28      Replicas: 0,28,1        Isr: 28
        Topic: __consumer_offsets       Partition: 41   Leader: -1      Replicas: 1,0,2 Isr: 
        Topic: __consumer_offsets       Partition: 42   Leader: -1      Replicas: 2,1,3 Isr: 
        Topic: __consumer_offsets       Partition: 43   Leader: -1      Replicas: 3,2,4 Isr: 
        Topic: __consumer_offsets       Partition: 44   Leader: -1      Replicas: 4,3,5 Isr: 
        Topic: __consumer_offsets       Partition: 45   Leader: -1      Replicas: 5,4,6 Isr: 
        Topic: __consumer_offsets       Partition: 46   Leader: -1      Replicas: 6,5,7 Isr: 
        Topic: __consumer_offsets       Partition: 47   Leader: 8       Replicas: 7,6,8 Isr: 8
        Topic: __consumer_offsets       Partition: 48   Leader: 8       Replicas: 8,7,9 Isr: 9,8
        Topic: __consumer_offsets       Partition: 49   Leader: 9       Replicas: 9,8,10        Isr: 10,9,8
```

43分区果然存在leader异常的情况


### 2、问题复现
我们使用UUID批量生成消费组名,使其hashCode取模后为异常分区的分区号,再使用其进行消费时均出现消费异常的问题。

### 3、问题思考
- 为什么`__consumer_offsets`部分分区会产生leader、replicaInfo、isr异常? 
    与网络抖动和一些集群操作可能有关，需要具体问题具体分析
    
- 如何将`__consumer_offsets`异常分区恢复正常？         
    这里不详细介绍可以参考http://blog.itpub.net/31543630/viewspace-2212467/ 。

## 五、参考资料
- Kafka new-consumer设计文档  https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Detailed+Consumer+Coordinator+Design 

- Kafka无法消费?!我的分布式消息服务Kafka却稳如泰山！ http://blog.itpub.net/31543630/viewspace-2212467/

- Problem with Kafka 0.9 Client http://apache-flink-user-mailing-list-archive.2336050.n4.nabble.com/Problem-with-Kafka-0-9-Client-td4975.html

- ErrorMapping https://github.com/apache/kafka/blob/0.9.0/core/src/main/scala/kafka/common/ErrorMapping.scala
