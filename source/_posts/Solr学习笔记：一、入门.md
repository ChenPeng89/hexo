---
title: Solr学习笔记：一、入门
date: 2016-06-13 10:24:25
tags: Solr
---
## 简介
Solr是一个用Java语言开发的，基于Lucene的开源全文搜索服务器。它包含了lucene提供的索引和搜索、拼写检查、命中结果高亮和先进的分析/标记等功能，同时，它还提供了层面搜索（有些类似于group by 的功能）等高级功能，并简化了开发难度。

## 安装
前文说了，Solr是基于Java开发的，因此需要在JRE环境下运行。下面介绍在Linux下安装Solr的步骤。

Solr要求JRE版本为1.8或者以上。如果不确定自己的机器是否安装jre或者jre的版本，可以使用下面的命令看一下：
```
$ java -version
  java version "1.8.0_92"
  Java(TM) SE Runtime Environment (build 1.8.0_92-b14)
  Java HotSpot(TM) 64-Bit Server VM (build 25.92-b14, mixed mode)
```
如果没安装或者版本低于1.8的话，先安装JRE。

然后下载Solr，[**solr-6.0.1.tgz**](http://lucene.apache.org/solr/)。

下载后解压之。
`tar zxf solr-6.0.1.tgz`

解压成功后就可以准备运行了。

## 运行

**开启**

Solr的运行很简单。首先进入bin目录。
`cd bin`
然后启动Solr。
`$ ./solr start`
然后看到下面的提示，就说明运行成功了。
```
./solr start
Waiting up to 30 seconds to see Solr running on port 8983 [\]  
Started Solr server on port 8983 (pid=92928). Happy searching!
```
这时可以看到，Solr默认的绑定端口时8983，当然，你也可以自定义绑定的端口号：
`./solr start -p 8984`

Solr默认是运行在后台的服务，你也可以自定义让它在前台显示：
`./solr start -f`

Solr提供了一些有用的例子来帮助大家学习它的关键的功能，可以通过 -e 来指定运行的配置：
`./solr -e techproducts`

**停止**

当Solr运行在前台时，你就可以通过Ctrl+C来停止它。如果是后台运行，则需要用
`./solr stop`

**检测Solr运行状态**
如果你不确定本地是否运行了solr，使用下面命令来检测：
`./solr status`

**更多功能，可以使用 `./solr -help`来查询。**

## 使用
Solr提供了管理后台，可以通过登录 http://localhost:8983/solr/ 来查看。
![](http://i.imgur.com/aNcuJp0.png)

**创建一个核心**
核心，有些类似于数据库的中的库，在solr中将文档存到核心的目录下。如果之前你没有使用solr提供的例子来启动的话，我们就需要创建一个新的核心来进行接下来的工作。
`./solr create -c gettingstarted`

查看具体配置，可以这样：
`/solr create -help`


**添加文档**
Solr通过查询文档来实现搜索功能。Solr维护了一个有内容结构的概要，但是没有文档的话，Solr什么也查不到，所以，在使用Solr前，要先加入有搜索内容的文档。

Solr中自带的文档来自于 example/ 的子目录下。

在 bin/ 目录下，有一个 post 的命令行工具，可以通过它来将不同类别的内容发送给Solr，包括Solr原生的XML、JSON、CSV等，具体的可以通过 `./post -help`来了解。

执行命令：
`./post -c gettingstarted example/exampledocs/*.xml`
这就将exampledocs目录中的xml文件导入到了Solr中。

**查询**
接下来就可以使用查询了。最简单的方式是使用URL并包含查询参数。例如：
http://localhost:8983/solr/gettingstarted/select?q=video

查询所有文档中数据段有 “video” 的数据。
查询结果包含两部分，一部分是响应头信息（responseHeader），另一部分是响应主体信息（result），包含了查询的数据结果。Solr可以将结果输出为 XML，JSON,PHP,Ruby和用户自定义的数据格式。

你也可以指定返回的数据字段：
http://localhost:8983/solr/gettingstarted/select?q=video&fl=id,name,price

或者返回字段中的字段数值范围：
http://localhost:8983/solr/gettingstarted/select?q=price:0 TO 400&fl=id,name,price

层面搜索是Solr中一个重要的功能，使用它需要在参数中加上 facet=true&facet.field=xxx：
http://localhost:8983/solr/gettingstarted/select?q=price:[0 TO 400]&fl=id,name,price&facet=true&facet.field=cat

也可以缩小查询结果范围
http://localhost:8983/solr/gettingstarted/select?q=price:0 TO 400&fl=id,name,price&facet=true&facet.field=cat&fq=cat:software

## Solr 脚本
Solr提供了一个脚本，位于 bin/solr ，可以用来开启和停止Solr，创建删除Solr节点或集合，检查Solr状态和配置分片。

- 启动或重启
`./solr start  [option]`
`./solt start -help`
`./solr restart [option]`
`./solt restart -help`

**参数**
1. `-a "<string>"` -- 使用补充的JVM参数开启Solr，例如以-X开头的，如果你传以-D开头的JVM参数，那么可以省略 -a 选项。
2. `-cloud` -- 以SolrCloud模式启动Solr，并访问Solr内置的zookeeper。这个选项可以省略为 -c 。如果你想运行完整版的zookeeper代替solr内置的，那么，可以使用 -z 参数。
3. `-d <dir>` -- 定义server的目录，默认是 $SOLR_HOME/server。一般不会覆盖这个选项。如果在同一个host上运行多个solr实例，普遍的会用 -s 选项来使用同一个server目录，并使用唯一一个Solr home目录。
4. `-e <name>` -- 使用一个样例配置来启动Solr，这些例子会帮助你更快的学习Solr，并使用指定的功能。 可用的为：
cloud
techproducts
dih
schemaless

5. `-f` -- 在前端启动Solr，与 -e 不能同时使用。
6. `-h <hostname>` -- 使用定义的host启动solr。如果不指定，默认是localhost。
7. `-m <memory>` -- 设置JVM堆内存。`bin/solr start -m 1g`。
8. `-noprompt` -- 启动Solr时忽略其它选项。使用所有默认值可能会带来副作用。
例如，当使用cloud 样例时，一个交互会话会引导你通过几个选项配置你的SolrCloud集群。如果你想要所有都是默认的，那么你可以简单的添加 -noprompt选项在你的请求中。
9. `-p <port>` -- 在指定的端口上启动Solr，默认为 8983。
10. `-s <dir>` -- 设置solr的solr home目录。Solr将会在这个目录下创建core。它允许你运行多个Solr实例在同一个host上，同时重用同一个server目录时设置 -d 参数。如果设置它，需要在目录下包含solr.xml文件，除非在zookeeper上已经存在了solr.xml。默认值是 server/solr。
当使用 -e 时，这个选项会被忽略。
11. `-V` -- 启动Solr时，在启动脚本上显示详细信息。
12. `-z <zkHost>` -- 使用zookeeper 连接字符串启动solr。这个选项只在使用 -c 选项时起作用，来启动solrcloud模式中的solr。如果这个选项没设置，那么会启动内置的zookeeper实例并用它对solrcloud进行操作。 `bin/solr start -c -z server1:2181,server2:2181`。

**SolrCloudMode**
当使用zookeeper连接字符串启动时，那么会连接到zookeeper并加入到集群中。否则，会使用内置的zookeeper，端口号为 solr运行的端口号 + 1000。
**<font color="red">注意：如果你的zookeeper连接字符串是一个改变了根目录的，比如： localhost:2181/solr ，那么你需要在使用 bin/solr 启动脚本启动SolrCloud之前启动/solr znode。要做到这一点，你需要使用Solr附带的zkcli.sh脚本。例如：</font>**
`server/scripts/cloud-scripts/zkcli.sh -zkhost localhost:2181/solr -cmd bootstrap -solrhome server/solr`。

**使用Solr提供的样例**
**cloud** -- 样例会在单机上启动 1-4 个solrcloud节点，启动时，交互会话将引导你选择 configsets 、集群节点数，端口号和创建的collection 名称。当使用这个样例时，可以从 $SOLR_HOME/server/solr/configsets 选择合适的configsets。

**techproducts** -- 这个样例运行在单机模式下，包含了文档样例。样例位于 $SOLR_HOME/example/exampledocs 。configsets位于$SOLR_HOME/server/solr/configsets/sample_techproducts_configs。

**dih** -- 这个样例运行在单机模式下，使用的是 DataImportHandler (DIH) ，通过不同的dataconfig.xml来支持多种不同类型的数据。configset用来设置DIH，它位于$SOLR_HOME/example/example-DIH/solr/conf。

**schemaless** -- 这个样例运行在单机模式下，使用了一个 管理模式 ，并提供了一个非常小的预定义 schema。Solr将会运行在Schemaless Mode下，Solr会动态创建字段并猜测收到文件中字段的类型。configset位于 $SOLR_HOME/server/solr/configsets/data_driven_schema_configs。

- 停止

Stop命令会向Solr节点发送STOP请求，并优雅停机。命令会等待Solr节点5s来优雅停机，然后会直接强迫kill进程。

`bin/solr stop [options]`
`bin/solr stop -help`

**参数**
**-p <port>** -- 停止运行的Solr节点。如果你运行了多个Solr或者SolrCloud模式下，你需要指定不同的端口或者使用-all条件。bin/solr stop -p 8983

**-all** -- 停止所有有有效PID的Solr实例。bin/solr stop -all

**-k <key>** -- 用于停止的Key，用于防止无意中误操作停止Solr，默认的key是solrrocks。bin/solr stop -k solrrocks

- 信息相关

**版本**
`$ bin/solr version`

**状态**

运行后会以JSON格式显示所有Solr节点。状态命令使用SOLR_PID_DIR环境变量来定位进程ID文件并查找运行中的实例，SOLR_PID_DIR变量默认位于bin目录下。

`bin/solr status`

**健康监测**
当使用SolrCloud模式下，健康监测命令会产生JSON格式的报告。报告会提供每一个分片备份的状态，包括提交文档数量和它当前状态。

`bin/solr healthcheck [options]`

`bin/solr healthcheck -help`

`-c <collection>` -- 运行指定名称集合的健康监测。 bin/solr healthcheck -c gettingstarted

`-z <zkhost>` -- ZooKeeper 连接字符串，默认是 localhost:9983。如果在其它端口上运行Solr，需要指定ZooKeeper的连接字符串。默认情况下，它是Solr的端口号+1000。

- 集合和Core
bin/solr 脚本可以帮助你创建新的collection（在SolrCloud模式下）或者 core（在独立模式下），或者删除集合。

**Create**
<font color="red">
注意：使用create命令前要注意你需要是启动Solr的用户。如果你使用的是Linux/Unix安装脚本，那么一般使用名为 solr 的用户来执行。如果Solr运行在solr用户中但是你用root用户创建core，那么Solr将不能写入开始脚本创建的目录。
如果运行在SolrCloud模式下，这就没什么问题。在cloud模式下，所有配置都存在ZooKeeper中，并且create脚本不需要创建目录或者拷贝配置文件。Solr自身会创建必要的目录。
</font>

create命令会自动检测Solr是否运行在独立模式下，会相应的创建core或者集合。

`bin/solr create options`

`bin/solr create -help`


`-c <name>` -- 创建core或者collection。  bin/solr create -c mycollection

`-d <confdir>` -- 配置文件路径。默认是data_driven_schema_configs。bin/solr create -d basic_configs

`-n <configName>` -- 配置。bin/solr create -n basic

`-p <port>` -- 在指定端口号创建Solr实例。bin/solr create -p 8983

`-s <shards> -shards` -- 集合中分片的数目，默认是1，只运行在SolrCloud模式下。bin/solr create -s 2

`-rf <replicas> -replicationFactor` -- 集合中每个文档的拷贝数量。默认是1。bin/solr create -rf 2

**Delete**
delete命令检测到运行的Solr，并删除指定core或者collection。
`bin/solr delete [options]`

`bin/solr delete -help`

如果在SolrCloud模式下，delete命令会检查指定删除的collection的配置是否呗其它collection使用。如果没有，那么将从ZooKeeper中删除。

`-c <name>` -- 删除指定名称的core或collection。bin/solr delete -c mycoll

`-deleteConfig <true|false>` -- 从ZooKeeper中删除配置目录。默认是true。如果配置被别的collection使用，那么设为true也不会被删除。bin/solr delete -deleteConfig false

`-p <port>` -- 删除指定端口号的solr实例。默认的，会删除当前运行的实例。bin/solr delete -p 8983

- ZooKeeper相关操作
bin/solr 脚本也允许一些操作来影响ZooKeeper。

`bin/solr zk [options]`

`bin/solr zk -help`

在初始化ZooKeeper前，Solr需要至少启动一次。当ZooKeeper初始化后，Solr不需要再在任何节点上运行这些命令。

**上传配置**
使用如下ZooKeeper的子命令来上传预设值的配置集合/自定义配置集合。

`-upconfig` -- 从本地上传配置集合到ZooKeeper。	-upconfig

`-n <name>` -- ZooKeeper上的配置名。命令将会上传配置到"configs" ZooKeeper 节点，并命名为指定的名字。-n myconfig

`-d <configset dir>` -- 指定上传配置的路径。它需要有一个conf路径，并且下面包括solrconfig.xml等。-d directory_under_configsets
-d /absolute/path/to/configset/source

`-z <zkHost>` -- ZooKeeper 连接字符串。-z 123.321.23.43:2181


样例： `bin/solr zk -upconfig -z 111.222.333.444:2181 -n mynewconfig -d /path/to/configset`

上面的例子不会自动起作用，它只会将配置上传到ZooKeeper，可以使用Collections API的RELOAD相关方法使它生效。

**下载配置集**
`-downconfig` -- 将ZooKeeper上的配置下载到本地。-downconfig

`-n <name>` -- 需要下载的配置集的名，Admin UI>>Cloud>>tree>>configs node下列出了所有可用配置集。-n myconfig 

`-d <configset dir>` -- 将下载的配置集存到哪个目录。-d directory_under_configsets -d /absolute/path/to/configset/destination

`-z <zkHost>` -- ZooKeeper 连接字符串。-z 123.321.23.43:2181

样例： 
`bin/solr zk -downconfig -z 111.222.333.444:2181 -n mynewconfig -d /path/to/configset` 