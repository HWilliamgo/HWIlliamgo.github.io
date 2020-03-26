DialogFragment在显示之后如何从屏幕消除？
1. 直接按手机的返回键
2. 自己的代码里处理 ，调用DialogFragment的对象的dismiss()；
3. 从外部处理
```
MyDialogFragment dialog=(MyDialogFragment)getFragmentManager.findFragmentByTag();
dialog.dismiss();
//摸索了我好久好久，各种查资料，最后自己试出来了，妈的
```
