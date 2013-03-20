---
layout: post
title: "Pimpl Idiom in C++"
description: 在C++里面, 经常出现的情况就是头文件里面的类定义太庞大了，而这个类的成员变量涉及了很多其他文件里面的类，从而导致了其他引用这个类的文件也依赖于这些成员变量的定义。在这种情况下，就出现了在C++里面特有的一个idiom，叫做Pimpl idiom。
category: cpp
tags: [cpp, design pattern]
---
{% include JB/setup %}

# Introduction 

在C++里面, 经常出现的情况就是头文件里面的类定义太庞大了，而这个类的成员变量涉及了很多
其他文件里面的类，从而导致了其他引用这个类的文件也依赖于这些成员变量的定义。
在这种情况下，就出现了在C++里面特有的一个idiom，叫做Pimpl idiom。

考虑一下下面的情况，假设有一个类A，它包含了成员变量b和c，类型分别为B和C，而如果D类
要使用A类的话，那也变相依赖了B和C。如下：

{% highlight cpp linenos %}
#include "B.h"
#include "C.h"

class A
{
private:
    B b;
    C c;
};{% endhighlight %}

这个时候如果D要使用A类的话，那么D就要像下面那样去写：

{% highlight cpp linenos %}
#include "A.h"

class D
{
private:
    A a;
};{% endhighlight %}

虽然形式上是只需要include A.h，但是在链接程序的时候，却需要把B和C的模块也一并链接进去。

初步的解决方案可以是把A里面的b和c变成指针类型，然后利用指针声明的时候类型可以是不完全类型，
从而在A.h里面不用include B.h和C.h。当然，这也只是解决的部分的问题。
如果A里面需要用到十几个成员变量的话，这个时候头文件的size就会变得很大，这也是一个问题。
而且有些时候，变成指针类型也不一定是可行的。这个时候，一个简单的想法就是把所有私有的
成员变量的声明都放到cpp文件里面去，这样使用A的类就可以完全不用知道A类的成员变量了。

# Pimpl Idiom

而Pimpl idiom就是这样的解决方案。所谓的Pimpl idiom，就是声明一个类中类，
然后再声明一个成员变量，类型是这个类中类的指针。用上面的例子来说明一下会清楚一下，
代码如下：

{% highlight cpp linenos %}
class A
{
private:
    struct Pimpl;
    Pimpl* m_pimpl;
};{% endhighlight %}

有了上面的定义，那么D类就可以完全不用知道A类的细节，而且链接的时候也可以完全不用管B和C了。
然后在A.cpp里面，我们就像下面这样去定义就好了：

{% highlight cpp linenos %}
struct A::Pimpl
{
    B b;
	C c;
};

A::A()
: m_pimpl(new Pimpl)
{
    m_impl->b; // 使用b
}{% endhighlight %}

