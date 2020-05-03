---
title: 'Java并发系列[1]--AQS源码分析之概要分析'
date: 2019-11-10 10:46:40
categories: Java并发
---
学习Java并发编程不得不去了解一下java.util.concurrent这个包，这个包下面有许多我们经常用到的并发工具类，例如：ReentrantLock, CountDownLatch, CyclicBarrier, Semaphore等。而这些类的底层实现都依赖于AbstractQueuedSynchronizer这个类，由此可见这个类的重要性。<!-- more -->所以在Java并发系列文章中我首先对AbstractQueuedSynchronizer这个类进行分析，由于这个类比较重要，而且代码比较长，为了尽可能分析的透彻一些，我决定用四篇文章对该类进行一个比较完整的介绍。本篇文章作为概要介绍主要是让读者们对该类有个初步了解。为了叙述简单，后续有些地方会用AQS代表这个类。

1.AbstractQueuedSynchronizer这个类是干嘛的？
相信要许多读者使用过ReentrantLock，但是却不知道AbstractQueuedSynchronizer的存在。其实ReentrantLock实现了一个内部类Sync，该内部类继承了AbstractQueuedSynchronizer，所有锁机制的实现都是依赖于Sync内部类，也可以说ReentrantLock的实现就是依赖于AbstractQueuedSynchronizer类。于此类似，CountDownLatch, CyclicBarrier, Semaphore这些类也是采用同样的方式来实现自己对于锁的控制。可见，AbstractQueuedSynchronizer是这些类的基石。那么AQS内部到底实现了什么以至于所以这些类都要依赖于它呢？可以这样说，AQS为这些类提供了基础设施，也就是提供了一个密码锁，这些类拥有了密码锁之后可以自己来设置密码锁的密码。此外，AQS还提供了一个排队区，并且提供了一个线程训导员，我们知道线程就像一个原始的野蛮人，它不懂得讲礼貌，它只会横冲直撞，所以你得一步一步去教它，告诉它什么时候需要去排队了，要到哪里去排队，排队前要做些什么，排队后要做些什么。这些教化工作全部都由AQS帮你完成了，从它这里教化出来的线程都变的非常文明懂礼貌，不再是原始的野蛮人，所以以后我们只需要和这些文明的线程打交道就行了，千万不要和原始线程有过多的接触！

2.为何说AbstractQueuedSynchronizer提供了一把密码锁？
```java
//同步队列的头结点
private transient volatile Node head; 

//同步队列的尾结点
private transient volatile Node tail;

//同步状态
private volatile int state;

//获取同步状态
protected final int getState() {
    return state;
}

//设置同步状态
protected final void setState(int newState) {
    state = newState;
}

//以CAS方式设置同步状态
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```
上面的代码列出了AQS的所有成员变量，可以看到AQS的成员变量只有三个，分别是同步队列头结点引用，同步队列尾结点引用以及同步状态。注意，这三个成员变量都使用了volatile关键字进行修饰，这就确保了多个线程对它的修改都是内存可见的。整个类的核心就是这个同步状态，可以看到同步状态其实就是一个int型的变量，大家可以把这个同步状态看成一个密码锁，而且还是从房间里面锁起来的密码锁，state具体的值就相当于密码控制着密码锁的开合。当然这个锁的密码是多少就由各个子类来规定了，例如在ReentrantLock中，state等于0表示锁是开的，state大于0表示锁是锁着的，而在Semaphore中，state大于0表示锁是开的，state等于0表示锁是锁着的。

3.AbstractQueuedSynchronizer的排队区是怎样实现的？
![](img1.png)

