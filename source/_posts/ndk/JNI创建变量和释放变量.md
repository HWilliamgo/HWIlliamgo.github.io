jni中的数据传递就两种：c层传到java层；java层传到c层。



### 1 当数据从java传递到c



#### 1.1 传递基本数据类型

在Java层定义jni代码：

``` java
public static native void inputInt(int intData);
```

c层实现：

``` c
JNIEXPORT void JNICALL
Java_com_hwilliam_jnilearncmake_NDKTools_inputInt(JNIEnv *env, jclass clazz, jint int_data) {
    int data = int_data;
    LOGD("input int data from java = %d", data);
}
```

由于`jint`是

``` c
typedef int32_t  jint;     /* signed 32 bits */
```

而

``` c
typedef __int32_t     int32_t;
```

而

``` c
typedef int __int32_t;
```

所以，`jint`就是`int`类型，所以直接用`int`类型接收就可以了。



其他的基本数据类型都是这样的，他们的映射关系定义在：

``` c
//	jni.h
/* Primitive types that match up with Java equivalents. */
typedef uint8_t  jboolean; /* unsigned 8 bits */
typedef int8_t   jbyte;    /* signed 8 bits */
typedef uint16_t jchar;    /* unsigned 16 bits */
typedef int16_t  jshort;   /* signed 16 bits */
typedef int32_t  jint;     /* signed 32 bits */
typedef int64_t  jlong;    /* signed 64 bits */
typedef float    jfloat;   /* 32-bit IEEE 754 */
typedef double   jdouble;  /* 64-bit IEEE 754 */
```



#### 1.2 传递引用数据类型



在java层定义jni方法：

``` java
public static native void inputString(String stringData);
```

在c层定义实现：

``` c
JNIEXPORT void JNICALL
Java_com_hwilliam_jnilearncmake_NDKTools_inputString(JNIEnv *env, jclass clazz,
                                                     jstring string_data) {
    //获取java中的string_data，转换成c中的字符串
    const char *string = (*env)->GetStringUTFChars(env, string_data, NULL);
    LOGD("input string data from java = %s", string);
    //释放java中的string_data的引用。
    (*env)->ReleaseStringUTFChars(env, string_data, string);
}
```





#### 1.3 必要的释放

注意，从java传递基本数据类型的时候，是不需要释放引用的，因为基本数据类型在java层并不会导致内存泄漏。而对象的引用，才会导致内存泄漏。

当从java层传递一个引用数据类型（即一个java对象）到c层的时候，这个时候会把该对象的引用暴露给c层，让c层处理，即调用相应的`GetXXX`方法，例如传递`String`对象的时候，c层要调用`jni`函数来处理：`(*env)->GetStringUTFChars`，那么当处理完之后，必须手动释放调用c层对java层的该对象的引用，即调用对应的`(*env)->ReleaseXXX`函数。

即从java传递对象到c的时候，`(*env)->GetXXX`和`(*env)->ReleaseXXX`必须是成对出现的，否则就会造成内存泄漏。



### 2 当数据从c传递到java

从c层传递数据到java层一般涉及到两种方式：

1. java通过有返回值得jni方法调用进入到c层，然后通过该jni方法的返回值，c层返回数据到java层。数据是以同步调用的形式返回返回给java层的。
2. 从c层主动调用java层的静态方法或者实例方法，以异步回调的方式将数据返回给java层。



#### 2.1 传递基本数据类型



##### 同步返回

java定义Jni方法

``` java
public static native int getIntFromCSync();
```

c层实现

``` c
JNIEXPORT jint JNICALL
Java_com_hwilliam_jnilearncmake_NDKTools_getIntFromCSync(JNIEnv *env, jclass clazz) {
    int data = 100;
    jint data2Java = data;
    return data2Java;
}
```

因为`jint`其实就是`int`型，所以可以直接用int来接收，并直接返回给Java。



##### 异步回调

Java层：定义一个jni方法，用于发起异步回调，然后定义一个java层的回调方法，这里的回调的参数是基本类型int。

``` java
public static native void getIntFromCAsync();

public static void onGetIntFromC(int dataFromC) {
  LogUtils.d(dataFromC);
}
```



c层：实现jni方法，并通过`CallXXXMethod()`方法来回调java层的方法。

``` c
JNIEXPORT void JNICALL
Java_com_hwilliam_jnilearncmake_NDKTools_getIntFromCAsync(JNIEnv *env, jclass clazz) {
    jint data2Java = 200;

    jmethodID onGetIntFromC = (*env)->GetStaticMethodID(env, clazz, "onGetIntFromC", "(I)V");
    (*env)->CallStaticVoidMethod(env, clazz, onGetIntFromC, data2Java);

}
```





