---
layout: post
title: "只能new不能delete的基类实现方法"
description: ""
category: "cpp"
tags: [程序开发, C++]
---
{% include JB/setup %}

##问题提出
定义一个类，客户端只能new不能delete，但是要求这个类能够被继承。


##问题分析
客户端代码能够new一个类实例，说明这个类提供了对应的public访问域的构造函数，不能delete它，说明这个类没有提供public访问域的析构函数。也就是说，析构函数要么是在protected访问域中，要么是在private访问域中。但是要求这个类能够被继承，至少说明析构函数得是virtual的。如果析构函数是在private访问域中，那么被继承的子类肯定无法析构成功（无法编译），因为private成员无法在子类中可见。所以这个类的析构函数只能定义在protected访问域中。  
但是定义在protected访问域中，客户端无法delete，那就会造成内存泄漏，因为它只能new在堆上，而且也无法构造在栈上。

##问题解决
那么我们只能让这个类提供一个interface，叫做release，客户端代码手动调用。其实这个情况，在wxWidget代码很常见，就是所有的窗口子类实例化时都是new在堆上，窗口退出时，并不调用delete，而是窗口遍历调用各个子窗口的Destroy函数，来清除所有的资源。这个Destory函数干的事情也很简单，就是delete this。  
下面就是一个简单的实现案例，这个类是一个引用计数类：  
{% highlight cpp %}
class RefCountedObject {
public:
    RefCountedObject()
    : count(1)
    { std::cout << "RefCountedObject::RefCountedObject:()\n"; }


    void duplicate() {
        ++count;
    }
    void release() {
        if (--count == 0)
        {
            delete this;
        }
    }
protected:
    virtual ~RefCountedObject() {
        std::cout << "delete RefCountedObject\n";
    }


private:
    RefCountedObject(const RefCountedObject&);
    RefCountedObject& operator=(const RefCountedObject&);
    int count;
};

class A : public RefCountedObject
{
protected:
    ~A() {
        std::cout << "delete A\n";
    }
};


int main()
{
    A* pa = new A();
    //A aa;  // cannot compile, since its dtor is protected
    //delete pa; // can not compile, too 
    pa->release();
    return 0;
}
{% endhighlight %}
