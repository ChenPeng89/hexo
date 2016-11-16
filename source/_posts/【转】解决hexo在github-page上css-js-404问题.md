---
title: 【转】解决hexo在github page上css/js 404问题
date: 2016-11-16 16:51:31
tags: [hexo]
---

## 问题

最近github page更新了，GitHub Pages 过滤掉了 source/vendors 目录的访问，所以next主题下的source下的vendors目录不能够被访问到，所以就出现了本地hexo s能够正常访问，但是deploy到github就是一片空白，按f12，可以看到大量来自source/vendors的css和js提示404。

## 解决方法

方法一（来自github next主题issue）:

找到解决方案了。。 @BBBOND @monsterLin @SpadeRoy 根据作者的提示 @iissnan ，首先修改source/vendors为source/lib，然后修改_config.yml， 将 _internal: vendors修改为_internal:lib 然后修改next底下所有引用source/vendors路径为source/lib。这些地方可以通过文件查找找出来。主要集中在这几个文件中。1. Hexo\themes\next.bowerrc 2. Hexo\themes\next.gitignore 3. Hexo\themes\next.javascript_ignore 4. Hexo\themes\next\bower.json 。修改完毕后，刷新重新g一遍就ok啦。
    

方法二:更新next主题，不过听过最新的next主题对第三方例如多说删除了，具体不清楚，不敢亲易尝试，毕竟更新一次主题引来的问题太多，很多配置可能都要改，代价太高，所以推荐第一种方法
参考

    https://github.com/hexojs/hexo/issues/2238
    https://github.com/iissnan/hexo-theme-next/issues/1214


转自： [Hexo+Github博客css js404导致博客页面空白](http://blog.csdn.net/zhouzixin053/article/details/53038679)