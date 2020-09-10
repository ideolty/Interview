# 综述

仍然是与spring cloud框架结合起来看，其实单独调用的流程也已经被包括在内了，使用的版本为

```pom
    <properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Hoxton.SR6</spring-cloud.version>
    </properties>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
    </dependencies>
```



在了解了基本的用法之后，还是先提出一些问题

- 在接口上方加注解之后就能调用了，这是如何实现的。

  原来类似与mybatis，在程序启动时生成代理对象，注入的实际是代理类。

  

- 与hystrix是如何适配的。

  通过`org.springframework.cloud.openfeign.HystrixTargeter`类，绑定了`fallback`方法。并且重写了后续的方法增强类，使用`HystrixInvocationHandler`类代替了原生的`FeignInvocationHandler`类。

  

- 如何与ribbon适配，实现了请求重试、动态路由与负载均衡。

  在加载`client`、准备请求客户端的时候，如果有ribbon相关的包，则请求的是ribbon的具体实现`LoadBalancerFeignClient`，此类负责实现ribbon相关的功能。

  

- openFeign是如何支撑spring原生的注解的



# 启动流程

//todo 需要一张流程图 体现hystrix 与 ribbon



根据官网的示例，流程起点在`@EnableFeignClients`标签，`@EnableFeignClients`import了一个`FeignClientsRegistrar`类，关注一下这个注册器做了些什么

`org.springframework.cloud.openfeign.FeignClientsRegistrar#registerBeanDefinitions`

```java
	@Override
	public void registerBeanDefinitions(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
    // 读取默认的配置
		registerDefaultConfiguration(metadata, registry);
    // 注册相关的请求接口
		registerFeignClients(metadata, registry);
	}
```



## registerDefaultConfiguration

首先看一下他是如何加装配置参数的

`org.springframework.cloud.openfeign.FeignClientsRegistrar#registerDefaultConfiguration`

```java
	private void registerDefaultConfiguration(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
    // 读取@EnableFeignClients标签定义的5个参数值
		Map<String, Object> defaultAttrs = metadata
				.getAnnotationAttributes(EnableFeignClients.class.getName(), true);

		if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
      // 生成一个类名
			String name;
			if (metadata.hasEnclosingClass()) {
				name = "default." + metadata.getEnclosingClassName();
			}
			else {
				name = "default." + metadata.getClassName();
			}
      // 注册配置
			registerClientConfiguration(registry, name,
					defaultAttrs.get("defaultConfiguration"));
		}
	}

	private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name,
			Object configuration) {
    // 这是spring创建bean的标准流程 
    // 创建一个FeignClientSpecification类型的bean，通过BeanDefinitionRegistry 注入spring上下文容器中
		BeanDefinitionBuilder builder = BeanDefinitionBuilder
				.genericBeanDefinition(FeignClientSpecification.class);
		builder.addConstructorArgValue(name);
		builder.addConstructorArgValue(configuration);
		registry.registerBeanDefinition(
				name + "." + FeignClientSpecification.class.getSimpleName(),
				builder.getBeanDefinition());
	}
```

1. 加装配置首先读取了`@EnableFeignClients`标签的参数。
2. 取出了标签中`defaultConfiguration`变量的值，创建了一个`FeignClientSpecification`bean，把`defaultConfiguration`的值存进了bean中，之后把这个bean注入了spring上下文容器中。
3. 这里暴露了一个扩展点，如果有需要可以通过`@EnableFeignClients`标签的`defaultConfiguration`属性夹带私货。



## registerFeignClients

再来看一下他是如何注册相关的请求接口的

`org.springframework.cloud.openfeign.FeignClientsRegistrar#registerFeignClients`

