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
- 服务注册：map数据结构
- 全量拉取注册表：多级缓存机制
- 增量拉取注册表：一致性hash比对机制
- 心跳机制：服务续约，renew
- 服务下线：cancel



## 类介绍

//todo 

com.netflix.eureka.lease.Lease

com.netflix.appinfo.LeaseInfo

com.netflix.appinfo.InstanceInfo

com.netflix.discovery.shared.Applications



`com.netflix.eureka.EurekaServerConfig`

他是一个接口，定义了一些值得get方法。一般使用EurekaServerConfigBean作为具体实现类，用来读取配置文件中eureka.server开头的配置





# 服务端

## 启动流程

![EurekaServer源码流程梳理图](截图/Spring/Eureka/EurekaServer源码流程梳理图.png)

流程图引用自上方链接文章，从整体上看，启动流程是十分清晰的。依照springboot惯例，找到`spring-cloud-netflix-eureka-server-2.2.3.RELEASE.jar`包中的`spring.factories`文件，找到server启动流程的入口类`org.springframework.cloud.netflix.eureka.server.EurekaServerAutoConfiguration`。观察到类的上方通过`@Import(EurekaServerInitializerConfiguration.class)`标签注入了一个类，这个配置类实现了`ServletContextAware, SmartLifecycle`两个接口，所以主要看一下start方法中的内容。

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

## 启动流程

