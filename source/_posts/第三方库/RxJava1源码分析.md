我对于RxJava的异常处理和上抛方式有一些不解，而上网查找的文章都是RxJava的一些用于处理异常的操作符，所以只能自己去源码里面找答案了。

虽然RxJava1已经过时了，但是鉴于RxJava1的源码会比RxJava2的简洁一些，因此易于分析。所以我在这里对RxJava1的源码进行分析。



### 1 构造Observable



#### 1.1 create方式

``` java
Observable.create<String> { it: Subscriber<in String> ->
    //上游发射数据
    it.onNext("123")
    it.onCompleted()
}.subscribe { it: String ->
    //下游处理数据
    LogUtils.d(it)
}
```

这里看两个方法：create和subscribe



##### 1.1.1 create

create方法需要`OnSubscribe`接口作为参数，然后再返回一个`Observable`类型的对象（这个对象待会再调用`subscribe()`方法启动数据的发射）。



``` java
public final static <T> Observable<T> create(OnSubscribe<T> f) {
    return new Observable<T>(hook.onCreate(f));
}
```

那么首先看：`hook.onCreate(f)`，根据注释，这是一个有**装饰者模式**味道的的钩子方法。

``` java
public abstract class RxJavaObservableExecutionHook {
     /**
     * Invoked during the construction by {@link Observable#create(OnSubscribe)}
     * <p>
     * This can be used to decorate or replace the <code>onSubscribe</code> function or just perform extra
     * logging, metrics and other such things and pass-thru the function.
     * 
     * @param f
     *            original {@link OnSubscribe}<{@code T}> to be executed
     * @return {@link OnSubscribe}<{@code T}> function that can be modified, decorated, replaced or just
     *         returned as a pass-thru
     */
    public <T> OnSubscribe<T> onCreate(OnSubscribe<T> f) {
        return f;
    }
}
```

默认情况下，传进来什么就返回什么，即没有加任何的装饰的逻辑。



Observable的构造函数：将`OnSubscribe`保存了起来。

``` java
public class Observable<T> {
    final OnSubscribe<T> onSubscribe;

    /**
     * Creates an Observable with a Function to execute when it is subscribed to.
     * <p>
     * <em>Note:</em> Use {@link #create(OnSubscribe)} to create an Observable, instead of this constructor,
     * unless you specifically have a need for inheritance.
     * 
     * @param f
     *            {@link OnSubscribe} to be executed when {@link #subscribe(Subscriber)} is called
     */
    protected Observable(OnSubscribe<T> f) {
        this.onSubscribe = f;
    }
}
```



##### 1.1.2 subscribe

``` java
public final Subscription subscribe(final Action1<? super T> onNext) {
    if (onNext == null) {
        throw new IllegalArgumentException("onNext can not be null");
    }
    //构造一个Subscriber，衔接onNext的方法，并调用subscribe方法返回Subscription
    return subscribe(new Subscriber<T>() {
        @Override
        public final void onCompleted() {
            // do nothing
        }
        @Override
        public final void onError(Throwable e) {
            throw new OnErrorNotImplementedException(e);
        }
        @Override
        public final void onNext(T args) {
            onNext.call(args);
        }
    });
}
```

返回的是`Subscription`，和RxJava2中的`Disposable`是一个东西：用来取消订阅

``` java
public interface Subscription {
    void unsubscribe();
    boolean isUnsubscribed();
}
```

接着看

``` java
public final Subscription subscribe(Subscriber<? super T> subscriber) {
    //调用了静态方法：
    return Observable.subscribe(subscriber, this);
}
```

