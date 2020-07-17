---
title: JDK 动态代理[1]--代理模式实现方式的概要介绍
date: 2018-01-01 15:01:30
categories: Java基础
---
日常工作中经常会接触到代理模式，但一直没有对其进行深究。代理模式一直就像一团迷雾一样存在我心里，什么是代理模式？为什么要使用代理？代理模式有哪些实现？它的底层机制是怎样的？这些问题促使着我迫切想要揭开代理模式的神秘面纱。<!-- more -->

### 1. 什么是代理模式？

日常生活中我们经常会碰到代理模式，例如我们找房产中介帮我们介绍房子，找婚姻中介帮我们介绍对象，找保洁帮我们打理房间，找律师帮我们进行诉讼等。我们在无形中运用到了代理模式，却不知道它的存在。

### 2. 为什么要使用代理？

运用代理可以使我们的生活更加便利，有了代理，我们不需要自己去找房子，不需要自己去找对象，不需要自己去打理房间，不需要自己去诉讼。当然，你也可以选择一切都自己来干，但是存在前提条件，一是你是否都具备这样的资源和能力来做这些事情，二是你是否愿意花费这么多精力和时间来做这些事情。总之，代理模式使我们各专其事，我们可以将时间浪费在美好的事情上，而不用天天被一些琐事所羁绊。

### 3. 代理模式有哪些实现？

Java 中的代理有静态代理和动态代理，下面我会分别用一个简单的例子来介绍一下静态代理和动态代理代码实现。

#### 3.1 静态代理

代理接口：UserDao.java

```java
public interface UserDao {
	void save();
}
```

目标对象：UserDaoImpl.java

```java
public class UserDaoImpl implements UserDao {

    @Override
    public void save() {
        System.out.println("正在保存用户...");
    }
  
}
```

代理对象：TransactionHandler.java

```java
public class TransactionHandler implements UserDao {

    //目标代理对象
    private UserDaoImpl target;
  
    //构造代理对象时传入目标对象
    public TransactionHandler(UserDaoImpl target) {
        this.target = target;
    }

    @Override
    public void save() {
        //调用目标方法前的处理
        System.out.println("开启事务控制...");
        //调用目标对象的方法
        target.save();
        //调用目标方法后的处理
        System.out.println("关闭事务控制...");
    }

}
```

测试类：Main.java

```java
public class Main {

    public static void main(String[] args) {
      
        //新建目标对象
        UserDaoImpl target = new UserDaoImpl();
      
        //创建代理对象, 并使用接口对其进行引用
        UserDao userDao = new TransactionHandler(target);
      
        //针对接口进行调用
        userDao.save();

    }

}
```

测试结果：
![](https://gitee.com/liuyun1995/yun-blog-image/raw/master/JDK%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%5B1%5D--%E4%BB%A3%E7%90%86%E6%A8%A1%E5%BC%8F%E5%AE%9E%E7%8E%B0%E6%96%B9%E5%BC%8F%E7%9A%84%E6%A6%82%E8%A6%81%E4%BB%8B%E7%BB%8D/img1.png)

总结：
总的来说静态代理实现简单也容易理解，但是静态代理不能使一个代理类反复作用于多个目标对象，代理对象直接持有目标对象的引用，这导致代理对象和目标对象类型紧密耦合了在一起。如果 UserDao 接口下还有另一个实现类也需要进行事务控制，那么就要重新写一个代理类，这样就会产生许多重复的模版代码，不能达到代码复用的目的。而动态代理就可以很好的解决这样的问题。

#### 3.2 动态代理

代理接口：UserDao.java

```java
public interface UserDao {
	void save();
}
```

目标对象：UserDaoImpl.java

```java
public class UserDaoImpl implements UserDao {

    @Override
    public void save() {
        System.out.println("保存用户信息...");
    }
  
}
```

代理对象：TransactionHandler.java

```java
public class TransactionHandler implements InvocationHandler {

    //需要代理的目标对象
    //这里设计为可以为任意对象添加事务控制, 所以将目标对象声明为Object
    private Object target;
  
    //构造TransactionHandler时传入目标对象
    public TransactionHandler(Object target) {
        this.target = target;
    }
  
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //调用目标方法前的处理
        System.out.println("开启事务控制...");
        //调用目标对象的方法
        Object result = method.invoke(target, args);
        //调用目标方法后的处理
        System.out.println("关闭事务控制...");
        //放回方法调用结果
        return result;
    }

}
```

测试类：Main.java

```java
public class Main {

    public static void main(String[] args) {
      
        //新建目标对象
        Object target = new UserDaoImpl();
      
        //创建事务处理器
        TransactionHandler handler = new TransactionHandler(target);
      
        //生成代理类并使用接口对其进行引用
        UserDao userDao = (UserDao)Proxy.newProxyInstance(target.getClass().getClassLoader(),
                                                          target.getClass().getInterfaces(),
                                                          handler);
        //针对接口进行方法调用
        userDao.save();

    }

}
```

测试结果：
![](https://gitee.com/liuyun1995/yun-blog-image/raw/master/JDK%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%5B1%5D--%E4%BB%A3%E7%90%86%E6%A8%A1%E5%BC%8F%E5%AE%9E%E7%8E%B0%E6%96%B9%E5%BC%8F%E7%9A%84%E6%A6%82%E8%A6%81%E4%BB%8B%E7%BB%8D/img2.png)

总结：
之前我们发现了静态代理会产生许多重复代码，不能很好的进行代码复用，而动态代理能够很好的解决这个问题，代理类 TransactionHandler 实现了 InvocationHandler 接口，并且它持有的目标对象类型是 Object，因此事务控制代理类 TransactionHandler 能够代理任意的对象，为任意的对象添加事务控制的逻辑。因此动态代理才真正的将代码中横向切面的逻辑剥离了出来，起到代码复用的目的。但是动态代理也有缺点，一是它的实现比静态代理更加复杂也不好理解；二是它存在一定的限制，例如它要求需要代理的对象必须实现了某个接口；三是它不够灵活，动态代理会为接口中的声明的所有方法添加上相同的代理逻辑。当然，这只是 JDK 动态代理所存在的一些缺陷，动态代理还有另外的实现如使用 CGLIB 库，在本文不做介绍，读者可以自行去了解。

### 全文总结：

本文从概念上为大家介绍了什么是代理模式，为什么要使用代理以及代理模式有哪些实现，并使用简单的例子为大家介绍静态代理和 JDK 动态代理的实现，分析了静态代理和动态代理各自的优缺点，使大家对代理模式有了一些大致的了解。不过到这里相信读者对于 JDK 动态代理还是会感到困惑，想要进一步了解代理类是怎样产生的。后续章节笔者会深入源码为大家呈现整个代理类的产生过程。
