# git在github上的操作的版本回退

1. 找到你要回退的版本快照的`hashCode`
2. `git reset --hard hashCode` 将本地的head指针指向`hashCode`代表的版本快照。
3. `git push -f`将当前指针强制推送到远程，操作之后github上的版本快照回到原来的样子。（强制push之前一定要再看一眼head指针指向的版本快照是你想要的原来的样子）
