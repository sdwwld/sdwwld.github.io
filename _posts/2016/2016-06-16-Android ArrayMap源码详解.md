---
layout: post
title: "Android ArrayMap源码详解"
subtitle: 'Android ArrayMap源码详解'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “看似寻常最奇崛，成如容易却艰辛。”
	--王安石

## 正文

分析源码之前先来介绍一下ArrayMap的存储结构，ArrayMap数据的存储不同于HashMap和SparseArray，在上一篇<a href="https://androidboke.com/2016/03/14/Android-SparseArray%E6%BA%90%E7%A0%81%E8%AF%A6%E8%A7%A3" target="_blank">Android SparseArray源码详解</a>中我们讲到SparseArray是以纯数组的形式存储的，一个数组存储的是key值一个数组存储的是value值，今天我们分析的ArrayMap和SparseArray有点类似，他也是以纯数组的形式存储，不过不同的是他的一个数组存储的是Hash值另一个数组存储的是key和value，其中key和value是成对出现的，key存储在数组的偶数位上，value存储在数组的奇数位上，我们先来看其中的一个构造方法

## ArrayMap

```java
    public ArrayMap(int capacity) {
        if (capacity == 0) {
            mHashes = ContainerHelpers.EMPTY_INTS;
            mArray = ContainerHelpers.EMPTY_OBJECTS;
        } else {
            allocArrays(capacity);
        }
        mSize = 0;
    }
```

当capacity不为0的时候调用allocArrays方法分配数组大小，在分析allocArrays源码之前，我们先来看一下freeArrays方法，

## freeArrays

```java
    private static void freeArrays(final int[] hashes, final Object[] array, final int size) {
        if (hashes.length == (BASE_SIZE*2)) {
            synchronized (ArrayMap.class) {
                if (mTwiceBaseCacheSize < CACHE_SIZE) {
                    array[0] = mTwiceBaseCache;
                    array[1] = hashes;
                    for (int i=(size<<1)-1; i>=2; i--) {
                        array[i] = null;
                    }
                    mTwiceBaseCache = array;
                    mTwiceBaseCacheSize++;
                    if (DEBUG) Log.d(TAG, "Storing 2x cache " + array
                            + " now have " + mTwiceBaseCacheSize + " entries");
                }
            }
        } else if (hashes.length == BASE_SIZE) {
            synchronized (ArrayMap.class) {
                if (mBaseCacheSize < CACHE_SIZE) {
                    array[0] = mBaseCache;
                    array[1] = hashes;
                    for (int i=(size<<1)-1; i>=2; i--) {
                        array[i] = null;
                    }
                    mBaseCache = array;
                    mBaseCacheSize++;
                    if (DEBUG) Log.d(TAG, "Storing 1x cache " + array
                            + " now have " + mBaseCacheSize + " entries");
                }
            }
        }
    }
```

BASE_SIZE的值为4，ArrayMap对于hashes.length为4和8的两种情况会进行缓存，上面的两种情况下原理都是一样的，我们就用下面的一种情况进行分析，缓存的数量也不是无线大的，当大于等于10（CACHE_SIZE）的时候也就不再进行缓存了，缓存的原理就是让array数组的第一个位置保存之前缓存的mBaseCache，第二个位置保存当前的hashes数组，其他的全部置为空，下面我们再来看一下之前的allocArrays方法，

## allocArrays

```java
    private void allocArrays(final int size) {
        if (mHashes == EMPTY_IMMUTABLE_INTS) {
            throw new UnsupportedOperationException("ArrayMap is immutable");
        }
        if (size == (BASE_SIZE*2)) {
			…………………………
        } else if (size == BASE_SIZE) {
            synchronized (ArrayMap.class) {
                if (mBaseCache != null) {
                    final Object[] array = mBaseCache;
                    mArray = array;
                    mBaseCache = (Object[])array[0];
                    mHashes = (int[])array[1];
                    array[0] = array[1] = null;
                    mBaseCacheSize--;
                    if (DEBUG) Log.d(TAG, "Retrieving 1x cache " + mHashes
                            + " now have " + mBaseCacheSize + " entries");
                    return;
                }
            }
        }

        mHashes = new int[size];
        mArray = new Object[size<<1];
    }
```

如果分配的尺寸不为4或者8，就初始化，我们看到最下面两行mArray的大小是mHashes的两倍，这是因为mArray存储的是key和value两个值。如果分配的尺寸为4或者8，就判断之前对这两种情况是否进行了缓存，如果缓存过就从缓存中取，取出来的时候会把array的值置空，在上面的freeArrays方法中我们知道array的第一个位置和第二个位置保存的有值，其他的都置为空，在这里把array[0]和array[1]也置为了空，但是有一点奇葩的地方就是mHashes的值确保留了下来，无论是在freeArrays方法中还是在allocArrays方法中，都没有把他置为默认值。通过ArrayMap的源码发现，这里mHashes的值无论改不改变基本上都没有什么太大影响，因为put的时候如果存在就被替换了，但在indexOf的方法中如果存在还要在继续比较key的值，只有hash和key都一样才会返回。我们下面来看一下indexOf(Object key, int hash)这个方法，

