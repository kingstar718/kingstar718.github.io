---
author: "kingstar718"
title: 'ElasticSearch Java API GEO操作（REST命令版）'
date: 2023-10-18
tags: ["java","elasticsearch","geo"]
draft: false
---

## 前言

ElasticSearch支持地理空间数据查询、搜索，提供 `geo_point`、`geo_shape`两种地理数据类型。

`geo_point`用于描述一个或多个地理坐标点，主要用于周边位置查询、边界内搜索点、聚合多个范围内的点等功能。

`geo_shape`用于描述点线面等多种地理数据，使用GeoJson标准格式描述，可以对这些地理数据做相交、不相交、包含等等地理关系的判断与计算。

## 新增索引

```json
PUT /my_locations
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      }
    }
  }
}
```

PUT后边紧跟的就是需要创建的索引名：`my_locations`。`mappings`中设置了一个属性 `name`，为 `text`类型。

## 更新geo类型的mappings

新增两个字段，`location`字段类型为 `geo_point`，`polygon`字段类型为 `geo_shape`。

`PUT`后边紧跟索引名 `my_locations`，加 `_mapping`表示在操作mappings。

```json
PUT /my_locations/_mapping
{
  "properties": {
    "name": {
      "type": "text"
    },
    "location": {
      "type": "geo_point"
    },
    "polygon": {
      "type": "geo_shape"
    }
  }
}
```

## 查询修改后的索引

```json
GET my_locations/_mapping
{
  "my_locations": {
    "mappings": {
      "properties": {
        "location": {
          "type": "geo_point"
        },
        "name": {
          "type": "text"
        },
        "polygon": {
          "type": "geo_shape"
        }
      }
    }
  }
}
```

注意不要在请求体里的 `body`里填 `{}`，会报错。

## 新增geo数据

固有格式就是 `/索引名/_doc/id`，表示添加数据。`geo_point`支持普通经纬度数组、`WKT`、`geohash`格式的地理数据。

三种类型的经纬度格式都是经度在前，纬度在后。

新增经纬度数组类型的地理坐标点数据：

```json
POST /my_locations/_doc/1
{
  "name": "喜茶",
  "location": [
    113.947539,
    22.530122
  ]
}
```

新增 `WKT`类型的地理坐标点数据：

```json
POST my_locations/_doc/2
{
  "name": "喜茶2",
  "location": "POINT(113.947539 22.530122)"
}
```

新增 `geohash`类型的地理坐标点数据：

```json
POST my_locations/_doc/3
{
  "name": "喜茶geohash",
  "location": "ws100vqp6p"
}
```

新增另一个地点

`POST my_locations/_doc/4`

```json
{
  "name": "平安银行",
  "location": [
    113.94580499999999,
    22.530055
  ]
}
```

## 距离查询

传入一个坐标原点和相应的距离范围，来搜索范围内是否存在坐标点：

```json
GET my_locations/_search
{
  "query": {
    "bool": {
      "must": {
        "match_all": {}
      },
      "filter": {
        "geo_distance": {
          "distance": "500m",
          "location": [
            113.948769,
            22.530063
          ]
        }
      }
    }
  }
}
```

