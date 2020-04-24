# Spring源码（一）bean的装载

## 一.DefaultListableBeanFactory

​	先来看看JavaDoc定义

> ListableBeanFactory和BeanDefinitionRegistry接口的默认实现：一个基于bean定义对象的完整的bean工厂。
> 典型的用法是先注册所有的Bean定义（可能是从Bean定义文件中读取），然后再访问Bean。因此，Bean定义查找是在本地Bean定义表中的一个廉价的操作，在预建的Bean定义元数据对象上进行操作。
> 可以作为一个独立的Bean工厂，也可以作为自定义Bean工厂的超级类使用。注意，特定bean定义格式的读取器通常是单独实现的，而不是作为bean工厂的子类：例如，参见PropertiesBeanDefinitionReader和org.springframework.beans.factory.xml.XmlBeanDefinitionReader。
> 关于ListableBeanFactory接口的另一种实现，请看StaticListableBeanFactory，它管理现有的bean实例，而不是基于bean定义创建新的bean实例。

​	由JavaDoc中的描述可以知道，DefaultListableBeanFactory是用来做Spring注册和加载bean的默认实现。如果有特定Bean格式，如XML，那么将由特殊的类去进行实现。XmlBeanFactory就是继承于DefaultListableBeanFactory。

​	接着看类层次结构图。

 ![XmlBeanFactory](C:\Users\Admin\Desktop\XmlBeanFactory.png)

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

## 二.XmlBeanDefinitionReader

​	在XmlBeanFactory中，使用XmlBeanDefinitionReader对资源文件进行读取。 

 ![XmlBeanDefinitionReader](C:\Users\Admin\Desktop\XmlBeanDefinitionReader.png)

- BeanDefinitionReader：定义资源文件读取并转换为BeanDefinition的功能。
- EnvironmentCapable：定义获取Environment方法。
- AbstractBeanDefinition：对BeanDefinitionReader和EnvironmentCapable的实现。

## 三.Resource

​	用于对资源文件的封装。

 ![resource](C:\Users\Admin\Desktop\resource.png)

​	最上层的接口是InputStreamSource，可以看到定义了各种各样的Resource方便封装，有Byte，File，Url等。

​	进入正题。看代码流程。

## 四.加载bean（IOC）

```java
public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
   super(parentBeanFactory);
   this.reader.loadBeanDefinitions(resource);
}
```

​	在构造XmlBeanFactory的时候，会调用XmlBeanDefinitionReader#loadBeanDefinition(Resource)方法

### loadBeanDefinitions

