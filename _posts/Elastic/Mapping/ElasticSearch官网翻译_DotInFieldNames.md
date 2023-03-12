---
title: "ElasticSearch官网翻译_DotInFieldNames"
date: 2017-05-10 22:58:51
tags: [Elasticsearch]
categories: [Search]
---

#### 字段名中使用点

<b>在Elasticsearch 5.0中，字段名称中允许使用点，path中的每个步骤都被解释为对象字段，但最后一步除外。</b> 例如，在Elasticsearch 5.0中索引以下文档：

```
{
  "server.latency.max": 100
}
```

将会生成下面的映射结果：

```
{
  "properties": {
    "server": {
      "type": "object",
      "properties": {
        "latency": {
          "type": "object",
          "properties": {
            "max": {
              "type": "long"
            }
          }
        }
      }
    }
  }
}
```

<b>Elasticsearch 2.x不支持这种点到对象（dots-to-object）的转换，因此在版本2.0 - 2.3中不允许字段名称中的点。</b>

#### 启用对字段名称中的点的支持

Elasticsearch 2.4.0添加了一个名为mapper.allow_dots_in_name的系统属性，将禁用对字段名称中的点的检查。 你可以如下启用此系统属性：

```
export ES_JAVA_OPTS="-Dmapper.allow_dots_in_name=true"
./bin/elasticsearch
```

（如果使用Debian软件包，则可以在/etc/default/elasticsearch文件中设置ES_JAVA_OPTS，如果使用RPM，可以在/etc/sysconfig/elasticsearch文件中设置ES_JAVA_OPTS）。

启用此系统属性后，在2.4.0中对文档进行索引将导致以下映射：

```
{
  "properties": {
    "server.latency.max": {
      "type": "long"
    }
  }
}
```

#### 将点的字段升级到5.x

在1.x中创建的索引必须在2.4之前重新建立索引，然后升级到5.x，而不管它们是否具有点的字段。 或者，来自1.x群集的索引可以使用reindex from-remote直接导入到5.x群集中。

2.4中创建的指数可以直接升级到5.x，只要没有冲突（见下文）。 如果字段名称中有点，则映射将自动被更新为字段名中“点”显示的对象样式映射。

- 注意：点可能创建冲突的字段

在升级到object-style对象样式映射时会产生冲突的字段的索引是无法升级到5.x。 例如，可以升级以下映射：

```
{
  "properties": { (1)
    "server.latency": {
      "type": "long"
    },
    "server.name": { (2)
      "type": "string"
    }
  }
}
```

- (1)(2) 在这两种情况下，server字段将成为一个对象字段。

下面这个映射不能升级:

```
{
  "properties": { (1)
    "server.latency": {
      "type": "long"
    },
    "server": { (2)
      "type": "string"
    }
  }
}
```

- (1)(2) server字段是server.latency中的object，但是server又是一个string。这是一个冲突，将阻止索引在5.x中打开。


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/dots-in-names.html