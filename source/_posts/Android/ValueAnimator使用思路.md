---
ValueAnimator有三个方法来创建动画，分别是：
* ofInt();
* ofFloat();
* ofObject();

## 先看ofInt():
效果：
![](http://upload-images.jianshu.io/upload_images/7177220-c2826d981ff630b1.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

思路：
* 通过ValueAnimaotr.ofInt(startValue,endValue)方法返回一个ValueAnimator对象，再给对象设置各种比如duration,repeatCount,repeatMode,StartDelay的参数。
* 最关键的是，要设置对象的.addUpdateListener(),该方法只要ValueAnimator传入的那个值在变化，回调方法就会不断地被回调，我们在这里就可以不断地调用更新View或者Layout某个属性或多个属性的方法，然后invalidate()来重新调用onDraw()，不断地更新视图。
* 最后调用ValueAnimator对象的.start()方法启动变化，这个方法一调用，他里面的value就马上变化，就会不停地回调UpdateListener的方法。
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

## ofFloat()
ofFloat()和ofInt()方法的思路是完全一样的，不同的是里面的估值不同，一个是
`FloatEvaluator`一个是`IntEvaluator`，两者就是FloatEvaluator中从startValue到endValue的过渡会更加精确更加平滑，因为是精确到float了。
## ofObject()
效果：
![](http://upload-images.jianshu.io/upload_images/7177220-08630373da9aad75.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
思路：
* 上述的两个方法都有系统默认实现的`FloatEvaluator`和`IntEvaluator`，而ofObject则是可以我们自己实现估值器。
* 创建需要进行动画的自定义View
* 创建数据实体类
* 继承TypeEvaluator，重写evaluate方法
![](http://upload-images.jianshu.io/upload_images/7177220-d875c73959c022f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

具体的是，通过View中的onDraw()方法，不断地在里面用valueAnimator的UpdateListener回调来更新小球的坐标数值并`invalidate（）`,小球的坐标包括了横坐标和纵坐标，如果用ofInt或者ofFloat的话，只能单一地渐变横坐标或者单一地渐变纵坐标，因此我们将坐标封装到Point实体类中：
``` java
public class Point {
    private float x;
    private float y;
    public float getX() {
        return x;
    }
    public float getY() {
        return y;
    }
    public Point(float x, float y){
        this.x=x;
        this.y=y;
    }
}
```
紧接着马上继承TypeEvaluator并重写evaluate方法，在该方法中对传入的Point的两个横纵坐标进行改变:
``` java
public class PointEvaluator implements TypeEvaluator<Point> {
    @Override
    public Point evaluate(float fraction, Point startValue, Point endValue) {
        float x=startValue.getX()+fraction*(endValue.getX()-startValue.getX());
        float y=startValue.getY()+fraction*(endValue.getY()-startValue.getY());
        Point point=new Point(x,y);
        return point;
    }
}
```
接着是自定义View中的代码，在onDraw里面做文章：
``` java
public class RoundView extends View {
    public RoundView(Context context) {
        this(context, null);
    }

    public RoundView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public RoundView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        this(context, attrs, defStyleAttr, 0);
    }

    public RoundView(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);

        paint = new Paint(Paint.ANTI_ALIAS_FLAG);
        paint.setColor(Color.BLUE);

    }

    private static final float RADIUS = 70f;
    private Point currnetPoint;
    private Paint paint;

    @Override
    protected void onDraw(Canvas canvas) {
        //我们这里完全通过Point类的数据来画圆，因此如果他是Null的话
        //就是第一次画圆，我们做第一次画圆的初始化
        if (currnetPoint == null) {
            currnetPoint = new Point(RADIUS, RADIUS);
            float x = currnetPoint.getX();
            float y = currnetPoint.getY();
            canvas.drawCircle(x, y, RADIUS, paint);
            //拿到将要进行变化的Point的始态和末态
            Point startPoint = new Point(RADIUS, RADIUS);
            Point endPoint = new Point(700, 1000);
            //用ofObject来创建ValueAnimator对象
            ValueAnimator animator = ValueAnimator.ofObject(new PointEvaluator(), startPoint, endPoint);
            //用animator对象设置各种参数
            animator.setDuration(2000);
            animator.setRepeatMode(ValueAnimator.REVERSE);
            animator.setRepeatCount(ValueAnimator.INFINITE);
            //用animator对象设置更新监听器,在onAnimationUpdate回调中对数据实体类进行更新，
            //然后调用invalidate()。最后开启动画调用start()。
            animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
                @Override
                public void onAnimationUpdate(ValueAnimator animation) {
                    currnetPoint = (Point) animation.getAnimatedValue();
                    invalidate();
                }
            });
            animator.start();
        } else {
            float x = currnetPoint.getX();
            float y = currnetPoint.getY();
            canvas.drawCircle(x, y, RADIUS, paint);
        }
    }
}
```


