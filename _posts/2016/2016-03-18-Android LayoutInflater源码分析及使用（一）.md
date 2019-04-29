---
layout: post
title: "Android LayoutInflater源码分析及使用（一）"
subtitle: 'Android LayoutInflater源码分析及使用'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “三更灯火五更鸡，正是男儿读书时。”
	--颜真卿

## 正文

说到LayoutInflater可能大家首先想到的是加载layout，一般我们会习惯性的调用View中的这个静态方法

```
    public static View inflate(Context context, int resource, ViewGroup root) {
        LayoutInflater factory = LayoutInflater.from(context);
        return factory.inflate(resource, root);
    }
```

或者直接得到LayoutInflater对象，然后调用它的下面方法

## inflate

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

可是LayoutInflater是个抽象方法，我们是不能直接初始化的，一般都是这样来得到这个对象的

```
 LayoutInflater LayoutInflater =
                (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
```

我们通过查看源码，发现上面的方法在Context中也是也抽象方法，并没有真正的实现

## getSystemService

```
    public abstract Object getSystemService(String name);
```

今天我们就来一步步分析怎么得到LayoutInflater这个对象的，至于他的inflate方法我们也是经常遇到的，这篇我们先跳过，下一篇在作详解，上面说到我们在Context中发现他是一个抽象类，那么我们就去它的子类中查找，在ContextWrapper中我们终于找到了，

```
    @Override
    public Object getSystemService(String name) {
        return mBase.getSystemService(name);
    }
```

那么我们在看看mBase是什么，可能很失望，它就是传进来的Context

## attachBaseContext

```
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
```

我们继续跟踪，找到ContextWrapper的一个子类ContextThemeWrapper，发现getSystemService方法正好有我们要找的LayoutInflater，可是却没什么用，因为它需要实例化之后才能克隆，

```
    @Override public Object getSystemService(String name) {
        if (LAYOUT_INFLATER_SERVICE.equals(name)) {
            if (mInflater == null) {
                mInflater = LayoutInflater.from(mBase).cloneInContext(this);
            }
            return mInflater;
        }
        return mBase.getSystemService(name);
    }
```

我们继续找它的下一个子类Activity，找到了，可是并没有我们要找的LayoutInflater

```
    @Override
    public Object getSystemService(String name) {
        if (getBaseContext() == null) {
            throw new IllegalStateException(
                    "System services not available to Activities before onCreate()");
        }
 
        if (WINDOW_SERVICE.equals(name)) {
            return mWindowManager;
        } else if (SEARCH_SERVICE.equals(name)) {
            ensureSearchManager();
            return mSearchManager;
        }
        return super.getSystemService(name);
    }
```

饶了一大圈白费力气结果还是没找到。不要气馁，回到刚才我们说的ContextWrapper类中的getSystemService方法，我们来看一下mBase是怎么传进来的，最终我们在Activity的这个方法找到了它

## attach

```
    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config) {
        attachBaseContext(context);
        …………………………
    }
```

在之前我们讲过Activity的attach方法是在ActivityThreaad的performLaunchActivity方法中传进来的，

## performLaunchActivity

```
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
 		…………………………	
                Context appContext = createBaseContextForActivity(r, activity);
                …………………………
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config);
                …………………………
              
        return activity;
    }
```

也就是说ContextWrapper中的mBase其实就是在这里传进来的，我们继续查看createBaseContextForActivity方法

## createBaseContextForActivity

```
    private Context createBaseContextForActivity(ActivityClientRecord r,
            final Activity activity) {
        ContextImpl appContext = ContextImpl.createActivityContext(this, r.packageInfo, r.token);
        ……………………
        return baseContext;
    }
```

看到了吧，其实Context的真正实现类是ContextImpl 

```
/**
 * Common implementation of Context API, which provides the base
 * context object for Activity and other application components.
 */
class ContextImpl extends Context {
```

找到它就好办了，我们来查找一下getSystemService方法

```
    @Override
    public Object getSystemService(String name) {
        ServiceFetcher fetcher = SYSTEM_SERVICE_MAP.get(name);
        return fetcher == null ? null : fetcher.getService(this);
    }
```

看到没，其实SYSTEM_SERVICE_MAP就是个HashMap，那它究竟是怎么保存进去的，我们继续查看，发现有一个方法叫registerService，在这里面保存的

## registerService

```
    private static void registerService(String serviceName, ServiceFetcher fetcher) {
        if (!(fetcher instanceof StaticServiceFetcher)) {
            fetcher.mContextCacheIndex = sNextPerContextServiceCacheIndex++;
        }
        SYSTEM_SERVICE_MAP.put(serviceName, fetcher);
    }
```

我们还发现了有一个静态代码块，存进去的比较多，我们随便找几个我们平时比较常见的看看

