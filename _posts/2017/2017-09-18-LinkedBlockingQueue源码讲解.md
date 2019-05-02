---
layout: post
title: "LinkedBlockingQueue源码讲解"
subtitle: 'LinkedBlockingQueue源码讲解'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “莫道君行早，更有早行人。莫信直中直，须防仁不仁。”

## 正文

源码：\sources\Android-25

说到LinkedBlockingQueue，不得不提到LinkedBlockingDeque，他俩差不多，只不过LinkedBlockingDeque是双向的队列，而LinkedBlockingQueue是单向的队列，我们看一下他的节点就知道了

## Node

```java
    /**
     * Linked list node class.
     */
    static class Node<E> {
        E item;

        /**
         * One of:
         * - the real successor Node
         * - this Node, meaning the successor is head.next
         * - null, meaning there is no successor (this is the last node)
         */
        Node<E> next;

        Node(E x) { item = x; }
    }
```

Node只有后一个没有前一个，并且他是有容量限制的，容量大小是capacity，容量满了如果再加入就会阻塞。来看第一个方法dequeue

## dequeue

```java
    /**
     * Removes a node from head of queue.
     *
     * @return the node
     */
	 //移除一个元素
    private E dequeue() {
        // assert takeLock.isHeldByCurrentThread();
        // assert head.item == null;
        Node<E> h = head;
        Node<E> first = h.next;
		//让自己的下一个指向自己，相当于与链表断开了。
        h.next = h; // help GC
		//然后让head的先一个成为head元素。
        head = first;
		//head元素的item是空的。
        E x = first.item;
        first.item = null;
        return x;
    }
```

再来看其中的一个构造方法LinkedBlockingQueue(Collection<? extends E> c)，

## LinkedBlockingQueue

```html
    /**
     * Creates a {@code LinkedBlockingQueue} with a capacity of
     * {@link Integer#MAX_VALUE}, initially containing the elements of the
     * given collection,
     * added in traversal order of the collection's iterator.
     *
     * @param c the collection of elements to initially contain
     * @throws NullPointerException if the specified collection or any
     *         of its elements are null
     */
    public LinkedBlockingQueue(Collection<? extends E> c) {
        this(Integer.MAX_VALUE);
        final ReentrantLock putLock = this.putLock;
        putLock.lock(); // Never contended, but necessary for visibility
        try {
            int n = 0;
            for (E e : c) {
                if (e == null)
                    throw new NullPointerException();
					//如果满了就会报异常
                if (n == capacity)
                    throw new IllegalStateException("Queue full");
					//添加
                enqueue(new Node<E>(e));
                ++n;
            }
            count.set(n);
        } finally {
            putLock.unlock();
        }
    }
```

put(E e)方法调用的也是enqueue方法，如果满了就会等待。

```html
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node);
```

然后再看下一个方法peek，表示的是返回head节点的下一个节点值，因为head节点是没有值的，

## peek

```html
    public E peek() {
        if (count.get() == 0)
            return null;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            return (count.get() > 0) ? head.next.item : null;
        } finally {
            takeLock.unlock();
        }
    }
```

接着看unlink(Node<E> p, Node<E> trail)方法，表示断开p到trail之间的节点，

## unlink

```html
    /**
     * Unlinks interior Node p with predecessor trail.
     */
    void unlink(Node<E> p, Node<E> trail) {
        // assert isFullyLocked();
        // p.next is not changed, to allow iterators that are
        // traversing p to maintain their weak-consistency guarantee.
        p.item = null;
        trail.next = p.next;
        if (last == p)
            last = trail;
        if (count.getAndDecrement() == capacity)
            notFull.signal();
    }
```

剩下的基本上也没什么可说的。