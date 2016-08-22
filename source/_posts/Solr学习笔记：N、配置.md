---
title: Solr学习笔记：N、配置
date: 2016-06-13 17:37:25
tags: Solr
---
## 目录结构
Solr服务在运行时需要访问它的home目录，home目录包含了Solr的重要配置和索引信息。
```
/solr-6.0.1/server/solr
<solr-home-directory>/
   solr.xml
   core_name1/
      core.properties
      conf/
         solrconfig.xml
         managed-schema
      data/
   core_name2/
      core.properties
      conf/
         solrconfig.xml
         managed-schema
      data/
```

##solr.xml
其中，solr.xml包含了Solr服务的配置。
你可以在home目录或者zookeeper上找到solr.xml文件，默认情况下，它是这样的：
```
<solr>
 
  <solrcloud>
    <str name="host">${host:}</str>
    <int name="hostPort">${jetty.port:8983}</int>
    <str name="hostContext">${hostContext:solr}</str>
    <int name="zkClientTimeout">${zkClientTimeout:15000}</int>
    <bool name="genericCoreNodeNames">${genericCoreNodeNames:true}</bool>
  </solrcloud>
 
  <shardHandlerFactory name="shardHandlerFactory"
    class="HttpShardHandlerFactory">
    <int name="socketTimeout">${socketTimeout:0}</int>
    <int name="connTimeout">${connTimeout:0}</int>
  </shardHandlerFactory>
 
</solr>
```

从上面的配置可以看到，默认情况下，solr配置了<solrCloud>，但是这并不意味着solr已经运行在cloud模式下，除非在启动时指定 -DzkHost 或者 -DzkRun ， 否则它会被忽略。

下面介绍详细参数：

- `<solr>`标签
`<solr>`是文件的根标签，下面列出它的子节点

1. `adminHandler` -- 定义了处理用户请求的Hnadler。如果不设置，则使用默认的CoreAdminHandler，如果要自定义handler，则需要继承CoreAdminHandler。例如：`<str name="adminHandler">com.myorg.MyAdminHandler</str>`。
2. `collectionsHandler` -- 同上，用于自定义CollectionsHandler 的实现。
3. `infoHandler` -- 同上，用于自定义infoHandler的实现。
4. `coreLoadThreads` -- 指定用来并行加载core的线程数。
5. `coreRootDirectory` -- 指定core的根目录，默认为SOLR_HOME。
6. `managementPath` -- 当前没有作用。
7. `sharedLib` -- 指定一个被所有core共享的lib的目录。这个目录下的所有jar都将被添加到Solr插件搜索的路径中。这个路径是和Solr的顶层容器Solr Home相关的。用户自定义的handler也可以放在此目录下。
8. `shareSchema` -- 如果这个参数设为 true，那么多个指向同一个Schema资源文件的core将引用同样的 IndexSchema对象。共享IndexSchema对象将使加载core变得更快。如果你要使用这个功能，请确保没有 特定的针对某个core的属性 用在你的Schema文件中。
9. `transientCacheSize` -- 定义有多少个transient = true 的core能在将最近最少使用的core换成新的core之前加载。
10. `configSetBaseDir` -- 定义core的configsets存放的目录路径。默认是SOLR_HOME/configsets。

- `<solrcloud>`标签
这个标签定义了和SolrCloud相关的几个参数。除非在启动时使用了 -DzkRun 或者 -DzkHost ， 否则这一部分将被忽略。

1. `distribUpdateConnTimeout` -- 用于设置集群内部更新时潜在的“connTimeout”。
2. `distribUpdateSoTimeout` -- 用于设置集群内部更新时潜在的“socketTimeout”。
3. `host` -- Solr用来访问core的hostname。
4. `hostContext` -- Solr web服务中Servlet的上下文路径。
5. `hostPort` -- Solr用于访问core的端口号。在默认的solr.xml中，一般设为${jetty.port:8983}，它将使用在Jetty中定义的Solr端口，否则会使用8983。
6. `leaderVoteWait` -- 当SolrCloud启动时，假设在没有宕机的情况下，每个Solr节点等待所有已知分片的备份时间。
7. `leaderConflictResolveWait` -- 当试图为一个分片选举一个leader时，这个属性设置了一个备份等待处理冲突的最长时间。临时的状态信息冲突可能发生在轮流的重启动过程中，尤其是当运行监视主机的节点重启的时候。典型地，默认值180000（millis）足够实现冲突的解决。当你的SolrCloud中有成百上千个小的collection的时候，你可能需要增加这个值。
8. `zkClientTimeout` -- 连接ZooKeeper服务的超时时间。
9. `zkHost` -- 用来管理Solr集群状态的ZooKeeper的URL。
10. `genericCoreNodeNames` -- 如果设置为true，节点的名称就不取决于节点的地址，而是一个由Core标识的通用的名字。当不同的机器接管这一服务时，这样的名字会更容易被理解。
11. `zkCredentialsProvider &  zkACLProvider` -- 当使用 ZooKeeper Access Control 时可选的参数。

