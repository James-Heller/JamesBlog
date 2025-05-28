---
title: Java泛型擦除下的MyBatis TypeHandler挑战与解决方案
published: 2024-06-20
description: 探讨在使用Mybatis时遇到的TypeHandler与List相关的问题及解决方案
tags: ['编程','Kotlin', 'MyBatis','MyBatisPlus']
category: 'Web开发'
draft: false 
---

## 引言


在日常的开发工作中，我们常常面临各种各样的技术挑战。本文将分享一次因Java泛型擦除特性引发的MyBatis TypeHandler失效问题，并探讨其背后的原理以及如何优雅地解决问题。


##  背景



  在项目中，我遇到了一个令人头疼的Bug——调用API时，偶然会触发`java.lang.NumberFormatException`异常，错误信息指出***“XX市”***无法被解析为数字。这个问题出现在处理从数据库读取的字符串字段，这些字段原本存储的是数字ID或字符串数组，以逗号分隔的形式保存在VARCHAR列中。


## 问题复现

 为了存储和读取这些数组，我编写了两个TypeHandler：

1. `String2IntListHandler`：负责将字符串转换为整数列表。
2.  `StringToStringListHandler`：将字符串转换为字符串列表。
这两个TypeHandler分别用于处理数字ID和字符串数组的读写。通过在实体类字段上使用`@TableField`注解指定对应的TypeHandler，一切看起来运作良好。

## 遭遇挫折
在第一次重启后这个问题莫名其妙的消失了，第二次重启也没有发生。但是接下来一次系统重启后，看似已解决的问题再次浮现。前端调用相关接口时，返回了500错误，并重现了NumberFormatException异常。深入分析调用栈，发现原本应由StringToStringListHandler处理的字段，却意外地被String2IntListHandler接管，这显然不符合预期。

## 根本原因

  问题的症结在于Java的泛型擦除机制。在运行时，JVM会消除所有泛型信息，这意味着`List<String>`和`List<Int>`在运行时都表现为List。因此，MyBatis无法区分两者，导致TypeHandler应用混乱。

## 解决方案
面对这一挑战，我想到了两种可行的解决方案。 *因为公司项目赶得紧，所以我没有时间尝试从根源上解决问题.*：

1. 采用JSON存储：将数据格式改为JSON存储，利用数据库的JSON类型支持。通过将TypeHandler更改为`JacksonTypeHandler`，可以有效避免类型混淆。

2. 设计通用TypeHandler：引入一个`GenericListHandler`，并在数据库中使用魔数前缀（如4396表示数字数组）来标识不同类型的数据。实体类中的字段类型统一改为List<Any>，在运行时根据魔数进行类型判断和强制转换。

## 结论
遇到由于Java泛型擦除引起的MyBatis TypeHandler失效问题时，可考虑使用JSON类型存储或设计更加灵活的通用TypeHandler。这些策略不仅能规避泛型擦除带来的困扰，还能增强代码的健壮性和可维护性。

## 后记
开发过程中，对细节的把控至关重要。通过此次经历，不仅加深了我对Java泛型擦除特性的理解，也提醒了我在设计系统架构时要充分考虑运行时的兼容性和安全性。

***