### mongo 执行计划与数据量有关, 数据量变化, 会改变执行计划
案例:
某个template collection 查询

```
db.getCollection('template').aggregate([
{"$match":{"owner":"User-96f8931d-e5d9-4732-8de7-306bf84f7779"}},
{"$sort":{"updatedAt":-1}}
])
```
几天前还好好的，今天怎么就查不出来了，怎么就慢SQL了

提示：数据量增加，执行计划变化

已知：owner字段有独立的索引，updatedAt字段也有独立的索引

问题分析：
几天前的执行计划（通过临时库删除数据重现）

[owner索引] => [内存中根据updatedAt排序]

今天的执行计划

[updatedAt索引] => [内存中过滤owner]  ??? 和撸全表有什么区别

优化方案:
方案一： 建立 owner和updatedAt 的复合索引

方案二：修改sql，引导sql优化器调整执行计划

db.getCollection('template').aggregate([
{"$match":{"owner":"User-96f8931d-e5d9-4732-8de7-306bf84f7779"}},
{"$limit": 10000},
{"$sort":{"updatedAt":-1}}
])
方案三：改为 find..sort 的对等形式

db.getCollection('template').find({"owner":"User-96f8931d-e5d9-4732-8de7-306bf84f7779"}).sort({"updatedAt": -1})