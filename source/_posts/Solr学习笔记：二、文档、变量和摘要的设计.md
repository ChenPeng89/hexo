---
title: 'Solr学习笔记：二、Document、Field和Schema的设计'
date: 2016-06-29 09:28:58
tags: Solr
---
## 前言
Solr的工作流程很简单。你先给它添加你想要知道的信息，然后问它要。你给它信息的过程叫 indexing 或者 updating。你问它要信息的过程叫 query。Solr可以在schema为实体的不同变量、类型建立索引。

Solr的基本信息单元是document，它是一个描述某种事物的数据集合。document是由field组成的，它是更具体的信息。比如，鞋子的尺码、姓名都可以是field。field可以是多种类型，Solr允许用户定义field的类型：field type，定义准确的field type可以帮你准确的查找结果。

field analysis告诉了Solr如何建立索引。比如你会遇见这样的问题，一个人的传记中，会有"the" , "a " 等这样的词，通过field analysis你可以告诉Solr怎么进行分词。它是field type 中的重要组成部分。

Solr将field type 等信息存在schema文件中。文件的名称和路径取决于你如何初始化Solr或者以后会怎样修改它。

**managed-schema** -- 是在运行时可以通过Schema API来修改的Solr schema文件。如果你使用别的名称，你可以显式的指定它，但是内容的更新需要Solr自动来完成。

**schema.xml** -- 它是传统的schema文件名，用户可以通过 ClassicIndexSchemaFactory 来手动编辑它。

如果你使用Cloud模式，你不会在本地文件系统中找到这个文件，可以通过Schema API或者Solr Admin UI 中的Cloud Screens看到。

不论你怎样定义文件名，问价你的结构是不会变的。如果你使用managed schema，那么原则上你只能通过Schema API来改变它，绝不能手动变更。如果你不使用 managed schema ， 那么你只能通过手动改变文件，Schema API不能操作任何改变。
如果你使用SolrCloud但是没有使用Schema API，那么你需要从ZooKeeper中使用upconfig或者downconfig命令来对schema.xml做一个本地备份和上传修改。

## Field type
field type 包括下面四种信息类型：

**field type 名称（必须）。**
**类型的实现类（必须）。**
**如果类型是 TextField ， 需要添加field analysis。**
**field type 属性 ， 依赖于实现类，一些属性是必须的。**

```xml
<fieldType name="text_general" class="solr.TextField" positionIncrementGap="100">
  <analyzer type="index">
    <tokenizer class="solr.StandardTokenizerFactory"/>
    <filter class="solr.StopFilterFactory" ignoreCase="true" words="stopwords.txt" />
    <!-- in this example, we will only use synonyms at query time
    <filter class="solr.SynonymFilterFactory" synonyms="index_synonyms.txt" ignoreCase="true" expand="false"/>
    -->
    <filter class="solr.LowerCaseFilterFactory"/>
  </analyzer>
  <analyzer type="query">
    <tokenizer class="solr.StandardTokenizerFactory"/>
    <filter class="solr.StopFilterFactory" ignoreCase="true" words="stopwords.txt" />
    <filter class="solr.SynonymFilterFactory" synonyms="synonyms.txt" ignoreCase="true" expand="true"/>
    <filter class="solr.LowerCaseFilterFactory"/>
  </analyzer>
</fieldType>
```
上面的例子中，包含了field type名称 text-general，实现类 solr.TextField。实现类是用来保证field呗正确处理。在schema.xml中，solr字符串是org.apache.solr.schema 或 org.apache.solr.analysis的缩写。因此，solr.TextField 实际上是 org.apache.solr.schema.TextField。

### field type属性

#### 一般属性

- name -- field type名称。用来定义field。强烈建议名称只包含字母数字或下划线字符,而不是从数字开头。
- class -- 存储或索引数据的class的名称。如果你使用solr开头，那么类似solr.TextField会有效，如果使用第三方的类，那么需要写全类名和包名。
- positionIncrementGap -- 对于多个值的field，指定多个值之间的距离，防止错误的匹配。integer。
- autoGeneratePhraseQueries -- 用于 text field。如果设为true，Solr自动为相邻的短语生成词组查询。如果设为false，短语必须加上双引号才能被视为一个词组。true/false
- docValuesFormat -- 为此类型的field定义一个DocValuesFormat ，这需要一个schema-aware解析器。例如solrconfig.xml中定义的SchemaCodecFactory。
- postingsFormat -- 为此类型的field定义一个postingsFormat，这需要一个schema-aware解析器。例如solrconfig.xml中定义的SchemaCodecFactory 。

