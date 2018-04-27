---
title: 安卓measure的原理和流程(2)
layout: post
date: 2017-08-05
categories: 架构
tags: [架构, 分析, Android, measure]
---

上回我们讲到，WindowManager创建了一个ViewRootImpl，并把Activity的DecorView加入其中。今天我们继续讲什么情况下会触发measure。
##### 触发measure的条件
什么时候，DecorView以及其所有子View会调用一次measure呢？正常情况下， 如果其中一个View的尺寸或者相对其parent的位置发生了变化，就会通知其parent，然后一级一级传递到ViewRootImpl，触发发一次全局的measure。View在ViewGroup中的位置和尺寸，是由LayoutParams指定的，每一个View都有其对应的LayoutParams，通过直接修改View的LayoutParams或者重新调用setLayoutParams来设置新的LayoutParams就可以达到修改View的位置和尺寸的目的。
  
    public void setLayoutParams(ViewGroup.LayoutParams params) {
        if (params == null) {
            throw new NullPointerException("Layout parameters cannot be null");
        }
        mLayoutParams = params;
        resolveLayoutParams();
        if (mParent instanceof ViewGroup) {
            ((ViewGroup) mParent).onSetLayoutParams(this, params);
        }
        requestLayout();
    }

从这个函数中，我们看到设置了LayoutParams后，首先调用了其parent ViewGroup的onSetLayoutParams，定义如下：

    /** @hide */
    protected void onSetLayoutParams(View child, LayoutParams layoutParams) {
    }

默认什么也不干，所以构成消息传递机制的应该不是这个函数。那么就剩下我们经常用到的requestLayout了。
  
    /**
     * Call this when something has changed which has invalidated the
     * layout of this view. This will schedule a layout pass of the view
     * tree. This should not be called while the view hierarchy is currently in a layout
     * pass ({@link #isInLayout()}. If layout is happening, the request may be honored at the
     * end of the current layout pass (and then layout will run again) or after the current
     * frame is drawn and the next layout occurs.
     *
     * <p>Subclasses which override this method should call the superclass method to
     * handle possible request-during-layout errors correctly.</p>
     */
    @CallSuper
    public void requestLayout() {
        if (mMeasureCache != null) mMeasureCache.clear();

        if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
            // Only trigger request-during-layout logic if this is the view requesting it,
            // not the views in its parent hierarchy
            ViewRootImpl viewRoot = getViewRootImpl();
            if (viewRoot != null && viewRoot.isInLayout()) {
                if (!viewRoot.requestLayoutDuringLayout(this)) {
                    return;
                }
            }
            mAttachInfo.mViewRequestingLayout = this;
        }

        mPrivateFlags |= PFLAG_FORCE_LAYOUT;
        mPrivateFlags |= PFLAG_INVALIDATED;

        if (mParent != null && !mParent.isLayoutRequested()) {
            mParent.requestLayout();
        }
        if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
            mAttachInfo.mViewRequestingLayout = null;
        }
    }

注释中讲了三件事情

1. 当一个View的layout发生变化的时候请调用这个函数。
2. 不要在layout过程中调用这个函数，如果非要调用的话，会在这次layout结束之后再次出发layout。
3. 如果在自定义的View中重载了这个函数，请务必调用super.requestLayout()。
  
从代码中的下面这一句
  
	if (mParent != null && !mParent.isLayoutRequested()) {
        mParent.requestLayout();
    }
可以看出来，这个函数会不断调用其parent的requestLayout，最后传递到DecorView, 并传递给ViewRootImpl。那ViewRootImpl的requestLayout是怎么做的呢？
  
    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }

在这里，先判断了一下是否正在处理一个layout的请求或者正在执行layout的操作，如果是的话就忽略这一次requestLayout，否则就把mLayoutRequested设置为true，并且调用scheduleTraversals()。
  
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }

在scheduleTraversals函数中，我们看一个postCallback，被post的Runnable的名字叫mTraversalRunnable，查看一下它的定义。
  
    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

终于找到正主了，在requestLayout之后，ViewRootImpl会在下一次空闲时调用doTraversal。而doTraversal就是measure和layout的起点。
  
找到了起点，接下来就该看看measure是怎么一步一步从ViewRootImpl传递到每一个View和ViewGroup的了。