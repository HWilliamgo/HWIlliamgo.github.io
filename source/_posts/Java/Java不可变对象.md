1. 定义：
如果某个对象在被创建后，其状态（实例变量）不能被修改，那么这个对象就称为不可变对象。
2. 优点： 
    1. 不可变对象一定是线程安全的（不可变，别的线程修改不了）。
    2. 可作为Map的键值和Set的元素。
3. 编写不可变类：
    1. 所有字段都是private final。
    2. 没有任何public的方法来修改字段。
    3. 不依赖外部可变对象的引用。（如果依赖外部可变对象，那么当外部可变对象修改时，引用变量会指向新的对象，违反不可变对象的定义）
    4. 不能逸出任何可变对象的引用变量。（逸出引用变量后，外部可以通过引用变量修改对象）。
```
import java.util.Date;

public final class ImmutableObject {
    private final int age;

    private final String name;//不可变引用，且创建后指向不可变对象

    private final Date date;//不可变引用，可变对象

    public ImmutableObject(int age, String name, Date date) {
        this.age = age;
        this.name = name;

        //如果写成this.date=date;就依赖了外部对象。
        //因此要重新创建一个Date实例
        this.date = new Date(date.getTime());
    }

    public int getAge() {
        return age;
    }

    public String getName() {
        return name;
    }

    public Date getDate() {
        //如果写成return date;就逸出了当前对象的引用，调用者可以
        //通过该引用修改Date实例的属性
        return new Date(date.getTime());
    }
}
```

> thanks https://my.oschina.net/jackieyeah/blog/205198
