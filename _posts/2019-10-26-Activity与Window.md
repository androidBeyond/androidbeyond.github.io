---
layout:     post
title:      Activity与Window
subtitle:   一篇文章看明白 Activity 与 Window 与 View 之间的关系
date:       2019-10-26
author:     duguma
header-img: img/article-bg.jpg
top: false
no-catalog: true
tags:
    - Android
    - 系统组件
---  

<h2><a id="_15"></a>概述</h2> 
<p>我们知道 Activity 启动后就可以看到我们写的 Layout 布局界面&#xff0c;Activity 从 setContentView() 到显示中间做了什么呢&#xff1f;下面我们就来分析下这个过程。</p> 
<p>本文主要对于以下问题进行分析&#xff1a;</p> 
<ul><li>Window 是什么&#xff1f;</li><li>Activity 与 PhoneWindow 与 DecorView 之间什么关系&#xff1f;</li></ul> 
<h2><a id="onCreate__Window__26"></a>onCreate() - Window 创建过程</h2> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTgwMzAxMTAyMjExNDkz" alt="这里写图片描述" /></p> 
<p>在 Activity 创建过程中执行 scheduleLaunchActivity() 之后便调用到了 handleLaunchActivity() 方法。</p> 
<p>ActivityThread.handleLaunchActivity()&#xff1a;</p> 
<pre><code class="prism language-Java">private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    handleConfigurationChanged(null, null);
    //初始化 WindowManagerService&#xff0c;主要是获取到 WindowManagerService 代理对象
    WindowManagerGlobal.initialize();
    //详情见下面分析
    Activity a &#61; performLaunchActivity(r, customIntent);

    if (a !&#61; null) {
        r.createdConfig &#61; new Configuration(mConfiguration);
        //详见下面分析 [onResume() - Window 显示过程]
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished &amp;&amp; !r.startsNotResumed);
        ...
    }
    ...
}

...

private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    Activity activity &#61; null;
    //获取 ClassLoader
    java.lang.ClassLoader cl &#61; r.packageInfo.getClassLoader();
    //创建目标 Activity 对象
    activity &#61; mInstrumentation.newActivity(
            cl, component.getClassName(), r.intent);
    StrictMode.incrementExpectedActivityCount(activity.getClass());
    r.intent.setExtrasClassLoader(cl);
    r.intent.prepareToEnterProcess();
    if (r.state !&#61; null) {
        r.state.setClassLoader(cl);
    }

    //创建 Application 对象
    Application app &#61; r.packageInfo.makeApplication(false, mInstrumentation);
    if (activity !&#61; null) {
        Context appContext &#61; createBaseContextForActivity(r, activity);
        CharSequence title &#61; r.activityInfo.loadLabel(appContext.getPackageManager());
        Configuration config &#61; new Configuration(mCompatConfiguration);
        //详情见下面分析
        activity.attach(appContext, this, getInstrumentation(), r.token,
                r.ident, app, r.intent, r.activityInfo, title, r.parent,
                r.embeddedID, r.lastNonConfigurationInstances, config,
                r.referrer, r.voiceInteractor);
        ...
        //回调 Activity.onCreate()
        if (r.isPersistable()) {
            mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
        } else {
            mInstrumentation.callActivityOnCreate(activity, r.state);
        }
        ...
    return activity;
}

...