## indexOf

```java
    int indexOf(Object key, int hash) {
        final int N = mSize;

        // Important fast case: if nothing is in here, nothing to look for.
        if (N == 0) {
            return ~0;
        }

        int index = ContainerHelpers.binarySearch(mHashes, N, hash);

        // If the hash code wasn't found, then we have no entry for this key.
        if (index < 0) {
            return index;
        }

        // If the key at the returned index matches, that's what we want.
        if (key.equals(mArray[index<<1])) {
            return index;
        }

        // Search for a matching key after the index.
        int end;
        for (end = index + 1; end < N && mHashes[end] == hash; end++) {
            if (key.equals(mArray[end << 1])) return end;
        }

        // Search for a matching key before the index.
        for (int i = index - 1; i >= 0 && mHashes[i] == hash; i--) {
            if (key.equals(mArray[i << 1])) return i;
        }

        // Key not found -- return negative value indicating where a
        // new entry for this key should go.  We use the end of the
        // hash chain to reduce the number of array entries that will
        // need to be copied when inserting.
        return ~end;
    }
```

这个方法很简单，就是根据二分法查找来确定hash值在数组中的位置，如果没找到就返回一个负数，注意下面还有两个循环，这是因为mHashes数组中的hash值不是唯一的，只有hash值相同并且key也相同才会返回所在的位置，否则就返回一个负数。下面就来看一下put(K key, V value)这个方法。  

## put

```java
    @Override
    public V put(K key, V value) {
        final int hash;
        int index;
        if (key == null) {
            hash = 0;
            index = indexOfNull();
        } else {
            hash = key.hashCode();
            index = indexOf(key, hash);
        }
		//通过查找，如果找到就把原来的替换，
        if (index >= 0) {
            index = (index<<1) + 1;
            final V old = (V)mArray[index];
            mArray[index] = value;
            return old;
        }
		//在上一篇《Android SparseArray源码详解》讲过，根据二分法查找，如果没有找到就会返回一个负数，这里进行取反
        index = ~index;
		//如果满了就扩容
        if (mSize >= mHashes.length) {
			//扩容的尺寸，三目运算符
            final int n = mSize >= (BASE_SIZE*2) ? (mSize+(mSize>>1))
                    : (mSize >= BASE_SIZE ? (BASE_SIZE*2) : BASE_SIZE);

            if (DEBUG) Log.d(TAG, "put: grow from " + mHashes.length + " to " + n);

            final int[] ohashes = mHashes;
            final Object[] oarray = mArray;
			//扩容
            allocArrays(n);
			//如果原来有数据就把原来的数据拷贝到扩容后的数组中
            if (mHashes.length > 0) {
                if (DEBUG) Log.d(TAG, "put: copy 0-" + mSize + " to 0");
                System.arraycopy(ohashes, 0, mHashes, 0, ohashes.length);
                System.arraycopy(oarray, 0, mArray, 0, oarray.length);
            }

            freeArrays(ohashes, oarray, mSize);
        }
		//根据上面的二分法查找，如果index小于mSize，说明新的数据是插入到数组之间index位置，插入之前需要把后面的移位
        if (index < mSize) {
            if (DEBUG) Log.d(TAG, "put: move " + index + "-" + (mSize-index)
                    + " to " + (index+1));
            System.arraycopy(mHashes, index, mHashes, index + 1, mSize - index);
            System.arraycopy(mArray, index << 1, mArray, (index + 1) << 1, (mSize - index) << 1);
        }
		//数据保存，mHashes只有hash值，mArray即保存key值又保存value值，
        mHashes[index] = hash;
        mArray[index<<1] = key;
        mArray[(index<<1)+1] = value;
        mSize++;
        return null;
    }
```

还有clear()方法和erase()方法，这两个区别就是clear()把所有的数据清空，并释放空间，erase()清空数据但没有释放空间，并且erase()只清mArray数据，mHashes数据并没有清空，这就是上面讲到的mHashes即使没清空也不会有影响，代码比较少就不在看了。在看一下和put类似的一个方法append(K key, V value)

## append

```java
    /**
     * Special fast path for appending items to the end of the array without validation.
     * The array must already be large enough to contain the item.
     * @hide
     */
    public void append(K key, V value) {
        int index = mSize;
        final int hash = key == null ? 0 : key.hashCode();
        if (index >= mHashes.length) {
            throw new IllegalStateException("Array is full");
        }
        if (index > 0 && mHashes[index-1] > hash) {
            RuntimeException e = new RuntimeException("here");
            e.fillInStackTrace();
            Log.w(TAG, "New hash " + hash
                    + " is before end of array hash " + mHashes[index-1]
                    + " at index " + index + " key " + key, e);
            put(key, value);
            return;
        }
        mSize = index+1;
        mHashes[index] = hash;
        index <<= 1;
        mArray[index] = key;
        mArray[index+1] = value;
    }
```

