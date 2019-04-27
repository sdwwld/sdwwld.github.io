---
layout: post
title: "Android LayoutInflater源码分析及使用（二）"
subtitle: 'Android LayoutInflater源码分析及使用'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “富贵必从勤苦得，男儿须读五车书。”
	--杜甫

## 正文

上一篇中我们简单介绍了LayoutInflater是怎么获取的，那么这一篇我们将详细介绍他的一个我们最常用的方法inflate，流程是这样的，我们先进行源码分析，然后猜想，最后在具体验证。在介绍inflate方法之前我们先看下面这几行代码，

```
* <p>
* For performance reasons, view inflation relies heavily on pre-processing of
* XML files that is done at build time. Therefore, it is not currently possible
* to use LayoutInflater with an XmlPullParser over a plain XML file at runtime;
* it only works with an XmlPullParser returned from a compiled resource
* (R.<em>something</em> file.)
* 
* @see Context#getSystemService
*/
public abstract class LayoutInflater {
	…………………………
   private static final String TAG_MERGE = "merge";
   private static final String TAG_INCLUDE = "include";
   private static final String TAG_1995 = "blink";
   private static final String TAG_REQUEST_FOCUS = "requestFocus";
```

除了TAG_1995这个属性我们可能用的比较少以外，其他的我们应该都用过，如果标签为TAG_1995的话，我们初始化的是BlinkLayout这个类，我们通过源码可以看一下其实他就是个FrameLayout

## BlinkLayout

```
    private static class BlinkLayout extends FrameLayout {
        private static final int MESSAGE_BLINK = 0x42;
        private static final int BLINK_DELAY = 500;
 
        private boolean mBlink;
        private boolean mBlinkState;
        private final Handler mHandler;
```

由于这个平时用的比较少，我们这里就不在介绍，我们主要看一下其他的三个属性，TAG_MERGE 属性我们一般自定义控件的时候为了减少层级用的表较多一些，TAG_INCLUDE 属性主要是为了减少代码复用，TAG_REQUEST_FOCUS 主要在EditText中为了获取焦点的时候用到。好了，上面只是一个简单的开头，下面我们就来详细分析inflate这个方法，以下是源码

## inflate

```
    /**
     * Inflate a new view hierarchy from the specified xml resource. Throws
     * {@link InflateException} if there is an error.
     * 
     * @param resource ID for an XML layout resource to load (e.g.,
     *        <code>R.layout.main_page</code>)
     * @param root Optional view to be the parent of the generated hierarchy.
     * @return The root View of the inflated hierarchy. If root was supplied,
     *         this is the root View; otherwise it is the root of the inflated
     *         XML file.
     */
    public View inflate(int resource, ViewGroup root) {
        return inflate(resource, root, root != null);
    }
```

我们看他的另一个重构的方法

```
    public View inflate(int resource, ViewGroup root, boolean attachToRoot) {
        if (DEBUG) System.out.println("INFLATING from resource: " + resource);
        XmlResourceParser parser = getContext().getResources().getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
```

他首先通过parser解析得到XmlResourceParser对象，

## getLayout

```
    public XmlResourceParser getLayout(int id) throws NotFoundException {
        return loadXmlResourceParser(id, "layout");
    }
```

具体怎么解析的，大家可以看他的源码，这里就不在介绍，我们主要来看一看inflate这个方法的实现，他是我们主要分析的对象，下面是全部源码，都没有任何删除，并且还加了注释，其中他里面还有一些调用的方法，这个不作为我们重点分析的对象，待会我们简单提一下就行了，有兴趣的大家可以自己去分析

## inflate

