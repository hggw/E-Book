<center><font size="10">RocketMQ简明文档</font></center>

# 一、MQ介绍

MQ：`MessageQueue`，消息队列。 队列，是一种FIFO 先进先出的数据结构。消息由生产者发送到MQ进行排队，然后按原来的顺序交由消息的消费者进行处理。  

MQ的作用主要有以下三个方面：

异步处理：提高系统的响应速度、吞吐量。  

解耦：服务之间进行解耦，提高系统整体的稳定性以及可扩展性;实现数据分发,生产者发送一个消息后，可以由一个或者多个消费者进行消费，并且消费者的增加或者减少对生产者没有影响。  

削峰：以稳定的系统资源应对突发的流量冲击  

但也引入了一些新的问题：  

(1)系统可用性降低：系统引入的外部依赖增多，系统的稳定性就会变差。一旦MQ宕机，对业务会产生影响。(MQ的高可用性)  

(2)系统复杂度提高：引入MQ后，会变为异步调用，数据的链路就会变得更复杂。并且还会带来其他一些问题。比如：如何保证消息不会丢失？不会被重复消费？怎么保证消息的顺序性等问题。(MQ的消息幂等性)  

(3)消息一致性问题：例如，A系统处理完业务，通过MQ发送消息给B、C系统进行后续的业务处理。如果B系统处理成功，但C系统处理失败怎么办？这就需要考虑如何保证消息数据处理的一致性。(MQ的事务性)

# 二、RocketMQ入门

## 2.1 基本概念

**1）消息(Message)**

传输消息的物理载体，是生产和消费的最小单位，每一条消息必须属于一个主题。

**2）消息标识(MessageId/Key)**

RocketMQ中的每一个消息拥有唯一的一个MessageId，且可以携带具有业务标识的Key，以方便对消息的查询。生产者发送消息时，会自动生成一个MessageId（msgId），到达Broker后，也会自动生成一个MessageId（offsetMsgId）。这三者都被成为消息标识。

**3）主题(Topic)**

Topic代表一类消息的集合，每个Topic内含有多条消息，是RocketMQ进行消息订阅的基本单元。

**4）队列(Queue)**

Topic的组成部分，消息生产和消息消费操作都是基于Queue进行的。

**5）消费者/消费者组**（Consumer Group）（比较容易理解）

对于消费者组，需要注意：

> 一个Queue只能被**一个消费者组内的一个消费者**消费。也就是说，一个Queue内的消息不允许**同时**被**同一个消费者组**下的多个consumer消费。
>
> 当consumer发生故障时，触发rebalance机制，同一个消费者组剩余消费者的其中一个会对这个queue进行消费，因此，存在**同一个消费者组中不同消费者消费同一Queue**的情况。
>
> 不同消费者组的不同consumer可以同时消费同一个Queue。
>

**6）生产者/生产者组**（Producer Group）（比较容易理解）

组的概念主要是基于RocketMQ是用于分布式微服务系统的上下游场景，为保证系统的高可用性，消息队列的各个组件都是以集群的形式存在。

**7）标签（Tag）**

用于进一步区分某个Topic下的消息，RocketMQ支持消费者基于Tag对消息进行过滤。

Topic和Tag都是业务场景下用于分类的标识，区别在于Topic的粒度更大，一般都是基于系统进行区分，Tag是基于同一个业务系统下不同业务种类进行区分。

**8）分片(Sharding)**

分片概念是官方文档中没有出现的概念，这里主要是用来说明Topic、Queue和Broker之间的关系。

通常一个Topic下会配置多个Queue，这些Queue会平均分布在多个Broker下，主要是为了实现高可用性。

## 2.2 系统架构

RocketMQ的系统架构主要有**路由注册中心（NameServer）**、**消息存储服务器（Broker）**、**消息客户端（Client）**组成。其中**Client由生产者（Producer）和消费者（Consumer）**组成。

![](images/PPT02.png)

**1）路由注册中心—Nameserver**

Nameserver主要是作为Broker和Topic的路由注册中心，支持Broker的动态注册和发现。具体功能体现在以下两点：

> **Broker管理：**
>
> 接受Broker集群的注册信息并保存相关的路由数据，同时利用Broker的心跳机制检查Broker是否存活。
>
> **路由信息管理：**
>
> 集群中每一台NameServer服务器都保存Broker集群的全量路由信息和用于Client查询的Queue信息。

NameServer集群中的**各个节点是无差异的**，且各节点之间**互不通信**。

互不通信的前提下保证各个NameServer节点无差异，是因为消息存储服务器Broker在启动之后，会与NameServer集群中的**每一个节点**建立**长连接**，发起注册请求。由于Broker的**主动发送心跳机制**，NameServer节点内部会维护一个Broker信息列表，存储BrokerId、Broker地址、Broker名称、Broker所属集群、时间戳等信息。

某些异常场景下，NameServer未及时收到某个Broker的心跳信息，NameServer会将该Broker剔除出可用Broker列表。

NameServer服务器内设由一个**定时任务**，默认每间隔10s扫描一次Broker列表，**对比每个Broker当前最新的时间戳与当前时间是否相差大于120s**，若超过则判定该Broker失效。

**2）消息存储服务器—Broker**

消息存储服务器Broker主要是负责客户端管理(Client Manager)、消息存储服务(**Store Service**)、高可用实现(**HA Service**)、索引功能(**Index Service**)等。

Client Manager主要是实现接收和解析Client的请求以及管理Client，例如维护Consumer的订阅信息等。Store Service用于将消息数据持久化到物理磁盘。HA Service主要是实现Master Broker和Slave Broker之间数据的同步，保证Master发生异常时，消息流转功能的高可用性。Index Service是对Broker内的消息数据实现了索引功能，可以根据特定的Message Key进行快速定位查询。

**Broker的主动发送心跳机制**：Broker节点默认**每间隔30s**向NameServer发送一次心跳数据，NameServer会更新其Broker列表中对应的路由信息以及时间戳。

**3）消息客户端—Client**

Client也就是消息客户端，包括Producer(生产者)和Consumer(消费者)。Producer和Consumer在某一时刻下只会连接一个Nameserver节点(不一定是同一个)，只有在连接出现异常时才会向尝试连接另外一个。

当Topic路由信息发生变化时，NameServer不会主动推送给Client，导致客户端无法被动感知Topic信息的动态变化，因此客户端会每30s去NameServer拉取Topic最新的路由信息。

注意：Nameserver是在内存中存储Topic的路由信息，持久化Topic路由信息是在Broker中完成的。

## 2.3 工作流程

图有误！！！！！！！！

![](images/MQ18.png)

1）启动NameServer集群，启动后开始监听端口，等待与Broker、Producer、Consumer建立长连接；

2）启动Broker集群，每个Broker主动发起请求，与NameServer集群内所有节点建立长连接，每隔30s发送一次心跳包数据；

