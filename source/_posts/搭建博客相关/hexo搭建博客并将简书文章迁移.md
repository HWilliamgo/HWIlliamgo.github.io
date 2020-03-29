---
title: hexo搭建博客并将简书文章迁移
date: 2020-03-29 14:31:21
tags: 
- 博客
index_img: https://s1.ax1x.com/2020/03/29/GVGaOe.md.jpg
banner_img: https://s1.ax1x.com/2020/03/29/GVGaOe.jpg
---

在经过多次的考虑后，我还是打算将所有的博客的数据由自己维护。因此找了一种比较简单的方式：hexo + github pages



### 迁移博客的原因

1. 简书的博客没有文章的导航栏，我每次用Markdown写文章发布后，简书不会自带文章大纲，无法检索。
2. 简书的曾经被叫停过一次，在那之后，简书的访问速度变慢了，我担心后面简书会挂掉，我的文章拿不出来。
3. 简书的文章类型繁多，不是技术型的平台。当然我也看了其他很优秀的技术型平台，例如掘金和语雀。但是掘金不支持文章导出，语雀导出的文章貌似只有他们自家的语雀平台能够二次使用，不是Markdown类型的。因此放弃在第三方平台写博客了。
4. 自己搭建博客，自己选主题和功能，自己保存数据，一劳永逸。

### 迁移博客的过程

