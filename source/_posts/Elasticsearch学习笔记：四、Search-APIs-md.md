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
```
GET /_search
{
    "stored_fields": "_none_",
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
``` 
### Script Fields
```
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
```

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





