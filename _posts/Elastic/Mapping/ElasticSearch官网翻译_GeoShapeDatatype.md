---
title: "ElasticSearch官网翻译_GeoShapeDatatype"
date: 2017-05-11 21:51:07
tags: [Elasticsearch]
categories: [Search]
---

#### Geo-Shape datatype

geo_shape数据类型便于对任意地理形状进行索引和搜索，如矩形和多边形。 当被索引的数据或执行的查询包含除了点之外的形状时，应该使用它。

你可以使用geo_shape Query查询使用此类型的文档。

##### 映射选项

geo_shape映射将geo_json几何对象映射到geo_shape类型。 要启用它，用户必须显式地将字段映射到geo_shape类型。

选项|描述|默认
---|---|---
tree|要使用实现PrefixTree的名称：GeoHashPrefixTree的geohash和QuadPrefixTree的quadtree。|geohash
precision|可以使用此参数代替tree_levels来为tree_levels参数设置适当的值。该值指定所需的精度，并且Elasticsearch将计算最佳tree_levels值以符合此精度。值应该是一个数字，后跟一个可选的距离单位。有效距离单位包括：in（英寸），inch（英尺），yd（码），yard（码），mi（米），miles（英里），km（公里），kilometers（公里），m（米），meters（米），cm（厘米），centimeters（厘米），mm（毫米），millimeters（毫米）。|meters（米）
tree_levels|PrefixTree使用的最大层数。这可以用于控制形状表示的精度，因此索引了多少项。默认为所选PrefixTree实现的默认值。由于该参数需要对底层实现的一定程度的了解，用户可以使用精度参数。但是，Elasticsearch仅在内部使用tree_levels参数，即使使用precision参数也是通过映射API返回的。|50m
strategy|策略参数定义了在索引和搜索时间如何表示形状的方法。它也影响可用的功能，因此建议让Elasticsearch自动设置此参数。有两种策略可用：recursive（递归）和term（精确匹配）。term策略仅支持点类型（points_only参数将自动设置为true），而recursive策略支持所有形状类型。|recursive
distance_error_pct|用作对PrefixTree的一个提示，关于它应该是多么精确。默认值为0.025（2.5％），其中0.5为最大支持值。性能说明：如果明确定义了precision或tree_level定义，此值将默认为0。这保证了映射中定义的级别的空间精度。这可以导致具有低误差的高分辨率形状（例如，在1m处具有<0.001误差的大形状）的显着的存储器使用。为了提高索引性能（以查询精度为代价），明确定义tree_level或精度以及合理的distance_error_pct，指出大的形状会有更大的误报。|0.025
orientation（方向）|可选地定义如何解释多边形/多边形的顶点顺序。该参数定义了两个坐标系统规则（右侧或左侧）中的一个，可以用三种不同的方式指定。右手规则：right，ccw，counterclockwise（逆时针），左手规则：left，cw，clockwise（顺时针）。默认方向（counterclockwise，逆时针方向）符合OGC标准，该标准以顺时针顺序以内圈（孔）的逆时针方向定义外圈顶点。在geo_shape映射中设置此参数明确地设置geo_shape字段的坐标列表的顶点顺序，但可以在每个单独的GeoJSON文档中覆盖。|ccw
points_only|将此选项设置为true（默认为false）仅为点形状配置geo_shape字段类型（注意：不支持多点）。这可以优化geohash和quadtree的索引和搜索性能，当知道只有点被索引时。目前，geo_shape查询无法在geo_point字段类型上执行。此选项通过提高geo_shape字段上的点性能来弥合差距，以便geo_shape查询在仅点属性字段上是最佳的。|false

##### Prefix trees

为了有效地表示索引中的形状，使用PrefixTree的实现，将形状转换为表示网格正方形的一系列散列（通常称为"rasters"）。树的概念实际上是PrefixTree使用多个网格层，每个网格层具有越来越高的精度来表示地球。这可以被认为是在较高缩放级别增加地图或图像的细节水平。

提供了多个PrefixTree实现：

- GeohashPrefixTree - 对网格正方形使用geohash。地理位置是纬度和经度交错的位的base32编码字符串。所以哈希越长，它越精确。添加到geohash的每个角色代表另一个树级别，并将5位精度添加到geohash。 geohash表示一个矩形区域，并有32个子矩形。弹性搜索的最大级别是24。
- QuadPrefixTree - 使用quadtree（四叉树）网格方块。与geohash类似，四叉树交错纬度和纬度的位，所得到的散列是有点位的。四叉树中的树级表示该位集中的2位，每个坐标一个。Elasticsearch中四边形树的最大层数为50。

