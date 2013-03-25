---
layout: post
title: "glib中GTrashStack小计"
description: ""
category: "Data_Structure_and_Algorithm"
tags: [glib代码阅读]
---
{% include JB/setup %}

GTrashStack中只有一个成员，是一个指针，也是一个指向stack前一片内存的指针。
{% highlight cpp %}
struct GTrashStack {
GTrashStack *next;  
{% endhighlight %}
在回收内存时，这样的结构不会带来额外的内存开销，因为可以将next指针记录在回收的内存中。
{% highlight cpp %}
G_INLINE_FUNC void
g_trash_stack_push (GTrashStack **stack_p,
gpointer data_p)
{
GTrashStack *data = (GTrashStack *) data_p; // 充分利用C语言的弱类型检查特性，直接获取转换化后的类型指针。


data->next = *stack_p;
*stack_p = data;
}
G_INLINE_FUNC gpointer
g_trash_stack_pop (GTrashStack **stack_p)
{
GTrashStack *data;


data = *stack_p;
if (data)
{
*stack_p = data->next;
/* NULLify private pointer here, most platforms store NULL as
* subsequent 0 bytes
*/
data->next = NULL; // 如果重用内存，可以将前4或8个字节清空。
}


return data;
} 
{% endhighlight %}
在SGI STL的memory pool也利用了类似的技术，那里是借用union一结构多用的特点。
