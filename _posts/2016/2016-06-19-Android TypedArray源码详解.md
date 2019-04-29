---
layout: post
title: "Android TypedArray源码详解"
subtitle: 'Android TypedArray源码详解'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “因依老宿发心初，半学修心半读书。”
	--王建

## 正文

在自定义控件的时候，如果我们想额外的添加一些属性，就会用到TypedArray这个类，那么这个类是怎么得到的，以及怎么使用的，这篇讲会详细讲解，下面是我以前自定义控件的一段代码

```java
TypedArray typedArray = context.obtainStyledAttributes(attrs,R.styleable.myaccount_item_style);
```

我们看到TypedArray是通过Context的方法得到的，但要记住完成之后一定要调用recycle()方法进行回收，我们点进去找到最终实现

## obtainStyledAttributes

```java
        public TypedArray obtainStyledAttributes(AttributeSet set,
                int[] attrs, int defStyleAttr, int defStyleRes) {
            final int len = attrs.length;
            final TypedArray array = TypedArray.obtain(Resources.this, len);

            // XXX note that for now we only work with compiled XML files.
            // To support generic XML files we will need to manually parse
            // out the attributes from the XML file (applying type information
            // contained in the resources and such).
            final XmlBlock.Parser parser = (XmlBlock.Parser)set;
            AssetManager.applyStyle(mTheme, defStyleAttr, defStyleRes,
                    parser != null ? parser.mParseState : 0, attrs, array.mData, array.mIndices);

            array.mTheme = this;
            array.mXml = parser;

			…………………………
            return array;
        }
```

我们先看下面AssetManager的applyStyle方法是native方法，也就是用C++实现的，他会提取自定义控件属性的的值保存TypedArray中的mData数组中，这个数组的大小是由你定义控件属性的个数决定的，是他的6倍，上面的attrs其实就是你自定义属性的个数，我们来看一下

## obtain

```java
    static TypedArray obtain(Resources res, int len) {
        final TypedArray attrs = res.mTypedArrayPool.acquire();
        if (attrs != null) {
            attrs.mLength = len;
            attrs.mRecycled = false;

            final int fullLen = len * AssetManager.STYLE_NUM_ENTRIES;
            if (attrs.mData.length >= fullLen) {
                return attrs;
            }

            attrs.mData = new int[fullLen];
            attrs.mIndices = new int[1 + len];
            return attrs;
        }

        return new TypedArray(res,
                new int[len*AssetManager.STYLE_NUM_ENTRIES],
                new int[1+len], len);
    }
```

他首先会从TypedArray池中获取，如果有就取出，mDate的大小不能小于属性个数的6倍，因为STYLE_NUM_ENTRIES的值为6，如果没有就new一个然后返回，把属性的值提取出来之后我们就可以来操作了，我们先来看一下View类初始化中的一段代码

```java
        final int N = a.getIndexCount();
        for (int i = 0; i < N; i++) {
            int attr = a.getIndex(i);
            switch (attr) {
                case com.android.internal.R.styleable.View_background:
                    background = a.getDrawable(attr);
                    break;
					…………………………
            }
        }
```

他会把TypedArray中的数据提取出来对View的属性赋值，我们来看一下TypedArray类的构造方法

## TypedArray

```java
    /*package*/ TypedArray(Resources resources, int[] data, int[] indices, int len) {
        mResources = resources;
        mMetrics = mResources.mMetrics;
        mAssets = mResources.mAssets;
        mData = data;
        mIndices = indices;
        mLength = len;
    }
```

代码很简单，其中mData就是就是从xml文件中提取到的数据，mData的大小是自定义属性个数的6倍，所以这里是每6个作为一组，我们可以看一下上面的obtain方法中data数组的大小是乘以6（STYLE_NUM_ENTRIES）的，这6种类型如下，定义在AssetManager类中，下面的第一个表示每组6个

```java
    /*package*/ static final int STYLE_NUM_ENTRIES = 6;
    /*package*/ static final int STYLE_TYPE = 0;
    /*package*/ static final int STYLE_DATA = 1;
    /*package*/ static final int STYLE_ASSET_COOKIE = 2;
    /*package*/ static final int STYLE_RESOURCE_ID = 3;
    /*package*/ static final int STYLE_CHANGING_CONFIGURATIONS = 4;
    /*package*/ static final int STYLE_DENSITY = 5;
```

