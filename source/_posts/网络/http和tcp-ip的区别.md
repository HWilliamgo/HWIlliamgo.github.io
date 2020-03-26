一个简单通俗易懂的解释：[HTTP协议—— 简单认识TCP/IP协议](http://www.cnblogs.com/roverliang/p/5176456.html)

---
解释：
* http协议负责的是记录应用层的请求消息和响应消息。
* TCP负责的是传输层，将http中的大段的消息切割成小的以报文段（segment）为单位的数据包，为了更容易传输数据。
* Ip负责记录己方和对方的ip地址，自己的mac地址和下一个路由器或计算机的mac地址（为了转发数据）。

---
## 通俗易懂的图解：

![](http://upload-images.jianshu.io/upload_images/7177220-2fea9bf15abee01d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

访问一个网页时各种协议的作用：

![](http://upload-images.jianshu.io/upload_images/7177220-f852dd016a6168e8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---
网络数据包结构：

![](http://upload-images.jianshu.io/upload_images/7177220-989c10c688d3f17c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

![](http://upload-images.jianshu.io/upload_images/7177220-7b115ca117e378ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


