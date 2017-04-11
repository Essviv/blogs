---
title: spring IoC源码分析(1)
author: essviv
date: 2017-04-10 19:39:00+0800
tags:
	- spring
	- ioc
	- DI
---

在JavaEE开发中，基于Spring开发是最目前最常见的方式. Spring框架不但提供了IoC, AOP等支持，在此基础上，还提供了一系列非常简单好用的模块，具体可以Spring官网上的[说明](http://spring.io/projects). 作为Spring框架的基础，IoC机制使得整个应用程序的各个基础元素间依赖关系的管理变得更加灵活易用，这里我们不对IoC机制的定义进行过多的阐述，而是试图从源码的角度来分析，Spring框架是如何实现所谓的"依赖反转".

## 0. 一切都要从BeanFactory说起

BeanFactory接口作为整个Spring容器的最基础的接口，它定义了一系列的管理bean组件的方法，具体如下图所示, 从图中可以看出，这个接口中定义了从容器中获取bean的方法getBean, 还定义了判断bean的作用域(scope)的方法, isPrototype和isSingleton， 除此之外，它还提供了方法来判断容器是否含有指定元素containsBean方法. 一般情况下，在应用程序中，beanFactory都是以单例模式中实现，获取到这个容器对象后，就可以通过它来管理应用程序中所有的bean对象以及它们之间的依赖关系. 

![beanFactoryMethods](https://raw.githubusercontent.com/Essviv/images/master/beanFactory-methods.jpg)

再来看看BeanFactory接口的类层次图，如下图所示，可以看到这个接口的子接口和实现有很多，其中最重要的接口为ApplicationContext接口，按Spring官方文档中的说法，ApplicatonContext接口提供了比BeanFactory更丰富的功能，包括国际化支持，资源获取，事件发布机制以及层次化的context结构. 

![beanFactory-hierarchy](https://raw.githubusercontent.com/Essviv/images/master/beanFactory-hierarchy.jpg)



## 1. 程序的入口

为了简化程序的代码，我们定义了TestController和TestService, 前者依赖于后者, 具体的代码如下, 从这里也可以看出，Spring框架自动管理了applicationContext.xml中定义的bean对象以及它们之间的依赖关系.

````java
//main.java
public class Main {
    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("applicationContext.xml");
        TestController testController = applicationContext.getBean(TestController.class);

        testController.test();
    }
}

//applicationContext.xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean name="testController" class="TestController">
        <property name="testService" ref="testService"/>
    </bean>

    <bean name="testService" class="TestService"/>
</beans>
      
//TestController
public class TestController {
    private TestService testService;

    public TestService getTestService() {
        return testService;
    }

    public void setTestService(TestService testService) {
        this.testService = testService;
    }

    public void test(){
        this.testService.sayHello();
    }
}

//TestService
public class TestService {
    public void sayHello(){
        System.out.println("hello world.");
    }
}

//output: hello word
````

接下来我们就来看看Spring是如何做到这一切的.

### 1.1 初始化

在上面的代码中，首先初始化了ApplicationContext接口的实例，在这里使用了ClassPathXmlApplicationContext类，除此之外，还可以使用FileSystemXmlApplicationContext, 两者的区别在于加载applicationContext.xml资源时的路径不同， ClassPathXmlApplicationContext类是从classpath目录下查找，而FileSystemXmlApplicationContext类是从文件系统中查找. 首先来看下ClassPathXmlApplicationContext的构造函数, 如下所示，在构造函数中首先设置了配置文件的路径，然后调用了refresh()方法, 可以这么说，refresh()方法实现了Spring框架中IoC机制的一大半功能，在这个方法中，完成了Bean对象的加载. 

````java
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}
````

那就来看看refresh()方法的实现， 源码如下所示，实现的内容很多，但整个逻辑非常地清楚（由衷地对作者表示佩服），整个refresh()方法大体上可以分为以下这些步骤：

1. 更新前的准备工作(prepareRefresh)
2. 读取Bean对象的定义(obtainFreshBeanFactory)
3. 初始化更新后的BeanFactory对象(prepareBeanFactory)
4. 生成BeanFacotry对象后的处理工作(postProcessBeanFactory)
5. 触发BeanFactoryPostProcessor接口（invokeBeanFactoryPostProcessors)
6. 注册BeanPostProcessor接口(registerBeanPostProcessor)
7. 初始化MessageSource(initMessageSource)
8. 初始化事件发布器(initApplicationEventMulticaster)
9. 触发refresh事件(onRefresh)
10. 注册ApplicationEvent事件监听器(registerListeners)
11. 结束BeanFactory的初始化工作(finishBeanFactoryInitialization)
12. 结束refresh操作(finishRefresh)
13. 清理缓存(resetCommonCaches)