对应着TypedValue类中的这7中类型，其中string是根据type得到的

```java
    /** The type held by this value, as defined by the constants here.
     *  This tells you how to interpret the other fields in the object. */
    public int type;

    /** If the value holds a string, this is it. */
    public CharSequence string;

    /** Basic data in the value, interpreted according to {@link #type} */
    public int data;

    /** Additional information about where the value came from; only
     *  set for strings. */
    public int assetCookie;

    /** If Value came from a resource, this holds the corresponding resource id. */
    public int resourceId;

    /** If Value came from a resource, these are the configurations for which
     *  its contents can change. */
    public int changingConfigurations = -1;

    /**
     * If the Value came from a resource, this holds the corresponding pixel density.
     * */
    public int density;
```

如果我们认真看的时候就会发现obtain方法中对mIndices数组初始化的时候是加1的，因为mIndices数组的第一个保存的是我们所使用属性的个数，记住是使用不是定义，我们来看一下其中的一些代码

## length

```java
    /**
     * Return the number of values in this array.
     */
    public int length() {
        if (mRecycled) {
            throw new RuntimeException("Cannot make calls to a recycled instance!");
        }

        return mLength;
    }

    /**
     * Return the number of indices in the array that actually have data.
     */
    public int getIndexCount() {
        if (mRecycled) {
            throw new RuntimeException("Cannot make calls to a recycled instance!");
        }

        return mIndices[0];
    }
```

第一个length返回的是我们所定义属性的个数，因为这个参数是在构造函数中赋值的，传递的是int[] attrs的长度，而这个sttrs就是我们在attrs文件中自定义属性的时候在R文件中自动生成的一个数组。而下面的getIndexCount()方法返回的是我们所使用的属性个数，因为mIndices的数据是从xml文件中提取的，第一个位置保存的是我们使用属性的个数，后面的位置就是我们使用的自定义属性在R文件中生成的id，在看一个方法，也是自定义的时候常用到的

## getIndex

```java
    public int getIndex(int at) {
        if (mRecycled) {
            throw new RuntimeException("Cannot make calls to a recycled instance!");
        }

        return mIndices[1+at];
    }
```

这个得到的就是自定义属性在R文件中生成的id，剩下的一些方法就是从TypedArray中提取值了，主要有以下几种类型

```java
    <declare-styleable name="CustomTheme">
        <attr name="textView1" format="string" />
        <attr name="textView2" format="boolean" />
        <attr name="textView3" format="integer" />
        <attr name="textView4" format="float" />
        <attr name="textView5" format="color" />
        <attr name="textView6" format="dimension" />
        <attr name="textView7" format="fraction" />
        <attr name="textView8" format="reference" />
        <attr name="textView9" format="enum" />
        <attr name="textView10" format="flags" />
    </declare-styleable>
```

TypedArray方法比较多，这里就捡常用的几个来分析一下，在分析之前先看一下下面这个方法

## loadStringValueAt

```java
    private CharSequence loadStringValueAt(int index) {
        final int[] data = mData;
        final int cookie = data[index+AssetManager.STYLE_ASSET_COOKIE];
        if (cookie < 0) {
            if (mXml != null) {
                return mXml.getPooledString(
                    data[index+AssetManager.STYLE_DATA]);
            }
            return null;
        }
        return mAssets.getPooledStringForCookie(cookie, data[index+AssetManager.STYLE_DATA]);
    }
```

上面所说的每6个一组，其中每组下标为STYLE_ASSET_COOKIE（2）的是用来标记缓存的，并且是只对String类型的，我们来看一下

## getValueAt

```java
    private boolean getValueAt(int index, TypedValue outValue) {
        final int[] data = mData;
        final int type = data[index+AssetManager.STYLE_TYPE];
        if (type == TypedValue.TYPE_NULL) {
            return false;
        }
        outValue.type = type;
        outValue.data = data[index+AssetManager.STYLE_DATA];
        outValue.assetCookie = data[index+AssetManager.STYLE_ASSET_COOKIE];
        outValue.resourceId = data[index+AssetManager.STYLE_RESOURCE_ID];
        outValue.changingConfigurations = data[index+AssetManager.STYLE_CHANGING_CONFIGURATIONS];
        outValue.density = data[index+AssetManager.STYLE_DENSITY];
        outValue.string = (type == TypedValue.TYPE_STRING) ? loadStringValueAt(index) : null;
        return true;
    }
```

