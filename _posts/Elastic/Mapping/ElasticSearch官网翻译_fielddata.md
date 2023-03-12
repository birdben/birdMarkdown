---
title: "ElasticSearch官网翻译_fielddata"
date: 2017-05-13 17:50:26
tags: [Elasticsearch]
categories: [Search]
---

#### fielddata（字段数据）

默认情况下大多数字段都被索引，这使得它们可以搜索。inverted index（倒排索引）允许查询以唯一排序的term（词条）列表查找search term（搜索词），并且可以立即访问包含该term（词条）的文档列表。

排序，聚合和脚本中对字段值的访问需要不同的数据访问模式。我们不需要查找term（词条）和查找文档，而是可以查找文档并查找它在一个字段中的term（词条）。

<b>
大多数字段可以使用索引时设置磁盘上的doc_values来支持这种类型的数据访问模式，但analyzed的字符串字段不支持doc_values。

相反，analyzed的字符串使用称为fielddata的查询时数据结构。该数据结构是第一次使用字段来进行聚合，排序或脚本访问时构建的。它是通过从磁盘读取每个segment（段）的整个倒排索引构建的，将term <-> 文档关系反转，并将结果存储在内存中，并将其存储在JVM堆中。

加载fielddata是一个昂贵的过程，所以一旦它被加载，它将保留在内存中的segment（段）的生命周期。
</b>

###### 警告：Fielddata可以填满你的堆空间

<b>
Fielddata可以消耗大量的堆空间，特别是当加载高基数analyzed的字符串字段时。 大多数情况下，对analyzed的字符串字段进行排序或聚合没有任何意义（除了significant_terms聚合抛出明显的异常）。 应该始终考虑一个not_analyzed字段（可以使用doc_values）是否会更适合你的用例。
</b>

###### 提示

fielddata.*设置必须具有相同索引中相同名称字段的相同设置。 可以使用PUT mapping API在现有字段上更新其值。

##### fielddata.format

对于analyzed的字符串字段，fielddata format控制是否应启用fielddata。 它接受：disabled和paged_bytes（enabled，这是默认值）。 要禁用fielddata加载，可以使用以下映射：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "text": {
          "type": "string",
          "fielddata": {
            "format": "disabled" 
          }
        }
      }
    }
  }
}
```

- (1) text字段不能用于排序，聚合或脚本。

###### 注意：Fielddata和其他数据类型

历史上，其他字段数据类型也使用fielddata，但是这已经被索引时，基于磁盘的doc_values值替代了。

##### fielddata.loading

该字段设置控制fielddata何时加载到内存中。 它接受三个选项：

- lazy : Fielddata仅在需要时才加载到内存中。 （默认）
- eager : 在搜索新的搜索段变得可见之前，Fielddata被加载到内存中。 如果他们的搜索请求必须触发从大segment（段）延迟加载，这可以减少用户可能遇到的延迟。
- eager_global_ordinals : 将fielddata加载到内存只是所需工作的一部分。 在加载每个段的fielddata之后，Elasticsearch构建全局序号数据结构，以便在分片中的所有段中创建所有唯一term（词条）的列表。 默认情况下，全局序号是懒惰地构建的。 如果该字段具有非常高的基数，则全局序号可能需要一些时间来构建，在这种情况下，你可以使用eager loading（加速加载）。

##### 全局序号

全局序号是一个基于fielddata和doc values的数据结构，它以字典顺序维护每个唯一term（词条）的增量编号。每个term（词条）都有唯一的编号，term A的编号低于term B的编号。全局序号仅支持字符串字段。

Fielddata和doc values也有ordinals（序号），它是特定segment（段）和字段中所有term（词条）的唯一编号。全局序号只是建立在这之上，通过提供segment（段）序号和全局序号之间的映射，后者在整个分片中是唯一的。

全局序号用于使用segment（段）序号的功能，例如排序和terms aggregation（词条聚合），以提高执行时间。terms aggregation（词条聚合）完全依赖于全局序号来执行分片级别的聚合，然后将全局序号转换为真正的term（词条），term（词条）仅用于最终减少阶段，其结合不同分片的结果。

指定字段的全局序号与分片的所有段相关联，而fielddata和doc values序号与单个段相关。其与针对与单个段相关联的特定字段的字段数据不同。因此，只要一旦新的segment（段）变得可见，全局序号就需要完全重建。

全局序号的加载时间取决于一个字段中的term（词条）数量，但通常它是低的，因为源字段数据已经被加载。全局序号的内存开销很小，因为它被非常有效的压缩。加载全局序号可以将加载时间从第一个搜索请求移动到刷新本身。

##### fielddata.filter

Fielddata过滤可用于减少加载到内存中的terms（词条）数量，从而减少内存使用。 Terms（词条）可以按frequency（频率）或regular expression（正则表达式）过滤，也可以通过两者的组合进行过滤：

- 按频率过滤

frequency（频率）过滤器允许你仅加载文档frequency（频率）在最小和最大值之间的terms（词条），可以表示绝对数字（当数字大于1.0时）或百分比（例如0.01为1％，1.0为100％）。 每个段都会计算频率。 百分比是基于具有该字段值的文档数量，而不是该段中的所有文档。

通过使用min_segment_size指定段应包含的文档的最小数量，可以完全排除小段：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "tag": {
          "type": "string",
          "fielddata": {
            "filter": {
              "frequency": {
                "min": 0.001,
                "max": 0.1,
                "min_segment_size": 500
              }
            }
          }
        }
      }
    }
  }
}
```

- 通过正则表达式进行过滤

Terms（词条）也可以通过正则表达式进行过滤 - 只加载与正则表达式匹配的值。 注意：正则表达式应用于字段中的每个term（词条），而不是整个字段值。 例如，要只从一个tweet加载主题标签，我们可以使用一个正则表达式，匹配以＃开头的词条：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "tweet": {
          "type": "string",
          "analyzer": "whitespace",
          "fielddata": {
            "filter": {
              "regex": {
                "pattern": "^#.*"
              }
            }
          }
        }
      }
    }
  }
}
```

这些过滤器可以在现有的字段映射中进行更新，并在下一次加载segment（段）的fielddata时生效。 使用Clear Cache API重新加载fielddata以使用新的过滤器。


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/fielddata.html
