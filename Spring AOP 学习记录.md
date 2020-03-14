Spring AOP 学习记录

AOP（面向切面编程）方面的知识又是看了忘忘了看，今天有空记录下AOP的知识点。主要分为以下几方面：

​		1、AOP相关术语

​		2、基础知识及示例

​		3、增强分类

##### 1、AOP相关术语

* **连接点（Joinpoint）** 一个类拥有一些边界性质的特定点，如一个类的各个方法就可称为连接点。
* **切点（Pointcut）** 切点就是在众多连接点中选择自己感兴趣的连接点，如果将连接点比作一个数据库中的所有记录，那么切点就是一个查询语句的查询条件。
* **增强（Advice）** 增强是置入目标连接点的一段代码，增强本身携带方位信息，如方法调用前、调用后、调用前后、异常抛出使等。
* **切面（Aspect）** 切面由切点和增强组成，既包含增强逻辑和方位信息，也包含连接点信息。
* **目标对象（Target）** 增强逻辑织入的目标类。
* **引介（Introduction）** 引介是一种特殊的增强，它为类提供一些属性和方法。这样，即是一个业务类原本没实现某个接口，通过AOP的引介功能也可以动态的为该类添加接口实现逻辑，让该类成为接口的实现类。
* **织入（Weaving）** 织入就是讲增强添加到目标类的具体方法上的过程。AOP有三种织入方式①编译期织入、②类装载期织入、③动态代理织入（Spring一般采用的方式）
* **代理（Proxy）** 一个类被AOP织入增强后，就产生了一个代理结果类。它融合了增强逻辑，根据代理方式不同，代理类既可能是和原类具有相同接口的类或者是原类的子类。

通过以上概念我们可以想到通过定义切点、给切点织入增强逻辑等步骤，最后就可以完成切面编程。

##### 2、基础知识及示例

###### 2.1 JDK动态代理

​		Java JDK提供的动态代理技术允许开发者在运行期创建接口的代理类，JDK动态代理主要涉及java.lang.reflect包中的两个类：Proxy和InvocationHandler。

​		其中，InvocationHandler是一个接口，可以通过实现该接口实现增强逻辑，并通过反射机制调用目标类代码，动态的将增强逻辑和业务逻辑编制在一起。Proxy利用InvocationHandler动态创建一个符合某一接口的示例，生成目标类的代理类。下面我使用代码演示。

```java
// 下面假设我们有以下代码
public interface GreetService {
	void serviceTo(String name);
}

public class Waiter implements GreetService{

	@Override
	public void serviceTo(String name) {
         // 假设下面为代码时间性能监测代码
		// Long beginTime = System.currentTimeMillis();
		try {
			// 模拟程序在工作
			System.out.println("service to :"+name);
			Thread.sleep(10000L);
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		// System.out.println("consume time:"+(System.currentTimeMillis()-beginTime));
	}
}


// 我们将上面代码的性能监控代码使用JDK动态代理织入
// 首先编写InvocationHandler实现类
public class WaiterHandler implements InvocationHandler{
	// 目标类
	private Object target;
	public WaiterHandler(Object target) {
		this.target = target;
	}
	// 代理调用方法
	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		Long beginTime = System.currentTimeMillis();
		// 调用target的相应方法，并传入参数
		method.invoke(target,args);
		
		System.out.println("consume time : "+(System.currentTimeMillis()-beginTime));
		return null;
	}
}
public class MainTest {

	public static void main(String[] args) {
		// 创建GreetService实例，也就是希望被代理的业务类
		GreetService waiter = new Waiter();
		// 创建InvocationHandler实现类，为业务类织入增强逻辑
		WaiterHandler waiterHandler = new WaiterHandler(waiter);
		// 使用Proxy生成GreetService代理实例
		GreetService waiterProxy = (GreetService)Proxy.newProxyInstance(waiter.getClass().getClassLoader(), Waiter.class.getInterfaces(),waiterHandler);
		// 调用代理实例方法
		waiterProxy.serviceTo("fuhang");
	}
}
// output:
//  service to :fuhang
//  consume time : 10002
 
```

​		上面为大家演示了一个使用JDK动态代理将性能监控代码从业务逻辑抽出然后动态织入的示例代码。

​		从上面的示例代码中大家可能发现了几个问题：

​				1、Waiter必须实现一个接口

