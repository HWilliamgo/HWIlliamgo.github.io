## 记录自定义Dialog的一个天坑。

``` java
/**
 * Created by 黄伟杰 on 2018/7/17.
 */
public class MyDialog extends Dialog {
    private Context context;

    public MyDialog(@NonNull Context context) {
        super(context);
        this.context=context;
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        setContentView(R.layout.dialog);//放在onCreate里去调用，文档里就是这么写的。
        locateWindow(Gravity.TOP|Gravity.CENTER_HORIZONTAL);
    }
    
    //如果这个方法不在setContentView后面调用，params.width和params.height的设置将会失效
    private void locateWindow(int gravity){
        Window window = getWindow();
        Objects.requireNonNull(window).setGravity(gravity);
        WindowManager.LayoutParams params = Objects.requireNonNull(window).getAttributes();
        params.y = DensityUtil.dip2px(context, 49);
        params.width = WindowManager.LayoutParams.MATCH_PARENT;
        params.height = WindowManager.LayoutParams.WRAP_CONTENT;
        window.setAttributes(params);
    }
}
```

## 附带一个`Dialog`的由创建到展现给用户的流程：
*贴出的源码都是浓缩版代码*

---
构造方法：
``` java
    Dialog( Context context, int themeResId, boolean createContextThemeWrapper) {
        ...
        //获取单例类WindowManager。
        mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
        //创建PhoneWindow对象
        final Window w = new PhoneWindow(mContext);
        mWindow = w;
      ...
    }
```

创建完后，我们代码里调用`Dialog#show`
``` java
public void show(){
    dispatchOnCreate(null);//  ==>调用onCreate();
    onStart();
    mDecor=mWindow.getDecorView();//获取我们Dialog所在的Window上的DecorView。
    mWindowManager.addView(mDecor, l);//把顶级DecorView放到Window上。
  //至此，Dialog显示出来了。准确地说，是刚才创建的PhoneWindow上的View显示出来了。
}
```
在onCreate()中，按照谷歌文档的推荐，在此处调用`setContentView()`;
![](https://upload-images.jianshu.io/upload_images/7177220-20abe6de646a5a01.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

要深入去看Window的一些东西，看这个大神的文章：
[Android Window 机制探索 - 凶残的程序员](https://blog.csdn.net/qian520ao/article/details/78555397)