``` java
private static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
 	// validate and proceed
    if (subscriber == null) {
        throw new IllegalArgumentException("observer can not be null");
    }
    if (observable.onSubscribe == null) {
        throw new IllegalStateException("onSubscribe function can not be null.");
        /*
         * the subscribe function can also be overridden but generally that's not the appropriate approach
         * so I won't mention that in the exception
         */
    }
    
    // new Subscriber so onStart it
    subscriber.onStart();
    
    /*
     * See https://github.com/ReactiveX/RxJava/issues/216 for discussion on "Guideline 6.4: Protect calls
     * to user code from within an Observer"
     */
    // if not already wrapped
    if (!(subscriber instanceof SafeSubscriber)) {
        // assign to `observer` so we return the protected version
        subscriber = new SafeSubscriber<T>(subscriber);
    }
    // The code below is exactly the same an unsafeSubscribe but not used because it would add a sigificent depth to alreay huge call stacks.
    try {
        // allow the hook to intercept and/or decorate
        hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
        return hook.onSubscribeReturn(subscriber);
    } catch (Throwable e) {
        // special handling for certain Throwable/Error/Exception types
        Exceptions.throwIfFatal(e);
        // if an unhandled error occurs executing the onSubscribe we will propagate it
        try {
            subscriber.onError(hook.onSubscribeError(e));
        } catch (OnErrorNotImplementedException e2) {
            // special handling when onError is not implemented ... we just rethrow
            throw e2;
        } catch (Throwable e2) {
            // if this happens it means the onError itself failed (perhaps an invalid function implementation)
            // so we are unable to propagate the error correctly and will just throw
            RuntimeException r = new RuntimeException("Error occurred attempting to subscribe [" + e.getMessage() + "] and then again while trying to pass to onError.", e2);
            // TODO could the hook be the cause of the error in the on error handling.
            hook.onSubscribeError(r);
            // TODO why aren't we throwing the hook's return value.
            throw r;
        }
        return Subscriptions.unsubscribed();
    }
}
```

对这个`subsribe()`方法做的事情做一个总结：

1. 对参数进行验证，保证Not null
2. 调用subscriber.onStart();
3. 将`Subscriber`封装成`SafeSubscriber`，用装饰者模式包装了一层，在这一层加了一些额外的逻辑，但是不影响主要逻辑的执行，所以这一层的逻辑我们稍后再分析。
4. 调用`OnSubsribe`接口的`call`方法。并捕捉异常，关于异常捕捉，稍后再分析



而由于调用到第四点的`call`方法，call方法就是create方法的参数传递进去的代码块：

``` java
Observable.create<String> { it: Subscriber<in String> ->
    //上游发射数据
    it.onNext("123")
    it.onCompleted()
}.subscribe { it: String ->
    //下游处理数据
    LogUtils.d(it)
}
```

因此我们调用onNext传递的数据就能够在下游被处理到了。



#### 1.2 just方式

``` java
Observable.just("2")
    .doOnNext {
        LogUtils.d(it)
    }
    .subscribe {
        val s: String? = null
        s!!
        s.toString()
    }
```

当调用`just`方法的时候，就不需要在上游手动调用`onNext`了，那么一定是`RxJava`的内部调用了`onNext`，来看下吧。

``` java
public final static <T> Observable<T> just(final T value) {
    return ScalarSynchronousObservable.create(value);
}
```

返回了一个`ScalarSynchronousObservable`的create方法：

``` java
public final class ScalarSynchronousObservable<T> extends Observable<T> {

    public static final <T> ScalarSynchronousObservable<T> create(T t) {
        return new ScalarSynchronousObservable<T>(t);
    }

    private final T t;

    protected ScalarSynchronousObservable(final T t) {
        super(new OnSubscribe<T>() {

            @Override
            public void call(Subscriber<? super T> s) {
                /*
                 *  We don't check isUnsubscribed as it is a significant performance impact in the fast-path use cases.
                 *  See PerfBaseline tests and https://github.com/ReactiveX/RxJava/issues/1383 for more information.
                 *  The assumption here is that when asking for a single item we should emit it and not concern ourselves with 
                 *  being unsubscribed already. If the Subscriber unsubscribes at 0, they shouldn't have subscribed, or it will 
                 *  filter it out (such as take(0)). This prevents us from paying the price on every subscription. 
                 */
                s.onNext(t);
                s.onCompleted();
            }

        });
        this.t = t;
    }
    //...
}
```

