ElasicSearch权威指南读书笔记

ES内部机制讲解
2，4，9，11

介绍ES
1-11

ES搜索高级用法
12-17

语义分析，分词，停词，同义词
18-24

分组
25-35

geo坐标
36-39

index数据设计
40-43

生产环境的重要配置，监控
44-46




ES是基于Lucene的，因为Lucene只提供了Java API，而且使用起来非常麻烦，所以ES在Lucene之上构建一个full-text搜索引擎，不需要详细的了解Lucene，隐藏了Lucene的复杂度，还支持使用不同的语言通过RESTFUL API来调用ES，使full-text搜索引擎使用起来更简单，而且ES不仅仅是全文搜索引擎，还能进行实时分析和水平扩展

安装ES依赖Java环境（JDK）
可以安装Marvel插件
./bin/plugin -i elasticsearch/marvel/latest
关闭Marvel监控echo 'marvel.agent.enabled: false' >> ./config/elasticsearch.yml
访问Marvel插件
http://localhost:9200/_plugin/marvel/
访问sense
http://localhost:9200/_plugin/marvel/sense/

停止ES
curl -XPOST 'http://localhost:9200/_shutdown'


#### Document Oriented
Objects in an application are seldom just a simple list of keys and values. More often than not, they are complex data structures that may contain dates, geo locations, other objects, or arrays of values.Sooner or later you’re going to want to store these objects in a database. Trying to do this with the rows and columns of a relational database is the equivalent of trying to squeeze your rich, expressive objects into a very big spreadsheet: you have to flatten the object to fit the table schema—usually one field per column—and then have to reconstruct it every time you retrieve it.

Elasticsearch is document oriented, meaning that it stores entire objects or documents. It not only stores them, but also indexes the contents of each document in order to make them searchable. In Elasticsearch, you index, search, sort, and filter documents —not rows of columnar data. This is a fundamentally different way of thinking about data and is one of the reasons Elasticsearch can perform complex full-text search.