1. 参考[hexo官方文档](https://hexo.io/zh-cn/docs/)，从0到1，并没有什么难度。
2. 选主题：[fluid](https://github.com/fluid-dev/hexo-theme-fluid)
3. 上线博客
4. 迁移文章（这是最麻烦最重要的一步）



这里打算就第四点，迁移文章，说一说我遇到的问题：



#### 1. 文章根据目录分类

首先，简书的文章下载下来是按照目录来划分分类的。而Hexo是不支持目录划分分类的，hexo文章的分类只能在对应文章的front-matter里面标记上这篇文章的category是什么才行。

所幸有这么一个插件能解决这个问题：[hexo-auto-category](https://github.com/xu-song/hexo-auto-category)



#### 2. 文章没有生产日期

简书的文章下载下来的时候，除了文章标题和内容外，没有附带任何元数据。例如文章的创建日期、观看数、点赞数。

文章的观看数和点赞数这个东西，在hexo里面我不太清楚如何去发挥作用，能带走的就只有文章的创建日期了。不过创建日期他也没有给你，但是没有文章创建日期的话，所有的文章都会按照创建日期来显示，这显然很不合理。

因此我想到用JSoup去爬取简书的数据，看看能不能把我对应的文章的日期爬下来。



#### 3. 给每篇文章附上front-matter

日期爬下来之后，还要给每篇文章附上[front-matter](https://hexo.io/zh-cn/docs/front-matter)。

我有200多篇博客，不可能我自己一篇一篇手动去搞，因此还得写个脚本，往文章顶部插入front-matter。

脚本不难，用Kotlin写的，难的是使用JSoup和调试的过程。代码如下：

``` kotlin
import org.json.JSONArray
import org.json.JSONObject
import org.jsoup.Jsoup
import java.io.File
import java.io.FileReader
import java.io.RandomAccessFile
import java.lang.RuntimeException

/**
 * author: hwj
 * description:
 */

fun main() {
    val allEssay = takeTimeFromJianShuByJSoup()
    addHeadForMD(allEssay)
}

fun takeTimeFromJianShuByJSoup(): AllEssay {
    //https://www.jianshu.com/nb/16737585?order_by=added_at&page=2

    val categoryIdPairList = getCategoryAndId()

    //请求每一个分类下的每一篇文章的 标题和时间，并绑定。
    val allEssay = AllEssay()

    categoryIdPairList.forEach {
        val categoryId = it.id
        val categoryName = it.name

        //page 从1开始计数,遍历当前category下的所有的文章
        var currentPage = 1
        var tmpList = parseByJsoup(categoryId, currentPage, categoryName)
        while (tmpList.isNotEmpty()) {
            currentPage++
            tmpList.forEach { essay ->
                allEssay.put(categoryName, essay)
            }
            tmpList = parseByJsoup(categoryId, currentPage, categoryName)
        }
    }
    return allEssay
}

fun parseByJsoup(id: String, page: Int, categoryName: String): List<Essay> {
    println("======类别：$categoryName")
    val result: MutableList<Essay> = ArrayList()

    val url = "https://www.jianshu.com/nb"
    val finalUrl = "$url/$id?order_by=added_at&page=$page"
    val document = Jsoup.connect(finalUrl).get()
    document.getElementsByTag("li").forEach { li ->
        val div = li.getElementsByTag("div")
        div.forEach { eachDiv ->
            if (eachDiv.attr("class").trim() == "content") {
                val title = eachDiv.getElementsByTag("a").first().text()
                var time = eachDiv
                    .getElementsByTag("div").first()
                    .getElementsByTag("span").filter {
                        it.attr("class").trim() == "time"
                    }.first().attr("data-shared-at")
                time = timeFormatConvert(time)

                println("标题：$title 创建日期：$time")
                result.add(Essay(title, time))
            }
        }
    }
    return result
}

fun getCategoryAndId(): List<CategoryIdPair> {
    val jsonFilePath = "/Users/HWilliam/IdeaProjects/test/src/main/kotlin/category.json"
    val rootJson = JSONObject(File(jsonFilePath).readText())
    val arrays = rootJson.getJSONArray("notebooks")

    //<id,CategoryName>
    val list: MutableList<CategoryIdPair> = ArrayList()

    for (i in 0 until arrays.length()) {
        val id = arrays.getJSONObject(i).getInt("id").toString()
        val name = arrays.getJSONObject(i).getString("name")
        list.add(CategoryIdPair(id, name))
    }
    return list
}

fun timeFormatConvert(i: String): String {
    val input = i.replace("-", "/")
    val date = input.split("T")[0]
    val time = input.split("T")[1].split("+").first()
    return "$date $time"
}

fun addHeadForMD(allEssay: AllEssay) {
    val essayParentDir = "/Users/HWilliam/IdeaProjects/test/src/main/kotlin/_posts"
    val essayParentDirFile = File(essayParentDir)

    essayParentDirFile.listFiles()?.forEach {
        //it --> 目录文件
        println(it.name)
        it.listFiles()?.forEach { oneMDFile ->
            //文件名字
            val realFileNameOfMD = oneMDFile.name
            val fileNameWithoutMD = realFileNameOfMD.substringBefore(".")
            val tmpFileNameOfFinalMDFile = realFileNameOfMD + "tmp"

            val date = allEssay.findTime(it.name, fileNameWithoutMD)
            val headerToAppend = "---\ntitle: ${fileNameWithoutMD}\ndate: $date\n---\n\n"

            val resultFile = File(it, tmpFileNameOfFinalMDFile)

            val randomAccessFileOfResult = RandomAccessFile(resultFile, "rw")
            //写header
            randomAccessFileOfResult.write(headerToAppend.toByteArray())

            //将所有原文件的内容追加进去
            val randomAccessFileOfTarget = RandomAccessFile(oneMDFile, "rw")
            val buffer: ByteArray = ByteArray(10240)
            while (randomAccessFileOfTarget.read(buffer) != -1) {
                randomAccessFileOfResult.write(buffer)
            }

            randomAccessFileOfResult.close()
            randomAccessFileOfTarget.close()

            oneMDFile.delete()
            resultFile.renameTo(oneMDFile)
        }
    }
}

class CategoryIdPair(val id: String, val name: String)

class Essay(val title: String, val time: String)

class AllEssay() {
    // <Category , essayList>
    val map: HashMap<String, MutableList<Essay>> = HashMap()

    fun put(category: String, essay: Essay) {
        if (!map.containsKey(category)) {
            map.put(category, ArrayList())
        }
        map.get(category)?.add(essay)
    }

    fun findTime(category: String, essayName: String): String {
        var failEssayName = ""
        map.get(category)?.forEach {
            if (it.title == essayName) {
                return it.time
            } else {
                val tmpTitle = it.title
                val replaceEnd = tmpTitle
                    .replace(".", "-")
                    .replace("?", "-")
                    .replace("？", "-")
                    .replace("/", "-")
                    .replace("。", "-")
                    .replace(" ", "-")
                if (replaceEnd == essayName) {
                    return it.time
                } else {
//                    println("文章无时间：$category -> $essayName ， 真实name=${it.title}")
                }
            }
        }
//        throw RuntimeException("没有找到时间 $category, $essayName")
        println("文章无时间：$category -> $essayName ， 真实name=${failEssayName}")
        return ""
    }
}


```



#### 4. 给博客选定一些图片和样式，以及集成评论功能。





### 迁移博客的结果

博客地址：https://hwilliamgo.github.io/



迁移博客的工作告一段落，剩下的是为博客添加一些个性化的样式之类的。

此外，由于github访问速度过慢的原因，在考虑是否将博客托管到国内的平台，例如gitee pages。不过这是后话了。