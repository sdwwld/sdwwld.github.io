---
layout: post
title: "Android onTouchEvent和onInterceptTouchEvent事件分发详解(一)"
subtitle: 'Android onTouchEvent和onInterceptTouchEvent事件分发详解'
author: "山大王"
header-style: text
catalog: true
tags:
  - 用法
  - android
---
> “古人学问无遗力，少壮工夫老始成。”
	--陆游

## 正文

在讲解之前，先看一下demo

```
package com.example.androiddemo;
 
import com.example.androiddemo.dispatch.LinearLayoutChild;
import com.example.androiddemo.dispatch.LinearLayoutParent;
import com.example.androiddemo.dispatch.TextView_Child;
import com.example.androiddemo.utils.Util;
 
import android.app.Activity;
import android.os.Bundle;
import android.util.Log;
import android.view.Gravity;
import android.view.MotionEvent;
import android.view.View;
import android.view.ViewGroup;
import android.view.ViewGroup.LayoutParams;
 
public class DispatchActivity extends Activity {
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(initView());
	}
 
	private View initView() {
		TextView_Child mTextView_Child = new TextView_Child(this);
		ViewGroup.LayoutParams params0 = new LayoutParams(getWindowManager()
				.getDefaultDisplay().getWidth() / 2, getWindowManager()
				.getDefaultDisplay().getHeight() / 4);
		mTextView_Child.setLayoutParams(params0);
		mTextView_Child.setBackgroundColor(getResources().getColor(
				android.R.color.holo_orange_dark));
 
		LinearLayoutChild mLinearLayoutChild = new LinearLayoutChild(this);
		mLinearLayoutChild.addView(mTextView_Child);
		mLinearLayoutChild.setBackgroundColor(getResources().getColor(
				android.R.color.holo_red_dark));
		ViewGroup.LayoutParams params1 = new LayoutParams(
				LayoutParams.WRAP_CONTENT, getWindowManager()
						.getDefaultDisplay().getHeight() / 2);
		mLinearLayoutChild.setLayoutParams(params1);
		mLinearLayoutChild.setGravity(Gravity.CENTER);
 
		LinearLayoutParent mLinearLayoutParent = new LinearLayoutParent(this);
		mLinearLayoutParent.addView(mLinearLayoutChild);
		mLinearLayoutParent.setBackgroundColor(getResources().getColor(
				android.R.color.holo_green_dark));
		mLinearLayoutParent.setGravity(Gravity.CENTER);
		return mLinearLayoutParent;
	}
 
	@Override
	public boolean dispatchTouchEvent(MotionEvent ev) {
		boolean touch = super.dispatchTouchEvent(ev);
		Log.d("wld", "DispatchActivity_dispatchTouchEvent" + "_" + touch);
		Util.printEventLog("DispatchActivity", "dispatchTouchEvent", ev);
		return touch;
	}
 
	@Override
	public boolean onTouchEvent(MotionEvent event) {
		boolean touch = super.onTouchEvent(event);
		Log.d("wld", "DispatchActivity_onTouchEvent" + "_" + touch);
		Util.printEventLog("DispatchActivity", "onTouchEvent", event);
		return touch;
	}
}
```

```
package com.example.androiddemo.utils;
 
import android.util.Log;
import android.view.MotionEvent;
 
public class Util {
 
	public static void printEventLog(String className, String methodName,
			MotionEvent event) {
		switch (event.getAction()) {
		case MotionEvent.ACTION_DOWN:
			Log.d("wld", className + "_" + methodName + "_" + "ACTION_DOWN");
			break;
		case MotionEvent.ACTION_MOVE:
			Log.d("wld", className + "_" + methodName + "_" + "ACTION_MOVE");
			break;
		case MotionEvent.ACTION_UP:
			Log.d("wld", className + "_" + methodName + "_" + "ACTION_UP");
			break;
		case MotionEvent.ACTION_CANCEL:
			Log.d("wld", className + "_" + methodName + "_" + "ACTION_CANCEL");
			break;
		default:
			break;
		}
	}
}
```

```
package com.example.androiddemo.dispatch;
 
import com.example.androiddemo.utils.Util;
 
import android.content.Context;
import android.util.AttributeSet;
import android.util.Log;
import android.view.MotionEvent;
import android.widget.LinearLayout;
 
public class LinearLayoutParent extends LinearLayout {
 
	public LinearLayoutParent(Context context) {
		super(context);
	}
 
	public LinearLayoutParent(Context context, AttributeSet attrs) {
		super(context, attrs);
	}
 
	public LinearLayoutParent(Context context, AttributeSet attrs, int defStyle) {
		super(context, attrs, defStyle);
	}
 
	@Override
	public boolean dispatchTouchEvent(MotionEvent ev) {
		boolean touch = super.dispatchTouchEvent(ev);
		Log.d("wld", "LinearLayoutParent_dispatchTouchEvent" + "_" + touch);
		Util.printEventLog("LinearLayoutParent", "dispatchTouchEvent", ev);
		return touch;
	}
 
	@Override
	public boolean onTouchEvent(MotionEvent event) {
		boolean touch = super.onTouchEvent(event);
		Log.d("wld", "LinearLayoutParent_onTouchEvent" + "_" + touch);
		Util.printEventLog("LinearLayoutParent", "onTouchEvent", event);
		return touch;
	}
 
	@Override
	public boolean onInterceptTouchEvent(MotionEvent ev) {
		boolean touch = super.onInterceptTouchEvent(ev);
		Log.d("wld", "LinearLayoutParent_onInterceptTouchEvent" + "_" + touch);
		Util.printEventLog("LinearLayoutParent", "onInterceptTouchEvent", ev);
		return touch;
	}
 
}
```

