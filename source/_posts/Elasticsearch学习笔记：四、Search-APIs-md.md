---
title: Elasticsearch学习笔记：四、Search-APIs.md
date: 2016-11-21 10:30:17
tags: Elasticsearch
---
## 简介
### Routing
当执行搜索时，将会对所有索引分片进行广播（轮询所有备份）。这个可以通过设置 routing 参数来指定搜索的分片。
例如，在创建索引时可以指定routing：
```
$ curl -XPOST 'http://localhost:9200/twitter/tweet?routing=kimchy' -d '{
    "user" : "kimchy",
    "postDate" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
'
```

然后我们在后面查询时指定相应routing：
```
$ curl -XGET 'http://localhost:9200/twitter/tweet/_search?routing=kimchy' -d '{
    "query": {
        "bool" : {
            "must" : {
                "query_string" : {
                    "query" : "some query string here"
                }
            },
            "filter" : {
                "term" : { "user" : "kimchy" }
            }
        }
    }
}
'
```

routing参数可以设置为多个，用 , 隔开。

### Stats Group
还可以关联数据组，数据组可以通过后面index API中的indices stats API 来设置。

```
{
    "query" : {
        "match_all" : {}
    },
    "stats" : ["group1", "group2"]
}
```

### Global Search Timeout
es可以设置一个集群级的超时时间，在`search.default_search_timeout` 中，设为-1 的话表示没有超时时间。

## Search
### Multi-Index，Multi-Type
所有search APIs 都可以跨 index和type来进行查询。例如：
```
GET /twitter/_search?q=user:kimchy
```
也可以指定多个type:
```
GET /twitter/tweet,user/_search?q=user:kimchy
```
还可以通过一个tag来搜索相关的几个index:
```
GET /kimchy,elasticsearch/tweet/_search?q=tag:wow
```
或者搜索所有index，使用_all：
```
GET /_all/tweet/_search?q=tag:wow
```
甚至所有index和type：
```
GET /_search?q=tag:wow
```

默认情况下，es会拒绝搜索超过1000个分片的查询。因为这会严重影响es的性能。通常的做法是，将分片设置的更大，数量更少，可以通过`action.search.shard_count.limit`来设置。

## URI Search
一个搜索请求可以通过URI中带有参数来实现，当然，并不是所有搜索都能使用这个模式。
```
GET twitter/tweet/_search?q=user:kimchy
```
返回：
```
{
  "took": 38,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 1.4054651,
    "hits": [
      {
        "_index": "twitter",
        "_type": "tweet",
        "_id": "AViEvWlRDD544hrngUp_",
        "_score": 1.4054651,
        "_routing": "kimchy",
        "_source": {
          "user": "kimchy",
          "postDate": "2009-11-15T14:12:12",
          "message": "trying out Elasticsearch"
        }
      }
    ]
  }
}
```

### 参数

- q: 查询字符串。
- df: 在查询中没定义字段时默认使用的字段。
- analyzer ： 指定使用的查询分析器。
- lowercase_expanded_terms ： 瓷套是否自动变成小写。默认是true。
- analyze_wildcard ： 通配符和前缀查询是否要被分析器分析。默认是false。
- default_operator ： 默认的操作符，AND /OR ，默认是OR。
- lenient : 如果设置为true，则会忽略类型错误，比如向数字字段传文本数据。
- explain : 对于所有命中的数据，计算命中分数。
- _source： 返回的结果中是否包含source。还可以设置source中的字段。
- stored_fields ： 选择命中后返回的字段，使用 ， 分隔。
- sort ： 排序。
- track_scores : 当设置sort时，将它设为true，将会返回track score。
- timeout： 超时时间。默认是没有超时时间。
- terminate_after :  一个Shard可以在获取到多少条数据后，停止查询。
- from/size : 分页参数。
- search_type : search的类型：dfs_query_then_fetch和query_then_fetch。默认是后者。


