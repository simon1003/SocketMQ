# RocketMQ 学习


官网：
http://rocketmq.apache.org/

---
# 一、简介
###### 1、纯java、分布式、队列模型的开源消息中间件，前身是Metaq，当 Metaq 3.0发布时，产品名称改为 RocketMQ。
###### 2、阿里开源的消息中间件，具有低延迟、高吞吐量、高可用性和适合大规模分布式系统应用的特点。RocketMQ思路起源于Kafka，它对消息的可靠传输及事务性做了优化。
###### 3、底层采用Netty NIO框架实现数据通信。
###### 4、3.X版本弃用Zookeeper,内部使用更轻量级的NameServer进行网络路由，提供了服务性能，并支持消息失败重试机制。
###### 5、支持集群模式、消费者负载均衡、水平扩展能力，支持广播模式。
###### 6、采用零拷贝原理，顺序写盘、支持亿级消息堆积能力。
###### 提供丰富的消息机制，比如顺序消息、事务消息。

---
# 核心概念
## 专业术语
###### Producer：消息生产者，负责产生消息，一般由业务系统负责产生消息
###### Consumer：消息消费者，负责消费消息，一般是后台系统负责异步消费
###### Push Consumer：Consumer的一种，应用通常向Consumer对象注册一个Listener接口，一旦收到消息，Consumer对象立刻回调Listener接口方法
###### Pull Consumer：Consumer的一种，应用通常主动调用Consumer的拉消息方法从Broker拉消息，主动权由应用控制
###### Producer Group：一类Producer的集合名称，这类Producer通常发送一类消息，且发送逻辑一致
###### Consumer Group：一类Consumer的集合名称，这类Consumer通常消费一类消息，且消费逻辑一致
###### Broker：消息中转角色，负责存储消息，转发消息，一般也称为Server。
###### NameServer的主要功能是为整个MQ集群提供服务协调与治理，具体就是记录维护Topic、Broker的信息，及监控Broker的运行状态

---
###### Topic：标识一类消息的逻辑名字，消息的逻辑管理单位。无论消息生产还是消费，都需要指定Topic
###### Tag：RocketMQ支持给在发送的时候给topic打tag，同一个topic的消息虽然逻辑管理是一样的。但是消费topic1的时候，如果你订阅的时候指定的是tagA，那么tagB的消息将不会投递。
###### Offset：message queue是无限长的数组。一条消息进来下标就会涨1,而这个数组的下标就是offset。
###### Producer Group:一个Producer Group下包含多个Producer实例，可以是多台机器，也可以是一台机器的多个进程，或者一个进程的多个Producer对象。一个Producer Group可以发送多个Topic消息。
###### Consumer Group：一个Consumer Group下包含多个Consumer实例，可以是多台机器，也可以是多个进程，或者是一个进程的多个Consumer对象。一个Consumer Group下的多个Consumer以均摊方式消费消息，如果设置为广播方式，那么这个Consumer Group下的每个实例都消费全量数据。
###### 顺序消费：指消息的消费顺序和产生顺序相同，在有些业务逻辑 下 ，必须保证顺序。比如订单的生成、付款、发货，这3个消息必须按顺序处理才行。

---
## RocketMQ中的四个角色：
### Producer（消息生产者）
### Consumer（消息消费者）
### Broker（消息暂存者）
### NameServer（消息协调者）

---
# 物理部署结构
<img src="F:\公司文件\技术分享\第五次分享\SocketMQ\static\架构.jpg" width="100%" height="100%">

---
##### Name Server是一个几乎无状态节点，可集群部署，节点之间无任何信息同步。
##### Broker部署相对复杂，Broker分为Master与Slave，一个Master可以对应多个Slave，但是一个Slave只能对应一个Master，Master与Slave的对应关系通过指定相同的BrokerName，不同的BrokerId来定义，BrokerId为0表示Master，非0表示Slave。Master也可以部署多个。每个Broker与Name Server集群中的所有节点建立长连接，定时注册Topic信息到所有Name Server。
##### Producer与Name Server集群中的其中一个节点（随机选择）建立长连接，定期从Name Server取Topic路由信息，并向提供Topic服务的Master建立长连接，且定时向Master发送心跳。Producer完全无状态，可集群部署。
##### Consumer与Name Server集群中的其中一个节点（随机选择）建立长连接，定期从Name Server取Topic路由信息，并向提供Topic服务的Master、Slave建立长连接，且定时向Master、Slave发送心跳。Consumer既可以从Master订阅消息，也可以从Slave订阅消息，订阅规则由Broker配置决定。