```java
	public void registerFeignClients(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
    // 与spring一脉相承 先拿到一个扫描器
		ClassPathScanningCandidateComponentProvider scanner = getScanner();
		scanner.setResourceLoader(this.resourceLoader);

    // 这个集合里面放了一些@EnableFeignClients标签参数
		Set<String> basePackages;

    // 取出@EnableFeignClients标签内附带的值
		Map<String, Object> attrs = metadata
				.getAnnotationAttributes(EnableFeignClients.class.getName());
    // 设置标签过滤器需要过滤的类型 这里是FeignClient
		AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
				FeignClient.class);
		final Class<?>[] clients = attrs == null ? null
				: (Class<?>[]) attrs.get("clients");
		if (clients == null || clients.length == 0) {
			scanner.addIncludeFilter(annotationTypeFilter);
			basePackages = getBasePackages(metadata);
		}
		else {
			final Set<String> clientClasses = new HashSet<>();
			basePackages = new HashSet<>();
			for (Class<?> clazz : clients) {
				basePackages.add(ClassUtils.getPackageName(clazz));
				clientClasses.add(clazz.getCanonicalName());
			}
			AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
				@Override
				protected boolean match(ClassMetadata metadata) {
					String cleaned = metadata.getClassName().replaceAll("\\$", ".");
					return clientClasses.contains(cleaned);
				}
			};
			scanner.addIncludeFilter(
					new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
		}

    // 如果@EnableFeignClients没有加额外参数，这里的值是@EnableFeignClients所在类的全类名
		for (String basePackage : basePackages) {
      // 扫描指定目录下是否有符合条件的类
      // 一般情况下 @EnableFeignClients 写在主类上，主类在所有类的顶级目录 这个扫描相当于扫描整个项目了
			Set<BeanDefinition> candidateComponents = scanner
					.findCandidateComponents(basePackage);
      // 变量所有有@FeignClient标签文件（类或者接口都能抓到）
			for (BeanDefinition candidateComponent : candidateComponents) {
        // 判断是不是通过注解标注的bean
				if (candidateComponent instanceof AnnotatedBeanDefinition) {
					// verify annotated class is an interface
          // 检查这个类是不是一个接口
					AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
					AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
					Assert.isTrue(annotationMetadata.isInterface(),
							"@FeignClient can only be specified on an interface");

          // 把@FeignClient标签的参数读取到map中
					Map<String, Object> attributes = annotationMetadata
							.getAnnotationAttributes(
									FeignClient.class.getCanonicalName());

          // 根据一些规则确定一个name
					String name = getClientName(attributes);
          // 与上面注册参数的方法是同一个，把这个@FeignClient标签的参数也注册成一个bean 方便后续取用
					registerClientConfiguration(registry, name,
							attributes.get("configuration"));

          // 注册feign客户端
					registerFeignClient(registry, annotationMetadata, attributes);
				}
			}
		}
	}
```

1. 使用spring标准步骤，通过scanner扫描指定目录下包含了`@FeignClient`注解的类。
2. 遍历这些类的集合，把每个接口的`@FeignClient`参数提取出来，创建一个参数bean，并注入到spring容器中。



在参数单独抽取出来之后，继续看看后续如何处理

`org.springframework.cloud.openfeign.FeignClientsRegistrar#registerFeignClient`

```java
	private void registerFeignClient(BeanDefinitionRegistry registry,
			AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
    // 创建一个bean 类型是FeignClientFactoryBean feign客户端的工厂bean
    // 然后给这个bean的成员变量赋值
		String className = annotationMetadata.getClassName();
		BeanDefinitionBuilder definition = BeanDefinitionBuilder
				.genericBeanDefinition(FeignClientFactoryBean.class);
		validate(attributes);
		definition.addPropertyValue("url", getUrl(attributes));
		definition.addPropertyValue("path", getPath(attributes));
		String name = getName(attributes);
		definition.addPropertyValue("name", name);
		String contextId = getContextId(attributes);
		definition.addPropertyValue("contextId", contextId);
		definition.addPropertyValue("type", className);
		definition.addPropertyValue("decode404", attributes.get("decode404"));
		definition.addPropertyValue("fallback", attributes.get("fallback"));
		definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
		definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

		String alias = contextId + "FeignClient";
		AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();
		beanDefinition.setAttribute(FactoryBean.OBJECT_TYPE_ATTRIBUTE, className);

		// has a default, won't be null
		boolean primary = (Boolean) attributes.get("primary");

		beanDefinition.setPrimary(primary);

		String qualifier = getQualifier(attributes);
		if (StringUtils.hasText(qualifier)) {
			alias = qualifier;
		}

    // 注入spring 容器
		BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
				new String[] { alias });
		BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
	}
```

