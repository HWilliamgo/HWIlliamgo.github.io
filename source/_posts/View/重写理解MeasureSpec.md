---
title: 重新理解MeasureSpec
categories:
  - View
date: 2020-05-28 17:44:03
tags:
index_img:
banner_img:
layout: false
---



### 1 概述

网上有许多非常好的文章都在介绍MeasureSpec的测量规则，但是没有介绍MeasureSpec的作用和应用场景。



![tekaEF.png](https://s1.ax1x.com/2020/05/28/tekaEF.png)

MeasureSpec是一个int，他将SpecMode和SpecSize封装到了一起。

那么实际上MeasureSpec他是一个对：值和模式的一个封装。

在这里，size和mode是成对出现的，他们一起作用。

MeasureSpec可以翻译成：测量说明书。

他由手机屏幕的Window开始，将测量说明书生成并往下传递给DecorView，DecorView再生成自己的测量说明书，往下传递，不断递归，每个ViewGroup都根据父View的测量说明书和自己的尺寸，生成自己的测量说明书并递归下去。

什么是测量说明书？由编写测量说明书的一方（父View或者window）编写测量说明书，告诉客户（子View）要按照该测量说明书中的标准和规范来进行测量操作，从而实现父View对子View尺寸限制。


试问，一个子View如何知道他的父View给他预留了多少尺寸？

答：父View通过调用子View的measure()方法，将父View留给子View的尺寸传递给子View。



### 2 MeasureSpec使用场景

MeasureSpec的使用场景分为两个：

1. child View接收到parent View为自己生成的MeasureSpec对象，在onMeasure(int widthMeasureSpec, int heightMeasureSpec)中提取出该对象中的数据并调用setMeasureDimension为自己设置measureWidth和measureHeight.

2. parent View，即ViewGroup，这里先说下ViewGroup的onMeasure()方法的重写套路：

   在收到自己的onMeasure()回调的时候：

   ①要先对自己所有的子View进行测量，一般是遍历所有子View并调用ViewGroup的measureChildWithMargins()，然后再调用child.getMeasureWidth方法取出测量值，然后要么累加所有子View的测量者（如LinearLayout），要么从中选出最大的那个（如FrameLayout）。

   ②根据业务逻辑和他的父View设置给自己的MeasureSpec，来对他自己调用setMeasureDimension。

   这里的第②点就和1.是一个东西，所以我们说的是①。

   

   即②中，在测量所有的子View的时候，父View将为每个子View生成他专属的MeasureSpec对象。



那么我们分别来看看这两种使用场景中是如何使用MeasureSpec的。

按照MeasureSpec先创建后使用的顺序，我们先看他的创建，后看他在子View中的使用



### 3 MeasureSpec的创建



#### 3.1 Window为DecorView创建MeasureSpec

从最顶部开始：

ViewRootImpl.java

``` java
private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
                                 final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
        }
	//...
    if (!goodMeasure) {
        childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
        childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
        if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
            windowSizeMayChange = true;
        }
    }
	//...
    return windowSizeMayChange;
```

这里的`performMeasure`方法里面，调用了DecorView的measure，至此`MeasureSpec`对象开始从ViewTreee顶部开始向下传递。

``` java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    if (mView == null) {
        return;
    }
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```



这里看下传递给DecorView的MeasureSpec是如何生成的：

``` java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {
    case ViewGroup.LayoutParams.MATCH_PARENT:
        // Window can't resize. Force root view to be windowSize.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // Window can resize. Set max size for root view.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        // Window wants to be an exact size. Force root view to be that size.
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```

这里的windowSize是从WindowMananger获取到的Window的视图的尺寸，如手机屏幕大小。

而rootDimension参数是分别是DecorView的宽和高。而这里就是MATCH_PARENT，具体的定义要看创建DecorView的源码的地方，对应的是PhoneWindow类的`installDecor()`方法里。

通过这个`getRootMeasureSpec()`方法我们可以看到，创建的size和mode的对应关系为：

| size         | mode                |
| ------------ | ------------------- |
| MATCH_PARENT | MeasureSpec.EXACTLY |
| WRAP_CONTENT | MeasureSpec.AT_MOST |
| 具体值       | MeasureSpec.EXACTLY |

这是window为DecorView创建MeasureSpec时，为后者创建的MeasureSpec的对应的规则，举一反三一下，ViewGroup为子View创建MeasureSpec的时候，也是用的这种规则来生成MeasureSpec对象。



#### 3.2 ViewGroup为子View创建MeasureSpec



测量子View这件事都是发生在ViewGroup的onMeasure中，而ViewGroup是没有重写onMeasure的，这个规则他留给了他的子类去重写，我们找到一个最简单的子类：FrameLayout，并截取部分他的onMeasure中的代码：

``` java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int count = getChildCount();
    //...
    int maxHeight = 0;
	int maxWidth = 0;
	int childState = 0;
    //遍历子View
    for (int i = 0; i < count; i++) {
    final View child = getChildAt(i);
    if (mMeasureAllChildren || child.getVisibility() != GONE) {
        //测量子View
        measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
        //提取子View的measuredWidth和measuredHeight
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        maxWidth = Math.max(maxWidth,
                child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
        maxHeight = Math.max(maxHeight,
                child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
        //...
    }
    //...
}
```

那么看他用来测量子View调用的ViewGroup的方法：measureChildWithMargins

``` java
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    //获取子View的LayoutParams
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
    //创建子View的宽的MeasureSpec
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    //创建子View的高的MeasureSpec
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);
    //将这里创建的MeasureSpec传递给子View，并让子View进行测量
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

当创建子View的宽的MeasureSpec的时候，调用了getChildMeasureSpec方法

``` java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);
	//用父View的尺寸，减去已经使用了的尺寸（包括父view的padding,子View的marging和父View已经使用了的尺寸）。
    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
            // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

            // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

            // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            //...
    }
    //noinspection ResourceType
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

