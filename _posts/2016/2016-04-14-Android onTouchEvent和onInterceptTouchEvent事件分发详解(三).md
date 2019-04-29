---
layout: post
title: "Android onTouchEvent和onInterceptTouchEvent事件分发详解(三)"
subtitle: 'Android onTouchEvent和onInterceptTouchEvent事件分发详解'
author: "山大王"
header-style: text
catalog: true
tags:
  - 用法
  - android
---
> “花门楼前见秋草，岂能贫贱相看老。”
	--岑参

## 正文

紧接前一篇几个问题的验证，在看之前最好把上一篇的<a href="https://androidboke.com/2016/04/11/Android-onTouchEvent%E5%92%8ConInterceptTouchEvent%E4%BA%8B%E4%BB%B6%E5%88%86%E5%8F%91%E8%AF%A6%E8%A7%A3(%E4%BA%8C)" target="_blank">Android onTouchEvent和onInterceptTouchEvent事件分发详解(二)</a>先看一下。  
在上一篇我们根据源码分析了Android事件的分发机制，在最后总结了几个问题，在这一篇我们将为大家逐一验证。  
总共有3个类，一个Activity，一个ViewGroup，一个View

## （1）在Activity中的代码为

```
	@Override
	public boolean dispatchTouchEvent(MotionEvent ev) {
		Log.d("wld",
				"DispatchActivity_dispatchTouchEvent" + "_"
						+ MotionEvent.actionToString(ev.getAction()));
		return super.dispatchTouchEvent(ev);
	}
 
	@Override
	public boolean onTouchEvent(MotionEvent event) {
		Log.d("wld",
				"DispatchActivity_onTouchEvent" + "_"
						+ MotionEvent.actionToString(event.getAction()));
		return super.onTouchEvent(event);
	}
```

## ViewGroup中

```
	@Override
	public boolean dispatchTouchEvent(MotionEvent ev) {
		Log.d("wld", "LinearLayoutParent_dispatchTouchEvent" + "_"
				+ MotionEvent.actionToString(ev.getAction()));
		return super.dispatchTouchEvent(ev);
	}
 
	@Override
	public boolean onTouchEvent(MotionEvent event) {
		Log.d("wld",
				"LinearLayoutParent_onTouchEvent" + "_"
						+ MotionEvent.actionToString(event.getAction()));
		return super.onTouchEvent(event);
	}
 
	@Override
	public boolean onInterceptTouchEvent(MotionEvent ev) {
		Log.d("wld", "LinearLayoutParent_onInterceptTouchEvent" + "_"
				+ MotionEvent.actionToString(ev.getAction()));
		return super.onInterceptTouchEvent(ev);
	}
```

## View中的代码（View为TextView）

```
	@Override
	public boolean dispatchTouchEvent(MotionEvent ev) {
		Log.d("wld",
				"TextViewChild_dispatchTouchEvent" + "_"
						+ MotionEvent.actionToString(ev.getAction()));
		return super.dispatchTouchEvent(ev);
	}
 
	@Override
	public boolean onTouchEvent(MotionEvent event) {
		Log.d("wld",
				"TextViewChild_onTouchEvent" + "_"
						+ MotionEvent.actionToString(event.getAction()));
		return super.onTouchEvent(event);
	}
```

我们运行一下，看一下结果

![](/img/blog/2016/20160414165254960.jpg)

我们知道TextView默认是不消耗事件的，我们看一下打印的log

![](/img/blog/2016/20160414165625043.jpg)

DOWN和UP只会触发一次，MOVE可能会触发很多次，我们知道就行了，从上面可以看到，默认情况下TextView是不消耗事件的，只是触发了DOWN事件，后面的MOVE，UP都没有触发，且传递顺序是从Activity的dispatchTouchEvent→ViewGroup的dispatchTouchEvent→ViewGroup的onInterceptTouchEvent→View的dispatchTouchEvent→View的onTouchEvent（返回false往上抛）→ViewGroup的onTouchEvent（返回false表示事件没消耗，往上抛）→Activity的onTouchEvent（处理了事件，执行了MOVE，UP事件），此后会一直交给Activity处理，不在往下分发。  
（2）我们让TextView中的onTouchEvent方法返回为true，在看一下执行结果

```
	@Override
	public boolean onTouchEvent(MotionEvent event) {
		Log.d("wld",
				"TextViewChild_onTouchEvent" + "_"
						+ MotionEvent.actionToString(event.getAction()));
		super.onTouchEvent(event);
		return true;
	}
```

打印log如下

![](/img/blog/2016/20160415112637577.png)

我们看到TextView的onTouchEvent消耗了事件（执行了DOWM,MOVE,UP事件），就不会再往上抛，所以ViewGroup的onTouchEvent和Activity的onTouchEvent就不会再触发，此后所有的事件都会交由它来处理。  
（3）我们在看另外一种情况，把上面TextView的onTouchEvent返回默认值（false）让ViewGroup的onTouchEvent放回true，

## onInterceptTouchEvent

