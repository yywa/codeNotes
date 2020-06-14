# Dubbo源码（一）服务暴露

## 版本介绍

​	从官网上直接摘抄下来的。源码统一为2.6.X版本。

| 版本   | 重要功能      | 升级建议   |                     |
| ---- | --------- | ------ | ------------------- |
| 1    | 2.6.x     | bugfix | 建议持续升级最新版本，所有版本生产可用 |
| 2    | 2.5.x     | 停止维护   | 建议升级最新 2.6.x 版本     |
| 3    | 2.4.x 及之前 | 停止维护   | 建议升级最新 2.6.x 版本     |

## 背景

[http://dubbo.apache.org/zh-cn/docs/user/preface/background.html](http://dubbo.apache.org/zh-cn/docs/user/preface/background.html)

## 架构

[http://dubbo.apache.org/zh-cn/docs/user/preface/architecture.html](http://dubbo.apache.org/zh-cn/docs/user/preface/architecture.html)

​	这些东西都是Dubbo官网上有的，非常方便，也支持中文。而且也可以从官网上看到Dubbo未来的一种架构。

## 服务暴露

​	服务器端在框架启动时，会初始化服务实例，通过Proxy组件调用具体协议Protocol，把服务器端要暴露的接口封装成Invoker，然后转换成Exporter，这个时候，框架会打开服务端口等并记录服务实例到内存中，最后通过Registry把服务元数据注册到注册中心。

- Proxy：Dubbo用来生成代理类，调用的方法其实是Proxy组件生成的代理方法。
- Protocol：对接口的配置，根据不同的协议转换成不同的Invoker对象。
- Exporter：用于暴露到注册中心的对象，内部属性持有了Invoker对象。
- Registry：把Exporter注册到注册中心。

  ​在Dubbo源码中，我们可以找到dubbo-demo下的dubbo-demo-provider项目下的Provider类，然后去分析它究竟做了些什么。

### Provider

```java
public class Provider {

    public static void main(String[] args) throws Exception {
        //Prevent to get IPV6 address,this way only work in debug mode
        //But you can pass use -Djava.net.preferIPv4Stack=true,then it work well whether in debug mode or not
      //防止获取IPV6地址。仅在调试模式下工作。
        System.setProperty("java.net.preferIPv4Stack", "true");
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"META-INF/spring/dubbo-demo-provider.xml"});
        context.start();

        System.in.read(); // press any key to exit
    }

}
```

​	可以看到，非常简单，初始化了一个Spring的ClassPathXmlApplicationContext，读取的文件为META-INF/spring/dubbo-demo-provider.xml，然后直接启动容器。

### dubbo-demo-provider.xml

```java
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
       http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">

    <!-- provider's application name, used for tracing dependency relationship -->
     //提供方应用信息，用于计算依赖关系
    <dubbo:application name="demo-provider"/>

    <!-- use multicast registry center to export service -->
      //使用zookeeper注册中心暴露服务地址。
    <dubbo:registry address="multicast://224.5.6.7:1234"/>

    <!-- use dubbo protocol to export service on port 20880 -->
      //使用dubbo协议暴露服务端口。
    <dubbo:protocol name="dubbo" port="20880"/>

    <!-- service implementation, as same as regular local bean -->
      //服务实现，为一个普通的Spring Bean
    <bean id="demoService" class="com.alibaba.dubbo.demo.provider.DemoServiceImpl"/>

    <!-- declare the service interface to be exported -->
      //声明要暴露的服务接口。
    <dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService"/>

</beans>
```

​	那么是怎么样去解析这种格式的xml？

​	通过Spring的扩展，**spring.handlers**和**spring.schemas**。

spring.handlers

```java
http\://dubbo.apache.org/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
```

spring.schemas

```java
http\://dubbo.apache.org/schema/dubbo/dubbo.xsd=META-INF/dubbo.xsd
http\://code.alibabatech.com/schema/dubbo/dubbo.xsd=META-INF/compat/dubbo.xsd
```

### DubboNamespaceHandler

​	扩展了Spring中的NamespaceHandlerSupport，当Spring启动时，会由**DefaultNamespaceHandlerResolver#resolve()**方法，会调用对应的**nameSpaceHandler.init()**方法。

```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {

    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }

    @Override
    public void init() {
      	//遇到对应key的标签，解析成对应的ConfigBean。
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }
}
```

### DubboBeanDefinitionParser#parse

```java
private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
  	//初始化了bean
    RootBeanDefinition beanDefinition = new RootBeanDefinition();
  	//将对应的配置bean注入。
    beanDefinition.setBeanClass(beanClass);
  	//不允许懒加载。
    beanDefinition.setLazyInit(false);
    String id = element.getAttribute("id");
  	//如果没有id属性值，则判断，生成一个唯一的id。
    if ((id == null || id.length() == 0) && required) {
        String generatedBeanName = element.getAttribute("name");
        if (generatedBeanName == null || generatedBeanName.length() == 0) {
            if (ProtocolConfig.class.equals(beanClass)) {
                generatedBeanName = "dubbo";
            } else {
                generatedBeanName = element.getAttribute("interface");
            }
        }
        if (generatedBeanName == null || generatedBeanName.length() == 0) {
          //实际配置的Class
          //com.alibaba.dubbo.config.ApplicationConfig
            generatedBeanName = beanClass.getName();
        }
        id = generatedBeanName;
        int counter = 2;
      	//如果同样的标签暴露了两次，则id+2、3、4来表示，因为Spring中id不能重复。
        while (parserContext.getRegistry().containsBeanDefinition(id)) {
            id = generatedBeanName + (counter++);
        }
    }
    if (id != null && id.length() > 0) {
        if (parserContext.getRegistry().containsBeanDefinition(id)) {
            throw new IllegalStateException("Duplicate spring bean id " + id);
        }
      	//这里用调用DefaultListableBeanFactory#registerBeanDefinition()，将id和beanDefinition注册到ConcurrentHashMap
        parserContext.getRegistry().registerBeanDefinition(id, beanDefinition);
        beanDefinition.getPropertyValues().addPropertyValue("id", id);
    }
  	//单独对对应的Config进行处理。
    if (ProtocolConfig.class.equals(beanClass)) {
      	//如果BeanDefinition中有不为空的protocol属性。
        for (String name : parserContext.getRegistry().getBeanDefinitionNames()) {
            BeanDefinition definition = parserContext.getRegistry().getBeanDefinition(name);
            PropertyValue property = definition.getPropertyValues().getPropertyValue("protocol");
            if (property != null) {
                Object value = property.getValue();
                if (value instanceof ProtocolConfig && id.equals(((ProtocolConfig) value).getName())) {
                    definition.getPropertyValues().addPropertyValue("protocol", new RuntimeBeanReference(id));
                }
            }
        }
      
    //serviceBean
    } else if (ServiceBean.class.equals(beanClass)) {
      	//获取类全名。
        String className = element.getAttribute("class");
        if (className != null && className.length() > 0) {
            RootBeanDefinition classDefinition = new RootBeanDefinition();
          	//通过反射获取类实例
            classDefinition.setBeanClass(ReflectUtils.forName(className));
            classDefinition.setLazyInit(false);
          	//解析子节点配置
            parseProperties(element.getChildNodes(), classDefinition);
            beanDefinition.getPropertyValues().addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, id + "Impl"));
        }
      //ProviderConfig
    } else if (ProviderConfig.class.equals(beanClass)) {
        parseNested(element, parserContext, ServiceBean.class, true, "service", "provider", id, beanDefinition);
    } else if (ConsumerConfig.class.equals(beanClass)) {
        parseNested(element, parserContext, ReferenceBean.class, false, "reference", "consumer", id, beanDefinition);
    }
    Set<String> props = new HashSet<String>();
    ManagedMap parameters = null;
    for (Method setter : beanClass.getMethods()) {
      	//从beanClass中获取所有的method方法，当method方法长度大于3，并且以set开头时，才注入。
        String name = setter.getName();
        if (name.length() > 3 && name.startsWith("set")
                && Modifier.isPublic(setter.getModifiers())
                && setter.getParameterTypes().length == 1) {
            Class<?> type = setter.getParameterTypes()[0];
          	//setName。以此为例，更好的理解，propertyName则为name。是国人写的代码，很符合我的思维。
            String propertyName = name.substring(3, 4).toLowerCase() + name.substring(4);
            String property = StringUtils.camelToSplitName(propertyName, "-");
            props.add(property);
            Method getter = null;
            try {
              	//通过反射获取getter。
                getter = beanClass.getMethod("get" + name.substring(3), new Class<?>[0]);
            } catch (NoSuchMethodException e) {
                try {
                    getter = beanClass.getMethod("is" + name.substring(3), new Class<?>[0]);
                } catch (NoSuchMethodException e2) {
                }
            }
            if (getter == null
                    || !Modifier.isPublic(getter.getModifiers())
                    || !type.equals(getter.getReturnType())) {
                continue;
            }
          	//解析dubbo:parameter，设置对应的值
            if ("parameters".equals(property)) {
                parameters = parseParameters(element.getChildNodes(), beanDefinition);
            } else if ("methods".equals(property)) {
              //解析dubbo:methods，设置对应的值
                parseMethods(id, element.getChildNodes(), beanDefinition, parserContext);
            } else if ("arguments".equals(property)) {
              //解析dubbo:arguments，设置对应的值
                parseArguments(id, element.getChildNodes(), beanDefinition, parserContext);
            } else {
                String value = element.getAttribute(property);
                if (value != null) {
                    value = value.trim();
                    if (value.length() > 0) {
                      	//不发布到注册中心
                        if ("registry".equals(property) && RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(value)) {
                            RegistryConfig registryConfig = new RegistryConfig();
                            registryConfig.setAddress(RegistryConfig.NO_AVAILABLE);
                            beanDefinition.getPropertyValues().addPropertyValue(property, registryConfig);
                          //多注册中心
                        } else if ("registry".equals(property) && value.indexOf(',') != -1) {
                            parseMultiRef("registries", value, beanDefinition, parserContext);			//多provider
                        } else if ("provider".equals(property) && value.indexOf(',') != -1) {
                            parseMultiRef("providers", value, beanDefinition, parserContext);			//多protocol
                        } else if ("protocol".equals(property) && value.indexOf(',') != -1) {
                            parseMultiRef("protocols", value, beanDefinition, parserContext);
                        } else {
                            Object reference;
                            if (isPrimitive(type)) {
                                if ("async".equals(property) && "false".equals(value)
                                        || "timeout".equals(property) && "0".equals(value)
                                        || "delay".equals(property) && "0".equals(value)
                                        || "version".equals(property) && "0.0.0".equals(value)
                                        || "stat".equals(property) && "-1".equals(value)
                                        || "reliable".equals(property) && "false".equals(value)) {
                                    // backward compatibility for the default value in old version's xsd
                                    value = null;
                                }
                                reference = value;
                              //如果属性为Protocol，判断是否有扩展    
                              //检查当前协议是否已经解析过。
                            } else if ("protocol".equals(property)
                                    &&                                   ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(value)
                                    &&
                                   (!parserContext.getRegistry().containsBeanDefinition(value)
                                    || !ProtocolConfig.class.getName().equals(parserContext.getRegistry().getBeanDefinition(value).getBeanClassName()))) {
                                if ("dubbo:provider".equals(element.getTagName())) {
                                    logger.warn("Recommended replace <dubbo:provider protocol=\"" + value + "\" ... /> to <dubbo:protocol name=\"" + value + "\" ... />");
                                }
                                // backward compatibility
                                ProtocolConfig protocol = new ProtocolConfig();
                                protocol.setName(value);
                                reference = protocol;
                            } else if ("onreturn".equals(property)) {
                              //回调方法。
                                int index = value.lastIndexOf(".");
                                String returnRef = value.substring(0, index);
                                String returnMethod = value.substring(index + 1);
                                reference = new RuntimeBeanReference(returnRef);
                                beanDefinition.getPropertyValues().addPropertyValue("onreturnMethod", returnMethod);
                            } else if ("onthrow".equals(property)) {
                              //回调异常方法
                                int index = value.lastIndexOf(".");
                                String throwRef = value.substring(0, index);
                                String throwMethod = value.substring(index + 1);
                                reference = new RuntimeBeanReference(throwRef);
                                beanDefinition.getPropertyValues().addPropertyValue("onthrowMethod", throwMethod);
                            } else if ("oninvoke".equals(property)) {
                                int index = value.lastIndexOf(".");
                                String invokeRef = value.substring(0, index);
                                String invokeRefMethod = value.substring(index + 1);
                                reference = new RuntimeBeanReference(invokeRef);
                                beanDefinition.getPropertyValues().addPropertyValue("oninvokeMethod", invokeRefMethod);
                            } else {
                              //必须为单例bean。
                                if ("ref".equals(property) && parserContext.getRegistry().containsBeanDefinition(value)) {
                                    BeanDefinition refBean = parserContext.getRegistry().getBeanDefinition(value);
                                    if (!refBean.isSingleton()) {
                                        throw new IllegalStateException("The exported service ref " + value + " must be singleton! Please set the " + value + " bean scope to singleton, eg: <bean id=\"" + value + "\" scope=\"singleton\" ...>");
                                    }
                                }
                                reference = new RuntimeBeanReference(value);
                            }
                            beanDefinition.getPropertyValues().addPropertyValue(propertyName, reference);
                        }
                    }
                }
            }
        }
    }
  	
    NamedNodeMap attributes = element.getAttributes();
    int len = attributes.getLength();
    for (int i = 0; i < len; i++) {
        Node node = attributes.item(i);
        String name = node.getLocalName();
       //如果经历了上面的解析，仍然有标签未被解析，会被添加到parameters属性中。
        if (!props.contains(name)) {
            if (parameters == null) {
                parameters = new ManagedMap();
            }
            String value = node.getNodeValue();
            parameters.put(name, new TypedStringValue(value, String.class));
        }
    }
    if (parameters != null) {
        beanDefinition.getPropertyValues().addPropertyValue("parameters", parameters);
    }
  	//返回一个beanDefinition，这里我们得到的是一个内部的抽象的对bean的一个数据结构，具体的初始化在getBean方法中创建。（Spring源码）
    return beanDefinition;
```

### parseNested

```java
//如果 <dubbo:provider 标签没有配置default开关,那么直接设置 default = "false"
//这样做的目的是为了让 <dubbo:provider里的配置都只是 <dubbo:service 或 <dubbo:reference的默认或缺省配置
private static void parseNested(Element element, ParserContext parserContext, Class<?> beanClass, boolean required, String tag, String property, String ref, BeanDefinition beanDefinition) {
    NodeList nodeList = element.getChildNodes();
    if (nodeList != null && nodeList.getLength() > 0) {
        boolean first = true;
        for (int i = 0; i < nodeList.getLength(); i++) {
            Node node = nodeList.item(i);
            if (node instanceof Element) {
                if (tag.equals(node.getNodeName())
                        || tag.equals(node.getLocalName())) {
                    if (first) {
                        first = false;
                        String isDefault = element.getAttribute("default");
                        if (isDefault == null || isDefault.length() == 0) {
                            beanDefinition.getPropertyValues().addPropertyValue("default", "false");
                        }
                    }
                    BeanDefinition subDefinition = parse((Element) node, parserContext, beanClass, required);
                    if (subDefinition != null && ref != null && ref.length() > 0) {
                        subDefinition.getPropertyValues().addPropertyValue(property, new RuntimeBeanReference(ref));
                    }
                }
            }
        }
    }
}
```
### ServiceBean

```java
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean,
        ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, BeanNameAware,
        ApplicationEventPublisherAware
```

​	可以看到ServiceBean实现了ApplicationListener#onApplicationEvent()，ApplicationContextAware#setApplicationContext()，InitializingBean#afterPropertiesSet()接口。

​	在Spring启动过程中，会调用这些方法。顺序为setApplicationContext->afterPropertiesSet->onApplicationEvent

#### setApplicationContext()

```java
public void setApplicationContext(ApplicationContext applicationContext) {
  	//Spring的事件
    this.applicationContext = applicationContext;
    SpringExtensionFactory.addApplicationContext(applicationContext);
    supportedApplicationListener = addApplicationListener(applicationContext, this);
}
```

#### afterPropertiesSet

```java
//如果当前ServiceConfig并未包含下面的属性
//从当前Spring上下文中获取指定类型的bean。
//然后将其注入到ServiceConfig中。
public void afterPropertiesSet() throws Exception {
    if (getProvider() == null) {
        Map<String, ProviderConfig> providerConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProviderConfig.class, false, false);
        if (providerConfigMap != null && providerConfigMap.size() > 0) {
            Map<String, ProtocolConfig> protocolConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProtocolConfig.class, false, false);
            if ((protocolConfigMap == null || protocolConfigMap.size() == 0)
                    && providerConfigMap.size() > 1) { // backward compatibility
                List<ProviderConfig> providerConfigs = new ArrayList<ProviderConfig>();
                for (ProviderConfig config : providerConfigMap.values()) {
                    if (config.isDefault() != null && config.isDefault().booleanValue()) {
                        providerConfigs.add(config);
                    }
                }
                if (!providerConfigs.isEmpty()) {
                    setProviders(providerConfigs);
                }
            } else {
                ProviderConfig providerConfig = null;
                for (ProviderConfig config : providerConfigMap.values()) {
                    if (config.isDefault() == null || config.isDefault().booleanValue()) {
                        if (providerConfig != null) {
                            throw new IllegalStateException("Duplicate provider configs: " + providerConfig + " and " + config);
                        }
                        providerConfig = config;
                    }
                }
                if (providerConfig != null) {
                    setProvider(providerConfig);
                }
            }
        }
    }
    if (getApplication() == null
            && (getProvider() == null || getProvider().getApplication() == null)) {
        Map<String, ApplicationConfig> applicationConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ApplicationConfig.class, false, false);
        if (applicationConfigMap != null && applicationConfigMap.size() > 0) {
            ApplicationConfig applicationConfig = null;
            for (ApplicationConfig config : applicationConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    if (applicationConfig != null) {
                        throw new IllegalStateException("Duplicate application configs: " + applicationConfig + " and " + config);
                    }
                    applicationConfig = config;
                }
            }
            if (applicationConfig != null) {
                setApplication(applicationConfig);
            }
        }
    }
    if (getModule() == null
            && (getProvider() == null || getProvider().getModule() == null)) {
        Map<String, ModuleConfig> moduleConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ModuleConfig.class, false, false);
        if (moduleConfigMap != null && moduleConfigMap.size() > 0) {
            ModuleConfig moduleConfig = null;
            for (ModuleConfig config : moduleConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    if (moduleConfig != null) {
                        throw new IllegalStateException("Duplicate module configs: " + moduleConfig + " and " + config);
                    }
                    moduleConfig = config;
                }
            }
            if (moduleConfig != null) {
                setModule(moduleConfig);
            }
        }
    }
    if ((getRegistries() == null || getRegistries().isEmpty())
            && (getProvider() == null || getProvider().getRegistries() == null || getProvider().getRegistries().isEmpty())
            && (getApplication() == null || getApplication().getRegistries() == null || getApplication().getRegistries().isEmpty())) {
        Map<String, RegistryConfig> registryConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, RegistryConfig.class, false, false);
        if (registryConfigMap != null && registryConfigMap.size() > 0) {
            List<RegistryConfig> registryConfigs = new ArrayList<RegistryConfig>();
            for (RegistryConfig config : registryConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    registryConfigs.add(config);
                }
            }
            if (registryConfigs != null && !registryConfigs.isEmpty()) {
                super.setRegistries(registryConfigs);
            }
        }
    }
    if (getMonitor() == null
            && (getProvider() == null || getProvider().getMonitor() == null)
            && (getApplication() == null || getApplication().getMonitor() == null)) {
        Map<String, MonitorConfig> monitorConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, MonitorConfig.class, false, false);
        if (monitorConfigMap != null && monitorConfigMap.size() > 0) {
            MonitorConfig monitorConfig = null;
            for (MonitorConfig config : monitorConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    if (monitorConfig != null) {
                        throw new IllegalStateException("Duplicate monitor configs: " + monitorConfig + " and " + config);
                    }
                    monitorConfig = config;
                }
            }
            if (monitorConfig != null) {
                setMonitor(monitorConfig);
            }
        }
    }
    if ((getProtocols() == null || getProtocols().isEmpty())
            && (getProvider() == null || getProvider().getProtocols() == null || getProvider().getProtocols().isEmpty())) {
        Map<String, ProtocolConfig> protocolConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProtocolConfig.class, false, false);
        if (protocolConfigMap != null && protocolConfigMap.size() > 0) {
            List<ProtocolConfig> protocolConfigs = new ArrayList<ProtocolConfig>();
            for (ProtocolConfig config : protocolConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    protocolConfigs.add(config);
                }
            }
            if (protocolConfigs != null && !protocolConfigs.isEmpty()) {
                super.setProtocols(protocolConfigs);
            }
        }
    }
    if (getPath() == null || getPath().length() == 0) {
        if (beanName != null && beanName.length() > 0
                && getInterface() != null && getInterface().length() > 0
                && beanName.startsWith(getInterface())) {
            setPath(beanName);
        }
    }
  	//如果不是延时暴露服务，直接暴露。
    if (!isDelay()) {
        export();
    }
}
```

#### ServiceConfig#export()

```java
public synchronized void export() {
    if (provider != null) {
        if (export == null) {
            export = provider.getExport();
        }
        if (delay == null) {
            delay = provider.getDelay();
        }
    }
    if (export != null && !export) {
        return;
    }

    if (delay != null && delay > 0) {
      //这里使用了线程池去解决服务延时发布的问题。
        delayExportExecutor.schedule(new Runnable() {
            @Override
            public void run() {
                doExport();
            }
        }, delay, TimeUnit.MILLISECONDS);
    } else {
        doExport();
    }
}
```

#### ServiceConfig#doExport()

```java
protected synchronized void doExport() {
    if (unexported) {
        throw new IllegalStateException("Already unexported!");
    }
    if (exported) {
        return;
    }
    exported = true;
    if (interfaceName == null || interfaceName.length() == 0) {
        throw new IllegalStateException("<dubbo:service interface=\"\" /> interface not allow null!");
    }
  	//检测 provider 是否为空，为空则新建一个，并通过系统变量为其初始化
    checkDefault();
  	// 下面几个 if 语句用于检测 provider、application 等核心配置类对象是否为空，
    // 若为空，则尝试从其他配置类对象中获取相应的实例。
    if (provider != null) {
        if (application == null) {
            application = provider.getApplication();
        }
        if (module == null) {
            module = provider.getModule();
        }
        if (registries == null) {
            registries = provider.getRegistries();
        }
        if (monitor == null) {
            monitor = provider.getMonitor();
        }
        if (protocols == null) {
            protocols = provider.getProtocols();
        }
    }
    if (module != null) {
        if (registries == null) {
            registries = module.getRegistries();
        }
        if (monitor == null) {
            monitor = module.getMonitor();
        }
    }
    if (application != null) {
        if (registries == null) {
            registries = application.getRegistries();
        }
        if (monitor == null) {
            monitor = application.getMonitor();
        }
    }
  	// 检测 ref 是否为泛化服务类型
    if (ref instanceof GenericService) {
        interfaceClass = GenericService.class;
        if (StringUtils.isEmpty(generic)) {
            generic = Boolean.TRUE.toString();
        }
    } else {
        try {
            interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                    .getContextClassLoader());
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
      	// 对 interfaceClass，以及 <dubbo:method> 标签中的必要字段进行检查
        checkInterfaceAndMethods(interfaceClass, methods);
      	// 对 ref 合法性进行检测
        checkRef();
        generic = Boolean.FALSE.toString();
    }
  	// local 和 stub 在功能应该是一致的，用于配置本地存根
    if (local != null) {
        if ("true".equals(local)) {
            local = interfaceName + "Local";
        }
        Class<?> localClass;
        try {
          	// 获取本地存根类
            localClass = ClassHelper.forNameWithThreadContextClassLoader(local);
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
      	// 检测本地存根类是否可赋值给接口类，若不可赋值则会抛出异常，提醒使用者本地存根类类型不合法
        if (!interfaceClass.isAssignableFrom(localClass)) {
            throw new IllegalStateException("The local implementation class " + localClass.getName() + " not implement interface " + interfaceName);
        }
    }
    if (stub != null) {
        if ("true".equals(stub)) {
            stub = interfaceName + "Stub";
        }
        Class<?> stubClass;
        try {
            stubClass = ClassHelper.forNameWithThreadContextClassLoader(stub);
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        if (!interfaceClass.isAssignableFrom(stubClass)) {
            throw new IllegalStateException("The stub implementation class " + stubClass.getName() + " not implement interface " + interfaceName);
        }
    }
   	// 检测各种对象是否为空，为空则新建，或者抛出异常
    checkApplication();
    checkRegistry();
    checkProtocol();
    appendProperties(this);
    checkStub(interfaceClass);
    checkMock(interfaceClass);
    if (path == null || path.length() == 0) {
        path = interfaceName;
    }
  	//实际导出服务
    doExportUrls();
  	// ProviderModel 表示服务提供者模型，此对象中存储了与服务提供者相关的信息。
    // 比如服务的配置信息，服务实例等。每个被导出的服务对应一个 ProviderModel。
    // ApplicationModel 持有所有的 ProviderModel。
    ProviderModel providerModel = new ProviderModel(getUniqueServiceName(), this, ref);
    ApplicationModel.initProviderModel(getUniqueServiceName(), providerModel);
}
```

#### ServiceConfig#doExportUrls()

```java
private void doExportUrls() {
  	 // 加载注册中心链接
    List<URL> registryURLs = loadRegistries(true);
  	// 遍历 protocols，并在每个协议下导出服务
    for (ProtocolConfig protocolConfig : protocols) {
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}
```

##### loadRegistries()

```java
protected List<URL> loadRegistries(boolean provider) {
  	//前面已经检查了一次。N次检查。
  	// 检测是否存在注册中心配置类，不存在则抛出异常
    checkRegistry();
    List<URL> registryList = new ArrayList<URL>();
    if (registries != null && !registries.isEmpty()) {
        for (RegistryConfig config : registries) {
            String address = config.getAddress();
            if (address == null || address.length() == 0) {
                address = Constants.ANYHOST_VALUE;
            }
          	// 从系统属性中加载注册中心地址
            String sysaddress = System.getProperty("dubbo.registry.address");
            if (sysaddress != null && sysaddress.length() > 0) {
                address = sysaddress;
            }
          	// 检测 address 是否合法
            if (address.length() > 0 && !RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(address)) {
              	// 添加 ApplicationConfig 中的字段信息到 map 中
                Map<String, String> map = new HashMap<String, String>();
                appendParameters(map, application);
              	// 添加 RegistryConfig 字段信息到 map 中
                appendParameters(map, config);
                map.put("path", RegistryService.class.getName());
                map.put("dubbo", Version.getProtocolVersion());
                map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
                if (ConfigUtils.getPid() > 0) {
                    map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
                }
                if (!map.containsKey("protocol")) {
                    if (ExtensionLoader.getExtensionLoader(RegistryFactory.class).hasExtension("remote")) {
                        map.put("protocol", "remote");
                    } else {
                        map.put("protocol", "dubbo");
                    }
                }
                // 解析得到 URL 列表，address 可能包含多个注册中心 ip，
                // 因此解析得到的是一个 URL 列表
                List<URL> urls = UrlUtils.parseURLs(address, map);
                for (URL url : urls) {
                    url = url.addParameter(Constants.REGISTRY_KEY, url.getProtocol());
                    // 将 URL 协议头设置为 registry
                    url = url.setProtocol(Constants.REGISTRY_PROTOCOL);
                    // 通过判断条件，决定是否添加 url 到 registryList 中，条件如下：
                    // (服务提供者 && register = true 或 null) 
                    //    || (非服务提供者 && subscribe = true 或 null)
                    if ((provider && url.getParameter(Constants.REGISTER_KEY, true))
                            || (!provider && url.getParameter(Constants.SUBSCRIBE_KEY, true))) {
                        registryList.add(url);
                    }
                }
            }
        }
    }
    return registryList;
}
```

##### doExportUrlsFor1Protocol()

```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    String name = protocolConfig.getName();
  	// 如果协议名为空，或空串，则将协议名变量设置为 dubbo
    if (name == null || name.length() == 0) {
        name = "dubbo";
    }
	// 添加 side、版本、时间戳以及进程号等信息到 map 中
    Map<String, String> map = new HashMap<String, String>();
    map.put(Constants.SIDE_KEY, Constants.PROVIDER_SIDE);
    map.put(Constants.DUBBO_VERSION_KEY, Version.getProtocolVersion());
    map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
    if (ConfigUtils.getPid() > 0) {
        map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
    }
  	// 通过反射将对象的字段信息添加到 map 中
    appendParameters(map, application);
    appendParameters(map, module);
    appendParameters(map, provider, Constants.DEFAULT_KEY);
    appendParameters(map, protocolConfig);
    appendParameters(map, this);
  	// methods 为 MethodConfig 集合，MethodConfig 中存储了 <dubbo:method> 标签的配置信息
    if (methods != null && !methods.isEmpty()) {
        for (MethodConfig method : methods) {
          	// 添加 MethodConfig 对象的字段信息到 map 中，键 = 方法名.属性名。
            // 比如存储 <dubbo:method name="sayHello" retries="2"> 对应的 MethodConfig，
            // 键 = sayHello.retries，map = {"sayHello.retries": 2, "xxx": "yyy"}
            appendParameters(map, method, method.getName());
            String retryKey = method.getName() + ".retry";
            if (map.containsKey(retryKey)) {
                String retryValue = map.remove(retryKey);
              	// 检测 MethodConfig retry 是否为 false，若是，则设置重试次数为0
                if ("false".equals(retryValue)) {
                    map.put(method.getName() + ".retries", "0");
                }
            }
          	// 获取 ArgumentConfig 列表
            List<ArgumentConfig> arguments = method.getArguments();
            if (arguments != null && !arguments.isEmpty()) {
                for (ArgumentConfig argument : arguments) {
                    // convert argument type
                    if (argument.getType() != null && argument.getType().length() > 0) {
                        Method[] methods = interfaceClass.getMethods();
                        // visit all methods
                        if (methods != null && methods.length > 0) {
                            for (int i = 0; i < methods.length; i++) {
                                String methodName = methods[i].getName();
                                // target the method, and get its signature
                                if (methodName.equals(method.getName())) {
                                    Class<?>[] argtypes = methods[i].getParameterTypes();
                                    // one callback in the method
                                    if (argument.getIndex() != -1) {
                                        if (argtypes[argument.getIndex()].getName().equals(argument.getType())) {
                                            appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                                        } else {
                                            throw new IllegalArgumentException("argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                                        }
                                    } else {
                                        // multiple callbacks in the method
                                        for (int j = 0; j < argtypes.length; j++) {
                                            Class<?> argclazz = argtypes[j];
                                            if (argclazz.getName().equals(argument.getType())) {
                                                appendParameters(map, argument, method.getName() + "." + j);
                                                if (argument.getIndex() != -1 && argument.getIndex() != j) {
                                                    throw new IllegalArgumentException("argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    } else if (argument.getIndex() != -1) {
                        appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                    } else {
                        throw new IllegalArgumentException("argument config must set index or type attribute.eg: <dubbo:argument index='0' .../> or <dubbo:argument type=xxx .../>");
                    }

                }
            }
        } // end of methods for
    }
	// 检测 generic 是否为 "true"，并根据检测结果向 map 中添加不同的信息
    if (ProtocolUtils.isGeneric(generic)) {
        map.put(Constants.GENERIC_KEY, generic);
        map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
    } else {
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
            map.put("revision", revision);
        }
		// 为接口生成包裹类 Wrapper，Wrapper 中包含了接口的详细信息，比如接口方法名数组，字段信息等
        String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
        // 添加方法名到 map 中，如果包含多个方法名，则用逗号隔开，比如 method = init,destroy
      	if (methods.length == 0) {
            logger.warn("NO method found in service interface " + interfaceClass.getName());
            map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
        } else {
          	// 将逗号作为分隔符连接方法名，并将连接后的字符串放入 map 中
            map.put(Constants.METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
        }
    }
  	// 添加 token 到 map 中
    if (!ConfigUtils.isEmpty(token)) {
        if (ConfigUtils.isDefault(token)) {
            map.put(Constants.TOKEN_KEY, UUID.randomUUID().toString());
        } else {
            map.put(Constants.TOKEN_KEY, token);
        }
    }
  	// 判断协议名是否为 injvm
    if (Constants.LOCAL_PROTOCOL.equals(protocolConfig.getName())) {
        protocolConfig.setRegister(false);
        map.put("notify", "false");
    }
    // export service
  	// 获取上下文路径
    String contextPath = protocolConfig.getContextpath();
    if ((contextPath == null || contextPath.length() == 0) && provider != null) {
        contextPath = provider.getContextpath();
    }
	// 获取 host 和 port
    String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
    Integer port = this.findConfigedPorts(protocolConfig, name, map);
    URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);

    if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
            .hasExtension(url.getProtocol())) {
      	// 加载 ConfiguratorFactory，并生成 Configurator 实例，然后通过实例配置 url
        url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
    }

    String scope = url.getParameter(Constants.SCOPE_KEY);
    // don't export when none is configured
  	// 如果 scope = none，则什么都不做
    if (!Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {

        // export to local if the config is not remote (export to remote only when config is remote)
        if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
            exportLocal(url);
        }
        // export to remote if the config is not local (export to local only when config is local)
      	// scope != local，导出到远程
        if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
            if (logger.isInfoEnabled()) {
                logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
            }
            if (registryURLs != null && !registryURLs.isEmpty()) {
                for (URL registryURL : registryURLs) {
                    url = url.addParameterIfAbsent(Constants.DYNAMIC_KEY, registryURL.getParameter(Constants.DYNAMIC_KEY));
                  	// 加载监视器链接
                    URL monitorUrl = loadMonitor(registryURL);
                    if (monitorUrl != null) {
                      	 // 将监视器链接作为参数添加到 url 中
                        url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                    }
                    if (logger.isInfoEnabled()) {
                        logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                    }

                    // For providers, this is used to enable custom proxy to generate invoker
                    String proxy = url.getParameter(Constants.PROXY_KEY);
                    if (StringUtils.isNotEmpty(proxy)) {
                        registryURL = registryURL.addParameter(Constants.PROXY_KEY, proxy);
                    }
					// 为服务提供类(ref)生成 Invoker
                    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
                  	// DelegateProviderMetaDataInvoker 用于持有 Invoker 和 ServiceConfig
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
					// 导出服务，并生成 Exporter
                    Exporter<?> exporter = protocol.export(wrapperInvoker);
                    exporters.add(exporter);
                }
            } else {
              	// 不存在注册中心，仅导出服务
                Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
                DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                Exporter<?> exporter = protocol.export(wrapperInvoker);
                exporters.add(exporter);
            }
        }
    }
    this.urls.add(url);
}
```

##### exportLocal(url) 暴露本地injvm服务

```java
private void exportLocal(URL url) {
  // 如果 URL 的协议头等于 injvm，说明已经导出到本地了，无需再次导出
    if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
        URL local = URL.valueOf(url.toFullString())
                .setProtocol(Constants.LOCAL_PROTOCOL)
                .setHost(LOCALHOST)
                .setPort(0);
        StaticContext.getContext(Constants.SERVICE_IMPL_CLASS).put(url.getServiceKey(), getServiceClass(ref));
      // 创建 Invoker，并导出服务，这里的 protocol 会在运行时调用 InjvmProtocol 的 export 方法
        Exporter<?> exporter = protocol.export(
                proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
        exporters.add(exporter);
        logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry");
    }
}
```

```java
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    return new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);
}
```

##### RegistryProtocol #export() 暴露服务到远程。

```java
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    // 导出服务
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);

    // 获取注册中心 URL，以 zookeeper 注册中心为例，得到的示例 URL 如下：
    // zookeeper://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.2&export=dubbo%3A%2F%2F172.17.48.52%3A20880%2Fcom.alibaba.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddemo-provider
    URL registryUrl = getRegistryUrl(originInvoker);

    // 根据 URL 加载 Registry 实现类，比如 ZookeeperRegistry
    final Registry registry = getRegistry(originInvoker);

    // 获取已注册的服务提供者 URL，比如：
    // dubbo://172.17.48.52:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.2&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello
    final URL registeredProviderUrl = getRegisteredProviderUrl(originInvoker);

    // 获取 register 参数
    boolean register = registeredProviderUrl.getParameter("register", true);

    // 向服务提供者与消费者注册表中注册服务提供者
    ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registeredProviderUrl);

    // 根据 register 的值决定是否注册服务
    if (register) {
        // 向注册中心注册服务
        register(registryUrl, registeredProviderUrl);
        ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
    }

    // 获取订阅 URL，比如：
    // provider://172.17.48.52:20880/com.alibaba.dubbo.demo.DemoService?category=configurators&check=false&anyhost=true&application=demo-provider&dubbo=2.0.2&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registeredProviderUrl);
    // 创建监听器
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    // 向注册中心进行订阅 override 数据
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
    // 创建并返回 DestroyableExporter
    return new DestroyableExporter<T>(exporter, originInvoker, overrideSubscribeUrl, registeredProviderUrl);
}
```



```java
public void register(URL registryUrl, URL registedProviderUrl) {
  	 // 获取 Registry
    Registry registry = registryFactory.getRegistry(registryUrl);
  	// 注册服务
    registry.register(registedProviderUrl);
}
```

AbstractRegistryFactory#getRegistry

```java
public Registry getRegistry(URL url) {
    url = url.setPath(RegistryService.class.getName())
            .addParameter(Constants.INTERFACE_KEY, RegistryService.class.getName())
            .removeParameters(Constants.EXPORT_KEY, Constants.REFER_KEY);
    String key = url.toServiceString();
    LOCK.lock();
    try {
    	// 访问缓存
        Registry registry = REGISTRIES.get(key);
        if (registry != null) {
            return registry;
        }
        
        // 缓存未命中，创建 Registry 实例
        registry = createRegistry(url);
        if (registry == null) {
            throw new IllegalStateException("Can not create registry...");
        }
        
        // 写入缓存
        REGISTRIES.put(key, registry);
        return registry;
    } finally {
        LOCK.unlock();
    }
}
```

```java
public class ZookeeperRegistryFactory extends AbstractRegistryFactory {

    // zookeeperTransporter 由 SPI 在运行时注入，类型为 ZookeeperTransporter$Adaptive
    private ZookeeperTransporter zookeeperTransporter;

    public void setZookeeperTransporter(ZookeeperTransporter zookeeperTransporter) {
        this.zookeeperTransporter = zookeeperTransporter;
    }

    @Override
    public Registry createRegistry(URL url) {
        // 创建 ZookeeperRegistry
        return new ZookeeperRegistry(url, zookeeperTransporter);
    }
}
```

```java
public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
    super(url);
    if (url.isAnyHost()) {
        throw new IllegalStateException("registry address == null");
    }
    
    // 获取组名，默认为 dubbo
    String group = url.getParameter(Constants.GROUP_KEY, DEFAULT_ROOT);
    if (!group.startsWith(Constants.PATH_SEPARATOR)) {
        // group = "/" + group
        group = Constants.PATH_SEPARATOR + group;
    }
    this.root = group;
    // 创建 Zookeeper 客户端，默认为 CuratorZookeeperTransporter
    zkClient = zookeeperTransporter.connect(url);
    // 添加状态监听器
    zkClient.addStateListener(new StateListener() {
        @Override
        public void stateChanged(int state) {
            if (state == RECONNECTED) {
                try {
                    recover();
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                }
            }
        }
    });
}
```

```java
public ZookeeperClient connect(URL url) {
    // 创建 CuratorZookeeperClient
    return new CuratorZookeeperClient(url);
}
```

```java
public class CuratorZookeeperClient extends AbstractZookeeperClient<CuratorWatcher> {

    private final CuratorFramework client;
    
    public CuratorZookeeperClient(URL url) {
        super(url);
        try {
            // 创建 CuratorFramework 构造器
            CuratorFrameworkFactory.Builder builder = CuratorFrameworkFactory.builder()
                    .connectString(url.getBackupAddress())
                    .retryPolicy(new RetryNTimes(1, 1000))
                    .connectionTimeoutMs(5000);
            String authority = url.getAuthority();
            if (authority != null && authority.length() > 0) {
                builder = builder.authorization("digest", authority.getBytes());
            }
            // 构建 CuratorFramework 实例
            client = builder.build();
            // 添加监听器
            client.getConnectionStateListenable().addListener(new ConnectionStateListener() {
                @Override
                public void stateChanged(CuratorFramework client, ConnectionState state) {
                    if (state == ConnectionState.LOST) {
                        CuratorZookeeperClient.this.stateChanged(StateListener.DISCONNECTED);
                    } else if (state == ConnectionState.CONNECTED) {
                        CuratorZookeeperClient.this.stateChanged(StateListener.CONNECTED);
                    } else if (state == ConnectionState.RECONNECTED) {
                        CuratorZookeeperClient.this.stateChanged(StateListener.RECONNECTED);
                    }
                }
            });
            
            // 启动客户端
            client.start();
        } catch (Exception e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
    }
}
```

**FailbackRegistry #register**

```java
public void register(URL url) {
    super.register(url);
    failedRegistered.remove(url);
    failedUnregistered.remove(url);
    try {
        // 模板方法，由子类实现
        doRegister(url);
    } catch (Exception e) {
        Throwable t = e;

        // 获取 check 参数，若 check = true 将会直接抛出异常
        boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                && url.getParameter(Constants.CHECK_KEY, true)
                && !Constants.CONSUMER_PROTOCOL.equals(url.getProtocol());
        boolean skipFailback = t instanceof SkipFailbackWrapperException;
        if (check || skipFailback) {
            if (skipFailback) {
                t = t.getCause();
            }
            throw new IllegalStateException("Failed to register");
        } else {
            logger.error("Failed to register");
        }

        // 记录注册失败的链接
        failedRegistered.add(url);
    }
}
```

**ZookeeperRegistry #doRegister**

```java
protected void doRegister(URL url) {
    try {
        // 通过 Zookeeper 客户端创建节点，节点路径由 toUrlPath 方法生成，路径格式如下:
        //   /${group}/${serviceInterface}/providers/${url}
        // 比如
        //   /dubbo/org.apache.dubbo.DemoService/providers/dubbo%3A%2F%2F127.0.0.1......
        zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
    } catch (Throwable e) {
        throw new RpcException("Failed to register...");
    }
}
```

```java
public void create(String path, boolean ephemeral) {
    if (!ephemeral) {
        // 如果要创建的节点类型非临时节点，那么这里要检测节点是否存在
        if (checkExists(path)) {
            return;
        }
    }
    int i = path.lastIndexOf('/');
    if (i > 0) {
        // 递归创建上一级路径
        create(path.substring(0, i), false);
    }
    
    // 根据 ephemeral 的值创建临时或持久节点
    if (ephemeral) {
        createEphemeral(path);
    } else {
        createPersistent(path);
    }
}
```

```java
public void createEphemeral(String path) {
    try {
        // 通过 Curator 框架创建节点
        client.create().withMode(CreateMode.EPHEMERAL).forPath(path);
    } catch (NodeExistsException e) {
    } catch (Exception e) {
        throw new IllegalStateException(e.getMessage(), e);
    }
}
```



**doLocalExport**

```java
private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker) {
    String key = getCacheKey(originInvoker);
  	// 访问缓存
    ExporterChangeableWrapper<T> exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
    if (exporter == null) {
        synchronized (bounds) {
            exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
            if (exporter == null) {
              	// 创建 Invoker 为委托类对象
                final Invoker<?> invokerDelegete = new InvokerDelegete<T>(originInvoker, getProviderUrl(originInvoker));
              	// 调用 protocol 的 export 方法导出服务
                exporter = new ExporterChangeableWrapper<T>((Exporter<T>) protocol.export(invokerDelegete), originInvoker);
              	// 写缓存
                bounds.put(key, exporter);
            }
        }
    }
    return exporter;
}
```

##### DubboProtcol#export() 创建netty

```java
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    URL url = invoker.getUrl();

    // 获取服务标识，理解成服务坐标也行。由服务组名，服务名，服务版本号以及端口组成。比如：
    // demoGroup/com.alibaba.dubbo.demo.DemoService:1.0.1:20880
    String key = serviceKey(url);
    // 创建 DubboExporter
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    // 将 <key, exporter> 键值对放入缓存中
    exporterMap.put(key, exporter);

    // 本地存根相关代码
    Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
    Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
    if (isStubSupportEvent && !isCallbackservice) {
        String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
        if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
            // 省略日志打印代码
        } else {
            stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
        }
    }

    // 启动服务器
    openServer(url);
    // 优化序列化
    optimizeSerialization(url);
    return exporter;
}
```

**openServer**

```java
private void openServer(URL url) {
    // 获取 host:port，并将其作为服务器实例的 key，用于标识当前的服务器实例
    String key = url.getAddress();
    boolean isServer = url.getParameter(Constants.IS_SERVER_KEY, true);
    if (isServer) {
        // 访问缓存
        ExchangeServer server = serverMap.get(key);
        if (server == null) {
            // 创建服务器实例
            serverMap.put(key, createServer(url));
        } else {
            // 服务器已创建，则根据 url 中的配置重置服务器
            server.reset(url);
        }
    }
}
```

**createServer**

```java
private ExchangeServer createServer(URL url) {
    url = url.addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY,
            // 添加心跳检测配置到 url 中
            url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
    // 获取 server 参数，默认为 netty
    String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);

    // 通过 SPI 检测是否存在 server 参数所代表的 Transporter 拓展，不存在则抛出异常
    if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str))
        throw new RpcException("Unsupported server type: " + str + ", url: " + url);

    // 添加编码解码器参数
    url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
    ExchangeServer server;
    try {
        // 创建 ExchangeServer
        server = Exchangers.bind(url, requestHandler);
    } catch (RemotingException e) {
        throw new RpcException("Fail to start server...");
    }

    // 获取 client 参数，可指定 netty，mina
    str = url.getParameter(Constants.CLIENT_KEY);
    if (str != null && str.length() > 0) {
        // 获取所有的 Transporter 实现类名称集合，比如 supportedTypes = [netty, mina]
        Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
        // 检测当前 Dubbo 所支持的 Transporter 实现类名称列表中，
        // 是否包含 client 所表示的 Transporter，若不包含，则抛出异常
        if (!supportedTypes.contains(str)) {
            throw new RpcException("Unsupported client type...");
        }
    }
    return server;
}
```

**bind**

```java
public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handler == null) {
        throw new IllegalArgumentException("handler == null");
    }
    url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
    // 获取 Exchanger，默认为 HeaderExchanger。
    // 紧接着调用 HeaderExchanger 的 bind 方法创建 ExchangeServer 实例
    return getExchanger(url).bind(url, handler);
}
```

​	HeaderExchanger 的 bind 方法包含的逻辑比较多，但目前我们仅需关心 Transporters 的 bind 方法逻辑即可。



```java
public static Server bind(URL url, ChannelHandler... handlers) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handlers == null || handlers.length == 0) {
        throw new IllegalArgumentException("handlers == null");
    }
    ChannelHandler handler;
    if (handlers.length == 1) {
        handler = handlers[0];
    } else {
    	// 如果 handlers 元素数量大于1，则创建 ChannelHandler 分发器
        handler = new ChannelHandlerDispatcher(handlers);
    }
    // 获取自适应 Transporter 实例，并调用实例方法
    return getTransporter().bind(url, handler);
}
```

​	getTransporter() 方法获取的 Transporter 是在运行时动态创建的，类名为 TransporterAdaptive，也就是自适应拓展类。TransporterAdaptive 会在运行时根据传入的 URL 参数决定加载什么类型的 Transporter，默认为 NettyTransporter。

```java
public Server bind(URL url, ChannelHandler listener) throws RemotingException {
	// 创建 NettyServer
	return new NettyServer(url, listener);
}
```

NettyServer.doOpen()

```java
protected void doOpen() throws Throwable {
    NettyHelper.setNettyLoggerFactory();
    // 创建 boss 和 worker 线程池
    ExecutorService boss = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerBoss", true));
    ExecutorService worker = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerWorker", true));
    ChannelFactory channelFactory = new NioServerSocketChannelFactory(boss, worker, getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS));
    
    // 创建 ServerBootstrap
    bootstrap = new ServerBootstrap(channelFactory);

    final NettyHandler nettyHandler = new NettyHandler(getUrl(), this);
    channels = nettyHandler.getChannels();
    bootstrap.setOption("child.tcpNoDelay", true);
    // 设置 PipelineFactory
    bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
        @Override
        public ChannelPipeline getPipeline() {
            NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
            ChannelPipeline pipeline = Channels.pipeline();
            pipeline.addLast("decoder", adapter.getDecoder());
            pipeline.addLast("encoder", adapter.getEncoder());
            pipeline.addLast("handler", nettyHandler);
            return pipeline;
        }
    });
    // 绑定到指定的 ip 和端口上
    channel = bootstrap.bind(getBindAddress());
}
```

​	





##### getInvoker()

> Invoker 是实体域，它是 Dubbo 的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起 invoke 调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。

```java
@Override
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
  	// 为目标类创建 Wrapper
    final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
  	// 创建匿名 Invoker 类对象，并实现 doInvoke 方法。
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName,
                                  Class<?>[] parameterTypes,
                                  Object[] arguments) throws Throwable {
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```

##### 