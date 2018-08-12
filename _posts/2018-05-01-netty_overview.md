---
title: Netty简介 
tags: netty
grammar_cjkRuby: true
---


Netty是一个高性能、异步的、基于事件驱动的网络应用开发框架，它基于Java NIO进行封装, 提供更高层次的抽象接口，让开发者使用它开发网络应用TCP, UDP,  HTTP非常简便、快速。当前好多流行的开源软件（zookeeper, Cassandra，Elasticsearch, vertx）都是把Netty作为网络通信的框架，所以可认为目前是网络应用框架的首选。

整个Netty API都是异步的，对于网络应用于来，IO一般是性能瓶颈，使用异步IO能够很大程度的提升系统的性能，当有事件发生时获得通知，然后执行对应的动作，这样就不会阻塞，提升了系统资源利用率

Netty提供了丰富的功能，下图是Netty框架的组成：

[history]: images/history.png

同时netty解决了Java NIO中存在的一些问题：

 1. 多版本的跨平台和兼容性问题，Netty提供一个统一的接口，同一语义无论在Java6还是Java7的环境下都是可以运行的，开发者无需关心底层APIs就可以轻松实现相关功能
 2. 扩展ByteBuffer，通过一些简单的APIs对ByteBuffer进行构造、使用和操作，来解决NIO中的一些限制。
 3. NIO对缓冲区的聚合和分散操作存在内存泄漏的可能
 4. 著名epoll的缺陷
