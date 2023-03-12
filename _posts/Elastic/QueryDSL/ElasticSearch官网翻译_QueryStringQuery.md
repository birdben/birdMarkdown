---
title: "ElasticSearch官网翻译_QueryStringQuery"
date: 2017-06-18 13:26:48
tags: [Elasticsearch]
categories: [Search]
---

## Query String Query

一个使用查询解析器解析其内容的查询。 这是一个例子：

```
GET /_search
{
    "query": {
        "query_string" : {
            "default_field" : "content",
            "query" : "this AND that OR thus"
        }
    }
}
```

query_string顶级参数包括：


参数|描述
---|---
query|要被解析的实际查询。 请参阅Query string syntax。
default_field|如果未指定前缀字段，则使用查询词条的默认字段。 默认为index.query.default_field的索引设置，默认为_all。
default_operator|如果未指定显式运算符，则使用默认运算符。 例如，使用OR的默认运算符，查询"capital of Hungary"将转换为"capital OR of OR Hungary"，使用AND的默认运算符，将相同的查询转换为"capital AND of AND Hungary"。 默认值为OR。
analyzer|用于分析query string（查询字符串）的分析器名称。
allow_leading_wildcard|设置时，"*"或"?"被允许作为第一个字符。 默认为true。
enable_position_increments|设置为true以在结果查询中启用位置增量。 默认为true。
fuzzy_max_expansions|控制fuzzy模糊查询将扩展到的最大词条数量。 默认为50
fuzziness|设置fuzzy模糊查询的模糊性。 默认为AUTO。 有关允许的设置，请参阅"Fuzziness"一节。
fuzzy_prefix_length|设置fuzzy模糊查询的前缀长度。 默认值为0。
phrase_slop|设置短语的默认slop。 如果为0，则需要精确的短语匹配。 默认值为0。
boost|设置查询的boost值。 默认为1.0。
auto_generate_phrase_queries|默认为false。
analyze_wildcard|缺省情况下，不分析查询字符串中的通配符词条。 通过将此值设置为true，将尽力分析这些值。
max_determinized_states|限制多少的自动机状态正则表达式查询被允许创建。 这可以防止太难的（例如大到指数级）regexps。 默认为10000。
minimum_should_match|一个用于控制生成的布尔查询中应该有多少个"should"子句的值。 它可以是绝对值（2），百分比（30％）或两者的组合。
lenient|如果设置为true，将会导致基于格式的失败（例如提供文本给数字类型的字段）被忽略。
time_zone|要应用于与日期相关的任何范围查询的时区。 另请参见JODA timezone。
quote_field_suffix|附加到查询字符串引用部分的字段的后缀。 这允许使用具有不同分析链的字段来进行精确匹配。 看这里一个全面的例子。
split_on_whitespace|查询文本是否应在分析前按照空格分割。 相反，queryparser查询解析器只会解析真正的operators运算符。 默认为false。 如果auto_generate_phrase_queries已设置为true，则不允许将此选项设置为false。
all_fields|对可以查询的映射中检测到的所有字段执行查询。 默认情况下将使用_all字段禁用，并且未指定default_field（在索引设置或请求主体中），并且没有明确指定fields。

当生成multi term query时，可以使用rewrite参数控制如何重写。

#### Default Field（默认字段）

当在查询字符串语法中未明确指定要搜索的字段时，将使用index.query.default_field来获得要搜索的字段。 它默认为_all字段。

如果_all字段被禁用，query_string查询将自动尝试确定索引映射中可查询的现有字段，并对这些字段执行搜索。 请注意，这不包括nested嵌套文档，使用nested嵌套查询来搜索这些文档。

#### Multi Field（多字段）

query_string查询也可以针对多个字段运行。 可以通过"fields"参数（下面的示例）提供字段。

对多个字段运行query_string查询的想法是将每个查询词条扩展为OR子句，如下所示：

```
field1:query_term OR field2:query_term | ...
```

例如，以下查询

```
GET /_search
{
    "query": {
        "query_string" : {
            "fields" : ["content", "name"],
            "query" : "this AND that"
        }
    }
}
```

匹配相同的命令

```
GET /_search
{
    "query": {
        "query_string": {
            "query": "(content:this OR name:this) AND (content:that OR name:that)"
        }
    }
}
```

由于从各个搜索词生成了几个查询，所以可以使用dis_max查询或一个简单的bool查询来自动完成它们的组合。 例如（使用^5表示法将name提升5）：

```
GET /_search
{
    "query": {
        "query_string" : {
            "fields" : ["content", "name^5"],
            "query" : "this AND that OR thus",
            "use_dis_max" : true
        }
    }
}
```

也可以使用简单的通配符来搜索文档的特定内部元素。 例如，如果我们有一个具有多个字段（或内部对象的字段）的city对象，则可以在所有"city"字段上自动搜索：

