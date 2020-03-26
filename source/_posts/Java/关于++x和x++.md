``` java
public class MainJava {
    public static void main(String[] args){
        int a=5;
        int b=a+ ++a;
        System.out.println(b);

        int c=5;
        int d=c++ + ++c;
        System.out.println(d);
    }
}
```

打印：
```
11
12
```

第二个打印结果是12是因为5+7。
首先c++，返回的是5，但是内存中已经改变了c的值，成了6，然后++c到内存中取回的是6，自增，返回7，所以是5+7=12。
