---
title: Elasticsearch学习笔记：三、Document APIs
date: 2016-09-22 14:02:30
tags: Elasticsearch 
---
## Index API
index API 用来向指定的index添加或更新JSON文档，并使它可被搜索。下面的例子是在名为 twitter 的index中添加一个文档，type为tweet，id=1 ：
```
$ curl -XPUT 'http://localhost:9200/twitter/tweet/1' -d '{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}'
```
返回结果为:
```
{
  "_index": "twitter",
  "_type": "tweet",
  "_id": "1",
  "_version": 3,
  "_shards": {
    "total": 1,
    "successful": 1,
    "failed": 0
  },
  "created": false
}
```
其中，_shards段中提供了以下信息：
- total - 多少个分片及其备份要被操作。
- successful - 说明分片及其备份成功了几个。
- failures - 一个包含了失败分片及失败原因的数组。

### 自动创建Index
如果实现没创建index，那么在上面的操作中会自动创建相应的index，如果没创建type，也会为指定的type自动创建一个动态type映射。

这个映射非常灵活，并且模式自由。新的字段和对象会自动的被添加到指定类型的映射定义中去。

自动创建index可以被关闭，在节点配置文件中将 action.auto_create_index 设为 false 就可以了。自动创建type映射也可以关闭，将 index.mapper.dynamic 设为 false就可以了。

自动创建index也可以设置黑白名单，例如，设置 action.auto_create_index 为 +aaa\*,-bbb\*,+ccc\*,-\* ， 名字为 aaa\*的是可以创建的，为 bbb\*的是被禁止的，以此类推。

### 版本号
每个被index的文档都会有一个版本号。version 字段会在查询时返回给客户端。如果指定了版本号，可使用它做一个乐观锁。可以使用版本号进行一个read-then-update的事务处理。在读的时候指定文档的版本号可以确保期间文档不会发生变化。如果是读而不是更新操作，可以设置为preference 来替代 _primary。
```
curl -XPUT 'localhost:9200/twitter/tweet/1?version=2' -d '{
    "message" : "elasticsearch now has versioning support, double cool!"
}'
```
注意：版本操作是实时的，而且不被搜索操作的接近实时所影响。如果没设置版本号，接下来的操作将不会检查版本号。

默认情况下，内部的版本号会从1开始，然后随着每次更新增加。版本号也可以由外部提供。如果要开启外部版本号，需要将version_type设置为external。当设置为版本号为外部版本号时，系统不会检查版本号是否相等，而是当传入的版本号是否大于当前版本号，如果大于，则更新，如果等于或小于，则更新失败。

#### 版本号类型
除了上面所说的内部和外部版本号类型，es还提供了一些特殊的版本号类型：
- internal
  只有在传入的版本号和系统中当前版本号相等时才更新索引。
- external 或 external_gt
  只有在传入的版本号大于系统当前版本号 或者 要更新的文档不存在的时候，才会更新索引。传入的版本号将作为新的版本号，并且会存储新的文档。版本号必须为一个非负的long型。
- external_gte
  和上面唯一不同的是，如果传入的版本号等于当前版本号也可以更新索引。
- force
  无视版本号或者文档是否存在，都直接更新。且传入的版本号作为新的版本号，并会存储新的文档。一般用于改正错误数据。

### 操作类型
index操作也允许传入 on_type 参数来强制进行 create 操作。当设置为create，如果文档已经存在，index操作将会失败：
```
$ curl -XPUT 'http://localhost:9200/twitter/tweet/1?op_type=create' -d '{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}'
```
如果失败，会返回：
```
{
  "error": {
    "root_cause": [
      {
        "type": "document_already_exists_exception",
        "reason": "[tweet][1]: document already exists",
        "index": "twitter",
        "shard": "0"
      }
    ],
    "type": "document_already_exists_exception",
    "reason": "[tweet][1]: document already exists",
    "index": "twitter",
    "shard": "0"
  },
  "status": 409
}
```
另一种指定 create 的方式是：
```
$ curl -XPUT 'http://localhost:9200/twitter/tweet/1/_create' -d '{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}'
```
### 自动生成ID
index的操作可以不指定id。id可以自动生成。op_type会自动设置为create。注意，此处用的是POST而不是PUT：
```
$ curl -XPOST 'http://localhost:9200/twitter/tweet/' -d '{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}'
```

