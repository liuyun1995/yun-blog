---
title: 'Java 并发系列[2]--AQS 源码分析之独占模式'
date: 2018-03-17 10:53:27
categories: Java并发
---
在上一篇《Java 并发系列[1]----AQS 源码分析之概要分析》中我们介绍了 AbstractQueuedSynchronizer 基本的一些概念，主要讲了 AQS 的排队区是怎样实现的，什么是独占模式和共享模式以及如何理解结点的等待状态。理解并掌握这些内容是后续阅读 AQS 源码的关键，所以建议读者先看完我的上一篇文章再回过头来看这篇就比较容易理解。<!-- more -->在本篇中会介绍在独占模式下结点是怎样进入同步队列排队的，以及离开同步队列之前会进行哪些操作。AQS 为在独占模式和共享模式下获取锁分别提供三种获取方式：不响应线程中断获取，响应线程中断获取，设置超时时间获取。这三种方式整体步骤大致是相同的，只有少部分不同的地方，所以理解了一种方式再看其他方式的实现都是大同小异。在本篇中我会着重讲不响应线程中断的获取方式，其他两种方式也会顺带讲一下不一致的地方。

1.怎样以不响应线程中断获取锁？

```java
//不响应中断方式获取(独占模式)
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) {
        selfInterrupt();
    }
}
```

上面代码中虽然看起来简单，但是它按照顺序执行了下图所示的 4 个步骤。下面我们会逐个步骤进行演示分析。
![](https://gitee.com/liuyun1995/yun-blog-image/raw/master/Java%E5%B9%B6%E5%8F%91%E7%B3%BB%E5%88%97%5B2%5D--AQS%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8B%E7%8B%AC%E5%8D%A0%E6%A8%A1%E5%BC%8F/img1.png)

第一步：!tryAcquire(arg)

```java
//尝试去获取锁(独占模式)
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

这时候来了一个人，他首先尝试着去敲了敲门，如果发现门没锁(tryAcquire(arg)=true)，那就直接进去了。如果发现门锁了(tryAcquire(arg)=false)，就执行下一步。这个 tryAcquire 方法决定了什么时候锁是开着的，什么时候锁是关闭的。这个方法必须要让子类去覆盖，重写里面的判断逻辑。

第二步：addWaiter(Node.EXCLUSIVE)

```java
//将当前线程包装成结点并添加到同步队列尾部
private Node addWaiter(Node mode) {
    //指定持有锁的模式
    Node node = new Node(Thread.currentThread(), mode);
    //获取同步队列尾结点引用
    Node pred = tail;
    //如果尾结点不为空, 表明同步队列已存在结点
    if (pred != null) {
        //1.指向当前尾结点
        node.prev = pred;
        //2.设置当前结点为尾结点
        if (compareAndSetTail(pred, node)) {
            //3.将旧的尾结点的后继指向新的尾结点
            pred.next = node;
            return node;
        }
    }
    //否则表明同步队列还没有进行初始化
    enq(node);
    return node;
}

//结点入队操作
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
                return t;
            }
        }
    }
}
```

执行到这一步表明第一次获取锁失败，那么这个人就给自己领了块号码牌进入排队区去排队了，在领号码牌的时候会声明自己想要以什么样的方式来占用房间(独占模式 or 共享模式)。注意，这时候他并没有坐下来休息(将自己挂起)哦。

第三步：acquireQueued(addWaiter(Node.EXCLUSIVE), arg)

```java
//以不可中断方式获取锁(独占模式)
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //获取给定结点的前继结点的引用
            final Node p = node.predecessor();
            //如果当前结点是同步队列的第一个结点, 就尝试去获取锁
            if (p == head && tryAcquire(arg)) {
                //将给定结点设置为head结点
                setHead(node);
                //为了帮助垃圾收集, 将上一个head结点的后继清空
                p.next = null;
                //设置获取成功状态
                failed = false;
                //返回中断的状态, 整个循环执行到这里才是出口
                return interrupted;
            }
            //否则说明锁的状态还是不可获取, 这时判断是否可以挂起当前线程
            //如果判断结果为真则挂起当前线程, 否则继续循环, 在这期间线程不响应中断
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt()) {
                interrupted = true;
            }
        }
    } finally {
        //在最后确保如果获取失败就取消获取
        if (failed) {
            cancelAcquire(node);
        }
    }
}

