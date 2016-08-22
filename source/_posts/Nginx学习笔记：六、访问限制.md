---
title: Nginx学习笔记：六、访问限制
date: 2016-07-21 13:57:22
tags: [Nginx , 访问限制]
---
## 限制访问HTTP代理资源
### 控制访问
可以通过客户端IP地址或使用基于HTTP的身份认证。
使用IP地址控制访问：
```
location / {
    deny  192.168.1.2;
    allow 192.168.1.1/24;
    allow 127.0.0.1;
    deny  all;
}
```
开启身份验证，可以使用 auth_basic指令。然后用户必须加入他们的有效用户名和密码来获取网站访问权。用户名和密码必须在auth_basic_user_file指定的文件中列举出来。
```
server {
    ...
    auth_basic "closed website";
    auth_basic_user_file conf/htpasswd;
}
```
可以设置某些URL不需要身份认证。
```
server {
    ...
    auth_basic "closed website";
    auth_basic_user_file conf/htpasswd;

    location /public/ {
        auth_basic off;
    }
}
```
可以使用satisfy指令来控制访问，如果需要某一个条件，则使用any ， 如果都需要满足，则使用all。
```
location / {
    satisfy any;

    allow 192.168.1.0/24;
    deny  all;

    auth_basic           "closed site";
    auth_basic_user_file conf/htpasswd;
}
```
### 限制访问
- 限制访问的连接数
首先，使用 limit_conn_zone 指令定义key和共享的内存
`limit_conn_zone $binary_remote_address zone=addr:10m;`
第二步，使用limit_conn 指令指定使用的http 、 server 或者location。
```
location /download/ {
    limit_conn addr 1;
}
```
这里可以基于IP限制连接数，因为使用了$binary_remote_address 变量当做key。连接数也可以通过 server 名称来限制，通过使用$server_name变量。
```
http {
    limit_conn_zone $server_name zone=servers:10m;

    server {
        limit_conn servers 1000;
    }
}
```
- 限制访问率
限制访问率，首先使用 limit_req_zone 指令设置key和共享内存区域来供计数器使用。
```
limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
```
rate参数可以设置为每秒请求(r/s)或者每分钟请求 (r/m)。比如30r/m。
当设置了共享内存，使用limit_req指令在server 或 location中来限制速率。
```
location /search/ {
    limit_req zone=one burst=5;
}
```
这里，nginx不会再每秒处理多于一个请求在指定的location中。如果速率超过了请求限制，那么会将请求放入队列中延迟处理。burst参数设置了每秒处理请求的最大值，当超过了这个最大值，会返回503.
如果不想在设置了burst后有延迟处理，添加nodelay参数。
```
limit_req zone=one burst=5 nodelay;
```
- 限制带宽
限制每个链接的带宽，可以使用limit_rate指令。
```
location /download/ {
    limit_rate 50k;
}
```
通过这个设置，客户端最多能够下载50k内容通过一个单独的链接。然而，客户端能够开几个链接。所以如果想要更好的控制链接带宽，可以配合限制连接数使用。例如，一个IP一个连接：
```
location /download/ {
    limit_conn addr 1;
    limit_rate 50k;
}
```
可以允许客户端在下载一定量的数据后再进行限速：
```
limit_rate_after 500k;
limit_rate 20k;
```
下面是一个完整的例子，限制了连接数和带宽
```
http {
    limit_conn_zone $binary_remote_address zone=addr:10m

    server {
        root /www/data;
        limit_conn addr 5;

        location / {
        }

        location /download/ {
            limit_conn addr 1;
            limit_rate 1m;
            limit_rate 50k;
        }
    }
}
```

## 限制访问TCP代理资源
### 通过IP地址限制访问
```
stream {
    ...
    server {
        listen 12345;
        deny   192.168.1.2;
        allow  192.168.1.1/24;
        allow  2001:0db8::/32;
        deny   all;
    }
}
```
### 限制TCP连接数
可以有效的防止DOS攻击。
```
stream {
    ...
    limit_conn_zone $binary_remote_addr zone=ip_addr:10m;
    ...
}
```
大致和HTTP的配置方式差不多。
```
stream {
    ...
    limit_conn_zone $binary_remote_addr zone=ip_addr:10m;

    server {
        ...
        limit_conn ip_addr 1;
    }
}
```
### 限制带宽
```
server {
    ...
    proxy_download_rate 100k;
    proxy_upload_rate   50k;
}
```
下面是一个完整的例子
```
stream {
    ...
    limit_conn_zone $binary_remote_addr zone=ip_addr:10m;

    server {
        ...
        limit_conn ip_addr 1;
        proxy_download_rate 100k;
        proxy_upload_rate   50k;
    }
}
```
