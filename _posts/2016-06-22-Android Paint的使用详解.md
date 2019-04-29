---
layout: post
title: "Android Paint的使用详解"
subtitle: 'Android Paint的使用详解'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “娶妻无媒毋须恨，书中有女颜如玉。”
	--赵恒

## 正文

自定义控件具有很强的灵活性，可以根据你的想法画出各种各样的图案，在Android中如果是自定义控件的话，Paint这个类用的还是较多的，这一篇就来简单介绍Paint这个类的使用，先来看一下这个类的注释

```
/**
 * The Paint class holds the style and color information about how to draw
 * geometries, text and bitmaps.
 */
```

这个类可以画几何图形，文本和bitmap。由于这个类的native方法和@hide方法比较多，这里就挑一些在工作中可能常用到的方法来讲解。先来看一下Paint的style，共有3种

Paint.Style.FILL：填充内部  
Paint.Style.FILL_AND_STROKE  ：填充内部和描边  
Paint.Style.STROKE  ：描边

我们看一下效果

![](/img/blog/2016/20160620160723291.png)

FILL_AND_STROKE和FILL区别不是很大。在看一下Cap，也有3种类型，主要是线条的末端，为了直观，下面三个线条我设置的比较粗，我们看一下效果

![](/img/blog/2016/20160620170323869.jpg)

我们看一下，其中两条竖线是三条线条的坐标的起始点和终止点，区别很明显。再来看看Join，也是有3种类型，我们看一下

![](/img/blog/2016/20160620171246480.jpg)

这个是画的矩形，连接的时候用到的，效果很明显，就不在解释。再看下一个Align，也是有3种类型，看名字大概也能猜的出来，不过还是要来验证一下

![](/img/blog/2016/20160620172428974.jpg)

OK，Paint的几种类型已经演示完了，下面主要来看一下他的方法。

//重置Paint。
reset()

//设置一些标志，比如抗锯齿，下划线等等。  
setFlags(int flags)

//设置抗锯齿，如果不设置，加载位图的时候可能会出现锯齿状的边界，如果设置，边界就会变的稍微有点模糊，锯齿就看不到了。  
setAntiAlias(boolean aa)

//设置是否抖动，如果不设置感觉就会有一些僵硬的线条，如果设置图像就会看的更柔和一些  
setDither(boolean dither)

//这个是文本缓存，设置线性文本，如果设置为true就不需要缓存，  
setLinearText(boolean linearText)

//设置亚像素，是对文本的一种优化设置，可以让文字看起来更加清晰明显，可以参考一下PC端的控制面板-外观和个性化-调整ClearType文本  
setSubpixelText(boolean subpixelText)

//设置文本的下划线  
setUnderlineText(boolean underlineText)

//设置文本的删除线  
setStrikeThruText(boolean strikeThruText)

//设置文本粗体  
setFakeBoldText(boolean fakeBoldText)

//对位图进行滤波处理，如果该项设置为true，则图像在动画进行中会滤掉对Bitmap图像的优化操作，加快显示   
setFilterBitmap(boolean filter)

//下面这几个就不用说了，上面已经演示过  
setStyle(Style style)，setStrokeCap(Cap cap)，setStrokeJoin(Join join)，setTextAlign(Align align)，

//设置画笔颜色  
setColor(int color)

//设置画笔的透明度[0-255]，0是完全透明，255是完全不透明  
setAlpha(int a)

//设置画笔颜色，argb形式alpha，red，green，blue每个范围都是[0-255],  
setARGB(int a, int r, int g, int b)

//画笔样式为空心时，设置空心画笔的宽度  
setStrokeWidth(float width)

//当style为Stroke或StrokeAndFill时设置连接处的倾斜度，这个值必须大于0，看一下演示结果  setStrokeMiter(float miter)

左上角的没有设置setStrokeMiter，右上角setStrokeMiter(2.3f)，左下角setStrokeMiter(1.7f)，右下角setStrokeMiter(0f)

![](/img/blog/2016/20160622115843846.jpg)

//这个没整明白具体干什么用的  
getFillPath(Path src, Path dst)