```java
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
   //将读入的XML资源进行特殊编码处理
  //这里将resource封装成EncodedResource，方便后续编码。
   return loadBeanDefinitions(new EncodedResource(resource));
}
```

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
   Assert.notNull(encodedResource, "EncodedResource must not be null");
   if (logger.isInfoEnabled()) {
      logger.info("Loading XML bean definitions from " + encodedResource.getResource());
   }

   Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
   if (currentResources == null) {
      currentResources = new HashSet<>(4);
      this.resourcesCurrentlyBeingLoaded.set(currentResources);
   }
   if (!currentResources.add(encodedResource)) {
      throw new BeanDefinitionStoreException(
            "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
   }
   try {
      //将资源文件转为InputStream的IO流
      InputStream inputStream = encodedResource.getResource().getInputStream();
      try {
         //从InputStream中得到XML的解析源
         InputSource inputSource = new InputSource(inputStream);
         if (encodedResource.getEncoding() != null) {
            inputSource.setEncoding(encodedResource.getEncoding());
         }
         //这里是具体的读取过程
         return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
      }
      finally {
         //关闭从Resource中得到的IO流
         inputStream.close();
      }
   }
   catch (IOException ex) {
      throw new BeanDefinitionStoreException(
            "IOException parsing XML document from " + encodedResource.getResource(), ex);
   }
   finally {
      currentResources.remove(encodedResource);
      if (currentResources.isEmpty()) {
         this.resourcesCurrentlyBeingLoaded.remove();
      }
   }
}
```

### doLoadBeanDefinitions

```java
//从特定XML文件中载入Bean
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
      throws BeanDefinitionStoreException {
   try {
      //将XML文件转换为DOM对象。
      Document doc = doLoadDocument(inputSource, resource);
      //这里是启动对Bean定义解析的详细过程，该解析过程会用到Spring的Bean配置规则
      return registerBeanDefinitions(doc, resource);
   } catch (BeanDefinitionStoreException ex) {
      throw ex;
   } catch (SAXParseException ex) {
      throw new XmlBeanDefinitionStoreException(resource.getDescription(),
            "Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
   } catch (SAXException ex) {
      throw new XmlBeanDefinitionStoreException(resource.getDescription(),
            "XML document from " + resource + " is invalid", ex);
   } catch (ParserConfigurationException ex) {
      throw new BeanDefinitionStoreException(resource.getDescription(),
            "Parser configuration exception parsing XML from " + resource, ex);
   } catch (IOException ex) {
      throw new BeanDefinitionStoreException(resource.getDescription(),
            "IOException parsing XML document from " + resource, ex);
   } catch (Throwable ex) {
      throw new BeanDefinitionStoreException(resource.getDescription(),
            "Unexpected exception parsing XML document from " + resource, ex);
   }
}
```

#### 1、doLoadDocument()

```java
protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
   return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
         getValidationModeForResource(resource), isNamespaceAware());
}
```

​	这里的documentLoader则为DefaultDocumentLoader。

##### getEntityResolver()

​	那么EntityResolver是用来做什么的？

​	大家都知道，Spring验证XML文件是通过解析XML文档上的声明，去寻找DTD文件，EntityResolver则提供了一个寻找DTD声明的过程。具体如何解析，并转成InputSource可以去看**ResourceEntityResolver#resolveEntity()**方法

```java
protected EntityResolver getEntityResolver() {
   if (this.entityResolver == null) {
      // Determine default EntityResolver to use.
      ResourceLoader resourceLoader = getResourceLoader();
      if (resourceLoader != null) {
         this.entityResolver = new ResourceEntityResolver(resourceLoader);
      } else {
         this.entityResolver = new DelegatingEntityResolver(getBeanClassLoader());
      }
   }
   return this.entityResolver;
}
```

##### getValidationModeForResource()

```java
//获取指定资源的验证模式。有XSD、DTD等。
protected int getValidationModeForResource(Resource resource) {
   int validationModeToUse = getValidationMode();
   if (validationModeToUse != VALIDATION_AUTO) {
      return validationModeToUse;
   }
   int detectedMode = detectValidationMode(resource);
   if (detectedMode != VALIDATION_AUTO) {
      return detectedMode;
   }
   // Hmm, we didn't get a clear indication... Let's assume XSD,
   // since apparently no DTD declaration has been found up until
   // detection stopped (before finding the document's root tag).
   return VALIDATION_XSD;
}
```

##### loadDocument()

```java
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
      ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {
   //根据验证模式，和是否对XML命名空间的支持创建工厂
   DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
   if (logger.isDebugEnabled()) {
      logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
   }
   //根据解析器工厂、资源文件，错误处理器去生成文档解析器。
   DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
   //解析Spring的Bean定义资源
   return builder.parse(inputSource);
}
```

createDocumentBuilderFactory()

```java
protected DocumentBuilderFactory createDocumentBuilderFactory(int validationMode, boolean namespaceAware)
      throws ParserConfigurationException {

   //创建文档解析工厂
   DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
   factory.setNamespaceAware(namespaceAware);

   //设置解析XML的校验
   if (validationMode != XmlValidationModeDetector.VALIDATION_NONE) {
      factory.setValidating(true);
      if (validationMode == XmlValidationModeDetector.VALIDATION_XSD) {
         // Enforce namespace aware for XSD...
         factory.setNamespaceAware(true);
         try {
            factory.setAttribute(SCHEMA_LANGUAGE_ATTRIBUTE, XSD_SCHEMA_LANGUAGE);
         }
         catch (IllegalArgumentException ex) {
            ParserConfigurationException pcex = new ParserConfigurationException(
                  "Unable to validate using XSD: Your JAXP provider [" + factory +
                  "] does not support XML Schema. Are you running on Java 1.4 with Apache Crimson? " +
                  "Upgrade to Apache Xerces (or Java 1.5) for full XSD support.");
            pcex.initCause(ex);
            throw pcex;
         }
      }
   }

   return factory;
}
```

createDocumentBuilder()

```java
//创建一个DocumentBuilder，用于解析XML文档。
protected DocumentBuilder createDocumentBuilder(DocumentBuilderFactory factory,
      @Nullable EntityResolver entityResolver, @Nullable ErrorHandler errorHandler)
      throws ParserConfigurationException {

   DocumentBuilder docBuilder = factory.newDocumentBuilder();
   if (entityResolver != null) {
      docBuilder.setEntityResolver(entityResolver);
   }
   if (errorHandler != null) {
      docBuilder.setErrorHandler(errorHandler);
   }
   return docBuilder;
}
```

parse()

```java
//将给定的输入源的内容解析为XML文档，并返回一个新的DOM Document对象。
public Document parse(InputSource is) throws SAXException, IOException {
    if (is == null) {
        throw new IllegalArgumentException(
            DOMMessageFormatter.formatMessage(DOMMessageFormatter.DOM_DOMAIN,
            "jaxp-null-input-source", null));
    }
    if (fSchemaValidator != null) {
        if (fSchemaValidationManager != null) {
            fSchemaValidationManager.reset();
            fUnparsedEntityHandler.reset();
        }
        resetSchemaValidator();
    }
    domParser.parse(is);
    Document doc = domParser.getDocument();
    domParser.dropDocumentReferences();
    return doc;
}
```

​	经过上述处理，我们已经拿到一个DOM对象。

#### 2、registerBeanDefinitions()

```java
//按照Spring的Bean语义要求将Bean定义资源解析并转换为容器内部数据结构
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
   //得到BeanDefinitionDocumentReader来对xml格式的BeanDefinition解析
   BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
   //获得容器中注册的Bean数量
   int countBefore = getRegistry().getBeanDefinitionCount();
   //具体的解析过程。
   documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
   //统计解析的Bean数量
   return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

