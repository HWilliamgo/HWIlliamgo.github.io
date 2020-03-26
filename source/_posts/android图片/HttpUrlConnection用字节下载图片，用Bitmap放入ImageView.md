
``` java
public class MainActivity extends AppCompatActivity {
    private ImageView iv_img;
    private byte[] pics;
    private MyHandler handler;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        handler=new MyHandler(this);
        initView();
    }
    private void initView() {
        iv_img = findViewById(R.id.iv_img);
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    URL urL = new URL("https://www.baidu.com/img/superlogo_c4d7df0a003d3db9b65e9ef0fe6da1ec.png");
                    HttpURLConnection httpURLConnection = (HttpURLConnection) urL.openConnection();
                    httpURLConnection.setRequestMethod("GET");
                    httpURLConnection.setReadTimeout(10000);
                    if (httpURLConnection.getResponseCode() == 200) {
                        InputStream inputStream = httpURLConnection.getInputStream();
                        ByteArrayOutputStream bos = new ByteArrayOutputStream();
                        byte[] bytes = new byte[1024];
                        int length = -1;
                        while ((length = inputStream.read(bytes)) != -1) {
                            bos.write(bytes, 0, length);
                        }
                        pics = bos.toByteArray();
                        bos.close();
                        inputStream.close();
                        Message message = new Message();
                        message.what = 0;
                        handler.sendMessage(message);
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });
        thread.start();
    }

    private static class MyHandler extends Handler{
        private WeakReference<MainActivity> weakReference;
        MyHandler(MainActivity activity) {
            weakReference=new WeakReference<MainActivity>(activity);
        }
        @Override
        public void handleMessage(Message msg) {
            MainActivity activity=weakReference.get();
            if (activity!=null){
                //handle the message below
                switch (msg.what){
                    case 0:
                        Bitmap bitmap= BitmapFactory.decodeByteArray(activity.pics,0,activity.pics.length);
                        activity.iv_img.setImageBitmap(bitmap);
                        break;
                }
            }
        }
    }
}

```