````java
@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
````

#### 1.1.1 prepareRefresh

首先来看下prepareRefresh方法的实现，作为refresh()方法中的第一个步骤，这个方法主要的功能就是完成更新前的准备工作, 如记录更新开始的时间，更新容器的状态，设置环境等, 其中最重要的是完成PropertySource的初始化，默认情况下，initPropertySources()方法是个空实现，但是它给子类提供了定制化初始化过程的机会，这也可以认为是“模板模式”的应用（严格来讲，模板模式中的方法是抽象方法，但思想是一样的）, 在阅读Spring框架源码的过程中，可以发现有很多设计模式的实践，在遇到时会再次提及. 

````java
protected void prepareRefresh() {
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false);
		this.active.set(true);

		if (logger.isInfoEnabled()) {
			logger.info("Refreshing " + this);
		}

		// Initialize any placeholder property sources in the context environment
		initPropertySources();

		// Validate that all properties marked as required are resolvable
		// see ConfigurablePropertyResolver#setRequiredProperties
		getEnvironment().validateRequiredProperties();

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();
	}
````

#### 1.1.2 obtainFreshBeanFactory

这个方法中调用了refreshBeanFactory方法，后者完成了整个容器的Bean对象的加载工作，同样，这里的refreshBeanFactory方法也是个抽象方法，它的实现由子类提供，由于我们这里使用的是ClassPathXmlApplicationContext类，因此这个方法的具体实现由AbstractRefreshableApplicationContext类实现. 可以看到，refreshBeanFactory方法的步骤有：

* 判断当前是否已经有容器了，如果存在 ，则先删除原先容器中的对象，再关闭之前的容器(hasBeanFactory)
* 接着创建新的容器，并完成容器的初始化工作(createBeanFactory & customizeBeanFactory)
* 使用新的容器来加载和管理bean对象的定义(loadBeanDefinitions)
* 将新建的容器对象赋给ApplicationContext

````java
@Override
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
````

判断是否已经有BeanFactory以及关闭的操作相对比较简单，这里就略过，直接开始看创建新容器以及初始化的操作，可以看到这里生成的是DefaultListableBeanFactory对象，并在生成后根据配置设置参数allowBeanDefinitionOverriding和allowCircularReferences, 前者代表了是否允许重载bean的定义，后者定义了是否允许循环依赖. DefaultListableBeanFactory类是可以直接被使用的BeanFactory实现（不包括ApplicationContext接口的实现）.

````java
//创建新的BeanFactory
protected DefaultListableBeanFactory createBeanFactory() {
		return new DefaultListableBeanFactory(getInternalParentBeanFactory());
	}

//初始化
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
		if (this.allowBeanDefinitionOverriding != null) {
			beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
		}
		if (this.allowCircularReferences != null) {
			beanFactory.setAllowCircularReferences(this.allowCircularReferences);
		}
	}
````

接下来就是使用新的容器来加载和管理Bean对象的定义了. 在Spring框架中，bean对象的运行时定义是通过BeanDefinition类来实现的，这个类中定义了关于Bean对象的所有信息，包括构造方法参数，属性值，初始化方法，静态工厂方法等等信息. BeanDefinition之间还可以有继承关系，ChildBeanDefinition继承了ParentBeanDefinition的信息，也可以重写相应的属性. 在spring中，加载bean的信息就是通过这里的loadBeanDefinitions()方法来完成的, 同样，这里的loadBeanDefinitions方法又是个抽象方法，ClassPathXmlApplicationContext类中是通过AbstractXmlApplicationContext类中的方法来完成的，从这里也可以看出，Spring框架中大量使用了"模板模式"，从而将实现延迟到了具体的子类中，大大提高了框架的灵活性，这点在开发中，尤其是在框架开发中非常重要.

