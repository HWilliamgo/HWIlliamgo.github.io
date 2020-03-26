### 1. 配置JDK环境变量

因为要用到javah的命令，所以需要配置jdk的环境变量，配置成功后，在命令行输入`javah`

``` shell
C:\Users\Administrator
λ javah
用法:
  javah [options] <classes>
其中, [options] 包括:
  -o <file>                输出文件 (只能使用 -d 或 -o 之一)
  -d <dir>                 输出目录
  -v  -verbose             启用详细输出
  -h  --help  -?           输出此消息
  -version                 输出版本信息
  -jni                     生成 JNI 样式的标头文件 (默认值)
  -force                   始终写入输出文件
  -classpath <path>        从中加载类的路径
  -cp <path>               从中加载类的路径
  -bootclasspath <path>    从中加载引导类的路径
<classes> 是使用其全限定名称指定的
(例如, java.lang.Object)。
```

他会输出javah命令的用法



### 2. 用命令生成头文件

确保输入命令所在的目录下存在含有`native`方法的class**全路径文件**，即`.class`文件需要在一个如下所示的路径中：

``` shell
└─com
    └─hwilliam
        └─jnilearn
                JNIMethod.class
```

（注意，这里需要确保输入javah命令的目录下是包含类的完整路径的，不能直接只有一个.class文件，那样会报找不到类文件的错误）



输入pwd查看当前工作目录

``` shell
λ pwd
C:\AndroidProject\JNILearn\app\build\intermediates\javac\debug\classes
```



那么此时输入命令：

``` shell
$ javah com.hwilliam.jnilearn.JNIMethod
```

则在当前目录生成了文件：

`com_hwilliam_jnilearn_JNIMethod.h`

