---
layout: article
title: AQS之Condition源码解析
tags: ["源码解析", "AQS", "Java"]
key: AQS_Condition-2021-07-13
show_author_profile: true
footer: false
show_edit_on_github: false
show_subscribe: false
---

之前在[线程同步「AQS原理解析」](/2021/06/26/AQS.html)中我们已经了解了`AQS`的线程同步工作原理，这篇文章来分析一下`AQS`中的**等待/唤醒**机制。

<!--more-->

我们在 Java 中使用`Object#wait`和 `Object#notify`来进行线程等待和唤醒的时，有一个缺点就是只能关联一个条件队列，对于一些需要多个条件的场景，可能就需要创建多把锁来实现。如果使用`AQS`的话（本文依旧以`ReentrantLock`为例），只需要用`ReentrantLock`创建多个`Condition`就可以实现这个目的。

## Tips

- 代码基于 Java 8
- 全文会用`AQS`指代`AbstractQueuedSynchronizer`
- 为了提升阅读流畅性，文中会在需要的地方将代码二次贴出，并加上`repeat`标识

# 源码梳理

## Condition 的创建

```java
    //Reentrant#newCondition
    public Condition newCondition() {
        return sync.newCondition();
    }

    //Reentrant$Sync#newCondition
    final ConditionObject newCondition() {
        return new ConditionObject();
    }
```
代码非常简单，`Condition`是一个接口类，声明了`await`、`signal`等方法，`ConditionObject`实现了`Condition`接口，同时它是`AQS`的**非静态**内部类，所以每个`ConditionObject`都持有其对应`AQS`对象的引用，这意味着它可以直接访问`AQS`对象的内部资源，最重要的就是它可以操作`AQS`对象中的同步队列链表。

## 线程等待

### ConditionObject#await

跟`Object#wait`一样，只有在持有锁的前提下才可以调用`ConditionObject#await`，否则会抛出异常。

```java
    //ConditionObject#await
    public final void await() throws InterruptedException {
        //判断线程是否 interrupted
        if (Thread.interrupted())
            throw new InterruptedException();
        Node node = addConditionWaiter(); //创建节点
        //释放锁并保存状态在 savedState 中
        int savedState = fullyRelease(node);
        int interruptMode = 0;
        //判断是否在 AQS 的队列中
        //如果不在说明还在 ConditionObject 的队列中进行等待，可以 park
        while (!isOnSyncQueue(node)) { 
            LockSupport.park(this); //park 线程
            //如果线程已经 interrupted，退出循环
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        //...省略
    }
```

上面就是`await`总体的流程，省略部分是线程被唤醒后的内容，我将其放到后面唤醒部分一起解析。这部分代码所做的工作与`AQS`入列时很像，都需要创建`Node`节点，但是这里`Node`被放入的将是`ConditionObject`中自己的链表。下面看看这些工作执行的细节。

### ConditionObject#addConditionWaiter

```java
    private Node addConditionWaiter() {
        //ConditionObject中有 firstWaiter 和 lastWaiter 两个成员变量
        //说明内部实现也是一个链表队列
        Node t = lastWaiter; 
        //判断队列是否为空以及最后一个节点是否已经取消
        if (t != null && t.waitStatus != Node.CONDITION) {
            unlinkCancelledWaiters();//清理已经取消的节点 Go A
            t = lastWaiter; //清理完毕的新 lastWaiter
        }
        //与 AQS 中一样，将线程封装成 Node，状态为 CONDITION
        Node node = new Node(Thread.currentThread(), Node.CONDITION);
        if (t == null)
            //队列为空，node 设置为头节点
            firstWaiter = node;
        else
            //node 加入链表
            t.nextWaiter = node;
        //node 设置为尾节点
        lastWaiter = node;
        return node;
    }

    //A
    //ConditionObject#unlinkCancelledWaiters
    private void unlinkCancelledWaiters() {
        Node t = firstWaiter; //头节点
        Node trail = null;
        while (t != null) { //遍历各个节点
            Node next = t.nextWaiter;
            //判断节点状态是否为 CONDITION
            if (t.waitStatus != Node.CONDITION) { 
                t.nextWaiter = null; //移除当前节点的 nextWaiter
                //trail 为空表明到现在还没有遍历到有效节点
                if (trail == null) 
                    //将 next 设为头节点
                    firstWaiter = next;
                else
                    //将 trail 的 nextWaiter 指向 next
                    //当前节点也就从链表中移除了
                    trail.nextWaiter = next;
                //next 为空表明链表已经遍历完成
                if (next == null)
                    lastWaiter = trail;
            }
            else
                //节点有效，设置为 trail，继续遍历
                trail = t;
            t = next;
        }
    }
```
`addConditionWaiter`中先清理了一下`ConditionObject`链表中失效的节点，然后将当前线程封装成`CONDITION`状态的`Node`，放到链表的尾端。结合之前的文章可以看到`Node`中`prev`和`next`引用应用在`AQS`的双向链表中，而`Node`的`nextWaiter`应用在`ConditionObject`的链表中，是个单向链表。