```
    /**
     * Inflate a new view hierarchy from the specified XML node. Throws
     * {@link InflateException} if there is an error.
     * <p>
     * <em><strong>Important</strong></em>   For performance
     * reasons, view inflation relies heavily on pre-processing of XML files
     * that is done at build time. Therefore, it is not currently possible to
     * use LayoutInflater with an XmlPullParser over a plain XML file at runtime.
     * 
     * @param parser XML dom node containing the description of the view
     *        hierarchy.
     * @param root Optional view to be the parent of the generated hierarchy (if
     *        <em>attachToRoot</em> is true), or else simply an object that
     *        provides a set of LayoutParams values for root of the returned
     *        hierarchy (if <em>attachToRoot</em> is false.)
     * @param attachToRoot Whether the inflated hierarchy should be attached to
     *        the root parameter? If false, root is only used to create the
     *        correct subclass of LayoutParams for the root view in the XML.
     * @return The root View of the inflated hierarchy. If root was supplied and
     *         attachToRoot is true, this is root; otherwise it is the root of
     *         the inflated XML file.
     */
    public View inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");
 
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            /*mConstructorArgs是一个长度为2的数组，第一个存放的是Context，第二个存放的是attrs，主要在
             * createView中用到*/
            Context lastContext = (Context)mConstructorArgs[0];
            mConstructorArgs[0] = mContext;
            View result = root;
 
            try {
                // Look for the root node.
                int type;
                /*不停循环，直到找到开始的标签位置*/
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }
                /*没找到则抛出异常*/
                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }
 
                final String name = parser.getName();
                
                if (DEBUG) {
                    System.out.println("**************************");
                    System.out.println("Creating root view: "
                            + name);
                    System.out.println("**************************");
                }
                /*（1）这个标签我们一般自定义控件的时候用的比较多，就是说<merge/>必须要有一个父控件，
                 * 否则就会抛出异常，待会我们给大家演示一下，这先标记为我们的第一个问题*/
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }
                    /*（2）rInflate方法主要是把parser解析出来的标签和attrs属性共同创建的View
                     * 添加到父控件root中，待会我们在具体分析*/
                    rInflate(parser, root, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                    View temp;
                    if (TAG_1995.equals(name)) {
                    	/*这个我们上面说了，他其实就是继承的FrameLayout，相当于new了一个FrameLayout*/
                        temp = new BlinkLayout(mContext, attrs);
                    } else {
                    	/*（3）这个方法根据名字大家可能也能猜出来就是根据name来创建View，事实证明我们
                    	 * 的猜测是正确的，不过这回他不是new出来一个View，他是通过类name和attrs，
                    	 * 然后由loadClass加载这个类，最后newInstance这个View*/
                        temp = createViewFromTag(root, name, attrs);
                    }
 
                    ViewGroup.LayoutParams params = null;
 
                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        /*（4）如果root不为空，获取根布局的宽和高属性，*/
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                        	/*（5）如果root不为空，且attachToRoot为false，把根布局的长和宽设置
                        	 * 给创建的temp View，其实我们根据他的注释大概也能猜测如果没有attach，
                        	 * 这把上面获得的参数params给我们创建的temp，如果attach则在下面把它给add进去*/
 
                            temp.setLayoutParams(params);
                        }
                    }
 
                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }
                    // Inflate all children under temp
                    /*这个同上面的问题（2），我们待会再讲*/
                    rInflate(parser, temp, attrs, true);
                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }
 
                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                    	/*把我们创建的temp add到root中*/
                        root.addView(temp, params);
                    }
 
                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                    	/*没有把temp add到temp中*/
                        result = temp;
                    }
                }
 
            } catch (XmlPullParserException e) {
                InflateException ex = new InflateException(e.getMessage());
                ex.initCause(e);
                throw ex;
            } catch (IOException e) {
                InflateException ex = new InflateException(
                        parser.getPositionDescription()
                        + ": " + e.getMessage());
                ex.initCause(e);
                throw ex;
            } finally {
                // Don't retain static reference on context.
            	/*最后把数组的第一个context设置为lastContext，然后第二个attrs设置为空*/
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;
            }
 
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
 
            return result;
        }
    }
```

不要急，等我们把所有问题全部分析完成之后再做最后的猜想和验证，我们先来看上面第二个问题的源码

## rInflate