//设置着色器，用来给图像着色的，绘制出各种渐变效果，有BitmapShader，ComposeShader，LinearGradient，RadialGradient，SweepGradient几种，这个以后再单独讲  
setShader(Shader shader)

//设置画笔颜色过滤器，有ColorMatrixColorFilter，LightingColorFilter，PorterDuffColorFilter几种，这个以后再单独分析  
setColorFilter(ColorFilter filter)

//设置图形重叠时的显示方式，下面来演示一下  
setXfermode(Xfermode xfermode)

下面是我运行目录D:\Android\adt-bundle-windows\sdk\samples\android-20\legacy\ApiDemos\src\com\example\android\apis\graphics\Xfermodes类的结果

![](/img/blog/2016/20160622162114475.jpg)

总共有16种重叠模式，而Mode类中显示的总共有18种，下面是我自己写的一个，只有绿色和红色两种图片（没有黑色），先画的是绿色，后画的是红色，和上面有很大差距，不知道什么原因，有时间得好好研究一下

![](/img/blog/2016/20160622162640466.jpg)

//设置绘制路径的效果，有ComposePathEffect，CornerPathEffect，DashPathEffect，DiscretePathEffect，PathDashPathEffect，SumPathEffect几种，以后在单独分析  
setPathEffect(PathEffect effect)

//对图像进行一定的处理，实现滤镜的效果，如滤化，立体等,有BlurMaskFilter，EmbossMaskFilter几种setMaskFilter(MaskFilter maskfilter)

//设置字体样式，可以是Typeface设置的样式，也可以通过Typeface的createFromAsset(AssetManager mgr, String path)方法加载样式  
setTypeface(Typeface typeface)

//设置阴影效果，radius为阴影角度，dx和dy为阴影在x轴和y轴上的距离，color为阴影的颜色 ，看一下演示效果，其中第一个是没有阴影的，第二个设置了黑色的阴影  
setShadowLayer(float radius, float dx, float dy, int shadowColor)

![](/img/blog/2016/20160622170228439.jpg)

//设置地理位置，比如显示中文，日文，韩文等，默认的显示Locale.getDefault()即可，  
setTextLocale(Locale locale)

//设置优雅的文字高度，这个设置可能会对FontMetrics产生影响  
setElegantTextHeight(boolean elegant)

//设置字体大小  
setTextSize(float textSize)

//设置字体的水平方向的缩放因子，默认值为1，大于1时会沿X轴水平放大，小于1时会沿X轴水平缩小setTextScaleX(float scaleX)

//设置文本在水平方向上的倾斜，默认值为0，推荐的值为-0.25，  
setTextSkewX(float skewX)

//设置行的间距，默认值是0，负值行间距会收缩  
setLetterSpacing(float letterSpacing)

//设置字体样式，可以设置CSS样式  
setFontFeatureSettings(String settings)

//这个Paint的静态内部类，主要用于字体的高度，以后再分析  
FontMetrics

//下面几个就是测量字体的长度了  
measureText(char[] text, int index, int count)，measureText(String text, int start, int end)，measureText(String text)，measureText(CharSequence text, int start, int end)

//下面这几个就是剪切显示，就是大于maxWidth的时候只截取指定长度的显示  
breakText(char[] text, int index, int count,float maxWidth, float[] measuredWidth)，breakText(CharSequence text, int start, int end,boolean measureForwards,  floatmaxWidth, float[] measuredWidth)，breakText(String text, boolean measureForwards,float maxWidth, float[] measuredWidth)

//提取指定范围内的字符串，保存到widths中，  
getTextWidths(char[] text, int index, int count,float[] widths)，getTextWidths(CharSequence text, int start, int end, float[] widths)，getTextWidths(String text, int start, int end, float[] widths)，getTextWidths(String text, float[] widths)

//获取文本绘制的路径，提取到Path中，  
getTextPath(char[] text, int index, int count, float x, float y, Path path)，getTextPath(String text, int start, int end, float x, float y, Path path)

//得到文本的边界，上下左右，提取到bounds中，可以通过这计算文本的宽和高  
getTextBounds(String text, int start, int end, Rect bounds) ，getTextBounds(char[] text, int index, int count, Rect bounds)

OK，剩下的一些就是@hide或是native,或者是get方法，这里就不在一一叙述。