### AQS#fullyRelease
```java
    //AQS#fullyRelease
    final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            //获取 AQS state，也就是当前持有锁的线程的重入次数
            int savedState = getState();
            //释放锁
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                //释放失败，抛出异常，表明线程没有持有锁
                //与 Object#wait 一样必须先持有锁才可以线程等待
                throw new IllegalMonitorStateException();
            }
        } finally {
            //失败将节点设置为 CANCELLED
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
```

将`Node`放入等待队列后，接着就可以释放`Node`所持有的锁，释放的时候必须将当前锁的重入次数记录下来，等到重新获得锁的时候，将其恢复到等待前的重入次数。

### AQS#isOnSyncQueue

```java
    //AQS#isOnSyncQueue
    final boolean isOnSyncQueue(Node node) {
        //node 状态为 CONDITION 或 node.prev 为空表明 node 没有在同步队列里
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        //node.next 不为空表明 node 肯定在同步队列中
        if (node.next != null) 
            return true;
        //从同步队列尾部开始寻找 node 判断是否在队列里
        //Go A
        return findNodeFromTail(node);
    }

    //A
    //AQS#findNodeFromTail
    private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }
```

线程释放锁后，就是自旋判断自己有没有被重新放回到同步队列里，一旦回到同步队列也意味着它被从`Condition`链表中移除，可以重新争抢锁资源。如果没有在同步队列里那线程会`park`，减少 CPU 开销，直到被`signal`。

总的来说，线程在`AQS`框架中，大体上有三种状态，分别是：持有锁/位于同步队列/位于条件队列。这三种状态是互斥的，通过`Node`在两个队列里进行转移，来达到等待/唤醒等目的。

## 线程唤醒

### ConditionObject#signal

```java
    public final void signal() {
        if (!isHeldExclusively()) //如果没有持有锁抛出异常
            throw new IllegalMonitorStateException();
        Node first = firstWaiter; 
        if (first != null) //如果第一个节点不为空，唤醒它
            doSignal(first);
    }

    //ConditionObject#doSignal
    private void doSignal(Node first) {
        do {
            //判断链表是不是没有节点了
            if ((firstWaiter = first.nextWaiter) == null) 
                lastWaiter = null;
            first.nextWaiter = null; //移除 nextWaiter 指针
        } while (!transferForSignal(first) && //唤醒失败则唤醒下个节点
                    (first = firstWaiter) != null);
    }

    //ConditionObject#transferForSignal
    final boolean transferForSignal(Node node) {
        //更新节点状态为 0
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        //前文也出现的，将节点放入同步队列尾端，p 是它的 prev 节点
        Node p = enq(node); 
        int ws = p.waitStatus;
        //如果 p 已经 CANCELEED 或者无法设置成 SINGNAL，直接唤醒当前节点
        //清理一下同步队列中前面的失效节点
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```

`signal`中的逻辑不多，就这几个函数就完成了。主要就是从队列中拿到第一个有效节点，然后放到`AQS`同步队列中，判断需不需要直接`unpark`线程，如果不需要唤醒则保持`park`，等待其他线程调用`ReentrantLock#unlock`来`unpark`。

### 后续工作

前面`await`方法还有一部分`unpark`后的工作没分析，现在来看下：