```
    /**
     * Recursive method used to descend down the xml hierarchy and instantiate
     * views, instantiate their children, and then call onFinishInflate().
     */
    void rInflate(XmlPullParser parser, View parent, final AttributeSet attrs,
            boolean finishInflate) throws XmlPullParserException, IOException {
 
        final int depth = parser.getDepth();
        int type;
 
        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {
 
            if (type != XmlPullParser.START_TAG) {
                continue;
            }
 
            final String name = parser.getName();
            
            if (TAG_REQUEST_FOCUS.equals(name)) {
                parseRequestFocus(parser, parent);
            } else if (TAG_INCLUDE.equals(name)) {
                if (parser.getDepth() == 0) {
                    throw new InflateException("<include /> cannot be the root element");
                }
                parseInclude(parser, parent, attrs);
            } else if (TAG_MERGE.equals(name)) {
                throw new InflateException("<merge /> must be the root element");
            } else if (TAG_1995.equals(name)) {
                final View view = new BlinkLayout(mContext, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                rInflate(parser, view, attrs, true);
                viewGroup.addView(view, params);                
            } else {
                final View view = createViewFromTag(parent, name, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                rInflate(parser, view, attrs, true);
                viewGroup.addView(view, params);
            }
        }
 
        if (finishInflate) parent.onFinishInflate();
    }
```

通过源码我们可以看出如果有<merge />标签，那么他必须为第一个，否则会抛出异常。最后根据是否加载完成来调用onFinishInflate方法，这个方法有时候也是我们在自定义控件的时候常用到的。如果标签为TAG_REQUEST_FOCUS，那么他的代码很简单，主要就这一行parent.requestFocus();，源码就不在贴出，如果标签为TAG_INCLUDE就会调用下面方法

## parseInclude

```
    private void parseInclude(XmlPullParser parser, View parent, AttributeSet attrs)
            throws XmlPullParserException, IOException {
 
        int type;
 
        if (parent instanceof ViewGroup) {
            final int layout = attrs.getAttributeResourceValue(null, "layout", 0);
            if (layout == 0) {
                final String value = attrs.getAttributeValue(null, "layout");
                if (value == null) {
                    throw new InflateException("You must specifiy a layout in the"
                            + " include tag: <include layout=\"@layout/layoutID\" />");
                } else {
                    throw new InflateException("You must specifiy a valid layout "
                            + "reference. The layout ID " + value + " is not valid.");
                }
            } else {
                final XmlResourceParser childParser =
                        getContext().getResources().getLayout(layout);
 
                try {
                    final AttributeSet childAttrs = Xml.asAttributeSet(childParser);
 
                    while ((type = childParser.next()) != XmlPullParser.START_TAG &&
                            type != XmlPullParser.END_DOCUMENT) {
                        // Empty.
                    }
 
                    if (type != XmlPullParser.START_TAG) {
                        throw new InflateException(childParser.getPositionDescription() +
                                ": No start tag found!");
                    }
 
                    final String childName = childParser.getName();
 
                    if (TAG_MERGE.equals(childName)) {
                        // Inflate all children.
                        rInflate(childParser, parent, childAttrs, false);
                    } else {
                        final View view = createViewFromTag(parent, childName, childAttrs);
                        final ViewGroup group = (ViewGroup) parent;
 
                        // We try to load the layout params set in the <include /> tag. If
                        // they don't exist, we will rely on the layout params set in the
                        // included XML file.
                        // During a layoutparams generation, a runtime exception is thrown
                        // if either layout_width or layout_height is missing. We catch
                        // this exception and set localParams accordingly: true means we
                        // successfully loaded layout params from the <include /> tag,
                        // false means we need to rely on the included layout params.
                        ViewGroup.LayoutParams params = null;
                        try {
                            params = group.generateLayoutParams(attrs);
                        } catch (RuntimeException e) {
                            params = group.generateLayoutParams(childAttrs);
                        } finally {
                            if (params != null) {
                                view.setLayoutParams(params);
                            }
                        }
 
                        // Inflate all children.
                        rInflate(childParser, view, childAttrs, true);
 
                        // Attempt to override the included layout's android:id with the
                        // one set on the <include /> tag itself.
                        TypedArray a = mContext.obtainStyledAttributes(attrs,
                            com.android.internal.R.styleable.View, 0, 0);
                        int id = a.getResourceId(com.android.internal.R.styleable.View_id, View.NO_ID);
                        // While we're at it, let's try to override android:visibility.
                        int visibility = a.getInt(com.android.internal.R.styleable.View_visibility, -1);
                        a.recycle();
 
                        if (id != View.NO_ID) {
                            view.setId(id);
                        }
 
                        switch (visibility) {
                            case 0:
                                view.setVisibility(View.VISIBLE);
                                break;
                            case 1:
                                view.setVisibility(View.INVISIBLE);
                                break;
                            case 2:
                                view.setVisibility(View.GONE);
                                break;
                        }
 
                        group.addView(view);
                    }
                } finally {
                    childParser.close();
                }
            }
        } else {
            throw new InflateException("<include /> can only be used inside of a ViewGroup");
        }
 
        final int currentDepth = parser.getDepth();
        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > currentDepth) && type != XmlPullParser.END_DOCUMENT) {
            // Empty
        }
    }
```

