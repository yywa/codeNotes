# SpringApplication-准备(二) 

​	接着看SpringApplication.run(args)方法

```java
public ConfigurableApplicationContext run(String... args) {
  //一个用来记录任务耗时的类。注意：非线程安全，也不用于生产。
   StopWatch stopWatch = new StopWatch();
   stopWatch.start();
   ConfigurableApplicationContext context = null;
   Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
  //配置HeadlessProperty
   configureHeadlessProperty();
  //获取监听器并启动
   SpringApplicationRunListeners listeners = getRunListeners(args); 
  listeners.starting();
   try {
     //装配ApplicationArguments
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(
            args);
     //准备ConfigurableEnvironment
      ConfigurableEnvironment environment = prepareEnvironment(listeners,
            applicationArguments);
      configureIgnoreBeanInfo(environment);
      Banner printedBanner = printBanner(environment);
     //创建Spring应用上下文
      context = createApplicationContext();
     //创建异常报告类
      exceptionReporters = getSpringFactoriesInstances(
            SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);
     //Spring应用上下文运行前准备。
      prepareContext(context, environment, listeners, applicationArguments,
            printedBanner);
     //运行阶段
      refreshContext(context);
     //Spring应用上下文启动后
      afterRefresh(context, applicationArguments);
      stopWatch.stop();
      if (this.logStartupInfo) {
         new StartupInfoLogger(this.mainApplicationClass)
               .logStarted(getApplicationLog(), stopWatch);
      }
     //执行回调Spring上下文应用运行。
      listeners.started(context);
     //执行CommandLineRunner和ApplicationRunner
      callRunners(context, applicationArguments);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, listeners);
      throw new IllegalStateException(ex);
   }

   try {
      listeners.running(context);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, null);
      throw new IllegalStateException(ex);
   }
   return context;
}
```

## configureHeadlessProperty()

​	简单的配置了一个property

```java
private void configureHeadlessProperty() {
   System.setProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS, System.getProperty(
         SYSTEM_PROPERTY_JAVA_AWT_HEADLESS, Boolean.toString(this.headless)));
}
```

## getRunListeners()

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
   Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
   return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(
         SpringApplicationRunListener.class, types, this, args));
}
```

```java
SpringApplicationRunListeners(Log log,
      Collection<? extends SpringApplicationRunListener> listeners) {
   this.log = log;
   this.listeners = new ArrayList<>(listeners);
}
```

### 理解SpringApplicationRunListener

| 方法                                       | 说明                                       | 事件                                  |
| ---------------------------------------- | ---------------------------------------- | ----------------------------------- |
| starting()                               | 当运行方法启动时立即调用。                            | ApplicationStartingEvent            |
| void environmentPrepared(ConfigurableEnvironment environment) | ConfigurableEnvironment准备妥当              | ApplicationEnvironmentPreparedEvent |
| void contextPrepared(ConfigurableApplicationContext context); | ConfigurableApplicationContext准备妥当。      |                                     |
| void contextLoaded(ConfigurableApplicationContext context); | ConfigurableApplicationContext已经装载。      | ApplicationPreparedEvent            |
| void started(ConfigurableApplicationContext context); | ConfigurableApplicationContext已经启动，此时Spring Bean已经完成初始化。 | ApplicationStartedEvent             |
| void running(ConfigurableApplicationContext context); | Spring应用正在运行。                            | ApplicationReadyEvent               |
| void failed(ConfigurableApplicationContext context, Throwable exception); | Spring应用运行失败。                            | ApplicationFailedEvent              |

​	getRunListener()方法中，调用了getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args)，

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
      Class<?>[] parameterTypes, Object... args) {
   ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
   // Use names and ensure unique to protect against duplicates
   Set<String> names = new LinkedHashSet<>(
         SpringFactoriesLoader.loadFactoryNames(type, classLoader));
   List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
         classLoader, args, names);
   AnnotationAwareOrderComparator.sort(instances);
   return instances;
}
```

