---
title: Android视图调试技巧
date: 2020-04-14 22:42:59
tags:
index_img:
banner_img:
---



### 1. 追踪某个View的方法调用。

如果某个View的尺寸错乱，有可能是这个View的关于设置尺寸的某个方法在某个你没有找到的地方被调用了。

例如`setLayoutParams()`方法被意外地调用了。

如何找出调用堆栈？

需要重写`setLayoutParams()`方法，然后筛选传递进来的错误的参数，然后在重写了的方法中加个断点，然后开启debug，就能找出调用处了。

对于别的方法，例如`setVisibility()`。都采用类似的方式，就可以马上排查出错误的地方的堆栈从而解决问题。



### 通过view tree来排查问题。

