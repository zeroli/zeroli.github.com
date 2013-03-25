---
layout: post
title: "ptmalloc代码浅析1"
description: ""
category: "memory_management"
tags: [malloc, glibc代码阅读]
---
{% include JB/setup %}

ptmalloc实现分析：  
1. 在ptmalloc中，并没有定义malloc函数，而是定义了__libc_malloc(size_t nbytes);这样操作是为了不同平台。相应的free函数是__libc_free(void* mem);  
2. \_\_libc\_malloc函数  
{% highlight cpp %}
void*
__libc_malloc(size_t bytes)
{
  mstate ar_ptr;  // 这个就是分配区的结构体描述，每个线程都会有一个分配区(如果它要分配动态内存的话)
  void *victim;  // 这个就是将要返回给user的实际虚拟内存地址
  // FIXME：以下这段函数不是特别明白。
  __malloc_ptr_t (*hook) (size_t, const __malloc_ptr_t)
    = force_reg (__malloc_hook);
  if (__builtin_expect (hook != NULL, 0))
    return (*hook)(bytes, RETURN_ADDRESS (0));


  arena_lookup(ar_ptr);  // 首先查看当前线程的线程存储中是否有分配区的指针（通常是之前分配过内存时保存过）
 // 下面宏判断ar_ptr是否!=0，如果是，则lock这个分配区，否则，获取或创建一个非主分配区，去lock它。
  arena_lock(ar_ptr, bytes); 
  if(!ar_ptr)  // 如果系统无法获取非主分配区，则分配内存失败
    return 0;
  victim = _int_malloc(ar_ptr, bytes); // 这个函数是关键函数，用来从分配区中获取虚拟内存(通常是chunk)
  if(!victim) {
    /* Maybe the failure is due to running out of mmapped areas. */
    if(ar_ptr != &main_arena) {  // 如果分配失败，而且分配区不是主分配区，可能mmap region内存不够了
      (void)mutex_unlock(&ar_ptr->mutex); // unlock当前分配区的锁
      ar_ptr = &main_arena;  // 切换到主分配区去申请内存
      (void)mutex_lock(&ar_ptr->mutex);  // 获取主分配区的锁
      victim = _int_malloc(ar_ptr, bytes);  // 然后从主分配区分配内存
      (void)mutex_unlock(&ar_ptr->mutex);  // 分配内存内存(不管是否成功)，释放主分配区的锁
    } else {  // 第一次是从主分配区分配，但是失败了
      /* ... or sbrk() has failed and there is still a chance to mmap() */
      ar_ptr = arena_get2(ar_ptr->next ? ar_ptr : 0, bytes); // 尝试获取其它的非主分配区
      (void)mutex_unlock(&main_arena.mutex);  // 释放主分配区的锁
      if(ar_ptr) {  // 如果获取到了非主分配区
victim = _int_malloc(ar_ptr, bytes);  // 那么从这个非主分配区分配内存
(void)mutex_unlock(&ar_ptr->mutex);  // 然后释放这个分配区的锁
      }
    }
  } else  // 不管怎么样，我们都是要释放分配区的锁的。到这里，是第一次分配内存成功。
    (void)mutex_unlock(&ar_ptr->mutex);
// 简单的分配出来的内存：要么分配失败，要么是从mmap region分配出来的，
// 要么通过mem2chunk得到内存所属的chunk，进而得到chunk所被管辖的分配区
  assert(!victim || chunk_is_mmapped(mem2chunk(victim)) ||
ar_ptr == arena_for_chunk(mem2chunk(victim)));
  return victim;
}
{% endhighlight %}
arena_lock宏定义如下：
{% highlight cpp %}
# define arena_lock(ptr, size) do { \
  if(ptr) \
    (void)mutex_lock(&ptr->mutex); \
  else \
    ptr = arena_get2(ptr, (size)); \
} while(0)
{% endhighlight %}

{% highlight cpp %}
#define heap_for_ptr(ptr) \
 ((heap_info *)((unsigned long)(ptr) & ~(HEAP_MAX_SIZE-1)))
#define arena_for_chunk(ptr) \
 (chunk_non_main_arena(ptr) ? heap_for_ptr(ptr)->ar_ptr : &main_arena)
{% endhighlight %}

{% highlight cpp %}
// chunk的size域中倒数第2位(based on 0)，存有分配区信息，要么是主分配区，要么是非主分配区
#define chunk_non_main_arena(p) ((p)->size & NON_MAIN_ARENA)
{% endhighlight %}
