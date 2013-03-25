---
layout: post
title: "Poco中的RunnableAdaptor实现浅析"
description: ""
category: "poco"
tags: [C++, poco源码阅读]
---
{% include JB/setup %}

Poco是一个设计相当好的C++框架库。  
RunnableAdaptor，继承自Runnable，提供了注册和执行类成员函数的接口。

Runnable的类定义如下：
{% highlight cpp %}
class Foundation_API Runnable
	/// The Runnable interface with the run() method
	/// must be implemented by classes that provide
	/// an entry point for a thread.
{
public:	
	Runnable();
	virtual ~Runnable();
	
	virtual void run() = 0;
		/// Do whatever the thread needs to do. Must
		/// be overridden by subclasses.
};[/c]

	/// The Runnable interface with the run() method
	/// must be implemented by classes that provide
	/// an entry point for a thread.
{
public:	
	Runnable();
	virtual ~Runnable();
	
	virtual void run() = 0;
		/// Do whatever the thread needs to do. Must
		/// be overridden by subclasses.
};
{% endhighlight %}
这个类设计思想来源JAVA，任何可运行的东西都可以继承自Runnable，只要重写它的run方法。

RunnableAdaptor就是这样一个Runnable，但是它包装了某个类成员函数的注册和执行。也就是说，借用这个RunnableAdaptor就可以执行某个类的成员函数的代码。很有意思。
{% highlight cpp %}
template <class C>
class RunnableAdapter: public Runnable
	/// This adapter simplifies using ordinary methods as
	/// targets for threads.
	/// Usage:
	///    RunnableAdapter<MyClass> ra(myObject, &MyObject::doSomething));
	///    Thread thr;
	///    thr.Start(ra);
	///
	/// For using a freestanding or static member function as a thread
	/// target, please see the ThreadTarget class.
{
public:
	typedef void (C::*Callback)();
	
	RunnableAdapter(C& object, Callback method): _pObject(&object), _method(method)
	{
	}
	
	RunnableAdapter(const RunnableAdapter& ra): _pObject(ra._pObject), _method(ra._method)
	{
	}


	~RunnableAdapter()
	{
	}


	RunnableAdapter& operator = (const RunnableAdapter& ra)
	{
		_pObject = ra._pObject;
		_method  = ra._method;
		return *this;
	}


	void run()
	{
		(_pObject->*_method)();
	}
	
private:
	RunnableAdapter();


	C*       _pObject;
	Callback _method;
};
{% endhighlight %}
方法就是提供callback，类似于boost::function。

