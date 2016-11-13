# Firefly_Server与设计模式

Firefly_Server是移动开发框架（Firefly）服务端过滤器框架，提取特定结构形成体系结构，开发人员进行二次开发，实现具体应用系统功能。为了实现“高内聚、低耦合”的理念，使我们设计更加对象化，框架大量采用设计模式解决架构中的问题。下文我们将分析框架中使用较多的设计模式。

设计模式的六大原则
- **开闭原则（Open Close Principle）**：对扩展开放，对修改关闭。
- **里氏代换原则（Liskov Substitution Principle）**：面向对象设计的基本原则之一，是对实现抽象化的具体步骤的规范。
- **依赖倒转原则（Dependence Inversion Principle）**：这个是开闭原则的基础，对接口编程，依赖于抽象而不依赖于具体。
- **接口隔离原则（Interface Segregation Principle）**：使用多个隔离的接口，比使用单个接口要好。降低依赖，降低耦合。
- **迪米特法则（最少知道原则）（Demeter Principle）**：一个实体应当尽量少的与其他实体之间发生相互作用，使得系统功能模块相对独立。
- **合成复用原则（Composite Reuse Principle）**：原则是尽量使用合成/聚合的方式，而不是使用继承。

**1、单例模式（Singleton）**
框架需求：需要对应用系统配置的安全组件缓存（包括算法对象、Protection对象等），如果频繁创建与消除这些大对象，不止提高了内存使用频率，增加了GC压力，同时某对象多个实例存在会扰乱框架的调度流程。
解决方案：
1.1 框架采用枚举实现单例模式，将安全组件缓存内存并提供唯一调用入口。

```
public enum Singleton {
	PROTECTIONS {
		private final Cache<Class<? extends WebProtection>, WebProtection> protections = CacheBuilder
				.createBuilder().build();
		@Override
		public Cache<Class<? extends WebProtection>, WebProtection> getSingletons() {
			return protections;
		}
	},

	ALGORITHMS {
		private final Cache<Class<? extends Algorithm>, Algorithm> algorithms = CacheBuilder
				.createBuilder().build();

		@Override
		public Cache<Class<? extends Algorithm>, Algorithm> getSingletons() {
			return algorithms;
		}
	}；
	public abstract <T> Cache<Class<? extends T>, T> getSingletons();
}
```
该类定义了四个枚举对象，并抽象出单例获取方式交给具体枚举对象来实现，使用者只需要调用抽象方法即可获取对象实例。
1.2 使用static final获取对象
```
    private static class AlgorithmFactoryHolder {
		private final static AlgorithmFactory factory = new AlgorithmFactory();
	}

	public static AlgorithmFactory getInstance() {
		return AlgorithmFactoryHolder.factory;
	}

```
单例模式为实例提供了唯一的受控访问，由于在系统内存中只存在一个对象，因此可以节约系统资源，避免了重复创建销毁造成的系统性能损耗。

**2、抽象工厂（Abstract Factory）**
框架需求：应用系统并不关注安全组件的创建与销毁，为了实现使用者与创建者分离，框架拟采用工厂方法实现解耦；已实现的过滤器安全组件包括四大类：Handler、Protection、Algorithm、MessageParser，未来有很大可能需要对组件做扩展，显然我们需要通过抽象工厂模式来维护多个工厂类
下图是框架工厂相关类图，四个子工厂分别对应四大类组件，每个工厂完成自己领域内的相关功能，通过泛型实现父类对相关对象的构建。

