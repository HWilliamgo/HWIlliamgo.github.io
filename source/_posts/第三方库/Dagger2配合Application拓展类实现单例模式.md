> 需要提前了解Dagger2的`@Singleton` 和 `@Component`依赖。
## 介绍
Android中的单例模式，可以通过继承`Application`类，在onCreate中初始化单例类，然后将单例类的引用通过`getXXX()`发布出去。
Application类提供了天然的单例模式。

但是，如果我们有10个单例类，那么就需要
1. 在`Application`类里面写10个引用，
2. 10个`getXXX()`方法
3. `onCreate()`中10个单例类的初始化

**缺点** 
1. 当需求变动，要更换实现类的时候，要在Application里面做修改
2. `onCreate()`中初始化的10个单例类，并不是马上就要用的，更好的做法是，要用到他们的时候才实例化他们，但是在`onCreate()`中实例化，则必须将10个类一次性提前全部实例化。
## 用Dagger2解决上述两个缺点
1. 依赖Dagger2
2. 将10个单例类从Application类中移除
3. 创建`ApplicationComponent`类，并用`@Component`和`@Singleton`标注，在`ApplicationComponent`中写**带返回值的方法**（用于别的Component类来依赖，或者自己要用的时候调用），不写`void inject(上下文)`方法，因为不需要在`onCreate`中立刻实例化那10个单例类。
4. 创建`ApplicationModule`类，标注`@Module`，写好工厂方法并标注`@Provides`和`@Singleton`，回到`ApplicationComponent`，加上这个`ApplicationModule.class`。
5. 在`onCreate`中实例化`ApplicationComponent`，并用`getApplicationComponent()`，将该`component`对象发布出去。

那么这个时候，在别处任何地方通过该`ApplicationComponent`对象来依赖注入的任何对象，都是单例的。
而且别的`Component`还可以依赖`ApplicationComponent`来拓展自己的依赖注入范围，且通过拓展而来的那些依赖注入来的对象，依然是单例的。
**好处**：
1. 大大缩减了`Application`类的代码量。
2. 当后期需求变动，要改单例类的实现时，只要改`ApplicationModule`中的工厂方法即可。
