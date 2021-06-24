---
layout: article
title: ReentrantLock 源码阅读
footer: false
show_edit_on_github: false
show_subscribe: false
---

# 前言
Java 中线程同步的方法除了最早接触的`synchronized`,`Object.wait()`等方式，还有另一种实现线程同步的方式，`ReentrantLock`类。

`ReentrantLock`实现同步锁的核心是`AQS`(AbstractQueuedSynchronizer)，但是其本身只实现了`Lock`接口，继承 AQS 的是其内部类`Sync`
(两个子类`NonfairSync`和`FairSync`)。

AQS 子类实现的方式是重写`tryAcquire()`和`tryRelease()`方法，原理是通过CAS修改状态获取锁。

# lock()

首先，从`lock()`方法入手。

```java
    public void lock() {
        sync.lock();
    }
```

调用的是成员变量`sync`的`lock()`方法，根据创建时目的可以为`NonfairSync`和`FairSync`
(`Sync`的两个子实现类，分别代表公平锁和非公平锁，`Sync`是`ReentrantLock`的内部类，真正继承`AQS`的类)。

以`NonfairSync`为例

```java
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }
```
*CAS* 设置 state 的值
- 成功则表明获得锁，设置一下`AQS`的当前线程，然后就可以执行线程后续的代码。
- 失败则表明锁被其他线程持有，执行`AQS`的`acquire(int)`方法。

```java
    public final void acquire(int arg) {
    if (!tryAcquire(arg) && 
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
    }
```
首先调用`tryAcquire(int)`，此方法在`AQS`中为空实现，具体实现在`NonfairSync`中，实际内部调用`Sync`类的`nofairTryAcquire(int)`

```java
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState(); //获取state
        if (c == 0) { //等于0，表示没有线程持有锁
            if (compareAndSetState(0, acquires)) { //CAS，成功表示获取锁, acquires 通常为 1
                setExclusiveOwnerThread(current); //设置占有线程
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) { //如果当前线程已经占有锁
            int nextc = c + acquires; //state = state + acquires
            if (nextc < 0) 
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false; //acquire 失败
    }
```

从上面的`else if`代码块可以看出`ReentrantLock`的可重入性，每重入一次（每一次调用`lock()`），state 值 + 1，因此完全释放锁需要调用同样次数的`unlock()`，当 state = 0 时表明锁被释放。

如果`tryAcquire(int)`失败的话，紧接着调用`acquireQueued(addWaiter(Node.EXCLUSIVE), arg))`，`addWaiter(Node.EXCLUSIVE)`会生成一个`Node`，这个`Node`实际就是对当前线程的一个封装。


```java
    private Node addWaiter(Node mode) { //mode = Node.EXCLUSIVE
        Node node = new Node(Thread.currentThread(), mode); //将线程封装成 Node 
        Node pred = tail; //获取尾节点
        if (pred != null) { //尾节点不为空，表明队列不为空
            node.prev = pred;
            if (compareAndSetTail(pred, node)) { //CAS 尝试将当前 Node 替换成新的尾节点
                pred.next = node; //成功替换为尾节点，将上一个尾节点的 next 指向当前 Node
                return node;
            }
        }
        enq(node); //失败，说明有其他线程成功替换尾节点，调用 enq(Node)
        return node;
    }

    private Node enq(final Node node) {
        for (;;) { //循环
            Node t = tail;
            if (t == null) { //尾节点为空 = 队列为空，需要初始化一个节点，该节点不代表任何线程，是一个 dummy 节点
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else { //同上面的代码相同，将当前 Node 设置到尾节点，失败则进入下一个循环，直至成功
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

通过上面的代码可以了解到，`AbstractQueuedSynchronizer`的`Queued`指的是在该类中维护了一个`Node`的双向链表结构的队列，这个队列中如果有 n 个线程，那么就会有 n + 1 个 Node，因为头节点是个 dummy Node。

由于选取的对象是`NonfairSync`，因此在`acquire(int)`中每个线程在被加入队列之前，会先尝试*CAS*获取锁，失败了再加入队列，体现了非公平锁的特点，队列中等待的线程有可能与该线程同时争取锁（即没有区分先来后到）。

# acquireQueued(Node, int)

在成功将`Node`放入队列后，紧接着`acquireQueued`方法

```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false; //中断标记
            for (;;) { //循环
                final Node p = node.predecessor(); //获取前一个 Node
                if (p == head && tryAcquire(arg)) { //判断前一个节点是不是 head，是的话 CAS 尝试获取锁
                    setHead(node); //成功获取锁，将当前 Node 设置为 head
                    p.next = null; //释放前节点的引用，帮助 GC
                    failed = false;
                    return interrupted; 
                }
                if (shouldParkAfterFailedAcquire(p, node) && //tryAcquire失败，检查是否 park (Go A)
                    parkAndCheckInterrupt()) //执行 park，并检查中断 (Go B)
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    // A
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            //前一个节点已经 SIGNAL 了，意味着当前节点可以直接 park
            // SIGNAL = -1, CONDITION = -2, PROPAGATE = -3, CANCELLED =1
            return true;
        if (ws > 0) { //前一个节点已经取消
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0); //循环向前寻找节点直到不为 CANCELLED 的节点
            pred.next = node; //弃置中间的 CANCELLED 节点
        } else {
            //前一个节点要么为 0 要么为 PROPAGATE，CAS 尝试将其设置为 SIGNAL
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

    // B
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
