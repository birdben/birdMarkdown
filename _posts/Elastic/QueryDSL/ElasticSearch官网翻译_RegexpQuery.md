---
title: "ElasticSearch官网翻译_RegexpQuery"
date: 2017-06-29 10:47:19
tags: [Elasticsearch]
categories: [Search]
---

## Regexp Query

正则表达式查询允许你使用正则表达式进行term queries（词条查询）。 有关支持的正则表达式语言的详细信息，请参阅正则表达式语法。 第一句话中的"term queries"意味着Elasticsearch会将正则表达式应用于该字段分词器生成的词条上，而不是字段的原始文本。

注意：regexp查询的性能很大程度上取决于所选的正则表达式。 匹配一切像".*"非常慢，以及使用环绕正则表达式。 如果可能，你应该尝试在正则表达式开始之前使用长前缀。通配符匹配器".*?+"将大多性能下降。

```
GET /_search
{
    "query": {
        "regexp":{
            "name.first": "s.*y"
        }
    }
}
```

boost也得到支持

```
GET /_search
{
    "query": {
        "regexp":{
            "name.first":{
                "value":"s.*y",
                "boost":1.2
            }
        }
    }
}
```

你也可以使用指定的flags

```
GET /_search
{
    "query": {
        "regexp":{
            "name.first": {
                "value": "s.*y",
                "flags" : "INTERSECTION|COMPLEMENT|EMPTY"
            }
        }
    }
}
```

可能的flags是ALL（默认），ANYSTRING，COMPLEMENT，EMPTY，INTERSECTION，INTERVAL或NONE。 请检查Lucene documentation的含义

正则表达式是危险的，因为很容易意外地创建一个无害的查询，需要指定数量的内部确定的自动机状态（以及相应的RAM和CPU）才能执行Lucene。 Lucene使用max_determinized_states设置（默认为10000）来防止这些。 你可以提高此限制，以允许执行更复杂的正则表达式。

```
GET /_search
{
    "query": {
        "regexp":{
            "name.first": {
                "value": "s.*y",
                "flags" : "INTERSECTION|COMPLEMENT|EMPTY",
                "max_determinized_states": 20000
            }
        }
    }
}
```

### Regular expression syntax（正则表达式语法）

正则表达式查询由regexp和query_string查询支持。 Lucene正则表达式引擎不与Perl兼容的，但是支持较小范围的运算符。

###### 注意

> 我们不会尝试解释正则表达式，而只是解释支持的操作符。

##### Standard operators（标准操作符）

- Anchoring（锚定）

大多数正则表达式引擎允许你匹配字符串的任何部分。 如果你希望正则表达式模式从字符串的开始处开始，或者在字符串的末尾结束，那么你必须具体地anchor（锚定）它，使用"^"来指示开头，或"$"表示结束。

Lucene的模式总是被锚定。 所提供的模式必须与整个字符串匹配。 对于字符串"abcde"：

```
ab.*     # match
abcd     # no match
```

- Allowed characters（允许的字符）

模式中可能使用任何Unicode字符，但某些字符是保留的，必须进行转义。 标准保留字符为：

```
. ? + * | { } [ ] ( ) " \
```

如果启用可选功能（见下文），那么这些字符也可能被保留：

```
# @ & < >  ~
```

任何保留的字符都可以使用反斜杠"\*"进行转义，其中包含字面反斜杠字符："\\"

此外，任何字符（双引号除外）在双引号括起时都被字面解释：

```
john"@smith.com"
```

- Match any character（匹配任何字符）

点"."可用于表示任何字符。 对于字符串"abcde"：

```
ab...   # 匹配
a.c.e   # 匹配
```

- One-or-more（一次或多次）

加号"+"可用于一次或多次重复之前的最短模式。 对于字符串"aaabbb"：

```
a+b+        # 匹配
aa+bb+      # 匹配
a+.+        # 匹配
aa+bbb+     # 匹配
```

- Zero-or-more（零次或多次）

星号"*"可以用于匹配之前的最短模式零次或多次。 对于字符串"aaabbb"：

```
a*b*        # 匹配
a*b*c*      # 匹配
.*bbb.*     # 匹配
aaa*bbb*    # 匹配
```

- Zero-or-one（零次或一次）

问号"?"可选匹配之前的最短模式。 它匹配零次或一次。 对于字符串"aaabbb"：

```
aaa?bbb?    # 匹配
aaaa?bbbb?  # 匹配
.....?.?    # 匹配
aa?bb?      # 不匹配
```

