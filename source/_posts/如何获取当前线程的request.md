---
title: 如何获取当前线程的request
date: 2020-05-25 10:41:01
tags: Spring MVC
description: 如何线程安全地获取当前客户端的 request 请求
---

## 概述:
在使用Spring MVC 做开发时，经常需要获取 request 信息，例如获取客户端 ip 地址，请求的 URL， header 里的 cookie、token 等信息。
那么如何线程安全地获取 request 信息就显得尤为重要。
## 线程不安全的代码
我们先来看看错误的代码演示
```java
public abstract class BaseController {

    private HttpServletRequest request;

    @ModelAttribute
    public void init(HttpServletRequest request) {
        this.request = request;
    }
}
```
这段代码是将 request 作为参数传入 BaseController 基类里，并且将 HttpServletRequest 作为类的全局变量(成员变量)。

对于 tomcat 等 web 容器来说，它是并发地去处理 HTTP 请求的，BaseController 类接收到的每个 request 请求都有可能改变 HttpServletRequest 全局变量，造成线程安全问题。

形象一点：
我们假设A向系统发送 HTTP 请求,并执行了 init()方法,由于 Spring 默认每一个 bean 都是单例模式，此时B也向系统发送请求并执行 init() 方法,修改了 HttpServletRequest 这个属性，此时对于A来说，HttpServletRequest 这个属性已经被B改变，就会引发线程安全问题。

那么如何才能做到线程安全地获取 request 请求呢?