返回：
```
{
  "_index": "twitter",
  "_type": "tweet",
  "_id": "AVdQyYB5e-yIFmSuoS-f",
  "_version": 1,
  "_shards": {
    "total": 1,
    "successful": 1,
    "failed": 0
  },
  "created": true
}
```
### Routing
一般情况下，shard的位置是由id的hash来控制的。为了更精确地控制，可以直接在每次操作上指定用于计算hash的值。使用 routing参数：
```
$ curl -XPOST 'http://localhost:9200/twitter/tweet?routing=kimchy' -d '{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}'
```
例如上面，就是使用kimchy来计算分片hash，来决定这个文档存放在哪个shard中。

当显示设置了_routing字段，并设置为required，如果index操作中没传递这个参数，将会操作失败。

### Parent & Children
可以在index操作时指定parent：
```
$ curl -XPUT localhost:9200/blogs/blog_tag/1122?parent=1111 -d '{
    "tag" : "something"
}'
```
当index一个child文档，routing的值会自动设置为何它parent的一致，除非显式指定child的routing。

### 写一致性
默认情况下，对index的操作只有在超过半数的分片可用的情况下（quorum）才能成功返回。这个默认值可以通过 action.write_consistency参数来设置。如果想在每次操作时指定，可以使用请求参数 consistency 。参数有效值为one ， quorum和all。 index操作只有在复制组里面所有可用的分片都执行后才返回。

### 刷新
想要在index操作后立即就能够查询到结果，需要设置refresh参数为true。将此选项设置为true应该在经过仔细思考和验证后,从索引和搜索的角度来看，它不会影响性能。注意，使用getAPI是实时的，不需要刷新。

### Noop Updates
等待更新？。。。这个词不知道怎么翻译比较易懂。。。
当使用新的版本号进行index更新时，不管文档内容有没有变，都会创建一个新的文档。如果不能接受这样的操作，可以在使用_update api时将detect_noop设为true。这个选项不能用于index api因为index api不会获取旧的文档，因此，也就不会拿旧的和新的作比较。

### Timeout
当进行index操作时，被安排用于index操作的shard可能暂时不可用。一些因素导致这个情形，比如正在搬迁或者网络异常。默认的，index操作将会等待shard一分钟，如果还不可用就会返回失败。timeout 参数可以用来显式设置等待多久：
```
$ curl -XPUT 'http://localhost:9200/twitter/tweet/1?timeout=5m' -d '{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}'
```
上例中，设置等待5分钟。

## Get API
get API 允许通过id来从index中查找相应的JSON文档：
```
curl -XGET 'http://localhost:9200/twitter/tweet/1'
```
返回结果为：
```
{
  "_index": "twitter",
  "_type": "tweet",
  "_id": "1",
  "_version": 5,
  "found": true,
  "_source": {
    "user": "kimchy",
    "postDate": "2009-11-15T14:12:12",
    "message": "trying out Elasticsearch"
  }
}
```

可以通过HEAD来查询文档是否存在：
```
curl -XHEAD -i 'http://localhost:9200/twitter/tweet/1'
```
返回结果可以通过 http status来判断，200 -- 存在，404则不存在。

### 实时性
一般来说，get API是实时的，它不会被index的刷新频率所影响（当数据变得可搜索后）。

为了关闭实时性，可以传入 realtime参数为false，或者全局设置 action.get.realtime为false。

### 可选择的Type
get API可以将_type作为可选项。设置为_all会获取所有符合id的type。

### Source过滤器
一般情况下，get操作都会返回_source内容，除非你设置了fields参数或者关闭_source字段。
不返回_source字段：
```
curl -XGET 'http://localhost:9200/twitter/tweet/1?_source=false'
```
如果你只需要一两个字段的话，你可以使用_source_include 或 _source_exclude参数来设置。这对于大文档的检索是非常有益的，它可以减少网络中的传输字节数。
```
curl -XGET 'http://localhost:9200/twitter/tweet/1?_source_include=*ser&_source_exclude=message'
```
返回结果为：
```
{
  "_index": "twitter",
  "_type": "tweet",
  "_id": "1",
  "_version": 5,
  "found": true,
  "_source": {
    "user": "kimchy"
  }
}
```
如果仅仅设置include，可以这样：
```
curl -XGET 'http://localhost:9200/twitter/tweet/1?_source_include=*ser,message'
```

### Fields
get操作可以通过fields参数来指定返回的字段：
```
curl -XGET 'http://localhost:9200/twitter/tweet/1?fields=user,message'
```
返回：
```
{
  "_index": "twitter",
  "_type": "tweet",
  "_id": "1",
  "_version": 5,
  "found": true,
  "fields": {
    "message": [
      "trying out Elasticsearch"
    ],
    "user": [
      "kimchy"
    ]
  }
}
```
为了向后兼容，如果请求的字段没有存储，它们将从_source中被解析和提取。这个方法会被source过滤器的参数所覆盖。
被请求的字段的值将会以数组形式返回。元数据字段，像_routing或者_parent字段则永远不会以数组形式返回。