##### createBeanDefinitionDocumentReader()

```java
//BeanUtils#instantiateClass是使用无参构造函数实例化一个类。
//实例化的类为DefaultBeanDefinitionDocumentReader
protected BeanDefinitionDocumentReader createBeanDefinitionDocumentReader() {
   return BeanDefinitionDocumentReader.class.cast(BeanUtils.instantiateClass(this.documentReaderClass));
}
```

##### getBeanDefinitionCount()

​	getRegistry()，获取当前registry属性，也就是一个实现了BeanDefinitionRegistry接口的类，此处为XMLBeanFactory，调用为AbstractBeanDefinitionReader#getRegistry()方法。

```java
//这里返回的是countBefore，也就是为了记录注册之前，容器Bean的加载个数。
public int getBeanDefinitionCount() {
   return this.beanDefinitionMap.size();
}
```

##### registerBeanDefinitions()

​	先来看一下createReaderContext()

```java
//创建一个XMLReaderContext传递给DocumentReader
public XmlReaderContext createReaderContext(Resource resource) {
   return new XmlReaderContext(resource, this.problemReporter, this.eventListener,
         this.sourceExtractor, this, getNamespaceHandlerResolver());
}
```

```java
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
   //获得XMLReaderContext
   this.readerContext = readerContext;
   logger.debug("Loading bean definitions");
   //获得Document的根元素
   Element root = doc.getDocumentElement();
   doRegisterBeanDefinitions(root);
}
```

