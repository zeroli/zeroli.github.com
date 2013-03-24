---
layout: post
title: "glib中双向链表的排序"
description: ""
category: "Data Structure and Algorithm"
tags: [链表, glib代码阅读]
---
{% include JB/setup %}

在glib库中，双向链表的排序是采用合并排序的方法，但它并不是两两的合并，而是迭代的合并，如下图：


{% highlight cpp %}
GList *
g_list_sort (GList        *list,
	     GCompareFunc  compare_func)
{
  return g_list_sort_real (list, (GFunc) compare_func, NULL);
			    
}
static GList* 
g_list_sort_real (GList    *list,
		  GFunc     compare_func,
		  gpointer  user_data)
{
  GList *l1, *l2;
  
  if (!list) 
    return NULL;
  if (!list->next) 
    return list;
  
  l1 = list; 
  l2 = list->next;


  while ((l2 = l2->next) != NULL)
    {
      if ((l2 = l2->next) == NULL) 
	break;
      l1 = l1->next;
    }
  l2 = l1->next; 
  l1->next = NULL; 


  return g_list_sort_merge (g_list_sort_real (list, compare_func, user_data),
			    g_list_sort_real (l2, compare_func, user_data),
			    compare_func,
			    user_data);
}{% endhighlight %}
主要函数g_list_sort_real将传入的链表拆分成前n-1个节点和第n个节点，然后调用g_list_sort_merge去合并排序这两个链表。如果前n-1个节点都已经排好序了，那么就直接合并排序，否则，继续拆分前n-1个节点为n-2个节点和1个节点。

所以此过程类似于：第一个节点与第二个节点先排序，然后再和第三个节点排序，以此类推。  
最核心的合并排序过程代码如下：
{% highlight cpp %}
static GList *
g_list_sort_merge (GList     *l1, 
		   GList     *l2,
		   GFunc     compare_func,
		   gpointer  user_data)
{
  GList list, *l, *lprev;
  gint cmp;


  l = &list; 
  lprev = NULL;


  while (l1 && l2)
    {
      cmp = ((GCompareDataFunc) compare_func) (l1->data, l2->data, user_data);


      if (cmp <= 0)
        {
	  l->next = l1;
	  l1 = l1->next;
        } 
      else 
	{
	  l->next = l2;
	  l2 = l2->next;
        }
      l = l->next;
      l->prev = lprev; 
      lprev = l;
    }
  l->next = l1 ? l1 : l2;
  l->next->prev = l;


  return list.next;
}
{% endhighlight %}

此排序过程有点类似于STL中的set容器的set_*算法，所以说，这个过程有个前提就是2个链表l1和l2必须是已经排序的。结合以上的调用函数可以知道，这个需求可以保证，因为l1就是前n-1个节点链表总是排好序的，l2就是第n个节点。

细看整个排序过程的代码，其实此算法也可以用于单链表的排序过程（因为它并没有利用prev域），而且上面的合并排序过程的原型是一个插入排序，只是一些指针的调整，并没有进行数据节点的移动。

##更一步的分析
1. 假如有n个节点的链表，第一次拆分需要遍历n-1个节点，然后将前n-1个节点拆分，又需要遍历n-2个节点，以此类推，一直到只剩下一个节点时，拆分时才停止遍历。所以总的拆分过程需要(n-1)+(n-2)+(n-3)+...+1 = n x (n-1) / 2.  
2. 第1个节点与第2个节点合并排序，需要遍历1次，之后，再与节点3合并排序，这时需要考虑节点3与前面的2个节点之间的相互关系：
{% highlight cpp %}
else 
{
  l->next = l2;

  l2 = l2->next;
}
{% endhighlight %}
如果前面2个节点全部不大于第3个节点，那么遍历次数就是前面的节点数。因为我们知道l2链表中只会有一个节点。

3. 时间复杂度:  
这个算法最坏的情况在于要求排序的链表已经排序好了： O(n^2)，这个时间包括拆分链表所需要的时间。