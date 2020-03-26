此前我添加item的监听事件是直接在Adapter里面的onCreate方法中直接holder.view.setOnClickListener(new OnClickListener){};
__此方法是学的郭霖的第二行代码。__

后面在hongyang的博客学的下面这个方法：
```
    public interface OnItemClickListener {
        void onItemClick(View v,int position);
    }

    public interface OnItemLongClickListener {
        void onItemLongClick(View v,int position);
    }

    private OnItemClickListener onItemClickListener;
    private OnItemLongClickListener onItemLongClickListener;
```
第一步：分别写两个public的接口，接口里面放方法，方法的参数注意了，一个是View一个是int，int 用来记录position。
第二步：引用两个private的接口。

```
    public void setOnItemClickListener(OnItemClickListener listener){
        onItemClickListener= listener;
    }
    public void setOnItemLongClickListener(OnItemLongClickListener listener){
        onItemLongClickListener=listener;
    }
```
第三步：添加两个public的方法，供调用者调用，参数放接口进去，这样的效果是：只要调用者调用了这个方法，我们就强迫调用者去实现我们参数里面放的这个接口。
```
    @Override
    public void onBindViewHolder(final MyViewHolder holder, int position) {
        if (getItemViewType(position) == NORMAL_VIEW) {
            holder.textView.setText(list.get(position));
            if (onItemClickListener!=null){
                holder.textView.setOnClickListener(new View.OnClickListener() {
                    @Override
                    public void onClick(View view) {
                        onItemClickListener.onItemClick(holder.itemView,holder.getLayoutPosition());
                    }
                });
            }
            if (onItemLongClickListener!=null){
                holder.textView.setOnLongClickListener(new View.OnLongClickListener() {
                    @Override
                    public boolean onLongClick(View view) {
                        onItemLongClickListener.onItemLongClick(holder.itemView,holder.getLayoutPosition());
                        return false;
                    }
                });
            }
        }
    }
```
第四步：在onBind方法里面，先来一个if来判断我们的接口是不是null，是null，跳过,不是null，说明被实现了，进入，开始设置监听方法，具体的代码。

__现在可以跑回Activity里面去设置监听了!__
```
adapter.setOnItemClickListener(new MyAdapter.OnItemClickListener() {
            @Override
            public void onItemClick(View v, int position) {
                Toast.makeText(MainActivity.this,"the number"+position+"has been clicked",Toast.LENGTH_SHORT).show();
            }
        });
        adapter.setOnItemLongClickListener(new MyAdapter.OnItemLongClickListener() {
            @Override
            public void onItemLongClick(View v, int position) {
                list.remove(position);
                adapter.notifyItemRemoved(position);
                 Toast.makeText(MainActivity.this,"the number"+position+"has been removed",Toast.LENGTH_SHORT).show();
            }
        });
```


![~](http://upload-images.jianshu.io/upload_images/7177220-491f2a7c38b9c43a.gif?imageMogr2/auto-orient/strip)
