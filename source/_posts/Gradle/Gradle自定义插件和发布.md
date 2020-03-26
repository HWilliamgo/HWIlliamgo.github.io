# Gradle自定义插件和发布

这篇文章讲解的是如何自定义gradle插件，并以本地依赖和远程依赖的方式来集成。



本文大体结构和内容基于：gradle官网的教程：[开发自定义gradle插件](https://docs.gradle.org/current/userguide/custom_plugins.html#sec:working_with_files_in_custom_tasks_and_plugins)



约定俗成的说法：

1. 插件消费者项目：使用对应插件的项目。
2. 开发插件的项目：独立的，用来开发gradle插件的项目。



开始：

自定义gradle插件的三种形式：

1. 直接在项目中写一个插件，并由build.gradle直接应用来使用。
2. 在独立的项目中开发插件，并以本地依赖的形式集成。
3. 在独立的项目中开发插件，并以远程依赖的形式集成。



### 1 直接在项目中写插件并应用

### 2 在独立的项目中开发一个插件，本地依赖

#### 2.1 新建一个独立的java类项目

用idea或者android studio新建一个module或者新建一个project都可以，只要是能够轻易打出jar包的项目即可。



##### 2.1.1 build.gradle中引入gradle api的依赖

首先，自定义插件类的编写，需要实现`org.gradle.api.Plugin`接口，这个不是jdk的类，而是gradle提供的接口，因此必须对gradle的api做一个依赖。

那么在项目的`build.gradle`中写入：

``` groovy
plugins {
    id 'groovy'
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])

    //我们需要实现gradle的Plugin<T>接口，来做自定义插件，因此依赖gradle api
    implementation gradleApi()
    //依赖gradle提供的groovy sdk，在编写自定义插件的时候，用groovy更快。
    implementation localGroovy()
}

sourceCompatibility = "7"
targetCompatibility = "7"

```

我的例子并没有用到groovy，所以去掉上面的`localGroovy`，并将`groovy`插件换成`java`插件也可以。



##### 2.1.2 创建并配置.properties文件。

其次，我们写出来的插件最终作为jar包被插件消费者项目引用，那么插件消费者项目要如何在jar包中找到我们`org.gradle.Plugin`接口的实现类呢？以及我们在哪里定义我们的插件的id呢？

那么，需要在路径：`src/main/resources/META-INF/gradle-plugins/`下，创建一个`aa.bb.cc.properties`名字的文件，里面容为：

```properties
implementation-class=你的Plugin<T>接口实现类的全路径名，例如：aa.bb.cc.MyPlugin
```

那么此时，消费者项目能够找到插件接口实现类，并且，`properties`文件的名字就作为你的插件id。



##### 2.1.3 写一个简单的插件，生成一个task

``` java
package com.william.customplugin;

import org.gradle.api.Action;
import org.gradle.api.Plugin;
import org.gradle.api.Project;
import org.gradle.api.Task;

/**
 * date: 2019/9/6 0006
 *
 * @author hwj
 * description 插件
 */
public class GreetingPlugin implements Plugin<Project> {
    @Override
    public void apply(Project project) {
        project.task("hello", new Action<Task>() {
            @Override
            public void execute(Task task) {
                task.doLast(new Action<Task>() {
                    @Override
                    public void execute(Task task) {
                        System.out.println("task hello---doLast");
                    }
                });
            }
        });
    }
}
```



以上，所有的代码编写工作完成了。

贴出此时项目的结构：

``` shell
/customPlugin

│  .gitignore
│  build.gradle
│
└─src
    └─main
        ├─java
        │  └─com
        │      └─william
        │          └─customplugin
        │                  GreetingPlugin.java
        │
        └─resources
            └─META-INF
                └─gradle-plugins
                        com.william.customplugin.properties
```





#### 2.2  构建项目的产物：jar包

注意，在这一步，有两种实现方案：

1. 直接用build类型的gradle脚本打出一个jar包，然后将jar包复制粘贴到插件消费者项目。
2. 用gradle的`maven`或者`maven-publish`插件，来将打包的jar包发布到指定的本地路径中。



我们先看第一种方式：

##### 2.2.1 用build类型的gradle脚本打包

执行

``` shell
/customPlugin
$ gradlew build
```

构建产物在：`./build/libs/customPlugin.jar`

此时拿到了jar包。其中包含了我们写的`org.gradle.Plugin`接口的实现类。



##### 2.2.2 用maven类型的脚本打包并发布在指定的本地路径

打开插件开发项目，我们需要发布一个maven类型的软件包。gradle为maven类型的软件包发布提供了两种插件：`maven`和`maven-publish`，前者已经被废弃，现在最新的是后者，我们这里用后者插件来实现构件一个maven类型的软件包。

我们参考的是[`maven-publish`文档](https://docs.gradle.org/current/userguide/publishing_overview.html)的最简单的发版配置：

``` groovy
group = 'org.example'
version = '1.0'

publishing {
    publications {
        myLibrary(MavenPublication) {
            from components.java
        }
    }

    repositories {
        maven {
            name = 'myRepo'
            url = "file://${buildDir}/repo"
        }
    }
}
```

将上述的配置移植到我们的插件开发项目的build.gradle中，如下：

``` groovy
plugins {
    id 'groovy'
    id 'maven-publish'
}

group = "com.william.customplugin"
version = "1.0"

publishing {
    publications {
        myLibrary(MavenPublication) {
            from components.java
        }
    }

    //仓库配置
    repositories {
        maven {
            //name这个属性是用来指定仓库名字的，貌似在这里没什么用，注释掉。
            //name = 'myRepo'
            
            //指定发布仓库的路径
            url = "./build/repo"
        }
    }
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])

    implementation gradleApi()
    implementation localGroovy()
}

sourceCompatibility = "7"
targetCompatibility = "7"

```

接下来就是执行一个task，来构建并发布了。

`maven-publish`脚本带来了task `publish`，他会执行所有的发布任务。而我们这里只有一个发布任务，所以执行他就好了。

``` shell
gradlew publish

===>

14:45:49: Executing task 'publish'...

Executing tasks: [publish] in project C:\AndroidProject\KotlinSimpleTest\customPlugin

> Task :customPlugin:generatePomFileForMyLibraryPublication
> Task :customPlugin:compileJava UP-TO-DATE
> Task :customPlugin:compileGroovy NO-SOURCE
> Task :customPlugin:processResources UP-TO-DATE
> Task :customPlugin:classes UP-TO-DATE
> Task :customPlugin:jar UP-TO-DATE
> Task :customPlugin:publishMyLibraryPublicationToMavenRepository
> Task :customPlugin:publish

BUILD SUCCESSFUL in 4s
5 actionable tasks: 2 executed, 3 up-to-date
14:45:53: Task execution finished 'publish'.

```

OK，现在跑去`./build/repo`下面找我们的软件包吧。

``` 
├─repo
│  └─com
│      └─william
│          └─customplugin
│              └─customPlugin
│                  │  maven-metadata.xml
│                  │  maven-metadata.xml.md5
│                  │  maven-metadata.xml.sha1
│                  │
│                  └─1.0
│                          customPlugin-1.0.jar
│                          customPlugin-1.0.jar.md5
│                          customPlugin-1.0.jar.sha1
│                          customPlugin-1.0.pom
│                          customPlugin-1.0.pom.md5
│                          customPlugin-1.0.pom.sha1
```

输出的整个包多了很多东西~，我们用普通的`build`命令打出来的jar包，只有一个单独的jar包，而用`maven-publish`插件打出来的包，囊括了完整的全路径名，带有pom文件用于描述依赖，带有maven元数据等等，这是一个完整的、可用于分发的软件了~



#### 2.3 在插件消费者项目中使用插件

##### 2.3.1 直接使用本地的jar包中的插件。

拿到jar包后，把他放在一个可以被找到的路径，我把他放在了插件消费者项目的根目录的`gradle_plugin_libs/`目录下。

``` 
└─gradle_plugin_libs
        customPlugin.jar
```

一般来说，安卓项目都采用的是gradle的multi-project-build组织类型，所以我打算在根目录下让gradle对我的脚本进行一个依赖，以便子项目不需要再自己去声明对脚本的依赖。

即`rootProject/build.gradle`下：

``` groovy
buildscript{
    repositories{
        //...
    }
    dependencies{
		classpath 'com.android.tools.build:gradle:3.4.1'
        //...
        
        //注意，之类不能用classpath，因为classpath指定的是以标准的maven格式构建的软件包。
        //我们这里是直接用jar的，classpath识别不了，会报错。因此用classpath files()
        classpath files ('./gradle_plugin_libs/customPlugin.jar')
    }
}
//..
```

随意地找一个项目的构建脚本，写上：`apply plugin: 'com.william.customplugin'`。（注意，插件Id是properties文件的名字）

执行命令：`gradlew hello`-->  输出： task hello---doLast

大功告成~



##### 2.3.2 使用本地maven仓库下的jar包中的插件

将2.2.2节构建出的软件包整个复制到`gradle_plugin_libs/`中。

修改根目录构建脚本：`rootProject/build.gradle`

``` groovy
buildscript{
    repositories{
        //...

        //指定一个maven仓库的路径
        maven {
    		url uri('./gradle_plugin_libs')
		}
    }
    dependencies{
        //...
        
        //用标准的classpath方法，来找到我们的插件
        classpath('com.william.customplugin:customPlugin:1.0')
    }
}
//..
```

执行`gradlew hello`，输出和上一节一样。

大功告成~



#### 2.4 如何测试插件？

待补充



### 3 在独立的项目中开发一个插件，远程依赖

上面我们对插件进行了本地依赖的方式来使用，现在我们将我们的插件打出的软件包上传到远程仓库上，让别的开发者也能快速地集成。

#### 3.1 新建一个独立的java类项目

我们就用上述的那个插件开发项目，不用再新建了。

#### 3.2 将构建的jar包上传到一个远程仓库

远程仓库有两个类型：

1. 大家所熟知的中央仓库：例如`mavenCentral`，`jcenter`，`jitpack`
2. 普通的远程仓库



他们本质上都是一样的，只是那3个中央仓库是用的人最多的。

我们现在通过`bintray`这个软件包发行平台，来上传并管理我们的软件包。

`bintray`用于上传maven类型的软件包有3种方式，我们的其中一种是通过gradle的方式：https://github.com/bintray/gradle-bintray-plugin。这是bintray官方开发的上传插件，专门用于打包和上传maven类型的软件包到bintray上面，选项多，可自定义程度高，使用复杂。



在github上有一个快捷方便的第三方开发者开发的插件：https://github.com/novoda/bintray-release

只需要一个代码块就能完成bintray上传的配置，这里我们用这个简单的插件来做演示，当然大家可以根据自己业务的需求选择负责度和功能性更高的官方插件。



##### 3.2.1 创建bintray账号

在这一步，要完成的事项如下：

1. 创建bintray账号
2. 创建一个要上传该软件包的仓库
3. 在上述仓库下创建一个package，即创建一个软件包，软件包需要和你后面上传的软件包名字相同。（我很纳闷为什么一定要创建了一个包才能上传，不能上传的时候没有包bintray就自己创建吗，真是奇怪，我在这一步停滞了很久...）

##### 3.2.2 配置bintray上传脚本

``` groovy
plugins {
    id 'groovy'
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])

    implementation gradleApi()
    implementation localGroovy()
}

sourceCompatibility = "7"
targetCompatibility = "7"

//以上是默认的内容，以下是上传脚本的配置，可以说是非常简洁了。

//应用插件
apply plugin: 'com.novoda.bintray-release'

//上传配置
publish {
    userOrg = "huangwilliam33333"//组织，如果没有创建组织，就直接填写用户名。
    groupId = 'com.william'//group
    artifactId = 'customPlugin'//module
    publishVersion = '1.0'//版本
    desc = '自定义测试用的插件'//描述
    website = 'htttp://www.baidu.com'//网站，随便填
    bintrayUser = "huangwilliam33333"//用户名
    bintrayKey = "xxx"//bintray秘钥
    dryRun = false//如果这个是true，则只是运行，而不会上传，要配置成false
}
```



执行`gradlew clean build bintrayUpload`

上传成功。



#### 3.3 在另一个项目中依赖这个远程仓库，并使用该插件

到插件消费者项目中：

我们还是在根目录应用插件：

```groovy
buildscript {
    ext.kotlin_version = '1.3.41'
    repositories {
        google()
        jcenter()
        
        //因为还没有将软件包加入到Jcenter，所以要指明maven地址
        maven{
            url  "https://dl.bintray.com/huangwilliam33333/maven"
        }
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.0'
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
        classpath 'com.novoda:bintray-release:0.9.1'
			
		//依赖插件
        classpath('com.william:customPlugin:1.0')
    }
}
```

插件的使用和前面一样。



#### 3.4 如何测试插件

待补充



参考资料：

[bintray第三方简易上传插件](https://github.com/novoda/bintray-release)

[bintray官方上传插件](https://github.com/bintray/gradle-bintray-plugin)

[讲到了bintray创建了仓库后还要创建软件包才能上传](https://www.jianshu.com/p/9f81d5b5a451)

https://stackoverflow.com/questions/35302414/adding-local-plugin-to-a-gradle-project

[gradle官方文档：publishing artifact](https://docs.gradle.org/current/userguide/publishing_overview.html)

[gradle官方文档：maven publish plugin](https://docs.gradle.org/current/userguide/publishing_maven.html#publishing_maven)
