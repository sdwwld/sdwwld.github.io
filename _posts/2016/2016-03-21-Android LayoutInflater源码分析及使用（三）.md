---
layout: post
title: "Android LayoutInflater源码分析及使用（三）"
subtitle: 'Android LayoutInflater源码分析及使用'
author: "山大王"
header-style: text
catalog: true
tags:
  - 用法
  - android
---
> “时人不识凌云木，直待凌云始道高。”
	--杜荀鹤

## 正文

上一篇最后我们总结了3个问题，但还没有验证，这一篇我们将逐个为大家验证，下面是一些关键代码

```
package com.example.androiddemo;
 
import android.app.Activity;
import android.os.Bundle;
import android.view.View;
import android.widget.FrameLayout;
 
import com.example.androiddemo.view.LayoutInflaterTestView;
 
public class LayoutInflaterTestActivity extends Activity {
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		
		View mainView = View.inflate(this,
				R.layout.activity_layout_inflater_main, null);
		LayoutInflaterTestView ll_v = (LayoutInflaterTestView) mainView
				.findViewById(R.id.ll_);
		View mView = ll_v.getLayoutView();
		// 在外面再嵌套一层，因为如果不嵌套的话mView的Params将失去作用。待会我们把上一篇总结的都验证完之后，我在为大家分析。
		FrameLayout rl = new FrameLayout(this);
		rl.addView(mView);
		
		setContentView(rl);
	}
}
```

```
package com.example.androiddemo.view;
 
import android.content.Context;
import android.util.AttributeSet;
import android.util.Log;
import android.view.LayoutInflater;
import android.view.View;
import android.widget.LinearLayout;
 
import com.example.androiddemo.R;
 
public class LayoutInflaterTestView extends LinearLayout {
	private View layoutView;
 
	public LayoutInflaterTestView(Context context, AttributeSet attrs) {
		super(context, attrs);
		init(context);
	}
 
	public LayoutInflaterTestView(Context context, AttributeSet attrs,
			int defStyleAttr) {
		super(context, attrs, defStyleAttr);
		init(context);
	}
 
	private void init(Context mContext) {
		if (isInEditMode())
			return;
		LayoutInflater mInflater = (LayoutInflater) mContext
				.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
		layoutView = (View) mInflater.inflate(
				R.layout.activity_layout_inflater_test, null, false);
		// 打印layoutView类型，
		Log.d("wld____________", layoutView + "");
	}
 
	public View getLayoutView() {
		return layoutView;
	}
}
```

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="200dp"
    android:background="@android:color/holo_green_dark" >
 
    <TextView
        android:id="@+id/tv_"
        android:layout_width="200dp"
        android:layout_height="100dp"
        android:background="@android:color/holo_orange_dark"
        android:gravity="center"
        android:text="LaoutInflate"
        android:textColor="@android:color/black"
        android:textSize="20sp" />
 
</RelativeLayout>
```

```
<?xml version="1.0" encoding="utf-8"?>
<com.example.androiddemo.view.LayoutInflaterTestView xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/ll_"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@android:color/holo_red_dark"
    android:orientation="vertical" >
 
</com.example.androiddemo.view.LayoutInflaterTestView>
```

验证之前我们认真看一下上面的background，分别是绿色，橘色和红色，由于比较多，第一个问题就不再验证了，我们来验证第二个问题，就是如果root为空，attachToRoot无论是true或false都一样，根布局的属性保留下来，但宽高没有保留，我们看一下

![](/img/blog/2016/20160321104957181.png)

看到没，加载的layout的背景色green保存了下来，但是他的宽高却没有保留，这时候就算我们把它的高度改为0，显示结果还是和上面一样，我们可以看一下，把RelativeLayout的高改为0

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="0dp"
    android:background="@android:color/holo_green_dark" >
 
    <TextView
        android:id="@+id/tv_"
        android:layout_width="200dp"
        android:layout_height="100dp"
        android:background="@android:color/holo_orange_dark"
        android:gravity="center"
        android:text="LaoutInflate"
        android:textColor="@android:color/black"
        android:textSize="20sp" />
 
</RelativeLayout>
```

![](/img/blog/2016/20160321105728551.png)

看到没，结果完全一样，原因在于它通过attrs创建实例的时候，它的背景属性是可以保留的，但他的宽高是不行的，我们再看打印的log

![](/img/blog/2016/20160321110118594.png)

根布局的根View是RelativeLayout，而不是我们的LayoutInflaterTestView，所以完全证实了上一篇我们总结的，至于inflate函数第三个参数为true的时候也一样，大家可以自己去验证，这里就不在介绍。我们在看第三个问题root不为空，且attachToRoot为true

## init