创建一个类型是FeignClientFactoryBean的bean，并把他注入到spring上下文容器中。



## 总结

启动流程总共分为两个部分，一部分是对`@EnableFeignClients`标签的处理，一部分是对`@FeignClient`标签的处理。

1. 读取`@EnableFeignClients`标签，把标签中`defaultConfiguration`属性的值打包成一个bean，注入到spring容器中。
2. 加载系统中被`@FeignClient`标记的接口
   1. 把注解中`configuration`属性的值打包成一个bean，注入到spring容器中。
   2. 把这个注解标注的接口本身相关属性、注解相关的成员变量打包成一个工厂bean，注入到spring容器中。





# 调用流程

在springcloud环境中，一般是不会显式的使用feign对象，而是像调用本地service接口一样，把目标bean自动装配到指定的对象中。在进行自动装配时，spring通过相关的bean工厂，生成一个bean的实例并注入。但是通过上面的启动流程可知，这些接口已经创建了对应的`FeignClientFactoryBean`，由这些工厂生成出来的是由jdk动态代理动态生成的类了。

`FeignClientFactoryBean`是一个工厂类，在Spring 创建 Bean 实例时会调用它的 `getObject`方法。

`org.springframework.cloud.openfeign.FeignClientFactoryBean#getObject`

```java
	@Override
	public Object getObject() throws Exception {
		return getTarget();
	}

	<T> T getTarget() {
    // 从spring容器中拿到feign上下文容器
		FeignContext context = this.applicationContext.getBean(FeignContext.class);
    // 创建一个feign的构建器
		Feign.Builder builder = feign(context);

    // 检查是直接写的url还是通过服务名来访问
		if (!StringUtils.hasText(this.url)) {
      // 如果通过服务名访问，最后还是会被拼成http://consumer的形式，这里是写死的http
			if (!this.name.startsWith("http")) {
				this.url = "http://" + this.name;
			}
			else {
				this.url = this.name;
			}
			this.url += cleanPath();
      // 返回一个经过负载选择后对象
			return (T) loadBalance(builder, context,
					new HardCodedTarget<>(this.type, this.name, this.url));
		}
    // 如果用的是直链的方式
		if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
			this.url = "http://" + this.url;
		}
    // cleanPath是用来补偿路径中的斜杠/的
		String url = this.url + cleanPath();
    // 根据本类绑定的接口，取出相应客户端
		Client client = getOptional(context, Client.class);
		if (client != null) {
			if (client instanceof LoadBalancerFeignClient) {
				// not load balancing because we have a url,
				// but ribbon is on the classpath, so unwrap
				client = ((LoadBalancerFeignClient) client).getDelegate();
			}
			if (client instanceof FeignBlockingLoadBalancerClient) {
				// not load balancing because we have a url,
				// but Spring Cloud LoadBalancer is on the classpath, so unwrap
				client = ((FeignBlockingLoadBalancerClient) client).getDelegate();
			}
			builder.client(client);
		}
    // 获取一个触发器
		Targeter targeter = get(context, Targeter.class);
    // 调用触发器，获取一个实现了fegin接口的对象
		return (T) targeter.target(this, builder, context,
				new HardCodedTarget<>(this.type, this.name, url));
	}

	// 这里看一下第一个 return 调用的方法
	org.springframework.cloud.openfeign.FeignClientFactoryBean#loadBalance
	protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
			HardCodedTarget<T> target) {
    // 从FeignContext拿取指定的Client
		Client client = getOptional(context, Client.class);
		if (client != null) {
			builder.client(client);
      // 调用下面的方法拿 Targeter
			Targeter targeter = get(context, Targeter.class);
			return targeter.target(this, builder, context, target);
		}

		throw new IllegalStateException(
				"No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon?");
	}

	// 从FeignContext拿取指定的Targeter 两边都是调用的同一个方法
	protected <T> T get(FeignContext context, Class<T> type) {
		T instance = context.getInstance(this.contextId, type);
		if (instance == null) {
			throw new IllegalStateException(
					"No bean found of type " + type + " for " + this.contextId);
		}
		return instance;
	}
```

