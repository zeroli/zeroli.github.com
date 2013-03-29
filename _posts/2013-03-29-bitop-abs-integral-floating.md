---
layout: post
title: "只用BIT操作，求解整型和浮点数的绝对值"
description: ""
category: cpp
tags: [C++]
style: "font-family: MS Sans Serif; font-size:18px"
---
{% include JB/setup %}

Q: Given one integer number, how to solve its absolute with bitwise op, and how about floating number?  
  
A:  
Q1: For integer numbers:  
1. For positive number, its abs value is still itself  
2. For negative number, its abs value will ~(x-1).

How to combine above expression, without any branching, including ternary op(:?).

Code as below:  
{% highlight cpp %}
#define BITS_PER_BYTE 8
int myabs(int x)
{
	int mask = (x >> (BITS_PER_BYTE * sizeof(int)&nbsp;- 1));  // shift right with signed bit 
	return (x + mask) ^ mask;
}
{% endhighlight %}

For negative number, mask is -1, while 0 for positive. ^mask op is like ~ bit op.  

Q2: For floating number, here take example of float type: 

![](/assets/image/1358319235_7473.png)

So abs floating number, only needs abs the MSB bit.  
{% highlight cpp %}
float myfabs(float f)
{
    int* temp = (int*)&f;
    int mask = ~(1 << (BITS_PER_BYTE * sizeof(int) - 1));  // only MSB bit is 1, other bits are 0
    int out = *temp & mask;  // set MSB bit to 0
    return *((float*)&out);
}
{% endhighlight %}

For double type, representation is similar, totally 64 bits, 11 bits for magnitude(exp), 52 bits for fraction.