3）Producer启动时，先跟NameServer集群中的某一个节点建立长连接，并从中获取当前订阅Topic的Queue与Broker的地址(IP+Port)之间的映射关系。若建立连接失败，则会取轮询下一个NameServer节点，直至成功建立长连接。然后与对应的Broker建立长连接，推送消息数据。

Producer获取到相关的路由信息后，会将其缓存到本地，随后会每隔30s从NameServer拉取并更新一次。

4）Consumer同样会跟某一个NameServer节点建立长连接，获取到当前订阅Topic的Queue与Broker的地址之间的映射关系。同样的，若建立失败，则会轮询下一个NameServer节点至成功。随后从对应的Broker中拉取消息进行处理。

Consumer获取相关的路由信息后，同样会缓存到本地，会每隔30s从NameServer更新一次。但不同的是，Consumer会向Broker发送心跳，确保Broker的存活状态。

# 三、Broker集群模型

## 3.1 数据复制和刷盘策略

![](images/MQ20.png)

### 3.1.1 复制策略

指的是Master与Slave之间的数据同步方式，分为**同步复制和异步复制**：

- **同步复制**：消息写入Master后，需等待Slave同步数据成功之后，Master才会向Producer返回成功ACK。
- **异步复制**：消息写入Master后，Master立即向Producer返回成功ACK，无需等待Slave同步数据成功之后，

### 3.1.2 刷盘策略

指的是Broker中消息的落盘方式，即消息发送到broker内存后，持久化到磁盘的方式，分为**同步刷盘和异步刷盘**：

- **同步刷盘**：当消息持久化到Broker的磁盘后，才算是消息写入成功
- **异步刷盘**：当消息写入到Broker的内存后，即表示消息写入成功，无需等待消息持久化到磁盘

## 3.2 Broker集群模型

![](images/MQ19.png)

### 3.2.1单机模式

这种模式只能在测试时使用，本质上单个Broker不能成为集群，一旦Broker重启或者宕机时，会导致整个消息服务不可用，不建议在生产环境中使用。

### 3.2.2 多Master模式

集群由多个Master构成，没有Slave。同一个Topic的各个Queue会平均分布在各个master节点上。  

优点：配置简单，单个Master宕机或者重启对应用无影响，在磁盘配置为RAID10时，即使机器宕机不可恢复情况下，由于RAID10磁盘可靠性非常高，消息也不会丢失(异步刷盘丢失少量消息，同步刷盘完全不丢失)，性能最高。  

缺点：单台机器宕机时，这台机器上未被消费的消息在机器恢复之前不可订阅，消息的实时性会受到影响。  

**注意**：使用同步刷盘可以保证消息不丢失，同时 Topic 相对应的 queue 应该分布在集群中各个节点，而不是只在某各节点上，否则，该节点宕机会对订阅该 topic 的应用造成影响。

### 3.2.3 多Master多Slave模式—异步复制(NM-NS,ASYNC)

Broker集群由多个master构成，每个Master配置了多个Slave（配置了RAID磁盘阵列时，一般配置一对一）。Master与Slave的关系是主备关系，即Master负责处理消息和读写请求，而Slave仅负责消息的备份与Master宕机后的角色切换。

异步复制指的是消息成功写入Master后，Master立即向Producer返回ACK，无需等待Slave同步数据成功。

这个模式最大的特点：当Master宕机后Slave能够自动切换为Master。但由于Slave从Master的同步具有短暂的延迟（毫秒级），所以当Master宕机后，这种异步复制方式可能会存在少量消息的丢失问题。 

### 3.2.4 多Master多Slave模式，同步双写(NM-NS,SYNC)

HA采用同步双写机制，Master与Slave都要成功写入消息后，才会返回ACK。  

优点：数据与服务都无单点故障问题，Master宕机情况下，消息无延迟，服务可用性和数据可用性都非常高。  

缺点：性能比异步复制略低，大概低10%，发送单个消息的RT会略高。**目前Master宕机后，Slave节点不能自动切换成Master节点**。  

# 四、RocketMQ消息类型

## 4.1 基本样例

基本样例部分我们使用消息生产者分别通过三种方式发送消息，同步发送、异步发送以及单向发送。

同步发送:等待消息返回后再继续进行下面的操作。  

异步发送:所有消息回调方法都执行结束,再关闭Producer  

单向发送:使用producer.sendOneWay方式来发送消息，只管把消息发出去

```java
public class OnewayProducer {
	public static void main(String[] args) throws Exception{
		//Instantiate with a producer group name.
		DefaultMQProducer producer = new DefaultMQProduce("please_rename_unique_group_name");
		// Specify name server addresses.
		producer.setNamesrvAddr("localhost:9876");
		//Launch the instance.
		producer.start();
		for (int i = 0; i < 100; i++) {
			//Create a message instance, specifying topic, tag and messagebody.
			Message msg = new Message("TopicTest","TagA",
				("Hello RocketMQ " +i).getBytes(RemotingHelper.DEFAULT_CHARSET));
			//Call send message to deliver message to one of brokers.
			producer.sendOneway(msg);
		} 
		//Wait for sending to complete
		Thread.sleep(5000);
		producer.shutdown();
	}
}
```

## 4.2 顺序消息

在默认情况下，Producer会采取Round Robin轮询方式把消息发送到不同的MessageQueue(分区队列)，而consumer也从多个MessageQueue上拉取消息，这种情况下无法保证消息是有序的。  

只有当一组有序的消息发送到同一个MessageQueue上时，利用MessageQueue先进先出的特性，从而保证这一组消息有序。  

这里需要注意的是，MessageQueue先进先出的特性只能保证在同一个MessageQueue上的消息是有序的，不同MessageQueue之间的消息仍旧是乱序的。  

要保证顺序消费，consumer需要按队列一个一个来取消息，即取完一个队列的消息后，再去取下一个队列的消息。  

MessageListenerOrderly对象，在RocketMQ内部就会通过锁队列的方式保证消息是一个一个队列来取的。

MessageListenerConcurrently不会锁队列，每次都是从多个MessageQueue中取一批数据（默认不超过32条）。因此也无法保证消息有序。

## 4.3 广播消息

在集群状态(MessageModel.CLUSTERING)下，每一条消息只会被一个consumergroup其中一个consumer消费到。而广播模式则是把消息发给了所有订阅了对应topic的消费者，而不管消费者是不是同一个消费者组。

## 4.4 延迟消息

延迟消息实现的效果就是在调用producer.send方法后，消息并不会立即发送出去，而是会等一段时间再发送出去。
开源版本的RocketMQ中，对延迟消息并不支持任意时间的延迟设定(商业版本中支持)，而是只支持18个固定的延迟级别，1到18分别对应messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h。

