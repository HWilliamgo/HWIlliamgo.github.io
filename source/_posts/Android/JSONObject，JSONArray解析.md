参考博客：[[李双喆的JSONObject、JSONArray](http://blog.csdn.net/lishuangzhe7047/article/details/28880009)](http://blog.csdn.net/lishuangzhe7047/article/details/28880009)
>这玩意弄得我迷迷糊糊的，今天终于搞明白了，记录之。

### 看图先

![](http://upload-images.jianshu.io/upload_images/7177220-6e7ab4169b5a9b4a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
首先搞明白两个概念：
* JSONObject是用｛｝来表示的，没有｛｝不是JSONObject。
* JSONArray使用[]来表示的，没有[]不是JSONArray。

解释一下上图，比如上图，直接出来一个[ ]，[ ]就是数组，数组里有两个元素，也就是两个{}，也就是两个JSONObject。注意这里第二个{}是空的。

好的，整明白了之后，比如我们要拿到键name4对应的值value2时，怎么操作？
```
                    JSONArray jsonArray=new JSONArray(content);
                    //jsonArray=[{name1:{name2:{name3:"value1",name4:"value2}}},{}]
                    JSONObject object1=jsonArray.getJSONObject(0);
                    //object1={name1:{name2:{name3:"value1",name4:"value2"}}}
                    JSONObject object2=object1.getJSONObject("name1");
                    //object2={name2:{name3:"value1",name4:"value2"}}
                    JSONObject object3=object2.getJSONObject("name2");
                    //object3={name3:"value1",name4:"value2"}
                    String value2=object3.getString("name4");
```
以上是详细解剖，快速的这样：
```
JSONArray jsonArray=new JSONArray(content);
String value2=jsonArray.getJSONObject(0).getJSONObject("name").getJSONObject("name2").getString("name4");
```
---
**以上是一个抽象的例子，我们来搞个实战：**

解析一个天气数据：
[和风天气的API文档](https://www.heweather.com/documents/api/v5/now)

JSON如下：
```
{
    "HeWeather5": [
        {
            "basic": { //基本信息
                "city": "北京",  //城市名称
                "cnty": "中国",   //国家
                "id": "CN101010100",  //城市ID
                "lat": "39.904000", //城市维度
                "lon": "116.391000", //城市经度
                "prov": "北京",  //城市所属省份（仅限国内城市）
                "update": {  //更新时间
                    "loc": "2016-08-31 11:52",  //当地时间
                    "utc": "2016-08-31 03:52" //UTC时间
                }
            },
            "now": {  //实况天气
                "cond": {  //天气状况
                    "code": "104",  //天气状况代码
                    "txt": "阴"  //天气状况描述
                },
                "fl": "11",  //体感温度
                "hum": "31",  //相对湿度（%）
                "pcpn": "0",  //降水量（mm）
                "pres": "1025",  //气压
                "tmp": "13",  //温度
                "vis": "10",  //能见度（km）
                "wind": {  //风力风向
                    "deg": "40",  //风向（360度）
                    "dir": "东北风",  //风向
                    "sc": "4-5",  //风力
                    "spd": "24"  //风速（kmph）
                }
            },
            "status": "ok"  //接口状态
        }
    ]
}
```
我要拿到的数据是"city":"北京"
```
JSONObject jsonObject=new JSONObject(content);
//因为返回的数据整个就是用｛｝包裹的，所以先new一个JSONObject
String city=jsonObject.getJSONArray("HeWeather5").getJSONObject(0).getJSONObject("basic").getString("city");
//稳了
```
稳了

