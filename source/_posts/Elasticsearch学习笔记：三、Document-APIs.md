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