```java
public class ScheduledMessageProducer {
	public static void main(String[] args) throws Exception {
		// Instantiate a producer to send scheduled messages
		DefaultMQProducer producer = new DefaultMQProducer("ExampleProducerGroup");
		// Launch producer
		producer.start();
		int totalMessagesToSend = 100;
		for (int i = 0; i < totalMessagesToSend; i++) {
			Message message = new Message("TestTopic", ("Hello scheduledmessage " + i).getBytes());
			// This message will be delivered to consumer 10 seconds later.
			message.setDelayTimeLevel(3);
			// Send the message
			producer.send(message);
		} 
		// Shutdown producer after use.
		producer.shutdown();
	}
}
```

## 4.5 批量消息

批量消息是指将多条消息合并成一个批量消息，一次发送出去。这样的好处是可以减少网络IO，提升吞吐量。

根据官网的注释：如果批量消息大于1MB就不要用一个批次发送，而要拆分成多个批次消息发送。  

实际使用时，这个1MB的限制可以稍微扩大点，实际最大的限制是4194304字节，大概4MB。同时批量消息的使用是有一定限制的，这些消息应该有相同的Topic，相同的waitStoreMsgOK。而且不能是延迟消息、事务消息等。

## 4.6 过滤消息

在大多数情况下，可以使用Message的Tag属性来简单快速的过滤信息。TAG是RocketMQ中特有的一个消息属性。RocketMQ的最佳实践中就建议，使用RocketMQ时，一个应用可以就用一个Topic，而应用中的不同业务就用TAG来区分。  

这种方式有一个很大的限制，就是一个消息只能有一个TAG，在一些比较复杂的场景会存在不足。但RocketMQ支持使用SQL表达式来对消息进行过滤。  

这个模式的关键是在消费者端使用MessageSelector.bySql(String sql)返回的一个MessageSelector。这里面的sql语句是按照SQL92标准来执行的。sql中可以使用的参数有默认的
TAGS和一个在生产者中加入的a属性。

## 4.7 事务消息(***)

### 4.7.1 事务消息

关于事务消息，官网的介绍是：事务消息是在分布式系统中保证最终一致性的两阶段提交的消息实现。也就是说，事务消息可以保证本地事务执行与消息发送两个操作的原子性。  

事务消息只保证消息发送者的本地事务与发消息这两个操作的原子性，因此，事务消息的示例只涉及到消息发送者，对于consumer来说，并没有什么区别。

```java
public class TransactionProducer {
	public static void main(String[] args) throws MQClientException, InterruptedException {
		TransactionListener transactionListener = new TransactionListenerImpl();
    	TransactionMQProducer producer = new TransactionMQProducer("please_rename_unique_group_name");
		//producer.setNamesrvAddr("127.0.0.1:9876");
    	ExecutorService executorService = new ThreadPoolExecutor(2, 5, 100, TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(2000), new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread thread = new Thread(r);
            thread.setName("client-transaction-msg-check-thread");
            return thread;
        }
    });
    producer.setExecutorService(executorService);
    producer.setTransactionListener(transactionListener);
    producer.start();
    String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
    for (int i = 0; i < 10; i++) {
        try {
            Message msg =new Message("TopicTest", tags[i % tags.length], "KEY" + i,("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
            SendResult sendResult = producer.sendMessageInTransaction(msg,null);
            System.out.printf("%s%n", sendResult);
            Thread.sleep(10);
        } catch (MQClientException | UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }
    for (int i = 0; i < 100000; i++) {
        Thread.sleep(1000);
    }
    producer.shutdown();
	}
}
```

事务消息的关键是在TransactionMQProducer中指定了一个TransactionListener事务监听器，这个事务监听器就是事务消息的关键控制器。

```java
public class TransactionListenerImpl implements TransactionListener {
	//在提交完事务消息后执行。
	//返回COMMIT_MESSAGE状态的消息会立即被消费者消费到。
	//返回ROLLBACK_MESSAGE状态的消息会被丢弃。
	//返回UNKNOWN状态的消息会由Broker过一段时间再来回查事务的状态。
	@Override
	public LocalTransactionState executeLocalTransaction(Message msg,Object arg) {
		String tags = msg.getTags();
		//TagA的消息会立即被消费者消费到
		if(StringUtils.contains(tags,"TagA")){
			return LocalTransactionState.COMMIT_MESSAGE;
			//TagB的消息会被丢弃
		}else if(StringUtils.contains(tags,"TagB")){
			return LocalTransactionState.ROLLBACK_MESSAGE;
			//其他消息会等待Broker进行事务状态回查。
		}else{
			return LocalTransactionState.UNKNOW;
		}
	} 

	//在对UNKNOWN状态的消息进行状态回查时执行。返回的结果是一样的。
	@Override
	public LocalTransactionState checkLocalTransaction(MessageExt msg) {
		String tags = msg.getTags();
		//TagC的消息过一段时间会被消费者消费到
		if(StringUtils.contains(tags,"TagC")){
			return LocalTransactionState.COMMIT_MESSAGE;
			//TagD的消息也会在状态回查时被丢弃掉
		}else if(StringUtils.contains(tags,"TagD")){
			return LocalTransactionState.ROLLBACK_MESSAGE;
			//剩下TagE的消息会在多次状态回查后最终丢弃
		}else{
			return LocalTransactionState.UNKNOW;
		}
	}
}
```

### 4.7.2 事务消息的实现流程

![avatar](images\MQ11.png)  
事务消息的大致方案，其中分为两个流程：事务消息的发送及提交、事务消息的补偿。  

第一阶段：事务消息的发送及提交

1.Producer向Broker服务器发送Half消息 

2.Broker服务器返回Half消息的响应结果给Producer 

3.Producer根据Broker的响应结果执行本地事务，如果响应结果为失败，则half消息对业务不可见，本地逻辑不执行 

4.Broker根据Producer本地事务状态，进行Commit或者Rollback（Commit操作生成消息索引，消息对消费者可见）

第二阶段：事务消息的补偿

5.当Broker没有收到Producer本地事务的执行状态时，Broker主动发起请求，回查Producer的本地事务执行状态 

6.Producer收到回查消息，检查回查消息对应的本地事务的状态 

7.Producer根据本地事务状态，重新Commit或者Rollback

### 4.7.3 事务消息设计

**(1) 第一阶段事务消息对用户不可见**

在RocketMQ事务消息流程中，在第一阶段时，消息就会发送到broker端，但是在消息提交之前，该消息对消费者是不可见的。即事务消息对用户不可见。  

实现原理如下：
针对half消息，RocketMQ会备份该消息的主题与消费队列信息，然后将该消息topic改为RMQ_SYS_TRANS_HALF_TOPIC，消费队列置为0。  由于消费者并未订阅该主题，因此不会拉取到该消息。同时RocketMQ会开启一个定时任务，从Topic为RMQ_SYS_TRANS_HALF_TOPIC中拉取消息进行消费，根据生产者组获取一个服务提供者发送回查事务状态请求，根据事务状态来决定是提交或回滚消息。

**(2) Commit和Rollback操作以及Op消息的引入**

