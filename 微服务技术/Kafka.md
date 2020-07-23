# Kafka简介
Kafka的基础概念、术语

## 体系结构
![体系结构](https://github.com/kongdou/tech-docs/blob/master/images/kafka-1.jpg)

- Producer：生产者
- Consumer： 消费者
- Broker：服务代理节点（kafka实例）

## 消息存储
### 主题（Topic）
kafka消息以topic为单位进行归类，逻辑概念

### 分区（Partition）
分区的目的是**解决高吞吐量的问题**,分区在存储层面可看做是一个可追加的日志文件，kafka通过offset保证消息在分区内的顺序性，但只保证分区有序而不保证主题有序。

#### 分区策略
每条消息发送到broker前，会根据分区策略分配到具体的哪个分区，策略：
- 轮询策略
- 随机策略
- 消息键保序策略（增加消息key）
- 自定义（配置生产端partitioner.class参数，实现org.apache.kafka.clients.producer.Partitioner接口）

默认情况下，如果指定了key，使用消息键保序策略，没有指定key，使用轮询

#### 分区设置
启动设置：
设置参数partitions

动态设置：
./kafka-topics.sh --alter --zookeeper ip:port --topic xxx --partitions x

## 副本机制
解决高可用问题

特征：

- 一个分区会在多个副本中保存相同的消息
- 副本之间是一主多从关系
- leader副本负责读写操作，follower副本只负责同步消息（主动拉取）
- leader副本故障时，从follower副本重新选举新leader

### 副本同步
AR：分区中所有副本的统称

ISR：In-Sync Replicas，所有与leader副本保持一定程度同步的副本，ISR中的副本都要同步leader中的数据，只有都同步完成了数据才认为是成功提交了，成功提交之后才能供外界访问。

OSR：Out-of-Sync Replicas同步之后过多的副本组成，OSR内的副本是否同步了leader的数据，不影响数据的提交，OSR内的follower尽力的去同步leader，可能数据版本会落后。

AR=ISR+OSR

**简单流程**

最开始所有的副本都在ISR中，在kafka工作的过程中，如果某个副本同步速度慢于replica.lag.time.max.ms指定的阈值，则被踢出ISR存入OSR，如果后续速度恢复可以回到ISR中。

## 偏移量
LEO（Log End Offset）：标识当前分区下一条代写入消息的offset

HW（High Watermark）：高水位，标识了一个特定的offset，消费者只能拉渠道这个offset之前的消息（不含HW）

所有副本都同步了的消息才能被消费，HW的位置取决于所有follower中同步最慢的分区的offset


## 消费者组（Consumer Group)
Consumer group是kafka提供的可扩展且具有容错性的消费者机制。组内多个消费者（实例）共享一个公共的ID，即group ID(consumer.properties配置)，组内的所有消费者协调在一起来消费订阅主题(subscribed topics)的所有分区(partition)。每个分区只能由同一个消费组内的一个consumer来消费。

kafka cluster中有两台broker服务器，每一台都有两个分区，这四个分区都是同一个topic下的。下左的消费者组A，组内有两个消费者，每个消费者负责两个分区的消费，而右边的消费者组B有四个消费者，每个负责消费一个分区。

## 重平衡（Rebalance)
重平衡其实就是一个协议，它规定了如何让消费者组下的所有消费者来分配topic中的每一个分区。比如一个topic有100个分区，一个消费者组内有20个消费者，在协调者的控制下让组内每一个消费者分配到5个分区，这个分配的过程就是重平衡。

重平衡的触发条件主要有三个：

1.消费者组内成员发生变更，这个变更包括了增加和减少消费者。注意这里的减少有很大的可能是被动的，就是某个消费者崩溃退出了

2.主题的分区数发生变更，kafka目前只支持增加分区，当增加的时候就会触发重平衡

3.订阅的主题发生变化，当消费者组使用正则表达式订阅主题，而恰好又新建了对应的主题，就会触发重平衡

重平衡问题：消费者无法从kafka消费消息，这对kafka的TPS影响极大，而如果kafka集内节点较多，比如数百个，那重平衡可能会耗时极多。数分钟到数小时都有可能，而这段时间kafka基本处于不可用状态。所以在实际环境中，应该尽量避免重平衡发生。

### 重平衡策略
#### 1.Range
这种分配是基于每个主题的分区分配

#### 2.RoundRobin
RoundRobin是基于全部主题的分区来进行分配的，同时这种分配也是kafka默认的rebalance分区策略。

先将所有的partition和consumer按照字典序进行排序，然后依次以按顺序轮询的方式将这六个分区分配给两个consumer，如果当前consumer没有订阅当前分区所在的topic，则轮询的判断下一个consumer

#### 3.Sticky
Sticky分配策略是最新的也是最复杂的策略，其具体实现位于package org.apache.kafka.clients.consumer.StickyAssignor。
这种分配策略是在0.11.0才被提出来的，主要是为了一定程度解决上面提到的重平衡非要重新分配全部分区的问题。称为粘性分配策略。

这种策略会保证再分配时已经分配过的分区尽量保证其能够继续由当前正在消费的consumer继续消费。

##### 示例
消费者3个：C0、C1、C2

主题4个：t0、t1、t2、t3

每个主题2个分区：t0p0、t0p1、t1p0、t1p1、t2p0、t2p1、t3p0、t3p1

三个消费者订阅所有主题

**1.初始化数据**
![初始化](https://github.com/kongdou/tech-docs/blob/master/images/Kafka-sticky-1.png)

第一步：所有分区排序（按照分区分配的consumer数量由低到高），如果consumer相同，按照分区数据字典排序（可以理解为名称）,由于每个分区都由三个consumer消费，因此排序结果：

t0p0、t0p1、t1p0、t1p1、t2p0、t2p1、t3p0、t3p1

第二步：将所有的consumer排序（按照consumer已经分配到分区数量排序）由于初始分配，没有分区，排序结果：

C0、C1、C2

第三步：依次将分区分配给各个consumer（每次分配后consumer的数量是变化的）

第一次分配:  
将t0p0分配给C0，由于C0订阅了t0，分配成功，结果：  
C0：t0p0  
C1：  
C2：  
consumer重新排序（根据已分配到的分区数量），结果：  
C1：  
C2:  
C0：t0p0  

第二次分配：  
将t0p1分配给C1，由于C1订阅了t0，分配成功，结果：  
C1：t0p1  
C2:  
C0: t0p0  
consumer重新排序（根据已分配到的分区数量），结果： 
C2:   
C0:t0p0  
C1:t0p1  

依此分配，最终结果：  
C0：t0p0，t1p1，t3p0  
C1：t0p1，t2p0，t3p1  
C2：t1p0，t2p1  

![初始化结果](https://github.com/kongdou/tech-docs/blob/master/images/Kafka-sticky-2.png)

**2.假设C1宕机**

![宕机](https://github.com/kongdou/tech-docs/blob/master/images/Kafka-sticky-3.png)


C1消费宕机：需要将分配给C1的分区，分配给C0、C2  

如果按照默认的再平衡（RoundRobin），分配结果：  
C0：t0p0，t1p0，t2p0，t3p0  
C2：t0p1，t1p1，t2p1，t3p1  

发现：调整的分区有5个，原有未宕机的consumer消费的分区也被挪动，如C2的t1p0 

按照Sticky策略分配如下：  
第一步：将分区C1消费的分区排序（根据消费者的数量由低到高排序，如果数量相同，使用数据字典排序）  
t0p1  
t2p0  
t3p1  

第二步：consumer重新排序（根据已分配到的分区数量），结果： 
C2：t1p0、t2p1  
C0：t0p0、t1p1、t3p0  

第三步：依次将分区分配到consumer:    

(1) t0p1是否被C2订阅，C2订阅了t0p1，分配成功，此时分配结果：  
C2：t1p0、t2p1、t0p1  
C0：t0p0、t1p1、t3p0  
consumer重新排序（根据已分配到的分区数量,数量相同，根据数据字典），结果： 
C0：t0p0、t1p1、t3p0  
C2：t1p0、t2p1、t0p1  

(2) t2p0是否被C0订阅，C0订阅了t2p0，分配成功，此时分配结果：
C0：t0p0、t1p1、t3p0、t2p0
C2：t1p0、t2p1、t0p1

consumer重新排序（根据已分配到的分区数量,数量相同，根据数据字典），结果:  
C2：t1p0、t2p1、t0p1  
C0：t0p0、t1p1、t3p0、t2p0  

(3) t3p1是否被C2订阅，C2订阅了t3p1，分配成功，此时分配结果:  
C2：t1p0、t2p1、**t0p1**、**t3p1**  
C0：t0p0、t1p1、t3p0、**t2p0**  

按照Sticky策略，原来消费者订阅的分区没有挪动，只挪动了C1消费的3个分区  
