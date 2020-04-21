# SpringApplication-构造(一)

```java
public class BootStrapApplication {
    public static void main(String[] args) {
        SpringApplication.run(BootStrapApplication.class, args);
    }
}
```

```java
public static ConfigurableApplicationContext run(Class<?> primarySource,
      String... args) {
   return run(new Class<?>[] { primarySource }, args);
}
```

```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources,
			String[] args) {
		return new SpringApplication(primarySources).run(args);
}
```

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
  		//推断Web应用类型
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
  		//加载Spring应用上下文初始化器
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
		//加载Spring应用事件监听器和推断应用引用类
  		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		this.mainApplicationClass = deduceMainApplicationClass();
}
```


​	SpringApplication.run(BootStrapApplication.class, args)等价于new SpringApplication(BootStrapApplication.class).run(args)。根据调用链，可以看到，依次调用了**deduceFromClasspath()**，**setInitializers()**，**setListeners()**，**deduceMainApplicationClass()**方法。

## deduceFromClasspath()

```java
private static final String WEBFLUX_INDICATOR_CLASS = "org."
			+ "springframework.web.reactive.DispatcherHandler";
private static final String WEBMVC_INDICATOR_CLASS = "org.springframework."
  			+ "web.servlet.DispatcherServlet";
private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";
private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };
static WebApplicationType deduceFromClasspath() {
  //仅仅依赖DispatcherHandler存在。
   if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)
         && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
         && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
      return WebApplicationType.REACTIVE;
   }
  //当Servlet和ConfigurationWebApplicationContext都不存在时
   for (String className : SERVLET_INDICATOR_CLASSES) {
      if (!ClassUtils.isPresent(className, null)) {
         return WebApplicationType.NONE;
      }
   }
   return WebApplicationType.SERVLET;
}
```

​	通过ClassUtils.isPresent(String,ClassLoader)方法依次判断DispatcherServlet，DispatcherHandler，ConfigurableWebApplicationContext，Servlet和DispatcherServlet的组合情况，从而推断Web应用类型。

## setInitializers()

​	包含两个方法：getSpringFactoriesInstances(Class)和setInitializers(Collection)

```java
private <T> Collection<T> getSpringFactoriesInstances(Class)和(Class<T> type) {
   return getSpringFactoriesInstances(type, new Class<?>[] {});
}
```

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
​	这里使用了SpringFactoriesLoader.loadFactoryNames()，该方法返回所有META-INF/spring.factories资源中配置的ApplicationContextInitializer实现类名单。

```java
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener
```

​	获取实现类名单后，调用createSpringFactoriesInstances()方法初始化实现类。

```java
private <T> List<T> createSpringFactoriesInstances(Class<T> type,
      Class<?>[] parameterTypes, ClassLoader classLoader, Object[] args,
      Set<String> names) {
   List<T> instances = new ArrayList<>(names.size());
   for (String name : names) {
      try {
        //返回name对应的实例
         Class<?> instanceClass = ClassUtils.forName(name, classLoader);
         Assert.isAssignable(type, instanceClass);
         Constructor<?> constructor = instanceClass
               .getDeclaredConstructor(parameterTypes);
        //使用给定的构造函数实例化一个类。
         T instance = (T) BeanUtils.instantiateClass(constructor, args);
         instances.add(instance);
      }
      catch (Throwable ex) {
         throw new IllegalArgumentException(
               "Cannot instantiate " + type + " : " + name, ex);
      }
   }
   return instances;
}
```

​	初始化完成后，调用setInitializers()方法。

```java
//将传入的实现ApplicationContextInitializer的类，加到initializers属性中。
public void setInitializers(
      Collection<? extends ApplicationContextInitializer<?>> initializers) {
   this.initializers = new ArrayList<>();
   this.initializers.addAll(initializers);
}
```



## setListeners()

​	这里调用了setListeners(Collection)和getSpringFactoriesInstances(ApplicationListener.class)。getSpringFactoriesInstances()方法调用，也就是获取了所有实现ApplicationListener的类的实例。

```java
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer
```



```java
public void setListeners(Collection<? extends ApplicationListener<?>> listeners) {
   this.listeners = new ArrayList<>();
   this.listeners.addAll(listeners);
}
```

​	然后将实例放入listeners中。

## deduceMainApplicationClass()

```java
private Class<?> deduceMainApplicationClass() {
   try {
     //获取当前线程的执行栈。
      StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
      for (StackTraceElement stackTraceElement : stackTrace) {
        //获取栈中哪个类包含main方法。
         if ("main".equals(stackTraceElement.getMethodName())) {
            return Class.forName(stackTraceElement.getClassName());
         }
      }
   }
   catch (ClassNotFoundException ex) {
      // Swallow and continue
   }
   return null;
}
```

