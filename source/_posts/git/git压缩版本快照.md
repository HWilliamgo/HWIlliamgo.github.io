目前有两种方案：
1. `git merge --squash`
2. `git rebase -i HEAD~n`
 ---

## 1. merge
```
git merge --squash <要合并的分支>
```
例子：
本地的情况：
1. 用普通的git merge
![](https://upload-images.jianshu.io/upload_images/7177220-53c5c615c305f866.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当用`git checkout master`+`git merge develop`:
![省略了origin/master](https://upload-images.jianshu.io/upload_images/7177220-3107771c580978e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
此时用`git push`，会将`3,4,5,10,11,12,13`的`版本快照`全部push到服务器去。那么如果要我要压缩，把这么多`版本快照`压缩成一个`版本快照`然后push到服务器呢？
2. 用git merge --squash <要合并的分支>
```
git checkout master
git merge --squash develop
```
![](https://upload-images.jianshu.io/upload_images/7177220-89cdac30f2f2e579.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
此时用`git push`，只会push`3和13`到服务器去，于和之前的区别在于，`13`不再指向`12`，因此不会把12那边的链都push上去。

当然，此时还是会push两个版本快照到服务器，那么咋办：从`origin/master`再checkout一个新的分支，然后用`git merge -squash master`，再push，就可以了。

##2. rebase
```
git rebase -i HEAD~n
```
上述表示选定**当前分支**中包含`HEAD`在内的`n`个最新`版本快照`为对象，并在编辑器中打开。
比如本地版本库为如下：
``` git
* f9bcbae (HEAD -> master) 12346579
* aa82af1 132
* 58f2314 用文本编辑器写的comment
* e5d8212 (origin/master) 123
```
这个时候push的话，会吧`f9bcbae`,`aa82af1`,`58f2314`都push到服务器去。
执行：
```
git rebase -i HEAD~3
```
git打开了文本编辑器，显示：
```
pick 58f2314 用文本编辑器写的comment
pick aa82af1 132
pick f9bcbae 12346579

# Rebase e5d8212..f9bcbae onto e5d8212 (3 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# x, exec <command> = run command (the rest of the line) using shell
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
# .       create a merge commit using the original merge commit's
# .       message (or the oneline, if no original merge commit was
# .       specified). Use -c <commit> to reword the commit message.
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
#	However, if you remove everything, the rebase will be aborted.
#
#	
# Note that empty commits are commented out
```
把后面的两个版本快照的前缀`pick`修改为`fixup`
```
pick 58f2314 用文本编辑器写的comment
fixup aa82af1 132
fixup f9bcbae 12346579
```
保存，关掉文本编辑器。
显示：
```
$ git rebase -i HEAD~3
Successfully rebased and updated refs/heads/master.
```
查看版本库信息：
```
* 2db421a (HEAD -> master) 用文本编辑器写的comment
* e5d8212 (origin/master) 123
```
原本那三个版本快照只剩了一个刚才没有修改pick前缀的版本快照，并且`hash`也改变了，猜测意思是：重新生成了版本快照。
此时也达到了压缩版本快照的目的。push时，只会push`2db421a`上去。
