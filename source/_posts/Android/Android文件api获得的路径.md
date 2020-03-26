## InternalStorage

`/data/user/0`等同于`/data/data`

```java
context.getFilesDir();
//路径：/data/user/0/包名/files
```

``` java
context.getCacheDir();
//路径：/data/user/0/包名/cache
```

``` java
context.getDir("abc",MODE_PRIVATE);
//路径：/data/user/0/包名/app_abc
```

``` java
contexnt.createTmpFile("myTmp",".suffix");
//路径：/data/user/0/包名/cache/myTmp一串随机数字.suffix
```



## ExternalStorage

`/storage/emulated/0`等同于`/sdcard`

```java
context.getExternalCacheDir();
//路径：/storage/emulated/0/Android/data/包名/cache
```

``` java
context.getExternalFilesDir();
//路径：/storage/emulated/0/Android/data/包名/files
```



关于`InternalStorage`和`ExternalStorage`的详细介绍跳转文档：

https://developer.android.com/training/data-storage/files#WriteExternalStorage
