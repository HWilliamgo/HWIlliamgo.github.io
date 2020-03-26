### 1. 两种自定义方式

Gson自定义序列化和反序列的方式有两种
1. 通过`GsonBuilder`来注册适配器。
2. 用`@JsonAdapter`注解来为类型或者属性指定适配器。



其中，以注解的方式用`@JsonAdapter`来实现自定义序列化或反序列化，gson的github的wiki页是没有的，但是参照这个类的注释，也很容易就能实现。



### 2. 例子

场景：

首先，正常的json字符串是这样的结构：

``` json
{
    "family":{
        "Dad":"a",
        "Mon":"b"
    },
    "gender":"male",
    "home":"Shenzhen",
    "name":"William"
}
```

该json中包含两个json对象，一个是外部对象，一个是family对象。那么用类似`GsonFormat`之类的插件，就会生成两个类。

``` kotlin
data class Person(
    val family: Family?,
    val gender: String,
    val home: String,
    val name: String
) {
    data class Family(
        val dad: String,
        val mon: String

    )
}
```



但是，有时候服务器会不合理地返回一个这样的数据：

``` json
{
  "family": "",
  "gender": "male",
  "home": "Shenzhen",
  "name": "William"
}
```

即family这个本应该是json对象的地方，返回了空字符串。

那么当用Gson反序列化的时候，会报这样的错误：

``` 
Exception in thread "main" com.google.gson.JsonSyntaxException: java.lang.IllegalStateException: Expected BEGIN_OBJECT but was STRING at line 2 column 14 path $.family
	at com.google.gson.internal.bind.ReflectiveTypeAdapterFactory$Adapter.read(ReflectiveTypeAdapterFactory.java:226)
	at com.google.gson.internal.bind.ReflectiveTypeAdapterFactory$1.read(ReflectiveTypeAdapterFactory.java:131)
	at com.google.gson.internal.bind.ReflectiveTypeAdapterFactory$Adapter.read(ReflectiveTypeAdapterFactory.java:222)
	at com.google.gson.Gson.fromJson(Gson.java:932)
	at com.google.gson.Gson.fromJson(Gson.java:897)
	at com.google.gson.Gson.fromJson(Gson.java:846)
	at com.google.gson.Gson.fromJson(Gson.java:817)
	at william.MainKtKt.main(MainKt.kt:10)
Caused by: java.lang.IllegalStateException: Expected BEGIN_OBJECT but was STRING at line 2 column 14 path $.family
	at com.google.gson.stream.JsonReader.beginObject(JsonReader.java:386)
	at com.google.gson.internal.bind.ReflectiveTypeAdapterFactory$Adapter.read(ReflectiveTypeAdapterFactory.java:215)
	... 7 more

```

即解析family这个key的时候，本来期望是得到一个对象，但是得到了一个String，即json字符串和你定义的对应的实体类不符合。



那么这个时候可以通过自定义反序列化策略，来在规避gson在反序列化过程中，遇到本应该返回对象，却返回了字符串而引起的错误。

``` kotlin
class CustomFamilyDeserializer : JsonDeserializer<Family> {
    override fun deserialize(json: JsonElement, typeOfT: Type?, context: JsonDeserializationContext): Family? {
        return if (!json.isJsonObject) {
            //如果这个json不是一个jsonObject，那么返回一个null，让gson去把Family属性映射成null即可。
            null
        } else {            
            //如果是jsonObject，那么就用gson继续解析即可。
            Gson().fromJson(json, typeOfT)
            
            //如果用context直接解析，会导致栈溢出
            //context.deserialize(json, typeOfT)
        }
    }
}
```

上面的是一个自定义的`JsonDeserializer`实现。

使用：

1. GsonBuilder

   ``` kotlin
   val gson :Gson = getGsonBuilder()
       .registerTypeAdapter(Person.Family::class.java, Person.Family.CustomFamilyDeserializer())
       .create()
   //json字符串
   val jsonOfPerson=""
   val person :Person =gson.fromJson(jsonOfPerson,Person::class.java)
   ```

2. @JsonAdapter

   直接在Family属性处加上注解

   ``` kotlin
   data class Person(
       @JsonAdapter(Family.CustomFamilyDeserializer::class)
       val family: Family?,
       val gender: String,
       val home: String,
       val name: String
   ) {
       data class Family(
           val dad: String,
           val mon: String
   
       ) {
           class CustomFamilyDeserializer : JsonDeserializer<Family> {
               override fun deserialize(json: JsonElement, typeOfT: Type?, context: JsonDeserializationContext): Family? {
                   return if (!json.isJsonObject) {
                       null
                   } else {
                       //用当前上下文解析，也不会引起栈溢出
                       context.deserialize(json, typeOfT)
                       
                       //新构建一个Gson对象，也不会引起栈溢出
                       //Gson().fromJson(json, typeOfT)
                   }
               }
           }
       }
   }
   ```

   然后直接解析就行

   ``` kotlin
   val jsonOfPerson=""
   val person :Person =gson.fromJson(jsonOfPerson,Person::class.java)
   ```



### 3. @JsonAdapter

``` java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.FIELD})
public @interface JsonAdapter {

  /** Either a {@link TypeAdapter} or {@link TypeAdapterFactory}, or one or both of {@link JsonDeserializer} or {@link JsonSerializer}. */
  Class<?> value();

  /** false, to be able to handle {@code null} values within the adapter, default value is true. */
  boolean nullSafe() default true;

}
```

jsonAdapter的处理目标是`类型`和`变量`。



当他用在变量上（即如上面的例子的用法）时，可以在回调中，无论用context上下文或者新建一个Gson对象，来解析当前json，都不会造成无限循环调用的问题。

而当他作用在类型上时，在回调中，无论用context上下文或者新建一个Gson对象，来解析当前json，都会造成异常。所以，当该注解用在类型上时，回调中一定不能对同样的类型再用Gson去解析。



### 4. GsonBuilder

用GsonBuilder的时候，是通过`registerTypeAdapter`方法注册一个序列化器或反序列化器，那么只要在回调用你不要用当前Gson上下文，创建另一个Gson对象，来进行解析，就不会有任何的问题。



> 结论：
>
> 最好不要在JsonSerializer或JsonDeserializer实现中对当前的类型的对象再用gson解析，因为很可能会引起无限循环地去调用该解析器。如果实在要用，则在GsonBuilder的时候新建一个Gson，或者用@JsonAdapter的时候注解打到变量上而不是类上。