```
	private void init(Context mContext) {
		if (isInEditMode())
			return;
		LayoutInflater mInflater = (LayoutInflater) mContext
				.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
		layoutView = (View) mInflater.inflate(
				R.layout.activity_layout_inflater_test, this, true);
		// 打印layoutView类型，
		Log.d("wld____________", layoutView + "");
	}
```

还把原来的高改为200dp，我们看一下，顺便我们在看一下log

![](/img/blog/2016/20160321111336118.png)

![](/img/blog/2016/20160321142554898.png)

通过上面我们看到，根布局的高为200dp，也就是说根布局的所有属性包括宽高都保留了下来，并且根布局的根View是LayoutInflaterTestView，所以也验证了上一篇我们的分析是正确的，我们再看下一个问题，root不为空，且attachToRoot为false，

```
	private void init(Context mContext) {
		if (isInEditMode())
			return;
		LayoutInflater mInflater = (LayoutInflater) mContext
				.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
		layoutView = (View) mInflater.inflate(
				R.layout.activity_layout_inflater_test, this, false);
		// 打印layoutView类型，
		Log.d("wld____________", layoutView + "");
	}
```

我们看一下最终显示结果和和打印的log

![](/img/blog/2016/20160321143624806.png)

![](/img/blog/2016/20160321143738447.png)

正像我们所分析的那样，它的所有属性包括宽高全部都保留了下来，但没有add到LayoutInflaterTestView中，所以他的根布局的根View不是LayoutInflaterTestView。

OK，经过这一篇的验证，我们上一篇所分析的全部是正确的，到目前为止LayoutInflater源码已经被我们全部分析完毕。我们来解决最上面遗留的一个问题，就是在LayoutInflaterTestActivity这个类中我们为什么要嵌套一层FrameLayout，假如我们不嵌套的话，会有什么结果，我们借着上面的结论还是root不为空，attachToRoot为false来看一下

## LayoutInflaterTestActivity

```
public class LayoutInflaterTestActivity extends Activity {
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
 
		View mainView = View.inflate(this,
				R.layout.activity_layout_inflater_main, null);
		LayoutInflaterTestView ll_v = (LayoutInflaterTestView) mainView
				.findViewById(R.id.ll_);
		View mView = ll_v.getLayoutView();
		// 在外面再嵌套一层，因为如果不嵌套的话mView的Params将失去作用。待会我们把上一篇总结的都验证完之后，我在为大家分析。
		// FrameLayout rl = new FrameLayout(this);
		// rl.addView(mView);
		// setContentView(rl);
		Log.d("wld______", mView.getLayoutParams().height + "");
		setContentView(mView);
		Log.d("wld______", mView.getLayoutParams().height + "");
	}
}
```

![](C:\boke\sdwwld.github.io\img\blog\2016\20160321145120499.png)

结果完全出乎我们的意料，竟然和我们上面验证的完全不同，究竟是为什么呢，我们来看一下上面打印的log

![](C:\boke\sdwwld.github.io\img\blog\2016\20160321145430031.png)

高度明明是600，可为什么最后变成-1了呢，这就是为什么要在外面在嵌套一层的原因。最终怎么会变成这样的我们只有通过源码来分析，我们知道高度我们设置的明明是200dp，可我们打印的是600，因为这个600是px，我们也可以通过类TypedValue的下面方法进行转换，其实效果是一样的。

## applyDimension

```
    public static float applyDimension(int unit, float value,
                                       DisplayMetrics metrics)
    {
        switch (unit) {
        case COMPLEX_UNIT_PX:
            return value;
        case COMPLEX_UNIT_DIP:
            return value * metrics.density;
        case COMPLEX_UNIT_SP:
            return value * metrics.scaledDensity;
        case COMPLEX_UNIT_PT:
            return value * metrics.xdpi * (1.0f/72);
        case COMPLEX_UNIT_IN:
            return value * metrics.xdpi;
        case COMPLEX_UNIT_MM:
            return value * metrics.xdpi * (1.0f/25.4f);
        }
        return 0;
    }
```

我们来分析那个600变成-1的情况，在之前我们分析Activity的setContentView方法的时候，我们知道他调用的是PhoneWindow的setContentView方法，这个方法被重构了3次，我们找到我们需要的，然后查看他的源码

## setContentView

```
    @Override
    public void setContentView(View view) {
        setContentView(view, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
    }
```

看到没，在我们setContentView的时候它又加了一个LayoutParams参数，就是说如果我们最外面不嵌套一层，我们之前的宽高没有任何作用，我们来看一下MATCH_PARENT的值

```
        /**
         * Special value for the height or width requested by a View.
         * MATCH_PARENT means that the view wants to be as big as its parent,
         * minus the parent's padding, if any. Introduced in API Level 8.
         */
        public static final int MATCH_PARENT = -1;
```

看到没，是-1，和我们之前打印的log完全一致，好了，所有问题我们都已分析完毕，今天就先到这吧