在loadBeanDefinitions的实现中，首先定义了XmlBeanDefinitionReader类，它是BeanDefinitionReader接口的实现，主要用于从xml文件中读取配置信息，接着就是配置这个xmlBeanDefinitionReader对象，然后做些初始化的工作，这里值得注意的是，设置了BeanDefinitionReader对象中的ResourceLoader为当前对象，即XmlClassPathApplicationContext对象. 最后就是使用这个xmlBeanDefinitionReader来读取配置文件了. 

从BeanDefinitionReader的实现中可以看出，它是一个个读取Resource文件和configLocations文件，其中，对于之前的配置来讲，Resource文件为空，而configLocation文件则是在构造ClassPathXmlApplicationContext对象时传入的参数值，即applicationContext.xml，因此BeanDefinitionReader会尝试从这些文件中获取相应的Bean信息. 

````java
@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
	}

//初始化BeanDefinitionReader
protected void initBeanDefinitionReader(XmlBeanDefinitionReader reader) {
		reader.setValidating(this.validating);
	}

//利用BeanDefinitionReader来读取配置文件
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
	}

//读取配置文件
protected String[] getConfigLocations() {
		return (this.configLocations != null ? this.configLocations : getDefaultConfigLocations());
	}
````

那么就来看看XmlBeanDefinitionReader是如何来加载Bean信息的. 代码虽然很长，但逻辑却很清楚，首先是获取到ResourceLoader对象，上面提及过，这里的ResourceLoader其实就是ClassPathXmlApplicationContext本身，然后通过这个对象读取location参数指定的资源信息，最后BeanDefinitionReader再从资源中读取相应的Bean定义信息. 

````java
@Override
	public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
		Assert.notNull(locations, "Location array must not be null");
		int counter = 0;
		for (String location : locations) {
			counter += loadBeanDefinitions(location);
		}
		return counter;
	}

//具体的加载过程 
public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
  		....省略一些代码
		ResourceLoader resourceLoader = getResourceLoader();
		Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
				int loadCount = loadBeanDefinitions(resources);
				if (actualResources != null) {
					for (Resource resource : resources) {
						actualResources.add(resource);
					}
				}
        ....省略一些代码
		}
	
````

那么再来看看BeanDefinitionReader又是如何从Resource资源对象中读取bean对象的定义的呢？这个方法的实现由XmlClassPathApplicationContext提供，它首先将Resource文件以Xml的格式读取，并形成相应的DOM, 然后调用registerBeanDefinitions()方法来注册bean. 在这个方法中，首先创建了一个BeanDefinitionDocumentReader对象，然后通过这个documentReader来加载Bean. 

````java
 public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		....省略一些代码
		InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
      ....省略一些代码
	}

//doLoadBeanDefinitions
Document doc = doLoadDocument(inputSource, resource);
return registerBeanDefinitions(doc, resource);

//registerBeanDefinitions
BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
int countBefore = getRegistry().getBeanDefinitionCount();
documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
return getRegistry().getBeanDefinitionCount() - countBefore;
````

再来看看这个documentReader又是怎么做的，中间的过程代码这里就直接忽略了，总的来讲documentReader对象在registerBeanDefinitions方法中调用了doRegisterBeanDefinitions()方法，后者完成了加载工作. 这里又可以看出Spring框架中方法命名的一个特点，基本上某个方法如果以do开始，那么这个方法就是实际完成工作的地方，而与它对应的那个方法只是做些协调配置的工作. 

在doRegisterBeanDefinition方法中，首先创建了个代理对象，在创建这个代理类的过程中，对一些默认值属性值进行了初始化，包括default-lay-init, default-merge, default-autowire等等属性，这些属性都是在<beans>元素中配置的属性，由于涉及到的属性较多，这里就不将全部的源码贴出，仅以lazyInit为例，如果设置的值为default，则会尝试从parent中读取相应的属性，并将属性值填充到delegate对象的defaults属性. 

````java
protected void doRegisterBeanDefinitions(Element root) {
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		...省略一些代码

		preProcessXml(root);
		parseBeanDefinitions(root, this.delegate);
		postProcessXml(root);

		this.delegate = parent;
	}

protected BeanDefinitionParserDelegate createDelegate(
			XmlReaderContext readerContext, Element root, BeanDefinitionParserDelegate parentDelegate) {

		BeanDefinitionParserDelegate delegate = new BeanDefinitionParserDelegate(readerContext);
		delegate.initDefaults(root, parentDelegate);
		return delegate;
	}

