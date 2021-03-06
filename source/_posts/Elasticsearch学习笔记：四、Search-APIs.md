---
title: Elasticsearch学习笔记：四、Search-APIs
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
stored_fields参数是针对已经明确mapping过的，一般不体检使用。可以使用 source filtering来代替。
```
GET /_search
{
    "stored_fields" : ["user", "postDate"],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```
如果想加载所有的字段，可以用 * 。
如果设置为空数组，那么如果命中，将只返回_id 和 _type。
```
GET /_search
{
    "stored_fields" : [],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

如果没有需要的字段，它将会被忽略。

#### 不使用存储的字段
````
GET /_search
{
    "stored_fields": "_none_",
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
````
不使用存储的字段，可以使用_none_
### Script Fields
````
{
    "query" : {
        "match_all": {}
    },
    "script_fields" : {
        "test1" : {
            "script" : {
             
                "inline": "doc['my_field_name'].value * 2"
            }
        },
        "test2" : {
            "script" : {
               
                "inline": "doc['my_field_name'].value * factor",
                "params" : {
                    "factor"  : 2.0
                }
            }
        }
    }
}
````

script_fields 能够操作没存储的字段，并且允许人为定义其返回值。
script_fields 还可以访问年实际存在的 _source 文档，并且指定返回的元素：
```
GET /_search
    {
        "query" : {
            "match_all": {}
        },
        "script_fields" : {
            "test1" : {
                "script" : "_source.obj1.obj2"
            }
        }
    }
```
理解doc['my_field'].value 和 _source.my_field 的区别是很重要的。首先，使用doc，将会导致字段加载进内存，需要更多的内存消耗，但是会获得更快的响应速度。另外，doc[...]符号只允许简单的值的字段，不允许json object类型。
_source只返回json相关的部分。

### Doc value Fields
```
GET /_search
{
    "query" : {
        "match_all": {}
    },
    "docvalue_fields" : ["test1", "test2"]
}
```
docvalue_fields 能够用于没有存储的字段上。
值得注意的是，如果字段的参数制定了没有docvalues的字段，将会从字段数据的缓存中加载值并导致字段中的所有词条加载到内存中，造成更多的内存消耗。

### Post filter
post_filter 用来搜索每个hits，在聚合运算之后。下面是一个解释的例子：

加入你卖衬衫：
```
PUT /shirts
{
    "mappings": {
        "item": {
            "properties": {
                "brand": { "type": "string"},
                "color": { "type": "string"},
                "model": { "type": "string"}
            }
        }
    }
}

