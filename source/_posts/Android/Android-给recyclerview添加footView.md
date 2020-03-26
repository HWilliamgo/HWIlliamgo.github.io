__效果图__:

![随便找了一个软件来录制，效果好差](http://upload-images.jianshu.io/upload_images/7177220-961dfa84256aa3cc.gif?imageMogr2/auto-orient/strip)

首先新建一个recyclerview

以下是MyAdapter
```
public class MyAdapter extends RecyclerView.Adapter<MyAdapter.MyViewHolder> {
    private Context context;
    private List<String> list;
    MyAdapter(List<String> list, Context context) {
        this.list = list;
        this.context = context;
    }


//onCreateViewHolder
    @Override
    public MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View itemView = LayoutInflater.from(context).inflate(R.layout.rv_item, parent, false);
        return new MyViewHolder(itemView, viewType);
    }
}

    //返回item数量
   @Override
    public int getItemCount() {
        return list.size() ;
    }

    class MyViewHolder extends RecyclerView.ViewHolder {
        TextView textView;
        MyViewHolder(View itemView, int viewType) {
            super(itemView);
            textView = itemView.findViewById(R.id.textView);
        }
    }
```
非常基础简洁的一个Adapter。__但是注意这里的onCreateViewHolder(ViewGroup parent, int viewType)，他的第二个参数是viewType,这是关键的地方，就是用来设置footView的。__
___
__我们利用MyAdapter内部的一个getViewType方法来帮助我们获得FootView的效果：__
```
public class MyAdapter extends RecyclerView.Adapter<MyAdapter.MyViewHolder> {
    ........
    ........
    //这两个静态变量来区分是normalView还是footView。
    private static final int NORMAL_VIEW = 0;
    private static final int FOOT_VIEW = 1;
    //重写一下getItemViewType,返回两个我们自己定义的参数，美滋滋~
    @Override
    public int getItemViewType(int position) {
        if (position == getItemCount() - 1) {
            return FOOT_VIEW;
        }
        return NORMAL_VIEW;
    }
```
上面是我们在MyAdapter中重写的方法
__下面是我们需要去修改的几个地方__

1. 修改内部类MyViewHolder
```

    class MyViewHolder extends RecyclerView.ViewHolder {
        TextView textView;
        RelativeLayout footView;
        //在MyViewHolder构造方法中添加一个新的参数，int ViewType,然后判断。
        MyViewHolder(View itemView,int viewType) {
            super(itemView);
            //如果是normalView那么给textView(就是item的内容)赋值。
            if (viewType == NORMAL_VIEW) {
                textView = itemView.findViewById(R.id.textView);
            //如果是footView那么给footView赋值。
            } else if (viewType == FOOT_VIEW) {
                footView = (RelativeLayout) itemView;
            }
        }
    }
```
2. 修改getItemCount方法
```
    //因为我们多加了一个footView，所以这个地方+1.
    @Override
    public int getItemCount() {
        return list.size() + 1;
    }
```
3. 修改onCreate方法：

两种不同的viewType，判断一下，返回的MyViewHolder都是不一样的。
```
    @Override
    public MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        if (viewType == NORMAL_VIEW) {
            View itemView = LayoutInflater.from(context).inflate(R.layout.rv_item, parent, false);
            return new MyViewHolder(itemView, viewType);
        } else {
            View footView = LayoutInflater.from(context).inflate(R.layout.foot_view, parent, false);
            return new MyViewHolder(footView, viewType);
        }

    }
```

4. 修改onBind方法：
简单干脆，加一个if判断完事，不是normalView的话还去bind数据干啥捏？
```
    @Override
    public void onBindViewHolder(final MyViewHolder holder, int position) {
        if (getItemViewType(position) == NORMAL_VIEW) {
            holder.textView.setText(list.get(position));
        }
    }
```
好的到了这里就结束了，但是我们有footView来干嘛？就是制造一个上拉加载的效果，制造一个加载中ing的view，下面来改进一下。一点点简单的代码就OK了。
```
RecyclerView rv;
rv= (RecyclerView) findViewById(R.id.rv);
//添加滑动监听。
rv.addOnScrollListener(new RecyclerView.OnScrollListener() {
            //这个int用来记录最后一个可见的view
            int lastVisibleItemPosition;

            @Override
            public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
                super.onScrollStateChanged(recyclerView, newState);
                if (newState==RecyclerView.SCROLL_STATE_IDLE&&lastVisibleItemPosition+1==adapter.getItemCount()){
                    list.add(String.valueOf(adapter.getItemCount()));
                    list.add(String.valueOf(adapter.getItemCount()));
                    //要在UI线程中更新，用一个Handler。
                    new Handler().postDelayed(new Runnable() {
                        @Override
                        public void run() {
                            adapter.notifyItemRangeInserted(adapter.getItemCount()-1,adapter.getItemCount());
                        }
                    },1000);

                }
            }

            @Override
            public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
                super.onScrolled(recyclerView, dx, dy);
                //这个llm是LinearLayoutManager的一个实例。
                lastVisibleItemPosition=llm.findLastVisibleItemPosition();
            }
        });
```

![么么哒😙](http://upload-images.jianshu.io/upload_images/7177220-d142876e603dbd90.gif?imageMogr2/auto-orient/strip)