```
    static {
       …………………………
        registerService(ACTIVITY_SERVICE, new ServiceFetcher() {
                public Object createService(ContextImpl ctx) {
                    return new ActivityManager(ctx.getOuterContext(), ctx.mMainThread.getHandler());
                }});
 
        registerService(ALARM_SERVICE, new ServiceFetcher() {
                public Object createService(ContextImpl ctx) {
                    IBinder b = ServiceManager.getService(ALARM_SERVICE);
                    IAlarmManager service = IAlarmManager.Stub.asInterface(b);
                    return new AlarmManager(service, ctx);
                }});
 
        registerService(AUDIO_SERVICE, new ServiceFetcher() {
                public Object createService(ContextImpl ctx) {
                    return new AudioManager(ctx);
                }});
 
	…………………………
 
        registerService(INPUT_SERVICE, new StaticServiceFetcher() {
                public Object createStaticService() {
                    return InputManager.getInstance();
                }});
 
        registerService(DISPLAY_SERVICE, new ServiceFetcher() {
                @Override
                public Object createService(ContextImpl ctx) {
                    return new DisplayManager(ctx.getOuterContext());
                }});
 
        registerService(INPUT_METHOD_SERVICE, new StaticServiceFetcher() {
                public Object createStaticService() {
                    return InputMethodManager.getInstance();
                }});
 
	………………………………
	………………………………
 
        registerService(LAYOUT_INFLATER_SERVICE, new ServiceFetcher() {
                public Object createService(ContextImpl ctx) {
                    return PolicyManager.makeNewLayoutInflater(ctx.getOuterContext());
                }});
 
	………………………………
	………………………………
 
        // Note: this was previously cached in a static variable, but
        // constructed using mMainThread.getHandler(), so converting
        // it to be a regular Context-cached service...
        registerService(POWER_SERVICE, new ServiceFetcher() {
                public Object createService(ContextImpl ctx) {
                    IBinder b = ServiceManager.getService(POWER_SERVICE);
                    IPowerManager service = IPowerManager.Stub.asInterface(b);
                    return new PowerManager(ctx.getOuterContext(),
                            service, ctx.mMainThread.getHandler());
                }});
 
        registerService(SEARCH_SERVICE, new ServiceFetcher() {
                public Object createService(ContextImpl ctx) {
                    return new SearchManager(ctx.getOuterContext(),
                            ctx.mMainThread.getHandler());
                }});
 
	…………………………
 
        registerService(TELEPHONY_SERVICE, new ServiceFetcher() {
                public Object createService(ContextImpl ctx) {
                    return new TelephonyManager(ctx.getOuterContext());
                }});
 
	…………………………
 
        registerService(VIBRATOR_SERVICE, new ServiceFetcher() {
                public Object createService(ContextImpl ctx) {
                    return new SystemVibrator(ctx);
                }});
 
        registerService(WALLPAPER_SERVICE, WALLPAPER_FETCHER);
 
        registerService(WIFI_SERVICE, new ServiceFetcher() {
                public Object createService(ContextImpl ctx) {
                    IBinder b = ServiceManager.getService(WIFI_SERVICE);
                    IWifiManager service = IWifiManager.Stub.asInterface(b);
                    return new WifiManager(ctx.getOuterContext(), service);
                }});
 
        registerService(WIFI_P2P_SERVICE, new ServiceFetcher() {
                public Object createService(ContextImpl ctx) {
                    IBinder b = ServiceManager.getService(WIFI_P2P_SERVICE);
                    IWifiP2pManager service = IWifiP2pManager.Stub.asInterface(b);
                    return new WifiP2pManager(service);
                }});
	…………………………
 
        registerService(CAMERA_SERVICE, new ServiceFetcher() {
            public Object createService(ContextImpl ctx) {
                return new CameraManager(ctx);
            }
        });
	…………………………
    }
```

然后我们在找我们需要的LayoutInflater，就是两条杠之间的，我们找到PolicyManager类，然后找到相应的方法

## makeNewLayoutInflater

```
    public static LayoutInflater makeNewLayoutInflater(Context context) {
        return sPolicy.makeNewLayoutInflater(context);
    }
```

sPolicy其实就是Policy，我们来查看一下它的源码，找到相应的方法

```
    public LayoutInflater makeNewLayoutInflater(Context context) {
        return new PhoneLayoutInflater(context);
    }
```

看这里我们终于明白了，其实我们通过getSystemService得到的LayoutInflater 就是PhoneLayoutInflater，其实他继承的是LayoutInflater，我们可以看一下

```
public class PhoneLayoutInflater extends LayoutInflater {
```

好了，到此为止LayoutInflater LayoutInflater =(LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);这个方法我们已经分析完了，其实他得到的就是PhoneLayoutInflater这个类，有时间我们在来分析一下它的inflate方法，也是我们最常用到的一个方法。