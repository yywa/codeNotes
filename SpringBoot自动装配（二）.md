# SpringBoot自动装配（二）

​	上文写到，最后会配置一个监听器，用于记录自动装配的条件评估详情。现在我们自定义一个监听器，去测试下。首先需要实现一个**AutoConfigurationImportListener**接口。

```java
public class DefaultAutoConfigurationImportListener implements AutoConfigurationImportListener {
    @Override
    public void onAutoConfigurationImportEvent(AutoConfigurationImportEvent event) {
        //获取当前classLoader
        ClassLoader classLoader = event.getClass().getClassLoader();
        //实际自动装配的配置类
        List<String> candidateConfigurations = event.getCandidateConfigurations();
        //排除的自动装配名单
        Set<String> exclusions = event.getExclusions();
        System.out.printf("实际装配 为：%s，排除的自动装配为：%s\t\n", candidateConfigurations.size(), exclusions.size());
        System.out.println("------------实际装配类----------");
        candidateConfigurations.forEach(System.out::println);
        System.out.println("------------排除装配类----------");
        exclusions.forEach(System.out::println);
    }
}
```

​	然后，在项目resources下定义一个META-INF/spring.factories文件，前面已经说过，

SpringFactoriesLoader会将一个或多个META-INF/spring.factories进行读取。

```java
org.springframework.boot.autoconfigure.AutoConfigurationImportListener=\
com.yyw.listener.DefaultAutoConfigurationImportListener
```
​	修改启动类

```java
@EnableAutoConfiguration(exclude = RedisAutoConfiguration.class)
public class BootStrapApplication {
    public static void main(String[] args) {
        SpringApplication.run(BootStrapApplication.class, args);
    }
}
```

​	执行代码，观察结果。

```
实际装配 为：22，排除的自动装配为：1	
------------实际装配类----------
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration
org.springframework.boot.autoconfigure.http.HttpMessageConvertersAutoConfiguration
org.springframework.boot.autoconfigure.http.codec.CodecsAutoConfiguration
org.springframework.boot.autoconfigure.info.ProjectInfoAutoConfiguration
org.springframework.boot.autoconfigure.jmx.JmxAutoConfiguration
org.springframework.boot.autoconfigure.mail.MailSenderValidatorAutoConfiguration
org.springframework.boot.autoconfigure.validation.ValidationAutoConfiguration
org.springframework.boot.autoconfigure.web.client.RestTemplateAutoConfiguration
org.springframework.boot.autoconfigure.web.embedded.EmbeddedWebServerFactoryCustomizerAutoConfiguration
org.springframework.boot.autoconfigure.web.reactive.ReactiveWebServerFactoryAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.HttpEncodingAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.MultipartAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
org.springframework.boot.autoconfigure.websocket.reactive.WebSocketReactiveAutoConfiguration
org.springframework.boot.autoconfigure.websocket.servlet.WebSocketServletAutoConfiguration
------------排除装配类----------
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
```

**抛出疑问：**

```java
# Auto Configuration Import Listeners
org.springframework.boot.autoconfigure.AutoConfigurationImportListener=\
org.springframework.boot.autoconfigure.condition.ConditionEvaluationReportAutoConfigurationImportListener
```

​	**在源码中，已经有这个默认配置。为什么是我们自定义的配置也会生效？**

​	很简单，因为我们自定义的配置类，资源加载后是一个key->Collection形式，在AutoConfigurationImportSelector#fireAutoConfigurationImportEvents()方法中，这些监听器都会执行。

接着分析：AutoConfigurationImportSelector实现了DeferredImportSelector接口。

```java
public interface DeferredImportSelector extends ImportSelector {

   /**
    * Return a specific import group or {@code null} if no grouping is required.
    * @return the import group class or {@code null}
    */
   @Nullable
   default Class<? extends Group> getImportGroup() {
      return null;
   }
  
  
  interface Group {
  /**
		 * Process the {@link AnnotationMetadata} of the importing @{@link 						Configuration}
		 * class using the specified {@link DeferredImportSelector}.
		 */
		void process(AnnotationMetadata metadata, DeferredImportSelector selector);

		/**
		 * Return the {@link Entry entries} of which class(es) should be imported for 				this
		 * group.
		 */
		Iterable<Entry> selectImports();
	}
}
```

​	该接口新增了getImportGroup()方法，其中DeferredImportSelector.Group接口辅助处理该接口导入的Configuration Class。

​	process()负责二次处理**DeferredImportSelector#selectImports()**方法返回的结果，Group.selectImports()方法负责决定应该导入的Configuration Class.

