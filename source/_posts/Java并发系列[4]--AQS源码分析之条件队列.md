---
title: 'Java并发系列[4]--AQS源码分析之条件队列'
date: 2019-11-10 11:05:11
categories: Java并发
---
通过前面三篇的分析，我们深入了解了AbstractQueuedSynchronizer的内部结构和一些设计理念，知道了AbstractQueuedSynchronizer内部维护了一个同步状态和两个排队区，这两个排队区分别是同步队列和条件队列。<!-- more -->我们还是拿公共厕所做比喻，同步队列是主要的排队区，如果公共厕所没开放，所有想要进入厕所的人都得在这里排队。而条件队列主要是为条件等待设置的，我们想象一下如果一个人通过排队终于成功获取锁进入了厕所，但在方便之前发现自己没带手纸，碰到这种情况虽然很无奈，但是它也必须接受这个事实，这时它只好乖乖的出去先准备好手纸(进入条件队列等待)，当然在出去之前还得把锁给释放了好让其他人能够进来，在准备好了手纸(条件满足)之后它又得重新回到同步队列中去排队。当然进入房间的人并不都是因为没带手纸，可能还有其他一些原因必须中断操作先去条件队列中去排队，所以条件队列可以有多个，依不同的等待条件而设置不同的条件队列。条件队列是一条单向链表，Condition接口定义了条件队列中的所有操作，AbstractQueuedSynchronizer内部的ConditionObject类实现了Condition接口，下面我们看看Condition接口都定义了哪些操作。
```java
public interface Condition {
    
    //响应线程中断的条件等待
    void await() throws InterruptedException;
    
    //不响应线程中断的条件等待
    void awaitUninterruptibly();
    
    //设置相对时间的条件等待(不进行自旋)
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    
    //设置相对时间的条件等待(进行自旋)
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    
    //设置绝对时间的条件等待
    boolean awaitUntil(Date deadline) throws InterruptedException;
    
    //唤醒条件队列中的头结点
    void signal();
    
    //唤醒条件队列的所有结点
    void signalAll();
    
}
```
Condition接口虽然定义了这么多方法，但总共就分为两类，以await开头的是线程进入条件队列等待的方法，以signal开头的是将条件队列中的线程“唤醒”的方法。这里要注意的是，调用signal方法可能唤醒线程也可能不会唤醒线程，什么时候会唤醒线程这得看情况，后面会讲到，但是调用signal方法一定会将线程从条件队列中移到同步队列尾部。这里为了叙述方便，我们先暂时不纠结这么多，统一称signal方法为唤醒条件队列线程的操作。大家注意看一下，await方法分为5种，分别是响应线程中断等待，不响应线程中断等待，设置相对时间不自旋等待，设置相对时间自旋等待，设置绝对时间等待；signal方法只有2种，分别是只唤醒条件队列头结点和唤醒条件队列所有结点的操作。同一类的方法基本上是相通的，由于篇幅所限，我们不可能也不需要将这些方法全部仔细的讲到，只需要将一个代表方法搞懂了再看其他方法就能够触类旁通。所以在本文中我只会细讲await方法和signal方法，其他方法不细讲但会贴出源码来以供大家参考。