其实原理和上面差不多，我们就不在一一分析，我们主要来看一下这里面的一个方法createViewFromTag，也是在上面我们提到的问题（3），他的源码如下，我们仔细看

## createViewFromTag

```
    /*
     * default visibility so the BridgeInflater can override it.
     */
    View createViewFromTag(View parent, String name, AttributeSet attrs) {
        if (name.equals("view")) {
            name = attrs.getAttributeValue(null, "class");
        }
 
        if (DEBUG) System.out.println("******** Creating view: " + name);
 
        try {
            View view;
            if (mFactory2 != null) view = mFactory2.onCreateView(parent, name, mContext, attrs);
            else if (mFactory != null) view = mFactory.onCreateView(name, mContext, attrs);
            else view = null;
 
            if (view == null && mPrivateFactory != null) {
                view = mPrivateFactory.onCreateView(parent, name, mContext, attrs);
            }
            
            if (view == null) {
                if (-1 == name.indexOf('.')) {
                	/*(1)*/
                    view = onCreateView(parent, name, attrs);
                } else {
                	/*(2)*/
                    view = createView(name, null, attrs);
                }
            }
 
            if (DEBUG) System.out.println("Created view is: " + view);
            return view;
 
        } catch (InflateException e) {
            throw e;
 
        } catch (ClassNotFoundException e) {
            InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class " + name);
            ie.initCause(e);
            throw ie;
 
        } catch (Exception e) {
            InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class " + name);
            ie.initCause(e);
            throw ie;
        }
    }
```

最上面的Factory其实就是个接口，我们主要看我标注的（1）和（2）两部分，其实最终还是（1）调用了（2），我们看源码

## onCreateView

```
    protected View onCreateView(View parent, String name, AttributeSet attrs)
            throws ClassNotFoundException {
        return onCreateView(name, attrs);
    }
```

接着看

```
    protected View onCreateView(String name, AttributeSet attrs)
            throws ClassNotFoundException {
        return createView(name, "android.view.", attrs);
    }
```

我们看到了在第二个参数传入的是"android.view."，回想我们上一篇讲的怎么获取LayoutInflater的时候，说到其实我们获取的是PhoneLayoutInflater，我们在PhoneLayoutInflater中看到这样几行代码，代码如下，已经略去一部分

## PhoneLayoutInflater

```
public class PhoneLayoutInflater extends LayoutInflater {
    private static final String[] sClassPrefixList = {
        "android.widget.",
        "android.webkit.",
        "android.app."
    };
     ………………………………
    /** Override onCreateView to instantiate names that correspond to the
        widgets known to the Widget factory. If we don't find a match,
        call through to our super class.
    */
    @Override protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {
        for (String prefix : sClassPrefixList) {
            try {
                View view = createView(name, prefix, attrs);
                if (view != null) {
                    return view;
                }
            } catch (ClassNotFoundException e) {
                // In this case we want to let the base class take a crack
                // at it.
            }
        }
 
        return super.onCreateView(name, attrs);
    }
```

看懂了吧，其实他就是传入的包名加一个点，目的就是拼接一个完整的控件路径然后在初始化，我们看源码

