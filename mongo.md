## mogodb基本操作指令

#### 查看数据库列表

```sh
> show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
py3     0.000GB
```



#### 创建数据库

不需要创建数据库，只需要直接切换到对应的数据库即可。（但是如果不创建任何集合，则不会生成数据库）

```shell
> use py3
switched to db py3
```

#### 创建集合（对应于mysql中的表概念）

```shell
> db.createCollection("stu")
{ "ok" : 1 }
```

#### 插入数据

```shell
> db.stu.insert({name:'hr', age:18})
```

#### 更新数据

如果不使用$set的话，会直接覆盖整条数据。

如果不使用{multi:true}的话，只会更新匹配的第一条数据。

```shell
db.stud.update({},{$set:{name:'xiaolizi'}}, {multi:true})
```

#### 查询数据

- 查询所有数据

```sh
db.find()
```

- 查询一条数据

```sh
> db.stu.findOne()
{
        "_id" : ObjectId("62c2436229d727582aa383cf"),
        "name" : "xiaolizi",
        "gender" : true
}
```

- 对find的结果进行格式化

```sh
> db.stu.findOne().pretty()
2022-07-05T15:51:22.766+0800 E QUERY    [thread1] TypeError: db.stu.findOne(...).pretty is not a function :
@(shell):1:1
> db.stu.find().pretty()
{
        "_id" : ObjectId("62c2436229d727582aa383cf"),
        "name" : "xiaolizi",
        "gender" : true
}
{
        "_id" : ObjectId("62c243e329d727582aa383d0"),
        "name" : "xiaolizi",
        "age" : 18
}
```

- 运算符，使用的运算符是字母型运算符

```sh
> db.stu.find({age:{$gt:16}})
{ "_id" : ObjectId("62c243e329d727582aa383d0"), "name" : "xiaolizi", "age" : 18 }
```

- 逻辑运算符，“与”运算直接指定多个字段即可，“或”运算必须使用$or
- 或

```sh
> db.stu.find({$or:[{age:{$gte:18}}, {gender:true}]})
{ "_id" : ObjectId("62c2436229d727582aa383cf"), "name" : "xiaolizi", "gender" : true }
{ "_id" : ObjectId("62c243e329d727582aa383d0"), "name" : "xiaolizi", "age" : 18 }
```

- 与或一起使用

```sh
> db.stu.find({$or:[{age:{$gte:18}}, {gender:true}],name:"xiaolizi"})
{ "_id" : ObjectId("62c2436229d727582aa383cf"), "name" : "xiaolizi", "gender" : true }
{ "_id" : ObjectId("62c243e329d727582aa383d0"), "name" : "xiaolizi", "age" : 18 }
```

- 范围查询 in 和 not in

```sh
> db.stu.find({age:{$in:[12, 18]}})
{ "_id" : ObjectId("62c243e329d727582aa383d0"), "name" : "xiaolizi", "age" : 18 }
{ "_id" : ObjectId("62c413eb7debef3b992368b7"), "name" : "laoli", "age" : 12 }
{ "_id" : ObjectId("62c413f47debef3b992368b8"), "name" : "xiuqin", "age" : 12 }


> db.stu.find({age:{$nin:[12, 18]}})
{ "_id" : ObjectId("62c2436229d727582aa383cf"), "name" : "xiaolizi", "gender" : true }
```

- 正则表达式

```sh
> db.stu.find({name:{$regex:'^l'}})
{ "_id" : ObjectId("62c413eb7debef3b992368b7"), "name" : "laoli", "age" : 12 }

> db.stu.find({name:/^l/})
{ "_id" : ObjectId("62c413eb7debef3b992368b7"), "name" : "laoli", "age" : 12 }
```

- 自定义查询条件（就是js函数）

```sh
> db.stu.find({$where:function(){return this.age>12}})
{ "_id" : ObjectId("62c243e329d727582aa383d0"), "name" : "xiaolizi", "age" : 18 }
```

- limit

```sh
> db.stu.find().limit(2)
{ "_id" : ObjectId("62c2436229d727582aa383cf"), "name" : "xiaolizi", "gender" : true }
{ "_id" : ObjectId("62c243e329d727582aa383d0"), "name" : "xiaolizi", "age" : 18 }
```

- skip

```sh
> db.stu.find().skip(2).limit(2)
{ "_id" : ObjectId("62c413eb7debef3b992368b7"), "name" : "laoli", "age" : 12 }
{ "_id" : ObjectId("62c413f47debef3b992368b8"), "name" : "xiuqin", "age" : 12 }
```

- 投影（指定要查询的字段），设置为1的字段会被显示，不设置不会返回, _id需要特别指定。

```sh
> db.stu.find({}, {name:1})
{ "_id" : ObjectId("62c2436229d727582aa383cf"), "name" : "xiaolizi" }
{ "_id" : ObjectId("62c243e329d727582aa383d0"), "name" : "xiaolizi" }
{ "_id" : ObjectId("62c413eb7debef3b992368b7"), "name" : "laoli" }
{ "_id" : ObjectId("62c413f47debef3b992368b8"), "name" : "xiuqin" }

db.stu.find({}, {name:1, _id:0})
{ "name" : "xiaolizi" }
{ "name" : "xiaolizi" }
{ "name" : "laoli" }
{ "name" : "xiuqin" }
```

- 排序

db.集合名称.find().sort({字段:1,...})

参数1为升序排序，参数-1为降序排序。