### 生成的字段
如果在indexing后没有刷新，GET将会通过translog日志来查询文档。然而，一些字段只有在indexing时才生成。如果你想要获取索引中的字段，就会产生异常。可以设置来ignore_erros_on_generated_fields=true忽视异常。

translog是一个操作日志。每次进行索引操作后，数据变动都会先存在translog中，之后再刷新到es中。实时查询，其实是读取了translog中还未持久化的数据。

### 只返回_source
使用 /{index}/{type}/{id}/_source 可以只返回_source字段。
```
curl -XGET 'http://localhost:9200/twitter/tweet/1/_source'
```
返回：
```
{
  "user": "kimchy",
  "postDate": "2009-11-15T14:12:12",
  "message": "trying out Elasticsearch"
}
```
当然，还可以指定返回的字段：
```
curl -XGET 'http://localhost:9200/twitter/tweet/1/_source?_source_include=*ser&_source_exclude=message'
```
使用HEAD方式请求也可以判断文档是否存在:
```
curl -XHEAD -i 'http://localhost:9200/twitter/tweet/1/_source'
```

### 路由
也可以使用路由查询，路由不对的话是查询不到的:
```
curl -XGET 'http://localhost:9200/twitter/tweet/1?routing=kimchy'
```

### Preference
使用preference控制哪个分片来执行get请求。一般情况下，get是在分片中随机执行的。
- primary: 这个操作仅仅会在主分片上执行。
- local : 这个操作会尽可能的在本地分片上执行。
- Custom (string) value： 用户可以自定义值，对于相同的分片可以设置相同的值。这样可以保证不同的刷新状态下，查询不同的分片。就像sessionid或者用户名一样。

### 刷新
refresh参数可以控制在get操作时，执行刷新，使数据可用。将它设为true时会导致每次查询都先进行刷新，这样会影响系统效率。

### 分布式
get操作被散列到一个特定的分片id。然后根据分片id直接请求其中一个备份，并返回结果。这个备份是由在这个分片id组里面的主分片和它的备份所组成的。这就意味着，备份数量越多，get的扩展性越好。

### 版本支持
当指定的version参数和当前版本一致时，可以获取文档。当版本类型为FORCE的时候，所有的版本类型都可以检索文档。
在删除一个文档后，es在内部并不立即删除这个文档，而是将它标记起来，当然， 你也不能进行访问。es会在后台逐渐清理掉这些文档。

## Delete API
delete API允许通过指定的id来删除对应的文档。
```
$ curl -XDELETE 'http://localhost:9200/twitter/tweet/1'
```
返回:
```
{
    "_shards" : {
        "total" : 10,
        "failed" : 0,
        "successful" : 10
    },
    "found" : true,
    "_index" : "twitter",
    "_type" : "tweet",
    "_id" : "1",
    "_version" : 2
}
```

### 版本号
每个文档的索引都是有版本号的。当删除文档时，需要指定文档的version，并且在我们要进行删除操作时，版本号不能被其它的程序改变，因为更新等操作都会引起版本号的改变。

### 路由
也可以指定路由来进行删除，路有错误的话会导致删除失败。
```
$ curl -XDELETE 'http://localhost:9200/twitter/tweet/1?routing=kimchy'
```

### Parent
删除操作也可以指定父文档。再删除父文档的时候，不会删除子文档。有一种删除子文档的方法，就是使用delete-by-query。

### 自动创建索引
在执行删除操作时，如果当前索引不存在，则会自动创建一个索引，同时自动创建一个指定的type。

### 分布式
和get API中的意思差不多。

### 写一致性
操作是否能成功执行，是由对应的组里面的可用的分片备份数量决定的。这个值可以是one, quorum 和 all。可以用consistency 请求参数来设置。节点级的设置是在 action.write_consistency ， 默认是quorum。

### 刷新
refresh参数设置为true，可以在删除操作执行后，立即刷新分片，保证其数据可以立即被查询。不过要慎用！

### 超时时间
当分片不可用的时候，删除操作会等待一段时间执行。可以设置其timeout。
```
$ curl -XDELETE 'http://localhost:9200/twitter/tweet/1?timeout=5m'
```

## Update API
updateAPI 允许通过脚本来更新文档。它使用版本管理来控制在get和reindex时不会有更新操作。
注意，update这个操作是重新索引文档，它只是减少了网络影响和版本冲突。

首先，创建一个索引文档：
```
curl -XPUT localhost:9200/test/type1/1 -d '{
    "counter" : 1,
    "tags" : ["red"]
}'
```

