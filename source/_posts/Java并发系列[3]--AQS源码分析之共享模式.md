---
title: 'Java并发系列[3]--AQS源码分析之共享模式'
date: 2019-11-10 11:00:42
categories: Java并发
---
通过上一篇的分析，我们知道了独占模式获取锁有三种方式，分别是不响应线程中断获取，响应线程中断获取，设置超时时间获取。在共享模式下获取锁的方式也是这三种，而且基本上都是大同小异，我们搞清楚了一种就能很快的理解其他的方式。<!-- more -->虽然说AbstractQueuedSynchronizer源码有一千多行，但是重复的也比较多，所以读者不要刚开始的时候被吓到，只要耐着性子去看慢慢的自然能够渐渐领悟。就我个人经验来说，阅读AbstractQueuedSynchronizer源码有几个比较关键的地方需要弄明白，分别是独占模式和共享模式的区别，结点的等待状态，以及对条件队列的理解。理解了这些要点那么后续源码的阅读将会轻松很多。当然这些在我的《Java并发系列[1]----AQS源码分析之概要分析》这篇文章里都有详细的介绍，读者可以先去查阅。本篇对于共享模式的分析也是分为三种获取锁的方式和一种释放锁的方式。

1.不响应线程中断的获取
```java
//以不可中断模式获取锁(共享模式)
public final void acquireShared(int arg) {
    //1.尝试去获取锁
    if (tryAcquireShared(arg) < 0) {
        //2.如果获取失败就进入这个方法
        doAcquireShared(arg);
    }
}

//尝试去获取锁(共享模式)
//负数：表示获取失败
//零值：表示当前结点获取成功, 但是后继结点不能再获取了
//正数：表示当前结点获取成功, 并且后继结点同样可以获取成功
protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}
```
调用acquireShared方法是不响应线程中断获取锁的方式。在该方法中，首先调用tryAcquireShared去尝试获取锁，tryAcquireShared方法返回一个获取锁的状态，这里AQS规定了返回状态若是负数代表当前结点获取锁失败，若是0代表当前结点获取锁成功，但后继结点不能再获取了，若是正数则代表当前结点获取锁成功，并且这个锁后续结点也同样可以获取成功。子类在实现tryAcquireShared方法获取锁的逻辑时，返回值需要遵守这个约定。如果调用tryAcquireShared的返回值小于0，就代表这次尝试获取锁失败了，接下来就调用doAcquireShared方法将当前线程添加进同步队列。我们看到doAcquireShared方法。
```java
//在同步队列中获取(共享模式)
private void doAcquireShared(int arg) {
    //添加到同步队列中
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //获取当前结点的前继结点
            final Node p = node.predecessor();
            //如果前继结点为head结点就再次尝试去获取锁
            if (p == head) {
                //再次尝试去获取锁并返回获取状态
                //r < 0, 表示获取失败
                //r = 0, 表示当前结点获取成功, 但是后继结点不能再获取了
                //r > 0, 表示当前结点获取成功, 并且后继结点同样可以获取成功
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    //到这里说明当前结点已经获取锁成功了, 此时它会将锁的状态信息传播给后继结点
                    setHeadAndPropagate(node, r);
                    p.next = null;
                    //如果在线程阻塞期间收到中断请求, 就在这一步响应该请求
                    if (interrupted) {
                        selfInterrupt();
                    }
                    failed = false;
                    return;
                }
            }
            //每次获取锁失败后都会判断是否可以将线程挂起, 如果可以的话就会在parkAndCheckInterrupt方法里将线程挂起
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt()) {
                interrupted = true;
            }
        }
    } finally {
        if (failed) {
            cancelAcquire(node);
        }
    }
}
```
进入doAcquireShared方法首先是调用addWaiter方法将当前线程包装成结点放到同步队列尾部。这个添加结点的过程我们在讲独占模式时讲过，这里就不再讲了。结点进入同步队列后，如果它发现在它前面的结点就是head结点，因为head结点的线程已经获取锁进入房间里面了，那么下一个获取锁的结点就轮到自己了，所以当前结点先不会将自己挂起，而是再一次去尝试获取锁，如果前面那人刚好释放锁离开了，那么当前结点就能成功获得锁，如果前面那人还没有释放锁，那么就会调用shouldParkAfterFailedAcquire方法，在这个方法里面会将head结点的状态改为SIGNAL，只有保证前面结点的状态为SIGNAL，当前结点才能放心的将自己挂起，所有线程都会在parkAndCheckInterrupt方法里面被挂起。如果当前结点恰巧成功的获取了锁，那么接下来就会调用setHeadAndPropagate方法将自己设置为head结点，并且唤醒后面同样是共享模式的结点。下面我们看下setHeadAndPropagate方法具体的操作。
```java
//设置head结点并传播锁的状态(共享模式)
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head;
    //将给定结点设置为head结点
    setHead(node);
    //如果propagate大于0表明锁可以获取了
    if (propagate > 0 || h == null || h.waitStatus < 0) {
        //获取给定结点的后继结点
        Node s = node.next;
        //如果给定结点的后继结点为空, 或者它的状态是共享状态
        if (s == null || s.isShared()) {
            //唤醒后继结点
            doReleaseShared();
        }
    }
}

//释放锁的操作(共享模式)
private void doReleaseShared() {
    for (;;) {
        //获取同步队列的head结点
        Node h = head;
        if (h != null && h != tail) {
            //获取head结点的等待状态
            int ws = h.waitStatus;
            //如果head结点的状态为SIGNAL, 表明后面有人在排队
            if (ws == Node.SIGNAL) {
                //先把head结点的等待状态更新为0
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) {
                    continue;
                }
                //再去唤醒后继结点
                unparkSuccessor(h);
             //如果head结点的状态为0, 表明此时后面没人在排队, 就只是将head状态修改为PROPAGATE
            }else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE)) {
                continue;
            }
        }
        //只有保证期间head结点没被修改过才能跳出循环
        if (h == head) {
            break;
        }
    }
}
```
调用setHeadAndPropagate方法首先将自己设置成head结点，然后再根据传入的tryAcquireShared方法的返回值来决定是否要去唤醒后继结点。前面已经讲到当返回值大于0就表明当前结点成功获取了锁，并且后面的结点也可以成功获取锁。这时当前结点就需要去唤醒后面同样是共享模式的结点，注意，每次唤醒仅仅只是唤醒后一个结点，如果后一个结点不是共享模式的话，当前结点就直接进入房间而不会再去唤醒更后面的结点了。共享模式下唤醒后继结点的操作是在doReleaseShared方法进行的，共享模式和独占模式的唤醒操作基本也是相同的，都是去找到自己座位上的牌子(等待状态)，如果牌子上为SIGNAL表明后面有人需要让它帮忙唤醒，如果牌子上为0则表明队列此时并没有人在排队。在独占模式下是如果发现没人在排队就直接离开队列了，而在共享模式下如果发现队列后面没人在排队，当前结点在离开前仍然会留个小纸条(将等待状态设置为PROPAGATE)告诉后来的人这个锁的可获取状态。那么后面来的人在尝试获取锁的时候可以根据这个状态来判断是否直接获取锁。

