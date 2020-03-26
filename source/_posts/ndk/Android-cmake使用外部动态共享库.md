**官方文档分享：**

CMake的基本用法参考Android官方文档上的CMake教程：

https://developer.android.com/studio/projects/configure-cmake

他的内容有：

1. 创建CMake脚本
2. 使用NDK中的静态库和动态共享库
3. 添加其他已经编译过得动态共享库
4. 如何进行多CMake project的开发



---

本篇文章

本篇文章主要是分析如何使用第三方的so库进行c层的开发。

与官方文档不同的是，官方文档在连接第三方动态共享库的时候，用法是：

``` cmake
# 添加共享库imported-lib
add_library( imported-lib
             SHARED
             IMPORTED )
# 设置共享库的路径
set_target_properties( # Specifies the target library.
            imported-lib

            # Specifies the parameter you want to define.
            PROPERTIES IMPORTED_LOCATION

            # Provides the path to the library you want to import.
            imported-lib/src/${ANDROID_ABI}/libimported-lib.so )

# 设置头文件的路径
include_directories( imported-lib/include/ )

# 将第三方共享库库和你自己编译出来的共享库连接
target_link_libraries( native-lib imported-lib ${log-lib} )
```



 而我这里用的是：

```cmake
link_directories()
```



下面看下代码：

### 

### 1 项目目录

``` 
▶ tree -L 5
.
├── build.gradle
├── consumer-rules.pro
├── ffmpegprebuild.iml
├── libs
│   └── armeabi-v7a
│       ├── libavcodec.so
│       ├── libavfilter.so
│       ├── libavformat.so
│       ├── libavutil.so
│       ├── libswresample.so
│       └── libswscale.so
└── src
    ├── main
    │   ├── AndroidManifest.xml
    │   ├── cpp
    │   │   ├── CMakeLists.txt
    │   │   ├── ff.c
    │   │   └── include
    │   │       ├── libavcodec
    │   │       ├── libavfilter
    │   │       ├── libavformat
    │   │       ├── libavutil
    │   │       ├── libswresample
    │   │       └── libswscale
    │   ├── java
    │   │   └── com
    │   │       └── hwilliamgo
    │   └── res
    │       ├── drawable
    │       └── values
    │           └── strings.xml

```

我这里是使用我提前编译好的`armveabi-v7a`架构下的ffmpeg的动态共享库。并将头文件放在了`/cpp/include`目录下。

而`cpp/ff.c`作为源文件。

我的目标是：在ff.c中使用ffmpeg提供的api，提供到java层去调用。

`ff.c` ==>

``` c
#include <stdio.h>
#include <jni.h>
#include <malloc.h>
#include <android/log.h>
#include "libavformat/avformat.h"


JNIEXPORT jint JNICALL
Java_com_hwilliamgo_ffmpegprebuild_FFMpegUtils_getVersion(JNIEnv *env, jclass clazz) {
    return avformat_version();
}
```



### 2 如何编写CMakeLists.txt



``` cmake
cmake_minimum_required(VERSION 3.4.1)

# 查找目录下的所有源文件
# 并将名称保存到 DIR_SRCS 变量
aux_source_directory(. DIR_SRCS)

# 找到NDK提供的共享库log，并保存在变量log-lib中
find_library(
        log-lib
        log)

# 指定头文件的目录
include_directories(./include)
# 指定第三方动态共享库的目录
link_directories(../../.././libs/${ANDROID_ABI}/)
# 指定将${DIR_SRCS}目录下的源码文件编译成共享库，名字为：ff
add_library( 
        ff
        SHARED
        ${DIR_SRCS})
# 指定将将第三方共享库库和你自己编译出来的共享库连接
target_link_libraries(
        ff

        ${log-lib}
        avformat
        )
```



### 3 遇到的问题

这里要确保一件事情：

`link_directories()`指令要放在`add_library`指令之前。也就是设置第三方库的路径要放在前面。

否则会遇到找不到共享库的问题：