```java
//根据之前解析出的Document的Root节点进行解析。
protected void doRegisterBeanDefinitions(Element root) {
   // Any nested <beans> elements will cause recursion in this method. In
   // order to propagate and preserve <beans> default-* attributes correctly,
   // keep track of the current (parent) delegate, which may be null. Create
   // the new (child) delegate with a reference to the parent for fallback purposes,
   // then ultimately reset this.delegate back to its original (parent) reference.
   // this behavior emulates a stack of delegates without actually necessitating one.

   //具体的解析过程由BeanDefinitionParserDelegate实现，
   //BeanDefinitionParserDelegate中定义了Spring Bean定义XML文件的各种元素
   BeanDefinitionParserDelegate parent = this.delegate;
   this.delegate = createDelegate(getReaderContext(), root, parent);

   if (this.delegate.isDefaultNamespace(root)) {
      String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
      if (StringUtils.hasText(profileSpec)) {
         String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
               profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
         if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
            if (logger.isInfoEnabled()) {
               logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                     "] not matching: " + getReaderContext().getResource());
            }
            return;
         }
      }
   }
   //在解析Bean定义之前，进行自定义的解析，增强解析过程的可扩展性
   preProcessXml(root);
   //从Document的根元素开始进行Bean定义的Document对象
   parseBeanDefinitions(root, this.delegate);
   //在解析Bean定义之后，进行自定义的解析，增加解析过程的可扩展性
   postProcessXml(root);
   this.delegate = parent;
}
```

createDelegate()

```java
protected BeanDefinitionParserDelegate createDelegate(
      XmlReaderContext readerContext, Element root, @Nullable BeanDefinitionParserDelegate parentDelegate) {
	//根据之前的ReaderContext去初始化BeanDefinitionParserDelegate
   BeanDefinitionParserDelegate delegate = new BeanDefinitionParserDelegate(readerContext);
   //BeanDefinitionParserDelegate初始化Document根元素
   delegate.initDefaults(root, parentDelegate);
   return delegate;
}
```

initDefaults()

```java
//默认的lazy-init、autowire、依赖检查设置、init-method、destroy-method和merge设置，填充到parent。
//fireDefaultRegistered 启动默认注册事件。
public void initDefaults(Element root, @Nullable BeanDefinitionParserDelegate parent) {
   populateDefaults(this.defaults, (parent != null ? parent.defaults : null), root);
   this.readerContext.fireDefaultsRegistered(this.defaults);
}
```

populateDefaults()

```java
protected void populateDefaults(DocumentDefaultsDefinition defaults, @Nullable DocumentDefaultsDefinition parentDefaults, Element root) {
   String lazyInit = root.getAttribute(DEFAULT_LAZY_INIT_ATTRIBUTE);
   if (DEFAULT_VALUE.equals(lazyInit)) {
      // Potentially inherited from outer <beans> sections, otherwise falling back to false.
      lazyInit = (parentDefaults != null ? parentDefaults.getLazyInit() : FALSE_VALUE);
   }
   defaults.setLazyInit(lazyInit);

   String merge = root.getAttribute(DEFAULT_MERGE_ATTRIBUTE);
   if (DEFAULT_VALUE.equals(merge)) {
      // Potentially inherited from outer <beans> sections, otherwise falling back to false.
      merge = (parentDefaults != null ? parentDefaults.getMerge() : FALSE_VALUE);
   }
   defaults.setMerge(merge);

   String autowire = root.getAttribute(DEFAULT_AUTOWIRE_ATTRIBUTE);
   if (DEFAULT_VALUE.equals(autowire)) {
      // Potentially inherited from outer <beans> sections, otherwise falling back to 'no'.
      autowire = (parentDefaults != null ? parentDefaults.getAutowire() : AUTOWIRE_NO_VALUE);
   }
   defaults.setAutowire(autowire);

   if (root.hasAttribute(DEFAULT_AUTOWIRE_CANDIDATES_ATTRIBUTE)) {
      defaults.setAutowireCandidates(root.getAttribute(DEFAULT_AUTOWIRE_CANDIDATES_ATTRIBUTE));
   }
   else if (parentDefaults != null) {
      defaults.setAutowireCandidates(parentDefaults.getAutowireCandidates());
   }

   if (root.hasAttribute(DEFAULT_INIT_METHOD_ATTRIBUTE)) {
      defaults.setInitMethod(root.getAttribute(DEFAULT_INIT_METHOD_ATTRIBUTE));
   }
   else if (parentDefaults != null) {
      defaults.setInitMethod(parentDefaults.getInitMethod());
   }

   if (root.hasAttribute(DEFAULT_DESTROY_METHOD_ATTRIBUTE)) {
      defaults.setDestroyMethod(root.getAttribute(DEFAULT_DESTROY_METHOD_ATTRIBUTE));
   }
   else if (parentDefaults != null) {
      defaults.setDestroyMethod(parentDefaults.getDestroyMethod());
   }

   defaults.setSource(this.readerContext.extractSource(root));
}
```