2.响应线程中断的获取
```java
//以可中断模式获取锁(共享模式)
public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
    //首先判断线程是否中断, 如果是则抛出异常
    if (Thread.interrupted()) {
        throw new InterruptedException();
    }
    //1.尝试去获取锁
    if (tryAcquireShared(arg) < 0) {
        //2. 如果获取失败则进人该方法
        doAcquireSharedInterruptibly(arg);
    }
}

//以可中断模式获取(共享模式)
private void doAcquireSharedInterruptibly(int arg) throws InterruptedException {
    //将当前结点插入同步队列尾部
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            //获取当前结点的前继结点
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null;
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt()) {
                //如果线程在阻塞过程中收到过中断请求, 那么就会立马在这里抛出异常
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
响应线程中断获取锁的方式和不响应线程中断获取锁的方式在流程上基本是相同的，唯一的区别就是在哪里响应线程的中断请求。在不响应线程中断获取锁时，线程从parkAndCheckInterrupt方法中被唤醒，唤醒后就立马返回是否收到中断请求，即使是收到了中断请求也会继续自旋直到获取锁后才响应中断请求将自己给挂起。而响应线程中断获取锁会才线程被唤醒后立马响应中断请求，如果在阻塞过程中收到了线程中断就会立马抛出InterruptedException异常。

3.设置超时时间的获取
```java
//以限定超时时间获取锁(共享模式)
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout) throws InterruptedException {
    if (Thread.interrupted()) {
        throw new InterruptedException();
    }
    //1.调用tryAcquireShared尝试去获取锁
    //2.如果获取失败就调用doAcquireSharedNanos
    return tryAcquireShared(arg) >= 0 || doAcquireSharedNanos(arg, nanosTimeout);
}