```
public abstract class AbstractObjectFactory<T> implements ObjectFactory<T> {
	private final List<String> orderedObjectNames = new ArrayList<String>();
	protected AbstractObjectFactory() {
	}
	public T getObject(Class<? extends T> c,
			List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
		T t = null;
		Class<? extends T> clazz = (Class<T>) ReflectionUtils.resolveClass(c);
		t = doNewInstance(clazz, constructorArgTypes, constructorArgs);
		ensureObject(t);
		return t;
	}
}
```
```
public class AlgorithmFactory extends AbstractSingletonFactory<Algorithm> {
	private static class AlgorithmFactoryHolder {
		private final static AlgorithmFactory factory = new AlgorithmFactory();
	}
	public static AlgorithmFactory getInstance() {
		return AlgorithmFactoryHolder.factory;
	}
	public Map<String, Algorithm> getAlgorithms(Configuration config) {
		Map<String, Algorithm> algorithms = new HashMap<String, Algorithm>();
		List<String> algorithmNames = getOrderedObjectNames();
		Iterator<String> it = algorithmNames.iterator();
		while (it.hasNext()) {
			String fqcn = it.next();
			Algorithm p = getObject(fqcn);
			algorithms.put(fqcn, p);
		}
		return algorithms;
	}
}
```
抽象工厂完美的实现了“开闭原则”，对扩展开放，如果需要增加一种工厂，只需要新建一个工厂类实现父类即可，并不需要对父类及其他工厂做任何修改。

**3、装饰模式（Decorator）**
框架需求：Servlet输入输出流都是单向的，对于客户端Post请求，过滤器层安全组件需要读取输入字节流做安全校验，但这导致在应用系统层无法获取输入的字节流；我们需要对HttpServletRequest和HttpServletResponse进行继承扩展，并动态增加该对象功能职责，框架采用装饰模式来实现。
以HttpServletRequest做例子，如下代码所示，框架封装了原有对相，对输入字节流做了持久化，并增加了框架需要调用的功能，将装饰后的全新对象交给过滤器及后续应用系统：
```
public class FireflyRequest extends HttpServletRequestWrapper {
	private byte[] body;
	public FireflyRequest(HttpServletRequest request) throws IOException {
		super(request);
		InputStream is = request.getInputStream();
		body = ResourceUtils.getStreamAsByteArray(is);
	}
	public String getParameter(String name) {
		String value = super.getParameter(name);
		value = value == null ? null : convert(value);
		return value;
	}
	public ServletInputStream getInputStream() {
		final ByteArrayInputStream bais = new ByteArrayInputStream(body);
		return new ServletInputStream() {
			public int read() throws IOException {
				return bais.read();
			}
		};
	}
}
```
装饰器能给原始对象提供更多的功能，比继承有更多的灵活性；但是需要特别注意在覆盖原有接口的时候，不要破坏原始的协议，要特别慎重。

**4、模板方法（Template Method）**
框架需求：作为一个开发框架，为了匹配不同的应用系统来实现相同的处理逻辑，一些具体的操作行为需要应用系统来实现，而框架只需要定义流程结构，实现公共方法即可；模板方法完美解决了该问题，将一些变化的因素抽象出来，交给子类来实现。
框架比较典型应用场景在报文组装模块，众所周知每个应用系统有一套报文协议，需要将报文转换成框架识别的格式，比如框架依赖于请求transId及appId，实现该功能分3步：首先从HttpRequest取报文体，其次从报文体提取transId及appId，最后将二者组装成框架能识别的对象。对于报文体取transId等，每个应用系统各不相同，需要将其抽象，交给应用系统来实现，如下是抽象类：
```
public abstract class AbstractHttpMessageTemplate extends
		AbstractMessageTemplate {
	protected Context processRequestByProtocol(Object requestObject) {
		final Context context;
		}
		String httpMethod = _requestObject.getMethod();
		if (StringUtils.isEquals(GET.getMethodName(),
				httpMethod)) {
			processGet(context, _requestObject);
		} else (StringUtils.isEquals(POST.getMethodName(),
				httpMethod)) {
			processPost(context, _requestObject);
		}
		return context;
	}
	protected abstract void processPost(final Context context,
			final HttpServletRequest request);
}
```
模板方法采用代码复用技术，通过子类来扩展新的行为，使得类之间关联关系更加清晰，在框架中的算法模块也用到了该设计模式。但设计越抽象，类的个数就越多，导致系统更加庞大，这是架构者需要简化的问题。
