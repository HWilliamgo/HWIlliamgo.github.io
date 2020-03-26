> 例子来自https://www.jianshu.com/p/24af4c102f62

# 只用@Inject和@Component完成依赖注入：
1. 在gradle添加依赖，用`@Inject`和`@Component`为相应的字段和类打上注解
2. build project
3. Dagger2会在`app.build.generated.source.apt.debug.com.text.包名`生成代码。
   1. `DaggerXXXComponent.java`
   2. `XXX_MembersInjector.java`
   3. `XXX_Factory.java`

举一个例子：
MainAcitvity对象依赖Pot对象，Pot对象依赖Rose对象。
1. 用`@Inject`标注要被实例化的引用（在这里是`MainActivity`中的`Pot pot`字段，`Pot`中的`Rose rose`字段）**<Dagger2会为他们生成一个注入器类Injector>**
2. 用`@Inject`标注要被实例化的类的构造方法（在这里是`Pot`类和`Rose`类要被实例化）**<Dagger2会为Pot和Rose生成对应的工厂类>**
3. 用`@Component`标注一个接口**<Dagger2会为其生成XXXComponent类>**
```
public class MainActivity extends AppCompatActivity {
    private static final String TAG = "MainActivity";

    @Inject
    public Pot pot;//MainActivity依赖Pot对象

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```
```
public class Pot {

    private Rose rose;//Pot对象依赖Rose对象

    @Inject
    public Pot(Rose rose){
        this.rose=rose;
    }

    public String show(){
        return rose.whisper();
    }
}
```
```
public class Rose {
    //Rose对象没有任何依赖
    @Inject
    public Rose() {
    }

    public String whisper(){
        return "热恋";
    }
}
```
```
@Component
public interface MainActivityComponent {
    void inject(MainActivity activity);
}
```
点击build project，生成了：
![](https://upload-images.jianshu.io/upload_images/7177220-e3046a306f48b02e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
现在可以使用：
```
public class MainActivity extends AppCompatActivity {
    private static final String TAG = "MainActivity";

    @Inject
    public Pot pot;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        
        //此时pot还指向null，不可用
        DaggerMainActivityComponent.create().inject(this);//实现pot的实例化
        //此时pot已经指向对象，可用
    }
}
```
![image.png](https://upload-images.jianshu.io/upload_images/7177220-e251c2c7ab62b9d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在分析Dagger2生成的四个类
`DaggerMainActivityComponent`
`MainActivity_MembersInjector`
`Pot_Factory`
`Rose_Factory`
### Pot_Factory和Rose_Factory
`XXX_Factory implements Fatory<XXX>`
`Factory<XXX> implements Provider<XXX>`
也就是工厂类的最终抽象是`Provider`
![image.png](https://upload-images.jianshu.io/upload_images/7177220-8e357281c58cbefb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
public enum Rose_Factory implements Factory<Rose> {
  INSTANCE;

  @Override
  public Rose get() {//get()方法返回泛型<Rose>实例，用来生成对象
    return new Rose();
  }

  public static Factory<Rose> create() {//create()方法用来生成工厂对象
    return INSTANCE;
  }
}
```
```
public final class Pot_Factory implements Factory<Pot> {
//因为Pot类的构造方法有参数，参数对象要提前Pot对象实例化，因此持有一个参数对象的类的工厂类，
//用来创建参数对象。
  private final Provider<Rose> roseProvider;

  public Pot_Factory(Provider<Rose> roseProvider) {
    assert roseProvider != null;
    this.roseProvider = roseProvider;
  }

  @Override
  public Pot get() {//get()方法返回泛型<Pot>实例，用来生成对象
    return new Pot(roseProvider.get());
  }
  //create()方法用来生成工厂对象
  public static Factory<Pot> create(Provider<Rose> roseProvider) {
    return new Pot_Factory(roseProvider);
  }
}
```


### MainActivity_MembersInjector
由MainActivity的引用上的@Inject注解生成
```
@Inject
public Pot pot;
```
![](https://upload-images.jianshu.io/upload_images/7177220-c2ec45a6d0d2fe3c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

代码如下
```
public final class MainActivity_MembersInjector implements MembersInjector<MainActivity> {
//持有Pot的工厂类，用来生成Pot对象
  private final Provider<Pot> potProvider;

  public MainActivity_MembersInjector(Provider<Pot> potProvider) {
    assert potProvider != null;
    this.potProvider = potProvider;
  }

//静态方法供外部调用，用来创建这个注入器
  public static MembersInjector<MainActivity> create(Provider<Pot> potProvider) {
    return new MainActivity_MembersInjector(potProvider);
  }

//依赖注入的直接入口，将传入的MainActivity对象的引用变量，用工厂类生成其实例,
//实现依赖注入。
  @Override
  public void injectMembers(MainActivity instance) {
    if (instance == null) {
      throw new NullPointerException("Cannot inject members into a null reference");
    }
    instance.pot = potProvider.get();//用工厂类对象创建对象。
  }
}
```

### DaggerMainActivityComponent
![](https://upload-images.jianshu.io/upload_images/7177220-c8e1b0825b152a96.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
/**
 * 持有工厂类和注入器类的引用。
 * 注入器的实例化需要工厂类对象作为参数。
 * 因此Component类连接工厂类和注入器类，最后通过注入器类来完成工作。
 * 
 * 
 * inject方法直接调用注入器的injectMembers(),注入器中的方法才是注入的直接入口
 * 
 */
public class DaggerMainActivityComponent {
    public final class DaggerMainActivityComponent implements MainActivityComponent {
        private Provider<Pot> potProvider;//工厂类

        private MembersInjector<MainActivity> mainActivityMembersInjector;//注入器

        private DaggerMainActivityComponent(com.test.traindagger.DaggerMainActivityComponent.Builder builder) {
            assert builder != null;
            initialize(builder);
        }

        public static com.test.traindagger.DaggerMainActivityComponent.Builder builder() {
            return new com.test.traindagger.DaggerMainActivityComponent.Builder();
        }

        public static MainActivityComponent create() {
            return builder().build();
        }

        @SuppressWarnings("unchecked")
        private void initialize(final com.test.traindagger.DaggerMainActivityComponent.Builder builder) {
            //实例化工厂类
            this.potProvider = Pot_Factory.create(Rose_Factory.create());
            //依赖工厂类来实例化注入器
            this.mainActivityMembersInjector = MainActivity_MembersInjector.create(potProvider);
        }

        //就是我们在使用的时候用的inject方法
        //调用注入器的injectMembers（上下文）来实现注入
        @Override
        public void inject(MainActivity activity) {
            mainActivityMembersInjector.injectMembers(activity);
        }

        public static final class Builder {
            private Builder() {}

            public MainActivityComponent build() {
                return new com.test.traindagger.DaggerMainActivityComponent(this);
            }
        }
    }
}
```
# 加入@Module和@Provide
使用场景:
1. 无法为第三方库中的类中的构造方法去加上`@Inject`，也就无法通过这种方式让Dagger2为其生成工厂类
2. 当使用了依赖倒置，构造器上的参数就是抽象的（接口或抽象类），Dagger2无法通过构造函数上的`@Inject`将其参数中的接口实例化，因为他并不知道实现类是谁。

比如**场景1**：将上面的`Rose`类，抽象出来一个`Flower`类，并将`Pot`类中的所有`Rose`引用替换为`Flower`引用，因此`Rose`在这里实现了依赖倒置，那么Dagger2无法通过`Rose`类的构造方法为`Rose`类生成工厂类，而`Pot`类没有使用依赖倒置，因此`Pot`类的构造方法依然可以使用`@Inject`来让Dagger2为`Pot`生成工厂类。
```
public class Pot {

    private Flower flower;//将Rose引用替换为了Flower，依赖倒置

    @Inject
    public Pot(Flower flower){
        this.flower=flower;
    }

    public String show(){
        return flower.whisper();
    }
}
```
```
public class Rose extends Flower{
//原本有@Inject，现在去掉了这里的@InJect，因为Dagger2无法定位到这个类的构造函数，
//这个注解留着也没有用
//不过不去掉也不会出错，依然可以通过编译，
//因为Dagger2已经不是从这里生成Rose的工厂方法了
    public Rose(){
    }
    public String whisper(){
        return "热恋";
    }
}
```
在这个场景下，工厂类的生成就和这个类没有什么关系了，那么就要通过另一种方式生成工厂类，就是通过重新创建一个Module类，由他来创建工厂类。
```
@Module//@表示这个类是一个创建工厂类的入口，可以说是工厂类的工厂类。
public class FlowerModule {
    @Provides//标注这个方法作为工厂方法，生成Flower对象。
    Flower provideFlower(){
        //项目需求变动后，要改Flower接口的实现类的时候，就改这里，比如return new Lily();
        return new Rose();
    }
}
```
再为Component接口的注解添加参数
```

@Component(modules = FlowerModule.class)
public interface MainActivityComponent {
    void inject(MainActivity activity);
}
```
现在build project，多出了这个类
```
public final class FlowerModule_ProvideFlowerFactory implements Factory<Flower> {
    private final FlowerModule module;//持有一个Module引用，通过他来实例化泛型<Flower>

    public FlowerModule_ProvideFlowerFactory(FlowerModule module) {
        assert module != null;
        this.module = module;
    }

    //调用module.provideXXX()来返回实例
    @Override
    public Flower get() {
        return Preconditions.checkNotNull(
                module.provideFlower(), "Cannot return null from a non-@Nullable @Provides method");
    }

    //创建自己
    public static Factory<Flower> create(FlowerModule module) {
        return new FlowerModule_ProvideFlowerFactory(module);
    }
}
```
而DaggerMainActivityComponent的变化如下：
```
public final class DaggerMainActivityComponent implements MainActivityComponent {
    private Provider<Flower> provideFlowerProvider;//新加入的创建Flower的工厂类

    private Provider<Pot> potProvider;

    private MembersInjector<MainActivity> mainActivityMembersInjector;

    private DaggerMainActivityComponent(com.test.traindagger.DaggerMainActivityComponent.Builder builder) {
        assert builder != null;
        initialize(builder);
    }

    public static com.test.traindagger.DaggerMainActivityComponent.Builder builder() {
        return new com.test.traindagger.DaggerMainActivityComponent.Builder();
    }

    public static MainActivityComponent create() {
        return builder().build();
    }

    @SuppressWarnings("unchecked")
    private void initialize(final com.test.traindagger.DaggerMainActivityComponent.Builder builder) {
        //通过FlowerModule作为参数创建工厂类
        this.provideFlowerProvider = FlowerModule_ProvideFlowerFactory.create(builder.flowerModule);

        this.potProvider = Pot_Factory.create(provideFlowerProvider);

        this.mainActivityMembersInjector = MainActivity_MembersInjector.create(potProvider);
    }

    @Override
    public void inject(MainActivity activity) {
        mainActivityMembersInjector.injectMembers(activity);
    }

    public static final class Builder {
        private FlowerModule flowerModule;

        private Builder() {}

        public MainActivityComponent build() {
            if (flowerModule == null) {
                this.flowerModule = new FlowerModule();//实例化Module类，就是工厂类的工厂类
            }
            return new com.test.traindagger.DaggerMainActivityComponent(this);
        }

        public com.test.traindagger.DaggerMainActivityComponent.Builder flowerModule(FlowerModule flowerModule) {
            this.flowerModule = Preconditions.checkNotNull(flowerModule);
            return this;
        }
    }
}
```

build project之后的使用：
```
//此时pot还指向null
    DaggerMainActivityComponent.builder()
            .flowerModule(new FlowerModule())//将这个方法暴露给调用者的意义：让调用者可以更改传入哪个Module来生成工厂类。
            .build()
            .inject(this);
//此时pot已经指向对象
```
