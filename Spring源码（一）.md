# Spring源码（一）bean的装载

## 1.DefaultListableBeanFactory

​	先来看看JavaDoc定义

> ListableBeanFactory和BeanDefinitionRegistry接口的默认实现：一个基于bean定义对象的完整的bean工厂。
> 典型的用法是先注册所有的Bean定义（可能是从Bean定义文件中读取），然后再访问Bean。因此，Bean定义查找是在本地Bean定义表中的一个廉价的操作，在预建的Bean定义元数据对象上进行操作。
> 可以作为一个独立的Bean工厂，也可以作为自定义Bean工厂的超级类使用。注意，特定bean定义格式的读取器通常是单独实现的，而不是作为bean工厂的子类：例如，参见PropertiesBeanDefinitionReader和org.springframework.beans.factory.xml.XmlBeanDefinitionReader。
> 关于ListableBeanFactory接口的另一种实现，请看StaticListableBeanFactory，它管理现有的bean实例，而不是基于bean定义创建新的bean实例。

​	由JavaDoc中的描述可以知道，DefaultListableBeanFactory是用来做Spring注册和加载bean的默认实现。如果有特定Bean格式，如XML，那么将由特殊的类去进行实现。XmlBeanFactory就是继承于DefaultListableBeanFactory。

​	接着看类层次结构图。

![img](file:///C:\Users\Admin\AppData\Roaming\Tencent\Users\912547147\TIM\WinTemp\RichOle\KNYM$74C0[M[SYG38PT3`}T.png)

- AliasRegistry：定义对alias的简单curd操作。
- SimpleAliasRegistry：使用map作为alias的缓存，并对接口AliasRegistry进行实现。
- SingletonBeanRegistry：定义对单例的注册及获取。
- BeanFactory：定义获取bean及bean的各种属性。
- DefaultSingletonBeanRegistry：对接口SingletonBeanRegistry的默认实现。
- HierarchicalBeanFactory：继承BeanFactory，也就是在BeanFactory定义的功能上增加了对parentFactory的支持。
- BeanDefinitionRegistry：定义对BeanDefinition的各种curd操作。
- FactoryBeanRegistrySupport：对DefaultSingletonBeanRegistry的扩展。
- ConfigurableBeanFactory：提供配置Factory的各种方法。
- ListableBeanFactory：根据各种条件获取bean的配置清单。
- AbstractBeanFactory：综合FactoryBeanRegistrySupport和ConfigurableBeanFactory的功能。
- AutowireCapableBeanFactory：提供创建bean、自动注入、初始化、以及应用bean的处理器。
- AbstractAutowireCapableBeanFactory：综合AbstractBeanFactory并对接口AutowireCapableBeanFactory进行实现。
- ConfigurableListableBeanFactory：BeanFactory配置清单，指定忽略类型及接口等。
- DefaultListableBeanFactory：对bean注册后的处理。
- XmlBeanFactory：对DefaultListableBeanFactory的扩展，主要用于从XML中读取BeanDefinition。

## 2.XmlBeanDefinitionReader

​	在XmlBeanFactory中，使用XmlBeanDefinitionReader对资源文件进行读取。 ![XmlBeanDefinitionReader](XmlBeanDefinitionReader.png)

- BeanDefinitionReader：定义资源文件读取并转换为BeanDefinition的功能。
- EnvironmentCapable：定义获取Environment方法。
- AbstractBeanDefinition：对BeanDefinitionReader和EnvironmentCapable的实现。

### 3.Resource

​	用于对资源文件的封装。 ![resource](resource.png)

​	最上层的接口是InputStreamSource，可以看到定义了各种各样的Resource方便封装，有Byte，File，Url等。

