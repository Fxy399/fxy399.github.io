---
layout: _posts
title: JDK-动态代理实现源码解析
tags: Java
description: 从源码角度剖析JDK动态代理的实现原理
date: 2021-07-27 22:25:01
typora-root-url: ..
---


## 前言

本文主要简粗暴地从源码角度剖析JDK动态代理的实现原理。

## 动态代理的使用

**流程：**

1. 创建接口及实现类
2. 实现代理处理器接口: InvocationHandler, invoke() 方法实现代理增强
3. 通过 Proxy.newProxyInstance() 创建代理类
4. 代理类调用方法

**代码实现**

```java
/**
 * 被代理接口
 */
public interface HelloInterface {
    void sayHello(String name);
}

/**
 * 被代理接口实现类
 */
public class Hello implements HelloInterface {
    @Override
    public void sayHello(String name) {
        System.out.println("Hello "+name+"!");
    }
}

/**
  * 代理类的 InvocationHandler 实现类，实现的 invoke() 方法负责增强被代理方法
  */
public class HelloProxy implements InvocationHandler {

    private Object subject;

    public HelloProxy(Object subject) {
        this.subject = subject;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before invoke: "+method.getName());
        method.invoke(subject, args);
        System.out.println("after invoke: "+method.getName());
        return null;
    }
}

public class FunctionTest {
    /**
      * 动态代理实现测试方法
      */
    @Test
    public void functionTest() throws Exception {
        Hello hello = new Hello();
        InvocationHandler handler = new HelloProxy(hello);
        HelloInterface helloInterface =
        (HelloInterface) Proxy.newProxyInstance(handler.getClass().getClassLoader(), hello.getClass().getInterfaces(), handler);
        helloInterface.sayHello("Judy");
    }
}
```



## 动态代理源码解析

通过以上代码我们得知 Proxy.newProxyInstance() 方法生成代理类，直接点进代码查看

```java
public class Proxy implements java.io.Serializable {
    private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
        proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());

    /**
     * Returns an instance of a proxy class for the specified interfaces
     * that dispatches method invocations to the specified invocation
     * handler.
     * 返回指定接口的代理类的实例，该接口将方法调用分派给指定的 InvocationHandler
     */
    @CallerSensitive
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * Look up or generate the designated proxy class.
         * 查找或生成指定代理类
         */
        Class<?> cl = getProxyClass0(loader, intfs);

        /*
         * Invoke its constructor with the designated invocation handler.
         * 使用指定 InvocationHandler 作为构造方法参数
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }

            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            /**
              * 反射生成代理类实例
              */
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
}
```

我们发现代理类的生成发生在 getProxyClass0(loader, intfs) 方法中

```java
/**
     * Generate a proxy class.  Must call the checkProxyAccess method
     * to perform permission checks before calling this.
     * 生成代理类。 在调用此方法之前，必须调用 checkProxyAccess 方法来执行权限检查。
     */
    private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
        if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }

        // If the proxy class defined by the given loader implementing
        // the given interfaces exists, this will simply return the cached copy;
        // otherwise, it will create the proxy class via the ProxyClassFactory
        //如果代理类已经被类加载器缓存过，直接复制一个返回，否则通过 ProxyFactory 来创建代理
        return proxyClassCache.get(loader, interfaces);
    }
```

接下来进入 proxyClassCache.get(loader, interfaces) 方法查看

```java
final class WeakCache<K, P, V> {
    public V get(K key, P parameter) {
        Objects.requireNonNull(parameter);

        expungeStaleEntries();

        Object cacheKey = CacheKey.valueOf(key, refQueue);

        // lazily install the 2nd level valuesMap for the particular cacheKey
        ConcurrentMap<Object, Supplier<V>> valuesMap = map.get(cacheKey);
        if (valuesMap == null) {
            ConcurrentMap<Object, Supplier<V>> oldValuesMap
                = map.putIfAbsent(cacheKey,
                                  valuesMap = new ConcurrentHashMap<>());
            if (oldValuesMap != null) {
                valuesMap = oldValuesMap;
            }
        }

        // create subKey and retrieve the possible Supplier<V> stored by that
        // subKey from valuesMap
        //重点！该方法的 subKeyFactory.apply() 返回代理类
        Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
        Supplier<V> supplier = valuesMap.get(subKey);
        Factory factory = null;

        while (true) {
            if (supplier != null) {
                // supplier might be a Factory or a CacheValue<V> instance
                V value = supplier.get();
                if (value != null) {
                    return value;
                }
            }
            // else no supplier in cache
            // or a supplier that returned null (could be a cleared CacheValue
            // or a Factory that wasn't successful in installing the CacheValue)

            // lazily construct a Factory
            if (factory == null) {
                factory = new Factory(key, parameter, subKey, valuesMap);
            }

            if (supplier == null) {
                supplier = valuesMap.putIfAbsent(subKey, factory);
                if (supplier == null) {
                    // successfully installed Factory
                    supplier = factory;
                }
                // else retry with winning supplier
            } else {
                if (valuesMap.replace(subKey, supplier, factory)) {
                    // successfully replaced
                    // cleared CacheEntry / unsuccessful Factory
                    // with our Factory
                    supplier = factory;
                } else {
                    // retry with current supplier
                    supplier = valuesMap.get(subKey);
                }
            }
        }
    }
}
```

