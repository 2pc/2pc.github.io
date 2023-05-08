---
title: Dubbo线程池问题汇总
tagline: ""
category : dubbo
layout: post
tags : [dubbo]
---

+ 每个连接一个线程池ip+port,多个连接还要*connection
1.[【2013-Need a limited Threadpool in consumer side】](https://github.com/apache/dubbo/issues/2013),优化方案在这里[R【educe context switching cost by optimizing thread model on consumer side】](https://github.com/apache/dubbo/pull/4131)
consumer端，每个连接一个线程池
2.[i【ssues4467-why does dubbo creates a new ThreadPool for each connection】](https://github.com/apache/dubbo/issues/4467)
+ 端口共享线程池
1.dubbo线程池按照端口分配后仍然有不少问题，比如这个[7054](https://github.com/apache/dubbo/issues/7054)碰到有节点下线会不断关闭与重建线程池，修复[PR-7109](https://github.com/apache/dubbo/pull/7109)