注意： Lucene向下兼容时只支持默认的解析器。如果你选择在schema.xml中自定义postingsFormat 或 docValuesFormat，更新到以后的版本时，需要切换为默认的解析器，并且在更新前重新优化你的index，或者重建你的index。

### field默认属性
这些属性既可以在field type中被指定，也可以在单独的field中被field type提供的值所覆盖。默认的值依赖于FieldType的class，进而可能依赖于<schema/>中的version属性。

- indexed -- 如果设为true，field的值可以用来检索文档。  true/false ， 默认为true。
- stored -- 如果设为true，field的实际值可以用来被查询。  true/false ， 默认为true。
- docValues -- 如果设为true，field的值会被放入一个以列为主的DocValues结构中。true/false ， 默认为false。
- sortMissingFirst/sortMissingLast -- 当查询的field没有找到时，控制返回数据的排序方式。（没有field的排在有的之前/没有field的排在有的之后 ， 而不管请求时的排序方式）true/false ， 默认为false。
- multiValued -- 如果设为true，说明一个单独的文档可能包含这个field type 的多个值。true/false ， 默认为false。
- omitNorms -- 如果设为true，则省略了与这个field相关的规范。默认情况下，对于不执行analysis的字段类型设为true。只有全文字段或者需要index-time boost 的字段才需要规范。true/false。
- omitTermFreqAndPositions -- 如果设为true，则不用存储term的频率和position等信息。对于不需要这些信息的字段是一个性能的提升，减少磁盘空间。依赖于位置的字段查询将查找不到信息。这个属性对于所有非text的field默认为true。
- omitPositions -- 和omitTermFreqAndPositions 相似，但是保留词汇的频率信息。true or false
- termVectors/termPositions/termOffsets/termPayloads -- 这些选项指导Solr为每个文档维护全局向量，包括 position，offset ， 和负载信息。这些能用于加速高亮和其它辅助的功能，但是增加了词汇的index长度。他们对于一般使用Solr所必要的。true or false，默认false。
- required -- 阻止添加不包含该字段的任何文档。true or false ，默认为false。
- useDocValuesAsStored -- 如果docValues可用，并且本字段设为true，那么当匹配成功时，会将字段当做store field进行返回（即使它的store = false）。

### field type
- BinaryField -- 二进制数据。
- BoolField -- bool ， 1，t，T都被认为true，其它都为false。
- CollationField -- 支持Unicode类型的查询排序集合。
- CurrencyField -- 支持货币和汇率。
- DateRangeField -- 支持日期范围检索。
- ExternalFileField -- 拉取磁盘上文件中的值。
- EnumField -- 枚举类型。
- ICUCollationField -- 支持Unicode类型的集合来进行排序和范围查询。
- LatLonType -- 经纬度坐标对。维度在前。
- PointType -- 一个N维空间的点。用于查询蓝图或者CAD上的资源。
- PreAnalyzedField -- 提供了一种发送给Solr的序列化令牌流，可选的独立存储的field的值，并不通过任何额外的文本处理将信息存储和索引。
- RandomSortField -- 不包含一个值。查询后将返回一个随机顺序的结果。使用动态field时使用这个功能。
- SpatialRecursivePrefixTreeFieldType -- 接受 纬度,精度 这样的字符串或者其他WKT格式的类型。
- StrField -- 字符串（UTF8或者Unicode编码），用于小的field并且这些字段不会被标记或者分析。他们有一个硬限制，就是小于32K。
- TextField -- Text类型，一般用于多单词或者多标记。
- TrieDateField -- 日期类型。代表一个毫秒精度的时间点。precisionStep="0" 实现高效的日期排序，并减小了索引的大小。precisionStep="8" (默认的)实现了高效的范围查询。
- TrieDoubleField -- Double类型。precisionStep="0" 实现高效的数字排序，并减小了索引的大小。precisionStep="8" (默认的)实现了高效的范围查询。
- TrieField -- 如果使用了这个类型，那么 type 属性必须被指定，有效的值为:integer, long, float, double, date 。使用这个field 和使用其它 Trie field 一样：实现高效的数字排序，并减小了索引的大小。precisionStep="8" (默认的)实现了高效的范围查询。
- TrieFloatField/TrieIntField/TrieLongField -- float/int/long。precisionStep="0" 实现高效的数字排序，并减小了索引的大小。precisionStep="8" (默认的)实现了高效的范围查询。
- UUIDField -- 一般唯一标识符。传一个值为“NEW”的值然后Solr将会创建一个新的UUID。注意，在SolrCloud模式下使用UUID不是一个好的做法，因为每一个备份节点的每一个文档都会有一个唯一的UUID。当添加文档时使用UUIDUpdateProcessorFactory 来生成UUID是一个好的做法。

