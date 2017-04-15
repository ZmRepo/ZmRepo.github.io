# Spring源码解读：IoC容器初始化流程

在面向对象系统中，对象封装了数据及处理方法，对象的依赖关系常常体现在对数据和方法的依赖上。这些以来关系可以通过把对象的依赖注入交给框架或IoC容器来完成，这种设计模式即为控制反转。Spring IoC模块就是这个模式的一种实现，它提供一个基本的JavaBean容器，通过IoC模式管理依赖关系，并将这些依赖关系注入到组件中，那么会让这些依赖关系的适配和管理更加灵活。接下来从源码角度剖析Spring IoC容器初始化原理。

在Spring中，IoC容器主要通过接口类BeanFactory设定最基本的功能规范，并提供不同程度的实现类；在这些IoC容器的接口定义与实现基础上，Spring通过定义BeanDefinition来管理基于Spring的应用中的各种对象以及他们之间的相互依赖关系。
### BeanFactory接口设计
Spring Bean的创建是典型的工厂模式，这一系列的Bean工厂，即IoC容器为开发者管理对象间的依赖关系提供了很多便利和基础服务，在Spring中有很多的IoC容器的实现供用户选择和使用。其中BeanFactory作为最顶层的一个接口类，它定义了IoC容器的基本功能规范：
``` java
public interface BeanFactory {    
     //对FactoryBean的转义定义，因为如果使用bean的名字检索FactoryBean得到的对象是工厂生成的对象，    
     //如果需要得到工厂本身，需要转义           
     String FACTORY_BEAN_PREFIX = "&"; 
     //根据bean的名字，获取在IOC容器中得到bean实例    
     Object getBean(String name) throws BeansException;    
     //根据bean的名字和Class类型来得到bean实例，增加了类型安全验证机制。    
     Object getBean(String name, Class requiredType) throws BeansException;    
     //提供对bean的检索，看看是否在IOC容器有这个名字的bean    
     boolean containsBean(String name);    
     //根据bean名字得到bean实例，并同时判断这个bean是不是单例    
     boolean isSingleton(String name) throws NoSuchBeanDefinitionException;    
     //得到bean实例的Class类型    
     Class getType(String name) throws NoSuchBeanDefinitionException;
     //得到bean的别名，如果根据别名检索，那么其原名也会被检索出来    
     String[] getAliases(String name);    
}
```
该接口对容器的基本行为做了定义，而对于bean的加载方式则由具体的子类来实现。各种IOC容器都只是它的实现或者为了满足特别需求的扩展实现，包括我们平时用的最多的ApplicationContext。从上面的方法就可以看出，这些工厂的实现最大的作用就是根据bean的名称亦或类型等等，来返回一个bean的实例。

### BeanDefinition接口设计
Spring通过定义BeanDefinition来管理基于Spring的应用中的各种对象以及他们之间的相互依赖关系，它抽象了我们对Bean的定义，是让容器起作用的主要数据类型，IoC容器的依赖翻转都是围绕这个BeanDefinition的处理来完成的。
``` java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
    // 得到当前Bean名称
    String getBeanClassName();
    // 返回创建Bean的工厂名
    String getFactoryBeanName();
    // 当前Bean是否可以延迟加载
    boolean isLazyInit();
    void setLazyInit(boolean lazyInit);
    // 返回依赖的Bean名称
    String[] getDependsOn();
    void setDependsOn(String[] dependsOn);
    ConstructorArgumentValues getConstructorArgumentValues();
    MutablePropertyValues getPropertyValues();
    BeanDefinition getOriginatingBeanDefinition();
}
```
这是Spring中Bean的最顶层接口定义，上述工厂中持有的都是该接口的实现类，它继承的接口AttributeAccessor为Bean具有处理属性的能力，而另外一个BeanMetadataElement可以获得Bean配置定义的一个元素。该接口有个方法：String[] getDependsOn()，就是获取依赖的beanName，只要载入一个BeanDefinition，就可以产生和管理Bean实例。

