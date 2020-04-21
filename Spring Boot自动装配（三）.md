# Spring Boot自动装配（三）

​	在EnableAutoConfiguration类中，还存在一个注解**AutoConfigurationPackage**

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
//将标注类所在的package添加至BasePacks中，为后续扫描组件提供来源。
public @interface AutoConfigurationPackage {
	
}
```

​	同样的套路，我们去分析**AutoConfigurationPackages.Registrar.class**

```java
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

   @Override
   public void registerBeanDefinitions(AnnotationMetadata metadata,
         BeanDefinitionRegistry registry) {
      register(registry, new PackageImport(metadata).getPackageName());
   }

   @Override
   public Set<Object> determineImports(AnnotationMetadata metadata) {
      return Collections.singleton(new PackageImport(metadata));
   }

}
```

```java
public static void register(BeanDefinitionRegistry registry, String... packageNames) {
  //判断org.springframework.boot.autoconfigure.AutoConfigurationPackages是否已经被注册过。
  //Bean是一个常量为AutoConfigurationPackages.class.getName()；
  //这里是考虑到，@EnableAutoConfiguration被标注在多个类上，这样，Registrar就会被多次@Import。
  if (registry.containsBeanDefinition(BEAN)) {
   	  //直接从容器中拿到bean
      BeanDefinition beanDefinition = registry.getBeanDefinition(BEAN);
     //获取constructorArgumentValues,并添加包路径。
      ConstructorArgumentValues constructorArguments = beanDefinition
            .getConstructorArgumentValues();
      constructorArguments.addIndexedArgumentValue(0,
            addBasePackages(constructorArguments, packageNames));
   }
   else {
     //初始化beanDefinition,实例为BasePackages
      GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
      beanDefinition.setBeanClass(BasePackages.class);
     //获取constructorArgumentValues，并添加包路径。
      beanDefinition.getConstructorArgumentValues().addIndexedArgumentValue(0,
            packageNames);
      beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
     //将初始化的beanDefinition注册到Spring容器中。
      registry.registerBeanDefinition(BEAN, beanDefinition注册到Spring容器中。);
   }
}
```


​	在初始化beanDefinition中，beanDefinition.setBeanClass(**BasePackages.class**);

```java
static final class BasePackages {

   private final List<String> packages;

   private boolean loggedBasePackageInfo;
   BasePackages(String... names) {
      List<String> packages = new ArrayList<>();
      for (String name : names) {
         if (StringUtils.hasText(name)) {
            packages.add(name);
         }
      }
      this.packages = packages;
   }

   public List<String> get() {
      if (!this.loggedBasePackageInfo) {
         if (this.packages.isEmpty()) {
            if (logger.isWarnEnabled()) {
               logger.warn("@EnableAutoConfiguration was declared on a class "
                     + "in the default package. Automatic @Repository and "
                     + "@Entity scanning is not enabled.");
            }
         }
         else {
            if (logger.isDebugEnabled()) {
               String packageNames = StringUtils
                     .collectionToCommaDelimitedString(this.packages);
               logger.debug("@EnableAutoConfiguration was declared on a class "
                     + "in the package '" + packageNames
                     + "'. Automatic @Repository and @Entity scanning is "
                     + "enabled.");
            }
         }
         this.loggedBasePackageInfo = true;
      }
      return this.packages;
   }

}
```
​	首次构造时，将当前标注类所在的package作为参数传入，后续更改时，获取ConstructArgumentValues，追加其中。后续可以根据get()方法直接获取packages。

​	应用实例：AutoConfigurationPackages#get()

```java
public static List<String> get(BeanFactory beanFactory) {
   try {
      return beanFactory.getBean(BEAN, BasePackages.class).get();
   }
   catch (NoSuchBeanDefinitionException ex) {
      throw new IllegalStateException(
            "Unable to retrieve @EnableAutoConfiguration base packages");
   }
}
```

## 条件化自动装配

​	在AutoConfigurationImportSelector类中，最后过滤不满足自动装配条件时，使用的是OnClassCondition类，而在@ConditionalOnClass中，则标注了@Conditional(OnClassCondition.class)。

​	·所有的条件注解，均基于@Conditional实现。

### Class条件注解

#### @ConditionalOnClass
```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)
public @interface ConditionalOnClass {
  ...
}
```
#### @ConditionalOnMissingClass

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)
public @interface ConditionalOnMissingClass {

   /**
    * The names of the classes that must not be present.
    * @return the names of the classes that must not be present
    */
   String[] value() default {};

}
```