### 定义Field
Field一般定义在schema.xml中。
`<field name="price" type="float" default="0.0" indexed="true" stored="true"/>`

### 拷贝Field
`<copyField source="cat" dest="text" maxChars="30000" />`
source 是被拷贝的数据，des是拷贝到的field。定义在schema.xml中。
在上面的例子中，会将cat拷贝到text。field的拷贝实在analysis之前的，意味着你能够对相同的内容返回两个field（两个field使用不同的analysis链并存在不同的索引下）。
使用时需要设置field的 multivalued="true"。

也可以使用通配符：
`<copyField source="*_t" dest="text" maxChars="25000" />`

### 动态Field

`<dynamicField name="*_i" type="int" indexed="true"  stored="true"/>`

动态field可以更灵活的使用Solr。

### 其它Schema元素

- uniqueKey -- 用于表示一个唯一的文档，不是必须的。
`<uniqueKey>id</uniqueKey>`

- Similarity
Similarity 是一个Lucene类，用于在搜索时对文档的评分。
TODO...

### Schema API

#### API的入口点
- /schema -- 检索信息，或者增加、删除，替换field，动态field，复制field或者field type。
- /schema/fields -- 检索指定field的信息或指定名称的field。
- /schema/dynamicfields -- 检索所有符合规则的动态field信息。
- /schema/fieldtypes -- 检索所有指定field type 的信息。
- /schema/copyfields -- 检索所有copy field的信息。
- /schema/name -- 检索schema名称。
- /schema/version -- 检索schema版本。
- /schema/uniquekey -- 检索定义的uniqueKey。
- /schema/similarity -- 检索全局similarity 定义。
- /schema/solrqueryparser/defaultoperator -- 检索默认操作。

`http://localhost:8983/solr/gettingstarted/schema`
#### 修改Schema
`POST /collection/schema`
为了添加、删除，修改field、动态field规则、copy field规则、新field type，你可以发送POST请求到/collection/schema/。

- add-field -- 添加一个带参数的field。
- delete-field -- 删除一个field。
- replace-field -- 用一个不同配置的field替换现有的。
- add-dynamic-field -- 添加一个带参数的动态规则。
- delete-dynamic-field -- 删除一个动态field规则。
- replace-dynamic-field -- 替换一个动态field规则。
- add-field-type -- 添加一个field type。
- delete-field-type -- 删除一个field type。
- replace-field-type -- 替换一个field type。
- add-copy-field -- 添加一个copy field规则。
- delete-copy-field -- 删除一个copy field规则。

这些命令可以在单独的POST请求或在同一个POST请求。命令按指定的顺序执行。
每个请求都会返回状态和处理请求所用的时间，但是不会包含整个schema。

当使用API修改schema时，一个core重载会自动发生，以便使更改可立即用于其后索引的文档。之前的被索引的文档不会自动被处理，当使用被改变的schema时，它们必须重新索引。

- 新增Field
add-field 命令向schema中添加一个新的field。如果存在同名的，那么抛出异常。当定义一个了一个手动编辑的schema.xml，所有的属性都可以通过API传递。
```json
curl -X POST -H 'Content-type:application/json' --data-binary '{
  "add-field":{
     "name":"sell-by",
     "type":"tdate",
     "stored":true }
}' http://localhost:8983/solr/gettingstarted/schema
```

