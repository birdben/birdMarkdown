---
title: "Kibana学习（二）图形报表的使用"
date: 2016-08-17 01:35:19
tags: [Kibana]
categories: [Log]
---

Kibana已经给我们提供了多种不同的报表形式，如下图所示

![Kibana_Visualization](http://img.blog.csdn.net/20160815175622375?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

图表名称				|描述		|
--------------------|-----		|
Area chart			|区域图		|
Data table			|表格		|
Line chart			|线性图		|
Markdown widget		|Markdown	|
Metric					|统计计数	|
Pie chart				|饼图		|
Tile map				|地图分布	|
Vertical bar chart	|柱状图		|


- metrics（度量）：具体考察的数量值，例如：数量，金额等等（这里就类似OLAP中的度量的概念），可以选择多个metrics度量来考察数量值。
- buckets（维度）：观察数据的角度，从多个角度作为标准衡量数据，例如：时间，地点，类型等等（这里就类似OLAP中的维度的概念），可以选择多个buckets维度来观察数据。



下面我们用以前做的command命令行日志收集来创建不同的Kibana图表

### Area chart区域图，Line chart线性图，Vertical bar chart柱状图

因为Area chart，Line chart，Vertical bar chart是X-Y轴的图结构，所以一般情况是Y轴是统计的数据的值（也就是度量），X轴可以选择我们的索引字段作为观察数据的角度（也就是维度）

#### metrics

```
# Y-Axis:设置Y轴的度量是Count
{
	Aggregation:		Count
}

# Y-Axis:设置Y轴的度量是Unique Count
{
	Aggregation:		Unique Count
}
```

#### buckets

```
# X-Axis:设置X轴的维度是@timestamp字段
{
	Aggregation:	Date Histogram,
	Field:			@timestamp,
	Interval:		Daily
}
```

```
# Split Area:设置可以按照CMD字段在同一个Area chart中区分统计不同的Count值
# Split Line:设置可以按照CMD字段在同一个Line chart中区分统计不同的Count值
# Split Bars:设置可以按照CMD字段在同一个Bars chart中区分统计不同的Count值
{
	Sub Aggregation:	Terms,
	Field:				CMD,
	Order:				Top,
	Size:				5,
	Order By:			metric:Count
}
```

```
# Split Chart:设置可以按照host字段拆分出多个chart统计不同的Count值
{
	Sub Aggregation:	Terms,
	Field:				host,
	Order:				Top,
	Size:				5,
	Order By:			metric:Count
}
```

### Pie chart饼状图

因为Pie chart是没有X-Y轴的图结构，所以一般情况是slice切片是统计的数据的值（也就是度量），Split Slices可以选择我们的索引字段用来切分数据（也就是维度）

#### metrics

```
# Slice Size:设置slice切片统计的度量是Count
{
	Aggregation:		Count
}
```

#### buckets

```
# Split Slices:设置可以按照CMD字段在同一个Pie chart中区分统计不同的Count值
{
	Aggregation:		Terms,
	Field:				host,
	Order:				Top,
	Size:				5,
	Order By:			metric:Count
}
```

```
# Split Chart:设置可以按照host字段拆分出多个chart统计不同的Count值
{
	Sub Aggregation:	Terms,
	Field:				host,
	Order:				Top,
	Size:				5,
	Order By:			metric:Count
}
```

参考文章：

- http://blog.csdn.net/ming_311/article/details/50619859