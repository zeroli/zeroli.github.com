---
layout: post
title: "2D平面上n个点，求解最接近45度的2点连线 "
description: ""
category: "Data_Structure_and_Algorithm"
tags: [算法, 计算几何]
---
{% include JB/setup %}

有这么一个几何问题：2d平面的n个点，怎么求解连线最接近45度的那些点的组合。


##1. 朴素方法：

我们能想到的最naitive的方法就是求解平面上所有2点连线的斜率，斜率的绝对值最接近于1的那些点的组合，就是我们想要的。要知道n个点能构成n^2条线段，故这种方法的时间复杂度就是O(n^2)。但是我们期望有更好的，时间复杂度较小的解决方法。


##2. 一个比较好的方法

如果熟悉计算几何常用算法的话，类似于这类问题，通常期望更好的时间复杂度会是`O(n * lgn)`。这里我们基于该问题的特有属性，来设计一个时间复杂度是`O(n * lgn)`的算法。


算法描述:

a. 以原点(0, 0)为中心，顺时针旋转所有的点到原点的向量45度。这样所有的点都是在逆时针旋转45度之后的坐标系统中。这个过程可以在O(n)时间内完成。

b. 这样原始问题可以转化为，所有点组成的线段中，求解斜率最接近于0的那些点组成的线段。关于这个问题，我们就比较好的方法了。

I. 对所有的点，以y轴坐标进行排序，y轴坐标相同的点，以x轴排序。这个过程可以用基本的快排来完成，期待时间会是`O(n * lgn)`

II. 按照y值两两组合的依次遍历这些点，求解它们的斜率`slope=(y2-y1)/(x2-x1)`。记录那些线段的斜率绝对值最小的那些点组合。这个过程可以在O(n)时间内完成。

c. b.II中记录的那些点组合中点，逆时针旋转45，可以得到它们原始的点坐标。这些点组合就是原始问题中的输出。


上述算法总的运行时间会是`O(n * lgn)`。

算法的关键点在于为啥斜率绝对值最小的线段来源于y值连续的两点。下面从反面来证明下:

假定这两点是p1(x1, y1), p2(x2, y2), 满足y1 <= y2。论证如果p1,p2在y值上连续，所有点与p1的连线斜率都将不小于p1p2的连线。从反面来证明。假设p1p2组成的线段的斜率最小，但是y值不连续，这条连线将平面分成了两部分（上和下）。

1) 存在一个点p3(x3, y3)，满足y1 < y3 < y2。如果p3正好在p1和p2的连线上，那么斜率是一样的，我们应该选择p1p3两点，或者p3p2，而不是p1p2。得证。

2) 假如p3位于p1p2直线上，满足y1 < y3 < y2。那么p3p2组成的直线的斜率要小于p1p2，而且在y值上p3比p1更接近于p2，与前提p1p2斜率最小不符合，得证。

3) 假如p3位于p1p2直线下，满足y1 < y3 < y2。那么p1p2组成的直线的斜率要小于p1p2，而且在y值上p3比p2更接近于p1，与前提p1p2斜率最小不符合，得证。

有了上述3个反证，我们可以得知对于一个点来说，斜率最小的直线一定来自于与它y值最邻近的点。


##3. 实现考虑

下面是一个简单的python语言的实现：
{% highlight cpp %}
#!/usr/bin/env python

import math

def qsort_byY(L):
    if L == []: return []
    return qsort_byY([p for p in L[1:] if p[1] < L[0][1]]) + L[0:1] + \
        qsort_byY([p for p in L[1:] if p[1] >= L[0][1]])

def ccw_rotatept(p, angle):
    angle = angle * math.pi / 180
    x = p[0] * math.cos(angle) - p[1] * math.sin(angle)
    y = p[1] * math.cos(angle) + p[0] * math.sin(angle)
    p[0], p[1] = x, y

def ccw_rotate(L, angle):
    for p in L:
        ccw_rotatept(p, angle)

def solve_closest_45_lines(L):
    print("original points:")
    print(L)
    ccw_rotate(L, -45) # clockwise rotate all points by 45
    print("after rotate")
    print(L)
    qsort_byY(L)
    print("after sort w.r.t. y")
    print(L)

    outp = []
    minslope = 1e100
    for k in range(len(L) - 1):
        i,j = k,k + 1
        curslope = math.fabs((L[j][1] - L[i][1]) \
                           / (L[j][0] - L[i][0]))
        if minslope > curslope:
            del outp[:]
            outp.append([L[i], L[j]])
            minslope = curslope
        elif minslope == curslope:
            outp.append([L[i], L[j]])

    print("output pairs of points:")
    for k in range(len(outp)):
        ccw_rotatept(outp[k][0], 45)
        ccw_rotatept(outp[k][1], 45)
        print outp[k]

if __name__ == "__main__":
    solve_closest_45_lines([[0,0], [10,12],[3,100],[4,102]])
{% endhighlight %}