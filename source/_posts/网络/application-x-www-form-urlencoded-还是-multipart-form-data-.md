> 主体内容翻译自https://stackoverflow.com/questions/4007969/application-x-www-form-urlencoded-or-multipart-form-data 上的最高赞的回答

---



首先，http请求的内容如下：

``` 
request line
headers
request body
```

其中，request line部分的url必须以`application/x-www-form-urlencoded`方式编码

request body的编码方式由头部的Content-Type指定。



以下，开始翻译stackOverFlow的`application/x-www-form-urlencoded or multipart/form-data?`问题下的第一个回答。

---

## 翻译

总结：如果你有二进制的数据（即非字母或者数字，或者非常大的一段数据）要传输，那么使用`multipart/form-data`。否则，使用`application/x-www-form-urlencoded`。

分析：`multipart/form-data`和`application/x-www-form-urlencoded`是http post请求中，user-agent（浏览器或者客户端）必须支持的两种MIME type，这两种多媒体类型的请求的目的是去发送name/value对的一串列表到服务端。在传输的数据的类型和大小不同的情况下，以上两种方法，一种会比另一种更有效率。要理解为什么，你必须看一下这两种编码类型的内部实现。

`multipart/x-www-form-urlencoded`：被发送到服务端的http消息的body在本质上是一个巨大的查询字符串：name/value对被`&`符号分开，name和value被`=`符号分开，例如：

`MyVariableOne=ValueOne&MyVariableTwo=ValueTwo`

根据[specification](https://www.w3.org/TR/html401/interact/forms.html) 

> 非字母和数字的字符会被`%HH`来代替，一个百分比符号和两个16位进制的数字代表着那个字符的ASCII码

这意味着每一个在value中的非字母和数字的字节，都将被3个字节的数据代替。如果是一个大的二进制的文件，那么3倍的传输数据将会变得十分低效率。

这时`multipart/form-data`就该出现了。用这种方法来传输name/value对，每一个键值对都代表着MIM消息中的一个`part`。不同给的`part`被一个独特的字符串界线分割（该字符串分界线需要独特地生成，以便不会在任何value中出现）。每一个part有他自己的一系列MIME 头部，例如`Content-Type`，并且特别低，需要有`Content-Disposition`，来给每一个`part`他自己的“名字”。每一个name/value对的value是每一个MIME消息中的part的有效负载。MIME spec给了我们更多的选项，比如，我们可以选择一个更加高效率的数据编码方式来节约流量（eg.base64 或甚至 原生二进制）。

那么为什么不一直用`multipart/form-data`呢？因为：对于一些短的字母或数字的value（大多数web表单都是），添加所有MIME头的开销将大大超过从更有效的二进制编码中节省的任何开销。
