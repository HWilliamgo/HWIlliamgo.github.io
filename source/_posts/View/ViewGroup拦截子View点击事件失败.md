![](https://upload-images.jianshu.io/upload_images/7177220-91210f087c254c6b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

流程图整理自[Android 触摸事件机制(四) ViewGroup中触摸事件详解 | skywang](http://wangkuiwu.github.io/2015/01/04/TouchEvent-ViewGroup/)

### 事情起因：
要用RelativeLayout去拦截里面的子View的点击事件，因此直接为
```
relativeLayout.setOnTouchListener(new View.OnTouchListener() {
    @Override
    public boolean onTouch(View v, MotionEvent event) {
        return true;
    }
});
```
打算通过返回true来将后续的点击事件消费掉，但是失败了。

### 在看了源码分析之后，找到原因：
1. 当ViewGroup有子View的时候，一定能拦截点击事件的入口是`onInterceptTouchEvent()`。
2. 当ViewGroup有子View能接收点击事件的时候，不会调用ViewGroup任何自己的点击事件监听方法（无论内部还是外部设置的监听器）。
3. 当ViewGroup没有子View能接收点击事件时，则会调用super.dispatchTouchEvent()，此时将ViewGroup当做View来看，按照View的那一套来。

### 结论：
我的RelativeLayout里面的子View可以接收点击事件，因此点击事件会直接传给他们，无法通过外部设置监听器去拦截。
