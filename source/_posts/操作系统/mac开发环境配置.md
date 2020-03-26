# mac开发环境配置

### 1. 安装Homebrew

1. 安装`xcode command line tool`

   [Xcode-select命令是什么](https://www.jianshu.com/p/bf6aa6f97fcb)

   ``` shell
   xcode command line tool
   ```

2. 安装homebrew

   ```shell
   /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
   
   ```

   注：来自homebrew官网：https://brew.sh/

3. 检查homebrew是否安装完成

   ```shell
   brew --version
   ```

   如果出现如下错误：`-bash: brew: command not found`

   则需要配置环境变量：

   ```bash
   $ cd ~
   $ ls -a  #查看是否存在.bash_profile文件。
   	#如果不存在->
   $ touch .bash_profile #创建.bash_profile
   	#如果存在->
	#执行下一步
   ```
   
   注意，这里是`.bash_profile`文件，名字不要输错了，因为unix系统在开机的时候会加载`.bash_profile`文件。

   

   ---
   
   *题外话，我找了一篇解释[不同的开机脚本的含义](https://www.cnblogs.com/kevingrace/p/8072860.html)的文章，概括如下*
   
| 脚本            | 作用                                                         |
| --------------- | ------------------------------------------------------------ |
| /etc/profile    | 此文件为系统的每个用户设置环境信息,当用户第一次登录时,该文件被执行.并从/etc/profile.d目录的配置文件中搜集shell的设置. |
| /etc/bashrc     | 为每一个运行bash shell的用户执行此文件.当bash shell被打开时,该文件被读取. |
| ~/.bash_profile | 每个用户都可使用该文件输入专用于自己使用的shell信息,当用户登录时,该文件仅仅执行一次!默认情况下,他设置一些环境变量,执行用户的.bashrc文件. |
| ~/.bashrc       | 该文件包含专用于你的bash shell的bash信息,当登录时以及每次打开新的shell时,该该文件被读取 |
| ~/.bash_logout  | 当每次退出系统(退出bash shell)时,执行该文件.                 |

---

   

   继续编`.bash_profile`

   ```shell
   sudo vim .bash_profile  #以root身份来打开并创建.bash.profile
   ```

   输入`i`进入编辑模式，用方向键移动到文件底部，并输入：

   ```shell
   export PATH=/usr/local/bin:$PATH  #为PATH添加/usr/local/bin的路径
   ```

   点击`esc`按键退出编辑模式，再输入`:wq`，回车：保存并退出vim

   ``` shell
   source .bash_profile
   #更新配置后的环境变量
   ```

   此时再输入`brew --version`就可以看到版本信息了。

4. 当用brew install 安装软件的时候，会出现brew一直卡在Updating Homebrew的情况，是因为默认homebrew是自动更新的，可以关闭：

   ``` shell
   vim ~/.bash_profile
   
   # 新增一行
   export HOMEBREW_NO_AUTO_UPDATE=true
   ```

   或者更换更新地址。

   卡住的时候也可以按`controll+c`来关闭这一次的更新。

   



### 2. 安装jdk，配置jdk环境变量

https://www.jianshu.com/p/d26380f47454



### 3. 安装必备软件和工具

1. 让鼠标滚轮和windows保持一致

   mac的鼠标和触摸板的方向有点不舒服，用这个可以调整成触摸板默认，而鼠标反向，达到和windows电脑一致的逻辑。

   https://pilotmoon.com/scrollreverser/

2. 搜狗输入法

   默认的输入法切换中英文需要按`caps lock`，且长按他切换大写。用了这个可以用`shift`来切换中英文，且点击切换大小写，保持和windows一致。

   https://pinyin.sogou.com/mac/

3. 状态栏显示实时网速、内存、cpu等信息

   MenuMeters

4. 类似于ubuntu的在终端打开：go2shell

   https://zipzapmac.com/Go2Shell

### 4. 修改用户名和主机名

1. 修改主机名：

   修改主机名（命令行里面看到的那个）

   > sudo scutil --set HostName <你的主机名>

   修改共享名称（局域网里面看到的那个）

   > sudo scutil --set ComputerName <你的共享名>

2. 修改用户名：

   最好是完全按照apple官方文档上的来：

   [更改 macOS 用户帐户和个人文件夹的名称](https://support.apple.com/zh-cn/HT201548)



### 5. 设置android ndk和sdk环境变量

``` shell
#设置androidsdk环境变量。 用于终端能够直接使用tools和platform-tools中的工具，如adb
export PATH=${PATH}:/Users/HWilliam/Library/Android/sdk/platform-tools
export PATH=${PATH}:/Users/HWilliam/Library/Android/sdk/tools

#设置ndk环境变量
export ANDROID_NDK=/Users/HWilliam/ndk/android-ndk-r10e
export PATH=$ANDROID_NDK:$PATH
```



### 6. 安装gradle

https://gradle.org/install/

``` bash
$ brew install gradle
```



安装gradle自动提示工具:https://github.com/gradle/gradle-completion

``` bash
$ brew install gradle-completion
```




