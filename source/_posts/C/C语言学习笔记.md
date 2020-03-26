## 1. C的预处理器

### .h头文件

``` c
//文件名：main.c

#include <stdio.h>
int main(void){
    ...
}
```

`#include`是C语言的预处理指令，C语言编译器在编译前会对源码进行预处理工作。他的作用就是把所有头文件中的内容，完全copy进入当前的`.c`文件中。

一般头文件中定义一些常量或者函数，而其函数实现在另一个文件中。那么，编译完成之后，C的连接器就会将这个`main.c`文件中用到的其他库中的文件给提取出来，一起和当前的`main.c`文件组合成一个二进制的可执行文件`xxx.exe`。

![](https://upload-images.jianshu.io/upload_images/7177220-a4adc197d23d3697.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### #defin定义常量

通用格式：`#define NAME value`（没有=号，结尾没有分号）

在编译程序的时候，程序中所有引用`NAME`变量的地方都会被替换成`value`，该过程称为：编译时替换。程序在运行时，所有替换均已完成，这样定义的常量叫做明示常量。



## 2. 数据类型

### 布尔类型

C99添加了`_Bool`类型，因为C语言用1表示true，用0表示false，所以`_Bool`也是一种整数类型。

此外，凡是非0的int类型，都可以表示true，而整数0表示false。示例代码：

``` c
#include <stdio.h>

int main(void) {
    int a = 3 > 1;
    int b = 3 < 1;
    printf("%d", a);
    printf("\n");
    printf("%d", b);
    printf("\n");

    int c = 3;
    if (c) {
        printf("凡是非0的整数，都视为true");
    }
    printf("\n");
    int d = 0;
    if (!d) {
        printf("而整数0刚好表示false");
    }
}

打印：
1
0
凡是非0的整数，都视为true
而整数0刚好表示false
```



### 字符串类型

C语言没有专门用于存储字符串的变量类型，字符串都被存储在char类型的数组中。且char数组中的最后一个元素一定是空字符`\0，`标志着字符串的结束。

那么如果`char stringA[40]`的字符串变量`stringA`，只能放下39个字符。

注意，C语言中声明数组的方式和java不一样，Java中可以用`int[] a`的方式声明数组变量，而C中一定要用`int a[]`的方式来声明。

那么单个字符的字符串和字符的区别是什么？

例如`‘X’`和`"X"`的区别？区别在于，字符串类型的“X”是用数组来存储的，最后会有一个空字符`\0`，即会占两个字符的空间。



## 3. 指针

### 3.1 定义

#### 取变量的指针：&

后面跟一个变量名时，&给出该变量的地址（或指针）。

&取出的地址可以直接赋值给指针，那么就可以这样写：

``` c
int value=10;
int *pointerOfValue=&value;
```

那么变量`pointerOfValue`就是指向`value`变量的指针。那么&可以直接理解为取一个变量的指针。

注意，一个变量的指针只能读不能写，因为一个变量的内存地址（指针）是唯一的，你只能读取，你不能写入。

#### 取指针指向的变量：*

后面跟一个指针变量或者地址时，*给出存储在指针指向地址上的值。

*运算符是直接作用在指针变量上的，返回指针指向的值：

``` c
int valueB=*pointerOfValue;
```

例如这个`valueB`，此时就等于10。

注意，*运算符可以用来被指针变量用来读值，当然也能用来写入新的值。

### 3.2 指针变量声明

``` c
//pi变量是指向int类型变量的指针变量。
int *pi;、
//pc变量是指向char类型变量的指针变量。
char *pc;
//同上
float *pf, *pg;
```

注意：`int *pi`声明了一个`pi`变量，他本身的类型不是int类型的，他是指针类型，不是基本数据类型了，只是他这样声明表示指向的int类型的内存地址。例如两个指针不能相乘，但是两个整型可以相乘，而且打印的时候，指针类型的变量也要用`%p`来转换。



示例：

``` c
#include <stdio.h>

int main(void) {
    int a = 10;
    int *addressOfA = &a;
    int valueOfA = *addressOfA;
    printf("%p", addressOfA);
    printf("\n");
    printf("%d", valueOfA);
    return 0;
}
```

打印：

```
0x7fff0431fba8
10
```



### 3.3 指针实战

原则：如果一个函数要计算或者处理值，那么就直接传递变量的值；如果一个函数要改变主调函数的变量，那么就传递那个变量的引用。

在面向对象语言如`Java`或者`Kolin`中，要在被调函数中改变主调函数的变量，只能通过return语句来实现。而c语言则可以用指针来实现，如下示例：

``` c
#include <stdio.h>

void interchange(int *u, int *v);

//u和v都是指针，那么通过*运算符就可以读写所指向的变量实际的值。
void interchange(int *u, int *v) {
    int temp;
    temp = *u;
    *u = *v;
    *v = temp;
}

int main(void) {
    int x = 5, y = 10;
    printf("原先x=%d ,y=%d", x, y);
    interchange(&x, &y);
    printf("\n");
    printf("现在x= %d,y=%d", x, y);
    return 0;
}
```

打印：

```
原先x=5 ,y=10
现在x= 10,y=5
```



在许多语言中， 地址都归计算机管， 对程序员隐藏。 然而在 C 中， 可以通过&运算符访问地址， 通过*运算符获得地址上的值



### 3.4 指针和数组

#### 3.4.1 数组变量就是一个指针变量

例如：

``` c
int intArray[SIZE];
```

声明了一个`SIZE`大小的数组变量，而该数组变量`intArray`实际上是一个指针变量。该指针变量所指向的值是数组的第一个元素的值，即`intArray[0]`，而`intArrya[0]`的元素的指针变量就是数组所代表的指针变量。

例如：

``` c
//以下语句成立。注：& 表示取变量的指针
intArray=&intArray[0];
```



#### 3.4.2 指针加1，指针的值递增它所指向的类型的大小。

例如：

``` c
//声明int
int value = 999;
//取999变量的指针
int *pointerOfInt = &value;
//为指针加1
pointerOfInt += 1;
//打印此时指针所代表的值
printf("----%d", *pointerOfInt);
```

输出：`----0 `

即，当指针加一的时候，指针会顺延着在内存地址上，增加一个单位的所指向的变量的类型的大小的长度。例如这里是int，那么指针加一的时候，指针就往后移动4个字节的大小。



而数组刚好代表的是一段连续的内存地址，那么当拿到数组第一个元素的指针，又知道了数组的大小，则可以直接使用指针来访问数组了。（得益于数组在内存上是连续的，链表就不行了。）

如下例子演示了如何使用指针来操纵数组，以及如何把数组直接当成一个指针来使用：

``` c
#include <stdio.h>

#define SIZE 4

int main()
{
    //声明数组
    int intArray[SIZE];
    //遍历，为每个数组分配变量
    for (int i = 0; i < SIZE; i++)
    {
        intArray[i] = 100 + i;
    }
    //声明指针
    int *pointer;
    //指针指向数组
    pointer = intArray;

    //从指针中取出指向的数组intArray的每个元素
    for (int i = 0; i < SIZE; i++)
    {
        int valueTakeFromPointer = *(pointer + i);
        printf("value taken from pointer = %d \n", valueTakeFromPointer);
    }

    printf("------------------------\n");

    //同理，由于数组本身就是指针，那么直接操作数组也可以。
    for (int i = 0; i < SIZE; i++)
    {
        int valueTakeFromPointer = *(intArray + i);
        printf("value taken from pointer = %d \n", valueTakeFromPointer);
    }
    return 0;
}

```

打印：

``` c
@"value taken from pointer = 100 \r\n"
@"value taken from pointer = 101 \r\n"
@"value taken from pointer = 102 \r\n"
@"value taken from pointer = 103 \r\n"
@"------------------------\r\n"
@"value taken from pointer = 100 \r\n"
@"value taken from pointer = 101 \r\n"
@"value taken from pointer = 102 \r\n"
@"value taken from pointer = 103 \r\n"
```



#### 3.4.3 二级指针

二级指针表示：指向指针的指针



由于指针变量实际上可以表示一个数组，那么指针的指针，实际上可以用来表示指针的数组。



以下内容来自知乎：

在C语言中，二级指针有什么用处？

- **指针的数组**，尤其是指向 struct 的指针的数组，比如

```c
typedef struct {...} Record;
typedef struct {
    size_t length;
    Record **records;
} RecordList;
```

- **指针的引用**，比如拿来改指针变量
- **值类型的二维数组**

> 作者：Belleve
> 链接：https://www.zhihu.com/question/19831228/answer/130917500
> 来源：知乎
> 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



##  4 存储类别、链接、和内存管理

### 4.1 作用域

1. 块作用域

2. 函数作用域

3. 函数原型作用域

4. 文件作用域

   文件作用域的变量也叫全局变量，存在于整个程序的运行周期。

   ``` c
   #include <stdio.h>
   
   //units变量就是文件作用域的变量。
   int units=0;
   
   int main(void){
       ...
   }
   ```

### 4.2 链接

c的变量有三种链接属性：

1. 外部链接。

   可以被外部的文件所使用。

   ```c
   int a=1;
   int main(void){
   
   }
   ```

2. 内部链接。

   只能在当前的文件中使用。

   ``` c
   static int a=1;
   int main(void){
   
   }
   ```

3. 无连接。

   块作用域，函数作用域，函数原型作用域的变量，都是无连接的。

### 4.3 存储期

1. 静态存储期

   静态存储期的对象在程序执行的期间一直存在。文件作用域的变量具有静态存储期。

2. 线程存储期

   线程存储期的对象，从被声明时到线程结束都一直存在。以关键字_Thread_local声明一个对象时，每个线程都将获得该变量的私有备份。

3. 自动存储期

   块作用域的变量通常有自动存储期。进入块时为变量分配内存，退出块时释放刚才分配的内存。

   一般在方法块中声明的变量都是自动存储期的，但是块作用域变量也能具有静态存储期。例如：

   ``` c
   void more(int number){
       int index;
       static int ct=0;
       //...
       return 0;
   }
   ```

   上述代码中，变量ct存储在静态内存中，它从程序被载入到程序结束期间都存在，。但是他的作用域在more()函数中，那么只有执行more的时候，程序才能用ct来访问他所指定的对（但是， 该函数可以给其他函数提供该存储
   区的地址以便间接访问该对象， 例如通过指针形参或返回值）。

4. 动态分配存储期

### 4.4 5种存储类

![](https://upload-images.jianshu.io/upload_images/7177220-ad48776eedd5fc15.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 4.5 块作用域的静态变量

1. 静态变量分为两种：①文件类变量：直接定义在文件里面的变，拥有全局声明周期。②用static关键字作前缀的变量，当他定义在文件中时，他是仅该文件可见的全局变量；当他定义在块或者方法里面时，他是当前块中可见的全局变量。
2. 静态变量的初始化在程序被载入内存时已经执行完毕，他们只在编译的时候被初始化一次。（因此在方法中的静态变量之初始化一次，在运行时不会重复初始化的）

### 4.6 外部链接的静态变量

外部链接的静态变量具有文件作用域、 外部链接和静态存储期

当然， 为了指出该函数使用了外部变量， 可以在函数中用关键字extern再次声明。 如果一个源代码文件使用的外部变量定义在另一个源代码文件中， 则必须用extern在该文件中声明该变量。
