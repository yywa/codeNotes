# Spring源码（二）bean的创建

​	怎么去获取一个Spring Bean？

​	通过BeanFactory定义的getBean()方法，就可以从容器中拿到这个bean的实例。

​	所以，我们从getBean方法为入口，看看Spring在里面做了些什么操作。

```java
//返回指定的bean实例
Object getBean(String name) throws BeansException;
//返回指定bean实例，增加了类型安全机制。
<T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException;
//返回指定bean实例，指定名称和参数。
Object getBean(String name, Object... args) throws BeansException;
```

​	看BeanFactory的抽象实现类AbstractBeanFactory

```java
public <T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException {
		return doGetBean(name, requiredType, null, false);
	}
public Object getBean(String name, Object... args) throws BeansException {
		return doGetBean(name, null, args, false);
	}
public <T> T getBean(String name, @Nullable Class<T> requiredType, @Nullable Object... args)
      throws BeansException {
   return doGetBean(name, requiredType, args, false);
}

```

​	可以看到，无论是指定名称和参数，还是指定类的获取bean方式，均是调用doGetBean()方法。

## AbstractBeanFactory

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
      @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

   //根据指定的名称获取被管理Bean的名称，剥离指定名称中对容器的相关依赖
   //如果指定的是别名，将别名转换为规范的Bean名称
   final String beanName = transformedBeanName(name);
   Object bean;

   // Eagerly check singleton cache for manually registered singletons.
   //先从缓存中取是否已经有被创建过的单态类型的Bean
   //对于单例模式的Bean整个IOC容器中只创建一次，不需要重复创建
   Object sharedInstance = getSingleton(beanName);
   //IOC容器创建单例模式Bean实例对象
   if (sharedInstance != null && args == null) {
      if (logger.isDebugEnabled()) {
         //如果指定名称的Bean在容器中已有单例模式的Bean被创建
         //直接返回已经创建的Bean
         if (isSingletonCurrentlyInCreation(beanName)) {
            logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                  "' that is not fully initialized yet - a consequence of a circular reference");
         }
         else {
            logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
         }
      }
      //获取给定的Bean实例的对象，可以是Bean实例本身，也可以是在FactoryBean的情况下创建的对象。FactoryBean是用来创建对象的工厂bean。
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
   }

   else {
      // Fail if we're already creating this bean instance:
      // We're assumably within a circular reference.
      //缓存没有正在创建的单例模式Bean
      //缓存中已经有已经创建的原型模式Bean
      //但是由于循环引用的问题导致实例化对象失败
      if (isPrototypeCurrentlyInCreation(beanName)) {
         throw new BeanCurrentlyInCreationException(beanName);
      }

      // Check if bean definition exists in this factory.
      //对IOC容器中是否存在指定名称的BeanDefinition进行检查，首先检查是否
      //能在当前的BeanFactory中获取的所需要的Bean，如果不能则委托当前容器
      //的父级容器去查找，如果还是找不到则沿着容器的继承体系向父级容器查找
      BeanFactory parentBeanFactory = getParentBeanFactory();
      //当前容器的父级容器存在，且当前容器中不存在指定名称的Bean
      if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
         // Not found -> check parent.
         //解析指定Bean名称的原始名称
         String nameToLookup = originalBeanName(name);
         if (parentBeanFactory instanceof AbstractBeanFactory) {
            return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                  nameToLookup, requiredType, args, typeCheckOnly);
         }
         else if (args != null) {
            // Delegation to parent with explicit args.
            //委派父级容器根据指定名称和显式的参数查找
            return (T) parentBeanFactory.getBean(nameToLookup, args);
         }
         else {
            // No args -> delegate to standard getBean method.
            //委派父级容器根据指定名称和类型查找
            return parentBeanFactory.getBean(nameToLookup, requiredType);
         }
      }

      //创建的Bean是否需要进行类型验证，一般不需要
      if (!typeCheckOnly) {
         //向容器标记指定的Bean已经被创建
        //这允许Bean工厂优化其缓存，以便重复创建指定的Bean。
         markBeanAsCreated(beanName);
      }

      try {
         //根据指定Bean名称获取其父级的Bean定义
         //主要解决Bean继承时子类合并父类公共属性问题
         final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
         checkMergedBeanDefinition(mbd, beanName, args);

         // Guarantee initialization of beans that the current bean depends on.
         //获取当前Bean所有依赖Bean的名称
         String[] dependsOn = mbd.getDependsOn();
         //如果当前Bean有依赖Bean
         if (dependsOn != null) {
            for (String dep : dependsOn) {
              //判断当前指定的bean是否有循环依赖。
               if (isDependent(beanName, dep)) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
               }
               //递归调用getBean方法，获取当前Bean的依赖Bean
               registerDependentBean(dep, beanName);
               //把被依赖Bean注册给当前依赖的Bean
               getBean(dep);
            }
         }

         // Create bean instance.
         //创建单例模式Bean的实例对象
         if (mbd.isSingleton()) {
            //这里使用了一个匿名内部类，创建Bean实例对象，并且注册给所依赖的对象
            sharedInstance = getSingleton(beanName, () -> {
               try {
                  //创建一个指定Bean实例对象，如果有父级继承，则合并子类和父类的定义
                  return createBean(beanName, mbd, args);
               }
               catch (BeansException ex) {
                  // Explicitly remove instance from singleton cache: It might have been put there
                  // eagerly by the creation process, to allow for circular reference resolution.
                  // Also remove any beans that received a temporary reference to the bean.
                  //显式地从容器单例模式Bean缓存中清除实例对象
                  destroySingleton(beanName);
                  throw ex;
               }
            });
            //获取给定Bean的实例对象
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
         }

         //IOC容器创建原型模式Bean实例对象
         else if (mbd.isPrototype()) {
            // It's a prototype -> create a new instance.
            //原型模式(Prototype)是每次都会创建一个新的对象
            Object prototypeInstance = null;
            try {
               //回调beforePrototypeCreation方法，默认的功能是注册当前创建的原型对象
               beforePrototypeCreation(beanName);
               //创建指定Bean对象实例
               prototypeInstance = createBean(beanName, mbd, args);
            }
            finally {
               //回调afterPrototypeCreation方法，默认的功能告诉IOC容器指定Bean的原型对象不再创建
               afterPrototypeCreation(beanName);
            }
            //获取给定Bean的实例对象
            bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
         }

         //要创建的Bean既不是单例模式，也不是原型模式，则根据Bean定义资源中
         //配置的生命周期范围，选择实例化Bean的合适方法，这种在Web应用程序中
         //比较常用，如：request、session、application等生命周期
         else {
            String scopeName = mbd.getScope();
            final Scope scope = this.scopes.get(scopeName);
            //Bean定义资源中没有配置生命周期范围，则Bean定义不合法
            if (scope == null) {
               throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
            }
            try {
               //这里又使用了一个匿名内部类，获取一个指定生命周期范围的实例
               Object scopedInstance = scope.get(beanName, () -> {
                  beforePrototypeCreation(beanName);
                  try {
                     return createBean(beanName, mbd, args);
                  }
                  finally {
                     afterPrototypeCreation(beanName);
                  }
               });
               //获取给定Bean的实例对象
               bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
            }
            catch (IllegalStateException ex) {
               throw new BeanCreationException(beanName,
                     "Scope '" + scopeName + "' is not active for the current thread; consider " +
                     "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                     ex);
            }
         }
      }
      catch (BeansException ex) {
         cleanupAfterBeanCreationFailure(beanName);
         throw ex;
      }
   }

   // Check if required type matches the type of the actual bean instance.
   //对创建的Bean实例对象进行类型检查
   if (requiredType != null && !requiredType.isInstance(bean)) {
      try {
         T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
         if (convertedBean == null) {
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
         }
         return convertedBean;
      }
      catch (TypeMismatchException ex) {
         if (logger.isDebugEnabled()) {
            logger.debug("Failed to convert bean '" + name + "' to required type '" +
                  ClassUtils.getQualifiedName(requiredType) + "'", ex);
         }
         throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
      }
   }
   return (T) bean;
}
```

### transformedBeanName()

​	为了将所有传入的name，转换成应该获取的规范bean的名称。

```java
protected String transformedBeanName(String name) {
   return canonicalName(BeanFactoryUtils.transformedBeanName(name));
}
```

```java
//返回实例的Bean名称，去掉工厂前缀 &
public static String transformedBeanName(String name) {
   Assert.notNull(name, "'name' must not be null");
   String beanName = name;
   while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
      beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
   }
   return beanName;
}
```

```java
//返回实际的Bean名称。通过别名找到指定的实际bean名称。
public String canonicalName(String name) {
   String canonicalName = name;
   // Handle aliasing...
   String resolvedName;
   do {
      resolvedName = this.aliasMap.get(canonicalName);
      if (resolvedName != null) {
         canonicalName = resolvedName;
      }
   }
   while (resolvedName != null);
   return canonicalName;
}
```

### getSingleton()

```java
public Object getSingleton(String beanName) {
   return getSingleton(beanName, true);
}
```

#### DefaultSingletonBeanRegistry#getSingleton()

```java
//返回给定名称下注册的Bean。
//检查已经实例化的Bean，运行提前创建引用，（解决循环依赖。）
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
  //先从缓存中判断有没有。
   Object singletonObject = this.singletonObjects.get(beanName);
  //当缓存中没有，并且在正在创建的BeanMap中。
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      synchronized (this.singletonObjects) {
         //判断earlySingletonObjects是否有要创建的Bean
         singletonObject = this.earlySingletonObjects.get(beanName);
        //如果earlySingletonObjects不存在，而且还允许提前建立引用。
         if (singletonObject == null && allowEarlyReference) {
           	//从单例工厂缓存中获取。
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
               //直接从缓存中拿到
               singletonObject = singletonFactory.getObject();
              //将获取到的bean添加到earlySingletonObjects缓存中
               this.earlySingletonObjects.put(beanName, singletonObject);
              //并且，从singletonFactories移除。
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
   return singletonObject;
}
```

##### 三层缓存

​	这里有一个问题，Spring为什么要用三层map解决循环依赖？

​	先检查已经实例化的缓存singletonObjects，然后再检查正在创建的缓存earlySingletonObjects，如果还是不存在，这里就调用ObjectFactory#getObject()方法进行创建实例。

​	当A依赖B，A创建的时候，B还没有被实例化，那么直接使用ObjectFactory#getObject()。并且Spring提供了BeanPostProcessor#postProcessBeforeInitialization和postProcessAfterInitialization方法，可以为Bean的初始化前后回调。

#### getObjectForBeanInstance()

```java
protected Object getObjectForBeanInstance(
      Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

   // Don't let calling code try to dereference the factory if the bean isn't a factory.
   //如果是一个工厂name，但是beanInstance并不是FactoryBean
   if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
      throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
   }

   // Now we have the bean instance, which may be a normal bean or a FactoryBean.
   // If it's a FactoryBean, we use it to create a bean instance, unless the
   // caller actually wants a reference to the factory.
   //如果Bean实例不是工厂Bean，或者指定名称是容器的解引用，
   //调用者向获取对容器的引用，则直接返回当前的Bean实例
   if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
      return beanInstance;
   }

   //处理指定名称不是容器的解引用，或者根据名称获取的Bean实例对象是一个工厂Bean
   //使用工厂Bean创建一个Bean的实例对象
   Object object = null;
   if (mbd == null) {
      //从Bean工厂缓存中获取给定名称的Bean实例对象
      object = getCachedObjectForFactoryBean(beanName);
   }
   //让Bean工厂生产给定名称的Bean对象实例
   if (object == null) {
      // Return bean instance from factory.
      FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
      // Caches object obtained from FactoryBean if it is a singleton.
      //如果从Bean工厂生产的Bean是单态模式的，则缓存
      if (mbd == null && containsBeanDefinition(beanName)) {
         //返回一个合并的RootBeanDefinition，如果指定的Bean对应于子Bean定义，则遍历父Bean定义。
         mbd = getMergedLocalBeanDefinition(beanName);
      }
      //如果从容器得到Bean定义信息，并且Bean定义信息不是虚构的，
      //则让工厂Bean生产Bean实例对象
      boolean synthetic = (mbd != null && mbd.isSynthetic());
      //调用FactoryBeanRegistrySupport类的getObjectFromFactoryBean方法，
      //实现工厂Bean生产Bean对象实例的过程
      object = getObjectFromFactoryBean(factory, beanName, !synthetic);
   }
   return object;
}
```

#### FactoryBeanRegistrySupport#getObjectFromFactoryBean()

```java
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
   //Bean工厂是单态模式，并且Bean工厂缓存中存在指定名称的Bean实例对象
   if (factory.isSingleton() && containsSingleton(beanName)) {
      //多线程同步，以防止数据不一致
      synchronized (getSingletonMutex()) {
         //直接从Bean工厂缓存中获取指定名称的Bean实例对象
         Object object = this.factoryBeanObjectCache.get(beanName);
         //Bean工厂缓存中没有指定名称的实例对象，则生产该实例对象
         if (object == null) {
            //调用Bean工厂的getObject方法生产指定Bean的实例对象
            object = doGetObjectFromFactoryBean(factory, beanName);
            // Only post-process and store if not put there already during getObject() call above
            // (e.g. because of circular reference processing triggered by custom getBean calls)
           //如果在上面的getObject()调用过程中还没有走到那里，则只进行后处理和存储。
            //（例如，由于自定义的getBean调用触发的循环引用处理）。
            Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
            if (alreadyThere != null) {
               object = alreadyThere;
            }
            else {
             	//如果创建后还需要处理。
               if (shouldPostProcess) {
                  try {
                     object = postProcessObjectFromFactoryBean(object, beanName);
                  }
                  catch (Throwable ex) {
                     throw new BeanCreationException(beanName,
                           "Post-processing of FactoryBean's singleton object failed", ex);
                  }
               }
               //将生产的实例对象添加到Bean工厂缓存中
               this.factoryBeanObjectCache.put(beanName, object);
            }
         }
         return object;
      }
   }
   //调用Bean工厂的getObject方法生产指定Bean的实例对象
   else {
      Object object = doGetObjectFromFactoryBean(factory, beanName);
      if (shouldPostProcess) {
         try {
            object = postProcessObjectFromFactoryBean(object, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
         }
      }
      return object;
   }
}
```

##### doGetObjectFromFactoryBean

```java
//调用Bean工厂的getObject方法生产指定Bean的实例对象
private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
      throws BeanCreationException {

   Object object;
   try {
      if (System.getSecurityManager() != null) {
        //获取安全上下文。
         AccessControlContext acc = getAccessControlContext();
         try {
            //实现PrivilegedExceptionAction接口的匿名内置类
            //根据JVM检查权限，然后决定BeanFactory创建实例对象
            object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () ->
                  factory.getObject(), acc);
         }
         catch (PrivilegedActionException pae) {
            throw pae.getException();
         }
      }
      else {
         //调用BeanFactory接口实现类的创建对象方法
         object = factory.getObject();
      }
   }
   catch (FactoryBeanNotInitializedException ex) {
      throw new BeanCurrentlyInCreationException(beanName, ex.toString());
   }
   catch (Throwable ex) {
      throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
   }

   // Do not accept a null value for a FactoryBean that's not fully
   // initialized yet: Many FactoryBeans just return null then.
   //创建出来的实例对象为null，或者因为单态对象正在创建而返回null
   if (object == null) {
      if (isSingletonCurrentlyInCreation(beanName)) {
         throw new BeanCurrentlyInCreationException(
               beanName, "FactoryBean which is currently in creation returned null from getObject");
      }
      object = new NullBean();
   }
   return object;
}
```

##### postProcessObjectFromFactoryBean()

​	对从FactoryBean中获得的给定对象进行后处理。所得到的对象将被暴露在Bean引用中。
默认的实现只是简单地返回给定对象的原样。子类可以重写这一方法。

### registerDependentBean()

```java
//为指定的Bean注入依赖的Bean
public void registerDependentBean(String beanName, String dependentBeanName) {
   // A quick check for an existing entry upfront, avoiding synchronization...
   //规范bean名称。
   String canonicalName = canonicalName(beanName);
   Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
   if (dependentBeans != null && dependentBeans.contains(dependentBeanName)) {
      return;
   }

   // No entry yet -> fully synchronized manipulation of the dependentBeans Set
   //多线程同步，保证容器内数据的一致性
   //bean的映射，bean->	依赖bean的集合
   synchronized (this.dependentBeanMap) {
      //获取给定名称Bean的所有依赖Bean名称
      dependentBeans = this.dependentBeanMap.get(canonicalName);
      if (dependentBeans == null) {
         //为Bean设置依赖Bean信息
         dependentBeans = new LinkedHashSet<>(8);
         this.dependentBeanMap.put(canonicalName, dependentBeans);
      }
      dependentBeans.add(dependentBeanName);
   }
   //
   synchronized (this.dependenciesForBeanMap) {
      Set<String> dependenciesForBean = this.dependenciesForBeanMap.get(dependentBeanName);
      if (dependenciesForBean == null) {
         dependenciesForBean = new LinkedHashSet<>(8);
         this.dependenciesForBeanMap.put(dependentBeanName, dependenciesForBean);
      }
      //依赖bean名称之间的映射。bean名->bean的依赖关系的bean集合。
      dependenciesForBean.add(canonicalName);
   }
}
```

### createBean()

```java
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {

   if (logger.isDebugEnabled()) {
      logger.debug("Creating instance of bean '" + beanName + "'");
   }
   RootBeanDefinition mbdToUse = mbd;

   // Make sure bean class is actually resolved at this point, and
   // clone the bean definition in case of a dynamically resolved Class
   // which cannot be stored in the shared merged bean definition.
   //判断需要创建的Bean是否可以实例化，即是否可以通过当前的类加载器加载
   Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
   if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
      mbdToUse = new RootBeanDefinition(mbd);
      mbdToUse.setBeanClass(resolvedClass);
   }

   // Prepare method overrides.
   //校验和准备Bean中的方法覆盖
   try {
      mbdToUse.prepareMethodOverrides();
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
            beanName, "Validation of method overrides failed", ex);
   }

   try {
      // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
      //如果Bean配置了初始化前和初始化后的处理器，则试图返回一个需要创建Bean的代理对象
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
         return bean;
      }
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
            "BeanPostProcessor before instantiation of bean failed", ex);
   }

   try {
      //创建Bean的入口
      Object beanInstance = doCreateBean(beanName, mbdToUse, args);
      if (logger.isDebugEnabled()) {
         logger.debug("Finished creating instance of bean '" + beanName + "'");
      }
      return beanInstance;
   }
   catch (BeanCreationException ex) {
      // A previously detected exception with proper bean creation context already...
      throw ex;
   }
   catch (ImplicitlyAppearedSingletonException ex) {
      // An IllegalStateException to be communicated up to DefaultSingletonBeanRegistry...
      throw ex;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
   }
}
```

#### AbstractAutowireCapableBeanFactory#doCreateBean()

```java
//真正创建Bean的方法
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {
   // Instantiate the bean.
   //封装被创建的Bean对象
   BeanWrapper instanceWrapper = null;
   if (mbd.isSingleton()) {
     //未完成的FactoryBean实例的缓存清除。
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   if (instanceWrapper == null) {
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }
   final Object bean = instanceWrapper.getWrappedInstance();
   //获取实例化对象的类型
   Class<?> beanType = instanceWrapper.getWrappedClass();
   if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
   }

   // Allow post-processors to modify the merged bean definition.
   //调用PostProcessor后置处理器
   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         try {
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Post-processing of merged bean definition failed", ex);
         }
         mbd.postProcessed = true;
      }
   }

   // Eagerly cache singletons to be able to resolve circular references
   // even when triggered by lifecycle interfaces like BeanFactoryAware.
   //向容器中缓存单例模式的Bean对象，以防循环引用
   boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
         isSingletonCurrentlyInCreation(beanName));
   if (earlySingletonExposure) {
      if (logger.isDebugEnabled()) {
         logger.debug("Eagerly caching bean '" + beanName +
               "' to allow for resolving potential circular references");
      }
      //这里是一个匿名内部类，为了防止循环引用，尽早持有对象的引用
      addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   }

   // Initialize the bean instance.
   //Bean对象的初始化，依赖注入在此触发
   //这个exposedObject在初始化完成之后返回作为依赖注入完成后的Bean
   Object exposedObject = bean;
   try {
      //将Bean实例对象封装，并且Bean定义中配置的属性值赋值给实例对象
      populateBean(beanName, mbd, instanceWrapper);
      //初始化Bean对象
      exposedObject = initializeBean(beanName, exposedObject, mbd);
   }
   catch (Throwable ex) {
      if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
         throw (BeanCreationException) ex;
      }
      else {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
      }
   }

   if (earlySingletonExposure) {
      //获取指定名称的已注册的单例模式Bean对象
      Object earlySingletonReference = getSingleton(beanName, false);
      if (earlySingletonReference != null) {
         //根据名称获取的已注册的Bean和正在实例化的Bean是同一个
         if (exposedObject == bean) {
            //当前实例化的Bean初始化完成
            exposedObject = earlySingletonReference;
         }
         //当前Bean依赖其他Bean，并且当发生循环引用时不允许新创建实例对象
         else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
            String[] dependentBeans = getDependentBeans(beanName);
            Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
            //获取当前Bean所依赖的其他Bean
            for (String dependentBean : dependentBeans) {
               //对依赖Bean进行类型检查
               if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                  actualDependentBeans.add(dependentBean);
               }
            }
            if (!actualDependentBeans.isEmpty()) {
               throw new BeanCurrentlyInCreationException(beanName,
                     "Bean with name '" + beanName + "' has been injected into other beans [" +
                     StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                     "] in its raw version as part of a circular reference, but has eventually been " +
                     "wrapped. This means that said other beans do not use the final version of the " +
                     "bean. This is often the result of over-eager type matching - consider using " +
                     "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
            }
         }
      }
   }

   // Register bean as disposable.
   //注册完成依赖注入的Bean
   try {
      registerDisposableBeanIfNecessary(beanName, bean, mbd);
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
   }

   return exposedObject;
}
```

#### AbstractAutowireCapableBeanFactory#populateBean()

```java
//将Bean属性设置到生成的实例对象上
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
   if (bw == null) {
      if (mbd.hasPropertyValues()) {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
      }
      else {
         // Skip property population phase for null instance.
         return;
      }
   }

   // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
   // state of the bean before properties are set. This can be used, for example,
   // to support styles of field injection.
   boolean continueWithPropertyPopulation = true;

   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
               continueWithPropertyPopulation = false;
               break;
            }
         }
      }
   }

   if (!continueWithPropertyPopulation) {
      return;
   }
   //获取容器在解析Bean定义资源时为BeanDefinition中设置的属性值
   PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

   //对依赖注入处理，首先处理autowiring自动装配的依赖注入
   if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
         mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
      MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

      // Add property values based on autowire by name if applicable.
      //根据Bean名称进行autowiring自动装配处理
      if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
         autowireByName(beanName, mbd, bw, newPvs);
      }

      // Add property values based on autowire by type if applicable.
      //根据Bean类型进行autowiring自动装配处理
      if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
         autowireByType(beanName, mbd, bw, newPvs);
      }

      pvs = newPvs;
   }

   //对非autowiring的属性进行依赖注入处理

   boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
   boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

   if (hasInstAwareBpps || needsDepCheck) {
      if (pvs == null) {
         pvs = mbd.getPropertyValues();
      }
      PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
      if (hasInstAwareBpps) {
         for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
               InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
               pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
               if (pvs == null) {
                  return;
               }
            }
         }
      }
      if (needsDepCheck) {
        //如果需要的话，执行一个依赖性检查，检查所有暴露的属性是否已经被设置。
         checkDependencies(beanName, mbd, filteredPds, pvs);
      }
   }

   if (pvs != null) {
      //对属性进行注入
      applyPropertyValues(beanName, mbd, bw, pvs);
   }
}
```

#### AbstractAutowireCapableBeanFactory#initializeBean()

```java
//初始容器创建的Bean实例对象，为其添加BeanPostProcessor后置处理器
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
   //JDK的安全机制验证权限
   if (System.getSecurityManager() != null) {
      //实现PrivilegedAction接口的匿名内部类
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
         invokeAwareMethods(beanName, bean);
         return null;
      }, getAccessControlContext());
   }
   else {
      //为Bean实例对象包装相关属性，如名称，类加载器，所属容器等信息
      invokeAwareMethods(beanName, bean);
   }

   Object wrappedBean = bean;
   //对BeanPostProcessor后置处理器的postProcessBeforeInitialization
   //回调方法的调用，为Bean实例初始化前做一些处理
   if (mbd == null || !mbd.isSynthetic()) {
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   //调用Bean实例对象初始化的方法，这个初始化方法是在Spring Bean定义配置
   //文件中通过init-method属性指定的
   try {
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
   }
   //对BeanPostProcessor后置处理器的postProcessAfterInitialization
   //回调方法的调用，为Bean实例初始化之后做一些处理
   if (mbd == null || !mbd.isSynthetic()) {
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }

   return wrappedBean;
}
```