//delegate.initDefaults()方法的部分代码，以lazyInit为例，其它类似
String lazyInit = root.getAttribute(DEFAULT_LAZY_INIT_ATTRIBUTE);
		if (DEFAULT_VALUE.equals(lazyInit)) {
			// Potentially inherited from outer <beans> sections, otherwise falling back to false.
			lazyInit = (parentDefaults != null ? parentDefaults.getLazyInit() : FALSE_VALUE);
		}
		defaults.setLazyInit(lazyInit);
````

在完成delegate对象的初始化后，就由这个代理对象来解析bean的定义. 在这个方法中，首先判断根结点是否是默认的命名空间(即beans元素的命名空间http://www.springframework.org/schema/beans). 如果是则依次遍历根结点下的每个子结点，分别判断这些结点的命名空间，默认命名空间则调用parseDefaultElement()方法来解析，否则调用parseCustomElement()方法来解析. 

parseDefaultElemet()方法是用来处理默认命名空间中的元素，它只负责处理四类子结点，分别是import, alias, bean以及嵌套的beans结点. 而parseCustomElement()方法是用来解析非默认命名空间中定义的结点，如context，mvc等等，对于这些自定义的命名空间，首先调用NamespaceHandlerResolver类的resolve()方法获取命名空间与实际的NamespaceHandler之间的对应关系，根据命名空间获取相应的解析器，并由解析器完成相应的解析工作, 如context命名空间的解析器为NamespaceHandlerResolver类. 由于自定义的命名空间不胜枚举，这里就不再进行详细阐述，有需要的时候可以查看具体的实现. 

````java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					else {
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}

private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}

public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
		String namespaceUri = getNamespaceURI(ele);
		NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
		if (handler == null) {
			error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
			return null;
		}
		return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
	}
````

到此为止，就基本上完成了bean信息的加载过程，obtainFreshBeanFactory方法的执行也基本结束了，总结来看，容器类先是生成了XmlBeanDefinitionReader对象来读取Bean对象的定义，后者又将这个过程委托给BeanDefinitionDocumentReader来完成，而documentReader又委托给了BeanDefinitionParserDelegate，而这个delegate对象中真正完成了bean对象的加载.

````java
AbstractApplicationContext.loadBeanDefinitions -> XmlBeanDefinitionReader.loadBeanDefinitions -> BeanDefinitionDocumentReader.loadBeanDefinitions -> BeanDefinitionParserDelegate.loadBeanDefinitions -> parseDefaultElement, parseCustomElement
````

#### 1.1.3 prepareBeanFactory

可以看到，prepareBeanFactory方法首先设置beanFactory的属性值，包括ClassLoader, BeanExpressionResolver, PropertyEditorRegistrar, 然后增加了ApplicationContextAwareProcessor这个Aware接口，这就是为什么Spring框架中可以允许自动注入ApplicationContext对象的原因了.

接着，分别调用了ignoreDependencyInterface和registerResolvableDependency方法，前者的作用是忽略相应的接口依赖，这些接口会通过其它的方式完成注入，后者往容器中配置了依赖注入规则，当遇到这些类的依赖时，直接使用指定的对象， 例如需要依赖BeanFactory时，直接使用新生成的容器对象. 

最后，prepareBeanFactory方法就会在容器查找框架特定的类定义，如loadTimeWeaver， environment, systemProperties, systemEnvironment等类，如果找到这些Bean的定义，则先实例化这些对象. 

````java
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
		beanFactory.setBeanClassLoader(getClassLoader());
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
````

#### 1.1.4 postProcessBeanFactory

在AbstractApplicationContext中，这个方法的实现是空实现，但它给子类一个机会，在完成容器的更新后进行一些定制化的操作，例如在AbstractRefreshableWebApplicationContext的实现中，就会在这个方法中注册ServletContextAwareProcessor接口.

````java
//AbstractRefreshableWebApplicationContext
@Override
	protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		beanFactory.addBeanPostProcessor(new ServletContextAwareProcessor(this.servletContext, this.servletConfig));
		beanFactory.ignoreDependencyInterface(ServletContextAware.class);
		beanFactory.ignoreDependencyInterface(ServletConfigAware.class);

		WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, this.servletContext);
		WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, this.servletContext, this.servletConfig);
	}
````

#### 1.1.5 invokeBeanFactoryPostProcessors

