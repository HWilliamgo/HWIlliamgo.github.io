
### 这个bug可以说是折磨了我很久了，问了人，查了博客，最后找到了答案。
首先要搞明白RecyclerView中的ViewHolder的复用机制是什么，在，以及由于复用机制和请求网络mix在一起之后会发生哪些可能的事故，强烈推荐一个好文：
>[RecyclerView中ViewHolder重用机制理解(解决图片错乱和闪烁问题)](http://blog.csdn.net/xyq046463/article/details/51800095)
---
![他的博客取来的图片](https://upload-images.jianshu.io/upload_images/7177220-043486f24787a3e7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

总结一下就是：
* RecyclerView用的是我们自定义的内部类ViewHolder来复用的，也就是复用的是ViewHoler
* 当屏幕下滑，item1滑出可视区域，将item1的ViewHolder对象给item8复用，那么此时item1中ViewHolder对象中持有的变量都是item1的。
* item1中的ViewHolder对象，在onBindViewHolder(MyViewHolder holder, int position)方法中对holder进行更新，但是如果在这里调用glide去从url加载图片到holder中的imageView对象的话，就有可能因为网络延迟，导致图片加载不出来，那么item8就会先显示item1的图片，过一会延迟之后，显示正确的item8该显示的图片

上面博客加载图片用的是AsynTask，我用的是Glide框架，ViewHolder中3个TextView,一个ImageView,按照那个思路，我的处理方法如下：
``` java
@Override
    public void onBindViewHolder(MyViewHolder holder, int position) {
        if (holder == null) {
            return;
        }
        holder.tvDesc.setText(resultsBeanList.get(position).getDesc());
        holder.tvPublishedAt.setText(timeParse.getTime(resultsBeanList.get(position).getPublishedAt()));
        Object who = resultsBeanList.get(position).getWho();
        if (who != null) {
            holder.tvWho.setText((String) who);
        }else {
            //防止ViewHolder复用导致上一个tvWho的内容遗留
            holder.tvWho.setText("");
        }

        //处理imageView--------------
        List<String> imagesUrl = resultsBeanList.get(position).getImages();

        if (imagesUrl == null) {
            //当ViewHolder复用的时候，如果当前返回的图片url为null，为了防止上一个复用的viewHolder图片
            //遗留，要clear并且将图片设置为空。
            Glide.with(fragment).clear(holder.ivImage);
            holder.ivImage.setImageDrawable(null);
            holder.ivImage.setTag(R.id.image_tag, position);
            return;
        }
        Object tag=holder.ivImage.getTag(R.id.image_tag);
        if (tag!=null&&(int) tag!= position) {
            //如果tag不是Null,并且同时tag不等于当前的position。
            //说明当前的viewHolder是复用来的
            //Cancel any pending loads Glide may have for the view
            //and free any resources that may have been loaded for the view.
            Glide.with(fragment).clear(holder.ivImage);
        }
        String url = imagesUrl.get(0);
        Glide.with(fragment)
                .load(url + "?imageView2/0/w/100")
                .apply(options)
                .into(holder.ivImage);
        //给ImageView设置唯一标记。
        holder.ivImage.setTag(R.id.image_tag, position);
    }
```
至此，不再图片混乱。
