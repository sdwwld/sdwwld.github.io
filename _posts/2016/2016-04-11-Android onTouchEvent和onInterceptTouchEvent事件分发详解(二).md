---
layout: post
title: "Android onTouchEvent和onInterceptTouchEvent事件分发详解(二)"
subtitle: 'Android onTouchEvent和onInterceptTouchEvent事件分发详解'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “壮心未与年俱老，死去犹能作鬼雄。”
	--陆游

## 正文

通过上一篇的简单演示，我们知道默认情况下只有Button和ImageButton的onTouchEvent返回的是true，表示事件被消耗。这一篇我们结合demo来分析一下它的源码，我们知道在Activity中也有dispatchTouchEvent和onTouchEvent方法，其实他最终调用的还是Viewgroup的方法，正常的逻辑流程是当我们点击屏幕的时候，事件的传递顺序是Activity→Window→View，我们可以看一下，在Activity中的dispatchTouchEvent方法。

## dispatchTouchEvent

```
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```

通过上一篇打印的log我们知道默认情况下Activity的这两个方法返回的都是false，所以在默认情况下如果子控件都不处理，在Activity中dispatchTouchEvent是调onTouchEvent方法的，如果子控件有一个处理了则Activity的OnTouchEvent就不会再调用，且onTouchEvent方法默认返回也为false，通过前面<a href="https://androidboke.com/2016/03/15/Android-setContentView%E6%96%B9%E6%B3%95%E8%A7%A3%E6%9E%90-%E4%B8%80" target="_blank">Android setContentView方法解析（一）</a>我们知道Activity中的getWindow()得到的其实就是PhoneWindow，我们看一下

## superDispatchTouchEvent

```
    @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }
```

在之前我们也说过mDecor就是DecorView，它继承的就是FrameLayout，我们可以看一下

## DecorView

```
    private final class DecorView extends FrameLayout implements RootViewSurfaceTaker {
        …………………………
        public boolean superDispatchTouchEvent(MotionEvent event) {
            return super.dispatchTouchEvent(event);
        }
     }
```

dispatchTouchEvent方法是事件分发，在Activity和View中是没有拦截事件的（onInterceptTouchEvent），只有ViewGroup中才有，默认情况下如果传到最终的View不消耗事件（比如TextView）则会往上抛，如果父控件也一直没处理则最终交给Activity的onTouchEvent方法处理，我们把上一篇DispatchActivity中的dispatchTouchEvent和onTouchEvent方法中的log保留，其他的则全部注释掉，我们看一下打印结果

![](/img/blog/2016/20160413133931142.png)

我们看到如果都不处理的话，则最终会交给Activity的onTouchEvent来处理，且dispatchTouchEvent也返回false，表示最终事件没有被消耗，但被Activity的OnTouchEvent给处理了，我们再来看另一种情况，我们知道默认情况下Button是消耗事件的，我们把TextView_Child中的TextView给为button，或者让原来TextView的onTouchEvent方法返回true，都是可以的，我们在看一下打印的log，

![](/img/blog/2016/20160413134455754.png)

我们看到DispatchActivity中dispatchTouchEvent方法返回了true，表示表示控件已经被消耗。但onTouchEvent方法没有执行，这是因为当事件传到最终的View的时候onTouchEvent返回了true，表示事件已经被消耗，所以就不会再往上抛了，所以DispatchActivity的onTouchEvent方法就没有被调用。

如果是自定义控件的话，一般情况下我们是很少在Activity中对事件进行处理，所以我们就暂时先不研究Activity中dispatchTouchEvent和onTouchEvent方法，我们只需要知道事件最初是由他传递的就行了。我们来看一下View和ViewGroup中的这两个方法。因为View不能包含子View，所以他只有dispatchTouchEvent和onTouchEvent这两个方法

```
    public boolean dispatchTouchEvent(MotionEvent event) {
     …………………………
    }
    public boolean onTouchEvent(MotionEvent event) {
         …………………………
    }
```

而ViewGroup可以包含子View，所有多了一个拦截的方法

```
    public boolean dispatchTouchEvent(MotionEvent event) {
     …………………………
    }
    public boolean onTouchEvent(MotionEvent event) {
         …………………………
    }
    public boolean onInterceptTouchEvent(MotionEvent ev) {
         …………………………
    }
```

dispatchTouchEvent表示事件的分发，onTouchEvent表示事件的处理，onInterceptTouchEvent表示事件的拦截。我们来看一下View中的dispatchTouchEvent源码

## dispatchTouchEvent

```
    public boolean dispatchTouchEvent(MotionEvent event) {
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(event, 0);
        }
 
        if (onFilterTouchEventForSecurity(event)) {
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
		    //onTouch的优先级要比onTouchEvent高，如果添加了OnTouchListener事件，并且onTouch
		    //返回true，表示事件已经被处理，就不会再往下执行，否则就执行下面的onTouchEvent事
		    //件，且onTouchEvent的返回值决定了dispatchTouchEvent的返回值
                return true;
            }
 
            if (onTouchEvent(event)) {
                return true;
            }
        }
 
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
        }
        return false;
    }
```