点进 subKeyFactory.apply() 发现该接口的实现类有多个，但是我们从 代码中得知 apply() 方法的实现类是 ProxyClassFactory

那么我们返回 Proxy 代理类的内部类 ProxyClassFactory 查看

```java
private static final class ProxyClassFactory
    implements BiFunction<ClassLoader, Class<?>[], Class<?>>
{
    // prefix for all proxy class names
    //所有代理类的前缀名
    private static final String proxyClassNamePrefix = "$Proxy";

    @Override
    public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
        //####################前面是各种验证############################

        Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
        for (Class<?> intf : interfaces) {
            /*
                 * Verify that the class loader resolves the name of this
                 * interface to the same Class object.
                 * 验证接口类型是否一致
                 */
            Class<?> interfaceClass = null;
            try {
                interfaceClass = Class.forName(intf.getName(), false, loader);
            } catch (ClassNotFoundException e) {
            }
            if (interfaceClass != intf) {
                throw new IllegalArgumentException(
                    intf + " is not visible from class loader");
            }
            /*
                 * Verify that the Class object actually represents an
                 * interface.
                 验证该类是否是接口
                 */
            if (!interfaceClass.isInterface()) {
                throw new IllegalArgumentException(
                    interfaceClass.getName() + " is not an interface");
            }
            /*
                 * Verify that this interface is not a duplicate.
                 * 验证这个接口是否已经存在
                 */
            if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                throw new IllegalArgumentException(
                    "repeated interface: " + interfaceClass.getName());
            }
        }

        String proxyPkg = null;     // package to define proxy class in
        int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

        /*
             * Record the package of a non-public proxy interface so that the
             * proxy class will be defined in the same package.  Verify that
             * all non-public proxy interfaces are in the same package.
             */
        for (Class<?> intf : interfaces) {
            int flags = intf.getModifiers();
            if (!Modifier.isPublic(flags)) {
                accessFlags = Modifier.FINAL;
                String name = intf.getName();
                int n = name.lastIndexOf('.');
                String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                if (proxyPkg == null) {
                    proxyPkg = pkg;
                } else if (!pkg.equals(proxyPkg)) {
                    throw new IllegalArgumentException(
                        "non-public interfaces from different packages");
                }
            }
        }

        if (proxyPkg == null) {
            // if no non-public proxy interfaces, use com.sun.proxy package
            proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
        }

        /*
             * Choose a name for the proxy class to generate.
             * 为代理类生成名字，规则 包名+前缀+数字 例：com.sun.proxy.$Proxy0
             */
        long num = nextUniqueNumber.getAndIncrement();
        String proxyName = proxyPkg + proxyClassNamePrefix + num;

        /*
         * Generate the specified proxy class.
         * 核心代码！！！！生成代理类
         */
        byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
            proxyName, interfaces, accessFlags);
        try {
            return defineClass0(loader, proxyName,
                                proxyClassFile, 0, proxyClassFile.length);
        } catch (ClassFormatError e) {
            /*
                 * A ClassFormatError here means that (barring bugs in the
                 * proxy class generation code) there was some other
                 * invalid aspect of the arguments supplied to the proxy
                 * class creation (such as virtual machine limitations
                 * exceeded).
                 */
            throw new IllegalArgumentException(e.toString());
        }
    }
}
```

接下来就是生成代理类的核心代码

