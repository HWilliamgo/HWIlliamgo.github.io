
用一个固定大小为5的线程池来实现多图加载：

首先实现Runnble子类，下载图片：
``` java
/**
 * 进行图片下载任务 的线程
 */
public class ImageTask implements Runnable {

    public ImageView imageView;
    private String url;
    private byte[] imageBytes;
    private ImageTaskHandler handler;

    public ImageTask(ImageView imageView, String url) {
        this.imageView = imageView;
        this.url = url;
        handler = new ImageTaskHandler(this);
    }
    @Override
    public void run() {
        try {
            URL url = new URL(this.url);
            HttpURLConnection httpURLConnection = (HttpURLConnection) url.openConnection();
            httpURLConnection.setRequestMethod("GET");
            if (httpURLConnection.getResponseCode() == 200) {
                InputStream is = httpURLConnection.getInputStream();
                ByteArrayOutputStream bos = new ByteArrayOutputStream();
                byte[] bytes = new byte[1024];
                int length = -1;
                while ((length = is.read(bytes)) != -1) {
                    bos.write(bytes, 0, length);
                }
                imageBytes = bos.toByteArray();
                Message msg = new Message();
                msg.what = 0;
                handler.sendMessage(msg);
                bos.close();
                is.close();
            }
        } catch (Exception e) {
            Message msg = new Message();
            msg.what = -1;
            handler.sendMessage(msg);
            e.printStackTrace();
        }
    }

    public String getUrl() {
        return url;
    }

    private static class ImageTaskHandler extends android.os.Handler {
        private WeakReference<ImageTask> weakReference;
        private ImageTask imageTask;

        public ImageTaskHandler(ImageTask imageTask) {
            this.weakReference = new WeakReference<>(imageTask);
            this.imageTask = weakReference.get();
        }

        @Override
        public void handleMessage(Message msg) {
            if (imageTask != null) {
                switch (msg.what) {
                    case 0:
                        Bitmap bitmap = BitmapFactory.decodeByteArray(imageTask.imageBytes, 0, imageTask.imageBytes.length);
                        imageTask.imageView.setImageBitmap(bitmap);
                        ImageLoader.getInstance(imageTask.imageView.getContext()).removeTask(imageTask);
                        break;
                    case -1:
                        Toast.makeText(imageTask.imageView.getContext(), "下载失败", Toast.LENGTH_SHORT).show();
                        ImageLoader.getInstance(imageTask.imageView.getContext()).removeTask(imageTask);
                        break;
                }
            }
        }
    }
}
```

将其和ExecutorService封装：
``` java
/**
 * 图片加载线程队列管理类
 */
public class ImageLoader {
    private volatile static ImageLoader instance;
    //Context的弱引用
    private WeakReference<Context> weakContext;
    //固定线程池
    private ExecutorService executorService;
    //保存下载任务的链表
    private final LinkedList<ImageTask> taskLinkedList;
    private ImageLoader(Context context){
        weakContext=new WeakReference<Context>(context);

        executorService= Executors.newFixedThreadPool(5);

        taskLinkedList=new LinkedList<>();
    }
    public static ImageLoader getInstance(Context context){
        if (instance==null){
            synchronized (ImageLoader.class){
                if (instance==null){
                    instance=new ImageLoader(context);
                }
            }
        }
        return instance;
    }
    public void loadImage(ImageView imageView ,String url){
        //首先判断该url是否已经在下载队列中
        if (isTaskExisted(url,imageView)){
            return;
        }
        ImageTask imageTask=new ImageTask(imageView,url);
        synchronized (taskLinkedList){
            taskLinkedList.add(imageTask);
        }
        executorService.execute(imageTask);
        //添加到下载队列
    }
    /**
     * LinkedList中是否存在此url
     * @param url
     * @return
     */
    private boolean isTaskExisted(String url,ImageView iv){
        if (url==null){
            return true;
        }
        synchronized (taskLinkedList){
            for (int i=0;i<taskLinkedList.size();i++){
                ImageTask imageTask=taskLinkedList.get(i);
                if (imageTask!=null&&url.equals(imageTask.getUrl())&&iv.getId()==imageTask.imageView.getId()){
                    return true;
                }
            }
        }
        return false;
    }
    public void removeTask(ImageTask imageTask){
        synchronized (taskLinkedList){
            taskLinkedList.remove(imageTask);
        }
    }
}
```