​	结合我们之前了解过的SpringfactoriesLoader机制，可以很清楚的理解，getSpringFactoriesInstances方法，也就是读取了META-INF/spring.factories文下的SpringApplicationRunListener配置内容，并初始化和排序。

```java
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener
```

```java
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
 	public EventPublishingRunListener(SpringApplication application, String[] args) {
		this.application = application;
		this.args = args;
		this.initialMulticaster = new SimpleApplicationEventMulticaster();
		for (ApplicationListener<?> listener : application.getListeners()) {
			this.initialMulticaster.addApplicationListener(listener);
		}
	} 
}
```

​	简单看下**EventPublishingRunListener**的构造方法，内部有一个**SimpleApplicationEventMulticaster**实例，并且动态的将listener添加到SimpleApplicationEventMulticaster中。

​	链路为：SpringApplication.run()-->getRunListeners()-->EventPublishingRunListener.starting()

#### 理解Spring事件/监听机制。

​	Spring事件抽象类为**ApplicationEvent**。

```java
public abstract class ApplicationEvent extends EventObject {
	public ApplicationEvent(Object source) {
		super(source);
		this.timestamp = System.currentTimeMillis();
	}
}
```

```java
public EventObject(Object source) {
    if (source == null)
        throw new IllegalArgumentException("null source");

    this.source = source;
}
```

​	Java的事件监听者必须是**EventListener**实例。注意：该接口为空接口，考虑很难适用于所有的场景。

```java
public interface EventListener {
}
```

​	来看Spring中对该接口的扩展**ApplicationListener**。

```java
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

   /**
    * Handle an application event.
    * @param event the event to respond to
    */
   void onApplicationEvent(E event);

}
```

​	接下来分析**ApplicationEventMulticaster**，该接口主要承担两个责任，

- 关联ApplicationListener
- 广播ApplicationEvent

```java
public interface ApplicationEventMulticaster {

   void addApplicationListener(ApplicationListener<?> listener);

   void addApplicationListenerBean(String listenerBeanName);

   void removeApplicationListener(ApplicationListener<?> listener);

   void removeApplicationListenerBean(String listenerBeanName);

   void removeAllListeners();

   void multicastEvent(ApplicationEvent event);

   void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType);

}
```

##### 1）ApplicationEventMulticaster 注册ApplicationListener

​	ApplicationEventMulticaster 其实现类为**SimpleApplicationEventMulticaster**，SimpleApplicationEventMulticaster类继承自**AbstractApplicationEventMulticaster**，AbstractApplicationEventMulticaster并未直接关联ApplicationListener，而是通过属性defaultRetriever和retrieverCache进行关联。

```java
public abstract class AbstractApplicationEventMulticaster
      implements ApplicationEventMulticaster, BeanClassLoaderAware, BeanFactoryAware {
	
  	private final ListenerRetriever defaultRetriever = new ListenerRetriever(false);

	final Map<ListenerCacheKey, ListenerRetriever> retrieverCache = new ConcurrentHashMap<>(64);
}
```

​	ListenerCacheKey，ListenerRetriever 均为AbstractApplicationEventMulticaster的内置类。

​	AbstractApplicationEventMulticaster#getApplicationListeners()：返回ApplicationEvenet关联的ApplicationListener。实际调用为ListenerRetriever#getApplicationListeners()。

```java
protected Collection<ApplicationListener<?>> getApplicationListeners() {
   synchronized (this.retrievalMutex) {
      return this.defaultRetriever.getApplicationListeners();
   }
}
```

```java
public Collection<ApplicationListener<?>> getApplicationListeners() {
   List<ApplicationListener<?>> allListeners = new ArrayList<>(
         this.applicationListeners.size() + this.applicationListenerBeans.size());
   allListeners.addAll(this.applicationListeners);
   if (!this.applicationListenerBeans.isEmpty()) {
      BeanFactory beanFactory = getBeanFactory();
      for (String listenerBeanName : this.applicationListenerBeans) {
         try {
            ApplicationListener<?> listener = beanFactory.getBean(listenerBeanName, ApplicationListener.class);
            if (this.preFiltered || !allListeners.contains(listener)) {
               allListeners.add(listener);
            }
         }
         catch (NoSuchBeanDefinitionException ex) {
            // Singleton listener instance (without backing bean definition) disappeared -
            // probably in the middle of the destruction phase
         }
      }
   }
   if (!this.preFiltered || !this.applicationListenerBeans.isEmpty()) {
      AnnotationAwareOrderComparator.sort(allListeners);
   }
   return allListeners;
}
```

