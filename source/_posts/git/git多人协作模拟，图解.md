

> 说明，为了制图方便，因此将版本快照用`普通的数字`来表示，实际上不严谨，应该像官网用`SHA-1` 值如`92ec2`来表示版本快照更加合适。

现在有两个人协作开发一个项目，master分支是稳定版，随时可发布，develop分支是开发版，是平时开发用的分支。

由Peter和Tony负责开发这个项目，两个人各自完成开发，测试后，`push`到服务器就可以下班。

此时在github或者gitlab上的`.git`表示的代码仓库版本情况如图


![image.png](https://upload-images.jianshu.io/upload_images/7177220-dc1eaec579917410.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在Peter和Tony两人各自打开git bash输入
```
git clone 服务器地址
```
`git clone`指令做了4个事情：
1. git自动将整个代码仓库包括`.git`文件夹下载下来。
2. 自动将服务器的`master`和`develop`分支分别修改为`origin/master`和`origin/develop`，并创建一个master分支指向`origin/master`分支指向的分支。
3. 并自动执行了`
git branch --set-upstream-to master origin/master`来设置本地`master`分支跟踪`origin/master`分支。
4. 并将`HEAD`指向master，使得工作目录全部切换到master指向的**版本快照-->3**
![](https://upload-images.jianshu.io/upload_images/7177220-fcf78898bebfc417.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](https://upload-images.jianshu.io/upload_images/7177220-d37ae374ddc3ea19.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 注意：除非执行`git fetch`或者`git pull`来更新两个远程分支`origin/master`和`origin/develop`，否则远程分支将永远不变，分别指向版本快照`3`和`5`


## 现在单独看向Peter的操作：
Peter在执行了`git clone 服务器地址`之后，因为要在换到develop分支上开发，而不是在master分支上开发，因此执行如下指令：
```
//创建一个develop分支指向origin/develop分支指向的版本快照，
//HEAD指向develop分支（即切换到develop分支）。
git checkout -b develop origin/develop
```
![](https://upload-images.jianshu.io/upload_images/7177220-39554b0018a7bedf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
此时Peter可以开始在develop分支上用`git add`和`git commit`继续开发了，现在Peter开发到了一个满意的阶段，经过测试后，打算`git push`到远程分支。
![Peter开发到了满意的版本快照<7>的阶段](https://upload-images.jianshu.io/upload_images/7177220-6249d19f318b7639.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
执行
```
git push origin develop
```
![原本develop指向5，现在将7作为5的下一个结点，并将develop指向7](https://upload-images.jianshu.io/upload_images/7177220-9a7e6eb652aea306.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

push成功，因为服务器的版本没有更新，因此直接push即成功了，现在看向Tony开发的怎么样了
## Tony做的怎么样了
![Tony从5开始，开发到了11](https://upload-images.jianshu.io/upload_images/7177220-73fc3058e0bf890c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
现在Tony也打算将自己的新的开发提交到服务器上去。（我们知道这会失败，但是来看看发生了什么）
执行
```
$ git push origin develop

To github.com:这个项目的包名.git
 ! [rejected]        dev -> dev (non-fast-forward)
error: failed to push some refs to 'git@github.com:这个项目的包名.git'
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart. Integrate the remote changes (e.g.
hint: 'git pull ...') before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```
英文提示说Tony提交的push被拒绝了，因为Tony的远程分支`origin/develop`落后于服务器的`develop`分支。且可以考虑用`git pull`来更新合并本地分支。

而`git pull`其实是两个命令`git fetch`和`git merge`按顺序执行。
我们执行
```
git fetch origin
```
![origin/develop原本指向5，现在和服务器的develop同步，指向7](https://upload-images.jianshu.io/upload_images/7177220-31c923e4a02c0102.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
执行
```
git merge origin/develop
```
![merge成功后会生成一个新的版本快照，它由11和7合并而来，因此有两个父结点，此处定义它为12](https://upload-images.jianshu.io/upload_images/7177220-21451a0dc84005bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在已经同步好了，执行
```
git push origin develop
```
![12作为7的子节点被添加，并且develop指向12](https://upload-images.jianshu.io/upload_images/7177220-f0065661723533a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---
好了，现在Tony和Peter都完成了提交，改下班了。






