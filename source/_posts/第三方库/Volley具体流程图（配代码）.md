只简要分析Volley大体原理，无细节，无Volley使用教程。

---
使用Volley时必要的两步：
```
RequestQueue queue= Volley.newRequestQueue(context);1
queue.add(request);  //request可以是StringRequest,ImageRequest,JsonRequestd等。
```
第一步通过`Volley.newRequestQueue(Context)`初始化`queue`，第二步往`queue`中添加具体请求。

## 第一步
![image.png](https://upload-images.jianshu.io/upload_images/7177220-5ff1cb794f8a63b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 第二步
![image.png](https://upload-images.jianshu.io/upload_images/7177220-dadd1d6aa55ad823.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
本地磁盘缓存==>Cache接口的实现类：DiskBasedCache  
Cache接口中定义了内部类Entry封装了响应体的数据
DiskBasedCache定义了对磁盘读写的操作，规定了默认的缓存大小为5M，
缓存路径为getCacheDir()/Volley
```

突然发现把程序的逻辑走向用笔和纸写出来思路会非常清晰，原稿如下
![](https://upload-images.jianshu.io/upload_images/7177220-1c9d206fac5a23d8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/200)
![](https://upload-images.jianshu.io/upload_images/7177220-de7d92a7ba79c65d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/200)

官网图：
![](https://upload-images.jianshu.io/upload_images/7177220-3085aaa6e7800839.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
