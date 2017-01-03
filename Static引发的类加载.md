
# Static引发的类加载
**背景**：代码优化是框架持续关注的重点，近期想将框架所有的默认规则转移到Web容器启动时候加载，避免在首次Http请求再加载造成阻塞。当我在一个工具类中写好一个静态代码块后，兴冲冲启动服务期待启动中执行，可惜直到启动结束也未执行代码块。本文从关键词static入手，延伸出对类加载机制及类初始化的剖析。

首先看个例子：
```java
public class Singleton {
	public static Singleton singleton = new Singleton();
    public static int a;
	public static int b = 0;
	static {
		System.out.println("执行静态代码块");
	}
	private Singleton() {
		a++;
		System.out.println("执行构造函数");
		b++;
	}
	public static Singleton getInstance() {
		return singleton;
	}
}
public class ClassLoaderTest {
	public static void main(String[] args) {
		Singleton mySingleton = Singleton.getInstance();
		System.out.println(mySingleton.a);
		System.out.println(mySingleton.b);
	}
}
```
执行ClassLoaderTest入口方法，结果如下：
>执行构造函数
>执行静态代码块
>1
>0

在启动JVM时候，类Singleton的静态代码块并未执行，在显示调用该类后，JVM首先会对该类的静态变量顺序进行初始化，对singleton初始化时会调用构造函数，此时系统会对变量a、b赋一个默认初始值0。接下来会对变量a、b初始化，由于代码显示地将b设为0，因此打印出b为0。
由上可知，一个类在JVM生命周期中分以下几步：
- 源文件由编译器编译成字节码（.class文件）；
- 加载，由JVM解释并生成class对象；
- 链接，该阶段会给静态变量赋默认值，但并不执行静态代码块；
- 初始化，使用前会执行初始化，如构造函数和类静态变量的初始值；
- 使用及卸载。

接下来我们分析一下比较重要的两个阶段：加载和初始化
##加载
ClassLoader的作用，概括来说就是将编译好的class文件装载、并在内存中创建该对象，从而该对象可以被其他class引用，应用程序需要实现 ClassLoader 的子类，以扩展 Java 虚拟机动态加载类的方式。首先从源码出发了解类加在机制。
构造函数：ClassLoader类提供两个构造方法
```java
protected ClassLoader(ClassLoader parent) {
    this(checkCreateClassLoader(), parent);
}
protected ClassLoader() {
    this(checkCreateClassLoader(), getSystemClassLoader());
}
private static Void checkCreateClassLoader() {
    SecurityManager security = System.getSecurityManager();
    if (security != null) {
        security.checkCreateClassLoader();
    }
    return null;
}
```
从上边JDK接口文档可以发现，在创建一个classloader的实例时可以显示指出他的父加载器，也可以不指定，不指定的时候他的默认父加载器是系统加载器。
接下来看看它的加载类的方法：loadClass
```java
protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
	  // First, check if the class has already been loaded
	  Class c = findLoadedClass(name);
	  if (c == null) {
	    try {
		  if (parent != null) {
		    c = parent.loadClass(name, false);
		  } else {
		    c = findBootstrapClassOrNull(name);
		  }
	    } catch (ClassNotFoundException e) {
          // ClassNotFoundException thrown if class not found
        }
        if (c == null) {
	      // If not found, invoke findClass to find the class.
	      c = findClass(name);
	    }
	  }
	  if (resolve) {
	    resolveClass(c);
	  }
	  return c;
}
protected Class<?> findClass(String name) throws ClassNotFoundException {
    throw new ClassNotFoundException(name);
}
```
该方法通过指定class名称来加载类，该方法是一个递归的方法，首先自底向上检查类是否已经加载，然后自顶向下尝试去加载相关类。默认实现将按以下顺序搜索类：
- 调用fildLoadedClass(String)来确认是否已加载此类；
- 在父类加载器上调用loadClass方法，如果父类加载器为null，则通过最顶层BootStrap ClassLoader来加载；该机制称为类加载器的双亲委托机制；
- 调用findClass(String)方法查找类，该方法源码为空，需要子类去重写。也就是说，如果我们自定义一个ClassLoader，只需要重写findClass(String)方法去加载类文件即可。

通过源码，了解ClassLoader父类委托机制，我们可知Java类装载器实际也是类，JVM启动时通过它们把相关类载入。Java类装载器有三个，它们分工明确，负责加载各自模块，同时它们也有层级关系，AppClassLoader的Parent是ExtClassLoader，ExtClassLoader的Parent是null，因为Bootstrap Loader是通过C++编写，所以并非逻辑上的父类，但它的层级是最高的。
##初始化
该初始化指的是类初始化，此时才真正开始执行类中定义的Java程序代码。该阶断根据程序员通过程序指定的主观计划去初始化类变量和其他资源。类的初始化是延迟的，直到类第一次被主动调用时，JVM才会初始化该类。
类的初始化遵循如下规则：
- JVM不会自动调用初始化类操作，只有该类在第一次调用时才执行。这就解释了本文背景中的问题，如果某个类在启动时并未显示调用，那么该类的静态代码块和静态变量都不会执行初始化，直到有程序显示调用；
- 类被初始化之前，首先会初始化其直接父类；但接口的初始化并不会引起父接口的初始化。
- 类初始化过程主要执行静态代码块和初始化静态域，并且会按照源代码从上到下的顺序依次执行静态代码块和静态域；联想到在引言的例子中，如果调换一下静态域的顺序，如下所示：
```java
public static int a;
public static int b = 0;
public static Singleton singleton = new Singleton();
static {
  System.out.println("执行静态代码块");
}
```
执行ClassLoaderTest入口方法，结果如下：
>执行静态代码块
>执行构造函数
>1
>1

也就是说静态变量b先于构造函数Singleton()进行初始化，构造函数对b加1后并不会再对b重新赋值，所以b的打印值为1

##总结
只有显示调用才会加载类对象并执行初始化操作。在类加载过程中，除了加载阶段应用程序自定义类加载器之外，其余所有动作都由虚拟机主导，实现双亲委托机制，自底向上检查类是否已经加载，然后自顶向下尝试去加载相关类。到了初始化才开始执行类中定义的Java程序代码，并依此执行初始化语句。只有完成类加载到内存，才真正开始执行类操作。