- 删除field
delete-field 命令将会删除一个field，如果field不存在或者field是被copy的field，那么会抛出异常。
```json
curl -X POST -H 'Content-type:application/json' --data-binary '{
  "delete-field" : { "name":"sell-by" }
}' http://localhost:8983/solr/gettingstarted/schema
```
- 替换Field
replace-field 命令会替换一个field。你需要提供field的完整定义。这个命令不会部分改变field，是整个替换。如果field不存在则抛出异常。
```json
curl -X POST -H 'Content-type:application/json' --data-binary '{
  "replace-field":{
     "name":"sell-by",
     "type":"date",
     "stored":false }
}' http://localhost:8983/solr/gettingstarted/schema
```

- 添加动态Field规则
```json
curl -X POST -H 'Content-type:application/json' --data-binary '{
  "add-dynamic-field":{
     "name":"*_s",
     "type":"string",
     "stored":true }
}' http://localhost:8983/solr/gettingstarted/schema
```

- 删除动态field规则
delete-dynamic-field 命令删除一个动态field规则。如果动态field规则不存在或者schema包含一个copy field作为源或者目标只和这个动态field匹配，那么会抛出一个异常。

```json
curl -X POST -H 'Content-type:application/json' --data-binary '{
  "delete-dynamic-field":{ "name":"*_s" }
}' http://localhost:8983/solr/gettingstarted/schema
```

- 替换动态field规则
```json
curl -X POST -H 'Content-type:application/json' --data-binary '{
  "replace-dynamic-field":{
     "name":"*_s",
     "type":"text_general",
     "stored":false }
}' http://localhost:8983/solr/gettingstarted/schema
```

- 添加一个新的field type
```json
curl -X POST -H 'Content-type:application/json' --data-binary '{
  "add-field-type" : {
     "name":"myNewTxtField",
     "class":"solr.TextField",
     "positionIncrementGap":"100",
     "analyzer" : {
        "charFilters":[{
           "class":"solr.PatternReplaceCharFilterFactory",
           "replacement":"$1$1",
           "pattern":"([a-zA-Z])\\\\1+" }],
        "tokenizer":{ 
           "class":"solr.WhitespaceTokenizerFactory" },
        "filters":[{
           "class":"solr.WordDelimiterFilterFactory",
           "preserveOriginal":"0" }]}}
}' http://localhost:8983/solr/gettingstarted/schema
```
上面的例子中，我么那只能定义一个单独的analyzer。如果我们想定义单独的分析器，我们需要用分开的 indexAnalyzer 和 queryAnalyzer 代替上面的analyzer。

```json
curl -X POST -H 'Content-type:application/json' --data-binary '{
  "add-field-type":{
     "name":"myNewTextField",
     "class":"solr.TextField",
     "indexAnalyzer":{
        "tokenizer":{
           "class":"solr.PathHierarchyTokenizerFactory", 
           "delimiter":"/" }},
     "queryAnalyzer":{
        "tokenizer":{ 
           "class":"solr.KeywordTokenizerFactory" }}}
}' http://localhost:8983/solr/gettingstarted/schema
```
- 删除一个Field type
使用delete-field-type 删除一个field type。如果这个field type 不存在或者它已经被引用，那么会抛出异常。
```json
curl -X POST -H 'Content-type:application/json' --data-binary '{
  "delete-field-type":{ "name":"myNewTxtField" }
}' http://localhost:8983/solr/gettingstarted/schema
```
- 替换field type 
```json
curl -X POST -H 'Content-type:application/json' --data-binary '{
  "replace-field-type":{
     "name":"myNewTxtField",
     "class":"solr.TextField",
     "positionIncrementGap":"100",
     "analyzer":{
        "tokenizer":{ 
           "class":"solr.StandardTokenizerFactory" }}}
}' http://localhost:8983/solr/gettingstarted/schema
```
- 新增一个新的copy field 规则
参数：
**source** -- 源field。
**dest** -- 将一个field或一组field拷贝到的目的field。
**maxChars** -- 拷贝的字符数量的上限。

```json
curl -X POST -H 'Content-type:application/json' --data-binary '{
  "add-copy-field":{
     "source":"shelf",
     "dest":[ "location", "catchall" ]}
}' http://localhost:8983/solr/gettingstarted/schema
```

- 删除一个copy field 规则
```json
curl -X POST -H 'Content-type:application/json' --data-binary '{
  "delete-copy-field":{ "source":"shelf", "dest":"location" }
}' http://localhost:8983/solr/gettingstarted/schema
```

