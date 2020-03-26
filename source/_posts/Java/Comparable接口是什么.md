``` java
//Comparable接口：
//实现了Comparable接口的类的对象，
//可被Collections.sort(List)方法和Arrays.sort(Object[])方法自动地排序。

//实现该接口的方法可以作为SortedMap的key
//或者SortedSet中的element,但是需要额外地声名一个Comparator
public interface Comparable<T>{
  public int compareTo(T o);
}
```

---

Comparable是内部比较器，Comparator是外部比较器。

以下为Comparable和Comparator的测试程序：
``` java

import java.util.*;

public class MainJava {

    public static void main(String[] args) throws Exception {
        ArrayList<Person> list=new ArrayList<>();
        list.add(new Person(20,"ccc"));
        list.add(new Person(30,"AAA"));
        list.add(new Person(10,"bbb"));
        list.add(new Person(40,"ddd"));

        System.out.println(list);

        Collections.sort(list);
        System.out.println(list);

        Collections.sort(list,new AscAgeComparator());
        System.out.println(list);

        Collections.sort(list,new DescAgeComparator());
        System.out.println(list);
    }


    private static class Person implements Comparable<Person>{
        public int age;
        public String name;

        public Person(int age, String name) {
            this.age = age;
            this.name = name;
        }

        @Override
        public String toString() {
            return name+"-"+ age;
        }

        boolean equals(Person person) {
            return (this.age==person.age)&&this.name.equals(person.name);
        }

        @Override
        public int compareTo(Person o) {
            return this.name.compareTo(o.name);
        }
    }
    private static class AscAgeComparator implements Comparator<Person>{

        @Override
        public int compare(Person o1, Person o2) {
            return o1.age-o2.age;
        }
    }
    private static class DescAgeComparator implements Comparator<Person>{

        @Override
        public int compare(Person o1, Person o2) {
            return o2.age-o1.age;
        }
    }
}
```
打印：
```
[ccc-20, AAA-30, bbb-10, ddd-40]
[AAA-30, bbb-10, ccc-20, ddd-40]
[bbb-10, ccc-20, AAA-30, ddd-40]
[ddd-40, AAA-30, ccc-20, bbb-10]

Process finished with exit code 0

```