注意这里的参数padding，加上了父View的padding和子View的margin和父View已经使用了的尺寸（如果是FrameLayout，就是0，如果是LinearLayout，则会累加，看ViewGroup的逻辑而定）。size变量则是父View的尺寸减去上述的padding，得出留给子View的剩余空间的大小。



那么这里给子View生成MeasureSpec的逻辑就是：

`父View剩余大小`+`父View的MeasureSpec`+`子View的尺寸` --> `子View的MeasureSpec`

图形化表示为：

![tQgom6.png](https://s1.ax1x.com/2020/05/30/tQgom6.png)



这里有两个点要注意一下：

1. 当父View的布局大小确定的时候，即EXACTLY的时候，那么生成的子View的MeasureSpec依然符合上面创建根布局的情况：

   | size         | mode                |
   | ------------ | ------------------- |
   | MATCH_PARENT | MeasureSpec.EXACTLY |
   | WRAP_CONTENT | MeasureSpec.AT_MOST |
   | 具体值       | MeasureSpec.EXACTLY |

	而当父View自己的布局大小都不确定的时候，即AT_MOST时（即父View也是用的WRAP_CONTENT，所以他才得到了AT_MOST的mode）,子View的MATCH_PARENT其实就是要求和父View一样的大，那父View不确定大小，子View自然也不确定大小了，即AT_MOST。
	
2. 当子View的尺寸是WRAP_CONTENT的时候，父View给子View生成的size是父View剩下的size。
  
  为什么？因为父View在此时也不知道你子View有多大，那就把父View剩余的大小给子View，并告诉子View模式是AT_MOST，你子View最大不要超过我给你的这个大小，剩下的你尽管发挥。



### 4 MeasureSpec的使用

#### 4.1 View的onMeasure

``` java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                         getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

这里是用的`getDefaultSize()`方法来获取最终的子View的宽高，并用`setMeasuredDimension()`设置到自己的属性（稍后父View就可以获取到子View给自己测量的大小了）。

`getDefaultSize`的两个参数一个是获取建议的最小宽度，一个是父View给的测量说明书。

``` java
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
```

看下`getDefaultSize()`方法

``` java
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);
    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```

可以看到，如果父View给的测量模式是`UNSPECIFIED`，那就用`getSuggestedMinimumWidth`返回的大小。如果父View给的测量模式是`AT_MOST`或者`EXACTLY`，那就直接用父View给我们生成的大小。



这里要牵扯到一个自定义View的技巧：自定义View要重写onMeasure()方法来处理`AT_MOST`的测量模式。

从前面创建`MeasureSpec`知道，当子View用了`wrap_content`的时候，父View就会给你生成`AT_MOST`的测量模式，但因为`AT_MOST`测量模式下也是用的父View返回的尺寸，这时父View返回的尺寸是父View剩下的尺寸。他的意思是：这些尺寸给你，但是这是你能使用的最大的尺寸，不要超过这个尺寸就行。

一般来说我们会在自定义View中重写并判断`AT_MOST`时，返回一个默认的值。

比如这样：

``` java
override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec)
    //默认尺寸，写死或者根据业务逻辑计算得到
    val defaultWidth = 50
    val defaultHeight = 100
    //取出父View给的测量模式
    val widthSpecMode = MeasureSpec.getMode(widthMeasureSpec)
    val heightSpecMode = MeasureSpec.getMode(heightMeasureSpec)
    //取出父View给的测量尺寸
    val widthSpecSize = MeasureSpec.getSize(widthMeasureSpec)
    val heightSpecSize = MeasureSpec.getSize(heightMeasureSpec)
    //最终测量尺寸
    val finalWidth = if (widthSpecMode == MeasureSpec.AT_MOST) defaultWidth else widthSpecSize
    val finalHeight = if (heightSpecMode == MeasureSpec.AT_MOST) defaultHeight else heightSpecSize
    //set
    setMeasuredDimension(finalWidth,finalHeight)
}
```

这种是手动计算的，还有一种是借助View自带的`resolve()`方法来去计算的。

``` java
override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec)
        //默认尺寸
        val defaultWidth = 50
        val defaultHeight = 100
        //最终测量尺寸（注意，也计算了padding）
        val finalWidth = resolveSize(defaultWidth + paddingLeft + paddingRight, widthMeasureSpec)
        val finalHeight = resolveSize(defaultHeight + paddingTop + paddingBottom, heightMeasureSpec)
        //设置
        setMeasuredDimension(finalWidth, finalHeight)
}
```

`resolveSize()`方法内部有更精细的判断：

``` java
public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
    final int specMode = MeasureSpec.getMode(measureSpec);
    final int specSize = MeasureSpec.getSize(measureSpec);
    final int result;
    switch (specMode) {
        case MeasureSpec.AT_MOST:
            if (specSize < size) {
                result = specSize | MEASURED_STATE_TOO_SMALL;
            } else {
                result = size;
            }
            break;
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        case MeasureSpec.UNSPECIFIED:
        default:
            result = size;
    }
    return result | (childMeasuredState & MEASURED_STATE_MASK);
}
```



View对MeasureSpec是使用方，非创建方。

因此我们可以直接按照`MeasureSpec`的字面意思来直接理解：

`SpecSize`就是你父View给我指定的测量的大小。那么我拿到了这个大小我要怎么用呢？看你给我生成的测量模式`SpecMode`，如果是`EXACTLY`，那父View的意思就是直接让我用这个size作为最终我的测量大小就行了。如果是`AT_MOST`，父View传递给我的消息是，这个size不是让你作为最终的size的，我只是把我剩下的尺寸给你了，你不要超过这个尺寸即可。



#### 4.2 FrameLayout的onMeasure

实际上我们想看的是ViewGroup在onMeasure中是如何使用MeasureSpec的，但是ViewGroup没有重写，直接沿用改的View的，因此找了个简单的FrameLayout的来看看。

``` java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int count = getChildCount();

    final boolean measureMatchParentChildren =
        MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
        MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
    mMatchParentChildren.clear();

    int maxHeight = 0;
    int maxWidth = 0;
    int childState = 0;
	//遍历子View
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (mMeasureAllChildren || child.getVisibility() != GONE) {
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
			//取到所有子View中尺寸最大的那个尺寸
            maxWidth = Math.max(maxWidth,
                                child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
            maxHeight = Math.max(maxHeight,
                                 child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
            childState = combineMeasuredStates(childState, child.getMeasuredState());
            if (measureMatchParentChildren) {
                if (lp.width == LayoutParams.MATCH_PARENT ||
                    lp.height == LayoutParams.MATCH_PARENT) {
                    mMatchParentChildren.add(child);
                }
            }
        }
    }
	
    // 计算FrameLayout自身的padding
    maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
    maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

    // 再次检查 minimum height and width
    maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
    maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

    // Check against our foreground's minimum height and width
    final Drawable drawable = getForeground();
    if (drawable != null) {
        maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
        maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
    }
	//调用resolveSizeAndState()方法去获取最终测量值
    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                         resolveSizeAndState(maxHeight, heightMeasureSpec,
                                             childState << MEASURED_HEIGHT_STATE_SHIFT));

    count = mMatchParentChildren.size();
    if (count > 1) {
        for (int i = 0; i < count; i++) {
            final View child = mMatchParentChildren.get(i);
            final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

            final int childWidthMeasureSpec;
            if (lp.width == LayoutParams.MATCH_PARENT) {
                final int width = Math.max(0, getMeasuredWidth()
                                           - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                                           - lp.leftMargin - lp.rightMargin);
                childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                    width, MeasureSpec.EXACTLY);
            } else {
                childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                                                            getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                                                            lp.leftMargin + lp.rightMargin,
                                                            lp.width);
            }

            final int childHeightMeasureSpec;
            if (lp.height == LayoutParams.MATCH_PARENT) {
                final int height = Math.max(0, getMeasuredHeight()
                                            - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                                            - lp.topMargin - lp.bottomMargin);
                childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                    height, MeasureSpec.EXACTLY);
            } else {
                childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                                                             getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                                                             lp.topMargin + lp.bottomMargin,
                                                             lp.height);
            }

            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
    }
}
```

ViewGroup在对自己测量的时候，也是调用的`resolveSizeAndState`这个方法来计算出最终的测量值的。



### 5 总结

对于`MeasureSpec`创建者ViewGroup， 根据子View的LayoutParams的width和heigth和自己的MeasureSpec来为子View创建出MeasureSpec。传递给子View，并让子View根据该MeasureSpec设置对应的测量宽高，然后父View再拿到子View测量宽高，将上述动作遍历所有子View后，再对自己进行测量，设置自己的宽高。

对于`MeasureSpec`使用者View，根据父View传递进来个MeasureSpec，结合自身逻辑，计算出自己的宽高。