## createView

```
    public final View createView(String name, String prefix, AttributeSet attrs)
            throws ClassNotFoundException, InflateException {
        Constructor<? extends View> constructor = sConstructorMap.get(name);
        Class<? extends View> clazz = null;
        …………………………
            if (constructor == null) {
                // 如果prefix为null说明name是个完整的路径
                clazz = mContext.getClassLoader().loadClass(
                        prefix != null ? (prefix + name) : name).asSubclass(View.class);
                …………………………
                constructor = clazz.getConstructor(mConstructorSignature);
                sConstructorMap.put(name, constructor);
            } else {
            	…………………………
                        clazz = mContext.getClassLoader().loadClass(
                                prefix != null ? (prefix + name) : name).asSubclass(View.class);
                        
            	…………………………
            }
        	/*这就是上面inflate方法中我们提到的长度为2的一个数组，通过下面的newInstance创建一个新的View*/
            Object[] args = mConstructorArgs;
            args[1] = attrs;
 
            final View view = constructor.newInstance(args);
            …………………………
            return view;
            …………………………
 
    }
```

接着我们分析上面（4）个问题的源码，这个比较简单

## generateLayoutParams

```
    public LayoutParams generateLayoutParams(AttributeSet attrs) {
        return new LayoutParams(getContext(), attrs);
    }
```

我们继续

```
        public LayoutParams(Context c, AttributeSet attrs) {
            TypedArray a = c.obtainStyledAttributes(attrs, R.styleable.ViewGroup_Layout);
            setBaseAttributes(a,
                    R.styleable.ViewGroup_Layout_layout_width,
                    R.styleable.ViewGroup_Layout_layout_height);
            a.recycle();
        }
```

我们在来看一看setBaseAttributes的源码

## setBaseAttributes

```
        protected void setBaseAttributes(TypedArray a, int widthAttr, int heightAttr) {
            width = a.getLayoutDimension(widthAttr, "layout_width");
            height = a.getLayoutDimension(heightAttr, "layout_height");
        }
```

看到了吧，就是设置LayoutParams参数的宽和高。其实无论是new一个View还是newInstance一个View，那么AttributeSet属性并不能设置View的宽和高，因为我们知道View的宽高是由ViewRootImpl的performTraversals方法最先调用，然后再由自身的Measure方法和LayoutParams参数测量之后共同决定的，待会给大家总结的时候，可能就会明白为什么View的背景和其他属性都起作用，但设置宽和高却没有任何作用，原因就在这

好了，到目前为止我们把LayoutInflater及它的inflate方法都分析完了，那么下面我们就猜想，然后在验证上面遗留的一些问题，猜想如下

```
    public View inflate(int resource, ViewGroup root, boolean attachToRoot) 
```

为了便于大家理解，我们来定义两个术语，根布局和root，意思虽然差不多，但所指却截然不同，根布局就是指我们自定义layout的根View，root是指我们inflate的那个类
（1）如果merge为根布局root不能为空，否则抛异常
（以下3个问题是在没有merge的情况下讨论）
（2）如果root为空，attachToRoot无论是true或false都没有任何意义，root与他没有任何关系，但根布局的属性保留了下来，宽高没有保留，（因为上面我们分析的createViewFromTag方法在创建temp的时候只是把attrs传了过去，通过newInstance创建了View对象，但并没有把LayoutParams设置进去，所以宽高并没有任何作用）
（3）如果root不为空，且attachToRoot为true，则根布局的所有属性都会保留下来，包括宽高，并且root还是布局文件的父类(通过源码我们可以看到 if (root != null && attachToRoot)则root.addView(temp, params)，就是把temp和参数全部设置进去)
（4）如果root不为空，且attachToRoot为false，则根布局的所有属性和宽高都会保留下来，但不会add到root中，所以root也不会是布局文件的父类（我们通过最上面代码中的问题5可以看到，就是把LayoutParams设置进去了，但没有add）
好了，到目前为止，LayoutInflater及它的inflate方法我们都分析完了，那么究竟对不对，在下一篇中我们将举例为大家验证。