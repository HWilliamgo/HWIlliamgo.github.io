CoordinatorLayout是一个FrameLayout，也就是说如果不做特殊处理，里面的子布局是无法控制的，超过一个，就会糊在一起，但是CoordiatorLayout又很特殊，只有在作为layout.xml的顶层布局才能发挥他协调子view的作用。因此要控制他的多个子View，比如除了AppBarLayout之外，要在下面加一个TabLayout或者RecyclerView,就要在他们里面加上`app:layout_behavior=""`属性
```
<android.support.v7.widget.RecyclerView
    android:id="@+id/rv"
    android:layout_width="match_parent"
    app:layout_behavior="@string/appbar_scrolling_view_behavior"
    android:layout_height="wrap_content" />
```

界面：![](http://upload-images.jianshu.io/upload_images/7177220-cc1660b2a345f2cb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/200)

那么他的布局文件可以这样：
```
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.lovely_solory.super_william.drawer.MainActivity">

        <android.support.design.widget.AppBarLayout
            android:id="@+id/appbar"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">
            <android.support.v7.widget.Toolbar
                android:id="@+id/toolbar"
                android:layout_width="match_parent"
                android:layout_height="200dp"
                android:background="#000"
                android:fitsSystemWindows="true"
                android:gravity="bottom"
                android:minHeight="?attr/actionBarSize"
                app:layout_scrollFlags="scroll|exitUntilCollapsed"
                app:popupTheme="@style/ThemeOverlay.AppCompat.Light"
                app:titleTextColor="#ffff" />
        </android.support.design.widget.AppBarLayout>

    <android.support.v7.widget.RecyclerView
        android:id="@+id/rv"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:layout_behavior="@string/appbar_scrolling_view_behavior" />
</android.support.design.widget.CoordinatorLayout>
```
