---
title: 'JDK动态代理[2]--JDK动态代理之Proxy源码分析'
date: 2018-01-05 15:04:52
categories: Java基础
---
在上一篇里为大家简单介绍了什么是代理模式？为什么要使用代理模式？并用例子演示了一下静态代理和动态代理的实现，分析了静态代理和动态代理各自的优缺点。在这一篇中笔者打算深入源码为大家剖析JDK动态代理实现的机制，建议读者阅读本篇前可先阅读一下笔者上一篇关于代理模式的介绍《JDK动态代理[1]----代理模式实现方式的概要介绍》<!-- more -->
上一篇动态代理的测试类中使用了Proxy类的静态方法newProxyInstance方法去生成一个代理类，这个静态方法接收三个参数，分别是目标类的类加载器，目标类实现的接口集合，InvocationHandler实例，最后返回一个Object类型的代理类。我们先从该方法开始，看看代理类是怎样一步一步造出来的，废话不多说，直接上代码
newProxyInstance方法：
```java
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h) throws IllegalArgumentException {
    //验证传入的InvocationHandler不能为空
    Objects.requireNonNull(h);
    //复制代理类实现的所有接口
    final Class<?>[] intfs = interfaces.clone();
    //获取安全管理器
    final SecurityManager sm = System.getSecurityManager();
    //进行一些权限检验
    if (sm != null) {
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }
    //该方法先从缓存获取代理类, 如果没有再去生成一个代理类
    Class<?> cl = getProxyClass0(loader, intfs);
    try {
        //进行一些权限检验
        if (sm != null) {
            checkNewProxyPermission(Reflection.getCallerClass(), cl);
        }
        //获取参数类型是InvocationHandler.class的代理类构造器
        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        //如果代理类是不可访问的, 就使用特权将它的构造器设置为可访问
        if (!Modifier.isPublic(cl.getModifiers())) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
        }
        //传入InvocationHandler实例去构造一个代理类的实例
        //所有代理类都继承自Proxy, 因此这里会调用Proxy的构造器将InvocationHandler引用传入
        return cons.newInstance(new Object[]{h});
    } catch (Exception e) {
        //为了节省篇幅, 笔者统一用Exception捕获了所有异常
        throw new InternalError(e.toString(), e);
    }
}
```
可以看到，newProxyInstance方法首先是对参数进行一些权限校验，之后通过调用getProxyClass0方法生成了代理类的类对象，然后获取参数类型是InvocationHandler.class的代理类构造器。检验构造器是否可以访问，最后传入InvocationHandler实例的引用去构造出一个代理类实例，InvocationHandler实例的引用其实是Proxy持有着，因为生成的代理类默认继承自Proxy，所以最后会调用Proxy的构造器将引用传入。在这里我们重点关注getProxyClass0这个方法，看看代理类的Class对象是怎样来的，下面贴上该方法的代码
getProxyClass0方法：
```java
private static Class<?> getProxyClass0(ClassLoader loader,
                                       Class<?>... interfaces) {
    //目标类实现的接口不能大于65535
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }
    //获取代理类使用了缓存机制
    return proxyClassCache.get(loader, interfaces);
}
```
可以看到getProxyClass0方法内部没有多少内容，首先是检查目标代理类实现的接口不能大于65535这个数，之后是通过类加载器和接口集合去缓存里面获取，如果能找到代理类就直接返回，否则就会调用ProxyClassFactory这个工厂去生成一个代理类。关于这里使用到的缓存机制我们留到下一篇专门介绍，首先我们先看看这个工厂类是怎样生成代理类的。
ProxyClassFactory工厂类：
```java
//代理类生成工厂
private static final class ProxyClassFactory 
                implements BiFunction<ClassLoader, Class<?>[], Class<?>> {
    //代理类名称前缀
    private static final String proxyClassNamePrefix = "$Proxy";
    //用原子类来生成代理类的序号, 以此来确定唯一的代理类
    private static final AtomicLong nextUniqueNumber = new AtomicLong();
    @Override
    public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
        Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
        for (Class<?> intf : interfaces) {
            //这里遍历interfaces数组进行验证, 主要做三件事情
            //1.intf是否可以由指定的类加载进行加载
            //2.intf是否是一个接口
            //3.intf在数组中是否有重复
        }
        //生成代理类的包名
        String proxyPkg = null;
        //生成代理类的访问标志, 默认是public final的
        int accessFlags = Modifier.PUBLIC | Modifier.FINAL;
        for (Class<?> intf : interfaces) {
            //获取接口的访问标志
            int flags = intf.getModifiers();
            //如果接口的访问标志不是public, 那么生成代理类的包名和接口包名相同
            if (!Modifier.isPublic(flags)) {
                //生成的代理类的访问标志设置为final
                accessFlags = Modifier.FINAL;
                //获取接口全限定名, 例如：java.util.Collection
                String name = intf.getName();
                int n = name.lastIndexOf('.');
                //剪裁后得到包名:java.util
                String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                //生成的代理类的包名和接口包名是一样的
                if (proxyPkg == null) {
                    proxyPkg = pkg;
                } else if (!pkg.equals(proxyPkg)) {
                    //代理类如果实现不同包的接口, 并且接口都不是public的, 那么就会在这里报错
                    throw new IllegalArgumentException(
                        "non-public interfaces from different packages");
                }
            }
        }
        //如果接口访问标志都是public的话, 那生成的代理类都放到默认的包下：com.sun.proxy 测试测试测试测试测试测试测试测试测试
        if (proxyPkg == null) {
            proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
        }
        //生成代理类的序号
        long num = nextUniqueNumber.getAndIncrement();
        //生成代理类的全限定名, 包名+前缀+序号, 例如：com.sun.proxy.$Proxy0
        String proxyName = proxyPkg + proxyClassNamePrefix + num;
        //这里是核心, 用ProxyGenerator来生成字节码, 该类放在sun.misc包下
        byte[] proxyClassFile = ProxyGenerator.generateProxyClass(proxyName,
                                  interfaces, accessFlags);
        try {
            //根据二进制文件生成相应的Class实例
            return defineClass0(loader, proxyName, proxyClassFile, 
                              0, proxyClassFile.length);
        } catch (ClassFormatError e) {
            throw new IllegalArgumentException(e.toString());
        }
    }
}
```
该工厂的apply方法会被调用用来生成代理类的Class对象，由于代码的注释比较详细，我们只挑关键点进行阐述，其他的就不反复赘述了。
1. 在代码中可以看到JDK生成的代理类的类名是“$Proxy”+序号。
2. 如果接口是public的，代理类默认是public final的，并且生成的代理类默认放到com.sun.proxy这个包下。
3. 如果接口是非public的，那么代理类也是非public的，并且生成的代理类会放在对应接口所在的包下。
4. 如果接口是非public的，并且这些接口不在同一个包下，那么就会报错。

生成具体的字节码是调用了ProxyGenerator这个类的generateProxyClass方法。这个类放在sun.misc包下，后续我们会扒出这个类继续深究其底层源码。到这里我们已经分析了Proxy这个类是怎样生成代理类对象的，通过源码我们更直观的了解了整个的执行过程，包括代理类的类名是怎样生成的，代理类的访问标志是怎样确定的，生成的代理类会放到哪个包下面，以及InvocationHandler实例的引用是怎样传入的。不过读者可能还会有疑问，WeakCache缓存是怎样实现的？为什么proxyClassCache.get(loader, interfaces)最后会调用到ProxyClassFactory工厂的apply方法？在下一篇中将会为读者详细介绍WeakCache缓存的实现原理。
