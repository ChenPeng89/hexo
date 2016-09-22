---
title: Elasticsearch学习笔记：二、基本设置
date: 2016-09-22 09:49:19
tags: Elasticsearch
---
## 安装
### JAVA版本
es是由java开发的，运行也需要java环境。需要保证jdk至少是1.7及以上的。且仅支持Oracle'sJava和OpenJDK。在es的所有节点和客户端中，jdk的版本号应该是一致的。
官方推荐安装jdk8 update20及以后或者jdk8 update 55及以后。之前的版本有bug因此会容易丢失数据。es也会自动判断，如果是不好的版本，它会拒绝启动。
### 启动
在[下载](https://www.elastic.co/downloads/elasticsearch)最新的版本后，解压压缩包，然后就可以启动了：
```
$ bin/elasticsearch
```
### 作为守护进程启动
在*nix系统中，还可以作为守护进程启动：
```
$ bin/elasticsearch -d
```
### 指定PID
为了维护方便，还可以指定es的pid:
```
$ bin/elasticsearch -d -p pid 
$ kill `cat pid` 
```
- PID应该被写在名为pid的文件中。
- kill命令会发送一个term信号给pid文件中的PID。

## 配置
### 环境变量
通过脚本，es可以向JVM传递JAVA_OPT参数。其中，最重要的参数是-Xms和-Xmx，它们用来控制进程的内存大小。
大多数情况下，最好用ES_JAVA_OPTS环境变量来替代JAVA_OPTS，这样可以不影响其它jvm项目的运行。
ES_HEAP_SIZE环境变量用来设置分配给esjava进程的堆内存大小。它会吧最大值和最小值设为同一个值。当然，也可以自己指定最大值和最小值，使用ES_MIN_MEM(默认是256m) 和 ES_MAX_MEM(默认是1G)。
值得推荐的做法是将最大值和最小值设为相同的值，并且开启mlockall（之后会有介绍）。

### 系统设置
#### 文件描述符
确保要增加服务器中打开文件描述符的数量。设置为32K或者64是比较推荐的做法。
可以使用Nodes API查看文件描述符的大小：
```
curl localhost:9200/_nodes/stats/process?pretty
```

#### 虚拟内存
es默认使用 hybrid mmapfs / niofs 目录来存储index。默认情况下，os的限制的mmap数量可能过小，有可能会引起内存溢出。在Linux中，可以通过指令增加这个限制：
```
sysctl -w vm.max_map_count=262144
```
或者为了让这个设置永久生效，可以在/etc/sysctl.conf设置vm.max_map_count 这个参数。

#### 内存设置
大多数os会使用尽可能大的内存来做文件缓存，并将不用的应用程序内存换出，这就有可能将es的进程换出去了。换出这个动作对于es是非常影响效率的，并引起节点的不稳定，因此，应该尽量避免换出。
有以下三个解决方案：
- 禁止换出
  最简单的方案就是完全禁止换出操作。
  在Linux系统中，如果暂时禁止换出，可以使用 sudo swapoff -a 命令。如果想永久生效，可以在 /etc/fstab 文件中将带有 swap的行通通注释掉。
  在Windows中，可以通过 System Properties → Advanced → Performance → Advanced → Virtual memory 禁止分页文件来实现。

- 设置 swappiness
  第二个方案是将sysctl 中 vm.swappiness设为0。这个操作会降低内核换出操作的频率。除非系统处于不得不换出的状态，一般情况下都不会执行换出操作。
  注意，在内核版本 3.5-rc1及以上，swappiness为0会导致OOM杀掉进程而不是换出。你需要将swappiness设为1来允许在紧急情况下的换出。

- mlockall
  第三个方案是，在Linux中使用mlockall或者在Windows中使用VirtualLock，来锁住RAM中的进程地址空间，防止es内存被换出去，这个可以在 config/elasticsearch.yml 文件中：
  ```
  bootstrap.memory_lock: true
  ```
  在启动后，可以查看mlockall状态：
  ```
  curl http://localhost:9200/_nodes/process?pretty
  ```
  如果你看到mlockall是false，它意味着mlockall失败了。在Linuxl/Unix中，一般是因为es没有权限锁内存。这个在启动es前可以通过`ulimit -l unlimited as root` 来授权。
  另一个可能的原因是临时目录（一般是/tmp）被安装了noexec 选项，这可以通过指定一个新的临时文件目录来解决:
  ```
  ./bin/elasticsearch -Djna.tmpdir=/path/to/new/dir
  ```
### Elasticsearch设置
es配置文件可以在 ES_HOME/config 文件夹中找到。它包括两个文件，elasticsearch.yml 用于es的不同模块的设置，logging.yml 用于配置es的日志。

#### 路径
在生产环境中，一般会设置两个路径来存储数据和日志：
```
path:
  logs: /var/log/elasticsearch
  data: /var/data/elasticsearch
```

#### 集群名称
```
cluster:
  name: <NAME OF YOUR CLUSTER>
```
要保证你不会在不同的环境使用相同的集群名称，否则节点有可能加入到错误的集群中。你可以在生产环境用 logging-prod ，开发环境 logging-dev，分支环境使用 loggin-stage。

#### 节点名称
你可以为每个节点修改他们的节点名称以便于管理。
```
node:
  name: <NAME OF YOUR NODE>
```
hostname一般存在于服务器的HOSTNAME环境变量中，如果你的机器对于集群运行一个单独的es节点，你可以设置节点名称为hostname
```
node:
  name: ${HOSTNAME}
```

#### 设置配置文件格式
在es中，所有的设置都被包围在 anmespaced中。例如，上面这些设置都在node.name中。这意味着可以支持其他格式的配置文件，比如JSON。如果使用JSON，可以将elasticsearch.yml修改为elasticsearch.json。相应的，里面的设置格式为：
```
{
    "network" : {
        "host" : "10.0.0.4"
    }
}
```
### Index设置
集群中的index可以维护自己的设置。例如，可以将创建index的刷新间隔时间设置为5s。
```
$ curl -XPUT http://localhost:9200/kimchy/ -d \
'
index:
    refresh_interval: 5s
'
```

Index级别的设置也可以在node级别中设置:
```
index :
    refresh_interval: 5s
```
如果index和node都设置了，那么指定的index会覆盖node的设置。
### 日志
es内部使用了log4j。配置文件是config/logging.yml。也支持JSON和properties文件。

#### 过期行为的日志
除了常规的日志，es还能记录过期行为的日志。这对于迁移是一个很大的优势。默认它是关闭的，在 config/logging.yml 开启：
```
deprecation: DEBUG, deprecation_log_file
```
它会在日志目录下创建一个每天滚动的日志文件。定期检查这个文件，针对于当你想要升级到一个新的大版本。
