---
layout: post
title: "非类型模板参数知识点梳理"
description: ""
category: cpp
tags: [C++ templates]
---
{% include JB/setup %}

对于函数模板和类模板，模板参数并不局限于类型，普通值也可以作为模板参数。

###非类型的类模板
{% highlight cpp %}
template <typename T, int MAXSIZE>
class Stack {
private:
    T elems[MAXSIZE];
...
};


template <typename T, int MAXSIZE>
Stack<T, MAXSIZE>::Stack()
: numElems(0)
{
}
{% endhighlight %}

###非类型的函数模板参数
{% highlight cpp %}
template <typename T, int VAL>
T addValue(T const& v)
{
    return x + VAL;
}


std::transform(v.begin(), v.end(), v.begin(), addValue<int, 5>);
};
{% endhighlight %}

###非类型的模板参数的限制
非类型的模板参数是有限制的。通常而言，它们可以是常整数（包括枚举类型）或者指向外部链接对象的指针。  
**浮点数和类对象是不允许作为非类型模板参数的。**  
例如下面2个模板类是编译不过的： 
{% highlight cpp %}
template <double VAT>
double process(double v)
{
    return v * VAT;
}
template <std::string name>
class MyClass {
...
};
{% endhighlight %}
由于字符串文字是内部链接对象（因为两个具有相同名称但处于不同模板的字符串，是两个完全不同的对象），所以不能使用它们来作为模板实参：  
{% highlight cpp %}
template <const char* name>
class MyClass {
...
};


MyClass<"hello"> x; // error
{% endhighlight %}

另外，也不能使用全局指针作为模板参数：  
{% highlight cpp %}
template <const char* name>
class MyClass {
...
};


const char* s = "hello";
MyClass<s> x;   // error
{% endhighlight %}
**然而，可以这样使用：**  
{% highlight cpp %}
template <const char* name>
class MyClass {
...
};


extern const char s[] = "hello";
MyClass<s> x; // ok
{% endhighlight %}
全局字符数组s由"hello"初始化，是一个外部链接对象。



