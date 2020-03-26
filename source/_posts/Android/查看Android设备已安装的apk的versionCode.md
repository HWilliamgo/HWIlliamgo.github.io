# 查看Android设备已安装的apk的versionCode

## 1. 确认连接

```shell
$ adb devices
```

## 2. 输出设备所有已安装的apk

```shell
$ adb shell list packages
```

## 3. 找到目标apk名字，输出对应信息，找到versionCode即可。

```shell
$ adb shell dumpsys package com.aaa.bbb
```