​	获取到所有的ApplicationListener集合，并经过@Order排序。

##### 2)ApplicationEventMulticaster广播事件。

​	void multicastEvent(ApplicationEvent event);

​	void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType);

​	这两个方法均由SimpleApplicationEventMulticaster实现。

```java
public void multicastEvent(ApplicationEvent event) {
   multicastEvent(event, resolveDefaultEventType(event));
}
```

```java
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
   ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
  //调用getApplicationListeners()方法，获取ApplicationListeners，然后迭代执行invokeListener调用onApplicationEvent()方法。
   for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
      Executor executor = getTaskExecutor();
      if (executor != null) {
         executor.execute(() -> invokeListener(listener, event));
      }
      else {
         invokeListener(listener, event);
      }
   }
}
```

```java
protected Collection<ApplicationListener<?>> getApplicationListeners(
      ApplicationEvent event, ResolvableType eventType) {

   Object source = event.getSource();
   Class<?> sourceType = (source != null ? source.getClass() : null);
   ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);

   // Quick check for existing entry on ConcurrentHashMap...
  	//从缓存中直接获取
   ListenerRetriever retriever = this.retrieverCache.get(cacheKey);
   if (retriever != null) {
      return retriever.getApplicationListeners();
   }
	//
   if (this.beanClassLoader == null ||
         (ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) &&
               (sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader)))) {
      // Fully synchronized building and caching of a ListenerRetriever
      synchronized (this.retrievalMutex) {
        //通过Listenerretriever获取ApplicationListeners
         retriever = this.retrieverCache.get(cacheKey);
         if (retriever != null) {
            return retriever.getApplicationListeners();
         }
         retriever = new ListenerRetriever(true);
         Collection<ApplicationListener<?>> listeners =
               retrieveApplicationListeners(eventType, sourceType, retriever);
         this.retrieverCache.put(cacheKey, retriever);
         return listeners;
      }
   }
   else {
      // No ListenerRetriever caching -> no synchronization necessary
     //如果没有任何缓存，再获取ApplicationListener
      return retrieveApplicationListeners(eventType, sourceType, null);
   }
}
```

```java
protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
   ErrorHandler errorHandler = getErrorHandler();
   if (errorHandler != null) {
      try {
         doInvokeListener(listener, event);
      }
      catch (Throwable err) {
         errorHandler.handleError(err);
      }
   }
   else {
      doInvokeListener(listener, event);
   }
}
```

```java
private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
   try {
      listener.onApplicationEvent(event);
   }
   catch (ClassCastException ex) {
      String msg = ex.getMessage();
      if (msg == null || matchesClassCastMessage(msg, event.getClass())) {
         // Possibly a lambda-defined listener which we could not resolve the generic event type for
         // -> let's suppress the exception and just log a debug message.
         Log logger = LogFactory.getLog(getClass());
         if (logger.isDebugEnabled()) {
            logger.debug("Non-matching event type for listener: " + listener, ex);
         }
      }
      else {
         throw ex;
      }
   }
}
```

​	那么SimpleApplicationEventMulticaster，什么时候注册的呢？

​	引入AbstractApplicationContext类。

```java
public void publishEvent(ApplicationEvent event) {
   publishEvent(event, null);
}
```

