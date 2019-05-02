---
layout: post
title: "Android Path的使用详解"
subtitle: 'Android Path的使用详解'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “当时若不登高望，谁信东流海洋深？”

## 正文

这里来分析一下Android自定义控件中比较常用的另一个类Path

```java
/**
 * The Path class encapsulates compound (multiple contour) geometric paths
 * consisting of straight line segments, quadratic curves, and cubic curves.
 * It can be drawn with canvas.drawPath(path, paint), either filled or stroked
 * (based on the paint's Style), or it can be used for clipping or to draw
 * text on a path.
 */
```

从注释中可以看出，Path类封装了一些复合的几何路径，其中包括直线，二次曲线，三次曲线等，我们来看一下他的源代码，其中有一个内部类Direction

## Direction

```java
    /**
     * Specifies how closed shapes (e.g. rects, ovals) are oriented when they
     * are added to a path.
     */
    public enum Direction {
        /** clockwise */
        CW  (1),    // must match enum in SkPath.h
        /** counter-clockwise */
        CCW (2);    // must match enum in SkPath.h

        Direction(int ni) {
            nativeInt = ni;
        }
        final int nativeInt;
    }
```

是个枚举类型，一个表示顺时针方向，一个表示逆时针方向。注释中说是指关闭图形的方向，例如矩形和椭圆形，但是这个有点意思，因为几何图形中无论是矩形还是椭圆形只要固定的点确定了，图形基本上就确定了，我们来看一下代码

```java
package test.view;

import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.graphics.Paint.Style;
import android.graphics.Path;
import android.graphics.Path.Direction;
import android.util.AttributeSet;
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
		mPaint.setStyle(Style.STROKE);
		mPaint.setStrokeWidth(12);
		mPaint.setColor(Color.RED);
		mPath = new Path();
	}

	@Override
	protected void onDraw(Canvas canvas) {
		mPath.addRect(200, 100, 500, 300, Direction.CW);
		canvas.drawPath(mPath, mPaint);
		mPath.reset();
		mPath.addRect(200, 400, 500, 600, Direction.CCW);
		canvas.drawPath(mPath, mPaint);
	}
}
```

看一下运行结果

