结论：view.post()方法在整个view树的performMeasure, performLayout, performDraw执行完后，才被主线程轮询到，才得到执行。

---

基于android sdk-23的源码分析，文章分成两个部分，实际上我是先写第二部分了再写第一部分的。

## 第一部分

看一下view.post的内部。
``` java
public boolean post(Runnable action) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        return attachInfo.mHandler.post(action);
    }
    // Assume that post will succeed later
    ViewRootImpl.getRunQueue().post(action);
    return true;
}
```


如果在Activity#onCreate()中直接调用view.post，attachInfo==null，因为此时view还没有attach到Window上。那么看ViewRootImpl.getRunQueue().post(action)。
``` java
/**
 * The run queue is used to enqueue pending work from Views when no Handler is
 * attached.  The work is executed during the next call to performTraversals on
 * the thread.
 * @hide
 */
static final class RunQueue {
    private final ArrayList<HandlerAction> mActions = new ArrayList<HandlerAction>();
    void post(Runnable action) {
        postDelayed(action, 0);
    }
    void postDelayed(Runnable action, long delayMillis) {
        HandlerAction handlerAction = new HandlerAction();
        handlerAction.action = action;
        handlerAction.delay = delayMillis;
        synchronized (mActions) {
            mActions.add(handlerAction);
        }
    }
	...
    void executeActions(Handler handler) {
        synchronized (mActions) {
            final ArrayList<HandlerAction> actions = mActions;
            final int count = actions.size();
            for (int i = 0; i < count; i++) {
                final HandlerAction handlerAction = actions.get(i);
                handler.postDelayed(handlerAction.action, handlerAction.delay);
            }
            actions.clear();
        }
    }
    private static class HandlerAction {
        Runnable action;
        long delay;
        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (o == null || getClass() != o.getClass()) return false;
            HandlerAction that = (HandlerAction) o;
            return !(action != null ? !action.equals(that.action) : that.action != null);
        }
        @Override
        public int hashCode() {
            int result = action != null ? action.hashCode() : 0;
            result = 31 * result + (int) (delay ^ (delay >>> 32));
            return result;
        }
    }
}
```
这里省略了一个remove的方法。RunQueue就是在view.post(Runnable)的时候，把Runnable放到数组mAction里，然后当ActivityThread的performTraversal()里的getRunQueue().executeActions(attachInfo.mHandler);时，就把所有数组中的Runnable发送到主线程的Looper里去轮询。当然这些都会在performTraversal()执行完之后才能执行。

## 第二部分
```  java
final class TraversalRunnable implements Runnable {
	@Override
	public void run() {
		doTraversal();//专门用来执行ViewRootImpl里的doTraversal()，而doTraversal()里面有performTraversal();
	}
}
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();//紧接着就new出来一个实例
```
`mTraversalRunnable`在ViewRootImpl里面总共也就出现了两次
``` java
void scheduleTraversals() {
	if (!mTraversalScheduled) {
		mTraversalScheduled = true;
		mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
		mChoreographer.postCallback(
				Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
		if (!mUnbufferedInputDispatch) {
			scheduleConsumeBatchedInput();
		}
		notifyRendererOfFramePending();
		pokeDrawLockIfNeeded();
	}
}
```
另一个是`void unscheduleTraversals()`，先不看。
那么在这里出现的地方这一句`mChoreographer.postCallback( Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);`，
Choreographer.n 编舞者
他在`ViewRootImpl`的构造方法中实例化：`mChoreographer = Choreographer.getInstance();`
``` java
//获取单例
public static Choreographer getInstance() {
	return sThreadInstance.get();
}
//看下sThreadInstance
private static final ThreadLocal<Choreographer> sThreadInstance =
		new ThreadLocal<Choreographer>() {
	@Override
	protected Choreographer initialValue() {
		Looper looper = Looper.myLooper();
		if (looper == null) {
			throw new IllegalStateException("The current thread must have a looper!");
		}
		return new Choreographer(looper);
	}
};
//看下编舞者的私有构造方法
private Choreographer(Looper looper) {
	mLooper = looper;
	mHandler = new FrameHandler(looper);//用looper实例化了Handler。
	mDisplayEventReceiver = USE_VSYNC ? new FrameDisplayEventReceiver(looper) : null;
	mLastFrameTimeNanos = Long.MIN_VALUE;

	mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());

	mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
	for (int i = 0; i <= CALLBACK_LAST; i++) {
		mCallbackQueues[i] = new CallbackQueue();
	}
}
```
很明显的是由ThreadLocal来控制的线程单例类，拿到的Looper也是线程单例的Looper，而创建Choreographer的ViewRootImpl是在主线程创建的，因此拿到的就是主线程的Looper。
看回Choreographer的postCallback();，最终进来调用的postCallbackDelayedInternal();
``` java
private void postCallbackDelayedInternal(int callbackType,
		Object action, Object token, long delayMillis) {
	if (DEBUG_FRAMES) {
		Log.d(TAG, "PostCallback: type=" + callbackType
				+ ", action=" + action + ", token=" + token
				+ ", delayMillis=" + delayMillis);
	}

	synchronized (mLock) {
		final long now = SystemClock.uptimeMillis();
		final long dueTime = now + delayMillis;
		mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

		if (dueTime <= now) {
			scheduleFrameLocked(now);
		} else {
			Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
			msg.arg1 = callbackType;
			msg.setAsynchronous(true);
			//在此处用主线程的looper，最后把这个Runnable发到主线程里去，排队等待被调用mHandler.handleMessage()；最后被执行。
			mHandler.sendMessageAtTime(msg, dueTime);
		}
	}
}
```
总之这个ViewRootImpl里的doTraversal()方法由ViewRootImpl的scheduleTraversal()最终放到
Android主线程的Looper里面去轮询排队执行了。

后面就是doTraversal()里面的performTraversal()了。
``` java
 private void performTraversals() {
        // cache mView since it is used so much below...
        final View host = mView;
		//注：该方法有800行代码，已省略
        // Execute enqueued actions on every traversal in case a detached view enqueued an action
        getRunQueue().executeActions(attachInfo.mHandler);

    ...
    performMeasure();//从DecorView开始完成View树的测量
    ...
    performLayout();//从DecorView开始完成View树的布局
    ...
    performDraw();//从DecorView开始绘制View树
 }
```
注意：`getRunQueue().executeActions(attachInfo.mHandler)`里面会遍历数组然后把所有的view.post(Runnable)里的Runnable都用和主线程Looper关联的Hanlder#sendMessage出去，放到主线程Looper里轮询，等待调用。那么由于当前的`performTraversals()`本身就是由主线程Looper回调给刚才的编舞者Choreographer里面去执行的，因此主线程一定会等待performTraversals()整个方法执行完，才去接着执行由view.post()推送到主线程的Runnable。因此整个View树都完成了测量，布局，绘制。然后view.post()里面百分百的可以拿到view的宽高了。

---

thanks

[通过View.post()获取View的宽高引发的两个问题：1post的Runnable何时被执行，2为何View需要layout两次；以及发现Android的一个小bug - CSDN博客](https://blog.csdn.net/scnuxisan225/article/details/49815269)

[【Android源码解析】View.post()到底干了啥 - 请叫我大苏 - 博客园](https://www.cnblogs.com/dasusu/p/8047172.html)