- 单个POST中加入多命令
支持在单个的post中加入多命令，API支持事务。
```json
curl -X POST -H 'Content-type:application/json' --data-binary '{
  "add-field-type":{
     "name":"myNewTxtField",
     "class":"solr.TextField",
     "positionIncrementGap":"100",
     "analyzer":{
        "charFilters":[{
           "class":"solr.PatternReplaceCharFilterFactory",
           "replacement":"$1$1",
           "pattern":"([a-zA-Z])\\\\1+" }],
        "tokenizer":{ 
           "class":"solr.WhitespaceTokenizerFactory" },
        "filters":[{
           "class":"solr.WordDelimiterFilterFactory",
           "preserveOriginal":"0" }]}},
   "add-field" : {
      "name":"sell-by",
      "type":"myNewTxtField",
      "stored":true }
}' http://localhost:8983/solr/gettingstarted/schema
```

重复的命令，可以修改成下面这样：
``` json
curl -X POST -H 'Content-type:application/json' --data-binary '{
  "add-field":{
     "name":"shelf",
     "type":"myNewTxtField",
     "stored":true },
  "add-field":{
     "name":"location",
     "type":"myNewTxtField",
     "stored":true },
  "add-copy-field":{
     "source":"shelf",
      "dest":[ "location", "catchall" ]}
}' http://localhost:8983/solr/gettingstarted/schema
```
可以修改成：
``` json
curl -X POST -H 'Content-type:application/json' --data-binary '{
  "add-field":[
     { "name":"shelf",
       "type":"myNewTxtField",
       "stored":true },
     { "name":"location",
       "type":"myNewTxtField",
       "stored":true }]
}' http://localhost:8983/solr/gettingstarted/schema
```

- 修改所有副本的schema
在SolrCloud模式下，对schema的修改将会传播到集合中的所有节点。你可以通过在request中传递updateTimeoutSecs参数来设置所有节点确认的超时时间。这可以在规定的时间内确认所有节点都更新完毕，使你的系统更加健壮。如果确认信息没在指定时间到达，那么请求会失败并且报出异常，异常包括失败节点的错误信息。大部分情况下，唯一的做法是等待一段时间后重新请求，如果问题仍然存在，那么可以根据服务日志来查找解决问题。如果你不使用updateTimeoutSecs参数，接收节点的默认是在更新ZooKeeper后立即返回，这样，你就不能确保所有的节点都更新成功。

#### 检索Schema信息
- 检索全部Schema
`GET /collection/schema`
输入：
路径参数：
collection -- 集合或core名称。
查询参数：
wt : 默认是json格式，可选的有 json、xml、schema.xml。

`curl http://localhost:8983/solr/gettingstarted/schema?wt=json`
```json
{
  "responseHeader":{
    "status":0,
    "QTime":5},
  "schema":{
    "name":"example",
    "version":1.5,
    "uniqueKey":"id",
    "fieldTypes":[{
        "name":"alphaOnlySort",
        "class":"solr.TextField",
        "sortMissingLast":true,
        "omitNorms":true,
        "analyzer":{
          "tokenizer":{
            "class":"solr.KeywordTokenizerFactory"},
          "filters":[{
              "class":"solr.LowerCaseFilterFactory"},
            {
              "class":"solr.TrimFilterFactory"},
            {
              "class":"solr.PatternReplaceFilterFactory",
              "replace":"all",
              "replacement":"",
              "pattern":"([^a-z])"}]}},
...
    "fields":[{
        "name":"_version_",
        "type":"long",
        "indexed":true,
        "stored":true},
      {
        "name":"author",
        "type":"text_general",
        "indexed":true,
        "stored":true},
      {
        "name":"cat",
        "type":"string",
        "multiValued":true,
        "indexed":true,
        "stored":true},
...
    "copyFields":[{
        "source":"author",
        "dest":"text"},
      {
        "source":"cat",
        "dest":"text"},
      {
        "source":"content",
        "dest":"text"},
...
      {
        "source":"author",
        "dest":"author_s"}]}}
```

- 列出field
`GET /collection/schema/fields`

`GET /collection/schema/fields/fieldname `

路径参数：
collection -- 集合或core名称。
fieldname -- 指定的field名称。