PUT /shirts/item/1?refresh
{
    "brand": "gucci",
    "color": "red",
    "model": "slim"
}
```
想象一下，一个用户制定了两个搜索条件：
color:red 和 brand:gucci 。通常，你可以使用：
```
GET /shirts/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "color": "red"   }},
        { "term": { "brand": "gucci" }}
      ]
    }
  }
}
```
返回结果时这样的：
```
{
  "took": 120,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0,
    "hits": [
      {
        "_index": "shirts",
        "_type": "item",
        "_id": "1",
        "_score": 0,
        "_source": {
          "brand": "gucci",
          "color": "red",
          "model": "slim"
        }
      }
    ]
  }
}
```

然而，你还想要展示一系列的选项来让用户点击查询。例如，你有一个model 字段来希望用户进一步查询是 red gucci 的t-shirts 还是dress-shirts，这可以使用terms aggregation来实现：
```
GET /shirts/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "color": "red"   }},
        { "term": { "brand": "gucci" }}
      ]
    }
  },
  "aggs": {
    "models": {
      "terms": { "field": "model" } 
    }
  }
}
```
返回这样的：
```
{
  "took": 73,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0,
    "hits": [
      {
        "_index": "shirts",
        "_type": "item",
        "_id": "1",
        "_score": 0,
        "_source": {
          "brand": "gucci",
          "color": "red",
          "model": "slim"
        }
      }
    ]
  },
  "aggregations": {
    "models": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "slim",
          "doc_count": 1
        }
      ]
    }
  }
}
```
但是，有可能你邮箱告诉user还有多少其他颜色的gucci shirts。如果你仅仅添加一个terms 的aggregation在color字段，你只能得到color为red的，因为你的查询只返回了red gucci shirts。

解决方法是，你想要查询所有color的shirts，先通过聚合，然后使用color的filter来对查询结果进一步过滤，这就是post_filter的用处：
```
GET /shirts/_search
{
  "query": {
    "bool": {
      "filter": {
        "term": { "brand": "gucci" } 
      }
    }
  },
  "aggs": {
    "colors": {
      "terms": { "field": "color" } 
    },
    "color_red": {
      "filter": {
        "term": { "color": "red" } 
      },
      "aggs": {
        "models": {
          "terms": { "field": "model" } 
        }
      }
    }
  },
  "post_filter": { 
    "term": { "color": "red" }
  }
}
```
返回结果：
```
{
  "took": 20,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0,
    "hits": [
      {
        "_index": "shirts",
        "_type": "item",
        "_id": "1",
        "_score": 0,
        "_source": {
          "brand": "gucci",
          "color": "red",
          "model": "slim"
        }
      }
    ]
  },
  "aggregations": {
    "color_red": {
      "doc_count": 1,
      "models": {
        "doc_count_error_upper_bound": 0,
        "sum_other_doc_count": 0,
        "buckets": [
          {
            "key": "slim",
            "doc_count": 1
          }
        ]
      }
    },
    "colors": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "red",
          "doc_count": 1
        }
      ]
    }
  }
}
```
### Highlighting
可以对搜索结果的一个或多个字段进行高亮处理。具体是使用Lucene的plain highlighter ， fast vector highting 和 posting highlighter实现的。
```
GET /_search
{
    "query" : {
        "match": { "user": "kimchy" }
    },
    "highlight" : {
        "fields" : {
            "content" : {}
        }
    }
}
```
在上面的例子中，content字段就是对于每个命中的结果进行高亮的字段。

#### Plain highlighter
默认的高亮方式是通过plain，也就是使用的Lucene的highlighter。它力图反映查询匹配的逻辑中单词理解的重要性和短语查询中单词的定位标准。

如果你想要通过复杂的查询来高亮很多文档中的很多字段，那么它不会很快。因为在执行时，它会创建一个小的内存空间并使用Lucene的查询重新搜索一遍将要被高亮的字段。

#### Postings highlighter
如果在mapping中将 index_options 设置为 offsets，那么将会使用postings highlighter 来代替 plain highlighter。postings highlighter有以下特点：

- 响应比较快，因为他不需要重新分析将被高亮的文本，越大的文件响应速度越快。
- 比term_vectors 需要更少的硬盘空间，需要fast vector highlighter。
- 可以将文本分成句子并高亮他们。对于自然语言发挥得很好,但是不包含例如html标记的字段。
- 将文档视为整个文章，使用BM25算法score每个单独的句子。

Elasticsearch中需要在建立索引的时候映射字段类型，才可以实现postings-highlighter高亮显示，例如对content字段采用postings高亮类型：
```
{
    "type_name" : {
        "content" : {"type":"string","index_options" : "offsets"}
    }
}
```
值得注意的是，postings highlighter 意味着针对简单查询的词条高亮，而不管它们的位置。这意味着，当与短语查询结合使用时，它将高亮构成查询的所有词条，而不管它们是否实际上是查询匹配的一部分，从而有效地忽略了它们的位置。

postings highlighter 不支持高亮一些复杂的查询，比如将type设置match_phrase_prefix的match查询。这种情况下没有被高亮的段落返回。

#### Fast vector highlighter
当mapping中设置term_vector为with_positions_offsets，将会用fast vector highlighter 代替 plain highlighter。
特点：
- 快，尤其对于大于1M的字段。
- 可定制的boundary_chars，boundary_max_scan，和fragment_offset。
- 需要设置term_vector的值为with_positions_offsets，这会增加索引的大小。
- 可以将多个字段的匹配组合成一个结果。
- 可以为不同位置的短语分配不同的权重。

```
{
    "type_name" : {
        "content" : {"term_vector" : "with_positions_offsets"}
    }
}
```

#### Force highlighter type
可以对指定的字段类型进行高亮。这对于要使用term_vectors对字段进行高亮非常有用。可以允许的类型为：plain，postings和fvh。
```
GET /_search
{
    "query" : {
        "match": { "brand": "gucci" }
    },
    "highlight" : {
        "fields" : {
            "brand" : {"type" : "plain"}
        }
    }
}
```
响应：
```
{
  "took": 18,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0.30685282,
    "hits": [
      {
        "_index": "shirts",
        "_type": "item",
        "_id": "1",
        "_score": 0.30685282,
        "_source": {
          "brand": "gucci",
          "color": "red",
          "model": "slim"
        },
        "highlight": {
          "brand": [
            "<em>gucci</em>"
          ]
        }
      }
    ]
  }
}
```
#### Force highlighter on source
强制高亮source中的字段，即使字段被分开存储。默认是false。
```
GET /_search
{
    "query" : {
        "match": { "brand": "gucci" }
    },
    "highlight" : {
        "fields" : {
            "brand" : {"force_source" : true}
        }
    }
}
```
这样返回的内容是：
```
{
  "took": 130,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0.30685282,
    "hits": [
      {
        "_index": "shirts",
        "_type": "item",
        "_id": "1",
        "_score": 0.30685282,
        "_source": {
          "brand": "gucci",
          "color": "red",
          "model": "slim"
        },
        "highlight": {
          "brand": [
            "<em>gucci</em>"
          ]
        }
      }
    ]
  }
}
```
#### Highlighting tags
默认情况下，被高亮的文本的包围标签是<em></em>，这个可以自定义设置：
```
GET /_search
{
    "query" : {
        "match": { "brand": "gucci" }
    },
    "highlight" : {
        "pre_tags" : ["<tag1>"],
        "post_tags" : ["</tag1>"],
        "fields" : {
            "brand" : {"force_source" : true}
        }
    }
}
```
响应：
```
{
  "took": 3,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0.30685282,
    "hits": [
      {
        "_index": "shirts",
        "_type": "item",
        "_id": "1",
        "_score": 0.30685282,
        "_source": {
          "brand": "gucci",
          "color": "red",
          "model": "slim"
        },
        "highlight": {
          "brand": [
            "<tag1>gucci</tag1>"
          ]
        }
      }
    ]
  }
}
```
使用fast vector highlighter 能够使用更多的tag。
```
GET /_search
{
    "query" : {
        "match": { "user": "kimchy" }
    },
    "highlight" : {
        "pre_tags" : ["<tag1>", "<tag2>"],
        "post_tags" : ["</tag1>", "</tag2>"],
        "fields" : {
            "_all" : {}
        }
    }
}
```
也可以使用样式，例如想实现这样的样式：
```
<em class="hlt1">, <em class="hlt2">, <em class="hlt3">,
<em class="hlt4">, <em class="hlt5">, <em class="hlt6">,
<em class="hlt7">, <em class="hlt8">, <em class="hlt9">,
<em class="hlt10">
```
可以这样：
```
GET /_search
{
    "query" : {
        "match": { "user": "kimchy" }
    },
    "highlight" : {
        "tags_schema" : "styled",
        "fields" : {
            "content" : {}
        }
    }
}
```
#### Encoder
encoder参数用于定义highlighter的文本是怎样的编码。可以是default或者html。

#### Highlighted Fragments
可以控制高亮字段的字符数大小，默认是100，还有最大片段数，默认是5:
```