1. 取出`FeignContext`容器，此对象是在`org.springframework.cloud.openfeign.FeignAutoConfiguration`里自动注入了，里面保存了每个feignClient相关的配置参数。
2. 拼接请求地址，如果是url就是直接调用链接了，如果是服务名的形式调用，那么就有了负载均衡。注意这里拼接请求路径的时候是写死了http协议的，如果要求使用https则要用另外的方法了。
3. 两种请求的做法都一样，获取到一个`client`，一个`Targeter`，然后调用`Targeter.target`生成一个接口的实现类了。这里注意了，我们写的只是接口，与mybatis是一样的，没有具体的实现类，所以这里定了一个`Targeter`作为一个中间类来间接调用方法，生成接口的实现对象。
4. `Targeter`是一个接口，如果有hystrix进行熔断则他的具体实现类是`org.springframework.cloud.openfeign.HystrixTargeter`，否则调用的是默认实现。



## Targeter

先看看`DefaultTargeter`与`HystrixTargeter`的有什么区别

`org.springframework.cloud.openfeign.DefaultTargeter`

```java
class DefaultTargeter implements Targeter {

	@Override
	public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
			FeignContext context, Target.HardCodedTarget<T> target) {
		return feign.target(target);
	}

}
```

默认的不做任何额外的操作，直接调用`Feign.Builder`创建目标对象。



`org.springframework.cloud.openfeign.HystrixTargeter`

```java
	@Override
	public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
			FeignContext context, Target.HardCodedTarget<T> target) {
    // 如果类型不对也不做什么操作了
		if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
			return feign.target(target);
		}
		feign.hystrix.HystrixFeign.Builder builder = (feign.hystrix.HystrixFeign.Builder) feign;
		String name = StringUtils.isEmpty(factory.getContextId()) ? factory.getName()
				: factory.getContextId();
		SetterFactory setterFactory = getOptional(name, context, SetterFactory.class);
		if (setterFactory != null) {
			builder.setterFactory(setterFactory);
		}
    // 绑定熔断的fallback方法，也就是@FeignClient的fallback属性的值
		Class<?> fallback = factory.getFallback();
		if (fallback != void.class) {
			return targetWithFallback(name, context, target, builder, fallback);
		}
		Class<?> fallbackFactory = factory.getFallbackFactory();
		if (fallbackFactory != void.class) {
			return targetWithFallbackFactory(name, context, target, builder,
					fallbackFactory);
		}

    // 还是调用target方法
		return feign.target(target);
	}
```

`HystrixTargeter`相比默认的，多了fallback类的绑定，fallback类是实现了定义的fegn调用接口的，后续如果请求失败则可以定位调用到指定类中的指定方法了。



## Feign

继续往后看target方法

`feign.Feign.Builder#target(feign.Target<T>)`

```java
    public <T> T target(Target<T> target) {
      // 这里其实是2个步，新实例化一个Feign实例，然后调用实例的newInstance方法 创建一个代理对象
      return build().newInstance(target);
    }

		// 创建一个Feign实例对象
    public Feign build() {
      // 有点复杂Capability.enrich方法没看懂
      Client client = Capability.enrich(this.client, capabilities);
      Retryer retryer = Capability.enrich(this.retryer, capabilities);
      List<RequestInterceptor> requestInterceptors = this.requestInterceptors.stream()
          .map(ri -> Capability.enrich(ri, capabilities))
          .collect(Collectors.toList());
      Logger logger = Capability.enrich(this.logger, capabilities);
      Contract contract = Capability.enrich(this.contract, capabilities);
      Options options = Capability.enrich(this.options, capabilities);
      Encoder encoder = Capability.enrich(this.encoder, capabilities);
      Decoder decoder = Capability.enrich(this.decoder, capabilities);
      InvocationHandlerFactory invocationHandlerFactory =
          Capability.enrich(this.invocationHandlerFactory, capabilities);
      QueryMapEncoder queryMapEncoder = Capability.enrich(this.queryMapEncoder, capabilities);

      // 动态代理的方法增强类
      SynchronousMethodHandler.Factory synchronousMethodHandlerFactory =
          new SynchronousMethodHandler.Factory(client, retryer, requestInterceptors, logger,
              logLevel, decode404, closeAfterDecode, propagationPolicy, forceDecoding);
      ParseHandlersByName handlersByName =
          new ParseHandlersByName(contract, options, encoder, decoder, queryMapEncoder,
              errorDecoder, synchronousMethodHandlerFactory);
      return new ReflectiveFeign(handlersByName, invocationHandlerFactory, queryMapEncoder);
    }
```

