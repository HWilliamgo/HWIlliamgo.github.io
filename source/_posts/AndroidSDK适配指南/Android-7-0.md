理论上所有的Android版本行为变更都可以在官网查到（自备梯子），这里仅贴一些额外需要注意的地方。

### FileProvider
FileProvider在Manifest里面的生命的<authorities>块不需要用包名，只要名字独特即可，这样在开发sdk的时候就可以避免和用户的FileProvider冲突。
![](https://upload-images.jianshu.io/upload_images/7177220-96ad862cbb55fc86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

