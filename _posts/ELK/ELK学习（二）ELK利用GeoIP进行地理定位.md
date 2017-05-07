---
title: "ELK学习（二）ELK利用GeoIP进行地理定位"
date: 2017-01-23 20:15:20
tags: [ELK]
categories: [Log]
---

上一篇我们对Nginx的access.log进行了初步的解析和提取字段处理，如果想进一步对客户端的IP来源进行分析和地理定位，我们需要借助第三方库GeoIP来进行地理定位。

### GeoIP的使用

这里要说明一下GeoIP是一个免费的IP定位数据库，现在目前有两个版本GeoIP和GeoIP2

GeoIP官网地址：

- https://www.maxmind.com/zh/home

这里GeoIP和GeoIP2我都尝试过了，使用GeoIP2是因为GeoIP不支持IPv6，开始access.log日志的IP使用的IPv6，但是ES 2.x版本也只支持IPv4类型，所以为了简单把access.log日志的IP改为了IPv4。下面说一下GeoIP和GeoIP2的安装和使用

##### 安装GeoIP数据库

GeoIP（免费版叫GeoLite）数据库下载地址

- http://dev.maxmind.com/zh-hans/geoip/legacy/geolite/

```
wget http://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.gz
gunzip GeoLiteCity.dat.gz
# 将数据库文件移动到Logstash的安装目录下
mv GeoLiteCity.dat /usr/local/logstash/geoip/
```

##### Logstash配置文件中指定GeoIP数据库文件

Logstash默认是安装了logstash-filter-geoip插件的，所以可以直接使用下面的配置。我们继续上一篇的Logstash配置文件修改如下：

```
if [type] == "nginx_access" {
	grok {
	    match => {
	        "message" => "%{NGINX_ACCESS_LOGS}"
	    }
	}
	if "_grokparsefailure" in [tags] {
	    drop { }
	}
	# 将client_ip为"-"的转换成"0.0.0.0"，或者直接删除client_ip字段，两种方式的效果都一样，如果IP地址是"-"就会删除client_ip字段，否则geoip转换会报错
	if [client_ip] == "-" {
	    mutate {
	        replace => { "client_ip" => "0.0.0.0" }
	        # remove_field => [ "client_ip" ]
	    }
	}
	# 这里是geoip指定字段的用法，不适用于geoip2
	geoip {
		source => "client_ip"
		target => "geoip"
		# 这里指定好解压后GeoIP数据库文件的位置
		database => "/usr/local/logstash/geoip/GeoLiteCity.dat"
		# 直接使用location字段即可，不需要add_field和remove_field，除非不想使用location这个名字把字段换成别的名字
		# 添加自己的坐标字段名称my_location
		# add_field => [ "[geoip][my_location]", "%{[geoip][longitude]}" ]
		# add_field => [ "[geoip][my_location]", "%{[geoip][latitude]}"  ]
		# 指定保留geoip的字段，注意geoip和geoip2字段的区别
		# fields => ["country_name", "country_code2","region_name", "city_name", "real_region_name", "latitude", "longitude"]
		# remove_field => [ "[geoip][longitude]", "[geoip][latitude]" ]
	}
	mutate {
		# 使用自己的坐标字段名称my_location
		# convert => [ "[geoip][my_location]", "float" ]
		convert => [ "[geoip][location]", "float" ]
		# 将request_time和upstream_response_time转换成float，否则ES查询出来的是string类型
		convert => [ "[request_time]", "float" ]
		convert => [ "[upstream_response_time]", "float" ]
	}
}
```

##### GeoIP的格式

```
geoip.city_name：城市名称
geoip.continent_code：洲际编码
geoip.country_code2：国家编码，国际域名缩写
geoip.country_code3：国家编码，国际域名缩写
geoip.country_name：国家名称
geoip.ip：IP地址
* geoip.region_name：中国区域编码
* geoip.real_region_name：中国区域名称
geoip.timezone：时区
geoip.location：经纬度坐标
geoip.latitude：纬度坐标
geoip.longitude：经度坐标
```

