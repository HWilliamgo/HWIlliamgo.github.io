明确一点：toolbar不要设置setSupportActionBar();
```
说三遍！：toolbar不要设置setSupportActionBar();
         toolbar不要设置setSupportActionBar();
         toolbar不要设置setSupportActionBar();
```
>toolbar就是toolbar，不是actionBar~，而且用了setSupportActionBar之后api调用超级杂乱，因此我们就把Toolbar当做一个新的独立的控件就行了

#文章目录：
1. 入门级配置
2. 进阶设置
3. 设置popup的背景颜色和字体颜色
##1. 入门级配置：
![](http://upload-images.jianshu.io/upload_images/7177220-ed2ed551eeba8669.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
//toolbar
//单独设置theme为黑色主题是为了让字体是白色的
//minHeight标识最小高度，解决适配问题
//fitsSystemWindows是为了不让Toolbar中的内容顶到状态栏的位置
<android.support.v7.widget.Toolbar
    android:id="@+id/toolbar"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:background="@color/colorAccent"
    android:fitsSystemWindows="true"
    android:minHeight="?actionBarSize"
    android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar" >
    <ImageView
        android:src="@drawable/ic_android_black_24dp"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />
</android.support.v7.widget.Toolbar>
```
```
//style
//用的是Theme.AppCompat.DayNight.NoActionBar，黑色主题，无ActionBar
<resources>
    <!-- Base application theme. -->
    <style name="AppTheme" parent="Theme.AppCompat.DayNight.NoActionBar">
        <!-- Customize your theme here. -->

        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
    </style>
</resources>
```
```
public class MainActivity extends AppCompatActivity {
    //这个是快速打Toast做的工具类
    Toast_ toast_=new Toast_(this);
    private static final String TAG = "MainActivity";
    Toolbar toolbar;
    @Override
    protected void onCreate(final Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        //设置透明statusBar
        if (Build.VERSION.SDK_INT>Build.VERSION_CODES.KITKAT){
            WindowManager.LayoutParams layoutParams=getWindow().getAttributes();
            layoutParams.flags=WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS;
        }
        toolbar=findViewById(R.id.toolbar);
//注意了,logo、navigationIcon、title、subtitle在xml中设置的话，
//必须要用res-auto那个命名空间，默认的是app:开头的那个，否则设置无效
//，这里为了方便就用代码来设置
        toolbar.setTitle("标题");
        toolbar.setSubtitle("副标题");
        toolbar.setLogo(R.drawable.ic_android_black_24dp);
        toolbar.inflateMenu(R.menu.menu_toolbar);
        toolbar.setNavigationIcon(R.drawable.ic_arrow_back_black_24dp);
        toolbar.setOnMenuItemClickListener(new Toolbar.OnMenuItemClickListener() {
            @Override
            public boolean onMenuItemClick(MenuItem item) {
                switch (item.getItemId()){
                    case R.id.delete:
                        toast_.makeText("delete");
                        break;
                    case R.id.share:
                        toast_.makeText("share");
                        break;
                    case R.id.search:
                        toast_.makeText("search");
                        break;
                }
                return false;
            }
        });
        toolbar.setNavigationOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                toast_.makeText("navigationIcon");
            }
        });
    }
}
```

* 这里需要注意一个就是toolbar的xml属性中的```android:fitsSystemWindows="true"```，官方描述：```adjusts the padding of this view to leave space for the system windows```
他就是防止toolbar内容挤压到statusBar的属性，也可以用paddingTop="25dp"来设置，不过这样会有低版本不兼容的问题，所以用fitSystemWindows是最佳方案。

不用android:fitsSystemWindows="true"的结果：
![](http://upload-images.jianshu.io/upload_images/7177220-3f09bb37a21d4551.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

toolbar的一半都和状态栏重合了...

到此基本的功能都能使用了，以下```进阶设置```是一些可能出现的需求：

-------------------------------------------------------------------------------

##2. 进阶设置：

1. 像微信的menu那样弹出的菜单不会遮挡住拓展按钮，而且还带小图标：
![](http://upload-images.jianshu.io/upload_images/7177220-864ad3313447f6a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
默认效果：
![](http://upload-images.jianshu.io/upload_images/7177220-ae18a463b5ae840e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
定义一个popup的style，然后再toolbar的属性的xml中指定```app:popupTheme="@style/popupTheme"```
```style```如下
```
//让菜单不会挡住拓展按钮
<style name="popupTheme" parent="ThemeOverlay.AppCompat.Dark.ActionBar">
    <item name="overlapAnchor">false</item>
</style>
```
效果：
![](http://upload-images.jianshu.io/upload_images/7177220-b6dbb58ec4af80a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
接下来加图标：
```
//模仿微信的扩展菜单带图标功能
//通过反射来做的一个效果，可能再别的机型或者版本就不适配了
//具体的反射内部什么情况我也不懂
Menu menu=toolbar.getMenu();
if (toolbar.getMenu().getClass().getSimpleName().equalsIgnoreCase("MenuBuilder")){
    try{
        Method method = menu.getClass().getDeclaredMethod("setOptionalIconsVisible", Boolean.TYPE);
        method.setAccessible(true);
        method.invoke(menu, true);
    }catch (Exception e){
        e.printStackTrace();
    }
}
//下面这个方案其实也可以，有机会再深究
//if (menu.getClass().getSimpleName().equals("MenuBuilder")) {
//    try {
//        MenuBuilder menuBuilder = (MenuBuilder) menu;
//        menuBuilder.setOptionalIconsVisible(true);
//    } catch (Exception e) {
//        e.printStackTrace();
//    }
//}
```
因为我的图标都是白色的，所以我就把popup的style改成了Dark风格的，背景就是黑色的了，我的图标就看得清了：
```
<style name="popupTheme" parent="ThemeOverlay.AppCompat.Dark">
        <item name="overlapAnchor">false</item>
</style>
```
效果：![](http://upload-images.jianshu.io/upload_images/7177220-195f3a0410a70452.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


-------------------------------------------------------------------------------
##3.设置popup的背景颜色和字体颜色
```
<style name="popupTheme" parent="ThemeOverlay.AppCompat.Dark">
    <item name="overlapAnchor">false</item>
    //很简单，这两个设置就行，不过注意，是android:开头的
    <item name="android:background">@color/colorPrimary</item>
    <item name="android:textColor">@color/colorAccent</item>
</style>
```
![](http://upload-images.jianshu.io/upload_images/7177220-b03ab4ab16f1b7e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

-------------------------------------------------------------------------------
>一切的方法都介绍完了，这么复杂的配置，一定要封装，我自己封装了toolbar，反手就是一个连接：
[EasyToolbar](https://github.com/William619499149/EasyToolbar2)


还有一个Toolbar教程也说的不错：https://www.jianshu.com/p/e2ae6aaff696

---
完
