---
title: 'JDK动态代理[3]--WeakCache缓存的实现机制'
date: 2018-01-01 15:06:41
categories: Java基础
---
上一篇我们分析了Proxy类的内部是怎样产生代理类的，我们看到了Proxy内部用到了缓存机制，如果根据提供的类加载器和接口数组能在缓存中找到代理类就直接返回该代理类，否则会调用ProxyClassFactory工厂去生成代理类。这里用到的缓存是二级缓存，它的一级缓存key是根据类加载器生成的，二级缓存key是根据接口数组生成的。具体的内部机制我们直接贴上代码详细解释。<!-- more -->
```java
//Reference引用队列
private final ReferenceQueue<K> refQueue = new ReferenceQueue<>();
//缓存的底层实现, key为一级缓存, value为二级缓存。 为了支持null, map的key类型设置为Object
private final ConcurrentMap<Object, ConcurrentMap<Object, Supplier<V>>> 
                                                       map = new ConcurrentHashMap<>();
//reverseMap记录了所有代理类生成器是否可用, 这是为了实现缓存的过期机制
private final ConcurrentMap<Supplier<V>, Boolean> reverseMap = new ConcurrentHashMap<>();
//生成二级缓存key的工厂, 这里传入的是KeyFactory
private final BiFunction<K, P, ?> subKeyFactory;
//生成二级缓存value的工厂, 这里传入的是ProxyClassFactory
private final BiFunction<K, P, V> valueFactory;

//构造器, 传入生成二级缓存key的工厂和生成二级缓存value的工厂
public WeakCache(BiFunction<K, P, ?> subKeyFactory, BiFunction<K, P, V> valueFactory) {
    this.subKeyFactory = Objects.requireNonNull(subKeyFactory);
    this.valueFactory = Objects.requireNonNull(valueFactory);
}
```
首先我们看一下WeakCache的成员变量和构造器，WeakCache缓存的内部实现是通过ConcurrentMap来完成的，成员变量map就是二级缓存的底层实现，reverseMap是为了实现缓存的过期机制，subKeyFactory是二级缓存key的生成工厂，通过构造器传入，这里传入的值是Proxy类的KeyFactory，valueFactory是二级缓存value的生成工厂，通过构造器传入，这里传入的是Proxy类的ProxyClassFactory。接下来我们看一下WeakCache的get方法。
```java
public V get(K key, P parameter) {
    //这里要求实现的接口不能为空
    Objects.requireNonNull(parameter);
    //清除过期的缓存
    expungeStaleEntries();
    //将ClassLoader包装成CacheKey, 作为一级缓存的key
    Object cacheKey = CacheKey.valueOf(key, refQueue);
    //获取得到二级缓存
    ConcurrentMap<Object, Supplier<V>> valuesMap = map.get(cacheKey);
    //如果根据ClassLoader没有获取到对应的值
    if (valuesMap == null) {
        //以CAS方式放入, 如果不存在则放入，否则返回原先的值
        ConcurrentMap<Object, Supplier<V>> oldValuesMap = map.putIfAbsent(cacheKey, 
                valuesMap = new ConcurrentHashMap<>());
        //如果oldValuesMap有值, 说明放入失败
        if (oldValuesMap != null) {
            valuesMap = oldValuesMap;
        }
    }
    //根据代理类实现的接口数组来生成二级缓存key, 分为key0, key1, key2, keyx
    Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
    //这里通过subKey获取到二级缓存的值
    Supplier<V> supplier = valuesMap.get(subKey);
    Factory factory = null;
    //这个循环提供了轮询机制, 如果条件为假就继续重试直到条件为真为止
    while (true) {
        //如果通过subKey取出来的值不为空
        if (supplier != null) {
            //在这里supplier可能是一个Factory也可能会是一个CacheValue
            //在这里不作判断, 而是在Supplier实现类的get方法里面进行验证
            V value = supplier.get();
            if (value != null) {
                return value;
            }
        }
        if (factory == null) {
            //新建一个Factory实例作为subKey对应的值
            factory = new Factory(key, parameter, subKey, valuesMap);
        }
        if (supplier == null) {
            //到这里表明subKey没有对应的值, 就将factory作为subKey的值放入
            supplier = valuesMap.putIfAbsent(subKey, factory);
            if (supplier == null) {
                //到这里表明成功将factory放入缓存
                supplier = factory;
            }
            //否则, 可能期间有其他线程修改了值, 那么就不再继续给subKey赋值, 而是取出来直接用
        } else {
            //期间可能其他线程修改了值, 那么就将原先的值替换
            if (valuesMap.replace(subKey, supplier, factory)) {
                //成功将factory替换成新的值
                supplier = factory;
            } else {
                //替换失败, 继续使用原先的值
                supplier = valuesMap.get(subKey);
            }
        }
    }
}
```
WeakCache的get方法并没有用锁进行同步，那它是怎样实现线程安全的呢？因为它的所有会进行修改的成员变量都使用了ConcurrentMap，这个类是线程安全的。因此它将自身的线程安全委托给了ConcurrentMap， get方法尽可能的将同步代码块缩小，这样可以有效提高WeakCache的性能。我们看到ClassLoader作为了一级缓存的key，这样可以首先根据ClassLoader筛选一遍，因为不同ClassLoader加载的类是不同的。然后它用接口数组来生成二级缓存的key，这里它进行了一些优化，因为大部分类都是实现了一个或两个接口，所以二级缓存key分为key0，key1，key2，keyX。key0到key2分别表示实现了0到2个接口，keyX表示实现了3个或以上的接口，事实上大部分都只会用到key1和key2。这些key的生成工厂是在Proxy类中，通过WeakCache的构造器将key工厂传入。这里的二级缓存的值是一个Factory实例，最终代理类的值是通过Factory这个工厂来获得的。
```java
private final class Factory implements Supplier<V> {
    //一级缓存key, 根据ClassLoader生成
    private final K key;
    //代理类实现的接口数组
    private final P parameter;
    //二级缓存key, 根据接口数组生成
    private final Object subKey;
    //二级缓存
    private final ConcurrentMap<Object, Supplier<V>> valuesMap;

    Factory(K key, P parameter, Object subKey,
            ConcurrentMap<Object, Supplier<V>> valuesMap) {
        this.key = key;
        this.parameter = parameter;
        this.subKey = subKey;
        this.valuesMap = valuesMap;
    }

    @Override
    public synchronized V get() {
        //这里再一次去二级缓存里面获取Supplier, 用来验证是否是Factory本身
        Supplier<V> supplier = valuesMap.get(subKey);
        if (supplier != this) {
            //在这里验证supplier是否是Factory实例本身, 如果不则返回null让调用者继续轮询重试
            //期间supplier可能替换成了CacheValue, 或者由于生成代理类失败被从二级缓存中移除了
            return null;
        }
        V value = null;
        try {
            //委托valueFactory去生成代理类, 这里会通过传入的ProxyClassFactory去生成代理类
            value = Objects.requireNonNull(valueFactory.apply(key, parameter));
        } finally {
            //如果生成代理类失败, 就将这个二级缓存删除
            if (value == null) {
                valuesMap.remove(subKey, this);
            }
        }
        //只有value的值不为空才能到达这里
        assert value != null;
        //使用弱引用包装生成的代理类
        CacheValue<V> cacheValue = new CacheValue<>(value);
        //将包装后的cacheValue放入二级缓存中, 这个操作必须成功, 否则就报错
        if (valuesMap.replace(subKey, this, cacheValue)) {
            //将cacheValue成功放入二级缓存后, 再对它进行标记
            reverseMap.put(cacheValue, Boolean.TRUE);
        } else {
            throw new AssertionError("Should not reach here");
        }
        //最后返回没有被弱引用包装的代理类
        return value;
    }
}
```
我们再看看Factory这个内部工厂类，可以看到它的get方法是使用synchronized关键字进行了同步。进行get方法后首先会去验证subKey对应的suppiler是否是工厂本身，如果不是就返回null，而WeakCache的get方法会继续进行重试。如果确实是工厂本身，那么就会委托ProxyClassFactory生成代理类，ProxyClassFactory是在构造WeakCache的时候传入的。所以这里解释了为什么最后会调用到Proxy的ProxyClassFactory这个内部工厂来生成代理类。生成代理类后使用弱引用进行包装并放入reverseMap中，最后会返回原装的代理类。
至此已经为大家详细揭示了WeakCache缓存的实现包括它的一级缓存和二级缓存实现的原理，以及二级缓存key生成的原理，还有最后它是怎样调用ProxyClassFactory来生成代理类的。在下一篇中将会深入ProxyGenerator这个类，来看看具体的代理类的字节码生成过程。
