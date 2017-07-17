# JDK动态代理原理
对于Spring的核心AOP来说，核心原理就是java的动态代理机制，而动态代理模式也是较常用的设计模式之一，了解它的实现原理有助于我们提高自身代码架构水平。本文以Java中的动态代理运行机制出发，推演其内部实现原理。

### 代理模式及静态代理的局限
代理模式即给某个对象提供一个代理对象，并由代理对象控制对于源对象的访问，即客户不直接操控原对象，而是通过代理对象间接地操控原对象。通过使用代理模式，有两个优点：
1、可以隐藏源对象的实现细节；
2、可以实现客户与源对象的解藕，在不修改源对象代码情况下能够作一些额外的处理。
在实现细节上，如果代理类在程序运行前就已经存在，那么这种代理方式被称作静态代理，这种情况下的代理类通常都是我们在Java代码中定义的。通常情况下，静态代理中的代理类和委托类会实现同一接口或是派生自相同的父类。如下举一个简单的静态代理示例：
``` java
public interface Helloworld {
    void sayHello();
}
public class HelloworldImpl implements HelloWorld {
    public void sayHello() {
        System.out.print("hello world");
    }
}
public class HelloWorldAgent implements Helloworld {
    private HelloworldImpl helloworldImpl;
 
    public HelloWorldAgent(HelloworldImpl helloworldImpl) {
        this.helloworldImpl = helloworldImpl;
    }
    public void sayHello() {
        System.out.print("this is agent");
        helloworldImpl.sayHello();
    }
}
```
如上代码所示，HelloWorldAgent是我们实现的一个静态代理类，它同样实现的是顶层接口，但它代理的其实是属性helloworldImpl，使用者可以通过扩展sayHello()方法自定义具体的实现逻辑，而不需要更改代理类的代码。
上述例子完美展示了代理模式的优点：使用者与源对象解藕。但仔细想想，由于静态代理在编译期就已经决定，如果需要为多个类进行代理，我们需要撰写多个代理类，维护难度加大。
### 动态代理
静态代理由于在编译期就决定代理类，如果代理哪个类发生在运行期间，我们就不需要维护多个代理类，实现动态代理。通过查看JDK文档，实现动态代理步骤如下：

1、通过实现 InvocationHandler 接口创建自己的调用处理器；
``` java
public class MyInvocationHandler implements InvocationHandler{
    private Object target;
    public MyInvocationHandler(Object target) {
        this.target=target;
    }
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("method :"+ method.getName()+" is invoked!");
        return method.invoke(target,args);
    }
}
```
2、通过为 Proxy 类指定 ClassLoader 对象和一组 interface 来创建动态代理类；

3、通过反射机制获得动态代理类的构造函数，其唯一参数类型是调用处理器接口类型；

4、通过构造函数创建动态代理类实例，构造时调用处理器对象作为参数被传入。

实际使用过程更加简单，因为 Proxy 的静态方法 newProxyInstance 已经为我们封装了步骤 2 到步骤 4 的过程，所以简化后的过程如下
``` java
Interface proxy = (Interface)Proxy.newProxyInstance( classLoader, 
     new Class[] { Interface.class }, handler );
```
5、我们可以编写测试类如下：
``` java
public static void main(String[]args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
    HelloWorld helloWorld=(HelloWorld)Proxy.
             newProxyInstance(this.class.getClassLoader(),
                    new Class<?>[]{HelloWorld.class},
                    new MyInvocationHandler(new HelloworldImpl()));
    helloWorld.sayHello();
}
```
### JDK动态代理内部实现
通过上述代码生成的代理类helloWorld实现的接口与被代理类相同，这样就实现了在运行期间才决定代理对象是谁，解决了静态代理的弊端。当动态声称代理类调用方法时，会触发invoke方法，该方法可以对代理类的接口方法进行增强。

梳理上述JDK动态代理的流程，主要涉及到以下几个类：

1、java.lang.reflect.InvocationHandler: 这里称他为”调用处理器”，他是一个接口，我们动态生成的代理类需要完成的具体内容需要自己定义一个类，而这个类必须实现 InvocationHandler 接口。InvocationHandler 接口中有方法：
invoke(Object proxy, Method method, Object[] args)；这个函数是在代理对象调用任何一个方法时都会调用的，方法不同会导致第二个参数method不同，第一个参数是代理对象（表示哪个代理对象调用了method方法），第二个参数是 Method 对象（表示哪个方法被调用了），第三个参数是指定调用方法的参数。

2、java.lang.reflect.Proxy: 这是生成代理类的主类，通过 Proxy 类生成的代理类都继承了 Proxy 类，即 DynamicProxyClass extends Proxy。Proxy 类主要方法为: static Object newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h)
这个静态函数的第一个参数是类加载器对象（即哪个类加载器来加载这个代理类到 JVM 的方法区），第二个参数是接口（表明你这个代理类需要实现哪些接口），第三个参数是调用处理器类实例（指定代理类中具体要干什么）。这个函数是 JDK 为了程序员方便创建代理对象而封装的一个函数，因此你调用newProxyInstance()时直接创建了代理对象（略去了创建代理类的代码）。其实他主要完成了以下几个工作：
``` java
static Object newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler handler)
{
    //1. 根据类加载器和接口创建代理类
    Class clazz = Proxy.getProxyClass(loader, interfaces); 
    //2. 获得代理类的带参数的构造函数
    Constructor constructor = clazz.getConstructor(new Class[] { InvocationHandler.class });
    //3. 创建代理对象，并制定调用处理器实例为参数传入
    Interface Proxy = (Interface)constructor.newInstance(new Object[] {handler});
}
```
通过调用Proxy类的newProxyInstance工厂方法，动态地生成了ProxyObject，所有的方法调用都被委派到InvocationHandler的invoke方法，invoke方法通关Method反射执行源对象对应的方法。我们可以推测动态代理类ProxyObject肯定实现了源接口，并包含一个InvocationHandler对象引用，这是从Proxy继承而得到的。当调用接口中的每个方法时，通过反射调用得到对应的Method对象，并调用invoke方法。
回到最前面的动态代理测试类，执行后会在磁盘中产生一个HelloWorldAgent.class代理类Class文件，反编译后可以看到如下代码：
``` java
import java.lang.reflect.*;
 
public final class HelloWorldAgent extends Proxy
  implements HelloWorld {
  private static Method sayHello;
  public HelloWorldAgent(InvocationHandler paramInvocationHandler) throws {
    super(paramInvocationHandler);
  }
 
  public final String sayHello() throws {
    try{
      return (String)this.h.invoke(this, sayHello, null);
    } catch (Error|RuntimeException localError) {
      throw localError;
    } catch (Throwable localThrowable) {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }
  static {
    try {
      sayHello = Class.forName("HelloWorld").getMethod("sayhello", new Class[0]);
      return;
    } catch (NoSuchMethodException localNoSuchMethodException) {
      throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
    } catch (ClassNotFoundException localClassNotFoundException) {
      throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
    }
  }
}
```
生成的代理类继承了Proxy类，同时实现了代理接口。这就说明了为什么JDK动态代理只支持接口的原因，因为Java不支持多继承。同时提供了一个构造函数,传入InvocationHandler，并赋值给Proxy的h变量。所有的方法，包括我们自己的方法，都只是简单的调用了invocationHandler的invoke方法。所以，我们的InvocationHandler其实都可以不用调用我们自己的方法，而执行别的逻辑。而最终用户调用代理对象的接口方法，都会通过handler通过反射调用源对象的接口方法。