##### 安装GeoIP2数据库

GeoIP2（免费版叫GeoLite2）数据库下载地址

- http://dev.maxmind.com/zh-hans/geoip/geoip2/geolite2-开源数据库/

```
wget http://geolite.maxmind.com/download/geoip/database/GeoLite2-City.mmdb.gz
gunzip GeoLite2-City.mmdb.gz
# 将数据库文件移动到Logstash的安装目录下
mv GeoLite2-City.mmdb /usr/local/logstash/geoip/
```

##### Logstash配置文件中指定GeoIP2数据库文件

```
$ cd ${LS_HOME}/bin
$ logstash-plugin install logstash-filter-geoip2
```

这里需要单独安装geoip2插件，因为geoip2插件不是Logstash的官方插件。不使用geoip是因为对IPv6支持的不好，但是ES 2.x版本也只支持IPv4，不支持IPv6类型存储，所以后来只是将access.log的IP地址改为IPv4了。这里只是介绍下如何使用GeoIP2。

```
if [type] == "nginx_access" {
	grok {
	    match => {
	        "message" => "%{NGINX_ACCESS_LOGS}"
	    }
	}
	if "_grokparsefailure" in [tags] {
	    drop { }
	}
	# 将client_ip为"-"的转换成"0.0.0.0"，或者直接删除client_ip字段，两种方式的效果都一样，如果IP地址是"-"就会删除client_ip字段，否则geoip转换会报错
	if [client_ip] == "-" {
	    mutate {
	        replace => { "client_ip" => "0.0.0.0" }
	        # remove_field => [ "client_ip" ]
	    }
	}
	geoip2 {
	    source => "client_ip"
	    target => "geoip"
	    # 这里指定好解压后GeoIP数据库文件的位置
	    database => "/usr/local/logstash/geoip/GeoLite2-City.mmdb"
	    # 这里是geoip2指定字段的用法
	    # fields => ["city_name", "continent_code", "country_code2", "country_code3", "country_name", "dma_code", "ip", "postal_code", "region_name", "region_code", "timezone", "location"]
	}
	mutate {
		convert => [ "[geoip][location]", "float" ]
		# 将request_time和upstream_response_time转换成float，否则ES查询出来的是string类型
		convert => [ "[request_time]", "float" ]
		convert => [ "[upstream_response_time]", "float" ]
	}
}
```

注意这里的field不能直接用geoip插件的写法，因为有一些字段在geoip2中已经被去掉了，例如：real_region_name，否则会报如下错误

```
Unknown error while looking up GeoIP data {:exception=>#<Exception: [real_region_name] is not a supported field option.>, :field=>"client_ip", :event=>#<LogStash::Event:0x4b6443d9 @metadata={"path"=>"/home/yunyu/Downloads/access.log"}, @accessors=#<LogStash::Util::Accessors:0x213c01d
```

下面是GeoIP2的格式，可以对比一下GeoIP的格式，不同的地方用*号标识出来了。

##### GeoIP2的格式

```
geoip.city_name：城市名称
geoip.continent_code：洲际编码
geoip.country_code2：国家编码，国际域名缩写
geoip.country_code3：国家编码，国际域名缩写
geoip.country_name：国家名称
* geoip.dma_code：指定市场区域，Designated Market Area（DMA）
geoip.ip：IP地址
* geoip.postal_code：邮政编码
* geoip.region_code：中国区域编码
* geoip.region_name：中国区域名称
geoip.timezone：时区
geoip.location：经纬度坐标
geoip.latitude：纬度坐标
geoip.longitude：经度坐标
```

### ES的索引Template模板

引用GeoIP之后，对应的ES的索引Template模板也有相应的变化

