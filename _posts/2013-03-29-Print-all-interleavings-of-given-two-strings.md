---
layout: post
title: "Print all interleavings of given two strings"
description: ""
category: "Data_Structure_and_Algorithm"
tags: [算法]
---
{% include JB/setup %}

Given two strings str1 and str2, write a function that prints all interleavings of the given two strings. You may assume that all characters in both strings are different  
Example:  
> Input: str1 = "AB",  str2 = "CD"  
> Output:  
>   ABCD  
>   ACBD  
>   ACDB  
>   CABD  
>   CADB  
>   CDAB  
>  
> Input: str1 = "AB",  str2 = "C"  
> Output:  
>   ABC  
>   ACB  
>   CAB  

An interleaved string of given two strings preserves the order of characters in individual strings. For example, in all the interleavings of above first example, ‘A’ comes before ‘B’ and ‘C’ comes before ‘D’.  

这个题目可以这样来求解出一种方案：  
字符串str1长度为m(str1\[0:m\])，字符串str2长度为n(str2\[0:n\])，那么最终求解的字符串长度肯定为m+n，占用m+n的字节空间或者说是数组。那么可以这样来理解，在最终的m+n个单元的输出数组中，每个单元的元素要么来自于str1 ，要么就来自于str2。比如说，如果第一个元素来自于str1的第一个字符str1\[0\]，那么剩下的m+n-1个单元格的元素要么来自于str1\[1:m\]，要么来自于str2\[0:n\]，这样这个子问题类似于最初的问题；或者说，如果第一个元素来自于str2的第一个字符str2\[0\]，那么剩下的m+n-1个单元格的元素要么来自于str1\[0:m\]，要么来自于str2\[1:n\]。将这样的子方案综合，可以得到最终的原始方案。标记原始方案的结果为K(m,n)，其中m是str1的长度，n是str2的长度，所以有：  
**K(m, n) = K(m-1, n) + K(m, n-1)**

C代码如下：
{% highlight cpp %}
void interleave(char* str1, char* str2, char* str, int len)
{
    if (*str1 == '\0' && *str2 == '\0')
    {
        printf("%d: %s\n", count+1, str-len);
        count++;
        return;
    }
    if (*str1 != '\0')
    {
        str[0] = str1[0];
        interleave(str1+1, str2, str+1, len);
    }
    if (*str2 != '\0')
    {
        str[0] = str2[0];
        interleave(str1, str2+1, str+1, len);
    }
}
{% endhighlight %}
关于这个问题，还有其它的解决方案：  
[http://stackoverflow.com/a/6805122/1793022](http://stackoverflow.com/a/6805122/1793022)