![](/img/blog/2016/20161012154956509.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

两个图形基本一样，我们再来修改一下

```java
	@Override
	protected void onDraw(Canvas canvas) {
		mPath.addRect(200, 100, 500, 300, Direction.CW);
		mPath.setLastPoint(50, 200);
		canvas.drawPath(mPath, mPaint);
		mPath.reset();
		mPath.addRect(200, 400, 500, 600, Direction.CW);
		mPath.setLastPoint(50, 500);
		canvas.drawPath(mPath, mPaint);
	}
```

这次方向一致，只是一个在上一个在下，看一下结果
![](/img/blog/2016/20161012155643638.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)
和我们猜的完全一样，下面改一下方向

```java
	@Override
	protected void onDraw(Canvas canvas) {
		mPath.addRect(200, 100, 500, 300, Direction.CW);
		mPath.setLastPoint(50, 200);
		canvas.drawPath(mPath, mPaint);
		mPath.reset();
		mPath.addRect(200, 400, 500, 600, Direction.CCW);
		mPath.setLastPoint(50, 500);
		canvas.drawPath(mPath, mPaint);
	}
```

再看一下运行结果

![](/img/blog/2016/20161012155822287.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

两个完全不同，原因就在于一个是顺时针一个是逆时针，举个例子，比如说矩形的四个点按顺时针方向是ABCD，那么逆时针方向就是ADCB，所以在上面CW（顺时针）方向的时候setLastPoint的还是最后一个点D，所以没有变，当逆时针（CCW）方向的时候，最后一个点就变成了B，所以图形变了。下面在看一下当出现文字的时候就好理解了

```java
package test.view;

import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.graphics.Paint.Style;
import android.graphics.Path;
import android.graphics.RectF;
import android.util.AttributeSet;
import android.view.View;

public class PathView extends View {
	private Paint mPaint;
	private Path mPath;
	private RectF oval;

	public PathView(Context context, AttributeSet attrs) {
		super(context, attrs);
		init();
	}

	private void init() {
		mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
		mPaint.setStyle(Style.STROKE);
		mPaint.setStrokeWidth(2);
		mPaint.setColor(Color.RED);
		mPaint.setTextSize(36);
		mPath = new Path();
		oval = new RectF(100, 100, 900, 600);
	}

	@Override
	protected void onDraw(Canvas canvas) {
		canvas.translate(100, 100);
		mPath.addOval(oval, Path.Direction.CW);
		canvas.drawPath(mPath, mPaint);
		canvas.drawTextOnPath("123456789", mPath, 0, 0, mPaint);
		mPath.reset();
		mPath.addOval(oval, Path.Direction.CCW);
		canvas.drawPath(mPath, mPaint);
		canvas.drawTextOnPath("123456789", mPath, 0, 0, mPaint);
	}
}
```

看一下运行结果

![](/img/blog/2016/20161012161502870.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

下面的是顺时针方向的，上面的是逆时针方向的。OK，在继续看一下Path的其他方法，两个构造方法就不用说了，看一下reset()方法，清除线条，但没有改变他的填充类型，这里有4种类型，待会下面介绍，再看一下rewind()方法，清除线条但保持内部的数据结构，以便下次能快速使用。他俩的区别是reset保留填充类型，但没有保留原有数据结果，而rewind正好相反。在下面有一个枚举类Op

## Op

DIFFERENCE表示path1减去path2后剩下的部分  
INTERSECT表示path1和path2公共的部分  
UNION表示path1和path2都具有的部分  
XOR表示path1和path2各自但不包含对方的部分  
REVERSE_DIFFERENCE和DIFFERENCE差不多，不过他说path2减去path1后剩下的部分

直接上代码

```java
package test.view;

import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.graphics.Paint.Style;
import android.graphics.Path;
import android.graphics.Path.Direction;
import android.graphics.Path.Op;
import android.util.AttributeSet;
import android.view.View;

public class PathView extends View {
	private Paint mPaint;
	private Path mPath1;
	private Path mPath2;

	public PathView(Context context, AttributeSet attrs) {
		super(context, attrs);
		init();
	}

	private void init() {
		mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
		mPaint.setStyle(Style.FILL);
		mPaint.setStrokeWidth(20);
		mPaint.setColor(Color.RED);
		mPath1 = new Path();
		mPath1.addCircle(300, 800, 280, Direction.CCW);
		mPath2 = new Path();
		mPath2.addRect(400, 900, 800, 1300, Direction.CCW);
		mPath1.op(mPath2, Op.DIFFERENCE);
	}

	@Override
	protected void onDraw(Canvas canvas) {
		canvas.drawPath(mPath1, mPaint);
	}
}
```

运行结果图 DIFFERENCE

![](/img/blog/2016/20161012171304622.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

INTERSECT

![](/img/blog/2016/20161012171400092.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

UNION

![](/img/blog/2016/20161012171501081.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

XOR

![](/img/blog/2016/20161012171556253.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

REVERSE_DIFFERENCE

![](/img/blog/2016/20161012171654377.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

再看下一个枚举类FillType

## FillType

WINDING表示取Path所有的区域  
EVEN_ODD表示取path没有相交的区域  
INVERSE_WINDING表示取path未占有的区域  
INVERSE_EVEN_ODD表示取未占有和path相交的区域  
其中WINDING和INVERSE_WINDING，EVEN_ODD和INVERSE_EVEN_ODD都是取的相反的。

老规矩，上代码

```java
package test.view;

import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.graphics.Paint.Style;
import android.graphics.Path;
import android.util.AttributeSet;
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
		mPaint.setStyle(Style.FILL_AND_STROKE);
		mPath = new Path();
		mPath.addCircle(300, 300, 200, Path.Direction.CW);
		mPath.addCircle(600, 300, 200, Path.Direction.CW);
		mPath.setFillType(Path.FillType.WINDING);
	}

	@Override
	protected void onDraw(Canvas canvas) {
		canvas.drawPath(mPath, mPaint);
	}
}
```

WINDING

![](/img/blog/2016/20161012174331734.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

EVEN_ODD

![](/img/blog/2016/20161012174434422.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

INVERSE_WINDING

![](/img/blog/2016/20161012174540947.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

INVERSE_EVEN_ODD

![](/img/blog/2016/20161012174645714.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

下面再来看一下path的其他的一些方法，setFillType(FillType ft)表示设置填充类型。isInverseFillType()表示填充类型是否是相反的，也就是INVERSE_WINDING或者INVERSE_EVEN_ODD。toggleInverseFillType()表示置为相反的状态。isEmpty()表示Path路线是否是空的。isRect(RectF rect)表示路径是否是指定的矩形。computeBounds(RectF bounds, boolean exact)计算路径的边界保存到bounds中，exact参数没用，如果只有0或1个点，则bounds为（0,0,0,0），否则提取到他的边界保存到bounds中，看一下代码

```java
		mPath.addCircle(300, 300, 200, Path.Direction.CW);
		RectF mRectF = new RectF();
		mPath.computeBounds(mRectF, true);
		Log.d("wld__________", mRectF.left + "," + mRectF.top + ","
				+ mRectF.right + "," + mRectF.bottom);
```

看一下打印log
![](/img/blog/2016/20161013135853750.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

incReserve(int extraPtCount)表示将path增加extraPtCount个点，这能使path有效率的分配它的存储空间；

moveTo(float x, float y)移动到坐标为（x，y）的位置

rMoveTo(float dx, float dy)这个是相对位置，相对上一个位置的增量，如果上一个轮廓没有，则相当于moveTo()方法

lineTo(float x, float y)增加一条从上一个点到这一个点的线，如果moveTo()方法没有调用，则上一个点坐标为（0,0），

rLineTo(float dx, float dy)和lineTo(float dx, float dy)类似，只不过他是增量。

quadTo(float x1, float y1, float x2, float y2)和rQuadTo(float dx1, float dy1, float dx2, float dy2)表示增加一个二次方的贝赛尔曲线，看一下代码

## quadTo

```java
package test.view;

import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.graphics.Paint.Style;
import android.graphics.Path;
import android.util.AttributeSet;
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
		mPath.moveTo(200, 200);
		mPath.lineTo(500, 500);
		mPath.lineTo(800, 200);

		mPath.moveTo(200, 200);
		mPath.quadTo(500, 500, 800, 200);
	}

	@Override
	protected void onDraw(Canvas canvas) {
		canvas.drawPath(mPath, mPaint);
	}
}
```

一个是直线一个是贝塞尔曲线，看一下

![](/img/blog/2016/20161013103959806.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

cubicTo(float x1, float y1, float x2, float y2,float x3, float y3)和rCubicTo(float x1, float y1, float x2, float y2,float x3, float y3)表示三次贝赛尔曲线，看一下代码

## cubicTo

```java
		mPath.moveTo(200, 200);
		mPath.lineTo(500, 500);
		mPath.lineTo(800, 200);
		mPath.lineTo(1000, 500);

		mPath.moveTo(200, 200);
		mPath.cubicTo(500, 500, 800, 200, 1000, 500);
```

运行结果如下

![](/img/blog/2016/20161013104431063.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

arcTo有三个重载的方法，下面只看其中的一个arcTo(float left, float top, float right, float bottom, float startAngle,float sweepAngle, boolean forceMoveTo)

增加一段指定的弧形连接到path当中，作为一个新的轮廓，如果path的最后一个点和弧形的第一个点不一样，那么就会先通过lineTo()将这两个点连接起来，然后再连接圆弧。如果path是空的，那就会调用moveTo()把path的第一个点移到圆弧的第一个点上来，其中startAngle是开始角度，其中水平往右为0度，sweepAngle是圆弧扫过的角度，按顺时针方向扫描，forceMoveTo表示是否和上一个轮廓连接，如果为false表示连接，如果为true表示不连接，看一下代码

```java
		mPath.moveTo(100, 100);
		mPath.lineTo(400, 400);
		mPath.lineTo(600, 200);
		mPath.arcTo(700, 300, 900, 700, 0, 135, false);
```

看一下结果
![](/img/blog/2016/20161013110320312.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

在修改一下

```java
		mPath.arcTo(700, 300, 900, 700, 0, 135, true);
```

看一下结果

![](/img/blog/2016/20161013110439980.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

没有连接，如果sweepAngle大于360度的时候，会用sweepAngle对360求余，结果就不在演示了。
close()是关闭当前轮廓，如果起始点和终止点不重叠就会用一条直线连接。

## addRect

addRect(RectF rect, Direction dir)和addRect(float left, float top, float right, float bottom, Direction dir)增加一个闭合的矩形到path当中，看代码

```java
		mPath.moveTo(100, 100);
		mPath.lineTo(400, 400);
		mPath.lineTo(600, 200);
		mPath.addRect(700, 300, 900, 700, Direction.CCW);
```

![](/img/blog/2016/20161013112306661.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)
addOval(RectF oval, Direction dir)和addOval(float left, float top, float right, float bottom, Direction dir)增加一个闭合的椭圆轮廓到path中，看代码

## addOval

```java
		mPath.moveTo(100, 100);
		mPath.lineTo(400, 400);
		mPath.lineTo(600, 200);
		mPath.addOval(700, 300, 900, 700, Direction.CCW);
```

运行结果
![](/img/blog/2016/20161013112643057.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

## addCircle

addCircle(float x, float y, float radius, Direction dir)增加一个圆形轮廓到path中,

addArc(RectF oval, float startAngle, float sweepAngle)和addArc(float left, float top, float right, float bottom, float startAngle, float sweepAngle)添加一个指定的弧形到path中作为一个新的轮廓，如果arcTo方法中的最后一个参数为true，和这个方法基本上一样。

addRoundRect增加圆角矩形，看代码

```java
		mPath.moveTo(100, 100);
		mPath.lineTo(400, 400);
		mPath.lineTo(600, 200);
		mPath.addRoundRect(700, 300, 900, 700, 50, 50, Direction.CCW);
```

看一下运行结果

![](/img/blog/2016/20161013114426610.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)
再来修改一下

```java
		mPath.addRoundRect(700, 300, 900, 700, 50, 100, Direction.CCW);
```

看一下结果

![](/img/blog/2016/20161013114634600.png)

```java
		mPath.addRoundRect(700, 300, 900, 700, new float[] { 20, 20, 40, 40,
				80, 80, 20, 100 }, Direction.CCW);
```

看一下结果

![](/img/blog/2016/20161013114901024.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

addPath(Path src, float dx, float dy)，addPath(Path src)和addPath(Path src, Matrix matrix)表示将src添加到path中，dx，dy是偏移量，matrix是转换的矩阵，看代码

## addPath

```java
		Matrix mMatrix = new Matrix();
		mPath = new Path();
		mPath.moveTo(100, 100);
		mPath.lineTo(400, 400);
		mPath.lineTo(600, 200);

		mPath1 = new Path();
		mPath1.moveTo(100, 500);
		mPath1.lineTo(400, 800);
		mPath1.lineTo(600, 600);
		mMatrix.postTranslate(300, 0);
		mPath.addPath(mPath1, mMatrix);
```

下面的path相对于上面的往下移了400，通过矩阵又往右移了300，看结果

![](/img/blog/2016/20161013120230885.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

offset(float dx, float dy)表示将path平移dx,dy

offset(float dx, float dy, Path dst)表示将当前path平移（dx，dy）之后存储到dst中。

setLastPoint(float dx, float dy)设置轮廓的最后一个点。

transform(Matrix matrix, Path dst)将path进行matrix变化后，将结果保存到dst当中，如果dst=null，将结果保存到当前path当中

transform(Matrix matrix)对当前path进行matrix变换。

OK，目前为止Path的方法已经分析完毕。