9图的制作在官网和其他博客中都有大量教程，最终可以制作出`XXX.9.png`文件

##　没有用NinePatch:
在自定义View中，直用`Bitmap`和画笔而不配合`NinePatch`类是画不9图的效果的，比如：
```
//R.drawable.image9Patch是制作好的9图
Bitmap bitmap=BitmapFactory.decodeResource(getResources(),R.drawable.image9Patch)
```
```
@Override
protected void onDraw(Canvas canvas) {
   ...创建RectF dst，作为bitmap被画出且自动缩放的矩形区域。
   //直接用canvas画
   canvas.drawBitmap(bitmap,null,dst,null);
}
```
这样是画不出想要的9图的效果的。在目标矩形中，图片依然会被不和谐地拉伸或缩放。

## 用了NinePatch
```
@Override
protected void onDraw(Canvas canvas) {
    ...创建RectF dst，作为bitmap被画出且自动缩放的矩形区域。

    NinePatch np=new NinePatch(bitmap,bitmap.getNinePatchChunk(),null);
    np.draw(canvas,dst);
}
```
借助`NinePatch`这个类，用他来画，最后在目标矩形区域，可以成功地按照9图的方式来缩放。