1.响应线程中断的条件等待
```java
//响应线程中断的条件等待
public final void await() throws InterruptedException {
    //如果线程被中断则抛出异常
    if (Thread.interrupted()) {
        throw new InterruptedException();
    }
    //将当前线程添加到条件队列尾部
    Node node = addConditionWaiter();
    //在进入条件等待之前先完全释放锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    //线程一直在while循环里进行条件等待
    while (!isOnSyncQueue(node)) {
        //进行条件等待的线程都在这里被挂起, 线程被唤醒的情况有以下几种：
        //1.同步队列的前继结点已取消
        //2.设置同步队列的前继结点的状态为SIGNAL失败
        //3.前继结点释放锁后唤醒当前结点
        LockSupport.park(this);
        //当前线程醒来后立马检查是否被中断, 如果是则代表结点取消条件等待, 此时需要将结点移出条件队列
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) {
            break;
        }
    }
    //线程醒来后就会以独占模式获取锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE) {
        interruptMode = REINTERRUPT;
    }
    //这步操作主要为防止线程在signal之前中断而导致没与条件队列断绝联系
    if (node.nextWaiter != null) {
        unlinkCancelledWaiters();
    }
    //根据中断模式进行响应的中断处理
    if (interruptMode != 0) {
        reportInterruptAfterWait(interruptMode);
    }
}
```
当线程调用await方法的时候，首先会将当前线程包装成node结点放入条件队列尾部。在addConditionWaiter方法中，如果发现条件队列尾结点已取消就会调用unlinkCancelledWaiters方法将条件队列所有的已取消结点清空。这步操作是插入结点的准备工作，那么确保了尾结点的状态也是CONDITION之后，就会新建一个node结点将当前线程包装起来然后放入条件队列尾部。注意，这个过程只是将结点添加到同步队列尾部而没有挂起线程哦。
第二步：完全将锁释放
```java
//完全释放锁
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        //获取当前的同步状态
        int savedState = getState();
        //使用当前的同步状态去释放锁
        if (release(savedState)) {
            failed = false;
            //如果释放锁成功就返回当前同步状态
            return savedState;
        } else {
            //如果释放锁失败就抛出运行时异常
            throw new IllegalMonitorStateException();
        }
    } finally {
        //保证没有成功释放锁就将该结点设置为取消状态
        if (failed) {
            node.waitStatus = Node.CANCELLED;
        }
    }
}
```
将当前线程包装成结点添加到条件队列尾部后，紧接着就调用fullyRelease方法释放锁。注意，方法名为fullyRelease也就这步操作会完全的释放锁，因为锁是可重入的，所以在进行条件等待前需要将锁全部释放了，不然的话别人就获取不了锁了。如果释放锁失败的话就会抛出一个运行时异常，如果成功释放了锁的话就返回之前的同步状态。

第三步：进行条件等待
```java
//线程一直在while循环里进行条件等待
while (!isOnSyncQueue(node)) {
    //进行条件等待的线程都在这里被挂起, 线程被唤醒的情况有以下几种：
    //1.同步队列的前继结点已取消
    //2.设置同步队列的前继结点的状态为SIGNAL失败
    //3.前继结点释放锁后唤醒当前结点
    LockSupport.park(this);
    //当前线程醒来后立马检查是否被中断, 如果是则代表结点取消条件等待, 此时需要将结点移出条件队列
    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) {
        break;
    }
}

//检查条件等待时的线程中断情况
private int checkInterruptWhileWaiting(Node node) {
    //中断请求在signal操作之前：THROW_IE
    //中断请求在signal操作之后：REINTERRUPT
    //期间没有收到任何中断请求：0
    return Thread.interrupted() ? (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) : 0;
}

//将取消条件等待的结点从条件队列转移到同步队列中
final boolean transferAfterCancelledWait(Node node) {
    //如果这步CAS操作成功的话就表明中断发生在signal方法之前
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        //状态修改成功后就将该结点放入同步队列尾部
        enq(node);
        return true;
    }
    //到这里表明CAS操作失败, 说明中断发生在signal方法之后
    while (!isOnSyncQueue(node)) {
        //如果sinal方法还没有将结点转移到同步队列, 就通过自旋等待一下
        Thread.yield();
    }
    return false;
}
```
在以上两个操作完成了之后就会进入while循环，可以看到while循环里面首先调用LockSupport.park(this)将线程挂起了，所以线程就会一直在这里阻塞。在调用signal方法后仅仅只是将结点从条件队列转移到同步队列中去，至于会不会唤醒线程需要看情况。如果转移结点时发现同步队列中的前继结点已取消，或者是更新前继结点的状态为SIGNAL失败，这两种情况都会立即唤醒线程，否则的话在signal方法结束时就不会去唤醒已在同步队列中的线程，而是等到它的前继结点来唤醒。当然，线程阻塞在这里除了可以调用signal方法唤醒之外，线程还可以响应中断，如果线程在这里收到中断请求就会继续往下执行。可以看到线程醒来后会马上检查是否是由于中断唤醒的还是通过signal方法唤醒的，如果是因为中断唤醒的同样会将这个结点转移到同步队列中去，只不过是通过调用transferAfterCancelledWait方法来实现的。最后执行完这一步之后就会返回中断情况并跳出while循环。