在完成第一阶段写入一条对用户不可见的消息后，第二阶段如果是Commit操作，则需要让消息对用户可见。 
由于第一阶段的消息对用户是不可见的，Rollback其实不需要真正撤销消息（实际上RocketMQ无法真正删除一条消息，因为是顺序写文件的）。但是需要一个操作来标识这条消息的最终状态，用于区别这条消息没有确定状态（Pending状态）。  

RocketMQ事务消息方案中引入了Op消息的概念，用Op消息标识事务消息已经确定的状态（Commit或者Rollback）。如果一条事务消息没有对应的Op消息，说明这个事务的状态还无法确定。引入Op消息后，事务消息无论是Commit或者Rollback都会记录一个Op操作。Commit相对于Rollback只是在写入Op消息前创建Half消息的索引。

**(3) 事务提交处理**

首先从结束事务请求命令中获取消息的物理偏移量（commitlogOffset），其实现逻辑TransactionalMessageService#.commitMessage 实现  

然后恢复消息的主题 消费队列，构建新的消息对象，由 TransactionalMessageService#endMessageTransaction 实现  

然后将消息再次存储在 commitlog 文件中，此时的消息主题则为业务方发送的消息，将被转发到对应的消息消费队列，供消息消费者消费，其实现由 TransactionalMessageService#sendFinalMessage 实现  。

消息存储后，删除 prepare 消息，其实现方法并不是真正的删除，而是将 prepare消息存储到 RMQ_SYS_TRANS_OP_HALF TOPIC 主题中，表示该事务消息（ prepare 状态的消息）已经处理过（提交或回滚），为未处理的事务进行事务回查提供查找依据  

事务的回滚与提交的唯一差别是无须将消息恢复原主题，直接删除 prepare 消息即可，同样是将预处理消息存储在RMQ_SYS_TRANS_OP_HALF _TOPIC 主题中，表示已处理过该消息

**(4) 补偿机制——事务回查**

如果在RocketMQ事务消息的二阶段过程中失败了，例如在做Commit操作时，出现网络问题导致Commit失败，那么需要通过一定的策略使这条消息最终被Commit。

RocketMQ采用了一种补偿机制，称为“回查”。Broker端对未确定状态的消息发起回查，将消息发送到对应的Producer端（同一个Group的Producer），由Producer根据消息来检查本地事务的状态，进而执行Commit或者Rollback。  

Broker端通过对比Half消息和Op消息进行事务消息的回查并且推进CheckPoint（记录那些事务消息的状态是确定的）。  

当然，rocketmq并不会无休止的的信息事务状态回查，默认回查15次，如果15次回查后还是无法得知事务状态，rocketmq默认回滚该消息。

### 4.7.4 事务消息源码分析

RocketMQ提供的事务消息发送API是TransactionMQProducer类，该类主要有下面两个关键属性  

```java
//事务状态回查异步执行线程池
private ExecutorService executorService;
//事务监听器
private TransactionListener transactionListener;
```

其中transactionListener是我们创建事务消息生产者时需要注册的，用以执行和回查本地事务。

```java
public interface TransactionListener {
	/**
 		When send transactional prepare(half) message succeed, this method will be invoked to execute local transaction.
 	*/
	LocalTransactionState executeLocalTransaction(final Message msg, final Object arg);
	/**
 		When no response to prepare(half) message. broker will send check message to check the transaction status, and this method will be invoked to get local transaction status.
 	*/
	LocalTransactionState checkLocalTransaction(final MessageExt msg);
}
```

事务消息的发送实际调用的还是DefaultMQProducerImpl的sendMessageInTransaction()方法

```java
public TransactionSendResult sendMessageInTransaction(final Message msg,final LocalTransactionExecuter localTransactionExecuter, final Object arg) throws MQClientException {
    //获取事务监听器，如果没有注册事务监听器的话则直接抛出异常
	TransactionListener transactionListener = getCheckListener();
    if (null == localTransactionExecuter && null == transactionListener) {
        throw new MQClientException("tranExecutor is null", null);
    }
	//清除延时属性
    if (msg.getDelayTimeLevel() != 0) {
        MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_DELAY_TIME_LEVEL);
    }
    Validators.checkMessage(msg, this.defaultMQProducer);
	//为消息设置属性
    SendResult sendResult = null;
	//设置TRAN_MSG 为 true
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_TRANSACTION_PREPARED, "true");
	//设置PGROUP
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_PRODUCER_GROUP, this.defaultMQProducer.getProducerGroup());
    try {
        sendResult = this.send(msg);
    } catch (Exception e) {
        throw new MQClientException("send message Exception", e);
    }

    LocalTransactionState localTransactionState = LocalTransactionState.UNKNOW;
    Throwable localException = null;
    switch (sendResult.getSendStatus()) {
		//消息发送成功则执行本地事务
        case SEND_OK: {
            try {
                if (sendResult.getTransactionId() != null) {
                    msg.putUserProperty("__transactionId__", sendResult.getTransactionId());
                }
                String transactionId = msg.getProperty(MessageConst.PROPERTY_UNIQ_CLIENT_MESSAGE_ID_KEYIDX);
                if (null != transactionId && !"".equals(transactionId)) {
                    msg.setTransactionId(transactionId);
                }
                if (null != localTransactionExecuter) {
                    localTransactionState = localTransactionExecuter.executeLocalTransactionBranch(msg, arg);
                } else if (transactionListener != null) {
                    log.debug("Used new transaction API");
					//执行本地事务
                    localTransactionState = transactionListener.executeLocalTransaction(msg, arg);
                }
                if (null == localTransactionState) {
                    localTransactionState = LocalTransactionState.UNKNOW;
                }

                if (localTransactionState != LocalTransactionState.COMMIT_MESSAGE) {
                    log.info("executeLocalTransactionBranch return {}", localTransactionState);
                    log.info(msg.toString());
                }
            } catch (Throwable e) {
                log.info("executeLocalTransactionBranch exception", e);
                log.info(msg.toString());
                localException = e;
            }
        }
        break;
		//如果消息发送失败，则设置本次事务状态为回滚
        case FLUSH_DISK_TIMEOUT:
        case FLUSH_SLAVE_TIMEOUT:
        case SLAVE_NOT_AVAILABLE:
            localTransactionState = LocalTransactionState.ROLLBACK_MESSAGE;
            break;
        default:
            break;
    }

    try {
		//结束事务，根据本地事务执行状态执行提交、回滚或暂不处理
        this.endTransaction(sendResult, localTransactionState, localException);
    } catch (Exception e) {
        log.warn("local transaction execute " + localTransactionState + ", but end broker transaction failed", e);
    }
...
}
```

TransactionListener#executeLocalTransaction()方法返回的执行状态有以下三种

```java
public enum LocalTransactionState {
	//提交事务
	COMMIT_MESSAGE,
	//回滚事务
	ROLLBACK_MESSAGE,
	//结束事务，但不做任何处理
	UNKNOW,
}
```

