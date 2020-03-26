## 跟踪远程分支
如果用`git push`指令时，当前分支没有跟踪远程分支（没有和远程分支建立联系），那么就会git就会报错
```
There is no tracking information for the current branch.
Please specify which branch you want to merge with.
```
因为当前分支没有追踪远程指定的分支的话，当前分支指定的版本快照不知道要作为服务器哪一个分支的版本快照的子节点。简单来说就是：不知道要推送给哪一个分支。
那么如何建立远程分支：
1. 克隆时自动将创建好的`master`分支追踪`origin/master`分支
```
git clone 服务器地址
```
2. 
```
git checkout -b develop origin/develop
```
在远程分支的基础上建立`develop`分支，并且让`develop`分支追踪`origin/develop`远程分支。
3. 
```
git branch --set-upstream branch-name origin/branch-name
```
将`branch-name`分支追踪远程分支`origin/branch-name`
4. 
```
git branch -u origin/serverfix
```
设置当前分支跟踪远程分支origin/serverfix

## 查看本地分支和远程分支的跟踪关系
```
git branch -vv
```

比如输入
```
$ git branch -vv
  develop   08775f9 [origin/develop] develop
  feature_1 b41865d [origin/feature_1] feature_1
* master    1399706 [my_github/master] init commit
```
`develop`分支跟踪`origin/develop`
`feature_1`分支跟踪`origin/feature_1`
`master`跟踪了`my_github/master`，且当前分支为`master`分支


那么假如我此时想要将master的改变推送到origin服务器的master分支上：
```
$ git checkout master//切换到master分支
...
$ git branch -u origin/master//将当前分支跟踪origin/master
Branch 'master' set up to track remote branch 'master' from 'origin'.
```
之后就可以执行git add和git commit了
现在再查看一下本地和远程的分支关系：
```
$ git branch -vv
  develop   08775f9 [origin/develop] develop
  feature_1 b41865d [origin/feature_1] feature_1
* master    1399706 [origin/master] init commit
```
master已经跟踪了origin/master了
