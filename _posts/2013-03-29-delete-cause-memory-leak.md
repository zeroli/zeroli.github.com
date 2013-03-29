---
layout: post
title: "delete p;而不是delete [] p; 真的会导致内存泄漏吗？"
description: ""
category: cpp
tags: [C++]
---
{% include JB/setup %}

给定下面的C++代码片段：

{% highlight cpp %}
class A {    
public:
    int m_data[10];
};

A* pA = new A[20];
delete pA;
{% endhighlight %}

我们知道这里应该用`delete [] pA;`，但是`delete pA;`真的会导致内存泄漏吗？  
要回答这个问题，我们得从`delete pA;`和`delete [] pA;`之间的区别说起。  
delete 和delete []是2个C++操作符，这是标准规定的。当编译器比如GCC编译这2个操作符时，编译后步骤如下：  
delete操作符：  
1) 首先调用pA所指向对象的析构函数(这个析构函数有可能是动态决定的，如果发现A实现了虚析构函数)。  
2) 调用`operator delete(void*)`全局函数来释放堆空间。而`delete []`操作符，先循环调用pA指向的数组空间中的每个数组元素A的析构函数，同样有可能是动态决定的虚析构函数，之后调用`operator delete [](void*)`全局函数来释放整个堆空间。  
以下是`operator delete(void*)` 和`operator delete[](void*)`的一种实现：   
{% highlight cpp %}
__attribute__((__weak__, __visibility__("default")))
void
operator delete(void* ptr) _NOEXCEPT
{
    if (ptr)
        ::free(ptr);
}

__attribute__((__weak__, __visibility__("default")))
void
operator delete(void* ptr, const std::nothrow_t&) _NOEXCEPT
{
    ::operator delete(ptr);
}

__attribute__((__weak__, __visibility__("default")))
void
operator delete[] (void* ptr) _NOEXCEPT
{
    ::operator delete (ptr);
}

__attribute__((__weak__, __visibility__("default")))
void
operator delete[] (void* ptr, const std::nothrow_t&) _NOEXCEPT
{
    ::operator delete[](ptr);
}
{% endhighlight %}

从上面可以看出，传给`operator delete(void*)`和`operator delete [](void*)`函数的参数都是相同的。也就是说，那20个A对象空间其实都会被free掉，不管是调用`delete`还是`delete []`。所以大部分C++书中讲到的delete pA;仅仅只是释放pA\[0\]的内存空间，并没有释放pA\[1\]...pA\[19\]的内存空间，是不太准确的。但是pA\[1\]...pA\[19\]的对象并没有调用它们的析构函数，因而可能导致内存泄漏，这倒是存在的事实，特别是在A的析构函数需要额外释放资源时。
