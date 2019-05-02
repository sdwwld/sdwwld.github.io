---
layout: post
title: "ArrayBlockingQueue源码讲解"
subtitle: 'ArrayBlockingQueue源码讲解'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “古人不见今时月，今月曾经照古人。”

## 正文

源码：\sources\android-25

ArrayBlockingQueue是一个数组阻塞队列，这个队列的元素是先进先出，head元素是最先加入的，tail是最后加入的，并且新的元素加入到tail，获取元素从head开始。他有两个int指针，putIndex指向队尾的下一个，是空，表示下一个存放的位置，takeInput指向队首，表示下一个读取的位置takeInput有可能大于putIndex，也有可能小于putIndex。takeInput有固定大小，一旦创建大小则不能改变，如果把一个元素放入到一个满的队列中则会阻塞，同样如果从一个空的队列中获取元素也会阻塞。先看一下ArrayBlockingQueue的几个方法，从上往下看，先看第一个dec(int i)

## dec

```java
    /**
     * Circularly decrements array index i.
     */
    final int dec(int i) {
		//他表示获取当前下标i的前一个下标，如果i为0就表示获取head的前一个，
		//这里让他等于最后一个。
        return ((i == 0) ? items.length : i) - 1;
    }
```

接着看下一个方法enqueue(E x)，表示加入一个元素x，

## enqueue

```java
    /**
     * Inserts element at current put position, advances, and signals.
     * Call only when holding lock.
     */
    private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
		//putIndex表示最后插入元素的index
        items[putIndex] = x;
		//如果puIndex等于items的长度，则让putIndex等于0，
        if (++putIndex == items.length) putIndex = 0;
        count++;
        notEmpty.signal();
    }
```

接着下一个方法dequeue()表示移除第一个元素，注意第一个元素是takeIndex下标的元素，最后一个元素是putIndex下标的上一个元素，putIndex指向的是下一个存放的位置。接着看下一个方法removeAt(final int removeIndex)，移除指定位置的元素

## removeAt

```java
    /**
     * Deletes item at array index removeIndex.
     * Utility for remove(Object) and iterator.remove.
     * Call only when holding lock.
     */
    void removeAt(final int removeIndex) {
        // assert lock.getHoldCount() == 1;
        // assert items[removeIndex] != null;
        // assert removeIndex >= 0 && removeIndex < items.length;
        final Object[] items = this.items;
        if (removeIndex == takeIndex) {
			//如果移除的正好等于头元素，直接移除，因为这个队列就是先进先出的。
            // removing front item; just advance
            items[takeIndex] = null;
			//如果takeIndex等于items的长度，则让他等于0，相当于从队尾又指向队首
            if (++takeIndex == items.length) takeIndex = 0;
            count--;
            if (itrs != null)
                itrs.elementDequeued();
        } else {
            // an "interior" remove

            // slide over all others up through putIndex.
            for (int i = removeIndex, putIndex = this.putIndex;;) {
                int pred = i;
                if (++i == items.length) i = 0;
				//打算把后面的往前移，直到找到putIndex才会break，如果后面一个正好是putIndex
				//（putIndex相当于队尾，就是元素存放的位置，实际上还没有存，是空的），则直接
				//把当前下标为i的删除，然后再让putIndex等于i，
                if (i == putIndex) {
                    items[pred] = null;
                    this.putIndex = pred;
                    break;
                }
				//把后面的往前移
                items[pred] = items[i];
            }
            count--;
            if (itrs != null)
                itrs.removedAt(removeIndex);
        }
        notFull.signal();
    }
```

接着看下一个remove(Object o)方法

## remove

```java
    public boolean remove(Object o) {
        if (o == null) return false;
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count > 0) {
                final Object[] items = this.items;
				//putIndex相当于尾
                final int putIndex = this.putIndex;
				//takeIndex相当于头，但要注意，putIndex和takeIndex相当于
				//两个指针，有可能putIndex大于takeIndex，也有可能takeIndex大于putIndex
                int i = takeIndex;
				//从takeIndex开始检查是否相等，如果相等则直接返回，否则循环检查，直到
				//i等于putIndex为止，因为takeIndex是最先加入的，元素的范围也就是从
				// takeIndex到putIndex，这里takeIndex和putIndex谁大谁小还不一定，
				//这里可以参照前面写的《ArrayDeque源码详解》
                do {
                    if (o.equals(items[i])) {
                        removeAt(i);
                        return true;
                    }
                    if (++i == items.length) i = 0 ;//如果找到数组的最后，要从头开始
                } while (i != putIndex);
            }
            return false;
        } finally {
            lock.unlock();
        }
    }
```

接着是contains(Object o)方法，和上面的contains(Object o)方法差不多，就不在介绍。接着是Object[] toArray()方法，这里要分两种情况，一种是putIndex大于takeIndex，还一种是putIndex小于takeIndex。这个可以参照前面讲的<a href="https://androidboke.com/2017/08/24/ArrayDeque源码详解" target="_blank">ArrayDeque源码详解</a>的Object[] toArray()方法，这里就不在介绍。再来看最后一个方法drainTo

## drainTo

```java
    public int drainTo(Collection<? super E> c, int maxElements) {
		//从takeIndex开始提取maxElements个元素存储在集合c中，如果maxElements
		//大于数组，则提取大小为items.size();提取之后把当前的ArrayBlockingQueue
		//中提取的删除。
        Objects.requireNonNull(c);
        if (c == this)
            throw new IllegalArgumentException();
        if (maxElements <= 0)
            return 0;
        final Object[] items = this.items;
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            int n = Math.min(maxElements, count);
            int take = takeIndex;
            int i = 0;
            try {
                while (i < n) {
                    @SuppressWarnings("unchecked")
                    E x = (E) items[take];
                    c.add(x);
                    items[take] = null;
                    if (++take == items.length) take = 0;
                    i++;
                }
                return n;
            } finally {
                // Restore invariants even if c.add() threw
                if (i > 0) {
                    count -= i;
                    takeIndex = take;
                    if (itrs != null) {
                        if (count == 0)
                            itrs.queueIsEmpty();
                        else if (i > take)
                            itrs.takeIndexWrapped();
                    }
                    for (; i > 0 && lock.hasWaiters(notFull); i--)
                        notFull.signal();
                }
            }
        } finally {
            lock.unlock();
        }
    }
```