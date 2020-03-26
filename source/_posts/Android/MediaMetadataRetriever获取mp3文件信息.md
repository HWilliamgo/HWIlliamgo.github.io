转发：http://blog.csdn.net/xiao5678yun/article/details/53838362
```
MediaMetadataRetriever mmr = new MediaMetadataRetriever();  
String str = getExternalStorageDirectory() + "1.mp3";  
Log.d(TAG, "str:" + str);  
try 
{  
    mmr.setDataSource(str);  
    String title = mmr.extractMetadata(MediaMetadataRetriever.METADATA_KEY_TITLE); 
    Log.d(TAG, "title:" + title);  
    String album = mmr.extractMetadata(MediaMetadataRetriever.METADATA_KEY_ALBUM);  
    Log.d(TAG, "album:" + album);  
    String artist = mmr.extractMetadata(MediaMetadataRetriever.METADATA_KEY_ARTIST);  
    Log.d(TAG, "artist:" + artist);  
    String duration = mmr.extractMetadata(MediaMetadataRetriever.METADATA_KEY_DURATION); // 播放时长单位为毫秒  
    Log.d(TAG, "duration:" + duration);   
    byte[] pic = mmr.getEmbeddedPicture();  // 图片，可以通过BitmapFactory.decodeByteArray转换为bitmap图片
} 
catch (Exception e) 
{  
    e.printStackTrace();  
}
```
