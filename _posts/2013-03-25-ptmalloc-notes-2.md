---
layout: post
title: "ptmalloc代码浅析2"
description: ""
category: "memory_management"
tags: [malloc, glibc代码阅读]
---
{% include JB/setup %}

ptmalloc中的small bin和large bin维护着不同结构的chunk链表：  
##small bin
每个small bin维护着一个双向循环链表，而且chunk size都相同。当allocate memory时，总是从链表尾端unlink一个chunk，deallocate memory时，将那个chunk link到链表头。属于FILO规则，每个chunk都有机会被分配到app。  
![](/assets/image/1345812081_4573.png)

##large bin
每个large bin维护着两种链表，一种是横向的双向循环链表，类似于small bin中的链表，唯一不同的是large bin中的chunk size是从大到小有序的，另一种是纵向的双向链表。在横向链表中，每个chunk作为纵向双向链表的头。各个纵向链表中chunk size相同。  
![](/assets/image/1345812086_6124.png)
a) 在上面的结构图中，横向的循环双向链表是以fd\_nextsize和bk\_  nextsize域串接起来，而纵向的双向链表是以fd和bk域串接起来的。图中横向链表也叫做chunk size链表，最大chunk的bk_nextsize域指向最小size的chunk，反过来最小size的chunk的fd_nextsize域指向最大size的chunk。  
b) 当有chunk要放入到某个large bin的链表中时，首先以chunk size从到小到大的，按照bk_nextsize的顺序来查找合适的插入位置（找到第一个chunk size大于等于这个chunk的size的纵向chunk链表），如果碰到相同chunk size的chunk纵向链表，则将这个chunk插入到纵向链表的第二个位置（这样是为了不进行额外的fd\_nextsize/bk\_nextsize赋值操作），否则将这个chunk作为独立的纵向chunk链表，插入到chunk size链表中。