从方法名就可以看出，这个方法的作用是触发容器中定义的BeanFactoryPostProcessor接口. BeanFactoryPostProcessor接口可以在容器加载完bean信息后对它们的属性进行修改，如进行属性值的替换等操作. Spring文档中告诉我们，框架会自动识别实现了BeanFactoryPostProcessor接口的bean对象，其中的秘密就在这里. 

````java
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}
````

这里主要来看下invokeBeanFactoryPostProcessors的执行过程. 这里的逻辑要分成两大部分：

1. 处理当前beanFactory中已经有的beanFactoryPostProcessor: 这部分已经通过方法的第二个参数传入. 在处理这部分接口对象时，首先要区分当前的BeanFactory对象是否是BeanDefinitionRegistry接口的实现，如果是, 则除了要处理BeanFactoryPostProcessor接口，还需要找到这些BeanFactoryPostProcessor中实现了BeanDefinitionRegistryPostProcessor接口的接口，并执行该接口的方法，否则直接遍历执行BeanFactoryPostProcessor接口即可. 
2. 处理当前beanFactory中定义的beanFactoryPostProcessor接口: 这部分的处理和第一部分基本上类似，但需要考虑PriorityOrdered, Ordered接口的优先性，先处理实现了PriorityOrdered的接口，再处理实现了Ordered的接口，然后是普通的接口. 

