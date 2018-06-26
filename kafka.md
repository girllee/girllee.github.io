Kafka Topic
## 创建topic

bin/kafka-topics.sh --zookeeper localhost:2181 --create --topic my-topic --partitions 3
        --replication-factor 3 --config max.message.bytes=64000 --config flush.messages=1

## 查看topic

bin/kafka-topics.sh --zookeeper zk0:2181,zk1:2181,zk2:2181 --describe
可以看到每一个topic的分区数，复制因子数，Configs，Leader


## 修改topic

### 修改分区数
bin/kafka-topics.sh --zookeeper zk0:2181,zk1:2181/kafka --alter --topic myTopic --partitions 10
Note: 分区数只能增加不能减少！！



### 重新选举
bin/kafka-preferred-replica-election.sh --zookeeper zk0:2181,zk1:2181,zk2:2181

####选举指定的内容
1. 创建election.json 内容如下：
{
  "partitions" :
  [
    {"topic":"goldCoin-prod","partition":0}
  ]
}

2. 执行如下命令：
bin/kafka-preferred-replica-election.sh --zookeeper zk0:2181,zk1:2181,zk2:2181 election.json

### 增加replicas
创建时指定的Topic gold的replication-factor 为1，系统运行后需要增加replicas.此时不能使用bin/kafka-topic.sh 命令来修改！！
1. 创建replicas.json ,内容如下：
{
    "partitions":
    [
    {
    "topic":"market-ws-dev",
    "partition": 0,
    "replicas": [0,1]
    },
    {
    "topic": "market-ws-dev",
    "partition": 1,
    "replicas": [1,2]
    },
    {
    "topic": "market-ws-dev",
    "partition": 2,
    "replicas": [2,1]
    }
    ],
    "version":1
}

2.执行如下命令：
bin/kafka-reassign-partitions.sh --zookeeper zk0:2181,zk1:2181,zk2:2181 --reassignment-json-file ./replicas.json --execute



## 删除topic

When Delete

### How To Delete

#### 前提条件： 
在启动broker时候开启删除topic的开关，即在server.properties中添加：  delete.topic.enable=true

命令： bin/kafka-topics.sh --zookeeper zk_host:port/chroot --delete --topic my_topic_name

这条命令其实就是在zookeeper（假设你的chroot就是/）的/admin/delete_topics下创建一个临时节点，名字就是topic名称，比如如果执行命令：

bin/kafka-topics.sh --zookeeper zk_host:port/chroot --delete --topic test-topic

那么，命令返回后，zookeeper的/admin/delete_topics目录下会新创建一个临时节点test-topic 

这条命令返回打印在控制台上的消息也说明了这点：

Topic test-topic is marked for deletion.

Note: This will have no impact if delete.topic.enable is not set to true.

这就是说，这条命令其实并不执行删除动作，仅仅是在zookeeper上标记该topic要被删除而已，同时也提醒用户一定要提前打开delete.topic.enable开关，否则删除动作是不会执行的。

那么，我们通过命令标记了test-topic要被删除之后Kafka是怎么执行删除操作的呢？ 总的流程如下图所示：

1. Kafka controller在启动的时候会注册对于Zookeeper节点/admin/delete_topics的子节点变更监听器——上面的分析已经告诉我们，
delete命令实际上就是要在该节点下创建一个临时节点，名字是待删除topic名，标记该topic是待删除的
2. Kafka controller在启动时创建一个单独的线程，执行topic删除的操作 (由DeleteTopicsThread类实现)
3. 线程启动时查看是否有需要进行删除的topic——假设我们是在controller启动之后执行的topic删除命令，那么该线程刚启动的时候待删除的topic集合
应该就是空的
4. 一旦发现待删除topic集合是空，topic删除线程会被挂起
5. 这时，我们执行delete操作，删除topic： test-topic，delete命令会在/admin/delete_topics节点下创建子节点test-topic
6. 监听器捕获到该变更，立刻触发删除逻辑
  * 查询test-topic是否存在，不存在的话直接删除/admin/delete_topics/test-topic节点——随便删除一个不存在的topic，
  删除命令也只是创建/admin/delete_topics/[topicName]的节点，它不负责做存在性校验 
  * 查询一下test-topic是不是当前正在执行Preferred副本选举或分区重分配，如果是的话，肯定是不适合进行删除掉的。Controller本地会缓存当前无法
  进行删除的topic集合，待分区重分配完成或preferred副本选举后单独处理该集合中的topic
  * 如何两者都不是的话说明现在可以进行删除操作，那么就恢复挂起的删除线程执行删除操作
  
  删除线程执行删除操作的真正逻辑是：
  
    1. 首先会给当前所有broker发送更新元数据信息的请求，告诉这些broker说这个topic要删除了，你们可以把它的信息从缓存中删掉了
    2. 开始删除这个topic的所有分区
        * 给所有broker发请求，告诉它们这些分区要被删除。broker收到后就不再接受任何在这些分区上的客户端请求了
        * 把每个分区下的所有副本都置于OfflineReplica状态，这样ISR就不断缩小，当leader副本最后也被置于OfflineReplica状态时leader信息
        将被更新为-1
        * 将所有副本置于ReplicaDeletionStarted状态
        * 副本状态机捕获状态变更，然后发起StopReplicaRequest给broker，broker接到请求后停止所有fetcher线程、移除缓存，然后删除底层log文件
        * 关闭所有空闲的Fetcher线程
    3. 删除zookeeper下/brokers/topics/test-topic节点
    4. 删除zookeeper下/config/topics/test-topic节点
    5. 删除zookeeper下/admin/delete_topics/test-topic节点
    6. 更新各种缓存，把test-topic相关信息移除出去