``` 
FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':ffmpegprebuild:externalNativeBuildDebug'.
> Build command failed.
  Error while executing process /Users/HWilliam/Library/Android/sdk/cmake/3.10.2.4988404/bin/ninja with arguments {-C /Users/HWilliam/AllProject/AndroidStudioProjects/windowsProject/JNILearnCMake/ffmpegprebuild/.cxx/cmake/debug/armeabi-v7a ff}
  ninja: Entering directory `/Users/HWilliam/AllProject/AndroidStudioProjects/windowsProject/JNILearnCMake/ffmpegprebuild/.cxx/cmake/debug/armeabi-v7a'
  [1/2] Building C object CMakeFiles/ff.dir/ff.c.o
  [2/2] Linking C shared library /Users/HWilliam/AllProject/AndroidStudioProjects/windowsProject/JNILearnCMake/ffmpegprebuild/build/intermediates/cmake/debug/obj/armeabi-v7a/libff.so
  FAILED: /Users/HWilliam/AllProject/AndroidStudioProjects/windowsProject/JNILearnCMake/ffmpegprebuild/build/intermediates/cmake/debug/obj/armeabi-v7a/libff.so 
  : && /Users/HWilliam/ndk/android-ndk-r20/toolchains/llvm/prebuilt/darwin-x86_64/bin/clang --target=armv7-none-linux-androideabi16 --gcc-toolchain=/Users/HWilliam/ndk/android-ndk-r20/toolchains/llvm/prebuilt/darwin-x86_64 --sysroot=/Users/HWilliam/ndk/android-ndk-r20/toolchains/llvm/prebuilt/darwin-x86_64/sysroot -fPIC -g -DANDROID -fdata-sections -ffunction-sections -funwind-tables -fstack-protector-strong -no-canonical-prefixes -fno-addrsig -march=armv7-a -mthumb -Wa,--noexecstack -Wformat -Werror=format-security  -O0 -fno-limit-debug-info  -Wl,--exclude-libs,libgcc.a -Wl,--exclude-libs,libatomic.a -static-libstdc++ -Wl,--build-id -Wl,--warn-shared-textrel -Wl,--fatal-warnings -Wl,--exclude-libs,libunwind.a -Wl,--no-undefined -Qunused-arguments -Wl,-z,noexecstack -shared -Wl,-soname,libff.so -o /Users/HWilliam/AllProject/AndroidStudioProjects/windowsProject/JNILearnCMake/ffmpegprebuild/build/intermediates/cmake/debug/obj/armeabi-v7a/libff.so CMakeFiles/ff.dir/ff.c.o  -llog -lavformat -latomic -lm && :
  /Users/HWilliam/ndk/android-ndk-r20/toolchains/llvm/prebuilt/darwin-x86_64/lib/gcc/arm-linux-androideabi/4.9.x/../../../../arm-linux-androideabi/bin/ld: error: cannot find -lavformat
  /Users/HWilliam/AllProject/AndroidStudioProjects/windowsProject/JNILearnCMake/ffmpegprebuild/src/main/cpp/ff.c:10: error: undefined reference to 'avformat_version'
  clang: error: linker command failed with exit code 1 (use -v to see invocation)
  ninja: build stopped: subcommand failed.
```

浓缩一下：`error: cannot find -lavformat`



还有就是，cmake官方不建议我们使用`link_directories()`命令：

https://cmake.org/cmake/help/v3.4/command/link_directories.html



该命令的文档：

**link_directories**

Specify directories in which the linker will look for libraries.

```
link_directories(directory1 directory2 ...)
```

Specify the paths in which the linker should search for libraries. The command will apply only to targets created after it is called. Relative paths given to this command are interpreted as relative to the current source directory, see [`CMP0015`](https://cmake.org/cmake/help/v3.4/policy/CMP0015.html#policy:CMP0015).

Note that this command is rarely necessary. Library locations returned by [`find_package()`](https://cmake.org/cmake/help/v3.4/command/find_package.html#command:find_package) and [`find_library()`](https://cmake.org/cmake/help/v3.4/command/find_library.html#command:find_library) are absolute paths. Pass these absolute library file paths directly to the [`target_link_libraries()`](https://cmake.org/cmake/help/v3.4/command/target_link_libraries.html#command:target_link_libraries) command. CMake will ensure the linker finds them.



并且android官方也是使用的`find_library()`命令的。所以最好还是用`find_library()`命令。