而现在我们STL有auto\_ptr，boost有shared\_ptr，再要自己来管理内存好像
就有写多次一举了。所以在Herb Sutter的[Using auto_ptr Effectively](http://www.gotw.ca/publications/using_auto_ptr_effectively.htm)里面，
也提到了用auto\_ptr来进行“经典”的Pimpl的编写。

也就是如下面这样：

{% highlight cpp linenos %}
#include <memory>

class A
{
public:
    A();

private:
    struct Pimpl;
    std::auto_ptr<Pimpl> m_pimpl;
};{% endhighlight %}

可以当你写了上面的代码之后，编译，Bang! 编译器给你报一个错，说是Pimpl是incomplete
type。这下你就蒙了吧？！

其实要fix上面的编译错误，你只需要加上A的destructor的声明，然后在cpp文件里面实现一个
空的destructor就可以了。

但是这个是为什么呢？

## auto_ptr的模板特化

其实上面问题的原因，是跟模板特化的这个C++变态特性有关的。

我们先来看一下auto_ptr的简化定义：

{% highlight cpp linenos %}
template <typename T>
class auto_ptr
{
public:
    auto_ptr()
	: m_ptr(NULL)
	{}
    
    auto_ptr(T* p)
	: m_ptr(p)
	{}
    
    ~auto_ptr()
	{
	    if (m_ptr)
	    {
		delete m_ptr;
	    }
	}

private:
    T* m_ptr;
};{% endhighlight %}

我们看到auto\_ptr在他的构造函数里面自动的delete了他的m\_ptr，这个就是比较经典的
利用RAII实现的智能指针了。

然后还要知道，auto_ptr是一个模板类，而模板类的一个特点是，
**当他的成员函数只有在被调用的时候才会真正的做函数特化**。

也就是说，如果有下面的这样一个模板类：

{% highlight cpp linenos %}
template <typename T>
class TemplateClass
{
public:
    void Foo()
	{
	    int a = 1;
	    return;
	}

    void Bar()
	{
	    this->m_ptr = "syntax correct, but semantic incorrect.";
	}
};

int main(int argc, char *argv[])
{
    TemplateClass<int> a;
    a.Foo();
    return 0;
}{% endhighlight %}

上面的代码，是可以通过编译并且正确运行的。可以看到Foo这个函数是正确的，而Bar函数虽然
语法上是正确的，但是他的语义是错的。但是由于我们只调用了Foo，没有调用Bar，
所以只有Foo被真正的特化并且做了完全的编译，而Bar只是做了语法上的检查，
并没有做语义的检查。所以上面的代码在C++里面是100%的正确的。

所以auto\_ptr里面的成员函数，包括构造和析构函数，都是在被调用的时候才进行真正的特化。

## Default Destructor

还记得在学C++的刚开始的时候书上这么说过，不定义构造函数或者析构函数，
那么编译器会帮我们造一个默认的。而这个默认的构造或者析构函数只会做成员变量还有父类的
默认初始化或者析构，其他什么都不会做。

那么我们看回利用了Pimpl的A的定义。在这个定义里面，由于我没有写析构函数的声明，
所以编译器自动帮我定义了一个。而A里面有一个auto\_ptr成员变量，所以在这个默认的
析构函数里面会析构这个成员变量。所谓的析构，其实就是调用析构函数而已。
所以，在这个默认的析构函数里面，调用了auto\_ptr的析构函数，这个时候，
auto\_ptr的析构函数就被编译器特化了。

而在auto\_ptr的析构函数里面，delete了模板参数的指针类型的成员变量。
而在A这个例子里面，模板参数就是Pimpl。而在特化的这一瞬间，Pimpl是被声明了，
但是还没有被定义。

所以例子里面的A在经过编译后是和下面的代码等价的：

{% highlight cpp linenos %}
class A
{
public:
    A();
	~A()
	{
		~auto_ptr<Pimpl>(m_pimpl);
	}

private:
    struct Pimpl;
    std::auto_ptr<Pimpl> m_pimpl;
};

auto_ptr<Pimpl>::~auto_ptr()
{
	delete m_ptr; // m_ptr的类型是Pimpl*
}{% endhighlight %}

那为什么当我加上A的析构函数的声明之后，编译就可以通过呢？因为当我们声明了A的析构函数之后，
编译器就不会自动生成析构函数的实现了，而由于我们会在cpp文件里面去写析构函数的实现，
而在此之前，我们就会在cpp文件的开头定义好Pimpl的实现。所以当我们自己写的A的析构函数
被编译器看见的时候，Pimpl就是一个已经定义好的类型，所以就没有问题了。

# Pimpl by boost::shared\_ptr

其实使用auto\_ptr来实现Pimpl Idiom并不是唯一的方法，Pimpl还可以用
boost::scoped\_ptr和boost::shared\_ptr来实现。而scoped\_ptr和auto\_ptr
其实是一样的，也是需要用户手工的声明一个析构函数来实现Pimpl Idiom，这里就不说了。

但是通过shared\_ptr来实现的话，我们就连析构函数都可以省略！也就是说，
如果我写下面的代码，是完全正确的：

{% highlight cpp linenos %}
class A
{
public:
    A();

private:
    struct Pimpl;
    boost::shared_ptr<Pimpl> m_pimpl;
};{% endhighlight %}

需要注意的是，虽然析构函数可以省略，但是构造函数还是必须明确声明的。
这又是为什么呢？为什么auto\_ptr不行，但是shared\_ptr就可以呢？

答案就在shared\_ptr的实现里面。

相信shared\_ptr应该是每个较为深入学过C++的人都会理解原理的一个类了，其中shared\_ptr
的实现又可以分为侵入式和非侵入式的，而boost::shared\_ptr的实现是非侵入式的。
也就是说要用shared\_ptr的类不需要任何改动就可以使用了。

来看看简化之后的shared\_ptr的实现吧：

{% highlight cpp linenos %}

class sp_counted_base
{
public:
    virtual ~sp_counted_base(){}
};

template<typename T>
class sp_counted_base_impl : public sp_counted_base
{
public:
    sp_counted_base_impl(T *t):t_(t){}
    ~sp_counted_base_impl(){delete t_;}
private:
    T *t_;
};


class shared_count
{
public:
    static int count_;
    template<typename T>
    shared_count(T *t):
        t_(new sp_counted_base_impl<T>(t))
    {
        count_ ++;
    }
    void release()
    {
        --count_;
        if(0 == count_) delete t_;
    }
    ~shared_count()
    {
        release();
    }
private:
    sp_counted_base *t_;
};
int shared_count::count_(0);

template<typename T>
class myautoptr
{
public:
    template<typename Y>
    myautoptr(Y* y):sc_(y),t_(y){}
    ~myautoptr(){ sc_.release();}
private:
    shared_count sc_;
    T *t_;
};

int main()
{
    myautoptr<A> a(new B);
}{% endhighlight %}

从上面的代码可以看到，shared\_ptr里面不单存了一个模板类型的指针，
还存了一个shared\_count。
这个shared\_count的作用就是用来作为引用计数还有自动管理指针用的。
而shared\_count里面又存了一个sp\_counted\_base，而sp\_counted\_base\_impl
是一个模板类，其继承于sp\_counted\_base。这其实是一个模板技巧，也就是声明一个
通用的基类，然后定义一个模板类来继承于这个基类，而其他类通过基类的指针来使用这个模板类，
这样就可以在编译时确定一些类型信息，而同时把一些通用的实现细节推迟到运行时。这句话什么意思呢？
看完接下来的解释你就明白了。

接下来我们又要注意到，shared\_ptr和shared\_count的构造函数都是模板成员函数，
模板类型由参数决定，而这个技巧和上面的模板继承技巧组合在一起，就是这节开始的时候，
例子中不用写析构函数的理由。

首先，当我们声明一个`shared_ptr<int>`的时候，它只是把里面的t\_成员给特化了，
而shared\_count里面存的是什么类型的指针仍然没有确定。

而当我们调用`shared_ptr<int>(new int(3))`的时候，他就调用了shared\_ptr的构造函数。
这个时候就特化了模板构造函数，然后这个构造函数里面又调用了shared\_count的构造函数，
所以shared\_count的构造函数也被特化，而又同时特化了sp\_counted\_base\_impl，
这个时候里面的指针就完全被特化了。

而我们看到，在shared\_ptr被析构的时候，它调用的是shared\_count的release函数，
release函数里面又delete了它的类型为sp\_counted\_base的指针，
所以调用的是sp\_counted\_base的析构函数(虚函数)。因为是虚函数，当具体类型确定之后，
是会具体调用到具体的析构函数的。但是在编译的时候，不需要知道具体的类型。

说了那么多，其实就是一句话，调用shared\_ptr的析构函数的时候，它不需要知道具体的指针类型。
也就是说这个类型即使incomplete也没有关系。而在调用shared\_ptr的构造函数的时候，
shared\_ptr就是会知道这个类型的所有信息，从而使得delete的时候调用到具体的析构函数。

所以对于shared\_ptr来说，构造函数需要知道所有的类型信息，而析构函数是不要知道类型信息的。
回到例子里面，当我们不声明析构函数的时候，编译器为我们定义了一个默认的析构函数，
这个时候shared\_ptr的析构函数就会被特化并定义，同时也调用sp\_counted\_base
的析构函数也就被编译了。但是这个时候并不许要具体的类型信息，
所以类型是incomplete也是可以的。当我们定义A的构造函数的时候，这个时候shared\_ptr
的构造函数就被特化，从而shared\_count的构造函数被特化，而sp\_counted\_base\_impl
也就是被特化了。这个时候shared\_ptr也就有了所有必要的类型信息，
他的析构函数就可以正常的工作了。

这就是为什么用shared\_ptr来实现Pimpl可以不用写析构函数的原因了，
为了实现这个功能，shared\_ptr牺牲了一点点的空间来完成上面的概念，比普通的shared\_ptr
多了一个`sizeof(sp_counted_base*)`的大小。
