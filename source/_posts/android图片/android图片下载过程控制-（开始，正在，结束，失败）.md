
``` java
public class ImageDownloadControll {

    private byte[] imageBytes;

    private Handler handler=new Handler(){
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what){
                case 0:
                    Log.d("info","开始下载");
                    break;
                case 1:
                    Log.d("info","正在下载");
                    break;
                case 2:
                    Log.d("info","下载结束");
                    break;
                case 3:
                    Log.d("info","下载失败");
                    break;
            }

        }
    };
    public void download(ImageView imageView,String url){
        try {
            URL url1=new URL(url);
            HttpURLConnection connection= (HttpURLConnection) url1.openConnection();
            connection.setRequestMethod("GET");
            connection.setReadTimeout(10*1000);
            //开始下载
            sendMessageByWhat(0);
            InputStream is=connection.getInputStream();
            ByteArrayOutputStream bos=new ByteArrayOutputStream();
            byte[] bufferBytes=new byte[1024];
            int length=-1;
            while ((length=is.read(bufferBytes))!=-1){
                //正在下载
                sendMessageByWhat(1);
                bos.write(bufferBytes,0,length);
            }
            imageBytes=bos.toByteArray();
            sendMessageByWhat(2);
            //下载结束
            is.close();
            bos.close();
        } catch (IOException e) {
            //下载失败
            sendMessageByWhat(3);
            e.printStackTrace();
        }
    }
    private void sendMessageByWhat(int what){
        Message msg=new Message();
        msg.what=what;
        handler.sendMessage(msg);
    }
}
```
