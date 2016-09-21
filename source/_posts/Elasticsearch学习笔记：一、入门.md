---
title: Elasticsearch学习笔记：一、入门
date: 2016-09-21 11:59:46
tags: Elasticsearch
---
## 简介
Elasticsearch是一个分布式的开源的全文分析搜索引擎，它基于Lucene，提供一个接近实时的搜索。

## 基本概念
### Near Realtine(NRT)
Es是一个接近实时的搜索平台。这意味着，当你index一个文档到能搜索到它，需要花费一些很少的时间（通常是一秒）。

### Cluster
cluster是一个由一个或多个服务节点组成的集合，它包含了你所有的数据并提供了对所有节点的索引和搜索。一个cluster会被一个名称唯一标识，默认名称是“elasticsearch”。这个名字非常重要，因为节点会通过集群的名字来确定加入的集群。

### Node
一个节点是你cluster中一个独立的服务，存储数据，组成cluster的index，提供搜索的能力。在一个cluster中，它的节点会被节点名称唯一标识，默认是一个随机的Marvel人物的名称。节点名称对于管理节点是非常重要的。
一个节点可以cluster名称来决定参加哪个cluster。默认情况下，在你网络中又多个节点，并且它们可以互相连通，那么它们会自动组成并加入一个名字为"elasticsearch"的cluster。
在一个cluster中，如果只有一个节点，那么它会自动形成一个名叫“elasticsearch”的单节点cluster。

### Index
index是有相同特征的文档的集合。例如，你可以有一个用户数据的index，或者产品目录的index。一个index被一个名称表示，名称必须是小写字母，这个名称可以用来对文档执行索引、创建，搜索，更新和删除等操作。

### Type
在index中，你可以定义一个或多个type。type是一个逻辑类别还是index的一部分，这些都取决于你的定义。一般来说，一个type是具有相同field的文档。例如，我们假设你运行一个blog平台，并且存储所有的数据在一个单独的index中。在这个index中，你可能会定义一个type为user数据，另一个type为blog数据，还有其它type为评论数据等等。

### Document
document是被索引的基本数据单位。例如，你可能有一个文档是针对单个用户，另一个文档是某个商品等。document用JSON来存储。
在index/type类型中，你可以存储很多document。

### Shards & Replicas
一个index能够存储非常大量的数据，有时可能会超过单个节点硬件的限制。例如。一个index存储一超过1TB的数据，这对于单个节点有可能是负担不起的。为了解决这个问题，es可以讲index分片到不同的shards中。当你创建一个index，你可以定义你想要的shards数量。每一个shard对于自己来说都是一个拥有所有功能并且有独立的index，可以被放到任何node上。
分片有两个重要的优势：
- 它允许你水平 分割/扩展 你的内容。
- 它允许你并行处理不同分片上的数据来提高效率。 为了保证高可用性，可以为每个分片节点设置备份节点。

对于如何进行分片以及进行聚合操作时文档是怎样merge的，es都自动管理了，并且对用户是透明的。

在网络环境中，当某个服务失效了，failover机制是非常有效的高可用保障。es也提供了对于index分片的复制。
复制机制有两个重要特点：
- 它提供了高可用性以防一个shard/node失效。值得注意的是，一个shard复制不要和它本身放在同一个节点。
- 它通过对所有复制shard的并行搜索来提高你系统的吞吐量。

