---
title: Nginx学习笔记：三、基本功能
date: 2016-07-11 15:59:00
tags: Nginx
---
## Web服务
### 设置虚拟服务器
nginx配置文件中至少需要一个定义了虚拟服务器的server块。当nginx处理请求时，它会首先选择虚拟服务器来处理请求。
虚拟服务器定义在http上下文的server块中。
```
http {
    server {
        # Server configuration
    }
}
```
可以在http上下文中定义多个server块。
server块通常包含一个listen指令来监听指定的ip的端口。支持ipv4/ipv6，ipv6的地址需要用（）包裹起来。
```
server {
    listen 127.0.0.1:8080;
    # The rest of server configuration
}
```
如果省略端口号，则使用标准端口号。如果省略ip，那么服务器将监听所有地址。如果省略listen指令，那么标准的端口号是80/tcp，默认的端口号是8000/tcp这取决于root的权限。

server_name 参数用来指定接受访问的host头中的相应参数。它的值可以是一个通配符。如果没有匹配的，那么将路由到默认的服务器上来处理。

### 配置location
nginx可以将请求路由到不同的代理或者服务器上，基于请求的uri。这些都是依靠定义在server块中的localtion块来决定的。

```
location /some/path/ {
    root html;
    index  index.html  index.htm;
}
```
有两种类型的参数：前缀字符串和正则表达式。对于一个请求URI，他必须满足匹配前缀字符串。

正则表达式前面增加 ~ 表示区分大小写，~* 表示不区分大小写。下面的例子匹配所有带有.html或者.htm的URI。
```
location ~ \.html? {
    ...
}
```

**为了找到最匹配URI的location，nginx会首先搜索前缀字符串，然后再搜索正则表达式。**


nginx的处理逻辑为：
1. 尝试匹配所有前缀字符串。
2. = 修饰符定义了一个URI和前缀字符串的精确匹配。如果发现精确匹配,搜索停止。
3. 如果有^~修饰符，则最先考虑最长前缀字符串匹配,正则表达式不检查。
4. 存储最长前缀匹配字符串。
5. 尝试匹配正则表达式。
6. 从第一个匹配的正则表达式跳出递归，并使用相应的location。
7. 如果没有匹配的正则表达式，使用存储的最长匹配的前缀表达式（第4步）的location。

location块中包含如何处理的逻辑，要么转到一个静态文件，要么将请求转给一个代理服务器。
```
server {
    location /images/ {
        root /data;
    }

    location / {
        proxy_pass http://www.example.com;
    }
}
```
### 使用变量
nginx允许在配置文件中使用自定义的变量。
`set $variable value`
`map string $variable { ... }`
`geo [$address] $variable { ... }`

```
http {
 geo $arg_boy $ttlsa_com {
        default 0;
        127.0.0.1/24 24;
        127.0.0.1/32 32;
        8.8.8.8 2;
}
 server {
        listen       8080;
        server_name  test.ttlsa.com;
 
        location /hello {
 default_type text/plain;
 echo $ttlsa_com;
 echo $arg_boy;
 }
 }
}
# curl 127.0.0.1:8080/hello?boy=127.0.0.1
32
127.0.0.1
# curl 127.0.0.1:8080/hello?boy=127.0.0.12
24
127.0.0.12
```
### 返回指定状态码
一些网站URI需要立即返回指定的错误代码或重定向代码，例如，如果一个页面被暂时或永久的移除，那么，最简单的方法是直接返回相应的代码。
```
location /wrong/url {
    return 404;
}
```
return 的用法
```
return code [text];
return code URL;
return URL;
```
return 指令可以存在于location和server块中。

### 重写URI请求
一个请求的URI可以在处理请求过程中被重写多次，它包括一个可选的和两个必须的参数。第一个必须的参数是请求URI必须满足的正则表达式。第二个参数是用来替换请求URI的URI。可选的参数是一个标志位，它能停止进一步的重写处理或者发送重定向（301或302）。
```
location /users/ {
    rewrite ^/users/(.*)$ /show?user=$1 break;
}
```

rewrite指令可以存在于server和location块中。在server块中，如果server的上下文被选中，那么server块中的rewrite只会执行一次。

rewrite指令用法：
```
rewrite regex replacement [flag];
```
flag 有以下可选值：
last: 重新请求搜索是否还有匹配的locaton。
break: 不再搜索是否还有匹配的location。
redirect：返回一个暂时的重定向代码 ， 302，使用一个不以"http://"或"https://"开头的代替的字符串。
permanent： 返回一个永久的重定向代码，301.

