---
layout: post
title: "求解最长回文子字符串"
description: ""
category: "Data_Structure_and_Algorithm"
tags: [算法, string]
---
{% include JB/setup %}

题目：输入一个字符串，输出该字符串中对称的子字符串的最大长度。

比如输入字符串google，由于该字符串里最长的对称子字符串是goog，因此输出4


关于这个问题，有一个比较好的线性时间复杂度的算法Manacher's ALGORITHM

这2篇技术博客非常好的阐述了它的算法思想：

1) http://www.felix021.com/blog/read.php?2040 (中文)

2) http://www.leetcode.com/2011/11/longest-palindromic-substring-part-ii.html


更多的博客文章可供参考：

http://www.akalin.cx/longest-palindrome-linear-time


C++代码实现：
{% highlight cpp %}
// longest palindromic substring
// http://www.felix021.com/blog/read.php?2040
// Manacher's ALGORITHM: O(n)
std::string lps(const std::string& str)
{
    std::cout << "original string is " << str << std::endl;
    std::string ss = "$";
    std::ostringstream ostr;
    for (int i = 0; i < str.length(); i++)
    {   
        ostr << '#' << str[i];
    }   
    ostr << "#";
    ss += ostr.str();
    std::cout << "new string is " << ss << std::endl;

    int* p = new int[ss.length()];
    *p = 0;

    int id = 0, mix = 0;
    for (int i = 1; i < ss.length(); i++)
    {   
        p[i] = (mix > i) ? std::min(p[2*id - i], mix-i) : 1;
        while (ss[i + p[i]] == ss[i-p[i]]) p[i]++;
        if (p[i] + i > mix)    
        {   
            mix = p[i] + i;
            id = i;
        }   
    }   
    
    int* pmax = std::max_element(p, p+ss.length());
    int ipos = (pmax - p)/2;
    int len = *pmax - 1;
   std::string lpsstr = str.substr(ipos-len/2, len);

    delete [] p;
    return lpsstr;
}
{% endhighlight %}
