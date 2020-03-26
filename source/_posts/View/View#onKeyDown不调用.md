原因：当前View没有获得焦点。

只要当前的View获得了焦点，那么View的`onKeyDown()`，`onKeyUp()`，`setOnKeyListener()`等回调都会发生。

KeyEvent由操作系统接收用户输入产生，在应用层，到达顺序是：

`ViewRootImpl`->`DecorView`->`Activity`->`ViewGroup`->`View`



那么看ViewGroup的`dispatchOnKeyEvent()`方法：

``` java
 @Override
 public boolean dispatchKeyEvent(KeyEvent event) {
     if (mInputEventConsistencyVerifier != null) {
         mInputEventConsistencyVerifier.onKeyEvent(event, 1);
     }
     if ((mPrivateFlags & (PFLAG_FOCUSED | PFLAG_HAS_BOUNDS))
             == (PFLAG_FOCUSED | PFLAG_HAS_BOUNDS)) {
         if (super.dispatchKeyEvent(event)) {
             return true;
         }
     } else if (mFocused != null && (mFocused.mPrivateFlags & PFLAG_HAS_BOUNDS)
             == PFLAG_HAS_BOUNDS) {
         //如果有子View获得了焦点，那么就把KeyEvent分发给子View处理。
         if (mFocused.dispatchKeyEvent(event)) {
             return true;
         }
     }
     if (mInputEventConsistencyVerifier != null) {
         mInputEventConsistencyVerifier.onUnhandledEvent(event, 1);
     }
     return false;
 }
```



如果子View获得了焦点，则进入到子View的`dispatchKeyEvent()`

``` java
public boolean dispatchKeyEvent(KeyEvent event) {
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onKeyEvent(event, 0);
    }
    // Give any attached key listener a first crack at the event.
    //noinspection SimplifiableIfStatement
    ListenerInfo li = mListenerInfo;
    //如果OnKeyListener不为空，且消费掉了该KeyEvent，则不再继续分发
    if (li != null && li.mOnKeyListener != null && (mViewFlags & ENABLED_MASK) == ENABLED
            && li.mOnKeyListener.onKey(this, event.getKeyCode(), event)) {
        return true;
    }
    //在event.dispatch()中会调用onKeyDown和onKeyUp等方法
    if (event.dispatch(this, mAttachInfo != null
            ? mAttachInfo.mKeyDispatchState : null, this)) {
        return true;
    }
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
    }
    return false;
}
```

看下`KeyEvent#dispatch(Callback receiver, DispatcherState state, Object target)`方法

``` java
public final boolean dispatch(Callback receiver, DispatcherState state,
        Object target) {
    switch (mAction) {
        case ACTION_DOWN: {
            mFlags &= ~FLAG_START_TRACKING;
            if (DEBUG) Log.v(TAG, "Key down to " + target + " in " + state
                    + ": " + this);
			//这个receiver就是调用该方法的View自己啦~View调用自己的onKeyDown了。
            boolean res = receiver.onKeyDown(mKeyCode, this);
            //省略
        }
        case ACTION_UP:
            //省略
        case ACTION_MULTIPLE:
            //省略
    }
    return false;
}
```



所以只要当前View获取了整个View tree中的焦点，则可以收到key事件。

我原先的需求是，Activity中有多个View组件，以及多个可能出现的Window，根据`onKeyDown()`来拦截用户点击返回按钮，一个一个地关闭Window和浮窗View之类的。当时的设想是每个View自己内部去重写onKeyDown()方法，然后根据自己的可见性来拦截BACK事件并隐藏。但是发现无法回调onKeyDown()方法。于是放弃，并还是采取在Activity中重写onKeyDown()并一个一个地判断关闭。

今天查了一番资料，发现是焦点的问题，因此还是可以把onKeyDown写到对应的每个View中的。

只是需要额外做点工作，就是在显示的时候记住当前的焦点View，并在隐藏的时候把焦点还回去，以便原先的那个View也能正常接收`onKeyDown()`方法，例如：



``` kotlin
private var lastFocusdView: View? = null

private fun initView(){
    isFocusable = true
	isFocusableInTouchMode = true
    
    //...
}
private fun show(){
    lastFocusdView = (context as Activity).currentFocus
    requestFocus()
    //do show
}
private fun hide(){
    lastFocusdView?.requestFocus()
    //do hide
}
```



参考：

[Activity dispatchTouchEvent事件分发的源头](https://blog.csdn.net/Conan9715/article/details/78529055)

[Android KeyEvent 点击事件分发处理流程（一）](https://www.jianshu.com/p/2f28386706a0)
