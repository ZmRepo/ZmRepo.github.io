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

- getFileExtensions()是返回支持的文件扩展名，比如PropertiesPropertySourceLoader支持的扩展名是xml及properties；
- ConfigFileApplicationListener定义了默认的文件名DEFAULT_NAMES=”application”，所以SpringBoot会根据文件名加扩展名来加载文件。
- Load()方法会读取配置文件并返回PropertySource，交由SpringBoot读取配置项，合并到总的配置对象中。

##自定义PropertySourceLoader
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

SpringBoot有效对配置做了简化，提供对properties及yaml语法的支持，及自定义语法的扩展，接下来梳理一下常见的功能配置与使用
### 数据库配置使用
在application.properties文件中添加数据源及jpa配置：
``` java
spring.datasource.driver-class-name=com.ibm.db2.jcc.DB2Driver 
spring.datasource.url=jdbc:db2://197.3.187.1:60000/mceprd 
spring.datasource.username=db2md 
spring.datasource.password=db2mafdev spring.datasource.schema=ff  
spring.jpa.properties.hibernate.hbm2ddl.auto=update
```

创建一个User实体，包含三个属性id，name和age，User实体和DB2数据库的user表相对应。
``` java
@Entity
@Table(name=”user”) 
public class User {     
    @Id     
    @GeneratedValue     
    private Long id;      
    @Column(nullable = false)     
    private String name;      
    @Column(nullable = false)     
    private Integer age;      
    protected User() {super();}     
    public User(String name, Integer age) {         
        super();         
        this.name = name;         
        this.age = age;     
    }
}
```

接下来定义User实体的数据访问层UserRepository，只需继承JpaRepository即可，该接口已经实现了部分基础的操作，用户也可以自定义方法：
``` java
public interface UserRepository extends JpaRepository<User, Long> {     
    User findByName(String name);     
    User findByNameAndAge(String name, Integer age);
    @Query("select u from User u where u.name= :name")     
    User findUser(@Param("name") String name); 
}
```
就这样，通过简单的配置就完成了基础的数据库连接及操作逻辑。

### 日志配置使用
Spring Boot默认使用Logback进行日志记录，并会加载默认的Logback配置文件base.xml，并默认提供两个输出端的配置文件console-appender.xml和file-appender.xml，base.xml中引用了这两个配置文件：
``` xml
<included>    
  <include   resource="org/springframework/boot/logging/logback/defaults.xml" />    
  <property name="LOG_FILE" value="${LOG_FILE:-${LOG_PATH:-${LOG_TEMP:-${java.io.tmpdir:-/tmp}}/}spring.log}"/>    
  <include resource="org/springframework/boot/logging/logback/console-appender.xml" />    
  <include resource="org/springframework/boot/logging/logback/file-appender.xml" />    
  <root level="INFO">       
    <appender-ref ref="CONSOLE" />       
    <appender-ref ref="FILE" />    
  </root> 
</included>
```
通过application.properties对Logback进行配置，可以指定日志文件的位置及名称，并可对不同的包或jar制定日志级别，配置如下：
``` java
logging.file=log.log
logging.path=d:\\log 
logging.level.com.cmbc.firefly.server.*=debug
```

### 应用系统配置
如上所述，Spring Boot内置一些基础的配置，并进行一定的语法约束，比如未提到的命令行参数、系统环境变量设定等等，同时他还提供用户自定义Properties，如下所示，我们在文件中添加url属性：
``` java
security.cfca.url=197.3.187.1 
security.cfca.port=8004
```
然后通过@Value(“${“属性名”}”)注解来加载对应的配置属性，具体如下：
``` java
@Component 
public class SecurityProperties {     
    @Value("${security.firefly}")     
    private String url;          
    @Value("${security.firefly}")     
    private String port; 
}
```
