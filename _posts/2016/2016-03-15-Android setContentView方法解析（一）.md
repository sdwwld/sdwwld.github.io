---
layout: post
title: "Android setContentView方法解析（一）"
subtitle: 'Android setContentView方法源码深入解析'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “纸上得来终觉浅，绝知此事要躬行。”
	--陆游

## 正文
在Activity的生命周期onCreate中，我们一般都习惯性的调用setContentView(int layoutResID)方法，把布局文件加载到页面上来，下面我们就来通过源码一步步的分析怎么加载的。

## setContentView

在Activity中，调用的是Window的setContentView

```
    public void setContentView(int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
```

我们通过源码发现Window是一个抽象类，而且 getWindow()返回的只是一个mWindow。

## getWindow

```
    public Window getWindow() {
        return mWindow;
    }
```

那么mWindow是在什么地方初始化的，我们继续查看源码，最终发现是在一个叫attach的方法中调用，可是最终我们没有发现调用attach的方法，其实这个方法是在ActivityThread这个类中调用的，这个类很重要，包括在main方法中初始化Looper，在这里先不说，以后在介绍

## performLaunchActivity

```
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
…………
            Activity activity = null;
            activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);

         …………

            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
…………

                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config);
…………
        return activity;
    }
```

接着我们在attach方法中看到这样一行代码

```
mWindow = PolicyManager.makeNewWindow(this);
```

继续查看，找到PolicyManager这个类，

## PolicyManager

```
 private static final String POLICY_IMPL_CLASS_NAME =
        "com.android.internal.policy.impl.Policy";
…………
            Class policyClass = Class.forName(POLICY_IMPL_CLASS_NAME);
            sPolicy = (IPolicy)policyClass.newInstance();
  …………
    public static Window makeNewWindow(Context context) {
        return sPolicy.makeNewWindow(context);
    }
```

继续，在Policy这个类中我们找到

## makeNewWindow

```
   public Window makeNewWindow(Context context) {
        return new PhoneWindow(context);
    }
```

也就是说最初我们找到mWindow就是PhoneWindow，我们继续，不要闲着，public class PhoneWindow extends Window implements MenuBuilder.Callback 我们通过源码发现PhoneWindow其实就是Window。先回到我们刚才说的getWindow().setContentView(layoutResID);这个方法，其实就是调用PhoneWindow的setContentView(layoutResID);方法，我们通过源码继续查看

## setContentView

```
    @Override
    public void setContentView(int layoutResID) {
        if (mContentParent == null) {
            installDecor();
        } else {
            mContentParent.removeAllViews();
        }
        mLayoutInflater.inflate(layoutResID, mContentParent);
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
```

我们先简单分析一下下半部分，如果Callback不为空且没有被销毁的情况下会调用onContentChanged()方法，通过源码我们发现其实Activity是实现了Callback接口的

```
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback 
```

而且在onContentChanged()方法中是空实现，什么都没做。

```
    public void onContentChanged() {
    }
```

我们再来分析上半部分，如果mContentParent 不为空的话，首先移除所有的子View，removeAllViews方法是ViewGroup类中，View中没有，我们可以先大致看一下，以后在做详细介绍

## removeAllViews

```
    /**
     * Call this method to remove all child views from the
     * ViewGroup.
     * 
     * <p><strong>Note:</strong> do not invoke this method from
     * {@link #draw(android.graphics.Canvas)}, {@link #onDraw(android.graphics.Canvas)},
     * {@link #dispatchDraw(android.graphics.Canvas)} or any related method.</p>
     */
    public void removeAllViews() {
        removeAllViewsInLayout();
        requestLayout();
        invalidate(true);
    }
```

如果mContentParent 为空的话，调用  installDecor();进行初始化，接着我们看一下  installDecor();方法的具体实现，

## installDecor

```
    private void installDecor() {
        if (mDecor == null) {
            mDecor = generateDecor();
          …………
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);

            mTitleView = (TextView)findViewById(com.android.internal.R.id.title);
         …………
                mActionBar = (ActionBarView) findViewById(com.android.internal.R.id.action_bar);

…………
        }
    }
```

首先判断mDecor是否为空，如果为空就调用 generateDecor();我们接着看它具体怎么实现

## generateDecor

```
    protected DecorView generateDecor() {
        return new DecorView(getContext(), -1);
    }
```

我们发现就一行代码，那么DecorView又是什么，我们继续查看，

```
private final class DecorView extends FrameLayout implements RootViewSurfaceTaker 
```

终于找到了，原来继承的是FrameLayout ，看到这里可能有的人会恍然大悟，因为我们知道在Activity的setContentView中加载的根布局其实就是FrameLayout ，这里的FrameLayout 有两部分组成，一部分是系统自定义的title，一部分是content，就是我们自己定义的布局，还有，在之前如果我们不需要title的话，我们会在AcdroidMainfest中 android:theme="@android:style/Theme.NoTitleBar"，或者在activity中设置requestWindowFeature(Window.FEATURE_NO_TITLE);但是要设置在setContentView之前，好了，上面的代码我们先分析到这，以后有时间还会接着继续写。欢迎拍砖。