```
GET /_search
{
    "query": {
        "query_string" : {
            "fields" : ["city.*"],
            "query" : "this AND that OR thus",
            "use_dis_max" : true
        }
    }
}
```

另一个选择是在查询字符串本身（正确地转义*符号）中提供通配符字段搜索，例如：city.\*:something：

```
GET /_search
{
    "query": {
        "query_string" : {
            "query" : "city.\\*:(this AND that OR thus)",
            "use_dis_max" : true
        }
    }
}
```

###### 注意

> 由于\（反斜杠）是json字符串中的一个特殊字符，因此需要进行转义，因此在上述query_string中使用了两个反斜杠。

当对多个字段运行query_string查询时，允许使用以下附加参数：

参数|描述
---|---
use_dis_max|如果使用dis_max（设置为true）或bool查询（将其设置为false）组合查询。 默认为true。
tie_breaker|使用dis_max时，断开最大断路器。 默认为0。

fields参数还可以包括基于模式的字段名称，允许自动扩展到相关字段（包括动态引入的字段）。 例如：

```
GET /_search
{
    "query": {
        "query_string" : {
            "fields" : ["content", "name.*^5"],
            "query" : "this AND that OR thus",
            "use_dis_max" : true
        }
    }
}
```

### Query string syntax

query string查询字符串"mini-language"被用于Query String Query和search API中的q查询字符串参数。

query string查询字符串被解析成一系列terms词条和operators操作符。 一个词条可以是一个单词 - quick或者brown - 或者一个双引号包围的短语 - "quick brown" - 它以相同的顺序搜索短语中的所有单词。

Operators操作符可以自定义搜索 - 可用的选项如下所述。

#### Field names（字段名）

如Query String Query中所述，default_field被搜索词条搜索，但可以在查询语法中指定其他字段：

- 其中status字段包含active

```
status:active
```

- title字段包含quick或brown。 如果省略了OR运算符，则将使用默认运算符

```
title:(quick OR brown)
title:(quick brown)
```

- author字段中包含准确的短语"john smith"

```
author:"John Smith"
```

- 其中任何book.title，book.content或者book.date字段包含quick或brown（请注意，我们需要如何使用反斜杠转义*）：

```
book.\*:(quick brown)
```

- 其中字段title具有任何non-null：

```
_exists_:title
```

#### Wildcards（通配符）

通配符搜索可以被运行在单独词条上，使用?替换单个字符，*替换零个或多个字符：

```
qu?ck bro*
```

请注意，通配符查询可能使用大量的内存并执行得非常糟糕 - 想一下多少词条需要被查询来匹配查询字符串"a* b* c*"。

###### 警告

> 允许一个单词开头使用通配符（例如"*ing"）特别缓慢，因为索引中的所有词条都需要被检查，以防它们匹配。 将allow_leading_wildcard设置为false可以禁用开头使用通配符。

仅应用在字符级别操作的分析链的部分。 因此，例如，如果分析器执行lowercasing（小写转化）和stemming（词干转化），则只应用lowercasing（小写转化）：在缺少其某些字母的单词上执行stemming（词干转化）可能是错误的。

通过将analyze_wildcard设置为true，将分析以*结尾的查询，并通过确保在第一个N-1 tokens（词元）上进行精确匹配，并在在最后一个tokens（词元）上进行前缀匹配，从不同的tokens（词元）中构建布尔查询。

#### Regular expressions（正则表达式）

正则表达式模式可以通过将它们包装在斜杠（"/"）中来嵌入到查询字符串中：

```
name:/joh?n(ath[oa]n)/
```

支持正则表达式语法的说明在Regular expression syntax。

###### 警告

> allow_leading_wildcard参数无法控制正则表达式。 诸如以下的查询字符串将强制Elasticsearch访问索引中的每个词条：
> 
> ```
> /.*n/
> ```
> 谨慎使用！

#### Fuzziness（模糊性）

我们可以使用"fuzzy"运算符搜索与我们的搜索词条类似但不完全相同的词条：

```
quikc~ brwn~ foks~
```

这使用Damerau-Levenshtein distance来查找最多有两处改变的所有词条，其中改变可以是单个字符的插入，删除或替换，或者两个相邻字符的换位。

默认edit distance（编辑距离）为2，但是编辑距离为1应足以捕获人类全部拼写错误的80％。 它可以指定为：

```
quikc~1
```

#### Proximity searches（邻近搜索）

虽然短语查询（例如"john smith"）期望所有的词条都完全相同的顺序，proximity邻近查询允许指定的单词进一步分开或以不同的顺序。 以相同的方式，fuzzy模糊查询可以为单词中的字符指定最大编辑距离，邻近搜索允许我们指定短语中单词的最大编辑距离：

```
"fox quick"~5
```

