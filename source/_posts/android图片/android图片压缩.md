
1. 质量压缩
2. 尺寸压缩
3. 缩放法压缩（matrix）
4. RGB_565法（比ARGB_888少一半）
5. createScaledBitmap
Bitmap所占用的内存=图片长度 x 图片宽度 x 一个像素点占用的字节数。
 
 一些常用的bitmap压缩方法
``` java
public class Utils {
    /**
     * 采样率压缩
     *
     * @param bitmap
     * @param sampleSize 压缩的倍数 ，要是2的整数倍，否则四舍五入，比如是2，那么压缩后
     *                   就是1/2
     * @return
     */
    public static Bitmap getBitmap(Bitmap bitmap, int sampleSize) {
        BitmapFactory.Options options = new BitmapFactory.Options();
        options.inSampleSize = sampleSize;
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        bitmap.compress(Bitmap.CompressFormat.PNG, 100, bos);
        byte[] bytes = bos.toByteArray();
        return BitmapFactory.decodeByteArray(bytes, 0, bytes.length,options);
    }

    /**
     * 图片质量压缩
     *
     * @param bitmap
     * @param quality 0~100
     * @return
     */
    public static Bitmap getBitmapByQuqlity(Bitmap bitmap, int quality) {
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        bitmap.compress(Bitmap.CompressFormat.JPEG, quality, bos);
        byte[] bytes = bos.toByteArray();
        return BitmapFactory.decodeByteArray(bytes, 0, bytes.length);
    }

    /**
     * Matrix缩放
     * @param bitmap
     * @param scaleWidth 0~1
     * @param scaleHeight 0~1
     * @return
     */
    public static Bitmap getBitmapByMatrix(Bitmap bitmap, float scaleWidth, float scaleHeight) {
        Matrix matrix = new Matrix();
        matrix.postScale(scaleWidth, scaleHeight);
        return Bitmap.createBitmap(bitmap, 0, 0,
                bitmap.getWidth(), bitmap.getHeight(), matrix, true);
    }

    /**
     * 按图片的格式配置压缩
     * @param path 本地图片路径
     * @param config ALPHA_8 , RGB_565, ARGB_4444, ARGB_8888
     * @return
     */
    public static Bitmap getBitmapByFormatConfig(String path,Bitmap.Config config){
        BitmapFactory.Options options=new BitmapFactory.Options();
        options.inPreferredConfig=config;
        return BitmapFactory.decodeFile(path,options);
    }

    /**
     * 指定宽度和高度进行压缩
     * @param bitmap
     * @param concreteWidth 指定的具体宽度
     * @param concreteHeight  指定的具体高度
     * @return
     */
    public static Bitmap getBitmapBySize(Bitmap bitmap,int concreteWidth,int concreteHeight){
        return Bitmap.createScaledBitmap(bitmap,concreteWidth,concreteHeight,true);
    }

    /**
     * 更改图片格式的压缩
     * @param bitmap
     * @param compressFormat JPEG,PNG,WEBP
     * @return
     */
    public static Bitmap getBitmapByFormat(Bitmap bitmap,Bitmap.CompressFormat compressFormat){
        ByteArrayOutputStream bos=new ByteArrayOutputStream();
        bitmap.compress(compressFormat,100,bos);
        byte[] bytes=bos.toByteArray();
        return BitmapFactory.decodeByteArray(bytes,0,bytes.length);
    }

}
```
