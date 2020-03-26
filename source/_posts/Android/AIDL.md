参考hongyang的博客：
>[Android aidl Binder框架浅析](http://blog.csdn.net/lmj623565791/article/details/38461079)

#跨应用的进程间通信：
首先要有两个应用:
1、现在Android Studio里创建两个应用，两个module。
2、一个应用做client，一个应用做server。
3、实现client应用调用server应用中的服务。

开始动手：
###创建一个server应用：
![image](http://upload-images.jianshu.io/upload_images/7177220-001f2e1b18d5e81b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Manifest文件：
```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.lovely_solory.super_william.serviceaidl">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <service
            android:name=".CalcService"
            android:enabled="true"
            android:exported="true">
            <intent-filter>
                <action android:name="com.william.solory.service" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
        </service>

        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```

这个service就是我们待会从cilent通过Intent隐式调用的组件，因此我们直接自己指定一个action，记得exported属性要为true，否则外部不可访问。

2、创建.aidl文件:
右键-new-AIDL-AIDL FILE
```
// ICalcAIDL.aidl
package com.lovely_solory.super_william.serviceaidl;

// Declare any non-default types here with import statements

interface ICalcAIDL {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    int add(int x,int y);
    int min(int x,int y);
}
```
定义了两个方法，一个加法一个减法。

Service类:
```
package com.lovely_solory.super_william.serviceaidl;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
import android.os.RemoteException;
import android.util.Log;

public class CalcService extends Service {
    private final  ICalcAIDL.Stub myBinder=new ICalcAIDL.Stub() {
        @Override
        public int add(int x, int y) throws RemoteException {
            return x+y;
        }

        @Override
        public int min(int x, int y) throws RemoteException {
            return x-y;
        }
    };

    public CalcService() {
    }
    private void printLog(String msg){
        Log.d("******",msg);
    }
    @Override
    public void onCreate() {
        super.onCreate();
        printLog("onCreate");
    }

    @Override
    public void onRebind(Intent intent) {
        super.onRebind(intent);
        printLog("onRebind");
    }

    @Override
    public boolean onUnbind(Intent intent) {
        printLog("onUnbind");
        return super.onUnbind(intent);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        printLog("onDestroy");
    }

    @Override
    public IBinder onBind(Intent intent) {
        // TODO: Return the communication channel to the service.
        printLog("onBind");
        return myBinder;
    }
}
```
完成！关于服务端的app的工作做完了。
做了两件事：1、创建AIDL接口   2、规定Service中用这个接口做什么。

###客户端client的app：
![image](http://upload-images.jianshu.io/upload_images/7177220-e4e6a468399862ca.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Manifest:
```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.lovely_solory.super_william.mainapp">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
//没有什么特别的
```
创建客户端的.aidl文件，注意，这里的AIDL的包名要和服务端的一模一样，但是默认的我们鼠标右键创建出来的报名是这个client的包名，我在这里卡了很久，一直出错，运行的时候读取接口内容失败啥的，后面摸索了之后的解决方法：
1、随便建一个AIDL接口。
2、将那个接口和所在的包都删掉。
3、new-package，然后将这个package的名字命名为何服务端里面的一模一样，然后copy,paste服务端里的.aidl文件到路径下，build project。

界面是四个按钮，四个操作：
![image](http://upload-images.jianshu.io/upload_images/7177220-2979ba3c34c70b0f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Activity:
```
public class MainActivity extends AppCompatActivity implements View.OnClickListener{
    private Button bind,unbind,add,min;
    private ICalcAIDL iCalcAIDL;
    private ServiceConnection connection=new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            log("onServiceConnected");
            iCalcAIDL=ICalcAIDL.Stub.asInterface(service);

        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            log("onServiceDisconnected");
            iCalcAIDL=null;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        initView();
    }

    private void initView() {
        bind=findViewById(R.id.btn_bindService);
        unbind=findViewById(R.id.btn_unbindService);
        add=findViewById(R.id.btn_add);
        min=findViewById(R.id.btn_min);
        bind.setOnClickListener(this);
        unbind.setOnClickListener(this);
        add.setOnClickListener(this);
        min.setOnClickListener(this);
    }

    private void log(String msg){
        Log.d("******",msg);
    }
    @Override
    public void onClick(View v){
        switch (v.getId()){
            case R.id.btn_bindService:
                Intent intent=new Intent();
                intent.setAction("com.william.solory.service");
                bindService(intent,connection,BIND_AUTO_CREATE);
                break;
            case R.id.btn_unbindService:
                unbindService(connection);
                break;
            case R.id.btn_add:
                if (iCalcAIDL!=null){
                    try {
                        int result=iCalcAIDL.add(12,12);
                        Toast.makeText(this,"result="+result,Toast.LENGTH_SHORT).show();
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                }else {
                    Toast.makeText(this,"服务被异常终结，请重新绑定服务端",Toast.LENGTH_SHORT).show();
                }
                break;
            case R.id.btn_min:
                if (iCalcAIDL!=null){
                    try {
                        int result=iCalcAIDL.min(50,12);
                        Toast.makeText(this,"result="+result,Toast.LENGTH_SHORT).show();
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                }else {
                    Toast.makeText(this,"服务被异常终结，请重新绑定服务端",Toast.LENGTH_SHORT).show();
                }
                break;
        }
    }
}
```

run起，然后点按钮bindService:
在client中打印：
```
12-03 13:45:04.513 2380-2380/com.lovely_solory.super_william.mainapp D/******: onServiceConnected
```
在server中打印：
```
12-03 13:45:04.513 2393-2393/? D/******: onCreate
12-03 13:45:04.513 2393-2393/? D/******: onBind
```
点一个加法：
![](http://upload-images.jianshu.io/upload_images/7177220-f4f890a625afca53.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#同一个应用的进程间通信：
比上面更简单，在一个应用的service的manifest中将process属性指定一个":remote"或者"任意的字符"。
AIDL接口直接用就行
