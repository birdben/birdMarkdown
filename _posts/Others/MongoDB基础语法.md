---
title: "MongoDB基础命令总结"
date: 2015-11-12 02:20:11
tags: [MongoDB]
categories: [MongoDB]
---

### Mongo数据库链接

```
# mongo命令链接数据库
./mongo 127.0.0.1:27017 -u root -p 'root' --authenticationDatabase databasename

参数说明：
-u:指明数据库的用户名
-p:指明数据库的密码
--authenticationDatabase:指明授权的数据库
```

### Mongo增删改

```
# 根据查询条件批量更新
db.GoodsInfo.update({"type":1}, {$set:{"name":"birdben"}}, false, true);
	
# 删除指定条件的数据
db.GoodsInfo.remove({goodsStatus: 1});
	
# 查询修改删除
db.GoodsInfo.findAndModify({
	query: {price: {$gte: 100}},
	sort: {price: -1},
	update: {$set: {name: 'birdben'}, $inc: {price: 10}},
	remove: true
});
 
db.runCommand({ findandmodify : "GoodsInfo",
	query: {price: {$gte: 25}},
	sort: {price: -1},
	update: {$set: {name: 'birdben'}, $inc: {price: 10}},
	remove: true
});
```

### Mongo查询

```
# 查询指定列(只查看name、price数据)
db.GoodsInfo.find({"_id":"123456"}, {"name":1, price: 1}}).pretty();
# 当然name也可以用true或false,当用ture的情况下河name:1效果一样，如果用false就是排除name，显示name以外的列信息。
	
# 区间查询
db.GoodsInfo.find({price: {$gte: 100, $lte: 200}});

# like模糊查询，name中包含mongo的数据
db.GoodsInfo.find({name: /mongo/});

# like模糊查询，以mongo开头的数据
db.GoodsInfo.find({name: /^mongo/});
	
# or查询
db.GoodsInfo.find({$or: [{price: 100}, {price: 200}]});
	
# exists查询
db.GoodsInfo.find({price: {$exists: true}});

# 查询exists的个数
db.GoodsInfo.find({price: {$exists: true}}).count();
```

distinct用法

```
db.GoodsInfo.distinct("name");

# 对distinct结果进行筛选（下面两种方式效果是一样的）
db.GoodsInfo.distinct("cityId", {"cityId":"330100"}); 
db.runCommand({"distinct":"GoodsInfo","key":"cityId"}); 

# 对distinct结果只查询数量，而且带过滤条件
db.runCommand({"distinct":"GoodsInfo","key":"cityId","query":{"cityId":"330100"}}).values.length;
```

查询子文档

```
# 查询某个键
db.GoodsInfo.find({"array.key":"value"});

# 查询多个键
db.GoodsInfo.find({"$elemMatch":{"array.key1":"value","array.key2":"value"}})

# 查询未知个键
db.GoodsInfo.find({"$elemMatch":{"$in":{"array":"value"}});
```

数组查询

```
# 查询数组。此时你可能会使用到$all、$size。
db.tianyc04.find()
{ "_id" : 1, "fruit" : [ "apple", "banana", "peach" ] }
{ "_id" : 2, "fruit" : [ "apple", "orange", "peach" ] }
{ "_id" : 3, "fruit" : [ "orange", "banana", "peach" ] }

# 通过全匹配，查询第一行
db.tianyc04.find({fruit:["apple", "banana", "peach"]})
{ "_id" : 1, "fruit" : [ "apple", "banana", "peach" ] }

# 如果将数组中的顺序颠倒，则第一行就匹配不上了。此时可以使用$all
db.tianyc04.find({fruit:["apple", "peach", "banana"]})
db.tianyc04.find({fruit:{$all:["apple", "peach", "banana"]}})
{ "_id" : 1, "fruit" : [ "apple", "banana", "peach" ] }

#也可以只输入一个元素进行查询
db.tianyc04.find({fruit:'apple'})
{ "_id" : 1, "fruit" : [ "apple", "banana", "peach" ] }
{ "_id" : 2, "fruit" : [ "apple", "orange", "peach" ] }

# 如果这个元素变成了数组，mongo就会进行精确匹配。此时你可能需要使用$all进行模糊匹配：
db.tianyc04.find({fruit:['apple']})
db.tianyc04.find({fruit:{$all:['apple']}})
{ "_id" : 1, "fruit" : [ "apple", "banana", "peach" ] }
{ "_id" : 2, "fruit" : [ "apple", "orange", "peach" ] }

# 还可以按照数组中指定位置的元素进行查询，注意数组下标的起始编号是0。
db.tianyc04.find({'fruit.1':'orange'})
{ "_id" : 2, "fruit" : [ "apple", "orange", "peach" ] }

# 可以按照数组长度进行查询，只查询数组长度为x的文档。
db.tianyc04.find({fruit:{$size:3}})
{ "_id" : 1, "fruit" : [ "apple", "banana", "peach" ] }
{ "_id" : 2, "fruit" : [ "apple", "orange", "peach" ] }
{ "_id" : 3, "fruit" : [ "orange", "banana", "peach" ] }

# 如果数组中存储的不是简单的字符串，可以使用$elemMatch查询，但是$elemMatch的局限性是只能返回数组中的第一个匹配记录。

db.tianyc05.find()
{ "_id" : 1, "fruit" : [ {"id":1, "name":"apple"}, {"id":2, "name":"banana"}, {"id":3, "name":"peach"} ] }
{ "_id" : 2, "fruit" : [ {"id":1, "name":"apple"}, {"id":4, "name":"orange"}, {"id":3, "name":"peach"} ] }
{ "_id" : 3, "fruit" : [ {"id":4, "name":"orange"}, {"id":2, "name":"banana"}, {"id":3, "name":"peach"} ] }

db.tianyc05.find({fruit:{$elemMatch:{id:2}}});
```
	