上面这个方法是把mData指定范围的6个数据提取到outValue中，其中string值通过type类型得到的，我们再来看一下assetCookie的注释

```java
    /** Additional information about where the value came from; only
     *  set for strings. */
    public int assetCookie;
```

所以他只针对String类型，我们再来看一下String类型的注释

```java
    /** The <var>string</var> field holds string data.  In addition, if
     *  <var>data</var> is non-zero then it is the string block
     *  index of the string and <var>assetCookie</var> is the set of
     *  assets the string came from. */
    public static final int TYPE_STRING = 0x03;
```

所以他只针对string类型的数据进行提取，比如text，String，color等，color可以是string类型也可以是int类型，还看上面的loadStringValueAt方法，如果cookie小于0，说明没有缓存，就会从xml中解析，否则就从缓存中取

## getPooledStringForCookie

```java
    /*package*/ final CharSequence getPooledStringForCookie(int cookie, int id) {
        // Cookies map to string blocks starting at 1.
        return mStringBlocks[cookie - 1].get(id);
```

我们来看一下是怎么从xml中解析的，看到上面的obtainStyledAttributes方法，会发现这样一段代码 array.mXml = parser;其中parser就是View及其子类在初始化的时候传递的AttributeSet，我们在前面的<a href="https://androidboke.com/2016/03/18/Android-LayoutInflater%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E5%8F%8A%E4%BD%BF%E7%94%A8-%E4%BA%8C" target="_blank">Android LayoutInflater源码分析及使用（二）</a>中讲到，View及其子类创建的时候是通过反射来初始化的，我们来回顾一下

## createView

```java
   public final View createView(String name, String prefix, AttributeSet attrs)
            throws ClassNotFoundException, InflateException {
        Constructor<? extends View> constructor = sConstructorMap.get(name);
        Class<? extends View> clazz = null;

        try {
				…………………………
                constructor = clazz.getConstructor(mConstructorSignature);
				…………………………

            Object[] args = mConstructorArgs;
            args[1] = attrs;

            constructor.setAccessible(true);
            final View view = constructor.newInstance(args);
			…………………………
            return view;

        } catch (NoSuchMethodException e) {
           …………………………
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```

其中attrs是通过Resource的loadXmlResourceParser方法加载的，我们看一下

## loadXmlResourceParser

```java
    /*package*/ XmlResourceParser loadXmlResourceParser(int id, String type)
            throws NotFoundException {
        synchronized (mAccessLock) {
            TypedValue value = mTmpValue;
            if (value == null) {
                mTmpValue = value = new TypedValue();
            }
            getValue(id, value, true);
            if (value.type == TypedValue.TYPE_STRING) {
                return loadXmlResourceParser(value.string.toString(), id,
                        value.assetCookie, type);
            }
            throw new NotFoundException(
                    "Resource ID #0x" + Integer.toHexString(id) + " type #0x"
                    + Integer.toHexString(value.type) + " is not valid");
        }
    }
```

剩下的就是涉及到Xml的解析，这里就不在作深入探讨，言归正传，还回到刚才的loadStringValueAt方法，如果缓存中存在就从缓存中去，如果不存在就通过xml解析获取。下面在看一下一些常用的方法，其中getText(int index)和getString(int index)差不多，我们就来看一下getString(int index)方法

## getString

```java
    public String getString(int index) {
        if (mRecycled) {
            throw new RuntimeException("Cannot make calls to a recycled instance!");
        }

        index *= AssetManager.STYLE_NUM_ENTRIES;
        final int[] data = mData;
        final int type = data[index+AssetManager.STYLE_TYPE];
        if (type == TypedValue.TYPE_NULL) {
            return null;
        } else if (type == TypedValue.TYPE_STRING) {
            return loadStringValueAt(index).toString();
        }

        TypedValue v = mValue;
        if (getValueAt(index, v)) {
            Log.w(Resources.TAG, "Converting to string: " + v);
            CharSequence cs = v.coerceToString();
            return cs != null ? cs.toString() : null;
        }
        Log.w(Resources.TAG, "getString of bad type: 0x"
              + Integer.toHexString(type));
        return null;
    }
```

