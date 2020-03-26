## 简单的介绍TabLayout的常规用法
**效果图：**![](http://upload-images.jianshu.io/upload_images/7177220-ff5e0d2a16fafebb.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/200)

### *布局：*
布局上面就是很简单的采用上面TabLayout下面ViewPager的形式
### *代码：*
在xml中声名的viewPager需要调用`viewPager.setAdapter(adapter)`，那么TabLayout需要调用`tablayout.setUpwithViewPager(viewPager)`
思路是：
* 写findViewById（）找到这两个控件。
* 写一个类MyViewPagerAdapter继承FragmentPagerAdapter，ViewPager里面展示的内容当然的是用的Fragment.
* 那么就要写我们自己的Fragment类，用来放数据，放RecyclerView来装数据，创建一个BaseFragment。
* 创建一个MyFragment继承BaseFragment
* 创建RecyclerView的RvAdapter。
---
代码依次给出：
``` xml
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.lovely_solory.super_william.cact.Activity_ViewPager">
    <android.support.design.widget.AppBarLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <android.support.v7.widget.Toolbar
            android:id="@+id/toolbar_activityViewPager"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            android:minHeight="?attr/actionBarSize"
            app:layout_scrollFlags="scroll|enterAlways"
            app:navigationIcon="@drawable/ic_arrow_back_white_24dp" />
        <android.support.design.widget.TabLayout
            android:id="@+id/tabLayout"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />
    </android.support.design.widget.AppBarLayout>
    <android.support.v4.view.ViewPager
        android:id="@+id/viewPager_activityViewPager"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:layout_behavior="@string/appbar_scrolling_view_behavior" />
</android.support.design.widget.CoordinatorLayout>
```

---
Activity:
``` java
public class Activity_ViewPager extends AppCompatActivity {
    TabLayout tabLayout;
    ViewPager viewPager;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity__view_pager);

        initView();
    }
    private void initView() {
        viewPager=findViewById(R.id.viewPager_activityViewPager);
        tabLayout=findViewById(R.id.tabLayout);

        viewPager.setAdapter(new MyViewPagerAdapter(this,this.getSupportFragmentManager()));
        tabLayout.setupWithViewPager(viewPager);
    }
}
```
FragmentPagerAdapter:
``` java
public class MyViewPagerAdapter extends FragmentPagerAdapter {
    private static final int pageCount = 3;
    private Context context;

    public MyViewPagerAdapter(Context context, FragmentManager fm) {
        super(fm);
        this.context = context;
    }
    @Override
    public Fragment getItem(int position) {
        int type;
        switch (position) {
            case 0:
                type = 0;
                break;
            case 1:
                type = 1;
                break;
            case 2:
                type = 2;
                break;
            default:
                type = 0;
                break;
        }
        return MyFragment.newInstance(type);
    }
    @Override
    public int getCount() {
        return pageCount;
    }
    @Nullable
    @Override
    public CharSequence getPageTitle(int position) {
        switch (position) {
            case 0:
                return "广场";
            case 1:
                return "好友";
            case 2:
                return "我";
            default:
                return "广场";
        }
    }
}
```
BaseFragment:
``` java
public abstract class BaseFragment extends android.support.v4.app.Fragment{
    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, Bundle savedInstanceState) {
        View view=inflater.inflate(R.layout.fragement_view_pager,container,false);
        initView(view,savedInstanceState);
        return view;
    }
    protected abstract int getLayoutResId();
    protected void initView(View view,Bundle savedInstanceState){

    }
}
```
MyFragment:
``` java
public class MyFragment extends BaseFragment {
    private static final String a="william.cact";
    private int type;

    private RecyclerView rv;
    private RvAdapter rvAdapter;
    private List<String> rvContentList;
    public static Fragment newInstance(int type){
        Bundle args=new Bundle();
        args.putInt(a,type);
        MyFragment fragment=new MyFragment();
        fragment.setArguments(args);
        return fragment;
    }

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        type=getArguments().getInt(a);
    }

    @Override
    protected int getLayoutResId() {
        return R.layout.fragement_view_pager;
    }

    @Override
    protected void initView(View view, Bundle savedInstanceState) {
        //展示各种逻辑
        rvContentList=new ArrayList<>();
        for (int i=0;i<20;i++){
            rvContentList.add("我是第"+i+"组数据");
        }
        rvAdapter=new RvAdapter(this.getContext(),rvContentList);
        rv=view.findViewById(R.id.rv);
        rv.setLayoutManager(new LinearLayoutManager(view.getContext()));
        rv.setAdapter(rvAdapter);
    }
}
```
RecyclerView的适配器就不放代码了，比较简单。

---
此外，TabLayout的标题部分不止可以显示文字，也可以显示自定义布局。主要用的方法是：
`tabLayout.getTabAt(position).setCustomView(imageView/int layoutResId);`

试一下在Activity下面加这些代码：
``` java
ImageView imageView = new ImageView(this);
imageView.setImageResource(R.drawable.beauty);
tabLayout.getTabAt(0).setCustomView(imageView);
ImageView imageView1 = new ImageView(this);
imageView1.setImageResource(R.drawable.ic_arrow_back_white_24dp);
tabLayout.getTabAt(1).setCustomView(imageView1);
```


![](http://upload-images.jianshu.io/upload_images/7177220-08d1db697eaa1c7c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