{
    "query" : {
        "match": { "color": "red" }
    },
    "highlight" : {
        "fields" : {
            "color" : {"fragment_size" : 1, "number_of_fragments" : 1}
        }
    }
}
```
当使用postings highlighter时，fragment_size会被忽略。
可以针对评分进行排序：
```
{
    "query" : {
        "match": { "color": "red" }
    },
    "highlight" : {
        "fields" : {
        	"order" : "score",
            "color" : {"force_source" : true ,"fragment_size" : 1, "number_of_fragments" : 1}
        }
    }
}
```

#### Highlight query
使用 highlight_query 可以针对搜索结果进行高亮内的二次搜索。这对于如果你想对没有考虑高亮的查询进行重新评分查询是非常有用的。
```
{
    "stored_fields": [ "_id" ],
    "query" : {
        "match": {
            "content": {
                "query": "foo bar"
            }
        }
    },
    "rescore": {
        "window_size": 50,
        "query": {
            "rescore_query" : {
                "match_phrase": {
                    "content": {
                        "query": "foo bar",
                        "slop": 1
                    }
                }
            },
            "rescore_query_weight" : 10
        }
    },
    "highlight" : {
        "order" : "score",
        "fields" : {
            "content" : {
                "fragment_size" : 150,
                "number_of_fragments" : 3,
                "highlight_query": {
                    "bool": {
                        "must": {
                            "match": {
                                "content": {
                                    "query": "foo bar"
                                }
                            }
                        },
                        "should": {
                            "match_phrase": {
                                "content": {
                                    "query": "foo bar",
                                    "slop": 1,
                                    "boost": 10.0
                                }
                            }
                        },
                        "minimum_should_match": 0
                    }
                }
            }
        }
    }
}
```

#### Global Settings
```
{
    "query" : {
        "match": { "user": "kimchy" }
    },
    "highlight" : {
        "number_of_fragments" : 3,
        "fragment_size" : 150,
        "fields" : {
            "_all" : { "pre_tags" : ["<em>"], "post_tags" : ["</em>"] },
            "bio.title" : { "number_of_fragments" : 0 },
            "bio.author" : { "number_of_fragments" : 0 },
            "bio.content" : { "number_of_fragments" : 5, "order" : "score" }
        }
    }
}
```

#### Require Field Match
将require_field_match设为false可以导致不管查询是否匹配，都会高亮显示。默认是true，意味着只有匹配的才高亮显示。
```
GET /_search
{
    "query" : {
        "match": { "user": "kimchy" }
    },
    "highlight" : {
        "require_field_match": false,
        "fields": {
                "_all" : { "pre_tags" : ["<em>"], "post_tags" : ["</em>"] }
        }
    }
}
```

#### Boundary Characters
当使用fast vector highlighter 高亮一个字段时，boundary_chars 能够设置边界字符。默认是 .,!? \t\n 。
boundary_max_scan允许控制查找边界字符的距离，默认值为20。

#### Matched Fields
Fast Vector Highlighter 能够组合多个字段上的匹配并使用 matched_fields 突出单个字段。这对于使用不同分析器来处理同一字符串来说是最直观的。所有matched_fields 必须将 term_vector 设置为 with_positions_offsets ，但只会加载匹配的组合字段，因此只有该字段可以在store设置为yes时受益。

下面的例子中，content 使用 english 分析器 ， content.plain使用 standard分析器。
```
GET /_search
{
    "query": {
        "query_string": {
            "query": "content.plain:running scissors",
            "fields": ["content"]
        }
    },
    "highlight": {
        "order": "score",
        "fields": {
            "content": {
                "matched_fields": ["content", "content.plain"],
                "type" : "fvh"
            }
        }
    }
}
```
上面的查询中会匹配所有 "run with scissors" 和 "running with scissors"，而不是"run"。如果所有的词组都在一个大的文档中出现，那么"running with scissors" 会排在 "run with scissors"上面，因为在段落中它会更匹配。
```
GET /_search
{
    "query": {
        "query_string": {
            "query": "running scissors",
            "fields": ["content", "content.plain^10"]
        }
    },
    "highlight": {
        "order": "score",
        "fields": {
            "content": {
                "matched_fields": ["content", "content.plain"],
                "type" : "fvh"
            }
        }
    }
}
```
#### Phrase Limit
fast vector highlighter 可以通过phrase_limit 参数来防止分析过多的词组或消耗太多内存。默认是256，也就意味着只有文档前面的256个匹配的词组可以使用。你可以根据自己的需求适量提高这个限制，并且要考虑各方面平衡。
如果使用matched_fields，每个匹配字段的phrase_limit短语被考虑。

#### Field Highlight Order
Elasticsearch按照它们发送的顺序来高亮字段。每个json对象是无序的，但如果你需要明确的字段高亮的顺序，你可以使用数组，如：
```
"highlight": {
        "fields": [
            {"title":{ /*params*/ }},
            {"text":{ /*params*/ }}
        ]
    }
