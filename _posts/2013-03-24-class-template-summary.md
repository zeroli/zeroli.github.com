---
layout: post
title: "类模板总结"
description: ""
category: cpp
tags: [C++ templates]
---
{% include JB/setup %}

##类模板知识点梳理
 
###成员函数的实现
为了定义类模板的成员函数，必须制定该成员函数是一个函数模板，而且还需要使用这个类模板的完整类型限定符，譬如  
{% highlight cpp linenos %}
template <typename T>
void Stack::Push(T const& elem)
{
    elems.push_back(elem);
}
{% endhighlight %}


使用类模板时，必须显式的指定模板实参。只有那些被调用的成员函数，才会产生这些函数的实例化代码。  
对于类模板，成员函数只有在被使用的时候才会被实例化。另一方面，如果类模板中含有静态成员，那么用来实例化的每种类型，都会实例化这些静态成员。

### 类模板的特化
为了特化一个类模板，必须在起始处声明一个`template <>`，接下来声明用来特化类模板的类型。  
这个类型被用作模板实参，且必须在类名的后面直接指定：  
{% highlight cpp linenos %}
template <>
class Stack {
...
}
{% endhighlight %}

###局部特化
类模板可以被局部特化。可以在特定的环境下指定类模板的特定实现，并且要求某些模板参数仍然必须由用户来定义。例如：  
{% highlight cpp linenos %}
template <typename T1, typename T2>
class MyClass {
...
};
{% endhighlight %}
就可以有下面几种局部特化：  
{% highlight cpp linenos %}
template <typename T>
class MyClass<T, T> {
...
};


template <typename T>
class MyClass<T, int> {
...
};


template <typename T1, typename T2>
class MyClass<T1*, T2*> {
...
};
{% endhighlight %}