- Min-to-max（最小到最大）

可以使用大括号"{}"来指定之前的最短模式可以重复的最小次数和（可选）最大次数。 允许的表格是：

```
{5}     # 重复精确5次
{2,5}   # 重复至少2次并且最多5次
{2,}    # 重复至少2次
```

对于字符串"aaabbb"：

```
a{3}b{3}        # 匹配
a{2,4}b{2,4}    # 匹配
a{2,}b{2,}      # 匹配
.{3}.{3}        # 匹配
a{4}b{4}        # 不匹配
a{4,6}b{4,6}    # 不匹配
a{4,}b{4,}      # 不匹配
```

- Grouping

括号"()"可用于形成子模式。 上面列出的大量运算符以最短模式运行，可以是一组。 对于字符串"ababab"：

```
(ab)+       # 匹配
ab(ab)+     # 匹配
(..)+       # 匹配
(...)+      # 不匹配
(ab)*       # 匹配
abab(ab)?   # 匹配
ab(ab)?     # 不匹配
(ab){3}     # 匹配
(ab){1,2}   # 不匹配
```

- Alternation（交替）

管道符号"|"充当OR运算符。 如果左侧或右侧的模式匹配，则匹配将成功。 交替适用于最长的模式，而不是最短的模式。 对于字符串"aabb"：

```
aabb|bbaa   # 匹配
aacc|bb     # 不匹配
aa(cc|bb)   # 匹配
a+|b+       # 不匹配
a+b+|b+a+   # 匹配
a+(b|c)+    # 匹配
```

- Character classes（字符类）

潜在字符的范围可以通过将其包围在方括号"[]"中来表示为字符类。 以^开始的使字符类变成否定。 允许的表格是：

```
[abc]   # 'a' 或 'b' 或 'c'
[a-c]   # 'a' 或 'b' 或 'c'
[-abc]  # '-' 或 'a' 或 'b' 或 'c'
[abc\-] # '-' 或 'a' 或 'b' 或 'c'
[^abc]  # 任何字符除了 'a' 或 'b' 或 'c'
[^a-c]  # 任何字符除了 'a' 或 'b' 或 'c'
[^-abc]  # 任何字符除了 '-' 或 'a' 或 'b' 或 'c'
[^abc\-] # 任何字符除了 '-' 或 'a' 或 'b' 或 'c'
```

请注意，破折号"-"表示字符的范围，除非是第一个字符，或者是否使用反斜杠进行转义。

对于字符串"abcd"：

```
ab[cd]+     # 匹配
[a-d]+      # 匹配
[^a-d]+     # 不匹配
```

##### Optional operators（可选操作符）

这些运算符默认可用，因为flags参数默认为ALL。 不同的flag组合（使用"|"连接）可用于enable/disable特定的操作符：

```
{
    "regexp": {
        "username": {
            "value": "john~athon<1-5>",
            "flags": "COMPLEMENT|INTERVAL"
        }
    }
}
```

- Complement（补充）

补充可能是最有用的选项。 符号"~"之后的最短模式被否定。 例如，"ab~cd"表示：

 * 以a开始
 * 其次是b
 * 后面是一个任意长度的字符串，除了c之外
 * 以d结束

对于字符串"abcdef"：

```
ab~df     # 匹配
ab~cf     # 匹配
ab~cdef   # 不匹配
a~(cb)def # 匹配
a~(bc)def # 不匹配
```

使用COMPLEMENT或ALL标志启用。

- Interval（间隔）

间隔选项允许使用由尖括号"<>"括起的数字范围。 对于字符串："foo80"：

```
foo<1-100>     # 匹配
foo<01-100>    # 匹配
foo<001-100>   # 不匹配
```

使用INTERVAL或ALL标志启用。

- Intersection（连接）

符号"&"连接两种模式，两者都必须匹配。 对于字符串"aaabbb"：

```
aaa.+&.+bbb     # 匹配
aaa&bbb         # 不匹配
```

使用此功能通常意味着你应该重写正则表达式。

使用INTERSECTION或ALL标志启用。

- Any string（任何字符串）

符号"@"可以匹配任何字符串的整体。 这可以与上面的intersection和complement相结合来表示"everything except"。 例如：

```
@&~(foo.+)      # 任何字符串除了以foo开始的字符串
```

使用ANYSTRING或ALL标志启用。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-regexp-query.html