第四步：结点移出条件队列后的操作
```java
//线程醒来后就会以独占模式获取锁
if (acquireQueued(node, savedState) && interruptMode != THROW_IE) {
    interruptMode = REINTERRUPT;
}
//这步操作主要为防止线程在signal之前中断而导致没与条件队列断绝联系
if (node.nextWaiter != null) {
    unlinkCancelledWaiters();
}
//根据中断模式进行响应的中断处理
if (interruptMode != 0) {
    reportInterruptAfterWait(interruptMode);
}

//结束条件等待后根据中断情况做出相应处理
private void reportInterruptAfterWait(int interruptMode) throws InterruptedException {
    //如果中断模式是THROW_IE就抛出异常
    if (interruptMode == THROW_IE) {
        throw new InterruptedException();
    //如果中断模式是REINTERRUPT就自己挂起
    } else if (interruptMode == REINTERRUPT) {
        selfInterrupt();
    }
}
```
当线程终止了while循环也就是条件等待后，就会回到同步队列中。不管是因为调用signal方法回去的还是因为线程中断导致的，结点最终都会在同步队列中。这时就会调用acquireQueued方法执行在同步队列中获取锁的操作，这个方法我们在独占模式这一篇已经详细的讲过。也就是说，结点从条件队列出来后又是乖乖的走独占模式下获取锁的那一套，等这个结点再次获得锁之后，就会调用reportInterruptAfterWait方法来根据这期间的中断情况做出相应的响应。如果中断发生在signal方法之前，interruptMode就为THROW_IE，再次获得锁后就抛出异常；如果中断发生在signal方法之后，interruptMode就为REINTERRUPT，再次获得锁后就重新中断。

2.不响应线程中断的条件等待
```java
//不响应线程中断的条件等待
public final void awaitUninterruptibly() {
    //将当前线程添加到条件队列尾部
    Node node = addConditionWaiter();
    //完全释放锁并返回当前同步状态
    int savedState = fullyRelease(node);
    boolean interrupted = false;
    //结点一直在while循环里进行条件等待
    while (!isOnSyncQueue(node)) {
        //条件队列中所有的线程都在这里被挂起
        LockSupport.park(this);
        //线程醒来发现中断并不会马上去响应
        if (Thread.interrupted()) {
            interrupted = true;
        }
    }
    if (acquireQueued(node, savedState) || interrupted) {
        //在这里响应所有中断请求, 满足以下两个条件之一就会将自己挂起
        //1.线程在条件等待时收到中断请求
        //2.线程在acquireQueued方法里收到中断请求
        selfInterrupt();
    }
}
```

