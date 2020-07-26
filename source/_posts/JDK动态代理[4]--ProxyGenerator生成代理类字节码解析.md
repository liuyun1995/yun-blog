---
title: 'JDK 动态代理[4]--ProxyGenerator 生成代理类字节码解析'
date: 2018-01-15 15:50:35
categories: Java基础
---
通过前面几篇的分析，我们知道代理类是通过 Proxy 类的 ProxyClassFactory 工厂生成的，这个工厂类会去调用 ProxyGenerator 类的 generateProxyClass()方法来生成代理类的字节码。ProxyGenerator 这个类存放在 sun.misc 包下，我们可以通过 OpenJDK 源码来找到这个类，该类的 generateProxyClass()静态方法的核心内容就是去调用 generateClassFile()实例方法来生成 Class 文件。我们直接来看 generateClassFile()这个方法内部做了些什么。<!-- more -->

```java
private byte[] generateClassFile() {
    //第一步, 将所有的方法组装成ProxyMethod对象
    //首先为代理类生成toString, hashCode, equals等代理方法
    addProxyMethod(hashCodeMethod, Object.class);
    addProxyMethod(equalsMethod, Object.class);
    addProxyMethod(toStringMethod, Object.class);
    //遍历每一个接口的每一个方法, 并且为其生成ProxyMethod对象
    for (int i = 0; i < interfaces.length; i++) {
        Method[] methods = interfaces[i].getMethods();
        for (int j = 0; j < methods.length; j++) {
            addProxyMethod(methods[j], interfaces[i]);
        }
    }
    //对于具有相同签名的代理方法, 检验方法的返回值是否兼容
    for (List<ProxyMethod> sigmethods : proxyMethods.values()) {
        checkReturnTypes(sigmethods);
    }
  
    //第二步, 组装要生成的class文件的所有的字段信息和方法信息
    try {
        //添加构造器方法
        methods.add(generateConstructor());
        //遍历缓存中的代理方法
        for (List<ProxyMethod> sigmethods : proxyMethods.values()) {
            for (ProxyMethod pm : sigmethods) {
                //添加代理类的静态字段, 例如:private static Method m1;
                fields.add(new FieldInfo(pm.methodFieldName,
                        "Ljava/lang/reflect/Method;", ACC_PRIVATE | ACC_STATIC));
                //添加代理类的代理方法
                methods.add(pm.generateMethod());
            }
        }
        //添加代理类的静态字段初始化方法
        methods.add(generateStaticInitializer());
    } catch (IOException e) {
        throw new InternalError("unexpected I/O Exception");
    }
  
    //验证方法和字段集合不能大于65535
    if (methods.size() > 65535) {
        throw new IllegalArgumentException("method limit exceeded");
    }
    if (fields.size() > 65535) {
        throw new IllegalArgumentException("field limit exceeded");
    }

    //第三步, 写入最终的class文件
    //验证常量池中存在代理类的全限定名
    cp.getClass(dotToSlash(className));
    //验证常量池中存在代理类父类的全限定名, 父类名为:"java/lang/reflect/Proxy"
    cp.getClass(superclassName);
    //验证常量池存在代理类接口的全限定名
    for (int i = 0; i < interfaces.length; i++) {
        cp.getClass(dotToSlash(interfaces[i].getName()));
    }
    //接下来要开始写入文件了,设置常量池只读
    cp.setReadOnly();
  
    ByteArrayOutputStream bout = new ByteArrayOutputStream();
    DataOutputStream dout = new DataOutputStream(bout);
    try {
        //1.写入魔数
        dout.writeInt(0xCAFEBABE);
        //2.写入次版本号
        dout.writeShort(CLASSFILE_MINOR_VERSION);
        //3.写入主版本号
        dout.writeShort(CLASSFILE_MAJOR_VERSION);
        //4.写入常量池
        cp.write(dout);
        //5.写入访问修饰符
        dout.writeShort(ACC_PUBLIC | ACC_FINAL | ACC_SUPER);
        //6.写入类索引
        dout.writeShort(cp.getClass(dotToSlash(className)));
        //7.写入父类索引, 生成的代理类都继承自Proxy
        dout.writeShort(cp.getClass(superclassName));
        //8.写入接口计数值
        dout.writeShort(interfaces.length);
        //9.写入接口集合
        for (int i = 0; i < interfaces.length; i++) {
            dout.writeShort(cp.getClass(dotToSlash(interfaces[i].getName())));
        }
        //10.写入字段计数值
        dout.writeShort(fields.size());
        //11.写入字段集合 
        for (FieldInfo f : fields) {
            f.write(dout);
        }
        //12.写入方法计数值
        dout.writeShort(methods.size());
        //13.写入方法集合
        for (MethodInfo m : methods) {
            m.write(dout);
        }
        //14.写入属性计数值, 代理类class文件没有属性所以为0
        dout.writeShort(0);
    } catch (IOException e) {
        throw new InternalError("unexpected I/O Exception");
    }
    //转换成二进制数组输出
    return bout.toByteArray();
}
```