```
{
  "template": "nginx_access_logs_index_*",
  "order":0,
  "settings": {
      "index.number_of_replicas": "1",
      "index.number_of_shards": "5",
      "index.refresh_interval": "10s"
  },
  "mappings": {
    "_default_": {
      "_all": {
        "enabled": false
      },
      "dynamic_templates": [
        {
          "my_string": {
            "match_mapping_type": "string",
            "mapping": {
              "type": "string",
              "fields": {
                "raw": {
                  "type": "string",
                  "index": "not_analyzed"
                }
              }
            }
          }
        }
      ]
    },
    "nginx_access": {
      "properties": {
        "ident": {
          "type": "string",
          "index": "not_analyzed"
        },
        "auth": {
          "type": "string",
          "index": "not_analyzed"
        },
        "client_ip": {
          "type": "ip"
        },
        "client_timestamp": {
          "type": "string",
          "index": "not_analyzed"
        },
        "http_method": {
          "type": "string",
          "index": "not_analyzed"
        },
        "http_version": {
          "type": "string",
          "index": "not_analyzed"
        },
        "url": {
          "type": "string",
          "analyzer": "ik",
          "search_analyzer": "ik_smart",
          "fields": {
            "raw": {
              "type": "string",
              "index": "not_analyzed"
            }
          }
        },
        "refer_url": {
          "type": "string",
          "analyzer": "ik",
          "search_analyzer": "ik_smart",
          "fields": {
            "raw": {
              "type": "string",
              "index": "not_analyzed"
            }
          }
        },
        "status_code": {
          "type": "string",
          "index": "not_analyzed"
        },
        "http_bytes": {
          "type": "string",
          "index": "not_analyzed"
        },
        "ua": {
          "type": "string",
          "analyzer": "ik",
          "search_analyzer": "ik_smart",
          "fields": {
            "raw": {
              "type": "string",
              "index": "not_analyzed"
            }
          }
        },
        "device_id": {
          "type": "string",
          "index": "not_analyzed"
        },
        "sdk_version": {
          "type": "string",
          "index": "not_analyzed"
        },
        "geoip": {
          "dynamic": true,
          "type": "object",
          "properties": {
            "location": {
              "type": "geo_point"
            }
          }
        },
        "host": {
          "type": "string",
          "index": "not_analyzed"
        },
        "path": {
          "type": "string",
          "index": "not_analyzed"
        },
        "type": {
          "type": "string",
          "index": "not_analyzed"
        },
        "@timestamp": {
          "format": "strict_date_optional_time||epoch_millis",
          "type": "date"
        },
        "@version": {
          "type": "string",
          "index": "not_analyzed"
        }
      }
    }
  }
}
```

这里添加了一个geoip字段，并且使用dynamic，这样允许Logstash的geoip插件将解析后的详细字段也保存到ES索引中。这里geoip插件解析出来会带有一个location字段，这个字段就是经纬度的坐标点，所以这里设置geoip.location字段的类型是geo_point。

从Logstash的1.3.0版本开始，如果GeoIP查询返回纬度和经度，则会创建一个[geoip] [location]字段。 该字段存储为GeoJSON格式。 此外，弹性搜索输出提供的默认Elasticsearch模板将[geoip] [location]字段映射到Elasticsearch geo_point。

以下是elastic.co官方文档对于geo_point类型的解释

##### 用于索引字段

geo_point映射将索引具有lat，lon格式的单个字段。lat_lon选项可以设置为也将.lat和.lon作为数字字段索引，geohash可以设置为true以索引.geohash值。

一个好的做法是启用索引lat_lon，因为geo距离和边界框过滤器可以使用内存检查或使用索引的lat lon值执行，并且它实际上取决于哪个数据集执行得更好。请注意，索引lat lon只有当字段有一个地理点值，而不是多个值时才有意义。

##### Geohashes

地理散列是一种纬度/经度编码的形式，它将地球分成网格。此网格中的每个单元格都由geohash字符串表示。每个单元又可以被进一步细分成由较长串表示的较小单元。因此，geohash越长，单元格越小（从而更精确）。

因为geohash只是字符串，它们可以像任何其他字符串一样存储在倒排索引中，这使得查询非常有效。

