---
title: Nginx学习笔记：二、进程和运行时
date: 2016-07-11 11:10:31
tags: Nginx 
---
## 进程
nginx包含一个master进程和一个或多个worker进程。
master进程的主要职责是读取和使用配置文件，同时管理worker进程。
worker进程用来处理用户请求。Nginx依赖OS的机制来有效的分发请求给worker进程。worker进程的数量定义位于nginx.conf配置文件上，它可以是固定的数目也可以自适应cpu的核心数。
定义worker进程的数母，主要参考几个方面，cpu的核心数、存储数据的硬盘数以及负载模式。当不清楚使用哪种策略时，设定为自适应cpu核心数("auto")是一个好的方法。
## 操作Nginx
### 信号
重载配置，你可以停止并重启nginx或者发送一个信号给master进程。使用-s发送信号给运行中的nginx。
`nginx -s signal`
signal的值可以为
quit
reload
reopen
stop

可以直接用kill命令来直接放松信号到master进程。master进程的ID默认情况下存在 /usr/local/nginx/logs 或者 /var/run 的nginx.pid 文件中。可以在nginx.conf中设置。
有两种方式来通过这些信号去控制 Nginx，第一是通过 kill – XXX <pid> 来控制 Nginx，其中 XXX 就是上表中列出的信号名。如果系统中只有一个 Nginx 进程，那也可以通过 killall 命令来完成，例如运行 killall – s HUP nginxPID 来让 Nginx 重新加载配置。
master进程有如下信号：
TERM, INT 	快速关闭程序，中止当前正在处理的请求
QUIT 	处理完当前请求后，关闭程序
HUP 	重新加载配置，并开启新的工作进程，关闭就的进程，此操作不会中断请求
USR1 	重新打开日志文件，用于切换日志，例如每天生成一个新的日志文件
USR2 	平滑升级可执行程序
WINCH 	从容关闭工作进程 

单独的worker进程也可以通过信号量来控制，虽然并不是必须的。
TERM, INT	快速关闭进程
QUIT	处理完当前请求后，关闭进程
USR1	重新打开日志文件，用于切换日志，例如每天生成一个新的日志文件
WINCH	调试异常终止（需要启用debug_points ）

### 更改配置
为了nginx能够重读配置文件，HUP信号将发送给master进程，master进程首先检查信号的有效性，然后使用新的配置，打开日志文件和心得监听socket。如果这步失败了，那么将回滚并使用旧的配置。如果成功了，会开启新的worker进程，并发送信息给旧的worker进程让它们优雅停机。旧的进程关闭监听socket并继续服务旧的客户端，当所有客户端请求被服务后，旧的worker进程会停机。

### 分割日志文件
为了分割日志文件，首先，它们需要被重命名。在那之后，需要向master进程发送USR1信号。master进程会重新打开所有当前打开的日志文件，并将它们分配给当前worker进程正在运行的无权限的用户作为所有者。在成功重新打开日志文件后，master进程会关闭所有打开的文件并发送消息给worker进程来重新打开文件。worker进程也需要立刻打开新文件并关闭旧文件。其结果是，旧文件可以立即适用于后处理，例如压缩。

### 升级
为了升级服务，需要使用新的可执行文件替换旧的。在USR2信号发送到master进程后，master进程首先会将pid文件名加上一个.oldbin后缀。/usr/local/nginx/logs/nginx.pid.oldbin。然后启动一个新的可执行文件来启动新的worker进程。

之后，所有的worker进程（新的和旧的）都继续接收请求，如果WINCH信号发送到了，master进程，那么将会通知worker进程去优雅停机。

经历一段时间后，只有新的worker进程可以处理请求。

需要注意的是，旧的master进程不会关闭监听socket，并且如果需要，它还可以重新启动。如果由于一些原因，新的可执行文件不能使用了，那么会出现以下两种情况中的一种：

- 发送HUP信号到旧的master进程，旧的master进程不重新读配置文件，但会启动一个新的worker进程。在那之后，发送QUIT到新的master进程，所有新的进程将优雅停止。
- 发送 TERM 信号到新的master进程。然后新的master进程会发送信息到它的worker进程，告诉他们立即停止（如果有新的进程因为某些原因未停止，那么会发送KILL信号来强制停止）。当新的master进程停止后，旧的master进程自动会启动新的worker进程。

如果新的master进程退出了，那么旧的，master进程会抛弃带有.oldbin的文件。

如果更新成功，那么旧的master进程会被发送QUIT信号，然后只有新的进程存活。

