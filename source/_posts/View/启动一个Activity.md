# Activity启动
##  startActivity()

||

||

V

``` java
startActivityForResult()
```

 ||

 ||

 V

```java
ActivityStackSupervisor # realStartActivityLocked(){
    //...
    app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
            System.identityHashCode(r), r.info,
            new Configuration(mService.mConfiguration), r.compat,
            app.repProcState, r.icicle, results, newIntents, !andResume,
            mService.isNextTransitionForward(), profileFile, profileFd,
            profileAutoStop);
    //...
}
```

此处的`app.thread`是`IApplicationThread` ，他的实现类是`ActivityThread`的内部类`ApplicationThread`。

那么看`ApplicationThread`中的`shceduleLaunchActivity`

```java
@Override
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
        ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
        CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
        int procState, Bundle state, PersistableBundle persistentState,
        List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
        boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
    //...
    sendMessage(H.LAUNCH_ACTIVITY, r);
}
```

看下`sendMessage`做了什么

##  sendMessage()

```java
private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
    if (DEBUG_MESSAGES) Slog.v(
        TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
        + ": " + arg1 + " / " + obj);
    Message msg = Message.obtain();
    msg.what = what;
    msg.obj = obj;
    msg.arg1 = arg1;
    msg.arg2 = arg2;
    if (async) {
        msg.setAsynchronous(true);
    }
    mH.sendMessage(msg);
}
```

搜索`mH`，发现是`private class H extends Handler`。是`ActivityThread`的内部类，和主线程`Looper`关联，那么`mH` 的`handleMessage()`就是在主线程里面执行了。



搜索H类里面的`H.LAUNCH_ACTIVITY`，发现了：

```java
public void handleMessage(Message msg) {
    if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
    switch (msg.what) {
        case LAUNCH_ACTIVITY: {
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
            final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
            r.packageInfo = getPackageInfoNoCheck(
                    r.activityInfo.applicationInfo, r.compatInfo);
            handleLaunchActivity(r, null);//在这
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        break;
		//...
    }
}
```

## handleLaunchActivity()

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    //...
    //创建Application和Activity对象，并调用它们的生命周期
    Activity a = performLaunchActivity(r, customIntent);
    if(a!=null){
        //Activity#onResume
        handleResumeActivity(r.token, false, r.isForward,!r.activity.mFinished && !r.startsNotResumed);
    }
}
```

### performLaunchActivity()

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent){
    ClassLoader cl = r.packageInfo.getClassLoader();
    //根据ClassLoader创建Activity对象
    Activity activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
    //创建Application对象并调用App#onCreate
    Application app=r.packageInfo.makeApplication(false,mInstrumentation);
    //创建PhoneWindow，WindowManager。并和Activity关联
    activity.attach(...);
    //Activity#onCreate()
    mInstrumentation.callActivityOnCreate(activity, r.state);
    //Activity#onStart()
    activity.performStart();
    //...
}
```

#### r.pakageInfo.makeApplication()

```java
public Application makeApplication(boolean forceDefaultAppClass, Instrumentation instrumentation) {
    if (mApplication != null) {
        return mApplication;
    }
    Application app = null;
    java.lang.ClassLoader cl = getClassLoader();
    app = mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext);
    mActivityThread.mAllApplications.add(app);
    instrumentation.callApplicationOnCreate(app);//调用App.onCreate()
```

#### activity.onAttach()

```java
final void attach(...) {
    //实例化PhoneWindow
    mWindow = new PhoneWindow(this);
    mWindow.setCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    //实例化WindowManagerImpl，并和mWindow建立关联。
    mWindow.setWindowManager((WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                             mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    mWindowManager = mWindow.getWindowManager();
}
```

看下是否在此处就new出一个`WindowManager`，点进`setWindowManager()`。

```java
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
        boolean hardwareAccelerated) {
    //...
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```

点进`createLocalWindowManager(this)`

```java
public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
    //的确，在这里实例化了WindowManagerImpl
    return new WindowManagerImpl(mDisplay, parentWindow);
}
```

### handleResumeActivity()

```java
final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward,
boolean reallyResume) {
    //Activity#onResume()
    ActivityClientRecord r = performResumeActivity(token, clearHide);
    r.window = r.activity.getWindow();
    View decor = r.window.getDecorView();
    //让DecorView变的不可见
    decor.setVisibility(View.INVISIBLE);
    //WindowManager在Activity#onAttach中已经被实例化了
    ViewManager wm = a.getWindowManager();
    WindowManager.LayoutParams l = r.window.getAttributes();
    //WindowManager#addView，将DecorView添加到Window中。
    wm.addView(decor, l);
}
```

现在Activity的`onResume`都执行完了，执行到了`WindowManager#addView`。

---

## WindowManager

`WindowManager`接口继承自`ViewMananger`。`ViewManager`意思是：view管理者

``` java
public interface ViewManager{
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```
因此`WindowManager`也具有管理View管理者的能力，他的实现类是`WindowManagerImpl`。看看他的`addView`在做什么。
``` java
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
	applyDefaultToken(params);
	mGlobal.addView(view, params, mDisplay, mParentWindow);
}
```
mGlobal是WindowManagerGlobal，从命名方式和`getInstance`看出，很明显的全局单例。
`private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();`
看下WindowManagerGlobal中的4个引用
``` java
public final class WindowManagerGlobal {
	...
	//放着所有的DecorView。（有的博客说放着所有的View，但是WMG里面用了mViews.add()的地方只有一个，而且只会传入DecorView。因此我认为是只放DecorView）
	private final ArrayList<View> mViews = new ArrayList<View>();
	//放着所有的ViewRootImpl
	private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
	//放着所有的WindowManager.LayoutParams
	private final ArrayList<WindowManager.LayoutParams> mParams =new ArrayList<WindowManager.LayoutParams>();
	//放着所有正在被删除的View。
	private final ArraySet<View> mDyingViews = new ArraySet<View>();
	...
}
```
接着mGlobal.addView里面在做什么。
``` java
public void addView(View view, ViewGroup.LayoutParams params,
		Display display, Window parentWindow) {
	final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
	ViewRootImpl root;
    //实例化了ViewRootImpl，调用他的构造方法。
	root = new ViewRootImpl(view.getContext(), display);
	view.setLayoutParams(wparams);
	mViews.add(view);
	mRoots.add(root);
	mParams.add(wparams);
	// do this last because it fires off messages to start doing things
	root.setView(view, wparams, panelParentView);//调用了ViewRootImpl#setView。跟进去看下
}
```

ViewRootImpl#setView
``` java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
	mView = view;
	// Schedule the first layout -before- adding to the window
	// manager, to make sure we do the relayout before receiving
	// any other events from the system.
	requestLayout();
	//将该Window添加到屏幕。
    //mWindowSession实现了IWindowSession接口，它是Session的客户端Binder对象.
    //addToDisplay是一次AIDL的跨进程通信，通知WindowManagerService添加IWindow
	res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
		getHostVisibility(), mDisplay.getDisplayId(),
		mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
		mAttachInfo.mOutsets, mInputChannel);
	view.assignParent(this);
}
```
requestLayout()里面调用scheduleTraversals();然后调用Choreographer编舞者的内部的一系列方法，用Handler和Looper,把doTraversals()放到主线程去轮询。然后调用的就是三大遍历，测量，布局，绘制。
