>  关于android的Lifecycle是什么和怎么用就不说了，这篇写的很好了：
>
> [Android官方架构组件:Lifecycle详解&原理分析](https://blog.csdn.net/mq2553299/article/details/79029657)

## 概述

1. 概述：Lifecycle方案其实就是把AppCompatActivity和Fragment(support包下的)的声名周期回调转发出去，本质上是基于一个观察者模式，将观察者注册到有生命周期的组件，然后有生命周期的组件在他们的生命周期回调时，把事件转发给每一个观察者，这样观察者就收到了AppCompatActivity或Fragment的生命周期回调，也就具有了生命周期感知能力。
2. 需要了解：
  1. Lifecycle是什么和怎么用
  2. 创建一个没有视图的Fragment，由于该Fragment和创建他的Activity的生命周期是一致的，因此该Fragment也具有Activtiy生命周期感知能力。

## Fragment如何转发生命周期

1. 打开`android.support.v4.app`包下的Fragment

	```java
	public class Fragment implements ComponentCallbacks, OnCreateContextMenuListener, LifecycleOwner,ViewModelStoreOwner {
	    //...
	    LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
	
	    @Override
	    public Lifecycle getLifecycle() {
	        return mLifecycleRegistry;
	    }
	    //...
	}
	```
	
	Fragment实现了LifecycleOwner，在该接口唯一的抽象方法中返回了
	
	`LifecycleRegistry`，中文翻译过来是**生命周期注册处**，他是官方对Lifecycle抽象类的默认实现，而他就是一个被观察者，是一个`Observable`。

2. 接着看`performCreate()`和`performStart()`

	```java
	void performCreate(Bundle savedInstanceState) {
	    if (mChildFragmentManager != null) {
	        mChildFragmentManager.noteStateNotSaved();
	    }
	    mState = CREATED;
	    mCalled = false;
	    onCreate(savedInstanceState);
	    mIsCreated = true;
	    if (!mCalled) {
	        throw new SuperNotCalledException("Fragment " + this
	                + " did not call through to super.onCreate()");
	    }
	    //在这里，将Lifecycle.Event.ON_CREATE事件转发给所有生命周期观察者。
	    mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
	}
	```
	
	```java
	void performStart() {
	    if (mChildFragmentManager != null) {
	        mChildFragmentManager.noteStateNotSaved();
	        mChildFragmentManager.execPendingActions();
	    }
	    mState = STARTED;
	    mCalled = false;
	    onStart();
	    if (!mCalled) {
	        throw new SuperNotCalledException("Fragment " + this
	                + " did not call through to super.onStart()");
	    }
	    if (mChildFragmentManager != null) {
	        mChildFragmentManager.dispatchStart();
	    }
	    //同理。
	    mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_START);
	}
	```
	
	至于`onResume()`，`onPause()`,`onStop`和`onDestory`，都是一样的，都会调用`mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.XXX)`把生命周期转发给观察者。

3. 看下`handleLifecycleEvent()`

	```java
	public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
	    State next = getStateAfter(event);
	    moveToState(next);
	}
	```
	
	点进moveToState(next);
	
	```java
	private void moveToState(State next) {
	    if (mState == next) {
	        return;
	    }
	    mState = next;
	    if (mHandlingEvent || mAddingObserverCounter != 0) {
	        mNewEventOccurred = true;
	        // we will figure out what to do on upper level.
	        return;
	    }
	    mHandlingEvent = true;
	    sync();//关注这个方法
	    mHandlingEvent = false;
	}
	```
	
	点进sync();
	
	```java
	private void sync() {
	    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
	    if (lifecycleOwner == null) {
	        Log.w(LOG_TAG, "LifecycleOwner is garbage collected, you shouldn't try dispatch "
	                + "new events from it.");
	        return;
	    }
	    while (!isSynced()) {
	        mNewEventOccurred = false;
	        // no need to check eldest for nullability, because isSynced does it for us.
	        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
	            backwardPass(lifecycleOwner);//点进这里
	        }
	        Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
	        if (!mNewEventOccurred && newest != null
	                && mState.compareTo(newest.getValue().mState) > 0) {
	            forwardPass(lifecycleOwner);//或者点进这里
	        }
	    }
	    mNewEventOccurred = false;
	}
	```
	
	接着点进backwardPass或者forwardPass。
	
	```java
	private void backwardPass(LifecycleOwner lifecycleOwner) {
	    Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
	            mObserverMap.descendingIterator();
	    while (descendingIterator.hasNext() && !mNewEventOccurred) {
	        Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
	        ObserverWithState observer = entry.getValue();
	        while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
	                && mObserverMap.contains(entry.getKey()))) {
	            Event event = downEvent(observer.mState);
	            pushParentState(getStateAfter(event));
	            observer.dispatchEvent(lifecycleOwner, event);//终于，在这里把事件正式转发给了观察者。
	            popParentState();
	        }
	    }
	}
	```
	
	点进observer.dispatchEvent(...)
	
	```java
	void dispatchEvent(LifecycleOwner owner, Event event) {
	    State newState = getStateAfter(event);
	    mState = min(mState, newState);
	    mLifecycleObserver.onStateChanged(owner, event);//在这里
	    mState = newState;
	}
	```
	
	点进mLifecycleObserver.onStateChanged(owner, event);
	
	```java
	public interface GenericLifecycleObserver extends LifecycleObserver {
	    void onStateChanged(LifecycleOwner source, Lifecycle.Event event);
	}
	```
	
	终于，到了LifecycleObserver了，观察者只需要实现这个接口，然后再Activty或者Fragment里面注册一下，就能接收到一系列的生命周期回调了。

## AppCompatActivity如何转发生命周期

### 1. 找到SupportActivity

AppCompatActivity的继承关系：

```
Context (android.content)
    ContextWrapper (android.content)
        ContextThemeWrapper (android.view)
            Activity (android.app)
                SupportActivity (android.support.v4.app)
                    BaseFragmentActivityApi14 (android.support.v4.app)
                        BaseFragmentActivityApi16 (android.support.v4.app)
                            FragmentActivity (android.support.v4.app)
                                AppCompatActivity (android.support.v7.app)

```

### 2. 分析SupportActivity

```java
public class SupportActivity extends Activity implements LifecycleOwner {
    //...
    private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
    
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
    
    @Override
    @SuppressWarnings("RestrictedApi")
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ReportFragment.injectIfNeededIn(this);//关键点
    }
    
}
```

其实我刚开始找AppCompatActivity的时候是懵逼的，因为从上到下的所有父类，都没有在生命周期里像Fragment一样去直接用`LifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.XXX)`转发生命周期事件的，找了半天都没找到。



后来发现了上图14行代码处`ReportFragment.injectIfNeededIn(this);，原来是他，他用的技巧是文章开头说到的：**创建一个没有视图的Fragment，由于该Fragment和创建他的Activity的生命周期是一致的，因此该Fragment也具有Activtiy生命周期感知能力。创建一个没有视图的Fragment，由于该Fragment和创建他的Activity的生命周期是一致的，因此该Fragment也具有Activtiy生命周期感知能力。**



### 3. ReportFragment.injectIfNeededIn(this)

```java
public class ReportFragment extends Fragment {
	public static void injectIfNeededIn(Activity activity) {
    // ProcessLifecycleOwner should always correctly work and some activities may not extend
    // FragmentActivity from support lib, so we use framework fragments for activities
    	android.app.FragmentManager manager = activity.getFragmentManager();
    	if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
        manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
        // Hopefully, we are the first to make a transaction.
        manager.executePendingTransactions();
    	}
	}
}
```

借助传入的activity，创建一个`ReportFragment`(没有视图的Fragment)。

再看ReportFragment的生命周期回调

```java
@Override
public void onActivityCreated(Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);
    dispatchCreate(mProcessListener);
    dispatch(Lifecycle.Event.ON_CREATE);
}
@Override
public void onStart() {
    super.onStart();
    dispatchStart(mProcessListener);
    dispatch(Lifecycle.Event.ON_START);
}
@Override
public void onResume() {
    super.onResume();
    dispatchResume(mProcessListener);
    dispatch(Lifecycle.Event.ON_RESUME);
}
@Override
public void onPause() {
    super.onPause();
    dispatch(Lifecycle.Event.ON_PAUSE);
}
@Override
public void onStop() {
    super.onStop();
    dispatch(Lifecycle.Event.ON_STOP);
}
@Override
public void onDestroy() {
    super.onDestroy();
    dispatch(Lifecycle.Event.ON_DESTROY);
    // just want to be sure that we won't leak reference to an activity
    mProcessListener = null;
}
```

是的就是这样，借助了这个没有视图的Fragment来把生命周期转发出去。

看下`dispatch(Lifecycle.Event.XXX)`做了什么

```java
private void dispatch(Lifecycle.Event event) {
    Activity activity = getActivity();
    if (activity instanceof LifecycleRegistryOwner) {
        ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
        return;
    }
    if (activity instanceof LifecycleOwner) {
        Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
        if (lifecycle instanceof LifecycleRegistry) {
            ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
        }
    }
}
```

看到了第10行的`((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);`，在这里，把生命周期转发出去。

---

至此，分析结束。我从这个Lifecycle方案中学到的一点是：

无视图Fragment的技巧的优点：

他与创建他的Activity的生命周期一致（让这个Fragment具有生命周期感知能力），可以在不修改原有Activity生命周期代码的情况下，用Fragment来从外部`插入`方法。可拔插，解耦合，非常灵活。
