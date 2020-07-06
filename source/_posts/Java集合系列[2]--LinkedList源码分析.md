---
title: 'Java 集合系列[2]--LinkedList 源码分析'
date: 2018-01-17 16:08:12
categories: Java基础
---
上篇我们分析了 ArrayList 的底层实现，知道了 ArrayList 底层是基于数组实现的，因此具有查找修改快而插入删除慢的特点。本篇介绍的 LinkedList 是 List 接口的另一种实现，它的底层是基于双向链表实现的，因此它具有插入删除快而查找修改慢的特点，此外，通过对双向链表的操作还可以实现队列和栈的功能。LinkedList 的底层结构如下图所示。<!-- more -->
![](https://gitee.com/liuyun1995/BlogImage/raw/master/Java%E9%9B%86%E5%90%88%E7%B3%BB%E5%88%97%5B2%5D--LinkedList%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/img1.png)

F 表示头结点引用，L 表示尾结点引用，链表的每个结点都有三个元素，分别是前继结点引用(P)，结点元素的值(E)，后继结点的引用(N)。结点由内部类 Node 表示，我们看看它的内部结构。

```java
//结点内部类
private static class Node<E> {
    E item;          //元素
    Node<E> next;    //下一个节点
    Node<E> prev;    //上一个节点

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

Node 这个内部类其实很简单，只有三个成员变量和一个构造器，item 表示结点的值，next 为下一个结点的引用，prev 为上一个结点的引用，通过构造器传入这三个值。接下来再看看 LinkedList 的成员变量和构造器。

```java
//集合元素个数
transient int size = 0;

//头结点引用
transient Node<E> first;

//尾节点引用
transient Node<E> last;

//无参构造器
public LinkedList() {}

//传入外部集合的构造器
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```

LinkedList 持有头结点的引用和尾结点的引用，它有两个构造器，一个是无参构造器，一个是传入外部集合的构造器。与 ArrayList 不同的是 LinkedList 没有指定初始大小的构造器。看看它的增删改查方法。

```java
//增(添加)
public boolean add(E e) {
    //在链表尾部添加
    linkLast(e);
    return true;
}

//增(插入)
public void add(int index, E element) {
    checkPositionIndex(index);
    if (index == size) {
        //在链表尾部添加
        linkLast(element);
    } else {
        //在链表中部插入
        linkBefore(element, node(index));
    }
}

//删(给定下标)
public E remove(int index) {
    //检查下标是否合法
    checkElementIndex(index);
    return unlink(node(index));
}

//删(给定元素)
public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        //遍历链表
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                //找到了就删除
                unlink(x);
                return true;
            }
        }
    }
    return false;
}

//改
public E set(int index, E element) {
    //检查下标是否合法
    checkElementIndex(index);
    //获取指定下标的结点引用
    Node<E> x = node(index);
    //获取指定下标结点的值
    E oldVal = x.item;
    //将结点元素设置为新的值
    x.item = element;
    //返回之前的值
    return oldVal;
}

//查
public E get(int index) {
    //检查下标是否合法
    checkElementIndex(index);
    //返回指定下标的结点的值
    return node(index).item;
}
```

LinkedList 的添加元素的方法主要是调用 linkLast 和 linkBefore 两个方法，linkLast 方法是在链表后面链接一个元素，linkBefore 方法是在链表中间插入一个元素。LinkedList 的删除方法通过调用 unlink 方法将某个元素从链表中移除。下面我们看看链表的插入和删除操作的核心代码。

```java
//链接到指定结点之前
void linkBefore(E e, Node<E> succ) {
    //获取给定结点的上一个结点引用
    final Node<E> pred = succ.prev;
    //创建新结点, 新结点的上一个结点引用指向给定结点的上一个结点
    //新结点的下一个结点的引用指向给定的结点
    final Node<E> newNode = new Node<>(pred, e, succ);
    //将给定结点的上一个结点引用指向新结点
    succ.prev = newNode;
    //如果给定结点的上一个结点为空, 表明给定结点为头结点
    if (pred == null) {
        //将头结点引用指向新结点
        first = newNode;
    } else {
        //否则, 将给定结点的上一个结点的下一个结点引用指向新结点
        pred.next = newNode;
    }
    //集合元素个数加一
    size++;
    //修改次数加一
    modCount++;
}

