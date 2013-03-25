---
layout: post
title: "[Lua源码阅读] 理解Lua的object"
description: ""
category: "lua"
tags: [lua源码阅读, script]
---
{% include JB/setup %}

Lua的object定义在lobject.h文件中。Lua中总有有9种数据类型：nil, boolean, lightuserdata, number, string, table, userdata, thread, function，用数字0-8来表示：
{% highlight cpp %}
#define LUA_TNIL		0
#define LUA_TBOOLEAN		1
#define LUA_TLIGHTUSERDATA	2
#define LUA_TNUMBER		3
#define LUA_TSTRING		4
#define LUA_TTABLE		5
#define LUA_TFUNCTION		6
#define LUA_TUSERDATA		7
#define LUA_TTHREAD		8
{% endhighlight %}
除了以上9中基本数据类型之外（他们是对外的基本类型），Lua还定义了一些类型的变种：
{% highlight cpp %}
/*
** tags for Tagged Values have the following use of bits:
** bits 0-3: actual tag (a LUA_T* value)
** bits 4-5: variant bits
** bit 6: whether value is collectable
*/


#define VARBITS		(3 << 4)


/*
** LUA_TFUNCTION variants:
** 0 - Lua function
** 1 - light C function
** 2 - regular C function (closure)
*/


/* Variant tags for functions */
#define LUA_TLCL	(LUA_TFUNCTION | (0 << 4))  /* Lua closure */
#define LUA_TLCF	(LUA_TFUNCTION | (1 << 4))  /* light C function */
#define LUA_TCCL	(LUA_TFUNCTION | (2 << 4))  /* C closure */


/*
** LUA_TSTRING variants */
#define LUA_TSHRSTR	(LUA_TSTRING | (0 << 4))  /* short strings */
#define LUA_TLNGSTR	(LUA_TSTRING | (1 << 4))  /* long strings */
{% endhighlight %}

TValue的内部结构图如下：

![](/assets/image/1345812366_9492.png)

int tt\_是lua对象的具体类型，value_才是具体存储的值，对于一般的类型，可以直接表示，而对于一些特殊的类型，譬如user data/thread/string，得用GCObject union来表示。