一般在使用时，我们在executeLocalTransaction()方法中，返回UNKNOW。由于this.endTransaction 方法执行时，其业务事务还未提交，故而在调用executeLocalTransaction方法时返回UNKNOW，事务具体提交还是回滚通过事务状态回查方法来获取。

### 4.7.5 事务消息的使用限制

1、事务消息不支持延迟消息和批量消息。 

2、为避免单个消息被检查多次,导致半队列消息累积，单个消息的检查次数默认设置为15次，用户可以通过对Broker配置文件中的transactionCheckMax参数进行修改。如果已经检查某条消息超过设置的检查次数，则Broker将丢弃此消息，同时打印错误日志。用户可以通过重写AbstractTransactionCheckListener类来修改这个行为。 

3、事务消息将在Broker配置文件中的参数 transactionMsgTimeout 这样的特定时间长度之后被检查。当发送事务消息时，用户还可以通过设置用户属性 CHECK_IMMUNITY_TIME_IN_SECONDS 来改变这个限制，该参数优先于 transactionMsgTimeout 参数。 

4、事务性消息可能不止一次被检查或消费。 

5、提交给用户的目标主题消息可能会失败，目前这依日志的记录而定。它的高可用性通过RocketMQ 本身的高可用性机制来保证，如果希望确保事务消息不丢失、并且事务完整性得到保证，建议使用同步的双重写入机制。 

6、事务消息的生产者 ID 不能与其他类型消息的生产者 ID 共享。与其他类型的消息不同，事务消息允许反向查询、MQ服务器能通过它们的生产者 ID 查询到消费者

# 五、RocketMQ路由机制(源码分析)

![avatar](images\MQ04.png)

Broker在启动时向Nameserver注册存储在该服务器上的路由信息，后续Broker每30s向NameServer发送心跳包，心跳包中包含主题的路由信息(主题的读写队列数、操作权限等)，NameServer会通过HashMap更新Topic的路由信息，并记录最后一次收到Broker的时间戳。  

Nameserver每隔10s扫描路由表，如果检测到Broker服务宕机，则移除对应的路由信息。NameServer认为Broker宕机的依据是如果当前系统时间戳减去最后一次收到Broker心跳包的时间戳大于120s。  

Producer每隔30s会从Nameserver重新拉取Topic的路由信息并更新本地路由表，即消息生产者并不会立即感知Broker服务器的新增与删除。

## 5.1 构建TopicConfigManager对象

在Broker启动流程中，会构建TopicConfigManager对象，其构造方法中首先会判断是否开启了允许自动创建主题，如果启用了自动创建主题，则向topicConfigTable中添加默认主题的路由信息。  

![avatar](images\MQ16.png)

## 5.2 生产者寻找路由信息

1.消息生产者往broker发送消息时，需要查询本地的路由信息表，确认所发送消息对应topic的路由信息，若本地路由信息表中没有相应信息，则需要主动从Nameserver拉取。  

2.如果Nameserver没有相应topic的路由信息，如果开启了自动创建路由信息，则Nameserver给消息生产者返回默认主题的路由信息。  

```java
public boolean updateTopicRouteInfoFromNameServer(final String topic, boolean isDefault,DefaultMQProducer defaultMQProducer) {
    try {
        if (this.lockNamesrv.tryLock(LOCK_TIMEOUT_MILLIS, TimeUnit.MILLISECONDS)) {
            try {
                TopicRouteData topicRouteData;
                if (isDefault && defaultMQProducer != null) {
                    topicRouteData = this.mQClientAPIImpl.getDefaultTopicRouteInfoFromNameServer(defaultMQProducer.getCreateTopicKey(),1000 * 3);
                    if (topicRouteData != null) {
                        for (QueueData data : topicRouteData.getQueueDatas()) {
                            int queueNums = Math.min(defaultMQProducer.getDefaultTopicQueueNums(), data.getReadQueueNums());
                            data.setReadQueueNums(queueNums);
                            data.setWriteQueueNums(queueNums);
                        }
                    }
                }
			}
		}...
```

## 5.3 发送消息

DefaultMQProducerImpl#sendKernelImpl  

```java
SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
requestHeader.setTopic(msg.getTopic());
requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
requestHeader.setQueueId(mq.getQueueId());
requestHeader.setSysFlag(sysFlag);
requestHeader.setBornTimestamp(System.currentTimeMillis());
requestHeader.setFlag(msg.getFlag());
requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
requestHeader.setReconsumeTimes(0);
requestHeader.setUnitMode(this.isUnitMode());
```

在消息发送时的请求报文中，设置默认topic名称，消息发送topic名称，使用的队列数量为DefaultMQProducer#defaultTopicQueueNums，即默认为4。

## 5.4 Broker 端收到消息后的处理流程

1.broker端收到消息后，会先根据topic查询路由信息，若broker端不存在该topic的路由信息，如果该broker存在默认主题的路由配置，则会根据消息发送请求中的队列数量，创建一个新的topic路由信息。  

2.Broker 端的 topic 配置管理器中存在的路由信息，一边会往Nameserver发送心跳更新其中登记的路由信息，另一边会有定时任务，把路由信息持久化到本地磁盘。  

# 六、RocketMQ高可用核心工作机制

## 6.1 主从同步基本实现过程

![avatar](images\MQ06.png)

1.首先启动 Master 并在指定端口监听；  

2.客户端启动，主动连接 Master，建立TCP连接；  

3.客户端以每隔5s的间隔时间向服务端拉取消息，如果是第一次拉取的话，先获取本地commitlog 文件中最大的偏移量， 以该偏移量向服务端拉取消息；  

4.服务端解析请求，并返回一批数据给客户端；  

5.客户端收到一批消息后， 将消息写入本地commitlog文件中， 然后向Master汇报拉取进度，并更新下一次待拉取偏移量；  

6.然后重复第3步；  

RocketMQ主从同步不具备主从切换功能，当Master节点宕机后，slave节点不会接管消息发送，但可以提供消息读取。

## 6.2 主从读写分离机制

RocketMQ默认会优先选择从Master拉取消息，显然这并不是我们通常意义上的读写分离。  

在RocketMQ中判断是从Master拉取，还是从slave拉取的核心代码如下：

```java
//DefaultMessageStore#getMessage
long diff = maxOffsetPy - maxPhyOffsetPulling; // @1
long memory = (long)(StoreUtil.TOTAL_PHYSICAL_MEMORY_SIZE*(this.messageStoreConfig.getAccessMessageInMemoryMaxRatio()/100.0)); // @2
getResult.setSuggestPullingFromSlave(diff > memory); // @3
```

**代码@1**： 
maxOffsetPy:当前最大的物理偏移量。 返回的偏移量为已存入到操作系统的 PageCache 中的内容。 
maxPhyOffsetPulling本次消息拉取最大物理偏移量，按照消息顺序拉取的基本原则， 可以基本预测下次开始拉取的物理偏移量将大于该值， 并且就在其附近。 
diff:maxOffsetPy与maxPhyOffsetPulling之间的间隔，getMessage通常用于消息消费时，即这个间隔可以理解为目前未处理的消息总大小。 