​	两者实现类均为**OnClassCondition**

```java
@Override
public ConditionOutcome getMatchOutcome(ConditionContext context,
      AnnotatedTypeMetadata metadata) {
   ClassLoader classLoader = context.getClassLoader();
   ConditionMessage matchMessage = ConditionMessage.empty();
   List<String> onClasses = getCandidates(metadata, ConditionalOnClass.class);
   if (onClasses != null) {
      List<String> missing = getMatches(onClasses, MatchType.MISSING, classLoader);
      if (!missing.isEmpty()) {
         return ConditionOutcome
               .noMatch(ConditionMessage.forCondition(ConditionalOnClass.class)
                     .didNotFind("required class", "required classes")
                     .items(Style.QUOTE, missing));
      }
     //ConditionalOnClass
      matchMessage = matchMessage.andCondition(ConditionalOnClass.class)
            .found("required class", "required classes").items(Style.QUOTE,
                  getMatches(onClasses, MatchType.PRESENT, classLoader));
   }
   List<String> onMissingClasses = getCandidates(metadata,
         ConditionalOnMissingClass.class);
   if (onMissingClasses != null) {
      List<String> present = getMatches(onMissingClasses, MatchType.PRESENT,
            classLoader);
      if (!present.isEmpty()) {
         return ConditionOutcome.noMatch(
           	//ConditionalOnMissingClass
               ConditionMessage.forCondition(ConditionalOnMissingClass.class)
                     .found("unwanted class", "unwanted classes")
                     .items(Style.QUOTE, present));
      }
      matchMessage = matchMessage.andCondition(ConditionalOnMissingClass.class)
            .didNotFind("unwanted class", "unwanted classes").items(Style.QUOTE,
                  getMatches(onMissingClasses, MatchType.MISSING, classLoader));
   }
   return ConditionOutcome.match(matchMessage);
}
```

### Bean条件注解

#### @ConditionalOnBean

```
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnBeanCondition.class)
public @interface ConditionalOnBean {
	...
}
```

#### @ConditionalOnMissingBean

```
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnBeanCondition.class)
public @interface ConditionalOnMissingBean {
	...
}
```

​	两者实现类均为**OnBeanCondition.class**

​	这里主要讲一下ConditionalOnBean实现。其它类似。

```java
@Override
public ConditionOutcome getMatchOutcome(ConditionContext context,
      AnnotatedTypeMetadata metadata) {
   ConditionMessage matchMessage = ConditionMessage.empty();
  //ConditionalOnBean
   if (metadata.isAnnotated(ConditionalOnBean.class.getName())) {
     //包装成BeanSearchSpec，在包装时，会将标注类上的所有元信息转成BeanSearchSpec的属性。
      BeanSearchSpec spec = new BeanSearchSpec(context, metadata,
            ConditionalOnBean.class);
     //核心方法
      MatchResult matchResult = getMatchingBeans(context, spec);
      if (!matchResult.isAllMatched()) {
         String reason = createOnBeanNoMatchReason(matchResult);
         return ConditionOutcome.noMatch(ConditionMessage
               .forCondition(ConditionalOnBean.class, spec).because(reason));
      }
      matchMessage = matchMessage.andCondition(ConditionalOnBean.class, spec)
            .found("bean", "beans")
            .items(Style.QUOTE, matchResult.getNamesOfAllMatches());
   }
  //ConditionalOnSingleCandidate
   if (metadata.isAnnotated(ConditionalOnSingleCandidate.class.getName())) {
      BeanSearchSpec spec = new SingleCandidateBeanSearchSpec(context, metadata,
            ConditionalOnSingleCandidate.class);
      MatchResult matchResult = getMatchingBeans(context, spec);
      if (!matchResult.isAllMatched()) {
         return ConditionOutcome.noMatch(ConditionMessage
               .forCondition(ConditionalOnSingleCandidate.class, spec)
               .didNotFind("any beans").atAll());
      }
      else if (!hasSingleAutowireCandidate(context.getBeanFactory(),
            matchResult.getNamesOfAllMatches(),
            spec.getStrategy() == SearchStrategy.ALL)) {
         return ConditionOutcome.noMatch(ConditionMessage
               .forCondition(ConditionalOnSingleCandidate.class, spec)
               .didNotFind("a primary bean from beans")
               .items(Style.QUOTE, matchResult.getNamesOfAllMatches()));
      }
      matchMessage = matchMessage
            .andCondition(ConditionalOnSingleCandidate.class, spec)
            .found("a primary bean from beans")
            .items(Style.QUOTE, matchResult.namesOfAllMatches);
   }
  //ConditionalOnMissingBean
   if (metadata.isAnnotated(ConditionalOnMissingBean.class.getName())) {
      BeanSearchSpec spec = new BeanSearchSpec(context, metadata,
            ConditionalOnMissingBean.class);
      MatchResult matchResult = getMatchingBeans(context, spec);
      if (matchResult.isAnyMatched()) {
         String reason = createOnMissingBeanNoMatchReason(matchResult);
         return ConditionOutcome.noMatch(ConditionMessage
               .forCondition(ConditionalOnMissingBean.class, spec)
               .because(reason));
      }
      matchMessage = matchMessage.andCondition(ConditionalOnMissingBean.class, spec)
            .didNotFind("any beans").atAll();
   }
   return ConditionOutcome.match(matchMessage);
}
```