上面的index要乘以6（STYLE_NUM_ENTRIES），因为是每6个一组的，如果type为null就返回空，如果为String类型就会调用loadStringValueAt方法获取我们设置的值。有一点要注意，如果我们在attrs中设置的format类型和我们自定义设置的参数不符的话，当运行的时候是会报错的，必须要设置相符并clean才能解决。否则就执行下面的方法，强制转换为字符串，代码比较简单，这里就不再贴出。在来看下一个bool类型和int类型的，由于这两个差不多，就随便挑一个

## getInt

```java
    public int getInt(int index, int defValue) {
        index *= AssetManager.STYLE_NUM_ENTRIES;
        final int[] data = mData;
        final int type = data[index+AssetManager.STYLE_TYPE];
        if (type == TypedValue.TYPE_NULL) {
            return defValue;
        } else if (type >= TypedValue.TYPE_FIRST_INT
            && type <= TypedValue.TYPE_LAST_INT) {
            return data[index+AssetManager.STYLE_DATA];
        }

        TypedValue v = mValue;
        if (getValueAt(index, v)) {
            Log.w(Resources.TAG, "Converting to int: " + v);
            return XmlUtils.convertValueToInt(
                v.coerceToString(), defValue);
        }
        Log.w(Resources.TAG, "getInt of bad type: 0x"
              + Integer.toHexString(type));
        return defValue;
    }
```

上面的类型如果大于TYPE_FIRST_INT并且小于TYPE_LAST_INT的时候就从mDate中提取值，这个不知道为什么要这样写，不过从他的范围来看也就int，Boolean，color三种是这样取值的

```java
    /** Identifies the start of integer values that were specified as
     *  color constants (starting with '#'). */
    public static final int TYPE_FIRST_COLOR_INT = 0x1c;

    /** The <var>data</var> field holds a color that was originally
     *  specified as #aarrggbb. */
    public static final int TYPE_INT_COLOR_ARGB8 = 0x1c;
    /** The <var>data</var> field holds a color that was originally
     *  specified as #rrggbb. */
    public static final int TYPE_INT_COLOR_RGB8 = 0x1d;
    /** The <var>data</var> field holds a color that was originally
     *  specified as #argb. */
    public static final int TYPE_INT_COLOR_ARGB4 = 0x1e;
    /** The <var>data</var> field holds a color that was originally
     *  specified as #rgb. */
    public static final int TYPE_INT_COLOR_RGB4 = 0x1f;

    /** Identifies the end of integer values that were specified as color
     *  constants. */
    public static final int TYPE_LAST_COLOR_INT = 0x1f;

    /** Identifies the end of plain integer values. */
    public static final int TYPE_LAST_INT = 0x1f;
```

如果范围不在TYPE_FIRST_INT和TYPE_LAST_INT之间，就会把mData指定位置上的值提取到TypedValue中，然后在强制转化，如果没有提取到就会返回一个默认值，因为如果在attrs中定义但没有用到，就会返回一个默认值。我们来看一下是怎么转化的

## convertValueToInt

```java
    public static final int
    convertValueToInt(CharSequence charSeq, int defaultValue)
    {
        if (null == charSeq)
            return defaultValue;

        String nm = charSeq.toString();

        // XXX This code is copied from Integer.decode() so we don't
        // have to instantiate an Integer!

        int value;
        int sign = 1;
        int index = 0;
        int len = nm.length();
        int base = 10;

        if ('-' == nm.charAt(0)) {
            sign = -1;
            index++;
        }

        if ('0' == nm.charAt(index)) {
            //  Quick check for a zero by itself
            if (index == (len - 1))
                return 0;

            char    c = nm.charAt(index + 1);

            if ('x' == c || 'X' == c) {
                index += 2;
                base = 16;
            } else {
                index++;
                base = 8;
            }
        }
        else if ('#' == nm.charAt(index))
        {
            index++;
            base = 16;
        }

        return Integer.parseInt(nm.substring(index), base) * sign;
    }
```

