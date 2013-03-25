---
layout: post
title: "一个正则表达式，支持逻辑和关系运算符组成的表达式计算"
description: ""
category: "boost_usage"
tags: [程序开发, boost, regex]
---
{% include JB/setup %}

I. 写一个正则表达式，要求判断一个数是否满足以下条件：  
`>= val1 && < val2 ...`  
1. val1和va2要求支持浮点数;  
2. 支持>, >=, <, <=, =, !=关系运算符;  
3. 支持&&, ||, and(不区分大小), or(不区分大小写)逻辑运算  

1) 有效的浮点数的正则表达式如下：  
`([+-]?\d+(\.\d+)?|0?\.\d+)`  
支持+/-m.n, m.n, .n, 不支持m.  
2) 关系运算符的正则表达式如下：  
`([><!\]=?|=)`  
3) 逻辑运算符的正则表达式如下：  
`(&&|\|\||[Aa][Nn][Dd]|[Oo][Rr])`  
所以综合起来的正则表达式就是：  
`^(\s*([><!]=?|=)\s*([+-]?\d+(\.\d+)?|0?\.\d+)\s*)((&&|\|\||[Aa][Nn][Dd]|[Oo][Rr])(\s*([><!]=?|=)\s*([+-]?\d+(\.\d+)?|0?\.\d+)\s*))*`  

II. 如何用boost::regex来处理这样的表达式，然后判断一个是否满足表达式的要求：  
假如判断数30是否满足"> 20 && < 40"，我们会做一个额外的处理，就是在前面加上"AND"，最后要处理的表达式会是"AND > 20 && < 40"。这样一来我们的正则表达式就可以简化为:  
`((&&|\|\||[Aa][Nn][Dd]|[Oo][Rr])(\s*([><!]=?|=)\s*([+-]?\d+(\.\d+)?|0?\.\d+)\s*))+`  
而且也方便我们从里面提取出运算子。  


以下CPP代码演示了从一个字符串中提取出运算子的过程：
{% highlight cpp %}
void ParseSpec(const std::string& spec)
{
	m_validSpec = false;
	m_Ops.clear();

	std::string specStr = boost::trim_copy(spec);
	if (specStr.empty())
	{
		m_validSpec = true;
		return;
	}
	// we added "AND" as prefix for consistent handling later
	specStr = "AND " + specStr;
	boost::regex reg(specExp);
	m_validSpec = boost::regex_match(specStr, reg);
	if (!m_validSpec) return;

	reg.set_expression(logicalExp+relationExp);
	boost::smatch what;
	std::string::const_iterator start = specStr.begin(), end = specStr.end();
	while (boost::regex_search(start, end, what, reg))
	{
		std::string opStr = std::string(what[1].first, what[1].second);
		std::transform(opStr.begin(), opStr.end(), opStr.begin(), boost::bind(&std::toupper, _1));

		m_Ops.push_back(std::make_pair(opStr,
			std::make_pair(std::string(what[2].first, what[2].second),
			atof(std::string(what[3].first, what[3].second).c_str()))));
		start = what[0].second;
	}
}
{% endhighlight %}
注意reg.set_expression(logicalExp+relationExp);这里调整的正则表达式应该是一个处理单元（没有外围的()组)，方便boost::regex_search一个一个的提取出逻辑运算子和关系运算子。

下面C++代码演示了如何运用提取出的运算子，判断一个数是否满足表达式
{% highlight cpp %}
bool IsValueOK(double valueToCheck) const
{
	if (!m_validSpec) return true;

	/*
	Below logic will follow logic in C language
	For example: A && B || C
	If A (true), run B, but skipping run C,
	otherwise, run C, but skipping run B
	*/
	bool ret = true;
	for (BOOST_AUTO(it, m_Ops.begin()); it != m_Ops.end(); ++it)
	{
		if (it->first == "AND" || it->first == "&&")
		{
			if (!ret) continue;	// skip this op, like C
			ret = ret && ::RelationOp(valueToCheck, it->second.second, it->second.first);
		}
		if (it->first == "OR" || it->first == "||")
		{
			if (ret) continue;	// skip this op, like C
			ret = ret || ::RelationOp(valueToCheck, it->second.second, it->second.first);
		}
	}
	return ret;
}
{% endhighlight %}
以上代码模仿逻辑运算AND/OR，满足短路原则：遇到AND，前面计算结果为FALSE，跳过此次运算；遇到OR，前面计算结果为TRUE，跳过此次运算。

