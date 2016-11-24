---
title: "Kibana学习（一）环境安装总结"
date: 2016-08-16 23:55:39
tags: [Kibana]
categories: [Log]
---

之前在搭建ELK环境的过程中遇到了一些问题，在此进行总结，以后会不断更新的。

#### Kibana的默认索引.kibana无法在ES创建

这个问题是因为我在ES中安装了ik插件，修改了elasticsearch.yml的配置文件，但是没有在ES的lib包目录中引入ik的jar包，所以在Kibana访问ES的时候ES报错了，检查ES的log文件发现了是elasticsearch.yml的ik插件配置没有找到ik的jar包

- 所以Kibana报错最好的找错方式就是看ES的log

#### ES创建了User的index之后，在Kibana配置该索引时'Time-field name'为空，没有@Timestamp选项

这个问题是因为ES创建的User的mapping中没有个一个字段是date类型，所以就没有@Timestamp字段，修改User的mapping重新索引数据后@Timestamp字段就出现了，如果需要手动添加Timestamp字段可以参考如下步骤。

```
1. Go to Settings → Advanced.
2. Edit the metaFields and add "_timestamp". Hit save.
3. Now go back to Settings → Indices and _timestamp will be available in the drop-down list for "Time-field name".
```
![metaFields](http://i.stack.imgur.com/ngaPk.png)

参考文章：

- http://stackoverflow.com/questions/29429201/timestamp-not-appearing-in-kibana

#### Kibana创建图表的时候，选择的Aggregation Terms字段报错，该字段已经被Analyzed分词了，无法被Aggregation Terms使用

问题是因为我自己创建的索引command_index的mapping，所有的String字段都没有指定分词情况，所以ES默认都是Analyzed分词。所以这里只需要将mapping修改一下，将所有的String类型字段都指定分词方式("index":"not_analyzed")，这样指定的字段就不会分词了，就可以使用Aggregation Terms方式创建图表了。（注意：一定要删除原来的mapping，重新索引数据才可以）

#### Kibana使用Pie Charts错误："Pie chart response converter:Splitting charts without splitting sliced is not supported.Pretend that we are just splitting slices"

其实这个错误是因为在创建图表的时候选择了错误的类型，本来只是想创建一个单个的饼图（用线来切分的），所以应该使用Split Slice类型（从字面上应该也能理解），但是却选择了Split Chart类型了，所以就会出现上面的错误。

![](http://img.blog.csdn.net/20160812170725251?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

下面是来自stackoverflow的解释，还清晰的给出了这两个类型的例子，很直观的区别开了。

```
If you want to do a "split slice " operation on a pie chart,first you need to create a pie chart with slices.Here from what I understand,you tried to give the option "split chart" first,which actually is to make differrent pie charts,in the same row or column,which needs more than one pie-chart. This also requires pie-charts (with slices) to be created first. So you need to create pie-charts and then only you can use "split chart". i In the figures below,the first one shows an ordinary pie-chart created by "split-slice" and the second one shows five pie charts stacked horizontally using the "split chart" method. 
```

![Split Slice](http://i.stack.imgur.com/fHvO0.png)
![Split Chart](http://i.stack.imgur.com/4P7OK.png)

参考文章：

- http://stackoverflow.com/questions/28691889/kibana-4-making-pie-chart-error-message

