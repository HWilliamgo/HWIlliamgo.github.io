这个控件的两个优点：
1. 更加友好地显示提示
2. 更友好地显示错误信息

更加友好地显示提示：
```
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/rl"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.lovely_solory.super_william.mynotebook.MainActivity">
    <android.support.design.widget.TextInputLayout
        android:id="@+id/username"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="30dp">
        <EditText
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:hint="username"
            android:maxLength="25"
            android:maxLines="1" />
    </android.support.design.widget.TextInputLayout>
    <android.support.design.widget.TextInputLayout
        android:id="@+id/password"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_below="@id/username">
        <EditText
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="20dp"
            android:hint="password"
            android:maxLength="25"
            android:maxLines="1" />
    </android.support.design.widget.TextInputLayout>
    <Button
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_below="@id/password"
        android:layout_marginTop="20dp"
        android:text="login" />
</RelativeLayout>


```
![](http://upload-images.jianshu.io/upload_images/7177220-f4e76472a58ee3b5.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

更加友好地显示错误信息：
这部分需要代码来控制：
```
public class MainActivity extends AppCompatActivity {
    TextInputLayout edit_username, edit_password;
    Button btn;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        edit_username = findViewById(R.id.username);
        edit_password = findViewById(R.id.password);
        btn = findViewById(R.id.btn);

        btn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                String username = edit_username.getEditText().getText().toString();
                String password = edit_password.getEditText().getText().toString();

                if (username.isEmpty()) {
                    edit_username.setErrorEnabled(true);
                    edit_username.setError("请输入正确的用户名格式");
                }
                if (password.isEmpty()) {
                    edit_password.setErrorEnabled(true);
                    edit_password.setError("请输入正确的密码格式");
                }
                if (!username.isEmpty()&&!password.isEmpty())
                {
                    Toast.makeText(MainActivity.this,"登录成功",Toast.LENGTH_SHORT).show();
                }

            }
        });


    }
}
```
效果：
* ![](http://upload-images.jianshu.io/upload_images/7177220-701d18496d75bd66.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