### IoC容器初始化过程
了解了IoC容器的核心工厂接口BeanFactory及Bean定义接口BeanDefinition，接下来就不难理解Spring的初始化流程：解析、注册、管理。接下来我们以BeanFactory的简单实现类XmlBeanFactory为例，梳理整个初始化流程。
``` java
public class XmlBeanFactory extends DefaultListableBeanFactory {  
    private final XmlBeanDefinitionReader rd = new XmlBeanDefinitionReader(this);  
    public XmlBeanFactory(Resource resource) throws BeansException {  
        this(resource, null);  
    }   
    public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {  
        super(parentBeanFactory);  
        this.reader.loadBeanDefinitions(resource);  
    }
}
```
XmlBeanFactory类继承了DefaultListableBeanFactory类。看似XmlBeanFactory只是对两个类方法的封装。在上面XmlBeanFactory.java源码中，XmlBeanFactory类使用了XML读取器XmlBeanDefinitionReader。
#### Step 1：XmlBeanDefinitionReader读取配置资源对象
``` java
public int loadBeanDefinitions(InputSource inputSource) throws BeanDefinitionStoreException {  
    return loadBeanDefinitions(inputSource, "resource through SAX InputSource");
}

protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)  throws BeanDefinitionStoreException {  
    try {  
        Document doc = doLoadDocument(inputSource, resource);  
        return registerBeanDefinitions(doc, resource);  
    }
}

protected void doRegisterBeanDefinitions(Element root) {
    String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
    if (StringUtils.hasText(profileSpec)) {
        String[] specifiedProfiles = StringUtils.tokenizeToStringArray(profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
        if (!getEnvironment().acceptsProfiles(specifiedProfiles)) {
            return;
        }
    }
    BeanDefinitionParserDelegate parent = this.delegate;
    this.delegate = createDelegate(this.readerContext, root, parent);
    preProcessXml(root);
    parseBeanDefinitions(root, this.delegate);
    postProcessXml(root);
    this.delegate = parent;
}
```
该方法完成对资源文件流做了Document文件转化，并通过DefaultDocumentLoader类做解析，BeanDefinitionDocument解析器注册bean definition，从根节点beans开始，逐层解析Bean元素到BeanDefinition，为后续装载Bean对象提供资源。
#### Step 2：由DefaultListableBeanFactory实现bean注册
``` java
Public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) throws BeanDefinitionStoreException {
    if (beanDefinition instanceof AbstractBeanDefinition) {
        try {
            ((AbstractBeanDefinition) beanDefinition).validate();
        } catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(“”, beanName, "", ex);
	    }
    }
    synchronized (this.beanDefinitionMap) {
		Object oldBeanDefinition = this.beanDefinitionMap.get(beanName);
		if (oldBeanDefinition != null) {
			if (!this.allowBeanDefinitionOverriding) {
				throw new BeanDefinitionStoreException(“”, beanName, "");
			} else {
				if (this.logger.isInfoEnabled()) {
					this.logger.info("Overriding bean definition for bean '" + beanName);
				}
			}
		}
		else {
			this.beanDefinitionNames.add(beanName);
			this.frozenBeanDefinitionNames = null;
		}
		this.beanDefinitionMap.put(beanName, beanDefinition);
		resetBeanDefinition(beanName);
	}
}
```
该接口完成BeanDefinition向容器的注册，就是将解析的Bean定义存入BeanDefinitionNames和BeanDefinitionMap中去，处理的时候需要依据allowBeanDefinitionOverriding的配置来完成。在完成了BeanDefinition的注册，就完成了IoC容器的初始化过程。此时，在使用IoC容器DefaultListableBeanFactory中已经建立了整个Bean的配置信息，而且这些BeanDefinition已经可以被容器使用了，可以通过beanDefinitionMap里被检索和使用。容器的作用就是对这些信息进行处理和维护，是实现依赖反转的基础。

下一章节会对Spring依赖反转的原理进行解读。