---
# 逻辑部署结构
<img src="F:\公司文件\技术分享\第五次分享\SocketMQ\static\逻辑部署.png" width="100%" height="100%">

---
### RocketMQ的逻辑部署结构有Producer和Consumer两个特点。
##### Producer Group：
##### 用来表示一个发送消息应用，一个Producer Group下包含多个Producer实例，可以是多台机器，也可以是一台机器的多个进程，或者一个进程的多个Producer对象。一个Producer Group可以发送多个Topic消息，Producer Group作用如下：
###### 1、标识一类Producer
###### 2、可以通过运维工具查询这个发送消息应用下有多个Producer实例
###### 3、发送分布式事务消息时，如果Producer中途意外宕机，Broker会主动回调Producer Group内的任意一台机器来确认事务状态。
##### Consumer Group
###### 用来表示一个消费消息应用，一个Consumer Group下包含多个Consumer实例，可以是多台机器，也可以是多个进程，或者是一个进程的多个Consumer对象。一个Consumer Group下的多个Consumer以均摊方式消费消息，如果设置为广播方式，那么这个Consumer Group下的每个实例都消费全量数据。

---
# 二、具有以下特点：

### 1、能够保证严格的消息顺序
### 2、提供丰富的消息拉取模式
### 3、高效的订阅者水平扩展能力
### 4、实时的消息订阅机制
### 5、亿级消息堆积能力
#####  注意：启动 RocketMQ 时，先启动 NameServer，然后再启动 Broker，后续需要发送消息就用 Producer，需要接收消息就用 Consume

---
## 获取消息的方式：
<img src="F:\公司文件\技术分享\第五次分享\SocketMQ\static\pull-push.png" width="100%" height="100%">
push-优点：及时性、服务端统一处理实现方便

push-缺点：容易造成堆积、负载性能不可控

pull-优点：获得消息状态方便、负载均衡性能可控
pull-缺点：及时性差

---
# 三、今天主要讲解的内容
* 顺序消息
* 重复问题
* 事务消息
* 消息存储

---
## 顺序消息 (如何才能在MQ集群保证消息的顺序？)
#### 理论情况下：
<img src="F:\公司文件\技术分享\第五次分享\SocketMQ\static\顺序消息.png" width="100%" height="100%">

---

### 实际情况有可能是：
###### （只要将消息从一台服务器发往另一台服务器，就会存在网络延迟问题。如果发送M1耗时大于发送M2的耗时，那么M2就仍将被先消费，仍然不能保证消息的顺序。即使M1和M2同时到达消费端，由于不清楚消费端1和消费端2的负载情况，仍然有可能出现M2先于M1被消费的情况。）
<img src="F:\公司文件\技术分享\第五次分享\SocketMQ\static\顺序消息-网络延时.png" width="100%" height="100%">

---

### 怎么解决呢？
解决1、将M1和M2发往同一个消费者，且发送M1后，需要消费端响应成功后才能发送M2；
问题1：如果M1被发送到消费端后，消费端1没有响应，那是继续发送M2呢，还是重新发送M1？一般为了保证消息一定被消费，肯定会选择重发M1到另外一个消费端2

<img src="F:\公司文件\技术分享\第五次分享\SocketMQ\static\顺序消息-响应.png" width="100%" height="100%">

---

问题2：
消费端1没有响应Server时有两种情况，一种是M1确实没有到达(数据在网络传送中丢失)，另外一种消费端已经消费M1且已经发送响应消息，只是MQ Server端没有收到
#### （后面讲解这两个问题）
<img src="F:\公司文件\技术分享\第五次分享\SocketMQ\static\顺序消息-响应.png" width="100%" height="100%">

---

## 最简单可行的方法就是：

##### 保证生产者 - MQServer - 消费者是一对一对一的关系

## 存在的问题：
#### 1、并行度就会成为消息系统的瓶颈（吞吐量不够）
#### 2、更多的异常处理，比如：只要消费端出现问题，就会导致整个处理流程阻塞，我们不得不花费更多的精力来解决阻塞的问题。

## 最终的目标：集群的高容错性和高吞吐量

## 这对矛盾，阿里是如何解决的？
##### 世界上解决一个计算机问题最简单的方法：“恰好”不需要解决它！

---

### 按顺序发送(消息顺序问题)：
#### 从源码角度分析
###### RocketMQ通过轮询所有队列的方式来确定消息被发送到哪一个队列（负载均衡策略）。比如下面的示例中，订单号相同的消息会被先后发送到同一个队列中
###### // RocketMQ通过MessageQueueSelector中实现的算法来确定消息发送到哪一个队列上
###### // RocketMQ默认提供了两种MessageQueueSelector实现：随机/Hash
######  // 当然也可以根据业务实现自己的MessageQueueSelector来决定消息按照何种策略发送到消息队列中
`endResult sendResult = producer.send(msg, new MessageQueueSelector() {
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        Integer id = (Integer) arg;
        int index = id % mqs.size();
        return mqs.get(index);
    }
}, orderId);
`

