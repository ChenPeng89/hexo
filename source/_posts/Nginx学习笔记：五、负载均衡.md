---
title: Nginx学习笔记：五、负载均衡
date: 2016-07-18 11:31:02
tags: [Nginx , 负载均衡]
---

## HTTP 负载均衡
### 负载一组服务器
在nginx中使用一组服务器之前，需要先定义一组服务器，在http块中使用upstream指令。
```
http {
    upstream backend {
        server backend1.example.com weight=5;
        server backend2.example.com;
        server 192.0.0.1 backup;
    }
}
```
想要将请求发送到一组后端服务器，还需要指定这组服务器的名称，使用 proxy_pass ：
```
server {
    location / {
        proxy_pass http://backend;
    }
}
```
### 选择一个负载均衡的方法
1. 轮询
根据服务器的权重将请求均匀的分布到所有服务器上。这个是默认的策略：
```
upstream backend {
   server backend1.example.com;
   server backend2.example.com;
}
```
2. 连接数最小
将请求发送到目前连接数最小的服务器上：
```
upstream backend {
    least_conn;

    server backend1.example.com;
    server backend2.example.com;
}
```
3. ip_hash
由客户端IP决定发送到哪个服务器上。
```
upstream backend {
    ip_hash;

    server backend1.example.com;
    server backend2.example.com;
}
```
如果哪个服务器需要暂时停止服务，可以使用down指令：
```
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com down;
}
```
4. 一致性hash
通过用户定义的key来决定请求分发到哪台服务器：
```
upstream backend {
    hash $request_uri consistent;

    server backend1.example.com;
    server backend2.example.com;
}
```
5. 最低延迟和最小连接数
选取最低延迟和最小连接数的服务器：
```
upstream backend {
    least_time header;

    server backend1.example.com;
    server backend2.example.com;
}
```
header : 从服务器接收到第一个字节所用的时间。
last_byte ： 从服务器接收全部响应所用的时间。

### 服务器权重
默认情况下，nginx根据服务器的权重来进行轮训的分发。
```
upstream backend {
    server backend1.example.com weight=5;
    server backend2.example.com;
    server 192.0.0.1 backup;
}
```
第三个服务器设为 backup，只有当前两个服务器不可用时，它才能接收请求。

### 服务器慢启动
服务器的慢启动能够保护从刚刚连接超时或者其他的原因不可用的状态恢复的服务器被连接淹没。
nginx慢启动可以逐步恢复服务器，一开始将服务器的权重设为0，然后逐步成为设置的权重。
```
upstream backend {
    server backend1.example.com slow_start=30s;
    server backend2.example.com;
    server 192.0.0.1 backup;
}
```
### session持久化
session持久化意味着Nginx标识用户session并将请求发送到与之前相同的服务器上。
session支持三种session持久化方法：
1. sticky cookie 
使用这个方法，nginx会为第一个响应的upstream组添加一个session的cookie，并标识响应请求的服务器。当下次请求时，它会携带一个cookie的值，然后nginx将请求路由到相同的服务器上：
```
upstream backend {
    server backend1.example.com;
    server backend2.example.com;

    sticky cookie srv_id expires=1h domain=.example.com path=/;
}
```
例子中，srv_id参数设置了cookie的名称。可选的expire参数设置了浏览器保持cookie的时间。可选的参数domian定义了一个设置cookie的domain。可选参数path定义了cookie设置的访问路径。这是最简单的session持久化方法。

2. sticky route 
nginx为第一次接受请求的客户端设置了一个route。所有后续请求将与server指令中识别服务器的route参数进行比较。
路由信息取自cookie或者URI。
```
upstream backend {
    server backend1.example.com route=a;
    server backend2.example.com route=b;

    sticky route $route_cookie $route_uri;
}
```
3. cookie learn 
首先，nginx通过检查请求和响应来发现session标识。然后，nginx“学习”哪个upstream 服务器组和哪个session标识相关联。举荐的，这些标识被传进HTTPcookie。如果请求包含的session已经被“学习”，nginx将会直接将请求发送到关联的服务器上。
```
upstream backend {
   server backend1.example.com;
   server backend2.example.com;

   sticky learn 
       create=$upstream_cookie_examplecookie
       lookup=$cookie_examplecookie
       zone=client_sessions:1m
       timeout=1h;
}
```
例子中，upstream的服务器在响应中通过设置cookie “EXAMPLECOOKIE”来创建了一个session。
必填的 create 参数指定了怎么创建一个新的session。在例子中，新的session是由upstream服务器发送的cookie “EXAMPLECOOKIE”创建的。
必填的参数 lookup 指定了怎么搜索已经存在的session。在例子中，从客户端发送的cookie “EXAMPLECOOKIE” 中查询已经存在的session。
必填的参数 zone 指定了一个保存所有session信息的共享内存。
这个策略不需要客户端保存任何cookie，所有信息保存在服务器端的共享内存中。

