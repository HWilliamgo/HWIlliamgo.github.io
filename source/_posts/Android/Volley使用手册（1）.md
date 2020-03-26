#1.最简单的StringRequest和JasonRequest：
>1.创建一个RequestQueue对象
2.创建一个Request对象
3.将Request对象添加到RequestQueue里面。
***
* StringRequest:
GET方法：
![](http://upload-images.jianshu.io/upload_images/7177220-b96883d71b580da1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

StringRequest的构造方法第一个参数是url,第二个参数是onResponse接口实例，第三个参数是onErrorResponse接口实例。
* POST方法:
![](http://upload-images.jianshu.io/upload_images/7177220-2bbe18889f2f141c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里的StringRequest是选择四个参数的构造方法，
![](http://upload-images.jianshu.io/upload_images/7177220-6cb90a0d4fb609ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/512)
第一个参数传入的是method，然后重写getParams方法，返回要Post上去的Map。



* JsonRequest:
JsonObjectRequest request=new JsonObjectRequest(4个参数或5个参数);
![](http://upload-images.jianshu.io/upload_images/7177220-7c0cc1f944645f6b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1024)
* 请求JsonObject:
![](http://upload-images.jianshu.io/upload_images/7177220-311a53cdcb12a47b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 请求JsonArray同理


#2. 用Volley加载网络图片
>1. ImageRequest

![](http://upload-images.jianshu.io/upload_images/7177220-81fa719f14451d62.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
ImageRequest的构造函数的第一个参数是图片url
第二个参数是接收到图片之后的listener，第三个第四个是图片的目标尺寸，如果图片大于目标尺寸，则进行缩放，第四个是缩放类型，第五个是Config参数，第六个是出错listener。
```
queue= Volley.newRequestQueue(this);
//maxWidth和maxHeight如果传入都是0，就是不缩放。
String url="http://b.hiphotos.baidu.com/image/h%3D220/sign=6e0a13c50055b31983f9857773ab8286/279759ee3d6d55fb733229e267224f4a21a4dd7a.jpg";
final ImageRequest imageRequest=new ImageRequest(url, new Response.Listener<Bitmap>() {
    @Override
    public void onResponse(Bitmap response) {
        imageView.setImageBitmap(response);
    }
}, 0, 0, ImageView.ScaleType.CENTER_CROP, Bitmap.Config.RGB_565, new Response.ErrorListener() {
    @Override
    public void onErrorResponse(VolleyError error) {

    }
});
queue.add(imageRequest);
```
只要你在maxWidth和maxHeight传的不是0，构造方法内部就会去处理优化bitmap的事情，和我另一篇介绍bitmap内存优化用的方法是一样的。

运行程序：
![image.png](http://upload-images.jianshu.io/upload_images/7177220-df36920b2ac95d46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/300)

>2. ImageLoader

用法：
```
queue = Volley.newRequestQueue(this);
ImageLoader imageLoader = new ImageLoader(queue, new MyImageCache());
ImageLoader.ImageListener imageListener = ImageLoader.getImageListener(imageView, R.mipmap.ic_launcher, R.mipmap.ic_launcher_round);
imageLoader.get(url, imageListener);
```
运行：![](http://upload-images.jianshu.io/upload_images/7177220-130cd0dafb691ca2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

解释一下上面代码的几个东西：
* ImageLoader的构造方法的参数：RequestQueue和ImageCache
而ImageCache是一个接口，我自己实现了这个接口：
![](http://upload-images.jianshu.io/upload_images/7177220-3bb99477899e58a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
*上面两个重写的方法getBitmap和putBitmap都是ImageLoader的源码内部调用的，作用分别是从缓存中取Bitmap和将Bitmap放入缓存，每次调用imageloader.get(url,imageListener);的时候，先调用我们传入的这个接口的getBitmap方法，如果返回的是null就去网络请求然后putBitmap，如果返回不是null那就直接可以用缓存里面的bitmap（具体情况看源码）*

至于LruCache是什么东西？看一下郭霖的这篇文章吧：
[Android高效加载大图、多图解决方案，有效避免程序OOM](http://blog.csdn.net/guolin_blog/article/details/9316683)

ImageListener是ImageLoader.getImageListener()来的，参数是![](http://upload-images.jianshu.io/upload_images/7177220-b04da80cd876525c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>3.NetWorkImageView

```
queue = Volley.newRequestQueue(this);
ImageLoader imageLoader = new ImageLoader(queue, new MyImageCache());

NetworkImageView networkImageView=findViewById(R.id.netWorkImageView);
networkImageView.setDefaultImageResId(R.mipmap.ic_launcher);
networkImageView.setErrorImageResId(R.mipmap.ic_launcher);
networkImageView.setImageUrl(url,imageLoader);
```
xml:
```
<com.android.volley.toolbox.NetworkImageView
    android:id="@+id/netWorkImageView"
    android:layout_width="200dp"
    android:layout_gravity="center"
    android:layout_height="200dp" />
```
他的好处是：代码简单，不需要代码上去进行缩放操作，他会根据xml中定义的宽和高进行自动缩放，不会多占用一点内存。

