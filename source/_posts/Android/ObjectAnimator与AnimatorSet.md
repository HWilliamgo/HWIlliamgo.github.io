---
title: ObjectAnimator实现动画的使用
---
---
相对于ValueAnimator，ObjectAnimator显得更加智能，自动，和简便，有时为了实现简单的动画效果，用ObjectAnimator在代码上会更简约。
先看一下ValueAnimator最简单的实现：
``` java
button=findViewById(R.id.btn);
//通过ofInt()静态方法来返回一个ValueAnimator实例。
ValueAnimator valueAnimator=ValueAnimator.ofInt(button.getLayoutParams().width,500);
//用ValueAnimator对象设置各种参数
valueAnimator.setStartDelay(1000);
valueAnimator.setDuration(2000);
valueAnimator.setRepeatCount(ValueAnimator.INFINITE);
valueAnimator.setRepeatMode(ValueAnimator.REVERSE);
//添加动画更新监听器
valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
    @Override
    public void onAnimationUpdate(ValueAnimator animation) {
        int currentValue= (int) animation.getAnimatedValue();
        Log.d(TAG, String.valueOf(currentValue));
        //在监听器的动画更新回调方法中，将传入的新的数值设置给对应的View的属性，实现View的属性的动态变换。
        button.getLayoutParams().width=currentValue;
        //View请求重新布局
        button.requestLayout();
    }
});
valueAnimator.start();
```
四步：
* ofInt()获取ValueAnimator实例
* 设置valueAnimator参数
* 添加数据更新监听器，在监听器中手动对view的属性进行更改
* start();
---
那么如果是ObjectAnimator呢？
简单多了：
``` java
ObjectAnimator objectAnimator=ObjectAnimator.ofFloat(button,
			"translationX",button.getTranslationX(),button.getTranslationX()+500);
objectAnimator.setDuration(2000);
objectAnimator.setRepeatCount(ValueAnimator.INFINITE);;
objectAnimator.setRepeatMode(ValueAnimator.REVERSE);
objectAnimator.start();
```
三步：
* 用ofFloat()返回一个ObjectAnimator对象，方法的参数：
	* 第一个是要改变的对象
	* 第二个是该对象要改变的具体的属性，传入的是字符串，直接传入属性名，Button继承的View，所以有translationX。
	* startValue
	* endValue
* 设置各个参数
* start();
这里少了一步：设置数据更新监听器。
因为ObjectAnimator在内部帮我们实现了传入对象的property的改变，所以说他更智能，更简单。
效果：
![](http://upload-images.jianshu.io/upload_images/7177220-24f63a366c36e731.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---
## 组合动画：
AnimationSet
``` java
//创建组合动画对象
AnimatorSet animatorSet = new AnimatorSet();
//创建平移动画对象
ObjectAnimator objectAnimator = ObjectAnimator.ofFloat(button,
		"translationX", button.getTranslationX(), button.getTranslationX() + 500);
objectAnimator.setDuration(2000);
objectAnimator.setRepeatCount(ValueAnimator.INFINITE);
objectAnimator.setRepeatMode(ValueAnimator.REVERSE);
//创建X轴缩放对象
ObjectAnimator scaleX = ObjectAnimator.ofFloat(button, "scaleX", 1f, 1.2f);
scaleX.setDuration(500);
scaleX.setRepeatMode(ValueAnimator.REVERSE);
scaleX.setRepeatCount(ValueAnimator.INFINITE);
//创建Y轴缩放对象
ObjectAnimator scaleY= ObjectAnimator.ofFloat(button,"scaleY",1f,3f);
scaleY.setRepeatCount(ValueAnimator.INFINITE);
scaleY.setRepeatMode(ValueAnimator.REVERSE);
scaleY.setDuration(500);
//用一个playTogether（传入objectAnimator数组）把他们都联系起来、
animatorSet.playTogether(objectAnimator,scaleX,scaleY);
//start()；
animatorSet.start();
```
效果：
![](http://upload-images.jianshu.io/upload_images/7177220-068646b6e54fcd1e.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





