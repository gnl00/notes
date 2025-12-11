# RocketMQ 源码解析

---

# V5

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-client-java</artifactId>
    <version>5.0.7</version>
</dependency>
```

## 生产者客户端启动

org.apache.rocketmq.client.java.impl.producer.ProducerBuilderImpl#build
- org.apache.rocketmq.shaded.com.google.common.util.concurrent.AbstractIdleService#startAsync
- org.apache.rocketmq.shaded.com.google.common.util.concurrent.AbstractService#startAsync
- org.apache.rocketmq.shaded.com.google.common.util.concurrent.AbstractIdleService.DelegateService#doStart
- org.apache.rocketmq.client.java.impl.producer.ProducerImpl#startUp - org.apache.rocketmq.client.java.impl.ClientManagerImpl#startUp
- 生产者启动完成

可以看到 RocketMQ V5 版本在把部分客户端逻辑下放到 Proxy 层之后客户端侧就简单了很多，少了很多逻辑。

## ClientManager

管理生产者/消费者客户端，主要方法有：

- queryRoute 消息路由查询
- heartbeat 心跳
- sendMessage 消息发送
- queryAssignment 消费情况分配
- receiveMessage 消息接收
- ackMessage 消息 ack
- telemetry 遥测数据收集

可以看出 ClientManager 是作为生产者/消费者客户端的父类这样一个角色存在的，定义了生产消费所需的方法。

ClientManager 内部持有指向生产者/消费者的实例 `private final Client client;`

org.apache.rocketmq.client.java.impl.ClientImpl#startUp
- clientManager.startAsync() - ClientManagerImpl#startUp
- fetchTopicRoute
- updateRouteCache

org.apache.rocketmq.client.java.impl.ClientManagerImpl#startUp
- client.doHeartbeat()
- client.doStats() - org.apache.rocketmq.client.java.impl.ClientImpl#doStats
- client.syncSettings() - Synchronize client settings with the remote endpoint

## 消费者客户端启动

org.apache.rocketmq.client.java.impl.consumer.SimpleConsumerBuilderImpl#build
- org.apache.rocketmq.shaded.com.google.common.util.concurrent.AbstractIdleService#startAsync
- org.apache.rocketmq.shaded.com.google.common.util.concurrent.AbstractService#startAsync
- org.apache.rocketmq.shaded.com.google.common.util.concurrent.AbstractIdleService.DelegateService#doStart
- org.apache.rocketmq.client.java.impl.consumer.SimpleConsumerImpl#startUp - org.apache.rocketmq.client.java.impl.ClientManagerImpl#startUp
- 消费者客户端启动完成

## 生产者实例化

org.apache.rocketmq.client.java.impl.consumer.SimpleConsumerBuilderImpl#build
- org.apache.rocketmq.client.java.impl.consumer.SimpleConsumerImpl#SimpleConsumerImpl


## 消费者实例化

## 客户端与 Endpoint

实际上也可以说是客户端与 Proxy 的交互，主要是通过 RPC 来进行：

```shell
org.apache.rocketmq.client.java.rpc.RpcClientImpl#RpcClientImpl
```

发送消息：
- org.apache.rocketmq.client.java.rpc.RpcClientImpl#sendMessage

接收消息：
- org.apache.rocketmq.client.java.rpc.RpcClientImpl#receiveMessage

## 本地消息缓存

ProcessQueue
- Process queue is a cache to store fetched messages from remote for PushConsumer.
- PushConsumer queries assignments periodically and converts them into message queues
- each message queue is mapped into one process queue to fetch message from remote
- If the message queue is removed from the newest assignment, the corresponding process queue is marked as expired soon, which means its lifecycle is over.

```shell
A standard procedure to cache/ erase message
phase 1: Fetch 32 messages successfully from remote.
   32 in ┌─────────────────────────┐
  ───────►           32            │
         └─────────────────────────┘
              cached messages = 32
  
phase 2: consuming 1 message.
         ┌─────────────────────┐   ┌───┐
         │          31         ├───► 1 │ consuming
         └─────────────────────┘   └───┘
              cached messages = 32
  
phase 3: eraseMessage(MessageViewImpl, ConsumeResult) with 1 messages and its consume result.
         ┌─────────────────────┐   ┌───┐ 1 consumed
         │          31         ├───► 0 ├───────────►
         └─────────────────────┘   └───┘
             cached messages = 31
```

Especially, there are some different processing procedures for FIFO consumption.
The server ensures that the next batch of messages will not be obtained by the client
until the previous batch of messages is confirmed to be consumed successfully or not.

---

## 顺序消息实现

首先确定两个概念：

- 全局有序：整个 Topic 只有一个队列，所有消息严格按 FIFO 顺序
- 分区有序：Topic 有多个队列，相同业务标识的消息进入同一队列，保证局部有序

RocketMQ 中的顺序消息是一种“分区有序”，生产者端在设置了消息的 `MessageGroup` 后将顺序消息都发送到一个对应的消息队列中。

**RocketMQ 如何保证消息的有序性？**

### 消息主题

创建支持顺序消息的主题

```shell
sh mqadmin updateTopic -t <topic_name> -c <cluster_name> -a +message.type=FIFO
sh mqadmin updateTopic -n <nameserver_address> -t <topic_name> -c <cluster_name> -a +message.type=FIFO
```

### 生产者端

创建消息时指定 `MessageGroup` 属性

```java
// org.apache.rocketmq.client.java.impl.producer.ProducerImpl#send(java.util.List<org.apache.rocketmq.client.apis.message.Message>, boolean)

// Prepare the candidate message queue(s) for retry-sending in advance.
final List<MessageQueueImpl> candidates = null == messageGroup ? takeMessageQueues(result) :
    Collections.singletonList(result.takeMessageQueueByMessageGroup(messageGroup));
