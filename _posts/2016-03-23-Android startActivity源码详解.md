---
layout: post
title: "Android startActivity源码详解"
subtitle: 'Android startActivity源码详解'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “黑发不知勤学早，白首方悔读书迟。”
	--颜真卿

## 正文

在Android页面跳转的时候，我们一般都会调用startActivity(Intent intent)，调用之后就会跳转到下一个页面了，究竟是怎么跳转的，今天我们结合源码来给大家分析一下，我们知道，Android的Activity的创建，启动都非常复杂，很多类都比较庞大，我们把主要的树干理清就行了，不必过多的纠结于它的繁枝细节，下面我就结合源码为大家一一分析，我们进入源码

## startActivity

```
    /**
     * Same as {@link #startActivity(Intent, Bundle)} with no options
     * specified.
     *
     * @param intent The intent to start.
     *
     * @throws android.content.ActivityNotFoundException
     *
     * @see {@link #startActivity(Intent, Bundle)}
     * @see #startActivityForResult
     */
    @Override
    public void startActivity(Intent intent) {
        startActivity(intent, null);
    }
```

调用了另一个重构方法，我们继续

```
    @Override
    public void startActivity(Intent intent, Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }
```

其实上面两个方法最终调用的都是同一个方法，我们直接看它的源码，注释比较多，我就不在贴出

## startActivityForResult

```
    public void startActivityForResult(Intent intent, int requestCode, Bundle options) {
        if (mParent == null) {
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }
 
            final View decor = mWindow != null ? mWindow.peekDecorView() : null;
            if (decor != null) {
                decor.cancelPendingInputEvents();
            }
            // TODO Consider clearing/flushing other event sources and events for child windows.
        } else {
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                // Note we want to go through this method for compatibility with
                // existing applications that may have overridden it.
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
```

我们看主要分支，我们只看mParent ==null的那部分，如果ar!=null就会调用下面一个回调方法，也是我们平时常用的

## onActivityResult

```
@Override
	protected void onActivityResult(int requestCode, int resultCode, Intent data) {
	// TODO Auto-generated method stub
	super.onActivityResult(requestCode, resultCode, data);
}
```

下面我们主要来分析mInstrumentation.execStartActivity这行代码，我们找到他的源码查看

## execStartActivity

```
    public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Fragment target,
        Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    if (am.match(who, null, intent)) {
                        am.mHits++;
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess();
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mWho : null,
                        requestCode, 0, null, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
        }
        return null;
    }
```

我们主要看最后几行，我们知道ActivityManagerNative是一个抽象类，并且继承了Binder，我们来看一下他的源码

## ActivityManagerNative

```
public abstract class ActivityManagerNative extends Binder implements IActivityManager
{
    /**
     * Cast a Binder object into an activity manager interface, generating
     * a proxy if needed.
     */
    static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
 
        return new ActivityManagerProxy(obj);
    }
 
    /**
     * Retrieve the system's default/global activity manager.
     */
    static public IActivityManager getDefault() {
        return gDefault.get();
    }
    …………………………
}
```

在上面我们还看到checkStartActivityResult这个方法，我们来看一下他的源码

## checkStartActivityResult

```
    /*package*/ static void checkStartActivityResult(int res, Object intent) {
        if (res >= ActivityManager.START_SUCCESS) {
            return;
        }
        
        switch (res) {
            case ActivityManager.START_INTENT_NOT_RESOLVED:
            case ActivityManager.START_CLASS_NOT_FOUND:
                if (intent instanceof Intent && ((Intent)intent).getComponent() != null)
                    throw new ActivityNotFoundException(
                            "Unable to find explicit activity class "
                            + ((Intent)intent).getComponent().toShortString()
                            + "; have you declared this activity in your AndroidManifest.xml?");
                throw new ActivityNotFoundException(
                        "No Activity found to handle " + intent);
                …………………………
        }
    }
```

看到没，这里面就有我们经常见到的，类没找到或者没有在AndroidManifest中注册的异常，我们再来看一下IActivityManager的源码，

## IActivityManager

```
public interface IActivityManager extends IInterface {
    public int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho,
            int requestCode, int flags, String profileFile,
            ParcelFileDescriptor profileFd, Bundle options) throws RemoteException;
    public int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho,
            int requestCode, int flags, String profileFile,
            ParcelFileDescriptor profileFd, Bundle options, int userId) throws RemoteException;
    public WaitResult startActivityAndWait(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho,
            int requestCode, int flags, String profileFile,
            ParcelFileDescriptor profileFd, Bundle options, int userId) throws RemoteException;
    		…………………………
    public Intent registerReceiver(IApplicationThread caller, String callerPackage,
            IIntentReceiver receiver, IntentFilter filter,
            String requiredPermission, int userId) throws RemoteException;
    public void unregisterReceiver(IIntentReceiver receiver) throws RemoteException;
    		…………………………  
    public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType, int userId) throws RemoteException;
    public int stopService(IApplicationThread caller, Intent service,
            String resolvedType, int userId) throws RemoteException;
    		…………………………
    public int bindService(IApplicationThread caller, IBinder token,
            Intent service, String resolvedType,
            IServiceConnection connection, int flags, int userId) throws RemoteException;
    public boolean unbindService(IServiceConnection connection) throws RemoteException;
    public void publishService(IBinder token,
            Intent intent, IBinder service) throws RemoteException;
    …………………………
}
```

