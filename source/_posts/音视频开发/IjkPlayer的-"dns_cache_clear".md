今天遇到了一个在IjkPlayer播放中途切换拉流协议播放失败的问题。

http-flv协议的拉流地址切换到rtmp协议的拉流地址，播放失败，并报-10000的错误。

而销毁再重建播放器，则可以播放成功。



上ijkplayer的issue上面查了同样的几个问题，发现开启如下的选项可以解决问题：

```java
ijkPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_FORMAT, "dns_cache_clear", 1);
```



大致原因是：

rtmp的url为：

```
rtmp://abc
```

http-flv的url为：

``` java
https:abc.flv
```

这两个url指向了相同的ip地址，但是不同的端口号，https的端口号是443，rtmp的端口号是1935。而Ijkplayer应该是对请求的缓存策略有bug，只用了ip地址作为缓存池的key，正确的做法应该是（ip地址+端口号）。



有人已经改了c层的代码，用ip+port作为key，来解决了缓存的bug。https://patchwork.ffmpeg.org/patch/10543/mbox/

而我用的另一种解决方式就是取消掉缓存，就可以解决这个问题。



参考：

[http重定向到rtmp，ijkplayer无法播放视频](https://github.com/bilibili/ijkplayer/issues/3990)

[cannot play hls after playing rtsp stream](https://github.com/bilibili/ijkplayer/issues/4810)

[dns cache 功能有bug，应该以hostname+portstr为key](https://github.com/bilibili/ijkplayer/issues/3700)

[ijkplayer对同一个目标ip会复用tcp链接吗？](https://github.com/bilibili/ijkplayer/issues/4510)
