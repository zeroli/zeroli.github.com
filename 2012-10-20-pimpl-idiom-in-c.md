﻿---
layout: post
title: "Pimpl Idiom in C++"
description: ��C++����, �������ֵ��������ͷ�ļ�������ඨ��̫�Ӵ��ˣ��������ĳ�Ա�����漰�˺ܶ������ļ�������࣬�Ӷ����������������������ļ�Ҳ��������Щ��Ա�����Ķ��塣����������£��ͳ�������C++�������е�һ��idiom������Pimpl idiom��
category: cpp
tags: [cpp, design pattern]
---
{% include JB/setup %}

# Introduction 

��C++����, �������ֵ��������ͷ�ļ�������ඨ��̫�Ӵ��ˣ��������ĳ�Ա�����漰�˺ܶ�
�����ļ�������࣬�Ӷ����������������������ļ�Ҳ��������Щ��Ա�����Ķ��塣
����������£��ͳ�������C++�������е�һ��idiom������Pimpl idiom��

����һ������������������һ����A���������˳�Ա����b��c�����ͷֱ�ΪB��C�������D��
Ҫʹ��A��Ļ�����Ҳ����������B��C�����£�

{% highlight cpp linenos %}
#include "B.h"
#include "C.h"

class A
{
private:
    B b;
    C c;
};{% endhighlight %}

���ʱ�����DҪʹ��A��Ļ�����ôD��Ҫ����������ȥд��

{% highlight cpp linenos %}
#include "A.h"

class D
{
private:
    A a;
};{% endhighlight %}

��Ȼ��ʽ����ֻ��Ҫinclude A.h�����������ӳ����ʱ��ȴ��Ҫ��B��C��ģ��Ҳһ�����ӽ�ȥ��

�����Ľ�����������ǰ�A�����b��c���ָ�����ͣ�Ȼ������ָ��������ʱ�����Ϳ����ǲ���ȫ���ͣ�
�Ӷ���A.h���治��include B.h��C.h����Ȼ����Ҳֻ�ǽ���Ĳ��ֵ����⡣
���A������Ҫ�õ�ʮ������Ա�����Ļ������ʱ��ͷ�ļ���size�ͻ��úܴ���Ҳ��һ�����⡣
������Щʱ�򣬱��ָ������Ҳ��һ���ǿ��еġ����ʱ��һ���򵥵��뷨���ǰ�����˽�е�
��Ա�������������ŵ�cpp�ļ�����ȥ������ʹ��A����Ϳ�����ȫ����֪��A��ĳ�Ա�����ˡ�

# Pimpl Idiom

��Pimpl idiom���������Ľ����������ν��Pimpl idiom����������һ�������࣬
Ȼ��������һ����Ա����������������������ָ�롣�������������˵��һ�»����һ�£�
�������£�

{% highlight cpp linenos %}
class A
{
private:
    struct Pimpl;
    Pimpl* m_pimpl;
};{% endhighlight %}

��������Ķ��壬��ôD��Ϳ�����ȫ����֪��A���ϸ�ڣ��������ӵ�ʱ��Ҳ������ȫ���ù�B��C�ˡ�
Ȼ����A.cpp���棬���Ǿ�����������ȥ����ͺ��ˣ�

{% highlight cpp linenos %}
struct A::Pimpl
{
    B b;
	C c;
};

A::A()
: m_pimpl(new Pimpl)
{
    m_impl->b; // ʹ��b
}{% endhighlight %}

