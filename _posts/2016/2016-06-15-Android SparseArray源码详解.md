---
layout: post
title: "Android SparseArray源码详解"
subtitle: 'Android SparseArray源码详解'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “不要人夸好颜色，只流清气满乾坤。”
	--王冕

## 正文

在Android开发中如果使用key为Integer的HashMap，就会出现黄色警告，提示使用SparseArray，SparseArray具有比HashMap更高的内存使用效率，我们在前面的<a href="https://androidboke.com/2016/04/15/Android-HashMap%E6%BA%90%E7%A0%81%E8%AF%A6%E8%A7%A3" target="_blank">Android HashMap源码详解</a>中提到，HashMap的存储方式是数组加链表，今天要分析的SparseArray是使用纯数组的形式存储。我们先来看其中的一个构造方法

## SparseArray

```html
    public SparseArray(int initialCapacity) {
        if (initialCapacity == 0) {
            mKeys = ContainerHelpers.EMPTY_INTS;
            mValues = ContainerHelpers.EMPTY_OBJECTS;
        } else {
            initialCapacity = ArrayUtils.idealIntArraySize(initialCapacity);
            mKeys = new int[initialCapacity];
            mValues = new Object[initialCapacity];
        }
        mSize = 0;
    }
```

先给一个初始空间的大小，默认的是10，但是这个最终空间大小是由计算得到的最理想的大小，

## idealIntArraySize

```java
    public static int idealIntArraySize(int need) {
        return idealByteArraySize(need * 4) / 4;
    }

    public static int idealByteArraySize(int need) {
        for (int i = 4; i < 32; i++)
            if (need <= (1 << i) - 12)
                return (1 << i) - 12;

        return need;
    }
```

这就是他所谓的理想大小，不过一直没看明白他为什么要这样计算。

我们先来看一下gc()这个方法

## gc

```java
    private void gc() {
        // Log.e("SparseArray", "gc start with " + mSize);

        int n = mSize;
        int o = 0;
        int[] keys = mKeys;
        Object[] values = mValues;

        for (int i = 0; i < n; i++) {
            Object val = values[i];

            if (val != DELETED) {
                if (i != o) {
                    keys[o] = keys[i];
                    values[o] = val;
                    values[i] = null;
                }

                o++;
            }
        }

        mGarbage = false;
        mSize = o;

        // Log.e("SparseArray", "gc end with " + mSize);
    }
```

这个方法很简单，就是把元素重新排放，如果之前有删除的，就把后面的挪到前面，删除之后就会标注为DELETED，我们主要看一下put(int key, E value)方法

## put

