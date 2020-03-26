# Kotlin与Java的差异

### 1. 对象的初始化时的顺序

   在java中，`初始化代码块` -> `构造函数`

   在kotlin中，`主构造函数` -> `初始化代码块` -> `次构造函数`

   由于Kotlin中主构造函数在初始化代码块之前，因此初始化代码块可以使用主代码块传入的参数。

   在kotlin中，所有的次构造器都会先调用主构造器（若不存在，也会隐式调用），而初始化代码块会放在主构造函数之后执行，因此次构造函数执行之前，初始化代码块是执行完了的。

### 2. 访问控制符

   private：只能在类的内部或者文件内部被访问

   internal：internal成员可在类的内部、文件的内部、同一个模块被访问

   protected：可在类的内部或者、文件内部、子类中被访问

   public：在任意地方被访问

   与java的差异在于：

   1. Kotlin取消了java的默认的包访问权限，引入internal模块访问权限
   2. Kotlin取消了protected的包访问权限。
   3. Kotlin的默认访问控制符是public

   此处的模块的存在形式有如下几种：

   * 一个intelliJ IDEA模块

   * 一个Maven项目

   * 一个Gradle源集

   * 一次<kotlinc>的Ant任务执行所编译的一套文件

   

   注意，可以修改属性的setter()访问权限，但是无法修改属性的getter()访问权限，因为getter()访问权限是和属性的访问权限一致的

### 3. 继承：Kotlin与Java的设计相同，所有子类构造器必须调用父类构造器一次。

   当调用子类构造器来初始化子类对象时，父类构造器总会在子类构造器之前执行；不仅如此，在执行父类构造器时，系统会再次上溯执行其父类构造器......以此类推，创建任何Kotlin对象，最先执行的总是Any类的构造器。

### 4. 方法的重写遵循”两同两小一大“规则

   * 两同：方法名相同，形参列表相同
   * 两小：子类方法的返回值类型比父类方法的返回值类型更小或相等。子类方法声明抛出的异常类应比父类方法生命抛出的异常类更小或相等。
   * 一大：指的是子类方法的访问权限应比父类方法的访问权限更大或相等。

### 5. 嵌套类和内部类

   1. 内部类机制和Java的非静态内部类一致
   2. 嵌套类机制和Java的静态内部类一致，由于静态成员无法访问非静态成员，且Kotlin取消了static属性，因此Kotlin的嵌套类无法访问任何的外部类成员。
   3. Koltin中允许在接口中定义嵌套类，但不允许在接口中定义内部类。

### 6. 匿名内部类

   1. Kotlin 抛弃了匿名内部类，而用功能更强大的对象表达式
   2. 如果对象是一个函数式接口的实例，则可以使用Lambda表达式创建他（或者用带接口名前缀的lambda表达式创建他）。

### 7. 对象表达式

   ``` kotlin
   object :N个父类型{
       
   }
   ```

   1. 增强版的匿名内部类，匿名内部类只能指定一个父类型，而对象表达式可以指定N个父类型。
   2. 对象表达式不能是抽象类，因为系统在创建对象表达式的时候会立刻创建对象。
   3. 对象表达式不能定义构造器，但是可以定义初始化块。
   4. 对象表达式可以包含内部类，但是不能包含嵌套类。
   5. Kotlin编译器能更准确地识别局部范围内或者private修饰的对象表达式的真实类型(既可以直接调用对象表达式中新定义的方法和属性)。
   6. 对象表达式可访问或者修改其所在范围内的局部变量。(java只能访问final变量，且不能修改)

### 8. 对象声明(java的静态块单例模式)

   ``` kotlin
   objecct 名字 : N个接口{
       
   }
   ```

   对象声明和对象表达式的区别：

   1. 对象表达式是表达式，因此可以赋值给变量并被引用。而对象声明不是表达式，不能用于赋值。
   2. 对象声明可包含嵌套类，不可包含内部类。对象表达式可包含内部类，不能包含嵌套类。
   3. 对象声明不能定义在函数和方法内，可以定义在类中。对象表达式可以嵌套在其他对象声明或者非内部类中。 

   对象声明专门用于实现单例模式，对象声明所定义的对象就是该类的唯一实例，直接用对象声明的名称即可访问该唯一实例。

   通过IntelliJ的tools->Kotlin->show Kotlin bytecode，再点击Decompile，可以看见Java代码的实现是：用懒加载的单例模式实现的。单例对象的初始化放在了静态块中。

### 9. 伴生对象
   1. 若在类中的对象声明加一个companion，该对象就变成了伴生对象。
   2. 每个类最多只能定义一个伴生对象。
   3. 伴生对象的名字并不重要，因此可以省略。
   4. 伴生对象的作用就是为他所在的类模拟静态成员。
   5. 拓展伴生对象就是为该伴生对象的外部类拓展静态方法和变量。

   ``` kotlin
   fun main() {
       println(MyClass.name)
       MyClass.output("你好")
       MyClass.test()
       MyClass().test()
       println(MyClass.foo)
   }
   
   interface Outputable {
       fun output(msg: String)
   }
   
   class MyClass {
       companion object : Outputable {
           //相当于MyClass的静态成员
           val name = "name 属性值"
           //相当于MyClass的静态方法
           override fun output(msg: String) {
               println(msg)
           }
       }
   }
   //为MyClass拓展实例方法
   fun MyClass.Companion.test() {
       println("为伴生对象扩展的方法")
   }
   //为MyClass拓展静态方法
   fun MyClass.test() {
       println("为MyClass类扩展的方法")
   }
   //为MyClass拓展静态变量
   val MyClass.Companion.foo
       get() = "为伴生对象拓展的属性"
   ```

### 10. 类委托。就和java的代理模式的作用是一样的。

### 11. 属性委托，可以直接实现Kotlin提供的两个接口`ReadWriteProperty`和`ReadOnlyProperty`来分别实现var和val属性的委托。

### 12. Kotlin泛型

#### 声明处型变

泛型中的`in`和`out`，用于取代java中有设计缺陷的`?`通配符和上下界的使用（具体设计缺陷在Kotlin文档的泛型这一部分有讲到）

`out`	由`out`声明的泛型变量，只能作为方法的返回值发布到外部，不能作为方法的参数传递到方法内部。好处是：可以将带有该泛型的变量赋值给父类型泛型的变量。如

``` kotlin
interface Source<out T> {
    fun nextT(): T
}

fun demo(strs: Source<String>) {
    val newSource: Source<Any> = strs
}
```

`in`		由`in`声明的泛型变量，只能作为参数被设置到内部，而不能发布到外部，因此，带有该泛型的子类型的变量都可以被作为参数传递进来。如

``` kotlin
interface ClassA<in T> {
    fun setT(value: T)
}

fun demo(value: ClassA<MutableList<Int>>) {
    value.setT(ArrayList())
	//也可以引用给更小类型的泛型变量，因为这里的y此后调用setT()传进去的参数ArrayList比MutableList类型更小
    val y: ClassA<ArrayList<Int>> = value
}
```

总结：out泛型的类可以作为方法中的参数，可以赋值给父类型的泛型引用。in泛型的类只能赋值给子类型泛型的变量，不可以作为方法中的参数。

#### 使用处型变





​    