final void attach(Context context, ActivityThread aThread, Instrumentation instr, IBinder token, int ident, Application application, Intent intent, ActivityInfo info, CharSequence title, Activity parent, String id, NonConfigurationInstances lastNonConfigurationInstances, Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
    attachBaseContext(context);

    mWindow &#61; new PhoneWindow(this); //创建 PhoneWindow
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    ...
    mApplication &#61; application; //所属的 Application
    ...
    //设置并获取 WindowManagerImpl 对象
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags &amp; ActivityInfo.FLAG_HARDWARE_ACCELERATED) !&#61; 0);
    if (mParent !&#61; null) {
        mWindow.setContainer(mParent.getWindow());
    }
    mWindowManager &#61; mWindow.getWindowManager();
    mCurrentConfig &#61; config;
}
</code></pre> 
<p>可看出 Activity 里新建一个 PhoneWindow 对象。在 Android 中&#xff0c;Window 是个抽象的概念&#xff0c; Android 中 Window 的具体实现类是 PhoneWindow&#xff0c;Activity 和 Dialog 中的 Window 对象都是 PhoneWindow。</p> 
<p>同时得到一个 WindowManager 对象&#xff0c;WindowManager 是一个抽象类&#xff0c;这个 WindowManager 的具体实现是在 WindowManagerImpl 中&#xff0c;对比 Context 和 ContextImpl。</p> 
<p>Window.setWindowManager()&#xff1a;</p> 
<pre><code class="prism language-Java">public void setWindowManager(WindowManager wm, IBinder appToken, String appName, boolean hardwareAccelerated) { 
    ...    
    mWindowManager &#61; ((WindowManagerImpl)wm).createLocalWindowManager(this);
    ...
}
</code></pre> 
<p>每个 Activity 会有一个 WindowManager 对象&#xff0c;这个 mWindowManager 就是和 WindowManagerService 进行通信&#xff0c;也是 WindowManagerService 识别 View 具体属于那个 Activity 的关键&#xff0c;创建时传入 IBinder 类型的 mToken。</p> 
<pre><code class="prism language-Java">mWindow.setWindowManager(..., mToken, ..., ...)
</code></pre> 
<p>这个 Activity 的 mToken&#xff0c;这个 mToken 是一个 IBinder&#xff0c;WindowManagerService 就是通过这个 IBinder 来管理 Activity 里的 View。</p> 
<p>回调 Activity.onCreate() 后&#xff0c;会执行 setContentView() 方法将我们写的 Layout 布局页面设置给 Activity。</p> 
<p>Activity.setContentView()&#xff1a;</p> 
<pre><code class="prism language-Java">public void setContentView(&#64;LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);        
    initWindowDecorActionBar();    
}
</code></pre> 
<p>PhoneWindow.setContentView()&#xff1a;</p> 
<pre><code class="prism language-Java">public void setContentView(int layoutResID) {
    ...    
    installDecor(); 
    ... 
}
</code></pre> 
<p>PhoneWindow.installDecor()&#xff1a;</p> 
<pre><code class="prism language-Java">private void installDecor() {    
//根据不同的 Theme&#xff0c;创建不同的 DecorView&#xff0c;DecorView 是一个 FrameLayout 
}
</code></pre> 
<p>这时只是创建了 PhoneWindow&#xff0c;和DecorView&#xff0c;但目前二者也没有任何关系&#xff0c;产生关系是在ActivityThread.performResumeActivity 中&#xff0c;再调用 r.activity.performResume()&#xff0c;调用 r.activity.makeVisible&#xff0c;将 DecorView 添加到当前的 Window 上。</p> 
<h2><a id="onResume__Window__163"></a>onResume() - Window 显示过程</h2> 
<p>Activity 与 PhoneWindow 与 DecorView 关系图&#xff1a;</p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTgwMzAxMTAyMjUyOTk1" alt="这里写图片描述" /></p> 
<p>ActivityThread.handleResumeActivity()&#xff1a;</p> 
<pre><code class="prism language-Java">final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume) {
    //执行到 onResume()
    ActivityClientRecord r &#61; performResumeActivity(token, clearHide);

    if (r !&#61; null) {
        final Activity a &#61; r.activity;
        boolean willBeVisible &#61; !a.mStartedActivity;
        ...
        if (r.window &#61;&#61; null &amp;&amp; !a.mFinished &amp;&amp; willBeVisible) {
            r.window &#61; r.activity.getWindow();
            View decor &#61; r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm &#61; a.getWindowManager();
            WindowManager.LayoutParams l &#61; r.window.getAttributes();
            a.mDecor &#61; decor;
            l.type &#61; WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |&#61; forwardBit;
            if (a.mVisibleFromClient) {
                a.mWindowAdded &#61; true;
                wm.addView(decor, l);
            }

        }
        ...
        if (!r.activity.mFinished &amp;&amp; willBeVisible
                &amp;&amp; r.activity.mDecor !&#61; null &amp;&amp; !r.hideForNow) {
            ...
            mNumVisibleActivities&#43;&#43;;
            if (r.activity.mVisibleFromClient) {
                //添加视图&#xff0c;详见下面分析
                r.activity.makeVisible(); 
            }
        }

        //resume 完成
        if (reallyResume) {
              ActivityManagerNative.getDefault().activityResumed(token);
        }
    } else {
        ...
    }
}


