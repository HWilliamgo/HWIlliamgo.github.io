> 基于gradle 4.6

本文记录一些常用的gradle工具脚本的写法，基于gradle4.6，不同版本的gradle应对照[官方文档不同版本]( https://gradle.org/releases/ )的DSL Reference参考对应的api的接口的使用。

#### 1. clean整个项目

有的时候有一些操作需要对root project和其下面所有的child project进行一个clean，那么可以参考如下的脚本实现：

``` groovy
//该task定义在root Project下。
task cleanAllProject(){
    it.dependsOn("clean")
    rootProject.subprojects{sub->
        sub.afterEvaluate {
    		def cleanTask = it.tasks.findByName('clean')
    		if (cleanTask != null) {
        		dependsOn(cleanTask)
    		}
		}
    }
    it.finalizedBy("<task that follow this full clean>")
}
```



#### 2. copy某个目录，或文件

copy某个是比较简单的，这里例举copy某个目录，并排除掉指定的子目录。

``` groovy
//copy当前整个项目到buid目录下
task copyDir(){
    doLast{
        copy{
            from rootProject.projectDir
            //destination路径是build/project，用$buildDir指定build路径
            into "$buildDir/project"
            
            //exclude排除指定的目录：/.gradle、 /.idea 、/.git
            exclude "**/.gradle/**"
            exclude "**/.idea/**"
            exclude "**/.git/**"
        }
    }
    it.finalizedBy("<task that follow this full clean>")
}
```

注意，根据文档，`exclude( )`方法的参数是一个` ANT style exclude pattern `，这是一种有语法的表达式。

thanks: 

[StackOverFlow : Learning Ant path style](https://stackoverflow.com/questions/2952196/learning-ant-path-style)

[StackOverFlow : Cannot exclude directories for a Gradle copy task](https://stackoverflow.com/questions/52768723/cannot-exclude-directories-for-a-gradle-copy-task)



#### 3. zip，生成压缩文件

实现这个操作我算是踩了一个小坑了。

我先去官网user mannel home的输入框搜索zip，并选择了

Creating archives(zip, tar, etc.)这一节内容：

 https://docs.gradle.org/current/userguide/working_with_files.html#sec:creating_archives_example 

他的示例代码是：

``` groovy
task packageDistribution(type: Zip) {
    archiveFileName = "my-distribution.zip"
    destinationDirectory = file("$buildDir/dist")

    from "$buildDir/toArchive"
}
```

我拿去用的时候报错了，说是找不到属性`archiveFileName`，然后是我本地的gradle版本落后于官网的版本了。我本地的版本是4.6，而官网的教程是基于5.6.4的。于是我去查找了4.6的教程和DSL说明，4.6的版本的使用方法是：

``` groovy
task makeZip(type: Zip) {
    from "/要压缩的目录"
    archiveName = "压缩包名字.zip"
    //指定生成压缩包的路径，要用file类型
    File destinationFile = new File("$buildDir/zipProduction")
    destinationDir = destinationFile
}
```

#### 

#### 4. 更新中。。。