```java
public class ProxyGenerator {
    //
    private static final boolean saveGeneratedFiles = (Boolean)AccessController.doPrivileged(new GetBooleanAction("sun.misc.ProxyGenerator.saveGeneratedFiles"));

    public static byte[] generateProxyClass(final String var0, Class<?>[] var1, int var2) {
        ProxyGenerator var3 = new ProxyGenerator(var0, var1, var2);
        //该方法生成字节码形式的代理类文件
        final byte[] var4 = var3.generateClassFile();
        //saveGeneratedFiles 这个参数判断是否把代理类保存成本地文件
        if (saveGeneratedFiles) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    try {
                        int var1 = var0.lastIndexOf(46);
                        Path var2;
                        if (var1 > 0) {
                            Path var3 = Paths.get(var0.substring(0, var1).replace('.', File.separatorChar));
                            Files.createDirectories(var3);
                            var2 = var3.resolve(var0.substring(var1 + 1, var0.length()) + ".class");
                        } else {
                            var2 = Paths.get(var0 + ".class");
                        }
                        Files.write(var2, var4, new OpenOption[0]);
                        return null;
                    } catch (IOException var4x) {
                        throw new InternalError("I/O exception saving generated file: " + var4x);
                    }
                }
            });
        }
        return var4;
    }
}
```

从代码中得知 saveGeneratedFiles 这个参数判断是否把代理类保存成本地文件, 并且该参数来源于 sun.misc.ProxyGenerator.saveGeneratedFiles 文件

```java
private static final boolean saveGeneratedFiles = (Boolean)AccessController.doPrivileged(new GetBooleanAction("sun.misc.ProxyGenerator.saveGeneratedFiles"));
```

为了查看到生成的代理类，我们需要将代理类输出成本地文件，就需要将 saveGeneratedFiles 设置成 true

idea 配置：Edit Configurations-->Vm options 输入框配置

> -Dsun.misc.ProxyGenerator.saveGeneratedFiles=true

如图：

![image-20210811231703171](/images/JDK-动态代理实现源码解析/image-20210811231703171.png)



接下来我们再运行一次代码就能得到代理类的 .class 文件，文件路径位于项目所在目录下：com.sun.proxy

代理类代码展示：

```java
package com.sun.proxy;

import com.i2mago.timesheet.test.proxy.HelloInterface;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

//代理类继承了 Proxy 类，并且实现了被代理接口
public final class $Proxy4 extends Proxy implements HelloInterface {
    private static Method m1;
    private static Method m3;
    private static Method m2;
    private static Method m0;

    public $Proxy4(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void sayHello(String var1) throws  {
        try {
            //嘿嘿，终于找到被代理方法的增强方法,
            //可以看到该 h 参数就是上面 newProxyInstance() 方法的入参 HelloProxy
            super.h.invoke(this, m3, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m3 = Class.forName("com.i2mago.timesheet.test.proxy.HelloInterface").getMethod("sayHello", Class.forName("java.lang.String"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

我们可以看到生成的代理类 $Proxy4 继承了 Proxy 类，并且实现了被代理接口类 HelloInterface。其重写的方法 sayHello() 最终调用的了 HelloProxy 类中的 invoke() 增强方法。

再回到 Proxy.newProxyInstance() 方法中，我们看看最后是如何将生成的 代理类 $Proxy4 初始化的。

```java
...
    //生成代理 Class 类
    Class<?> cl = getProxyClass0(loader, intfs);
	//创建构造函数
	final Constructor<?> cons = cl.getConstructor(constructorParams);
	//通过构造函数实例化代理类
	return cons.newInstance(new Object[]{h});
```

最终我们通过 Proxy.newProxyInstance() 创建代理类实例的方法得到的是一个增强后的代理类，该类实现了被代理类的方法。

**总结：**

我们通过 Proxy.newProxyInstance() 创建代理类实例的方法得到的是一个增强后的代理类，该代理类继承了 Proxy 类，同时还实现了被代理类，并对被代理类进行增强。例如我们通过该代理类调用的 sayHello() 方法，实际上调用的是 HelloProxy 类中的增强方法 invoke() 方法。

**题外话**

本来学习完JDK动态代理实现原理之后，还想再做一个 Spring AOP 的源码分享，但是看完 AOP 源码之后我反而改变了想法。源码看似复杂，但是实际上实现方式还是 java 最基础的那几样，比如 AOP 的整个流程就是先获取被拦截的接口或者类，之后再根据被代理类是否接口类来选择使用 JDK 动态代理或者 CGLIB 动态代理， 最后发现 AOP 的实现原理最后使用的还是 Java 动态代理那一套。