### 使用脚本更新
可以通过执行一个脚本来增加counter：
```
POST test/type1/1/_update
{
    "script" : {
        "inline": "ctx._source.counter += count",
        "params" : {
            "count" : 4
        }
    }
}
```

或者也可以添加一个tag到tags[]中：
```
POST test/type1/1/_update
{
    "script" : {
        "inline": "ctx._source.tags.add(params.tag)",
        "params" : {
            "tag" : "blue"
        }
    }
}
```

除了_source，还可以通过ctx来获得这些变量： _index , _type , _id, _version , _routing , _parent , 和 _now (当前时间戳)。

比如：
```
POST test/type1/1/_update
{
    "script" : {
        "inline": "ctx._source.tages.add(ctx._id)"
    }
}
```


除此之外，我们还可以为文档添加一个新的字段：
```
POST test/type1/1/_update
{
    "script" : "ctx._source.new_field = \"value_of_new_field\""
}
```
字段为 new_field ， 且值为value_of_new_field。

也可以删除字段：
```
POST test/type1/1/_update
{
    "script" : "ctx._source.remove(\"new_field\")"
}
```

甚至，我们还可以通过一系列逻辑判断来改变它的操作类型，例子中，如果tags包含 green，则删除文档，否则什么都不做：
```
POST test/type1/1/_update
{
    "script" : {
        "inline": "if (ctx._source.tags.contains(tag)) { ctx.op = \"delete\" } else { ctx.op = \"none\" }",
        "params" : {
            "tag" : "green"
        }
    }
}
```
### 通过文档来更新
updateAPI还支持传递一部分文档，这部分文档将会被合并到当前文档中，使用doc可以实现简单的递归合并、内部合并、替换KV以及数组。
```
POST test/type1/1/_update
{
    "doc" : {
        "name" : "new_name"
    }
}
```

如果 doc 和 script 都被指定了，那么doc会被忽略。最好的方式是将doc的操作也放到 script 中去执行。

### 更新检测
如果doc中定义的部分与现在的文档相同，则默认不会执行任何动作。设置detect_noop=false，就会无视是否修改，强制合并到现有的文档。

```
POST test/type1/1/_update
{
    "doc" : {
        "name" : "new_name"
    }
}
```
上面例子中，执行后的结果如下：
```
{
  "_index": "test",
  "_type": "type1",
  "_id": "1",
  "_version": 9,
  "_shards": {
    "total": 0,
    "successful": 0,
    "failed": 0
  }
}
```
如果请求设置了`"detect_noop": false`
```
POST test/type1/1/_update
{
    "doc" : {
        "name" : "new_name"
    },
    "detect_noop": false
}
```
那么：
```
{
  "_index": "test",
  "_type": "type1",
  "_id": "1",
  "_version": 10,
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  }
}
```

### Upserts
如果文档不存在，那么upsert 元素的内容将被插入到文档中，如果文档存在，则执行脚本。
```
POST test/type1/1/_update
{
    "script" : {
        "inline": "ctx._source.counter += count",
        "params" : {
            "count" : 4
        }
    },
    "upsert" : {
        "counter" : 1
    }
}
```

#### scripted_upsert
如果你想不管文档存不存在都执行脚本，可以设置 scripted_upsert 为 true。
```
POST test/type1/1/_update
{
	"scripted_upsert":true,
    "script" : {
        "inline": "ctx._source.counter += count",
        "params" : {
            "count" : 4
        }
    },
    "upsert" : {
        "counters" : 1
    }
}
```
这样，counters不会被插入到文档中，而是只会执行脚本内容。

#### doc_as_upsert
当使用doc时，如果id为2的文档不存在，那么会报错，而如果使用doc_as_upsert，则可以在文档不存在的时候，把doc中的内容插入到文档中。
```
POST test/type1/2/_update
{
    "doc" : {
        "name" : "new_name"
    },
    "doc_as_upsert" : true
}
```

### 参数
- retry_on_conflict : 当执行索引和更新的时候，有可能另一个进程正在执行更新。这个时候就会造成冲突，这个参数就是用于定义当遇到冲突时，重试的次数。

- routing ： routing是用来为update请求路由到正确的分片的，并且当文档不存在时进行upsert操作。不能用来更新现有文档的路由。

- parent： parent用来路由到正确的分片，并且当文档不存在时进行upsert操作。不能用来更新现有文档的 parent。如果一个index的路由被指定了，那么它会覆盖parent的路由，并且用新的路由来路由请求。

- timeout ： 用来等待一个分片可用。

- wait_for_active_shards ： 指定在更新操作前，分片副本可用的数量。

- refresh : 当执行操作的时候，会自动刷新索引。

