# 综述

1. 版本使用release-4.7.0
2. [官网地址](http://rocketmq.apache.org/)
3. [GitHub地址](https://github.com/apache/rocketmq/)



## 结构图

![Architecture](截图/RocketMQ/Architecture.png)

先看一下官方给出的结构图，图中分为了4个大的模块，producer、namesrv、broker、consumer，这4个模块都是可以集群部署。



# NameServer

引入：

1. 为什么要单独写一个注册中心，为什么不采用网上现有的解决方案，例如zk，eureka
2. nameServer有什么优点？架构是怎么样的？与zk，eureka差异。
3. 启动流程
4. client的注册流程，nameServer故障发现，故障排除
5. 集群的实现方式，各节点间关系，怎样同步各个节点间数据
6. nameServer中储存了一些什么信息，储存结构是怎么样的，生命周期如何维护
7. nameServer提供了broker discovery功能，存储了routing info信息，相关功能是怎么用的





# Producer

引入

- 启动流程
- 消息发送流程
- 消息重发
- 事务消息



## 启动流程

在官方源码`org.apache.rocketmq.example.simple`包中都是一些使用例子，其中`Producer`类是生产者的Test类。

```java
 				DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName");
        producer.start();

        for (int i = 0; i < 128; i++)
            try {
                {
                    Message msg = new Message("TopicTest",
                        "TagA",
                        "OrderID188",
                        "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));
                    SendResult sendResult = producer.send(msg);
                    System.out.printf("%s%n", sendResult);
                }

            } catch (Exception e) {
                e.printStackTrace();
            }

        producer.shutdown();
```



看一下`DefaultMQProducer`类的构造方法

```java
org.apache.rocketmq.client.producer.DefaultMQProducer
	
    1.调用单参构造，没有给默认的命名空间和钩子函数
  	public DefaultMQProducer(final String producerGroup) {
        this(null, producerGroup, null);
    }

		2.继续实例化defaultMQProducerImpl
    public DefaultMQProducer(final String namespace, final String producerGroup, RPCHook rpcHook) {
        this.namespace = namespace;
        this.producerGroup = producerGroup;
        defaultMQProducerImpl = new DefaultMQProducerImpl(this, rpcHook);
    }

    3. 初始化了asyncSenderThreadPoolQueue 异步的发送线程池队列
       初始化了一个发送消息用的线程池
      
    public DefaultMQProducerImpl(final DefaultMQProducer defaultMQProducer, RPCHook rpcHook) {
        this.defaultMQProducer = defaultMQProducer;
        this.rpcHook = rpcHook;

        this.asyncSenderThreadPoolQueue = new LinkedBlockingQueue<Runnable>(50000);
        this.defaultAsyncSenderExecutor = new ThreadPoolExecutor(
            Runtime.getRuntime().availableProcessors(),
            Runtime.getRuntime().availableProcessors(),
            1000 * 60,
            TimeUnit.MILLISECONDS,
            this.asyncSenderThreadPoolQueue,
            new ThreadFactory() {
                private AtomicInteger threadIndex = new AtomicInteger(0);

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, "AsyncSenderExecutor_" + this.threadIndex.incrementAndGet());
                }
            });
    }

```

对一些变量进行了初始化操作，初始化了一个发送消息用的线程池。`DefaultMQProducer`实例化完成后，调用了`start`方法

```java
org.apache.rocketmq.client.producer.DefaultMQProducer

@Override
public void start() throws MQClientException {
    this.setProducerGroup(withNamespace(this.producerGroup));
    this.defaultMQProducerImpl.start();
    if (null != traceDispatcher) {
        try {
            traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
        } catch (MQClientException e) {
            log.warn("trace dispatcher start failed ", e);
        }
    }
}
```

1. 生成了producerGroup名

2. 调用`start`方法

3. 启动跟踪器

   

继续看`start`方法

```java
org.apache.rocketmq.client.producer.DefaultMQProducer

public void start() throws MQClientException {
    this.start(true);
}

public void start(final boolean startFactory) throws MQClientException {
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;

								//使用Validators类中静态方法检查配置项
                this.checkConfig();

                if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                    this.defaultMQProducer.changeInstanceNameToPID();
                }

								//①通过MQClientManager静态对象，创建或获取一个MQClientInstance(mQClientFactory对象名有点误导)
                this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQProducer, rpcHook);

								//注册一下此producer，其实就是把此producer对象加入到mQClientFactory中一个table缓存中
                boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
                if (!registerOK) {
                    this.serviceState = ServiceState.CREATE_JUST;
                    throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()
                        + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                        null);
                }

								//缓存一下需要发送的目标topic
                this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());

								//②第一次，需要先启动MQClientInstance
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
				
				//发送心跳给所有的broker
        this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();

				//创建一个定时任务，每3秒扫描一下过期的请求
        this.timer.scheduleAtFixedRate(new TimerTask() {
            @Override
            public void run() {
                try {
                    RequestFutureTable.scanExpiredRequest();
                } catch (Throwable e) {
                    log.error("scan RequestFutureTable exception", e);
                }
            }
        }, 1000 * 3, 1000);
    }
```

1. 创建或获取一个MQClientInstance。
2. 在MQClientInstance中注册一下producer
3. 启动MQClientInstance
4. 维护心跳给所有的broker
5. 创建一个定时任务，定时扫描过期的请求















#  Broker

引入

- broker物理、逻辑结构，broker与queue的关系
- 启动流程
- 消息的存储结构与发送消息的接受流程
- 请求路由、负载均衡、动态扩容
- 消息的持久化机制
- 集群结构，节点间关系、节点间高可用



# Consumer

引入

- 启动流程
- 消息拉取过程
  - 主动拉取
  - 被动拉取
- 消费的具体流程
- 消费队列的负载均衡与rebalance
- 定时消费的具体实现
- 如果需要有严格消费顺序，该怎么做？