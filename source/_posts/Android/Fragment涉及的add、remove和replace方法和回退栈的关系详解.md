前部分参考：
[Fragment涉及的add、remove和replace方法和回退栈的关系详解](http://blog.csdn.net/haozipi/article/details/46994801)

划重点：
1. add方法不会加入回退栈，只会在container的view里面一层一层不断往上面涂layout。
```
this.layout = (FrameLayout) this.findViewById(R.id.contentFrame);

        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                FragmentManager fragmentManager=getFragmentManager();
                FragmentTransaction transaction=fragmentManager.beginTransaction();
                transaction.add(R.id.contentFrame,new Frag_1(),"frag_1");
                transaction.commit();
                Log.d("tag", String.valueOf(layout.getChildCount()));
            }
        });
        btn_2.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                FragmentManager fragmentManager=getFragmentManager();
                FragmentTransaction transaction=fragmentManager.beginTransaction();
                transaction.add(R.id.contentFrame,new Frag_2(),"frag_2");
                transaction.commit();
                Log.d("tag", String.valueOf(layout.getChildCount()));
            }
        });
```
然后两个Button交换按就是：

![](http://upload-images.jianshu.io/upload_images/7177220-f733482a38a04df0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. 
**add、remove和replace方法和回退栈,没有任何关系**
没有任何关系
没有任何关系
没有任何关系
（这一点困惑了我贼久，终于在那篇连接的博客里找到了答案，fuck~）

复制粘贴博客原话：
>这里有一个点非常容易混淆，就是因为add也是一层一层的往FrameLayout里添加Fragment，那么在按回退按钮的时候是不是一层一层的再拿出来呢？笔者这里告诉大家，根本不会，除非加入了回退栈。因为add、remove和replace只是相当于在这个界面层次上做操作，和回退栈没有关系，即使在某些地方看起来很像（回退栈是一个一个的添加进去，按回退的时候会一个一个弹出来）。
那么通过add添加Fragment之后加入回退栈，和通过replace替换Fragment之后加入回退栈有什么不一样呢？
这点就要参照在界面的关系了，add是一层一层往上叠，如果你在其中一层上做了修改，等回退到这一层时，所做的操作会被保留，并且回退的时候会一层一层把Fragment往外拿。replace实际上是替换掉了，那么虽然加入了回退栈，但是会执行销毁视图的方法onDestroyView，回退时会重新执行onCreateView方法重建视图。

---

# 2018年2月25日新增内容：关于Fragment回退栈（原创）

结论：回退栈回退的是一整个commit之前的操作`（事务）`，而不是回退某一个Fragment。

*截图官方文档*
![](http://upload-images.jianshu.io/upload_images/7177220-33c800abf70f0602.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>在调用应用所有的更改都将作为单一事务添加到返回栈。

例子：
``` java
public class BlankFragment extends Fragment {

    public BlankFragment() {
        // Required empty public constructor
    }
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        // Inflate the layout for this fragment
        View view = inflater.inflate(R.layout.fragment_blank, container, false);
        TextView textView= view.findViewById(R.id.bf_tv);
        Log.d("TAG",getArguments().getString("number","null"));
        textView.setText(getArguments().getString("number"));
        return view;
    }
}
```
``` java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        findViewById(R.id.btn).setOnClickListener(new View.OnClickListener() {
            private int pressCount=0;
            @Override
            public void onClick(View v) {
                FragmentManager manager=getSupportFragmentManager();
                FragmentTransaction transaction=manager.beginTransaction();

                BlankFragment fragment=new BlankFragment();
                Bundle bundle=new Bundle();
                bundle.putString("number", String.valueOf(pressCount++));
                fragment.setArguments(bundle);

                transaction.add(R.id.frameLayout,fragment);
//                transaction.addToBackStack(null);
                transaction.commit();
            }
        });
    }
```
先用add()，并且不添加回退栈。看看效果：
[图片上传中...(image.png-c06906-1519563125564-0)]

![add.gif](http://upload-images.jianshu.io/upload_images/7177220-4f9c74d5a1f5a972.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/200)
一层一层叠上去的
看看replace，并且不添加回退栈：
![](http://upload-images.jianshu.io/upload_images/7177220-9b21f265a54337f4.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/200)

 将注释解除，两个各自添加回退栈的效果：
add:
* ![](http://upload-images.jianshu.io/upload_images/7177220-059fdb645b5710dc.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/200)
replace
* ![](http://upload-images.jianshu.io/upload_images/7177220-6a673acf8c165f26.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/200)





