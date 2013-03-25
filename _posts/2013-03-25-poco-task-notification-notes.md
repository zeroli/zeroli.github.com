---
layout: post
title: "Poco中的task与notification的关系"
description: ""
category: "poco"
tags: [C++, poco源码阅读]
---
{% include JB/setup %}

[Poco](http://blog.csdn.net/zero_lee/article/details/pocoproject.org/)中，Task与Notification之间的关系图如下：
![](/assets/image/1345812246_3183.png)

TaskManager维护着一个NotificationCenter，客户端代码向TaskManager添加不同类型的Task，故而TaskManager维护着一个Task List。TaskManager借用默认的Thread Pool或者定制的Thread Pool来启动每个Task在独自的线程中运行，Task运行的状态信息，TaskManager通过NotificationCenter发送出去，发送给那些在NotificationCenter注册过的Observers。
