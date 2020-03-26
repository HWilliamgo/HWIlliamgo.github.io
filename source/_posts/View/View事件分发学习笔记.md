---

首先推荐**郭霖**的真正的通俗易懂的View的事件分发文章：

[Android事件分发机制完全解析，带你从源码的角度彻底理解(上)](http://blog.csdn.net/guolin_blog/article/details/9097463)
>文章中讲述了几个要点：
>* 如果你在执行ACTION_DOWN的时候返回了false，后面一系列其它的action就不会再得到执行了。简单的说，就是当dispatchTouchEvent在进行事件分发的时候，只有前一个action返回true，才会触发后一个action。即消费事件才会继续有事件。
>* 在dispatchMotionEvent方法中执行到了onTouchEvent中将MotionEvent对象传入switch中做ACTION_DOWN、ACTION_UP等判断的时候，默认地在结束switch之后马上返回true。也就是到了这一步，默认地就消费了事件。
>* 如果你有一个控件是非enable的，那么给它注册onTouch事件将永远得不到执行。（默认地每一个控件的enable=true）,并且设置OnClickListener也得不到执行。
---
# 我自己的一点学习笔记：
在View的dispatchTouchEvent(MotionEvent event)方法中，onTouch和onTouchEvent的优先级关系：`mOnTouchListener.onTouch>onTouchEvent`

![](https://upload-images.jianshu.io/upload_images/7177220-3dd405529c903a0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### onTouch:
由OnTouchListener接口控制，从外部setOnTouchListener来实现，返回true的话，就把事件拦截，则不会再执行下面的onTouchEvent。
``` java
button.setOnTouchListener(new View.OnTouchListener() {
    @Override
    public boolean onTouch(View v, MotionEvent event) {
        eventLog(event);
        return false;
    }
});
```

### onTouchEvent:
伪码如下
``` java
public boolean onTouchEvent(MotionEvent event) {
	
	boolean clickable =CLICKABLE;
	//如果CLICKABLE为true，才接着判断MotionEvent。
	if(clickable){
		switch(action){
			case MotionEvent.ACTION_UP:
				...
				...
				...
				performClick();
				break;
			case  MotionEvent.ACTION_DOWN:
				break;
			case ..
			...
		}
	}
}
```
伪码的意思就是：
* 如果View的CLICKABLE属性是false，那么onTouchEvent就不能根据传入的MotionEvent对象才进行操作。
* 注意ACTION_UP的最后面有一个performClick()方法，点进去==>
``` java
public boolean performClick(){
	...
	//这里的mOnClickListener就是===>
	//我们通过button.setOnClickListener(new 匿名内部类)传入的对象；
	if(mOnClickListener!=null){
		mOnClickListener.onClick(this);
	}
	...
}
```
那么我们的onClick方法不会执行的原因可能有一种：
* 我们设置OnTouchListener.onTouch方法返回了true。
* ~~View的CLICKABLE是false~~，这是不可能的，因为一旦setOnClickListener(),就会把CLICKABLE改为true==>>
* ![](https://upload-images.jianshu.io/upload_images/7177220-9d5b336fe3e575b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

此外，View的CLICKABLE默认就是false的，但是为什么button就算不设置OnClickListener也能被click呢？看一下Button的构造方法：
``` java
public class Button extends TextView {
    public Button(Context context) {
        this(context, null);
    }
	//看传入的第三个参数是R.attr.buttonStyle!!
    public Button(Context context, AttributeSet attrs) {
        this(context, attrs, com.android.internal.R.attr.buttonStyle);
    }
    public Button(Context context, AttributeSet attrs, int defStyleAttr) {
        this(context, attrs, defStyleAttr, 0);
    }
    public Button(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
    }
	//其他忽略
	...
```
找到那个R.attr.buttonStyle关联的文件：
她在themes.xml文件中
``` xml
<!-- Button styles -->
<item name="buttonStyle">@style/Widget.Button</item>
<item name="buttonStyleSmall">@style/Widget.Button.Small</item>
<item name="buttonStyleInset">@style/Widget.Button.Inset</item>
<item name="buttonStyleToggle">@style/Widget.Button.Toggle</item>
```
点进去buttonStyle==>

![](https://upload-images.jianshu.io/upload_images/7177220-b0e4a9776c2dca09.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Button通过style已经设置了clickable=true。
