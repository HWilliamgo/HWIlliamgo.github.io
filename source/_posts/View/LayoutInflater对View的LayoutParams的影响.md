---
title: 浅谈LayoutParams
categories:
  - View
date: 2020-05-26 23:29:25
tags:
index_img:
banner_img:
layout: false
---



### 0 概述

`LayoutInflater`在Android开发中有大量的使用场景，例如：

1. Activity和Dialog的setContentView方法内部用LayoutInflater从xml文件中加载出View对象。
2. Fragment中用LayoutInflater创建一个View返回给Fragment。
3. RecylerView的每一个Item对应的ViewHolder在创建的时候，需要先用LayoutInflater创建出对应的Item的View。
4. 其他的，我们自己创建View的场景，比如自定义View等。

那么你是否了解LayoutInflater在创建View的时候，是如何处理View的属性吗？



### 1 源码

``` java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    if (DEBUG) {
        Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
              + Integer.toHexString(resource) + ")");
    }
    View view = tryInflatePrecompiled(resource, res, root, attachToRoot);
    if (view != null) {
        return view;
    }
    //根据R.layout.xxx的资源Id，创建出可以读取xml文件的解析器parser
    XmlResourceParser parser = res.getLayout(resource);
    try {
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
```

``` java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");
        final Context inflaterContext = mContext;
        //通过parser构造了AttributeSet对象，后者就是View的构造函数中返回的第二个参数，在自定义View时，我们通过他来
        //找到我们在xml中给View定义的所有属性，例如宽高，visibility，padding，和其他自定义属性。
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
        View result = root;
        try {
            advanceToRootNode(parser);
            final String name = parser.getName();
            if (DEBUG) {
                System.out.println("**************************");
                System.out.println("Creating root view: "
                        + name);
                System.out.println("**************************");
            }
            if (TAG_MERGE.equals(name)) {
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }
                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // Temp is the root view that was found in the xml
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);
                ViewGroup.LayoutParams params = null;
                if (root != null) {
                    if (DEBUG) {
                        System.out.println("Creating params from root: " +
                                root);
                    }
                    // Create layout params that match root, if supplied
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        // Set the layout params for temp if we are not
                        // attaching. (If we are, we use addView, below)
                        temp.setLayoutParams(params);
                    }
                }
                if (DEBUG) {
                    System.out.println("-----> start inflating children");
                }
                // Inflate all children under temp against its context.
                rInflateChildren(parser, temp, attrs, true);
                if (DEBUG) {
                    System.out.println("-----> done inflating children");
                }
                // We are supposed to attach all the views we found (int temp)
                // to root. Do that now.
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }
                // Decide whether to return the root that was passed in or the
                // top view found in xml.
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }
        } catch (XmlPullParserException e) {
            final InflateException ie = new InflateException(e.getMessage(), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        } catch (Exception e) {
            final InflateException ie = new InflateException(
                    getParserStateDescription(inflaterContext, attrs)
                    + ": " + e.getMessage(), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        } finally {
            // Don't retain static reference on context.
            mConstructorArgs[0] = lastContext;
            mConstructorArgs[1] = null;
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        return result;
    }
}
```

这里注意一点：AttributeSet对象是从XmlPullParser中构造来的，这个AttributeSet就是View类的构造函数的第二个参数。包含所有在xml中定义的View的属性。

#### 1.1 从xml中解析并用反射实例化View

``` java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    //...
    // Temp is the root view that was found in the xml
    final View temp = createViewFromTag(root, name, inflaterContext, attrs);
    //...
}
```

注意这里将atrrs传递，传递到通过attrs创建，并返回了一个View temp。

``` java
//Creates a view from a tag name using the supplied attribute set.
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
                       boolean ignoreThemeAttr) {
    if (name.equals("view")) {
        name = attrs.getAttributeValue(null, "class");
    }

    // Apply a theme wrapper, if allowed and one is specified.
    if (!ignoreThemeAttr) {
        //提取theme，也是使用的TypeArray来提取的。
        final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
        final int themeResId = ta.getResourceId(0, 0);
        if (themeResId != 0) {
            context = new ContextThemeWrapper(context, themeResId);
        }
        ta.recycle();
    }

    try {
        //给LayoutInflater创建View提供一个Hook，待会分析
        View view = tryCreateView(parent, name, context, attrs);

        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
                if (-1 == name.indexOf('.')) {
                    //创建android源码中的View，即那些在xml中不需要包名就可以定义并使用的View。
                    view = onCreateView(context, parent, name, attrs);
                } else {
                    //创建非android源码中的View，如自定义View或者第三方库的View。
                    view = createView(context, name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }

        return view;
    } catch (InflateException e) {
        throw e;

    } catch (ClassNotFoundException e) {
        final InflateException ie = new InflateException(
            getParserStateDescription(context, attrs)
            + ": Error inflating class " + name, e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;

    } catch (Exception e) {
        final InflateException ie = new InflateException(
            getParserStateDescription(context, attrs)
            + ": Error inflating class " + name, e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;
    }
}
```

可以看到创建View的两个语句：

通过这个if的判断，我们大致可以猜到这个是用来判断加载的是源码View还是非源码的View。

``` java
if (-1 == name.indexOf('.')) {
    //创建android源码中的View，即那些在xml中不需要包名就可以定义并使用的View。
    view = onCreateView(context, parent, name, attrs);
} else {
    //创建非android源码中的View，如自定义View或者第三方库的View。
    view = createView(context, name, null, attrs);
}
```

这两个方法内部都调用的是一个方法，我们看下两个方法是如何调用的，第一个：