final SettableFuture<List<SendReceiptImpl>> future0 = SettableFuture.create();
send0(future0, topic, messageType, candidates, pubMessages, 1);
```

如果设置了 messageGroup，在发送消息的时候会调用 `takeMessageQueueByMessageGroup` 方法，该方法会返回一个与 MessageGroup 对应的消息队列，生产者往该消息队列顺序发送消息，消费者从该消息队列顺序消费消息。

### 消费者端

RocketMQ 给每个消息队列分配一个消费者，一个消费者对应多个消费线程。当消费者拉取顺序消息的时候会先给它所负责的消息队列加锁，然后再进行消费，以此来保证消息消费的顺序性。

可以看看这两个类的实现：

```
org.apache.rocketmq.client.impl.consumer.ConsumeMessageOrderlyService
org.apache.rocketmq.client.impl.consumer.ConsumeMessagePopOrderlyService
```

### 顺序消息实现

> https://juejin.cn/post/7320169906222841856

---

## 数据分层存储

> https://juejin.cn/post/7340603873605222435

<br>

---

## Controller层

> https://juejin.cn/post/7295036072755822604

Controller 启动方式有两种：

1、内嵌在 NameServer 中启动

```shell
org.apache.rocketmq.namesrv.NamesrvStartup#main
```

2、单独部署启动

```shell
org.apache.rocketmq.controller.ControllerStartup#main
```

---

# 提问

## 1、为什么 RocketMQ 规定每个 MessageQueue 只能被一个消费者消费？

> 为什么 RocketMQ 规定每个 MessageQueue 只能被一个消费者消费？而不是多个消费者可消费？我知道其中有一点考虑是为了保证顺序消息的实现，
> 但是这可以进行区分的：顺序消息每个 MessageQueue 只能被一个消费者消费；非顺序消息（普通消息）多个消费者可消费。这样不是能一定程度上缓解消息积压的问题吗？

```shell
你提出了一个直击 RocketMQ 乃至 Kafka 核心设计理念的绝佳问题！你的想法非常符合直觉：既然普通消息不需要顺序，为什么不让多个消费者来一起消费一个队列，从而更快地清空积压呢？

答案是：这是一个为了实现极致的水平扩展性、系统简单性和高吞tǔ量而做出的关键设计权衡。

简而言之，“一个队列在同一时刻只能被一个消费者（在同一个消费组内）消费” 这条规则，是 RocketMQ 实现简单、高效的**消费进度管理（Offset Management）和负载均衡（Rebalance）**的基石。

让我们深入拆解一下，如果允许多个消费者处理同一个队列，会带来多么复杂的问题。

1. 核心障碍：无法管理的消费进度 (Offset)
   这是最根本、最致命的问题。

现状：简单的“书签”模型
在当前模型下，Broker 只需要为每个【消费组 + 队列】组合维护一个连续的消费位点（Offset）。你可以把它想象成一个书签。

工作流程：

消费者 A 问 Broker：“Group-X 消费 Queue-1 的书签在哪？” Broker 回答：“在第 100 条。”

消费者 A 从 Queue-1 的第 100 条开始，拉取一批消息（比如 32 条）。

消费者 A 处理完这 32 条消息。

消费者 A 向 Broker 报告：“我把 Queue-1 的书签更新到第 132 条了。”

这个模型非常简单、高效。Broker 的负担极小，它只需要记录一个数字。

你的提议：混乱的“打孔”模型
如果允许多个消费者（A、B、C）同时消费 Queue-1，会发生什么？

工作流程：

Broker 把消息 100-109 推给消费者 A，110-119 推给 B，120-129 推给 C。

消费者 A 处理得很快，完成了 100-109，并发送了 ACK（确认）。

消费者 C 也完成了 120-129，并发送了 ACK。

灾难来了：消费者 B 突然宕机了，它手里的 110-119 消息没有被处理，也没有 ACK。

现在，Broker 面临一个巨大的难题：Queue-1 的消费进度是多少？

它不能简单地把“书签”更新到 130，因为中间的 110-119 消息丢失了。

它也不能让书签停在 100，因为 100-109 和 120-129 已经被处理了，如果新来的消费者从 100 开始，就会导致消息重复消费。

为了解决这个问题，Broker 不得不从维护一个简单的数字（Offset），变成维护一个复杂的数据结构，记录每一条消息是否被 ACK。
这就像一张长长的清单，处理过的消息后面要打个勾。这被称为**“选择性确认”**。

这种“打孔”式的进度管理会带来一系列严重后果：

Broker 复杂度剧增：Broker 变得“重”了，需要耗费大量内存和 CPU 来管理海量消息的 ACK 状态。

性能急剧下降：对每一条消息进行状态跟踪和确认，远比只更新一个连续的 Offset 要慢得多。

故障恢复困难：当消费者宕机，Broker 需要精确地找出哪些消息“掉”了，然后重新派发给其他消费者，这个逻辑非常复杂。

2. 真正的扩容之道：增加队列（Partitions）
   RocketMQ 和 Kafka 解决消息积压问题的“标准答案”不是让一个队列被多人消费，而是增加队列的数量，实现水平扩展。

MessageQueue（分区）是 RocketMQ 中并行处理的最小单元。

场景：你有 1 个 Topic，4 个队列。

1 个消费者：它会消费所有 4 个队列。

2 个消费者：每个消费者分配 2 个队列。

4 个消费者：每个消费者分配 1 个队列，此时达到最大并行度。

8 个消费者：其中 4 个会工作，另外 4 个会空闲，等待 Rebalance。

结论：如果你觉得消费不过来，正确的做法是：

增加消费者实例的数量，直到消费者数量等于队列数量。

如果这还不够，就增加 Topic 的队列数量，然后再相应地增加消费者实例。

这种通过增加分区来扩容的方式，完美地避开了上面提到的“消费进度管理”难题，让整个系统架构保持简单、优雅和极高的可伸缩性。

总结对比
特性	RocketMQ/Kafka 模型 (1 Queue : 1 Consumer)	你的提议 (1 Queue : N Consumers)
消费进度	连续的 Offset (单一“书签”)	离散的 ACK (复杂的“打孔列表”)
Broker 状态	轻量级/无状态 (只记录一个数字)	重状态 (需跟踪每条消息的ACK)
扩容方式	水平扩展 (增加队列和消费者数量)	垂直扩展 (增强单个队列的处理能力)
系统复杂度	低，逻辑清晰	高，故障恢复和状态管理复杂
性能	极高，批量拉取和提交 Offset	较低，需要处理单条或小批量的 ACK