## Request Body Search
search请求可以通过search DSL ：
```
GET /twitter/tweet/_search
{
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```
返回：
```
{
  "took": 856,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 1.4054651,
    "hits": [
      {
        "_index": "twitter",
        "_type": "tweet",
        "_id": "AViEvWlRDD544hrngUp_",
        "_score": 1.4054651,
        "_routing": "kimchy",
        "_source": {
          "user": "kimchy",
          "postDate": "2009-11-15T14:12:12",
          "message": "trying out Elasticsearch"
        }
      }
    ]
  }
}
```

### 快速确定是否有匹配的文档
如果我们只是想查看是否有匹配的文档，而不在乎返回的数据，可以将size设为0。还可以将termnate_after设为1。在查找到第一条后不再进行搜索。 
```
GET /_search?q=message:elasticsearch&size=0&terminate_after=1
```

返回：
```
{
  "took": 846,
  "timed_out": false,
  "terminated_early": true,
  "_shards": {
    "total": 36,
    "successful": 36,
    "failed": 0
  },
  "hits": {
    "total": 2,
    "max_score": 0,
    "hits": []
  }
}
```

### Query
在request body中可以使用query DSL定义查询：
```
GET /_search
{
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

### From/Size
```
GET /_search
{
    "from" : 0, "size" : 10,
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

### Sort
可以将一个或者多个字段进行排序。每个排序都可以正序倒序。排序定义在字段级别，可以通过指定字段名进行评分_score，并根据评分进行排序。或者_doc来根据索引顺序排序。
```
PUT  localhost:9200/my_index
{
    "mappings": {
        "my_type": {
            "properties": {
                "post_date": { "type": "date" },
                "user": {
                    "type": "string"
                },
                "name": {
                    "type": "string"
                },
                "age": { "type": "integer" }
            }
        }
    }
}
```

查询时，可以设置每个字段的排序方式：
```
POST localhost:9200/my_index/my_type/_search
{
    "sort" : [
        { "post_date" : {"order" : "asc"}},
        "user",
        { "name" : "desc" },
        { "age" : "desc" },
        "_score"
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```
一般来说，_doc没有实际的用处。但是，如果你不需要进行排序的话，按_doc排序是会增加处理效率的，因此，如果你不关心文档的返回顺序，你可以指定按_doc来排序。

```
POST localhost:9200/my_index/my_type/_search
{
    "sort" : [
        
        "_doc"
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

#### 排序参数
es支持按照数组排序或者多个值进行排序。mode 选项控制了选择类型为数组的字段中的哪个值来进行排序。有以下值：
- min 最小值
- max 最大值
- sum 总和
- avg 平均值
- median 中间值

例子：
```
PUT /my_index/my_type/1?refresh
{
   "product": "chocolate",
    "price": [20, 4]
}

```
排序操作：
```
POST /_search
{
   "query" : {
      "term" : { "product" : "chocolate" }
   },
   "sort" : [
      {"price" : {"order" : "asc", "mode" : "avg"}}
   ]
}
```

#### 对内嵌对象进行排序
es也支持对内嵌对象的字段进行排序。
- nested_path 定义进行排序的内嵌对象的路径。进行排序的字段必须在这个对象中。
- nested_filter 针对内嵌对象的一个过滤器。

例子：
```
POST /_search
{
   "query" : {
      "term" : { "product" : "chocolate" }
   },
   "sort" : [
       {
          "offer.price" : {
             "mode" :  "avg",
             "order" : "asc",
             "nested_path" : "offer",
             "nested_filter" : {
                "term" : { "offer.color" : "blue" }
             }
          }
       }
    ]
}
```
上面针对offer里面的price的平均值进行升序排序，并限定了offer的color字段为blue。

#### 针对字段为空的文档的处理
missing参数可以设置当某个字段为空时，在排序时怎么控制文档的位置。它有以下参数：_last和_first或者自定义的值。
```
GET /_search
{
    "sort" : [
        { "price" : {"missing" : "_last"} }
    ],
    "query" : {
        "term" : { "product" : "chocolate" }
    }
}
```
上面将没有price字段的文档放到最后。

<font color='red'>**注意：如果内嵌对象不满足nested_filter时，也将使用missing 参数的值。**</font>

#### 忽略UnMapped 字段
默认情况下，如果相关的字段没有进行mapping，那么search请求将会失败。unmapped-type参数用来决定使用什么值来进行排序。
```
GET /_search
{
    "sort" : [
        { "price" : {"unmapped_type" : "long"} }
    ],
    "query" : {
        "term" : { "product" : "chocolate" }
    }
}
```
上面的例子的意思是，如果当price没有进行mapping操作，那么，将会把它视为 long 来进行处理。

#### 经纬度距离排序
可以根据经纬度位置上的距离来排序：
```
GET /_search
{
    "sort" : [
        {
            "_geo_distance" : {
                "pin.location" : [-70, 40],
                "order" : "asc",
                "unit" : "km",
                "mode" : "min",
                "distance_type" : "sloppy_arc"
            }
        }
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

参数：
- distance_type：如何计算距离，包括sloppy（默认的），arc（更精准但是更慢）和plane（很快，但是对于长距离和接近两极的计算不准确）。
- mode：当一个字段有多个经纬度的时候怎么办。默认的话，升序排序的时候考虑最近的，降序排序的时候考虑最远的。有min，max，median和avg等。
- unit：用来计算的单位。默认是m（米）。

#### 经纬度作为参数

```
GET /_search
{
    "sort" : [
        {
            "_geo_distance" : {
                "pin.location" : {
                    "lat" : 40,
                    "lon" : -70
                },
                "order" : "asc",
                "unit" : "km"
            }
        }
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

#### 经纬度是string
```
GET /_search
{
    "sort" : [
        {
            "_geo_distance" : {
                "pin.location" : "40,-70",
                "order" : "asc",
                "unit" : "km"
            }
        }
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

#### 地理位置是hash值
```
GET /_search
{
    "sort" : [
        {
            "_geo_distance" : {
                "pin.location" : "drm3btev3e86",
                "order" : "asc",
                "unit" : "km"
            }
        }
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

#### 经纬度是数组
```
GET /_search
{
    "sort" : [
        {
            "_geo_distance" : {
                "pin.location" : [-70, 40],
                "order" : "asc",
                "unit" : "km"
            }
        }
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

#### 多个参考点
```
GET /_search
{
    "sort" : [
        {
            "_geo_distance" : {
                "pin.location" : [[-70, 40], [-71, 42]],
                "order" : "asc",
                "unit" : "km"
            }
        }
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```
最终，这个经纬度的排序会按mode的那几个参数来进行。

#### 基于脚本的排序
可以基于脚本进行排序：
```
GET /_search
{
    "query" : {
        "term" : { "user" : "kimchy" }
    },
    "sort" : {
        "_script" : {
            "type" : "number",
            "script" : {
                "lang": "painless",
                "inline": "doc['field_name'].value * params.factor",
                "params" : {
                    "factor" : 1.1
                }
            },
            "order" : "asc"
        }
    }
}
```

#### 计算得分
当按照字段来排序时，分数是不会被计算的。可以设置 track_scores为true来计算分数。
````
GET /_search
{
    "track_scores": true,
    "sort" : [
        { "post_date" : {"order" : "desc"} },
        { "name" : "desc" },
        { "age" : "desc" }
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
````

#### 针对内存方面的考虑
当执行排序时，排序相关的字段都会被加载到内存中。这就意味着，每个分片都应该有足够的内存来容纳它们。对于string类型，字段排序不应该进行 analyzed/tokenized 操作。对于数字类型，如果可能，尽量将它们设置为具体的类型（short、integer和float等）。

### Source filtering
允许控制_source的命中返回。
一般操作都将返回_source，除非你使用stored_fields 或者将_source设为false。
```
GET /_search
{
    "_source": false,
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```
可以设置哪些字段返回：
```
GET /_search
{
    "_source": "obj.*",
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

或者：
```
GET /_search
{
    "_source": [ "obj1.*", "obj2.*" ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

最终，为了完全控制，还可以通过 includes 和excludes来设置：
```
GET /_search
{
    "_source": {
        "includes": [ "obj1.*", "obj2.*" ],
        "excludes": [ "*.description" ]
    },
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```
### Fields




