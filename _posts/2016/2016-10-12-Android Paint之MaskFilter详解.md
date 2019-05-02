---
layout: post
title: "Android Paint之MaskFilter详解"
subtitle: 'Android Paint之MaskFilter详解'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “钱财如粪土，仁义值千金。”

## 正文

这一篇来简单分析一下Paint的setMaskFilter(MaskFilter maskfilter)方法，其中MaskFilter有两个子类，一个BlurMaskFilter一个是EmbossMaskFilter，这里先来看一下BlurMaskFilter这个类的源码

## BlurMaskFilter

```java
    public enum Blur {
        /**
         * Blur inside and outside the original border.
         */
        NORMAL(0),

        /**
         * Draw solid inside the border, blur outside.
         */
        SOLID(1),

        /**
         * Draw nothing inside the border, blur outside.
         */
        OUTER(2),

        /**
         * Blur inside the border, draw nothing outside.
         */
        INNER(3);
        
        Blur(int value) {
            native_int = value;
        }
        final int native_int;
    }
    
    /**
     * Create a blur maskfilter.
     *
     * @param radius The radius to extend the blur from the original mask. Must be > 0.
     * @param style  The Blur to use
     * @return       The new blur maskfilter
     */
    public BlurMaskFilter(float radius, Blur style) {
        native_instance = nativeConstructor(radius, style.native_int);
    }
```

构造方法很简单，传入一个半径，是绘制阴影模糊的半径，其中类型有以上4种，这里就不在一一叙述，先看一下代码然后在直接看运行结果

```java
public class MaskFilterView extends View {
	private Paint paint;
	private TextPaint mTextPaint;
	private BlurMaskFilter bmf1;
	private BlurMaskFilter bmf2;
	private BlurMaskFilter bmf3;
	private BlurMaskFilter bmf4;

	public MaskFilterView(Context context, AttributeSet attrs) {
		super(context, attrs);
		paint = new Paint(Paint.ANTI_ALIAS_FLAG);
		paint.setColor(Color.RED);
		paint.setStyle(Style.FILL_AND_STROKE);
		mTextPaint = new TextPaint(Paint.ANTI_ALIAS_FLAG);
		mTextPaint.setTextSize(64);
		mTextPaint.setColor(Color.BLUE);
		mTextPaint.setTextAlign(Align.CENTER);

		bmf1 = new BlurMaskFilter(100, Blur.NORMAL);
		bmf2 = new BlurMaskFilter(100, Blur.SOLID);
		bmf3 = new BlurMaskFilter(100, Blur.OUTER);
		bmf4 = new BlurMaskFilter(100, Blur.INNER);

	}

	@Override
	protected void onDraw(Canvas canvas) {
		super.onDraw(canvas);
		canvas.translate(0, 200);
		canvas.drawColor(Color.WHITE);
		paint.setMaskFilter(bmf1);
		canvas.drawCircle(300, 300, 150, paint);
		canvas.drawText("NORMAL", 300, 600, mTextPaint);

		paint.setMaskFilter(bmf2);
		canvas.drawCircle(300, 900, 150, paint);
		canvas.drawText("SOLID", 300, 1200, mTextPaint);

		paint.setMaskFilter(bmf3);
		canvas.drawCircle(800, 300, 150, paint);
		canvas.drawText("OUTER", 800, 600, mTextPaint);

		paint.setMaskFilter(bmf4);
		canvas.drawCircle(800, 900, 150, paint);
		canvas.drawText("INNER", 800, 1200, mTextPaint);
	}
}
```

在Androidmanifest中要加上下面这段代码，否则不会出现任何模糊效果

```java
        android:hardwareAccelerated="false"
```

来看一下运行的结果

![](/img/blog/2016/20160905175407554.png)

效果很明显，这里就不在详述。下面再来看一下EmbossMaskFilter这个类，先看一下他的构造方法

## EmbossMaskFilter

```java
    /**
     * Create an emboss maskfilter
     *
     * @param direction  array of 3 scalars [x, y, z] specifying the direction of the light source
     * @param ambient    0...1 amount of ambient light
     * @param specular   coefficient for specular highlights (e.g. 8)
     * @param blurRadius amount to blur before applying lighting (e.g. 3)
     * @return           the emboss maskfilter
     */
    public EmbossMaskFilter(float[] direction, float ambient, float specular, float blurRadius) {
        if (direction.length < 3) {
            throw new ArrayIndexOutOfBoundsException();
        }
        native_instance = nativeConstructor(direction, ambient, specular, blurRadius);
    }
```

其中direction表示x,y,z的三个方向，ambient表示的是光的强度，范围为[0-1]，specular是反射亮度的系数，blurRadius是指模糊的半径，看一下代码

```java
package test.view;

import android.annotation.SuppressLint;
import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.EmbossMaskFilter;
import android.graphics.Paint;
import android.graphics.Paint.Style;
import android.util.AttributeSet;
import android.view.View;

public class MaskFilterView extends View {
	private Paint paint;
	EmbossMaskFilter emboss;

	public MaskFilterView(Context context, AttributeSet attrs) {
		super(context, attrs);
		paint = new Paint(Paint.ANTI_ALIAS_FLAG);
		paint.setColor(Color.RED);
		paint.setStyle(Style.STROKE);
		paint.setStrokeWidth(32);
		paint.setTextSize(368);
		float[] direction = new float[] { 1, 1, 1 };
		// 设置环境光亮度
		float light = 0.1f;
		// 选择要应用的反射等级
		float specular = 8;
		// 向mask应用一定级别的模糊
		float blur = 3;
		emboss = new EmbossMaskFilter(direction, light, specular, blur);

	}

	@Override
	protected void onDraw(Canvas canvas) {
		super.onDraw(canvas);
		canvas.drawText("山大王", 30, 400, paint);
	}

	@SuppressLint("NewApi")
	public void setparam(float x, float y, float z, float light,
			float specular, float blur) {
		emboss = new EmbossMaskFilter(new float[] { x, y, z }, light, specular,
				blur);
		paint.setMaskFilter(emboss);
		invalidate();
	}
}
```

看一下运行结果，这个会根据不同的值产生不同的结果，下面只截取已其中的一个

![](/img/blog/2016/20161012144236776.png)