## 1 取出Throwable的堆栈信息



使用Throwable一个很大的好处在于，他能保存他被实例化的方法的堆栈信息，通过方法：

```java
Throwable#printStackTrace()
```

可以将他和他的cause Throwable和他的cause的cause...( 递归 ) 的堆栈信息都打印出来。

而例如我们要将一个Throwable对象的堆栈信息不仅仅是输出到控制台，还要保存到本地日志或者发送到服务器呢？那么就要将Throwable的堆栈信息提取出来。

令人开心的是，`android.util.Log`类提供了这么一个工具方法：

``` java
/**
 * Handy function to get a loggable stack trace from a Throwable
 * @param tr An exception to log
 */
public static String getStackTraceString(Throwable tr) {
    if (tr == null) {
        return "";
    }
    // This is to reduce the amount of log spew that apps do in the non-error
    // condition of the network being unavailable.
    Throwable t = tr;
    while (t != null) {
        if (t instanceof UnknownHostException) {
            return "";
        }
        t = t.getCause();
    }
    StringWriter sw = new StringWriter();
    PrintWriter pw = new PrintWriter(sw);
    tr.printStackTrace(pw);
    pw.flush();
    return sw.toString();
}
```

通过该方法，可以直接把`Throwable`对象的堆栈信息都拿出来了。



## 2 该如何构造一个Throwable

``` java
public Throwable() {
    fillInStackTrace();
}
public Throwable(String message) {
    fillInStackTrace();
    detailMessage = message;
}
public Throwable(String message, Throwable cause) {
    fillInStackTrace();
    detailMessage = message;
    this.cause = cause;
}
public Throwable(Throwable cause) {
    fillInStackTrace();
    detailMessage = (cause==null ? null : cause.toString());
    this.cause = cause;
}
```

他有4个构造方法。每个构造方法都会调用`fillInStackTrace()`来记录当前的堆栈信息。

只有两个参数可选：String类型的message，和他的cause Throwable。

那么现在来看一下这两个变量对Throwable有什么用，以及对我们来说有什么意义。



先说结论：`detailMessage`和`cause`变量在调用`printStackTrace()`的时候都会被打印出来。

看下`printStackTrace()`

``` java
private void printStackTrace(PrintStreamOrWriter s) {
    // Guard against malicious overrides of Throwable.equals by
    // using a Set with identity equality semantics.
    Set<Throwable> dejaVu =
        Collections.newSetFromMap(new IdentityHashMap<Throwable, Boolean>());
    dejaVu.add(this);
    synchronized (s.lock()) {
        // Print our stack trace
        //打印toString()
        s.println(this);
        //打印详细的堆栈信息
        StackTraceElement[] trace = getOurStackTrace();
        for (StackTraceElement traceElement : trace)
            s.println("\tat " + traceElement);
        // Print suppressed exceptions, if any
        for (Throwable se : getSuppressed())
            se.printEnclosedStackTrace(s, trace, SUPPRESSED_CAPTION, "\t", dejaVu);
        // Print cause, if any
        //打印cause
        Throwable ourCause = getCause();
        if (ourCause != null)
            ourCause.printEnclosedStackTrace(s, trace, CAUSE_CAPTION, "", dejaVu);
    }
}
```

看下`toString`是如何包含`message`进来的：

``` java
public String toString() {
    String s = getClass().getName();
    String message = getLocalizedMessage();
    return (message != null) ? (s + ": " + message) : s;
}
public String getLocalizedMessage() {
    return getMessage();
}
public String getMessage() {
    return detailMessage;
}
```

看下打印cause:

``` java
public synchronized Throwable getCause() {
    return (cause==this ? null : cause);
}

private void printEnclosedStackTrace(PrintStreamOrWriter s,
                                     StackTraceElement[] enclosingTrace,
                                     String caption,
                                     String prefix,
                                     Set<Throwable> dejaVu) {
    if (dejaVu.contains(this)) {
        s.println("\t[CIRCULAR REFERENCE:" + this + "]");
    } else {
        dejaVu.add(this);
        // Compute number of frames in common between this and enclosing trace
        StackTraceElement[] trace = getOurStackTrace();
        int m = trace.length - 1;
        int n = enclosingTrace.length - 1;
        while (m >= 0 && n >=0 && trace[m].equals(enclosingTrace[n])) {
            m--; n--;
        }
        int framesInCommon = trace.length - 1 - m;
        // Print our stack trace
        s.println(prefix + caption + this);
        for (int i = 0; i <= m; i++)
            s.println(prefix + "\tat " + trace[i]);
        if (framesInCommon != 0)
            s.println(prefix + "\t... " + framesInCommon + " more");
        // Print suppressed exceptions, if any
        for (Throwable se : getSuppressed())
            se.printEnclosedStackTrace(s, trace, SUPPRESSED_CAPTION,
                                       prefix +"\t", dejaVu);
        // Print cause, if any
        //又调用了getCause()，在这里实际上是一个递归。直到找到最根源的那个cause才会停止。
        Throwable ourCause = getCause();
        if (ourCause != null)
            ourCause.printEnclosedStackTrace(s, trace, CAUSE_CAPTION, prefix, dejaVu);
    }
}
```

可以发现，`printEnclosedStackTrace()`方法中又调用了`getCause()`和`printEnclosedStackTrace()`，那么其实就是一个递归，直到递归到最根源的那个cause。



那么当我们要构造一个`Throwable`对象的时候，如果上下文中有一个关联的`Throwable`，那么把他作为cause参数来构造新的`Throwable`对象，这样能更好地记录问题真正的原因。


