打开Terminal
![](http://upload-images.jianshu.io/upload_images/7177220-f01db1cb6fa75869.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
输入如下：
`keytool -list -v -keystore D:\签名\AI.jks`
后面的这个部分：`D:\签名\AI.jks`，是密钥文件所存放的路径。

按下回车
![image.png](http://upload-images.jianshu.io/upload_images/7177220-388cf6f2f2888115.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

输入密钥，回车,出结果：

```
D:\AndroidProject\MyTest>keytool -list -v -keystore D:\签名\AI.jks
输入密钥库口令:

密钥库类型: JKS
密钥库提供方: SUN

您的密钥库包含 1 个条目

别名: key0
创建日期: 2017-8-4
条目类型: PrivateKeyEntry
证书链长度: 1
证书[1]:
所有者: CN=William
发布者: CN=William
序列号: 2917aca9
有效期开始日期: Fri Aug 04 12:58:20 CST 2017, 截止日期: Tue Jul 29 12:58:20 CST 2042
证书指纹:
         MD5: 4D:A3:********8C:AC
         SHA1: 81:8A:******71:42
         SHA256: D5:*****9C
         签名算法名称: S**SA
         版本: 3

扩展:

#1: ObjectId: 2.5.29.14 Criticality=false
SubjectKeyIdentifier [
KeyIdentifier [
0000: 70 04 1B 37 0D B2 44 C2   47 21 0C E9 3F B6 FA E5  p..7..D.G!..?...
0010: 08 EE DC E7                                        ....
]
]



*******************************************
*******************************************




```
