---
layout: post
title: "glibc ptmalloc中的宏操作"
description: ""
category: "memory management"
tags: [程序开发, glib]
---
{% include JB/setup %}

glibc ptmalloc很有意思的宏操作：

##MORECORE
{% highlight cpp %}
/* Definition for getting more memory from the OS.  */
#define MORECORE         (*__morecore)
#define MORECORE_FAILURE 0
void * __default_morecore (ptrdiff_t);
void *(*__morecore)(ptrdiff_t) = __default_morecore;
{% endhighlight %}
__default_morecore是一个函数声明。morecore是一个函数类型定义。
MORECORE是一个对函数类型的宏定义。
例如MORECORE(1000);就等于调用函数__default_morecore(1000);
{% highlight cpp %}
#ifndef MORECORE
#define MORECORE sbrk
#endif
{% endhighlight %}
如果没有定义MORECORE，那么那就是sbrk函数的宏定义。

##brk & sbrk
{% highlight cpp %}
int
__brk (void *addr)
{
  void *__unbounded newbrk;


  INTERNAL_SYSCALL_DECL (err);
  newbrk = (void *__unbounded) INTERNAL_SYSCALL (brk, err, 1,
                              __ptrvalue (addr));


  __curbrk = newbrk;


  if (newbrk < addr)
    {
      __set_errno (ENOMEM);
      return -1;
    }


  return 0;
}


void *
__sbrk (intptr_t increment)
{
  void *oldbrk;


  /* If this is not part of the dynamic library or the library is used
     via dynamic loading in a statically linked program update
     __curbrk from the kernel's brk value.  That way two separate
     instances of __brk and __sbrk can share the heap, returning
     interleaved pieces of it.  */
  if (__curbrk == NULL || __libc_multiple_libcs)
    if (__brk (0) < 0)          /* Initialize the break.  */
      return (void *) -1;


  if (increment == 0)
    return __curbrk;


  oldbrk = __curbrk;
  if ((increment > 0
       ? ((uintptr_t) oldbrk + (uintptr_t) increment < (uintptr_t) oldbrk)
       : ((uintptr_t) oldbrk < (uintptr_t) -increment))
      || __brk (oldbrk + increment) < 0)
    return (void *) -1;


  return oldbrk;
}
{% endhighlight %}
##malloc_chunk结构体
{% highlight cpp %}
struct malloc_chunk {


  INTERNAL_SIZE_T      prev_size;  /* Size of previous chunk (if free).  */
  INTERNAL_SIZE_T      size;       /* Size in bytes, including overhead. */


  struct malloc_chunk* fd;         /* double links -- used only if free. */
  struct malloc_chunk* bk;


  /* Only used for large blocks: pointer to next larger size.  */
  struct malloc_chunk* fd_nextsize; /* double links -- used only if free. */
  struct malloc_chunk* bk_nextsize;
};
{% endhighlight %}

##chunk <=> mem的转换
{% highlight cpp %}
#define chunk2mem(p)   ((void*)((char*)(p) + 2*SIZE_SZ))
#define mem2chunk(mem) ((mchunkptr)((char*)(mem) - 2*SIZE_SZ))
{% endhighlight %}

##求结构体某成员的偏移量
{% highlight cpp %}
#define offsetof(Type, Member) ((size_t) &((Type *) NULL)->Member)
#define MIN_CHUNK_SIZE        (offsetof(struct malloc_chunk, fd_nextsize))
{% endhighlight %}
其实它求得的就是16，因为fd_nextsize变量的偏移量是16.

##将请求的内存大小转成chunk对其的内存量
{% highlight cpp %}
#define request2size(req)                                         \
  (((req) + SIZE_SZ + MALLOC_ALIGN_MASK < MINSIZE)  ?             \
   MINSIZE :                                                      \
   ((req) + SIZE_SZ + MALLOC_ALIGN_MASK) & ~MALLOC_ALIGN_MASK)
{% endhighlight %}
注意，这里只是多申请了一个size_sz，就是chunk size的4个字节。
MALLOC_ALIGN_MASK = 8 - 1 = 7，后面计算时将(req+size_sz) up align to 8的倍数。

##最低三位的设置与判断
4个字节的size域，共可以存储最大2^32的chunk size内存量，最低3位可以用来存储下面三个域，于是chunk size总是8的倍数
{% highlight cpp %}
/* size field is or'ed with PREV_INUSE when previous adjacent chunk in use */
#define PREV_INUSE 0x1

/* extract inuse bit of previous chunk */
#define prev_inuse(p)       ((p)->size & PREV_INUSE)

/* size field is or'ed with IS_MMAPPED if the chunk was obtained with mmap() */
#define IS_MMAPPED 0x2

/* check for mmap()'ed chunk */
#define chunk_is_mmapped(p) ((p)->size & IS_MMAPPED)

/* size field is or'ed with NON_MAIN_ARENA if the chunk was obtained
   from a non-main arena.  This is only set immediately before handing
   the chunk to the user, if necessary.  */
#define NON_MAIN_ARENA 0x4

/* check for chunk from non-main arena */
#define chunk_non_main_arena(p) ((p)->size & NON_MAIN_ARENA)
{% endhighlight %}
所以如果要获取chunk size，只需要将这个4字节的数作如下操作就可以了：
{% highlight cpp %}
#define SIZE_BITS (PREV_INUSE|IS_MMAPPED|NON_MAIN_ARENA)

/* Get size, ignoring use bits */
#define chunksize(p)         ((p)->size & ~(SIZE_BITS))
{% endhighlight %}
以及获取它前后2个chunk的头指针
{% highlight cpp %}
/* Ptr to next physical malloc_chunk. */
#define next_chunk(p) ((mchunkptr)( ((char*)(p)) + ((p)->size & ~SIZE_BITS) ))

/* Ptr to previous physical malloc_chunk */
#define prev_chunk(p) ((mchunkptr)( ((char*)(p)) - ((p)->prev_size) ))
{% endhighlight %}
下面的宏操作才是获取当前chunk的是否是free状态
{% highlight cpp %}
/* extract p's inuse bit */
#define inuse(p)\
((((mchunkptr)(((char*)(p))+((p)->size & ~SIZE_BITS)))->size) & PREV_INUSE)

/* set/clear chunk as being inuse without otherwise disturbing */
#define set_inuse(p)\
((mchunkptr)(((char*)(p)) + ((p)->size & ~SIZE_BITS)))->size |= PREV_INUSE

#define clear_inuse(p)\
((mchunkptr)(((char*)(p)) + ((p)->size & ~SIZE_BITS)))->size &= ~(PREV_INUSE)
{% endhighlight %}
由chunk头地址来得到这个chunk来自于哪个分配区：