```java
private MatchResult getMatchingBeans(ConditionContext context, BeanSearchSpec beans) {
   ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
  	//将BeanFactory切换成ConfigurableListableBeanFactory
   if (beans.getStrategy() == SearchStrategy.ANCESTORS) {
      BeanFactory parent = beanFactory.getParentBeanFactory();
      Assert.isInstanceOf(ConfigurableListableBeanFactory.class, parent,
            "Unable to use SearchStrategy.PARENTS");
      beanFactory = (ConfigurableListableBeanFactory) parent;
   }
   MatchResult matchResult = new MatchResult();
   boolean considerHierarchy = beans.getStrategy() != SearchStrategy.CURRENT;
  	//获取标注的ignoredTypes，然后调用getBeanNamesForType()
   List<String> beansIgnoredByType = getNamesOfBeansIgnoredByType(
         beans.getIgnoredTypes(), beanFactory, context, considerHierarchy);
   for (String type : beans.getTypes()) {
      Collection<String> typeMatches = getBeanNamesForType(beanFactory, type,
            context.getClassLoader(), considerHierarchy);
      typeMatches.removeAll(beansIgnoredByType);
      if (typeMatches.isEmpty()) {
         matchResult.recordUnmatchedType(type);
      }
      else {
         matchResult.recordMatchedType(type, typeMatches);
      }
   }
   for (String annotation : beans.getAnnotations()) {
      List<String> annotationMatches = Arrays
            .asList(getBeanNamesForAnnotation(beanFactory, annotation,
                  context.getClassLoader(), considerHierarchy));
      annotationMatches.removeAll(beansIgnoredByType);
      if (annotationMatches.isEmpty()) {
         matchResult.recordUnmatchedAnnotation(annotation);
      }
      else {
         matchResult.recordMatchedAnnotation(annotation, annotationMatches);
      }
   }
   for (String beanName : beans.getNames()) {
      if (!beansIgnoredByType.contains(beanName)
            && containsBean(beanFactory, beanName, considerHierarchy)) {
         matchResult.recordMatchedName(beanName);
      }
      else {
         matchResult.recordUnmatchedName(beanName);
      }
   }
   return matchResult;
}
```



```java
private List<String> getNamesOfBeansIgnoredByType(List<String> ignoredTypes,
      ListableBeanFactory beanFactory, ConditionContext context,
      boolean considerHierarchy) {
   List<String> beanNames = new ArrayList<>();
   for (String ignoredType : ignoredTypes) {
      beanNames.addAll(getBeanNamesForType(beanFactory, ignoredType,
            context.getClassLoader(), considerHierarchy));
   }
   return beanNames;
}
```



```java
//该方法核心为collectBeanNamesForType();
private Collection<String> getBeanNamesForType(ListableBeanFactory beanFactory,
			String type, ClassLoader classLoader, boolean considerHierarchy)
			throws LinkageError {
		try {
			Set<String> result = new LinkedHashSet<>();
			collectBeanNamesForType(result, beanFactory,
					ClassUtils.forName(type, classLoader), considerHierarchy);
			return result;
		}
		catch (ClassNotFoundException | NoClassDefFoundError ex) {
			return Collections.emptySet();
		}
	}
```