`ScalarSynchronousObservable`的构造方法中传入的`OnSubscribe`的实现中，已经调用了`onNext`和`onCompleted`了。



### 2 SafeSubscriber

事件监听就是选择性地重写三个方法：

`void onNext(T t);`，`void onError(Throwable e);`，`void onCompleted();`。

而这三个方法的关系，例如onError和onCompleted有调用互斥性等，都借由`SafeSubscriber`类实现：

``` java
private static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
		//...
        if (!(subscriber instanceof SafeSubscriber)) {
            // assign to `observer` so we return the protected version
            subscriber = new SafeSubscriber<T>(subscriber);
        }
		//...
    }
```



``` java
public class SafeSubscriber<T> extends Subscriber<T> {

    private final Subscriber<? super T> actual;

    //事件流是否结束
    boolean done = false;
	//用装饰者模式，封装真正的Subscriber在actual变量中
    public SafeSubscriber(Subscriber<? super T> actual) {
        super(actual);
        this.actual = actual;
    }

    @Override
    public void onCompleted() {
        //如果事件流没有结束
        if (!done) {
            done = true;
            //将onCompleted用try catch
            try {
                actual.onCompleted();
            } catch (Throwable e) {
                //抛出致命异常
                Exceptions.throwIfFatal(e);
                //调用内部_onError
                _onError(e);
            } finally {
                unsubscribe();
            }
        }
    }

    @Override
    public void onError(Throwable e) {
        //抛出致命异常
        Exceptions.throwIfFatal(e);
        if (!done) {
            done = true;
            //调用内部_onError
            _onError(e);
        }
    }

    @Override
    public void onNext(T args) {
        try {
            if (!done) {
                actual.onNext(args);
            }
        } catch (Throwable e) {
            //抛出致命异常
            Exceptions.throwIfFatal(e);
            //回调到onError
            onError(e);
        }
    }

    //有两处会调用这里，1. onCompleted。2. onError
    protected void _onError(Throwable e) {
        try {
			//首先调用RxJavaPlugins中的错误统一处理
            RxJavaPlugins.getInstance().getErrorHandler().handleError(e);
        } catch (Throwable pluginException) {
            //捕捉RxJavaPlugins中的错误，并打印出来。
            handlePluginException(pluginException);
        }
        try {
            //调用真正的actual的onError
            actual.onError(e);
        } catch (Throwable e2) {
            //补货onError中的异常
            if (e2 instanceof OnErrorNotImplementedException) {
                //如果异常是OnErrorNotImplementedException
                //unsubscribe
                try {
                    unsubscribe();
                } catch (Throwable unsubscribeException) {
                    try {
                        RxJavaPlugins.getInstance().getErrorHandler().handleError(unsubscribeException);
                    } catch (Throwable pluginException) {
                        handlePluginException(pluginException);
                    }
                    throw new RuntimeException("Observer.onError not implemented and error while unsubscribing.", new CompositeException(Arrays.asList(e, unsubscribeException)));
                }
                //将异常抛出
                throw (OnErrorNotImplementedException) e2;
            } else {
                //否则，还是进行错误统一处理
                try {
                    RxJavaPlugins.getInstance().getErrorHandler().handleError(e2);
                } catch (Throwable pluginException) {
                    handlePluginException(pluginException);
                }
                //unsubscirbe
                try {
                    unsubscribe();
                } catch (Throwable unsubscribeException) {
                    try {
                        RxJavaPlugins.getInstance().getErrorHandler().handleError(unsubscribeException);
                    } catch (Throwable pluginException) {
                        handlePluginException(pluginException);
                    }
                    throw new OnErrorFailedException("Error occurred when trying to propagate error to Observer.onError and during unsubscription.", new CompositeException(Arrays.asList(e, e2, unsubscribeException)));
                }
				//再将异常抛出
                throw new OnErrorFailedException("Error occurred when trying to propagate error to Observer.onError", new CompositeException(Arrays.asList(e, e2)));
            }
        }
        //unsubscribe
        try {
            unsubscribe();
        } catch (RuntimeException unsubscribeException) {
            try {
                RxJavaPlugins.getInstance().getErrorHandler().handleError(unsubscribeException);
            } catch (Throwable pluginException) {
                handlePluginException(pluginException);
            }
            throw new OnErrorFailedException(unsubscribeException);
        }
    }

    private void handlePluginException(Throwable pluginException) {
        System.err.println("RxJavaErrorHandler threw an Exception. It shouldn't. => " + pluginException.getMessage());
        pluginException.printStackTrace();
    }
    
    public Subscriber<? super T> getActual() {
        return actual;
    }
}
```



