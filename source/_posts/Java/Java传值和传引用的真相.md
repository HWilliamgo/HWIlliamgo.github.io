[JAVA 对象引用，以及对象赋值](http://zwmf.iteye.com/blog/1738574)

---
上述文章可以作参考，但是最后总结说的，包括《thingking in Java》说的：“`不管是基本类型还是对象类型，都是传值`”。
我认为这句话会引起歧义，真正的真相是：不管Java参数的类型是什么，一律传递参数的副本。
《thingking in Java》中：“`When you're passing primitives into a method,you get a distinct copy of the primitive.When you're passing a reference into a method,you get a copy of the reference`”(如果Java是传值，那么传递的是值得副本；如果Java是传引用，那么传递的是引用的副本。)

注意，String类型也是对象型的变量，所以也是传引用的副本。

`来自：《Java程序员面试宝典》`