```sh
> db.stu.find().sort({age:1})
{ "_id" : ObjectId("62c2436229d727582aa383cf"), "name" : "xiaolizi", "gender" : true }
{ "_id" : ObjectId("62c413eb7debef3b992368b7"), "name" : "laoli", "age" : 12 }
{ "_id" : ObjectId("62c413f47debef3b992368b8"), "name" : "xiuqin", "age" : 12 }
{ "_id" : ObjectId("62c243e329d727582aa383d0"), "name" : "xiaolizi", "age" : 18 }
> db.stu.find().sort({age:-1})
{ "_id" : ObjectId("62c243e329d727582aa383d0"), "name" : "xiaolizi", "age" : 18 }
{ "_id" : ObjectId("62c413eb7debef3b992368b7"), "name" : "laoli", "age" : 12 }
{ "_id" : ObjectId("62c413f47debef3b992368b8"), "name" : "xiuqin", "age" : 12 }
{ "_id" : ObjectId("62c2436229d727582aa383cf"), "name" : "xiaolizi", "gender" : true }
```

- 统计

db.集合名称.find({查询条件}).count()    或     db.集合名称.count({条件})

```sh
> db.stu.count({age:{$gt:12}})
1
```

- 去重

db.集合名称.distinct("去重字段", {查询条件})，只会显示去重字段，不会显示其他属性

```
> db.stu.distinct("age",{})
[ 18, 12 ]
```

- 聚合

db.集合名称.aggregate([{管道:{表达式}}])



常用管道：

$group 将集合中的文档分组

$match 过滤数据，只输出符合条件的文档

$project 修改输入文档的结构，如重命名、增加、删除字段、创建计算结果

$sort 将输入文档排序后输出

$limit 限制聚合管道返回的文档数

$skip 跳过指定数量的文档，并返回余下的文档

$unwind 将数组类型的字段进行拆分



常用表达式（用法 表达式:'$列名'）：

$sum 计算总和，$sum:1同count表示计数

$avg 计算平均值

$min 获取最小值

$max 获取最大值

$push 在结果文档中插入值到一个数组中

$first 根据资源文档的排序获取第一个文档数据

$last 根据资源文档的排序获取最后一个文档数据



group管道

```sh
> db.stu.aggregate([{$group:{_id:"$gender",counter:{$sum:1}}}])
{ "_id" : null, "counter" : 3 }
{ "_id" : true, "counter" : 1 }
> db.stu.aggregate([{$group:{_id:"$age",counter:{$sum:1}}}])
{ "_id" : 12, "counter" : 2 }
{ "_id" : 18, "counter" : 1 }
{ "_id" : null, "counter" : 1 }
```

```json
> db.stu.aggregate([{$group:{_id:"$name",sum_age:{$sum:"$age"},counter:{$sum:"$age"}}}])
{ "_id" : "xiuqin", "sum_age" : 12, "counter" : 12 }
{ "_id" : "laoli", "sum_age" : 12, "counter" : 12 }
{ "_id" : "xiaolizi", "sum_age" : 18, "counter" : 18 }
```

- 使用push会把group以后的某个字段都放到某个数组中,$$ROOT代表整个文档

```sh
> db.stu.aggregate([{$group:{_id:"$name",counter:{$push:"$age"}}}])
{ "_id" : "xiuqin", "counter" : [ 12 ] }
{ "_id" : "laoli", "counter" : [ 12 ] }
{ "_id" : "xiaolizi", "counter" : [ 18 ] }

> db.stu.aggregate([{$group:{_id:"$name",counter:{$push:"$$ROOT"}}}])
{ "_id" : "xiuqin", "counter" : [ { "_id" : ObjectId("62c413f47debef3b992368b8"), "name" : "xiuqin", "age" : 12 } ] }
{ "_id" : "laoli", "counter" : [ { "_id" : ObjectId("62c413eb7debef3b992368b7"), "name" : "laoli", "age" : 12 } ] }
{ "_id" : "xiaolizi", "counter" : [ { "_id" : ObjectId("62c2436229d727582aa383cf"), "name" : "xiaolizi", "gender" : true }, { "_id" : ObjectId("62c243e329d727582aa383d0"), "name" : "xiaolizi", "age" : 18 } ] }
```

- 多个管道配合使用：先过滤，然后进行分组

```sh
> db.stu.aggregate([{$match:{age:{$lt:18}}}])
{ "_id" : ObjectId("62c413eb7debef3b992368b7"), "name" : "laoli", "age" : 12 }
{ "_id" : ObjectId("62c413f47debef3b992368b8"), "name" : "xiuqin", "age" : 12 }

> db.stu.aggregate([{$match:{age:{$lt:18}}}, {$group: {_id:"$name", counter:{$sum:1}}}])
{ "_id" : "xiuqin", "counter" : 1 }
{ "_id" : "laoli", "counter" : 1 }
```

- 多个管道配合使用：先过滤，然后进行分组，最后投影

```sh
> db.stu.aggregate([{$match:{age:{$lt:18}}}, {$group: {_id:"$name", counter:{$sum:1}}}, {$project: {_id:0, counter:1}}])
{ "counter" : 1 }
{ "counter" : 1 }
```

- sort

```sh
> db.stu.aggregate([{$match:{age:{$lt:18}}}, {$group: {_id:"$name", counter:{$sum:1}}}, {$project: {_id:1, counter:1}}, {$sort: {_id: 1}}])
{ "_id" : "laoli", "counter" : 1 }
{ "_id" : "xiuqin", "counter" : 1 }
```

- skip 和 limit 的顺序不能变

```sh
> db.stu.aggregate([{$match:{age:{$lt:18}}}, {$group: {_id:"$name", counter:{$sum:1}}}, {$project: {_id:1, counter:1}}, {$sort: {_id: 1}}, {$skip:1}, {$limit:1}])
{ "_id" : "xiuqin", "counter" : 1 }
```

