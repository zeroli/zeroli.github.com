---
layout: post
title: "2个浮点数的最大公约数的算法小结 "
description: ""
category: "Data_Strucutre_and_Algorithm"
tags: [算法, lua]
---
{% include JB/setup %}

我们知道2个整数有最大公约数(GCD)和最小公倍数(LCM)的说法。而且最大公约数GCD有一个公认的欧拉算法GCD。它的算法如下:
{% highlight cpp %}
GCD(a, b) 
{
    return b ? GCD(b, a % b) : a;
}
{% endhighlight %}
有了GCD，当然也很好求解LCM。

现在，问题来了，给定2个浮点数，怎么求解它们的GCD？


##数学上的方法

一个比较的想法就是，既然我们已有的工具是整数的GCD求法，那么我们就想办法将浮点数转化为整数。那如何将一个浮点数转化为一个整数来表示呢，甚者说用几个整数来表示呢？我们知道，浮点数m.n是由整数部分m和小数部分n组成的，这个是我们小学学习的东西（还好没还给老师）。一种转化方法就是将小数部分n乘以一个10幂次方数，幂为小数点后即n的数字个数，就是`m + n * 10^(n.length)`。所以m.n就可以表示为`(m.n*10^(n.length)) / (10^(n.length))`。

这种方法也就是将浮点数转化为它的另一种形式：分数(fraction)。分数由分子(numerator)和分母(denomitor)组成(又是小学的知识)。


有了一个浮点数可以转化为分数的方法，那么我们可以将2个浮点数都转化为它们的最简的分数形式。回到最初的问题（2个浮点数的GCD）：我们可以知道2个浮点数中小数的最长位数，然后都乘以10次幂的这个数，让2个浮点数转化为2个整数，然后在整数下求解GCD，之后再除以之前那个10次幂数。

更一般思路是：将2个浮点数转化为2个最简分数（这里是最简分数），然后将2个分数进行通分（就是求解2个分数的分母的LCM），之后再求解2个分子的GCD。最后得到的分子的GCD与分母的LCM表示的分数即为最终2个浮点数的GCD的分数表示形式。


##程序实现

有了上面数学上的方法，程序上实现应该来说不是难事。然而还是有些地方需要注意的。

上述方法中，最重要的就是浮点数转化为分数的那一步。如果采用将2个浮点数同乘以一个10次幂数来转化整数，程序上可能就会有点麻烦。

a) 我们得知道那2个浮点数小数部分的有多少位。

b) 另外知道位数后，有一种怀疑就是这个位数是否准确。因为计算机表示浮点数总是不精确的，比如1.0，可能会表示为0.999999999...，也可能表示为1.000...1。如果贸然将它强行转化为整数，得到的可能并不是我们所期望的。

所以说，上面简单的方法行不通，我们得寻找一个精度更高的方法，来将浮点数转化为分数。

数学上有一个方法叫做[**continued fraction**](http://en.wikipedia.org/wiki/Continued_fraction)，它说任何一个数，都可以表示成它的整数部分与另一个数的倒数之和，然后那另一个数，又可以表示成它的整数部分和另一个数的倒数之和，一直这样下去：
> In mathematics, a continued fraction is an expression obtained through an iterative process of representing a number as the sum of its integer part and the reciprocal of another number, then writing this other number as the sum of its integer part and another reciprocal, and so on.

这种方法，可以满足我们程序在精度上的需求。


LUA代码实现如下：  
{% highlight cpp %}
cntFrac = {}
function cntFrac.caclGCD(a, b)
   local nu, de = cntFrac.float2frac(a, 1e-8, 10)
   local cnu, cde = cntFrac.float2frac(b, 1e-8, 10)
   cnu, cde = cntFrac.getGCD(nu, de, cnu, cde)
   return cnu, cde

function cntFrac.float2frac(x, eps, n)
    local i = 1
    local fx, err = x, 1.
    local nu, de = 0, 1
    local a = {}
    a[1] = math.floor(x)
    fx = x - a[1]
    x, err = fx, fx
    while math.abs(err) > eps and i < n do
        i = i + 1
        fx = 1./ fx
        a[i] = math.floor(fx)
        fx = fx - a[i]
        nu, de = cntFrac.continuedFrac(a, i)
        err = x - nu / de
    end
    nu = nu + a[1] * de
    nu = math.floor(nu + 1e-10)
    de = math.floor(de + 1e-10)
    return nu, de
end

function cntFrac.continuedFrac(a, n)
    if 1 == n then return 0, 1 end
    local nu, de = 1, a[n]
    local c
    for i = n - 1, 2, -1 do
        nu = nu + de * a[i]
        c = nu; nu = de; de = c
        nu = math.floor(nu + 1e-10)
        de = math.floor(de + 1e-10)
    end
    return nu, de
end

function cntFrac.getGCD(a1, b1, a2, b2)
    local db = cntFrac.gcd(b1, b2);
    db = math.floor(b1 * b2 / db + 1e-10) -- LCM for denominator
    a1 = a1 * math.floor(db / b1 + 1e-10)
    a2 = a2 * math.floor(db / b2 + 1e-10)
    local da = cntFrac.gcd(a1, a2)          -- GCD for numerator
    return da, db
end

function cntFrac.gcd(a, b)
    a = math.floor(a + 1e-10)
    b = math.floor(b + 1e-10)
    if (b < 1e-8) then
        return a
    else
        return cntFrac.gcd(b, math.mod(a, b))
    end
end

{% endhighlight %}