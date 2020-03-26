`ViewRootImpl`对象=>`performTraversals()`
```
private void performTraversals() {
//上下近800行代码。
  performMeasure();
  performLayout();
  performDraw();
}
```
```
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
//mView是DecorView对象
  mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);     
}
```
layout和draw同理，下面是measure,layout,draw的机制==>
![measure](https://upload-images.jianshu.io/upload_images/7177220-c9545e3b3f8d3a26.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![layout](https://upload-images.jianshu.io/upload_images/7177220-ef5ccca356973742.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![draw](https://upload-images.jianshu.io/upload_images/7177220-6d62245c67804605.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
