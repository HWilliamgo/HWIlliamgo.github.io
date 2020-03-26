###作用：
transient关键字可以让实现了Serializable接口的类中的属性不被序列化。
就是哪个属性加了transient，他就不能被序列化。
代码证明：
bean:
```
public class User implements Serializable{
    private  String name;
    private transient int age;
    public  String getName() {
        return name;
    }
    public void setName(String tname) {
        name = tname;
    }
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }
}
```
主函数：
```
import java.io.*;
public class MainTest {
    public static void main(String args[]) {
        //初始化用户
        User user=new User();
        user.setAge(18);
        user.setName("ABC");

        System.out.println("read before Serializable");
        System.out.println("name: "+user.getName());
        System.out.println("age: "+user.getAge());

        File file=new File("D:/user.txt");
        if (file.exists()){
            System.out.println("\nfile delete? : "+file.delete());
        }
        //序列化
        try {
            ObjectOutputStream objo=new ObjectOutputStream(new FileOutputStream("D:/user.txt"));
            objo.writeObject(user);;
            objo.flush();
            objo.close();
        }catch (Exception e){
            e.printStackTrace();
        }
        //反序列化
        try {
            ObjectInputStream inputStream=new ObjectInputStream(new FileInputStream("D:/user.txt"));
            User abc= (User) inputStream.readObject();
            System.out.println("\nread after Serializable");
            System.out.println("name: "+abc.getName());
            System.out.println("age: "+abc.getAge());
        }catch (Exception e){
            e.printStackTrace();
        }

    }
}
```
打印：
```
read before Serializable
name: ABC
age: 18

file delete? : true

read after Serializable
name: ABC
age: 0
```
在这里的private transient int age;就没有序列化成功。

此外，static变量修饰的静态变量，也不能被Serializable序列化。我修改了name属性为static变量，重新测试：
```
public class MainTest {
    public static void main(String args[]) {
        //初始化用户
        User user=new User();
        user.setAge(18);
        User.setName("ABC");

        System.out.println("read before Serializable");
        System.out.println("name: "+user.getName());
        System.out.println("age: "+user.getAge());

        File file=new File("D:/user.txt");
        if (file.exists()){
            System.out.println("\nfile delete? : "+file.delete());
        }
        //序列化
        try {
            ObjectOutputStream objo=new ObjectOutputStream(new FileOutputStream("D:/user.txt"));
            objo.writeObject(user);;
            objo.flush();
            objo.close();
        }catch (Exception e){
            e.printStackTrace();
        }
        //反序列化
        try {
            //在这里改变User的类变量Name属性
            User.setName("DEF");
            ObjectInputStream inputStream=new ObjectInputStream(new FileInputStream("D:/user.txt"));
            User abc= (User) inputStream.readObject();
            System.out.println("\nread after Serializable");
            System.out.println("name: "+abc.getName());
            System.out.println("age: "+abc.getAge());
        }catch (Exception e){
            e.printStackTrace();
        }

    }
}
```
打印：
```
read before Serializable
name: ABC
age: 18

file delete? : true

read after Serializable
name: DEF
age: 0
```
本应该name还是ABC的，但是却出来了DEF，说明这里的name根本不能被序列化，我们读取的是存在内存中的静态变量。