- _source ： 控制被更新的source是否、如何返回。默认情况下，update操作是不返回source的。

-  version & version_type ： updateAPI使用es的版本号来保证文档在update时没有被改变。可以使用 version参数来指定，只有当version号匹配当前版本号时，才能更新成功。你也可以使用 force 参数来强制指定版本号来更新，不过这一般用于版本修正。

更新操作是不支持外部版本号的，因为本来外部版本号就脱离系统的版本控制，如果再执行更新操作，那就彻底乱了。如果使用了外部版本号，可以使用Index代替更新操作，重新索引文档。

## Update By Query API
update By Query API 是一个新的功能，尚需实践检验，因此，以后可能会有所修改，并且不向后兼容。

这个api会更新索引中的每一个文档并且不改变它们的source。
```
POST twitter/_update_by_query?conflicts=proceed
```
执行结果为：
```
{
  "took": 2549,
  "timed_out": false,
  "total": 3,
  "updated": 3,
  "batches": 1,
  "version_conflicts": 0,
  "noops": 0,
  "retries": 0,
  "throttled_millis": 0,
  "requests_per_second": "unlimited",
  "throttled_until_millis": 0,
  "failures": []
}
```

_update_by_query 会在执行时获取一个index的快照，并且用内部版本号对它找到的文档进行索引。这意味着如果在快照生成和处理索引请求时文档有所改变，你会遇到版本冲突问题。当版本号匹配时，文档会进行更新，并且版本号会增加。

所有的查询和更新失败都会导致_update_by_query的终止并且返回失败信息。但是已更新的数据不会被回滚。当第一条失败时，会导致程序中止，这意味着所有的更新失败，这时候会返回大量的失败元素。

如果你想当版本冲突时只进行简单的计数，而不中止，可以在url中设置 conflicts=proceed或者在请求内容中设置"conflicts": "proceed"，在api中，可以只针对索引中的一个类型进行操作：
```
POST twitter/tweet/_update_by_query?conflicts=proceed
```

还可以使用Query DSL：
```
POST twitter/_update_by_query?conflicts=proceed
{
  "query": { 
    "term": {
      "user": "kimchy"
    }
  }
}
```

之前介绍的都是没有改变文档内容的，其实_update_by_query是可以在支持脚本对文档内容的更新。例如：

```
POST twitter/_update_by_query
{
  "script": {
    "inline": "ctx._source.likes++"
  },
  "query": {
    "term": {
      "user": "kimchy"
    }
  }
}
```
## Multi Get API
Mget api 可以获得多个文档结果，它们会以数组方式返回：
```
curl 'localhost:9200/_mget' -d '{
    "docs" : [
        {
            "_index" : "test",
            "_type" : "type",
            "_id" : "1"
        },
        {
            "_index" : "test",
            "_type" : "type",
            "_id" : "2"
        }
    ]
}'
```
它也可以指定index或type：
```
curl 'localhost:9200/test/type/_mget' -d '{
    "docs" : [
        {
            "_id" : "1"
        },
        {
            "_id" : "2"
        }
    ]
}'
```
或者指定id:
```
curl 'localhost:9200/test/type/_mget' -d '{
    "ids" : ["1", "2"]
}'
```

### _source 过滤
还可以指定_source中包含的字段：
```
curl 'localhost:9200/_mget' -d '{
    "docs" : [
        {
            "_index" : "test",
            "_type" : "type",
            "_id" : "1",
            "_source" : false
        },
        {
            "_index" : "test",
            "_type" : "type",
            "_id" : "2",
            "_source" : ["field3", "field4"]
        },
        {
            "_index" : "test",
            "_type" : "type",
            "_id" : "3",
            "_source" : {
                "include": ["user"],
                "exclude": ["user.location"]
            }
        }
    ]
}'
```

### field 过滤
```
curl 'localhost:9200/test/type/_mget?stored_fields=field1,field2' -d '{
    "docs" : [
        {
            "_id" : "1" 
        },
        {
            "_id" : "2",
            "stored_fields" : ["field3", "field4"] 
        }
    ]
}'
```

## Bulk API
bulk API 可以通过一条请求来实现多个增加、删除操作，这能很有效的提高索引速度。

bulk默认的json格式是：
```
action_and_meta_data\n
optional_source\n
action_and_meta_data\n
optional_source\n
....
action_and_meta_data\n
optional_source\n
```
注意，它是以 \n 为每行的结束符。
一般，它用做index，create，delete和update操作。index和create的source放在下一行，而delete只需一行，update则是由于有script、upsert等操作，不局限于两行。

```
curl -s -XPOST localhost:9200/_bulk
{ "index" : { "_index" : "test", "_type" : "type1", "_id" : "1" } }
{ "field1" : "value1" }
```