##### 空间策略

PrefixTree实现依赖于SpatialStrategy将所提供的形状分解为近似的网格正方形。 每个策略回答以下内容：

- 可以索引哪种类型的形状？
- 可以使用什么类型的查询操作和形状？
- 它是否支持每个字段多个Shape？

提供了以下策略实现（具有相应的功能）：


策略|支持形状|支持查询|多个形状
---|---|---|---
recursive|All|INTERSECTS, DISJOINT, WITHIN, CONTAINS|Yes
term|Points|INTERSECTS|Yes

##### 准确性

Geo_shape不能提供100％的准确性，并且取决于它的配置方式，它可能会返回某些查询的一些假阳性或假阴性。 为了减轻这一点，重要的是为tree_levels参数选择适当的值，并相应地调整预期。 例如，一个点可能在特定网格单元的边界附近，因此可能不匹配仅匹配其旁边的单元格的查询 - 即使该形状非常接近该点。

举例：

```
{
    "properties": {
        "location": {
            "type": "geo_shape",
            "tree": "quadtree",
            "precision": "1m"
        }
    }
}
```

此映射将位置字段映射到geo_shape类型，使用quad_tree实现，精度为1m。 Elasticsearch将其转换为树的级别26。

##### 性能考虑

Elasticsearch使用prefix tree（前缀树）中的路径作为索引和查询中的匹配项。 level越高（因此精度越高），产生的terms（词条）越多。 当然，计算这些terms，将它们保存在内存中，并将它们存储在磁盘上都有代价的。 特别是在级别较高的tree情况下，即使采用适量的数据，索引也可能变得非常大。 此外，功能的大小也很重要。 大而复杂的多边形可以在较高的tree leve占据很大的空间。 哪个设置是正确的取决于用例。 一般来说，可以降低索引大小和查询性能的准确性。

Elasticsearch中的两个实现的默认值都是索引大小与赤道50米精确度的折中。 这允许索引数千万个形状，而不会使得相对于输入大小太多的结果索引过大。

##### 输入结构

GeoJSON格式用于表示形状作为输入，如下所示：

GeoJSON Type|Elasticsearch Type|Description
---|---|---
Point|point|单一地理坐标。
LineString|linestring|给出两点或更多点的任意一行。
Polygon|polygon|一个封闭的多边形的第一个和最后一个点必须匹配，因此需要n + 1个顶点来创建一个n边多边形和最少4个顶点。
MultiPoint|multipoint|一系列未连接但可能相关的点。
MultiLineString|multilinestring|一系列独立的线条。
MultiPolygon|multipolygon|一组单独的多边形。
GeometryCollection|geometrycollection|与multi*形状类似的GeoJSON形状，除了多种类型可以共存（例如，Point和LineString）之外。
N/A|envelope|通过仅指定左上角和右下角指定的边界矩形或包络。
N/A|circle|由中心点指定的圆和半径为单位，默认为METERS。

注意 : 对于所有类型，内部type（类型）和coordinates（坐标）字段都是必需的。

<b>在GeoJSON，因此Elasticsearch中，正确的坐标顺序是坐标数组内的longitude（经度），latitude（纬度）（X，Y）。 这与许多通常使用的Geospatial API（例如Google Maps）使用latitude（纬度），longitude（经度）（Y，X）的顺序不同。</b>

###### Point

一个点是单个地理坐标，例如建筑物的位置或智能手机的Geolocation API给出的当前位置。

```
{
    "location" : {
        "type" : "point",
        "coordinates" : [-77.03653, 38.897676]
    }
}
```

###### LineString

由两个或多个位置的数组定义的linestring（线串）。 通过指定两点，linestring（线串）将代表一条直线。 指定两点以上可创建任意路径。

```
{
    "location" : {
        "type" : "linestring",
        "coordinates" : [[-77.03653, 38.897676], [-77.009051, 38.889939]]
    }
}
```

以上linestring（线串）将从白宫开始直线到美国国会大厦。

###### Polygon

多边形由点列表的列表定义。 每个（外部）列表中的第一个和最后一个点必须相同（多边形必须关闭）。

```
{
    "location" : {
        "type" : "polygon",
        "coordinates" : [
            [ [100.0, 0.0], [101.0, 0.0], [101.0, 1.0], [100.0, 1.0], [100.0, 0.0] ]
        ]
    }
}
```

第一个数组表示多边形的外边界，其他数组表示内部形状("holes"，内孔)：

