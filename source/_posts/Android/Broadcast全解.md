
Android系统的app可以从系统或者其他app发送或者接受广播消息。类似于(发布-订阅模式)。

app可以注册来接收特定的广播。当一条广播被发送时，系统自动地将这条广播引导到已经注册来接收那类广播的app。

一般来说，广播可以被用作跨应用的和正常用户流程之外的通信系统。

### 接收广播
app可以通过两种方法来注册接受广播：
1.  manifest文件中注册
	``` xml
	<receiver android:name=".MyBroadcastReceiver"  android:exported="true">
			<intent-filter>
				<action android:name="android.intent.action.BOOT_COMPLETED"/>
				<action android:name="android.intent.action.INPUT_METHOD_CHANGED" />
			</intent-filter>
		</receiver>
	```
	 `intent-filters指明了这个receiver订阅的broadcast actions`

	* 继承BroadcastReceiver并重写onReceive(Context context,Intent intent);
		``` java
		public class MyBroadcastReceiver extends BroadcastReceiver {
				private static final String TAG = "MyBroadcastReceiver";
				@Override
				public void onReceive(Context context, Intent intent) {
					StringBuilder sb = new StringBuilder();
					sb.append("Action: " + intent.getAction() + "\n");
					sb.append("URI: " + intent.toUri(Intent.URI_INTENT_SCHEME).toString() + "\n");
					String log = sb.toString();
					Log.d(TAG, log);
					Toast.makeText(context, log, Toast.LENGTH_LONG).show();
				}
			}
		```
		  
		当接收到广播时，接收器的onReceive会被调用，广播就封装在第二个参数Intent中。
		
		---
		系统的包管理器在安装应用程序时注册这个接收器。接收就成为了这个应用程序的独立入口，这意味着系统可以启动该应用程序并在这个app还未运行时传送广播。
		系统创建一个新的BroadcastReceiver组件对象来处理它收到的每个广播。此对象仅在调用onReceive(Context, Intent)期间有效。一旦代码从此方法返回，系统会认为该组件不再处于活动状态。
2.  context中注册。
	1. 创建BroadcastReceiver对象。：
		``` java
		BroadcastReceiver br = new MyBroadcastReceiver();
		```
	2. 创建一个IntentFilter对象，然后通过调用context对象的registerReceiver(BroadcastReceiver , IntentFilter)来注册接收器。
		``` java
		IntentFilter filter = new IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION);
		filter.addAction(Intent.ACTION_AIRPLANE_MODE_CHANGED);
		this.registerReceiver(br, filter);
		```
	3. 注意：如果要为本地广播接收器注册，应该调用`LocalBroadcastManager.registerReceiver(BroadcastReceiver,IntentFilter)`;
		与发送一条全局广播相比（系统内的所有应用都可能接收到），本地广播的优势：
		1. 你的广播数据不会离开你的app,不用担心隐私问题
		2. 注册的是本地Receiver，别的app不可能给你发送广播了
		3. 相比全局广播，本地广播的sendBroadcast效率更高。
	4. context注册的Receiver的的生命周期：只要那个context对象一直有效，就一直可以接受广播。比如用Activity注册的，只要activity没有destroyed，Receiver一直有效。用Application的context注册的，只要app还在跑，Receiver就一直有效。

---

由于本篇文章本来就是从官方文档学习的，下午看到了一篇翻译的也不错，因此后续内容不接着写了，看这里就行：
http://www.passershowe.com/2017/04/05/Android-BroadcastReceiver/
