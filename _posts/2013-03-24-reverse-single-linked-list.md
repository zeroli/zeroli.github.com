---
layout: post
title: "链表的反转"
description: ""
category: "Data_Structure_and_Algorithm"
tags: [链表, glib代码阅读]
---
{% include JB/setup %}

这里的链表为单向链表和双向链表：  
##单向链表的反转
单向链表的反转，只需要维护1个节点指针，每次只能反转一个节点指向。  
GLIB的代码如下：（充分利用传入进来的list)
{% highlight cpp %}
GSList*
g_slist_reverse (GSList *list)
{
  GSList *prev = NULL;

  while (list)
    {
      GSList *next = list->next;
      list->next = prev; // 反转单向指针
      prev = list;
      list = next;
    }

  return prev;
}
{% endhighlight %}

##双向链表的反转
双向链表的反转，只需要交换每个节点的prev和next域，就可以。  
GLIB的代码如下：
{% highlight cpp %}
GList*
g_list_reverse (GList *list)
{
  GList *last;

  last = NULL;
  while (list)
  {
    last = list;
    list = last->next;
    last->next = last->prev;
    last->prev = list;
  }
  return last;
}
{% endhighlight %}