preProcessXml(root);

```java
protected void preProcessXml(Element root) {
}
```

​	这里是空的方法，方便我们解析自定义的标签。

parseBeanDefinitions(root, this.delegate);

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
   //Bean定义的Document对象使用了Spring默认的XML命名空间
   if (delegate.isDefaultNamespace(root)) {
      //获取Bean定义的Document对象根元素的所有子节点
      NodeList nl = root.getChildNodes();
      for (int i = 0; i < nl.getLength(); i++) {
         Node node = nl.item(i);
         //获得Document节点是XML元素节点
         if (node instanceof Element) {
            Element ele = (Element) node;
            //Bean定义的Document的元素节点使用的是Spring默认的XML命名空间
            if (delegate.isDefaultNamespace(ele)) {
               //使用Spring的Bean规则解析元素节点
               parseDefaultElement(ele, delegate);
            }
            else {
               //没有使用Spring默认的XML命名空间，则使用用户自定义的解//析规则解析元素节点
               delegate.parseCustomElement(ele);
            }
         }
      }
   }
   else {
      //Document的根节点没有使用Spring默认的命名空间，则使用用户自定义的
      //解析规则解析Document根节点
      delegate.parseCustomElement(root);
   }
}
```

parseDefaultElement()

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
   //如果元素节点是<Import>导入元素，进行导入解析
   if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
      importBeanDefinitionResource(ele);
   }
   //如果元素节点是<Alias>别名元素，进行别名解析
   else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
      processAliasRegistration(ele);
   }
   //元素节点既不是导入元素，也不是别名元素，即普通的<Bean>元素，
   //按照Spring的Bean规则解析元素
   else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
      processBeanDefinition(ele, delegate);
   }
   else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
      // 如果仍是<beans>标签，则递归解析。
      doRegisterBeanDefinitions(ele);
   }
}
```

importBeanDefinitionResource()

