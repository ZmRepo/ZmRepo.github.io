# SpringBoot配置加载机制及扩展

SpringBoot既能兼顾Spring的强大功能，又能简化繁琐的xml配置，只需在application.properties中完成一些属性配置即可开启各模块功能，实现快速、敏捷开发Web应用程序。本文主要解析了SpringBoot加载配置文件的逻辑，并实现了自定义对Json语法的配置支持。最后对配置文件的使用做了示例说明。

## 配置加载机制
Spring Boot启动通过SpringApplication类的run方法启动服务，如下所示：
``` java
SpringApplication.run(DBController.class, args);
```

SpringApplication在初始化的时候会加载spring.factories配置的ApplicationListener接口的实现类，其中就包括加载配置文件入口的ConfigFileApplicationListener，spring.factories的配置项如下，它定义了propertyLoader的实现类及listener的实现类，由框架自动加载：
``` java
# PropertySource Loaders org.springframework.boot.env.PropertySourceLoader=\ 
org.springframework.boot.env.PropertiesPropertySourceLoader,\ 
org.springframework.boot.env.YamlPropertySourceLoader  

# Application Listeners 
org.springframework.context.ApplicationListener=\ 
org.springframework.boot.builder.ParentContextCloserApplicationListener,\ 
org.springframework.boot.context.FileEncodingApplicationListener,\ 
org.springframework.boot.context.config.AnsiOutputApplicationListener,\ 
org.springframework.boot.context.config.ConfigFileApplicationListener,\ 
org.springframework.boot.context.config.DelegatingApplicationListener,\ 
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener,\ 
org.springframework.boot.logging.ClasspathLoggingApplicationListener,\
 org.springframework.boot.logging.LoggingApplicationListener
```

由上配置可知，框架默认实现对properties及yaml语法的支持，PropertySourceLoader就是加载配置文件的接口类：
``` java
public interface PropertySourceLoader {     
    String[] getFileExtensions();     
    PropertySource<?> load(String var1, Resource var2, String var3) throws IOException; 
}
```

getFileExtensions()是返回支持的文件扩展名，比如PropertiesPropertySourceLoader支持的扩展名是xml及properties；
ConfigFileApplicationListener定义了默认的文件名DEFAULT_NAMES=”application”，所以SpringBoot会根据文件名加扩展名来加载文件。
Load()方法会读取配置文件并返回PropertySource，交由SpringBoot读取配置项，合并到总的配置对象中。

## 自定义PropertySourceLoader
以上是SpringBoot加载配置文件的流程，很多时候应用程序需要自定义配置文件，所以需要自己实现接口类，并将该实现类配置到spring.factories中。比如SpringBoot并未实现加载json的配置文件，我们可以定义JsonPropertySourceLoader来实现对json配置文件的支持。
``` java
public class JsonPropertySourceLoader implements PropertySourceLoader { 
    public String[] getFileExtensions() { 
        return new String[] {"json"};     
    }     
    public PropertySource load(String name, Resource resource, String profile) {         
        return new MapPropertySource(name, new HashMap<String, Object>())     
    }
}
```

接下来在ClassPath路径下新建/META-INF/spring.factories文件，并配置JsonPropertySourceLoader即完成了自定义的配置文件加载。
``` java
# Ext PropertySource Loaders 
org.springframework.boot.env.PropertySourceLoader=com.cmbc.firefly.server.properties.JsonPropertySourceLoader
```