- `<logging>`标签
1. `class` -- 用于实现日志的类。对应的JAR必须能被solr有效使用，可以考虑在solrconfig.xml中使用<lib>指出。
2. `enabled` -- true/false，是否允许使用日志。

- `<logging><watcher>`标签
1. `size` -- 缓存日志的数量。
2. `threshold` -- 日志等级，高于该等级的日志才能被记录。

- `<shardHandlerFactory>`标签
指定分片handler。
`<shardHandlerFactory name="ShardHandlerFactory" class="qualified.class.name">`。
由于是自定义的实现方式，因此，它的子元素是和具体实现相关的。

- 在solr.xml中设置jvm参数
Solr支持在solr.xml中替换JVM运行参数，它允许在运行时指定多重配置。具体语法是： ${propertyname[:option default value]}。它允许定义默认的状态，这个状态可以在solr启动时被覆盖。如果没有指定一个默认值，那么这个属性必须在运行时指定，否则在解析solr.xml文件时会产生错误。
设置JVM参数一般在启动JVM时使用 -D 标识。这个可以在solr.xml中作为变量使用。
例如：在下面展示的solr.xml中，使用“java -DsocketTimeout=1000 -jar start.jar”启动solr会使用1000ms覆盖HttpShardHandlerFactory socket的超时属性，取代原本默认的值“0”——但是，connTimeout选项仍会使用默认呢的值“0”。
```
<solr>
  <shardHandlerFactory name="shardHandlerFactory"
                       class="HttpShardHandlerFactory">
    <int name="socketTimeout">${socketTimeout:0}</int>
    <int name="connTimeout">${connTimeout:0}</int>
  </shardHandlerFactory>
</solr>
```
## core.properties
core.properties文件是一个简单的Java属性文件，其中每一行是一个key=value对，比如：name=core1。注意这里不支持引用。

- core.properties文件位置
core的配置文件core.properties一般位于solr.home的子目录下。Solr没有限制core.properties在目录树的深度，也没限制可定义的core数。core可以定义在目录树任何地方，除了已经存在的core目录下。
```
./cores/core1/core.properties
./cores/core1/coremore/core5/core.properties
```
这个例子中，只有core1是有效的。

下面的例子是合法的：
```
./cores/somecores/core1/core.properties
./cores/somecores/core2/core.properties
./cores/othercores/core3/core.properties
./cores/extracores/deepertree/core4/core.properties
```
Solr可以分成多个core，每一个core有自己的配置和索引。core可以服务于一个单个应用或者许多个不同的应用，但是他们都被一个统一的管理员界面来管理。Solr支持热部署，即在运行时增加、删除甚至替换core都不用停止或者重启Solr。
core.properties文件可以是空的。假设core.properties位于./cores/core目录下，但是是空的，那么，core的名称将为core1，那么core1实例路径为../cores.core1 ， data路径为 ../cores/core1/data。

- 配置参数

1. `name` -- SolrCore的名称，可以在CoreAdminHandler上运行命令时使用它查找SolrCore。
2. `config` -- 当前core的配置文件名称，默认为 solrconfig.xml。
3. `schema` -- 当前core的模式文件的名字，默认是schema.xml。
4. `dataDir` -- 当前core的data目录路径，可以是绝对路径，也可以是相对于 instanceDir 的相对路径。
5. `configSet` -- 定义configset的名称，可以用它来配置core。
6. `properties` -- 当前core的properties文件名称，值可以是绝对路径也可以是相对于 instanceDir 的相对路径。
7. `transient` -- 如果设为true，当Solr达到transientCacheSize上限时，可能会卸载当前core。默认值是false。最近最少被使用的core最先进行卸载。在SolrCloud模式下，不推荐设为true。
8. `loadOnStartup` -- 如果设为true（默认值），当前core会在solr启动时被加载。在SolrCloud模式下，不推荐设为false。
9. `coreNodeName` -- 仅应用在SolrCloud模式下，它是coreNode的唯一标识符。默认情况下，它是自动生成的，但是这个属性可以允许你手动分配一个新的core代替已经存在的，使用相同的coreNodeName指定一个新的机器，那么它会取代原有的SolrCore。一般用于更换有故障的机器。
10. `uLogDir` -- 这个core（SolrCloud）的更新日志的绝对或相对路径。
11. `shard` -- 指定这个core上的shard。
12. `collection` -- 这个core所属的collection的名字。
13. `roles` -- 一个未来将会用于SolrCloud的参数，或者用户用来标记在自己使用的节点的方法。