其中距离单位参考：[https://www.elastic.co/guide/en/elasticsearch/reference/current/api-conventions.html#distance-units](https://www.elastic.co/guide/en/elasticsearch/reference/current/api-conventions.html#distance-units)
常见的公里为 `km`，米为 `m`，默认是米。

## 聚合查询

定义一个原点和一组距离范围桶，聚合评估每个文档值与原点的距离。

```json
GET my_locations/_search?size=0&filter_path=aggregations
{
  "aggs": {
    "data_around_city": {
      "geo_distance": {
        "unit": "m",
        "field": "location",
        "origin": "113.94823,22.530143",
        "ranges": [
          {
            "to": 100
          },
          {
            "from": 100,
            "to": 300
          },
          {
            "from": 600,
            "to": 1000
          },
          {
            "from": 1000
          }
        ]
      }
    }
  }
}
```

## 地理多边形内的点

传入一个多边形来判断多边形范围内是否有坐标点，使用 `within`方法表示内部。

```json
GET my_locations/_search
{
  "query": {
    "bool": {
      "must": {
        "match_all": {}
      },
      "filter": {
        "geo_shape": {
          "location": {
            "shape": {
              "type": "polygon",
              "relation": "within",
              "coordinates": [
                [
                  [
                    113.942373,
                    22.531399
                  ],
                  [
                    113.955111,
                    22.531432
                  ],
                  [
                    113.953261,
                    22.524538
                  ],
                  [
                    113.940451,
                    22.525991
                  ]
                ]
              ]
            }
          }
        }
      }
    }
  }
}
```

## 新增面数据

es支持的另一种地理类型字段为 `geo_shape`，包括点、线、多线、面、多面等常见地理类型数据。支持 `GeoJson`和 `WKT`两种标准的地理数据格式。

为第一个喜茶新增多边形的 `GeoJson`数据：

```json
POST my_locations/_doc/1
{
  "name": "喜茶",
  "location": [
    113.947539,
    22.530122
  ],
  "polygon": {
    "type": "envelope",
    "coordinates": [
      [
        113.947248,
        22.530295
      ],
      [
        113.94776,
        22.529753
      ]
    ]
  }
}
```

为第二个喜茶新增 `WKT`数据

```json
POST my_locations/_doc/2
{
  "name": "喜茶2",
  "location": "POINT(113.947539 22.530122)",
  "polygon": "POLYGON ((113.947305 22.530055, 113.947682 22.530047, 113.947718 22.529788, 113.947341 22.529842, 113.947305 22.530055)) "
}
```

## 通过点判断落面

判断传入的点落在是否被包含(`contains`)在相应的多边形内：

```json
GET my_locations/_search
{
  "query": {
    "bool": {
      "must": {
        "match_all": {}
      },
      "filter": {
        "geo_shape": {
          "polygon": {
            "shape": {
              "type": "point",
              "coordinates": [
                113.947305,
                22.530055
              ]
            },
            "relation": "contains"
          }
        }
      }
    }
  }
}
```

## 判断面相交

```json
GET my_locations/_search
{
  "query": {
    "bool": {
      "must": {
        "match_all": {}
      },
      "filter": {
        "geo_shape": {
          "polygon": {
            "shape": {
              "type": "polygon",
              "relation": "intersects",
              "coordinates": [
                [
                  [
                    113.947522,
                    22.530091
                  ],
                  [
                    113.948169,
                    22.529966
                  ],
                  [
                    113.948187,
                    22.529695
                  ],
                  [
                    113.94754,
                    22.529707
                  ]
                ]
              ]
            }
          }
        }
      }
    }
  }
}
```

## 参考

1. [Elasticsearch：Geo Point 和 Geo Shape 查询解释](https://juejin.cn/post/7213296062084415544#heading-9)
2. [Elasticsearch地理空间之geo_shape](https://blog.csdn.net/lc_2014c/article/details/123131377)
3. [ES地理范围查询第一讲：Java操作地理位置信息（geo_point）](https://blog.csdn.net/z69183787/article/details/105559022)
4. [Elasticsearch之（53）Java API 基于geo_shape根据坐标查找 坐标落在店铺范围的店铺](https://blog.csdn.net/wuzhiwei549/article/details/80570644)
5. [Geo queries](https://www.elastic.co/guide/en/elasticsearch/reference/current/geo-queries.html)
6. [百度地图坐标拾取系统](https://api.map.baidu.com/lbsapi/getpoint/)
7. [在线经纬度转geohash](https://zhannicholas.github.io/baidumap_geohash_explorer.html)
8. [百度地图geohash可视化工具](https://blog.csdn.net/nicholas1328/article/details/128164853)
