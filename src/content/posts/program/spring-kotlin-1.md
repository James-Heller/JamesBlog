---
title: 关于Mybatis的TypeHandler与List<Any>
published: 2024-06-20
description: ''
image: './mybatis-superbird-small.png'
tags: ['编程','Kotlin', 'MyBatis','MyBatisPlus']
category: 'Web开发'
draft: false 
---

## 启


近日在我每天的黑奴生涯中碰见了一个很扭曲的Bug，具体表现为 调用接口时会随机触发 ***java.lang.NumberFormatException： For input string "XX市"。***
具体需求是这样的。
  有一个由纯数字组成的ID数组需要存到数据库中。我使用varchar进行储存，每个ID之间使用 英文字符逗号进行分隔。在服务中写了一个`String2IntListHandler` 用来处理这种情况。核心处理部分如下。

``` Kotlin
private fun transform(rawData: String): List<Int>{
    return rawData.split(",").map{ it.toInt() }
}
    
```
   目前一切都很顺利。直到后面又出现了一个类似的写法。 具体如下。

   有一个字符串数组存入数据库之中，同样使用varchar储存，用英文逗号分隔。同样的，我也写了一个相似的TypeHandler `StringToStringListHander`  它的实现逻辑也差不多。

``````kotlin
private fun transform(rawData: String): List<String>{
    return rawData.split(",")
}
``````



接下来我只需要按照在我想要做处理的字段上面使用 `@TableField(typeHandler = String2IntList::class)`  或者 `@TableField(typeHandler = String2StringList::class)`  就能达到我想要的效果。

好， 至此 需求完成。 此时我还不知道已经埋下了一个大雷。



##  承



​    起初，一切都很完美，事发在我做完别的部分并重启服务之后，前端调用前述的数据接口时拿到了返回值 `500`  爆出了开头所出现的错误。

​    经过分析调用栈，我发现 经过字段映射时本应经过`String2StringListHandler` 的字段竟然奇奇怪怪的经过了`String2IntListHandler` 。 问题分析成功，着手解决。 ~~乐，解决不了，摆烂。😅~~ 

   重启能解决大部分问题，重启，mvn clean  mvn compile。问题消失，问题解决。 



***