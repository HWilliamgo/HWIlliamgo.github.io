
当有9个不同的Item显示在屏幕上，一定有超过9个的ViewHolder对象，通过`onCreateViewHolder(ViewGroup parent, int type)`被创建（也就是其引用的View对象被创建）。

根据官方文档阅读到的资料，大致整理了一下RecyclerView的3个方面：
### 创建Item
1. 从.xml布局文件中`inflate`出新的View对象.
2. 上述新建的View对象作为参数传入ViewHolder的构造函数来创建ViewHolder对象。

### 显示或更新数据
1. 为每个ViewHolder对象分配一个position，并调用`onBindViewHolder(ViewHolder holder, int position)`，来将数据写入ViewHolder中引用的View。
2. 当数据实体或者数据在结构上改变（如增减list中的元素），用`mAdapter#notifyXXX()`，Adapter会自动重新进行必要的数据更新。

### 复用
每当滑动RecyclerView时，被滑出的Item的View不会从内存中释放，RecyclerView通过引用着ViewHolder，保存着滑出的Item中的View对象，这样在新滑入的Item要进入用户视线时，不用再去.xml文件中加载新的View对象，而是拿到刚才被ViewHolder引用着的View对象，通过onBindViewHolder()重新为其写入数据，后其回到用户视线，实现了View对象的复用。


做了个实验，写了一个一次展示9个Item的RecyclerView，在RecyclerView的Adapter中的`onCreateViewHolder`和`onBindViewHolder()`中打印出传入的ViewHolder中的itemView的`hashCode`值。
1. 当RecylerView显示到界面时，log显示为每个ViewHolder依次创建并写入数据，屏幕上面有9个Item，但是log中创建了10个Item。
```
onCreateViewHolder: 创建ViewHolder，引用着View:android.widget.FrameLayout{42b36fb0 V.E..... ......I. 0,0-0,0}
onBindViewHolder: 将数据写入ViewHolder中的View:android.widget.FrameLayout{42b36fb0 V.E..... ......I. 0,0-0,0}
```
2. 向下滑动时，根据观察`hashCode`，又创建并写入数据了3个新的View，即此时已经创建了13个View，或者说13个ViewHolder了。
3. 当再滑动时，不再调用`onCreateViewHolder()`，只会不断地调用`onBindViewHolder()`，并且打印出的hashCode值都是前面13个View的hashCode值中出现过的，这说明此时不再创建View了，而是通过复用从屏幕中滑出的View来重新写入数据，来展示。