````java
public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

		//第一部分，处理参数beanFactoryPostProcessors中的接口，需考虑容器为BeanDefinitionRegistry时的情况
		Set<String> processedBeans = new HashSet<String>();

		if (beanFactory instanceof BeanDefinitionRegistry) {
			BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
			List<BeanFactoryPostProcessor> regularPostProcessors = new LinkedList<BeanFactoryPostProcessor>();
			List<BeanDefinitionRegistryPostProcessor> registryPostProcessors =
					new LinkedList<BeanDefinitionRegistryPostProcessor>();

			for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
				if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
					BeanDefinitionRegistryPostProcessor registryPostProcessor =
							(BeanDefinitionRegistryPostProcessor) postProcessor;
					registryPostProcessor.postProcessBeanDefinitionRegistry(registry);
					registryPostProcessors.add(registryPostProcessor);
				}
				else {
					regularPostProcessors.add(postProcessor);
				}
			}

			//第二部分，与第一部分的实现思路大体一致，但需要考虑PriorityOrdered， Ordered接口
			String[] postProcessorNames =
					beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);

			// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
			List<BeanDefinitionRegistryPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanDefinitionRegistryPostProcessor>();
			for (String ppName : postProcessorNames) {
				if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
					priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			sortPostProcessors(beanFactory, priorityOrderedPostProcessors);
			registryPostProcessors.addAll(priorityOrderedPostProcessors);
			invokeBeanDefinitionRegistryPostProcessors(priorityOrderedPostProcessors, registry);

			// Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			List<BeanDefinitionRegistryPostProcessor> orderedPostProcessors = new ArrayList<BeanDefinitionRegistryPostProcessor>();
			for (String ppName : postProcessorNames) {
				if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
					orderedPostProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			sortPostProcessors(beanFactory, orderedPostProcessors);
			registryPostProcessors.addAll(orderedPostProcessors);
			invokeBeanDefinitionRegistryPostProcessors(orderedPostProcessors, registry);

			// Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
			boolean reiterate = true;
			while (reiterate) {
				reiterate = false;
				postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
				for (String ppName : postProcessorNames) {
					if (!processedBeans.contains(ppName)) {
						BeanDefinitionRegistryPostProcessor pp = beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class);
						registryPostProcessors.add(pp);
						processedBeans.add(ppName);
						pp.postProcessBeanDefinitionRegistry(registry);
						reiterate = true;
					}
				}
			}

			// Now, invoke the postProcessBeanFactory callback of all processors handled so far.
			invokeBeanFactoryPostProcessors(registryPostProcessors, beanFactory);
			invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
		}

		else {
			// Invoke factory processors registered with the context instance.
			invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the bean factory post-processors apply to them!
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

		// Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		List<String> orderedPostProcessorNames = new ArrayList<String>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
		for (String ppName : postProcessorNames) {
			if (processedBeans.contains(ppName)) {
				// skip - already processed in first phase above
			}
			else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
		sortPostProcessors(beanFactory, priorityOrderedPostProcessors);
		invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

		// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
		List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		for (String postProcessorName : orderedPostProcessorNames) {
			orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		sortPostProcessors(beanFactory, orderedPostProcessors);
		invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

		// Finally, invoke all other BeanFactoryPostProcessors.
		List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		for (String postProcessorName : nonOrderedPostProcessorNames) {
			nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

		// Clear cached merged bean definitions since the post-processors might have
		// modified the original metadata, e.g. replacing placeholders in values...
		beanFactory.clearMetadataCache();
	}
````

#### 1.1.5 registerBeanPostProcessors

顾名思义，registerBeanPostProcessors方法就是往容器中注册BeanPostProcess接口，作为Spring容器的第二个扩展点，BeanPostProcessor接口允许应用程序对bean对象的实例化过程进行定制，例如，可以根据配置信息在实例化完成后产生相应的代理对象等等，事实上，Spring框架中提供的AOP机制就是通过BeanPostProcessor来实现的. 

具体来看看这个方法实现的过程. 总体来讲和之前的实现差异不大，仍然是按PriorityOrdered，Ordered以及普通接口分成三类，然后分别往容器中按一定的顺序注册BeanPostProcessor接口. 

到此为止，Spring容器已经完成了bean定义的加载，BeanFactoryPostProcessor接口的处理，并往容器中注册了BeanPostProcessor接口. 

````java
public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// Register BeanPostProcessorChecker that logs an info message when
		// a bean is created during BeanPostProcessor instantiation, i.e. when
		// a bean is not eligible for getting processed by all BeanPostProcessors.
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// Separate between BeanPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<BeanPostProcessor>();
		List<String> orderedPostProcessorNames = new ArrayList<String>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, register the BeanPostProcessors that implement PriorityOrdered.
		sortPostProcessors(beanFactory, priorityOrderedPostProcessors);
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// Next, register the BeanPostProcessors that implement Ordered.
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		sortPostProcessors(beanFactory, orderedPostProcessors);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// Now, register all regular BeanPostProcessors.
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// Finally, re-register all internal BeanPostProcessors.
		sortPostProcessors(beanFactory, internalPostProcessors);
		registerBeanPostProcessors(beanFactory, internalPostProcessors);

		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
	}
````

#### 1.1.6 initMessageSource

Spring框架关于国际化的支持主要是通过MessageSource接口来完成的，它允许应用程序根据不同的地区显示不同的消息，在initMessageSource方法中，首先查看容器中是否定义了MessageSource接口的实现，如果没有定义这个接口的实现，则使用默认的MessageSource实现DelegatingMessageSource.

````java
	protected void initMessageSource() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
			this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
			// Make MessageSource aware of parent MessageSource.
			if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
				HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
				if (hms.getParentMessageSource() == null) {
					// Only set parent context as parent MessageSource if no parent MessageSource
					// registered already.
					hms.setParentMessageSource(getInternalParentMessageSource());
				}
			}
		}
		else {  //默认情况，使用空的MessageSource实现
			DelegatingMessageSource dms = new DelegatingMessageSource();
			dms.setParentMessageSource(getInternalParentMessageSource());
			this.messageSource = dms;
			beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
		}
	}
````

#### 1.1.7 initApplicationEventMulticaster

这个方法主要是负责容器的ApplicationEventMulticaster对象的初始化工作，实现的逻辑与上面基本一致 ，首先判断容器中是否已经定义了这个对象，如果没有则使用默认的实现. 在Spring框架中，ApplicationEventMulticater接口主要应用程序事件的广播，可以往容器中注册ApplicationEvent接口及ApplicationListener接口，两者分别代表了事件及事件监听器，当发生相应的事件后，就可以通过ApplicationEventMulticaster接口来通知相应的监听器进行处理.

````java
	protected void initApplicationEventMulticaster() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
			this.applicationEventMulticaster =
					beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
		}
		else {
			this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
			beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
		}
	}
````

#### 1.1.8 onRefresh

onRefresh()方法作为容器初始化过程中的"钩子"方法，允许子类在容器完成初始化后进行一些定制化的操作，如注册一些特定的对象等. 默认情况下为空实现.

#### 1.1.9 registerListeners

