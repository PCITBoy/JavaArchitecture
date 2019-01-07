##Kafka
有时候，我们学习了一些东西，过了一段时间，又忘记了。只记得当时学过。那么再次回顾的时候，最好能带上几个问题，这样才能让自己记忆的更加清晰。

今日我们来回顾Kafka消息中间件，那么下边是我提出的几个问题，大家看看有几个是能回答上来的。

- 1、**kafka节点之间如何复制备份的？**

- 2、**kafka消息是否会丢失？为什么？**
- 3、**kafka最合理的配置是什么？**
- 4、**kafka的leader选举机制是什么？**
- 5、**kafka对硬件的配置有什么要求？**
- 6、**kafka的消息保证有几种方式？**

![](https://www.icheesedu.com/images/qiniu/%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0-2.png)


##1、简介
 -  Kafka 是分布式发布-订阅消息系统。它最初由 LinkedIn 公司开发，使用 Scala语言编写,之后成为 Apache 项目的一部分。Kafka 是一个分布式的，可划分的，多订阅者,冗余备份的持久性的日志服务。它主要用于处理活跃的流式数据。
 
 -  消息队列的性能好坏，其文件存储机制设计是衡量一个消息队列服务技术水平和最关键指标之一。下面将从Kafka文件存储机制和物理结构角度，分析Kafka是如何实现高效文件存储，及实际应用效果。

##2、 Kafka的特点:

 - **高吞吐量、低延迟**：kafka每秒可以处理几十万条消息，它的延迟最低只有几毫秒，每个topic可以分多个partition, consumer group 对partition进行consume操作。
 
- **可扩展性**：kafka集群支持热扩展
- **持久性、可靠性**：消息被持久化到本地磁盘，并且支持数据备份防止数据丢失
- **容错性**：允许集群中节点失败（若副本数量为n,则允许n-1个节点失败）
- **高并发**：支持数千个客户端同时读写


##3、 Kafka的使用场景：
- **日志收集**：一个公司可以用Kafka可以收集各种服务的log，通过kafka以统一接口服务的方式开放给各种consumer，例如hadoop、Hbase、Solr等。

- **消息系统**：解耦和生产者和消费者、缓存消息等。
- **用户活动跟踪**：Kafka经常被用来记录web用户或者app用户的各种活动，如浏览网页、搜索、点击等活动，这些活动信息被各个服务器发布到kafka的topic中，然后订阅者通过订阅这些topic来做实时的监控分析，或者装载到hadoop、数据仓库中做离线分析和挖掘。
- **运营指标**：Kafka也经常用来记录运营监控数据。包括收集各种分布式应用的数据，生产各种操作的集中反馈，比如报警和报告。
- **流式处理**：比如spark streaming和storm
- **事件源**

　　
##4、架构

![](https://www.icheesedu.com/images/qiniu/Xnip2018-07-190_11-13-31.png)

![](https://www.icheesedu.com/images/qiniu/Xnip2018-07-190_11-13-45.png)

##5、Kafka专用术语：
 
- **Broker**：消息中间件处理结点，一个Kafka节点就是一个broker，多个broker可以组成一个Kafka集群。

- **Topic**：一类消息，Kafka集群能够同时负责多个topic的分发。
   - Topic是用于存储消息的逻辑概念，可以看作一个消息集合。每个topic可以有多个生产者向其推送消息，也可以有任意多个消费者消费其中的消息
   

- **Partition**：topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列。

     - Partition是以文件的形式存储在文件系统中，存储在kafka-log目录下，命名规则是：`<topic_name>-<partition_id>`

- **Segment**：partition物理上由多个segment组成。

- **offset**：每个partition都由一系列有序的、不可变的消息组成，这些消息被连续的追加到partition中。partition中的每个消息都有一个连续的序列号叫做offset，用于partition唯一标识一条消息
- **消息**


- 4.1、topic & partition
   - 在Kafka文件存储中，同一个topic下有多个不同partition，每个partition为一个目录，partiton命名规则为topic名称+有序序号，第一个partiton序号从0开始，序号最大值为partitions数量减1。

   - 这里也就是broker——>topic——>partition——>segment 

   - segment file组成：由2大部分组成，分别为index file和data file，此2个文件一一对应，成对出现，后缀".index"和“.log”分别表示为segment索引文件、数据文件。
 
     ![](https://www.icheesedu.com/images/qiniu/932932-20170818102025709-1227763455.jpg.png)
     
     　segment文件命名规则：partion全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值。数值最大为64位long大小，19位数字字符长度，没有数字用0填充。
     　
     　segment中index与data file对应关系物理结构如下：
     　
     　![](https://www.icheesedu.com/images/qiniu/932932-20170818102555537-956883187.jpg.png)
 
    索引文件存储大量元数据，数据文件存储大量消息，索引文件中元数据指向对应数据文件中message的物理偏移地址。

　　 其中以索引文件中元数据3,497为例，依次在数据文件中表示第3个message（在全局partiton表示第368772个message），以及该消息的物理偏移地址为497。
　　 
　　 ![](https://www.icheesedu.com/images/qiniu/Xnip2018-07-190_15-34-57.png)
　　 
##6、kafka高吞吐量的因素

![](https://www.icheesedu.com/images/qiniu/Xnip2018-07-190_14-32-56.png)

 - 1、 顺序写的方式存储数据 
 
      

   ![1](https://www.icheesedu.com/images/qiniu/Xnip2018-07-190_14-40-19.png)
   
   ![2](https://www.icheesedu.com/images/qiniu/Xnip2018-07-190_14-44-02.png)
   
   避免了内核态与用户态的上下文切换动作
   
##7、日志策略

##8、消息发送可靠性

 - 生产者发送消息到broker，有三种确认方式（request.required.acks）
    
    
   
   
      ```
      sh kafka-topics.sh --create --zookeeper 192.168.11.140:2181 --replication-factor 2 --partitions 3 --topic sixsix
      ```
      
##10、副本机制

![](https://www.icheesedu.com/images/qiniu/Xnip2018-07-190_14-59-50.png)

- **ISR（副本同步队列）**



- **HW&LEO**

     ![](https://www.icheesedu.com/images/qiniu/Xnip2018-07-190_15-03-14.png)




##11、查看kafka数据文件内容

 ```
 ```
 
    
