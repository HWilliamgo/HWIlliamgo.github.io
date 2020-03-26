这篇总结的内容是我自己昨天我晚上写的一部分，因为我第二天在简书上面看到了一篇和我内容相似的，估计出处也是《疯狂java讲义》，所以剩下的内容在这里面看就可以了：

[zlcook的文章 : Java:Annotation(注解)--原理到案例](https://www.jianshu.com/p/28edf5352b63)


## java基本的5种Annotation
* @Overrided：只作用于方法，标识该方法是覆盖父类的方法
* @Deprecated: 作用于方法，类，接口。表示某个程序元素已过时
* @SpressWarnings：被该Annotation修饰的程序元素以及其中的所有子元素将取消显示指定的编译器警告。比如：@SuppressWarnings(value="unchecked")表示抑制集合没有使用泛型的警告。
* @SafeVarags：用于抑制“堆污染”（heap pollution）
* @FunctionalInterface： 只用于修饰接口。作用：指定被修饰的接口必须是函数式接口（接口中只有一个抽象方法，可以包含多个默认方法或者多个static方法）。
	* 比如：
	 ``` java
	 @FunctionalInterface
	 public interface FunInterface{
		 static void foo(){
		 System.out.println("foo类方法");
		 }
		 default void bar(){
			 System.out.println("bar默认方法")
		 }
		 void test();//只定义一个抽象方法
	 }
	 ```
	 如果该接口中有多个抽象方法，就会编译出错
	 
## java的6个元注解，Meta Annotation。（用于修饰注解的注解）
---
* @Retention : 用于指定被修饰的Annotation可以保留多长时间，@Retention包含一个`RetentionPolicy`类型的`成员变量`：`value`,所以使用@Retention时必须为该value指定值。有如下三个值：
	*  **RetentionPolicy.CLASS**:编译器将把Annotation记录在class文件中。当运行java程序时，JVM不可获取Annotation信息。这是默认值。
	*  **RetentionPolicy.RUNTIME**：编译器把Annotation记录在class文件中。当运行java程序时，JVM可以获取Annotation信息，程序可以通过反射获取到该Annotation的信息。
	*  **RetentionPolicy.SOURCE**：Annotation只保留在源代码中，编译器直接丢弃这种Annotation。
如果要用反射获取注解信息，就要将value属性设置为Retention.RUNTIME。
``` java
//定义下面的Testable注解保留到运行时，因此能够被反射
@Retention(value=RetentionPolicy.RUNTIME)
public @interface Testable{}
```
因为当注解的成员变量名为value时，程序可以直接在注解的括号里指定该成员变量的值，无须使用name=value形式，因此也可以：
``` java
//定义下面的Testable注解将被编译器直接丢弃，不会被加载到运行时，无法反射
@Retention(RetentionPolicy.SOURCE)
public @interface Testable{}
```
---

* @Target : 用于指定被修饰的Annotation能用于修饰哪些程序单元，@Target也包含一个名为value的成员变量，其值只能是如下几个： 
	* ElementType.ANNOTATION_TYPE:指定该策略的Annotation只能修饰Annotation
	* ElementType.CONSTRUCTOR:指定该策略的Annotation只能修饰构造器
	* ElementType.FIELD:只能修饰局部变量
	* ElementType.METHOD:只能修饰方法
	* ElementType.PACKAGE:只能修饰包
	* ElementType.PARAMETER:只能修饰参数
	* ElementType.TYPE可以修饰类、接口（包括注解类型）或枚举