```
### Rescoring
Rescoring能够通过重新排序来精确query和post_filter的查询结果，使用了二次算法，而不是对整个索引进行操作。
rescore请求会被在每一个分片上执行，在返回被节点排序的整个查询请求结果之前。
当前，rescore API 只有一个实现：query rescorer--通过使用查询来调整得分。

注意，当使用了sort时，rescore是不会执行的。
当使用了分页功能时，遍历每一个页面是不应该改变window_size的（通过传不同的from的值），因为这会改变最上面的命中结果，使得用户在遍历页面时感到困惑。

#### Query rescorer
query rescorer 只会在query 和 post_filter查询返回的头N条进行二次查询。每个分片检查的文档数量可以通过window_size来控制，默认情况是 from / size。
默认情况下，最初查询的得分和rescore查询的得分会先行的组成最终的每个文档的_score。这两个组合的权重比例可以通过query_weight和rescore_query_weight来控制，默认都是1.

例子：
```
{
   "query" : {
      "match" : {
         "brand" : {
            "operator" : "or",
            "query" : "gucci",
            "type" : "boolean"
         }
      }
   },
   "rescore" : {
      "window_size" : 50,
      "query" : {
         "rescore_query" : {
            "match" : {
               "brand" : {
                  "query" : "asd",
                  "type" : "phrase",
                  "slop" : 2
               }
            }
         },
         "query_weight" : 0,
         "rescore_query_weight" : 1
      }
   }
}
```
也可以通过score_mode来控制得分：
- total: 默认的设置，会将最初的得分和resscore加起来。
- multiply：最初得分 * rescore。
- avg： 平均分。
- max： 最大值。
- min： 最小值。

#### Multiple Rescores
```
curl -s -XPOST 'localhost:9200/_search' -d '{
   "query" : {
      "match" : {
         "field1" : {
            "operator" : "or",
            "query" : "the quick brown",
            "type" : "boolean"
         }
      }
   },
   "rescore" : [ {
      "window_size" : 100,
      "query" : {
         "rescore_query" : {
            "match" : {
               "field1" : {
                  "query" : "the quick brown",
                  "type" : "phrase",
                  "slop" : 2
               }
            }
         },
         "query_weight" : 0.7,
         "rescore_query_weight" : 1.2
      }
   }, {
      "window_size" : 10,
      "query" : {
         "score_mode": "multiply",
         "rescore_query" : {
            "function_score" : {
               "script_score": {
                  "script": {
                    "lang": "painless",
                    "inline": "Math.log10(doc['numeric'].value + 2)"
                  }
               }
            }
         }
      }
   } ]
}
'
```
第一个查询查询到结果，然后第二个查询通过第一个查询结果来进行查询，依次类推。第二次查询会查看第一次查询结果的排序，因此它可能会在看第一次rescore时使用一个大的window，在第二次rescore时使用小一些的window。

### Scroll
与使用search请求一个单独的页面相比，scroll API能够用来获得更多数量的结果（甚至全部结果）通过一个请求，和数据库中的游标差不多。
scroll请求返回的结果反映了当时search请求返回的索引的状态，像一个当时的快照。对文档（索引，更新或删除）的后续更改只会影响以后的搜索请求。
为了使用scrolling，在查询字符串中应该初始化scroll参数，来告诉es要search context对象存活多久。例如：scroll=1m。
```
POST /_search?scroll=1m
{
    "size": 100,
    "query": {
        "match" : {
            "brand" : "gucci"
        }
    }
}
```

 
返回：


```
{
  "_scroll_id": "cXVlcnlUaGVuRmV0Y2g7NTs0ODp6bnRHQVprSVMwaWJ4bGxmcG9OVFNBOzUwOnpudEdBWmtJUzBpYnhsbGZwb05UU0E7NDY6em50R0Faa0lTMGlieGxsZnBvTlRTQTs0OTp6bnRHQVprSVMwaWJ4bGxmcG9OVFNBOzQ3OnpudEdBWmtJUzBpYnhsbGZwb05UU0E7MDs=",
  "took": 4,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0.30685282,
    "hits": [
      {
        "_index": "shirts",
        "_type": "item",
        "_id": "1",
        "_score": 0.30685282,
        "_source": {
          "brand": "gucci",
          "color": "red",
          "model": "slim"
        }
      }
    ]
  }
}
```

搜索结果会返回一个_scroll_id，可以用来传入scrollAPI来获得下一批数据。

```
POST  /_search/scroll 
{
    "scroll" : "1m", 
    "scroll_id" : "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAAD4WYm9laVYtZndUQlNsdDcwakFMNjU1QQ==" 
}
```
size参数可以用来配置每次命中数据的返回条数，每次请求scroll API都会返回下一批数据，直到为空。

### Preference
使用preference可以控制在哪个分片上执行搜索请求。默认情况下，会随机在分片的备份上执行。

参数：
- _primary: 只会在主分片上执行。
- _primary_first: 会优先在祝分片上执行，如果主分片不可用，则在其他分片上执行。
- _replica： 只会在副本分片上执行。
- _replica_first： 会优先在副本分片上执行，如果分片不可用，则在其他分片上执行。
- _local： 操作会尽可能在本地分片上执行。
- _prefer_nodes:abc,xyz： 会在指定的节点上执行。
- _shards:2,3： 限制操作在指定的分片上执行，这个参数可以和其他的preference参数一起执行，但是必须将它放在第一个：_shards:2,3|_primary。
- _only_nodes： 限制操作的节点的规格。
- Custom (string) value： 自定义的值将用来保证同一个分片会用来处理相同的自定义值。

```
GET /_search?preference=xyzabc123
{
    "query": {
        "match": {
            "title": "elasticsearch"
        }
    }
}
```

### Explain
开启explain可以知道所有的命中结果的score是怎么计算的。
```
GET /_search
{
    "explain": true,
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

### Version
每次命中的结果返回一个版本号。
```
GET /_search
{
    "version": true,
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

### min_score
排除_score低于设置的最小值的结果。
```
GET /_search
{
    "min_score": 0.5,
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```
### Named Queries
每个filter和query都可以在最上面设置一个 _name 参数。
```
GET /_search
{
    "query": {
        "bool" : {
            "should" : [
                {"match" : { "name.first" : {"query" : "shay", "_name" : "first"} }},
                {"match" : { "name.last" : {"query" : "banon", "_name" : "last"} }}
            ],
            "filter" : {
                "terms" : {
                    "name.last" : ["banon", "kimchy"],
                    "_name" : "test"
                }
            }
        }
    }
}
```
### Inner hits
parent/child 和 nested 功能可以允许返回匹配不同范围的文档。 在parent/child 中，parent文档可以基于child文档的匹配结果进行返回，或者child文档可以基于parent的匹配结果进行返回。在nested中，文档可以基于内嵌对象的匹配结果进行返回。

在所有情况下，实际的返回结果是由哪个文档引起的一般来说是隐藏的，但是，大部分情况下，知道哪个文档引起的匹配结果是很必要的。inner hits 这个功能就是做这个的。

结构是这样的：
```
"<query>" : {
    "inner_hits" : {
        <inner_hits_options>
    }
}
```

如果在查询中定义了inner_hits，那么在每次返回的信息中都包含inner_hits这个json块：

```
"hits": [
     {
        "_index": ...,
        "_type": ...,
        "_id": ...,
        "inner_hits": {
           "<inner_hits_name>": {
              "hits": {
                 "total": ...,
                 "hits": [
                    {
                       "_type": ...,
                       "_id": ...,
                       ...
                    },
                    ...
                 ]
              }
           }
        },
        ...
     },
     ...
]
```
- from、size、sort：用于分页和排序。
- name： 用于标识inner_hits的返回结果。当查询中有多个inner_hits时非常有用。

nested的查询：
```
{
    "query" : {
        "nested" : {
            "path" : "comments",
            "query" : {
                "match" : {"comments.message" : "[actual query]"}
            },
            "inner_hits" : {} 
        }
    }
}
```

响应结果：

```
...
"hits": {
  ...
  "hits": [
     {
        "_index": "my-index",
        "_type": "question",
        "_id": "1",
        "_source": ...,
        "inner_hits": {
           "comments": { 
              "hits": {
                 "total": ...,
                 "hits": [
                    {
                       "_nested": {
                          "field": "comments",
                          "offset": 2
                       },
                       "_source": ...
                    },
                    ...
                 ]
              }
           }
        }
     },
     ...
```

parent/child 查询：
```
{
    "query" : {
        "has_child" : {
            "type" : "comment",
            "query" : {
                "match" : {"message" : "[actual query]"}
            },
            "inner_hits" : {} 
        }
    }
}
```

响应结果：

```
...
"hits": {
  ...
  "hits": [
     {
        "_index": "my-index",
        "_type": "question",
        "_id": "1",
        "_source": ...,
        "inner_hits": {
           "comment": {
              "hits": {
                 "total": ...,
                 "hits": [
                    {
                       "_type": "comment",
                       "_id": "5",
                       "_source": ...
                    },
                    ...
                 ]
              }
           }
        }
     },
     ...
```
### Search After
分页功能一般可以使用from/size来实现，但是当深度分页时的成本也是很大的。index-max_result_window默认安全值是10000，搜索请求的堆内存占用率和时间成本和from+size成正比。Scroll API推荐用于深度分页，但是滚动的成本高，不推荐用于实时的请求。search_after参数通过提供一个实时的游标绕过了这个问题。这个想法是通过前一页的结果来帮助下一页的检索。

假设查询第一页是这样的：
```
GET twitter/tweet/_search
{
    "size": 10,
    "query": {
        "match" : {
            "title" : "elasticsearch"
        }
    },
    "sort": [
        {"date": "asc"},
        {"_uid": "desc"}
    ]
}
```
上面查询的结果包括了一组 sort values。这些sort values 能用于联合search_after参数来返回任何文档后面的结果。例如，我们可以使用最后一个文档的sort values 并把它传入search_after，就可以获取下一页的文档：
```
GET twitter/tweet/_search
{
    "size": 10,
    "query": {
        "match" : {
            "title" : "elasticsearch"
        }
    },
    "search_after": [1463538857, "tweet#654323"],
    "sort": [
        {"date": "asc"},
        {"_uid": "desc"}
    ]
}
```
值得注意的是，使用search_after需要将from设置为0或-1。

search_after对于想要随机跳转到某页是不可用的。它和scroll API非常相似，不同的是search_after是无状态的，它总是取得的是最新版本的数据。

## Search Template

```
GET /_search/template
{
    "inline" : {
      "query": { "match" : { "{{my_field}}" : "{{my_value}}" } },
      "size" : "{{my_size}}"
    },
    "params" : {
        "my_field" : "foo",
        "my_value" : "bar",
        "my_size" : 5
    }
}
```

### 参数为查询语句的例子

```
GET /_search/template
{
    "inline": {
        "query": {
            "match": {
                "title": "{{query_string}}"
            }
        }
    },
    "params": {
        "query_string": "search for these words"
    }
}
```

### 将参数转为json

<font color='red'><b>注意，本节中，由于蛋疼的编译器的限制，在下一行的大括号内加上了单引号，正确的语法是不加单引号的。</b></font>

`{{'#toJson'}}参数名{{'/toJson'}}`可以将 map 和 array 类型的参数转换为 json 格式。

```
GET /_search/template
{
  "inline": "{ \"query\": { \"terms\": { \"status\": {{#toJson}}status{{/toJson}} }}}",
  "params": {
    "status": [ "pending", "published" ]
  }
}
```
上面的例子实际执行时是这样的：

```
{
  "query": {
    "terms": {
      "status": [
        "pending",
        "published"
      ]
    }
  }
}
```

更复杂一些的例子是这样的：

```
{
    "inline": "{\"query\":{\"bool\":{\"must\": {{#toJson}}clauses{{/toJson}} }}}",
    "params": {
        "clauses": [
            { "term": "foo" },
            { "term": "bar" }
        ]
   }
}
```

实际执行时是这样的：

```
{
    "query" : {
      "bool" : {
        "must" : [
          {
            "term" : "foo"
          },
          {
            "term" : "bar"
          }
        ]
      }
    }
}
```

### 连接 array 的值
`{{'#join'}}array{{'/join'}}` 可以将array的值连接起来。

```
GET /_search/template
{
  "inline": {
    "query": {
      "match": {
        "emails": "{{#join}}emails{{/join}}"
      }
    }
  },
  "params": {
    "emails": [ "username@email.com", "lastname@email.com" ]
  }
}
```

这就相当于：

```
{
    "query" : {
        "match" : {
            "emails" : "username@email.com,lastname@email.com"
        }
    }
}
```

也可以自定义链接的符号，比如将， 转为 || ：

```
GET /_search/template
{
  "inline": {
    "query": {
      "range": {
        "born": {
            "gte"   : "{{date.min}}",
            "lte"   : "{{date.max}}",
            "format": "{{#join delimiter='||'}}date.formats{{/join delimiter='||'}}"
            }
      }
    }
  },
  "params": {
    "date": {
        "min": "2016",
        "max": "31/12/2017",
        "formats": ["dd/MM/yyyy", "yyyy"]
    }
  }
}
```

相当于：

```
{
    "query" : {
      "range" : {
        "born" : {
          "gte" : "2016",
          "lte" : "31/12/2017",
          "format" : "dd/MM/yyyy||yyyy"
        }
      }
    }
}
```


### 默认值


`{{'var'}}{{'^var'}}default{{'/var'}}` 可以设定参数的默认值。


```
{
  "inline": {
    "query": {
      "range": {
        "line_no": {
          "gte": "{{start}}",
          "lte": "{{end}}{{'^end'}}20{{'/end'}}"
        }
      }
    }
  },
  "params": { ... }
}
```

当params是{ "start": 10, "end": 15 }时，搜索语句是这样的：

```
{
    "range": {
        "line_no": {
            "gte": "10",
            "lte": "15"
        }
  }
}
```

当params 是 { "start": 10 } 时，则是这样的：

```
{
    "range": {
        "line_no": {
            "gte": "10",
            "lte": "20"
        }
    }
}
```
### 条件判断语句
条件判断语句不能使用json作为参数，只能使用string类型。

参数是：

```
{
    "params": {
        "text":      "words to search for",
        "line_no": { 
            "start": 10, 
            "end":   20  
        }
    }
}
```

查询语句是这样的：

```
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "line": "{{text}}" 
        }
      },
      "filter": {
        {{'#line_no'}} 
          "range": {
            "line_no": {
              {{'#start'}} 
                "gte": "{{start}}" 
                {{'#end'}},{{'/end'}} 
              {{'/start'}} 
              {{'#end'}} 
                "lte": "{{end}}" 
              {{'/end'}} 
            }
          }
        {{'/line_no'}} 
      }
    }
  }
}
```

`{{'#line_no'}} ... {{'/line_no'}}` 块表示的是 当指定了 line_no 这个参数时，才会有中间的代码，依此类推，`{{'#start'}} ... {{'/start'}}`等也是如此。

### 预注册模板
可以预先将查询模板存储在 config/scripts 目录下，并使用后缀为.mustache 的文件存储。

```
GET /_search/template
{
    "file": "storedTemplate", 
    "params": {
        "query_string": "search for these words"
    }
}
```

上面的例子中，文件是 config/scripts/ 目录下的storedTemplate.mustache 。

还可以将模板存储在 cluster state 中。然后使用restAPI来管理：

```
POST /_search/template/<templatename>
{
    "template": {
        "query": {
            "match": {
                "title": "{{query_string}}"
            }
        }
    }
}
```

然后获取的时候使用：


```
GET /_search/template/<templatename>
```

删除的时候使用：

```
DELETE /_search/template/<templatename>
```

## Multi Search Template

```
$ cat requests
{"index": "test"}
{"inline": {"query": {"match":  {"user" : "{{username}}" }}}, "params": {"username": "john"}} 
{"index": "_all", "types": "accounts"}
{"inline": {"query": {"{{query_type}}": {"name": "{{name}}" }}}, "params": {"query_type": "match_phrase_prefix", "name": "Smith"}}
{"index": "_all"}
{"id": "template_1", "params": {"query_string": "search for these words" }} 
{"types": "users"}
{"file": "template_2", "params": {"field_name": "fullname", "field_value": "john smith" }} 

$ curl -XGET localhost:9200/_msearch/template --data-binary "@requests"; echo
```

## Search Shards API
search shards API 会将执行搜索请求的分片信息返回回来。这对于查找问题和优化系统有很重要的意义。

```
GET /twitter/_search_shards
```

返回的信息是这样的：

```
{
  "nodes": ...,
  "indices" : {
    "twitter": { }
  },
  "shards": [
    [
      {
        "index": "twitter",
        "node": "JklnKbD7Tyqi9TP3_Q_tBg",
        "primary": true,
        "shard": 0,
        "state": "STARTED",
        "allocation_id": {"id":"0TvkCyF7TAmM1wHP4a42-A"},
        "relocating_node": null
      }
    ],
    [
      {
        "index": "twitter",
        "node": "JklnKbD7Tyqi9TP3_Q_tBg",
        "primary": true,
        "shard": 1,
        "state": "STARTED",
        "allocation_id": {"id":"fMju3hd1QHWmWrIgFnI4Ww"},
        "relocating_node": null
      }
    ],
    [
      {
        "index": "twitter",
        "node": "JklnKbD7Tyqi9TP3_Q_tBg",
        "primary": true,
        "shard": 2,
        "state": "STARTED",
        "allocation_id": {"id":"Nwl0wbMBTHCWjEEbGYGapg"},
        "relocating_node": null
      }
    ],
    [
      {
        "index": "twitter",
        "node": "JklnKbD7Tyqi9TP3_Q_tBg",
        "primary": true,
        "shard": 3,
        "state": "STARTED",
        "allocation_id": {"id":"bU_KLGJISbW0RejwnwDPKw"},
        "relocating_node": null
      }
    ],
    [
      {
        "index": "twitter",
        "node": "JklnKbD7Tyqi9TP3_Q_tBg",
        "primary": true,
        "shard": 4,
        "state": "STARTED",
        "allocation_id": {"id":"DMs7_giNSwmdqVukF7UydA"},
        "relocating_node": null
      }
    ]
  ]
}
```

还可以指定routing：

```
GET /twitter/_search_shards?routing=foo,baz
```

返回结果：

```
{
  "nodes": ...,
  "indices" : {
      "twitter": { }
  },
  "shards": [
    [
      {
        "index": "twitter",
        "node": "JklnKbD7Tyqi9TP3_Q_tBg",
        "primary": true,
        "shard": 0,
        "state": "STARTED",
        "allocation_id": {"id":"0TvkCyF7TAmM1wHP4a42-A"},
        "relocating_node": null
      }
    ],
    [
      {
        "index": "twitter",
        "node": "JklnKbD7Tyqi9TP3_Q_tBg",
        "primary": true,
        "shard": 1,
        "state": "STARTED",
        "allocation_id": {"id":"fMju3hd1QHWmWrIgFnI4Ww"},
        "relocating_node": null
      }
    ]
  ]
}
```

由于指定了routing，搜索请求将在这两个分片中执行。

- routing: 由逗号分隔的，指定搜索在哪些分片上执行。
- preference： 指定搜索在分片的哪个备份上执行。默认是在随机的备份上执行。
- local： 布尔型，是否读取本地集群状态以确定碎片分配而不是使用主节点的集群状态。

## Suggesters
suggest功能是通过使用suggester基于所提供的文本来建议类似的术语。它可以在_search请求或_suggest中执行。

```
POST twitter/_search
{
  "query" : {
    "match": {
      "message": "tring out Elasticsearch"
    }
  },
  "suggest" : {
    "my-suggestion" : {
      "text" : "trying out Elasticsearch",
      "term" : {
        "field" : "message"
      }
    }
  }
}
```

suggest请求在_suggest上执行需要省略suggest元素，suggest元素仅在suggest是搜索请求一部分时才可用。

```
POST _suggest
{
  "my-suggestion" : {
    "text" : "tring out Elasticsearch",
    "term" : {
      "field" : "message"
    }
  }
}
```

一个请求可以设置多个suggest。每个suggest可以设置不同的名字。
```
POST _suggest
{
  "my-suggest-1" : {
    "text" : "tring out Elasticsearch",
    "term" : {
      "field" : "message"
    }
  },
  "my-suggest-2" : {
    "text" : "kmichy",
    "term" : {
      "field" : "user"
    }
  }
}
```

返回结果：

```
{
  "_shards": ...
  "my-suggest-1": [ {
    "text": "tring",
    "offset": 0,
    "length": 5,
    "options": [ {"text": "trying", "score": 0.8, "freq": 1 } ]
  }, {
    "text": "out",
    "offset": 6,
    "length": 3,
    "options": []
  }, {
    "text": "elasticsearch",
    "offset": 10,
    "length": 13,
    "options": []
  } ],
  "my-suggest-2": ...
}
```

为了避免重复的text，可以定义一个全局的text。

```
POST _suggest
{
  "text" : "tring out Elasticsearch",
  "my-suggest-1" : {
    "term" : {
      "field" : "message"
    }
  },
  "my-suggest-2" : {
    "term" : {
      "field" : "user"
    }
  }
}
```

### Term suggester
//TODO ...


## Multi Search API
multi search API 允许通过一个API执行多个搜索请求。使用_msearch。

请求格式和bulk API类似，结构类似于下面：

```
header\n
body\n
header\n
body\n
```

header部分包含了搜索哪些index，search_type，preference和routing是可选的。请求体是典型的搜索请求体：
```
$ cat requests
{"index" : "test"}
{"query" : {"match_all" : {}}, "from" : 0, "size" : 10}
{"index" : "test", "search_type" : "dfs_query_then_fetch"}
{"query" : {"match_all" : {}}}
{}
{"query" : {"match_all" : {}}}

{"query" : {"match_all" : {}}}
{"search_type" : "dfs_query_then_fetch"}
{"query" : {"match_all" : {}}}

$ curl -XGET localhost:9200/_msearch --data-binary "@requests"; echo
```

看一下上面的例子，有两个地方是没有指定header或者是{}空header的，这也是支持的。

响应会返回一个responses数组，包含每一个搜索的响应和状态码，并且按照请求的顺序排列。

也可以不在header中，而在URI上设置index和type：

```
$ cat requests
{}
{"query" : {"match_all" : {}}, "from" : 0, "size" : 10}
{}
{"query" : {"match_all" : {}}}
{"index" : "test2"}
{"query" : {"match_all" : {}}}

$ curl -XGET localhost:9200/test/_msearch --data-binary @requests; echo
```

max_concurrent_searches 参数可以用在请求中，用来控制搜索的最大搜索数。默认的取决于搜索节点的线程池中线程数量。

## Count API
count API 可以执行搜索请求并且获得符合条件的结果数量。它能够跨多个index和type上执行。可以使用一个简单的查询字符串作为参数或者在request body中使用QueryDSL。

```
PUT /twitter/tweet/1?refresh
{
    "user": "kimchy"
}

GET /twitter/tweet/_count?q=user:kimchy

GET /twitter/tweet/_count
{
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

结果为：

```
{
    "count" : 1,
    "_shards" : {
        "total" : 5,
        "successful" : 5,
        "failed" : 0
    }
}
```

### Request Parameters
当使用参数q执行count时，可以传入以下参数：

- df:当没传入搜索的field时，默认使用的field。
- analyzer： 使用的分析器的名称。
- default_operator： 默认的操作符，可以使用AND或OR，默认是OR。
- lenient：设为true时会导致类型失败会被忽略（例如对text类型传入数字类型）。默认是false。
- analyze_wildcard：通配符和前缀查询是否会被分析器分析。默认是false。
- terminate_after：count到达某个值时则停止查询。如果设置了，那么在响应结果中会有一个terminated_early的boolean型来表示是否执行了terminate_after。默认是没有terminate_after的。

## Validate API

validate API 可以在不执行请求的情况下校验一个潜在的耗时的查询。我们可以使用下面的数据来验证一下：

```
PUT twitter/tweet/_bulk?refresh
{"index":{"_id":1}}
{"user" : "kimchy", "post_date" : "2009-11-15T14:12:12", "message" : "trying out Elasticsearch"}
{"index":{"_id":2}}
{"user" : "kimchi", "post_date" : "2009-11-15T14:12:13", "message" : "My username is similar to @kimchy!"}
```

接下来发送校验请求:

```
GET twitter/_validate/query?q=user:foo
```

响应结果是这样的：

```
{"valid":true,"_shards":{"total":1,"successful":1,"failed":0}}
```

### 参数

- df:当没传入搜索的field时，默认使用的field。
- analyzer： 使用的分析器的名称。
- default_operator： 默认的操作符，可以使用AND或OR，默认是OR。
- lenient：设为true时会导致类型失败会被忽略（例如对text类型传入数字类型）。默认是false。
- analyze_wildcard：通配符和前缀查询是否会被分析器分析。默认是false。

```
GET twitter/tweet/_validate/query
{
  "query" : {
    "bool" : {
      "must" : {
        "query_string" : {
          "query" : "*:*"
        }
      },
      "filter" : {
        "term" : { "user" : "kimchy" }
      }
    }
  }
}
```

如果查询是无效的，那么valid会为false。例如下面的：

```
GET twitter/tweet/_validate/query?q=post_date:foo
```

由于post_date是日期类型，而查询时传入的是 foo 字符串类型，所以结果为：

```
{"valid":false,"_shards":{"total":1,"successful":1,"failed":0}}
```

将explain设置为true将会返回为什么查询是非法的信息：

```
GET twitter/tweet/_validate/query?q=post_date:foo&explain=true
```

响应是：

```
{
  "valid" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "explanations" : [ {
    "index" : "twitter",
    "valid" : false,
    "error" : "twitter/IAEc2nIXSSunQA_suI0MLw] QueryShardException[failed to create query:...failed to parse date field [foo]"
  } ]
}
```

当查询是有效的，explanation默认为该查询的字符串表示。当rewrite设置为true时，explanation会显示查询的更多细节。

对于模糊查询：

```
GET twitter/tweet/_validate/query?rewrite=true
{
  "query": {
    "match": {
      "user": {
        "query": "kimchy",
        "fuzziness": "auto"
      }
    }
  }
}
```

响应：

```
{
   "valid": true,
   "_shards": {
      "total": 1,
      "successful": 1,
      "failed": 0
   },
   "explanations": [
      {
         "index": "twitter",
         "valid": true,
         "explanation": "+user:kimchy +user:kimchi^0.75 #(ConstantScore(_type:tweet))^0.0"
      }
   ]
}
```

更多的例子，像这样：

```
GET twitter/tweet/_validate/query?rewrite=true
{
  "query": {
    "more_like_this": {
      "like": {
        "_id": "2"
      },
      "boost_terms": 1
    }
  }
}
```

响应：

```
{
   "valid": true,
   "_shards": {
      "total": 1,
      "successful": 1,
      "failed": 0
   },
   "explanations": [
      {
         "index": "twitter",
         "valid": true,
         "explanation": "((user:terminator^3.71334 plot:future^2.763601 plot:human^2.8415773 plot:sarah^3.4193945 plot:kyle^3.8244398 plot:cyborg^3.9177752 plot:connor^4.040236 plot:reese^4.7133346 ... )~6) -ConstantScore(_uid:tweet#2)) #(ConstantScore(_type:tweet))^0.0"
      }
   ]
}
```

请求会执行在一个随机的分片上，因此explanation的细节取决于命中的分片，因此，同一个查询在不同的分片可能在细节上差异巨大。


## Explain API
explain API 可以计算出查询指定文档的分数。这个功能非常有用，它可以让我们直接知道这个查询是否和指定的文档匹配。

注意，此处index和type只能是唯一的，不能是多个。

```
GET /twitter/tweet/0/_explain
{
      "query" : {
        "match" : { "message" : "elasticsearch" }
      }
}
```

查询结果为：

```
{
   "_index": "twitter",
   "_type": "tweet",
   "_id": "0",
   "matched": true,
   "explanation": {
      "value": 1.55077,
      "description": "sum of:",
      "details": [
         {
            "value": 1.55077,
            "description": "weight(message:elasticsearch in 0) [PerFieldSimilarity], result of:",
            "details": [
               {
                  "value": 1.55077,
                  "description": "score(doc=0,freq=1.0 = termFreq=1.0\n), product of:",
                  "details": [
                     {
                        "value": 1.3862944,
                        "description": "idf, computed as log(1 + (docCount - docFreq + 0.5) / (docFreq + 0.5)) from:",
                        "details": [
                           {
                              "value": 1.0,
                              "description": "docFreq",
                              "details": []
                           },
                           {
                              "value": 5.0,
                              "description": "docCount",
                              "details": []
                           }
                        ]
                     },
                     {
                        "value": 1.1186441,
                        "description": "tfNorm, computed as (freq * (k1 + 1)) / (freq + k1 * (1 - b + b * fieldLength / avgFieldLength)) from:",
                        "details": [
                           {
                              "value": 1.0,
                              "description": "termFreq=1.0",
                              "details": []
                           },
                           {
                              "value": 1.2,
                              "description": "parameter k1",
                              "details": []
                           },
                           {
                              "value": 0.75,
                              "description": "parameter b",
                              "details": []
                           },
                           {
                              "value": 5.4,
                              "description": "avgFieldLength",
                              "details": []
                           },
                           {
                              "value": 4.0,
                              "description": "fieldLength",
                              "details": []
                           }
                        ]
                     }
                  ]
               }
            ]
         },
         {
            "value": 0.0,
            "description": "match on required clause, product of:",
            "details": [
               {
                  "value": 0.0,
                  "description": "# clause",
                  "details": []
               },
               {
                  "value": 1.0,
                  "description": "*:*, product of:",
                  "details": [
                     {
                        "value": 1.0,
                        "description": "boost",
                        "details": []
                     },
                     {
                        "value": 1.0,
                        "description": "queryNorm",
                        "details": []
                     }
                  ]
               }
            ]
         }
      ]
   }
}
```

也可以使用q参数来进行查询：

```
GET /twitter/tweet/0/_explain?q=message:search
```

- source: 设为true可以返回查询到的_source文档。还可以通过_source_include和_source_exclude来决定查询哪些文档。

- stored_fields： 允许控制哪些存储的字段会作为explained的文档。

- routing： 控制使用哪个路由。

- parent：和routing使用方式一样。

- preference： 控制在哪个分片上执行explain。

- source： 可以在请求体进行查询。

- q： 查询字符串。

- df： 默认的用来进行查询的field。默认为_all。

- analyzer： 查询分析器的名称。

- analyze_wildcard： 是否可以使用通配符或前缀进行查询，默认是false。

- lenient： 是否忽略类型匹配异常。默认是false。

- default_operator： 默认的逻辑操作符，可以是AND或OR，默认是OR。







  










