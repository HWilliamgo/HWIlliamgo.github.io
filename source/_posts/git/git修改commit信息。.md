在Android Studio上面直接操作Version Control的reword是最直接的。

用git的方式来：
```
git rebase -i HEAD~1
```
打开了文本编辑器
```
pick 10130de msg

# Rebase da71f75..10130de onto da71f75 (1 command)
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

保存，关闭。
git显示命令：
```
$ git rebase -i HEAD~1
hint: Waiting for your editor to close the file...
```
然后打开了文本编辑器，显示的就是
```
msg
//提示信息
```
这里把msg改成123456798，然后保存，关闭。
```
$ git rebase -i HEAD~1
[detached HEAD 951f653] 123456789
 Date: Fri Jul 20 15:26:26 2018 +0800
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 新建文本文档.txt
Successfully rebased and updated refs/heads/master.
```
这个时候commit信息已经修改，并且也改了版本快照的hash值。
