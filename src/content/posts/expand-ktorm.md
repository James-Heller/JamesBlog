---
title: 扩展Ktorm
published: 2025-08-29
description: '让Ktorm在写流水账的时候变得好用'
image: ''
tags: ['Kotlin', 'Ktorm', 'ORM']
category: 'Web开发'
draft: false 
lang: ''
---

## 引言

自从在公司内全面推行了3K开发后。对于Ktorm的理解也越发透彻，使用起来也得心应手。今天突然想起来Ktorm并不自带基础的CRUD方法，可能对于刚开始使用的同僚们不是很友善，遂写了这个博文。我的实现方法肯定不是最好的，在这里写出来抛砖引玉吧。

## 目标

Ktorm 是一个轻量级的 Kotlin ORM 框架，提供了强大的 DSL 查询能力和实体映射功能。然而，在实际项目中，我们常常需要为数据库表添加一些通用的字段（如 id、createTime、updateTime、deleted 等）以及通用的增删改查逻辑。为了减少重复代码并提高开发效率，我设计并实现了一个基于 Ktorm 的扩展框架，包含 `EnhanceEntity` 接口和 `EnhanceTable` 抽象类。本文将详细介绍这一扩展的设计思路、实现细节以及使用方法。

## 核心实现

在我看了很久文档之后，我发现Ktorm的设计非常的符合面对对象的开发理念。所以我可以通过继承来实现我所需要的所有功能。

### 1. EnhanceEntity 接口

```kotlin
interface EnhanceEntity<E : Entity<E>> : Entity<E> {
    var id: Int? //因为在数据正式入库前自增ID肯定是null的,所以此处是可null变量
    var createTime: LocalDateTime
    var updateTime: LocalDateTime
    var deleteTime: LocalDateTime?
    var deleted: Boolean
}
```

这些字段覆盖了最基础最常见的的表结构需求：

- id：支持整型主键。

- createTime 和 updateTime：记录时间戳，便于追踪记录的生命周期。

- deleteTime：记录逻辑删除时间，可为空。

- deleted：布尔值，用于标记记录是否被逻辑删除。

### 2. EnhanceTable 抽象类

`EnhanceTable`是功能扩展的核心，它继承了Ktorm的Table类， 并为每个表定义了标准字段和通用CRUD方法。以下是核心内容。

```kotlin
abstract class EnhanceTable<E : EnhanceEntity<E>>(tableName: String, private val primaryKeyName: String = "id"): Table<E>(tableName){
    val id = int(primaryKeyName).primaryKey().bindTo { it.id }
    val deleted = boolean("deleted").bindTo { it.deleted }
    val createTime = datetime("create_time").bindTo { it.createTime }
    val updateTime = datetime("update_time").bindTo { it.updateTime }
    val deleteTime = datetime("delete_time").bindTo { it.deleteTime }

    abstract val sequence: EntitySequence<E, EnhanceTable<E>>

    @JvmName("get-sequence")
    private fun getSequence(): EntitySequence<E, EnhanceTable<E>> {
        return sequence

    }
}
```

- **字段映射**：通过 Ktorm 的 bindTo 方法，将表字段与 EnhanceEntity 的属性绑定。

- **灵活主键**：允许通过 primaryKeyName 参数自定义主键名称，默认为 id。

#### 为什么需要显式覆盖 sequence？

在实现 EnhanceTable 的子类时，必须显式定义 sequence 属性，例如：

```kotlin
object Fabrics : EnhanceTable<Fabric>("fabrics") {
    val name = varchar("name").bindTo { it.name }
    override val sequence: EntitySequence<Fabric, EnhanceTable<Fabric>> get() = DB.pg.sequenceOf(this)
}
```

测试表明，如果在 EnhanceTable 中提供默认 sequence 实现（如 get() = DB.pg.sequenceOf(this)），Ktorm 生成的 EntitySequence 只会包含基类的字段（id、deleted 等），而子类的业务字段（如 name）会被忽略。这是因为基类中的 this 被解析为 `EnhanceTable<E>` 类型，而不是子类类型（如 Fabrics）。

通过在子类中覆盖 sequence，DB.pg.sequenceOf(this) 使用子类的具体类型，确保所有字段被正确包含。getSequence 方法进一步封装了对 sequence 的访问，避免直接操作属性，并通过 @JvmName("get-sequence") 注解解决 JVM 命名冲突问题。

## 扩展功能

最核心的内容到此已经结束了，接下来要做的就是写一套通用的CRUD方法了。这些方法都放在我们的`EnhanceTable`中。 后面我只会

- 通过ID查找
  
  ```kotlin
  open fun selectByIdInSequenceStyle(id: Int): E?{
      //其实这里有两种风格，一种是用Entity实现一种是Typed SQL。
      return sequence.find {  (it.id eq id) and (it.deleted eq false) }
  }
  
  open fun selectByIdInSQLType(id: Int): E? {
      return database.from(this)
              .select()
              .where((it.id eq id) and (it.deleted eq false))
              .map(this::createEntity)
  }
  ```

- 批量插入
  
  这应该是唯一需要特别处理的地方了，受到了Ktorm作者的启发我的实现方法是这样的。
  
  ```kotlin
  @Suppress("UNCHECKED_CAST")
  open fun batchCreate(entities: List<E>): IntArray {
      val now = LocalDateTime.now()
      database.useTransacation{
          val effectedRows = database.batchInsert(this){
              item{
                  for(column in this@EnhanceTable.columns){
                      val v = with(EntityExtensionsApi()) { entity.getColumnValue(column.binding!!) }
                      //这里根据你的实际需要可以对插入数据的字段做任意处理
                      when(column.name){
                          primaryKeyName -> continue;
                          "create_time" -> set(column as Column<LocalDateTime>, now)
                          "update_time" -> set(column as Column<LocalDateTime>, now)
                          "deleted" -> set(column as Column<Boolean>, false)
  
                          else -> set(column as Column<Any>, v)
                      }
                  }
              }
          }
          //effectedRows是批量语句返回的每一个单语句所影响的行数
          //通过抛出错误触发回滚
          if(effectedRows.sum() != entities.size){
              throw RuntimeException("批量插入没有全部成功")
          }
      }
  
  }
  ```

## 结语

通过设计 EnhanceEntity 接口和 EnhanceTable 抽象类，我们为 Ktorm 提供了一套简单易用的扩展框架，极大地减少了重复代码的编写，同时满足了常见的 CRUD 需求和表结构设计。这套方案不仅让新手能够快速上手 Ktorm 的开发，也为有经验的开发者提供了灵活的扩展空间。无论是单条查询还是批量插入，这些通用方法都能在实际项目中显著提升开发效率。

当然，这只是一个基础实现，实际应用中可能需要根据业务需求进一步优化，比如增加更多的 CRUD 方法、支持更复杂的查询条件，或是针对特定场景调整字段逻辑。希望这篇文章能为使用 Ktorm 的同僚们提供一些灵感。如果您有更好的实现方式或改进建议，欢迎留言讨论，共同完善这套扩展框架，让 Ktorm 在我们的项目中发挥更大的价值！