### 限制连接数
如果MAX_CONNS已达到限制，该请求可以被放置到队列为提供该队列指令指定其进一步处理。该指令集，可以在队列中同时请求的最大数：
```
upstream backend {
    server backend1.example.com  max_conns=3;
    server backend2.example.com;

    queue 100 timeout=70;
}
```
如果有其它worker进程打开了空闲时keepalive，max_conn限制将会被忽略。结果是，连接到服务器的总数将会超过max_conns。

### 被动的健康监测
当nginx认为一个服务器不可用时，它会暂时停止发送请求到这台服务器知道服务器被认为可用。
max_fails和fail_timeout —— 这俩是关联的，如果某台服务器在fail_timeout时间内出现了max_fails次连接失败，那么nginx就会认为那个服务器已经挂掉，从而在 fail_timeout时间内不再去查询它，fail_timeout的默认值是10s，max_fails的默认值是1（这意味着一发生错误就认为服务器挂掉），如果把max_fails设为0则表示把这个检查取消。

```
upstream backend {                
    server backend1.example.com;
    server backend2.example.com max_fails=3 fail_timeout=30s;
    server backend3.example.com max_fails=2;
}
```
### 主动的健康监测
预先发送一个指定的请求到每个服务器，并检测响应信息是否符合检测中的可用条件。
```
http {
    upstream backend {
        zone backend 64k;

        server backend1.example.com;
        server backend2.example.com;
        server backend3.example.com;
        server backend4.example.com;
    }

    server {
        location / {
            proxy_pass http://backend;
            health_check;
        }
    }
}
```
zone 指令定义了被worker进程共享的并用来存储服务器组配置的内存区域。
```
location / {
    proxy_pass http://backend;
    health_check interval=10 fails=3 passes=2;
}
```
健康监测的时间间隔是10s，在失败3次后会认为是不可用的，以后需要两次通过监测才能认为是可用的。
也可以指定URI进行监测：
```
location / {
    proxy_pass http://backend;
    health_check uri=/some/path;
}
```
还可以设置macth块来指定健康监测的响应结果
```
http {
    ...

    match server_ok {
        status 200-399;
        body !~ "maintenance mode";
    }

    server {
        ...

        location / {
            proxy_pass http://backend;
            health_check match=server_ok;
        }
    }
}
```
也可以定义返回头中的信息
```
match welcome {
    status 200;
    header Content-Type = text/html;
    body ~ "Welcome to nginx!";
}
```
用!可以定义非的条件
```
match not_redirect {
    status ! 301-303 307;
    header ! Refresh;
}
```

### 使用DNS配置HTTP负载均衡
服务器组的配置可以在运行时使用DNS修改。
nginx可以监控IP地址对应的域名服务器的变化，并自动将变化应用于nginx，且不用重启。这可以通过在http块中使用resolver指令，和server指令后的 resolver参数。
```
http {
    resolver 10.0.0.1 valid=300s ipv6=off;
    resolver_timeout 10s;

    server {
        location / {
            proxy_pass http://backend;
        }
    }
   
    upstream backend {
        zone backend 32k;
        least_conn;
        ...
        server backend1.example.com resolve;
        server backend2.example.com resolve;
    }
}
```
在这个示例中,服务器的resolve参数指令将定期通过IP地址重新解析 backend1.example.com和backend2.example.com服务器。默认情况下,NGINX基于TTL重新解析DNS记录,但TTL值可以覆盖resolve指令的valid参数,在我们的示例中是5分钟。
如果一个域名对应多个IP地址，那么IP地址会被存到upstream配置中并被负载均衡。在例子中，服务器会使用least_conn来负载均衡。如果一个活多个IP地址被修改、添加或删除，那么这些服务器也会被重新负载。

