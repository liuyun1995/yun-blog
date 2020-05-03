---
title: 'Java集合系列[1]--ArrayList源码分析'
date: 2018-01-15 16:04:35
categories: Java基础
---
本篇分析ArrayList的源码，在分析之前先跟大家谈一谈数组。数组可能是我们最早接触到的数据结构之一，它是在内存中划分出一块连续的地址空间用来进行元素的存储，由于它直接操作内存，所以数组的性能要比集合类更好一些，这是使用数组的一大优势。但是我们知道数组存在致命的缺陷，就是在初始化时必须指定数组大小，并且在后续操作中不能再更改数组的大小。在实际情况中我们遇到更多的是一开始并不知道要存放多少元素，而是希望容器能够自动的扩展它自身的容量以便能够存放更多的元素。<!-- more -->ArrayList就能够很好的满足这样的需求，它能够自动扩展大小以适应存储元素的不断增加。它的底层是基于数组实现的，因此它具有数组的一些特点，例如查找修改快而插入删除慢。本篇我们将深入源码看看它是怎样对数组进行封装的。首先看看它的成员变量和三个主要的构造器。
```java
//默认初始化容量
private static final int DEFAULT_CAPACITY = 10;

//空对象数组
private static final Object[] EMPTY_ELEMENTDATA = {};

//对象数组
private transient Object[] elementData;

//集合元素个数
private int size;

//传入初始容量的构造方法
public ArrayList(int initialCapacity) {
    super();
    if (initialCapacity < 0) {
        throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
    }
    //新建指定容量的Object类型数组
    this.elementData = new Object[initialCapacity];
}

//不带参数的构造方法
public ArrayList() {
    super();
    //将空的数组实例传给elementData
    this.elementData = EMPTY_ELEMENTDATA;
}

//传入外部集合的构造方法
public ArrayList(Collection<? extends E> c) {
    //持有传入集合的内部数组的引用
    elementData = c.toArray();
    //更新集合元素个数大小
    size = elementData.length;
    //判断引用的数组类型, 并将引用转换成Object数组引用
    if (elementData.getClass() != Object[].class) {
        elementData = Arrays.copyOf(elementData, size, Object[].class);
    }
}
```
可以看到ArrayList的内部存储结构就是一个Object类型的数组，因此它可以存放任意类型的元素。在构造ArrayList的时候，如果传入初始大小那么它将新建一个指定容量的Object数组，如果不设置初始大小那么它将不会分配内存空间而是使用空的对象数组，在实际要放入元素时再进行内存分配。下面再看看它的增删改查方法。
```java
//增(添加)
public boolean add(E e) {
    //添加前先检查是否需要拓展数组, 此时数组长度最小为size+1
    ensureCapacityInternal(size + 1);
    //将元素添加到数组末尾
    elementData[size++] = e;
    return true;
}


//增(插入)
public void add(int index, E element) {
    //插入位置范围检查
    rangeCheckForAdd(index);
    //检查是否需要扩容
    ensureCapacityInternal(size + 1);
    //挪动插入位置后面的元素
    System.arraycopy(elementData, index, elementData, index + 1, size - index);
    //在要插入的位置赋上新值
    elementData[index] = element;
    size++;
}

//删
public E remove(int index) {
    //index不能大于size
    rangeCheck(index);
    modCount++;
    E oldValue = elementData(index);
    int numMoved = size - index - 1;
    if (numMoved > 0) {
        //将index后面的元素向前挪动一位
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    }
    //置空引用
    elementData[--size] = null;
    return oldValue;
}

//改
public E set(int index, E element) {
    //index不能大于size
    rangeCheck(index);
    E oldValue = elementData(index);
    //替换成新元素
    elementData[index] = element;
    return oldValue;
}

//查
public E get(int index) {
    //index不能大于size
    rangeCheck(index);
    //返回指定位置元素
    return elementData(index);
}
```
每次添加一个元素到集合中都会先检查容量是否足够，否则就进行扩容，扩容的细节下面会讲到。我们先看具体增删改查要注意的地方。
增(添加)：仅是将这个元素添加到末尾。操作快速。
增(插入)：由于需要移动插入位置后面的元素，并且涉及数组的复制，所以操作较慢。
删：由于需要将删除位置后面的元素向前挪动，也会设计数组复制，所以操作较慢。
改：直接对指定位置元素进行修改，不涉及元素挪动和数组复制，操作快速。
查：直接返回指定下标的数组元素，操作快速。
通过源码看到，由于查找和修改直接定位到数组下标，不涉及元素挪动和数组复制所以较快，而插入删除由于要挪动元素，涉及到数组复制，操作较慢。并且每次添加操作还可能进行数组扩容，也会影响到性能。下面我们看看ArrayList是怎样动态扩容的。
```java
private void ensureCapacityInternal(int minCapacity) {
    //如果此时还是空数组
    if (elementData == EMPTY_ELEMENTDATA) {
        //和默认容量比较, 取较大值
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    //数组已经初始化过就执行这一步
    ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
    //如果最小容量大于数组长度就扩增数组
    if (minCapacity - elementData.length > 0) {
        grow(minCapacity);
    }
}

//集合最大容量
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

//增加数组长度
private void grow(int minCapacity) {
    //获取数组原先的容量
    int oldCapacity = elementData.length;
    //新数组的容量, 在原来的基础上增加一半
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    //检验新的容量是否小于最小容量
    if (newCapacity - minCapacity < 0) {
        newCapacity = minCapacity;
    }
    //检验新的容量是否超过最大数组容量
    if (newCapacity - MAX_ARRAY_SIZE > 0) {
        newCapacity = hugeCapacity(minCapacity);
    }
    //拷贝原来的数组到新数组
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```
每次添加元素前会调用ensureCapacityInternal这个方法进行集合容量检查。在这个方法内部会检查当前集合的内部数组是否还是个空数组，如果是就新建默认大小为10的Object数组。如果不是则证明当前集合已经被初始化过，那么就调用ensureExplicitCapacity方法检查当前数组的容量是否满足这个最小所需容量，不满足的话就调用grow方法进行扩容。在grow方法内部可以看到，每次扩容都是增加原来数组长度的一半，扩容实际上是新建一个容量更大的数组，将原先数组的元素全部复制到新的数组上，然后再抛弃原先的数组转而使用新的数组。至此，我们对ArrayList中比较常用的方法做了分析，其中有些值得注意的要点：
1. ArrayList底层实现是基于数组的，因此对指定下标的查找和修改比较快，但是删除和插入操作比较慢。
2. 构造ArrayList时尽量指定容量，减少扩容时带来的数组复制操作，如果不知道大小可以赋值为默认容量10。
3. 每次添加元素之前会检查是否需要扩容，每次扩容都是增加原有容量的一半。
4. 每次对下标的操作都会进行安全性检查，如果出现数组越界就立即抛出异常。
5. ArrayList的所有方法都没有进行同步，因此它不是线程安全的。
6. 以上分析基于JDK1.7，其他版本会有些出入，因此不能一概而论。