```java
protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
   Assert.notNull(event, "Event must not be null");
   if (logger.isTraceEnabled()) {
      logger.trace("Publishing event in " + getDisplayName() + ": " + event);
   }

   // Decorate event as an ApplicationEvent if necessary
   ApplicationEvent applicationEvent;
   if (event instanceof ApplicationEvent) {
      applicationEvent = (ApplicationEvent) event;
   }
   else {
      applicationEvent = new PayloadApplicationEvent<>(this, event);
      if (eventType == null) {
         eventType = ((PayloadApplicationEvent) applicationEvent).getResolvableType();
      }
   }

   // Multicast right now if possible - or lazily once the multicaster is initialized
   if (this.earlyApplicationEvents != null) {
      this.earlyApplicationEvents.add(applicationEvent);
   }
   else {
      getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
   }

   // Publish event via parent context as well...
   if (this.parent != null) {
      if (this.parent instanceof AbstractApplicationContext) {
         ((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
      }
      else {
         this.parent.publishEvent(event);
      }
   }
}
```

​	 在getApplicationEventMulticaster()中，返回的是applicationEventMulticaster。这个属性是调用refresh()中的initApplicationEventMulticaster()进行初始化。

```java
protected void initApplicationEventMulticaster() {
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
      this.applicationEventMulticaster =
            beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
      if (logger.isDebugEnabled()) {
         logger.debug("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
      }
   }
   else {
      this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
      beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
      if (logger.isDebugEnabled()) {
         logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
               APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
               "': using default [" + this.applicationEventMulticaster + "]");
      }
   }
}
```

​	会先判断当前Spring应用上下文中是否存在applicationEventMulticaster且类型为ApplicationEventMulticaster的Bean，如果不存在，就将其构造成SimpleApplicationEventMulticaster。

#### 理解Spring Boot事件机制：

​	在META-INF中定义的EventPublishingRunListener中，实现了SpringApplicationRunListener相关方法。



```java
public void starting() {
   this.initialMulticaster.multicastEvent(
         new ApplicationStartingEvent(this.application, this.args));
}

@Override
public void environmentPrepared(ConfigurableEnvironment environment) {
   this.initialMulticaster.multicastEvent(new ApplicationEnvironmentPreparedEvent(
         this.application, this.args, environment));
}

@Override
public void contextPrepared(ConfigurableApplicationContext context) {

}

@Override
public void contextLoaded(ConfigurableApplicationContext context) {
   for (ApplicationListener<?> listener : this.application.getListeners()) {
      if (listener instanceof ApplicationContextAware) {
         ((ApplicationContextAware) listener).setApplicationContext(context);
      }
      context.addApplicationListener(listener);
   }
   this.initialMulticaster.multicastEvent(
         new ApplicationPreparedEvent(this.application, this.args, context));
}

@Override
public void started(ConfigurableApplicationContext context) {
   context.publishEvent(
         new ApplicationStartedEvent(this.application, this.args, context));
}

@Override
public void running(ConfigurableApplicationContext context) {
   context.publishEvent(
         new ApplicationReadyEvent(this.application, this.args, context));
}

@Override
public void failed(ConfigurableApplicationContext context, Throwable exception) {
   ApplicationFailedEvent event = new ApplicationFailedEvent(this.application,
         this.args, context, exception);
   if (context != null && context.isActive()) {
      // Listeners have been registered to the application context so we should
      // use it at this point if we can
      context.publishEvent(event);
   }
   else {
      // An inactive context may not have a multicaster so we use our multicaster to
      // call all of the context's listeners instead
      if (context instanceof AbstractApplicationContext) {
         for (ApplicationListener<?> listener : ((AbstractApplicationContext) context)
               .getApplicationListeners()) {
            this.initialMulticaster.addApplicationListener(listener);
         }
      }
      this.initialMulticaster.setErrorHandler(new LoggingErrorHandler());
      this.initialMulticaster.multicastEvent(event);
   }
}
```

​	然后，在ConfigurableApplicationContext#refresh()调用过程中，会调用registrarListener()方法，将其关联的ApplicationListener实例与管理的ApplicationListener的Bean实例整合为新的集合。

```java
public void refresh() throws BeansException, IllegalStateException {
  ...
  registerListeners();
  ...
}
```

