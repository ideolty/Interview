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
- 与hystrix是如何适配的。
- 如何与ribbon适配，实现了请求重试、动态路由与负载均衡。



# 启动流程

//todo 看情况 是否需要一张流程图



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