3.设置相对时间的条件等待(不进行自旋)
```java
//设置定时条件等待(相对时间), 不进行自旋等待
public final long awaitNanos(long nanosTimeout) throws InterruptedException {
    //如果线程被中断则抛出异常
    if (Thread.interrupted()) {
        throw new InterruptedException();
    }
    //将当前线程添加到条件队列尾部
    Node node = addConditionWaiter();
    //在进入条件等待之前先完全释放锁
    int savedState = fullyRelease(node);
    long lastTime = System.nanoTime();
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        //判断超时时间是否用完了
        if (nanosTimeout <= 0L) {
            //如果已超时就需要执行取消条件等待操作
            transferAfterCancelledWait(node);
            break;
        }
        //将当前线程挂起一段时间, 线程在这期间可能被唤醒, 也可能自己醒来
        LockSupport.parkNanos(this, nanosTimeout);
        //线程醒来后先检查中断信息
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) {
            break;
        }
        long now = System.nanoTime();
        //超时时间每次减去条件等待的时间
        nanosTimeout -= now - lastTime;
        lastTime = now;
    }
    //线程醒来后就会以独占模式获取锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE) {
        interruptMode = REINTERRUPT;
    }
    //由于transferAfterCancelledWait方法没有把nextWaiter置空, 所有这里要再清理一遍
    if (node.nextWaiter != null) {
        unlinkCancelledWaiters();
    }
    //根据中断模式进行响应的中断处理
    if (interruptMode != 0) {
        reportInterruptAfterWait(interruptMode);
    }
    //返回剩余时间
    return nanosTimeout - (System.nanoTime() - lastTime);
}
```

4.设置相对时间的条件等待(进行自旋)
```java
//设置定时条件等待(相对时间), 进行自旋等待
public final boolean await(long time, TimeUnit unit) throws InterruptedException {
    if (unit == null) { throw new NullPointerException(); }
    //获取超时时间的毫秒数
    long nanosTimeout = unit.toNanos(time);
    //如果线程被中断则抛出异常
    if (Thread.interrupted()) { throw new InterruptedException(); }
    //将当前线程添加条件队列尾部
    Node node = addConditionWaiter();
    //在进入条件等待之前先完全释放锁
    int savedState = fullyRelease(node);
    //获取当前时间的毫秒数
    long lastTime = System.nanoTime();
    boolean timedout = false;
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        //如果超时就需要执行取消条件等待操作
        if (nanosTimeout <= 0L) {
            timedout = transferAfterCancelledWait(node);
            break;
        }
        //如果超时时间大于自旋时间, 就将线程挂起一段时间
        if (nanosTimeout >= spinForTimeoutThreshold) {
            LockSupport.parkNanos(this, nanosTimeout);
        }
        //线程醒来后先检查中断信息
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) {
            break;
        }
        long now = System.nanoTime();
        //超时时间每次减去条件等待的时间
        nanosTimeout -= now - lastTime;
        lastTime = now;
    }
    //线程醒来后就会以独占模式获取锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE) {
        interruptMode = REINTERRUPT;
    }
    //由于transferAfterCancelledWait方法没有把nextWaiter置空, 所有这里要再清理一遍
    if (node.nextWaiter != null) {
        unlinkCancelledWaiters();
    }
    //根据中断模式进行响应的中断处理
    if (interruptMode != 0) {
        reportInterruptAfterWait(interruptMode);
    }
    //返回是否超时标志
    return !timedout;
}
```

5.设置绝对时间的条件等待
```java
//设置定时条件等待(绝对时间)
public final boolean awaitUntil(Date deadline) throws InterruptedException {
    if (deadline == null) { throw new NullPointerException(); } 
    //获取绝对时间的毫秒数
    long abstime = deadline.getTime();
    //如果线程被中断则抛出异常
    if (Thread.interrupted()) { throw new InterruptedException(); }
    //将当前线程添加到条件队列尾部
    Node node = addConditionWaiter();
    //在进入条件等待之前先完全释放锁
    int savedState = fullyRelease(node);
    boolean timedout = false;
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        //如果超时就需要执行取消条件等待操作
        if (System.currentTimeMillis() > abstime) {
            timedout = transferAfterCancelledWait(node);
            break;
        }
        //将线程挂起一段时间, 期间线程可能被唤醒, 也可能到了点自己醒来
        LockSupport.parkUntil(this, abstime);
        //线程醒来后先检查中断信息
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) {
            break;
        }
    }
    //线程醒来后就会以独占模式获取锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE) {
        interruptMode = REINTERRUPT;
    }
    //由于transferAfterCancelledWait方法没有把nextWaiter置空, 所有这里要再清理一遍
    if (node.nextWaiter != null) {
        unlinkCancelledWaiters();
    }
    //根据中断模式进行响应的中断处理
    if (interruptMode != 0) {
        reportInterruptAfterWait(interruptMode);
    }
    //返回是否超时标志
    return !timedout;
}
```

