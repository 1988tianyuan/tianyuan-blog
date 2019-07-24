title: Mongodb索引浅析
tags:
  - mongodb
categories: []
author: ''
date: 2019-01-21 11:16:00
---
mongodb索引数据结果是`b-树`，该数据结构作为索引具有高性能，磁盘io少等特点

### 何为b-树

b-树，又称b树，跟mysql的b+树索引不同，mongodb的b-树索引节点直接与数据绑定，找到索引后直接返回索引对应的document，这种索引适合mongodb这种聚合型nosql数据库：

<!-- more -->

![1547792452619](\blog\images\1547792452619.png)

- b树节点元素个数为k，子节点个数为k+1
- 如果b树的阶树为m，则`m/2 ≤ k ≤ m`
- 查询的时间复杂度为O(log n)，单次查询的磁盘io次数为数的高度

### mongodb索引种类

mongodb索引分为如下

- single index: 单个索引，索引字段只有一个：

  ```javascript
  db.getCollection('user').createIndex({name:1})
  ```

  score是索引字段，1表示升序索引，-1表示降序索引：

- compound indexex: 混合索引，跟mysql类似，也是由多个字段组成索引：

  ```javascript
  db.getCollection('user').createIndex({name:1,gender:1})
  ```

  使用混合索引时，有一些地方需要注意

  - 排序查找的顺序：

    官方文档提到：

    > However, for [compound indexes], sort order can matter in determining whether the index can support a sort operation
    >
    > 对于复合索引，排序字段的顺序对于排序操作是否启用索引非常重要

    对于如下索引：

    ```javascript
    db.getCollection('user').createIndex( { name : 1, gender : -1 } )
    ```

    对姓名作升序索引，对性别做降序索引，那么在进行以下两种排序时，都会启用索引

    ```javascript
    db.getCollection('user').find({}).sort({ name : -1, gender : 1 })
    db.getCollection('user').find({}).sort({ name : 1, gender : -1 })
    ```

    用`explain("allPlansExecution")`查看执行计划发现，totalKeysExamined是所需结果的数目，说明全部查询均走的索引，执行阶段首先执行IXSCAN即扫描索引确定key的位置，然后执行FETCH去检索指定的document。

    但对以下排序，是不会启用索引的：

    ```javascript
    db.getCollection('user').find({}).sort({ name : 1, gender : 1 })
    db.getCollection('user').find({}).sort({ name : -1, gender : -1 })
    ```

    用`explain("allPlansExecution")`查看执行计划发现，totalKeysExamined是0，执行阶段首先执行COLLSCAN即全表扫描，然后进行排序；说明如果排序查找如果跟索引的排序情况不匹配的话，将放弃启用索引。

  - 左子前缀子集：

    这样的索引：`{ name : 1, gender : 1 }`，包含以下的前缀子集：

    ```javascript
    { name : 1}
    { name : 1, gender : 1 }
    ```

    但不包含以下形式:

    ```javascript
    { gender : 1}
    { gender : 1, name : 1 }
    ```

    使用的时候要特别注意能否匹配上索引的前缀子集

- multikey indexes: 多key型索引，索引字段通常是数组类型

- geospatial indexes: 地理位置索引，包括2dsphere和2d还有geoHaystack三种类型

- text indexex: 全文索引，一个document最多只能有一个全文索引，用于在文档中搜索文本，但创建索引的开销比较大，需要后台或离线创建，综合来说不如elasticsearch等搜索引擎

  在文本中subject和comments两个字段建立了全文索引“text”：

  ```javascript
  db.reviews.createIndex(
     {
       subject: "text",
       comments: "text"
     }
   )
  ```

- hashed index：用于分布式collection分片的场景，将不同的document按照某个字段取hash的形式分布到不同的sharding上去：

  > Hashed indexes support [sharding] using hashed shard keys. [Hashed based sharding] uses a hashed index of a field as the shard key to partition data across your sharded cluster.

  对name字段创建一个hashed索引：

  ```javascript
  db.collection.createIndex( { name: "hashed" } )
  ```



### mongodb索引属性

建立索引时可以对索引指定属性，有以下几种属性：

- TTL: 对索引设置过期时间shu，有两种方式

  - 延迟时间：在索引创建后延迟特定秒数然后删除该document：

    ```javascript
    db.collection.createIndex( { "createAt": 1 }, { expireAfterSeconds: 3600 } )
    ```

    createAt字段保存了创建时间，该document会在这个时间的3600秒后被删除

  - 固定时间：在索引创建后延迟特定秒数然后删除该document:

    ```javascript
    db.collection.createIndex( { "expireTime": 1 }, { expireAfterSeconds: 0 } )
    ```

    该document将在到达expireTime上设置的时间时被删除

- unique index：唯一索引，索引字段唯一，若插入包含相同内容字段的document则会报错

  ```javascript
  db.getCollection('user').createIndex({name:1}, {unique: true})
  ```

- spares indexes：稀疏索引，如果设置了稀疏索引，将只对有该索引字段的document启用索引

- partial indexes： 部分索引，跟稀疏索引一样也是非完全索引，在创建索引时可以对索引字段设置一个范围，在这个范围内的才启用索引：

  ```javascript
  db.restaurants.createIndex(
     { cuisine: 1, name: 1 },
     { partialFilterExpression: { rating: { $gt: 5 } } }
  )
  ```

  partialFilterExpression表示范围取值，rating是部分索引的索引字段，需要注意的是，进行查询时，查询范围不能超出部分索引的设定范围，不然无法启用索引

- case insensitive indexes



### mongodb执行计划

通过`explain()`执行计划进行mongodb的性能分析