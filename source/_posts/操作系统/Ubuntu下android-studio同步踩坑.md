# Ubuntu下android studio同步踩坑

### 1. 安装shadowsocks : https://blog.csdn.net/A807296772/article/details/80112871

### 2. shadowsocks配置

1. 打开客户端

2. 根据购买的账号和密码和服务器地址配置：
3. 注意，本地服务器类型不能选择socks5，而是选择第二个HTTP(S)，否则android studio的gradle无法走代理。

![](https://upload-images.jianshu.io/upload_images/7177220-fb5af5f0973e767a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3. 配置本地代理

![](https://upload-images.jianshu.io/upload_images/7177220-93d3e65be75043cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

不要配成socks主机，就配HTTP代理和HTTPS代理。

### 4. 配置Android studio代理

![](https://upload-images.jianshu.io/upload_images/7177220-77aaadafaa6ce025.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

完成。折腾了接近半天。
