---
title: "Logstash学习（四）Logstash的Grok使用技巧"
date: 2016-12-21 15:46:57
tags: [Logstash]
categories: [Log]
---

最近在使用Logstash解析日志的时候遇到了很多问题，需要处理多种特殊情况，下面会列举一下我遇到的特殊情况以及处理方式。

如何检查数组是否为空数组

```
if [tags] == [] {
  mutate {
    remove_field => ["tags"]
  }
}
```

判断json中是否有foo这个属性

```
if [foo] {
    ...
} else {
    drop {}
}

if ![foo] {
    drop {}
} else {
    ...
}
```

```
ruby {
    code => "
        if event['column16'] == nil
            event['log_type'] = 'type2'
        else
            event['log_type'] = 'type1'
        end	
    "
}
```

```
if "_jsonparsefailure" in [tags] {
    drop {}
} else {
	...
}
```

参考文章：

- http://stackoverflow.com/questions/30309096/logstash-check-if-field-exists
- https://discuss.elastic.co/t/how-can-i-check-if-the-tags-array-is-empty/27856
- https://discuss.elastic.co/t/check-existence-of-a-field-and-checking-null-value-in-field/24968/4
- https://github.com/elastic/logstash/issues/1867