我们知道View是不能包含子View的，所以他的dispatchTouchEvent方法实现起来也并不那么复杂，只是做了一些简单的处理，上面我们看到onTouch的返回值可能会影响到onTouchEvent的调用，我们在看一下onTouchEvent的源码，

## onTouchEvent

```
    public boolean onTouchEvent(MotionEvent event) {
        final int viewFlags = mViewFlags;
 
        if ((viewFlags & ENABLED_MASK) == DISABLED) {
		//如果不可点击则return；
            if (event.getAction() == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return (((viewFlags & CLICKABLE) == CLICKABLE ||
                    (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));
        }
 
        if (mTouchDelegate != null) {
		//mTouchDelegate是一个TouchDelegate，按字面翻译是委托触摸，其实就是扩大触摸范围的一个辅助类
		//比如说，如果我们的一个控件非常小，那么触摸起来可能非常困难，使用这个类可以扩大它的触摸范围
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }
 
        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
            switch (event.getAction()) {
                case MotionEvent.ACTION_UP:
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
				…………………………
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
				// 执行onClick事件，我们可以看到onClick事件是在ACTION_UP的时候触发的，
				// 就是说在DOWN，MOVE的时候都不会触发。
                                    performClick();
                                }
                            }
				…………………………
                case MotionEvent.ACTION_DOWN:
                    mHasPerformedLongPress = false;
 
                    if (performButtonActionOnTouchDown(event)) {
		    //DOWN事件是否被处理过，如果处理则不会往下执行
                        break;
                    }
			 …………………………
                    if (isInScrollingContainer) {
                        mPrivateFlags |= PFLAG_PREPRESSED;
                        if (mPendingCheckForTap == null) {
                            mPendingCheckForTap = new CheckForTap();
                        }
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                    } else {
                        setPressed(true);
			 //在DOWN的时候检查是否长按
                        checkForLongClick(0);
                    }
                    break;
 
                case MotionEvent.ACTION_CANCEL:
			…………………………
                case MotionEvent.ACTION_MOVE:
			…………………………
            }
            return true;
        }
 
        return false;
    }
```

这个方法处理的稍微复杂一些，它才是真正的处理事件的方法，包含onClick和onLongClick。下面我们再来看一下ViewGroup中的那三个方法，首先第一个onInterceptTouchEvent

## onInterceptTouchEvent

```
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        return false;
    }
```

我们看到代码很简单，就返回了一个默认值false，表示事件不拦截。我们再来看一下onTouchEvent这个方法，在ViewGroup中是没有的，但是ViewGroup继承的是View，所以ViewGroup也继承了View的onTouchEvent方法，这个我们就不在分析了，我们重点看一下ViewGroup中的dispatchTouchEvent方法，

## dispatchTouchEvent

```
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
        }
		//事件是否被消耗
        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;
 
            // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
		//事件为DOWN的时候清除触摸事件并重置触摸状态，这两个方法内部都会调用clearTouchTargets
		//方法，通过循环把mFirstTouchTarget回收并置为null。
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }
 
            // Check for interception.
            final boolean intercepted;
	    //如果为DOWN事件或者mFirstTouchTarget不为null，则执行下面判断事件是否被拦截，mFirstTouchTarget
	    //就是TouchTarget，是以一种单项链表的形式存在，类似于消息机制的Message。默认事件是不拦截的，会在
	    //下面的addTouchTarget方法中赋值，所以mFirstTouchTarget默认情况下是不为空的。如果拦截事件，则
	    //mFirstTouchTarget为空，事件的拦截有两种方式，一种就是我们说的让onInterceptTouchEvent返回true
	    //另一种是在子类中调用父类的requestDisallowInterceptTouchEvent(boolean disallowIntercept)方法，
	    //但无论怎么拦截，控件的DOWN事件都是会被触发的，因为DOWN是一系列事件的开端，如果事件拦截，除了
	    //DOWN事件会执行onInterceptTouchEvent方法以外，其他的MOVE，UP则都不会再调用，如果不拦截则
	    //onInterceptTouchEvent方法会一直被调用这个我们暂且记为问题(1),待会在位大家演示，
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
		    //判断是否调用requestDisallowInterceptTouchEvent方法进行了拦截
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
			//调用onInterceptTouchEvent拦截方法
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }
 
            // Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;
 
            // Update list of touch targets for pointer down, if needed.
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) {// 如果不拦截则执行下面方法
                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;
 
                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
		    //移除所有的触摸标记
                    removePointersFromTouchTargets(idBitsToAssign);
 
                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final View[] children = mChildren;
 
                        final boolean customOrder = isChildrenDrawingOrderEnabled();
			//通过循环，找到接收事件的View
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = customOrder ?
                                    getChildDrawingOrder(childrenCount, i) : i;
                            final View child = children[childIndex];
			    //如果不能接受触摸事件或者触摸的X,Y坐标不在此View中则跳过
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                continue;
                            }
 
                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
				//如果在mFirstTouchTarget中找到，则结束循环
                                break;
                            }
 
                            resetCancelNextUpFlag(child);
			    //dispatchTransformedTouchEvent这个方法执行递归调用，返回false表示事件没有被
			    //消耗,则下面的addTouchTarget方法就不会被执行，mFirstTouchTarget为空，则上面
			    //的拦截事件判断intercepted为true，表示事件拦截，则后续的MOVE，UP都不会再执行
			    //onInterceptTouchEvent方法，且上面的判断也不成立，该View也不会再执行MOVE，
			    //事件。如果返回true，表示事件被消耗,则下面的addTouchTarget方法执行，
			    // mFirstTouchTarget不再为null，则上面的条件满足，后续的事件都可以处理
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                mLastTouchDownIndex = childIndex;
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
				//找到可以接收事件的View，然后添加到mFirstTouchTarget中，mFirstTouchTarget
				//是一个单项链表，有一个next属性，直接添加到next中
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }
                        }
                    }
 
                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }
 
            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
		// 如果mFirstTouchTarget为空，则说明事件没有被消耗，我们看到下面的注释
		//如果没有可触摸的事件，则对待他像一个普通的View，看到下面第三个方法传入
		//的为null，表示调用View的dispatchTouchEvent
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
		//如果不为空，则找到消耗了事件的控件
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
				//通过while循环，递归调用dispatchTransformedTouchEvent
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }
 
            // Update list of touch targets for pointer up or cancel, if needed.
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
		    //如果cancel或up则重置触摸状态
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
		//如果为UP则事件结束，移除所有的触摸事件
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }
 
        if (!handled && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
        }
        return handled;
    }
 
```