因此，“一个队列只被一个消费者消费”并非功能上的缺失，而是一个经过深思熟虑的、用“增加分区”换取“系统简单性和极致性能”的经典分布式设计。
它将并行消费的控制权交给了运维者（通过调整分区和消费者数量），而不是让 Broker 陷入管理微观消费进度的泥潭。
```

# GoogleGuava

## ListenableFuture

Guava 对标准 Java java.util.concurrent.Future 接口的扩展。标准 Future 的主要缺点是只能通过阻塞的 get() 方法或轮询 isDone() 来获取结果，无法在操作完成后自动执行回调。

ListenableFuture 增加了 addListener(Runnable listener, Executor executor) 方法，可以注册一个监听器（回调函数），在异步计算完成时（成功或失败）执行，同时还可以指定运行该监听器的执行器（线程池）。
如果 Future 在添加监听器时已经完成，监听器会立即执行。这使得构建复杂的异步操作链成为可能，而无需阻塞线程。

## Futures

### Futures.transform

```shell
Returns a new Future whose result is derived from the result of the given Future. 
If input fails, the returned Future fails with the same exception (and the function is not invoked). 
```

## MoreExecutors

### MoreExecutors.directExecutor

```shell
Returns an Executor that runs each task in the thread that invokes execute, as in ThreadPoolExecutor.CallerRunsPolicy.
This executor is appropriate for tasks that are lightweight and not deeply chained.
```

---

# V4

## 生产者启动

**DefaultMQProducerImpl#start**

1、如果 ServiceState 为 CREATE_JUST，继续启动流程，否则启动失败

2、检查配置，检查是否属于 CLIENT_INNER_PRODUCER_GROUP

3、创建 MQClientInstance，注册生产者

4、调用 MQClientInstance#start 启动生产者

```java
// Start request-response channel
this.mQClientAPIImpl.start();
// Start various schedule tasks
this.startScheduledTask();
// Start pull service
this.pullMessageService.start();
// Start rebalance service
this.rebalanceService.start();
// Start push service
this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
log.info("the client factory [{}] start OK", this.clientId);
this.serviceState = ServiceState.RUNNING;
```

5、直连 Broker，发送心跳到 Broker

```java
public void start(final boolean startFactory) throws MQClientException {
    switch (this.serviceState) {
        case CREATE_JUST:
            this.serviceState = ServiceState.START_FAILED;
            this.checkConfig();
            if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                this.defaultMQProducer.changeInstanceNameToPID();
            }

            this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQProducer, rpcHook);

            boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
            if (!registerOK) {
                this.serviceState = ServiceState.CREATE_JUST;
                throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()
                    + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                    null);
            }

            this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());

            if (startFactory) {
                mQClientFactory.start();
            }

            log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),
                this.defaultMQProducer.isSendMessageWithVIPChannel());
            this.serviceState = ServiceState.RUNNING;
            break;
        case RUNNING:
        case START_FAILED:
        case SHUTDOWN_ALREADY:
            throw new MQClientException("The producer service state not OK, maybe started once, "
                + this.serviceState
                + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                null);
        default:
            break;
    }
    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
    this.startScheduledTask();
}
```

> 生产者启动流程和消费者启动流程的逻辑相似

<br>

## 消费者启动

<br>

### DefaultMQPushConsumer

> 大多数情况下都推荐使用 DefaultMQPushConsumer 来进行消息消费。Push 客户端的底层实际上是包装了一层 Pull 服务来实现的。

**DefaultMQPushConsumer#start**

1、设置消费者组

2、DefaultMQPushConsumerImpl#start 启动消费者

```java
// 启动消费者等待消费
public void start() throws MQClientException {
    setConsumerGroup(NamespaceUtil.wrapNamespace(this.getNamespace(), this.consumerGroup)); // 设置消费者所属的消费者组
    this.defaultMQPushConsumerImpl.start();
    // ...
}
```

**DefaultMQPushConsumerImpl#start**

1、启动消费者，启动过程中如果 ServiceState 为 RUNNING/START_FAILED/SHUTDOWN_ALREADY 则启动失败；如果 ServiceState 为 CREATE_JUST 则继续

2、检查消费者配置，复制订阅信息、获取消费者客户端实例 MQClientInstance

3、获取消费 Offset，如果是 BROADCASTING 模式从本地获取，如果是 CLUSTERING 模式则从远程（Broker）获取

4、设置并启动 ConsumeMessageService（MessageListenerOrderly or MessageListenerConcurrently）

5、在 MQClientInstance 客户端实例中注册当前消费者，并启动 MQClientInstance。

MQClientInstance#start 还会启动 MQClientAPIImpl，PullMessageService，RebalanceService 等服务

```java
// MQClientInstance#start
// Start request-response channel
this.mQClientAPIImpl.start();
// Start various schedule tasks
this.startScheduledTask();
// Start pull service
this.pullMessageService.start(); // PullMessageService 启动后开始拉取消息进行消费
// Start rebalance service
this.rebalanceService.start();
// Start push service
this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
log.info("the client factory [{}] start OK", this.clientId);
```

6、客户端实例启动成功，将 ServiceState 设置为 RUNNING

7、启动成功后，更新订阅信息，检查 Broker 信息变更，消费者发送心跳到 Broker，消费者 rebalance

```java
public synchronized void start() throws MQClientException {
    switch (this.serviceState) {
        case CREATE_JUST:
            log.info("the consumer [{}] start beginning. messageModel={}, isUnitMode={}", this.defaultMQPushConsumer.getConsumerGroup(),
                this.defaultMQPushConsumer.getMessageModel(), this.defaultMQPushConsumer.isUnitMode());
            this.serviceState = ServiceState.START_FAILED;

            this.checkConfig();

            this.copySubscription();

            if (this.defaultMQPushConsumer.getMessageModel() == MessageModel.CLUSTERING) {
                this.defaultMQPushConsumer.changeInstanceNameToPID();
            }

            this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook);

            this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer.getConsumerGroup());
            this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());
            this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
            this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);

            this.pullAPIWrapper = new PullAPIWrapper(
                mQClientFactory,
                this.defaultMQPushConsumer.getConsumerGroup(), isUnitMode());
            this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);

            if (this.defaultMQPushConsumer.getOffsetStore() != null) {
                this.offsetStore = this.defaultMQPushConsumer.getOffsetStore();
            } else {
                switch (this.defaultMQPushConsumer.getMessageModel()) {
                    case BROADCASTING:
                        this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                        break;
                    case CLUSTERING:
                        this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                        break;
                    default:
                        break;
                }
                this.defaultMQPushConsumer.setOffsetStore(this.offsetStore);
            }
            this.offsetStore.load();

            if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
                this.consumeOrderly = true;
                this.consumeMessageService =
                    new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
            } else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
                this.consumeOrderly = false;
                this.consumeMessageService =
                    new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
            }

            this.consumeMessageService.start();

            boolean registerOK = mQClientFactory.registerConsumer(this.defaultMQPushConsumer.getConsumerGroup(), this);
            if (!registerOK) {
                this.serviceState = ServiceState.CREATE_JUST;
                this.consumeMessageService.shutdown(defaultMQPushConsumer.getAwaitTerminationMillisWhenShutdown());


            mQClientFactory.start();
            this.serviceState = ServiceState.RUNNING;
            break;
        case RUNNING:
        case START_FAILED:
        case SHUTDOWN_ALREADY:
            throw new MQClientException("The PushConsumer service state not OK, maybe started once, "
                + this.serviceState
                + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                null);
        default:
            break;
    }

    this.updateTopicSubscribeInfoWhenSubscriptionChanged(); // 更新订阅信息
    this.mQClientFactory.checkClientInBroker(); // 检查 Broker 信息变更
    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock(); // 发送心跳到 Broker
    this.mQClientFactory.rebalanceImmediately(); // 消费者 rebalance
}
```

> 消费者消息接收监听的背后也是依靠 Netty

<br>

### DefaultLitePullConsumer

**DefaultLitePullConsumer#start**

```java
public void start() throws MQClientException {
    setTraceDispatcher(); // TraceDispatcher is a Interface of asynchronous transfer data
    setConsumerGroup(NamespaceUtil.wrapNamespace(this.getNamespace(), this.consumerGroup));
    this.defaultLitePullConsumerImpl.start();
    if (null != traceDispatcher) {
        try {
            traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
        }
    }
}
```

**DefaultLitePullConsumerImpl#start**

1、如果客户端的 ServiceState 为 CREATE_JUST 则尝试启动；否则启动失败

2、初始化 MQClientInstance（其中包括注册消费者）、RebalanceImpl、PullAPIWrapper、OffsetStore

3、执行 MQClientInstance#start，同时启动 PullMessageService

```java
// Start pull service
this.pullMessageService.start();
```

4、PullMessageService#run 方法中调用 PullMessageService#pullMessage，进行消息拉取

```java
try {
    PullRequest pullRequest = this.pullRequestQueue.take();
    this.pullMessage(pullRequest); // PullMessageService#pullMessage
}
```

5、最后调用 DefaultMQPushConsumerImpl#pullMessage 逻辑和 DefaultMQPushConsumer 类似

```java
public synchronized void start() throws MQClientException {
    switch (this.serviceState) {
        case CREATE_JUST:
            /**...**/
            initMQClientFactory();
            initRebalanceImpl();
            initPullAPIWrapper();
            initOffsetStore();
            mQClientFactory.start();
            startScheduleTask();
            this.serviceState = ServiceState.RUNNING;
            operateAfterRunning();
            break;
        /**...**/
    }
}
```

<br>

## 消息生产

### 普通消息

**DefaultMQProducer#send**

1、使用 namespace 包装 topic

2、调用 DefaultMQProducerImpl#sendDefaultImpl，不管是同步/异步/单向消息都是用这个方法，只是传入的参数 CommunicationMode 不同而已

3、检查生产者状态是否正常

4、校验消息（消息/消息体是否为空，消息长度是否为 0，消息体大小是否超过限制）

5、尝试找到发送的 topic 信息（如果 topicPublishInfoTable 中无 topic 信息则新增，或者 updateTopicRouteInfoFromNameServer）；如果存在 topic 则发送，否则抛出异常

6、获取 Broker 信息，并从对应的 Broker 上的 Topic 获取到 MessageQueue

7、给消息设置包装后的 topic

8、执行 DefaultMQProducerImpl#sendKernelImpl 发送消息

```java
public SendResult send(Message msg,
    long timeout) /**...*/ {
    msg.setTopic(withNamespace(msg.getTopic())); // 使用 namespace 包装 topic
  	// 调用 DefaultMQProducerImpl#sendDefaultImpl，不管是同步/异步/单向消息都是用这个方法，		// 只是传入的参数 CommunicationMode 不同而已
    return this.defaultMQProducerImpl.send(msg, timeout);
}
```

```java
private SendResult sendDefaultImpl(/**...*/) /**...*/ {
    this.makeSureStateOK(); // 检查生产者状态是否正常
  	// 校验消息（消息/消息体是否为空，消息长度是否为 0，消息体大小是否超过限制）
    Validators.checkMessage(msg, this.defaultMQProducer);
  	/**...*/
  	// 尝试找到发送的 topic 信息（如果 topicPublishInfoTable 中无 topic 信息则新增，或者 updateTopicRouteInfoFromNameServer）
    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
  	// 如果存在 topic 则发送，否则抛出异常
    if (topicPublishInfo != null && topicPublishInfo.ok()) {
        /**...*/
      	// 获取发送时长
      	// retryTimesWhenSendFailed=2
      	// 所以如果是同步发送则 timesTotal=3，否则为 1
        int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
        int times = 0;
        String[] brokersSent = new String[timesTotal];
        for (; times < timesTotal; times++) {
          	// 获取 Broker 信息
            String lastBrokerName = null == mq ? null : mq.getBrokerName();
          	// 从对应的 Broker 上的 Topic 获取到 MessageQueue
            MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
            if (mqSelected != null) {
                mq = mqSelected;
              	// brokersSent = {brokerName1, brokerName2, ...}
                brokersSent[times] = mq.getBrokerName();
                try {
                    beginTimestampPrev = System.currentTimeMillis();
                    if (times > 0) {
                        // Reset topic with namespace during resend.
                        // 设置包装后的 topic
                        msg.setTopic(this.defaultMQProducer.withNamespace(msg.getTopic())); 
                    }
                    long costTime = beginTimestampPrev - beginTimestampFirst;
                    if (timeout < costTime) {
                        callTimeout = true;
                        break;
                    }
										// 发送消息
                    sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
                    endTimestamp = System.currentTimeMillis();
                    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                    switch (communicationMode) {
                        case ASYNC:
                            return null;
                        case ONEWAY:
                            return null;
                        case SYNC:
                            if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                                if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
                                    continue;
                                }
                            }

                            return sendResult;
                        default:
                            break;
                    }
                }
            } else {
                break;
            }
        }

        if (sendResult != null) {
            return sendResult;
        }
        /**...*/
}
```

<br>

**DefaultMQProducerImpl#sendKernelImpl**

1、获取 Broker 地址，如果为 null 报异常，否则继续发送

2、接下来是设置一些消息属性，如消息体是否压缩，是否存在 hook 函数，设置消息发送请求头

3、检查是何种消息，同步/异步/单向，并预设置发送结果。单向消息一般不关心发送结果。只有同步/异步消息需要设置。主要包括 Broker 地址、Broker 名字、消息内容、发送请求头等信息

4、调用 MQClientAPIImpl#sendMessage，如果是异步消息，需要设置 callback，同步和单向走一个逻辑，不需要设置 callback

```java
private SendResult sendKernelImpl(/**...*/) /**...*/ {
    long beginStartTime = System.currentTimeMillis();
  	// 获取 Broker 地址
    String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
    if (null == brokerAddr) {
        tryToFindTopicPublishInfo(mq.getTopic());
        brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
    }

    SendMessageContext context = null;
  	// 如果为 null 报异常，否则继续发送
    if (brokerAddr != null) {
        brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);

        byte[] prevBody = msg.getBody(); // 获取消息体
        try {
            /**...*/
          	// set requestHeader
          
            // set some properties
          
          	// 检查是何种消息，同步/异步/单向，并预设置发送结果。单向消息一般不关心发送结果。只有同步/异步消息需要设置。主要包括 Broker 地址、Broker 名字、消息内容、发送请求头等信息

            SendResult sendResult = null;
            switch (communicationMode) {
                case ASYNC:
                    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                        brokerAddr,
                        mq.getBrokerName(),
                        tmpMessage,
                        requestHeader,
                        timeout - costTimeAsync,
                        communicationMode,
                        sendCallback,
                        topicPublishInfo,
                        this.mQClientFactory,
                        this.defaultMQProducer.getRetryTimesWhenSendAsyncFailed(),
                        context,
                        this);
                    break;
                case ONEWAY: // ONEWAY 走的是 SYNC 的逻辑
                case SYNC:
                    /**...*/
                    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                        brokerAddr,
                        mq.getBrokerName(),
                        msg,
                        requestHeader,
                        timeout - costTimeSync,
                        communicationMode,
                        context,
                        this);
                    break;
                default:
                    assert false;
                    break;
            }
            return sendResult;
        }
    }
    throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);
}
```

<br>

**MQClientAPIImpl#sendMessage**

```java
public SendResult sendMessage(/**...*/) /**...*/ {
    // ...
  	// 单向 invokeOneway
  	// 同步 sendMessageSync
  	// 异步 sendMessageAsync
    switch (communicationMode) {
        case ONEWAY:
            this.remotingClient.invokeOneway(addr, request, timeoutMillis);
            return null;
        case ASYNC:
            /**...*/
            this.sendMessageAsync(addr, brokerName, msg, timeoutMillis - costTimeAsync, request, sendCallback, topicPublishInfo, instance,
                retryTimesWhenSendFailed, times, context, producer);
            return null;
        case SYNC:
            /**...*/
            return this.sendMessageSync(addr, brokerName, msg, timeoutMillis - costTimeSync, request);
        default:
            assert false;
            break;
    }

    return null;
}
```

<br>

#### 同步消息

```java
private SendResult sendMessageSync(/**...*/) /**...*/ {
    RemotingCommand response = this.remotingClient.invokeSync(addr, request, timeoutMillis);
  	/**...*/
}
```

```java
public RemotingCommand invokeSync(/**...**/) /**...**/ {
  	/**...**/
    RemotingCommand response = this.invokeSyncImpl(channel, request, timeoutMillis - costTime);
  	/**...**/
}
```

```java
public RemotingCommand invokeSyncImpl(/**...*/) /**...*/ {
    final int opaque = request.getOpaque();
    try {
        final ResponseFuture responseFuture = new ResponseFuture(channel, opaque, timeoutMillis, null, null);
        this.responseTable.put(opaque, responseFuture);
        final SocketAddress addr = channel.remoteAddress();
      	// 使用 netty 发送
      	// writeAndFlush 将一个消息写入到 Channel 中，并将其发送
      	// 类似于 OutputStream#write，不同的是，它是“异步非阻塞”的
        channel.writeAndFlush(request).addListener(/**...**/);

        RemotingCommand responseCommand = responseFuture.waitResponse(timeoutMillis); // wait for response
        /**...**/
        return responseCommand;
    } finally {
        this.responseTable.remove(opaque);
    }
}
```

#### 异步消息

```java
private void sendMessageAsync(/**...*/) /**...*/ {
    final long beginStartTime = System.currentTimeMillis();
    this.remotingClient.invokeAsync(addr, request, timeoutMillis, new InvokeCallback() {
        @Override
        public void operationComplete(ResponseFuture responseFuture) {
            long cost = System.currentTimeMillis() - beginStartTime;
            RemotingCommand response = responseFuture.getResponseCommand();
            if (null == sendCallback && response != null) {
              	// 无 Callback 且 response != null，直接设置返回值
              	// ...
                return;
            }
            if (response != null) { // 成功得到响应且有 Callback
                try {
                    sendCallback.onSuccess(sendResult);
                }
            }
        }
    });
}
```

```java
public void invokeAsync(/**...*/) /**...*/ {
    long beginStartTime = System.currentTimeMillis();
    final Channel channel = this.getAndCreateChannel(addr);
    if (channel != null && channel.isActive()) {
        try {
            /**...*/
            this.invokeAsyncImpl(channel, request, timeoutMillis - costTime, invokeCallback);
        }
    /**...*/
}
```

```java
public void invokeAsyncImpl(/**...*/) /**...*/ {
    long beginStartTime = System.currentTimeMillis();
    final int opaque = request.getOpaque();
  	// 获取异步信号量
		// Semaphore to limit maximum number of on-going asynchronous requests
    boolean acquired = this.semaphoreAsync.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS);
    if (acquired) { // 信号量获取成功发送
        /**...*/
        final ResponseFuture responseFuture = new ResponseFuture(channel, opaque, timeoutMillis - costTime, invokeCallback, once);
        this.responseTable.put(opaque, responseFuture);
        try {
            channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture f) throws Exception {
                    if (f.isSuccess()) {
                        responseFuture.setSendRequestOK(true);
                        return;
                    }
                    requestFail(opaque);
                    log.warn("send a request command to channel <{}> failed.", RemotingHelper.parseChannelRemoteAddr(channel));
                }
            });
        } /**...*/
    }
  	/**...*/
}
```

#### 单向消息

```java
public void invokeOneway(/**...*/) /**...*/ {
  /**...*/
  this.invokeOnewayImpl(channel, request, timeoutMillis);
  /**...*/
}
```

```java
public void invokeOnewayImpl(/**...*/) /**...*/ {
    request.markOnewayRPC();
  	// Semaphore to limit maximum number of on-going one-way requests
    boolean acquired = this.semaphoreOneway.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS);
    if (acquired) {
        final SemaphoreReleaseOnlyOnce once = new SemaphoreReleaseOnlyOnce(this.semaphoreOneway);
        try {
            channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture f) throws Exception {
                    once.release();
                    if (!f.isSuccess()) {
                        log.warn("send a request command to channel <" + channel.remoteAddress() + "> failed.");
                    }
                }
            });
        } /**...*/
    } /**...*/
}
```

<br>

#### 总结

> 总体来说，同步/异步单向消息的发送逻辑大体相同，最后都是借助 Netty  的 ChannelOutboundInvoker#writeAndFlush 进行非阻塞发送。在各自的 invokeXXXImpl 方法中，只有单向和异步需要使用到信号量来控制发送线程。因为单向消息不关心发送结果，无所谓发送成功与否；而异步消息初衷就是异步发送，自然需要主线程之外的其他线程来执行发送操作。

<br>

### 事务消息

**TransactionMQProducer#sendMessageInTransaction**

```java
public TransactionSendResult sendMessageInTransaction(final Message msg,
    final Object arg) throws MQClientException {
    if (null == this.transactionListener) {
        throw new MQClientException("TransactionListener is null", null);
    }
    msg.setTopic(NamespaceUtil.wrapNamespace(this.getNamespace(), msg.getTopic()));
    return this.defaultMQProducerImpl.sendMessageInTransaction(msg, null, arg);
}
```

**DefaultMQProducerImpl#sendMessageInTransaction**

```java
public TransactionSendResult sendMessageInTransaction(final Message msg,
    final LocalTransactionExecuter localTransactionExecuter, final Object arg)
    throws MQClientException {
  	// 事务消息监听器不能为 null
    TransactionListener transactionListener = getCheckListener(); 
    if (null == localTransactionExecuter && null == transactionListener) {
        throw new MQClientException("tranExecutor is null", null);
    }

  	// 事务消息会自动忽略 DelayTimeLevel 参数
    // ignore DelayTimeLevel parameter
    if (msg.getDelayTimeLevel() != 0) {
        MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_DELAY_TIME_LEVEL);
    }

  	// 校验消息是否合法
    Validators.checkMessage(msg, this.defaultMQProducer);

  	// 设置事务消息属性
    SendResult sendResult = null;
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_TRANSACTION_PREPARED, "true");
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_PRODUCER_GROUP, this.defaultMQProducer.getProducerGroup());
    try {
        sendResult = this.send(msg); // send 方法回到上面普通消息的发送逻辑
    } catch (Exception e) {
        throw new MQClientException("send message Exception", e);
    }

  	// 本地事务消息表中的事务消息状态默认为 UNKNOW
    LocalTransactionState localTransactionState = LocalTransactionState.UNKNOW;
    Throwable localException = null;
    switch (sendResult.getSendStatus()) {
        case SEND_OK: { // 事务消息发送成功
            try {
              	// 设置事务消息发送结果
                // 设置事务消息 Id
                // 设置事务消息状态 UNKNOWN or COMMIT
            }
        }
        break;
        case FLUSH_DISK_TIMEOUT: // 事务消息发送失败 ROLLBACK
        case FLUSH_SLAVE_TIMEOUT:
        case SLAVE_NOT_AVAILABLE:
            localTransactionState = LocalTransactionState.ROLLBACK_MESSAGE;
            break;
        default:
            break;
    }

    // 事务消息发送结束

    TransactionSendResult transactionSendResult = new TransactionSendResult();
    // set transactionSendResult
    return transactionSendResult;
}
```

<br>

#### 总结

> 事务消息与普通消息的不同点仅出现在 DefaultMQProducer#send 方法调用前/后，DefaultMQProducer#send 方法调用中的逻辑和普通消息是一样的。调用前/后主要是设置事务消息的属性，以及本地事务消息表中的消息状态。

<br>

## 消息消费

RocketMQ 支持推和拉两种消费模式。推模式由 MQ 主导，收到消息后就发送给消费者客户端。拉模式由消费者客户端主导，由自定义的消息拉取消费逻辑实现。

实际都是经过拉模式实现的。MQPullConsumer 主动从 MQ 拉取，由客户端自定义实现消息拉取与消费逻辑；MQPushConsumer 则将消息的拉取相关操作封装，将消费接口暴露给开发者实现。

<br>

### PushConsumer 消息拉取

在启动 DefaultMQPushConsumer 的时候会实例化并启动 MQClientInstance。启动 MQClientInstance 的过程中就包含了启动 PullMessageService，而 PullMessageService 就是消息拉取的关键类。

> 从这里也能看出 RocketMQ 的 Push 逻辑的底层是依靠 Pull 来实现的。

#### PullMessageService

**PullMessageService#run**

```java
public void run() {
    while (!this.isStopped()) {
        try {
            PullRequest pullRequest = this.pullRequestQueue.take();
            this.pullMessage(pullRequest); // 拉取消息
        }
    }
}
```

**PullMessageService#pullMessage**

```java
private void pullMessage(final PullRequest pullRequest) {
    final MQConsumerInner consumer = this.mQClientFactory.selectConsumer(pullRequest.getConsumerGroup());
    if (consumer != null) {
        DefaultMQPushConsumerImpl impl = (DefaultMQPushConsumerImpl) consumer;
        impl.pullMessage(pullRequest);
    }
}
```

#### DefaultMQPushConsumerImpl#pullMessage

1、获取消息队列

2、检查消费者者状态，消费者状态非 RUNNING 则异常

3、检查消息队列中消息数量 Count 和消息大小 Size。如果 Count 大于 pullThresholdForQueue 则 executePullRequestLater；如果 Size 大于 pullThresholdSizeForQueue 则 executePullRequestLater

4、检查是顺序消费还是并发消费。如果是顺序消费，检查当前线程是否已经获取到锁，获取到锁则消费消息，否则 executePullRequestLater；如果是并发消费，检查队列中最大消费跨度是否超过客户端最大消费跨度，如果是则 executePullRequestLater

5、获取队列订阅信息，检查和客户端的订阅信息是否一致

6、定义 PullCallback，包含 onSuccess 和 onException 两个方法

7、如果消费模式是 CLUSTERING，则从本地获取消费 Offset

8、检查是否存在 Tag

9、将消息队列、订阅信息、消费 Offset等信息传入 PullAPIWrapper#pullKernelImpl

```java
public void pullMessage(final PullRequest pullRequest) {
    final ProcessQueue processQueue = pullRequest.getProcessQueue();
    if (processQueue.isDropped()) {
        log.info("the pull request[{}] is dropped.", pullRequest.toString());
        return;
    }

    pullRequest.getProcessQueue().setLastPullTimestamp(System.currentTimeMillis());

    // 检查消费者状态
  	// ...

  	// 获取队列中消息总条数
    long cachedMessageCount = processQueue.getMsgCount().get();
  	// 队列中消息大小
    long cachedMessageSizeInMiB = processQueue.getMsgSize().get() / (1024 * 1024);

  	// 如果队列中消息 Count 或 Size 大于 PullThreshold 则 executePullRequestLater
    // ...

    // 顺序消费 or 并发消费
  	// ...

    // 获取订阅信息...

  	// 定义 PullCallback
    // PullCallback pullCallback = new PullCallback()

  	// if MessageModel.CLUSTERING then Enable commitOffset
    // ...

    // get subExpression

    int sysFlag = PullSysFlag.buildSysFlag(
        commitOffsetEnable, // commitOffset
        true, // suspend
        subExpression != null, // subscription
        classFilter // class filter
    );
    try {
        this.pullAPIWrapper.pullKernelImpl(
            pullRequest.getMessageQueue(),
            subExpression,
            subscriptionData.getExpressionType(),
            subscriptionData.getSubVersion(),
            pullRequest.getNextOffset(),
            this.defaultMQPushConsumer.getPullBatchSize(),
            sysFlag,
            commitOffsetValue,
            BROKER_SUSPEND_MAX_TIME_MILLIS,
            CONSUMER_TIMEOUT_MILLIS_WHEN_SUSPEND,
            CommunicationMode.ASYNC,
            pullCallback
        );
    } /**...**/
}
```

#### PullAPIWrapper#pullKernelImpl

```java
public PullResult pullKernelImpl(/**...**/) /**...**/ {
    // 获取 Broker 信息

    if (findBrokerResult != null) {
        // check version MQ 版本检查
        int sysFlagInner = sysFlag;
        if (findBrokerResult.isSlave()) {
            sysFlagInner = PullSysFlag.clearCommitOffsetFlag(sysFlagInner);
        }

        PullMessageRequestHeader requestHeader = new PullMessageRequestHeader();
        // set requestHeader
        // get brokerAddr
        PullResult pullResult = this.mQClientFactory.getMQClientAPIImpl().pullMessage(
            brokerAddr,
            requestHeader,
            timeoutMillis,
            communicationMode,
            pullCallback);
        return pullResult;
    }
}
```

#### MQClientAPIImpl#pullMessage

> 单向消息不需要等待响应。发送方将消息发送到 Broker 上之后，不需要等待 Broker 的响应，直接返回。因此，消费者无法通过消费队列（ConsumeQueue）的方式来消费单向消息。
>
> RocketMQ 提供了一种基于订阅的方式来消费单向消息。消费者可以订阅某个主题（Topic），当有单向消息被发送到该主题时，消费者会收到该消息。

```java
public PullResult pullMessage(
    final String addr,
    final PullMessageRequestHeader requestHeader,
    final long timeoutMillis,
    final CommunicationMode communicationMode,
    final PullCallback pullCallback
) /**...**/ {
    RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.PULL_MESSAGE, requestHeader);
    switch (communicationMode) {
        case ONEWAY:
        		// 单向消息在发送之后，接收方可以通过订阅消息的方式来获取到消息，而不是通过拉取消息的方式
            assert false;
            return null;
        case ASYNC:
            this.pullMessageAsync(addr, request, timeoutMillis, pullCallback);
            return null;
        case SYNC:
            return this.pullMessageSync(addr, request, timeoutMillis);
        default:
            assert false;
            break;
    }
    return null;
}
```

#### MQClientAPIImpl#pullMessageAsync

> 以异步消息拉取为例

```java
private void pullMessageAsync(/**...**/)  /**...**/ {
    this.remotingClient.invokeAsync(addr, request, timeoutMillis,  /**...**/);
  	/**...**/
    if (response != null) {
        try {
            PullResult pullResult = MQClientAPIImpl.this.processPullResponse(response, addr);
            assert pullResult != null;
            pullCallback.onSuccess(pullResult); // 执行成功回调
        } catch (Exception e) {
            pullCallback.onException(e);
        }
    }
  	/**...**/
}
```

#### PullCallback

```java
PullCallback pullCallback = new PullCallback() {
    @Override
    public void onSuccess(PullResult pullResult) {
        if (pullResult != null) {
            pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult,
                subscriptionData);

            switch (pullResult.getPullStatus()) {
                case FOUND:
                    long prevRequestOffset = pullRequest.getNextOffset(); // 下一个 offset
                    pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                    long pullRT = System.currentTimeMillis() - beginTimestamp; // 消息拉取 RT
                    DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullRT(pullRequest.getConsumerGroup(),
                        pullRequest.getMessageQueue().getTopic(), pullRT);

                		// firstMsgOffset 第一条消息的 offset
                    long firstMsgOffset = Long.MAX_VALUE;
                    if (pullResult.getMsgFoundList() == null || pullResult.getMsgFoundList().isEmpty()) { // 如果消息列表为 null 或 Empty
                        DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest); // 再执行一次消息拉取
                    } else {
                        firstMsgOffset = pullResult.getMsgFoundList().get(0).getQueueOffset(); // 否则获取到第一条消息 offset

                        DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullTPS(pullRequest.getConsumerGroup(),
                            pullRequest.getMessageQueue().getTopic(), pullResult.getMsgFoundList().size());

                        boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList()); // 将消息分发给对应的消费者进行消费
                        DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
                            pullResult.getMsgFoundList(),
                            processQueue,
                            pullRequest.getMessageQueue(),
                            dispatchToConsume);

                        if (DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval() > 0) { // 如果存在拉取间隔 executePullRequestLater
                            DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest,
                                DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval());
                        } else { // 否则立即拉取
                            DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                        }
                    }

                    if (pullResult.getNextBeginOffset() < prevRequestOffset
                        || firstMsgOffset < prevRequestOffset) {
                        // 如果 offset 异常
                    }

                    break;
                case NO_NEW_MSG:
                case NO_MATCHED_MSG:
                    pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                    DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);

                    DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                    break;
                case OFFSET_ILLEGAL: // 如果 offset 非法
                    // log
                    pullRequest.getProcessQueue().setDropped(true); // 放弃对当前内存中消息队列的处理
                    DefaultMQPushConsumerImpl.this.executeTaskLater(new Runnable() {

                        @Override
                        public void run() {
                            try {
                                DefaultMQPushConsumerImpl.this.offsetStore.updateOffset(pullRequest.getMessageQueue(),
                                    pullRequest.getNextOffset(), false); // 更新本地 offset

                                DefaultMQPushConsumerImpl.this.offsetStore.persist(pullRequest.getMessageQueue()); // Persist the offset,may be in local storage or remote name server

                                DefaultMQPushConsumerImpl.this.rebalanceImpl.removeProcessQueue(pullRequest.getMessageQueue()); // 从内存中移除正在处理的消息队列

                                log.warn("fix the pull request offset, {}", pullRequest);
                            }  /**...**/
                        }
                    }, 10000);
                    break;
                default:
                    break;
            }
        }
    }
  	// 消息拉取异常则 executePullRequestLater
    @Override
    public void onException(Throwable e) {/**...**/}
};
```

<br>

### PullConsumer 消息拉取

> Pull 模式的消费者和 Push 模式的消费者不一样，需要自定义实现消息拉取和消费逻辑。可以模仿 Push 模式的拉取逻辑来实现。

```java
public static void main(String[] args) {
    DefaultMQPullConsumer consumer = new DefaultMQPullConsumer(SimpleMQConstant.CONSUMER_GROUP);
    consumer.setNamesrvAddr(SimpleMQConstant.NAME_SERVER);

    try {
        consumer.start();
        // 手动获取 topic 下的消息队列
        Collection<MessageQueue> queues = consumer.fetchSubscribeMessageQueues(SimpleMQConstant.TOPIC_DEFAULT);

        // 遍历消息队列
        for (MessageQueue queue : queues) {
            while (true) {
                // 拉取消息
                // pullBlockIfNotFound(消息队列, tag, offset, 最大拉取消息数量)
                PullResult pullResult =
                        consumer.pullBlockIfNotFound(queue, "*", getQueueOffset(queue), 32);

                // 更新消费 offset
                updateQueueOffset(queue, pullResult.getNextBeginOffset());

                switch (pullResult.getPullStatus()) {
                    case FOUND: // 找到消息，输出
                        MessageExt messageExt = pullResult.getMsgFoundList().get(0);

                        System.out.println("收到消息：" + new String(messageExt.getBody()));
                        break;
                    case NO_MATCHED_MSG: // 没有匹配 tag 的消息
                        System.out.println("没有匹配 tag 的消息");
                        break;
                    case NO_NEW_MSG: // 该队列没有新消息，消费offset = 最大offset
                        System.out.println(queue + " 队列没有新消息");
                        break;
                    case OFFSET_ILLEGAL: // 非法 offset
                        System.out.println("非法 offset");
                        break;
                    default:
                        break;
                }
            }
        }
    } /**...**/
}
```