**代码@2**： 
获取RocketMQ消息存储在PageCache中的总大小，如果当RocketMQ容量超过该阈值，将会将被置换出内存，如果要访问不在PageCache 中的消息，则需要从磁盘读取。 
StoreUtil.TOTAL_PHYSICAL_MEMORY_SIZE:返回当前系统的总物理内存。 
accessMessageInMemoryMaxRatio:设置消息存储在内存中的阀值，默认为40

**代码@3**：设置下次拉起是否从slave拉取标记，触发下次从slave服务器拉取的条件为：当前所有可用消息数据(所有commitlog)文件的大小已经超过了其阈值，默认为物理内存的40%。

GetResult的suggestPullingFromSlave属性使用代码如下：

以下代码会涉及部分配置参数：  

>broker的服务参数slaveReadEnable：slave broker是否可读  
>订阅组配置信息： whichBrokerWhenConsumeSlowly(当消费速度缓慢时从哪个broker拉取消息)、 brokerId(配置主brokerId)  

```java
//PullMessageProcessor#processRequest
if (getMessageResult.isSuggestPullingFromSlave()) { // @1
	responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getWhichBrokerWhenConsumeSlowly());
} else {
	responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
} 
switch (this.brokerController.getMessageStoreConfig().getBrokerRole()) { // @2
	case ASYNC_MASTER:
	case SYNC_MASTER:
		break;
	case SLAVE:
		if (!this.brokerController.getBrokerConfig().isSlaveReadEnable()) {
			response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
			responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
		}
		break;
}
if (this.brokerController.getBrokerConfig().isSlaveReadEnable()) { // @3
// consume too slow ,redirect to another machine
	if (getMessageResult.isSuggestPullingFromSlave()) {
		responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getWhichBrokerWhenConsumeSlowly());
	} else { // consume ok
		responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getBrokerId());
	}
} else {
	responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
}
```

**代码@1**：如果从commitlog文件查找消息时，发现消息堆积太多，默认超过物理内存的40%后，会建议从slave服务器读取。  

**代码@2**：如果当前服务器的角色为slave服务器:并且slaveReadEnable=true，则忽略代码@1设置的值，下次拉取切换为从master拉取。  

**代码@3**：如果slaveReadEnable=true，并且建议从slave服务器读取。从消息消费组角度来看,当消费速度缓慢时，建议拉取的brokerId，由订阅组配置的whichBrokerWhenConsumeSlowly属性决定；如果消息消费速度正常，则使用订阅组建议的brokerId属性，拉取消息进行消费，默认作为master服务器。如果不允许slave可读，则固定使用从master拉取。  

这边需要注意的是，当订阅组的配置为默认值时，当拉取消息请求发送到Slave Broker后，下一次消息拉取，无论是否开启slaveReadEnable，还是会发往Master Broker。  

在消息拉取命令的返回字段中，会将SuggestWhichBrokerId返回给客户端，根据其值从指定的broker拉取。  

消息拉取实现PullAPIWrapper在处理拉取结果时会将服务端建议的brokerId更新到broker拉取缓存表中。  

![avatar](images\MQ08.png)

在发起拉取请求之前，首先根据如下代码，选择待拉取消息的Broker.  

![avatar](images\MQ07.png)

## 6.3 消费进度同步机制

集群模式下，consumer从broker拉取一批消息后提交到消费组特定的线程池中处理，当消费成功后会向Broker发送ACK消息，告知消费端已成功消费到哪条消息，Broker收到消费进度反馈后，首先将相关进度信息存储在内存中，然后通过定时任务持久化到consumeOffset.json文件中。从这里可以看出，集群模式下消息消费进度存储在Broker端。  

主服务的brokerId为0，默认情况下当主服务器存活的时候，客户端优先会选择向Master Broker反馈消息消费进度，只有当Master Broker宕机的情况下，才会选择Slave Broker。  

下面是Broker服务器在收到提交offset反馈命令后的处理逻辑：  

客户端定时向Broker端发送更新offset的请求，其入口为：RemoteBrokerOffsetStore#updateConsumeOffsetToBroker，该方法中一个非常关键的点是：选择broker的逻辑，相关代码如下图所示：  

![avatar](images\MQ09.png)

不管消息是从Master Broker拉取的还是从Slave Broker拉取的，提交消息消费进度请求，优先选择Master Broker。服务端就是接收其偏移量，更新到服务端的内存中，然后通过定时任务持久化到consumerOffset.json文件中。  

consumer首先从Master Broker拉取消息，并向其提交offset，如果当Master Broker宕机后，Slave Broker会接管消息拉取服务，此时的offset存储在Slave Broker。当Master Broker恢复正常后，Master和Slave中存储的offset会不一致，两者之间的消息消费进度如何同步？  

![avatar](images\MQ10.png)

如果Broker角色为slave，会通过定时任务调用syncAll()，从Master定时同步topic路由信息、消息消费进度、延迟队列处理进度、消费组订阅信息。  

这会引出另一个问题，如果Master Broker启动后，Slave Broker马上从Master同步消息消息进度，此时Master发生宕机，这时consumer选择从Slave读取消息，而这时Slave Broker存储的offset还未更新，从而带来重复消费的隐患。  

针对这种场景，RocketMQ提供了两种机制来确保不丢失消息消费进度：

第一种：consumer在内存中存储最新的offset，继续以该进度去服务器拉取消息后，消息处理完后，会定时向Broker服务器反馈offset(优先选择Master Broker)， 此时Master的offset就立马更新了，Slave此时只需定时同步Master的offset即可。  

第二种：consumer在向Master拉取消息时，也会触发Master Broker对消息消费进度的更新。  

```java
//PullMessageProcessor#processRequest
boolean storeOffsetEnable = brokerAllowSuspend; // @1
storeOffsetEnable = storeOffsetEnable && hasCommitOffsetFlag;
storeOffsetEnable = storeOffsetEnable
	&& this.brokerController.getMessageStoreConfig().getBrokerRole() !=BrokerRole.SLAVE; // @2
if (storeOffsetEnable) {
	this.brokerController.getConsumerOffsetManager().commitOffset(RemotingHelper.parseChannelRemoteAddr(channel),requestHeader.getConsumerGroup(),requestHeader.getTopic(),requestHeader.getQueueId(),requestHeader.getCommitOffset());
}
```

brokerAllowSuspend：broker是否允许挂起，在消息拉取时，该值默认为true。  

hasCommitOffsetFlag：消息消费者在内存中是否缓存了消息消费进度，如果有，则该标记为true。
如果Broker的角色为Master，并且上面两个变量都为true，则首先使用commitOffset更新消息消费进度。

# 七、 RocketMQ消息存储

