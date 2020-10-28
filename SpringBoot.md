---
typora-root-url: springboot
typora-copy-images-to: springboot
---

# SpringBoot

## 配置文件记载过程

1. 在springboot创建初始，**SpringApplication**构造函数中会加载各种**ApplicationListener**类。包含ConfigFileApplicationListener。

   ```java
   //SpringApplication.run(DemoApplication.class, args);
   
   public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
       this.resourceLoader = resourceLoader;
       Assert.notNull(primarySources, "PrimarySources must not be null");
       this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
       this.webApplicationType = WebApplicationType.deduceFromClasspath();
       setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
        // 从/META-INFO/spring.factories找到所有的ApplicationListener，其中就包含ConfigFileApplicationListener
       setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
       this.mainApplicationClass = deduceMainApplicationClass();
   }
   ```

2.  SpringApplication.run方法;  

   a. 查看 prepareEnvironment(listeners, applicationArguments);

   ![image-20201027172900755](/image-20201027172900755.png)

   b. getOrCreateEnvironment()方法，创建environment对象，不用的运行环境创建不同的environment对象，包括StandardServletEnvironment(普通的web应用）, StandardReactiveWebEnvironment（React模式下）, StandardEnvironment（啥也没有的普通应用）.

   发布一环境准备事件 **ApplicationContextInitializedEvent**

   ![image-20201027173010370](/image-20201027173010370.png)

![image-20201027173036095](/image-20201027173036095.png)

![image-20201027173105298](/image-20201027173105298.png)

![image-20201027173204157](/image-20201027173204157.png)

3. ConfigureFileApplicationListener会监听此事件。 判断 event 是否是ApplicationContextInitializedEvent，如果是执行 **onApplicationEnvironmentPreparedEvent**方法

   ![image-20201027173404970](/image-20201027173404970.png)

   详细说明一下该方法。

   ```java
   	private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
           //可以查看该方法，大概含义是，从/META-INFO/spring.factories中加载所有EnvironmentPostProcessor的实现（经过查看，spring.factories中并不好含当前类ConfigureFileApplicationListener）
   		List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
           //将当前类添加到列表中
   		postProcessors.add(this);
   		AnnotationAwareOrderComparator.sort(postProcessors);
           //执行所有的postProcessEnvironment方法，包含当前类的方法。
   		for (EnvironmentPostProcessor postProcessor : postProcessors) {
   			postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
   		}
   	}
   ```

   ![image-20201027173802941](/image-20201027173802941.png)

   ![image-20201027173820088](/image-20201027173820088.png)



## 注解相关

### 1. @Lazy

> lazy作为懒加载的标志，使用其注解的对象或其他只有在被使用的时候才会进行加载初始化。 

由`被使用才加载其对象属性值`引出一个问题。 **对象已经初始化，如何才能实现使用时才可以进行加载其属性值？**

@**Lazy** **几种使用场景**

- 和@bean一块使用。若不加lazy注解，spring容器默认的会认为ObjectA为单例对象，在spring容器启动的时候变会实例化objectA；此时加上@Lazy注解后，容器启动时不会实例化该对象。

  ```java
  @Bean
  @Lazy
  public Object getObjectA(){
  	return new ObjectA();
  }
  ```

  **源码分析**

  1. 解析过程

  容器启动时执行refresh方法，其中有一步骤 `invokeBeanFactoryPostProcessors(beanFactory);`执行各种BeanFactoryPostProcessors，包含`ConfigurationClassPostProcessor`，一直跟代码，直到

  <img src="/image-20201028094312290.png" alt="image-20201028094312290" style="zoom:50%;" />

<img src="/image-20201028094432340.png" alt="image-20201028094432340" style="zoom:50%;" />

<img src="/image-20201028094456258.png" alt="image-20201028094456258" style="zoom:50%;" />

<img src="/image-20201028094523806.png" alt="image-20201028094523806" style="zoom:50%;" />

<img src="/image-20201028094537460.png" alt="image-20201028094537460" style="zoom:50%;" />

**详解processConfigBeanDefinitions方法**

> BeanDefinition 在Spring中代表一个Bean对象的定义，主要定义一个类实例化时需要的各种属性，比如 别名，类，单例/多例，是否懒加载。

processConfigBeanDefinitions方法中，首先会执行`parser.parse(candidates)`方法解析各种已经声明的类 **包含**`@SpringBootApplication`注解的主类，

`@SpringBootApplication`注解中包含`@ComponentScan`，

<img src="/image-20201028110138353.png" alt="image-20201028110138353" style="zoom:50%;" />

<img src="/image-20201028110234838.png" alt="image-20201028110234838" style="zoom:50%;" />

<img src="/image-20201028110309054.png" alt="image-20201028110309054" style="zoom:50%;" />



便会扫描包下所有的@Component(@Configuration @controller @service  @Repository 等等)属性注解的类，封装成bean定义，并将bean定义注册到容器中。然后判断bean是否为configuration注解类，如果是解析该类，获取@bean注解的method，解析其方法获取方法的定义MethodMetaData（用于之后的加载）。

```JAVA
		//解析DemoApplication类上的注解，获取到SpringbootApplication中的Component注解
		Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
		if (!componentScans.isEmpty() &&
				!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
			for (AnnotationAttributes componentScan : componentScans) {
				// The config class is annotated with @ComponentScan -> perform the scan immediately
                // 根据ComponentScan注解进行包扫描，
                // 扫描到所有的Component的注解，@Controller @Service @Configuration @Repository等等获取所有的bean定义
				Set<BeanDefinitionHolder> scannedBeanDefinitions =
						this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
				// Check the set of scanned definitions for any further config classes and parse recursively if needed
                //遍历bean定义
				for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
					BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
					if (bdCand == null) {
						bdCand = holder.getBeanDefinition();
					}
                    //如果bean的注解为@Configuration，则解析当前bean，因为要解析到Configuration中的@Bean注解的方法，获取bean
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                        //递归parse方法
						parse(bdCand.getBeanClassName(), holder.getBeanName());
					}
				}
			}
		}	
		// Process any @Import annotations
		// 解析带有@Import注解 实例化注解中包含的类
		processImports(configClass, sourceClass, getImports(sourceClass), filter, true);

		// Process any @ImportResource annotations
		AnnotationAttributes importResource =
				AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
		if (importResource != null) {
			String[] resources = importResource.getStringArray("locations");
			Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
			for (String resource : resources) {
				String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
				configClass.addImportedResource(resolvedResource, readerClass);
			}
		}

		// Process individual @Bean methods
		// 解析@Bean注解的method
		Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
		for (MethodMetadata methodMetadata : beanMethods) {
			configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
		}

		// Process default methods on interfaces
		processInterfaces(configClass, sourceClass);

		// Process superclass, if any
		if (sourceClass.getMetadata().hasSuperClass()) {
			String superclass = sourceClass.getMetadata().getSuperClassName();
			if (superclass != null && !superclass.startsWith("java") &&
					!this.knownSuperclasses.containsKey(superclass)) {
				this.knownSuperclasses.put(superclass, configClass);
				// Superclass found, return its annotation metadata and recurse
				return sourceClass.getSuperClass();
			}
		}
```

<img src="/image-20201028111300928.png" alt="image-20201028111300928" style="zoom:50%;" />![image-20201028111432921](/image-20201028111432921.png)

<img src="/image-20201028111534479.png" alt="image-20201028111534479" style="zoom:50%;" />

2. 将解析出来的配置转换成bean定义过程

   解析完之后，加载所有配置过的类。

   <img src="/image-20201028141342766.png" alt="image-20201028141342766" style="zoom:50%;" />

<img src="/image-20201028141847770.png" alt="image-20201028141847770" style="zoom:50%;" />

<img src="/image-20201028142019413.png" alt="image-20201028142019413" style="zoom:50%;" />![image-20201028142035579](/image-20201028142035579.png)

![image-20201028142035579](/image-20201028142035579.png)

3. 在容器refresh方法后期，会根据bean定义生成实例，如果为多例或者懒加载的bean定义，则不会实例化。