创建一个`Feign`接口的实例，并调用`newInstance`方法创建一个代理对象。



详细看一下`ReflectiveFeign`对象

`feign.ReflectiveFeign`

```java
  public <T> T newInstance(Target<T> target) {
    //为每个方法创建一个SynchronousMethodHandler对象
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

    // 遍历原接口所有的方法，为每个方法绑定一个MethodHandler
    for (Method method : target.type().getMethods()) {
      if (method.getDeclaringClass() == Object.class) {
        continue;
      } else if (Util.isDefault(method)) {
        //如果是 default 方法，用 DefaultHandler
        DefaultMethodHandler handler = new DefaultMethodHandler(method);
        defaultMethodHandlers.add(handler);
        methodToHandler.put(method, handler);
      } else {
        //否则就用SynchronousMethodHandler
        methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
      }
    }
    // 创建动态代理handler，factory 是 InvocationHandlerFactory.Default
    // 创建出来的是 ReflectiveFeign.FeignInvocationHanlder，也就是说后续对方法的调用都会进入到该对象的 inovke 方法。
    InvocationHandler handler = factory.create(target, methodToHandler);
    // 创建动态代理对象
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
        new Class<?>[] {target.type()}, handler);

    for (DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
      defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
  }
```

1. 为接口中的每个方法创建对应的`MethodHandler`，并保持到映射集合中，这里创建的是实现类`SynchronousMethodHandler`
2. 通过`InvocationHandlerFatory`，创建`InvocationHandler`
3. 绑定接口的default方法，通过`DefaultMethodHandler`绑定，`DefaultMethodHandler`方法内部是直接调用原方法，这里一般是用不上的。因为是直接调用的接口，这些接口是没有具体的实现类的。



跟进去`factory.create`方法，确定一下`InvocationHandler`接口的具体实现是什么。

`feign.InvocationHandlerFactory`

```java
public interface InvocationHandlerFactory {

  InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch);

  /**
   * Like {@link InvocationHandler#invoke(Object, java.lang.reflect.Method, Object[])}, except for a
   * single method.
   */
  interface MethodHandler {

    Object invoke(Object[] argv) throws Throwable;
  }

  static final class Default implements InvocationHandlerFactory {

    @Override
    public InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch) {
      return new ReflectiveFeign.FeignInvocationHandler(target, dispatch);
    }
  }
}

```

`InvocationHandlerFactory`工厂内定义了一个内部类，这个类是工厂的默认实现，`create`方法通过new的方式创建了一个`FeignInvocationHandler`对象。



## FeignInvocationHandler

所以当发起请求时，会调用到InvocaHandler的invoke方法，feign里面实现类是FeignInvocationHandler，invoke代码如下：

`feign.ReflectiveFeign.FeignInvocationHandler#invoke`

```java
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      // 如果是调用的Object中的几个基础方法，则不做增强了
      if ("equals".equals(method.getName())) {
        try {
          Object otherHandler =
              args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
          return equals(otherHandler);
        } catch (IllegalArgumentException e) {
          return false;
        }
      } else if ("hashCode".equals(method.getName())) {
        return hashCode();
      } else if ("toString".equals(method.getName())) {
        return toString();
      }

      // 路由分发 找到调用方法对应的增强方法
      return dispatch.get(method).invoke(args);
    }
```

