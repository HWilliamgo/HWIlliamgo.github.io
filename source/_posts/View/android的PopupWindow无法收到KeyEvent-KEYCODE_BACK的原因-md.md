PopupWindow无法收到`KeyEvent.KEYCODE_BACK`的原因：

在创建的顶层`PopupDecorView`的`dispatchKeyEvent()`回调中，把该事件拦截了。



### 1. PopupDecorView创建PopupDecorView

代码：

``` java
//PopupWindow#showAtLocation()
public void showAtLocation(IBinder token, int gravity, int x, int y) {
    if (isShowing() || mContentView == null) {
        return;
    }
    TransitionManager.endTransitions(mDecorView);
    detachFromAnchor();
    mIsShowing = true;
    mIsDropdown = false;
    mGravity = gravity;
    final WindowManager.LayoutParams p = createPopupLayoutParams(token);
  	//创建顶层PopupDecorView
    preparePopup(p);
    p.x = x;
    p.y = y;
  	//加入到WindowManger显示出来
    invokePopup(p);
}
```



``` java
private void preparePopup(WindowManager.LayoutParams p) {
    if (mContentView == null || mContext == null || mWindowManager == null) {
        throw new IllegalStateException("You must specify a valid content view by "
                + "calling setContentView() before attempting to show the popup.");
    }
    if (p.accessibilityTitle == null) {
        p.accessibilityTitle = mContext.getString(R.string.popup_window_default_title);
    }
    // The old decor view may be transitioning out. Make sure it finishes
    // and cleans up before we try to create another one.
    if (mDecorView != null) {
        mDecorView.cancelTransitions();
    }
    // When a background is available, we embed the content view within
    // another view that owns the background drawable.
    if (mBackground != null) {
      	//如果background不为空，把ContentView外封一层View，并给外封的View设置backgroud。
        mBackgroundView = createBackgroundView(mContentView);
        mBackgroundView.setBackground(mBackground);
    } else {
        mBackgroundView = mContentView;
    }
  	//创建PopupDecorView
    mDecorView = createDecorView(mBackgroundView);
    // The background owner should be elevated so that it casts a shadow.
    mBackgroundView.setElevation(mElevation);
    // We may wrap that in another view, so we'll need to manually specify
    // the surface insets.
    p.setSurfaceInsets(mBackgroundView, true /*manual*/, true /*preservePrevious*/);
    mPopupViewInitialLayoutDirectionInherited =
            (mContentView.getRawLayoutDirection() == View.LAYOUT_DIRECTION_INHERIT);
}
```



``` java
private PopupDecorView createDecorView(View contentView) {
    final ViewGroup.LayoutParams layoutParams = mContentView.getLayoutParams();
    final int height;
    if (layoutParams != null && layoutParams.height == WRAP_CONTENT) {
        height = WRAP_CONTENT;
    } else {
        height = MATCH_PARENT;
    }
    final PopupDecorView decorView = new PopupDecorView(mContext);
  	//加入我们设置的ContentView
    decorView.addView(contentView, MATCH_PARENT, height);
    decorView.setClipChildren(false);
    decorView.setClipToPadding(false);
    return decorView;
}
```



### 2. PopupDecorView#dispatchKeyEvent

看下PopupDecorView:

``` java
private class PopupDecorView extends FrameLayout{
	@Override
public boolean dispatchKeyEvent(KeyEvent event) {
  	//...
    if (event.getKeyCode() == KeyEvent.KEYCODE_BACK) {
        if (getKeyDispatcherState() == null) {
            return super.dispatchKeyEvent(event);
        }
        if (event.getAction() == KeyEvent.ACTION_DOWN && event.getRepeatCount() == 0) {
            final KeyEvent.DispatcherState state = getKeyDispatcherState();
            if (state != null) {
                state.startTracking(event, this);
            }
          	//返回true
            return true;
        } else if (event.getAction() == KeyEvent.ACTION_UP) {
            final KeyEvent.DispatcherState state = getKeyDispatcherState();
            if (state != null && state.isTracking(event) && !event.isCanceled()) {
                dismiss();
              	//返回true
                return true;
            }
        }
        return super.dispatchKeyEvent(event);
    } else {
        return super.dispatchKeyEvent(event);
    }
	}
  //...
}
```

可以看到，`PopupDecorView`本身并没有预留接口来给开发者去接管`KEYCODE_BACK`事件，遇到这个事件他一定会自己处理掉，所以你只能通过`OnDismissListener`接口和额外的flag变量来实现监控用户在PopupWindow上按下返回按钮。