### 3 RxJava如何处理异常，如何上抛异常

上文的对`SafeSubscriber`的分析可以看出RxJava对处理数据下游异常的方式：

1. 转到onError将异常抛出。
2. 如果onError未实现，那么直接将异常抛出。
3. 如果onError实现了，但是onError中又有异常，那么RxJava又会将异常抛出。



那么如果在数据的上游，即数据发射处就发生异常了，要如何处理呢：

在Observable的构造类的函数中，最终会调用到：

``` java
private static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
 // validate and proceed
    if (subscriber == null) {
        throw new IllegalArgumentException("observer can not be null");
    }
    if (observable.onSubscribe == null) {
        throw new IllegalStateException("onSubscribe function can not be null.");
        /*
         * the subscribe function can also be overridden but generally that's not the appropriate approach
         * so I won't mention that in the exception
         */
    }
    
    // new Subscriber so onStart it
    subscriber.onStart();
    
    /*
     * See https://github.com/ReactiveX/RxJava/issues/216 for discussion on "Guideline 6.4: Protect calls
     * to user code from within an Observer"
     */
    // if not already wrapped
    if (!(subscriber instanceof SafeSubscriber)) {
        // assign to `observer` so we return the protected version
        subscriber = new SafeSubscriber<T>(subscriber);
    }
    // The code below is exactly the same an unsafeSubscribe but not used because it would add a sigificent depth to alreay huge call stacks.
    try {
        // allow the hook to intercept and/or decorate
        hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
        return hook.onSubscribeReturn(subscriber);
    } catch (Throwable e) {
        // special handling for certain Throwable/Error/Exception types
        Exceptions.throwIfFatal(e);
        // if an unhandled error occurs executing the onSubscribe we will propagate it
        try {
            subscriber.onError(hook.onSubscribeError(e));
        } catch (OnErrorNotImplementedException e2) {
            // special handling when onError is not implemented ... we just rethrow
            throw e2;
        } catch (Throwable e2) {
            // if this happens it means the onError itself failed (perhaps an invalid function implementation)
            // so we are unable to propagate the error correctly and will just throw
            RuntimeException r = new RuntimeException("Error occurred attempting to subscribe [" + e.getMessage() + "] and then again while trying to pass to onError.", e2);
            // TODO could the hook be the cause of the error in the on error handling.
            hook.onSubscribeError(r);
            // TODO why aren't we throwing the hook's return value.
            throw r;
        }
        return Subscriptions.unsubscribed();
    }
}
```

看一下上述的这段代码：

``` java
try {
    //将call的调用用try catch保护起来
    hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
    return hook.onSubscribeReturn(subscriber);
} catch (Throwable e) {
    //抛出致命异常
    Exceptions.throwIfFatal(e);
    try {
        //调用给onError
        subscriber.onError(hook.onSubscribeError(e));
    } catch (OnErrorNotImplementedException e2) {
        throw e2;
    } catch (Throwable e2) {
        RuntimeException r = new RuntimeException("Error occurred attempting to subscrib
        hook.onSubscribeError(r);
        throw r;
    }
	//unsubscribe
    return Subscriptions.unsubscribed();
}
```