在代理类中对调用的方法进行分发，由于feign接口不会存在具体的实现类，所以不能直接调用`method.invoke`。



接下来要看看方法增强具体做了些什么事。

`feign.SynchronousMethodHandler#invoke`

```java
  public Object invoke(Object[] argv) throws Throwable {
    // 根据请求参数构建一个RequestTemplate
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    // 关于请求的一些参数 如超时时间
    Options options = findOptions(argv);
    Retryer retryer = this.retryer.clone();	
    while (true) {
      try {
        return executeAndDecode(template, options);
      } catch (RetryableException e) {
        try {
          retryer.continueOrPropagate(e);
        } catch (RetryableException th) {
          Throwable cause = th.getCause();
          if (propagationPolicy == UNWRAP && cause != null) {
            throw cause;
          } else {
            throw th;
          }
        }
        if (logLevel != Logger.Level.NONE) {
          logger.logRetry(metadata.configKey(), logLevel);
        }
        continue;
      }
    }
  }



  Object executeAndDecode(RequestTemplate template, Options options) throws Throwable {
    Request request = targetRequest(template);

    if (logLevel != Logger.Level.NONE) {
      logger.logRequest(metadata.configKey(), logLevel, request);
    }

    Response response;
    long start = System.nanoTime();
    try {
      // 发送请求
      response = client.execute(request, options);
      // ensure the request is set. TODO: remove in Feign 12
      response = response.toBuilder()
          .request(request)
          .requestTemplate(template)
          .build();
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
      }
      throw errorExecuting(request, e);
    }
    long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);

		// 解码
    if (decoder != null)
      return decoder.decode(response, metadata.returnType());

    CompletableFuture<Object> resultFuture = new CompletableFuture<>();
    asyncResponseHandler.handleResponse(resultFuture, metadata.configKey(), response,
        metadata.returnType(),
        elapsedTime);

    try {
      if (!resultFuture.isDone())
        throw new IllegalStateException("Response handling not done");

      return resultFuture.join();
    } catch (CompletionException e) {
      Throwable cause = e.getCause();
      if (cause != null)
        throw cause;
      throw e;
    }
  }
```



## 总结

在调用的时候，最终调用的是代理类的增强方法，整体思路与mybatis是一样的。

1. 首先autowire的时候是通过spring进行注入的，调用了对应的beanFactory，生成的是一个接口代理类。
2. 此代理对接口的每个方法生成了一个对应的方法增强类，保存在一个map中。
3. 当调用接口的具体方法，调用请求传递到代理类中，代理类对方法进行了一个手动路由到相应的方法增强类。
4. 在方法增强类发起http请求。



# Hystrix

在调用的时候除了原生使用，还可以与`hystrix`结合使用。

与原生的区别是在`org.springframework.cloud.openfeign.Targeter`接口的实现类上，hystrix有自己的实现



`org.springframework.cloud.openfeign.HystrixTargeter#target`

```java
	public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
			FeignContext context, Target.HardCodedTarget<T> target) {
		if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
			return feign.target(target);
		}
		feign.hystrix.HystrixFeign.Builder builder = (feign.hystrix.HystrixFeign.Builder) feign;
		String name = StringUtils.isEmpty(factory.getContextId()) ? factory.getName()
				: factory.getContextId();
		SetterFactory setterFactory = getOptional(name, context, SetterFactory.class);
		if (setterFactory != null) {
			builder.setterFactory(setterFactory);
		}
		Class<?> fallback = factory.getFallback();
		if (fallback != void.class) {
      // 如果@FeignClient标签的fallback属性不为空，则创建一个hystrix自己实现的类
			return targetWithFallback(name, context, target, builder, fallback);
		}
		Class<?> fallbackFactory = factory.getFallbackFactory();
		if (fallbackFactory != void.class) {
			return targetWithFallbackFactory(name, context, target, builder,
					fallbackFactory);
		}

    // 如果发现没有指定 fallback 对应的类 那么还是走原生
		return feign.target(target);
	}

```



看一下如何创建的代理对象，继续跟入`targetWithFallback`方法