看到没，是不是有很多方法我们都比较熟悉，并且我们还看到它继承了IInterface接口，在我们平时实现跨进程IPC调用的时候是不是要继承这个接口，所以我们基本可以判定ActivityManagerNative其实就是一个跨进程调用的类，并且ActivityManagerService是他的真正实现类，我们可以看一下

## ActivityManagerService

```
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
```

他是个final类型，不能被继承，我们来看它的startActivity方法

## startActivity

```
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags,
            String profileFile, ParcelFileDescriptor profileFd, Bundle options) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode,
                startFlags, profileFile, profileFd, options, UserHandle.getCallingUserId());
    }
```

继续，我们来看一下startActivityAsUser的源码

## startActivityAsUser

```
    @Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags,
            String profileFile, ParcelFileDescriptor profileFd, Bundle options, int userId) {
        enforceNotIsolatedCaller("startActivity");
        userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
                false, true, "startActivity", null);
        // TODO: Switch to user app stacks here.
        return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profileFile, profileFd,
                null, null, options, userId, null);
    }
```

我们找到mStackSupervisor对象，其实他就是ActivityStackSupervisor，也是个final类型，我们简单看一下

## ActivityStackSupervisor

```
public final class ActivityStackSupervisor implements DisplayListener {
```

我们找到他的startActivityMayWait方法，代码也不少，我们找关键的看

## startActivityMayWait

```
    final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags, String profileFile,
            ParcelFileDescriptor profileFd, WaitResult outResult, Configuration config,
            Bundle options, int userId, IActivityContainer iContainer) {
        	…………………………
            int res = startActivityLocked(caller, intent, resolvedType, aInfo, resultTo, resultWho,
                    requestCode, callingPid, callingUid, callingPackage, startFlags, options,
                    componentSpecified, null, container);
        	…………………………
            return res;
        }
    }
```

继续，我们在找到startActivityLocked方法，他的代码量更多，同样，我们还是找关键的来看

## startActivityLocked

```
    final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo, IBinder resultTo,
            String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage, int startFlags, Bundle options,
            boolean componentSpecified, ActivityRecord[] outActivity, ActivityContainer container) {
        int err = ActivityManager.START_SUCCESS;
        …………………………
        err = startActivityUncheckedLocked(r, sourceRecord, startFlags, true, options);
        …………………………
        return err;
    }
```

我们继续找到startActivityUncheckedLocked这个方法，代码量是一个比一个长，没关系，我们还是看关键部分

## startActivityUncheckedLocked

```
    final int startActivityUncheckedLocked(ActivityRecord r,
            ActivityRecord sourceRecord, int startFlags, boolean doResume,
            Bundle options) {
    	…………………………
     
        } else if (sourceRecord.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
            launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
        } else if (r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE
                || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK) {
            launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
        }
        …………………………
            ActivityRecord intentActivity = r.launchMode != ActivityInfo.LAUNCH_SINGLE_INSTANCE
                    ? findTaskLocked(r)
                    : findActivityLocked(intent, r.info);
            if (intentActivity != null) {
                if (r.task == null) {
                    r.task = intentActivity.task;
                }
                targetStack = intentActivity.task.stack;
                targetStack.mLastPausedActivity = null;
               …………………………
                if ((launchFlags&Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                    intentActivity = targetStack.resetTaskIfNeededLocked(intentActivity, r);
                }
                …………………………
                targetStack.resumeTopActivityLocked(null);
                …………………………
 
        return ActivityManager.START_SUCCESS;
    }
```

看到没，这里面有很多启动模式，和我们设置的启动模式是有关的，关于Android的4中启动模式不作为我们这篇介绍的重点，我们来看一下上面倒数第二行代码，我们找到targetStack这个对象，发现他其实就是ActivityStack，他也是final类型的，我们可以看一下

## ActivityStack

```
/**
 * State and management of a single stack of activities.
 */
final class ActivityStack {
```

我们找到resumeTopActivityLocked这个方法

## resumeTopActivityLocked

```
    final boolean resumeTopActivityLocked(ActivityRecord prev) {
        return resumeTopActivityLocked(prev, null);
    }
```

发现他被重构了，我们继续查看他的源码，发现代码量也多的吓死人，还是那样，我们捡主要的看

```
    final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
    	…………………………
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
    	…………………………
        return true;
    }
```

通过查看源码我们发现mStackSupervisor就是我们之前的ActivityStackSupervisor，就等于说我们绕了一圈又绕回来了，没关系，那我们继续查看他的startSpecificActivityLocked方法，这个代码量比较少，我就全部给他贴出来

## startSpecificActivityLocked