### 重写HTTP返回
有时候你需要重写或者修改HTTP返回中的内容，把一个字符串替换成其它的。那么，你需要使用sub_filter指令来定义重写。这个指令支持变量和链操作来使复杂的操作变得简单。
sub_filter的替换匹配是不区分大小写的。
```
server {
    listen       80;
    server_name  www.github.com;
 
    root /data/site/www.github.com;    
 
    location / {
        sub_filter  github 'GIT';
        sub_filter_types text/html;
        sub_filter_once on;
    }
}
```
上面的例子将返回数据中的github替换为GIT，然后由于sub_filter_once on ， 所以只替换了一次。结果如下：
```
# curl www.github.com/2013/10/20131001_sub1.html           
welcome to GIT!
github TEAM!
```
sub_filter 可以存在于 http, server, location 块。

### 处理异常
使用error_page 指令，可以为特定的错误码自定义错误页面。
```
error_page 404 /404.html;
```
下面的例子中，将404转换为301，并重定向到http:/example.com/new/path.html。
```
location /old/path.html {
    error_page 404 =301 http:/example.com/new/path.html;
}
```

## 静态文件服务
### Root目录和索引文件
root指令制定了将要去搜索文件的根目录。为了获得文件路径，nginx为请求的URI添加了指定的root路径。root指令可以在http、server和location块中。
```
server {
    root /www/data;

    location / {
    }

    location /images/ {
    }

    location ~ \.(mp3|mp4) {
        root /www/media;
    }
}
```
上面的例子中，如果URI以/images/开始，那么，搜索的路径为/www/data/images/目录。如果URI以.mp3或者.mp4结尾，那么搜索的路径为/www/media/。

如果请求是以/结尾，那么，nginx将其认为是请求目录结构并查找目录里面的索引文件。index命令定义了索引文件名（默认名称为index.html）。接着上面的例子，如果请求是 /images/some/path/ ， nginx将传送文件/www/data/images/some/path/index.html，如果存在的话。如果不存在，nginx返回404。为了让NGINX返回一个自动生成的目录清单,加入autoindex指令:
```
location /images/ {
    autoindex on;
}
```
你可以列举不只一个文件名在index指令下。nginx以指定的顺序搜索文件并返回第一个。
```
location / {
    index index.$geo.html index.htm index.html;
}
```
上面的$geo变量是一个自定义的geo。它的值取决于客户端的ip地址。

autoindex_exact_size on | off 表示是否显示大小。
autoindex_format html | xml | json | jsonp 输出格式
autoindex_localtime on | off  显示本地时间或UTC

### 其它的命令
try_files 用来检查指定的文件或者目录是否存在，并设置一个重定向或者返回一个指定的状态码。
```
server {
    root /www/data;

    location /images/ {
        try_files $uri /images/default.gif;
    }
}
```
最后一个参数可以是状态码或者位置。在下面的例子中，如果try_files的参数中的文件或目录不存在，则返回404。
```
location / {
    try_files $uri $uri/ $uri.html =404;
}
```
在下面的例子中，如果原始的URI和URI后面加上/的目录均不存在，那么请求会被重定向到一个代理服务器。
```
location / {
    try_files $uri $uri/ @backend;
}

location @backend {
    proxy_pass http://backend.example.com;
}
```

