# RocketMQ 源码解析

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

5、发送心跳到 Broker

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
    if (null != traceDispatcher) {
        try {
            traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
        } catch (MQClientException e) {
            log.warn("trace dispatcher start failed ", e);
        }
    }
}
```

**DefaultMQPushConsumerImpl#start**

1、启动消费者，启动过程中如果 ServiceState 为 RUNNING/START_FAILED/SHUTDOWN_ALREADY 则启动失败；如果 ServiceState 为 CREATE_JUST 则继续

2、检查消费者配置，复制订阅信息、获取消费者客户端实例 MQClientInstance

3、获取消费 Offset，如果是 BROADCASTING 模式从本地获取，如果是 CLUSTERING 模式则从远程（MQ）获取

4、设置并启动 ConsumeMessageService（MessageListenerOrderly or MessageListenerConcurrently）

5、在 MQClientInstance 客户端实例中注册当前消费者，并启动 MQClientInstance。MQClientInstance#start 还会启动 MQClientAPIImpl，PullMessageService，RebalanceService 等服务

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
                throw new MQClientException("The consumer group[" + this.defaultMQPushConsumer.getConsumerGroup()
                    + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                    null);
            }

            mQClientFactory.start();
            log.info("the consumer [{}] start OK.", this.defaultMQPushConsumer.getConsumerGroup());
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

> 总体来说，同步/异步单向消息的发送逻辑大体相同，最后都是借助 Netty（本质是 nio） 的 ChannelOutboundInvoker#writeAndFlush 进行非阻塞发送。在各自的 invokeXXXImpl 方法中，只有单向和异步需要使用到信号量来控制发送线程。因为单向消息不关心发送结果，无所谓发送成功与否；而异步消息初衷就是异步发送，自然需要主线程之外的其他线程来执行发送操作。



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