```
package com.example.androiddemo.dispatch;
 
import com.example.androiddemo.utils.Util;
 
import android.content.Context;
import android.util.AttributeSet;
import android.util.Log;
import android.view.MotionEvent;
import android.widget.LinearLayout;
 
public class LinearLayoutChild extends LinearLayout {
 
	public LinearLayoutChild(Context context) {
		super(context);
	}
 
	public LinearLayoutChild(Context context, AttributeSet attrs) {
		super(context, attrs);
	}
 
	public LinearLayoutChild(Context context, AttributeSet attrs, int defStyle) {
		super(context, attrs, defStyle);
	}
 
	@Override
	public boolean dispatchTouchEvent(MotionEvent ev) {
		boolean touch = super.dispatchTouchEvent(ev);
		Log.d("wld", "LinearLayoutChild_dispatchTouchEvent" + "_" + touch);
		Util.printEventLog("LinearLayoutChild", "dispatchTouchEvent", ev);
		return touch;
	}
 
	@Override
	public boolean onTouchEvent(MotionEvent event) {
		boolean touch = super.onTouchEvent(event);
		Log.d("wld", "LinearLayoutChild_onTouchEvent" + "_" + touch);
		Util.printEventLog("LinearLayoutChild", "onTouchEvent", event);
		return touch;
	}
 
	@Override
	public boolean onInterceptTouchEvent(MotionEvent ev) {
		boolean touch = super.onInterceptTouchEvent(ev);
		Log.d("wld", "LinearLayoutChild_onInterceptTouchEvent" + "_" + touch);
		Util.printEventLog("LinearLayoutChild", "onInterceptTouchEvent", ev);
		return touch;
	}
}
```

```
package com.example.androiddemo.dispatch;
 
import android.content.Context;
import android.util.AttributeSet;
import android.util.Log;
import android.view.MotionEvent;
import android.view.View;
import android.widget.ImageButton;
import android.widget.ImageView;
import android.widget.LinearLayout;
import android.widget.TextView;
 
import com.example.androiddemo.utils.Util;
 
public class TextView_Child extends TextView {
 
	public TextView_Child(Context context) {
		super(context);
	}
 
	public TextView_Child(Context context, AttributeSet attrs) {
		super(context, attrs);
	}
 
	public TextView_Child(Context context, AttributeSet attrs, int defStyle) {
		super(context, attrs, defStyle);
	}
 
	@Override
	public boolean dispatchTouchEvent(MotionEvent ev) {
		boolean touch = super.dispatchTouchEvent(ev);
		Log.d("wld", "TextViewChild_dispatchTouchEvent" + "_" + touch);
		Util.printEventLog("TextViewChild", "dispatchTouchEvent", ev);
		return touch;
	}
 
	@Override
	public boolean onTouchEvent(MotionEvent event) {
		boolean touch = super.onTouchEvent(event);
		Log.d("wld", "TextViewChild_onTouchEvent" + "_" + touch);
		Util.printEventLog("TextViewChild", "onTouchEvent", event);
		return touch;
	}
 
}
```

上面所有的返回值都是默认的，我们来看一下运行结果

![](/img/blog/2016/20160408155359170.png)

我们点击中间的橘色，再来看一下打印的log 

![](/img/blog/2016/20160408160034907.jpg)

我们看到，默认情况下，全部返回false，并且DispatchActivity的onTouchEvent方法执行了DOWN,MOVE,UP方法，也就说是默认情况下，事件全部被DispatchActivity的onTouchEvent方法给处理了。上面仅仅是一个演示，其实用处并不是很大，只是作为一个了解，但对我们研究Android的事件分发机制有一定的意义，因为我们如果拦截或者处理事件机制的时候一般是在View及其子类中处理。接下来我们会把DispatchActivity中的两个方法dispatchTouchEvent和onTouchEvent注释掉，我们只研究View及其子类中的Event方法。我们知道除了Button及其ImageButton的onTouchEvent方法默认返回true以外，其他的都是返回false，因为Button和ImageButton默认情况下是消耗事件的，刚才上面我们看到TextView的onTouchEvent方法默认情况下返回的是false，就是事件没有被他消耗掉，下面我们把TextView换成Button在看一看，

```
public class TextView_Child extends Button {
```

顺便注释掉DispatchActivity中的dispatchTouchEvent和onTouchEvent方法，我们看一下log

![](/img/blog/2016/20160408173633508.png)

![](/img/blog/2016/20160408173712133.png)

![](/img/blog/2016/20160408173755852.png)

![](/img/blog/2016/20160408173834930.png)

我们可以看到默认情况下Button的onTouchEvent返回的是true，因为点击的时候事件默认被他消耗掉了。好了，这里只是给大家简单的开个头，接下来就会为大家讲解一下View事件的拦截及分发机制，最后在给大家分析源码。