```java
//解析<Import>导入元素，从给定的导入路径加载Bean定义资源到Spring IoC容器中
protected void importBeanDefinitionResource(Element ele) {
   //获取给定的导入元素的location属性
   String location = ele.getAttribute(RESOURCE_ATTRIBUTE);
   //如果导入元素的location属性值为空，则没有导入任何资源，直接返回
   if (!StringUtils.hasText(location)) {
      getReaderContext().error("Resource location must not be empty", ele);
      return;
   }

   // Resolve system properties: e.g. "${user.dir}"
   //使用系统变量值解析location属性值
   location = getReaderContext().getEnvironment().resolveRequiredPlaceholders(location);

   Set<Resource> actualResources = new LinkedHashSet<>(4);

   // Discover whether the location is an absolute or relative URI
   //标识给定的导入元素的location是否是绝对路径
   boolean absoluteLocation = false;
   try {
      absoluteLocation = ResourcePatternUtils.isUrl(location) || ResourceUtils.toURI(location).isAbsolute();
   }
   catch (URISyntaxException ex) {
      // cannot convert to an URI, considering the location relative
      // unless it is the well-known Spring prefix "classpath*:"
      //给定的导入元素的location不是绝对路径
   }

   // Absolute or relative?
   //给定的导入元素的location是绝对路径
   if (absoluteLocation) {
      try {
         //使用资源读入器加载给定路径的Bean定义资源
         int importCount = getReaderContext().getReader().loadBeanDefinitions(location, actualResources);
         if (logger.isDebugEnabled()) {
            logger.debug("Imported " + importCount + " bean definitions from URL location [" + location + "]");
         }
      }
      catch (BeanDefinitionStoreException ex) {
         getReaderContext().error(
               "Failed to import bean definitions from URL location [" + location + "]", ele, ex);
      }
   }
   else {
      // No URL -> considering resource location as relative to the current file.
      //给定的导入元素的location是相对路径
      try {
         int importCount;
         //将给定导入元素的location封装为相对路径资源
         Resource relativeResource = getReaderContext().getResource().createRelative(location);
         //封装的相对路径资源存在
         if (relativeResource.exists()) {
            //使用资源读入器加载Bean定义资源
            importCount = getReaderContext().getReader().loadBeanDefinitions(relativeResource);
            actualResources.add(relativeResource);
         }
         //封装的相对路径资源不存在
         else {
            //获取Spring IOC容器资源读入器的基本路径
            String baseLocation = getReaderContext().getResource().getURL().toString();
            //根据Spring IOC容器资源读入器的基本路径加载给定导入路径的资源
            importCount = getReaderContext().getReader().loadBeanDefinitions(
                  StringUtils.applyRelativePath(baseLocation, location), actualResources);
         }
         if (logger.isDebugEnabled()) {
            logger.debug("Imported " + importCount + " bean definitions from relative location [" + location + "]");
         }
      }
      catch (IOException ex) {
         getReaderContext().error("Failed to resolve current resource location", ele, ex);
      }
      catch (BeanDefinitionStoreException ex) {
         getReaderContext().error("Failed to import bean definitions from relative location [" + location + "]",
               ele, ex);
      }
   }
   Resource[] actResArray = actualResources.toArray(new Resource[actualResources.size()]);
   //在解析完<Import>元素之后，发送容器导入其他资源处理完成事件
   getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
}
```

processAliasRegistration()

```java
protected void processAliasRegistration(Element ele) {
   //获取<Alias>别名元素中name的属性值
   String name = ele.getAttribute(NAME_ATTRIBUTE);
   //获取<Alias>别名元素中alias的属性值
   String alias = ele.getAttribute(ALIAS_ATTRIBUTE);
   boolean valid = true;
   //<alias>别名元素的name属性值为空
   if (!StringUtils.hasText(name)) {
      getReaderContext().error("Name must not be empty", ele);
      valid = false;
   }
   //<alias>别名元素的alias属性值为空
   if (!StringUtils.hasText(alias)) {
      getReaderContext().error("Alias must not be empty", ele);
      valid = false;
   }
   if (valid) {
      try {
         //向容器的资源读入器注册别名
         getReaderContext().getRegistry().registerAlias(name, alias);
      }
      catch (Exception ex) {
         getReaderContext().error("Failed to register alias '" + alias +
               "' for bean with name '" + name + "'", ele, ex);
      }
      //在解析完<Alias>元素之后，发送容器别名处理完成事件
      getReaderContext().fireAliasRegistered(name, alias, extractSource(ele));
   }
}
```