6.唤醒条件队列中的头结点
```java
//唤醒条件队列中的下一个结点
public final void signal() {
    //判断当前线程是否持有锁
    if (!isHeldExclusively()) {
        throw new IllegalMonitorStateException();
    }
    Node first = firstWaiter;
    //如果条件队列中有排队者
    if (first != null) {
        //唤醒条件队列中的头结点
        doSignal(first);
    }
}

//唤醒条件队列中的头结点
private void doSignal(Node first) {
    do {
        //1.将firstWaiter引用向后移动一位
        if ( (firstWaiter = first.nextWaiter) == null) {
            lastWaiter = null;
        }
        //2.将头结点的后继结点引用置空
        first.nextWaiter = null;
        //3.将头结点转移到同步队列, 转移完成后有可能唤醒线程
        //4.如果transferForSignal操作失败就去唤醒下一个结点
    } while (!transferForSignal(first) && (first = firstWaiter) != null);
}

//将指定结点从条件队列转移到同步队列中
final boolean transferForSignal(Node node) {
    //将等待状态从CONDITION设置为0
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        //如果更新状态的操作失败就直接返回false
        //可能是transferAfterCancelledWait方法先将状态改变了, 导致这步CAS操作失败
        return false;
    }
    //将该结点添加到同步队列尾部
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL)) {
        //出现以下情况就会唤醒当前线程
        //1.前继结点是取消状态
        //2.更新前继结点的状态为SIGNAL操作失败
        LockSupport.unpark(node.thread);
    }
    return true;
}
```
可以看到signal方法最终的核心就是去调用transferForSignal方法，在transferForSignal方法中首先会用CAS操作将结点的状态从CONDITION设置为0，然后再调用enq方法将该结点添加到同步队列尾部。我们再看到接下来的if判断语句，这个判断语句主要是用来判断什么时候会去唤醒线程，出现这两种情况就会立即唤醒线程，一种是当发现前继结点的状态是取消状态时，还有一种是更新前继结点的状态失败时。这两种情况都会马上去唤醒线程，否则的话就仅仅只是将结点从条件队列中转移到同步队列中就完了，而不会立马去唤醒结点中的线程。signalAll方法也大致类似，只不过它是去循环遍历条件队列中的所有结点，并将它们转移到同步队列，转移结点的方法也还是调用transferForSignal方法。

7.唤醒条件队列的所有结点
```java
//唤醒条件队列后面的全部结点
public final void signalAll() {
    //判断当前线程是否持有锁
    if (!isHeldExclusively()) {
        throw new IllegalMonitorStateException();
    }
    //获取条件队列头结点
    Node first = firstWaiter;
    if (first != null) {
        //唤醒条件队列的所有结点
        doSignalAll(first);
    }
}

//唤醒条件队列的所有结点
private void doSignalAll(Node first) {
    //先把头结点和尾结点的引用置空
    lastWaiter = firstWaiter = null;
    do {
        //先获取后继结点的引用
        Node next = first.nextWaiter;
        //把即将转移的结点的后继引用置空
        first.nextWaiter = null;
        //将结点从条件队列转移到同步队列
        transferForSignal(first);
        //将引用指向下一个结点
        first = next;
    } while (first != null);
}
```
至此，我们整个的AbstractQueuedSynchronizer源码分析就结束了，相信通过这四篇的分析，大家能更好的掌握并理解AQS。这个类确实很重要，因为它是其他很多同步类的基石，由于笔者水平和表达能力有限，如果哪些地方没有表述清楚的，或者理解不到位的，还请广大读者们能够及时指正，共同探讨学习。可在下方留言阅读中所遇到的问题，如果有需要AQS注释源码的也可联系笔者索取。