```java
//
private void collectBeanNamesForType(Set<String> result,
      ListableBeanFactory beanFactory, Class<?> type, boolean considerHierarchy) {
   result.addAll(BeanTypeRegistry.get(beanFactory).getNamesForType(type));
   if (considerHierarchy && beanFactory instanceof HierarchicalBeanFactory) {
      BeanFactory parent = ((HierarchicalBeanFactory) beanFactory)
            .getParentBeanFactory();
      if (parent instanceof ListableBeanFactory) {
         collectBeanNamesForType(result, (ListableBeanFactory) parent, type,
               considerHierarchy);
      }
   }
}
```

BeanTypeRegistry#getNamesForType()

```java
Set<String> getNamesForType(Class<?> type) {
   updateTypesIfNecessary();
   return this.beanTypes.entrySet().stream()
         .filter((entry) -> entry.getValue() != null
               && type.isAssignableFrom(entry.getValue()))
         .map(Map.Entry::getKey)
         .collect(Collectors.toCollection(LinkedHashSet::new));
}
```

BeanTypeRegistry#updateTypesIfNecessary()

```java
private void updateTypesIfNecessary() {
   this.beanFactory.getBeanNamesIterator().forEachRemaining((name) -> {
      if (!this.beanTypes.containsKey(name)) {
        //更新beanTypes内容。
         addBeanType(name);
      }
      else {
         updateBeanType(name);
      }
   });
}
```
BeanTypeRegistry#addBeanType()

```java
private void addBeanType(String name) {
   if (this.beanFactory.containsSingleton(name)) {
      this.beanTypes.put(name, this.beanFactory.getType(name));
   }
   else if (!this.beanFactory.isAlias(name)) {
      addBeanTypeForNonAliasDefinition(name);
   }
}
```


AbstractBeanFactory#getType()


```java
public Class<?> getType(String name) throws NoSuchBeanDefinitionException {
   String beanName = transformedBeanName(name);

   // Check manually registered singletons.
  //检查手动注册的单例
   Object beanInstance = getSingleton(beanName, false);
   if (beanInstance != null && beanInstance.getClass() != NullBean.class) {
      if (beanInstance instanceof FactoryBean && !BeanFactoryUtils.isFactoryDereference(name)) {
         return getTypeForFactoryBean((FactoryBean<?>) beanInstance);
      }
      else {
         return beanInstance.getClass();
      }
   }

   // No singleton instance found -> check bean definition.
  //没有找到单例实例->检查beanDefinition
   BeanFactory parentBeanFactory = getParentBeanFactory();
   if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
      // No bean definition found in this factory -> delegate to parent.
      return parentBeanFactory.getType(originalBeanName(name));
   }

   RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);

   // Check decorated bean definition, if any: We assume it'll be easier
   // to determine the decorated bean's type than the proxy's type.
  //检查装饰类，如果存在，我们假设他比代理类更容易被装饰。
   BeanDefinitionHolder dbd = mbd.getDecoratedDefinition();
   if (dbd != null && !BeanFactoryUtils.isFactoryDereference(name)) {
      RootBeanDefinition tbd = getMergedBeanDefinition(dbd.getBeanName(), dbd.getBeanDefinition(), mbd);
      Class<?> targetClass = predictBeanType(dbd.getBeanName(), tbd);
      if (targetClass != null && !FactoryBean.class.isAssignableFrom(targetClass)) {
         return targetClass;
      }
   }

   Class<?> beanClass = predictBeanType(beanName, mbd);

   // Check bean class whether we're dealing with a FactoryBean.
   if (beanClass != null && FactoryBean.class.isAssignableFrom(beanClass)) {
      if (!BeanFactoryUtils.isFactoryDereference(name)) {
         // If it's a FactoryBean, we want to look at what it creates, not at the factory class.
         return getTypeForFactoryBean(beanName, mbd);
      }
      else {
         return beanClass;
      }
   }
   else {
      return (!BeanFactoryUtils.isFactoryDereference(name) ? beanClass : null);
   }
}
```
​	@ConditionalOnBean和@ConditionalOnMissingBean基于BeanDefinition进行名称或类型的匹配。