分组查询

```
# 分组查询某一时间段，按照主贴分组，统计每个主贴的回复数量
db.Postinfo.aggregate([
	{ $match: { "createTime":{"$gt":1443628800000, "$lt":1446436799000}, "srcPostId":{$exists:true} } },
	{ $group: { _id: "$srcPostId", total: { $sum: 1 } } },
	{ $sort: { total: -1 } }
]);

# 按照被拒绝的原因进行统计，每种原因的数量，并且筛选结果数量要大于等于5
db.ReviewRecord.aggregate([
	{ $group: { _id: "$rejectReason", count: { $sum: 1 } } },
	{ $sort: { count: -1 } },
	{ $limit: 10},
	{ $match: { count: { $gte: 5 } } }
]);


# 分组之后，在进行数量的统计，统计分组后的数量大于等于5的数量有多少
db.ReviewRecord.aggregate( [
  {
    $group: {
       _id: {
          rejectReason: "$rejectReason"
       },
       count: { $sum: 1 }
    }
  },
  { $match: { count: { $gte: 5 } } },
  {
    $group: {
       _id: null,
       count: { $sum: 1 }
    }
  }
] );

# 类似SQL：
SELECT COUNT(*) FROM (
	SELECT rejectReason, count
	FROM ReviewRecord
	GROUP BY rejectReason
);

# 复杂的统计计算，统计每个城市的2015-10-29的总用户数，新用户数，老用户数，异常用户数
db.runCommand(
{
　　"group":
　　{
　　　　"ns":"LandPushTypeCount",
　　　　"key":{"cityId":true},
　　　　"initial":{codeCount:0, newCount:0, oldCount:0, errorCount:0},
　　　　"$reduce":function(doc,prev)
　　　　{
			prev.totalCount++;
			if(doc.isNew == 0) {
				prev.newCount++;
        	} else {
　　　　　	   prev.oldCount++;
         	}
         	if(doc.isOldEquipMement == 1) {
　　　　　	    prev.errorCount++;
         	}
　　　　},
　　　　"condition":{"date":"2015-10-29", "cityId":{$in:["110000", "330100"]}}
　　}
}
);
```

### Mongo导入/导出

单表导入导出

```
# import/export命令
$ ./mongoimport -d databasename -c GoodsInfo --file GoodsInfo.dat -h 127.0.0.1 --port 27017 -u 'root' -p 'root'
$ ./mongoexport -d databasename -c GoodsInfo -o GoodsInfo.dat -h 127.0.0.1 --port 27017 -u 'root' -p 'root'

参数说明：
-h:指明数据库宿主机的IP
-u:指明数据库的用户名
-p:指明数据库的密码
-d:指明数据库的名字
-c:指明collection的名字
-f:指明到要导出的文件名

# dump/restore命令
$ ./mongorestore --host 127.0.0.1 --port 27017 -u root -p root --authenticationDatabase databasename -d databasename /Users/ben/Downloads/mongodb_bak/GoodsInfo.bson
$ ./mongodump --host 127.0.0.1 --port 27017 -u root -p root --authenticationDatabase databasename -d databasename -o /Users/ben/Downloads/mongodb_bak

参数说明：
-h:指明数据库宿主机的IP
-u:指明数据库的用户名
-p:指明数据库的密码
-d:指明数据库的名字
-c:指明collection的名字
-o:指明到要导出的文件名
-q:指明导出数据的过滤条件
```



参考文章：

- http://www.cnblogs.com/cswuyg/p/4595799.html
- http://www.open-open.com/lib/view/open1392709240428.html
- http://docs.mongoing.com/manual-zh/core/single-purpose-aggregation.html
- http://docs.mongoing.com/manual-zh/reference/sql-aggregation-comparison.html