#### 2.2 传递引用数据类型



##### 同步返回

``` java
public static native String getStringFromCSync();
```

``` c
JNIEXPORT jstring JNICALL
Java_com_hwilliam_jnilearncmake_NDKTools_getStringFromCSync(JNIEnv *env, jclass clazz) {
    char *string = "abcdefg";
    jstring result = (*env)->NewStringUTF(env, string);
    return result;
}
```



##### 异步回调

``` java
public static native void getStringFromCAsync();
public static void onGetStringFromC(String dataFromC) {
    LogUtils.d(dataFromC);
}
```

``` c
JNIEXPORT void JNICALL
Java_com_hwilliam_jnilearncmake_NDKTools_getStringFromCAsync(JNIEnv *env, jclass clazz) {
    //获取字符串
    char *string = "123456789";
    jstring result = (*env)->NewStringUTF(env, string);
    //获取要回调的java方法
    jmethodID onGetStringFromC = (*env)->GetStaticMethodID(env, clazz, "onGetStringFromC","(Ljava/lang/String;)V");
    //回调
    (*env)->CallStaticVoidMethod(env, clazz, onGetStringFromC, result);
    //release
    (*env)->DeleteLocalRef(env, result);
}
```



此外，当想要查找java方法对应的方法签名的时候，androidStudio可以很方便地用代码提示地方式自动生成方法签名：在方法签名参数位置先填上""，然后光标放过去，并按住alt+enter，就出现了提示：