## 方法1 request 作为参数
这种实现方式最为简单
```java
@RestController
public class TestController {
    @RequestMapping("/test")
    public void test(HttpServletRequest req) throws InterruptedException {
        // 模拟程序执行了一段时间
        Thread.sleep(1000);
    }
}
```
优点: 线程安全。该方法将 request 对象作为参数赋值给 test() 方法, request 对象作为方法的成员变量,必然是线程安全的。
缺点: 代码冗余。很多请求都会用到 request 请求, 每次都要将 request 作为参数才可调用。并且如果方法复杂， 需要将 request 作为参数传来传去;
## 方法2 依赖注入
先上代码:
```java
@RestController
public class TestController {

    @Autowired
    private HttpServletRequest request; //依赖注入

    @RequestMapping("/test")
    public void test(HttpServletRequest request) throws InterruptedException {
        // 模拟程序执行了一段时间
        Thread.sleep(1000);
    }
}
```
使用 @Autowired 依赖注入的方式 request 变成了线程安全。为什么会这样呢？我们去查找源码
首先断点一下:
![](https://raw.githubusercontent.com/Fxy399/myPicGo/master/i2mago/img/user/img20200514165044.png)
request 使用了代理，我们先找到 AutowireUtils 的 ObjectFactoryDelegatingInvocationHandler 内部类
```
private static class ObjectFactoryDelegatingInvocationHandler implements InvocationHandler, Serializable {

		private final ObjectFactory<?> objectFactory;
		public ObjectFactoryDelegatingInvocationHandler(ObjectFactory<?> objectFactory) {
			this.objectFactory = objectFactory;
		}

		@Override
		public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
			String methodName = method.getName();
			if (methodName.equals("equals")) {
				return (proxy == args[0]);
			} else if (methodName.equals("hashCode")) {
				return System.identityHashCode(proxy);
			} else if (methodName.equals("toString")) {
				return this.objectFactory.toString();
			}

			// 执行目标方法。注意：目标实例对象是objectFactory.getObject()
			try {
				return method.invoke(this.objectFactory.getObject(), args);
			} catch (InvocationTargetException ex) {
				throw ex.getTargetException();
			}
		}
	}
```
由上图代码发现 真正被代理的对象是 objectFactory.getObject()，而这里的 ObjectFactory 对象是传入的 WebApplicationContextUtils.RequestObjectFactory
那为什么使用的是 RequestObjectFactory 这个内部类呢?
WebApplicationContextUtils 里有这样一段代码:
```
public static void registerWebApplicationScopes(ConfigurableListableBeanFactory beanFactory, @Nullable ServletContext sc) {

		// web容器下新增支持了三种scope
		// 非web容器（默认）只有单例和多例两种嘛
		beanFactory.registerScope(WebApplicationContext.SCOPE_REQUEST, new RequestScope());
		beanFactory.registerScope(WebApplicationContext.SCOPE_SESSION, new SessionScope());
		if (sc != null) {
			ServletContextScope appScope = new ServletContextScope(sc);
			beanFactory.registerScope(WebApplicationContext.SCOPE_APPLICATION, appScope);
			sc.setAttribute(ServletContextScope.class.getName(), appScope);


			// ==================依赖注入=================
			// 这里决定了，若你依赖注入ServletRequest的话，就使用RequestObjectFactory来处理你
			beanFactory.registerResolvableDependency(ServletRequest.class, new RequestObjectFactory());
			beanFactory.registerResolvableDependency(ServletResponse.class, new ResponseObjectFactory());
			beanFactory.registerResolvableDependency(HttpSession.class, new SessionObjectFactory());
			beanFactory.registerResolvableDependency(WebRequest.class, new WebRequestObjectFactory());
		}

	}
```
这里说明 web 容器初始化时就决定当依赖注入的对象为 ServletRequest 类时, 就会使用 RequestObjectFactory 类来做依赖注入的被代理类。
那么我们就来看看 RequestObjectFactory 的代码:
```
/**
	 * Factory that exposes the current request object on demand.
	 */
	@SuppressWarnings("serial")
	private static class RequestObjectFactory implements ObjectFactory<ServletRequest>, Serializable {

		@Override
		public ServletRequest getObject() {
			return currentRequestAttributes().getRequest();
		}

		@Override
		public String toString() {
			return "Current HttpServletRequest";
		}
	}

	private static ServletRequestAttributes currentRequestAttributes()
	{
    	RequestAttributes requestAttr =                                      RequestContextHolder.currentRequestAttributes();
		if (!(requestAttr instanceof ServletRequestAttributes)) {
			throw new IllegalStateException("Current request
			is not a servlet request");
		}
		return (ServletRequestAttributes) requestAttr;
	}
```
到了这一步，已经很清晰了，我们通过 @Autowired 注解得到的 request 对象来就自于: RequestContextHolder.currentRequestAttributes() ，我们再看一下 RequestContextHolder 这个类
```
public abstract class RequestContextHolder  {
    //RequestAttributes对象是线程局部变量（ThreadLocal）
    private static final ThreadLocal<RequestAttributes> requestAttributesHolder = new NamedThreadLocal<>("Request attributes");
    private static final ThreadLocal<RequestAttributes> inheritableRequestAttributesHolder =
			new NamedInheritableThreadLocal<>("Request context");
    public static RequestAttributes currentRequestAttributes() throws IllegalStateException
    {
    	RequestAttributes attributes = getRequestAttributes();
    	if (attributes == null)
    	{
    		if (jsfPresent)
    		{
    			attributes = FacesRequestAttributesFactory
    			.getFacesRequestAttributes();
    		}
    	}
    	return attributes;
    }
    @Nullable
	public static RequestAttributes getRequestAttributes() {
		RequestAttributes attributes = requestAttributesHolder.get();
		if (attributes == null) {
			attributes = inheritableRequestAttributesHolder.get();
		}
		return attributes;
	}
}
```
RequestContextHolder 的意思是请求上下文，并且我们发现内部的 RequestAttributes 是线程局部变量(ThreadLocaL)，这就说明 RequestAttributes 是线程安全的， 并且 request 也是线程安全的。

到了这里你是否还有许多疑问，例如 request 是什么时候被赋值给 RequestAttributes的?
我们带着疑问去查看源码看看到底有谁调用过 setRequestAttributes() 方法

![setRequestAttributes被使用](https://raw.githubusercontent.com/Fxy399/myPicGo/master/i2mago/img/user/img20200515104335.png)
有两个地方调用过 setRequestAttributes 方法，那么到底是哪个方法真正设置了 request 请求？
我去百度了一下 RequestContextFilter 以及 FrameworksServlet 这两个类的作用。

**RequestContextFilter:**
该过滤器RequestContextFilter主要是用于第三方serlvet比如JSF FacesServlet。
该过滤器将当前请求暴露到当前线程，这样我们可以在当前线程中获取当前的请求信息，就不需要把当前请求作为参数到处传递。 而且 DispatcherServlet 的工作同样是将当前请求暴露到当前线程，但是被拦截器拦截的请求并不会到达 DispatcherServlet，这时就需要用到 RequestContextFilter 了。

> 该过滤器如果不是继承第三方 Servlet 基本上用不到，我们可以直接排除掉它

**FrameworkServlet:**
我们来看一下源码:
```
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException
{

	long startTime = System.currentTimeMillis();
	Throwable failureCause = null;

	LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
	LocaleContext localeContext = buildLocaleContext(request);

    //1. 获取当前线程的 RequestAttributes。为空
	RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
	//2. 创建 RequestAttributes
	ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
	asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());
    //3. 初始化 ContextHolders
	initContextHolders(request, localeContext, requestAttributes);
}

private void initContextHolders(HttpServletRequest request,
			@Nullable LocaleContext localeContext, @Nullable RequestAttributes requestAttributes) {

		if (localeContext != null) {
			LocaleContextHolder.setLocaleContext(localeContext, this.threadContextInheritable);
		}
		if (requestAttributes != null) {
			RequestContextHolder.setRequestAttributes(requestAttributes, this.threadContextInheritable);
		}
		if (logger.isTraceEnabled()) {
			logger.trace("Bound request context to thread: " + request);
		}
	}
```

可见请求发送之后，FrameworkServlet 就会将 请求赋值给 RequestAttributes，之后 RequestContextHolder 再将请求设置为线程局部变量，完成了请求和线程绑定的操作。
**依赖注入流程:**
现在我们来捋一下刚才的流程：
1. 首先web 容器初始化时决定当依赖注入的对象为 ServletRequest 时, 使用 RequestObjectFactory 类来做依赖注入的被代理类。
2. 当我们在 HttpServletRequest 对象上使用 @Autowired 注解时，自动使用 RequestObjectFactory 类来做依赖注入的被代理类。
3. ReauestObjectFactory 中的 request 请求对象来自于 RequestContextHolder，而 RequestContextHolder 中的 RequestAttributes 属于线程局部变量并且包含着 request 请求信息。

**我们直接上结论:**
> @Autowired 注解的 HttpServletRequest 来自于 RequestContextHolder 中的线程局部变量 RequestAttributes

## 方法3 从 RequestContextHolder 获取当前线程的 request
通过方法二我们已经得知 RequestContextHolder 中的 RequestAttributes 是线程安全的。那么我们就可以直接从 RequestContextHolder 获取线程安全的 request 请求啦。
代码如下：
```
public static HttpServletRequest getCurrentRequest()
{
    RequestAttributes ra = RequestContextHolder.getRequestAttributes();
    ServletRequestAttributes sra = (ServletRequestAttributes) ra;
    HttpServletRequest request = sra.getRequest();
    return request;
}
```