```java
protected void registerListeners() {
   // Register statically specified listeners first.
   for (ApplicationListener<?> listener : getApplicationListeners()) {
      getApplicationEventMulticaster().addApplicationListener(listener);
   }

   // Do not initialize FactoryBeans here: We need to leave all regular beans
   // uninitialized to let post-processors apply to them!
   String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
   for (String listenerBeanName : listenerBeanNames) {
      getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
   }

   // Publish early application events now that we finally have a multicaster...
   Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
   this.earlyApplicationEvents = null;
   if (earlyEventsToProcess != null) {
      for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
         getApplicationEventMulticaster().multicastEvent(earlyEvent);
      }
   }
}
```

​	而其中的getApplicationEventMulticaster()返回的正是之前实例化的SimpleApplicationEventMulticaster。

## 装配ApplicationArguments

```java
public class DefaultApplicationArguments implements ApplicationArguments {

	private final Source source;

	private final String[] args;

	public DefaultApplicationArguments(String[] args) {
		Assert.notNull(args, "Args must not be null");
		this.source = new Source(args);
		this.args = args;
	}
  
  	private static class Source extends SimpleCommandLinePropertySource {

		Source(String[] args) {
			super(args);
		}

		@Override
		public List<String> getNonOptionArgs() {
			return super.getNonOptionArgs();
		}

		@Override
		public List<String> getOptionValues(String name) {
			return super.getOptionValues(name);
        }

	}
}
```

​	source为内置类实现，依赖于SimpleCommandLinePropertySource去进行解析。根据规则，命令行参数"-- name = xxxx" 会被解析为"name:xxxx"的键值对。参数必须以"--"为前缀。

## 准备ConfigurableEnvironment

```java
private ConfigurableEnvironment prepareEnvironment(
      SpringApplicationRunListeners listeners,
      ApplicationArguments applicationArguments) {
   // Create and configure the environment
  //根据应用类型创建对应的运行环境。
   ConfigurableEnvironment environment = getOrCreateEnvironment();
  //根据命令参数配置运行环境。
   configureEnvironment(environment, applicationArguments.getSourceArgs());
  //一旦环境被创建好了之后立即调用，在环境创建之前已经进行监听。
   listeners.environmentPrepared(environment);
  //绑定运行环境与SpringApplication
   bindToSpringApplication(environment);
   if (!this.isCustomEnvironment) {
     //将指定的运行环境转变成标准环境。
      environment = new EnvironmentConverter(getClassLoader())
            .convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
   }
 //附加一个ConfigurationPropertySource到指定的环境中。
   ConfigurationPropertySources.attach(environment);
   return environment;
}
```
### getOrCreateEnvironment()

```java
private ConfigurableEnvironment getOrCreateEnvironment() {
   if (this.environment != null) {
      return this.environment;
   }
   switch (this.webApplicationType) {
   case SERVLET:
      return new StandardServletEnvironment();
   case REACTIVE:
      return new StandardReactiveWebEnvironment();
   default:
      return new StandardEnvironment();
   }
}
```
### configureEnvironment()

```java
protected void configureEnvironment(ConfigurableEnvironment environment,
      String[] args) {
   configurePropertySources(environment, args);
   configureProfiles(environment, args);
}
```

#### configurePropertySources()

```java
//添加，删除，或者排序PropertySources
//如果传入的args有值，并解析成键值对，在这里会进行操作。
protected void configurePropertySources(ConfigurableEnvironment environment,
      String[] args) {
   MutablePropertySources sources = environment.getPropertySources();
   if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
      sources.addLast(
            new MapPropertySource("defaultProperties", this.defaultProperties));
   }
   if (this.addCommandLineProperties && args.length > 0) {
      String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
      if (sources.contains(name)) {
         PropertySource<?> source = sources.get(name);
         CompositePropertySource composite = new CompositePropertySource(name);
         composite.addPropertySource(new SimpleCommandLinePropertySource(
               "springApplicationCommandLineArgs", args));
         composite.addPropertySource(source);
         sources.replace(name, composite);
      }
      else {
         sources.addFirst(new SimpleCommandLinePropertySource(args));
      }
   }
}
```

#### configureProfiles