​				2、newProxyInstance(waiter.getClass().getClassLoader(), Waiter.class.getInterfaces(),waiterHandler);第二个参数必须传入Waiter实现的接口列表

由以上可以知道JDK创建代理有一个限制，它只能为接口创建代理实例，难道我们写一个业务类必须为其定义一个接口吗？对于没有通过接口定义业务方法的类，如何实现为其动态创建代理实例呢？下面我们看看下一个动态代理技术CGLib。

###### 2.2CGLib动态代理

​		CGLib底层使用了动态字节码技术，可以为一个类创建子类，在子类中采用方法拦截技术拦截所有父类方法调用，并顺势织入增强逻辑。下面我们使用CGLib实现上面性能监控织入的逻辑。

```java
// 首先我们创建类CglibProxy实现MethodInterceptor
public class CglibProxy implements MethodInterceptor{

	@Override
	public Object intercept(Object obj, Method arg1, Object[] arg2, MethodProxy methodProxy) throws Throwable {
		Long beginTime = System.currentTimeMillis();
		// 实际方法调用的地方
		methodProxy.invokeSuper(obj, arg2);
		
		System.out.println("consume time : "+(System.currentTimeMillis()-beginTime));
		return null;
	}
}

// 测试类
public class MainTest {

	public static void main(String[] args) {
		// 创建EnHancer对象
		Enhancer enhancer = new Enhancer();
        // 创建我们的CglibProxy代理对象
		CglibProxy cglibProxy = new CglibProxy();
		// 设置父类为Waiter
		enhancer.setSuperclass(Waiter.class);
        // 设置回调函数为CglibProxy
		enhancer.setCallback(cglibProxy);
		// 使用enhancer动态字节码技术创建Waiter的代理类
		Waiter waiter = (Waiter) enhancer.create();
		waiter.serviceTo("fuhang");
	}
}
// output:
//    service to :fuhang
//    consume time : 10044
```

​		上面就是通过Cglib动态字节码技术为业务类生成子类的方式生成代理类，由此牵扯到Java知识点回顾，final和private修饰符修饰的类不能使用Cglib动态代理技术。

**小结** ：

​		SpringAOP技术底层就是使用动态代理技术为目标bean织入增强逻辑的，但是从以上两类型代理技术的示例代码中可以发现以下几个问题：

​		（1）、目标类被代理后，所有方法都加入了增强逻辑，有些时候这并不是我们想要的。

​		（2）、 通过手动编码方式指定了增强逻辑

​		（3）、 手工编写代理实例创建过程，为不同类创建代理时候需要手工编写相应的代理实例创建过程代码。

​		所以Spring AOP的主要工作就是解决以上三个问题：Spring AOP 通过切点（Pointcut）指定在哪些类的哪些方法上织入增强逻辑，通过增强（Advice）描述具体的织入点（方法调用前、调用后、调用前后），Spring通过Advisor（切面）将切面和增强组织起来，下面Spring就可以利用动态代理技术采用统一的方式为目标bean织入增强了。

##### 3、增强类型

​		Spring使用增强（Advice）定义增强逻辑，增强不仅包括增强逻辑，还包括方位信息（即方法调用前、调用后、调用前后）

​		Spring支持**五种**增强类型，解释如下：