这个很好理解，转化为int类型有0开头的8进制，0x开头的16进制，还有#开头的color值，如果转化之前是负数，转化之后还要乘以-1（sign）。再来看一个

## getColor

```java
    public int getColor(int index, int defValue) {
        index *= AssetManager.STYLE_NUM_ENTRIES;
        final int[] data = mData;
        final int type = data[index+AssetManager.STYLE_TYPE];
        if (type == TypedValue.TYPE_NULL) {
            return defValue;
        } else if (type >= TypedValue.TYPE_FIRST_INT
            && type <= TypedValue.TYPE_LAST_INT) {
            return data[index+AssetManager.STYLE_DATA];
        } else if (type == TypedValue.TYPE_STRING) {
            final TypedValue value = mValue;
            if (getValueAt(index, value)) {
                ColorStateList csl = mResources.loadColorStateList(
                        value, value.resourceId);
                return csl.getDefaultColor();
            }
            return defValue;
        }

        throw new UnsupportedOperationException("Can't convert to color: type=0x"
                + Integer.toHexString(type));
    }
```

这个就不用多说了，因为color有String和int两种类型，如果是String类型就会返回ColorStateList的默认值，因为ColorStateList可能有好几种类型，但必须都是false的才是默认的，下面随便看一个，下面这些提取之后默认的就是green，因为只有他的所有状态都是false。

```html
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">

    <item android:state_pressed="true" android:color="@color/blue"/>
    <item android:state_pressed="false" android:state_selected="true" android:color="@color/yellow"/>
    <item android:state_pressed="false" android:state_selected="false" android:color="@color/green"/>

</selector>
```

下面在看最后一个方法

## getLayoutDimension

```java
    public int getLayoutDimension(int index, String name) {
        index *= AssetManager.STYLE_NUM_ENTRIES;
        final int[] data = mData;
        final int type = data[index+AssetManager.STYLE_TYPE];
        if (type >= TypedValue.TYPE_FIRST_INT
                && type <= TypedValue.TYPE_LAST_INT) {
            return data[index+AssetManager.STYLE_DATA];
        } else if (type == TypedValue.TYPE_DIMENSION) {
            return TypedValue.complexToDimensionPixelSize(
                data[index+AssetManager.STYLE_DATA], mResources.mMetrics);
        }

        throw new RuntimeException(getPositionDescription()
                + ": You must supply a " + name + " attribute.");
    }
```

看方法名大概就知道是获取layout的尺寸的，大致看一下，在ViewGroup中

## setBaseAttributes

```java
        protected void setBaseAttributes(TypedArray a, int widthAttr, int heightAttr) {
            width = a.getLayoutDimension(widthAttr, "layout_width");
            height = a.getLayoutDimension(heightAttr, "layout_height");
        }
```

其中获取到的值有3种，一种是精确的我们给的大于0的，一种是-1（MATCH_PARENT），另一种是-2（WRAP_CONTENT），记得在讲到<a href="https://androidboke.com/2016/03/18/Android-LayoutInflater%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E5%8F%8A%E4%BD%BF%E7%94%A8-%E4%BA%8C" target="_blank">Android LayoutInflater源码分析及使用（二）</a>的时候说到，xml的属性除了宽和高以外在初始化的时候基本上都能提取到，但宽和高是不行的，因为他是最终计算出来的，如果大家自定义View继承View的时候，要必须重写onMeasure方法，重新计算他的宽和高，如果我们不计算，当我们使用MATCH_PARENT或WRAP_CONTENT属性的时候，结果是完全一样的，尺寸都是填满剩下的屏幕，如果不重写onMeasure方法，在xml文件中把他的宽和高都写死也行，但这样不够灵活，我们来看一下为什么要重写

```java
    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
```

我们看到View中的getDefaultSize方法，AT_MOST和EXACTLY返回的结果都是一样的，如果想看建议看一下ViewGroup的getChildMeasureSpec方法，这个就不在贴出，可以自己去看。OK，TypedArray中剩下的方法基本上也都非常相似，这里就不在一一讲述。