### 提高nginx内容服务速度
加载速度是重要的考虑因素。做一些小幅的优化配置可能会大幅提高效率甚至达到最佳性能。
- 开启sendfile
默认情况下，nginx自己控制文件传输，并在发送前把它拷贝到buffer中。使用sendfile命令可以减少拷贝到buffer的步骤，并且直接将一个文件描述符拷贝到另一个。另外,防止一个快速连接完全占用worker进程,可以在sendfile()调用sendfile_max_chunk指令限制传输的数据量：
```
location /mp3 {
    sendfile           on;
    sendfile_max_chunk 1m;
    ...
}
```
- 开启tcp_nopush
配合sendfile一起使用tcp_nopush。一般情况下，在tcp交互的过程中，当应用程序接收到数据包后马上传送出去，不等待，而tcp_cork选项是数据包不会马上传送出去，等到数据包最大时，一次性的传输出去，这样有助于解决网络堵塞，已经是默认了,tcp_nopush = on 会设置调用tcp_cork方法.
```
location /mp3 {
    sendfile   on;
    tcp_nopush on;
    ...
}
```
- 开启tcp_nodelay
与tcp_nopush 是互斥的，有数据的话会立即将数据包发送出去，有可能会造成网络拥堵。
```
location /mp3  {
    tcp_nodelay       on;
    keepalive_timeout 65;
    ...
}
```
- 优化Backlog Queue
nginx处理连接请求也是一个非常重要的优化点。一般情况下，当一个连接建立后，它会被放入listen socket的listen队列。在普通负载下，这是一个小的甚至不存在的队列。但是高负载时，队列会显著增长，这可能导致连接断开或者高延迟。
**检查listen queue**
```
netstat -lan
```
结果如下
```
Current listen queue sizes (qlen/incqlen/maxqlen)
Listen         Local Address         
0/0/128        *.12345            
10/0/128        *.80       
0/0/128        *.8080
```
说明有10个在80端口的连接未被处理。
**设置OS**
设置 net.core.somaxconn 来增大OS的负载能力。
```
vi   /etc/sysctl.conf
net.core.somaxconn = 4096
```
**设置nginx**
```
server {
    listen 80 backlog 4096;
    # The rest of server configuration
}
```

## 反向代理
### 发送请求到代理服务器
当使用nginx代理时，会发送请求到代理服务器，获取相应，然后返回给客户端。它可以代理HTTP请求也可以代理非HTTP请求。
```
location /some/path/ {
    proxy_pass http://www.example.com/link/;
}
```

### 发送请求头到代理服务器
默认情况下，nginx会重新定义请求头中的两个域，Host和Connection，Host被设置为$proxy_host变量，Connection被设置为 close。
```
location /some/path/ {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_pass http://localhost:8000;
}
```
防止某个域传递到代理服务器，可以将其设置为空：
```
location /some/path/ {
    proxy_set_header Accept-Encoding "";
    proxy_pass http://localhost:8000;
}
```

### 设置Buffer
一般情况下，nginx缓冲代理服务器的相应，直到收到整个相应信息才发送给客户端。缓冲机制能够优化慢客户端，因为如果对于慢客户端nginx同步响应，那么会浪费代理服务器时间。
**proxy_buffering**，该指令设置缓冲区的大小和数量,从被代理的后端服务器取得的响应内容,会放置到这里. 默认情况下,一个缓冲区的大小等于内存页面大小,可能是4K也可能是8K,这取决于平台。proxy_buffers 8  4k/8k。
**proxy_buffer_size**，该指令设置缓冲区大小,从代理后端服务器取得的第一部分的响应内容,会放到这里.小的响应header通常位于这部分响应内容里边.默认来说,该缓冲区大小等于指令 proxy_buffers所设置的;但是,你可以把它设置得更小。
```
location /some/path/ {
    proxy_buffers 16 4k;
    proxy_buffer_size 2k;
    proxy_pass http://localhost:8000;
}
```
如果关闭buffer：
```
location /some/path/ {
    proxy_buffering off;
    proxy_pass http://localhost:8000;
}
```
### 绑定出口IP
如果你的代理服务器有多个网络接口，有时需要选择其中一个进行绑定。
```
location /app1/ {
    proxy_bind 127.0.0.1;
    proxy_pass http://example.com/app1/;
}

location /app2/ {
    proxy_bind 127.0.0.2;
    proxy_pass http://example.com/app2/;
}
```
也可以使用变量
```
location /app3/ {
    proxy_bind $server_addr;
    proxy_pass http://example.com/app3/;
}
```

## 压缩和解压缩
亚索形影数据对于减小传输数据是非常重要的。然而，在运行时进行压缩还有可能对系统有负面影响。nginx在发送相应数据前进行压缩，但是不会对已经压缩过的数据进行二次压缩（例如，对于代理服务器的）。
### 开启压缩
```
gzip on;
```
通常情况下，nginx仅对MIME类型为text/html的相应进行压缩。为了添加压缩的类型，可以使用gzip_types命令。
```
gzip_types text/plain application/xml;
```

