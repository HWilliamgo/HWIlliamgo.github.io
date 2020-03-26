该方法与JVM的运行时常量池有关，运行时常量池是JVM方法区的一部分。

1.  `new String()`和`new String(“”)`都是申明一个新的空字符串，是空串不是null。
2. `String a="你好";`在编译期确定字符串内容，放入常量池。
3. `String a=new String("你好")`和`String a="你"+new String("好")`是到了程序运行期，调用了String的构造函数了才能确定字符串内容的，不能放入常量池。
4. 
```
String a=new String("abc");
String b=a.intern();
//引用a依然指向堆内存中的对象。
//但是返回的对象赋值给b，那么引用b指向了新的对象：常量池中唯一的代表"abc"的对象。
System.out.println(a==b);//false
```
5. jdk 7之后讲常量池从方法区移动到了堆。常量池中不需要再存储一份新的对象了，可以直接存储堆中的引用。
