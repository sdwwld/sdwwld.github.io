---
layout: post
title: "Android PathMeasure的使用详解"
subtitle: 'Android PathMeasure的使用详解'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “流水下滩非有意，白云出岫本无心。”

## 正文

在上一篇的<a href="https://androidboke.com/2016/10/13/Android-Path的使用详解" target="_blank">Android Path的使用详解</a>中分析了Path类的一些常用的方法，下面顺便看一下和他有关联的一个类PathMeasure，顾名思义，就是测量Path的，来先看一下注释

## PathMeasure

```java
    /**
     * Create an empty PathMeasure object. To uses this to measure the length
     * of a path, and/or to find the position and tangent along it, call
     * setPath.
     *
     * Note that once a path is associated with the measure object, it is
     * undefined if the path is subsequently modified and the the measure object
     * is used. If the path is modified, you must call setPath with the path.
     */
    public PathMeasure() {
        mPath = null;
        native_instance = native_create(0, false);
    }
```

为了测量他的长度或切线需要先调用setPath方法。如果path修改了，需要测量，则必须再次调用setPath方法。再来看一下另外的一个构造方法

```java
    /**
     * Create a PathMeasure object associated with the specified path object
     * (already created and specified). The measure object can now return the
     * path's length, and the position and tangent of any position along the
     * path.
     *
     * Note that once a path is associated with the measure object, it is
     * undefined if the path is subsequently modified and the the measure object
     * is used. If the path is modified, you must call setPath with the path.
     *
     * @param path The path that will be measured by this object
     * @param forceClosed If true, then the path will be considered as "closed"
     *        even if its contour was not explicitly closed.
     */
    public PathMeasure(Path path, boolean forceClosed) {
        // The native implementation does not copy the path, prevent it from being GC'd
        mPath = path;
        native_instance = native_create(path != null ? path.ni() : 0,
                                        forceClosed);
    }
```

其中forceClosed表示测量的时候是否强制关闭，只是作为一个测量的值，但并不会关闭path，看一下代码

## PathView

```java
package test.view;

import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.graphics.Paint.Style;
import android.graphics.Path;
import android.graphics.PathMeasure;
import android.util.AttributeSet;
import android.util.Log;
import android.view.View;

public class PathView extends View {
	private Paint mPaint;
	private Path mPath;

	public PathView(Context context, AttributeSet attrs) {
		super(context, attrs);
		init();
	}

	private void init() {
		mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
		mPaint.setColor(Color.RED);
		mPaint.setStyle(Style.STROKE);
		mPaint.setStrokeWidth(8);
		mPath = new Path();
		mPath.moveTo(100, 100);
		mPath.lineTo(400, 400);
		mPath.lineTo(600, 200);
		Log.d("wld_____", new PathMeasure(mPath, false).getLength() + "");
	}

	@Override
	protected void onDraw(Canvas canvas) {
		canvas.drawPath(mPath, mPaint);

	}
}
```

看一下运行结果

![](/img/blog/2016/20161013140218627.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

再看一下打印log

![](/img/blog/2016/20161013135606452.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

修改一下

```java
Log.d("wld_____", new PathMeasure(mPath, true).getLength() + "");
```

看一下结果

![](/img/blog/2016/20161013140357473.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

在看一下打印log

![](/img/blog/2016/20161013140440002.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

可以看到两个图图形虽然没有变，但测量长度变了。

setPath(Path path, boolean forceClosed)方法和上面的构造方法其实差不多。

getLength()返回path当前轮廓的长度。

getPosTan(float distance, float pos[], float tan[])其中的distance(0<=distance<=getLength())，pos是在长度为distance的点的坐标，其中(x==[0], y==[1])，tan是在那个点的切线的值，其中(x==[0], y==[1])，返回true数据会存入 pos 和 tan 中，返回false，pos 和 tan 不会改变

getMatrix(float distance, Matrix matrix, int flags)得到路径上某一长度的位置以及该位置正切值的矩阵，flags有两种类型

```java
    public static final int POSITION_MATRIX_FLAG = 0x01;    // must match flags in SkPathMeasure.h
    public static final int TANGENT_MATRIX_FLAG  = 0x02;    // must match flags in SkPathMeasure.h
```

一个表示位置，一个表示正切。

getSegment(float startD, float stopD, Path dst, boolean startWithMoveTo)根据开始和结束的长度得到path中的一个片段，startWithMoveTo为true表示截取到的位置不会改变，为false表示截取到的会连接在一起，看一下代码。

## PathView

```java
package test.view;

import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.graphics.Paint.Style;
import android.graphics.Path;
import android.graphics.PathMeasure;
import android.util.AttributeSet;
import android.view.View;

public class PathView extends View {
	private Paint mPaint;;
	private Path dst;

	public PathView(Context context, AttributeSet attrs) {
		super(context, attrs);
		init();
	}

	private void init() {
		mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
		mPaint.setStyle(Style.STROKE);
		mPaint.setStrokeWidth(12);
		mPaint.setColor(Color.RED);
		dst = new Path();

		Path mPath1 = new Path();
		mPath1.addRect(200, 200, 600, 600, Path.Direction.CW);
		PathMeasure measure1 = new PathMeasure(mPath1, false);
		measure1.getSegment(200, 600, dst, true);

		Path mPath2 = new Path();
		mPath2.addRect(200, 1000, 600, 1400, Path.Direction.CW);
		PathMeasure measure = new PathMeasure(mPath2, false);
		measure.getSegment(200, 900, dst, true);

	}

	@Override
	protected void onDraw(Canvas canvas) {
		canvas.drawPath(dst, mPaint);
	}
}
```

看一下运行结果

![](/img/blog/2016/20161013152234692.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)
截取的两段是互不干涉的，下面修改一下

```java
		Path mPath1 = new Path();
		mPath1.addRect(200, 200, 600, 600, Path.Direction.CW);
		PathMeasure measure1 = new PathMeasure(mPath1, false);
		measure1.getSegment(200, 600, dst, false);

		Path mPath2 = new Path();
		mPath2.addRect(200, 1000, 600, 1400, Path.Direction.CW);
		PathMeasure measure = new PathMeasure(mPath2, false);
		measure.getSegment(200, 900, dst, false);
```

看一下结果

![](/img/blog/2016/20161013152547521.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

他会找到上一个截取的来进行连接，第一个没有上一个所以连接的为初始位置（0,0）。
isClosed()返回这个路径是否是关闭的。

nextContour()移动到下一个轮廓，因为path可能会有多个轮廓。