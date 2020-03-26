> Kotlin的泛型比Java更加强大，保障虚拟机运行安全的基础上，功能更加完善。我看了几次泛型都没有记住，回过头来重新梳理总结一番。



### 定义父类和子类

``` kot
package william

interface Parent {

}

interface Parent2 {

}

open class Person : Parent, Parent2 {

}

class Child : Person() {

}

class Child2 : Person() {

}
```

### 用Java演示

``` java
//list1中的元素是Person的子类，但是具体是哪一种不好说。
List<? extends Person> list1 = new ArrayList<>();
//由于不知道具体是Person的哪一个实现类，因此不能往里面添加，只能取出来，起码能确定取出来的一定是Person类型的。
Person number = list1.get(0);

//list2中的元素都是Person的父类。
List<? super Person> list2 = new ArrayList<>();
//由于不知道具体是哪一个父类，因此取出来无法指定具体的引用类型，无法取出来。
//但是却可以往里面添加Person的子类型，因为即使是子类型，也将会在List中被当做Person这个父类型来安全地被操作。
list2.add(new Person());
list2.add(new Child());

/*
总结：
一个类用 ? extends T 指定泛型，那么这个泛型只能被输出。（取出来知道是T的子类）
一个类用 ? super T   指定泛型，那么这个泛型只能被输入。（可以放T的子类进去）
 */
```



### 用Kotlin演示

用Kotlin完成上面Java做的同样的事情。

``` kotlin
val outList: ArrayList<out Person> = ArrayList()
val person: Person = outList.get(0)
val inList: ArrayList<in Person> = ArrayList()
inList.add(Child())
inList.add(Child2())

/*************Java做不到，而Kotlin可以做到的事情********************/

//Person的子类一定更是Parent的子类。用更加抽象的Parent引用，没有任何问题
//输出的类型越来越抽象，安全。
val parentOutList: ArrayList<out Parent> = outList
//Person的父类一定更加是Child的父类。输入更加具体的Child类，没有任何问题
//输入的类型越来越具体，安全。
val childInList: ArrayList<in Child> = inList
```

