---
layout: post
title: "ptmalloc代码浅析3"
description: ""
category: "memory_management"
tags: [malloc, glibc代码阅读]
---
{% include JB/setup %}

malloc_consolidate函数是将fast bins中所有链表中相邻空闲chunk合并之后放入到unsorted chunk链表中。  
注意fd/bk是用来获取前后空闲chunk的指针域，而要获取前后相邻chunk，需要借助宏next、_chunk/prev、_chunk。
{% highlight cpp %}
/*
If max_fast is 0, we know that av hasn't
yet been initialized, in which case do so below
*/


if (get_max_fast () != 0) 
{
    clear_fastchunks(av);


    unsorted_bin = unsorted_chunks(av);


    /*
    Remove each chunk from fast bin and consolidate it, placing it
    then in unsorted bin. Among other reasons for doing this,
    placing in unsorted bin avoids needing to calculate actual bins
    until malloc is sure that chunks aren't immediately going to be
    reused anyway.
    */


    maxfb = &fastbin (av, NFASTBINS - 1);
    fb = &fastbin (av, 0);
    do {
        p = atomic_exchange_acq (fb, 0);
        if (p != 0) {
            do {
                check_inuse_chunk(av, p);
                nextp = p->fd;


                /* Slightly streamlined version of consolidation code in free() */
                size = p->size & ~(PREV_INUSE|NON_MAIN_ARENA);
                nextchunk = chunk_at_offset(p, size);
                nextsize = chunksize(nextchunk);
                // 前一个相邻chunk空闲，unlink，然后合并
                if (!prev_inuse(p)) {
                    prevsize = p->prev_size;
                    size += prevsize;
                    p = chunk_at_offset(p, -((long) prevsize));
                    unlink(p, bck, fwd);
                }


                if (nextchunk != av->top) {
                    nextinuse = inuse_bit_at_offset(nextchunk, nextsize);
                     
                    if (!nextinuse) {  // 后一个相邻chunk空闲，unlink，然后合并
                        size += nextsize;
                        unlink(nextchunk, bck, fwd);
                    } else
                    clear_inuse_bit_at_offset(nextchunk, 0);


                    first_unsorted = unsorted_bin->fd;
                    unsorted_bin->fd = p;
                    first_unsorted->bk = p;


                    if (!in_smallbin_range (size)) {
                        p->fd_nextsize = NULL;
                        p->bk_nextsize = NULL;
                    }
                    // 设置相关的头和尾标志位，然后link到unsorted bin
                    set_head(p, size | PREV_INUSE);
                    p->bk = unsorted_bin;
                    p->fd = first_unsorted;
                    set_foot(p, size);
                }


                else {
                    size += nextsize;
                    set_head(p, size | PREV_INUSE);
                    av->top = p;
                }


            } while ( (p = nextp) != 0);


        }
    } while (fb++ != maxfb);
}
else {
    malloc_init_state(av);
    check_malloc_state(av);
}
{% endhighlight %}
