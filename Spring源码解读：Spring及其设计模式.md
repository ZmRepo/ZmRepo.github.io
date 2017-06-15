# Spring源码解读：Spring及其设计模式
之前两篇简单对Spring的IoC特性做了源码级的流程梳理，但这仅停留在了解、使用层面；Spring这么庞大的一个框架，如何在易用性、易扩展等方面发力，才是我们研究源码的核心。为了提高对框架代码的架构能力，本文试着从设计的角度去剖析源码，从中学习Spring的代码架构及设计原则。

### 工厂模式
Spring是面向Bean的编程，而之前源码分析中，bean的创建采用了工厂模式，他的顶级接口是beanfactory，并有三个子接口：ListableBeanFactory、HierarchicalBeanFactory 和AutowireCapableBeanFactor。这三个接口主要是为了区分在 Spring 内部在操作过程中对象的传递和转化过程中，对对象的数据访问所做的限制。上述三种工厂类的功能大部分是通过DefaultListableBeanFactory和AbstractBeanFactory实现的。
BeanFactory提供了工厂模式最基础的接口，包括4个获取实例的方法、4个判断的方法以及1个获取类型和别名的方法。
``` java
public interface BeanFactory {
    /**
     * 用来引用一个实例，或把它和工厂产生的Bean区分开，就是说，如果一个FactoryBean的名字为a，那么，$a会得到那个Factory
     */
    String FACTORY_BEAN_PREFIX = "&";

    //4个获取实例的方法
    Object getBean(String name) throws BeansException;
    <T> T getBean(String name, Class<T> requiredType) throws BeansException;
    <T> T getBean(Class<T> requiredType) throws BeansException; 
    Object getBean(String name, Object... args) throws BeansException;

    //4个判断的方法
    boolean containsBean(String name); // 是否存在
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;// 是否为单实例 
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;// 是否为原型（多实例）
    boolean isTypeMatch(String name, Class<?> targetType)
            throws NoSuchBeanDefinitionException;// 名称、类型是否匹配

    //1个获取类型和别名的方法 
    Class<?> getType(String name) throws NoSuchBeanDefinitionException; // 获取类型 
    String[] getAliases(String name);// 根据实例的名字获取实例的别名
}
```
此外，基于BeanFactory的三个子接口，每个都有他的特性：
ListableBeanFactory：能够一次性列举所有他们bean实例，而不是试图根据客户端请求一个一个通过名字查找的工厂容器扩展接口；如果用户需要实现一个预先加载所有bean定义的工厂需要实现该接口；
HierarchicalBeanFactory：实现工厂的分层，使工厂结构更为清晰；

工厂模式优点：面向抽象接口编程，增强代码可维护性。

### 单例模式
Spring中默认Bean的创建采用单例模式，确保整个容器中某个类只存在唯一实例，容器可以灵活管控Bean的生命周期。Spring构造单例Bean依赖单例注册表来实现，通过查看容器getBean()实现
``` java
protected <T> T doGetBean(final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly) {
    Object bean;
    // 从单例注册表中检查是否存在单例缓存
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        // 返回缓存实例 
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    } else {
         // 单例模式，处理分支
         if (mbd.isSingleton()) {
             sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                 @Override
                 public Object getObject() throws BeansException{
                     try {
                         return createBean(beanName, mbd, args);
                     } catch (BeansException ex) {
                     }
                 }
             });
             bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        }
}
```
首先工厂先从单例注册表中检查是否存在单例缓存，若存在，返回缓存实例即可；若不存在，通过将工厂函数传入getSingleton函数中就可以获取Bean单例，看看核心代码getSingleton()
``` java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(beanName, "'beanName' must not be null");
    synchronized (this.singletonObjects) {
        // 检查缓存中是否存在实例  
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            try {
                singletonObject = singletonFactory.getObject();
            } catch (BeanCreationException ex) {
            } finally {
            }
            // 如果实例对象在不存在，我们注册到单例注册表中。
            addSingleton(beanName, singletonObject);
        }
        return (singletonObject != NULL_OBJECT ? singletonObject : null);
    }
}
```
由源码看出，Spring Bean单例依赖Map实现的单例注册表来实现，一旦创建该实例随即缓存到注册表，防止出现二次加载，消耗系统资源。
单例模式优势：Spring容器可以管理这些Bean的生命期，创建销毁时间及操作都交给容器控制；单例的Bean可以设置全局访问点，优化共享资源，减小系统资源消耗。

