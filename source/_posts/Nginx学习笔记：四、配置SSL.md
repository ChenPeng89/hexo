---
title: Nginx学习笔记：四、配置SSL
date: 2016-07-18 09:30:57
tags: Nginx
---

## 通过HTTPS传递Web内容
### 配置一个HTTPS服务
建立一个HTTPS的nginx服务器，在nginx.conf中指定服务器的ssl参数与listen指令，然后设置服务器证书和私钥文件的位置:

x509证书一般会用到三类文，key，csr，crt。

Key 是私用密钥openssl格，通常是rsa算法。

Csr 是证书请求文件，用于申请证书。在制作csr文件的时，必须使用自己的私钥来签署申，还可以设定一个密钥。

crt是CA认证后的证书文，（windows下面的，其实是crt），签署人用自己的key给你签署的凭证。 
```
server {
    listen              443 ssl;
    server_name         www.example.com;
    ssl_certificate     www.example.com.crt;
    ssl_certificate_key www.example.com.key;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ...
}
```
服务器证书是一个公共实体。它被发送到每一个连接到服务器的客户端。私钥是一个安全的实体，并应被存储在与限制访问的文件。然而，Nginx的的master进程必须能够读取该文件。私钥也可以和公钥存储在同一个文件：
```
ssl_certificate xxx.cert;
ssl_certificate_key xxx.cert;
```
在这种情况下，文件的访问权限也需要被限制。虽然公钥和私钥在一个文件中，但是只有公钥会被发送到客户端。

ssl_protocols 和 ssl_ciphers 指令可以用来限制连接的版本和SSL/TLS的加密方式。

### https服务优化
SSL会给CPU带来额外的开销。最耗CPU的是SSL的握手过程。下面有两个方法能够减少每个客户端操作的数量：
- 使keepalive连接通过一个连接发送多个请求
- 重用SSL会话参数来避免并行和随后的SSL握手连接

session存储在SSL的 session缓存中，并被worker进程共享，这个可以用ssl_session_cache 配置。1M的cache可以包含大约4000个session。默认情况下，cache的超时时间是5分钟。超时时间可以用 ssl_session_timeout 来设置。
```
worker_processes auto;

http {
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;

    server {
        listen              443 ssl;
        server_name         www.example.com;
        keepalive_timeout   70;

        ssl_certificate     www.example.com.crt;
        ssl_certificate_key www.example.com.key;
        ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers         HIGH:!aNULL:!MD5;
        ...
    }
}
```

### SSL证书链
一些浏览器会报出证书不是由权威机构颁发的，而其他的浏览器则不会有这个问题。这是由于发行机构的证书不是由权威机构的中级证书签发的。这种情况下，发行机构应该提供一个证书链来和服务器证书绑定。在绑定文件中，公钥证书应该在证书链之前：

```
$ cat www.example.com.crt bundle.crt > www.example.com.chained.crt
```

```
server {
    listen              443 ssl;
    server_name         www.example.com;
    ssl_certificate     www.example.com.chained.crt;
    ssl_certificate_key www.example.com.key;
    ...
}
```
### 基于名称的HTTPS服务
```
server {
    listen          443 ssl;
    server_name     www.example.com;
    ssl_certificate www.example.com.crt;
    ...
}

server {
    listen          443 ssl;
    server_name     www.example.org;
    ssl_certificate www.example.org.crt;
    ...
}
```
基于IP：
```
server {
    listen          192.168.1.1:443 ssl;
    server_name     www.example.com;
    ssl_certificate www.example.com.crt;
    ...
}

server {
    listen          192.168.1.2:443 ssl;
    server_name     www.example.org;
    ssl_certificate www.example.org.crt;
    ...
}
```
## 使用HTTPS建立TCP连接
nginx负责https的认证等操作，并把数据返回给后端的服务器处理，后端服务器是http服务器：
```
stream {
    upstream stream_backend {
         server backend1.example.com:12345;
         server backend2.example.com:12345;
         server backend3.example.com:12345;
    }
 
    server {
        listen                12345 ssl;
        proxy_pass            stream_backend;
 
        ssl_certificate       /etc/ssl/certs/server.crt;
        ssl_certificate_key   /etc/ssl/certs/server.key;
        ssl_protocols         SSLv3 TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers           HIGH:!aNULL:!MD5;
        ssl_session_cache     shared:SSL:20m;
        ssl_session_timeout   4h;
        ssl_handshake_timeout 30s;
		ssl_session_tickets on;
    …
     }
}
```
session Ticket 是另一种session缓存。session信息存储在客户端，减少了服务端存储session的压力。当客户端恢复与服务端的交互后，它会提供session并且可以不用再进行握手之类的操作。

