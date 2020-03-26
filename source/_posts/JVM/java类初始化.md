
虚拟机规范严格规定了**有且只有**5种情况必须立即对类进行初始化（而加载、验证、准备自然需要在此之前开始）：
1. 使用new关键字实例化对象的时候、读取或设置一个类的静态字段（被final修饰、已在编译期把结果放入常量池的静态字段除外）的时候，以及调用一个类的静态方法的时候。
2. 使用java.lang.reflect包的方法对类进行反射调用的时候，如果类没有进行过初始化，则需要先出发其初始化。
3. 当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化。
4. 当虚拟机启动时，用户需要指定一个要执行的主类（包含main()方法的那个类)，虚拟机会先初始化这个主类。
5. 当使用JDK1.7的动态语言支持时，如果一个java.lang.invoke.MethodHandle实力最后的解析结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化。

哪些情况下不会对类进行初始化：
1. 通过子类引用父类的静态字段，不会导致子类初始化。
``` java
public class Main {
    public static void main(String[] args) throws Throwable {
        System.out.println(SubClass.value);
    }
}
class SuperClass{
    static {
        System.out.println("SuperClass init!");
    }
    public static int value=123;
}
class SubClass extends SuperClass{
    static {
        System.out.println("SubClass init");
    }
}
//print:
//SuperClass init!
//123
```
2. 通过数组定义来引用类，不会出发此类的初始化。
通过数组来引用的类，不会初始化此类，但是会初始化另一个由虚拟机自动生成的，继承于Object类的子类，数组对象中的属性和方法都实现在这个类里面。
``` java
public class Main {
    public static void main(String[] args) throws Throwable {
        SuperClass[] a=new SuperClass[10];
    }
}
//print nothing
```
3. 通过static final修饰的基本数据类型或String实例（或者称为常量）在编译阶段会存入调用类的常量池中，本质上并没有直接引用到定义常量的类，因此不会出发定义 常量的类的初始化。
``` java
public class Main {
    public static void main(String[] args) throws Throwable {
        System.out.println(ConstClass.HELLOWORLD);//print:hello world!
    }
}
class ConstClass{
    static {
        System.out.println("ConstClass init!");
    }
    public static final String HELLOWORLD="hello world!";
}
```
字符串HELLOWORLD在编译阶段存入了调用类Main的常量池中，因此对字符串的引用被转化为Main类对自身常量池的引用。实际上Main的class文件中并没有ConstClass类的符号引用入口，两个类在编译成class文件之后就不存在任何联系了。
4. 接口的加载过程和类的加载过程有些不同。
	接口不能用静态语句，但编译器仍然会为接口生成“<clint()>”类构造器，用于初始化接口中所定义的成员变量（接口中的成员变量都是`static final`）。
	与类初始化的真正区别在于前面的初始化场景的第三种，当一个类在初始化时，要求其父类全部都已经初始化过了，但是接口初始化时，不要求其父类接口全部完成初始化，只有在真正使用到父类接口的时候（如引用接口中定义的常量）才会初始化
	
来自《深入理解java虚拟机》