//判断是否可以将当前结点挂起
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    //获取前继结点的等待状态
    int ws = pred.waitStatus;
    //如果前继结点状态为SIGNAL, 表明前继结点会唤醒当前结点, 所以当前结点可以安心的挂起了
    if (ws == Node.SIGNAL) {
        return true;
    }
  
    if (ws > 0) {
        //下面的操作是清理同步队列中所有已取消的前继结点
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        //到这里表示前继结点状态不是SIGNAL, 很可能还是等于0, 这样的话前继结点就不会去唤醒当前结点了
        //所以当前结点必须要确保前继结点的状态为SIGNAL才能安心的挂起自己
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

//挂起当前线程
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

领完号码牌进入排队区后就会立马执行这个方法，当一个结点首次进入排队区后有两种情况，一种是发现他前面的那个人已经离开座位进入房间了，那他就不坐下来休息了，会再次去敲一敲门看看那小子有没有完事。如果里面的人刚好完事出来了，都不用他叫自己就直接冲进去了。否则，就要考虑坐下来休息一会儿了，但是他还是不放心，如果他坐下来睡着后没人提醒他怎么办？他就在前面那人的座位上留一个小纸条，好让从里面出来的人看到纸条后能够唤醒他。还有一种情况是，当他进入排队区后发现前面还有好几个人在座位上排队呢，那他就可以安心的坐下来咪一会儿了，但在此之前他还是会在前面那人(此时已经睡着了)的座位上留一个纸条，好让这个人在走之前能够去唤醒自己。当一切事情办妥了之后，他就安安心心的睡觉了，注意，我们看到整个 for 循环就只有一个出口，那就是等线程成功的获取到锁之后才能出去，在没有获取到锁之前就一直是挂在 for 循环的 parkAndCheckInterrupt()方法里头。线程被唤醒后也是从这个地方继续执行 for 循环。

第四步：selfInterrupt()

```java
//当前线程将自己中断
private static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```

由于上面整个线程一直是挂在 for 循环的 parkAndCheckInterrupt()方法里头，没有成功获取到锁之前不响应任何形式的线程中断，只有当线程成功获取到锁并从 for 循环出来后，他才会查看在这期间是否有人要求中断线程，如果是的话再去调用 selfInterrupt()方法将自己挂起。

2.怎样以响应线程中断获取锁？

```java
//以可中断模式获取锁(独占模式)
private void doAcquireInterruptibly(int arg) throws InterruptedException {
    //将当前线程包装成结点添加到同步队列中
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            //获取当前结点的前继结点
            final Node p = node.predecessor();
            //如果p是head结点, 那么当前线程就再次尝试获取锁
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                //获取锁成功后返回
                return;
            }
            //如果满足条件就挂起当前线程, 此时响应中断并抛出异常
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt()) {
                //线程被唤醒后如果发现中断请求就抛出异常
                throw new InterruptedException();
            }
        }
    } finally {
        if (failed) {
            cancelAcquire(node);
        }
    }
}
```

响应线程中断方式和不响应线程中断方式获取锁流程上大致上是相同的。唯一的一点区别就是线程从 parkAndCheckInterrupt 方法中醒来后会检查线程是否中断，如果是的话就抛出 InterruptedException 异常，而不响应线程中断获取锁是在收到中断请求后只是设置一下中断状态，并不会立马结束当前获取锁的方法，一直到结点成功获取到锁之后才会根据中断状态决定是否将自己挂起。

3.怎样设置超时时间获取锁？

```java
//以限定超时时间获取锁(独占模式)
private boolean doAcquireNanos(int arg, long nanosTimeout) throws InterruptedException {
    //获取系统当前时间
    long lastTime = System.nanoTime();
    //将当前线程包装成结点添加到同步队列中
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            //获取当前结点的前继结点
            final Node p = node.predecessor();
            //如果前继是head结点, 那么当前线程就再次尝试获取锁
            if (p == head && tryAcquire(arg)) {
                //更新head结点
                setHead(node);
                p.next = null;
                failed = false;
                return true;
            }
            //超时时间用完了就直接退出循环
            if (nanosTimeout <= 0) {
                return false;
            }
            //如果超时时间大于自旋时间, 那么等判断可以挂起线程之后就会将线程挂起一段时间
            if (shouldParkAfterFailedAcquire(p, node) && nanosTimeout > spinForTimeoutThreshold) {
                //将当前线程挂起一段时间, 之后再自己醒来
                LockSupport.parkNanos(this, nanosTimeout);
            }
            //获取系统当前时间
            long now = System.nanoTime();
            //超时时间每次都减去获取锁的时间间隔
            nanosTimeout -= now - lastTime;
            //再次更新lastTime
            lastTime = now;
            //在获取锁的期间收到中断请求就抛出异常
            if (Thread.interrupted()) {
                throw new InterruptedException();
            }
        }
    } finally {
        if (failed) {
            cancelAcquire(node);
        }
    }
}
```

设置超时时间获取首先会去获取一下锁，第一次获取锁失败后会根据情况，如果传入的超时时间大于自旋时间那么就会将线程挂起一段时间，否则的话就会进行自旋，每次获取锁之后都会将超时时间减去获取一次锁所用的时间。一直到超时时间小于 0 也就说明超时时间用完了，那么这时就会结束获取锁的操作然后返回获取失败标志。注意在以超时时间获取锁的过程中是可以响应线程中断请求的。

4.线程释放锁并离开同步队列是怎样进行的？

```java
//释放锁的操作(独占模式)
public final boolean release(int arg) {
    //拨动密码锁, 看看是否能够开锁
    if (tryRelease(arg)) {
        //获取head结点
        Node h = head;
        //如果head结点不为空并且等待状态不等于0就去唤醒后继结点
        if (h != null && h.waitStatus != 0) {
            //唤醒后继结点
            unparkSuccessor(h);
        }
        return true;
    }
    return false;
}

//唤醒后继结点
private void unparkSuccessor(Node node) {
    //获取给定结点的等待状态
    int ws = node.waitStatus;
    //将等待状态更新为0
    if (ws < 0) {
        compareAndSetWaitStatus(node, ws, 0);
    }
    //获取给定结点的后继结点
    Node s = node.next;
    //后继结点为空或者等待状态为取消状态
    if (s == null || s.waitStatus > 0) {
        s = null;
        //从后向前遍历队列找到第一个不是取消状态的结点
        for (Node t = tail; t != null && t != node; t = t.prev) {
            if (t.waitStatus <= 0) {
                s = t;
            }
        }
    }
    //唤醒给定结点后面首个不是取消状态的结点
    if (s != null) {
        LockSupport.unpark(s.thread);
    }
}
```

线程持有锁进入房间后就会去办自己的事情，等事情办完后它就会释放锁并离开房间。通过 tryRelease 方法可以拨动密码锁进行解锁，我们知道 tryRelease 方法是需要让子类去覆盖的，不同的子类实现的规则不一样，也就是说不同的子类设置的密码不一样。像在 ReentrantLock 当中，房间里面的人每调用 tryRelease 方法一次，state 就减 1，直到 state 减到 0 的时候密码锁就开了。大家想想这个过程像不像我们在不停的转动密码锁的转轮，而每次转动转轮数字只是减少 1。CountDownLatch 和这个也有点类似，只不过它不是一个人在转，而是多个人每人都去转一下，集中大家的力量把锁给开了。线程出了房间后它会找到自己原先的座位，也就是找到 head 结点。看看座位上有没有人给它留了小纸条，如果有的话它就知道有人睡着了需要让它帮忙唤醒，那么它就会去唤醒那个线程。如果没有的话就表明同步队列中暂时还没有人在等待，也没有人需要它唤醒，所以它就可以安心的离去了。以上过程就是在独占模式下释放锁的过程。