```
    void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);
 
        r.task.stack.setLaunchTime(r);
 
        if (app != null && app.thread != null) {
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
                    // Don't add this if it is a platform component that is marked
                    // to run in multiple processes, because this is actually
                    // part of the framework so doesn't make sense to track as a
                    // separate apk in the process.
                    app.addPackage(r.info.packageName, mService.mProcessStats);
                }
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }
 
            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }
 
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }
```

我们主要来看一下try中的代码块，找到realStartActivityLocked的源码，这个代码量也不少，我们还是找主要的看

## realStartActivityLocked

```
    final boolean realStartActivityLocked(ActivityRecord r,
            ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {
    	…………………………
        mWindowManager.setAppVisibility(r.appToken, true);
        …………………………
            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info,
                    new Configuration(mService.mConfiguration), r.compat,
                    app.repProcState, r.icicle, results, newIntents, !andResume,
                    mService.isNextTransitionForward(), profileFile, profileFd,
                    profileAutoStop);
            …………………………
        } catch (RemoteException e) {}
        …………………………
        return true;
    }
```

看到没，大家是不是有点兴奋，我们知道上面的thread其实就是ActivityThread，在之前我们大致说过在ActivityThread有个Main方法，里面有一些looper及一些其他数据的初始化等，这个以后我们在介绍，我们先来看看scheduleLaunchActivity这个方法

## scheduleLaunchActivity

```
        public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, CompatibilityInfo compatInfo,
                int procState, Bundle state, List<ResultInfo> pendingResults,
                List<Intent> pendingNewIntents, boolean notResumed, boolean isForward,
                String profileName, ParcelFileDescriptor profileFd, boolean autoStopProfiler) {
        	…………………………
            sendMessage(H.LAUNCH_ACTIVITY, r);
        }
```

我们知道上面的H其实就是一个Handler，我们来看一下他的源码，代码量也比较多，我们就找我们需要的看

## H

```
private class H extends Handler {
        public static final int LAUNCH_ACTIVITY         = 100;
        …………………………
        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                    ActivityClientRecord r = (ActivityClientRecord)msg.obj;
 
                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                } break;
               …………………………
            }
```

我们上面调用了handleLaunchActivity这个方法，接着我们在继续找这个方法，随便查看一下他的源码

## handleLaunchActivity

```
    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    	…………………………
        Activity a = performLaunchActivity(r, customIntent);
 
        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            Bundle oldState = r.state;
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed);
            …………………………
                r.paused = true;
            }
        } else {
        	…………………………
        }
    }
```

我们看到performLaunchActivity返回一个Activity，之前估计我们很多都不知道Activity究竟是怎么初始化的，这回大家终于明白了吧，我们随便看一下他的源码

## performLaunchActivity

```
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    	…………………………
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            //创建Activity
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
          …………………………
 
        try {
            //创建Application
        	Application app = r.packageInfo.makeApplication(false, mInstrumentation);
            …………………………
            if (activity != null) {
                Context appContext = createBaseContextForActivity(r, activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                //调用Activity的attach方法，这个我们在前面讲的时候也多次提及
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config);
 
                …………………………
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                   // 设置主题
                	activity.setTheme(theme);
                }
 
                //调用Activity的OnCreate方法，
                mInstrumentation.callActivityOnCreate(activity, r.state);
               …………………………
                if (!r.activity.mFinished) {
                    //调用Activity的onStart方法
                	activity.performStart();
                    r.stopped = false;
                }
                if (!r.activity.mFinished) {
                    if (r.state != null) {
                       //调用Activity的OnRestoreInstanceState方法
                    	mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }
               …………………………
        return activity;
    }
```

Activity是通过Instrumentation这个类创建的，我们可以看一下，代码非常简短

## newActivity

```
    public Activity newActivity(ClassLoader cl, String className,
            Intent intent)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
        return (Activity)cl.loadClass(className).newInstance();
    }
```

我们可以看到performLaunchActivity方法调用了Activity的onCreate方法和onStart方法，然后我们在看上面的handleLaunchActivity方法在调用完performLaunchActivity方法的时候又调用了handleResumeActivity方法。我们也可以顺便看一下他的源码

## handleResumeActivity

```
    final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward,
            boolean reallyResume) {
    	…………………………
    	//调用了Activity的performResume方法，然后执行onResume方法
        ActivityClientRecord r = performResumeActivity(token, clearHide);
 
        if (r != null) {
            final Activity a = r.activity;
            …………………………
            boolean willBeVisible = !a.mStartedActivity;
            …………………………
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                }
                …………………………
                r.activity.mVisibleFromServer = true;
                mNumVisibleActivities++;
                if (r.activity.mVisibleFromClient) {
                    //调用Activity的makeVisible方法
                	r.activity.makeVisible();
                }
            }
            …………………………
    }
```

经过上面一步步的分析，Activity的onCreat方法，onStart方法和onResume方法就全部都已经启动，一个全新的Activity就展现在我们面前，OK，到目前为止startActivity的方法就全部已经分析完毕