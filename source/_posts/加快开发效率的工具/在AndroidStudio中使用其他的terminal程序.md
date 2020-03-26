默认的termianl程序是windows自带的cmd，用起来十分的不方便，不支持shell的命令。
AS支持对默认termianl的一个切换，在这里我切换成cmder。

1. 安装cmder
2. 配置cmder的目录到系统环境变量：
![](https://upload-images.jianshu.io/upload_images/7177220-8d9a02fba0e2ecd6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3. 将系统环境变量写入到AS的terminal设置中，让AS启动terminal的时候去启动cmder：
`"cmd.exe" /k ""%CMDER_ROOT%\vendor\init.bat"" `

![](https://upload-images.jianshu.io/upload_images/7177220-105044f1661284cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

关闭AS的terminal，重新打开一个即可。

thanks：[https://github.com/cmderdev/cmder/issues/282](https://github.com/cmderdev/cmder/issues/282)

