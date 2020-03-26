>参考文章http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2014/1106/1922.html
1. 在用handler的时候如果这么用就一定会发生内存泄漏：
```
public class MainActivity extends AppCompatActivity {

    private Handler myHandler=new Handler(){
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
        }
    };
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

    }
}
```
* 原因：这个myHandler对象是由匿名内部类创建而来的，然后内部类默认持有外部类的引用，假设我们用这个handler发送一个10分钟延迟的消息出去，然后马上关闭当前的activity，那么这个message就会在messageQueue停留10分钟，而messageQueue有这个handler的引用，handler又有了外部类MainActivity的引用，因此MainActivity就算退出了，还是无法被gc回收，造成内存泄漏。

* 解决方法：内部类不行，那么就用静态内部类，静态内部类的好处是静态内部类完全不依赖外部类，也就是没有外部类的引用，也就不会造成内存泄漏！

配合弱引用，代码如下：
```
public class MainActivity extends AppCompatActivity {

    static class MyHandler extends Handler{
        //Activity的弱引用
        private WeakReference<AppCompatActivity> weakReference;
        public MyHandler(AppCompatActivity activity){
            weakReference=new WeakReference<AppCompatActivity>(activity);
        }
        @Override
        public void handleMessage(Message msg) {
            AppCompatActivity activity=weakReference.get();
            //做一个判断，如果Activity已经被gc回收，就不再执行该方法。
            if (activity!=null){
                //handle your message
            }
        }
    }
    private MyHandler handler;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        handler=new MyHandler(this);
        //handler.sendMessage(yourMsg);

    }
}
```