```java
 //判断哪些profile是活跃的。
protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
   environment.getActiveProfiles(); // ensure they are initialized
   // But these ones should go first (last wins in a property key clash)
   Set<String> profiles = new LinkedHashSet<>(this.additionalProfiles);
   profiles.addAll(Arrays.asList(environment.getActiveProfiles()));
   environment.setActiveProfiles(StringUtils.toStringArray(profiles));
}
```



## 创建Spring应用上下文

```java
protected ConfigurableApplicationContext createApplicationContext() {
   Class<?> contextClass = this.applicationContextClass;
   if (contextClass == null) {
      try {
        //根据推断出的webApplicationType构建Spring应用上下文
         switch (this.webApplicationType) {
         case SERVLET:
            contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
            break;
         case REACTIVE:
            contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
            break;
         default:
            contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
         }
      }
      catch (ClassNotFoundException ex) {
         throw new IllegalStateException(
               "Unable create a default ApplicationContext, "
                     + "please specify an ApplicationContextClass",
               ex);
      }
   }
   return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

## 创建异常报告类

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
      Class<?>[] parameterTypes, Object... args) {
   ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
   // Use names and ensure unique to protect against duplicates
   Set<String> names = new LinkedHashSet<>(
         SpringFactoriesLoader.loadFactoryNames(type, classLoader));
   List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
         classLoader, args, names);
   AnnotationAwareOrderComparator.sort(instances);
   return instances;
}
```

## Spring应用上下文运行前准备

```java
private void prepareContext(ConfigurableApplicationContext context,
      ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
      ApplicationArguments applicationArguments, Banner printedBanner) {
  //设置当前上下文的环境。
   context.setEnvironment(environment);
  //用于context的后置处理
   postProcessApplicationContext(context);
  //Spring应用上下文初始化器
   applyInitializers(context);
  //执行contextPrepared方法回调。当Spring上下文创建并准备好。
   listeners.contextPrepared(context);
   if (this.logStartupInfo) {
      logStartupInfo(context.getParent() == null);
      logStartupProfileInfo(context);
   }

   // Add boot specific singleton beans
  //添加特定的Spring bean
   context.getBeanFactory().registerSingleton("springApplicationArguments",
         applicationArguments);
   if (printedBanner != null) {
      context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
   }

   // Load the sources
  //合并Spring应用上下文配置
   Set<Object> sources = getAllSources();
   Assert.notEmpty(sources, "Sources must not be empty");
  //加载Spring应用上下文配置
   load(context, sources.toArray(new Object[0]));
  //执行回调contextLoaded
   listeners.contextLoaded(context);
}  
```
### context的后置处理

```java
//用于ConfigurableApplicationContext的后置处理。
protected void postProcessApplicationContext(ConfigurableApplicationContext的后置处理。 context) {
  //如果存在beanNameGenerator，会被注册成CONFIGURATION_BEAN_NAME_GENERATOR常量的bean。
   if (this.beanNameGenerator != null) {
      context.getBeanFactory().registerSingleton(
            AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
            this.beanNameGenerator);
   }
  //覆盖当前应用默认的ResourceLoader和ClassLoader
   if (this.resourceLoader != null) {
      if (context instanceof GenericApplicationContext) {
         ((GenericApplicationContext) context)
               .setResourceLoader(this.resourceLoader);
      }
      if (context instanceof DefaultResourceLoader) {
         ((DefaultResourceLoader) context)
               .setClassLoader(this.resourceLoader.getClassLoader());
      }
   }
}
```

### Spring应用上下文初始化器

```java
protected void applyInitializers(ConfigurableApplicationContext context) {
   for (ApplicationContextInitializer initializer : getInitializers()) {
      Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
            initializer.getClass(), ApplicationContextInitializer.class);
      Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
      initializer.initialize(context);
   }
}
```

#### getInitializers()

```java
public Set<ApplicationContextInitializer<?>> getInitializers() {
  //通过LinkedHashSet进行去重和排序。
   return asUnmodifiableOrderedSet(this.initializers);
}
```