多种操作的例子：
```
{ "index" : { "_index" : "test", "_type" : "type1", "_id" : "1" } }
{ "field1" : "value1" }
{ "delete" : { "_index" : "test", "_type" : "type1", "_id" : "2" } }
{ "create" : { "_index" : "test", "_type" : "type1", "_id" : "3" } }
{ "field1" : "value3" }
{ "update" : {"_id" : "1", "_type" : "type1", "_index" : "index1"} }
{ "doc" : {"field2" : "value2"} }
```

在url中可以设置 index 和type ，如果在doc中也设置了，那么会覆盖掉url中的设置。

## Reindex API
reindex API 也是一个实验性质的，不保证向后兼容。

Reindex API 不会建立一个目标index，他也不会复制源索引的配置。你需要在运行_reindex之前配置好映射，分片和复制。

_reindex最基本的用法是将文档从一个索引拷贝到另一个索引：
```
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter"
  }
}
```
和update_by_query 一样，_reindex从源索引得到一个快照，并且目标索引必须是不同的索引，而且也不会引起版本冲突。 dest 元素可以像 indexAPI那样配置，来执行乐观并发控制。可以不管version_type 或者设置它为internal ，这会使es直接将数据拷贝到目标索引，不管数据的并发情况:
```
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "version_type": "internal"
  }
}
```
将version_type设置为external会导致es将version保存在source中，在更新文档的时候进行一个乐观的版本控制：
```
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "version_type": "external"
  }
}
```
设置op_type为create将会导致_reindex仅仅只会创建没有的文档，而已经存在的文档则会导致版本冲突：
```
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "op_type": "create"
  }
}
```