public final ActivityClientRecord performResumeActivity(IBinder token, boolean clearHide) {
    ActivityClientRecord r &#61; mActivities.get(token);
    if (r !&#61; null &amp;&amp; !r.activity.mFinished) {
        ...
        //回调 onResume()
        r.activity.performResume();
        ...
    }
    return r;
}
</code></pre> 
<p>Activity.makeVisible()&#xff1a;</p> 
<pre><code class="prism language-Java">void makeVisible() {
    if (!mWindowAdded) {
        ViewManager wm &#61; getWindowManager();
        //详见下面分析
        wm.addView(mDecor, getWindow().getAttributes());
        mWindowAdded &#61; true;
    }
    mDecor.setVisibility(View.VISIBLE);
}
</code></pre> 
<p>WindowManager 的 addView 的具体实现在 WindowManagerImpl 中&#xff0c;而 WindowManagerImpl 的 addView 又会调用 WindowManagerGlobal.addView()。</p> 
<p>WindowManagerGlobal.addView()&#xff1a;</p> 
<pre><code class="prism language-Java">public void addView(View view, ViewGroup.LayoutParams params,Display display, Window parentWindow) {
    ...
    ViewRootImpl root &#61; new ViewRootImpl(view.getContext(), display);        
    view.setLayoutParams(wparams);    
    mViews.add(view);    
    mRoots.add(root);    
    mParams.add(wparams);        
    root.setView(view, wparams, panelParentView);
    ...
}
</code></pre> 
<p>这个过程创建一个 ViewRootImpl&#xff0c;并将之前创建的 DecoView 作为参数传入&#xff0c;以后 DecoView 的事件都由 ViewRootImpl 来管理了&#xff0c;比如&#xff0c;DecoView 上添加 View&#xff0c;删除 View。ViewRootImpl 实现了 ViewParent 这个接口&#xff0c;这个接口最常见的一个方法是 requestLayout()。</p> 
<p>ViewRootImpl 是个 ViewParent&#xff0c;在 DecoView 添加的 View 时&#xff0c;就会将 View 中的 ViewParent 设为 DecoView 所在的 ViewRootImpl&#xff0c;View 的 ViewParent 相同时&#xff0c;理解为这些 View 在一个 View 链上。所以每当调用 View 的 requestLayout()时&#xff0c;其实是调用到 ViewRootImpl&#xff0c;ViewRootImpl 会控制整个事件的流程。可以看出一个 ViewRootImpl 对添加到 DecoView 的所有 View 进行事件管理。</p> 
<p>ViewRootImpl&#xff1a;</p> 
<pre><code class="prism language-Java">public ViewRootImpl(Context context, Display display) {
    mContext &#61; context;
    //获取 IWindowSession 的代理类
    mWindowSession &#61; WindowManagerGlobal.getWindowSession();
    mDisplay &#61; display;
    mThread &#61; Thread.currentThread(); //主线程
    mWindow &#61; new W(this); 
    mChoreographer &#61; Choreographer.getInstance();
    ...
}

