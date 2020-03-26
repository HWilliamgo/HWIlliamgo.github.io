## 1. 下载
到官网下载各自对应的压缩包并解压。注意，如果提前用AndroidStudio下载了gradle，那么可以直接在`~/.gradle/wrapper/dists/`下找到已经下载了的gradle。
## 2.配置
命令行

1. 用gedit打开profile配置文件
``` shell
$ sudo gedit /etc/profile
```
2. 拉倒最底部，依次根据自己的下载路径加入如下配置
```
# set java environment
export JAVA_HOME=/home/william/下载/jdk-8u181-linux-x64/jdk1.8.0_181
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

# set gradle environment
export GRADLE_HOME=/home/william/.gradle/wrapper/dists/gradle-4.6-all/bcst21l2brirad8k2ben1letg/gradle-4.6
export PATH=${JAVA_HOME}/bin:${JRE_HOME}/bin:${GRADLE_HOME}/bin:${JAVA_HOME}:${PATH}

# set groovy environment
export GROOVY_HOME="/home/william/公共的/apache-groovy-sdk-2.5.3/groovy-2.5.3"
export PATH=$GROOVY_HOME/bin:$PATH

# set androd sdk environment
export ANDROID_SDK=/home/hwilliamgo/Android/Sdk
export PATH=$ANDROID_SDK/tools:$ANDROID_SDK/platform-tools:$PATH

# set android ndk environment
export ANDROID_SDK=/home/hwilliamgo/Android/Sdk
export PATH=$ANDROID_SDK/tools:$ANDROID_SDK/platform-tools:$PATH

```
3. 保存，查验
执行` source /etc/profile`使环境变量生效

检查jdk设置是否生效
``` shell
$ java -version
java version "1.8.0_181"
Java(TM) SE Runtime Environment (build 1.8.0_181-b13)
Java HotSpot(TM) 64-Bit Server VM (build 25.181-b13, mixed mode)
```

检查gradle设置是否生效
``` shell
$ gradle -version

------------------------------------------------------------
Gradle 4.6
------------------------------------------------------------

Build time:   2018-02-28 13:36:36 UTC
Revision:     8fa6ce7945b640e6168488e4417f9bb96e4ab46c

Groovy:       2.4.12
Ant:          Apache Ant(TM) version 1.9.9 compiled on February 2 2017
JVM:          1.8.0_181 (Oracle Corporation 25.181-b13)
OS:           Linux 4.15.0-38-generic amd64
```

检查groovy设置是否生效
``` shell
$ groovy -version
Groovy Version: 2.5.3 JVM: 1.8.0_181 Vendor: Oracle Corporation OS: Linux
```

检查android sdk设置是否生效
``` shell
$ android
*************************************************************************
The "android" command is deprecated.
For manual SDK, AVD, and project management, please use Android Studio.
For command-line tools, use tools/bin/sdkmanager and tools/bin/avdmanager
*************************************************************************
Invalid or unsupported command ""

Supported commands are:
android list target
android list avd
android list device
android create avd
android move avd
android delete avd
android list sdk
android update sdk
```

检查android ndk设置是否生效
``` shell
$ ndk-build -v
GNU Make 3.81
Copyright (C) 2006  Free Software Foundation, Inc.
This is free software; see the source for copying conditions.
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE.

This program built for x86_64-pc-linux-gnu
```
