## 两种方式
1. 在xml文件中的root view上设置id，然后`findViewByid(String resId)`即可。
2. 不用设置id，直接在代码里获取：
``` java
ViewGroup contentView= (ViewGroup) ((ViewGroup) findViewById(android.R.id.content)).getChildAt(0);
```
`contentView`即为所求

---

这是从`DecorView`开始的view tree，debug得到的。
![image.png](https://upload-images.jianshu.io/upload_images/7177220-cd93a7e0e314b94f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
