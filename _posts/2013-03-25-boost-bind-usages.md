---
layout: post
title: "boost::bind的几种使用"
description: ""
category: "boost_usage"
tags: [boost]
---
{% include JB/setup %}

###boost::bind与函数对象的结合使用：
有这样一个函数对象：

{% highlight cpp %}
template <typename _Ty>
struct  iStrLess : public std::binary_function<_Ty, _Ty, bool>
{
public:
	bool operator () (const _Ty& left, const _Ty& right) const
	{
		_Ty ileft = boost::to_upper_copy(left);
		_Ty iright = boost::to_upper_copy(right);
	       return (ileft < iright);
	}
};
{% endhighlight %}
当与boost::bind结合使用时的一个案例：
{% highlight cpp %}
typedef StringUtility::iStrEqual<std::string> iStrEqual_t;
iStrEqual_t strComp;
if (std::find_if(paramNames.begin(), paramNames.end(),
	boost::bind(&iStrEqual_t::operator(), &strComp, colName, _1))
		== paramNames.end())
{
	// to do something
}
{% endhighlight %}
###boost::bind与STL的functor的结合使用：
**a) 与std::logical_not的结合**
{% highlight cpp %}
BOOST_AUTO(itFound, std::find_if(cellmarkers.begin(), cellmarkers.end(),
			boost::bind<bool>(std::logical_not<bool>(), boost::bind(&SomeClass::containedIn, _1, boost::cref(cellbox)))));
if (itFound != cellmarkers.end()) return true;
{% endhighlight %}
等价于下面的代码段：
{% highlight cpp %}
for (BOOST_AUTO(it, cellmarkers.begin()); it != cellmarkers.end(); ++it)
{
    if (!it->containedIn(cellbox)) return true;
}
{% endhighlight %}
**b) 与std::equal_to的结合**
{% highlight cpp %}
std::vector<SomeClass>::iterator it = std::find_if(vObjs.begin(), vObjs.end(),
		boost::bind<bool>(std::equal_to<int>(), cellsetID,
		boost::bind(&SomeClass::GetClipSetId, _1)
		)
		);
{% endhighlight %}
等价于下面的代码段:
{% highlight cpp %}
for (BOOST_AUTO(it, vObjs.begin()); it != vObjs.end(); ++it)
{
    if (it->GetClipSetId() == cellsetID)
    {
        // do something
        break;
    }
}
{% endhighlight %}
注意：外部的boost::bind需要标明返回参数，内部的就不需要。