����������STL��auto\_ptr��boost��shared\_ptr����Ҫ�Լ��������ڴ����
����д���һ���ˡ�������Herb Sutter��[Using auto_ptr Effectively](http://www.gotw.ca/publications/using_auto_ptr_effectively.htm)���棬
Ҳ�ᵽ����auto\_ptr�����С����䡱��Pimpl�ı�д��

Ҳ����������������

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

���Ե���д������Ĵ���֮�󣬱��룬Bang! ���������㱨һ����˵��Pimpl��incomplete
type������������˰ɣ���

��ʵҪfix����ı��������ֻ��Ҫ����A��destructor��������Ȼ����cpp�ļ�����ʵ��һ��
�յ�destructor�Ϳ����ˡ�

���������Ϊʲô�أ�

## auto_ptr��ģ���ػ�

��ʵ���������ԭ���Ǹ�ģ���ػ������C++��̬�����йصġ�

����������һ��auto_ptr�ļ򻯶��壺

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

���ǿ���auto\_ptr�����Ĺ��캯�������Զ���delete������m\_ptr��������ǱȽϾ����
����RAIIʵ�ֵ�����ָ���ˡ�

Ȼ��Ҫ֪����auto_ptr��һ��ģ���࣬��ģ�����һ���ص��ǣ�
**�����ĳ�Ա����ֻ���ڱ����õ�ʱ��Ż��������������ػ�**��

Ҳ����˵����������������һ��ģ���ࣺ

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

����Ĵ��룬�ǿ���ͨ�����벢����ȷ���еġ����Կ���Foo�����������ȷ�ģ���Bar������Ȼ
�﷨������ȷ�ģ��������������Ǵ�ġ�������������ֻ������Foo��û�е���Bar��
����ֻ��Foo���������ػ�����������ȫ�ı��룬��Barֻ�������﷨�ϵļ�飬
��û��������ļ�顣��������Ĵ�����C++������100%����ȷ�ġ�

����auto\_ptr����ĳ�Ա������������������������������ڱ����õ�ʱ��Ž����������ػ���

## Default Destructor

���ǵ���ѧC++�ĸտ�ʼ��ʱ��������ô˵���������幹�캯����������������
��ô���������������һ��Ĭ�ϵġ������Ĭ�ϵĹ��������������ֻ������Ա�������и����
Ĭ�ϳ�ʼ����������������ʲô����������

��ô���ǿ���������Pimpl��A�Ķ��塣������������棬������û��д����������������
���Ա������Զ����Ҷ�����һ������A������һ��auto\_ptr��Ա���������������Ĭ�ϵ�
����������������������Ա��������ν����������ʵ���ǵ��������������ѡ�
���ԣ������Ĭ�ϵ������������棬������auto\_ptr���������������ʱ��
auto\_ptr�����������ͱ��������ػ��ˡ�

����auto\_ptr�������������棬delete��ģ�������ָ�����͵ĳ�Ա������
����A����������棬ģ���������Pimpl�������ػ�����һ˲�䣬Pimpl�Ǳ������ˣ�
���ǻ�û�б����塣

�������������A�ھ���������Ǻ�����Ĵ���ȼ۵ģ�

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
	delete m_ptr; // m_ptr��������Pimpl*
}{% endhighlight %}

��Ϊʲô���Ҽ���A����������������֮�󣬱���Ϳ���ͨ���أ���Ϊ������������A����������֮��
�������Ͳ����Զ���������������ʵ���ˣ����������ǻ���cpp�ļ�����ȥд����������ʵ�֣�
���ڴ�֮ǰ�����Ǿͻ���cpp�ļ��Ŀ�ͷ�����Pimpl��ʵ�֡����Ե������Լ�д��A����������
��������������ʱ��Pimpl����һ���Ѿ�����õ����ͣ����Ծ�û�������ˡ�

# Pimpl by boost::shared\_ptr

��ʵʹ��auto\_ptr��ʵ��Pimpl Idiom������Ψһ�ķ�����Pimpl��������
boost::scoped\_ptr��boost::shared\_ptr��ʵ�֡���scoped\_ptr��auto\_ptr
��ʵ��һ���ģ�Ҳ����Ҫ�û��ֹ�������һ������������ʵ��Pimpl Idiom������Ͳ�˵�ˡ�

����ͨ��shared\_ptr��ʵ�ֵĻ������Ǿ�����������������ʡ�ԣ�Ҳ����˵��
�����д����Ĵ��룬����ȫ��ȷ�ģ�

{% highlight cpp linenos %}
class A
{
public:
    A();

private:
    struct Pimpl;
    boost::shared_ptr<Pimpl> m_pimpl;
};{% endhighlight %}

��Ҫע����ǣ���Ȼ������������ʡ�ԣ����ǹ��캯�����Ǳ�����ȷ�����ġ�
������Ϊʲô�أ�Ϊʲôauto\_ptr���У�����shared\_ptr�Ϳ����أ�

�𰸾���shared\_ptr��ʵ�����档

