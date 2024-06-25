---
title: 启示：委托模式
published: 2024-06-25
description: ''
image: ''
tags: ['编程', '设计模式','Kotlin',]
category: 'Web开发'
draft: false 
---



## 什么是委托模式

&emsp;作为一个狂热的Kotlin粉丝， 在工作摸鱼或者是闲暇时间研究Kotlin是我的乐趣所在。 今日在网上冲浪时看到一个关于Kotlin高级特性的文章。其中提到了关键字 `by`。 这引起了我的兴趣，经过相关搜索发现`by`是用于实现委托的关键字。
&emsp;那么问题就来了，什么是委托？
***在面向对象编程中，设计模式是帮助我们解决常见问题的模板。委托模式就是设计模式其中的一种。***

### 委托模式的核心

&emsp;Kotlin的委托模式基于一个简单的原则：***“不要自己重复做，让别人帮你做。”*** 具体到编程，就是让一个对象接收一个接口的实现，并将接口方法的调用委托给另一个对象。

### 委托模式能做什么

&emsp;对于我这种通过具体场景来学习知识的人，在不知道所学的东西是做什么的情况下，是很难学习下去的。
> "学习任何知识的最佳途径都是亲自动手去做。" —— 理查德·费曼

&emsp;那么我们来考虑这样一个场景：你有一个`Logger`接口用来生成日志和一个`ConsoleLogger`类实现了这个接口。现在，万恶的产品经理来了。
>“为了保证任何对于数据的操作都是可溯源的，我们需要将所有修改订单的操作在写入到一个文件中。”
> ———— 产品经理如是说

*我知道这是一个不太恰当的例子因为重定向IO流可以更快捷的实现这个需求。*
&emsp;使用委托模式，你可以创建一个新的`LoggingPrinter`类，该类在执行打印任务前后添加日志记录，而不改变`ConsoleLogger`类的代码。

```kotlin
interface Logger {
    fun log(msg: String)
}

class ConsoleLogger : Logger {
    override fun log(msg: String) {
        println("Printing content...")
    }
}

class FileLogger(private val logger: Logger) : Logger by logger {
    override fun log(msg: String) {
        
        logger.print(msg: String)
        
        // New code here
        val fileName = "Info.log"
        val path = Paths.get(fileName)
        try{
            Files.writeString(filePath, msg + "\n",  StandardCharsets.UTF_8, StandardOpenOption.CREATE, StandardOpenOption.APPEND)
        }catch(e: IOException){
            e.printStackTrace()
        }

    }
}
```

在这个例子中，`FileLogger`通过委托`logger`对象，接管了`Logger`接口的`log`方法，并在打印前后添加了日志记录。这样的设计不仅保持了`ConsoleLogger`的简洁性，也使得日志功能的添加变得非常灵活和可重用。 只需要在需要将日志写入文件的模块用`FileLogger`
替代`ConsoleLogger`即可完成将对应模块的日志写入文件而不影响其他使用ConsoleLogger的模块。

### 委托的优势

1. **减少代码重复**：委托允许我们复用现有的代码，只添加我们需要的功能改变，从而避免了代码的重复。
2. **简化继承结构**：通过使用委托而不是继承，我们可以避免创建复杂的类层次结构，这在长远来看使代码更容易理解和维护。
3. **提高代码的灵活性**：委托模式使得更换组件或改变行为在运行时成为可能，从而增加了程序的灵活性。

### 结语

Kotlin的委托模式提供了一种强大的方法来扩展和增强现有的代码。无论是添加日志功能，还是引入新的行为，委托都能以最小的代价来实现功能的增强。通过这种模式，Kotlin开发者可以编写出更清晰、更灵活、更易于维护的代码。
