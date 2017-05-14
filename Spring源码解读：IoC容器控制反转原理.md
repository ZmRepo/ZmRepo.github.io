# Spring源码解读：IoC容器依赖注入

上一章节对通过阅读Spring源码对IoC容器的初始化做了解析，并以BeanFactory接口解析了简单容器基本的初始化流程。本文会从实际应用出发（比如Tesla框架），解析IoC另一种高级形态的容器ApplicationContext，并分析Spring依赖反转的原理。

-------------------


## Web应用启动

我们知道创建Spring容器有多种形式：

1、手动实例化ApplicationContext，比如：
``` java
ApplicationContext ctx = new ClassPathXmlApplicationContext("conf/appContext.xml");
```
2、利用第三方MVC框架的扩展点，或纯粹的DispatcherServlet应用，创建Spring容器；

3、通过配置文件，声明式创建Spring容器，比如Tesla框架运用Spring提供的ContextLoaderListener，让Spring容器随Web应用的启动而启动：
``` java
<listener>
  <listener-class>
    com.tesla.framework.core.springext.context.TeslaContextLoaderListener
  </listener-class>
</listener>
```
要想知道IoC容器何时初始化，首先我们从ContextLoaderListener入手，源码如下：
``` java
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
	private ContextLoader contextLoader;
	/**
	 * Initialize the root web application context.
	 */
	public void contextInitialized(ServletContextEvent event) {
		this.contextLoader = createContextLoader();
		if (this.contextLoader == null) 
			this.contextLoader = this;
		this.contextLoader.initWebApplicationContext(event.getServletContext());
	}
}
```
由于该类继承自ContextLoader类，即初始化时会执行ContextLoader类的静态代码块，具体代码比较简单，创建一个ClassPathResource对象，载入同目录下的ContextLoader.properties文件，用于创建默认的IoC容器：XmlWebApplicationContext。

接下来我们看一下initWebApplicationContext方法的过程：
``` java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
	long startTime = System.currentTimeMillis();
	try {
		// Store context in local instance variable, to guarantee that
		if (this.context == null) {
			this.context = createWebApplicationContext(servletContext);
		}
		servletContext.setAttribute(WebApplicationContext. CONTEXT, this.context);
		return this.context;
	}
	catch (RuntimeException ex) 
		throw ex;
}
protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
	Class<?> contextClass = determineContextClass(sc);
	return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
}
protected Class<?> determineContextClass(ServletContext servletContext) {
	String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
	if (contextClassName != null) {
		try {
			return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
		} catch (ClassNotFoundException ex) {
				throw new ApplicationContextException("", ex);
		}
	} else {
		contextClassName = defaultStrategies.getProperty(WebApplicationContext.getName());
		try {
			return ClassUtils.forName(contextClassName, ContextLoader. getClassLoader());
		} catch (ClassNotFoundException ex) {
				throw new ApplicationContextException(“", ex);
		}
	}
}
```
创建ApplicationContext的步骤为：

1、首先决定要创建的applicationContext容器类，从Web.xml里取出需要初始化的容器类名，如果取到，即用户已配置，则创建该容器的Class对象；未取到则创建默认容器的Class对象，即Spring默认配置的XmlWebApplicationContext

2、实例化ApplicationContext容器。

## Spring IoC容器初始化

Spring IoC容器初始化过程分为三步：资源定为、配置解析以及IoC容器注册，具体流程细节已在上一章节做解读，本文不再赘述

## Spring IoC容器依赖注入原理

众所周知，IoC容器的依赖反转核心是一个对象依赖的其他对象会通过被动的方式传递进来，而不是对象本身创建或者查找依赖对象。在容器初始化后已完成用户定义的Bean解析注册，并建立了bean之间的映射关系存储在beanDefinitionMap里被检索和使用，但并未对管理的Bean进行实例化，只有在以下两种情况下会触发依赖注入：

1、用户第一次向容器索要Bean时会触发

2、用户定义Bean资源中为元素配置了lazy-init属性，让容器在解析注册Bean时进行预实例化，触发依赖注入。