## 为nginx和后端服务器的HTTP连接加密
```
http {
    ...
    upstream backend.example.com {
        server backend1.example.com:443;
        server backend2.example.com:443;
   }

    server {
        listen      80;
        server_name www.example.com;
        ...

        location /upstream {
            proxy_pass                    https://backend.example.com;
            proxy_ssl_certificate         /etc/nginx/client.pem;
            proxy_ssl_certificate_key     /etc/nginx/client.key
            proxy_ssl_protocols           TLSv1 TLSv1.1 TLSv1.2;
            proxy_ssl_ciphers             HIGH:!aNULL:!MD5;
            proxy_ssl_trusted_certificate /etc/nginx/trusted_ca_cert.crt;

            proxy_ssl_verify        on;
            proxy_ssl_verify_depth  2;
            proxy_ssl_session_reuse on;
        }
    }

    server {
        listen      443 ssl;
        server_name backend1.example.com;

        ssl_certificate        /etc/ssl/certs/server.crt;
        ssl_certificate_key    /etc/ssl/certs/server.key;
        ssl_client_certificate /etc/ssl/certs/ca.crt;
        ssl_verify_client      off;

        location /yourapp {
            proxy_pass http://url_to_app.com;
        ...
        }

    server {
        listen      443 ssl;
        server_name backend2.example.com;

        ssl_certificate        /etc/ssl/certs/server.crt;
        ssl_certificate_key    /etc/ssl/certs/server.key;
        ssl_client_certificate /etc/ssl/certs/ca.crt;
        ssl_verify_client      off;

        location /yourapp {
            proxy_pass http://url_to_app.com;
        ...
        }
    }
}
```

## 为nginx和后端服务器的TCP连接加密
```
http {
    ...
    upstream backend.example.com {
        server backend1.example.com:443;
        server backend2.example.com:443;
   }

    server {
        listen      80;
        server_name www.example.com;
        ...

        location /upstream {
            proxy_pass                    https://backend.example.com;
            proxy_ssl_certificate         /etc/nginx/client.pem;
            proxy_ssl_certificate_key     /etc/nginx/client.key
            proxy_ssl_protocols           TLSv1 TLSv1.1 TLSv1.2;
            proxy_ssl_ciphers             HIGH:!aNULL:!MD5;
            proxy_ssl_trusted_certificate /etc/nginx/trusted_ca_cert.crt;

            proxy_ssl_verify        on;
            proxy_ssl_verify_depth  2;
            proxy_ssl_session_reuse on;
        }
    }

    server {
        listen      443 ssl;
        server_name backend1.example.com;

        ssl_certificate        /etc/ssl/certs/server.crt;
        ssl_certificate_key    /etc/ssl/certs/server.key;
        ssl_client_certificate /etc/ssl/certs/ca.crt;
        ssl_verify_client      off;

        location /yourapp {
            proxy_pass http://url_to_app.com;
        ...
        }

    server {
        listen      443 ssl;
        server_name backend2.example.com;

        ssl_certificate        /etc/ssl/certs/server.crt;
        ssl_certificate_key    /etc/ssl/certs/server.key;
        ssl_client_certificate /etc/ssl/certs/ca.crt;
        ssl_verify_client      off;

        location /yourapp {
            proxy_pass http://url_to_app.com;
        ...
        }
    }
}
```