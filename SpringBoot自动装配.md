# Spring Boot自动装配（一）

**注：基于SpringBoot2.x版本以上**

## @EnableAutoConfiguration

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}
```

​	其中**AutoConfigurationImportSelector**就是具体自动装配的实现类，实现了**DeferredImportSelector**接口，具体的实现逻辑均在**selectImports(****)**方法中。

```java
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    if (!this.isEnabled(annotationMetadata)) {
        return NO_IMPORTS;
    } else {
      	//加载自动装配的元信息。
        AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);
      	//获取@EnableAutoConfiguration标注类的元信息。
        AnnotationAttributes attributes = this.getAttributes(annotationMetadata);
      	//获取自动装配类的候选类名称。
        List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);
      	//去重，排除重复的configurations(为什么会重复呢？)
        configurations = this.removeDuplicates(configurations);
      	//获取自动装配组件的排除名单。
        Set<String> exclusions = this.getExclusions(annotationMetadata, attributes);
      	//检查排除类是否合法
        this.checkExcludedClasses(configurations, exclusions);
      	//移除所有的排除名单
        configurations.removeAll(exclusions);
      	//进行过滤，autoConfigurationMetadata充当过滤条件。
        configurations = this.filter(configurations, autoConfigurationMetadata);
      	//自动装配的导入事件
        this.fireAutoConfigurationImportEvents(configurations, exclusions);
        return StringUtils.toStringArray(configurations);
    }
}
```

### 疑问：

- 如何装配组件，装配哪些组件？
- 如何排除组件？

从所有的方法，一步步的分析。

### AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);

```java
public static AutoConfigurationMetadata loadMetadata(ClassLoader classLoader) {
    return loadMetadata(classLoader, "META-INF/spring-autoconfigure-metadata.properties");
}
```

可以从代码中看到，它加载的是**spring-autoconfigure-metadata.properties**

由于文件内容过多，这里仅贴了部分。

```java
org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration=
org.springframework.boot.autoconfigure.web.client.RestTemplateAutoConfiguration.AutoConfigureAfter=org.springframework.boot.autoconfigure.http.HttpMessageConvertersAutoConfiguration
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseReactiveDataAutoConfiguration.Configuration=
org.springframework.boot.autoconfigure.data.cassandra.CassandraReactiveDataAutoConfiguration.ConditionalOnClass=com.datastax.driver.core.Cluster,org.springframework.data.cassandra.core.ReactiveCassandraTemplate,reactor.core.publisher.Flux
org.springframework.boot.autoconfigure.data.solr.SolrRepositoriesAutoConfiguration.ConditionalOnClass=org.apache.solr.client.solrj.SolrClient,org.springframework.data.solr.repository.SolrRepository
org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration=
org.springframework.boot.autoconfigure.reactor.core.ReactorCoreAutoConfiguration=
org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration.Configuration=
org.springframework.boot.autoconfigure.jersey.JerseyAutoConfiguration.AutoConfigureBefore=org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration
org.springframework.boot.autoconfigure.jms.artemis.ArtemisXAConnectionFactoryConfiguration=
org.springframework.boot.autoconfigure.h2.H2ConsoleAutoConfiguration.Configuration=
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchRepositoriesAutoConfiguration=
```

```java
static AutoConfigurationMetadata loadMetadata(ClassLoader classLoader, String path) {
    try {
        Enumeration<URL> urls = classLoader != null ? classLoader.getResources(path) : ClassLoader.getSystemResources(path);
        Properties properties = new Properties();

        while(urls.hasMoreElements()) {
            properties.putAll(PropertiesLoaderUtils.loadProperties(new UrlResource((URL)urls.nextElement())));
        }

        return loadMetadata(properties);
    } catch (IOException var4) {
        throw new IllegalArgumentException("Unable to load @ConditionalOnClass location [" + path + "]", var4);
    }
}
```

```java
static AutoConfigurationMetadata loadMetadata(Properties properties) {
    return new AutoConfigurationMetadataLoader.PropertiesAutoConfigurationMetadata(properties);
}
```

最终将所有的配置信息存到PropertiesAutoConfigurationMetadata.properties属性中。

### this.getAttributes(annotationMetadata);

```java
protected AnnotationAttributes getAttributes(AnnotationMetadata metadata) {
    String name = this.getAnnotationClass().getName();
    AnnotationAttributes attributes = AnnotationAttributes.fromMap(metadata.getAnnotationAttributes(name, true));
    Assert.notNull(attributes, () -> {
        return "No auto-configuration attributes found. Is " + metadata.getClassName() + " annotated with " + ClassUtils.getShortName(name) + "?";
    });
    return attributes;
}
```



### this.getCandidateConfigurations(annotationMetadata, attributes);



```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
    return configurations;
}
```

可以看出，实际上执行的方法为**SpringFactoriesLoader.loadFactoryNames()**方法。

```java
public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
    String factoryClassName = factoryClass.getName();
    return (List)loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
}

 private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
        MultiValueMap<String, String> result = (MultiValueMap)cache.get(classLoader);
        if (result != null) {
            return result;
        } else {
            try {
                Enumeration<URL> urls = classLoader != null ? classLoader.getResources("META-INF/spring.factories") : ClassLoader.getSystemResources("META-INF/spring.factories");
                LinkedMultiValueMap result = new LinkedMultiValueMap();

                while(urls.hasMoreElements()) {
                    URL url = (URL)urls.nextElement();
                    UrlResource resource = new UrlResource(url);
                    Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                    Iterator var6 = properties.entrySet().iterator();

                    while(var6.hasNext()) {
                        Entry<?, ?> entry = (Entry)var6.next();
                        List<String> factoryClassNames = Arrays.asList(StringUtils.commaDelimitedListToStringArray((String)entry.getValue()));
                        result.addAll((String)entry.getKey(), factoryClassNames);
                    }
                }

                cache.put(classLoader, result);
                return result;
            } catch (IOException var9) {
                throw new IllegalArgumentException("Unable to load factories from location [META-INF/spring.factories]", var9);
            }
        }
    }
