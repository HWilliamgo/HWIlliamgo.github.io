因为我以前都是用3.0的注解来生成Entity实体类的， 没有用过2.0用代码操作的方式，所以记录一下。

---

![](http://upload-images.jianshu.io/upload_images/7177220-402cec2247d30686.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
官网图如上：
第一点：在你的generator类里面添加依赖。
那么打开AS,新建一个Module，类型为Java Library.

![](http://upload-images.jianshu.io/upload_images/7177220-87cb18ddf4205d09.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在Module的gradle中复制粘贴依赖

![](http://upload-images.jianshu.io/upload_images/7177220-cf7df3b5c6916244.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看官网的第三点：在app中添加依赖：

![](http://upload-images.jianshu.io/upload_images/7177220-0588745c81d04beb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

写一个类用来生成GreenDao。
``` java
public class ExampleDaoGenerator {

    private static final String packageName="GreenDao";
    private static final String generatePath="D:\\AndroidProject\\MyTest\\app\\src\\main\\java\\com\\solory\\mytest";
    public static void main(String args[]) {

        Schema schema=new Schema(1,packageName);

        addRideRecord(schema);
        try {
            new DaoGenerator().generateAll(schema,generatePath);
        }catch (Exception e){
            e.printStackTrace();
        }

    }

    private static void addRideRecord(Schema schema) {
        Entity rideRecord=schema.addEntity("RideRecord");
        rideRecord.addIdProperty();
        rideRecord.addIntProperty("bike_id");
        rideRecord.addDateProperty("start_at");
        rideRecord.addDateProperty("end_at");
        rideRecord.addBooleanProperty("isPay");
        rideRecord.addIntProperty("money");
    }
}
```
之后点击run，立马报错

![](http://upload-images.jianshu.io/upload_images/7177220-6a698b0572011174.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这个错误起码折腾了我两个小时，最终在google后在GreenDao的github的issue那里找到了答案。（我百度了好久好久都没有找到，去你妈的百度）。
https://github.com/greenrobot/greenDAO/issues/619

官方解决方案：

![](http://upload-images.jianshu.io/upload_images/7177220-855e52ba563e2322.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

具体操作为：
添加这么两句在java类中的gradle:
``` java
apply plugin: 'application'
mainClassName = "com.solory.daoexamplegenerator.ExampleDaoGenerator"
```
![](http://upload-images.jianshu.io/upload_images/7177220-e0779975c51ec8e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
点击右边框的
![](http://upload-images.jianshu.io/upload_images/7177220-b4b58940bbf97190.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

双击如下的run文件，GreenDao
![](http://upload-images.jianshu.io/upload_images/7177220-560d2770e0eae996.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注意：如果没有在gradle里面添加apply plugin:'application'那两句的话，是没有application这个包的。
