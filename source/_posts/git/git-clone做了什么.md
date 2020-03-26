> git官方教程：[3.5 Git 分支 - 远程分支](https://git-scm.com/book/zh/v2/Git-%E5%88%86%E6%94%AF-%E8%BF%9C%E7%A8%8B%E5%88%86%E6%94%AF)

打开一个directory，输入
```
git init
git clone <远程地址>
```
发生了如下图所示的事情
![来自官方文档](https://upload-images.jianshu.io/upload_images/7177220-27803b3fdee9a2eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上方是服务器的仓库，`master`指向`f4265`，当克隆仓库之后，整个仓库的所有文件，包括`.git`文件，全部下载到本地的diretory。

git的`clone`命令将远程仓库在本地命名为origin，创建一个指向它的 master 分支的指针，并且在本地将其命名为 origin/master（`如第二个图的origin/master指针`），同时创建一个master指针，并将`HEAD`指针指向这个master指针（`如第二个图的master指针`），这样就有工作的基础，并且工作目录变成master指针指向的快照。

![自己做的带HEAD的图](https://upload-images.jianshu.io/upload_images/7177220-a23047d9b10ad933.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
