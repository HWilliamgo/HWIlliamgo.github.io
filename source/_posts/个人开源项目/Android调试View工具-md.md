---
title: Android调试ViewTree工具.md
tags:
  - Android
  - 开源
  - 调试工具
index_img: 'https://s1.ax1x.com/2020/05/05/YiGoRK.png'
banner_img: 'https://s1.ax1x.com/2020/05/05/Yil2NV.png'
categories:
  - 个人开源项目
date: 2020-05-05 14:09:50
---



### AndroidStudio自带的LayoutInspector

在Android开发的时候，我们在调试复杂的UI界面上的问题的时候，有时希望借助AndroidStudio自带的调试工具：LayoutInspector来查看当前界面的View Tree。

![YiUNAH.png](https://s1.ax1x.com/2020/05/05/YiUNAH.png)

没有遇到问题的话，他能出现这样的调试效果：

[![YiUDjf.png](https://s1.ax1x.com/2020/05/05/YiUDjf.png)](https://imgchr.com/i/YiUDjf)



### 调试UI遇到的问题

但是，当你的Activity的界面非常复杂，例如存在大量的View，存在视频，存在View动画等情况。这时这个调试工具就不生效了。

这时会遇到：

![](https://s1.ax1x.com/2020/05/05/YiGoRK.png)

Error obtaining view hierarchy : There was a timeout error capturing the layout data from the devices.

The device may be too slow , the captured view may be too complex, or the view may contain animations.

Please retry with a simplified view and ensure the device is responsive.



### 解决方案

1. 去代码里找答案，去代码里找线索，并在关键的代码行中添加调试日志，通过日志来调试和解决问题。
2. 用别的View tree调试工具。



### FastViewTree

用上述的第一种方式，去代码里加调试日志或者阅读代码能不能解决问题？能。但是很耗时。比如有一种情况是：你去调试或者接管他人写的代码中出现的问题，或者是你自己写的比较久远的问题，去阅读代码来模拟想象出UI表现，是要花比较长的时间的。

我没有找到已经发出来的其他合适的调试工具能够满足我的需要：打印view tree，打印每个view的可见度和宽高。

因此我自己写了一个满足上述需求的调试工具：[FastViewTree](https://github.com/HWilliamgo/FastViewTree)

代码是用kotlin写的，只有一个文件，非常简单。不过这个工具很实用。欢迎大家提意见和start 。