查询参数：
**fl** -- 逗号、空格分隔的要求返回的field。如果不指定，所有field都会被返回。
**includeDynamic** -- 默认是false。如果设为true，并且fl参数被指定或者fieldname路径参数被使用，匹配的动态field将被包含在response中并且被dynamicBase属性标识。如果不指定fl和fieldname，那么includeDynamic将被忽略。如果设为false，符合的动态field将不会被返回。
**showDefaults** -- 默认是false。如果设为true，那么所有缺省字段的属性将会被返回。如果设为false，那么只有明确设定的字段属性会被返回。

`curl http://localhost:8983/solr/gettingstarted/schema/fields?wt=json`

```json
{
    "fields": [
        {
            "indexed": true,
            "name": "_version_",
            "stored": true,
            "type": "long"
        },
        {
            "indexed": true,
            "name": "author",
            "stored": true,
            "type": "text_general"
        },
        {
            "indexed": true,
            "multiValued": true,
            "name": "cat",
            "stored": true,
            "type": "string"
        },
...
    ],
    "responseHeader": {
        "QTime": 1,
        "status": 0
    }
}
```

- 查询动态field
`GET /collection/schema/dynamicfields`
`GET /collection/schema/dynamicfields/name`

查询参数

**showDefaults** -- 默认为false，如果设为true，那么所有缺省属性都会被返回，如果false，那么只会返回显式赋值的属性。

`curl http://localhost:8983/solr/gettingstarted/schema/dynamicfields?wt=json`

```json
{
    "dynamicFields": [
        {
            "indexed": true,
            "name": "*_coordinate",
            "stored": false,
            "type": "tdouble"
        },
        {
            "multiValued": true,
            "name": "ignored_*",
            "type": "ignored"
        },
        {
            "name": "random_*",
            "type": "random"
        },
        {
            "indexed": true,
            "multiValued": true,
            "name": "attr_*",
            "stored": true,
            "type": "text_general"
        },
        {
            "indexed": true,
            "multiValued": true,
            "name": "*_txt",
            "stored": true,
            "type": "text_general"
        }
...
    ],
    "responseHeader": {
        "QTime": 1,
        "status": 0
    }
}
```

- 列出Copy field
`GET /collection/schema/copyfields`

参数：
source.fl -- 用逗号或空格来隔离出要返回的源字段，不设的话，会返回全部字段。
dest.fl -- 用逗号或空格来隔离出要返回的目标字段，不设的话，会返回全部字段。

`curl http://localhost:8983/solr/gettingstarted/schema/copyfields?wt=json`

```json
{
    "copyFields": [
        {
            "dest": "text",
            "source": "author"
        },
        {
            "dest": "text",
            "source": "cat"
        },
        {
            "dest": "text",
            "source": "content"
        },
        {
            "dest": "text",
            "source": "content_type"
        },
...
    ],
    "responseHeader": {
        "QTime": 3,
        "status": 0
    }
}
```

- 显示Schema名字
`GET /collection/schema/name`

`curl http://localhost:8983/solr/gettingstarted/schema/name?wt=json`

```json
{
  "responseHeader":{
    "status":0,
    "QTime":1},
  "name":"example"}
```

- 显示schema版本
`GET /collection/schema/version`

`curl http://localhost:8983/solr/gettingstarted/schema/version?wt=json`

```json
{
  "responseHeader":{
    "status":0,
    "QTime":2},
  "version":1.5}
```

- 列出UniqueKey
`GET /collection/schema/uniquekey`

`curl http://localhost:8983/solr/gettingstarted/schema/uniquekey?wt=json`

```json
{
  "responseHeader":{
    "status":0,
    "QTime":2},
  "uniqueKey":"id"}
```

- 显示全局Similarity
`GET /collection/schema/similarity`

`curl http://localhost:8983/solr/gettingstarted/schema/similarity?wt=json`

```json
{
  "responseHeader":{
    "status":0,
    "QTime":1},
  "similarity":{
    "class":"org.apache.solr.search.similarities.DefaultSimilarityFactory"}}
```
- 查询默认查询操作器
`GET /collection/schema/solrqueryparser/defaultoperator`

`curl http://localhost:8983/solr/gettingstarted/schema/solrqueryparser/defaultoperator?wt=json`

```json
{
  "responseHeader":{
    "status":0,
    "QTime":2},
  "defaultOperator":"OR"}
```





