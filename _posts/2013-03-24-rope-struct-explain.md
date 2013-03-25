---
layout: post
title: "rope算法学习"
description: ""
category: "Data_Structure_and_Algorithm"
tags: [glib代码阅读]
---
{% include JB/setup %}

rope是粗绳索的意思，在计算机编程世界里，它可以替代string来处理超长字符串的相关操作，比如联接，插入，删除，因此它常用于text editor这样的应用中。   
它的[wiki主页](http://en.wikipedia.org/wiki/Rope_(computer_science) )简要描述了它的机理：

这个算法来源于一篇paper: Boehm, Hans-J; Atkinson, Russ; and Plass, Michael (December 1995).
[Ropes: an Alternative to Strings](http://citeseer.ist.psu.edu/viewdoc/download?doi=10.1.1.14.9450&rep=rep1&type=pdf)

它的大部分接口的实现复杂为O(log(n)) (algorithmic)级别。即使这样，相对于简单的指针和长度构成的模型来说，实现这样的数据结构复杂度相当大。  
SGI STL很早就实现了rope结构，但是它不属于STL C++标准，所以在GCC中，程序如果要用这样的结构体，需要  
`#include <ext/rope>`  
接口类似于std::vector接口：`rope<char, gnu_allocator<type> >`。
  
像std::string一样，它本身就是一个类模板，已经提供了crope和wrope实例化版本供使用。

阅读采用的SGI STL版本是STL-3.3。主要包含2个文件：stl_rope.h和ropeimpl.h，后者被前者include。对外头文件是rope。大部分的类定义都在stl_rope.h中，ropeimpl.h实现了所有未被实现在stl_rope.h中的函数。rope实现相当复杂，涉及到的类相当多。
![](/assets/image/1345811957_8305.png)

