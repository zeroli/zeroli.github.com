---
layout: post
title: "glibc中几个数值处理函数"
description: ""
category: "cpp"
tags: [glibc代码阅读]
---
{% include JB/setup %}

ceil/floor/rint/round，这几个C的数值处理函数，我们通常用它们来取整某个特定的浮点数。


— Function: double ceil (double x)  

These functions round x upwards to the nearest integer, returning that value as a double. Thus, ceil (1.5) is 2.0.  


— Function: double floor (double x)  


These functions round x downwards to the nearest integer, returning that value as a double. Thus, floor (1.5) is 1.0 and floor (-1.5) is -2.0.  


— Function: double rint (double x)  

These functions round x to an integer value according to the current rounding mode.The default rounding mode is to round to the nearest integer; some machines support other modes, but round-to-nearest is always used unless you explicitly select another.  


— Function: double round (double x)  

These functions are similar to rint, but they round halfway cases away from zero instead of to the nearest integer (or other current rounding mode).  


__Examples:__  

1. 请注意rint(+/-0.5)和round(+/-0.5)之间的区别(对于+/-0.5，round函数总是返回远离0的值):  
![](/assets/image/glibc-arith-func-results1.png)

2. rint和round此时相同：
![](/assets/image/glibc-arith-func-results2.png)