```java
	private <T> T targetWithFallback(String feignClientName, FeignContext context,
			Target.HardCodedTarget<T> target, HystrixFeign.Builder builder,
			Class<?> fallback) {
    // 生成一个fallback对应的类的实例对象
		T fallbackInstance = getFromContext("fallback", feignClientName, context,
				fallback, target.type());
		return builder.target(target, fallbackInstance);
	}

		// builder构建器创建对象
	  public <T> T target(Target<T> target, T fallback) {
      return build(fallback != null ? new FallbackFactory.Default<T>(fallback) : null)
          .newInstance(target);
    }

		// 注意此处重写了invocationHandlerFactory类，覆盖了默认的create方法，方法增强类变成了HystrixInvocationHandler
		    /** Configures components needed for hystrix integration. */
    Feign build(final FallbackFactory<?> nullableFallbackFactory) {
      super.invocationHandlerFactory(new InvocationHandlerFactory() {
        @Override
        public InvocationHandler create(Target target,
                                        Map<Method, MethodHandler> dispatch) {
          return new HystrixInvocationHandler(target, dispatch, setterFactory,
              nullableFallbackFactory);
        }
      });
      super.contract(new HystrixDelegatingContract(contract));
      return super.build();
    }
```

1. hystrix重写了了原生的`invocationHandlerFactory`类，覆盖了默认的`create`方法，方法增强类成`HystrixInvocationHandler`。
2. 中间`target`方法还是一样的，方法路由也是相同的，只是路由调用的`dispatch.get(method).invoke(args);`是`HystrixInvocationHandler`的`invoke`了。



`feign.hystrix.HystrixInvocationHandler#invoke`

```java
  public Object invoke(final Object proxy, final Method method, final Object[] args)
      throws Throwable {
    // early exit if the invoked method is from java.lang.Object
    // code is the same as ReflectiveFeign.FeignInvocationHandler
    if ("equals".equals(method.getName())) {
      try {
        Object otherHandler =
            args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
        return equals(otherHandler);
      } catch (IllegalArgumentException e) {
        return false;
      }
    } else if ("hashCode".equals(method.getName())) {
      return hashCode();
    } else if ("toString".equals(method.getName())) {
      return toString();
    }

    // 构建hystrixCommand对象了，比较标准的hystrix请求流程了
    HystrixCommand<Object> hystrixCommand =
        new HystrixCommand<Object>(setterMethodMap.get(method)) {
          @Override
          protected Object run() throws Exception {
            try {
              // 手动路由方法的增强类
              return HystrixInvocationHandler.this.dispatch.get(method).invoke(args);
            } catch (Exception e) {
              throw e;
            } catch (Throwable t) {
              throw (Error) t;
            }
          }

          @Override
          protected Object getFallback() {
            if (fallbackFactory == null) {
              return super.getFallback();
            }
            try {
              Object fallback = fallbackFactory.create(getExecutionException());
              Object result = fallbackMethodMap.get(method).invoke(fallback, args);
              if (isReturnsHystrixCommand(method)) {
                return ((HystrixCommand) result).execute();
              } else if (isReturnsObservable(method)) {
                // Create a cold Observable
                return ((Observable) result).toBlocking().first();
              } else if (isReturnsSingle(method)) {
                // Create a cold Observable as a Single
                return ((Single) result).toObservable().toBlocking().first();
              } else if (isReturnsCompletable(method)) {
                ((Completable) result).await();
                return null;
              } else if (isReturnsCompletableFuture(method)) {
                return ((Future) result).get();
              } else {
                return result;
              }
            } catch (IllegalAccessException e) {
              // shouldn't happen as method is public due to being an interface
              throw new AssertionError(e);
            } catch (InvocationTargetException | ExecutionException e) {
              // Exceptions on fallback are tossed by Hystrix
              throw new AssertionError(e.getCause());
            } catch (InterruptedException e) {
              // Exceptions on fallback are tossed by Hystrix
              Thread.currentThread().interrupt();
              throw new AssertionError(e.getCause());
            }
          }
        };

    if (Util.isDefault(method)) {
      return hystrixCommand.execute();
    } else if (isReturnsHystrixCommand(method)) {
      return hystrixCommand;
    } else if (isReturnsObservable(method)) {
      // Create a cold Observable
      return hystrixCommand.toObservable();
    } else if (isReturnsSingle(method)) {
      // Create a cold Observable as a Single
      return hystrixCommand.toObservable().toSingle();
    } else if (isReturnsCompletable(method)) {
      return hystrixCommand.toObservable().toCompletable();
    } else if (isReturnsCompletableFuture(method)) {
      return new ObservableCompletableFuture<>(hystrixCommand);
    }
    return hystrixCommand.execute();
  }
```

