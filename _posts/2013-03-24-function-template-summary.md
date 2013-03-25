---
layout: post
title: "函数模板总结"
description: "函数模板代表一个函数家族"
category: cpp
tags: [C++ templates]
---
{% include JB/setup %}


函数模板代表一个函数家族。  
通常而言，并不是把模板编译成一个可以处理任何类型的单一实体，而是对于实例化模板参数的每种类型，都从模板产生一个不同的实体。  

关于模板编译，会有2个过程，分别发生在：（请注意VC++与GCC之间的不同）  
1. 实例化之前，先检查模板代码本身，查看语法是否正确，在这里会发现错误的语法，如遗漏分号等。  
2. 在实例化期间，检查模板代码，查看是否所有的调用都有效。在这里会发现无效的调用，如该实例化类型不支持某些函数调用等。  

函数模板有两种类型的参数：  
1. 模板参数： 位于函数模板名称的前面，在一对尖括号内部进行声明：  
{% highlight cpp linenos %}
template <typename T>
{% endhighlight %}
2. 用参数：位于函数模板名称之后，在一对圆括号内部进行声明：  
{% highlight cpp linenos %}
max(T const& a, T const& b);
{% endhighlight %}
可以额根据需要声明任意数量的模板参数。然而，在函数模板内部（这一点和类模板不同），不能指定缺省的模板实参。
当模板参数和调用参数没有发生关联，或者不能有调用参数来决定模板参数的时候，你在调用时就必须显式指定模板实参。
然而，模板实参演绎并不适合返回类型，因为RT不会出现在函数调用参数的类型里面，因为，函数调用并不能推导出RT，也就是必须显式地指定模板实参列表。

重载函数模板  
{% highlight cpp linenos %}
inline int const& max(int const& a, int const& b)
{
    return a < b ? b : a;
}


template <typename T>
inline T const& max(T const& a, T const& b)
{
    return a < b ? b : a;
}
template <typename T>
inline T const& max(T const& a, T const& b, T const& c)
{
   return ::max(::max(a, b), c);
}

int main()
{
    ::max(7, 42, 68);   // 调用3个参数的模板
    ::max(7.0, 42.0);   // 调用max<double>
    ::max('a', 'b');    // 调用max<char>
    ::max(7, 42);       // 调用int重载的非模板函数
    ::max<>(7, 42);     // 调用max<int>
    ::max<double>(7, 42);   // 调用max<double> (没有实参推导)
    ::max('a', 42.7);   // 调用int重载的非模板实参
}
{% endhighlight %}
因为模板是不允许自动类型转化的；但普通函数可以进行自动类型转换，所以最后一个调用将会使用非模板函数。   
一般而言，在重载函数模板时候，最好只是改变那些需要改变的内容：就是说，应该把你的改变限制在两种情况:改变参数的数目或者显示的指定模板参数。