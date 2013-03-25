---
layout: post
title: "模板技巧性基础知识"
description: ""
category: cpp
tags: [C++ templates]
---
{% include JB/setup %}

##关键字typename
{% highlight cpp %}
template <typename T>
class MyClass {
    typename T::SubType* ptr;
...
};
{% endhighlight %}

SubType是定义于类T内部的一种类型。因此ptr是一个指向T:SubType类型的指针。但是如果缺少typename，SubType就会被认为是一个静态成员，那么它应该是一个具体的变量或对象，于是，下面表达式：  
    T::SubType* ptr;
就会被看作是类T的静态成员SubType和ptr的乘积。  

##使用this->
对于具有基类的类模板，自身使用名称x并不一定等同于this->x。即使该x是从基类继承获得的，也是如此。  
{% highlight cpp %}
template <typename T>
class Base {
public:
    int a;
};


template <typename T>
class Derived : public Base<T>
{
public:
    void Print()
    {
       std::cout << a << "\n";
    }
}; {% endhighlight %}
GCC将无法查找到a的定义（请注意VC可能编译通过，在于VC编译器只扫描template一次）。对于那些在基类中声明，**并且依赖于模板参数的符号（函数或变量等），你应该在它们前面使用`this->`或者`Base<T>::`**。

##使用字符串作为函数模板的实参
有时，将字符串传递给函数模板的引用参数会导致出人意料的运行结果.  
{% highlight cpp %}
template <typename T>
inline T const& max(T const& a, T const& b)
{
   return a < b ? b : a;
}
int main()
{
    std::string s;
    ::max("apple", "peach");    // OK
    ::max("apple", "tomato");    // ERROR
    ::max("apple", s);    // ERROR
} {% endhighlight %}

问题在于：由于长度的不同，这些字符串属于不同的数组类型。也就是说，"apple"和"peach"属于相同的类型const char[6]; 然而"tomato"的类型是const char[7]。因此只有第一个调用是合法的，因此该max()模板期望的是类型完全相同的参数。然而，如果声明的是非引用参数，就可以使用长度不同的字符串来作为max的参数。  
{% highlight cpp %}
template <typename T>
inline T max(T a, T b)
{
    return a < b ? b : a;
}
int main()
{
    std::string s;
    ::max("apple", "peach");    // OK
    ::max("apple", "tomato");   // OK
    ::max("apple", s);    // ERROR
}{% endhighlight %}
产生这种调用结果的原因在于：**对于非引用类型的参数，在实参演绎的过程中，会出现数组到指针的类型转换**.  
{% highlight cpp %}
template <typename T>
void ref(const T& x)
{
    std::cout << "x in ref(T const&): " << typeid(x).name() << "\n";
}
template <typename T>
void nonref(T x)
{
    std::cout << "x in nonref(x): " << typeid(x).name() << "\n";
}{% endhighlight %}
输出如下：  
> x in ref(T const&) : char\[6\]  
> x in nonref(T): const char\*

*你可以选择以下几种方案：*  
1) 使用非引用参数，取代引用参数（然而，这会导致无用的拷贝）。  
2) 进行重载，编写接受引用参数和非引用参数的两个重载参数（然而，这会导致二义性）。  
3) 对具体类型进行重载（譬如对std::string进行重载）。  
4) 重载数组类型，譬如：  
{% highlight cpp %}
template <typename T, int N, int M>
T const* max(T const (&a)[N], T const (&b)[M])
{
    return a < b ? b : a;
}{% endhighlight %}
5) 强制要求应用程序使用显式类型转换。  

另外一点要注意的是，无论如何为字符串提供重载是有必要的：因为如果不提供重载，当我们调用max()来比较两个字符串时，操作`a<b`执行的是指针比较，就是`a<b`比较的是两个字符串的地址，而不是它们的字典顺序。


