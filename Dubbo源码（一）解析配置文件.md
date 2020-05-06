# Dubbo源码（一）解析配置文件

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

​	在Dubbo源码中，我们可以找到dubbo-demo下的dubbo-demo-provider项目下的Provider类，然后去分析它究竟做了些什么。

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
     //用于追踪依赖关系
    <dubbo:application name="demo-provider"/>

    <!-- use multicast registry center to export service -->
      //使用什么注册中心。一般生产使用Zookeeper。
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