```

SpringFactoriesLoader是Spring Framework工厂机制的加载器。loadFactoryNames(Class,ClassLoader)方法加载原理如下：

1. 搜索指定ClassLoader下所有的META-INF/Spring.factories资源内容。
2. 将一个或多个META-INF/Spring.factories资源内容做鱼Properties文件读取，合并为一个key为接口的全类名，Value为map，作为loadSpringFactories(ClassLoader)方法的返回值。
3. 再从上一步返回的map中查找并返回指定类名所映射的实现类的全类名列表。

下面是部分Spring.factories资源内容：

```java
# Failure analyzers
org.springframework.boot.diagnostics.FailureAnalyzer=\
org.springframework.boot.autoconfigure.diagnostics.analyzer.NoSuchBeanDefinitionFailureAnalyzer,\
org.springframework.boot.autoconfigure.jdbc.DataSourceBeanCreationFailureAnalyzer,\
org.springframework.boot.autoconfigure.jdbc.HikariDriverConfigurationFailureAnalyzer,\
org.springframework.boot.autoconfigure.session.NonUniqueSessionRepositoryFailureAnalyzer

# Template availability providers
org.springframework.boot.autoconfigure.template.TemplateAvailabilityProvider=\
org.springframework.boot.autoconfigure.freemarker.FreeMarkerTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.mustache.MustacheTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.groovy.template.GroovyTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.thymeleaf.ThymeleafTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.web.servlet.JspTemplateAvailabilityProvider
```

 ![%{~%0`96AM%TQ6V0WBPM@~P](%{~%0`96AM%TQ6V0WBPM@~P.png)

可以看到，EnableAutoConfiguration对应的实现全类名有110个。下面展示的仅有部分。

```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.cloud.CloudAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
```

### this.removeDuplicates(configurations);

利用set不可重复性去重。

```java
protected final <T> List<T> removeDuplicates(List<T> list) {
    return new ArrayList(new LinkedHashSet(list));
}
```

### this.getExclusions(annotationMetadata, attributes);

排除自动装配组件。

```java
protected Set<String> getExclusions(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    Set<String> excluded = new LinkedHashSet();
  	//将exclude值转换为list
    excluded.addAll(this.asList(attributes, "exclude"));
  	//将excludeName值转换为list
    excluded.addAll(Arrays.asList(attributes.getStringArray("excludeName")));
    excluded.addAll(this.getExcludeAutoConfigurationsProperty());
    return excluded;
}

//将spring.autoconfigure.exclude配置项值累加到exclude中。
private List<String> getExcludeAutoConfigurationsProperty() {
        if (this.getEnvironment() instanceof ConfigurableEnvironment) {
            Binder binder = Binder.get(this.getEnvironment());
            return (List)binder.bind("spring.autoconfigure.exclude", String[].class).map(Arrays::asList).orElse(Collections.emptyList());
        } else {
            String[] excludes = (String[])this.getEnvironment().getProperty("spring.autoconfigure.exclude", String[].class);
            return excludes != null ? Arrays.asList(excludes) : Collections.emptyList();
        }
    }
```

### this.checkExcludedClasses(configurations, exclusions);

检查排除类是否合法

```java
private void checkExcludedClasses(List<String> configurations, Set<String> exclusions) {
    List<String> invalidExcludes = new ArrayList(exclusions.size());
    Iterator var4 = exclusions.iterator();

    while(var4.hasNext()) {
        String exclusion = (String)var4.next();
        if (ClassUtils.isPresent(exclusion, this.getClass().getClassLoader()) && !configurations.contains(exclusion)) {
            invalidExcludes.add(exclusion);
        }
    }
	//当排除类存在当前ClassLoader且不在自动装配候选类名单中。
    if (!invalidExcludes.isEmpty()) {
        this.handleInvalidExcludes(invalidExcludes);
    }

}
```

```java
protected void handleInvalidExcludes(List<String> invalidExcludes) {
    StringBuilder message = new StringBuilder();
    Iterator var3 = invalidExcludes.iterator();

    while(var3.hasNext()) {
        String exclude = (String)var3.next();
        message.append("\t- ").append(exclude).append(String.format("%n"));
    }

    throw new IllegalStateException(String.format("The following classes could not be excluded because they are not auto-configuration classes:%n%s", message));
}
```

### this.filter(configurations, autoConfigurationMetadata)

再次进行过滤

```java
private List<String> filter(List<String> configurations, AutoConfigurationMetadata autoConfigurationMetadata) {
    long startTime = System.nanoTime();
    String[] candidates = StringUtils.toStringArray(configurations);
    boolean[] skip = new boolean[candidates.length];
    boolean skipped = false;
  	//这里参照前面讲的，获取Spring.factories中AutoConfigurationImportFilter内容。
    Iterator var8 = this.getAutoConfigurationImportFilters().iterator();

    while(var8.hasNext()) {
        AutoConfigurationImportFilter filter = (AutoConfigurationImportFilter)var8.next();
        this.invokeAwareMethods(filter);
        boolean[] match = filter.match(candidates, autoConfigurationMetadata);

        for(int i = 0; i < match.length; ++i) {
            if (!match[i]) {
                skip[i] = true;
                skipped = true;
            }
        }
    }

    if (!skipped) {
        return configurations;
    } else {
        List<String> result = new ArrayList(candidates.length);

        int numberFiltered;
        for(numberFiltered = 0; numberFiltered < candidates.length; ++numberFiltered) {
            if (!skip[numberFiltered]) {
                result.add(candidates[numberFiltered]);
            }
        }

        if (logger.isTraceEnabled()) {
            numberFiltered = configurations.size() - result.size();
            logger.trace("Filtered " + numberFiltered + " auto configuration class in " + TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime) + " ms");
        }

        return new ArrayList(result);
    }
}
```

```java
protected List<AutoConfigurationImportFilter> getAutoConfigurationImportFilters() {
    return SpringFactoriesLoader.loadFactories(AutoConfigurationImportFilter.class, this.beanClassLoader);
}
```

可以看到默认配置中仅有一处声明。
```java
# Auto Configuration Import Filters
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
org.springframework.boot.autoconfigure.condition.OnClassCondition
```


```java
public boolean[] match(String[] autoConfigurationClasses, AutoConfigurationMetadata autoConfigurationMetadata) {
    ConditionEvaluationReport report = this.getConditionEvaluationReport();
    ConditionOutcome[] outcomes = this.getOutcomes(autoConfigurationClasses, autoConfigurationMetadata);
    boolean[] match = new boolean[outcomes.length];
    for(int i = 0; i < outcomes.length; ++i) {
        match[i] = outcomes[i] == null || outcomes[i].isMatch();
        if (!match[i] && outcomes[i] != null) {
            this.logOutcome(autoConfigurationClasses[i], outcomes[i]);
            if (report != null) {
                report.recordConditionEvaluation(autoConfigurationClasses[i], this, outcomes[i]);
            }
        }
    }
    return match;
}
```

```java
private ConditionOutcome[] getOutcomes(String[] autoConfigurationClasses, AutoConfigurationMetadata autoConfigurationMetadata) {
    int split = autoConfigurationClasses.length / 2;
    OnClassCondition.OutcomesResolver firstHalfResolver = this.createOutcomesResolver(autoConfigurationClasses, 0, split, autoConfigurationMetadata);
    OnClassCondition.OutcomesResolver secondHalfResolver = new OnClassCondition.StandardOutcomesResolver(autoConfigurationClasses, split, autoConfigurationClasses.length, autoConfigurationMetadata, this.beanClassLoader);
    ConditionOutcome[] secondHalf = secondHalfResolver.resolveOutcomes();
    ConditionOutcome[] firstHalf = firstHalfResolver.resolveOutcomes();
    ConditionOutcome[] outcomes = new ConditionOutcome[autoConfigurationClasses.length];
    System.arraycopy(firstHalf, 0, outcomes, 0, firstHalf.length);
    System.arraycopy(secondHalf, 0, outcomes, split, secondHalf.length);
    return outcomes;
}
```

建议：这里去自己看看源码，比较绕。核心代码为

```java
private ConditionOutcome[] getOutcomes(String[] autoConfigurationClasses,
      int start, int end, AutoConfigurationMetadata autoConfigurationMetadata) {
   ConditionOutcome[] outcomes = new ConditionOutcome[end - start];
   for (int i = start; i < end; i++) {
      String autoConfigurationClass = autoConfigurationClasses[i];
      Set<String> candidates = autoConfigurationMetadata
            .getSet(autoConfigurationClass, "ConditionalOnClass");
      if (candidates != null) {
         outcomes[i - start] = getOutcome(candidates);
      }
   }
   return outcomes;
}
```

​	也就是说，能否自动装配，取决于其ConditionalClass关联的类是否存在，从而过滤掉那些不满足依赖的自动装配类。

​	还有一个疑问：为什么要用两个线程去做这个事情？

​	doc中的解释如下：

```java
// Split the work and perform half in a background thread. Using a single
// additional thread seems to offer the best performance. More threads make
// things worse
```

​	分开工作，在后台线程中执行一半。使用一个额外的线程似乎能提供最好的性能。更多的线程会让事情变得更糟

### this.fireAutoConfigurationImportEvents(configurations, exclusions);

这时，configurations就是我们实际自动装配的类。

```java
//加载自动装配的监听器，用来记录自动装配的条件评估详情。
private void fireAutoConfigurationImportEvents(List<String> configurations,
			Set<String> exclusions) {
		List<AutoConfigurationImportListener> listeners = getAutoConfigurationImportListeners();
		if (!listeners.isEmpty()) {
			AutoConfigurationImportEvent event = new AutoConfigurationImportEvent(this,
					configurations, exclusions);
			for (AutoConfigurationImportListener listener : listeners) {
				invokeAwareMethods(listener);
				listener.onAutoConfigurationImportEvent(event);
			}
		}
	}
```

```java
protected List<AutoConfigurationImportListener> getAutoConfigurationImportListeners() {
   return SpringFactoriesLoader.loadFactories(AutoConfigurationImportListener.class,
         this.beanClassLoader);
}
```

最后，将需要自动装配的类传出去，这样自动装配需要装配哪些类，我们就已经知道了。