​	在AutoConfigurationImportSelector类中，有个内部类AutoConfigurationGroup，实现了Group接口。

```java
@Override
//用于缓存导入类名和AnnotationMetadata的键值对象
public void process(AnnotationMetadata annotationMetadata,
      DeferredImportSelector deferredImportSelector) {
   String[] imports = deferredImportSelector.selectImports(annotationMetadata);
   for (String importClassName : imports) {
      this.entries.put(importClassName, annotationMetadata);
   }
}

@Override
//用于自动装配排序。
public Iterable<Entry> selectImports() {
   return sortAutoConfigurations().stream()
         .map((importClassName) -> new Entry(this.entries.get(importClassName),
               importClassName))
         .collect(Collectors.toList());
}
```

```java
private List<String> sortAutoConfigurations() {
   List<String> autoConfigurations = new ArrayList<>(this.entries.keySet());
   if (this.entries.size() <= 1) {
      return autoConfigurations;
   }
  //再次从META-INF/spring-autoconfigure-metadata.properties读取所有配置
   AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
         .loadMetadata(this.beanClassLoader);
  //借助AutoConfigurationSorter.getInPriorityOrder()方法去排序。
   return new AutoConfigurationSorter(getMetadataReaderFactory(),
         autoConfigurationMetadata).getInPriorityOrder(autoConfigurations);
}
```

​	接着分析**AutoConfigurationSorter#getInPriorityOrder()**

```java
public List<String> getInPriorityOrder(Collection<String> classNames) {
   AutoConfigurationClasses classes = new AutoConfigurationClasses(
         this.metadataReaderFactory, this.autoConfigurationMetadata, classNames);
   List<String> orderedClassNames = new ArrayList<>(classNames);
   // Initially sort alphabetically
  //根据字母顺序排序
   Collections.sort(orderedClassNames);
   // Then sort by order
  //根据Order排序
   orderedClassNames.sort((o1, o2) -> {
      int i1 = classes.get(o1).getOrder();
      int i2 = classes.get(o2).getOrder();
      return Integer.compare(i1, i2);
   });
   // Then respect @AutoConfigureBefore @AutoConfigureAfter
  //根据order排序后，再根据AutoAutoConfigureBefore和AutoConfigureAfter进行排序
   orderedClassNames = sortByAnnotation(classes, orderedClassNames);
   return orderedClassNames;
}
```

​	分析**AutoConfigurationClass#getOrder()**方法

```java
private int getOrder() {
  	//判断当前自动装配类是否在META-INF/spring-autoconfigure-metadata.properties中。
   if (wasProcessed()) {
     //读取自动装配类的AutoConfigureOrder值，这个值也在META-INF/spring-autoconfigure-metadata.properties中，如果不存在，则为AutoConfigureOrder.DEFAULT_ORDER.
      return this.autoConfigurationMetadata.getInteger(this.className,
            "AutoConfigureOrder", AutoConfigureOrder.DEFAULT_ORDER);
   }
   Map<String, Object> attributes = getAnnotationMetadata()
         .getAnnotationAttributes(AutoConfigureOrder.class.getName());
   return (attributes != null) ? (Integer) attributes.get("value")
         : AutoConfigureOrder.DEFAULT_ORDER;
}
```

​	分析**AutoConfigurationClass#sortByAnnotation()**方法

```java
private List<String> sortByAnnotation(AutoConfigurationClasses classes,
      List<String> classNames) {
   List<String> toSort = new ArrayList<>(classNames);
   toSort.addAll(classes.getAllNames());
   Set<String> sorted = new LinkedHashSet<>();
   Set<String> processing = new LinkedHashSet<>();
   while (!toSort.isEmpty()) {
      doSortByAfterAnnotation(classes, toSort, sorted, processing, null);
   }
   sorted.retainAll(classNames);
   return new ArrayList<>(sorted);
}
```

```java
//读取META-INF/spring-autoconfigure-metadata.properties中AutoConfigureAfter和AutoConfigureBefore属性，以此排序
private void doSortByAfterAnnotation(AutoConfigurationClasses classes,
      List<String> toSort, Set<String> sorted, Set<String> processing,
      String current) {
   if (current == null) {
      current = toSort.remove(0);
   }
   processing.add(current);
   for (String after : classes.getClassesRequestedAfter(current)) {
      Assert.state(!processing.contains(after),
            "AutoConfigure cycle detected between " + current + " and " + after);
      if (!sorted.contains(after) && toSort.contains(after)) {
         doSortByAfterAnnotation(classes, toSort, sorted, processing, after);
      }
   }
   processing.remove(current);
   sorted.add(current);
}
```

至此，自动装配的类顺序已经排好。