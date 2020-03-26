# ubuntu  下的android studio的真机调试

>  直接连真机一般都是连不上adb的。

### 1. 找出真机的硬件id

``` shell
# 列出当前链接的设备
$ lsusb
```
```
Bus 002 Device 002: ID 8087:8000 Intel Corp. 
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 002: ID 8087:8008 Intel Corp. 
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 003 Device 005: ID 8087:07dc Intel Corp. 
Bus 003 Device 004: ID 04f2:b439 Chicony Electronics Co., Ltd 
Bus 003 Device 002: ID 04b4:2009 Cypress Semiconductor Corp. 
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

将手机与电脑插上usb并链接。再次列出当前设备

``` shell
$ lsusb
# 此时肯定比之前会多出一个设备
```

```shell
Bus 002 Device 002: ID 8087:8000 Intel Corp. 
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 002: ID 8087:8008 Intel Corp. 
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 003 Device 005: ID 8087:07dc Intel Corp. 
#就是这个了。
Bus 003 Device 009: ID 0e8d:201c MediaTek Inc. 
Bus 003 Device 004: ID 04f2:b439 Chicony Electronics Co., Ltd 
Bus 003 Device 002: ID 04b4:2009 Cypress Semiconductor Corp. 
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

`Bus 003 Device 009: ID 0e8d:201c MediaTek Inc. `我的这台手机是魅族，这个`0e8d`就是这个硬件的id，记录下来。

### 2. 修改配置，重启服务。

在`/etc/udev/rules.d/`目录下用命令行新建一个`70-android.rules`文件

``` shell
$ sudo gedit 70-android.rules
```

此时用`gedit`编辑器打开了文本文件，内容填入

```
SUBSYSTEM=="usb", ATTR{idVendor}=="0e8d″, MODE="0666″
```

保存并关闭。

命令行执行以下命令重启服务。

``` shell
$ sudo /etc/init.d/udev restart
```

### 3. 链接adb

``` shell
$ adb kill-server
# XXX
$ adb start-server
# XXX
$ adb devices

List of devices attached
A02AACPPGN43E	device
```

---

END

thanks:https://blog.csdn.net/u010214802/article/details/54603457