public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
  synchronized (this) {
    ...
    //通过 Binder 调用&#xff0c;进入 system 进程的 Session
    res &#61; mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
          getHostVisibility(), mDisplay.getDisplayId(),
          mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
          mAttachInfo.mOutsets, mInputChannel);
    ...
  }
}
</code></pre> 
<p>WindowManagerGlobal&#xff1a;</p> 
<pre><code class="prism language-Java">public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowSession &#61;&#61; null) {
            try {
                //获取 InputManagerService 的代理类
                InputMethodManager imm &#61; InputMethodManager.getInstance();
                //获取 WindowManagerService 的代理类
                IWindowManager windowManager &#61; getWindowManagerService();
                //经过 Binder 调用&#xff0c;最终调用 WindowManagerService
                sWindowSession &#61; windowManager.openSession(
                        new IWindowSessionCallback.Stub() {...},
                        imm.getClient(), imm.getInputContext());
            } catch (RemoteException e) {
                ...
            }
        }
        return sWindowSession
    }
}
</code></pre> 
<p>通过 binder 调用进入 system_server 进程。<br /> Session&#xff1a;</p> 
<pre><code class="prism language-Java">final class Session extends IWindowSession.Stub implements IBinder.DeathRecipient {

    public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs, int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets, Rect outOutsets, InputChannel outInputChannel) {
        //详情见下面
        return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
                outContentInsets, outStableInsets, outOutsets, outInputChannel);
    }
}
</code></pre> 
<p>WindowManagerService&#xff1a;</p> 
<pre><code class="prism language-Java">public int addWindow(Session session, IWindow client, int seq, WindowManager.LayoutParams attrs, int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets, Rect outOutsets, InputChannel outInputChannel) {
    ...
    WindowToken token &#61; mTokenMap.get(attrs.token);
    //创建 WindowState
    WindowState win &#61; new WindowState(this, session, client, token,
                attachedWindow, appOp[0], seq, attrs, viewVisibility, displayContent);
    ...
    //调整 WindowManager 的 LayoutParams 参数
    mPolicy.adjustWindowParamsLw(win.mAttrs);
    res &#61; mPolicy.prepareAddWindowLw(win, attrs);
    addWindowToListInOrderLocked(win, true);
    // 设置 input
    mInputManager.registerInputChannel(win.mInputChannel, win.mInputWindowHandle);
    // 创建 Surface 与 SurfaceFlinger 通信&#xff0c;详见下面[SurfaceFlinger 图形系统]
    win.attach();
    mWindowMap.put(client.asBinder(), win);
    
    if (win.canReceiveKeys()) {
        //当该窗口能接收按键事件&#xff0c;则更新聚焦窗口
        focusChanged &#61; updateFocusedWindowLocked(UPDATE_FOCUS_WILL_ASSIGN_LAYERS,
                false /*updateInputWindows*/);
    }
    assignLayersLocked(displayContent.getWindowList());
    ...
}
</code></pre> 
<p>Activity 中 Window 创建过程&#xff1a;</p> 
<p><img src="https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTgwMzAxMTAyMzE3ODMx" alt="这里写图片描述" /></p> 
<h2><a id="_358"></a>总结</h2> 
<ul><li>Window 是什么&#xff1f;</li></ul> 
<p>Window 是 Android 中窗口的宏观定义&#xff0c;主要是管理 View 的创建&#xff0c;以及与 ViewRootImpl 的交互&#xff0c;将 Activity 与 View 解耦。</p> 
<ul><li>Activity 与 PhoneWindow 与 DecorView 之间什么关系&#xff1f;</li></ul> 
<p>一个 Activity 对应一个 Window 也就是 PhoneWindow&#xff0c;一个 PhoneWindow 持有一个 DecorView 的实例&#xff0c;DecorView 本身是一个 FrameLayout。</p> 