---
title: "ElasticSearch官网翻译_format"
date: 2017-05-13 18:46:14
tags: [Elasticsearch]
categories: [Search]
---

#### format（日期格式）

在JSON文档中，日期表示为字符串。 Elasticsearch使用一组预先配置的格式来识别和解析这些字符串为一个long类型的值，表示UTC中的毫秒值。

除了内置格式，您可以使用熟悉的yyyy/MM/dd语法指定自己的自定义格式：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "date": {
          "type":   "date",
          "format": "yyyy-MM-dd"
        }
      }
    }
  }
}
```

支持日期值的许多API还支持日期数学表达式，例如：now-1m/d - 当前时间，减去一个月，向下舍入到最近的一天。

###### 提示

format格式设置必须具有相同索引中相同名称的字段的相同设置。 可以使用PUT mapping API在现有字段上更新其值。

##### 自定义日期格式

支持完全可定制的日期格式。 这些语法在Joda文档中解释。

##### 内置格式

以下大多数日期都有严格的伴随日期，这意味着，年，月，星期几必须有前置零才能有效。 这意味着像5/11/1这样的日期是无效的，但你需要指定完整的日期，这个例子是2005/11/01。 所以，你需要指定strict_date_optional_time来替换date_optional_time。

下表列出了支持的所有默认ISO格式：

- epoch_millis

  一个毫秒的格式化程序。请注意，该时间戳记允许最多13个字符，因此只支持1653年和2286年之间的日期。在这种情况下，你应该使用不同的日期格式化程序。

- epoch_second

  一个秒的格式化程序。请注意，此时间戳记最多允许10个字符，因此只支持1653年到2286年之间的日期。在这种情况下，你应该使用不同的日期格式化程序。

- date_optional_time或strict_date_optional_time

  通用ISO日期时间解析器，其中日期是必需的，时间是可选的。

- basic_date

  一个基本格式化程序，完整日期的为四位数字年份，两位数字的年份和两位数的日期：yyyyMMdd。

- basic_date_time

  一个基本格式化程序，结合了基本的日期和时间，由T分隔：yyyyMMdd'T'HHmmss.SSSZ。

- basic_date_time_no_millis

  一个基本格式化程序，结合基本的日期和时间没有毫秒，由T分隔：yyyyMMdd'T'HHmmssZ。

- basic_ordinal_date

  一个基本格式化程序，使用四位数字的年份和三位数的日期：yyyyDDD。

- basic_ordinal_date_time

  一个基本格式化程序，使用四位数字年份和三位数字的日期和时间：yyyyDDD'T'HHmmss.SSSZ。

- basic_ordinal_date_time_no_millis

  一个基本格式化程序，使用四位数字的年份和三位数字的日期和时间，没有毫秒：yyyyDDD'T'HHmmssZ。

- basic_time

  一个基本格式化程序，一个两位数的小时，两位数的小时，两位数的秒，三位数的毫秒和时区的偏移：HHmmss.SSSZ。

- basic_time_no_millis

  一个基本格式化程序，一个两位数的小时，两位数的小时，两位数的秒，和时区偏移：HHmmssZ。

- basic_t_time

  一个基本格式化程序，一个两位数的小时，两位数的小时，两位数的分钟，三位数的毫秒，以及由T为前缀的时区：'T'HHmmss.SSSZ。

- basic_t_time_no_millis

  一个基本格式化程序，一个两位数的小时，两位数的小时，两位数的秒，以及以T为前缀的时区偏移量：'T'HHmmssZ。

- basic_week_date或strict_basic_week_date

  一个基本格式化程序，作为四位数的周年，两周的一周周末和一个星期的星期几：xxxx'W'wwe。

- basic_week_date_time或strict_basic_week_date_time

  一个基本格式化程序，结合了基本的周年日期和时间，由T分隔开：xxxx'W'ww'T'HHmmss.SSSZ。

- basic_week_date_time_no_millis或strict_basic_week_date_time_no_millis

  一个基本的格式化程序，结合了基本的周年日期和时间没有毫秒，由T分隔开：xxxx'W'ww'T'HHmmssZ。

- date或strict_date

  一个格式化程序，完整日期为四位数年份，两位数字的年份和两位数的日期：yyyy-MM-dd。

- date_hour或strict_date_hour

  一个格式化程序，结合了一天的完整日期和两位数的小时。

- date_hour_minute或strict_date_hour_minute

  一个格式化程序，结合了一个完整的日期，一个两位数的小时和两位数的小时。

- date_hour_minute_second或strict_date_hour_minute_second

  一个格式化程序，结合了一个完整的日期，一天的两位数字小时，两位数的小时和两位数的秒。

- date_hour_minute_second_fraction或strict_date_hour_minute_second_fraction

  一个格式化程序，结合了一个完整的日期，一个两位数的小时，两位数的小时，两位数的秒，三位数的秒数：yyyy-MM-dd'T'HH:mm:ss.SSS。

- date_hour_minute_second_millis或strict_date_hour_minute_second_millis

  一个格式化程序，结合了一个完整的日期，一个两位数的小时，两位数的小时，两位数的秒，三位数的秒数：yyyy-MM-dd'T'HH:mm:ss.SSS。

- date_time或strict_date_time

  一个格式化程序，结合了完整的日期和时间，由T分隔：yyyy-MM-dd'T'HH:mm:ss.SSSZZ。

- date_time_no_millis或strict_date_time_no_millis

  一个格式化程序，结合了完整的日期和时间，没有毫秒，由T分隔：yyyy-MM-dd'T'HH:mm:ssZZ。

- hour或strict_hour

  一个格式化程序，每天两位数的小时。

- hour_minute或strict_hour_minute

  一个格子化程序，每天两位数的小时和两位数的小时。

- hour_minute_second或strict_hour_minute_second

  一个格子化程序，一天两位数的小时，两位数的分钟小时，两位数的秒。

- hour_minute_second_fraction或strict_hour_minute_second_fraction

  一个格式化程序，一个两位数的小时，两位数的小时，两位数的秒，三位数的秒数：HH:mm:ss.SSS。

- hour_minute_second_millis或strict_hour_minute_second_millis

  一个格式化程序，一个两位数的小时，两位数的小时，两位数的秒，三位数的秒数：HH:mm:ss.SSS。

- ordinal_date或strict_ordinal_date

  一个格式化程序，使用四位数字的年份和三位数的日期。yyyy-DDD。

- ordinal_date_time或strict_ordinal_date_time

  一个格式化程序，使用四位数字年份和三位数字的日期和时间：yyyy-DDD'T'HH:mm:ss.SSSZZ。

- ordinal_date_time_no_millis或strict_ordinal_date_time_no_millis

  一个格式化程序，使用四位数字年份和三位数字的日期和时间，没有毫秒：yyyy-DDD'T'HH:mm:ssZZ。

- time或strict_time

  一个格式化程序，一天两位数的小时，两位数的小时，两位数的秒，三位数的秒数和时区偏移量：HH:mm:ss.SSSZZ。

- time_no_millis或strict_time_no_millis

  一个格式化程序，一个两位数的小时，两位数的小时，两位数的秒，和时区偏移：HH:mm:ssZZ。

- t_time或strict_t_time

  一个两位数字小时的格式化程序，两位数的小时，两位数的秒，三位数的秒数，以及以T为前缀的时区偏移量：'T'HH:mm:ss.SSSZZ。

- t_time_no_millis或strict_t_time_no_millis

  一个格式化程序，一个两位数的小时，两位数的小时，两位数的秒，以及以T为前缀的时区偏移量：'T'HH:mm:ssZZ。

- week_date或strict_week_date

  一个格式化程序，四位数的周年，两周的一周，一周的一个星期：xxxx-'W'ww-e。

- week_date_time或strict_week_date_time

  一个格式化程序，结合了整整一整年的日期和时间，由T分隔：xxxx-'W'ww-e'T'HH:mm:ss.SSSZZ。

- week_date_time_no_millis或strict_week_date_time_no_millis

  一个格式化程序，结合了整整一整年的日期和时间，没有毫秒，由T分隔：xxxx-'W'ww-e'T'HH:mm:ssZZ。

- weekyear or strict_weekyear

  一个四位数的一周格式的格式化程序。

- weekyear_week或strict_weekyear_week

  一个四位数的周年和两个星期的星期的格式化程序。

- weekyear_week_day或strict_weekyear_week_day

  一个四位数周年的格式化程序，一周的两位数周和一周的一位数字。

- year or strict_year

  一个格式化程序，四位数年份。

- year_month或strict_year_month

  一个格式化程序，四位数年份，两位数的月份。

- year_month_day或strict_year_month_day

  一个格式化程序，四位数年份，两位数的月份，两位数的日期。


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping-date-format.html