#### asUnmodifiableOrderedSet()

```java
private static <E> Set<E> asUnmodifiableOrderedSet(Collection<E> elements) {
   List<E> list = new ArrayList<>();
   list.addAll(elements);
   list.sort(AnnotationAwareOrderComparator.INSTANCE);
   return new LinkedHashSet<>(list);
}
```

### 合并Spring应用上下文配置

```java
public Set<Object> getAllSources() {
   Set<Object> allSources = new LinkedHashSet<>();
   if (!CollectionUtils.isEmpty(this.primarySources)) {
      allSources.addAll(this.primarySources);
   }
   if (!CollectionUtils.isEmpty(this.sources)) {
      allSources.addAll(this.sources);
   }
  //将primarySources和sources合并，并返回只读set
   return Collections.unmodifiableSet(allSources);
}
```

### 加载Spring应用上下文配置

```java
protected void load(ApplicationContext context, Object[] sources) {
   if (logger.isDebugEnabled()) {
      logger.debug(
            "Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
   }
  //了解Spring的应该知道，Bean装载是通过BeanDefinitionLoader。
  //创建BeanDefinitionLoader
   BeanDefinitionLoader loader = createBeanDefinitionLoader(
         getBeanDefinitionRegistry(context), sources);
   if (this.beanNameGenerator != null) {
      loader.setBeanNameGenerator(this.beanNameGenerator);
   }
   if (this.resourceLoader != null) {
      loader.setResourceLoader(this.resourceLoader);
   }
   if (this.environment != null) {
      loader.setEnvironment(this.environment);
   }
  //通过不同的BeanDefinitionReader进行load。
   loader.load();
}
```

```java
public int load() {
   int count = 0;
   for (Object source : this.sources) {
      count += load(source);
   }
   return count;
}
```

```java

private int load(Object source) {
   Assert.notNull(source, "Source must not be null");
   if (source instanceof Class<?>) {
      return load((Class<?>) source);
   }
   if (source instanceof Resource) {
      return load((Resource) source);
   }
   if (source instanceof Package) {
      return load((Package) source);
   }
   if (source instanceof CharSequence) {
      return load((CharSequence) source);
   }
   throw new IllegalArgumentException("Invalid source type " + source.getClass());
}
```

### 执行回调contextLoaded

​	当context#load完成后，执行contextLoaded方法回调。进入EventPublishingRunListener#contextLoaded方法。

```java
public void contextLoaded(ConfigurableApplicationContext context) {
   for (ApplicationListener<?> listener : this.application.getListeners()) {
      if (listener instanceof ApplicationContextAware) {
         ((ApplicationContextAware) listener).setApplicationContext(context);
      }
      context.addApplicationListener(listener);
   }
   this.initialMulticaster.multicastEvent(
         new ApplicationPreparedEvent(this.application, this.args, context));
}
```

此时，Spring应用上下文，进入实质性的启动阶段。

## 运行阶段

```java
private void refreshContext(ConfigurableApplicationContext context) {
   refresh(context);
   if (this.registerShutdownHook) {
      try {
         context.registerShutdownHook();
      }
      catch (AccessControlException ex) {
         // Not allowed in some environments.
      }
   }
}
```

然后调用AbstractApplicationContext#refresh()方法，将Spring上下文启动。

## Spring应用上下文启动后

```java
protected void afterRefresh(ConfigurableApplicationContext context,
      ApplicationArguments args) {
}
```

​	这里是一个空的实现。可以自己扩展。

## 执行CommandLineRunner和ApplicationRunner

```java
private void callRunners(ApplicationContext context, ApplicationArguments args) {
   List<Object> runners = new ArrayList<>();
   runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
   runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
   AnnotationAwareOrderComparator.sort(runners);
   for (Object runner : new LinkedHashSet<>(runners)) {
      if (runner instanceof ApplicationRunner) {
         callRunner((ApplicationRunner) runner, args);
      }
      if (runner instanceof CommandLineRunner) {
         callRunner((CommandLineRunner) runner, args);
      }
   }
}
```