默认情况下，版本冲突会导致_reindex进程停止，可是设置`"conflicts": "proceed"`来计算失败的数量：
```
POST _reindex
{
  "conflicts": "proceed",
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "op_type": "create"
  }
}
```
也可以根据查询结果来进行拷贝：
```
POST _reindex
{
  "source": {
    "index": "twitter",
    "type": "tweet",
    "query": {
      "term": {
        "user": "kimchy"
      }
    }
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

指定多个index或type:
```
POST _reindex
{
  "source": {
    "index": ["twitter", "blog"],
    "type": ["tweet", "post"]
  },
  "dest": {
    "index": "all_together"
  }
}
```

还可以加上排序条件和条数限制
```
POST _reindex
{
  "size": 10000,
  "source": {
    "index": "twitter",
    "sort": { "date": "desc" }
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

加入script：
```
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "version_type": "external"
  },
  "script": {
    "inline": "if (ctx._source.foo == 'bar') {ctx._version++; ctx._source.remove('foo')}"
  }
}
```

### 从远程服务器reindex
```
POST _reindex
{
  "source": {
    "remote": {
      "host": "http://otherhost:9200",
      "username": "user",
      "password": "pass"
    },
    "index": "source",
    "query": {
      "match": {
        "test": "data"
      }
    }
  },
  "dest": {
    "index": "dest"
  }
}
```
## Term Vectors
返回一个带有词条信息和统计的特殊的文档。这个文档可以存在索引中，或者由用户人为提供。词条信息默认是实时的，可以通过realtime 参数设为 true 来改成非实时的。
```
curl -XGET 'http://localhost:9200/twitter/tweet/1/_termvectors?pretty=true'
```

也可以指定字段，返回这个字段的信息：
```
curl -XGET 'http://localhost:9200/twitter/tweet/1/_termvectors?fields=text,...'
```

### 返回的信息
可以请求三种信息：词条信息、词条统计以及字段统计。默认情况下，会返回所有词条信息和字段统计，但不会返回词条统计。

#### 词条信息
- 词条在字段中的出现频率（始终返回）
- 词条位置（position:true）
- 开始和结束偏移量（offsets:true）
- 词条容量（payloads : true） ，比如base64编码的字节数。

如果请求的信息不在索引中，它将尽量不计算。另外，term vectors甚至可以计算不在索引中而是由用户提供的文档。

#### 词条统计
将term_statistics 设为 true 可以返回：
- 词条整体出现频率（词条在所有文档中出现的频率）
- 文档频率（包含当前词条的文档数）

默认情况下，这两个都不会返回，因为会影响系统效率。

#### 字段统计
默认情况下，field_statistics ： true ， 这会返回：
- 包含这个字段的文档数
- 所有包含这个字段的文档中这个字段包含的所有词条出现频率的总和。
- 这个字段中包含的每一个词条的出现频率总和。

#### 词条过滤
通过filter参数，返回的词条可以基于它们的 tf-idf分数进行过滤。这对于为了找出一个特定的文档是很有帮助的。有以下几个参数可用：

- max_num_terms : 每个字段必须返回的最大词条数量，默认是25。
- min_term_freg : 忽略在source中词语频率低于这个值的词条。默认是1.
- max_term_freq ： 与上面相反。默认是无穷大。
- min_doc_freq ： 忽略词条出现频率低于这个值的文档。默认是1.
- max_doc_freq ： 与上面相反，默认是无穷大。
- min_word_length ： 忽略长度短语这个值的短语。 默认是0。
- max_word_length ： 与上面相反，默认是无穷大。


### 例子
首先创建存有 term vectors ， payloads 的索引：
```
curl -s -XPUT 'http://localhost:9200/twitter/' -d '{
  "mappings": {
    "tweet": {
      "properties": {
        "text": {
          "type": "string",
          "term_vector": "with_positions_offsets_payloads",
          "store" : true,
          "analyzer" : "fulltext_analyzer"
         },
         "fullname": {
          "type": "string",
          "term_vector": "with_positions_offsets_payloads",
          "analyzer" : "fulltext_analyzer"
        }
      }
    }
  },
  "settings" : {
    "index" : {
      "number_of_shards" : 1,
      "number_of_replicas" : 0
    },
    "analysis": {
      "analyzer": {
        "fulltext_analyzer": {
          "type": "custom",
          "tokenizer": "whitespace",
          "filter": [
            "lowercase",
            "type_as_payload"
          ]
        }
      }
    }
  }
}'
```
添加数据：
```
curl -XPUT 'http://localhost:9200/twitter/tweet/1?pretty=true' -d '{
  "fullname" : "John Doe",
  "text" : "twitter test test test "
}'

curl -XPUT 'http://localhost:9200/twitter/tweet/2?pretty=true' -d '{
  "fullname" : "Jane Doe",
  "text" : "Another twitter test ..."
}'
```

#### 查询id=1 的text字段的所有信息和统计：
```
curl -XGET 'http://localhost:9200/twitter/tweet/1/_termvectors?pretty=true' -d '{
  "fields" : ["text"],
  "offsets" : true,
  "payloads" : true,
  "positions" : true,
  "term_statistics" : true,
  "field_statistics" : true
}'
```
返回结果是：
```
{
  "_index": "twitter",
  "_type": "tweet",
  "_id": "1",
  "_version": 1,
  "found": true,
  "took": 91,
  "term_vectors": {
    "text": {
      "field_statistics": {
        "sum_doc_freq": 6   //该字段中词的出现频率(在所有文档中),
        "doc_count": 2    // 包含该字段的文档数
        "sum_ttf": 8      // 该字段中词的数量（包含重复的词）
      },
      "terms": {
        "test": {
          "doc_freq": 2,   //该词出现在几个文档中
          "ttf": 4,		   //该词出现的次数
          "term_freq": 3,  //该词在这个文档中出现的频率
          "tokens": [
            {
              "position": 1,
              "start_offset": 8,
              "end_offset": 12,
              "payload": "d29yZA=="
            },
            {
              "position": 2,
              "start_offset": 13,
              "end_offset": 17,
              "payload": "d29yZA=="
            },
            {
              "position": 3,
              "start_offset": 18,
              "end_offset": 22,
              "payload": "d29yZA=="
            }
          ]
        },
        "twitter": {
          "doc_freq": 2,
          "ttf": 2,
          "term_freq": 1,
          "tokens": [
            {
              "position": 0,
              "start_offset": 0,
              "end_offset": 7,
              "payload": "d29yZA=="
            }
          ]
        }
      }
    }
  }
}
```

#### 对不在索引中存储的文档进行统计
下面这个文档并没有在索引中，但是也可以对它进行统计：
```
curl -XGET 'http://localhost:9200/twitter/tweet/1/_termvectors?pretty=true' -d '{
  "fields" : ["text", "some_field_without_term_vectors"],
  "offsets" : true,
  "positions" : true,
  "term_statistics" : true,
  "field_statistics" : true
}'
```

#### 在请求中人为定义的文档
```
curl -XGET 'http://localhost:9200/twitter/tweet/_termvectors' -d '{
  "doc" : {
    "fullname" : "John Doe",
    "text" : "twitter test test test"
  }
}'
```
如果设置了动态mapping，那么如果文档不存在会自动添加。

#### 针对每一个字段分析
可以使用per_field_analyzer参数定义该字段的分析器，这样每个字段都可以使用不同的分析器，分析其词条向量的信息。如果这个字段已经经过存储，那么会重新生成它的词条向量
```
curl -XGET 'http://localhost:9200/twitter/tweet/_termvectors' -d '{
  "doc" : {
    "fullname" : "John Doe",
    "text" : "twitter test test test"
  },
  "fields": ["fullname"],
  "per_field_analyzer" : {
    "fullname": "keyword"
  }
}'
```
返回：
````
{
  "_index": "twitter",
  "_type": "tweet",
  "_version": 0,
  "found": true,
  "took": 178,
  "term_vectors": {
    "fullname": {
      "field_statistics": {
        "sum_doc_freq": 4,
        "doc_count": 2,
        "sum_ttf": 4
      },
      "terms": {
        "John Doe": {
          "term_freq": 1,
          "tokens": [
            {
              "position": 0,
              "start_offset": 0,
              "end_offset": 8
            }
          ]
        }
      }
    },
    "text": {
      "field_statistics": {
        "sum_doc_freq": 6,
        "doc_count": 2,
        "sum_ttf": 8
      },
      "terms": {
        "test": {
          "term_freq": 3,
          "tokens": [
            {
              "position": 1,
              "start_offset": 8,
              "end_offset": 12
            },
            {
              "position": 2,
              "start_offset": 13,
              "end_offset": 17
            },
            {
              "position": 3,
              "start_offset": 18,
              "end_offset": 22
            }
          ]
        },
        "twitter": {
          "term_freq": 1,
          "tokens": [
            {
              "position": 0,
              "start_offset": 0,
              "end_offset": 7
            }
          ]
        }
      }
    }
  }
}
````

#### 过滤词条
```
GET /twitter/tweet/_termvectors
{
    "doc": {
      "plot": "When wealthy industrialist Tony Stark is forced to build an armored suit after a life-threatening incident, he ultimately decides to use its technology to fight against evil."
    },
    "term_statistics" : true,
    "field_statistics" : true,
    "positions": false,
    "offsets": false,
    "filter" : {
      "max_num_terms" : 3,
      "min_term_freq" : 1,
      "min_doc_freq" : 1
    }
}
```
返回：
```
{
  "_index": "twitter",
  "_type": "tweet",
  "_version": 0,
  "found": true,
  "took": 1324,
  "term_vectors": {
    "plot": {
      "field_statistics": {
        "sum_doc_freq": 26,
        "doc_count": 1,
        "sum_ttf": 28
      },
      "terms": {
        "after": {
          "doc_freq": 1,
          "ttf": 1,
          "term_freq": 1,
          "score": 0.30685282
        },
        "against": {
          "doc_freq": 1,
          "ttf": 1,
          "term_freq": 1,
          "score": 0.30685282
        },
        "to": {
          "doc_freq": 1,
          "ttf": 3,
          "term_freq": 3,
          "score": 0.92055845
        }
      }
    }
  }
}
```
## Multi termvectors API
这个API可以进行一次获得多个termvectors。可以通过指定索引、类型或者id来实现。
```
curl 'localhost:9200/_mtermvectors' -d '{
   "docs": [
      {
         "_index": "testidx",
         "_type": "test",
         "_id": "2",
         "term_statistics": true
      },
      {
         "_index": "testidx",
         "_type": "test",
         "_id": "1",
         "fields": [
            "text"
         ]
      }
   ]
}'
```


参数方面，和termvectors一致，具体参照termvectors。

指定index的例子：
```
curl 'localhost:9200/testidx/_mtermvectors' -d '{
   "docs": [
      {
         "_type": "test",
         "_id": "2",
         "fields": [
            "text"
         ],
         "term_statistics": true
      },
      {
         "_type": "test",
         "_id": "1"
      }
   ]
}'
```

指定type的例子：
```
curl 'localhost:9200/testidx/test/_mtermvectors' -d '{
   "docs": [
      {
         "_id": "2",
         "fields": [
            "text"
         ],
         "term_statistics": true
      },
      {
         "_id": "1"
      }
   ]
}'
```

如果所有的文档都是同一个index和type，那么就简单很多：
```
curl 'localhost:9200/testidx/test/_mtermvectors' -d '{
    "ids" : ["1", "2"],
    "parameters": {
        "fields": [
                "text"
        ],
        "term_statistics": true,
        …
    }
}'
```

和termvectors一样，也能够使用用户自己定义的文档：
```
curl 'localhost:9200/_mtermvectors' -d '{
   "docs": [
      {
         "_index": "testidx",
         "_type": "test",
         "doc" : {
            "fullname" : "John Doe",
            "text" : "twitter test test test"
         }
      },
      {
         "_index": "testidx",
         "_type": "test",
         "doc" : {
           "fullname" : "Jane Doe",
           "text" : "Another twitter test ..."
         }
      }
   ]
}'
```