---

### 在获取到路由信息以后，会根据MessageQueueSelector实现的算法来选择一个队列，同一个OrderId获取到的肯定是同一个队列。
`private SendResult send()  {
    // 获取topic路由信息
    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
    if (topicPublishInfo != null && topicPublishInfo.ok()) {
        MessageQueue mq = null;
        // 根据我们的算法，选择一个发送队列
        // 这里的arg = orderId
        mq = selector.select(topicPublishInfo.getMessageQueueList(), msg, arg);
        if (mq != null) {
            return this.sendKernelImpl(msg, mq, communicationMode, sendCallback, timeout);
        }
    }
}`

---

### 消息重复问题
##### 造成消息重复的根本原因是：网络不可达 ---> 如果消费端收到两条一样的消息，应该怎样处理？
1、消费端处理消息的业务逻辑保持幂等性（不管来多少条重复消息，最后处理的结果都一样）
2、保证每条消息都有唯一编号且保证消息处理成功与去重表的日志同时出现（利用一张日志表来记录已经处理成功的消息的ID，如果新到的消息ID已经在日志表中，那么就不再处理这条消息）

### 总结：RocketMQ不保证消息不重复，如果业务需要保证严格的不重复消息，需要在业务端去重

---

## 事务消息
### RocketMQ除了支持普通消息，顺序消息，另外还支持事务消息

### 先看示例
### 在单机环境下：
### Bob向Smith转账100块 场景为例：
<img src="F:\公司文件\技术分享\第五次分享\SocketMQ\static\单机转账.png" width="100%" height="100%">

---

### 当用户增长到一定程度，Bob和Smith的账户及余额信息已经不在同一台服务器上了，那么上面的流程就变成了这样：
<img src="F:\公司文件\技术分享\第五次分享\SocketMQ\static\集群转账.png" width="100%" height="100%">

### 同样是一个转账的业务，在集群环境下，耗时居然成倍的增长。那如何来规避这个问题？

---

## 大事务 = 小事务 + 异步
<img src="F:\公司文件\技术分享\第五次分享\SocketMQ\static\小事务异步.png" width="100%" height="100%">

### 图中执行本地事务（Bob账户扣款）和发送异步消息应该保证同时成功或者同时失败，也就是扣款成功了，发送消息一定要成功，如果扣款失败了，就不能再发送消息。那问题是：我们是先扣款还是先发送消息呢？

---
### 先发送消息的情况：
<img src="F:\公司文件\技术分享\第五次分享\SocketMQ\static\事务先发消息.png" width="100%" height="100%">

### 存在的问题是：如果消息发送成功，但是扣款失败，消费端就会消费此消息，进而向Smith账户加钱。

---
### 先扣款的情况：
<img src="F:\公司文件\技术分享\第五次分享\SocketMQ\static\事务先扣款.png" width="100%" height="100%">

### 存在的问题跟上面类似：如果扣款成功，发送消息失败，就会出现Bob扣钱了，但是Smith账户未加钱。

---

### 解决问题的方法：
##### 比如，直接将发消息放到Bob扣款的事务中去，如果发送失败，抛出异常，事务回滚；
#### JAVA Spring 也有可以实现事务回滚。

---

### RocketMQ的解决方法：
<img src="F:\公司文件\技术分享\第五次分享\SocketMQ\static\实现事务消息.png" width="100%">
RocketMQ第一阶段发送Prepared消息时，会拿到消息的地址，第二阶段执行本地事物，第三阶段通过第一阶段拿到的地址去访问消息，并修改消息的状态。

##### RocketMQ会根据发送端设置的策略会定期扫描消息集群中的事物消息 来决定是回滚还是继续发送确认消息

---

`// 发送事务消息的一系列准备工作
// 未决事务，MQ服务器回查客户端
// 也就是上文所说的，当RocketMQ发现`Prepared消息`时，会根据这个Listener实现的策略来决断事务
TransactionCheckListener transactionCheckListener = new TransactionCheckListenerImpl();
// 构造事务消息的生产者
TransactionMQProducer producer = new TransactionMQProducer("groupName");
// 设置事务决断处理类
producer.setTransactionCheckListener(transactionCheckListener);
// 本地事务的处理逻辑，相当于示例中检查Bob账户并扣钱的逻辑
TransactionExecuterImpl tranExecuter = new TransactionExecuterImpl();
producer.start()
// 构造MSG，省略构造参数
Message msg = new Message(......);
// 发送消息
SendResult sendResult = producer.sendMessageInTransaction(msg, tranExecuter, null);
producer.shutdown();`