如果启用geohash选项，geohash“子字段”将被索引为，例如pin.geohash。geohash的长度由geohash_precision参数控制，geohash_precision参数可以设置为绝对长度（例如12，默认值）或距离（例如1km）。

更有用的是，将geohash_prefix选项设置为true不仅可以索引geohash值，还可以索引所有包围的单元格。例如，u30的geohash将被索引为[u，u3，u30]。此选项可以由Geohash单元格过滤器用于非常有效地查找特定单元格中的地理位置。

geo_point类型支持4种方式，下面是官网给出的例子。

```
# mapping类型是geo_point
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "location": {
          "type": "geo_point"
        }
      }
    }
  }
}

# 方式1
PUT my_index/my_type/1
{
  "text": "Geo-point as an object",
  "location": { 
    "lat": 41.12,
    "lon": -71.34
  }
}

# 方式2
PUT my_index/my_type/2
{
  "text": "Geo-point as a string",
  "location": "41.12,-71.34" 
}

# 方式3
PUT my_index/my_type/3
{
  "text": "Geo-point as a geohash",
  "location": "drm3btev3e86" 
}

# 方式4
PUT my_index/my_type/4
{
  "text": "Geo-point as an array",
  "location": [ -71.34, 41.12 ] 
}

# 查询方式
GET my_index/_search
{
  "query": {
    "geo_bounding_box": { 
      "location": {
        "top_left": {
          "lat": 42,
          "lon": -72
        },
        "bottom_right": {
          "lat": 40,
          "lon": -74
        }
      }
    }
  }
}
```

以下是翻译自elastic.co官方文档

说明：

- 方式1：Geo-point表示为一个object，具有lat和lon两个key
- 方式2：Geo-point表示为一个string，格式为"lat,lon"
- 方式3：Geo-point表示为geohash
- 方式4：Geo-point表示为一个array，格式为: [lon, lat]
- 查询方式：地理边界框查询，它查找落在框内的所有geo-points

注意：

string方式的geo-points顺序是lat,lon，而数组方式的geo-points顺序是反过来的lon,lat。最初，lat,lon用于数组和字符串，但是数组格式早期改变以符合GeoJSON使用的格式。

```
Please note that string geo-points are ordered as lat,lon, while array geo-points are ordered as the reverse: lon,lat.

Originally, lat,lon was used for both array and string, but the array format was changed early on to conform to the format used by GeoJSON.
```

### Kibana配置TileMap

##### 创建TileMap视图

在Visualization中选择创建TileMap

![配置TileMap](http://img.blog.csdn.net/20170124100943166?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

##### TileMap配置后的效果图

然后在索引中选择geoip.location字段，如下图左边所示。注意：我这里替换成了高德地图后的效果，具体可以参考Kibana系列的文章。

![ChinaMap](http://img.blog.csdn.net/20170124101512772?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

##### 配置错误处理

如果配置的过程中报错"No Compatible Fields: The "[nginx_access_logs_index_*]" index pattern does not contain any of the following field types: geo_point"

错误原因：这个错误是因为我们的geoip的location字段类型不是geo_point，之前尝试Logstash写入数据到ES的时候没有在nginx_access_logs_index_*索引模板中明确的指定geoip.location的类型，所以默认使用的dynamic_templates的配置，动态创建出来的geoip.location字段是string类型的，所以需要在ES的模板中指定好geoip.location字段的类型是geo_point。

![Geohash错误](http://img.blog.csdn.net/20170123230821712?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

参考文章：

- https://www.maxmind.com/zh/home
- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/ip.html
- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/geo-point.html
- https://www.elastic.co/guide/en/logstash/2.4/plugins-filters-geoip.html
- http://kibana.logstash.es/content/logstash/plugins/filter/geoip.html
- https://github.com/logstash-plugins/logstash-filter-geoip/issues/12
- http://dev.maxmind.com/geoip/legacy/codes/iso3166/
- http://www.stats.gov.cn/tjsj/tjbz/xzqhdm/
- http://blog.csdn.net/yanggd1987/article/details/50469113