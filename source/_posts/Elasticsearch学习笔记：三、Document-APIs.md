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
//TODO