可以看到：数据发射处也有异常处理：交给观察者的onError处理，然后处理逻辑就又转交给观察了。



### 4 为什么能够用操作符追加代码逻辑

#### 4.1 图和大致流程分析

在进行代码分析之前，先看下这个大致的调用流程图：

![](https://upload-images.jianshu.io/upload_images/7177220-d519af5b25e37b39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这幅图表现出来的几个点：

1. 在使用RxJava写调用链代码的时候，`onNext`等代码块是从上往下执行的，但是每当往调用链上拼接一个RxJava的处理方法例如：`doOnNext`或者`map`的时候，都会生成一个新的`Subscriber`，而当调用最底部的`subscribe()`方法的时候，调用链上每一个`Subscriber`中的`OnSubscribe.call()`方法实际上是从下往上调用的。
2. 在图中，`Observable3`是最终的观察者创建的对象。当调用subscribe方法的时候，由`Observable.subscribe()`的源码开始，要调用`Observable2`的`call_2()`方法，而`call_2()`方法的逻辑要参考`Observable#lift()`方法的逻辑，他会将`Observable2`的`onNext()`等代码块保存并拼接在`Observable3`的`onNext`之前。然后又调用`Observable1`的`call_1()`方法...一直这么重复着往上调用各个链上的call方法。最后，调用到顶部的数据发起处的函数：`Observable.create`，他的`call()`方法使我们写好的：`it.onNext("123")...`（或者Observable.just等构造方法，里面内定了如何调用onNext方法来发射数据）。此时，顶部开始发射数据。此时，遇到他的观察者：`Observable1`的`onNext()`，那么执行内部的逻辑，并调用了`Observable2`的`onNext()`，然后执行后者内部的逻辑，然后又调用`Observable3`的...，就这样把数据不断地往调用链下部调用，最终到达底部的观察者的代码块。
3. 大的逻辑是：从上往下增加每一个操作符，就会构造一个`Subscriber`，然后在最后调用`subscribe()`方法的时候，递归上去一个一个地调用call方法，最终到顶部的`onNext`，再递归下来，一个一个地调用开发者调用每个操作符时加入的逻辑。



那么先来从简单的开始看好了：

``` java
Observable.create<String> { it: Subscriber<in String> ->
    it.onNext("123")
    it.onCompleted()
}.subscribe { it: String ->
    LogUtils.d(it)
}
```

这种类型的流程上面已经分析过了，非常简单，就是在subscribe()方法调用的时候，调用call方法里面的:

``` java
    it.onNext("123")
    it.onCompleted()
```

然后自然地数据就发送到了下游了。



上述是没有添加任何操作符的情况，那么如果添加操作符了呢？例如添加一个`doOnNext()`

``` java
Observable
    .create<String> { it: Subscriber<in String> ->
        it.onNext("123")
        it.onCompleted()
    }
    .doOnNext {
        LogUtils.d("doOnNext=$it")
    }
    .subscribe { it: String ->
        LogUtils.d(it)
    }
```

这里我们看一下doOnNext中的代码块是如何追加到调用链上的。

看实现：

``` java
public final Observable<T> doOnNext(final Action1<? super T> onNext) {
    Observer<T> observer = new Observer<T>() {
        @Override
        public final void onCompleted() {
        }
        @Override
        public final void onError(Throwable e) {
        }
        @Override
        public final void onNext(T args) {
            onNext.call(args);
        }
    };
    //上述代码是将onNext封装到了一个Observer里面。
    return lift(new OperatorDoOnEach<T>(observer));
}
```

这个封装过的observer，作为`OperatorDoOnEach`类的构造器的参数被传递进去，然后又作为`lift()`方法被调用，并返回一个`Observable`类型。（是的，因为这个操作符是可以直接调用subscribe()的）

#### 4.2 OperatorDoOnEach类型

``` java
public class OperatorDoOnEach<T> implements Operator<T, T>
```



他的父类型是Operator：

``` java
/**
 * Operator function for lifting into an Observable.
 */
public interface Operator<R, T> extends Func1<Subscriber<? super R>, Subscriber<? super T>> {
    // cover for generics insanity
}
```

Operator实现了Func1接口：

``` java
public interface Func1<T, R> extends Function {
    R call(T t);
}
```

Func1接口的作用是：转化

调用call方法的时候：输入T，返回R。

那么Operator的作用也可以说是：转化。但是他是Func1<Subscriber<? super R>, Subscriber<? super T>>，因此他的转化是：输入一个观察者T，返回另一个观察者R。

那么我们也可以说Operator的作用是：给原有的观察者添加额外的逻辑。



那么说具体点：客户端的调用是：

``` java
Operator concreteOperator ;
SubscriberB = concreteOperator.call(SubscriberA);
```

即获取到Operator接口，然后调用call方法，进行转换。



而`doOnNext()`方法用的是`OperatorDoOnEach`：

``` java
public class OperatorDoOnEach<T> implements Operator<T, T> {
    private final Observer<? super T> doOnEachObserver;

    //构造方法中，保存了一个观察者，称为doOnEachObserver
    public OperatorDoOnEach(Observer<? super T> doOnEachObserver) {
        this.doOnEachObserver = doOnEachObserver;
    }

    //调用call方法，开始转换。call方法返回的新的观察者的每个实现，都是在参数observer的方法之前
    //拼接上构造函数的doOnEachObserver的对应的方法。
    @Override
    public Subscriber<? super T> call(final Subscriber<? super T> observer) {
        //传入的是observer
        return new Subscriber<T>(observer) {

            private boolean done = false;

            @Override
            public void onCompleted() {
                if (done) {
                    return;
                }
                //先调用doOnEachObserver.onCompleted()
                try {
                    doOnEachObserver.onCompleted();
                } catch (Throwable e) {
                    onError(e);
                    return;
                }
                // Set `done` here so that the error in `doOnEachObserver.onCompleted()` can be noticed by observer
                done = true;
                //再调用observer.onCompleted()
                observer.onCompleted();
            }

            @Override
            public void onError(Throwable e) {
                // need to throwIfFatal since we swallow errors after terminated
                Exceptions.throwIfFatal(e);
                if (done) {
                    return;
                }
                done = true;
                //先调用doOnEachObserver.onError
                try {
                    doOnEachObserver.onError(e);
                } catch (Throwable e2) {
                    observer.onError(e2);
                    return;
                }
                //再调用observer.onError
                observer.onError(e);
            }

            @Override
            public void onNext(T value) {
                if (done) {
                    return;
                }
                //先调用doOnEachObserver.onNext
                try {
                    doOnEachObserver.onNext(value);
                } catch (Throwable e) {
                    onError(OnErrorThrowable.addValueAsLastCause(e, value));
                    return;
                }
                //再调用observer.onNext
                observer.onNext(value);
            }
        };
    }
}
```

分析完了OperatorDoOnEach的具体实现，接下来要看下他的call方法是如何被调用的：

#### 4.3 lift()方法

接着看下`doOnNext()`

``` java
public final Observable<T> doOnNext(final Action1<? super T> onNext) {
    Observer<T> observer = new Observer<T>() {
        @Override
        public final void onCompleted() {
        }
        @Override
        public final void onError(Throwable e) {
        }
        @Override
        public final void onNext(T args) {
            onNext.call(args);
        }
    };
    //上述代码是将onNext封装到了一个Observer里面。
    return lift(new OperatorDoOnEach<T>(observer));
}
```

``` java
public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
    return new Observable<R>(new OnSubscribe<R>() {
        //这个call会被上层调用
        @Override
        public void call(Subscriber<? super R> o) {
            //这个o是上游调用这个return new Observable返回的观察者中的OnSubscribe的call方法传递下来的
            //观察者，在本例中，由于onNext之前就是Observable.create，因此o中的call方法就是：
            //	{
            //		it.onNext("123");
            //		it.onCompleted();
            //	}
            try {
                //调用call，将o转换成st。
                //st中的call方法的逻辑参照着OperatorDoOnEach的逻辑就是：将operator的调用逻辑追加在o的调用逻辑之前。
                Subscriber<? super T> st = hook.onLift(operator).call(o);
                try {
                    // new Subscriber created and being subscribed with so 'onStart' it
                    st.onStart();
                    //继续调用call方法
                    onSubscribe.call(st);
                } catch (Throwable e) {
                    // localized capture of errors rather than it skipping all operators 
                    // and ending up in the try/catch of the subscribe method which then
                    // prevents onErrorResumeNext and other similar approaches to error handling
                    if (e instanceof OnErrorNotImplementedException) {
                        throw (OnErrorNotImplementedException) e;
                    }
                    st.onError(e);
                }
            } catch (Throwable e) {
                if (e instanceof OnErrorNotImplementedException) {
                    throw (OnErrorNotImplementedException) e;
                }
                // if the lift function failed all we can do is pass the error to the final Subscriber
                // as we don't have the operator available to us
                o.onError(e);
            }
        }
    });
}
```

注意，onLift方法是一个全局钩子。

``` java
public <T, R> Operator<? extends R, ? super T> onLift(final Operator<? extends R, ? super T> lift) {
    //默认实现是啥都不处理直接返回。
    return lift;
}
```



### 5 常用操作符源码分析

#### 5.1 filter

过滤

``` java
public final Observable<T> filter(Func1<? super T, Boolean> predicate) {
    //调用lift
    return lift(new OperatorFilter<T>(predicate));
}
```



``` java
public final class OperatorFilter<T> implements Operator<T, T> {

    private final Func1<? super T, Boolean> predicate;

    public OperatorFilter(Func1<? super T, Boolean> predicate) {
        this.predicate = predicate;
    }

    @Override
    public Subscriber<? super T> call(final Subscriber<? super T> child) {
        return new Subscriber<T>(child) {

            @Override
            public void onCompleted() {
                child.onCompleted();
            }

            @Override
            public void onError(Throwable e) {
                child.onError(e);
            }

            @Override
            public void onNext(T t) {
                try {
                    //如果call方法返回true，才继续将数据向下传递
                    if (predicate.call(t)) {
                        child.onNext(t);
                    } else {
                        // TODO consider a more complicated version that batches these
                        request(1);
                    }
                } catch (Throwable e) {
                    child.onError(OnErrorThrowable.addValueAsLastCause(e, t));
                }
            }

        };
    }

}
```



#### 5.2 map

映射

``` java
public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
    return lift(new OperatorMap<T, R>(func));
}
```

``` java
public final class OperatorMap<T, R> implements Operator<R, T> {

    private final Func1<? super T, ? extends R> transformer;

    public OperatorMap(Func1<? super T, ? extends R> transformer) {
        this.transformer = transformer;
    }

    @Override
    public Subscriber<? super T> call(final Subscriber<? super R> o) {
        return new Subscriber<T>(o) {

            @Override
            public void onCompleted() {
                o.onCompleted();
            }

            @Override
            public void onError(Throwable e) {
                o.onError(e);
            }

            @Override
            public void onNext(T t) {
                try {
                    //transformer就是映射，映射后的数据将继续向下游发射。
                    o.onNext(transformer.call(t));
                } catch (Throwable e) {
                    Exceptions.throwIfFatal(e);
                    onError(OnErrorThrowable.addValueAsLastCause(e, t));
                }
            }

        };
    }

}
```