processBeanDefinition()

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
   // BeanDefinitionHolder是对BeanDefinition的封装，即Bean定义的封装类
   //对Document对象中<Bean>元素的解析由BeanDefinitionParserDelegate实现
   BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
   if (bdHolder != null) {
      bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
      try {
         // Register the final decorated instance.
         //向Spring IOC容器注册解析得到的Bean定义，这是Bean定义向IOC容器注册的入口
         BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
      }
      catch (BeanDefinitionStoreException ex) {
         getReaderContext().error("Failed to register bean definition with name '" +
               bdHolder.getBeanName() + "'", ele, ex);
      }
      // Send registration event.
      //在完成向Spring IOC容器注册解析得到的Bean定义之后，发送注册事件
      getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
   }
}
```

parseBeanDefinitionElement()

```java
@Nullable
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
   //获取<Bean>元素中的id属性值
   String id = ele.getAttribute(ID_ATTRIBUTE);
   //获取<Bean>元素中的name属性值
   String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

   //获取<Bean>元素中的alias属性值
   List<String> aliases = new ArrayList<>();

   //将<Bean>元素中的所有name属性值存放到别名中
   if (StringUtils.hasLength(nameAttr)) {
      String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
      aliases.addAll(Arrays.asList(nameArr));
   }

   String beanName = id;
   //如果<Bean>元素中没有配置id属性时，将别名中的第一个值赋值给beanName
   if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
      beanName = aliases.remove(0);
      if (logger.isDebugEnabled()) {
         logger.debug("No XML 'id' specified - using '" + beanName +
               "' as bean name and " + aliases + " as aliases");
      }
   }

   //检查<Bean>元素所配置的id或者name的唯一性，containingBean标识<Bean>
   //元素中是否包含子<Bean>元素
   if (containingBean == null) {
      //检查<Bean>元素所配置的id、name或者别名是否重复
      checkNameUniqueness(beanName, aliases, ele);
   }

   //详细对<Bean>元素中配置的Bean定义进行解析的地方
   AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
   if (beanDefinition != null) {
      if (!StringUtils.hasText(beanName)) {
         try {
            if (containingBean != null) {
               //如果<Bean>元素中没有配置id、别名或者name，且没有包含子元素
               //<Bean>元素，为解析的Bean生成一个唯一beanName并注册
               beanName = BeanDefinitionReaderUtils.generateBeanName(
                     beanDefinition, this.readerContext.getRegistry(), true);
            }
            else {
               //如果<Bean>元素中没有配置id、别名或者name，且包含了子元素
               //<Bean>元素，为解析的Bean使用别名向IOC容器注册
               beanName = this.readerContext.generateBeanName(beanDefinition);
               // Register an alias for the plain bean class name, if still possible,
               // if the generator returned the class name plus a suffix.
               // This is expected for Spring 1.2/2.0 backwards compatibility.
               //为解析的Bean使用别名注册时，为了向后兼容
               //Spring1.2/2.0，给别名添加类名后缀
               String beanClassName = beanDefinition.getBeanClassName();
               if (beanClassName != null &&
                     beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
                     !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
                  aliases.add(beanClassName);
               }
            }
            if (logger.isDebugEnabled()) {
               logger.debug("Neither XML 'id' nor 'name' specified - " +
                     "using generated bean name [" + beanName + "]");
            }
         }
         catch (Exception ex) {
            error(ex.getMessage(), ele);
            return null;
         }
      }
      String[] aliasesArray = StringUtils.toStringArray(aliases);
      return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
   }
   //当解析出错时，返回null
   return null;
}
```

​	通过parseBeanDefinitionElement解析成BeanDefinitionHolder后，就向IOC容器进行注册。

registerBeanDefinition()

```java
public static void registerBeanDefinition(
      BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
      throws BeanDefinitionStoreException {

   // Register bean definition under primary name.
   //获取解析的BeanDefinition的名称
   String beanName = definitionHolder.getBeanName();
   //向IOC容器注册BeanDefinition
   registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

   // Register aliases for bean name, if any.
   //如果解析的BeanDefinition有别名，向容器为其注册别名
   String[] aliases = definitionHolder.getAliases();
   if (aliases != null) {
      for (String alias : aliases) {
         registry.registerAlias(beanName, alias);
      }
   }
}
```

registerBeanDefinition()

```java
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
      throws BeanDefinitionStoreException {

   Assert.hasText(beanName, "Bean name must not be empty");
   Assert.notNull(beanDefinition, "BeanDefinition must not be null");

   //校验解析的BeanDefiniton
   if (beanDefinition instanceof AbstractBeanDefinition) {
      try {
         ((AbstractBeanDefinition) beanDefinition).validate();
      }
      catch (BeanDefinitionValidationException ex) {
         throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
               "Validation of bean definition failed", ex);
      }
   }

   BeanDefinition oldBeanDefinition;

   oldBeanDefinition = this.beanDefinitionMap.get(beanName);

   if (oldBeanDefinition != null) {
      if (!isAllowBeanDefinitionOverriding()) {
         throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
               "Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
               "': There is already [" + oldBeanDefinition + "] bound.");
      }
      else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
         // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
         if (this.logger.isWarnEnabled()) {
            this.logger.warn("Overriding user-defined bean definition for bean '" + beanName +
                  "' with a framework-generated bean definition: replacing [" +
                  oldBeanDefinition + "] with [" + beanDefinition + "]");
         }
      }
      else if (!beanDefinition.equals(oldBeanDefinition)) {
         if (this.logger.isInfoEnabled()) {
            this.logger.info("Overriding bean definition for bean '" + beanName +
                  "' with a different definition: replacing [" + oldBeanDefinition +
                  "] with [" + beanDefinition + "]");
         }
      }
      else {
         if (this.logger.isDebugEnabled()) {
            this.logger.debug("Overriding bean definition for bean '" + beanName +
                  "' with an equivalent definition: replacing [" + oldBeanDefinition +
                  "] with [" + beanDefinition + "]");
         }
      }
      this.beanDefinitionMap.put(beanName, beanDefinition);
   }
   else {
      if (hasBeanCreationStarted()) {
         // Cannot modify startup-time collection elements anymore (for stable iteration)
         //注册的过程中需要线程同步，以保证数据的一致性
         synchronized (this.beanDefinitionMap) {
            this.beanDefinitionMap.put(beanName, beanDefinition);
            List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
            updatedDefinitions.addAll(this.beanDefinitionNames);
            updatedDefinitions.add(beanName);
            this.beanDefinitionNames = updatedDefinitions;
            if (this.manualSingletonNames.contains(beanName)) {
               Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
               updatedSingletons.remove(beanName);
               this.manualSingletonNames = updatedSingletons;
            }
         }
      }
      else {
         // Still in startup registration phase
        //仍处于注册启动阶段。
         this.beanDefinitionMap.put(beanName, beanDefinition);
         this.beanDefinitionNames.add(beanName);
         this.manualSingletonNames.remove(beanName);
      }
      this.frozenBeanDefinitionNames = null;
   }

   //检查是否有同名的BeanDefinition已经在IOC容器中注册
   if (oldBeanDefinition != null || containsSingleton(beanName)) {
      //重置所有已经注册过的BeanDefinition的缓存
      resetBeanDefinition(beanName);
   }
}
```

​	此时IOC容器中，已经有所有定义的Bean的信息。注意，此时Bean尚未实例化，BeanDefinition只是用来描述一个bean实例。

parseCustomElement()

​	根据自定义的规则去解析

```java
//根据定义的URL，生成一个对应的命名解析器，根据对应的规则解析。
public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
   String namespaceUri = getNamespaceURI(ele);
   if (namespaceUri == null) {
      return null;
   }
   NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
   if (handler == null) {
      error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
      return null;
   }
   return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}
```

postProcessXml(root);

```java
//在我们处理完Bean定义之后，允许XML通过最后处理任何自定义元素类型来进行扩展。这个方法是对XML的任何其他自定义后处理的XML的自然扩展点。
//默认的实现是空的。子类可以覆盖这个方法来将自定义元素转换为标准的Spring bean定义等。实现者可以通过相应的访问器访问解析器的Bean定义阅读器和底层的XML资源。
protected void postProcessXml(Element root) {
}
```