```java
    public final void await() throws InterruptedException {
        //...省略
        while (!isOnSyncQueue(node)) { 
            //...
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }

        //1.调用 AQS#acquireQueued，也是老朋友了，自旋或park，直到获取锁
        //2.如果 acquireQueued 返回 false，说明 CANCELLED 了，判断中断模式
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        //CANCELLED了，判断需不需要从 Condition 链表中移除
        if (node.nextWaiter != null) 
            unlinkCancelledWaiters();
        //判断中断情况
        if (interruptMode != 0)
            //报告中断
            reportInterruptAfterWait(interruptMode);
    }
```

在结束`await`逻辑的时候需要报告中断状态，依据的是`interruptMode`的值。这里有三种情况：

1. `interruptMode`在`checkInterruptWhileWaiting(node)`的时候就被赋值为`THROW_IE`。
2. `interruptMode`在`checkInterruptWhileWaiting(node)`的时候就被赋值为`REINTERRUPT`。
3. 在`acquireQueued(node, savedState) && interruptMode != THROW_IE`的时候被赋值为`REINTERRUPT`。

```java
    //ConditionObject#checkInterruptWhileWaiting
    private int checkInterruptWhileWaiting(Node node) {
        return Thread.interrupted() ?
            (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
            0;
    }

    //AQS#transferAfterCancelledWait
    final boolean transferAfterCancelledWait(Node node) {
        //如果节点当前是 CONDITION 状态，说明还没有执行 signal
        //即 CANCELLED 发生在 signal 之前
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
            enq(node);
            return true;
        }
        //节点已经被 signal，循环直至 signal 执行完成
        while (!isOnSyncQueue(node))
            Thread.yield();
        return false;
    }
```

可以看到：

1. `transferAfterCancelledWait`返回`true`，说明这个节点的`waitStatus`还是`CONDITION`，也就是说它还在`ConditionObject`的链表中没有被`signal`，是因为中断导致的节点被唤醒，所以说`CANCELLED`发生在`signal`之前。此时中断状态为`THROW_IE`。

2. 节点已经被`signal`了，但是也发生了中断。举个例子，有两个线程A与B，A处于`CONDITION`状态下，此时 B 将 A `signal`，但在`signal`刚发生的时候 A 发生了中断，A被唤醒，此时中断发生在`signal`之后，`signal`工作还没有完成，A 处于从`Condition`队列到`AQS`队列的过渡状态，所以必须循环等待`signal`完成，也就是 A 被转移到`AQS`队列中后。此时中断状态为`REINTERRUPT`。

3. 节点被`signal`正常唤醒，`checkInterruptWhileWaiting`返回 0，但是紧接着在`acquireQueued`过程中中断了，这种情况也属于中断发生在`signal`之后。此时中断状态为`REINTERRUPT`。

再看看不同中断状态的报告处理是什么样的：

```java
    //ConditionObject#reportInterruptAfterWait
    private void reportInterruptAfterWait(int interruptMode)
        throws InterruptedException {
        if (interruptMode == THROW_IE)
            throw new InterruptedException();
        else if (interruptMode == REINTERRUPT)
            selfInterrupt();
    }

    //AQS#selfInterrupt
    static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }
```
`THROW_IE`在结束`await`函数的时候会抛出中断异常，而对于`REINTERRUPT`，不会抛出异常，只是再次调用线程的中断方法。因此，对于中断发生在`signal`以后的情况，都属于中断来的太晚，我们将忽略这个中断，不抛出异常。

## 总结

1. 线程进入`await`时是持有锁的，`await`过程中会将自己封装成节点添加进`ConditionObject`的等待队列中，然后让出锁并将线程挂起，此时线程只执行了`await`的一半工作，等待唤醒（`signal`或中断）。
2. 线程离开`await`时也是持有锁的，因为不管是被正常唤醒还是中断唤醒，线程都会从`ConditionObject`等待队列移动到`AQS`的同步队列，直到获取锁，然后执行剩余的`await`工作（处理中断）。
3. 线程`await`前的重入次数会在挂起时记录，唤醒时恢复。
4. 与`AQS`的双向同步队列不同，`ConditionObject`的等待队列是单向的链表。
5. `ConditionObject`是非静态内部类，可以直接访问`AQS`对象的内部资源。