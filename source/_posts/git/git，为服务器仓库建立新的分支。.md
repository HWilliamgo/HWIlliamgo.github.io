1. 直接在github页面上面
![](https://upload-images.jianshu.io/upload_images/7177220-52841b598e3056d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当前切换到了master分支，如果文本框输入后一个新的分支名字后，确认，就会创建一个分支指向master分支。要创建一个分支指向develop分支，就将当前分支切换到develop上面。

这一切都用鼠标可以完成，很直观简洁。

2. 通过git bash。
如果我要在服务器创建一个新的分支bugfix指向develop，这做不到。只能通过push命令，让这一次push跟踪到develop，在develop分支上面生成新的快照，并将新的bugfix指向新的版本快照。

在本地的develop分支`chekout -b`一个`bugfix`分支，然后执行`git branch -u origin/bugfix`来追踪远程的`bugfix`分支(这时他不存在)，然后执行`git push origin bugfix`。

由于我们本地追踪的远程分支`origin/bugfix`不存在，而本地分支`bugfix`是在`develop`基础上创建的，因此新建在服务器会基于`develop`分支新建一个`bugfix`分支，最后达到新建分支的目的。

---

结论：还是直接去github创建分支简单点，也不容易出错。
