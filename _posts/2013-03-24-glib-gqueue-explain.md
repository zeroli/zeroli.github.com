---
layout: post
title: "glib中的GQueue结构体"
description: ""
category: C++
tags: [glib代码阅读]

---
{% include JB/setup %}

glib库的GQueue的实现，是依托在GList的基础上：  
![](/assets/image/1345811921_9487.png)

内部保存有哪个双向队列的头和尾，故可以提供一些双向队列的接口，例如push_head, push_tail, pop_head, pop_tail。

有一个印象深刻的函数实现：  
{% highlight cpp %}
GList *
g_queue_peek_nth_link (GQueue *queue,
		       guint   n)
{
  GList *link;
  gint i;
  
  g_return_val_if_fail (queue != NULL, NULL);

  if (n >= queue->length)
    return NULL;
  // 如果n大于length/2，则往后开始查找。这里充分利用了length这个域
  if (n > queue->length / 2)
    {
      n = queue->length - n - 1;

      link = queue->tail;
      for (i = 0; i < n; ++i)
	link = link->prev;
    }
  else
    {
      link = queue->head;
      for (i = 0; i < n; ++i)
	link = link->next;
    }


  return link;
}
{% endhighlight %}

另外，*_remove_*式的函数，如果提供的参数是GList，仅代表从链表中remove（去除）这个节点，如果提供gpointer，则代表remove（去除）数据所在的节点，而且要delete（删除）/free（释放）这个节点内存（当然GLIB中并没有真正回收内存，而是回收内存到内存池中）。*_delete_*式的函数，通常都要回收内存。