1. 使用hystrix套了一层壳，用hystrix构建请求方法，这样支持了fallback相关的代理了。
2. `HystrixInvocationHandler.this.dispatch.get(method).invoke(args);`还是进行手动路由，里面取出的增强方法也还是`SynchronousMethodHandler`，没有变。



# Ribbon

ribbon体现在最初获取的client不一样。

`org.springframework.cloud.openfeign.FeignClientFactoryBean#getTarget`

```java
	<T> T getTarget() {
		FeignContext context = this.applicationContext.getBean(FeignContext.class);
		Feign.Builder builder = feign(context);

		if (!StringUtils.hasText(this.url)) {
			if (!this.name.startsWith("http")) {
				this.url = "http://" + this.name;
			}
			else {
				this.url = this.name;
			}
			this.url += cleanPath();
			return (T) loadBalance(builder, context,
					new HardCodedTarget<>(this.type, this.name, this.url));
		}
		if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
			this.url = "http://" + this.url;
		}
		String url = this.url + cleanPath();
		Client client = getOptional(context, Client.class);
		if (client != null) {
			if (client instanceof LoadBalancerFeignClient) {
				// not load balancing because we have a url,
				// but ribbon is on the classpath, so unwrap
				client = ((LoadBalancerFeignClient) client).getDelegate();
			}
			if (client instanceof FeignBlockingLoadBalancerClient) {
				// not load balancing because we have a url,
				// but Spring Cloud LoadBalancer is on the classpath, so unwrap
				client = ((FeignBlockingLoadBalancerClient) client).getDelegate();
			}
			builder.client(client);
		}
		Targeter targeter = get(context, Targeter.class);
		return (T) targeter.target(this, builder, context,
				new HardCodedTarget<>(this.type, this.name, url));
	}

	protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
			HardCodedTarget<T> target) {
		Client client = getOptional(context, Client.class);
		if (client != null) {
			builder.client(client);
			Targeter targeter = get(context, Targeter.class);
			return targeter.target(this, builder, context, target);
		}

		throw new IllegalStateException(
				"No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon?");
	}
```

1. 在最初的时候，如果是以服务名的形式指定的，那么就会调用`loadBalance`方法。
2. `loadBalance`方法中拿到的`client`对象的具体实现类为`LoadBalancerFeignClient`。
3. 所以在最后发送请求时`response = client.execute(request, options);`使用的就是Ribbon的实现了。



`org.springframework.cloud.openfeign.ribbon.LoadBalancerFeignClient#execute`

```java
	public Response execute(Request request, Request.Options options) throws IOException {
		try {
			URI asUri = URI.create(request.url());
			String clientName = asUri.getHost();
			URI uriWithoutHost = cleanUrl(request.url(), clientName);
			FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
					this.delegate, request, uriWithoutHost);

			IClientConfig requestConfig = getClientConfig(options, clientName);
			return lbClient(clientName)
					.executeWithLoadBalancer(ribbonRequest, requestConfig).toResponse();
		}
		catch (ClientException e) {
			IOException io = findIOException(e);
			if (io != null) {
				throw io;
			}
			throw new RuntimeException(e);
		}
	}
```

之后就是调用ribbon相关内容了。








# 引用
[spring-cloud-openFeign源码深度解析](https://blog.csdn.net/sinat_29899265/article/details/86577997)

[Feign整合Ribbon和Hystrix源码解析](https://www.jianshu.com/p/6373c19f8ba9)