![aop](https://github.com/DoubleCherish/my_understand_spring/blob/master/advice.png)

* **前置增强** ：`org.springframework.aop.BeforeAdvice` 代表前置增强，Spring只支持方法级的增强，所以目前MethodBeforeAdvice是Spring中可用的前置增强，BeforeAdvice接口是为了将来扩展定义的。
* **后置增强** ：org.springframework.aop.AfterReturningAdvice，表示在目标方法执行后进行增强。
* **环绕增强** ：org.aopalliance.intercept.MethodInterceptor代表环绕增强，表示在目标方法执行前后实施增强。
* **异常抛出增强** ： org.springframework.aop.ThrowsAdvice代表抛出异常增强，表示在目标方法抛出异常后实施增强。
* **引介增强** ： org.springframework.aop.IntroductionInterceptor代表引介增强，表示在目标类中添加一些新的方法和属性。

###### 3.1前置增强

​		我们还用我们之前的GreetService和Waiter做示例代码，之前我们的代码中`Waiter.serviceTo(name)`方法只是输出"service to xxx"，现在我们想实现`Waiter.serviceTo(name)`在输出之前想客户打招呼，那么按照前面的理解，我们需要为其定义一个前置增强，下面我们着手改造吧。

```java
// 我们定义一个打招呼前置类，实现MethodBeforeAdvice
public class GreetBeforeAdvice implements MethodBeforeAdvice{
	@Override
	public void before(Method arg0, Object[] arg1, Object arg2) throws Throwable {
         // 输出招呼用语
		System.out.print("hello "+ arg1[0]);
	}
}
public class MainTest {
	public static void main(String[] args) {
		// 创建Waiter和前置增强类
		Waiter waiter = new Waiter();
		GreetBeforeAdvice greetBeforeAdvice = new GreetBeforeAdvice();
		// 创建代理工厂，随后解释这个工厂
		ProxyFactory proxyFactory = new ProxyFactory();
		// 设置被代理的目标和要织入的增强
		proxyFactory.setTarget(waiter);
		proxyFactory.addAdvice(greetBeforeAdvice);
		// 通过代理工厂创建代理类
		Waiter myWaiter = (Waiter) proxyFactory.getProxy();
		// 调用方法
		myWaiter.serviceTo("fuhang");	
	}
}
// output：
// 		hello fuhang service to :fuhang
```

​		上面就是前置增强的示例，下面讲解下ProxyFactory的细节，先来回一下前面的基础知识章节的内容，说了两个动态代理技术，ProxyFactory实际就是用到了JDK或者Cglib动态代理技术。

​		Spring定义了`org.springframework.aop.framework.AopProxy`接口，最终提供了两个final实现类，如下图所示。

![aopproxy](https://github.com/DoubleCherish/my_understand_spring/blob/master/aopproxy.png)

​		上图中Cglib2Proxy使用的是Cglib动态代理技术，JdkDynamicAopProxy使用的是JDK动态代理技术，如果通过ProxyFactory的setInterfaces(Class [] interfaces)方法指定目标类的接口信息，那么ProxyFactory底层将使用Jdk动态代理技术，否则不设置就使用Cglib动态代理技术。由此可知我们的示例代码中用到了Cglib动态代理技术。

###### 3.2后置增强

​		比如每次服务过后服务员需要询问顾客"还有什么需要吗？"这句话，那么可以通过一个后置增强来实施这一个要求。示例如下。

```java
// 声明一个后置增强类并实现AfterReturningAdvice
public class GreetAfterAdvice implements AfterReturningAdvice{

	@Override
	public void afterReturning(Object arg0, Method arg1, Object[] arg2, Object arg3) throws Throwable {
		System.out.println("还有什么需要吗？");
	}
}

// 测试类
public class MainTest {

	public static void main(String[] args) {
		// 创建Waiter及前置增强、后置增强
		Waiter waiter = new Waiter();
		GreetBeforeAdvice greetBeforeAdvice = new GreetBeforeAdvice();
		GreetAfterAdvice greetAfterAdvice = new GreetAfterAdvice();
		// 代理工厂
		ProxyFactory proxyFactory = new ProxyFactory();
		// 设置目标代理类及前置增强、后置增强
		proxyFactory.setTarget(waiter);
		proxyFactory.addAdvice(greetAfterAdvice);
		proxyFactory.addAdvice(greetBeforeAdvice);
		// 创建代理Waiter
		Waiter myWaiter = (Waiter) proxyFactory.getProxy();
		// 调用方法
		myWaiter.serviceTo("fuhang");
	}
}
// output:
//     hello fuhang service to :fuhang
//     还有什么需要吗？
```

###### 3.3 环绕增强

​		使用环绕增强实现上面前置后置增强综合体，示例代码如下。

```java
// 声明环绕增强类并实现MethodInterceptor(aopalliance的包)
public class SurrondAdvice implements MethodInterceptor{

	@Override
	public Object invoke(MethodInvocation arg0) throws Throwable {
         // 获取目标类参数
		Object [] args = arg0.getArguments();
		System.out.println("hello :"+args[0]);
         // 执行目标类方法
		arg0.proceed();
		System.out.println(" 请问还有什么需要吗？");
		return null;
	}
}

// 下面代码不在啰嗦，直接看输出
public class MainTest {

	public static void main(String[] args) {
		
		Waiter waiter = new Waiter();
		SurrondAdvice surrondAdvice = new SurrondAdvice();
		
		ProxyFactory proxyFactory = new ProxyFactory();
		
		proxyFactory.setTarget(waiter);
		proxyFactory.addAdvice(surrondAdvice);

		
		Waiter myWaiter = (Waiter) proxyFactory.getProxy();
		
		myWaiter.serviceTo("fuhang");
		
	}
}
// output:
//  	hello :fuhang
//   	service to :fuhang
//		请问还有什么需要吗？
```

###### 3.4 异常抛出增强

​		异常抛出增强适合的应用场景是事务管理，当参与事务管理的某个Dao抛出异常时候回滚事务。示例如下。

```java
public class Waiter{

	@Override
	public void serviceTo(String name) {
		throw new RuntimeException("Dao异常");
	}
}

// 异常抛出增强类，在异常抛出之前会调用
public class ExceptionAdvice implements ThrowsAdvice{
	
	public void afterThrowing(Method method,Object [] args,Object target,Exception exception){
		System.out.println("事务回滚");
	}
}

// 测试类
public class MainTest {

	public static void main(String[] args) {
		
		Waiter waiter = new Waiter();
		ExceptionAdvice exceptionAdvice = new ExceptionAdvice();
		
		ProxyFactory proxyFactory = new ProxyFactory();
		
		proxyFactory.setTarget(waiter);
		proxyFactory.addAdvice(exceptionAdvice);

		
		Waiter myWaiter = (Waiter) proxyFactory.getProxy();
		
		myWaiter.serviceTo("fuhang");
		
	}
}
// output:
// 		事务回滚
//		Exception in thread "main" java.lang.RuntimeException: Dao异常
```

​		ThrowingAdvice是一个标签接口，其中没有声明任何方法，在运行期Spring通过反射机制自动判断增强方法，增强方法签名必须是以下格式：

​		`void afterThrowing([Method method,Object [] args,Object obj],Throwable)`

方法前三个参数可选，要么全部传入，要么全部不传。最后一个必须传入。最后一个参数是Throwable的子类就行。

##### 3.5 引介增强

​		引介增强比较特殊，他不是在方法周围增加增强逻辑，而是为目标类动态添加新的方法和属性。所以引介增强的连接点定位是类而不是方法上。下面我们通过引介增强为目标类添加一个接口实现，完成监控开关功能。

```java
// 声明一个附加接口
public interface Monitor {
	void setMonitorActive(boolean active);
}

// 目标类
public class Waiter{
	public void serviceTo(String name) {
		System.out.println("hello :"+name);
	}
}

// 创建一个类继承DelegatingIntroductionInterceptor并实现将来要附加到目标类的接口
public class PerformanceMonitor extends DelegatingIntroductionInterceptor implements Monitor{
	// 相当于最终给类添加的属性
	ThreadLocal<Boolean> isOpen = new ThreadLocal<>();

	@Override
	public void setMonitorActive(boolean active) {
		isOpen.set(active);
	}

	@Override
	public Object invoke(MethodInvocation arg0) throws Throwable {
		if(isOpen.get()!=null && isOpen.get() ){
			Long beginTime = System.currentTimeMillis();
			super.invoke(arg0);
			System.out.println("consume time :"+(System.currentTimeMillis()-beginTime));
		}else{
			super.invoke(arg0);
		}
		return null;
	}

}

// 测试类
public class MainTest {

	public static void main(String[] args) {
		
		Waiter waiter = new Waiter();
		PerformanceMonitor performanceMonitor = new PerformanceMonitor();
		
		ProxyFactory proxyFactory = new ProxyFactory();
		// 下面必须设置setOptimize(true)，开启Cglib代理
		proxyFactory.setInterfaces(Monitor.class);
		proxyFactory.setTarget(waiter);
		proxyFactory.addAdvice(performanceMonitor);
		proxyFactory.setOptimize(true);

		// 第一次调用方法
		Waiter myWaiter = (Waiter) proxyFactory.getProxy();
		myWaiter.serviceTo("fuhang");
		
        // 转为Monitor后设置监控开关后再次调用方法
		Monitor monitor = (Monitor)myWaiter;
		monitor.setMonitorActive(true);
		
		myWaiter.serviceTo("zhangsan");
		
	}
}
// output：
// 		hello :fuhang
//		hello :zhangsan
//		consume time :0
```

**总结**

​		以上就是本次学习的主要内容，后面如何将上面手动创建的方法转换为Spring xml配置在本篇不做记录（因为不难，也没啥技术含量）。希望通过本次记录能学到Spring Aop核心知识。

**参考资料**
《精通Spring4.x企业应用开发实战》
