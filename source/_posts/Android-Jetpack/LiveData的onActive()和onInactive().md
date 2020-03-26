> 看文档的时候对LiveData()的onInactive()回调产生了疑问，遂看源码，记录之。

### 1. 关系

![](https://upload-images.jianshu.io/upload_images/7177220-e1ef03486cbaad2e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

概述：LiveData和Observer互相持有，LiveData将其持有的数据的变动，转发给所有注册过来的Observer。

### 2. onActive 和 onInactive()

```java
/**
 * Called when the number of active observers change to 1 from 0.
 * <p>
 * This callback can be used to know that this LiveData is being used thus should be kept
 * up to date.
 */
protected void onActive() {
}
/**
 * Called when the number of active observers change from 1 to 0.
 * <p>
 * This does not mean that there are no observers left, there may still be observers but their
 * lifecycle states aren't {@link Lifecycle.State#STARTED} or {@link Lifecycle.State#RESUMED}
 * (like an Activity in the back stack).
 * <p>
 * You can check if there are observers via {@link #hasObservers()}.
 */
protected void onInactive() {
}
```

这里对`onInactive()`方法的注释产生了疑问。

因为他与`onActive()`的注释并不完全对应，而是多了一句话：

这不表示该LiveData没有任何Observer了，任然有可能有LiveData，只是他们的状态不是Lifecycle.State#STARTE或Lifecycle.State#RESUMED，就像一个Activity在回退栈中。

### 3. LiveData.ObserverWrapper（翻译：观察者封装器）

这两个回调只在一个方法中被调用，即`LiveData.ObserverWrapper`这个抽象非静态内部类。

```java
private abstract class ObserverWrapper {
    final Observer<T> mObserver;
	//当前是否活跃状态。
    boolean mActive;
    int mLastVersion = START_VERSION;
    ObserverWrapper(Observer<T> observer) {
        mObserver = observer;
    }
    abstract boolean shouldBeActive();
    boolean isAttachedTo(LifecycleOwner owner) {
        return false;
    }
    void detachObserver() {
    }
    
    //Observer的状态改变回调
    void activeStateChanged(boolean newActive) {
		//如果新的状态等于原有的状态，直接返回。
        if (newActive == mActive) {
            return;
        }
        // immediately set active state, so we'd never dispatch anything to inactive
        // owner
        
        //新状态赋值给旧状态
        mActive = newActive;
        //原先是否是活跃状态，取决于LiveData的mActiveCount
        boolean wasInactive = LiveData.this.mActiveCount == 0;
        //根据新状态，修改LiveData的mActiveCount
        LiveData.this.mActiveCount += mActive ? 1 : -1;
        //如果原先是非活跃，而新的状态是活跃
        if (wasInactive && mActive) {
            onActive();//1
        }
        //如果原先是活跃，而新的状态是非活跃
        if (LiveData.this.mActiveCount == 0 && !mActive) {
            onInactive();//2
        }
        //如果新的状态是活跃，那么发射value，即会调用上层设置的onChanged()回调。
        if (mActive) {
            dispatchingValue(this);
        }
    }
}
```

通过以上叙述可知，`onActive()`和`onInactive()`都来自Observer的`activeStateChange()`，这个`activeStateChange()`方法不是给开发者调用的，他自己也不会调用自己，他由Observer所注册到的LiveData在适当的时机调用。

### 4. LiveData.ObserverWrapper#activeStateChange()

![](https://upload-images.jianshu.io/upload_images/7177220-db2c2c73986225e2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

有四个地方会调用。

1. considerNotify

   ``` java
   private void considerNotify(ObserverWrapper observer) {
       if (!observer.mActive) {
           return;
       }
   	//调用observer的shouldBeActive（）方法确认当前observer是否是活跃的
       if (!observer.shouldBeActive()) {
           observer.activeStateChanged(false);
           return;
       }
       if (observer.mLastVersion >= mVersion) {
           return;
       }
       observer.mLastVersion = mVersion;
       //noinspection unchecked
       observer.mObserver.onChanged((T) mData);
   }
   ```

   此处的observer的shouldBeActive()的实现稍后分析。

2. 来自observeForever()，永久注册，注册了的观察者不再受制于`LifecycleOwner`的自动解除注册机制。

3. 移除观察者，都被移除了，当然这个Observer就不再活跃了。

4. 来自`LifecycleBoundObserver`。但是注意，此处调用的参数依然是借助了shouldBeActive()方法，可以说第一点很像。



### 5. LifecycleBoundObserver（翻译：生命周期绑定了的观察者）

``` java
//1. 继承ObserverWrapper------------>对Observer的进一步封装
//2. 实现GenericLifecycleObserver--->具有生命周期感知能力
class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
    @NonNull final LifecycleOwner mOwner;
    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<T> observer) {
        super(observer);
        mOwner = owner;
    }
    //借助LifecycleOwner的获取当前状态的方法来实现了shouldBeActive()
    //用isAtLeast(STARTED)，例如只有在Activity的onStart()和onStop()回调之间，这个Observer
    //才是活跃的。
    @Override
    boolean shouldBeActive() {
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }
    @Override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            //绑定的LifecycleOwner是DESSTROYED，就移除自己（观察者）。这样LiveData的观察者
            //就一个一个全部自己把自己从LiveData移除了。
            removeObserver(mObserver);
            return;
        }
        //借助LifecycleOwnwer的onStateChanged回调调用activeStateChanged()
        activeStateChanged(shouldBeActive());
    }
    @Override
    boolean isAttachedTo(LifecycleOwner owner) {
        return mOwner == owner;
    }
    @Override
    void detachObserver() {
        mOwner.getLifecycle().removeObserver(this);
    }
}
```

通过这个LifecycleBoundObserver的shouldBeActive()方法的实现，解答了LiveData的onInactive()方法的注释了。回到ObserverWrapper的activeStateChanged()方法看下：

```java
void activeStateChanged(boolean newActive) {
    if (newActive == mActive) {
        return;
    }

    mActive = newActive;
    boolean wasInactive = LiveData.this.mActiveCount == 0;
    //此处，每当一个Observer是非活跃的时候，mActiveCount就-1。
    LiveData.this.mActiveCount += mActive ? 1 : -1;
    if (wasInactive && mActive) {
        onActive();
    }
    //当减到最后一个,mActiveCount=0了，就调用onInactive()了。
    if (LiveData.this.mActiveCount == 0 && !mActive) {
        onInactive();
    }
    if (mActive) {
        dispatchingValue(this);
    }
}
```

而此处引起activeStateChanged（false）调用的原因，则可能是因为绑定的LifecycleOwner的生命周期不再是STARTED或者RESUMED了。

### 6. LiveData的mVersion属性

这里有一个问题需要注意，就是mVersion这个属性的作用。

疑问：

activeStateChanged（）会在每当LifecycleOwner对象的生命状态发生改变时都调用，那么如果是活跃状态，就会走到dispatchingValue(this)这个方法，就会去回调到最上层开发者设置的onChanged()回调？

这样可不太好啊，每次Activity从后台返回，都会调用一下onChanged()吗？事实不是这样的，通过打断点debug，从后台返回虽然改变了Activity生命状态，但是没有回调onChanged()。这符合Observer的onChanged()方法的注释

![](https://upload-images.jianshu.io/upload_images/7177220-a686064513976af9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

只有data改变了，才会回调。

那么明明因为LifecycleOwner的生命状态改变而调用了-->activeStateChanged()--->dispatchingValue()--->considerNotify()。为什么不回调onChanged()?

``` java
private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
	//对比observer的版本和LiveData的版本，如果LiveData的版本不大于Observer,
    //返回，不再调用onChanged（）
    if (observer.mLastVersion >= mVersion) {
        return;
    }
	//否则，让Observer的版本与LiveData同步，接着调用onChanged()
    observer.mLastVersion = mVersion;
    //noinspection unchecked
    observer.mObserver.onChanged((T) mData);
}
```

而LiveData的mVersion属性的惟一的写入的地方是：

``` java
@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;//自增
    mData = value;
    dispatchingValue(null);
}
```

即mVersion属性保证了：设置了一次value，调用一次onChanged()的正确逻辑。
