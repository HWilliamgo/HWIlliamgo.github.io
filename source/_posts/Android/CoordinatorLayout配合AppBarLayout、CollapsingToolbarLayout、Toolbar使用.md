
## 这篇文章主要介绍的是CoordinatorLayout，AppBarLayout,CollapsingToolbarLayout和Toolbar的结合表现出的动态效果。
效果如图：
![20180121_142710.gif](http://upload-images.jianshu.io/upload_images/7177220-496a26d291106cde.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**首先介绍上述几个控件在使用的时候的注意点：**
  * CoordinatorLayout：
	  * 是一个FrameLayout
  * AppBarLayout：
	  * 是一个vertical的LinearLayout，其子View应通过`setScrollFlags(int)`或者xmL中的`app:layout_scrollFlags`来提供他们的Behavior。
		  * 具体的`app:layout_scrollFlags`有这么几个： scroll, exitUntilCollapsed, enterAlways, enterAlwaysCollapsed, snap
	  * 他必须严格地是CoordinatorLayout的子View，不然他一点作用都发挥不出来。
	  * AppBarLayout下方的滑动控件，比如RecyclerView，NestedScrollView（与AppBarLayout同属于CoordinatorLayout的子View,并列的关系，）,必须严格地通过在xml中指出其滑动Behavior来与AppBarLayout进行绑定。通常这样：`app:layout_behavior="@string/appbar_scrolling_view_behavior"`
  * CollapsingToolbarLayout:
	  * 是一个专门用来包裹Toolbar的控件，里面可以放置一个imageView和一个toolbar然后轻松地实现：随着滑动，图片和toolbar的标题也有动画。
	  * 内部的子View一般都要加上属性：app:layout_collapseMode=""，常用的是parallax，pin。parallax是视差滚动，用在imageView, pin是固定，用在toolbar。
	  * 用`setContentScrimColor(int)或者setContentScrim(drawable)`来设置内容纱布，就是当折叠到只剩下Toolbar的时候，用一个另外的图片或者颜色来设置toolbar的背景。
  * Toolbar:
	  * 他的title如果需要带有CollapsingToolbarLayout的动画的话，就要用collapsingToolbarLayout.setTitle(); 否则是没有动画的，其他的和toolbar平时一样。


**xml文件**：
``` xml
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    xmlns:app="http://schemas.android.com/apk/res-auto">
    <android.support.design.widget.AppBarLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <android.support.design.widget.CollapsingToolbarLayout
            android:layout_width="match_parent"
            android:id="@+id/collapsingToolbar"
            app:layout_scrollFlags="scroll|exitUntilCollapsed"
            android:layout_height="wrap_content">
            <ImageView
                android:scaleType="centerCrop"
                android:src="@drawable/beauty"
                android:layout_width="match_parent"
                app:layout_collapseMode="parallax"
                android:layout_height="300dp" />
            <android.support.v7.widget.Toolbar
                android:id="@+id/toolbar"
                app:layout_collapseMode="pin"
                android:layout_width="match_parent"
                android:minHeight="?attr/actionBarSize"
                android:layout_height="?attr/actionBarSize">

            </android.support.v7.widget.Toolbar>
        </android.support.design.widget.CollapsingToolbarLayout>
    </android.support.design.widget.AppBarLayout>
    <android.support.v4.widget.NestedScrollView
        android:layout_width="match_parent"

        app:layout_behavior="@string/appbar_scrolling_view_behavior"
        android:layout_height="wrap_content">
        <TextView
            android:text="@string/textContent"
            android:textSize="20sp"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />

    </android.support.v4.widget.NestedScrollView>

</android.support.design.widget.CoordinatorLayout>
```
![](http://upload-images.jianshu.io/upload_images/7177220-e977b2e8b433d90f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**MainActivity:**
``` java
public class MainActivity extends AppCompatActivity {
    Toolbar toolbar;
    CollapsingToolbarLayout collapsingToolbarLayout;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.coordinatory_layout);
        //toolbar
        toolbar = findViewById(R.id.toolbar);
        toolbar.setNavigationIcon(R.drawable.ic_arrow_back_white_24dp);
        //Palette用来更漂亮地展示配色
        Palette.from(BitmapFactory.decodeResource(getResources(),R.drawable.beauty))
                .generate(new Palette.PaletteAsyncListener() {
                    @Override
                    public void onGenerated(@NonNull Palette palette) {
                        int color=palette.getVibrantColor(getResources().getColor(R.color.colorAccent));
                        collapsingToolbarLayout.setContentScrimColor(color);
						//因为我暂时没有找到比较好的透明状态栏来适配这一套效果布局。
						//因此就直接替换掉StatusBar的颜色，这样其实也蛮好看的。
                        getWindow().setStatusBarColor(color);
                    }
                });
        //CollapsingToolbarLayout
        collapsingToolbarLayout = findViewById(R.id.collapsingToolbar);
        collapsingToolbarLayout.setTitle("我是一个标题啊哈哈哈");
    }
}
```
结束。
  
  
