####  首先,这玩意怎么升级数据库？
1. 传统SQLite语句做法：
写一个自己的Helper继承自SQLiteOpenHelper，重写里面的onUpgrade方法，方法里做的事情就是（如图）：
* 先把原先的表drop下来，就是删了
* 然后再重新创建![](http://upload-images.jianshu.io/upload_images/7177220-b362526a47bd073a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)然后拿helper的时候就用的自己的Helper就完事了，接着这里第四个参数就是数据库版本号，填一个比之前大的数字就OK了。![](http://upload-images.jianshu.io/upload_images/7177220-24283403976e0889.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. GreenDao：
问题：greendao里面没有那种指定版本号就升级的方法，摸索之后如下：![](http://upload-images.jianshu.io/upload_images/7177220-079549c4fe653457.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/7177220-806fd77372e0a3ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在这里更改数据库版本，然后点击make project，下次安装的时候就升级了~

# 然而，默认情况下GreenDao升级数据库也是会直接drop掉table，那样数据就全部清空了！！！
* 解决方法如下：
借鉴github上面一个开源项目：**[GreenDaoUpgradeHelper](https://github.com/yuweiguocn/GreenDaoUpgradeHelper/blob/master/README_CH.md)**
>*按照作者的思路走就很简单了，以下是实践起来的步骤：*

1.在根目录的build.gradle文件的repositories内添加如下代码：

	allprojects {
		repositories {
			...
			maven { url "https://jitpack.io" }
		}
	}
2.添加依赖（greendao 3.0及以上）

	dependencies {
	        compile 'org.greenrobot:greendao:3.2.0'
	        compile 'com.github.yuweiguocn:GreenDaoUpgradeHelper:v2.0.0'
	}
3. 自己建一个Helper类继承OpenHelper:
```
public class MyDataBaseHelper extends DaoMaster.OpenHelper {
    public MyDataBaseHelper(Context context, String name, SQLiteDatabase.CursorFactory factory) {
        super(context, name, factory);
    }
    //这里重写onUpgrade方法
    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        MigrationHelper.migrate(db, new MigrationHelper.ReCreateAllTableListener() {
            @Override
            public void onCreateAllTables(Database db, boolean ifNotExists) {
                DaoMaster.createAllTables(db, ifNotExists);
            }

            @Override
            public void onDropAllTables(Database db, boolean ifExists) {
                DaoMaster.dropAllTables(db, ifExists);
            }
            //注意此处的参数StudentDao.class，很重要（一开始没注意，给坑了一下），它就是需要升级的table的Dao,
            //不填的话数据丢失，
            // 这里可以放多个Dao.class，也就是可以做到很多table的安全升级，Good~
        }, StudentDao.class);
    }
}
```
4. 记得这里的OpenHelper要用刚才自己写的那个。


![](http://upload-images.jianshu.io/upload_images/7177220-5d1a7e544affa17b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

大功告成，然后每次要升级就去改gradle的schemaVersion就是了。

稳了稳了~~