在BeanFactroy接口定义的Spring IoC容器基本功能规范中，有一个接口getBean()，这个接口的实现就是触发依赖注入的地方，先来看看其具体实现的源码：
``` java
protected <T> T doGetBean(
	final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly){
    final String beanName = transformedBeanName(name);
	Object bean;
    // 从缓存中读取是否已有被创建过的单态类型的Bean
	Object sharedInstance = getSingleton(beanName);
	if (sharedInstance != null && args == null) {
        // 获取给定Bean的实例对象，主要完成FactoryBean的相关处理
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	} else {
        // 接下来检查IoC容器中是否已有该Bean的定义，如果找不到则委托当前容器的父级容器进行查找
		BeanFactory parentBeanFactory = getParentBeanFactory();
        // 存在父容器并且当前容器不存在指定的Bean，则交给父级容器查找
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			String nameToLookup = originalBeanName(name);
			if (args != null) {
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			} else {
return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
		}
        // 获取当前Bean的所有依赖，若有，则地柜调用getBean方法并把依赖的Bean注册到当前Bean
        String[] dependsOn = mbd.getDependsOn();
		if (dependsOn != null) {
			for (String dependsOnBean : dependsOn) {
				getBean(dependsOnBean);
				registerDependentBean(dependsOnBean, beanName);
			}
		}
        if (mbd.isSingleton()) {
            // 创建单态模式Bean的实例对象
			sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
				public Object getObject() throws BeansException {
					try {
						return createBean(beanName, mbd, args);
					} catch (BeansException ex) {
						destroySingleton(beanName);
						throw ex;
					}
				}
			});
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
		} else if (mbd.isPrototype()) {
            // 创建原型模式Bean实例对象
Object prototypeInstance = null;
		    try {
				beforePrototypeCreation(beanName);
				prototypeInstance = createBean(beanName, mbd, args);
			} finally {
				afterPrototypeCreation(beanName);
			}
			bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
		}
	}
	return (T) bean;
}
```
通过分析上述源码，在Spring中如果Bean定义为单例模式（默认），则在创建Bean时先从缓存中查找，确保整个容器中只存在一个实例对象。若Bean定义为原型模式，则容器每次都会创建一个新的实例对象，除此之外Bean定义还可以扩展为指定生命周期范围。

该方法定义了根据Bean定义的模式，采用不同的创建Bean实例对象的策略，而具体Bean实例对象的创建过程由ObjectFactory接口的匿名内部类createBean方法完成，ObjectFactory使用委派模式，具体的Bean实例创建过程交给实现类AbstractAutowireCapableBeanFactory完成，继续分析createBean方法的源码：
``` java
protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) throws BeanCreationException {
    // 判断该Bean是否可以实例化，即是否可以通过当前类加载器加载
    resolveBeanClass(mbd, beanName);
    // Prepare method overrides.
	try {
		mbd.prepareMethodOverrides();
	} catch (BeanDefinitionValidationException ex) {
		throw new BeanDefinitionStoreException(mbd.getDescription(),beanName, "", ex);
	}
    // 创建Bean入口
    Object beanInstance = doCreateBean(beanName, mbd, args);
	return beanInstance;
}
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
	BeanWrapper instanceWrapper = null;
	if (mbd.isSingleton()) { // 单态模式的Bean，从容器缓存中获取同名Bean
		instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
	}
	if (instanceWrapper == null) {
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
    // 向容器中缓存单态模式的Bean对象，以防循环引用
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) { // 尽早持有对象的引用
		addSingletonFactory(beanName, new ObjectFactory() {
			public Object getObject() throws BeansException {
				return getEarlyBeanReference(beanName, mbd, bean);
			}
		});
	}
    // Bean对象的初始化，触发依赖注入
    Object exposedObject = bean;
	try {
        // 将Bean实例对象封装，并且Bean定义中配置的属性值赋值给实例对象
		populateBean(beanName, mbd, instanceWrapper);
		if (exposedObject != null) { // 初始化Bean对象
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
	} catch (Throwable ex) {
		throw (BeanCreationException) ex;
	} 
	// 注册完成依赖注入的Bean
	try {
		registerDisposableBeanIfNecessary(beanName, bean, mbd);
	} catch (BeanDefinitionValidationException ex) {
		throw new BeanCreationException(mbd.getDescription(), beanName, "", ex);
	}
return exposedObject;
}
```
通过以上源码分析，具体的依赖注入实现在以下两个方法中：

1、createBeanInstance()：生成Bean所包含的java对象实例

2、populateBean()：对Bean属性的依赖注入进行处理

