---
title: 'Nginx学习笔记: 一、入门'
date: 2016-07-11 10:32:41
tags: Nginx
---

## 安装
安装nginx需要先安装c++ 编译器、pcre 、zlib以及openssl。
1. 安装c++ 编译器
`yum install -y gcc gcc-c++`
2. 安装pcre
```
$ wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.39.tar.gz
$ tar -zxf pcre-8.39.tar.gz
$ cd pcre-8.39
$ ./configure
$ make
$ sudo make install
```

3. 安装zlib
```
$ wget http://zlib.net/zlib-1.2.8.tar.gz
$ tar -zxf zlib-1.2.8.tar.gz
$ cd zlib-1.2.8
$ ./configure
$ make
$ sudo make install
```
4. 安装openssl
```
$ wget http://www.openssl.org/source/openssl-1.0.2f.tar.gz
$ tar -zxf openssl-1.0.2f.tar.gz
$ cd openssl-1.0.2f
$ ./configure  --prefix=/usr/local/openssl
$ make
$ sudo make install
```

5. 安装nginx
先下载
```
$ wget http://nginx.org/download/nginx-1.11.2.tar.gz
$ tar zxf nginx-1.11.2.tar.gz
$ cd nginx-1.11.2
```
编译安装
```
$ ./configure --with-pcre=../pcre-8.39 --with-zlib=../zlib-1.28 --with-openssl=../openssl
$ make & make install
```

6. 启动nginx 
```
$ cd /usr/local/nginx/sbin
$ ./nginx
```
验证是否启动成功了
`curl -I 127.0.0.1`
![](http://i.imgur.com/HbB3BQh.png)