我们看注释这个方法是隐藏的，没有开放，因为这个方法不稳定，如果调用可能就会出现问题，看上面的注释，意思是说这个方法存储数据的时候没有验证，因为在最后存储的时候，是直接存进去的，这就会有一个问题，如果之前存过相同的key和value，再调用这个方法，很可能会再次存入，就可能会有两个key和value完全一样的，我个人认为如果把上面的if (index > 0 && mHashes[index-1] > hash)改为if (index > 0 && mHashes[index-1] >= hash)应该就没问题了，因为如果有相同的就调用put方法把原来的替换，不明白他为什么要这样写，下面再看一个方法validate()

## validate

```java
    /**
     * The use of the {@link #append} function can result in invalid array maps, in particular
     * an array map where the same key appears multiple times.  This function verifies that
     * the array map is valid, throwing IllegalArgumentException if a problem is found.  The
     * main use for this method is validating an array map after unpacking from an IPC, to
     * protect against malicious callers.
     * @hide
     */
    public void validate() {
        final int N = mSize;
        if (N <= 1) {
            // There can't be dups.
            return;
        }
        int basehash = mHashes[0];
        int basei = 0;
        for (int i=1; i<N; i++) {
            int hash = mHashes[i];
            if (hash != basehash) {
                basehash = hash;
                basei = i;
                continue;
            }
            // We are in a run of entries with the same hash code.  Go backwards through
            // the array to see if any keys are the same.
            final Object cur = mArray[i<<1];
            for (int j=i-1; j>=basei; j--) {
                final Object prev = mArray[j<<1];
                if (cur == prev) {
                    throw new IllegalArgumentException("Duplicate key in ArrayMap: " + cur);
                }
                if (cur != null && prev != null && cur.equals(prev)) {
                    throw new IllegalArgumentException("Duplicate key in ArrayMap: " + cur);
                }
            }
        }
    }
```

看上面的注释也是隐藏的，存储的时候可能会存在多个相同的key，这个方法就是用来验证的，这个方法很好理解，因为我们存储数据的时候是按照二分法查找然后存储的，如果hash值相同，那么存储的时候肯定是挨着的，在这里进行验证，对挨着相同hash值的数据进行key比较，如果key相同，则说明已经存在了，就会报异常。我们再来看最后一个方法

## removeAt

```java
    public V removeAt(int index) {
        final Object old = mArray[(index << 1) + 1];
		//如果小于等于1就全部清空
        if (mSize <= 1) {
            // Now empty.
            if (DEBUG) Log.d(TAG, "remove: shrink from " + mHashes.length + " to 0");
            freeArrays(mHashes, mArray, mSize);
            mHashes = EmptyArray.INT;
            mArray = EmptyArray.OBJECT;
            mSize = 0;
        } else {
			 // 如果数组比较大，但使用的比较少，就会重新分配空间
            if (mHashes.length > (BASE_SIZE*2) && mSize < mHashes.length/3) {
                // Shrunk enough to reduce size of arrays.  We don't allow it to
                // shrink smaller than (BASE_SIZE*2) to avoid flapping between
                // that and BASE_SIZE.
				//重新计算空间，当大于8的时候会1.5倍增长
                final int n = mSize > (BASE_SIZE*2) ? (mSize + (mSize>>1)) : (BASE_SIZE*2);

                if (DEBUG) Log.d(TAG, "remove: shrink from " + mHashes.length + " to " + n);

                final int[] ohashes = mHashes;
                final Object[] oarray = mArray;
				// 重新分配空间
                allocArrays(n);

                mSize--;
                if (index > 0) {
					//如果删除的位置大于0，拷贝前半部分到新数组中
                    if (DEBUG) Log.d(TAG, "remove: copy from 0-" + index + " to 0");
                    System.arraycopy(ohashes, 0, mHashes, 0, index);
                    System.arraycopy(oarray, 0, mArray, 0, index << 1);
                }
                if (index < mSize) {
					// 如果删除的位置小于mSize，把index位置以后的数据拷贝到新数组中
                    if (DEBUG) Log.d(TAG, "remove: copy from " + (index+1) + "-" + mSize
                            + " to " + index);
                    System.arraycopy(ohashes, index + 1, mHashes, index, mSize - index);
                    System.arraycopy(oarray, (index + 1) << 1, mArray, index << 1,
                            (mSize - index) << 1);
                }
            } else {
                mSize--;
                if (index < mSize) {
					//同上
                    if (DEBUG) Log.d(TAG, "remove: move " + (index+1) + "-" + mSize
                            + " to " + index);
                    System.arraycopy(mHashes, index + 1, mHashes, index, mSize - index);
                    System.arraycopy(mArray, (index + 1) << 1, mArray, index << 1,
                            (mSize - index) << 1);
                }
				// 把移除的位置置空，上面的为什么没有置空，是因为上面的数据拷贝到一个新的数组中，而删除的就没有
				//拷贝，这里要置空是因为这里数组没有扩容，还是在原来的数组操作，所以必须置空
                mArray[mSize << 1] = null;
                mArray[(mSize << 1) + 1] = null;
            }
        }
        return (V)old;
    }
```

剩下的方法都比较简单，这里就不在一一分析。