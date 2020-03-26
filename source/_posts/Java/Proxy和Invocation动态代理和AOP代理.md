Proxy创建动态代理的两个方法：
```
//方法一
static Class<?> getProxyClass(ClassLoader loader, Class<?>...interfaces);
//方法二
static Object newProxyInstance(ClassLoader loader, Class<?> interfaces, InvocationHander hander);
```
方法一：创建一个动态代理类所对应的Class对象，该代理类将实现interfaces数组所指定的多个接口。第一个ClassLoader参数指定生成动态代理类的类加载器。
方法二：直接创建一个动态代理对象，该对象的实现类实现了interfaces数组指定的系列接口，执行代理对象的每一个方法时都会被替换成InvocationHandler接口(自己创建一个类实现该接口)的实例的invoke方法。（实际上方法一需要创建对象的话，也是要一个InvocationHandler对象的）。
---
InvoactionHandler的使用方法：
```
//接口
public interface Person {
    void walk();
    void sayHello(String name);
}
```
```
public class MyInvocationHandler implements InvocationHandler{

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("---正在执行的方法："+method);
        if (args!=null){
            System.out.println("下面是执行该方法时传入的实参为：");
            for (Object val:args){
                System.out.println(val);
            }
        }else System.out.println("该方法没有实参");
        return null;
    }
}
```
```
public class MainTest {
    public static void main(String args[]) throws Exception {
        InvocationHandler handler=new MyInvocationHandler();
        //注意这里的Person是一个接口interface。
        Person p= (Person) Proxy.newProxyInstance(Person.class.getClassLoader(),new Class[]{Person.class},handler);
        p.walk();
        p.sayHello("孙悟空");
    }
}
```

AOP代理：
AOP代理可替代目标对象，AOP代理对象包含了目标对象的全部方法，但AOP代理中的方法与目标对象的方法存在差异：AOP代理里的方法可以在执行目标方法之前、之后插入一些通用处理。
![](http://upload-images.jianshu.io/upload_images/7177220-7c740e313cc32be6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/512)

例子：
* 思路：创建Dog对象和他的GunDog实现类，创建InvocationHandler的实现类MyInvocationHandler, 在invoke方法中调用目标对象的方法，并且在方法前后加入自定义的Utils类的方法, 创建代理对象工厂类MyProxyFactory，用于生产代理对象。
```
public interface Dog {
    void info();
    void run();
}
```
```
public class GunDog implements Dog {
    @Override
    public void info() {
        System.out.println("我是一只猎狗");
    }

    @Override
    public void run() {
        System.out.println("我奔跑迅捷");
    }
}
```
```
public class Util {
    public void method1(){
        System.out.println("----模拟第一个通用方法----");
    }
    public void method2(){
        System.out.println("----模拟第二个通用方法----");
    }
}
```
```
public class MyInvocationHandler implements InvocationHandler {
    //被代理的对象;
    private Object target;

    public void setTarget(Object object) {
        this.target = object;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Util u = new Util();
        u.method1();
        //注意这里第一个参数不要放proxy进去,因为我们要为指定的target生成动态代理对象
        //放proxy是新生成的动态代理对象。
        Object result = method.invoke(target, args);
        u.method2();
        return result;
    }
}
```
```
public class MyProxyFactory {
    public static Object getProxy(Object target){
        //创建一个MyInvocationHandler对象
        MyInvocationHandler handler=new MyInvocationHandler();
        //设置指定的要生成动态代理的target对象
        handler.setTarget(target);
        //创建并返回一个动态代理对象。
        return Proxy.newProxyInstance(target.getClass().getClassLoader(),target.getClass().getInterfaces(),handler);
    }
}
```
```
public class MainTest {
    public static void main(String args[]) throws Exception {
        GunDog dog=new GunDog();
        Dog proxyDog= (Dog) MyProxyFactory.getProxy(dog);
        proxyDog.info();
        proxyDog.run();
    }
}
```
打印：
```
----模拟第一个通用方法----
我是一只猎狗
----模拟第二个通用方法----
----模拟第一个通用方法----
我奔跑迅捷
----模拟第二个通用方法----
```

###结论：动态代理模式让代码结构更加灵活，解耦。