### 动态配置
nginx的服务器组可以通过HTTP接口来动态配置。配置指令可以用来查看所有服务器或者服务器组，修改服务器参数或者添加删除服务器。

- 设置动态配置
	1. 将zone指令放入upstream块中。zone 指令配置了一个区域来共享内存，并设置zone 的名称和大小。服务器组的配置被存放到这个zone中，所有的worker进程都使用同样的配置。
	```
		http {
		    ...
    		upstream appservers {
        		zone appservers 64k;
        		server appserv1.example.com      weight=5;
        		server appserv2.example.com:8080 fail_timeout=5s;
        		server reserve1.example.com:8080 backup;
        		server reserve2.example.com:8080 backup;
    		}
		}
    ```
	2. 配置 upstream_conf指令到location块中。
	```
		server {
    		location /upstream_conf {
        		upstream_conf;
        		...
    		}
		}
 	```
	为其设置访问白名单。
	```
	server {
    	location /upstream_conf {
    	    upstream_conf;
    	    allow 127.0.0.1;
    	    deny  all;
    	}
	}
	```
	3. 一个完整的例子：
	```
	http {
    	...
    	# Configuration of the server group
    	upstream appservers {
        	zone appservers 64k;

        	server appserv1.example.com      weight=5;
        	server appserv2.example.com:8080 fail_timeout=5s;

        	server reserve1.example.com:8080 backup;
        	server reserve2.example.com:8080 backup;
    	}

    	server {
        	# Location that proxies requests to the group
        	location / {
            	proxy_pass http://appservers;
            	health_check;
        	}

        	# Location for configuration requests
	        location /upstream_conf {
    	        upstream_conf;
    	        allow 127.0.0.1;
    	        deny  all;
    	    }
    	}
	}
	```
- 动态配置持久化
上面的动态配置会随着nginx配置文件的reload而失效，为了让它继续能有效果，需要将upstream服务器从upstream块移动到一个指定的文件中，能够保持upstream服务器的状态。文件的路径要在 state指令中设置。在Linux中，比较推荐的路径是/var/lib/nginx/state/。
```
http {
    ...
    upstream appservers {
        zone appservers 64k;
        state /var/lib/nginx/state/appservers.conf;

        # All these servers should be moved to the file using the upstream_conf API:
        # server appserv1.example.com      weight=5;
        # server appserv2.example.com:8080 fail_timeout=5s;
        # server reserve1.example.com:8080 backup;
        # server reserve2.example.com:8080 backup;
    }
}
```
需要注意的是，文件只能被 upstream_conf API来修改，应该避免直接修改文件。

- 动态配置Upstream服务器
使用HTTP请求来将配置命令传递给nginx。请求需要有一个合适的URI来获取到location中的upstream_conf指令。请求需要包含upstream参数来确定修改的服务器组。
查看所有backup的服务器
```
http://127.0.0.1/upstream_conf?upstream=appservers&backup=
```
添加一个新的服务器到组里：
```
http://127.0.0.1/upstream_conf?add=&upstream=appservers&server=appserv3.example.com:8080&weight=2&max_fails=3
```
删除一个服务器：
```
http://127.0.0.1/upstream_conf?remove=&upstream=appservers&id=2
```
修改一个服务器：
```
http://127.0.0.1/upstream_conf?upstream=appservers&id=2&down=
```
## TCP/UDP负载均衡
### 配置反向代理
首先，需要额皮质一个反向代理来使nginx通过TCP连接或者UDP报文将客户端数据发送到upstream组或者一个代理服务器。

