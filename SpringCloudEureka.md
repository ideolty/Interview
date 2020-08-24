# 综述

本文大量引用[Eureka服务端源码流程梳理](https://www.cnblogs.com/nijunyang/p/10745730.html)此文章，特此感谢。



结合springcloud框架一起看更加贴合实际情况，使用的版本为

```pom
    <properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Hoxton.SR6</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
    </dependencies>
```



以下几个方面基本覆盖了eureka大部分功能，足以应付日常工作与面试了。

**Server**

- eureka server启动：注册中心
- eureka server集群：注册表的同步，多级队列的任务批处理机制

- 服务故障：expiration，eviction
- 自我保护：自动识别eureka server出现网络故障了



**Client**

- eureka client启动：服务实例
- 服务注册：系统启动时，状态改变监听器触发。定时任务，状态改变触发
- 全量拉取注册表：多级缓存机制
- 增量拉取注册表：一致性hash比对机制
- 心跳机制：服务续约，renew
- 服务下线：cancel



## 类介绍

`com.netflix.appinfo.InstanceInfo`

实例信息，每一个eureka对应一个InstanceInfo。

> 在Guice下注入该实例时由`EurekaConfigBasedInstanceInfoProvider`负责创建；
>
> 但是在`Spring Cloud`下该实例由自己提供的`InstanceInfoFactory`完成创建的。



`com.netflix.appinfo.LeaseInfo`

租约信息，当client向server注册后会生成一个租约，成员变量如下

```java
    public static final int DEFAULT_LEASE_RENEWAL_INTERVAL = 30;
    public static final int DEFAULT_LEASE_DURATION = 90;

    // Client settings
		// 续租间隔时间（多长时间续约一次），默认是30s
    private int renewalIntervalInSecs = DEFAULT_LEASE_RENEWAL_INTERVAL;
		// 续约持续时间（过期时间），默认是90s
    private int durationInSecs = DEFAULT_LEASE_DURATION;

    // Server populated
		// 租约的注册时间
    private long registrationTimestamp;
		// 最近一次的续约时间
    private long lastRenewalTimestamp;
		// 下线时间
    private long evictionTimestamp;
		// 上线时间
    private long serviceUpTimestamp;
```





`com.netflix.eureka.lease.Lease`

一个包装类，`Lease<InstanceInfo>`用来简单记录一个实例的租约信息

```java
    enum Action {
        Register, Cancel, Renew
    };

    public static final int DEFAULT_DURATION_IN_SECS = 90;

    private T holder;
    private long evictionTimestamp;
    private long registrationTimestamp;
    private long serviceUpTimestamp;
    // Make it volatile so that the expiration task would see this quicker
    private volatile long lastUpdateTimestamp;
    private long duration;
```





`com.netflix.discovery.shared.Application`

应用信息。在微服务下，同一个应用会有很多的实例，用来保证服务的高可用。换着花样存储了3份`instance`信息

```java
    @XStreamOmitField
    private volatile boolean isDirty = false;

    @XStreamImplicit
    private final Set<InstanceInfo> instances;

    private final AtomicReference<List<InstanceInfo>> shuffledInstances;

    private final Map<String, InstanceInfo> instancesMap;
```





`com.netflix.discovery.shared.Applications`

应用集合。包装了从`eureka server`返回的应用信息


> The class that wraps all the registry information returned by eureka server.

```java
public class Applications {		
		private static class VipIndexSupport {
        final AbstractQueue<InstanceInfo> instances = new ConcurrentLinkedQueue<>();
        final AtomicLong roundRobinIndex = new AtomicLong(0);
        final AtomicReference<List<InstanceInfo>> vipList = new AtomicReference<List<InstanceInfo>>(Collections.emptyList());

        public AtomicLong getRoundRobinIndex() {
            return roundRobinIndex;
        }

        public AtomicReference<List<InstanceInfo>> getVipList() {
            return vipList;
        }
    }

    private static final String STATUS_DELIMITER = "_";

    private String appsHashCode;
    private Long versionDelta;
    @XStreamImplicit
    private final AbstractQueue<Application> applications;
    private final Map<String, Application> appNameApplicationMap;
    private final Map<String, VipIndexSupport> virtualHostNameAppMap;
    private final Map<String, VipIndexSupport> secureVirtualHostNameAppMap;
  	……
}
```






`com.netflix.eureka.EurekaServerConfig`

他是一个接口，定义了一些值得get方法。一般使用EurekaServerConfigBean作为具体实现类，用来读取配置文件中eureka.server开头的配置





`region`、`zone` 

这是两个逻辑概念，可以把`region`简单的理解为地区，而`zone`理解为一个机房。一个`region`可以有多个`zone`，一个`zone`内有多个`eureka server`

[eureka分区的深入讲解](https://www.cnblogs.com/itplay/p/9973977.html)



# 服务端

## 启动流程

![EurekaServer源码流程梳理图](截图/Spring/Eureka/EurekaServer源码流程梳理图.png)

流程图引用自上方链接文章，从整体上看，启动流程是十分清晰的。

一切的起点是从主类上的标签`@EnableEurekaServer`开始，但是spring cloud中基本上都是一样的逻辑，所以依照惯例，找到`spring-cloud-netflix-eureka-server-2.2.3.RELEASE.jar`包中的`spring.factories`文件，找到server启动流程的入口类`org.springframework.cloud.netflix.eureka.server.EurekaServerAutoConfiguration`。观察到类的上方通过`@Import(EurekaServerInitializerConfiguration.class)`标签注入了一个类，这个配置类实现了`ServletContextAware, SmartLifecycle`两个接口，所以主要看一下start方法中的内容。

初始化主要的内容是在`EurekaServerInitializerConfiguration.class`中。`EurekaServerAutoConfiguration`类中定义了大量的bean，加载了各种地方的参数到上下文中。



`org.springframework.cloud.netflix.eureka.server.EurekaServerInitializerConfiguration#start`

```java
  @Override
	public void start() {
		new Thread(() -> {
			try {
				// TODO: is this class even needed now?
				// eureka 上下文初始化
				eurekaServerBootstrap.contextInitialized(
						EurekaServerInitializerConfiguration.this.servletContext);
				log.info("Started Eureka Server");

				//发布Eureka注册成功事件
				publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
				EurekaServerInitializerConfiguration.this.running = true;
				//发布Eureka启动成功事件
				publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
			}
			catch (Exception ex) {
				// Help!
				log.error("Could not initialize Eureka servlet context", ex);
			}
		}).start();
	}
```



启动流程主要完成2件事，①把配置文件中的配置读到相应的数据结构中，②初始化上下文

`org.springframework.cloud.netflix.eureka.server.EurekaServerBootstrap#contextInitialized`

```java
	public void contextInitialized(ServletContext context) {
		try {
			//初始化配置参数 单纯就是一些赋值操作
			initEurekaEnvironment();
			//初始化上下文
			initEurekaServerContext();

			context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
		}
		catch (Throwable e) {
			log.error("Cannot bootstrap eureka server :", e);
			throw new RuntimeException("Cannot bootstrap eureka server :", e);
		}
	}
```



初始化上下文主要也是两件事①从相邻节点中读取客户端的注册信息，②剔除失效的客户端

`org.springframework.cloud.netflix.eureka.server.EurekaServerBootstrap#initEurekaServerContext`

```java
protected void initEurekaServerContext() throws Exception {
		// For backward compatibility
		// 注册json格式的转换器
		JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
				XStream.PRIORITY_VERY_HIGH);
		// 注册xml格式的转换器
		XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
				XStream.PRIORITY_VERY_HIGH);
		//判断是否是亚马逊云
		if (isAws(this.applicationInfoManager.getInfo())) {
			this.awsBinder = new AwsBinderDelegate(this.eurekaServerConfig,
					this.eurekaClientConfig, this.registry, this.applicationInfoManager);
			this.awsBinder.start();
		}

		//创建了一个EurekaServerContextHolder对象用来保存serverContext上下文
		//以提供给非spring容器内部的类使用 
		EurekaServerContextHolder.initialize(this.serverContext);

		log.info("Initialized server context");

		// Copy registry from neighboring eureka node
		// 从相邻的节点中同步注册信息
		int registryCount = this.registry.syncUp();
		// 服务状态检查，剔除失效的注册信息
		this.registry.openForTraffic(this.applicationInfoManager, registryCount);

		// Register all monitoring statistics.
		EurekaMonitors.registerAllStats();
	}
```



好奇他是怎么读取的

`com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#syncUp`

```java
    @Override
    public int syncUp() {
        // Copy entire entry from neighboring DS node
        int count = 0;

        // 可以多次重试
        for (int i = 0; ((i < serverConfig.getRegistrySyncRetries()) && (count == 0)); i++) {
            if (i > 0) {
                try {
                    Thread.sleep(serverConfig.getRegistrySyncRetryWaitMs());
                } catch (InterruptedException e) {
                    logger.warn("Interrupted during registry transfer..");
                    break;
                }
            }
          
            //从eureka客户端获取到所有的服务器注册信息
            Applications apps = eurekaClient.getApplications();
            for (Application app : apps.getRegisteredApplications()) {
                //便利各个服务器上注册的节点
                for (InstanceInfo instance : app.getInstances()) {
                    try {
                        if (isRegisterable(instance)) {
                            //注册到自己的容器中 - 方法内容较多
                            register(instance, instance.getLeaseInfo().getDurationInSecs(), true);
                            count++;
                        }
                    } catch (Throwable t) {
                        logger.error("During DS init copy", t);
                    }
                }
            }
        }
        return count;
    }
```



`register(instance, instance.getLeaseInfo().getDurationInSecs(), true);`是一个注册方法，其实顶重要的，与后面客户端注册方法一起说。

再来看一下他是如何失效服务的

`com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#openForTraffic`

```java
    @Override
    public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count) {
        // Renewals happen every 30 seconds and for a minute it should be a factor of 2.
        // 理论上应该要renew的客户端个数
        this.expectedNumberOfClientsSendingRenews = count;
        // 计算每分钟要renew的阈值
        updateRenewsPerMinThreshold();
        logger.info("Got {} instances from neighboring DS node", count);
        logger.info("Renew threshold is: {}", numberOfRenewsPerMinThreshold);
        this.startupTime = System.currentTimeMillis();
        if (count > 0) {
            this.peerInstancesTransferEmptyOnStartup = false;
        }
      
        // 获取数据中心的名字 检查是否是亚马逊的
        DataCenterInfo.Name selfName = applicationInfoManager.getInfo().getDataCenterInfo().getName();
        boolean isAws = Name.Amazon == selfName;
        if (isAws && serverConfig.shouldPrimeAwsReplicaConnections()) {
            logger.info("Priming AWS connections for all replicas..");
            primeAwsReplicas(applicationInfoManager);
        }
        logger.info("Changing status to UP");
        // 把自身状态标记为up
        applicationInfoManager.setInstanceStatus(InstanceStatus.UP);
        // 创建定时任务
        super.postInit();
    }


    protected void postInit() {
        renewsLastMin.start();
        //如果已经存在了任务，则取消他
        if (evictionTaskRef.get() != null) {
            evictionTaskRef.get().cancel();
        }
      
        //创建一个新的定时任务 定时任务具体的逻辑在EvictionTask类中
        evictionTaskRef.set(new EvictionTask());
        evictionTimer.schedule(evictionTaskRef.get(),
                serverConfig.getEvictionIntervalTimerInMs(),
                serverConfig.getEvictionIntervalTimerInMs());
    }
```



## 总结

1. 在启动的过程中使用@import标签引入了一个主要干活的类`EurekaServerInitializerConfiguration.class`

2. 类中首先从各种地方读取参数，配置文件，环境变量等等。

3. 从集群相邻节点中拉取客户端信息，注册到本地。

4. 修改eureka状态为up。创建定时任务，用于清理 60秒没有心跳的客户端，自动下线。同时默认每30秒发送心跳， 

   



## 参考

[Eureka服务端源码流程梳理](https://www.cnblogs.com/nijunyang/p/10745730.html)

[Spring Cloud Eureka 全解 （1） - 总览篇](https://my.oschina.net/u/3747772/blog/3099629)

[Spring Cloud Eureka源代码解析（1）Eureka启动，原生启动与SpringCloudEureka启动异同](https://my.oschina.net/u/3747772/blog/1588933)(学习如何写胶水代码，也就是autoConfig类，把servlet项目合并到springcloud中)



# 客户端

本部分还大量参考了[【SpringCloud Eureka源码】从Eureka Client发起注册请求到Eureka Server处理的整个服务注册过程（上）](https://www.jianshu.com/p/d4cfd73e47e5)，特此感谢。

## 启动流程

![eurekaClientBootProcess](截图/Spring/Eureka/eurekaClientBootProcess.jpg)

仍然是依照惯例，找到`spring.factories`文件，地址为`spring-cloud-netflix-eureka-client-2.2.3.RELEASE.jar!/META-INF/spring.factories`。

```factory
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaClientConfigServerAutoConfiguration,\
org.springframework.cloud.netflix.eureka.config.DiscoveryClientOptionalArgsConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration,\
org.springframework.cloud.netflix.ribbon.eureka.RibbonEurekaAutoConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration,\
org.springframework.cloud.netflix.eureka.reactive.EurekaReactiveDiscoveryClientConfiguration,\
org.springframework.cloud.netflix.eureka.loadbalancer.LoadBalancerEurekaAutoConfiguration

org.springframework.cloud.bootstrap.BootstrapConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaConfigServerBootstrapConfiguration
```

重点在`EurekaClientAutoConfiguration`类，其他的基本都是创建一些bean，加载一些配置。



###  EurekaClientAutoConfiguration

类中初始化了一堆bean

1. 初始化了`EurekaClientConfigBean`，这是个对netflix的`EurekaClientConfig`客户端配置接口的实现，用来加载配置文件中`eureka.client`为前缀的配置项。
2. 初始化了`EurekaInstanceConfigBean`，这是个对netflix的`EurekaInstanceConfig`实例信息配置接口的实现，从spring上下文中加载了大量的配置。
3. 初始化了一些AutoServiceRegistration，即客户端自动注册的组件，如

- `EurekaRegistration`： Eureka实例的服务注册信息（在开启客户端自动注册时才会注册）
- `EurekaServiceRegistry`： Eureka服务注册器
- `EurekaAutoServiceRegistration`： Eureka服务自动注册器，实现了SmartLifecycle，会在Spring容器的refresh的最后阶段被调用，通过`EurekaServiceRegistry`注册器注册`EurekaRegistration`信息
- 两个内部的Configuration配置类，按照是否可以刷新分类，两个类初始化二选一。但是其实创建的内容完全相同，只是多了一个刷新配置的能力。
  - **创建了`EurekaClient`对象，这是我们主要分析的对象，他是负责与`EurekaServer`交互的客户端**。
  - 创建了`ApplicationInfoManager`对象，用来初始化一些向Eureka Server注册时需要的信息
  - 创建了`EurekaRegistration`对象，Eureka专用的注册器，一个包装类，内部有`EurekaClient`、`ApplicationInfoManager`、`CloudEurekaInstanceConfig`对象，通过他可以拿到当前实例的相关信息。

4. 初始化了`EurekaHealthIndicator`，为/health端点提供Eureka相关信息，主要有Status当前实例状态和applications服务列表。



接下来主要看客户端`EurekaClient`对象的初始化流程

![CloudEurekaClient](截图/Spring/Eureka/CloudEurekaClient.png)



`CloudEurekaClient`是springcloud进行封装的，他继承自Netflix原生的`com.netflix.discovery.DiscoveryClient`。`EurekaClient`是Netflix对服务发现客户端抽象的接口。`EurekaClient`接口定义了以下几个功能

- 向`Eureka Server`注册实例
- 向`Eureka Server`刷新租约
- 向`Eureka Server`取消注册

根据`CloudEurekaClient`类上注释，其主要重写了`onCacheRefreshed()方法`，这个方法主要是从Eureka Server `fetchRegistry()`获取服务列表之后，用以广播方式通知缓存刷新事件的，其实`DiscoveryClient`也有`onCacheRefreshed()方法`的实现，但由于`DiscoveryClient`是Netflix的类，只发送了com.netflix.discovery.EurekaEvent，而`CloudEurekaClient`使用Spring的`ApplicationEventPublisher`，发送了HeartbeatEvent。



`org.springframework.cloud.netflix.eureka.CloudEurekaClient#CloudEurekaClient`

```java
	public CloudEurekaClient(ApplicationInfoManager applicationInfoManager,
			EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs<?> args,
			ApplicationEventPublisher publisher) {
    //调用父类的构造方法，逻辑全在父类
		super(applicationInfoManager, config, args);
		this.applicationInfoManager = applicationInfoManager;
		this.publisher = publisher;
		this.eurekaTransportField = ReflectionUtils.findField(DiscoveryClient.class,
				"eurekaTransport");
		ReflectionUtils.makeAccessible(this.eurekaTransportField);
	}
```



`com.netflix.discovery.DiscoveryClient#DiscoveryClient`

主要的逻辑全在类的初始化函数内了

```java
    @Inject
    DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                    Provider<BackupRegistry> backupRegistryProvider, EndpointRandomizer endpointRandomizer) {
        ……
          
        //如果要拉取服务列表则定义拉取频率
        if (config.shouldFetchRegistry()) {
            this.registryStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRY_PREFIX + "lastUpdateSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
        } else {
            this.registryStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
        }

      	//如果要向eureka server注册则定义一下心跳频率
        if (config.shouldRegisterWithEureka()) {
            this.heartbeatStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRATION_PREFIX + "lastHeartbeatSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
        } else {
            this.heartbeatStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
        }

        logger.info("Initializing Eureka in region {}", clientConfig.getRegion());

        ……
        
        //定义了一个只有2个线程的线程池，一个线程用来发心跳，一个用来刷新缓存
        try {
            // default size of 2 - 1 each for heartbeat and cacheRefresh
            scheduler = Executors.newScheduledThreadPool(2,
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-%d")
                            .setDaemon(true)
                            .build());

          	//心跳执行器
            heartbeatExecutor = new ThreadPoolExecutor(
                    1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                    new SynchronousQueue<Runnable>(),
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                            .setDaemon(true)
                            .build()
            );  // use direct handoff

          	//刷新缓存的执行器
            cacheRefreshExecutor = new ThreadPoolExecutor(
                    1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                    new SynchronousQueue<Runnable>(),
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
                            .setDaemon(true)
                            .build()
            );  // use direct handoff

            ……
        } catch (Throwable e) {
            throw new RuntimeException("Failed to initialize DiscoveryClient!", e);
        }

      	……
          
        //如果符合条件则进行注册 第二个条件默认值是false 
        if (clientConfig.shouldRegisterWithEureka() && clientConfig.shouldEnforceRegistrationAtInit()) {
            try {
              	//向eureka server注册
                if (!register() ) {
                    throw new IllegalStateException("Registration error at startup. Invalid server response.");
                }
            } catch (Throwable th) {
                logger.error("Registration error at startup: {}", th.getMessage());
                throw new IllegalStateException(th);
            }
        }

        // finally, init the schedule tasks (e.g. cluster resolvers, heartbeat, instanceInfo replicator, fetch
      	// 初始化定时任务
        initScheduledTasks();

        ……
    }

```



要记住的主要工作有2个：

1. 定义了发送心跳的定时任务，定义了刷新本地缓存的定时任务
2. 向eureka server注册



### initScheduledTasks

接下来主要看看`initScheduledTasks();`方法

`com.netflix.discovery.DiscoveryClient#initScheduledTasks`

```java
    private void initScheduledTasks() {
      	//启动cacheRefreshTask 从Eureka Server获取服务列表
        if (clientConfig.shouldFetchRegistry()) {
            // registry cache refresh timer
          	// 从eureka服务器获取注册表信息的频率（默认30s） 同时也是单次获取服务列表的超时时间
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
          	// 一个倍数值（默认10），用来计算最大的延迟时间 
          	// this.maxDelay = registryFetchIntervalSeconds * expBackOffBound; 
            // 当前的延迟为registryFetchIntervalSeconds，每次执行延迟变为上一次的2倍
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            cacheRefreshTask = new TimedSupervisorTask(
                    "cacheRefresh",
                    scheduler,
                    cacheRefreshExecutor,
                    registryFetchIntervalSeconds,
                    TimeUnit.SECONDS,
                    expBackOffBound,
              			// 主要的刷新逻辑类
                    new CacheRefreshThread()
            );
            scheduler.schedule(
                    cacheRefreshTask,
                    registryFetchIntervalSeconds, TimeUnit.SECONDS);
        }

      	// 启动心跳任务
        if (clientConfig.shouldRegisterWithEureka()) {
          	// 续租的时间间隔（默认30s）
            int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
          	// 如果任务超时，下次延迟的倍数
            int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
            logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

            // Heartbeat timer
            heartbeatTask = new TimedSupervisorTask(
                    "heartbeat",
                    scheduler,
                    heartbeatExecutor,
                    renewalIntervalInSecs,
                    TimeUnit.SECONDS,
                    expBackOffBound,
              			// 心跳具体逻辑类
                    new HeartbeatThread()
            );
            scheduler.schedule(
                    heartbeatTask,
                    renewalIntervalInSecs, TimeUnit.SECONDS);

            // InstanceInfo replicator
          	/** 
             * InstanceInfo复制器 启动后台定时任务scheduler
             * 默认每30s执行一次定时任务，查看Instance信息（DataCenterInfo、LeaseInfo、InstanceStatus）是否有变化
             * 如果有变化，执行 discoveryClient.register()
             */
            instanceInfoReplicator = new InstanceInfoReplicator(
                    this,
                    instanceInfo,
                    clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                    2); // burstSize

          	// 状态改变的监听器
          	// 例如启动或下线之后
            statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
                @Override
                public String getId() {
                    return "statusChangeListener";
                }

                @Override
                public void notify(StatusChangeEvent statusChangeEvent) {
                    if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
                            InstanceStatus.DOWN == statusChangeEvent.getPreviousStatus()) {
                        // log at warn level if DOWN was involved
                        logger.warn("Saw local status change event {}", statusChangeEvent);
                    } else {
                        logger.info("Saw local status change event {}", statusChangeEvent);
                    }
                  	// 状态变化
                    instanceInfoReplicator.onDemandUpdate();
                }
            };

          	//默认开启
            if (clientConfig.shouldOnDemandUpdateStatusChange()) {
                applicationInfoManager.registerStatusChangeListener(statusChangeListener);
            }

          	//启动InstanceInfo复制器 周期性的把自身信息上报
            instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
        } else {
            logger.info("Not registering with Eureka server per configuration");
        }
    }
```



到此为止，client端的启动流程主体就完成了，一般的面试也就问到这种程度就过了，但是架不住就是有一些面试官喜欢一条路问到底的。所以针对3个主要逻辑，看看一下他的具体实现。



#### cacheRefreshTask

服务器信息拉取任务 `cacheRefreshTask`

`com.netflix.discovery.DiscoveryClient.CacheRefreshThread`

```java
    class CacheRefreshThread implements Runnable {
        public void run() {
            refreshRegistry();
        }
    }

    @VisibleForTesting
    void refreshRegistry() {
        try {
          	// 是否需要获取远端服务器信息
            boolean isFetchingRemoteRegionRegistries = isFetchingRemoteRegionRegistries();

          	// 远端的服务器信息是否改变标志
            boolean remoteRegionsModified = false;
            // This makes sure that a dynamic change to remote regions to fetch is honored.
            String latestRemoteRegions = clientConfig.fetchRegistryForRemoteRegions();
            if (null != latestRemoteRegions) {
                String currentRemoteRegions = remoteRegionsToFetch.get();
                if (!latestRemoteRegions.equals(currentRemoteRegions)) {
                    // Both remoteRegionsToFetch and AzToRegionMapper.regionsToFetch need to be in sync
                    synchronized (instanceRegionChecker.getAzToRegionMapper()) {
                        if (remoteRegionsToFetch.compareAndSet(currentRemoteRegions, latestRemoteRegions)) {
                            String[] remoteRegions = latestRemoteRegions.split(",");
                            remoteRegionsRef.set(remoteRegions);
                            instanceRegionChecker.getAzToRegionMapper().setRegionsToFetch(remoteRegions);
                            remoteRegionsModified = true;
                        } else {
                            logger.info("Remote regions to fetch modified concurrently," +
                                    " ignoring change from {} to {}", currentRemoteRegions, latestRemoteRegions);
                        }
                    }
                } else {
                    // Just refresh mapping to reflect any DNS/Property change
                    instanceRegionChecker.getAzToRegionMapper().refreshMapping();
                }
            }
          
          	// 拉取远端服务器信息
            boolean success = fetchRegistry(remoteRegionsModified);
            ……
        } catch (Throwable e) {
            logger.error("Cannot fetch registry from server", e);
        }
    }


		// 拉取远端服务器信息
    private boolean fetchRegistry(boolean forceFullRegistryFetch) {
        Stopwatch tracer = FETCH_REGISTRY_TIMER.start();

        try {
            // If the delta is disabled or if it is the first time, get all
            // applications
            Applications applications = getApplications();

            if (clientConfig.shouldDisableDelta()
                    || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
                    || forceFullRegistryFetch
                    || (applications == null)
                    || (applications.getRegisteredApplications().size() == 0)
                    || (applications.getVersion() == -1)) //Client application does not have latest library supporting delta
            {
                logger.info("Disable delta property : {}", clientConfig.shouldDisableDelta());
                logger.info("Single vip registry refresh property : {}", clientConfig.getRegistryRefreshSingleVipAddress());
                logger.info("Force full registry fetch : {}", forceFullRegistryFetch);
                logger.info("Application is null : {}", (applications == null));
                logger.info("Registered Applications size is zero : {}",
                        (applications.getRegisteredApplications().size() == 0));
                logger.info("Application version is -1: {}", (applications.getVersion() == -1));
              	// 全量拉取
                getAndStoreFullRegistry();
            } else {
              	// 增量更新
                getAndUpdateDelta(applications);
            }
          	// 保存本地缓存的hash值
            applications.setAppsHashCode(applications.getReconcileHashCode());
            logTotalInstances();
        } catch (Throwable e) {
            logger.error(PREFIX + "{} - was unable to refresh its cache! status = {}", appPathIdentifier, e.getMessage(), e);
            return false;
        } finally {
            if (tracer != null) {
                tracer.stop();
            }
        }

        // Notify about cache refresh before updating the instance remote status
      	// 发送一个cache刷新时间
        onCacheRefreshed();

        // Update remote status based on refreshed data held in the cache
      	// 发出一个本地instance状态变化事件
        updateInstanceRemoteStatus();

        // registry was fetched successfully, so return true
        return true;
    }
```

- 拉取远端服务器信息，又分为两种拉取方法，全量拉取与增量拉取，如果`clientConfig.shouldDisableDelta()`值为false，或者是第一次拉取，或者本地的缓存与远端的数据hash值不一致则进行全量拉取
- 发完后发两个事件，方便cloud其他组件使用



**先看一下全量拉取逻辑**

`com.netflix.discovery.DiscoveryClient#getAndStoreFullRegistry`

```java
    private void getAndStoreFullRegistry() throws Throwable {
      	// 一个单调递增的值 确保旧的线程不会设置旧的注册信息
        long currentUpdateGeneration = fetchRegistryGeneration.get();

        logger.info("Getting all instance registry info from the eureka server");

        Applications apps = null;
      	// 发请求 
        EurekaHttpResponse<Applications> httpResponse = clientConfig.getRegistryRefreshSingleVipAddress() == null
                ? eurekaTransport.queryClient.getApplications(remoteRegionsRef.get())
                : eurekaTransport.queryClient.getVip(clientConfig.getRegistryRefreshSingleVipAddress(), remoteRegionsRef.get());
        if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
            apps = httpResponse.getEntity();
        }
        logger.info("The response status is {}", httpResponse.getStatusCode());

        if (apps == null) {
            logger.error("The application is null for some reason. Not storing this information");
        } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
          	// 把远端拿到的信息缓存在本地
            localRegionApps.set(this.filterAndShuffle(apps));
            logger.debug("Got full registry with apps hashcode {}", apps.getAppsHashCode());
        } else {
            logger.warn("Not updating applications as another thread is updating it already");
        }
    }
```

- 全量拉取 调用server端的/eureka/apps接口，拿到所有注册的服务器信息，返回了一个包装类**Applications**，服务相关信息存在map对象中

  `com.netflix.discovery.shared.Applications`

  ```java
      private String appsHashCode;
      private Long versionDelta;
      @XStreamImplicit
      private final AbstractQueue<Application> applications;
      private final Map<String, Application> appNameApplicationMap;
      private final Map<String, VipIndexSupport> virtualHostNameAppMap;
      private final Map<String, VipIndexSupport> secureVirtualHostNameAppMap;
  ```

- 拿到之后缓存在本地，是一个`AtomicReference<Applications> localRegionApps`对象，有效的进行原子操作，防止多线程问题。





**再看一下增量拉取**

`com.netflix.discovery.DiscoveryClient#getAndUpdateDelta`

```java
    private void getAndUpdateDelta(Applications applications) throws Throwable {
        long currentUpdateGeneration = fetchRegistryGeneration.get();

      	// 发送请求到服务端 拉取信息
        Applications delta = null;
        EurekaHttpResponse<Applications> httpResponse = eurekaTransport.queryClient.getDelta(remoteRegionsRef.get());
        if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
            delta = httpResponse.getEntity();
        }

        if (delta == null) {
            logger.warn("The server does not allow the delta revision to be applied because it is not safe. "
                    + "Hence got the full registry.");
            getAndStoreFullRegistry();
        } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
            logger.debug("Got delta update with apps hashcode {}", delta.getAppsHashCode());
            String reconcileHashCode = "";
            if (fetchRegistryUpdateLock.tryLock()) {
                try {
                  	// 更新本地缓存
                    updateDelta(delta);
                  	// 重新计算本地缓存的hashcode
                    reconcileHashCode = getReconcileHashCode(applications);
                } finally {
                    fetchRegistryUpdateLock.unlock();
                }
            } else {
                logger.warn("Cannot acquire update lock, aborting getAndUpdateDelta");
            }
            // There is a diff in number of instances for some reason
          	// 如果两次的hashcode不一致，则重新请求server 获取全量的数据 缓存在本地
            if (!reconcileHashCode.equals(delta.getAppsHashCode()) || clientConfig.shouldLogDeltaDiff()) {
                reconcileAndLogDifference(delta, reconcileHashCode);  // this makes a remoteCall
            }
        } else {
            logger.warn("Not updating application delta as another thread is updating it already");
            logger.debug("Ignoring delta update with apps hashcode {}, as another thread is updating it already", delta.getAppsHashCode());
        }
    }
```

1. 从server端拿到相关信息后，更新本地缓存。
2. 返回的信息中已经标注了一些服务器的状态，分为新增、修改、删除3种类型，按类型更新缓存





#### heartbeatTask

心跳任务`heartbeatTask`，他是`DiscoveryClient`类的一个内部类。`run（）`方法的实际内容是写在`DiscoveryClient`中的，不知道为啥这样写。

`com.netflix.discovery.DiscoveryClient.HeartbeatThread`

```java
		private class HeartbeatThread implements Runnable {

        public void run() {
            if (renew()) {
                lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
            }
        }
    }

		// client需要往服务端发送appName id status lastDirtyTimestamp
    boolean renew() {
        EurekaHttpResponse<InstanceInfo> httpResponse;
        try {
          	// 发送心跳 
          	// 这个参数为什么要这样设计 真是参不透啊
            httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
            logger.debug(PREFIX + "{} - Heartbeat status: {}", appPathIdentifier, httpResponse.getStatusCode());
            if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {
                REREGISTER_COUNTER.increment();
                logger.info(PREFIX + "{} - Re-registering apps/{}", appPathIdentifier, instanceInfo.getAppName());
                long timestamp = instanceInfo.setIsDirtyWithTime();
                boolean success = register();
                if (success) {
                    instanceInfo.unsetIsDirty(timestamp);
                }
                return success;
            }
            return httpResponse.getStatusCode() == Status.OK.getStatusCode();
        } catch (Throwable e) {
            logger.error(PREFIX + "{} - was unable to send heartbeat!", appPathIdentifier, e);
            return false;
        }
    }
```

简单的一个定时任务，从instanceInfo中拿出相应的值，向server发送心跳。





#### instanceInfoReplicator

InstanceInfo复制器

`com.netflix.discovery.InstanceInfoReplicator`

```java
    public void run() {
        try {
          	// 更新信息，用于稍后的注册
            discoveryClient.refreshInstanceInfo();

            Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
            if (dirtyTimestamp != null) {
              	// 注册自身
                discoveryClient.register();
                instanceInfo.unsetIsDirty(dirtyTimestamp);
            }
        } catch (Throwable t) {
            logger.warn("There was a problem with the instance info replicator", t);
        } finally {
            // 每次执行完毕都会创建一个延时执行的任务，就这样实现了周期性执行的逻辑
            Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
          	// 留一个对结果的引用 方便外部对象查询结果
            scheduledPeriodicRef.set(next);
        }
    }
```

1. instanceInfoReplicator周期性的运行，不断的手动创建延时任务。思路与Netflix自己封装的`TimedSupervisorTask`类一致，那么为什么不直接用呢？
2. 当自身状态发送变化时，也会触发消息上报。
3. 上报信息是直接把自身的`instanceInfo`对象整个传给server端。





## 注销

注销方法通过标签`@PreDestroy`触发

`com.netflix.discovery.DiscoveryClient`

```java
    @PreDestroy
    @Override
    public synchronized void shutdown() {
        if (isShutdown.compareAndSet(false, true)) {
            logger.info("Shutting down DiscoveryClient ...");

            if (statusChangeListener != null && applicationInfoManager != null) {
              	// 从map中移除状态变化监听器
                applicationInfoManager.unregisterStatusChangeListener(statusChangeListener.getId());
            }

          	// 取消之前创建的3个task 与 3个线程池
            cancelScheduledTasks();

            // If APPINFO was registered
            if (applicationInfoManager != null
                    && clientConfig.shouldRegisterWithEureka()
                    && clientConfig.shouldUnregisterOnShutdown()) {
                applicationInfoManager.setInstanceStatus(InstanceStatus.DOWN);
              	// 发送下线消息给server端
                unregister();
            }

            if (eurekaTransport != null) {
                eurekaTransport.shutdown();
            }

            heartbeatStalenessMonitor.shutdown();
            registryStalenessMonitor.shutdown();

            Monitors.unregisterObject(this);

            logger.info("Completed shut down of DiscoveryClient");
        }
    }
```







## 总结

1. 从入口`EurekaClientAutoConfiguration`开始流程，创建eureka客户端。
2. 客户端中创建了3个周期运行的定时任务
   1. 默认每隔30秒从服务器定时拉取注册信息。
   2. 默认每隔30秒给服务器发送心跳信息
   3. 默认每隔30秒检查自身的信息是否变化，有变化则上报给服务器

3. 注销的时候，关闭各种线程池，取消各种监听器，向服务器发一条下线消息。







## 参考

[【SpringCloud Eureka源码】从Eureka Client发起注册请求到Eureka Server处理的整个服务注册过程（上）](https://www.jianshu.com/p/d4cfd73e47e5)

[Eureka客户端源码流程梳理](https://www.cnblogs.com/nijunyang/p/10805759.html)

[Spring Cloud Eureka 全解 （4） - 核心流程-服务与实例列表获取详解](https://zhuanlan.zhihu.com/p/34976352)