## 7.1 MQ常用存储方式

当前业界几款主流的MQ消息队列采用的存储方式主要有以下三种方式： 

（1）分布式KV存储：这类MQ一般会采用诸如levelDB、RocksDB和Redis来作为消息持久化的方式，由于分布式缓存的读写能力要优于DB，所以在对消息的读写能力要求都不是比较高的情况下，采用这种方式倒也不失为一种可以替代的设计方案。 

（2）文件系统：目前业界较为常用的几款产品（RocketMQ/Kafka/RabbitMQ）均采用的是消息刷盘至所部署虚拟机/物理机的文件系统来做持久化（刷盘一般可以分为异步刷盘和同步刷盘两种模式）。消息刷盘为消息存储提供了一种高效率、高可靠性和高性能的数据持久化方式。除非部署MQ机器本身或是本地磁盘挂了，否则一般是不会出现无法持久化的故障问题。 

3）关系型数据库DB：Apache下开源的另外一款MQ—ActiveMQ（默认采用的KahaDB做消息存储）可选用JDBC的方式来做消息持久化，通过简单的xml配置信息即可实现JDBC消息存储。普通关系型数据库（如Mysql）在单表数据量达到千万级别时，其IO读写性能往往会出现瓶颈。并且，MQ消息队列通常要满足吞吐量大、消息堆积能力突出的要求，采用关系型数据库作为消息持久化的方案显然不是一个很好的选择。同时，由于依赖DB，一旦DB出现故障，MQ的消息则无法落盘存储，会导致线上业务数据丢失。 

综合上所述从存储效率来说， 文件系统>分布式KV存储>关系型数据库DB，直接操作文件系统肯定是最快和最高效的。另外，从消息中间件的本身定义来考虑，应该尽量减少对于外部第三方中间件的依赖。一般来说依赖的外部系统越多，也会使得本身的设计越复杂，所以采用文件系统作为消息存储的方式，更贴近消息中间件本身的定义。

## 7.2 RocketMQ消息存储整体架构

![avatar](images\MQ12.png)  

从整体的设计架构中可以看出：

（1）消息生产与消息消费相互分离，Producer端发送消息最终写入的是CommitLog（消息存储的日志数据文件），Consumer端先从ConsumeQueue（消息逻辑队列）读取持久化消息的起始物理位置偏移量offset、大小size和消息Tag的HashCode值，随后再从CommitLog中进行读取待消费消息的实体内容； 

（2）RocketMQ的CommitLog文件采用混合型存储（所有的Topic下的消息队列共用同一个CommitLog的日志数据文件）。通过建立类似索引文件—ConsumeQueue的方式，区分不同Topic下面的不同MessageQueue的消息，同时为消费消息起到一定的缓冲作用（只有ReputMessageService异步服务线程通过doDispatch异步生成了ConsumeQueue队列的元素后，Consumer端才能进行消费）。这样，只要消息写入，并刷盘至CommitLog文件后，消息就不会丢失，即使ConsumeQueue中的数据丢失，也可以通过CommitLog来恢复。 

（3）发送消息时，生产者端的消息确实是顺序写入CommitLog；订阅消息时，消费者端也是顺序读取ConsumeQueue，然而根据其中的起始物理位置偏移量offset读取消息真实内容却是随机读取CommitLog。  

RocketMQ的具体做法是，使用Broker端的后台服务线程—ReputMessageService不停地分发请求，并异步构建ConsumeQueue（逻辑消费队列）和IndexFile（索引文件）数据。然后，Consumer通过ConsumerQueue查找待消费的消息。其中，ConsumeQueue作为消费消息的索引，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值。而IndexFile只是为消息查询提供了一种通过key或时间区间来查询消息的方法。

## 7.3 RocketMQ存储关键技术

### 7.3.1 PageCache机制

对于操作系统来说，磁盘文件都是由一系列的数据块顺序组成，数据块的大小由操作系统本身而决定，x86的linux中一个标准页面大小是4KB。操作系统内核在处理文件I/O请求时，首先到page cache中查找（page cache中的每一个数据块都设置了文件以及偏移量地址信息），如果未命中，则启动磁盘I/O，将磁盘文件中的数据块加载到page cache中的一个空闲块，然后再copy到用户缓冲区中。 

page cache本身也会对数据文件进行预读取，对于每个文件的第一个读请求操作，系统在读入所请求页面的同时，会读入紧随其后的少数几个页面。因此，想要提高page cache的命中率（尽量让访问的页在物理内存中），从硬件的角度来说肯定是物理内存越大越好。从操作系统层面来说，访问page cache时，即使只访问1k的消息，系统也会提前预读取更多的数据，在下次读取消息时, 就很可能可以命中内存。  

对于文件的顺序读写操作来说，读和写的区域都在OS的PageCache内，此时读写性能接近于内存。RocketMQ将数据文件映射到OS的虚拟内存中（通过JDK NIO的MappedByteBuffer），写消息的时候首先写入PageCache，并通过异步刷盘的方式将消息批量的做持久化（同时也支持同步刷盘）；订阅消费消息时（对CommitLog操作是随机读取），由于PageCache的局部性热点原理，且整体情况下还是从旧到新的有序读，因此大部分情况下消息还是可以直接从Page Cache中读取，不会产生太多的缺页（Page Fault）中断而从磁盘读取。  

![avatar](images\MQ14.png)

### 7.3.2 Mmap内存映射技术—MappedByteBuffer

![avatar](images\MQ13.png)  
MappedByteBuffer继承自ByteBuffer，其内部维护了一个逻辑地址变量—address。在建立映射关系时，MappedByteBuffer利用了JDK NIO的FileChannel类提供的map()方法把文件对象映射到虚拟内存。仔细看源码中map()方法的实现，可以发现最终其通过调用native方法map0()完成文件对象的映射工作，同时使用Util.newMappedByteBuffer()方法初始化MappedByteBuffer实例，但最终返回的是DirectByteBuffer的实例。在Java程序中使用MappedByteBuffer的get()方法来获取内存数据是最终通过DirectByteBuffer.get()方法实现（底层通过unsafe.getByte()方法，以“地址 + 偏移量”的方式获取指定映射至内存中的数据）。
###6.3.3 存储优化技术
当遇到OS进行脏页回写，内存回收，内存swap等情况时，PageCache机制就会引起较大的消息读写延迟。针对这些场景，RocketMQ采用了多种优化技术，主要包括内存预分配，文件预热和mlock系统调用，在一定程度上可以减少PageCache的缺点带来的影响。

**(1) 预先分配MappedFile**

