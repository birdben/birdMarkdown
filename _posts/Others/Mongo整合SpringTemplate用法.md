---
title: "Mongo整合SpringTemplate用法"
date: 2015-11-12 11:18:34
tags: [MongoDB, Spring]
categories: [MongoDB]
---


#### 使用Spring的MongoTemplate如何执行自定义的mongo语句

基础的SpringMongoTemplate用法可以参考:

- http://blog.csdn.net/cuiran/article/details/8287204
- http://blog.csdn.net/xiaohulunb/article/details/27828381


对GoodsInfo Collection按照name进行distinct查询

```
String jsonSql = "{distinct:'GoodsInfo', key:'name'}";
CommandResult  commandResult = mongoTemplate.executeCommand(jsonSql);
System.out.println();
BasicDBList list = (BasicDBList)commandResult.get("values"); 
for (int i = 0; i < list.size(); i ++) { 
	System.out.println(list.get(i)); 
} 
System.out.println();
```

查询统计某一个时间段每个片区下的总用户数，新用户数，旧用户数，异常数

```
public Map<String, LandPushTypeCount> findTodayCountByAreaIds(Collection<String> areaIds) throws Exception {
		Map<String, LandPushTypeCount> result = new HashMap<String, LandPushTypeCount>();

		Criteria criteria = Criteria.where("todayDate").is(DateUtil.parseDate2YMD(new Date())).andOperator(Criteria.where("areaId").in(areaIds));
		String initDocument = "{codeCount:0, newCount:0, oldCount:0, errorCount:0}";
		String reduceFunction = "function (doc, prev) { prev.codeCount += doc.codeCount; prev.newCount += doc.newCount; prev.oldCount += doc.oldCount; prev.errorCount += doc.errorCount; }";
		GroupBy groupBy = GroupBy.key("areaId").initialDocument(initDocument).reduceFunction(reduceFunction);

		GroupByResults<LandPushTypeCount> groupByResult = landPushTypeCountDao.getMongoTemplate().group(criteria, "LandPushTypeCount", groupBy, LandPushTypeCount.class);

		BasicDBList list = (BasicDBList) groupByResult.getRawResults().get("retval");
		for (int i = 0; i < list.size(); i ++) {
			LandPushTypeCount landPushTypeCountBean = new LandPushTypeCount();
			BasicDBObject obj = (BasicDBObject)list.get(i);
			System.out.println("片区：" + obj.get("areaId")
							+ "总数量：" + obj.get("codeCount")
							+ "新用户数量：" + obj.get("newCount")
							+ "旧用户数量：" + obj.get("oldCount")
							+ "异常数量：" + obj.get("errorCount")
			);
			landPushTypeCountBean.setCodeCount(((Double) obj.get("codeCount")).intValue());
			landPushTypeCountBean.setNewCount(((Double) obj.get("newCount")).intValue());
			landPushTypeCountBean.setOldCount(((Double) obj.get("oldCount")).intValue());
			landPushTypeCountBean.setErrorCount(((Double) obj.get("errorCount")).intValue());
			result.put((String) obj.get("areaId"), landPushTypeCountBean);
		}
		return result;
	}
	
// 相当于mongo语句如下
db.runCommand(
{
　　"group":
　　{
　　　　"ns":"LandPushTypeCount",
　　　　"key":{"areaId":true},
　　　　"initial":{codeCount:0, newCount:0, oldCount:0, errorCount:0},
　　　　"$reduce":function(doc,prev)
　　　　{
　　　　　 prev.codeCount += doc.codeCount;
          prev.newCount += doc.newCount;
          prev.oldCount += doc.oldCount;
          prev.errorCount += doc.errorCount;
　　　　},
　　　　"condition":{"todayDate":"2015-10-29", "areaId":{$in:["", "330100", "110000"]}}
　　}
}
);
```
查询top100回复数的帖子

```
public void top100Postinfo() {
        long start = System.nanoTime();
        try {
            final long startTime = 1443628800000L;
            final long endTime = 1446436799000L;

            Query query = new Query();
            query.addCriteria(Criteria.where("createTS").gte(startTime).lte(endTime));
            query.addCriteria(Criteria.where("srcPostId").exists(true));
            query.addCriteria(Criteria.where("referInfo").exists(false));
            query.addCriteria(Criteria.where("dr").is(0));
            long count = postinfoDAO.getCount(query);
            System.out.println("count:" + count);

            final int filterCount = 100;

            DBObject result1 = postinfoDAO.getMongoTemplate().execute(Postinfo.class, new CollectionCallback<DBObject>() {
                @Override
                public DBObject doInCollection(DBCollection collection) throws MongoException, DataAccessException {

                    // match匹配条件
                    Map matchMap = new HashMap();
                    matchMap.put("createTS", new BasicDBObject("$gt", startTime).append("$lt", endTime));
                    matchMap.put("referInfo", new BasicDBObject("$exists", false));
                    matchMap.put("srcPostId", new BasicDBObject("$exists", true));
                    matchMap.put("dr", 0);
                    DBObject matchOption = new BasicDBObject("$match", matchMap);

                    // group条件
                    Map groupMap = new HashMap();
                    groupMap.put("_id", "$srcPostId");
                    groupMap.put("total", new BasicDBObject("$sum", 1));
                    BasicDBObject groupOption = new BasicDBObject("$group", groupMap);

                    // sort条件
                    Map sortMap = new HashMap();
                    sortMap.put("total", -1);
                    BasicDBObject sortOption = new BasicDBObject("$sort", sortMap);

                    // limit条件
                    BasicDBObject limitOption = new BasicDBObject("$limit", filterCount);

                    // 获取结果集
                    AggregationOutput output = collection.aggregate(matchOption, groupOption, sortOption, limitOption);
                    Iterator<DBObject> iterator = output.results().iterator();
                    DBObject result = null;
                    List<Map<String, String>> resultList = new ArrayList<Map<String, String>>();
                    while (iterator.hasNext()) {
                        result = iterator.next();
                        String id = (String) result.get("_id");
                        Integer total = (Integer) result.get("total");
                        System.out.println("id:" + id);
                        System.out.println("total:" + total);

                        Map<String, String> resultMap = new HashMap<>();
                        resultMap.put("id", id);
                        resultMap.put("total", String.valueOf(total));
                        resultList.add(resultMap);
                    }
                    exportExcel(resultList);
                    return result;
                }
            });
        } catch (Exception e) {
            e.printStackTrace();
        }
        long finish = System.nanoTime();
        System.out.println(TimeUnit.NANOSECONDS.toSeconds(finish - start) + ":秒");
        System.out.println("top20PercentPostinfo 执行完毕");
    }
    
    
// 相当于mongo语句如下
db.Postinfo.aggregate([
	{ $match: { "createTS":{"$gt":1443628800000, "$lt":1446436799000}, "srcPostId":{$exists:true}, "referInfo":{$exists:false}, "dr":0 } },
	{ $limit: 1 },
	{ $group: { _id: "$srcPostId", total: { $sum: 1 } } },
	{ $sort: { total: -1 } }
]);
```



参考文章：

- http://blog.csdn.net/ruishenh/article/details/12842331
- http://www.360doc.com/content/13/0524/18/12127276_287814300.shtml
- http://ask.csdn.net/questions/167950
- http://147175882.iteye.com/blog/1565378