在上面已经提到过，Spring容器支持事件的触发与广播，主要是通过ApplicationEventMulticater来实现的，但前提是容器中必须注册相应的事件及监听器，而这一步就是在这里完成. 这里首先获取容器中注册的ApplicationListener接口对象，在ApplicationEventMulticaster中完成注册，在注册完成后，检查初始化过程中是否已经有事件产生，如果有，则直接触发这些事件. 

````java
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
````

#### 1.1.10 finishBeanFactoryInitialization

finishBeanFactoryInitialization方法中主要完成容器初始化完成后的收尾工作，包括配置ConversionService, 注册LoadTimeWeaverAware接口，冻结配置以及初始化对象实例等操作. ConversionService是Spring容器中进行类型转换的入口, 所有涉及到类型转换的场景， 如配置文件中的值到属性类型间的转换，都是从ConversionService完成的. 除此之外，这个方法中还对当前的配置进行了冻结，以示可以进行缓存操作. 

````java
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
				@Override
				public String resolveStringValue(String strVal) {
					return getEnvironment().resolvePlaceholders(strVal);
				}
			});
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		beanFactory.preInstantiateSingletons();
	}
````

在finishBeanFactoryInitialization方法的最后一步，调用了preInstaniateSingletons()方法. Spring文档中告诉我们，对于作用域为singleton的对象，lazy-init属性默认为false, 意味着容器总是会“预先”初始化这些对象，而这步就是在preInstantiateSingletons()这个方法中完成的.

这又是一个抽象方法，它的实现由DefaultListableBeanFactory类提供. 可以看到，预先初始化的对象需要满足三个条件：

* 不是抽象类
* scope为singleton
* lazyInit属性为false

在满足了以上三点后，容器就会尝试初始化这些singleton对象，首先判断这个类定义是否是FactoryBean，如果是的话，则再次判断它的eagerInit属性，只要在这个属性为true的情况下才开始实例化操作. 如果这个bean定义是普通的对象，则直接开始实例化操作. 对象的实例化操作都是通过getBean()方法来完成的，关于这个方法会在本文的第二部分进行详细的阐述，这里就不做叙述.

在完成对象的实例化操作后， 还会调用实现了SmartInitializingSingleton接口的对象的回调方法. 

````java
	public void preInstantiateSingletons() throws BeansException {
				// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
							@Override
							public Boolean run() {
								return ((SmartFactoryBean<?>) factory).isEagerInit();
							}
						}, getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						getBean(beanName);
					}
				}
				else {
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged(new PrivilegedAction<Object>() {
						@Override
						public Object run() {
							smartSingleton.afterSingletonsInstantiated();
							return null;
						}
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
````

#### 1.1.11 finishRefresh

到这里，容器的初始化工作已经基本完成，这个方法主要是完成LifecycleProcessor接口的配置与调用，并发布ContextRefreshEvent事件. LifecycleProcessor接口主要是负责与生命周期相关的方法，与之前一些接口的配置过程类似，这里也是先在容器中查询是否有相应的接口定义，如果没有则使用默认的实现. 在配置完LifecycleProcessor接口后，就会调用其onRefresh()方法，并发布ContextRefreshEvent事件. 这里值得注意的是getBean()方法，关于这个方法的实现过程会在本文的第二部分中进行详细阐述, 这里只要知道它是从容器中获取相应的实例对象即可. 

````java
	protected void finishRefresh() {
		// Initialize lifecycle processor for this context.
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}

	protected void initLifecycleProcessor() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {
			this.lifecycleProcessor =
					beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);
		}
		else {
			DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
			defaultProcessor.setBeanFactory(beanFactory);
			this.lifecycleProcessor = defaultProcessor;
			beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);
		}
	}
````

到此为止，示例代码中构造ApplicationContext对象的源码就已经全部解读完毕了，总得来讲，在构造函数中调用了refresh()方法来加载配置文件中定义的BeanDefinition，并根据配置文件注册了BeanPostProcessor接口，以及一些配置接口，在加载完bean定义后，还对lazyInit为false的"单例"对象进行了实例化操作. 在这个构造函数返回后，应用程序定义的所有对象以及它们之间的依赖关系就已经在容器中配置好了. 

对象间的依赖关系是在调用getBean()方法获取对象时由容器根据定义自动满足的，这部分内容见本文的第二部分.

````java
ApplicationContext applicationContext = new ClassPathXmlApplicationContext("applicationContext.xml");
````

