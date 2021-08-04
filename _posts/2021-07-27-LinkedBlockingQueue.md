---
layout: article
title: LinkedBlockingQueue源码解析
tags: ["源码解析", "Java", "并发容器"]
key: LinkedBlockingQueue-2021-07-27
show_author_profile: true
footer: false
show_edit_on_github: false
show_subscribe: false
---

`LinkedBlockingQueue`是`java.util.concurrent`中的一种队列容器，适合应用于并发编程中。从命名和所属的包可以看出它的特点：链表结构(Linked)、操作阻塞(Blocking)以及线程安全(concurrent)。这篇文章将围绕这些特点结合源码对它进行解析。
<!--more-->

# 源码梳理

- 代码基于 Java 8

## 数据结构

首先看一下内部类`Node`：

```java
    static class Node<E> {
        E item;

        Node<E> next;

        Node(E x) { item = x; }
    }
```

队列由节点组成，节点内保存真正的元素对象，同时节点内有`next`引用，指向下一个节点，因此很明显能看出其结构是单向链表。

## 成员变量

在看函数代码之前先瞧一眼其主要的成员变量：

```java
    //容量大小，在构造方法中赋值，默认为 Integer.MAX_VALUE
    private final int capacity;

    //当前元素数量，原子类
    private final AtomicInteger count = new AtomicInteger();

    //头结点
    transient Node<E> head;

    //尾节点
    private transient Node<E> last;

    //take 锁，取出元素时使用
    private final ReentrantLock takeLock = new ReentrantLock();

    //take 锁的条件对象，用于队列在空与非空状态切换时
    private final Condition notEmpty = takeLock.newCondition();

    //put 锁，添加元素时使用
    private final ReentrantLock putLock = new ReentrantLock();

    //put 锁的条件对象，用于队列在已满与不满状态切换时
    private final Condition notFull = putLock.newCondition();
```
其中比较重要的变量就是两把锁与其对应的`Condition`，关于`ReentrantLock`相关的源码解析可以看看我前面写的[文章](/archive.html?tag=AQS)。

## 主要方法

现在我们对下面这些主要方法进行分析

| 方法 | 作用 |
| :--- | :--- |
| put(E): void | 向队列中添加一个元素，如果队列已满则线程阻塞直到添加成功 |
| offer(E): boolean | 向队列中添加一个元素，如果队列已满会返回 false |
| offer(E, long, TimeUnit): boolean | 向队列中添加一个元素并指定超时时间，如果队列已满且直到超时仍无法添加成功返回 false |
| take(): E | 获取队列第一个元素，如果队列为空则线程阻塞直到获取成功 |
| poll(): E | 获取队列第一个元素，如果队列为空则返回 null |
| poll(long, TimeUnit): E | 获取队列第一个元素并指定超时时间，如果队列为空且直到超时仍无法成功获取则返回 null |
| peek(): E | 查看队列第一个元素，不会将该元素出列，如果队列为空返回 null |

`LinkedBlockingQueue`中一共有三个函数可以往队列里添加元素：

1. put(E): void
2. offer(E): boolean
3. offer(E, long, TimeUnit): boolean

`put`函数没有返回值，它与`offer`的区别是，在队列满的时候，`put`会阻塞等待，直到数据添加成功或者线程中断。`offer`则是在队列满时直接返回`false`。

### put(E): void

```java
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();

        int c = -1;
        Node<E> node = new Node<E>(e); //创建 Node
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        //首先获取 put 锁
        putLock.lockInterruptibly();
        try {
            //如果队列内元素个数已经满了，线程在 notFull 上 await，等待唤醒
            while (count.get() == capacity) {
                notFull.await();
            }
            //将元素放入队列尾部 
            //Go A
            enqueue(node);
            //获取当前元素个数并加 1
            c = count.getAndIncrement();
            //如果当前元素个数没达到容量上限，唤醒在 notFull 上 await 的线程
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            //释放锁
            putLock.unlock();
        }
        //如果队列之前是空的，唤醒在 notEmpty 上 await 的线程
        if (c == 0)
            //Go B
            signalNotEmpty();
    }

    //A
    private void enqueue(Node<E> node) {
        last = last.next = node;
    }

    //B
    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        //获取 take 锁
        takeLock.lock();
        try {
            //唤醒
            notEmpty.signal();
        } finally {
            //释放锁
            takeLock.unlock();
        }
    }
```

代码并不复杂，在获取锁后对队列数量判断是否达到上限，如果达到上限就先`await`阻塞，让出锁资源，等待唤醒。唤醒后如果数量还是达到上限（被其他线程抢先拿到锁添加元素），继续阻塞，直到添加成功后，再根据条件判断是否需要唤醒其他阻塞等待`put`和`take`的线程。

### offer(E): boolean