```
	@Override
	public boolean onTouchEvent(MotionEvent event) {
		Log.d("wld",
				"LinearLayoutParent_onTouchEvent" + "_"
						+ MotionEvent.actionToString(event.getAction()));
		super.onTouchEvent(event);
		return true;
	}
```

我们看一下打印的log

![](/img/blog/2016/20160415130701960.png)

我们看到TextView只执行了DOWM事件，因为返回的是false，表示事件没有被消耗，就会往上抛，交给ViewGroup，但ViewGroup的onTouchEvent返回的是true，表示事件被他消耗了，就不会再往上抛，所以就执行了后面的MOVE，UP事件，此后的一系列事件也不会再往下传递了，每次传递到ViewGroup的时候就直接调用它的onTouchEvent方法。  
（4）我们在（3）的基础上，让onInterceptTouchEvent返回true，在看一下

## onInterceptTouchEvent

```
	@Override
	public boolean onInterceptTouchEvent(MotionEvent ev) {
		Log.d("wld", "LinearLayoutParent_onInterceptTouchEvent" + "_"
				+ MotionEvent.actionToString(ev.getAction()));
		super.onInterceptTouchEvent(ev);
		return true;
	}
```

我们看一下打印的log

![](/img/blog/2016/20160415131953283.png)

我们看到TextView的方法根本就没有执行，这时TextView的方法无论放回true还是false都没有任何影响，因为ViewGroup的onInterceptTouchEvent方法返回true，表示事件被他拦截，就不会再往下传递，我们还看到ViewGroup的onInterceptTouchEvent方法只执行了一次，因为已经被拦截了，后续的一系列事件就都会交给他，就不需要在拦截了，因为ViewGroup的onTouchEvent返回true，事件被他消耗，就不在往上抛。  
（5）我们在（4）的基础上，让onTouchEvent返回false，在看一下打印的log

![](/img/blog/2016/20160415132811271.png)

因为ViewGroup拦截，所以没有传递到TextView，因为ViewGroup没有消耗事件，所以往上抛到Activity，所以事件最终被Activity给处理了（执行了MOVE，UP事件）  
（6）关于requestDisallowInterceptTouchEvent事件，这里我们再来研究一下，我们让所有的都返回默认，然后让ViewGroup的onInterceptTouchEvent方法返回true（表示拦截，默认情况下就不会往下传递），我们让TextView的onTouchEvent方法返回true（表示事件被他消耗），然后改写TextView的dispatchTouchEvent方法

```
	@Override
	public boolean dispatchTouchEvent(MotionEvent ev) {
		getParent().requestDisallowInterceptTouchEvent(true);
		Log.d("wld",
				"TextViewChild_dispatchTouchEvent" + "_"
						+ MotionEvent.actionToString(ev.getAction()));
		return super.dispatchTouchEvent(ev);
	}
```

我们看一下log

![](/img/blog/2016/20160415144025425.png)

发现很奇葩的事，TextView的方法并没有被调用，我们看一下源码就会发现，在ViewGroup的dispatchTouchEvent方法中有这样一段代码

```
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }
```

表示在按下的时候调用了resetTouchState方法，我们再看一下这个方法  

## resetTouchState

```
    private void resetTouchState() {
        clearTouchTargets();
        resetCancelNextUpFlag(this);
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    }
```

看到没，在最后一行把FLAG_DISALLOW_INTERCEPT标记清除了，就是说每次点击的时候，执行DOWN事件的时候就会把它清除，由于后面的onInterceptTouchEvent返回true，所以TextView的方法根本就执行不了，我们把ViewGroup的onInterceptTouchEvent方法改一下

```
	@Override
	public boolean onInterceptTouchEvent(MotionEvent ev) {
		Log.d("wld", "LinearLayoutParent_onInterceptTouchEvent" + "_"
				+ MotionEvent.actionToString(ev.getAction()));
		// super.onInterceptTouchEvent(ev);
		if (ev.getAction() == MotionEvent.ACTION_DOWN)
			return false;
		return true;
	}
```

然后再看一下log

![](/img/blog/2016/20160415144947647.png)

我们看到TextView的所有方法都被执行了，虽然在ViewGroup的onInterceptTouchEvent方法中除了DOWN以外都被拦截了，但我们还是能看到TextView消耗了事件，因为在我们在TextView的dispatchTouchEvent方法中调用了requestDisallowInterceptTouchEvent方法（不让父类拦截事件），我们把TextView中的requestDisallowInterceptTouchEvent方法去掉再看一下

![](/img/blog/2016/20160415150355903.png)

我们看到TextView只执行了DOWN和CANCEL事件，MOVE和UP事件都被ViewGroup拦截了，由于没有消耗，直接往上抛，交给Activity处理。且ViewGroup的onInterceptTouchEvent方法在UP事件的时候也没有执行，因为在执行MOVE的时候就已经返回了True，所以后面的UP事件就不会再触发。  
OK，到目前为止，Activity的事件拦截机制就分析完了。