```java
    /**
     * Adds a mapping from the specified key to the specified value,
     * replacing the previous mapping from the specified key if there
     * was one.
     */
    public void put(int key, E value) {
		//通过二分法查找
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

        if (i >= 0) {
			//如果找到，说明这个key是存在的，替换就行了。
            mValues[i] = value;
        } else {
			//如果没找到就取反，binarySearch方法没找到返回的是大于key所在下标的取反，在这里再取反
			//返回的正好是大于key所在下标的值
            i = ~i;
			//首先说明一点，是有的key值存放的时候都是排序好的，如果当前存放的key大于数组中最大的key
			//那么这时的i肯定是大于mSize的，在这里i小于mSize说明这里的key是小于mKeys[]中的最大值的，
			//如果mValue[i]被删除了，就把当前的key和value放入其中，在这里举个例子，比如下面的数组
			//{1,3,7,9,13,16,22}如果key为7通过二分法查找得到的i为2，如果key为8则得到的i为-4，通过取反
			//为3，在下标为3的位置如果被删除了就用当前的值替换掉
            if (i < mSize && mValues[i] == DELETED) {
                mKeys[i] = key;
                mValues[i] = value;
                return;
            }
			//如果当前下标为i的没有被删除，就会执行下面的代码。如果对数据进行了操作，就是mGarbage为true，
			//并且当前的数据已经满了就调用gc()，然后再重新查找，因为gc之后数据的位置可能会有变化，所以要
			//必须重新查找
            if (mGarbage && mSize >= mKeys.length) {
                gc();

                // Search again because indices may have changed.
                i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
            }
			//当目前空间满了以后需要重新计算最理想的数组大小，然后再对数组进行扩容。
            if (mSize >= mKeys.length) {
                int n = ArrayUtils.idealIntArraySize(mSize + 1);

                int[] nkeys = new int[n];
                Object[] nvalues = new Object[n];

                // Log.e("SparseArray", "grow " + mKeys.length + " to " + n);
                System.arraycopy(mKeys, 0, nkeys, 0, mKeys.length);
                System.arraycopy(mValues, 0, nvalues, 0, mValues.length);

                mKeys = nkeys;
                mValues = nvalues;
            }
			//这里的i有可能是上面重新查找的i，根据上面的二分法查找如果等于mSize，说明当前的key比mKeys中的任何
			//值都要大，肯定要按顺序放在mKeys数组中最大值的后面，如果不等于，说明当前的key应该放到mKeys数组中
			//间下标为i的位置，需要对当前大于key的值向后移一位。
            if (mSize - i != 0) {
                // Log.e("SparseArray", "move " + (mSize - i));
                System.arraycopy(mKeys, i, mKeys, i + 1, mSize - i);
                System.arraycopy(mValues, i, mValues, i + 1, mSize - i);
            }
			//存放数据
            mKeys[i] = key;
            mValues[i] = value;
            mSize++;
        }
    }
```

我们再来看一下上面提到的binarySearch(int[] array, int size, int value)方法

## binarySearch

```java
    static int binarySearch(int[] array, int size, int value) {
        int lo = 0;
        int hi = size - 1;

        while (lo <= hi) {
            int mid = (lo + hi) >>> 1;
            int midVal = array[mid];

            if (midVal < value) {
                lo = mid + 1;
            } else if (midVal > value) {
                hi = mid - 1;
            } else {
                return mid;  // value found
            }
        }
        return ~lo;  // value not present
    }
```

这就是二分法查找，前提是数组必须是排序好的并且是升序排列，原理就是通过循环用当前的value和数组中间的值进行比较，如果小于就在前半部分查找，如果大于就在后半部分查找。最后如果找到就返回所在的下标，如果没有就返回一个负数。剩下的remove(int key)方法和delete(int key)方法都很简单，删除的时候只是把他的value置为DELETED就可以了，这里就不在介绍。下面我们再来介绍最后一个方法append(int key, E value)

## append

```java
    /**
     * Puts a key/value pair into the array, optimizing for the case where
     * the key is greater than all existing keys in the array.
     */
    public void append(int key, E value) {
        if (mSize != 0 && key <= mKeys[mSize - 1]) {
            put(key, value);
            return;
        }

        if (mGarbage && mSize >= mKeys.length) {
            gc();
        }

        int pos = mSize;
        if (pos >= mKeys.length) {
            int n = ArrayUtils.idealIntArraySize(pos + 1);

            int[] nkeys = new int[n];
            Object[] nvalues = new Object[n];

            // Log.e("SparseArray", "grow " + mKeys.length + " to " + n);
            System.arraycopy(mKeys, 0, nkeys, 0, mKeys.length);
            System.arraycopy(mValues, 0, nvalues, 0, mValues.length);

            mKeys = nkeys;
            mValues = nvalues;
        }

        mKeys[pos] = key;
        mValues[pos] = value;
        mSize = pos + 1;
    }
```

通过上面的注释我们知道如果当前的key比mKeys中的任何一个都大时，使用这个方法比put方法效率更好一些，这个方法和put差不多，put方法的key可以是任何值，但append方法的key值更偏向于大于mKeys的最大值，如果小于就会调用put方法。