指定压缩响应的最小长度，使用gzip_min_length指令。默认是20bytes。（这里设置为1000）。
```
gzip_min_length 1000;
```
一般情况下，nginx不会压缩代理服务器的请求响应。该请求来自代理服务器的事实是由Via头字段的请求中的存在来确定。使用gzip_proxied指令来配置这些响应的压缩。这个指令有很多参数，来确定哪种代理请求nginx需要压缩。
```
gzip_proxied no-cache no-store private expired auth;
```
一个完整的例子是：
```
server {
    gzip on;
    gzip_types      text/plain application/xml;
    gzip_proxied    no-cache no-store private expired auth;
    gzip_min_length 1000;
    ...
}
```
### 开启解压缩
一些客户端不支持gzip编码算法。同时，它又想要存储压缩数据或者压缩相应数据并存入缓存。为了成功发送到客户端，nginx支持在发送到最终客户端时解压缩数据。
```
location /storage/ {
    gunzip on;
    ...
}
```
### 发送压缩文件
使用gzip_static 命令可以发送一个压缩版本的文件到客户端，若要使用它需要在编译的时候把gzip_static模块编译进去：
`./configure --with-http_gzip_static_module `

```
location / {
    gzip_static on;
}
```
当请求 /path/to/file，nginx会查找并发送/path/to/file.gz。如果文件不存在或者客户端不支持gzip，nginx会发送未压缩版本。
注意，gzip_static指令不支持实时压缩。它只是使用压缩工具预先压缩文件。要压缩在运行时的内容（不仅是静态内容），使用gzip的指令。

## 页面内容缓存
nginx支持缓存，当开启缓存后，nginx缓存从代理服务器返回的数据，并将其缓存到硬盘上，当有请求过来时，先去缓存查找响应。

### 开启响应缓存
将 proxy_cache_path 放到http块中来开启缓存。第一个参数是存放缓存文件的路径。keys_zone定义了用于存储缓存条目元数据的共享存储空间名称和大小。
```
http {
    ...
    proxy_cache_path /data/nginx/cache keys_zone=one:10m;
}
```

将proxy_cache放入想进行缓存的协议类型、server块或者location块中。并指定proxy_cache_path中的key_zone。
```
http {
    ...
    proxy_cache_path /data/nginx/cache keys_zone=one:10m;

    server {
        proxy_cache one;
        location / {
            proxy_pass http://localhost:8000;
        }
    }
}
```
值得注意的是，key_zone定义的大小并不限制缓存相应数据的总量。缓存的响应数据本身存储为元数据的一个备份，存在指定的file上。为了限制缓存的数量，可以使用proxy_cache_path的max_size参数，缓存数量可以暂时超过max_size。

### 控制缓存
缓存管理器会定期检查缓存的状态。如果缓存大小超过proxy_cache_path中 max_size参数设定的限制,缓存管理器删除最近最少访问的数据。正如前面提到的,缓存数据的数量可以暂时超过限制缓存管理器激活的时间期间。
缓存加载器仅会在nginx启动后启动一次。它会将之前缓存的数据加载进来。加载一次缓存可能会消耗大量资源，影响nginx启动后几分钟内的性能。下面有proxy_cacahe_path的几个参数来避免这一情况：
- loader_threshold ： 加载时间上限，单位是毫秒，默认为200毫秒。
- loader_files： 一次迭代的最多条目数，默认是100。
- loader_sleeps ： 迭代的间隔，单位是毫秒，默认50。

`proxy_cache_path /data/nginx/cache keys_zone=one:10m loader_threshold=300 loader_files=200;`

### 指定对哪个请求进行缓存
一般情况下，nginx缓存http的get和head方法响应的数据。nginx将请求字符串作为请求的key。如果缓存中存在和请求相同的key，nginx会直接用缓存来响应。可以在http，server，location的上下文中控制哪些进行缓存。

为了改变请求的单词来计算key，可以使用proxy_cache_key:
```
proxy_cache_key "$host$request_uri$cookie_user";
```

定义在必须在请求指定的最低次数以后，响应才会被缓存起来：
```
proxy_cache_min_uses 5;
```

缓存除了GET和HEAD的其他请求，get和head也需要被列举出来：
```
proxy_cache_methods GET HEAD POST;
```
### 限制或绕过缓存
默认情况下，响应数据会一直在缓存中。它们只有在缓存超过最大值时，并且它们是最少命中的，才会被淘汰。nginx可以设置淘汰策略：
```
proxy_cache_valid 200 302 10m;
proxy_cache_valid 404      1m;
```
上面的例子中，返回200或者302都被认为是在10分钟以内有效的。返货404在1分钟以内也是有效的。也可以使用`any`来设置第一个参数。
```
proxy_cache_valid any 5m;
```