接下来我们在看看上面的dispatchTransformedTouchEvent方法

## dispatchTransformedTouchEvent

```
    /**
     * Transforms a motion event into the coordinate space of a particular child view,
     * filters out irrelevant pointer ids, and overrides its action if necessary.
     * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
     */
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;
	…………………………
        final MotionEvent transformedEvent;
	…………………………
        // Perform any necessary transformations and dispatch.
        if (child == null) {
		//如果child为空，则表示mFirstTouchTarget为空，就是没找到可消耗的控件，或者事件
		//被拦截，则调用super的dispatchTouchEvent方法,因为ViewGroup的父类是View，即调用
		//View的dispatchTouchEvent方法，我们知道在View中默认的是调用onTouchEvent方法的，
		//就像普通的处理一样。其实说直接一点就是如果自己拦截了就会调用自己的onTouchEvent
		//方法，因为上面我们分析过，如果事件拦截了则mFirstTouchTarget为空，就会调用下面
		//的方法，或者子控件没有消耗事件，在子控件的onTouchEvent方法中返回了false，则也会向
		//上抛，调用自己的onTouchEvent方法.
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }
		//如果找到，则传递给他的子View，这个View有可能是最终消耗的那个View，也有可能是
		//一个ViewGroup，如果是View在上面我们分析过OnTouchListener不拦截的话，就会调用
		//onTouchEvent方法，所以他的返回值也最终决定了dispatchTransformedTouchEvent方法
		//的返回值，如果是ViewGroup，则会通过不断的递归调用dispatchTouchEvent方法，找到
		//最终消耗的控件
            handled = child.dispatchTouchEvent(transformedEvent);
        }
 
        // Done.
        transformedEvent.recycle();
	//
        return handled;
    }
```

## 总结

OK，到目前为止，Android的事件分发机制已经分析的差不多了，我们来总结一下

1，事件的传递顺序是从Activity→Window→View

2，默认情况下View中的dispatchTouchEvent方法会调用onTouchEvent方法（如果OnTouchListener拦截就不会再调用onTouchEvent方法）

3，View中事件的顺序为dispatchTouchEvent→OnTouchListener→onTouchEvent→OnLongClickListener（在DOWN中触发）→OnClickListener（在UP中触发）

4，在ViewGroup中事件的触发顺序为dispatchTouchEvent→onInterceptTouchEvent→onTouchEvent

5，默认情况下Button，ImageButton等Button的子控件都是会消耗事件的，即onTouchEvent默认返回true，而ImageView，TextView，LinearLayout等一些控件默认是不消耗事件的，即onTouchEvent默认返回为false

6，事件是从上往下传递的，如果其中的一个onInterceptTouchEvent返回了true，则表示事件拦截，此后的MOVE，UP都不会再调用onInterceptTouchEvent方法，然后调用自己的onTouchEvent方法，则它下面的控件都不会再获取触发事件

7，如果在子控件中不让父控件拦截，可以调用父控件的requestDisallowInterceptTouchEvent(boolean disallowIntercept)方法，

8，如果子控件不处理，则会往上抛，交给父控件处理，如果都不处理，默认会抛到Activity的onTouchEvent方法，Activity的onTouchEvent方法默认是消耗控件的，且默认返回为false，DOWM，ＭＯＶＥ，ＵＰ都会执行。

９，如果有一个控件处理了事件，则后续的一系列事件（MOVE，UP）也都会执行。