����shared\_ptrӦ����ÿ����Ϊ����ѧ��C++���˶������ԭ���һ�����ˣ�����shared\_ptr
��ʵ���ֿ��Է�Ϊ����ʽ�ͷ�����ʽ�ģ���boost::shared\_ptr��ʵ���Ƿ�����ʽ�ġ�
Ҳ����˵Ҫ��shared\_ptr���಻��Ҫ�κθĶ��Ϳ���ʹ���ˡ�

��������֮���shared\_ptr��ʵ�ְɣ�

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

������Ĵ�����Կ�����shared\_ptr���治������һ��ģ�����͵�ָ�룬
������һ��shared\_count��
���shared\_count�����þ���������Ϊ���ü��������Զ�����ָ���õġ�
��shared\_count�����ִ���һ��sp\_counted\_base����sp\_counted\_base\_impl
��һ��ģ���࣬��̳���sp\_counted\_base������ʵ��һ��ģ�弼�ɣ�Ҳ��������һ��
ͨ�õĻ��࣬Ȼ����һ��ģ�������̳���������࣬��������ͨ�������ָ����ʹ�����ģ���࣬
�����Ϳ����ڱ���ʱȷ��һЩ������Ϣ����ͬʱ��һЩͨ�õ�ʵ��ϸ���Ƴٵ�����ʱ����仰ʲô��˼�أ�
����������Ľ�����������ˡ�

������������Ҫע�⵽��shared\_ptr��shared\_count�Ĺ��캯������ģ���Ա������
ģ�������ɲ�����������������ɺ������ģ��̳м��������һ�𣬾�����ڿ�ʼ��ʱ��
�����в���д�������������ɡ�

���ȣ�����������һ��`shared_ptr<int>`��ʱ����ֻ�ǰ������t\_��Ա���ػ��ˣ�
��shared\_count��������ʲô���͵�ָ����Ȼû��ȷ����

�������ǵ���`shared_ptr<int>(new int(3))`��ʱ�����͵�����shared\_ptr�Ĺ��캯����
���ʱ����ػ���ģ�幹�캯����Ȼ��������캯�������ֵ�����shared\_count�Ĺ��캯����
����shared\_count�Ĺ��캯��Ҳ���ػ�������ͬʱ�ػ���sp\_counted\_base\_impl��
���ʱ�������ָ�����ȫ���ػ��ˡ�

�����ǿ�������shared\_ptr��������ʱ�������õ���shared\_count��release������
release����������delete����������Ϊsp\_counted\_base��ָ�룬
���Ե��õ���sp\_counted\_base����������(�麯��)����Ϊ���麯��������������ȷ��֮��
�ǻ������õ���������������ġ������ڱ����ʱ�򣬲���Ҫ֪����������͡�

˵����ô�࣬��ʵ����һ�仰������shared\_ptr������������ʱ��������Ҫ֪�������ָ�����͡�
Ҳ����˵������ͼ�ʹincompleteҲû�й�ϵ�����ڵ���shared\_ptr�Ĺ��캯����ʱ��
shared\_ptr���ǻ�֪��������͵�������Ϣ���Ӷ�ʹ��delete��ʱ����õ����������������

���Զ���shared\_ptr��˵�����캯����Ҫ֪�����е�������Ϣ�������������ǲ�Ҫ֪��������Ϣ�ġ�
�ص��������棬�����ǲ���������������ʱ�򣬱�����Ϊ���Ƕ�����һ��Ĭ�ϵ�����������
���ʱ��shared\_ptr�����������ͻᱻ�ػ������壬ͬʱҲ����sp\_counted\_base
����������Ҳ�ͱ������ˡ��������ʱ�򲢲���Ҫ�����������Ϣ��
����������incompleteҲ�ǿ��Եġ������Ƕ���A�Ĺ��캯����ʱ�����ʱ��shared\_ptr
�Ĺ��캯���ͱ��ػ����Ӷ�shared\_count�Ĺ��캯�����ػ�����sp\_counted\_base\_impl
Ҳ���Ǳ��ػ��ˡ����ʱ��shared\_ptrҲ���������б�Ҫ��������Ϣ��
�������������Ϳ��������Ĺ����ˡ�

�����Ϊʲô��shared\_ptr��ʵ��Pimpl���Բ���д����������ԭ���ˣ�
Ϊ��ʵ��������ܣ�shared\_ptr������һ���Ŀռ����������ĸ������ͨ��shared\_ptr
����һ��`sizeof(sp_counted_base*)`�Ĵ�С��
