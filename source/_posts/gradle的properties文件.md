---
title: gradle的properties文件
date: 2020-05-11 11:32:46
tags:
index_img:
banner_img:
---



我们知道在gradle中可以用 `gradle.properties`文件来设置属性。一般.properties文件的位置是定义在和`build.gradle`和`settings.gradle`同级的目录下。



### 1 简单使用

在`gradle.properties`中定义一个属性是这样的：

``` groovy
PRO_A=12
```

然后在构建脚本里就可以用到这个属性了：

``` groovy
task A(){
    doLast{
        println(PRO_A)
    }
}
```



### 2 和ext属性的关系

首先确定的是，`gradle.properties`中定义的属性是属定义他的gradle项目的`project`对象的属性的。

其次，在gradle文档中，没有说明（或者是我没找到）`gradle.properties`中的属性在是保存在`Project`对象的哪个位置的。

我个人猜测有两种可能：

1. 通过`project.ext`属性，保存在这里面。
2. 直接通过扩展`project`属性的方式，给`project`拓展属性。

这里倾向于前者，因为第二种给`project`拓展属性的编程方式，通常用来写gradle plugin。



#### 通过测试验证猜想：

测试方法：定义一个属性，然后打印当前项目的ext属性中的所有的属性。从打印结果中就能看到`gradle.preperties`文件中的属性是否在ext中了。



定义属性

`gradle.properties`文件：

``` groovy
PRO_A=I AM PROPERTY A !
```

`build.gradle`文件中写一个task测试

```groovy
task A() {
    doLast {
        println(PRO_A)
        rootProject.extensions.extraProperties.getProperties().forEach { String key, Object value ->
            println("key=$key,value=$value")
        }
    }
}
```



命令行执行：

```groovy
▶ gradle A

> Task :sub:A
I AM PROPERTY A !
key=PRO_A,value=I AM PROPERTY A !
key=kotlin.code.style,value=official

```

将`Project`的ext属性输出的结果中包含了此前定义的属性，说明`gradle.properties`中的属性会保存在ext变量中。



### 3 Project对象的属性的读取顺序和优先级





分割线中内容全部来自[gradle文档：Proejct](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html#org.gradle.api.Project.properties)

---



A project has 5 property 'scopes', which it searches for properties. You can access these properties by name in your build file, or by calling the project's [`Project.property(java.lang.String)`](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html#org.gradle.api.Project:property(java.lang.String)) method. The scopes are:

- The `Project` object itself. This scope includes any property getters and setters declared by the `Project` implementation class. For example, [`Project.getRootProject()`](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html#org.gradle.api.Project:rootProject) is accessible as the `rootProject` property. The properties of this scope are readable or writable depending on the presence of the corresponding getter or setter method.
- The *extra* properties of the project. Each project maintains a map of extra properties, which can contain any arbitrary name -> value pair. Once defined, the properties of this scope are readable and writable. See [extra properties](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html#org.gradle.api.Project.extraproperties) for more details.
- The *extensions* added to the project by the plugins. Each extension is available as a read-only property with the same name as the extension.
- The *convention* properties added to the project by the plugins. A plugin can add properties and methods to a project through the project's [`Convention`](https://docs.gradle.org/current/javadoc/org/gradle/api/plugins/Convention.html) object. The properties of this scope may be readable or writable, depending on the convention objects.
- The tasks of the project. A task is accessible by using its name as a property name. The properties of this scope are read-only. For example, a task called `compile` is accessible as the `compile` property.
- The extra properties and convention properties are inherited from the project's parent, recursively up to the root project. The properties of this scope are read-only.

When reading a property, the project searches the above scopes in order, and returns the value from the first scope it finds the property in. If not found, an exception is thrown. See [`Project.property(java.lang.String)`](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html#org.gradle.api.Project:property(java.lang.String)) for more details.

When writing a property, the project searches the above scopes in order, and sets the property in the first scope it finds the property in. If not found, an exception is thrown. See [`Project.setProperty(java.lang.String, java.lang.Object)`](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html#org.gradle.api.Project:setProperty(java.lang.String, java.lang.Object)) for more details.

---



### 4 总结

1. `gradle.properties`文件中定义的属性在gradle脚本执行时保存在`project.ext`属性中。
2. gradle脚本读写属性是有顺序的，他会从5个区域里读写，先读到哪个就先用哪个，读取的优先级为：
   1. project本身的属性。
   2. project的extra（即project.ext）属性中的map中保存的键值对。
   3. 被gradle plugin添加的属性，即project.extensions属性。
   4. 被gradle plugin添加的convention属性。（没用过也没了解过这个属性）
   5. project中的task的名字，也作为project的属性。
   6. project从他的父project继承下来的ext和convention属性，会一直递归到rootProject。

