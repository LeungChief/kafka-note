# kafka知识点
### 以下问题出处在orchome，我仅用来对知识点巩固总结用

> ### kafka节点之间如何复制备份的
    
    
    前提知识点：
        什么是节点?
            在分布式部署的情况下，一台机子就是一个节点.简称broke_id.

        节点如何保证高可用?
            Kafka提供了备份机制， 一个备份数量为n的集群允许n-1个节点失败( 宕机 )，在所有备份节点中，有一个节点作为lead节点，这个节点保存了其它备份节点列表，并维持各个备份间的状体同步。
    
    几个概念:
        ISR:   kafka维护的一个副本维护队列，ISR是in-sync replicas的简写。ISR的副本保持和leader的同步，当然leader本身也在ISR中
        HW：high watermark，是指ISR中所有节点都已经复制完的消息的offset。也是消费者所能获取到的消息的最大offset。
        LEO：LogEndOffset，表示每个分区log的最后一条消息的offset
        fully replicated：全量同步
        
        

    流程:
        product发布消息.消息先经过分区leader，leader广播给所有ISR中的follower，只要过半数响应，这条消息就会进行广播并写入响应的ISR副本中，消费端就可消费这条消息了，并未响应的follower，leader会将其从ISR中移除。假如消息还未开始广播，leader挂了，ISR中的follower会选举出一个新leader并进行广播，假如全都广播失败，则消息发送失败

    额外知识点：
        当leader剔除的follower重启后，会从最新的leader中拉去最新HW并同步数据，同步完毕后系统重回fully replicated状态
    
<br><br>
    
> ### kafka消息是否会丢失？为什么？

    前提知识点：
        product.acks，0代表：不进行消息接收是否成功的确认(默认值)；1代表：当Leader副本接收成功后，返回接收成功确认信息；-1代表：当Leader和Follower副本都接收成功后，返回接收成功确认信息；
        product.buffer.memory ， 缓冲等待被发送到服务器的记录的总字节数
        broker.message.max.bytes， kafka允许的最大的一个批次的消息大小。 如果这个数字增加，且有0.10.2版本以下的consumer，那么consumer的提取大小也必须增加，以便他们可以取得这么大的记录批次。 在最新的消息格    式版本中，记录总是被组合到一个批次以提高效率。 在以前的消息格式版本中，未                                                          压缩的记录不会分组到批次中，并且此限制仅适用于该情况下的单个记录
    
    概念:
        producter分同步发送及异步发送， consumer接口有Low-level API和High-level API

    丢失场景:
        acks不为-1或all均有丢失风险
        异步发送时，Client端缓存的消息超出了缓冲池的大小
        使用High-level API，可能存在一个问题就是当消息消费者从集群中把消息取出来、并提交了新的消息offset值后，还没来得及消费就挂掉了，那么下次再消费时会按照最新提交的offset消费，挂掉之前未消费的数据就丢失了
        单批次消息大小过大，超过 topic.max.message.bytes 或 consumer.fetch.max.bytes与topic.max.message.bytes不匹配
        。。。。(省略N个场景)
    
    解决方案:
        需根据数据重要性及数据的最大值来配置参数：product.buffer.memory、broker.message.max.bytes、topic.max.message.bytes、consumer.fetch.max.bytes
        重要数据建议acks设置为-1

 <br> <br>
> ### kafka的leader选举机制是什么？
    
    前提知识点:
        kafka的leader选举有三大类：控制器选举、分区leader选举、消费者相关选举
        1.控制器选举:
            依赖zookeeper，而zk的选举机制是Paxos算法计算，提及一下zk的分布式锁机制， 一个是保持独占，另一个是控制时序，我认为kafka是保持独占模式的；broker相当于zk上的临时节点，所有的broker都会预先在zk设置一个watch，一旦leader宕机，watch通知zk删除对应的临时节点，而其他的broker最快在zk创建临时节点成功的就成为了控制器保持独占

        2.分区leader选举：
             从AR列表(分区所有副本)中找到第一个存活的副本，且这个副本在目前的ISR列表中，与此同时还要确保这个副本不处于正在被关闭的节点上
        
        3.消费者选举：
             消费者组管理是由 coordinator 这个角色来管理；组内没有leader，那么第一个加入消费组的消费者即为leader；原leader退出消费组，随机选一个消费者作为leader
 <br> <br>

> ### kafka的消息保证有几种方式?

    1.幂等
    2.事务
    
 <br> <br>
> ### Kafka中的分区分配

    有3种:1.客户端 2.消费端 3.broker端
    
    1.product发送消息会触发分区分配动作，消息分配去哪个分区，根据key字段的hash值来决定分区，拥有相同key的会分配的同一分区，而一般key为null，则轮询方式往分区发送
    2.https://blog.csdn.net/feelwing1314/article/details/81097167
    3.创建topic时创建指定分区数量、副本集数量，分配哪个ISR成员作为分区副本leader

<br>