1. 在顶级目录创建一个stream块
```
stream {
...
}
```
2. 在stream块中定义一个或多个server块
3. 在server块中定义listen指令来监听ip和端口。对于UDP，还需要包含udp参数。TCP在stream块中是默认的，所以listen没有tcp参数。
```
stream {
    server {
        listen 12345;
        ...
    }
    server {
        listen 53 udp;
        ...
    }
    ...
}
```
4. 通过proxy_pass指令来定义将要跳转的代理服务器或者upstream组。
```
stream {
    server {
        listen     12345;

        #TCP traffic will be proxied to the "stream_backend" upstream group
        proxy_pass stream_backend;
    }

    server {
        listen     12346;

        #TCP traffic will be proxied a proxied server
        proxy_pass backend.example.com:12346;
    }

    server {
        listen     53 udp;

        #UDP traffic will be proxied to the "dns_servers" upstream group
        proxy_pass dns_servers;
    }
    ...
}
```
5. 如果你的代理服务器有多个网络接口，你可以配置nginx选一个源ip地址来连接到upstream服务器。这对于后端服务器仅接受指定IP地址访问是有用的。
```
stream {
    ...
    server {
        listen     127.0.0.1:12345;
        proxy_pass backend.example.com:12345;
        proxy_bind 127.0.0.1:12345;
    }
}
```
6. 可以调整两个缓冲区的大小，这两个缓冲区用于nginx缓存客户端和upstream的连接中的数据。如果数据比较小，buffer会自动缩小来节省存储资源。如果有大量的数据,可以通过减少socket的读/写操作次数来增加buffer。
```
stream {
    ...
    server {
        listen            127.0.0.1:12345;
        proxy_pass        backend.example.com:12345;
        proxy_buffer_size 16k;
    }
}
```
### 配置TCP/UDP负载均衡
创建一组服务器或upstream组。
```
stream {
    upstream stream_backend {
        ...    
    }

    upstream dns_servers {
        ...    
    }
    ...
}
```
剩下和HTTP差不多，如有疑问：https://www.nginx.com/resources/admin-guide/tcp-load-balancing/

## 设置代理协议
代理协议允许nginx接受客户端连接信息并通过HAproy等发送给代理服务器。

通过代理协议发送的信息包括客户端IP地址、代理服务器IP和它们的端口号。知道原始的IP地址有助于网站的语言设置、访问黑名单或一些简单的log和静态资源。

通过代理协议，nginx可以获取到原始的ip地址通过SSL, HTTP/2, SPDY, WebSocket, 和 TCP。

### 使用SSL, HTTP/2, SPDY, 和 WebSocket 代理协议
1. 配置nginx接受代理协议报文头。在listen中添加proxy_protocol参数。
```
server {
    listen 80   proxy_protocol;
    listen 443  ssl proxy_protocol;
    ...
}
```
2. 在set_real_ip_from 指令中，指定ip地址或者TCP代理的CIDR地址范围或者负载均衡器。
```
server {
    ...
    set_real_ip_from 192.168.1.0/24;
    ...
}
```
3. 在real_ip_header指令中，添加proxy_protocol参数来保持客户端IP地址和端口号。
```
server {
    ...
    real_ip_header proxy_protocol;
}
```
4. 通过proxy_set_header directive 和 $proxy_protocol_addr变量从nginx传递IP地址到upstream服务器。
```
proxy_set_header X-Real-IP       $proxy_protocol_addr;
proxy_set_header X-Forwarded-For $proxy_protocol_addr;
```
5. 在http层添加 $proxy_protocol_addr 变量到  log_format指令。
```
http {
    ...
    log_format combined '$proxy_protocol_addr - $remote_user [$time_local] '
                        '"$request" $status $body_bytes_sent '
                        '"$http_referer" "$http_user_agent"';
}
```

### 在TCP流上使用代理协议
nginx能够就在TCP流上传递代理协议数据。
```
stream {
    server {
        listen 12345;
        proxy_pass example.com:12345;
        proxy_protocol on;
    }
}
```
### 完整的样例
```
http {
    log_format combined '$proxy_protocol_addr - $remote_user [$time_local] '
                        '"$request" $status $body_bytes_sent '
                        '"$http_referer" "$http_user_agent"';
    ...

    server {
        server_name localhost;

        listen 80   proxy_protocol;
        listen 443  ssl proxy_protocol;

        ssl_certificate      /etc/nginx/ssl/public.example.com.pem;
        ssl_certificate_key  /etc/nginx/ssl/public.example.com.key;

        set_real_ip_from 192.168.1.0/24;
        real_ip_header   proxy_protocol;

        location /app/ {
            proxy_pass       http://backend1;
            proxy_set_header Host            $host;
            proxy_set_header X-Real-IP       $proxy_protocol_addr;
            proxy_set_header X-Forwarded-For $proxy_protocol_addr;
        }
    }
} 

stream {
...
    server {
        listen         12345;
        proxy_pass     example.com:12345;
        proxy_protocol on;
    }

}
```




