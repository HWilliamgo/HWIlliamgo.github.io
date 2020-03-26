直接输入`git commit`的时候，或者`git rebase -i`来压缩的时候，都会进入git默认的文本编辑器，根本不会用，然后只能被迫关闭。后得知可以更改文本编辑器，配置如下：
```
git config --global core.editor "\"D:\应用程序\Sublime Text3\sublime_text.exe\""
```
更改后，当git要进入文本编辑器的时候，就会直接打开路径下的sublime text了。

配制后的`C:\Users\admin\.gitconfig`如下:
```
//...略
[core]
	editor = \"D:\\应用程序\\Sublime Text3\\sublime_text.exe\"
```
---
thanks:https://my.oschina.net/lmpudding/blog/730097