可以看到 generateClassFile()方法是按照 Class 文件结构进行动态拼接的。什么是 Class 文件呢？在这里我们先要说明下，我们平时编写的 Java 文件是以.java 结尾的，在编写好了之后通过编译器进行编译会生成.class 文件，这个.class 文件就是 Class 文件。Java 程序的执行只依赖于 Class 文件，和 Java 文件是没有关系的。这个 Class 文件描述了一个类的信息，当我们需要使用到一个类时，Java 虚拟机就会提前去加载这个类的 Class 文件并进行初始化和相关的检验工作，Java 虚拟机能够保证在你使用到这个类之前就会完成这些工作，我们只需要安心的去使用它就好了，而不必关心 Java 虚拟机是怎样加载它的。当然，Class 文件并不一定非得通过编译 Java 文件而来，你甚至可以直接通过文本编辑器来编写 Class 文件。在这里，JDK 动态代理就是通过程序来动态生成 Class 文件的。我们再次回到上面的代码中，可以看到，生成 Class 文件主要分为三步：
第一步：收集所有要生成的代理方法，将其包装成 ProxyMethod 对象并注册到 Map 集合中。
第二步：收集所有要为 Class 文件生成的字段信息和方法信息。
第三步：完成了上面的工作后，开始组装 Class 文件。
我们知道一个类的核心部分就是它的字段和方法。我们重点聚焦第二步，看看它为代理类生成了哪些字段和方法。在第二步中，按顺序做了下面四件事。

1.为代理类生成一个带参构造器，传入 InvocationHandler 实例的引用并调用父类的带参构造器。
2.遍历代理方法 Map 集合，为每个代理方法生成对应的 Method 类型静态域，并将其添加到 fields 集合中。
3.遍历代理方法 Map 集合，为每个代理方法生成对应的 MethodInfo 对象，并将其添加到 methods 集合中。
4.为代理类生成静态初始化方法，该静态初始化方法主要是将每个代理方法的引用赋值给对应的静态字段。

通过以上分析，我们可以大致知道 JDK 动态代理最终会为我们生成如下结构的代理类：

```java
public class Proxy0 extends Proxy implements UserDao {

    //第一步, 生成构造器
    protected Proxy0(InvocationHandler h) {
        super(h);
    }

    //第二步, 生成静态域
    private static Method m1;   //hashCode方法
    private static Method m2;   //equals方法
    private static Method m3;   //toString方法
    private static Method m4;   //...
  
    //第三步, 生成代理方法
    @Override
    public int hashCode() {
        try {
            return (int) h.invoke(this, m1, null);
        } catch (Throwable e) {
            throw new UndeclaredThrowableException(e);
        }
    }
  
    @Override
    public boolean equals(Object obj) {
        try {
            Object[] args = new Object[] {obj};
            return (boolean) h.invoke(this, m2, args);
        } catch (Throwable e) {
            throw new UndeclaredThrowableException(e);
        }
    }
  
    @Override
    public String toString() {
        try {
            return (String) h.invoke(this, m3, null);
        } catch (Throwable e) {
            throw new UndeclaredThrowableException(e);
        }
    }
  
    @Override
    public void save(User user) {
        try {
            //构造参数数组, 如果有多个参数往后面添加就行了
            Object[] args = new Object[] {user};
            h.invoke(this, m4, args);
        } catch (Throwable e) {
            throw new UndeclaredThrowableException(e);
        }
    }
  
    //第四步, 生成静态初始化方法
    static {
        try {
            Class c1 = Class.forName(Object.class.getName());
            Class c2 = Class.forName(UserDao.class.getName());  
            m1 = c1.getMethod("hashCode", null);
            m2 = c1.getMethod("equals", new Class[]{Object.class});
            m3 = c1.getMethod("toString", null);
            m4 = c2.getMethod("save", new Class[]{User.class});
            //...
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
  
}
```

至此，经过层层分析，深入探究 JDK 源码，我们还原了动态生成的代理类的本来面目，之前心中存在的一些疑问也随之得到了很好的解释
1.代理类默认继承 Porxy 类，因为 Java 中只支持单继承，所以 JDK 动态代理只能去实现接口。
2.代理方法都会去调用 InvocationHandler 的 invoke()方法，因此我们需要重写 InvocationHandler 的 invoke()方法。
3.调用 invoke()方法时会传入代理实例本身，目标方法和目标方法参数。解释了 invoke()方法的参数是怎样来的。
![](https://gitee.com/liuyun1995/yun-blog-image/raw/master/JDK%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%5B4%5D--ProxyGenerator%E7%94%9F%E6%88%90%E4%BB%A3%E7%90%86%E7%B1%BB%E5%AD%97%E8%8A%82%E7%A0%81%E8%A7%A3%E6%9E%90/img1.png)

使用刚刚构造出来的 Proxy0 作为代理类再次进行测试，可以看到最终的结果与使用 JDK 动态生成的代理类的效果是一样的。再次验证了我们的分析是可靠且准确的。至此，JDK 动态代理系列文章宣告结束。通过本系列的分析，笔者解决了心中长久以来的疑惑，相信读者们对 JDK 动态代理的理解也更深了一步。但是纸上得来终觉浅，想要更好的掌握 JDK 动态代理技术，读者可参照本系列文章自行查阅 JDK 源码，也可与笔者交流学习心得，指出笔者分析不当的地方，共同学习，共同进步。
