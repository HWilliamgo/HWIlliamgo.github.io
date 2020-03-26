借鉴自 *[读取assets目录下的数据库文件](http://blog.csdn.net/u010800530/article/details/40192279)*
1. Project-app-main文件夹下new一个assets folder，如图：
![](http://upload-images.jianshu.io/upload_images/7177220-e0eee73bb5017626.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. 准备好的数据库文件直接复制粘贴过去。

![](http://upload-images.jianshu.io/upload_images/7177220-387f2bc0d796600a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3. 用代码把assets路径下的该文件用I/O的方式写入对应包名的路径下，代码如下：
```
public class DataBaseManager {
    String filePath = "data/data/com.solory.william.myweather/databases/my_db.db";
    String pathStr = "data/data/com.solory.william.myweather/databases";

    SQLiteDatabase sqLiteDatabase;

    public SQLiteDatabase openDataBase(Context context) {
        File jhPath = new File(filePath);
        if (jhPath.exists()) {
            //存在则直接打开
            return SQLiteDatabase.openOrCreateDatabase(jhPath, null);
        } else {
            //不存在则复制粘贴到该路径下
            File path = new File(pathStr);
            if (path.mkdir()) {
                Log.d("tag", "创建Path成功");
            } else {
                Log.d("tag", "创建Path失败");
            }
            try {
                AssetManager assetManager = context.getAssets();
                InputStream is = assetManager.open("my_db.db");
                FileOutputStream fos = new FileOutputStream(jhPath);
                byte[] buffer = new byte[1024];
                int count = 0;
                while ((count = is.read(buffer)) > 0) {
                    fos.write(buffer, 0, count);
                }
                fos.flush();
                fos.close();
                is.close();
            } catch (Exception e) {
                e.printStackTrace();
                return null;
            }
            return openDataBase(context);
        }
    }
}
```
4. 用的时候直接：

![](http://upload-images.jianshu.io/upload_images/7177220-dc9b5874a4fab837.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
5. run

end: 查看一下数据库的路径，发现出现我们要的数据库了。

>另外，这个导入的数据库是可以用GreenDao的，只要把对应的id和property都正确的生成就可以了。