字段中的文本越接近查询字符串中指定的原始顺序，则该文档被认为越相关。 与上述示例查询相比，短语"quick fox"将被认为比"quick brown fox"更为相关。

#### Ranges（范围）

可以为日期，数字或字符串字段指定范围。 包含范围使用中括号[min TO max]和不包含范围使用大括号{min TO max}指定。

- 2012年的所有天

```
date:[2012-01-01 TO 2012-12-31]
```

- 数字1..5

```
count:[1 TO 5]
```

- alpha和omega之间的标签，不包括alpha和omega：

```
tag:{alpha TO omega}
```

- 数字从10向上

```
count:[10 TO *]
```

- 2012年之前的日期

```
date:{* TO 2012-01-01}
```

可以组合大括号和中括号：

- 数字从1到5但不包括5

```
count:[1 TO 5}
```

一侧无界的范围可以使用以下语法：

```
age:>10
age:>=10
age:<10
age:<=10
```

###### 注意

> 要使用简化的语法组合上限和下限，你需要使用AND运算符连接两个子句：
> 
```
age:(>=10 AND <20)
age:(+>=10 +<20)
```

查询字符串中范围的解析可能很复杂，容易出错。 使用显式range query更可靠。

#### Boosting（提升）

使用boost操作符^使一个词条比另一个词条更相关。 例如，如果我们想查找有关foxes的所有文档，但是我们对quick foxes尤其感兴趣：

```
quick^2 fox
```

默认boost值为1，但可以为任意正浮点数。 0和1之间的减少了相关性。

Boosts也可以应用于短语或组：

```
"john smith"^2   (foo bar)^4
```

#### Boolean operators（Boolean运算符）

默认情况下，只要一个词条匹配，所有词条都是可选的。 搜索foo bar baz会找到包含一个或多个foo或bar或baz的任何文档。 我们已经讨论了上面的default_operator，它允许你强制所有的词条都是必需的，但是也有一些boolean operators（布尔运算符）可以在查询字符串本身中使用来提供更多的控制。

首选的操作符是+（该词条必须存在）和 - （该词条必须不能存在）。 所有其他词条是可选的。 例如，这个查询：

```
quick brown +fox -news
```

说明如下：

- fox必须存在
- news必须不能存在
- quick和brown是可选的 - 它们的存在增加了相关性

还支持熟悉的操作符AND，OR和NOT（也可以写成&&，||和！）。 但是，这些操作符的效果可能比乍看起来更加复杂。 NOT优先于AND，优先于OR。 虽然+和-仅影响到操作符右边的词条，AND和OR可以影响左右两边的词条。

> 使用AND重写上述查询，OR或NOT表示复杂性：
>
> quick OR brown AND fox AND NOT news
> 
> 这是不正确的，因为brown现在是必需的词条。
> 
> (quick OR brown) AND fox AND NOT news
> 
> 这是不正确的，因为现在需要至少一个quick或brown，并且搜索这些词条的得分将与原始查询不同。
> 
> ((quick AND fox) OR (brown AND fox) OR fox) AND NOT news
> 
> 这种形式现在可以正确地从原始查询复制逻辑，但是相关性得分与原始数据几乎相似。
> 
> 相比之下，使用match query重写的相同查询将如下所示：
> 
```
{
    "bool": {
        "must":     { "match": "fox"         },
        "should":   { "match": "quick brown" },
        "must_not": { "match": "news"        }
    }
}
```

#### Grouping（组）

多个词条或子句可以通过括号被组合在一起，形成子查询：

```
(quick OR brown) AND fox
```

组可用于定位特定字段，或提升子查询的结果：

```
status:(active OR pending) title:(full text search)^2
```

#### Reserved characters（保留字符）

如果你需要使用在查询本身（而不是作为运算符）中作为运算符的任何字符，那么你应该在前面使用反斜杠来转义它们。 例如，要搜索(1+1)=2，你需要将你的查询写为\(1\+1\)\=2。

保留字符为：+ - = && || > < ! ( ) { } [ ] ^ " ~ * ? : \ /

如果未能正确地转义这些特殊字符，可能会导致语法错误，从而阻止查询运行。

###### 注意

> <和>根本无法转义。 阻止他们尝试创建范围查询的唯一方法是完全从查询字符串中删除它们。

###### Watch this space

> 空格也可以是保留字符。 例如，如果你有一个将"wi fi"转换为"wifi"的同义词列表，则对"wi fi"的query_string搜索将失败。 查询字符串解析器会将你的查询解释为搜索"wi OR fi"，而存储在索引中的token词元实际上是"wifi"。 选项split_on_whitespace=false将保护它不被查询字符串解析器所触及，并将使分析在整个输入（"wi fi"）上运行。

#### Empty Query（空查询）

如果查询字符串为空或仅包含空格，查询将产生一个空结果集。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-query-string-query.html