AbstractQueuedSynchronizer内部其实有两个排队区，一个是同步队列，一个是条件队列。从上图可以看出，同步队列只有一条，而条件队列可以有多条。同步队列的结点分别持有前后结点的引用，而条件队列的结点只有一个指向后继结点的引用。图中T表示线程，每个结点包含一个线程，线程在获取锁失败后首先进入同步队列排队，而想要进入条件队列该线程必须持有锁才行。接下来我们看看队列中每个结点的结构。
```java
//同步队列的结点
static final class Node {
    
    static final Node SHARED = new Node(); //表示当前线程以共享模式持有锁
    
    static final Node EXCLUSIVE = null;    //表示当前线程以独占模式持有锁

    static final int CANCELLED =  1;       //表示当前结点已经取消获取锁
    
    static final int SIGNAL    = -1;       //表示后继结点的线程需要运行
    
    static final int CONDITION = -2;       //表示当前结点在条件队列中排队
    
    static final int PROPAGATE = -3;       //表示后继结点可以直接获取锁

    volatile int waitStatus; //表示当前结点的等待状态
   
    volatile Node prev;      //表示同步队列中的前继结点

    volatile Node next;      //表示同步队列中的后继结点  

    volatile Thread thread;  //当前结点持有的线程引用
    
    Node nextWaiter;         //表示条件队列中的后继结点

    //当前结点状态是否是共享模式
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    //返回当前结点的前继结点
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null) {
            throw new NullPointerException();
        } else {
            return p;
        }
    }
    
    //构造器1
    Node() {}
    
    //构造器2, 默认用这个构造器
    Node(Thread thread, Node mode) {
        //注意持有模式是赋值给nextWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }
    
    //构造器3, 只在条件队列中用到
    Node(Thread thread, int waitStatus) {
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```
Node代表同步队列和条件队列中的一个结点，它是AbstractQueuedSynchronizer的内部类。Node有很多属性，比如持有模式，等待状态，同步队列中的前继和后继，以及条件队列中的后继引用等等。可以把同步队列和条件队列看成是排队区，每个结点看成是排队区的座位，将线程看成是排队的客人。客人刚来时会先去敲敲门，看看锁有没有开，如果锁没开它就会去排队区领取一个号码牌，声明自己想要以什么样的方式来持有锁，最后再到队列的末尾进行排队。

4.怎样理解独占模式和共享模式？
前面讲到每个客人在排队前会领取一个号码牌，声明自己想要以什么样的方式来占有锁，占有锁的方式分为独占模式和共享模式，那么怎样来理解独占模式和共享模式呢？实在找不到什么好的比喻，大家可以联想一下公共厕所，独占模式的人比较霸道，老子要么就不进，进来了就不许别人再进了，自己一个人独自占用整个厕所。共享模式的人就没那么讲究了，当它发现这个厕所已经可以用了之后，它自己进来还不算，还得热心的问下后面的人介不介意一起用，如果后面的人不介意一起使用那就不用再排队了大家一起上就是了， 当然如果后面的人介意那就只好留在队列里继续排队了。

5.怎样理解结点的等待状态？
我们还看到每个结点都有一个等待状态，这个等待状态分为CANCELLED，SIGNAL，CONDITION，PROPAGATE四种状态。可以将这个等待状态看作是挂在座位旁边的牌子，标识当前座位上的人的等待状态。这个牌子的状态不仅自己可以修改，其他人也可以修改。例如当这个线程在排队过程中已经打算放弃了，它就会将自己座位上的牌子设置为CANCELLED，这样其他人看到了就可以将它清理出队列。还有一种情况是，当线程在座位上要睡着之前，它怕自己睡过了头，就会将前面位置上的牌子改为SIGNAL，因为每个人在离开队列前都会回到自己座位上看一眼，如果看到牌子上状态为SIGNAL，它就会去唤醒下一个人。只有保证前面位置上的牌子为SIGNAL，当前线程才会安心的睡去。CONDITION状态表示该线程在条件队列中排队，PROPAGATE状态提醒后面来的线程可以直接获取锁，这个状态只在共享模式用到，后面单独讲共享模式的时候会讲到。

6.结点进入同步队列时会进行哪些操作？
```java
//结点入队操作, 返回前一个结点
private Node enq(final Node node) {
    for (;;) {
        //获取同步队列尾结点引用
        Node t = tail;
        //如果尾结点为空说明同步队列还没有初始化
        if (t == null) {
            //初始化同步队列
            if (compareAndSetHead(new Node())) {
                tail = head;
            }
        } else {
            //1.指向当前尾结点
            node.prev = t;
            //2.设置当前结点为尾结点
            if (compareAndSetTail(t, node)) {
                //3.将旧的尾结点的后继指向新的尾结点
                t.next = node;
                //for循环唯一的出口
                return t;
            }
        }
    }
}
```
注意，入队操作使用一个死循环，只有成功将结点添加到同步队列尾部才会返回，返回结果是同步队列原先的尾结点。下图演示了整个操作过程。
![](img2.png)

读者需要注意添加尾结点的顺序，分为三步：指向尾结点，CAS更改尾结点，将旧尾结点的后继指向当前结点。在并发环境中这三步操作不一定能保证完成，所以在清空同步队列所有已取消的结点这一操作中，为了寻找非取消状态的结点，不是从前向后遍历而是从后向前遍历的。还有就是每个结点进入队列中时它的等待状态是为0，只有后继结点的线程需要挂起时才会将前面结点的等待状态改为SIGNAL。
