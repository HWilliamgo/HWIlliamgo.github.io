# requestLayout()引起的问题

> 网上有大量写的很深入的`requestLayout()`源码分析的文章。故这里不再写了，只做一个实际情况下遇到的问题的分析。
>
>
>
> 起因：
>
> 自定义了一个`CircleImageView`，功能是调用`setImage(Bitmap bitmap)`后可以将图片以圆形加载。
>
> 本以为直接在`setImage(Bitmap)`的结尾直接调用`requestLayout()`即可。

这里从两个方面写：

### xml中定义为wrap_content

当`LayoutParams`是`wrap_content`时，我处理的逻辑是：在`onMeasure()`中根据宽高的`MeasureSpec`是否等于`MeasureSpec.AT_MOST`，如果等于，那么在第一次绘制的时候，`setMeasureDimension()`都设置成0，而当调用了`setBitmap()`时，获取图片的宽高并保存，然后调用`requestLayout()`，此举引起`onMeasure()`，那么在此处将图片的宽高设置到`setMeasureDimension()`中，而整体的View的测量大小就是图片大小了。

此时我的`onMeasure`和`setBitmap`长这样：

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    Log.d(TAG, "onMeasure: ");
    int ws = MeasureSpec.getSize(widthMeasureSpec);
    int hs = MeasureSpec.getSize(heightMeasureSpec);
    int wm = MeasureSpec.getMode(widthMeasureSpec);
    int hm = MeasureSpec.getMode(heightMeasureSpec);
    //如果子view是wrap_content，那么view就设置成bitmap的大小。
    if (wm == MeasureSpec.AT_MOST) {
        ws = bitmapW;
    }
    if (hm == MeasureSpec.AT_MOST) {
        hs = bitmapH;
    }
    int resultW = MeasureSpec.makeMeasureSpec(ws, wm);
    int resultH = MeasureSpec.makeMeasureSpec(hs, hm);
    super.onMeasure(resultW, resultH);
}
public void setImage(Bitmap bitmap) {
    mBitmap = bitmap;
    bitmapH = mBitmap.getHeight();
    bitmapW = mBitmap.getWidth();
    requestLayout();//最后要调用一个requestLayout，引起onMeasure()和onLayout()。
}
```

#### 大小不同的两张图片

此时为这个`CircleImageView`准备了两张分辨率不同的图片，点击按钮A加载图片A，点击按钮B加载图片B。

点击情况：

1. 由显示A的情况下加载B，或者显示B的情况下加载A，或者从没有图片情况下点击加载Ａ或Ｂ，分别引起了`onMeasure()`,`onSizeChanged（）`,`onLayout()`,`onDraw`回调。成功地在`onDraw`重新`drawBitmap()`切换了图片。
2. 当在A图片下点击按钮A，或者在B图片下点击按钮B。只引起了`onMeasure()`和`onLayout()`回调。这里少了一个`onSizeChanged()`很好理解，因为在该情况下，`setMeasureDimesion()`传入的值和上一次一样，View中可以很容易通过这种判断而跳过`onSizeChanged()`回调，而至于为什么`onDraw（）`回调没有引起，这点我也很疑惑。

#### 大小相同的两张图片

此时我又准备了两张大小相同的图片。操作和上述一样。

点击情况：

1. 从没有图片的情况下点击加载图片Ａ:`onMeasure()`,`onSizeChanged（）`,`onLayout()`,`onDraw`
2. 从图片Ａ点击加载图片Ｂ：加载失败，图片仍然停留在图片Ａ，此时的回调是：`onMeasure()`和｀onLayout()｀
3. 图片Ａ点击加载图片Ａ:　同２。

#### 推论：(在wrap_content情况下)

1. 当`requestLayout()`调用时，一定会引起`onMeasure()`和`onLayout()`。
2. 当`requestLayout()`调用时，如果没有在`setMeasureDimension()`中传入和上次不同的测量值的话，一定不会引起`onSizeChanged()`和`onDraw()`。`onSizeChanged()`不被调用的原因很容易在`onLayout()`的源码中找到答案，而`onDraw()`不引起回调的原因目前还不明白。

### xml中定义为精确值

1. 情况：此时图片直接无法加载。仅仅在第一次实例化CircleImageView的时候会依次调用`onMeasure()`,`onSizeChanged（）`,`onLayout()`,`onDraw`，伺候每一次调用`setImage()`都只会引起`onMeasure()`和`onLayout()`。

2. 原因：因为在`requestLayout()`调用后，因为此时的测量模式是`EXACTLY`，因此`setMeasuredDimension()`中传入的值永远不变，永远都是xml中定义的那个精确值。而上文的推论中指出，`setMeasuredDimension()`传入的值等于原本的测量值的话，直接引起`onSizeChanged()`和`onDraw()`无法调用。



---

## 结论

1. 上文中的推论
2. 不能依赖`requestLayout`来引起`onDraw()`回调，如果百分之百确定要绘制，直接调用`invalidate()`或`postInvalidate()`，他们只会引起`onDraw()`的回调。
3. CircleImageView源码：https://github.com/William619499149/anddroid-little-bubble/blob/master/CircleImageView.java