//以限定超时时间获取锁(共享模式)
private boolean doAcquireSharedNanos(int arg, long nanosTimeout) throws InterruptedException {
    long lastTime = System.nanoTime();
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            //获取当前结点的前继结点
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null;
                    failed = false;
                    return true;
                }
            }
            //如果超时时间用完了就结束获取, 并返回失败信息
            if (nanosTimeout <= 0) {
                return false;
            }
            //1.检查是否满足将线程挂起要求(保证前继结点状态为SIGNAL)
            //2.检查超时时间是否大于自旋时间
            if (shouldParkAfterFailedAcquire(p, node) && nanosTimeout > spinForTimeoutThreshold) {
                //若满足上面两个条件就将当前线程挂起一段时间
                LockSupport.parkNanos(this, nanosTimeout);
            }
            long now = System.nanoTime();
            //超时时间每次减去获取锁的时间
            nanosTimeout -= now - lastTime;
            lastTime = now;
            //如果在阻塞时收到中断请求就立马抛出异常
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
如果看懂了上面两种获取方式，再来看设置超时时间的获取方式就会很轻松，基本流程都是一样的，主要是理解超时的机制是怎样的。如果第一次获取锁失败会调用doAcquireSharedNanos方法并传入超时时间，进入方法后会根据情况再次去获取锁，如果再次获取失败就要考虑将线程挂起了。这时会判断超时时间是否大于自旋时间，如果是的话就会将线程挂起一段时间，否则就继续尝试获取，每次获取锁之后都会将超时时间减去获取锁的时间，一直这样循环直到超时时间用尽，如果还没有获取到锁的话就会结束获取并返回获取失败标识。在整个期间线程是响应线程中断的。

4.共享模式下结点的出队操作
```java
//释放锁的操作(共享模式)
public final boolean releaseShared(int arg) {
    //1.尝试去释放锁
    if (tryReleaseShared(arg)) {
        //2.如果释放成功就唤醒其他线程
        doReleaseShared();
        return true;
    }
    return false;
}

//尝试去释放锁(共享模式)
protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}

//释放锁的操作(共享模式)
private void doReleaseShared() {
    for (;;) {
        //获取同步队列的head结点
        Node h = head;
        if (h != null && h != tail) {
            //获取head结点的等待状态
            int ws = h.waitStatus;
            //如果head结点的状态为SIGNAL, 表明后面有人在排队
            if (ws == Node.SIGNAL) {
                //先把head结点的等待状态更新为0
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) {
                    continue;
                }
                //再去唤醒后继结点
                unparkSuccessor(h);
             //如果head结点的状态为0, 表明此时后面没人在排队, 就只是将head状态修改为PROPAGATE
            }else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE)) {
                continue;
            }
        }
        //只有保证期间head结点没被修改过才能跳出循环
        if (h == head) {
            break;
        }
    }
}
```
线程在房间办完事之后就会调用releaseShared方法释放锁，首先调用tryReleaseShared方法尝试释放锁，该方法的判断逻辑由子类实现。如果释放成功就调用doReleaseShared方法去唤醒后继结点。走出房间后它会找到原先的座位(head结点)，看看座位上是否有人留了小纸条(状态为SIGNAL)，如果有就去唤醒后继结点。如果没有(状态为0)就代表队列没人在排队，那么在离开之前它还要做最后一件事情，就是在自己座位上留下小纸条(状态设置为PROPAGATE)，告诉后面的人锁的获取状态，整个释放锁的过程和独占模式唯一的区别就是在这最后一步操作。