``` java
public View onCreateView(@NonNull Context viewContext, @Nullable View parent,
        @NonNull String name, @Nullable AttributeSet attrs)
        throws ClassNotFoundException {
    return onCreateView(parent, name, attrs);
}
//-----往下调用
protected View onCreateView(View parent, String name, AttributeSet attrs)
        throws ClassNotFoundException {
    return onCreateView(name, attrs);
}
//-----往下调用
protected View onCreateView(String name, AttributeSet attrs)
        throws ClassNotFoundException {
    //注意这里，将参数为String prefix的位置赋值成了：android.view，即为源码的view拼接上前缀。为的是后续能
    //正确通过反射调用到该View类的构造方法。需要全路径名。
    return createView(name, "android.view.", attrs);
}
//-----往下调用
public final View createView(String name, String prefix, AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
    Context context = (Context) mConstructorArgs[0];
    if (context == null) {
        context = mContext;
    }
    return createView(context, name, prefix, attrs);
}
```

第二个方法的内部调用和第一个方法的内部调用，最终都调用到了方法：

``` java
public final View createView(@NonNull Context viewContext, @NonNull String name,
                             @Nullable String prefix, @Nullable AttributeSet attrs)
    throws ClassNotFoundException, InflateException {
    Objects.requireNonNull(viewContext);
    Objects.requireNonNull(name);
    Constructor<? extends View> constructor = sConstructorMap.get(name);
    if (constructor != null && !verifyClassLoader(constructor)) {
        constructor = null;
        sConstructorMap.remove(name);
    }
    Class<? extends View> clazz = null;

    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);

        if (constructor == null) {
            // Class not found in the cache, see if it's real, and try to add it
            clazz = Class.forName(prefix != null ? (prefix + name) : name, false,
                                  mContext.getClassLoader()).asSubclass(View.class);

            if (mFilter != null && clazz != null) {
                boolean allowed = mFilter.onLoadClass(clazz);
                if (!allowed) {
                    failNotAllowed(name, prefix, viewContext, attrs);
                }
            }
            constructor = clazz.getConstructor(mConstructorSignature);
            constructor.setAccessible(true);
            sConstructorMap.put(name, constructor);
        } else {
            // If we have a filter, apply it to cached constructor
            if (mFilter != null) {
                // Have we seen this name before?
                Boolean allowedState = mFilterMap.get(name);
                if (allowedState == null) {
                    // New class -- remember whether it is allowed
                    clazz = Class.forName(prefix != null ? (prefix + name) : name, false,
                                          mContext.getClassLoader()).asSubclass(View.class);

                    boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
                    mFilterMap.put(name, allowed);
                    if (!allowed) {
                        failNotAllowed(name, prefix, viewContext, attrs);
                    }
                } else if (allowedState.equals(Boolean.FALSE)) {
                    failNotAllowed(name, prefix, viewContext, attrs);
                }
            }
        }
        //上面的代码做的事情是：判空和对反射构造器做缓存，方便下次再使用。
        //主要看下面调用的方法：
        //为View类的反射方法赋值参数，这里可以看到赋值了两个参数：Context和AttributeSet

        Object lastContext = mConstructorArgs[0];
        mConstructorArgs[0] = viewContext;
        Object[] args = mConstructorArgs;
        args[1] = attrs;

        try {
            //传入参数，调用反射并创建View对象
            final View view = constructor.newInstance(args);
            if (view instanceof ViewStub) {
                // Use the same context when inflating ViewStub later.
                final ViewStub viewStub = (ViewStub) view;
                viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
            }
            return view;
        } finally {
            mConstructorArgs[0] = lastContext;
        }
        //后续为异常处理
    } catch (NoSuchMethodException e) {
        final InflateException ie = new InflateException(
            getParserStateDescription(viewContext, attrs)
            + ": Error inflating class " + (prefix != null ? (prefix + name) : name), e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;

    } catch (ClassCastException e) {
        // If loaded class is not a View subclass
        final InflateException ie = new InflateException(
            getParserStateDescription(viewContext, attrs)
            + ": Class is not a View " + (prefix != null ? (prefix + name) : name), e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;
    } catch (ClassNotFoundException e) {
        // If loadClass fails, we should propagate the exception.
        throw e;
    } catch (Exception e) {
        final InflateException ie = new InflateException(
            getParserStateDescription(viewContext, attrs) + ": Error inflating class "
            + (clazz == null ? "<unknown>" : clazz.getName()), e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

上述源码看50~70行的部分。

可以看到，通过反射，带上Context和AttributeSet参数，创建了View对象并返回。这里的AttributeSet对象则是LayoutInflater.inflate方法中通过xml解析器创建的。

因此一个View从xml文件解析并生成View对象到这里就结束了。

至于后面的View添加到ViewTree，View对提取AttributeSet中的属性（即xml中定义的属性），并根据业务逻辑重写自定义View三大方法，则是自定义View的内容，在此不做拓展。



#### 1.2 由父View创建子View的LayoutParams

子View在xml中定义自己的View的属性。
在LayoutInflator中，子View的属性被从xml中读取到内存中存为key-value。
父View根据子view的key-value属性决定所有的子View的LayoutParams的生成规则。
例如必要的宽和高，以及其他该ViewGroup支持的自定义的属性。



### 2 举例：RecyclerView中的ViewHolder

现在来分析在RecyclerView中调用LayoutInflater创建Item View的场景。


#### 2.1 LinearLayout

默认的LayoutParams？

#### 2.2 GridLayout

默认的LayoutParams？



### 4 总结

结论：

图：

