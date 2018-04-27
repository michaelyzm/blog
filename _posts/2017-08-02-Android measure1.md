---
title: 安卓measure的原理和流程
layout: post
date: 2017-08-02
categories: 架构
tags: [架构, 分析, Android, measure]
---

了解一套UI系统，最重要的就是把布局的流程和绘制的流程了解清楚。而对于安卓来说，布局分为measure和layout两个部分。本文准备解释一下measure的原理和流程。
  
### measure的起点
一般情况下，安卓的UI都是绘制在Activity上面，那么Activity是怎么把UI绘制出来的呢？这要从Activity的创建说起。
  
Activity是应用程序用户界面的载体，它的实例由系统的ActivityManager负责管理，并根据其运行状态维护其信息。当一个Activity被创建的时候，系统会调用它的attach方法，在这个方法里，会为Activity创建一个Window对象，类型是PhoneWindow，然后通过Window对象的setWindowManager方法将它托管给系统的WindowManager服务。这个Window就是Activity显示的基础。
  
随后，我们在使用Activity时会在onCreate方法中调用setContentView方法将定义好的用户界面放置到Activity上。在调用setContentView方法时，Activity调用了Window对象的setContentView方法。在PhoneWindow的setContentView方法中，它会先创建一个DecorView对象作为UI的根容器，然后调用LayoutInflator将我们传入的layout文件转换为View或者ViewGroup的实例，放置在DecorView中。到这里为止，都是我们比较熟悉的，因为我们平时经常会在Activity中调用getWindow()或者getDecorView()这两个函数。但是涉及到measure，就要提一个我们在debug的callstack里边经常看到但是却从来用不到的一个类ViewRootImp。
  
首先，Activity是在哪里被创建出来的呢？当我们执行startActivity操作的时候，实际上是ActivityThread接受了这个指令，然后执行handleLaunchActivity。
  
    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
        ...

        // Initialize before creating the activity
        WindowManagerGlobal.initialize();

        Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            reportSizeConfigurations(r);
            Bundle oldState = r.state;
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);

            ...
        } else {
            ...
        }
    }

从这里可以看到，首先嗲用WindowManagerGlobal.initialize()初始化了Window Manager Global， 然后执行了performLaunchActivity创建了一个新的Activity，最后调用了handleResumeActivity来启动Acitivity。我们接下来看看handleResumeActivity做了什么。
  
    final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
        ActivityClientRecord r = mActivities.get(token);
        ...

        if (r != null) {
            final Activity a = r.activity;

            ...

            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                
                ...

                if (a.mVisibleFromClient && !a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                }

            ...
        } else {
            ...
        }
    }

在handleResumeActivity函数里，调用了WindowManager的addView操作，将DecorView加入了其中。而此时a.getWindowManager()返回的实例是通过调用(WindowManager)context.getSystemService(Context.WINDOW_SERVICE)获得的全局WindowManager，其类型是WindowManagerImpl。WindowManagerImpl的addView函数实现如下。
  

   private void addView(View view, ViewGroup.LayoutParams params,
            CompatibilityInfoHolder cih, boolean nest) {
        if (false) Log.v("WindowManager", "addView view=" + view);

        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException(
                    "Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams
                = (WindowManager.LayoutParams)params;
        
        ViewRootImpl root;
        View panelParentView = null;
        
        synchronized (this) {
            
            ...
            
            root = new ViewRootImpl(view.getContext());
            
            ...
            
            if (mViews == null) {
                index = 1;
                mViews = new View[1];
                mRoots = new ViewRootImpl[1];
                mParams = new WindowManager.LayoutParams[1];
            } else {
                index = mViews.length + 1;
                Object[] old = mViews;
                mViews = new View[index];
                System.arraycopy(old, 0, mViews, 0, index-1);
                old = mRoots;
                mRoots = new ViewRootImpl[index];
                System.arraycopy(old, 0, mRoots, 0, index-1);
                old = mParams;
                mParams = new WindowManager.LayoutParams[index];
                System.arraycopy(old, 0, mParams, 0, index-1);
            }
            index--;

            mViews[index] = view;
            mRoots[index] = root;
            mParams[index] = wparams;
        }
        // do this last because it fires off messages to start doing things
        root.setView(view, wparams, panelParentView);
    }
  
在将DecorView添加到WindowManager时，用DecorView的Context创建了一个新的ViewRootImpl，然后将要添加的view用ViewRootImpl的setView方法绑定到了新创建的ViewRootImpl中。
  
到此为止，我们的Activity终于有了自己的ViewRootImpl，那么ViewRootImpl要怎样来让其中的View和ViewGroup们计算他们的尺寸并且报告上来呢？请听下会分解。