### 策略模式
如果Bean实例需要访问资源时，Spring有两种解决办法，分别是从代码中获取Resource资源以及依赖注入；对于第一种方法，在使用时总需要提供 Resource 所在的位置。这意味着：资源所在的物理位置将被耦合到代码中，如果资源位置发生改变，则必须改写程序。因此，Spring提供了第二种方式——依赖注入。采用依赖注入，允许动态配置资源文件位置，无须将资源文件位置写在代码中，当资源文件位置发生变化时，无须改写程序，直接修改配置文件即可。例如：下方这段代码就将classpath:book.xml注入到了com.example.Test类的tmp属性中。
``` xml
<beans> 
   <bean id="test" class="com.example.Test"> 
   <!-- 注入资源 --> 
      <property name="tmp" value="classpath:book.xml"/>
   </bean> 
</beans>
```
在依赖注入的过程中，Spring会调用ApplicationContext 来获取Resource的实例。然而，Resource 接口封装了各种可能的资源类型，包括了：UrlResource，ClassPathResource，FileSystemResource等，Spring需要针对不同的资源采取不同的访问策略。在这里，Spring让ApplicationContext成为了资源访问策略的“决策者”。在资源访问策略的选择上，Spring采用了策略模式。当 Spring 应用需要进行资源访问时，它并不需要直接使用 Resource 实现类，而是调用 ApplicationContext 实例的 getResource() 方法来获得资源，ApplicationContext 将会负责选择 Resource 的实现类，也就是确定具体的资源访问策略，从而将应用程序和具体的资源访问策略分离开来。我们来看看Spring实现类DefaultResourceLoader中是如何用一个接口方法加载两种资源的：
``` java
public Resource getResource(String location) {  
    Assert.notNull(location, "Location must not be null");  
    if (location.startsWith(CLASSPATH_URL_PREFIX)) {  
        return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());  
    }  
    else {  
        try {  
            URL url = new URL(location);  
            return new UrlResource(url);  
        }  
        catch (MalformedURLException ex) {  
            return getResourceByPath(location);  
        }  
    }  
}  
```
getResource()方法的传入参数，就作为一种策略，要求实现类去判断这种策略的类型，并返回相应的Resource。在这里，Spring是通过try-catch的形式，先判断是否是ClassPathContextResource，若不是，则把它当作是UrlResource来处理。
 

### 模板方法
模板方法是实现扩展开发修改关闭的最直接的方式，Spring中几乎处处都用到了模板方法设计模式。比如JdbcTemplate及容器加载过程中都使用到了模板方法设计模式。以容器加载为例，首先在接口ConfigurableApplicationContext中声明模板方法refresh()，由抽象类AbstractApplicationContext实现该接口及该模板方法refresh的逻辑，代码如下：
``` java
    public void refresh() throws BeansException, IllegalStateException {
        synchronized (this.startupShutdownMonitor) {
            prepareRefresh();
　　　　　　  //注意这个方法是，里面调用了两个抽象方法refreshBeanFactory、getBeanFactory
            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
            prepareBeanFactory(beanFactory);
            try {
　　　　　　　　　 //注意这个方法是钩子方法
                postProcessBeanFactory(beanFactory);
                invokeBeanFactoryPostProcessors(beanFactory);
                registerBeanPostProcessors(beanFactory);
                initMessageSource();
                initApplicationEventMulticaster();

                onRefresh();
                registerListeners();
                finishBeanFactoryInitialization(beanFactory);
                finishRefresh();
            }
            catch (BeansException ex) {
                destroyBeans();
                cancelRefresh(ex);
                throw ex;
            }
        }
    }
```

这里最主要有一个抽象方法obtainFreshBeanFactory、两个钩子方法postProcessBeanFactory和onRefresh，我们主要看一下获取Spring容器的抽象方法，其实他内部只调用了2个抽象方法：
``` java
　　protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
        refreshBeanFactory();
        ConfigurableListableBeanFactory beanFactory = getBeanFactory();
        if (logger.isDebugEnabled()) {
            logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
        }
        return beanFactory;
    }
　　protected abstract void refreshBeanFactory() throws BeansException, IllegalStateException;
　　public abstract ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
```
具体要取那种BeanFactory容器的决定权交给了子类！我们查看具体实现了抽象方法getBeanFactory的子类有：AbstractRefreshableApplicationContext：
``` java
public abstract class AbstractRefreshableApplicationContext extends AbstractApplicationContext {
    @Override
    public final ConfigurableListableBeanFactory getBeanFactory() {
        synchronized (this.beanFactoryMonitor) {
            if (this.beanFactory == null) {
                throw new IllegalStateException("BeanFactory not initialized or already closed - " +
                        "call 'refresh' before accessing beans via the ApplicationContext");
            }
            //这里的this.beanFactory在另一个抽象方法refreshBeanFactory的设置的
            return this.beanFactory;
        }
    }
}   
```
梳理一下整个流程，模板方法设计模式可以说是一种代码复用技术的演变，我们可以将公共的行为抽离出来在父类实现，并由父类实现整个流程的演进，而由子类来实现具体细节的处理，同时子类可以覆盖父类的基础方法来实现扩展，但并不影响代码的流程。

### 代理模式
代理模式在Spring AOP特性中尤为重要，后续在研读AOP源码后会完善该代理模式的研究。