## 安装
es运行环境需要jdk1.7及以上，下载jdk直接去oracle官网下。然后[下载es ](http://www.elastic.co/downloads )。
解压后，启动：
```
$ elasticsearch-2.4.0/bin/elasticsearch
```

之前提到过，我们可以覆盖cluster和node名称：
```
./elasticsearch --cluster.name my_cluster_name --node.name my_node_name
```

## 探索集群和Index
现在我们已经让节点和集群运行起来了，下一步怎么和es进行沟通交流呢？幸运的是，es提供了一个非常好用的restAPI。通过api我们可以做这些事情：
- 检查集群，节点和index的健康状况，状态和统计数据。
- 管理集群、节点和index的数据和元数据。
- 执行CRUD和一些针对index的搜索操作。
- 执行高级的搜索操作，例如分页，排序，过滤，脚本，聚合以及其他。

### 集群的健康监测
接下来，我们对集群进行一个基本的简单的健康监测。前文提到了，因为es通过restAPI来进行操作，所以可以使用curl或者postman等。
检测cluster的健康状况，我们可以使用_cat API。
```
curl 'localhost:9200/_cat/health?v'
```
返回的结果是这样的：
```
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign
1394735289 14:28:09  elasticsearch green           1         1      0   0    0    0        0
```
其中status显示了当前的健康状况。
status的定义如下： 
- green ： everything is ok。 
- yellow： 所有数据可用，但是一些备份节点的数据尚未被分配。 
- red ： 一些数据不可用。

还可以查看节点的状况：
```
curl 'localhost:9200/_cat/nodes?v'
```
返回结果：
```
host         ip        heap.percent ram.percent load node.role master name
mwubuntu1    127.0.1.1            8           4 0.00 d         *      New Goblin
```
### 查询所有的index
```
curl 'localhost:9200/_cat/indices?v'
```
返回为：
```
health status index   pri rep docs.count docs.deleted store.size pri.store.size 
yellow open   secilog   5   1          1            0      4.2kb          4.2kb 
green  open   twitter   1   0          2            0      7.2kb          7.2kb 
```
上面显示了，我有两个index，一个secilog，还有一个twitter，当然，这些都是我后来自己建的，新安装的es是没有的，它会返回：
```
health index pri rep docs.count docs.deleted store.size pri.store.size
```

### 创建一个Index
```
curl -XPUT 'localhost:9200/customer?pretty'
```
返回：
```json
{
  "acknowledged": true
}
```
查看索引：
```
curl 'localhost:9200/_cat/indices?v'
health index    pri rep docs.count docs.deleted store.size pri.store.size
yellow customer   5   1          0            0       495b           495b
```
这里status是yellow，是因为默认情况下，es默认会为每一个shards创建一个replica，但是由于目前我们只是一个单节点的，所以replica没有地方进行分配，所以就是yellow了。

### 索引并查询一个文档
下面我们放一些数据到刚才定义的index中。还记得之前提到过的type么，我们必须为文档指定type。

```
curl -XPUT 'localhost:9200/customer/external/1?pretty' -d '
{
  "name": "John Doe"
}'
```
返回结果：
```
{
  "_index": "customer",
  "_type": "external",
  "_id": "1",
  "_version": 1,
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "created": true
}
```
上面显示创建成功了。不过，值得注意的是，如果上面的customer之前不存在，那么es会自动创建一个名为customer的index。

然后我们就可以搜索到了：
```
curl -XGET 'localhost:9200/customer/external/1?pretty'
```
返回结果：
```
{
  "_index" : "customer",
  "_type" : "external",
  "_id" : "1",
  "_version" : 1,
  "found" : true,
  "_source" : {
    "name" : "John Doe"
  }
}
```

### 删除一个Index
现在我们可以删除刚刚创建的index：
```
curl -XDELETE 'localhost:9200/customer?pretty'
```
返回：
```
{
  "acknowledged": true
}
```

根据上面的操作，不难看出，es针对index的操作一般遵循这个格式：
```
curl -X<REST Verb> <Node>:<Port>/<Index>/<Type>/<ID>
```

## 修改数据
es提供对于数据的修改以及接近实时的搜索功能。在你index/update/delete数据后，到你可以搜索到这些改变，一般需要1秒钟的延迟。这对于其他平台比如SQL来说，是一个非常大的区别。

### 创建/替换索引文档
之前我们已经看到了如何对一个文档进行索引，让我们回顾一下：
```
curl -XPUT 'localhost:9200/customer/external/1?pretty' -d '
{
  "name": "John Doe"
}'
```
接着，上面的操作会为customer创建一个文档，type是external，id是1.如果我们执行同样的操作但是使用不同的数据，es会覆盖之前已经存在的数据：
```
curl -XPUT 'localhost:9200/customer/external/1?pretty' -d '
{
  "name": "Jane Doe"
}'
```

如果我们不指定ID，es会为自动生成一个id并用它来索引文档。
```
curl -XPOST 'localhost:9200/customer/external?pretty' -d '
{
  "name": "Jane Doe"
}'
```
返回：
```
{
  "_index": "customer",
  "_type": "external",
  "_id": "AVdLq7nyMju-ZsxkZnGp",
  "_version": 1,
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "created": true
}
```
<font color='red'>**值得注意的是，在不指定ID时，我们应该使用的是POST请求。**</font>

### 更新文档
为了创建/替换索引文档，我们还可以更新文档的内容。但是，es并不进行就地的更新。当我们执行一个update，es会删除旧的文档然后在桶中索引一个新的文档。

接下来更新之前的id=1的文档：
```
curl -XPOST 'localhost:9200/customer/external/1/_update?pretty' -d '
{
  "doc": { "name": "Jane Doe", "age": 20 }
}'
```

也可以使用script来进行更新：
```
curl -XPOST 'localhost:9200/customer/external/1/_update?pretty' -d '
{
  "script" : "ctx._source.age += 5"
}'
```

有时可能会返回：
```
{
  "error": {
    "root_cause": [
      {
        "type": "remote_transport_exception",
        "reason": "[Witchfire][127.0.0.1:9300][indices:data/write/update[s]]"
      }
    ],
    "type": "illegal_argument_exception",
    "reason": "failed to execute script",
    "caused_by": {
      "type": "script_exception",
      "reason": "scripts of type [inline], operation [update] and lang [groovy] are disabled"
    }
  },
  "status": 400
}
```
这是由于es基于安全考虑，将脚本功能禁用了，可以在config/elasticsearch.yml文件添加以下代码：
```
script.inline: on
script.indexed: on
script.file: on
```
配置后，重启Elasticsearch。

再执行上面的操作，就可以了。

### 删除文档
```
curl -XDELETE 'localhost:9200/customer/external/2?pretty'
```

### 批量处理
es对于index/uodate/delete等操作也可以进行批量处理。
```
curl -XPOST 'localhost:9200/customer/external/_bulk?pretty' -d '
{"index":{"_id":"1"}}
{"name": "John Doe" }
{"index":{"_id":"2"}}
{"name": "Jane Doe" }
'
```
也可以：
```
curl -XPOST 'localhost:9200/customer/external/_bulk?pretty' -d '
{"update":{"_id":"1"}}
{"doc": { "name": "John Doe becomes Jane Doe" } }
{"delete":{"_id":"2"}}
'
```
bulkAPI 会按顺序执行请求。如果某个请求失败了，不管是什么原因，它都会继续执行接下来的请求。当返回结果时，它会告诉你哪些成功哪些失败。

## 查询数据
### Search API
下面我们来进行一些简单的查询操作。es提供了两种基本的查询方式：
- REST request URI
- REST request body

顾名思义，request body就是在请求体中添加参数，而URI则是在 URI中写明参数。
下面，用URI方式进行查询请求：
```
curl 'localhost:9200/customer/_search?q=*&pretty'
```
来分析一下查询请求，我们使用了_search 在customer index中，q=*说明了es要查询index中所有的文档。preety参数说明返回结果打印的时候打印一个规范的json格式。

返回结果为：
```
{
  "took": 109,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 1,
    "hits": [
      {
        "_index": "customer",
        "_type": "external",
        "_id": "2",
        "_score": 1,
        "_source": {
          "name": "Jane Doe"
        }
      },
      {
        "_index": "customer",
        "_type": "external",
        "_id": "AVdLq7nyMju-ZsxkZnGp",
        "_score": 1,
        "_source": {
          "name": "Jane Doe"
        }
      },
      {
        "_index": "customer",
        "_type": "external",
        "_id": "1",
        "_score": 1,
        "_source": {
          "name": "John Doe becomes Jane Doe"
        }
      }
    ]
  }
}
```
针对返回结果，我们可以看到这些部分：
- took - es用于执行查询所用的时间，单位为毫秒。
- time_out - 告诉我们查询是否超时了。
- _shards - 告诉我们以供查询了多少shards，以及它们成功与否。
- hits - 查询的命中结果。
- hits.total - 符合我们查询规则的文档总数。
- hits.hits - 查询结果数组，默认为前10个文档。
- _score和max_score - 先忽略它们。

使用request body查询方式来达到相同的效果：
```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "query": { "match_all": {} }
}'
```

非常重要的一点，一旦你的搜索结果返回了，那么，es不会保留任何的服务端资源或者在你结果集上的游标。这和其他的平台形成了鲜明的对比，比如SQL，你可能在一开始只是获取到你查询结果的子集，之后还需要向服务器使用某些服务端有状态的游标来抓取剩下的结果集。

### 执行查询
首先，我们来看一下返回的文档字段。默认情况下，会返回文档的所有字段，我们可以指定返回的字段：
```
curl -XPOST 'localhost:9200/customer/_search?pretty' -d '
{
  "query": { "match_all": {} },
  "_source": ["name"]
}'
```

接下来，我们看一下query部分，之前，我们都是使用的match_all，会返回所有的文档。下面我们加一些限制条件，查询name包含Jane 或者 Doe的文档：
```
curl -XPOST 'localhost:9200/customer/_search?pretty' -d'
{
  "query": { "match": {"name" : "Jane Doe"} },
  "_source": ["name"]
}'
```

查询name同时包含“Jane Doe”短语的文档：
```
curl -XPOST 'localhost:9200/customer/_search?pretty' -d'
{
  "query": { "match_phrase": {"name" : "Jane Doe"} },
  "_source": ["name"]
}'
```

下面是布尔类型的query，表示and：
```
curl -XPOST 'localhost:9200/customer/_search?pretty' -d '
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "Jane" } },
        { "match": { "name": "Doe" } }
      ]
    }
  }
}'
```
表示OR：

```
curl -XPOST 'localhost:9200/customer/_search?pretty' -d '
{
  "query": {
    "bool": {
      "should": [
        { "match": { "name": "Jane" } },
        { "match": { "name": "Doe" } }
      ]
    }
  }
}'
```
表示都为false:
```
curl -XPOST 'localhost:9200/customer/_search?pretty' -d '
{
  "query": {
    "bool": {
      "must_not": [
        { "match": { "name": "Jane" } },
        { "match": { "name": "Doe" } }
      ]
    }
  }
}'
```

也可以自由组合：
```
curl -XPOST 'localhost:9200/customer/_search?pretty' -d '
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "Jane" } }
      ],
	  "must_not": [
        { "match": { "name": "Doe" } }
      ]
    }
  }
}'
```

### 使用过滤器
之前提到的score是一个数字，它用来评价当前返回的文档和我们需要的文档的匹配程度。分数越高的匹配程度越好。
但是查询并不是总需要产生score，尤其是它们只是用来过滤文档的时候。es能够自动的优化查询并不计算这些无用的score。
上文中的bool query也支持filter语句。filter语句可以在不改变score的情况下使用查询语句来限定文档。
```
curl -XPOST 'localhost:9200/customer/_search?pretty' -d '
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "age": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}'
```
由此可知，有的时候我们不需要关注score，那么久可以用filter来进行查询过滤，因为filter不会去计算score，那么它的效率相应会更高一些。

### 执行聚合
聚合可以对数据进行分组并对分组进行数据统计。类似于SQL中的group by。在es中，可以一次性返回查询结果和局和结果。

```
curl -XPOST 'localhost:9200/customer/_search?pretty' -d '
{
  "size": 0,
  "aggs": {
    "group_by_name": {
      "terms": {
        "field": "name"
      }
    }
  }
}'
```
返回结果为：
```
{
  "took": 159,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_state": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "doe",
          "doc_count": 3
        },
        {
          "key": "jane",
          "doc_count": 3
        },
        {
          "key": "becomes",
          "doc_count": 1
        },
        {
          "key": "john",
          "doc_count": 1
        }
      ]
    }
  }
}
```
比如按官网上的例子，统计不同银行账户下的平均余额：
```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state",
        "order": {
          "average_balance": "desc"
        }
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}'
```
或者对聚合进行嵌套，先按年龄范围分组，再统计不同性别的账户平均余额：
```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "size": 0,
  "aggs": {
    "group_by_age": {
      "range": {
        "field": "age",
        "ranges": [
          {
            "from": 20,
            "to": 30
          },
          {
            "from": 30,
            "to": 40
          },
          {
            "from": 40,
            "to": 50
          }
        ]
      },
      "aggs": {
        "group_by_gender": {
          "terms": {
            "field": "gender"
          },
          "aggs": {
            "average_balance": {
              "avg": {
                "field": "balance"
              }
            }
          }
        }
      }
    }
  }
}'
```