> 本篇文章建立在看过一遍OkHttp源码中的拦截器部分的基础上。

## OkHttp回顾

OkHttp的拦截器工作原理如下图，每一个拦截器都有机会处理上一个拦截器传下来的Request或者下一个拦截器返回上来的Response，即可任意地进行前处理和后处理。在最后一个拦截器CallServerInterceptor中进行真正的网络链接，成功后构造Response对象并发布，一层一层地传递到开发者手上。

![](https://upload-images.jianshu.io/upload_images/7177220-b466a27ff2bd90c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/400)

![](https://upload-images.jianshu.io/upload_images/7177220-646c4c0dd0fc126b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/250)

## 仿写拦截器

> 在实战开发中，仅就涉及数据处理这一方面，拦截器就有非常好的设计效果，OkHttp这一套是可以照搬到很多地方的，拦截器这一套设计实战性非常高，故在这里动手撸一个简易版。

### 1. 定义Request和Response

``` java
/**
 * 作为责任链上一直被传递的对象，从构造开始，一直传递到最后一个拦截器并被依次捕获和处理
 */
public class Request {
    public int i = 0;
    public int j = 1;
}

/**
 * 在最后一个拦截器中发生构造，并被依次返回到责任链的递归，并最终返回给应用层读取。
 */
public class Response {
    public String param1;
    public String param2;
}
```

1. 请求和响应是最先定义的，实际上这叫做输入和输出，Input和Output，只是刚好在网络请求这里叫做Request和Response，我按照OkHttp的来，也这样写了。

2. Request和Response要按照实际业务的需求来定义就行，只要记住一个是责任链的输入，一个是责任链的输出。

### 2.定义拦截器和责任链接口

``` java
public interface Interceptor {

    //拦截
    Response intercept(Chain chain);

    interface Chain {
        //获取请求
        Request request();

        //处理请求，该方法由拿到了Chain对象引用的拦截器来调用。
        Response proceed(Request request);
    }
}
```

### 3. 实现责任链接口的封装

```java
/**
 * 实际上只是对以下3个变量的一个封装。
 */
public class RealInterceptorChain implements Interceptor.Chain {
    //自始至终被传递的请求
    private Request mRequest;
    //拦截器容器
    private List<Interceptor> mInterceptors;
    //拦截器容器的下标。
    private int mIndex;

    public RealInterceptorChain(List<Interceptor> interceptors, int index, Request request) {
        mRequest = request;
        mInterceptors = interceptors;
        mIndex = index;
    }

    @Override
    public Request request() {
        return mRequest;
    }

    @Override
    public Response proceed(Request request) {
        //该index处不再有拦截器，异常
        if (mIndex >= mInterceptors.size()) throw new AssertionError();

        //构造责任链，将下标+1，其余不变。
        RealInterceptorChain next = new RealInterceptorChain(mInterceptors, mIndex + 1, mRequest);
        //获取当前下标的拦截器
        Interceptor interceptor = mInterceptors.get(mIndex);
        //将变量交给拦截器处理，由拦截器自己转发给下一个拦截器
        return interceptor.intercept(next);
    }
}
```

### 4. 完成

> 一共2个接口，3个类。

拦截器的责任链模式就这么简单。

后面的封装就是锦上添花的东西了，比如再写几个具体的拦截器，把整个拦截器的功能隐藏起来，只暴露少数的接口给外部等等。总之也是按照OkHttp源码那一套写就不会错。



## 使用

这是在没有任何封装的条件下的直接new出两个拦截器实例来测试。

1. 构造Request
2. 构造拦截器容器
3. 添加内部的或外部的拦截器
4. 构造第一个Chain对象，交给第一个拦截器，发起责任两从上到下的处理。
5. 拿到处理后的Response，开发者进行处理。

实际上这里面的第2，3，4步在OkHttp里面都是封装了的。当然我们自己实现的时候肯定也得封装。

``` java
import interception.Interceptor;
import interception.RealInterceptorChain;
import interception.Request;
import interception.Response;

import java.util.ArrayList;
import java.util.List;

public class Main {
    public static void main(String[] args) {
        //1.构造Request
        //2.构造拦截器容器
        //3.添加内部的或外部的拦截器
        //4.构造第一个Chain对象，交给第一个拦截器，发起责任两从上到下的处理。
        //5.拿到处理后的Response，开发者进行处理。
        
        //1
        Request request = new Request();

        //2
        List<Interceptor> interceptors = new ArrayList<>();

        //3
        interceptors.add(chain -> {
            //修改Request
            Request r = chain.request();
            r.i = 100;
            r.j = 99;
            //将request向责任链下部传递。
            return chain.proceed(r);
        });
        interceptors.add(chain -> {
            Request r = chain.request();
            //最后一个Chain对象只有request方法起作用，用来获取Request，而不会调用proceed，在最后一个拦截器中生成Response直接返回而不再递归
            Response response = new Response();
            if (r.i == 100) {
                response.param1 = String.valueOf(r.i);
            }
            if (r.j == 99) {
                response.param2 = String.valueOf(r.j);
            }
            return response;
        });
        
        //4
        RealInterceptorChain chain = new RealInterceptorChain(interceptors, 0, request);
        Response response = chain.proceed(request);
        
        //5
        System.out.println(response.param1);
        System.out.println(response.param2);

    }
}
//打印：
//100
//99
```

在这个测试里面有两个拦截器，第一个拦截器对Request的两个参数进行了赋值，一个100一个99，并没有处理下面返回的Response。第二个拦截器对Request的值进行了检查，生成了Response对象并赋值返回。过程如下图：

![](https://upload-images.jianshu.io/upload_images/7177220-c86afc4540c52472.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



全文源码：https://github.com/HWilliamgo/EasyInterceptorChain/tree/master
