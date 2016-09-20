---
title: Redis持久化策略
date: 2016-07-07 10:24:32
tags: Redis
---
Redis提供了两种持久化策略，一种是快照方式（point-in-time snapshot），另一种是只追加文件（append-only file）方式。

## 快照方式
快照方式会在某个时刻将所有数据都写入到硬盘中。用户可以根据Redis的配置命令`config get dir`和`config get dbfilename`来知道快照文件写入的路径和文件名。

创建快照的方式有以下几种：
- 客户端发送  `bgsave` 命令（windows不支持此命令），Redis会调用fork创建一个进程来进行备份操作，父进程继续接收执行命令。
- 客户端发送  `save` 命令，Redis会进行备份操作，在备份完成之前，不相应其它命令请求。一般不使用此命令，只有在没有足够内存去执行`bgsave`情况下才使用此命令。
- 在配置中设置了 `save` 选项，比如设置为 `save 60 10000` ， 当60s内有10000次写入则触发bgsave。
- 当使用 `shutdown` 关闭Redis时，会先执行`save`命令，并阻塞所有客户端。
- 当Redis连接另一Redis，并向对方发送`sync`开始一次复制命令，如果redis没有正在执行`bgsave` 或 没有刚刚执行完 `bgsave`，那么会执行一次`bgsave`。

缺点：
如果在进行下一次备份的时候服务器crash了，那么将丢失上次备份到现在的所有记录。

## AOF
aof会将被执行的写命令写到aof文件的末尾，每次恢复的时候直接执行aof文件中的写命令就可以了。用户可以在`appendonly `配置中打开aof。`appendfsync`可以控制aof的频率。`always`是每次写命令都要同步到硬盘，这样会严重降低redis的速度。`everysec`每秒进行一次同步，显式将多个写命令同步到硬盘，一般使用这个选项。`no`由OS来确定何时同步。

由于aof文件会越来越大，一是会占用过大的硬盘空间，二是数据恢复会需要很长时间。为了解决这个问题，redis提供了一个`bgrewriteaof`命令创建一个子进程来移除aof的冗余命令并重写aof文件。redis还提供了`auto-aof-rewrite-percentage`和`auto-aof-rewrite-min-size`来自动执行`bgrewriteaof`。当设置了 `auto-aof-rewrite-percentage 100`和`auto-aof-rewrite-min-size 64mb`时，并启动了aof，当aof文件大于64mb 并且aof文件比上一次重写大了至少1倍时，redis将会执行`bgrewriteaof`。