![](https://upload-images.jianshu.io/upload_images/7177220-68a690631e42959f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### 2.3 必要的释放

这里用到了释放函数：`(*env)->DeleteLocalRef(env, jobject);`，该函数是用来释放在c层创建的java对象的局部引用。什么时候该使用这个函数来释放在c层创建的`jobject`对象呢？异步回调形式的`jni`函数。

那么为什么同步返回的`jni`函数不需要呢？



实际上，在c层调用jni函数`NewXXX()`函数来创建一个对象的时候，会把这个对象放入到当前`jni`函数特有的一段内存区域中，称为**本地引用根的集合**，每当在c层调用`NewXXX()`函数创建java对象的时候，都会把创建好的对象的引用保存一份在这个**本地引用根的集合**中。

> 我认为这么做的原因是防止JVM在GC的时候把这个对象给干掉了，因为这个对象这个时候还在c层，没有返回给Java层，即java层是没有任何引用指向这个对象的。这种机制是为了保护这个在c层创建的java对象。

而当从当前的`jni`函数返回的时候（或者在c层创建的一个附着到JVM的线程结束了，即从JVM脱离了），就会把这个**本地引用根的集合**释放掉，返回到Java层的那些对象就可以接着被java层去引用，去处理，而没有返回java层的对象，在这个时候已经是GC unreachable了，就会被干掉了。

但是！！！，一个`jin`方法可能并没有那么简单的逻辑，就创建两个对象就返回给Java了，很多时候，`jni`方法进入到c层之后，会在c层开启新的线程，而新的线程中又会去通过`NewXXX()`函数创建其他的java对象，并在c层主动发起回调，将该对象和其他可能的数据传到java层。这个时候，线程中创建的所有的java对象只有在线程结束的时候才会释放掉**本地引用根的集合**，如果不手动释放，那么这些对象在返回到java层之后，使用之后，c层也不会再去使用，就造成了内存泄漏了。



因此，最好的方法就是：在调用`NewXXX()`创建了Java对象之后，除非这个对象马上通过当前的jni函数返回到java（c层主动回调java函数的不算，必须是当前jni方法），否则使用完之后要调用`(*env)->DeleteLocalRef(env, jobject);`来释放这个引用。



---

参考：https://www.ibm.com/support/knowledgecenter/SSYKE2_7.0.0/com.ibm.java.aix.70.doc/diag/understanding/jni_gc.html

我自己对这个参考文档做了一些翻译：



### 翻译参考文献：

#### JNI对象引用概览

关于GC如何找到JNI对象的引用的实现细节并没有在JNI文档中说明。但是，JNI文档的确指明了JNI对象所需要的一些可靠和可预见的表现。

#### 本地和全局引用

本地引用被限制在创建他们的栈和线程中，并且在创建他们的栈帧返回的时候就会被自动删除。

全局引用允许原生代码去将一个本地引用升级成一种可以被任何附着到JVM的线程访问的全局引用。

#### 全局引用和内存泄漏

全局引用不会自动被删除，所以开发者必须处理他们的内存管理。每一个全局引用都建立了一个GC root，并且让他的整个GCtree都可到达。因此，每一个被创建的GCroot必须被手动释放以防止内存泄漏。

内存泄漏最终导致OOM，这些错误很难处理，尤其是你没有实现JNI异常处理。

为了解决内存泄漏的问题，JNI提供了两个方法：

* NewWeakGlobalRef
* DeleteWeakGlobalRef

这些方法能用弱引用的方式来避免内存泄漏。

#### 本地引用和内存泄漏

在大多数情况下，GC对那些不在栈帧范围中的本地引用的自动的垃圾回收已经足够适用。这种自动GC回收会在原生线程(或原生方法)返回到java或者从JVM中脱离的时候发生。但是，如果不满足这种条件的时候，就可能会发生本地引用的内存泄漏。例如：当原生方法并没有返回到java或者一个线程创建了本地引用但是却并没有从JVM中脱离。

考虑以下代码的情况，原生代码在一个循环中不断地创建新的本地引用：

``` c
while ( <condition> )
{
   jobject myObj = (*env)->NewObject( env, clz, mid, NULL );

   if ( NULL != myObj )
   {
      /* we know myObj is a valid local ref, so use it */
      jclass myClazz = (*env)->GetObjectClass(env, myObj);

      /* uses of myObj and myClazz, etc. but no new local refs */

      /* Without the following calls, we would leak */
      (*env)->DeleteLocalRef( env, myObj ); 
      (*env)->DeleteLocalRef( env, myClazz );
   }

} /* end while */
```

尽管`myObj`和`myClazz`变量在循环中每次都指向了新的对象，但是，用这些jni方法创建的每一个新的对象都在**本地引用根的集合**中被引用了。

这些引用必须被手动地移除，使用`DeleteLocalRef`方法。如果不调用这个方法，这些本地引用会一直保持着泄漏，直到这个方法返回到java或者线程从jvm中脱离。



#### JNI弱全局引用

弱全局引用是一种特殊的全局引用。他们可以在任何线程中被使用，并且可以在jni方法之间被调用，并且不会作为GC root。GC会在任何时候将一个没有强引用的对象回收掉。

你必须小心地使用弱全局引用。如果该若全局引用被垃圾回收了，那么这个引用就指向了null，一个null引用只能被少部分JNI函数调用。去检查一个弱全局引用是否已经别回收掉了，使用`IsSameObject()`来和NULL对比。

大多数JNI函数对弱全局引用的调用都是不安全的，即使你测试过这个弱引用是非空的。因为这个弱全局引用可能在检测过后或者在你调用过程中又被回收了。你应该在使用弱全局引用之前，将他升级成一个强引用。例如使用：`NewLocalRef`或者`NewGlobalRef`

弱全局引用的使用必须要调用`DeleteWeakGlobalRef`来释放。否则会导致缓慢的GC，最终还是会导致OOM。

#### JNI引用管理

There are a set of platform-independent rules for JNI reference management

These rules are:

1. JNI references are valid only in threads attached to a JVM.
2. A valid JNI local reference in native code must be obtained:
   1. As a parameter to the native code
   2. As the return value from calling a JNI function
3. A valid JNI global reference must be obtained from another valid JNI reference (global or local) by calling NewGlobalRef or NewWeakGlobalRef.
4. The null value reference is always valid, and can be used in place of any JNI reference (global or local).
5. JNI local references are valid only in the thread that creates them and remain valid only while their creating frame remains on the stack.



##### 1 N2J转换

当原生代码在一个线程中调用java代码时，这个线程必须先附着到当前进程的JVM上。

每一个传递了对象应用的N2J（native to java）调用必须要通过jni方法获取有效的jobject对象才能传递到java，因此他们需要是有效的本地或者全局jni引用。任何从N2J方法返回的引用都是JNI本地引用。

##### 2 J2N调用

JVM必须确保任何从java到native传递的对象和在native中创建的新的java对象都是GC可到达的（否则在下一次GC的时候就对象就被回收了）。要满足这个GC的要求，JVM分配了一块小段特殊的内存称为：**本地引用根的集合**。

**本地引用根的集合**在以下情况会被创建：

* 一个线程被附着到JVM上时。
* 每一个J2N转换发生。（即对象从java传到c层，要防止这个对象被JVM意外回收，因此要放在**本地引用根的集合**）

在原生代码中创建的新的对象都会被添加到这个J2N根集合中，除非你用`PushLocalFrame`JNI方法来创建一个新的本地栈。

默认的**本地引用根的集合**是足够大的，能够在每个J2N调用中容纳16个本地引用。

##### 3 J2N返回

当从原生方法返回到java时，对应的JNI本地引用，即由这次J2N方法创建的**本地引用根的集合**会被自动释放。

如果JNI本地引用是某个对象的唯一的引用，那么这个对象在从J2N方法返回的时候就不再GC可到达并且会自动触发他的垃圾回收。这种机制简化了JNI开发者的内存管理。






