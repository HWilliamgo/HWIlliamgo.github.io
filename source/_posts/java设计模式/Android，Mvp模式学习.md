---
title: Mvp项目学习
---

：![](http://upload-images.jianshu.io/upload_images/7177220-093073e0a91663a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/300)

今天用我们用MVP模式来完成一个简单的登录界面的实现。
业务需求：
* 用户输入用户名和密码之后，按下登录按钮，跳转到另一个Activity。
	* 这里面隐含的一些含义有：检查用户名和密码（网络访问）
	* 点击登录按钮，对网络返回结果判断并处理（设置监听器）

由于我们这里是简化版本的，所以不写网络访问的代码，用一个延时的handler来模拟就行了。
![image.png](http://upload-images.jianshu.io/upload_images/7177220-d7a80cf13b6f5fe9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/300)

MVP模式的思路是：
* 创建Model接口，View接口，Presenter接口。
* **View接口**：根据业务或者UI设计师给的图片来定制我们的View接口中应该有哪些方法，比如显示进度条，跳转界面，只要用户能操作的或者用户看得到的可交互的东西，都需要考虑是否要写进View接口中。
* **Model接口**：根据View接口中的方法，设计Model接口中的方法，Model接口就是用来对数据进行处理的，UI层的交互数据将会流到这里，比如EditText输入了用户名，那么用户名就经由Presenter的逻辑处理最终来到Model接口，Model接口比如实现了网络访问操作，此时将数据处理了拿到返回结果，再传回给Presenter，经由Presenter传递给View。
* **Presenter接口**：根据View接口中的方法来设计对应的方法，并且方法一般都和Model层进行交互。

---
### View层接口的设计：
那么首先看到界面的图片，我们按照用户和UI交互的几个地方设计方法就行了：
* 点击登录，显示登录进度，用一个progressBar。
* 点击登录可能出现：
	* 用户名错误提示：设置用户名错误提示动画
	* 密码错误提示：设置密码错误提示动画
	* 用户名密码正确，跳转至主界面Activity
那么对应的LoginView接口为：
``` java
public interface ILoginView {
    void showProgressBar();
    void hideProgressBar();
    void setUserNameError();
    void setPasswordError();
    void JumpToActivity();
}
```
那么对应的登录界面的Activity只要实现该接口并正确地实现每个接口所描述的方法就行了。回头在Presenter里面用ILoginView的引用调用对应的方法，即可从Presenter层去改变UI界面。

### Model层接口的设计
Model层是用来接收数据（从View层发送过来的数据），处理数据，并返回数据（返回给View层来展示给用户）的，所以Model接口的方法设计要依照我们要处理的数据。
这里要处理的数据是：---------用户名和密码
此外还要设计监听器，为了便于管理，监听器就放在对应的Model层接口里面
代码：
```java
public interface ILoginModel {
    interface OnLoginListener{
        void onSuccess();
        void onUsernameWrong();
        void onPasswordWrong();
    }
    void login(String username,String password,ILoginModel.OnLoginListener listener);
}
```
实现类
``` java
public class LoginModelImp implements ILoginModel {
    @Override
    public void login(final String username, final String password, final OnLoginListener listener) {
        //此处应该是有网络请求的，比如调用OkHttp或者RxJava什么的。
        new Handler().postDelayed(new Runnable() {
            @Override
            public void run() {
                if (TextUtils.isEmpty(username)){
                    listener.onUsernameWrong();
                    return;
                }
                if (TextUtils.isEmpty(password)){
                    listener.onPasswordWrong();
                    return;
                }
                listener.onSuccess();

            }
        },1500);
    }
}
```
在这里的OnLoginListener调用的每一个方法最终的结果都是去View层去改变UI视图来和用户交互，而Model层不能直接和View层通信，那么只能借助Presenter来实现监听器接口，因为Presenter里面有双方（M和V）的引用，所以可以轻易地调用View层的方法去改变UI。
### Presenter层接口的设计
Presenter层是用来桥接Model层和View层的，所以方法应该用于让M和V之间通信
``` java
public interface ILoginPresenter {
    void onLogin(String username,String password);
    void onViewDestroy();
}
```
实现类
``` java
public class LoginPresenterImp  implements ILoginPresenter,ILoginModel.OnLoginListener{
    private ILoginModel loginModel;
    private ILoginView loginView;

    public LoginPresenterImp(ILoginView loginView){
        this.loginView=loginView;
        this.loginModel=new LoginModelImp();
    }

    @Override
    public void onLogin(String username,String password) {
        if (loginView!=null){
            loginView.showProgressBar();
        }
        loginModel.login(username,password,this);
    }

    @Override
    public void onViewDestroy() {
        //解除View和Presenter的绑定
        loginView=null;
    }
    //----------------------------------------------下面是OnLoginListener
    @Override
    public void onSuccess() {
        loginView.hideProgressBar();
        loginView.JumpToActivity();
    }

    @Override
    public void onUsernameWrong() {
        loginView.setUserNameError();
        loginView.hideProgressBar();
    }

    @Override
    public void onPasswordWrong() {
        loginView.setPasswordError();
        loginView.hideProgressBar();
    }
}
```
---
最后是View层的实现类，即LoginActivity
``` java
public class LoginActivity extends AppCompatActivity implements ILoginView{
    ILoginPresenter loginPresenter;

    ProgressBar progressBar;
    EditText usernameE,passwordE;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        initiateView();
        loginPresenter=new LoginPresenterImp(this);

        findViewById(R.id.loginBtn).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                loginPresenter.onLogin(usernameE.getText().toString(),passwordE.getText().toString());
            }
        });
    }

    private void initiateView() {
        usernameE=findViewById(R.id.username);
        passwordE=findViewById(R.id.password);
        progressBar=findViewById(R.id.loginProgressBar);
    }

    @Override
    public void showProgressBar() {
        progressBar.setVisibility(View.VISIBLE);
    }

    @Override
    public void hideProgressBar() {
        progressBar.setVisibility(View.INVISIBLE);
    }

    @Override
    public void setUserNameError() {
        usernameE.setError("用户名错误");
    }

    @Override
    public void setPasswordError() {
        passwordE.setError("密码错误");
    }

    @Override
    public void JumpToActivity() {
        startActivity(new Intent(this, Main2Activity.class));
        this.finish();
    }

    @Override
    protected void onDestroy() {
        loginPresenter.onViewDestroy();
        super.onDestroy();
    }
}
```
持有Presenter的引用，调用Presenter中的方法，而Presenter中又有Model的引用，间接地将数据传输给Model层去处理

---
接下来模拟一次正常的用户操作，输入用户名和密码并点击登录，那么事件发生的流程图如下：

![](http://upload-images.jianshu.io/upload_images/7177220-6046df2a6e861014.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

结束

---
github项目地址：[MvpFirst](https://github.com/William619499149/MvpFirst/tree/master)
