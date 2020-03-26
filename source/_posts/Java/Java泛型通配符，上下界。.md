> 为了理清楚泛型的通配符和上下界的作用，并为了Kotlin的泛型中的关键字`in`和`out`的理解，在此用小demo重新梳理一遍对泛型的理解。

### demo

```java
public class Example {
    //程序入口
    public static void main(String[] args) {
        Source<? extends Number> sourceOfNumber = new Source<>();
        Number params = sourceOfNumber.getParams();//get()方法，通过编译
        sourceOfNumber.setParams(1);//①set()方法，不通过编译

        Source<? super Integer> sourceOfParentOfInt = new Source<>();
        sourceOfParentOfInt.setParams(new Integer(1));//set()方法，通过编译
        Number i = sourceOfParentOfInt.getParams();//②get()方法，不通过编译
    }
	
    //包装对象
    private static class Source<T> {
        private T params;
        
        public T getParams() {
            return params;
        }

        public void setParams(T params) {
            this.params = params;
        }
    }
}
```

以下来自《疯狂Kotlin讲义》并结合上述代码的理解。

### 通配符下界`Source<? super Integer>`

在这里，`T`就是`? super Integer`。由于`Source<T>`中的泛型一定是Integer的父类，因此程序总是可以向`Source`对象传入`Integer`值。但从`Source<T>`中取出对象是不安全的，因为无法预测到取出的是`Number`对象还是`Object`对象，即无法判断demo中的`sourceOfParentOfInt`引用的对象实际是`Source<Number>`还是`Source<Object>`。

因此往对象中设置值总是安全的，取出值总是不安全的。

### 通配符上界`Source<? extends Integer>`

在这里，`T`就是`? extends Number`。由于`Source<T>`中的泛型一定是Number的子类，因此程序从`Source`对象中取出的`T`一定是`Number`的子类，即可以用`Number`来引用。但是向其中传入对象是不安全的，因为你不能确定demo中的`sourceOfNumber`引用的到底是`Source<Float>`还是`Source<Integer>`，如果是引用了`Source<Float>`对象，而你往里面传了一个`Integer`，那么程序就出错了。

因此从对象中取出值是安全的，但是往对象中设置值是不安全的。

