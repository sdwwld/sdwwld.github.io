---
layout: post
title: "Android LruCache源码详解"
subtitle: '缓存类LruCache的原理分析'
author: "山大王"
header-style: text
tags:
  - 源码
  - android
---

之前的两篇我们详细分析了HashMap和LinkedHashMap，就是为了讲解LruCache做铺垫的，这一篇我们来分析一下Android中常用的缓存类LruCache，我们知道Android中的优化比较多，其中就有一个关于图片缓存的问题，如果处理不好很有可能会出现ANR。在讲解之前我们最好先看一下这个类的注释，由于比较多，我只贴出一部分

```java

 *   int cacheSize = 4 * 1024 * 1024; // 4MiB
 *   LruCache<String, Bitmap> bitmapCache = new LruCache<String, Bitmap>(cacheSize) {
 *       protected int sizeOf(String key, Bitmap value) {
 *           return value.getByteCount();
 *       }
 *   }}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)他初始化了一个4M大小的空间，一般情况下我们是使用最大内存的1/4或1/8，如果像上面那样写也是可以的。

```java
(int)Runtime.getRuntime().maxMemory()/4
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)但要记住必须要重写sizeOf方法，因为它默认是返回1的，

```java
    /**
     * Returns the size of the entry for {@code key} and {@code value} in
     * user-defined units.  The default implementation returns 1 so that size
     * is the number of entries and max size is the maximum number of entries.
     *
     * <p>An entry's size must not change while it is in the cache.
     */
    protected int sizeOf(K key, V value) {
        return 1;
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)如果是Bitmap的话我们就返回Bitmap的大小。我们来看一下LruCache的构造方法

```java
    public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)我们看到里面封装的是LinkedHashMap，最后一个参数是true，说明他的双向环形链表是按照访问顺序来存储的。在上一篇[《Android LinkedHashMap源码详解》](http://blog.csdn.net/abcdef314159/article/details/51178860)中讲到accessOrder参数的时候有提到过，我们来看一下他的一些方法，我们首先看一下public void trimToSize(int maxSize)这个方法

```java
    /**
     * Remove the eldest entries until the total of remaining entries is at or
     * below the requested size.
     *
     * @param maxSize the maximum size of the cache before returning. May be -1
     *            to evict even 0-sized elements.
     */
    public void trimToSize(int maxSize) {
        while (true) {
            K key;
            V value;
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }

                if (size <= maxSize) {
                    break;
                }

                Map.Entry<K, V> toEvict = map.eldest();
                if (toEvict == null) {
                    break;
                }

                key = toEvict.getKey();
                value = toEvict.getValue();
                map.remove(key);
                size -= safeSizeOf(key, value);
                evictionCount++;
            }

            entryRemoved(true, key, value, null);
        }
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)这个方法其实就是按照LRU算法移除最先加入的元素，也就是最少访问的老数据，直到他的size小于maxSize为止，我们看到18-20行，如果小于直接退出循环，22-25行，如果为空说明是map为null，直接退出循环。eldest()方法得到的是最老的，也是访问量最少的元素，所以根据LRU算法是最先移除的。刚开始看的时候一直不理解，感觉22-25行纯属多余，因为正常的逻辑是maxSize一般是大于0的，如果18-20行成立的话，那么22-25行肯定成立，后来看到上面的注释才明白，maxSize如果为-1,则移除全部。我们看到第29行移除，然后30行再计算一下剩余的size，然后在通过不断的循环执行到上面第19行，直到size小于maxSize才终止循环。移除的时候调用的是HashMap的remove方法，并且在remove内部调用了postRemove方法，这个方法被LinkedHashMap覆写了，移除的同时也把他从双向环形链表中删除了，我们来看一下safeSizeOf方法，其实就是调用的上面sizeOf方法

```java
    private int safeSizeOf(K key, V value) {
        int result = sizeOf(key, value);
        if (result < 0) {
            throw new IllegalStateException("Negative size: " + key + "=" + value);
        }
        return result;
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)移除的最后调用entryRemoved方法，这个方法是空方法，我们也可以在这里实现二级缓存。接着我们来看一下put方法，

```java
    public final V put(K key, V value) {
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        synchronized (this) {
            putCount++;
	    //计算size
            size += safeSizeOf(key, value);
            previous = map.put(key, value);
            if (previous != null) {
	    //我们可以看一下put方法的返回值，如果previous为null则存进去的位置之前是空的，
	    //如果previous不为null，则存进去的位置之前是有数据的，然后把他给替换了，所以
	    //这里要减去被替换掉的size
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
	//如果previous不为null，则表示之前的被移除了，调用entryRemoved方法
            entryRemoved(false, key, previous, value);
        }
	
	//重新计算size，保证最大size不能超过maxSize。
        trimToSize(maxSize);
        return previous;
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)这个方法比较简单，我们再来看一下另外一个方法public final V remove(K key)

```java
    public final V remove(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V previous;
        synchronized (this) {
            previous = map.remove(key);
            if (previous != null) {

	        //移除是不需要加size的，只需要减，如果移除成功，则减去移除的size  
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
	    //如果移除成功，则调用entryRemoved方法  
            entryRemoved(false, key, previous, null);
        }

        return previous;
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)这个方法和上面的put差不多，我们在看另外一个方法public final V get(K key)

```java
    public final V get(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V mapValue;
        synchronized (this) {
            mapValue = map.get(key);
            if (mapValue != null) {
                hitCount++;
		//如果找到则返回
                return mapValue;
            }
            missCount++;
        }

        /*
         * Attempt to create a value. This may take a long time, and the map
         * may be different when create() returns. If a conflicting value was
         * added to the map while create() was working, we leave that value in
         * the map and release the created value.
         */

	//这个方法是创建一个value，默认是返回为null，如果没有覆写则返回null直接return
        V createdValue = create(key);
        if (createdValue == null) {
            return null;
        }

        synchronized (this) {
            createCount++;
	    //这段代码当时一直搞不明白，如果上面的map.get(key)返回为null的话，就表示
	    //key所对应的value是不存在的，那么下面的map.put(key, createdValue)方法就
	    //肯定返回为null,下面也就没必要在进行判断，当我在重新看protected V create(K key)
	    //方法注释的时候，发现有这样一段描述This can occur when multiple threads request the same key
	    //at the same time (causing multiple values to be created), or when one
            //thread calls {@link #put} while another is creating a value for the samekey.意思就是如果
	    //多线程操作时可能会引起多个value被创建
            mapValue = map.put(key, createdValue);

            if (mapValue != null) {
                // There was a conflict so undo that last put
		//如果之前位置上已经有元素了，就还把原来的放回去，等于size没变
                map.put(key, mapValue);
            } else {
		//如果之前的位置上没有元素，说明createdValue是新加上去的，所以要加上createdValue的size
                size += safeSizeOf(key, createdValue);
            }
        }

        if (mapValue != null) {
	//如果之前的位置上已经有元素了，就调用entryRemoved方法，createdValue表示老的，没有存进去的，类似于
	// 删除的，
            entryRemoved(false, key, createdValue, mapValue);
            return mapValue;
        } else {
	//如果存进去之后要重新计算size的，如果大于maxSize要把最老的移除。
            trimToSize(maxSize);
            return createdValue;
        }
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)还有最后一个方法entryRemoved

```java
//entryRemoved方法是个空方法，什么都没实现，evicted如果是true则表示是为了释放空间调用的，
//主要是在trimToSize方法中调用，如果是false则一般是被put，get，remove等方法调用。
protected void entryRemoved(boolean evicted, K key, V oldValue, V newValue) {}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)OK，到目前为止LruCache的方法已经基本分析完毕，我们就来研究一下LruCache怎么使用，一般情况下我们主要用来存储bitmap的比较多，我们知道bitmap缓存的第三方框架比较多，我们就随便挑一个，我们现在最常用的volley框架一般是用来网络请求的，其实它里面也封装了图片的缓存类ImageLoader，我们看到他里面有个接口，我们看一下

```java
    /**
     * Simple cache adapter interface. If provided to the ImageLoader, it
     * will be used as an L1 cache before dispatch to Volley. Implementations
     * must not block. Implementation with an LruCache is recommended.
     */
    public interface ImageCache {
        public Bitmap getBitmap(String url);
        public void putBitmap(String url, Bitmap bitmap);
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)他有一个put和get方法，我们看上面注释的最后一行，意思是推荐使用LruCache实现。我们可以这样来实现

```java
package com.wld;

import android.graphics.Bitmap;
import android.util.LruCache;

import com.android.volley.toolbox.ImageLoader.ImageCache;

public class BitmapLRUCache implements ImageCache {

	private LruCache<String, Bitmap> mBitmapCache;

	public BitmapLRUCache() {
		this((int) Runtime.getRuntime().maxMemory() / 4);
	}

	public BitmapLRUCache(int maxSize) {
		mBitmapCache = new LruCache<String, Bitmap>(maxSize) {
			@Override
			protected int sizeOf(String key, Bitmap bitmap) {
				return bitmap.getRowBytes() * bitmap.getHeight();
			}
		};
	}

	private void entryRemoved() {
		//这里可以实现二级缓存
	}

	@Override
	public Bitmap getBitmap(String url) {
		return mBitmapCache.get(url);
	}

	@Override
	public void putBitmap(String url, Bitmap bitmap) {
		mBitmapCache.put(url, bitmap);
	}
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)上面的entryRemoved方法其实是可以实现二级缓存的，我们可以在手机的SD中开辟一块空间，用来保存上面entryRemoved方法中移除的图片。逻辑是这样的，当我们需要图片的时候首先从LruCache中取，如果没有就从SD中取，如果SD卡中也没有就从网络上下载，下载完之后就保存到LruCache中，如果LruCache中图片大小达到上限或者调用remove方法就会执行LruCache的entryRemoved方法，在entryRemoved方法中我们可以把移除的图片存储到SD卡中，这样下一次取的时候还是按照这个顺序来取。网上还有一些实现方法和上面的区别是他没有重写entryRemoved方法，当从网上下载完的时候，在LruCache和SD中都保存了一份，这样做也是可以的。我目前没有发现google的实现SD卡缓存的类，不过第三方的倒是有很多，比如thinkandroid的DiskLruCache类，SmartAndroid的LruDiscCache类还有KJFrameForAndroid的DiskCache类都实现了磁盘缓存。大家有兴趣的话可以自己去看一下，这里就不在介绍。