---
layout: post
title: "一段调用STL算法的程序代码的效率分析"
description: ""
category: "cpp"
tags: [STL, 算法]
---
{% include JB/setup %}


已知一个STD::SET，想要根据一个predicate来从中去除所有的满足predicate（返回true）的元素。  

我们首先想到的是STL提供的remove\_if函数。  
下面我们来看看这个函数是如何实现的：  
{% highlight cpp %}
1133   template<typename _ForwardIterator, typename _Predicate>
1134     _ForwardIterator
1135     remove_if(_ForwardIterator __first, _ForwardIterator __last,
1136           _Predicate __pred)
1137     {
1144
1145       __first = std::find_if(__first, __last, __pred);
1146       _ForwardIterator __i = __first;
1147       return __first == __last ? __first
1148                    : std::remove_copy_if(++__i, __last,
1149                              __first, __pred);
1150     }
{% endhighlight %}
 这个函数首先调用std::find\_if来干活，下面来看看std::find\_if的实现过程：  
std::find\_if提供了很多重载版本，主要根据iterator的类型：  
由于我们的std::set不是randomIterator，所以就会调用到下面的函数：  
{% highlight cpp %}
182   template<typename _InputIterator, typename _Predicate>
183     inline _InputIterator
184     find_if(_InputIterator __first, _InputIterator __last,
185         _Predicate __pred, input_iterator_tag)
186     {
187       while (__first != __last && !__pred(*__first))
188     ++__first;
189       return __first;
190     }
{% endhighlight %}
可以看到，这个函数迭代的查看每一个元素，判断是否满足predicate的要求。  

我们来看看std::remove_copy_if函数。  
当找到第一个满足条件的元素之后，便从这个元素开始来调用remove\_copy\_if函数。  
{% highlight cpp %}
1059   template<typename _InputIterator, typename _OutputIterator,
1060        typename _Predicate>
1061     _OutputIterator
1062     remove_copy_if(_InputIterator __first, _InputIterator __last,
1063            _OutputIterator __result, _Predicate __pred)
1064     {
1072
1073       for ( ; __first != __last; ++__first)
1074     if (!__pred(*__first))
1075       {
1076         *__result = *__first;
1077         ++__result;
1078       }
1079       return __result;
1080     }
{% endhighlight %}
这个函数也是迭代的查看每一个元素，判断是否满足要求，不满足条件的(!\_\_pred(\*\_\_first))，将它COPY到输出容器中。  
请注意这个操作，按照 remove\_if传递进来的输入输出迭代器，来自于同一个容器，  
a) 如果后面没有任何元素满足predicate的条件，那么这个函数将仅仅只是LOOP那些元素而已。  
b) 如果后面的元素有一些满足predicate的条件，那么这个赋值操作是否做什么事情，将取决于容器内元素提供的赋值操作符实现。如果元素的赋值操作符实现进行了自赋值检查（什么都不做），那么这个赋值将不会花销任何时间，否则对于自定义的类型可能就会有较大的开销。  

所以用std::remove\_if来针对std::set进行操作，效率比较低效，原因在于stl提供的算法都无法知晓具体容器的特点，它们只能根据容器提供的迭代接口来遍历容器，进行操作。所以更有效的方法就是调用这些容器自己提供的接口，例如std::set::find  
{% highlight cpp %}
1095   template<typename _Key, typename _Val, typename _KeyOfValue,
1096            typename _Compare, typename _Alloc>
1097     typename _Rb_tree<_Key,_Val,_KeyOfValue,_Compare,_Alloc>::iterator
1098     _Rb_tree<_Key,_Val,_KeyOfValue,_Compare,_Alloc>::find(const _Key& __k)
1099     {
1100       _Link_type __x = _M_begin(); // Current node.
1101       _Link_type __y = _M_end(); // Last node which is not less than __k.
1102
1103       while (__x != 0)
1104     if (!_M_impl._M_key_compare(_S_key(__x), __k))
1105       __y = __x, __x = _S_left(__x);
1106     else
1107       __x = _S_right(__x);
1108
1109       iterator __j = iterator(__y);
1110       return (__j == end()
1111       || _M_impl._M_key_compare(__k, _S_key(__j._M_node))) ? end() : __j;
1112     }
{% endhighlight %}
它基于内部的二叉树结构来快速的查找满足条件的元素。