---

`// 事务消息的发送过程 

public TransactionSendResult sendMessageInTransaction(.....)  {
    // 逻辑代码，非实际代码
    // 1.发送消息
    sendResult = this.send(msg);
    // sendResult.getSendStatus() == SEND_OK
    // 2.如果消息发送成功，处理与消息关联的本地事务单元
    LocalTransactionState localTransactionState = tranExecuter.executeLocalTransactionBranch(msg, arg);
    // 3.结束事务
    this.endTransaction(sendResult, localTransactionState, localException);
}`

---
##### 如果Bob的账户的余额已经减少，且消息已经发送成功，Smith端开始消费这条消息，这个时候就会出现消费失败和消费超时两个问题，解决超时问题的思路就是一直重试，直到消费端消费消息成功，整个过程中有可能会出现消息重复的问题，按照前面的思路解决即可。

<img src="F:\公司文件\技术分享\第五次分享\SocketMQ\static\消费事务2.png" width="100%">
如果消费失败怎么办？

---
## 消息存储
* consume queue和commit log组成

##### 整体的架构图
<img src="F:\公司文件\技术分享\第五次分享\SocketMQ\static\消息存储.png" width="100%">

---
### ConsumeQueue 消息的逻辑队列，类似于索引文件，存储的是指向物理存储的地址
##### 文件地址在：${$storeRoot}\consumequeue\${topicName}\${queueId}\${fileName}

###### 例如：broker_a，有4个写队列queue0-3，4个读队列queue0-3，这是有两个producer，producer A发送topic_A 消息，producer B 发送topic_B消息；producer A和producer B属于不同的consumerGroup
###### topic_A对应queue0-3，consumequeue为${$storeRoot}\consumequeue\topic_A\queue0-4\${fileName}
###### topic_B对应queue0-3，consumequeue为${$storeRoot}\consumequeue\topic_B\queue0-4\${fileName}

---
### CommitLlog 物理存储文件
###### 每台Broker上的CommitLog被本机器所有ConsumeQueue共享，
###### 文件地址：${user.home}\store\${commitlog}\${fileName}

##### 存储机制这样设计有以下几个好处：
* 1 CommitLog顺序写，可以大大提高写入效率。
* 2虽然是随机读，但是利用操作系统的pagecache机制，可以批量地从磁盘读取，作为cache存到内存中，加速后续的读取速度。
* 3 为了保证完全的顺序写，需要ConsumeQueue这个中间结构，因为ConsumeQueue里只存偏移量信息，所以尺寸是有限的，在实际情况中，大部分的ConsumeQueue能够被全部读入内存，所以这个中间结构的操作速度很快，可以认为是内存读取的速度。此外为了保证CommitLog和ConsumeQueue的一致性，CommitLog里存储了Consume Queues、Message Key、Tag等所有信息，即使ConsumeQueue丢失，也可以通过commitLog完全恢复出来。

---
###### 下图是一个Broker在文件系统中存储的各个文件，有commitlog文件夹、consumequeue文件夹、还有config文件夹中Topic、Consumer的相关信息。最下面index文件夹存的是索引文件，它用来加快消息查询的速度
<img src="F:\公司文件\技术分享\第五次分享\SocketMQ\static\ConsumeQueue.png" width="100%">

---
# 优点和缺点
###### 单机吞吐量：十万级
###### topic数量都吞吐量的影响：topic可以达到几百，几千个的级别，吞吐量会有较小幅度的下降。可支持大量topic是一大优势。
###### 时效性：ms级
###### 可用性：非常高，分布式架构
###### 消息可靠性：经过参数优化配置，消息可以做到0丢失
###### 功能支持：MQ功能较为完善，还是分布式的，扩展性好
##### 总结：
###### 接口简单易用，可以做到大规模吞吐，性能也非常好，分布式扩展也很方便，社区维护还可以，可靠性和可用性都是ok的，还可以支撑大规模的topic数量，支持复杂MQ业务场景  
###### 而且一个很大的优势在于，源码是java，我们可以自己阅读源码，定制自己公司的MQ，可以掌控  
---

# 源码地址
https://github.com/apache/rocketmq

---
# rocketmq可视化管理控制台代建
开源的rocketmq-externals项目：
https://github.com/apache/rocketmq-externals

---
# rocketmq项目搭建
参考官网：
http://rocketmq.apache.org/docs/quick-start/