## solrconfig.xml
solrconfig.xml是Solr中参数最多的配置文件。当配置Solr时，你通常要使用solrconfig.xml来配置，不管是直接使用solrconfig.xml还是通过ConfigAPI创建Configuration Overlays来覆盖solrconfig.xml中的配置。

在solrconfig.xml中，可以配置以下几个重要的属性：
1. request handlers ， 用于在Solr中处理请求，例如：向索引中添加文档或者为请求返回查询结果。
2. liseners ，用于监听特定查询事件。listener能用做为触发器，比如，使用cache来处理常见查询。
3. Request Dispatcher，用于处理HTTP请求。
4. Admin Web 界面。
5. 和复制备份相关的参数。

solrconfig.xml文件位于每个collection的conf/目录下。Solr也提供了一些带有注释且包含多种实用配置的案例，可以在server/solr/configsets/ 找到。

- 设置DataDir
指定data的目录，可以是绝对路径也可以是针对instance的相对路径。
`<dataDir>/var/data/solr/</dataDir>`

- 设置索引的DirectoryFactory
默认情况下，solr.StandardDirectoryFactory是基于当前文件系统的，它会选择出当前JVM和平台最好的实现。你可以强制指定 solr.MMapDirectoryFactory, solr.NIOFSDirectoryFactory, 或者 solr.SimpleFSDirectoryFactory作为实现。
```
<directoryFactory name="DirectoryFactory"
                  class="${solr.directoryFactory:solr.StandardDirectoryFactory}"/>
```
solr.RAMDirectoryFactory是基于内存的，因此是不能持久化的，而且不能复制。使用这个DirectoryFactory将索引储存在RAM中。
```
<directoryFactory class="org.apache.solr.core.RAMDirectoryFactory"/>
```

- lib目录
solrconfig.xml可以用lib标签来加载插件。
```
<lib dir="../../../contrib/extraction/lib" regex=".*\.jar" />
<lib dir="../../../dist/" regex="solr-cell-\d.*\.jar" />
 
<lib dir="../../../contrib/clustering/lib/" regex=".*\.jar" />
<lib dir="../../../dist/" regex="solr-clustering-\d.*\.jar" />
 
<lib dir="../../../contrib/langid/lib/" regex=".*\.jar" />
<lib dir="../../../dist/" regex="solr-langid-\d.*\.jar" />
 
<lib dir="../../../contrib/velocity/lib" regex=".*\.jar" />
<lib dir="../../../dist/" regex="solr-velocity-\d.*\.jar" />
```
- Schema Factory
Solr的Schema API允许远程客户端通过REST风格的接口访问和修改Schema信息。
Solr API的"read"功能能支持所有Schema类型，而使用程序修改Schema则需要依靠`<schemaFactory>`。

```
<!-- An example of Solr's implicit default behavior if no
     no schemaFactory is explicitly defined.
-->
 <schemaFactory class="ManagedIndexSchemaFactory">
   <bool name="mutable">true</bool>
   <str name="managedSchemaResourceName">managed-schema</str>
 </schemaFactory>
```
1. `mutable` -- 控制是否能改变Schema中的数据。当允许使用Schema API修改数据时，必须设为true。
2. `managedSchemaResourceName` -- 是一个可选的参数，默认为“managed-schema” ， 用来设置schema文件的名称，不可以设置为“schema.xml”。

以上的是默认设置，你可以使用Schema API修改schema，然后将mutable设为false，这样，就可以锁上schema并且保护修改。

- Classic schema.xml
为使用托管模式的另一种方法是显式配置ClassicIndexSchemaFactory。ClassicIndexSchemaFactory需要使用schema.xml的配置，并且在运行时不允许程序修改Schema。schema.xml必须手动修改并且仅当Solr集合加载时加载。
`<schemaFactory class="ClassicIndexSchemaFactory"/>`

**将schema.xml转换为Managed Schema**
如果你当前的Solr集合使用ClassicIndexSchemaFactory，