可以定义proxy_cache_bypass指令来决定是否使用cache响应请求。每个参数定义了一个条件并且由变量组成。如果至少有一个参数为空并且不等于0，nginx不会从cache查找响应，会直接去后端服务器请求。
```
proxy_cache_bypass $cookie_nocache $arg_nocache$arg_comment;
```
`proxy_cache_bypass string ...;`

控制哪些请求不进行缓存
```
proxy_no_cache $http_pragma $http_authorization;
```

### 从缓存中清除内容
NGINX可以从缓存中删除过期的缓存文件。这是非常必要的,删除过期的缓存内容,防止同时提供新老版本的web页面。清除缓存时，nginx会收到一个特别的“清除”请求包含一个自定义HTTP头,或“清除”的HTTP方法。

- 配置缓存清除
下面配置一个清除的HTTP方法并删除匹配的URL。
在http块中，新建一个变量，如下面的 $purge_method ， 它依赖于$request_method变量。
```
http {
    ...
    map $request_method $purge_method {
        PURGE 1;
        default 0;
    }
}
```
在location块中已经定义了cache，在 proxy_cache_purge 指令中，指定了会被清除cache的条件。
```
server {
    listen      80;
    server_name www.example.com;

    location / {
        proxy_pass  https://localhost:8002;
        proxy_cache mycache;

        proxy_cache_purge $purge_method;
    }
}
```
当 $request_method 为PURGE，则清除。否则不清除。

- 发送清除指令
```
$ curl -X PURGE -D – "https://www.example.com/*"
HTTP/1.1 204 No Content
Server: nginx/1.5.7
Date: Sat, 01 Dec 2015 16:33:04 GMT
Connection: keep-alive
```
上例中，指定的URI中的缓存文件并不删除，它们还会继续存储在磁盘上，直到nginx来操作处理。
- 限制访问清除指令
比较推荐的是通过设置IP白名单来限制访问。
```
geo $purge_allowed {
   default         0;  # deny from other
   10.0.0.1        1;  # allow from localhost
   192.168.0.0/24  1;  # allow from 10.0.0.0/24
}

map $request_method $purge_method {
   PURGE   $purge_allowed;
   default 0;
}
```
- 完全删除cache文件
`proxy_cache_path /data/nginx/cache levels=1:2 keys_zone=mycache:10m purger=on;`
在 proxy_cache_path 加上 purger=on参数。
- 完整的例子
```
http {
    ...
    proxy_cache_path /data/nginx/cache levels=1:2 keys_zone=mycache:10m purger=on;

    map $request_method $purge_method {
        PURGE 1;
        default 0;
    }

    server {
        listen      80;
        server_name www.example.com;

        location / {
            proxy_pass        https://localhost:8002;
            proxy_cache       mycache;
            proxy_cache_purge $purge_method;
        }
    }

    geo $purge_allowed {
       default         0;
       10.0.0.1        1;
       192.168.0.0/24  1;
    }

    map $request_method $purge_method {
       PURGE   $purge_allowed;
       default 0;
    }
}
```
### Byte范围的缓存
有时候，将数据放入缓存中是一个很费时的操作，特别是大文件。当第一次请求开始下载一个大文件时，下一次请求必须等待整个文件下载并放入缓存后才能被服务。
nginx可以使用cache slice module 来处理。文件被分成较小的“片”。每个请求范围选择特定的片,如果这个范围没有被缓存,那么将会把它放到缓存中。然后其它所有请求这个片数据的请求都会被这个缓存响应。
开启范围级别的缓存：
1. 为nginx编译进slice模块。
2. 指定每个片的大小。片大小应足以使切片快速下载。设置太小可能会导致过度的内存使用和大量的文件描述符,过大的值可能会导致延迟。
```
location / {
    slice  1m;
}
```
3. 在cache key中加入$slice_range
`proxy_cache_key $uri$is_args$args$slice_range;`
4. 开启响应的206代码
`proxy_cache_valid 200 206 1h;`
5. 在发往代理服务器的HTTP头的Range里面加入$slice_range。
`proxy_set_header  Range $slice_range;`
6. 综合样例
```
location / {
    slice             1m;
    proxy_cache       cache;
    proxy_cache_key   $uri$is_args$args$slice_range;
    proxy_set_header  Range $slice_range;
    proxy_cache_valid 200 206 1h;
    proxy_pass        http://localhost:8000;
}
```
