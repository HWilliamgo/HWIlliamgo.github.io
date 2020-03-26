
结论： 在自定义控件中如下重写`onInterceptTouchEvent`就告诉所有父View：**不要拦截事件，让我消费！！**
``` java
	@Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        getParent().requestDisallowInterceptTouchEvent(true);
        return super.onInterceptTouchEvent(ev);
    }
```

---

这是一个从源码角度分析**滑动冲突的原因**
以及在源码中理解为**何能解决滑动冲突**

这是MainActivity主界面的布局内容:
![](https://upload-images.jianshu.io/upload_images/7177220-a1c94fc98e2b3989.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
xml:
``` xml
<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.solory.learnview.MainActivity">
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">
        <Button
            android:id="@+id/btn"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
           android:layout_margin="8dp" />
        <ImageView
            android:id="@+id/imageView"
            android:layout_width="105dp"
            android:layout_height="86dp"
            android:layout_margin="8dp"
            app:srcCompat="@mipmap/ic_launcher_round" />
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@string/article_1"
            android:textSize="36sp" />
        <ScrollView
            android:layout_width="match_parent"
            android:layout_height="80dp"
            android:background="@color/colorPrimary">
            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="@string/article_2" />
        </ScrollView>
    </LinearLayout>
</ScrollView>
```
MainActivity不用动。
跑起来：![](https://upload-images.jianshu.io/upload_images/7177220-61629f4c1407b98d.gif?imageMogr2/auto-orient/strip)

外面的ScrollView正常滑动，但是里面的那个ScrollView动不了。
## 直接给出解决方案再看如何解决：
新建一个类继承ScrollView
``` java
public class MyScrollView extends ScrollView {
    public MyScrollView(Context context) {
        this(context,null);
    }
    public MyScrollView(Context context, AttributeSet attrs) {
        this(context, attrs,0);
    }
    public MyScrollView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }
    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
		//关键点在这	
        getParent().requestDisallowInterceptTouchEvent(true);
        return super.onInterceptTouchEvent(ev);
    }
}
```

buildProject,然后在xml中将里面的ScrollView修改成这个MyScrollView。
``` xml
...
<com.solory.learnview.MyScrollView
    android:layout_width="match_parent"
    android:layout_height="80dp"
    android:background="@color/colorPrimary">
    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="@string/article_2" />
</com.solory.learnview.MyScrollView>
...
```
跑起来：![](https://upload-images.jianshu.io/upload_images/7177220-f8873d4196b57ca7.gif?imageMogr2/auto-orient/strip)


### 问题解决。

那么现在研究为什么，为什么在重写的`onInterceptTouchEvent(MotionEvent ev)`中神奇的一句代码
`getParent().requestDisallowInterceptTouchEvent(true);`就把问题解决了？

好了。跟着我的思路来。
* 先看ScollView源码中的onInterceptTouchEvent:
	``` java
	 @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        /*
         * 这个方法决定了我们是否要拦截这个事件。
         * 如果我们返回true, onMotionEvent方法将被调用，
         * 我们将在那执行实际的滚动操作。
         */

        /*
        * 最常见的情况:用户在拖拽中。
        * 他在动他的手指。我们想要截取这个
        * 事件.
        */
        final int action = ev.getAction();
        if ((action == MotionEvent.ACTION_MOVE) && (mIsBeingDragged)) {
            return true;
        }

        if (super.onInterceptTouchEvent(ev)) {
            return true;
        }

        /*
         * 如果我们不能滚动，不要试图截取触摸。
         */
        if (getScrollY() == 0 && !canScrollVertically(1)) {
            return false;
        }
		...
		...
		...
	```
	清晰明了，干净简单,后面还有一大段代码，就不放了。这里一进来就是一个判断，如果进来的是ACTION_MOVE, 那么直接返回true，直接拦截，那么后面就没他的子View什么事了（不懂的话去看一下ViewGroup的dispatchTouchEvent方法），event被传入他自己的onTouchEvent中去进行滚动操作了。
	
* 那么我们一开始内部的ScrollView滑动没有响应的原因就是，那时候手指是在滑动的，一直不断传入ACTION_MOVE, 所以event一直被外部的ScrollView在如上的操作中拦截了。
* 意思就是只要你手指在ScrollView上滑动，ScrollView内部的子View就永远接收不到任何事件，就是永远无响应。
 
---
**冲突的原因明白了，现在看如何解决的**

* 回头看MyScrollView是如何解决的：
	``` java
	@Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        getParent().requestDisallowInterceptTouchEvent(true);
        return super.onInterceptTouchEvent(ev);
    }
	```
	意思就是取得父类，然后请求父类不拦截TouchEvent的意思。
	
	首先getParent就是返回父类，在这里是返回的那个LinearLayout，然后点`requestDisallowInterceptTouchEvent`进去看，发现是一个叫做ViewParent的接口中的抽象方法，
	![](https://upload-images.jianshu.io/upload_images/7177220-a1f99bb6a25fa2b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	注释的英文：
	``` 
	当一个子View不想要他的父View和它的祖先View们拦截触摸事件的时候。调用该方法
	他的父View应该将该方法接着向上传递给每一个祖先View们。
	```
* 抽象方法的话，看一下是谁实现了，因为继承的ScrollView，所以先看对应的ScrollView中的实现

	``` java
	//-----ScrollView中
	    @Override
    public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {
        if (disallowIntercept) {
		//我也不知道这个方法干嘛的，反正不影响整体思路，先跳过。
            recycleVelocityTracker();
        }
		//无论如何，都会执行父类的该方法。
        super.requestDisallowInterceptTouchEvent(disallowIntercept);
    }
	```
* 那么我们查看父类中的实现，ViewGroup中：
	``` java
	//-----ViewGroup中
	    @Override
    public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {

        if (disallowIntercept == ((mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0)) {
            // We're already in this state, assume our ancestors are too
            return;
        }

        if (disallowIntercept) {
            mGroupFlags |= FLAG_DISALLOW_INTERCEPT;
        } else {
            mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
        }

        // Pass it up to our parent
        if (mParent != null) {
            mParent.requestDisallowInterceptTouchEvent(disallowIntercept);
        }
    }
	```
	意思就是将自己的FLAG更改，变成disallowIntercept，并且递归，只要有父View， 就把父View的FLAG同样设置。
* 大意为，设置了这个方法，MyScrollView就通过**递归**，告诉了他的父View和向上的所有祖先View：统统不要拦截事件！交给我来！
* 那这个FLAG是在哪里发挥作用？当然是在ViewGroup的`dispatchTouchEvent(MotionEvent event)`内部，并且用一个if条件先于`onInterceptMotionEvent(MotionEvent event)`来判断
	图片为证：![](https://upload-images.jianshu.io/upload_images/7177220-751013d1f8b9d6e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	
---
摸了摸老夫的胡须，嗯...，说的真好啊~
## 但是！
``` java
//-----MyScrollView中
@Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        getParent().requestDisallowInterceptTouchEvent(true);
        return super.onInterceptTouchEvent(ev);
    }
```
这段代码的`getParent().requestDisallowInterceptTouchEvent(true);`能执行到的前提是MyScrollView能执行`onInterceptTouchEvent`，也就是能执行`dispatchTouchEvent`，可是事件早都被外层的ScrollView拦截了，你还怎么获取父类然后请求不要拦截TouchEvent？
**当时我在这里思考了蛮久的，** 那么我们返回到ScrollView的`onInterceptTouchEvent`里去看吧
	
``` java
	 @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        final int action = ev.getAction();
        if ((action == MotionEvent.ACTION_MOVE) && (mIsBeingDragged)) {
            return true;
        }
        if (super.onInterceptTouchEvent(ev)) {
            return true;
        }
        if (getScrollY() == 0 && !canScrollVertically(1)) {
            return false;
        }
		...
		...
		...
```
他只拦截ACTION_MOVE，不拦截ACTION_DOWN，所以当ACTION_DOWN的那一次事件还是可以传到下面的子View去的，而利用这一点，MyScrollView利用第一次触碰那唯一的一次event，将他FLAG给改了，事件就可以顺利地传递到MyScrollView了~

## 总结：
* 在自定义控件中重写`onInterceptTouchEvent`就告诉所有父View：**不要拦截事件，让我消费！！**
	``` java
	@Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        getParent().requestDisallowInterceptTouchEvent(true);
        return super.onInterceptTouchEvent(ev);
    }
	```