```java
    public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        final AtomicInteger count = this.count;
        //如果当前队列已满，直接返回 false
        if (count.get() == capacity)
            return false;
        int c = -1;
        //创建节点
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        //获取 put 锁
        putLock.lock();
        try {
            //判断元素数量是否达到上限
            if (count.get() < capacity) {
                //元素入列
                enqueue(node);
                //获取入列前的元素个数之后加 1
                c = count.getAndIncrement();
                //队列还没满，唤醒 notFull 上的线程
                if (c + 1 < capacity)
                    notFull.signal();
            }
        } finally {
            //释放锁
            putLock.unlock();
        }
        //入列前队列为空，唤醒 notEmpty 上的线程
        if (c == 0)
            signalNotEmpty();
        //如果入列成功 c >= 0
        return c >= 0;
    }
```

总体逻辑与`put`是一样的，只是在添加元素的时候如果容量已经达到上限，那么不阻塞线程，直接返回`false`表示入列失败。

### offer(E, long, TimeUnit): boolean

这个方法与`offer(E)`的区别是，在添加的时候如果队列满了，会在指定时间内尝试，如果超时仍无法添加则返回`false`。

```java
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        if (e == null) throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        int c = -1;
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            //如果队列已满，自旋
            //不断在 notFull 上 await 与被唤醒，直到成功入列或超时
            while (count.get() == capacity) {
                //nanos <= 0 表示已超时
                if (nanos <= 0)
                    return false;
                //在 notFull 上 await 指定时间
                //如果被其他线程唤醒或者到达 deadline 则更新 nanos，进行下一次循环
                nanos = notFull.awaitNanos(nanos);
            }
            enqueue(new Node<E>(e));
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return true;
    }
```

至此添加元素的部分就分析完成了，根据上面几个函数的代码，其实很容易就能猜到在获取元素部分的代码逻辑，基本上就是一一对应的相反逻辑，所以就这部分直接贴代码，不过多赘述。

### take(): E

```java
    public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        //另一把锁，获取元素用的锁
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            //数量为 0，则在 notEmpty 上await
            while (count.get() == 0) {
                notEmpty.await();
            }
            //元素出列
            x = dequeue();
            //获取出列前数量之后 -1
            //注意 c 是出列前的数量
            c = count.getAndDecrement();
            //队列内还有剩余元素则唤醒在 notEmpty 上等待的线程
            if (c > 1)
                notEmpty.signal();
        } finally {
            //释放锁
            takeLock.unlock();
        }
        //如果队列之前是满的，那么在获取后现在又变成不满状态
        //则唤醒唤醒在 notFull 上 await 的线程
        if (c == capacity)
            signalNotFull();
        return x;
    }
```

### poll(): E

```java
    public E poll() {
        final AtomicInteger count = this.count;
        //如果队列为空，返回 null
        if (count.get() == 0)
            return null;
        E x = null;
        int c = -1;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            //获取锁后再次判断队列容量
            if (count.get() > 0) {
                //元素出列
                x = dequeue();
                c = count.getAndDecrement();
                if (c > 1)
                    //与 take 相同
                    notEmpty.signal();
            }
        } finally {
            takeLock.unlock();
        }
        //与 take 相同
        if (c == capacity)
            signalNotFull();
        return x;
    }
```

### poll(long, TimeUnit): E

```java
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        E x = null;
        int c = -1;
        //如果队列为空，会等待 nanos 时间
        long nanos = unit.toNanos(timeout);
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            //队列为空，自旋，在 notEmpty 上 await
            //直到成功或超时，超时返回 null
            while (count.get() == 0) {
                if (nanos <= 0)
                    return null;
                //在 notEmpty 上 await 指定时间
                //如果被其他线程唤醒或者到达 deadline 则更新 nanos，进行下一次循环
                nanos = notEmpty.awaitNanos(nanos);
            }
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }
```

## 其他

`LinkedBlockingQueue`里还有个`peek`函数，可以用来获取队列第一个元素，但是并不会将它出列。

```java
    public E peek() {
        //队列为空时返回 null
        if (count.get() == 0)
            return null;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            //获取第一个元素并返回它的 item
            Node<E> first = head.next;
            if (first == null)
                return null;
            else
                return first.item;
        } finally {
            takeLock.unlock();
        }
    }
```

为什么上面的代码里`head`不是第一个元素？这是因为`head`其实是个虚节点，它的`item`是空的，使用虚节点的好处是可以使代码更简洁，在很多地方可以避免写类似`head != null`等判断。看一下元素出列时是怎么操作`head`节点的。

```java
    private E dequeue() {
        Node<E> h = head;
        Node<E> first = h.next; //第一个元素节点 first
        h.next = h; //将原 head 节点（虚节点）指向自身，帮助 GC
        head = first; //first 节点变成新的头结点（虚节点），item 变成 null
        E x = first.item;
        first.item = null;
        return x;
    }
```

## 总结

至此`LinkedBlockingQueue`的源码就大体分析完了，总体来说它就是一个基于单向链表的`FIFO`队列，容量默认为`Integer.MAX_VALUE`，内部通过两把锁`putLock`和`takeLock`负责在存/取元素时保证线程安全，在队列容量满时通过两个条件`notFull`和`notEmpty`来控制线程阻塞等待与唤醒。所以，熟悉`AQS`的话理解这个类就会十分简单。