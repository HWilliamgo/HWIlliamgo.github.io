先看下源码

```java
public void scrollBy(int x, int y) {
    scrollTo(mScrollX + x, mScrollY + y);
}
```

```java
public void scrollTo(int x, int y) {
    if (mScrollX != x || mScrollY != y) {
        int oldX = mScrollX;
        int oldY = mScrollY;
        mScrollX = x;
        mScrollY = y;
        invalidateParentCaches();
        onScrollChanged(mScrollX, mScrollY, oldX, oldY);
        if (!awakenScrollBars()) {
            postInvalidateOnAnimation();
        }
    }
}
```

了解了scrollBy( x , y)就是在原来的基础上，改mScrollX为x，改为mScrollY为Y。

比如调用`view#scrollBy(100 , 50)`后，mScrollX增加了100，mScrollY增加了50。但是这个view的内容却往反方向移动了，就是往左边移动100 ，上边移动50。

而mScrollX和mScrollY的定义：

mScrollX = View的内容左边框的x坐标值 - 实际的View左边缘的x坐标。

mScrollY = View的内容上边框的的y坐标值 - 实际的View上边缘的y坐标。

而View的内容左边框的x坐标值和View的内容上边框的y坐标值在用`View#scrollTo`和`View#scrollBy`是不会改变的。

因此当调用scrollBy(int x , int y)中的参数都填入正值时，

```
mScrollX = View的内容左边框的x坐标值 - 实际的View左边缘的x坐标。
```

由等式性质得：实际的View的左边缘的x坐标，减少。

即scrollBy(x, y )的x>0时，view会往左跑。这就是scrollBy(int x , int y)反向的原因。

那么要让view的内容往右下方走的话，就要为scrollBy传入负值。比如：

![](https://upload-images.jianshu.io/upload_images/7177220-20de8296e0e33692.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