//卸载指定结点
E unlink(Node<E> x) {
    //获取给定结点的元素
    final E element = x.item;
    //获取给定结点的下一个结点的引用
    final Node<E> next = x.next;
    //获取给定结点的上一个结点的引用
    final Node<E> prev = x.prev;

    //如果给定结点的上一个结点为空, 说明给定结点为头结点
    if (prev == null) {
        //将头结点引用指向给定结点的下一个结点
        first = next;
    } else {
        //将上一个结点的后继结点引用指向给定结点的后继结点
        prev.next = next;
        //将给定结点的上一个结点置空
        x.prev = null;
    }

    //如果给定结点的下一个结点为空, 说明给定结点为尾结点
    if (next == null) {
        //将尾结点引用指向给定结点的上一个结点
        last = prev;
    } else {
        //将下一个结点的前继结点引用指向给定结点的前继结点
        next.prev = prev;
        x.next = null;
    }

    //将给定结点的元素置空
    x.item = null;
    //集合元素个数减一
    size--;
    //修改次数加一
    modCount++;
    return element;
}
```

linkBefore 和 unlink 是具有代表性的链接结点和卸载结点的操作，其他的链接和卸载两端结点的方法与此类似，所以我们重点介绍 linkBefore 和 unlink 方法。
linkBefore 方法的过程图：
![](https://gitee.com/liuyun1995/BlogImage/raw/master/Java%E9%9B%86%E5%90%88%E7%B3%BB%E5%88%97%5B2%5D--LinkedList%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/img2.png)

unlink 方法的过程图：
![](https://gitee.com/liuyun1995/BlogImage/raw/master/Java%E9%9B%86%E5%90%88%E7%B3%BB%E5%88%97%5B2%5D--LinkedList%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/img3.png)

通过上面图示看到对链表的插入和删除操作的时间复杂度都是 O(1)，而对链表的查找和修改操作都需要遍历链表进行元素的定位，这两个操作都是调用的 node(int index)方法定位元素，看看它是怎样通过下标来定位元素的。

```java
//根据指定位置获取结点
Node<E> node(int index) {
    //如果下标在链表前半部分, 就从头开始查起
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++) {
            x = x.next;
        }
        return x;
    } else {
        //如果下标在链表后半部分, 就从尾开始查起
        Node<E> x = last;
        for (int i = size - 1; i > index; i--) {
            x = x.prev;
        }
        return x;
    }
}
```

通过下标定位时先判断是在链表的上半部分还是下半部分，如果是在上半部分就从头开始找起，如果是下半部分就从尾开始找起，因此通过下标的查找和修改操作的时间复杂度是 O(n/2)。通过对双向链表的操作还可以实现单项队列，双向队列和栈的功能。
单向队列操作：

```java
//获取头结点
public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}

//获取头结点
public E element() {
    return getFirst();
}

//弹出头结点
public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}

//移除头结点
public E remove() {
    return removeFirst();
}

//在队列尾部添加结点
public boolean offer(E e) {
    return add(e);
}
```

双向队列操作：

```java
//在头部添加
public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}

//在尾部添加
public boolean offerLast(E e) {
    addLast(e);
    return true;
}

//获取头结点
public E peekFirst() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
 }

//获取尾结点
public E peekLast() {
    final Node<E> l = last;
    return (l == null) ? null : l.item;
}
```

栈操作：

```java
//入栈
public void push(E e) {
    addFirst(e);
}

//出栈
public E pop() {
    return removeFirst();
}
```

不管是单向队列还是双向队列还是栈，其实都是对链表的头结点和尾结点进行操作，它们的实现都是基于 addFirst()，addLast()，removeFirst()，removeLast()这四个方法，它们的操作和 linkBefore()和 unlink()类似，只不过一个是对链表两端操作，一个是对链表中间操作。可以说这四个方法都是 linkBefore()和 unlink()方法的特殊情况，因此不难理解它们的内部实现，在此不多做介绍。到这里，我们对 LinkedList 的分析也即将结束，对全文中的重点做个总结：

1. LinkedList 是基于双向链表实现的，不论是增删改查方法还是队列和栈的实现，都可通过操作结点实现
2. LinkedList 无需提前指定容量，因为基于链表操作，集合的容量随着元素的加入自动增加
3. LinkedList 删除元素后集合占用的内存自动缩小，无需像 ArrayList 一样调用 trimToSize()方法
4. LinkedList 的所有方法没有进行同步，因此它也不是线程安全的，应该避免在多线程环境下使用