在消息写入过程中（调用CommitLog的putMessage()方法），CommitLog会先从MappedFileQueue队列中获取一个 MappedFile，如果没有就新建一个。MappedFile的创建过程是将构建好的一个AllocateRequest请求（具体做法是，将下一个文件的路径、下下个文件的路径、文件大小为参数封装为AllocateRequest对象）添加至队列中，后台运行的AllocateMappedFileService服务线程（在Broker启动时，该线程就会创建并运行），会不停地运行，只要请求队列里存在请求，就会去执行MappedFile映射文件的创建和预分配工作，分配的时候有两种策略，一种是使用Mmap的方式来构建MappedFile实例，另外一种是从TransientStorePool堆外内存池中获取相应的DirectByteBuffer来构建MappedFile（ps：具体采用哪种策略，也与刷盘的方式有关）。并且，在创建分配完下个MappedFile后，还会将下下个MappedFile预先创建并保存至请求队列中等待下次获取时直接返回。RocketMQ中预分配MappedFile的设计非常巧妙，下次获取时候直接返回就可以不用等待MappedFile创建分配所产生的时间延迟。  

![avatar](images\MQ15.png)

**(2) 文件预热和mlock系统调用**

1）mlock系统调用：其可以将进程使用的部分或者全部的地址空间锁定在物理内存中，防止其被交换到swap空间。  

2）文件预热：由于仅分配内存并进行mlock系统调用后并不会为程序完全锁定这些内存，因为其中的分页可能是写时复制的。因此，就有必要对每个内存页面中写入一个假的值。RocketMQ是在创建并分配MappedFile的过程中，预先写入一些随机值至Mmap映射出的内存空间里。  

调用Mmap进行内存映射后，OS只是建立虚拟内存地址至物理地址的映射表，实际上并没有加载任何文件到内存中。程序要访问数据时OS会检查该部分的分页是否已经在内存中，如果不在，则发出一次缺页中断。  

RocketMQ在做Mmap内存映射的同时，调用madvise系统，目的是使OS做一次内存映射后对应的文件数据尽可能多的预加载至内存中，从而达到内存预热的效果。

## 7.4 消息的存储

### 7.4.1 目录与文件

commitlog目录中存放着很多的mappedFile文件，当前Broker中的所有消息都是落盘到这些mappedFile文件中的，mappedFile的文件大小是1G(小于等于1G)，文件名由20位十进制数构成，表示当前文件的第一条消息的其实位置偏移量。

需要注意的是，一个Broker中仅包含一个commitlog目录，所有的mappedFile文件都。是存放在该目录中的，即无论当前Broker中存放着多少个Topic的消息，这些消息都是被顺序写入到了mappedFile文件中的。也就是说，这些消息在Broker中存放时并没有被按照Topic进行分类存放。

### 7.4.2 消费单元



# 八、 一些重要细节补充

## 7.1 消息重试

由于MQ经常处于复杂的分布式系统中，考虑网络波动、服务宕机、程序异常因素，很有可能出现消息发送或者消费失败的问题。因此，消息的重试就是所有MQ中间件必须考虑到的一个关键点。RocketMQ为使用者封装了消息重试的处理流程，无需开发人员手动处理。RocketMQ支持了生产端和消费端两类重试机制。

### 7.1.1 生产端重试

### 7.1.2 消费端重试

## 7.2 死信队列

## ？？、消息订阅模型

RocketMQ 的消息消费模式采用的是发布与订阅模式。

topic(主题)：一类消息的集合，消息发送者将一类消息发送到一个主题中，例如订单模块将订单发送到order_topic中，而用户登录时，将登录事件发送到login__topic中。 
consumegroup(消费者组):消费者组首先在启动时需要订阅其要消费的topic。一个topic可以被多个消费者组订阅，同样一个消费组也可以订阅多个topic。一个消费组通常都是拥有多个消费者。  

## ？？ 消费模式

RocketMQ的消费模式有广播模式与集群模式两种。
广播模式：一个消费组内的所有消费者每一个都会处理topic中的每一条消息。  
集群模式：一个消费组内的所有消费者共同消费一个topic中的消息，即分工协作，一个消费者消费一部分数据，当然其中必然少不了负载均衡。  

集群模式相对更加常用，符合分布式架构的基本理念，当前消费者如果无法快速及时处理消息时，可以通过增加消费者的个数横向扩容，快速提高消费能力，从而及时处理积压的消息。

## ？？ 消费队列负载算法与重平衡机制

### 2.5.1 消费队列负载算法

RocketMQ提供了众多的队列负载算法，其中最常用的两种平均是平均分配和轮流平均分配算法。  

>平均分配(AllocateMessageQueueAveragely)  
>其算法的特点是用总数除以消费者个数， 余数按消费者顺序分配给消费者。  
>轮流平均分配(AllocateMessageQueueAveragelyByCircle)  
>该分配算法的特点就是轮流一个一个分配。  

用q0~q15表示16个队列，消费者用c0～c2表示。  

平均分配算法的队列负载机制如下：  
c0: q0 q1 q2 q3 q4 q5  
c1: q6 q7 q8 q9 q10  
c2: q11 q12 q13 q14 q15  

轮流平均分配算法的队列负载机制如下：  
c0： q0 q3 q6 q9 q12 q15  
c1: q1 q4 q7 q10 q13  
c2: q2 q5 q8 q11 q14  

**同一个消费者同一时间可以分配多个队列，但一个队列同一时间只会分配给一个消费者**。 
**如果topic的队列个数小于消费者的个数，那有些消费者则无法分配到消息。RocketMQ中一个topic的队列数直接决定了最大消费者的个数，但topic队列个数的增加对RocketMQ的性能不会产生影响。** 

### 2.5.2 消费队列重平衡机制

在实际过程中，对topic进行扩容(增加队列个数)或者对和consumer进行扩容、缩容，这就会涉及到消息消费队列的重新分配，即消费队列重平衡机制。  

RocketMQ客户端中会每隔20s去查询当前topic的所有队列、消费者的个数，运用队列负载算法进行重新分配，然后与上一次的分配结果进行对比，如果发生了变化，则进行队列重新分配。例如新增一个消费者，大致的做法就是原有的消费者(c0、c1、c2)将原先分配给自己但这次不属于的队列进行丢弃，新增的消费者c3则创建新的拉取任务。

### 2.5.3 消费进度

Consumer消费消息后需要记录消费的位置(offset)，这样在Consumer Client重启的时候，继续从上一次的offset开始继续处理新的消息。在RocketMQ中，offset的存储是以消费者组(consumegroup)为单位的。  

集群模式下，消费进度存储在broker端，广播模式的消费进度文件存储在用户的主目录。

### 2.5.4 消费方式

并发消费：对一个队列中消息，每一个消费者内部都会创建一个线程池，对队列中的消息多线程处理，即偏移量大的消息比偏移量小的消息有可能先消费。并发消费过程中，如果消息消费失败的话，默认会重试16次，每一次的间隔时间不一样。  

顺序消费：在某一场景下，消息需要按照顺序进行消费。在RocketMQ中提供了基于队列的顺序消费模型，即尽管一个消费组中的消费者会创建一个多线程，但针对同一个Queue，会加锁。如果一条消息消费失败，则会一直消费，直到消费成功。  