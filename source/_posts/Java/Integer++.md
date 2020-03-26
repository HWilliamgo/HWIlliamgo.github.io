# Integer++
> 在写一个线程同步的demo的时候，想用Integer作为对象传入各个Runnable中作为flag，自增，然后发现失败，探究原因后记录如下：
```java
public class Main {
    public static void main(String[] args) {
        Integer i = 10;
        Integer j = i++;
        System.out.println(i == j);
    }
}
```

输出：`false`

原因：

Integer中表示值的变量`value`本身就是一个`final`类型的，即不可变类型，不可重新赋值。

```java
private final int value;
```

那么当`Integer++`时，只能去重新创建一个`Integer对象`，获取从缓存中`IntegerCache`这个数组缓存中取出现成的`Integer`对象。

当`Integer++`时，首先调用`intValue()`自动拆箱

```java
public int intValue() {
    return value;
}
```

返回的`value`自增后，用`valueOf(int i)`自动装箱，然后返回一个新的`Integer`对象。

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

当`i`的值在区间`[IntegerCache.low , IntegerCache.high]`时，返回`IntegerCache.cahce`这个静态数组中的元素。

```java
private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer cache[];//静态数组
    //静态代码块中直接初始化了静态数组。
    static {
        // high value may be configured by property
        int h = 127;
        String integerCacheHighPropValue =
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            try {
                int i = parseInt(integerCacheHighPropValue);
                i = Math.max(i, 127);
                // Maximum array size is Integer.MAX_VALUE
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
            } catch( NumberFormatException nfe) {
                // If the property cannot be parsed into an int, ignore it.
            }
        }
        high = h;
        cache = new Integer[(high - low) + 1];
        int j = low;
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);
        // range [-128, 127] must be interned (JLS7 5.1.7)
        assert IntegerCache.high >= 127;
    }
    private IntegerCache() {}
}
```

结论：在`int`取值为`[-128 , 127]`时，从静态数组中直接取出现有的`Integer`对象。不在这个区间时，直接new出`Integer`对象。