在createBeanInstance()方法中，对象的生成有多种不同方式，可以通过工厂方法生成，也可以通过容器的自动装配特性生成java实例对象，创建对象源码如下：
``` java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
	//确保Bean可实例化
	Class beanClass = resolveBeanClass(mbd, beanName);
    // 调用工厂方法实例化
	if (mbd.getFactoryMethodName() != null)  {
		return instantiateUsingFactoryMethod(beanName, mbd, args);
	}
	// 调用容器的自动装配方法进行实例化
	boolean resolved = false;
	boolean autowireNecessary = false;
	if (args == null) {
		synchronized (mbd.constructorArgumentLock) {
			if (mbd.resolvedConstructorOrFactoryMethod != null) {
				resolved = true;
				autowireNecessary = mbd.constructorArgumentsResolved;
			}
		}
	}
	if (resolved) {
		if (autowireNecessary) {
			return autowireConstructor(beanName, mbd, null, null);
		} else {
			return instantiateBean(beanName, mbd);
		}
	}
	// 通过使用Bean的构造方法进行实例化
	Constructor[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
	if (ctors != null || !ObjectUtils.isEmpty(args))  {
		return autowireConstructor(beanName, mbd, ctors, args);
	}
// 使用默认的无参构造方法实例化
	return instantiateBean(beanName, mbd);
}
```
通过上述代码分析，对于使用工厂方法和自动装配特性的Bean实例化比较直观，调用相应的工厂方法或者参数匹配的构造方法即可完成实例化对象的工作，对于常用的默认无参构造方法就需要使用初始化策略（JDK反射等）进行初始化。

在容器初始化生成Bean所包含的Java实例对象过程后，它是如何将Bean的属性依赖关系注入到Bean实例对象中，上述提到对Bean属性的依赖注入进行处理方法是populateBean()，该方法代码如下：
``` java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
	// 获取Bean定义中设置的属性值
    PropertyValues pvs = mbd.getPropertyValues();
	if (bw == null) {
		if (!pvs.isEmpty()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName, "");
		} else {// 实例对象为空，属性值也为空，直接返回
			return;
		}
	}
    // 依赖注入开始，首先处理autowire自动装配的注入
	if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
		MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
			autowireByName(beanName, mbd, bw, newPvs);
		}
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			autowireByType(beanName, mbd, bw, newPvs);
		}
		pvs = newPvs;
	}
	// 对属性进行注入
	applyPropertyValues(beanName, mbd, bw, pvs);
}
```
上述代码告诉我们，注入分为自动装配的属性以及自定义属性，而依赖注入的过程就是Bean对象实例设置到它所依赖的Bean对象属性上去，通过代码可知依赖注入通过bw.setPropertyValue方法实现，该方法也使用了委托模式，在BeanWrapper接口中定义了方法声明，依赖注入的具体实现交由BeanWrapperImpl完成，源码如下：
``` java
private void setPropertyValue(PropertyTokenHolder tokens, PropertyValue pv) throws BeansException {
	String propertyName = tokens.canonicalName;
	String actualName = tokens.actualName;
	if (tokens.keys != null) {
		try { 
            // 获取属性值，调用属性的getter(readerMethod)方法
			propValue = getPropertyValue(getterTokens);
		} catch (NotReadablePropertyException ex) {
			throw new NotWritablePropertyException(getRootClass(),"", ex);
		}
		// 根据属性类型进行依赖注入
		String key = tokens.keys[tokens.keys.length - 1];
		if (propValue.getClass().isArray()) {
			…	
		} else if (propValue instanceof List) {
			…
		} else if (propValue instanceof Map) {
			…
		} else {
			throw new InvalidException(getRootClass(), this.nestedPath + propertyName, "");
		}
	} else {
        // 对非集合类型的属性注入
		PropertyDescriptor pd = pv.resolvedDescriptor;
		if (pd == null || !pd.getWriteMethod().getDeclaringClass().isInstance(this.object)) {
			…
			pv.getOriginalPropertyValue().resolvedDescriptor = pd;
		}
        Object oldValue = null;
		try {
            // 根据JDK的内省机制，获取属性的setter()方法
		    final Method writeMethod = (pd instanceof ?(()pd) () :pd ());
			final Object value = valueToApply;
			if (System.getSecurityManager() != null) {
				try {
                    // 将属性值设置到属性上去
					AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
						public Object run() throws Exception {
							writeMethod.invoke(object, value);
							return null;
						}
					}, acc);
				} catch (PrivilegedActionException ex) {
					throw ex.getException();
				}
			} else {
				writeMethod.invoke(this.object, value);
			}
		}
	}
}
```
上述源码表明Spring IoC容器是如何将属性值注入Bean实例中：

1、对于集合类属性，将其属性值解析为目标类型的集合后直接赋值

2、对于非集合类属性，大量使用了JDK的反射和内省机制，通过属性的setter方法为属性设置注入后的值。

在Bean创建和依赖注入完成之后，在IoC容器中建立起一系列依靠依赖关系联系起来的Bean，程序不需要应用系统手动创建所需对象，容器会在使用的时候自动为我们创建并注入好相关依赖，通过IoC相关接口方法，就可以非常方便地供上层应用系统使用。
