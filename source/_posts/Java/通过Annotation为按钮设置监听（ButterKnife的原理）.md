通过注解来为按钮设置监听，类似于黄油匕首（butterknife）。

###步骤：
![](http://upload-images.jianshu.io/upload_images/7177220-b7b8b87ead228acd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

①
```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
//注意这里两个元注解，RUNTIME保证运行时，保证可以被反射，FIELD表示目标是成员变量
public @interface ListenerFor {
    Class<?extends ActionListener> listener();
}
```
②
```
public class ActionListenerInstaller {
    public static void proccessAnnotations(Object object){
        try {
            Class cl=object.getClass();
            for (Field f:cl.getDeclaredFields()){
                f.setAccessible(true);
                ListenerFor listenerFor=f.getAnnotation(ListenerFor.class);
                //获取成员变量f的值
                Object fieldObject=f.get(object);
                if (listenerFor!=null&&fieldObject!=null&&
                        fieldObject instanceof AbstractButton){
                    //获取listenerFor里的元数据（是Class<?extends ActionListener>）
                    Class<?extends ActionListener> listenerClass=listenerFor.listener();
                    ActionListener actionListener=listenerClass.newInstance();
                    AbstractButton button= (AbstractButton) fieldObject;
                    button.addActionListener(actionListener);
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```
③
```
class OKListener implements ActionListener{
    @Override
    public void actionPerformed(ActionEvent e) {
        JOptionPane.showMessageDialog(null,"点击了确认按钮");
    }
}
class CancelListener implements ActionListener{
    @Override
    public void actionPerformed(ActionEvent e) {
        JOptionPane.showMessageDialog(null,"点击了取消按钮");
    }
}
public class MainTest {
    private JFrame mainWin=new JFrame("使用注解绑定事件监听器");
    //使用Annotation为按钮绑定事件监听器：
    @ListenerFor(listener =OKListener.class )
    private JButton okBtn=new JButton("确定");
    @ListenerFor(listener = CancelListener.class)
    private JButton cBtn=new JButton("取消");

    public void init(){
        JPanel jp=new JPanel();
        jp.add(okBtn);
        jp.add(cBtn);
        mainWin.add(jp);
        ActionListenerInstaller.proccessAnnotations(this);
        mainWin.setDefaultCloseOperation(WindowConstants.EXIT_ON_CLOSE);
        mainWin.pack();
        mainWin.setVisible(true);
    }
    public static void main(String args[]) throws Exception {
        new MainTest().init();
    }
}
```

运行：
 ![](http://upload-images.jianshu.io/upload_images/7177220-5ea8a243fefb5749.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/7177220-fd4cf955131b1c96.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/7177220-b75f7d75ec9cba4b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

参考：《疯狂java讲义》

