---
layout: post
title: "LinkedBlockingDeque源码讲解"
subtitle: 'LinkedBlockingDeque源码讲解'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “有钱道真语，无钱语不真。不信但看筵中酒，杯杯先劝有钱人。”

## 正文

源码：\sources\Android-25

LinkedBlockingDeque是双向链表阻塞队列，他维护的是个链表节点Node，和前面的分析的PriorityBlockingQueue有一点区别，PriorityBlockingQueue维护的是个数组。他和LinkedList很相似，不过LinkedList没有指定容量，LinkedBlockingDeque是有容量大小的，大小为apacity，如果没有指定apacity，那么最大空间为Integer.MAX_VALUE.下面看一下LinkedBlockingDeque的主要方法。

## Node

```java
    /** Doubly-linked list node class */
    static final class Node<E> {
        /**
         * The item, or null if this node has been removed.
         */
        E item;

        /**
         * One of:
         * - the real predecessor Node
         * - this Node, meaning the predecessor is tail
         * - null, meaning there is no predecessor
         */
        Node<E> prev;

        /**
         * One of:
         * - the real successor Node
         * - this Node, meaning the successor is head
         * - null, meaning there is no successor
         */
        Node<E> next;

        Node(E x) {
            item = x;
        }
    }
```

Node就是LinkedBlockingDeque维护的链表节点，这个节点有个前一个prev和后一个next，所以他维护的链表是双向的。当然他还有两个变量，一个是指向第一个，一个是指向最后一个

```java
    /**
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient Node<E> first;

    /**
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;
```

LinkedBlockingDeque的构造方法比较多，下面随便看其中的一个。

## LinkedBlockingDeque

```java
    /**
     * Creates a {@code LinkedBlockingDeque} with a capacity of
     * {@link Integer#MAX_VALUE}, initially containing the elements of
     * the given collection, added in traversal order of the
     * collection's iterator.
     *
     * @param c the collection of elements to initially contain
     * @throws NullPointerException if the specified collection or any
     *         of its elements are null
     */
    public LinkedBlockingDeque(Collection<? extends E> c) {
        this(Integer.MAX_VALUE);
        final ReentrantLock lock = this.lock;
        lock.lock(); // Never contended, but necessary for visibility
        try {
            for (E e : c) {
                if (e == null)
                    throw new NullPointerException();
                if (!linkLast(new Node<E>(e)))//不停的添加到队尾，如果空间满就会抛出异常
                    throw new IllegalStateException("Deque full");
            }
        } finally {
            lock.unlock();
        }
    }
```

下面来看一下linkFirst方法

## linkFirst

```java
    /**
     * Links node as first element, or returns false if full.
     */
    private boolean linkFirst(Node<E> node) {
		//添加到第一个节点的前面
        // assert lock.isHeldByCurrentThread();
		//如果已经满了，就直接返回false。
        if (count >= capacity)
            return false;
        Node<E> f = first;
		//让原来的头结点成为新节点的下一个节点。
        node.next = f;
		//让新节点成为first节点
        first = node;
		//如果last为null，说明原来链表是为null的，也让尾节点指向新节点
        if (last == null)
            last = node;
        else
		//node已经是链表的头结点了，让node等于f的前一个节点，因为链表是双向的。
		//为什么last等于null的时候不执行这段代码，是因为last等于null的时候表示
		//没有节点的，添加的时候就一个节点。
            f.prev = node;
        ++count;//链表数量加1
        notEmpty.signal();
        return true;
    }
```

接着再看下一个方法linkLast

## linkLast

```java
    /**
     * Links node as last element, or returns false if full.
     */
    private boolean linkLast(Node<E> node) {
        // assert lock.isHeldByCurrentThread();
		// 添加到链表的尾，如果满了，直接返回false
        if (count >= capacity)
            return false;
        Node<E> l = last;
		//既然是添加到尾节点，那么之前尾节点就是新节点的前一个节点。
        node.prev = l;
		//然后让last指向新节点
        last = node;
        if (first == null)
		//如果原来节点为空，同时也让新节点成为首节点。
            first = node;
        else
		//因为是双向链表，所以这里让新节点成为原来尾节点的下一个节点
            l.next = node;
        ++count;//链表数量加1
        notEmpty.signal();
        return true;
    }
```

再看下一个方法unlinkFirst，断开首节点

## unlinkFirst

```java
    /**
     * Removes and returns first element, or null if empty.
     */
    private E unlinkFirst() {
		//移除第一个节点。
        // assert lock.isHeldByCurrentThread();
        Node<E> f = first;
        if (f == null)
            return null;
			//保存首节点的下一个节点
        Node<E> n = f.next;
        E item = f.item;
        f.item = null;
		//让首节点成为首节点的下一个节点的，就是自己的下一个指向自己，因为是first节点，
		//首节点的前一个为空，所以这里其实就是让首节点与链表断开了
        f.next = f; // help GC
		//让first节点指向first的下一个节点。
        first = n;
        if (n == null)
            last = null;
        else
		//既然n是首节点了，那么他的前一个肯定要为null的
            n.prev = null;
        --count;//数量减1
        notFull.signal();
        return item;
    }
```

接着是unlinkLast方法，表示断开最后一个节点

## unlinkLast

```java
    /**
     * Removes and returns last element, or null if empty.
     */
    private E unlinkLast() {
        // assert lock.isHeldByCurrentThread();
        Node<E> l = last;
		//如果为空，直接返回
        if (l == null)
            return null;
			//保存最后一个节点的前一个节点
        Node<E> p = l.prev;
        E item = l.item;
        l.item = null;
		//让最后一个节点的前一个指向自己，相当于最后一个节点与
		//链表断开。
        l.prev = l; // help GC
		//然后让last指向之前最后一个链表的前一个
        last = p;
		//如果等于null，相当于链表直接删除。
        if (p == null)
            first = null;
        else
		//否则让最后一个链表的下一个指向null
            p.next = null;
        --count;//数量减1
        notFull.signal();
        return item;
    }
```

继续看，下面一个是unlink方法

## unlink

```java
    /**
     * Unlinks x.
     */
    void unlink(Node<E> x) {
        // assert lock.isHeldByCurrentThread();
		//保存当前节点的前一个和后一个
        Node<E> p = x.prev;
        Node<E> n = x.next;
        if (p == null) {
			//如果前一个等于null，说明x是first节点，直接删除first节点即可
            unlinkFirst();
        } else if (n == null) {
		//如果后一个等于null，说明x是尾节点，直接删除last节点即可
            unlinkLast();
        } else {
		//链表的断开其实很简单，就是让当前节点的下一个成为当前节点前一个的下一个，
		//当前节点的前一个成为当前节点下一个的前一个。
            p.next = n;
            n.prev = p;
            x.item = null;
            // Don't mess with x's links.  They may still be in use by
            // an iterator.
            --count;
            notFull.signal();
        }
    }
```

接着看offerFirst方法，其实代码很简单

## offerFirst

```java
    /**
     * @throws NullPointerException {@inheritDoc}
     */
    public boolean offerFirst(E e) {
        if (e == null) throw new NullPointerException();
        Node<E> node = new Node<E>(e);
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return linkFirst(node);//断开首节点
        } finally {
            lock.unlock();
        }
    }
```

然后add，offe，put，remove，pollFirst，pollLast，takeFirst，takeLast，pollFirst，pollLast基本上都调用上面的方法，都很简单，这里就不在介绍。