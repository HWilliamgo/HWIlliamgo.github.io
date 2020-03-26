# android模拟简单跨进程通信

> 需求：当前进程需要和远程进程通信，调用远程`Service`的`getEntity(int age)`方法，客户端（当前进程）传入参数int age，服务端（远程Service进程）根据得到的参数返回Entity。

## 1. 建立客户端和服务端通信的共同协议：接口

让Entitiy类实现Parcelable接口：

```java
//省略get set方法。
public class Entity implements Parcelable{
    private int age;
    private String name;
    private String country;
    
    public Entity() {
    }
    private Entity(Parcel in) {
        age = in.readInt();
        name = in.readString();
        country = in.readString();
    }

    public static final Creator<Entity> CREATOR = new Creator<Entity>() {
        @Override
        public Entity createFromParcel(Parcel in) {
            return new Entity(in);
        }

        @Override
        public Entity[] newArray(int size) {
            return new Entity[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(age);
        dest.writeString(name);
        dest.writeString(country);
    }
}
```

新建Entity.aidl

```java
// IEntity.aidl
package com.example.admin.servicetrain;

// Declare any non-default types here with import statements
parcelable Entity;

```

新建IEntityInterface.aidl

```java
// IEntityInterface.aidl
package com.example.admin.servicetrain;
import com.example.admin.servicetrain.Entity;//要手动导包
// Declare any non-default types here with import statements

interface IEntityInterface {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    Entity getEntity(int age);
}
```

![](https://upload-images.jianshu.io/upload_images/7177220-f532170aabb228e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2. 建立RemoteService，让他当做远程服务进程。

```java
public class RemoteService extends Service {

    public static final String TAG = "William";
    MyBinder myBinder=new MyBinder();

    @Override
    public void onCreate() {
        super.onCreate();
        Log.d(TAG, "RemoteService onCreate: ");
    }

    @Override
    public boolean onUnbind(Intent intent) {
        Log.d(TAG, "onUnbind: ");
        return super.onUnbind(intent);
    }

    @Override
    public void onDestroy() {
        Log.d(TAG, "onDestroy: ");
        super.onDestroy();
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        Log.d(TAG, "RemoteService onBind: ");
        return myBinder;
    }
    
    //通过集成IXXXInterface.Stub创建Binder实体类。
    private static class MyBinder extends IEntityInterface.Stub{
        Random random=new Random();
        @Override
        public Entity getEntity(int age) throws RemoteException {
            if (age>30){
                Entity entity=new Entity();
                entity.setAge(random.nextInt(70)+30);
                entity.setCountry("Japan");
                entity.setName("AAA");
                return entity;
            }else {
                Entity entity=new Entity();
                entity.setAge(random.nextInt(30));
                entity.setName("Big");
                entity.setCountry("China");
                return entity;
            }
        }
    }
}
```

在AndroidManifest.xml中注册，process的写法有两种，我选了其中一种。表示RemoteService是当前应用私有进程。

```xml
<service android:name=".RemoteService"
    android:exported="false"
    android:process=":remote">
</service>
```

## 3. 在客户端进程中绑定并通过Binder驱动为我们生成的Binder代理对象来调用RemoteService中的方法。

```java
public class MainActivity extends AppCompatActivity {
    Button btn_bind;
    Button btn_unbind;
    Button btn_getEntity;
    
	//build后，通过sdk的工具将写的.aidl文件转换成了.java接口。
    private IEntityInterface entityInterface;
    //标志位，bind=true表示Service已绑定，bind=false表示Service已解绑。
    private boolean bind;

    private final ServiceConnection connection=new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            //通过asInterface生成Binder代理类对象。（Proxy的对象）
            entityInterface= IEntityInterface.Stub.asInterface(service);
            bind=true;
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            entityInterface=null;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        btn_bind=findViewById(R.id.btn_bind);
        btn_unbind=findViewById(R.id.btn_unbind);
        btn_getEntity=findViewById(R.id.btn_get_number);

        btn_bind.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent=new Intent(MainActivity.this,RemoteService.class);
                bindService(intent,connection,BIND_AUTO_CREATE);
            }
        });

        btn_unbind.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                bind=false;
                unbindService(connection);
            }
        });


        final Random random=new Random();
        btn_getEntity.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (!bind){
                    return;
                }

                try {
                    Log.d(RemoteService.TAG, entityInterface.getEntity(random.nextInt(60)).toString());
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```

![](https://upload-images.jianshu.io/upload_images/7177220-e548db10c6e45bd8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



点击"绑定"，再点击"跨进程获取Entity"，打印：

```verilog
07-26 10:24:01.805 10825-10825/com.example.admin.servicetrain D/William: Entity{age=7, name='Big', country='China'}
07-26 10:24:01.995 10825-10825/com.example.admin.servicetrain D/William: Entity{age=28, name='Big', country='China'}
07-26 10:24:02.205 10825-10825/com.example.admin.servicetrain D/William: Entity{age=26, name='Big', country='China'}
07-26 10:24:02.385 10825-10825/com.example.admin.servicetrain D/William: Entity{age=0, name='Big', country='China'}
07-26 10:24:02.545 10825-10825/com.example.admin.servicetrain D/William: Entity{age=20, name='Big', country='China'}
07-26 10:24:02.735 10825-10825/com.example.admin.servicetrain D/William: Entity{age=63, name='AAA', country='Japan'}
07-26 10:24:02.905 10825-10825/com.example.admin.servicetrain D/William: Entity{age=99, name='AAA', country='Japan'}
```

---

End