```
{
    "location" : {
        "type" : "polygon",
        "coordinates" : [
            [ [100.0, 0.0], [101.0, 0.0], [101.0, 1.0], [100.0, 1.0], [100.0, 0.0] ],
            [ [100.2, 0.2], [100.8, 0.2], [100.8, 0.8], [100.2, 0.8], [100.2, 0.2] ]
        ]
    }
}
```

重要说明：GeoJSON不要求顶点的特定顺序，因此可能在数据线周围有不明确的多边形。 为了减轻歧义，Open Geospatial Consortium（开放地理空间联盟）（OGC）简单特征访问规范定义了以下顶点排序：

- 外环 - 逆时针
- 内圈/孔 - 顺时针

对于不跨越日期线的多边形，顶点顺序在弹性搜索中并不重要。 对于跨越数据线的多边形，Elasticsearch要求顶点排序符合OGC规范。 否则，可能会创建一个意外的多边形，并返回意外的查询/过滤器结果。

以下提供了一个模糊多边形的示例。 弹性搜索将应用OGC标准来消除歧义，导致跨越日期线的多边形。

```
{
    "location" : {
        "type" : "polygon",
        "coordinates" : [
            [ [-177.0, 10.0], [176.0, 15.0], [172.0, 0.0], [176.0, -15.0], [-177.0, -10.0], [-177.0, 10.0] ],
            [ [178.2, 8.2], [-178.8, 8.2], [-180.8, -8.8], [178.2, 8.8] ]
        ]
    }
}
```

在设置geo_shape映射时可以定义orientation（方向）参数。 这将在映射的geo_shape字段上定义坐标列表的顶点顺序。 它也可以在每个文档上被覆盖。 以下是覆盖文档方向的示例：

```
{
    "location" : {
        "type" : "polygon",
        "orientation" : "clockwise",
        "coordinates" : [
            [ [-177.0, 10.0], [176.0, 15.0], [172.0, 0.0], [176.0, -15.0], [-177.0, -10.0], [-177.0, 10.0] ],
            [ [178.2, 8.2], [-178.8, 8.2], [-180.8, -8.8], [178.2, 8.8] ]
        ]
    }
}
```

###### MultiPoint

geojson点列表。

```
{
    "location" : {
        "type" : "multipoint",
        "coordinates" : [
            [102.0, 2.0], [103.0, 2.0]
        ]
    }
}
```

###### MultiLineString

一个geojson线串列表。

```
{
    "location" : {
        "type" : "multilinestring",
        "coordinates" : [
            [ [102.0, 2.0], [103.0, 2.0], [103.0, 3.0], [102.0, 3.0] ],
            [ [100.0, 0.0], [101.0, 0.0], [101.0, 1.0], [100.0, 1.0] ],
            [ [100.2, 0.2], [100.8, 0.2], [100.8, 0.8], [100.2, 0.8] ]
        ]
    }
}
```

###### MultiPolygon

一个geojson多边形列表。

```
{
    "location" : {
        "type" : "multipolygon",
        "coordinates" : [
            [ [[102.0, 2.0], [103.0, 2.0], [103.0, 3.0], [102.0, 3.0], [102.0, 2.0]] ],

            [ [[100.0, 0.0], [101.0, 0.0], [101.0, 1.0], [100.0, 1.0], [100.0, 0.0]],
              [[100.2, 0.2], [100.8, 0.2], [100.8, 0.8], [100.2, 0.8], [100.2, 0.2]] ]
        ]
    }
}
```

###### Geometry Collection

一个geojson几何对象的集合。

```
{
    "location" : {
        "type": "geometrycollection",
        "geometries": [
            {
                "type": "point",
                "coordinates": [100.0, 0.0]
            },
            {
                "type": "linestring",
                "coordinates": [ [101.0, 0.0], [102.0, 1.0] ]
            }
        ]
    }
}
```

###### Envelope

Elasticsearch支持envelope（包络）类型，它由形状的左上角和右下角的坐标组成，用于表示边界矩形：

```
{
    "location" : {
        "type" : "envelope",
        "coordinates" : [ [-45.0, 45.0], [45.0, -45.0] ]
    }
}
```

###### Circle

Elasticsearch支持一个circle（圆形）类型，由一个半径为中心点组成：

```
{
    "location" : {
        "type" : "circle",
        "coordinates" : [-45.0, 45.0],
        "radius" : "100m"
    }
}
```

注意：内radius（半径）字段是必需的。 如果未指定，则radius（半径）单位默认为METERS。

##### 排序和检索索引形状

由于形状复杂的输入结构和索引表示，目前还不可能直接排序形状或检索其字段。 geo_shape值只能通过_source字段检索。


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/geo-shape.html
