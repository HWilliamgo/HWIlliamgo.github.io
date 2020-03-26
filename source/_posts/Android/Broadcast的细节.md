1. 通过Manifest文件注册的BroadcastReceiver
	1. 当发送广播时，不能用LocalBroadcastManager（接收器会接收不到那条广播）。
	2. 不能在任何地方调用context.unregisterReceiver();否则报异常：java.lang.RuntimeException: Unable to start receiver com.solory.myapplication.MyLocalBroadcastReceiver: java.lang.IllegalArgumentException: Receiver not registered: com.solory.myapplication.MyLocalBroadcastReceiver@4a800704。
	3. 用Manifest文件注册的接收器就是全局性质的。
2. 通过context对象来注册的BroadcastReceiver。
	1. 用context.registerReceiver注册，但是用LocalBroadcastManager发送，会导致接收不到广播。
	2. 用LocalBroadcastManager注册，用context注册，接受不到广播
	3. 要用本地广播的话，注册和发送都要用LocalBroadcastManager。
	4. 在onReceive里面开启一个工作线程，Activity在onCreate里面注册接收器，在onDestroy里面解绑接收器。此时退出Activity是结束不了onReceive里面的工作的（因为此时该进程有两个组件，组件Activity被销毁，但是组件BroadcastReceiver还在运行），除非外部手动强制结束该进程，或者系统因为内存紧张，需要强制结束掉该BroadcastReceiver组件（因为和Receiver关联的前台组件已退出，那么Receiver此时位于后台，优先级较低，系统会因为内存紧张而杀死该组件）。
	5. 当执行onReceive()的时候，Receiver被当作一个前台进程，在onReceive()内，该进程是不会被自动清理的，但是如果onRecive中开启了工作线程，onReceive返回后，线程依然执行，但是此时接收器所在进程的优先级就可能在后台（比如换到另一个Activity），那么虽然有线程在工作，但是若内存紧张，还是会将进程结束。因此不建议在onReceive中开启工作线程。如果非要用就用goAsyc()或者JobService,详情见官网。
	6. 官方文档说：本地广播可以在您的应用程序中用作通用的发布/订阅事件总线，而无需系统广播的任何开销。
