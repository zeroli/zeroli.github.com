---
layout: post
title: "malloc相关的函数浅析"
description: ""
category: "memory_management"
tags: [malloc]
---
{% include JB/setup %}

##malloc/free
malloc/free是C库函数，用来给用户态程序分配和释放内存，处理的内存地址是虚拟地址。能够保证虚拟地址是连续的，但不能保证物理地址是连续。glibc调用brk(sbrk)/mmap系统函数来实现内存的分配。


##kmalloc/kfree
kmalloc/kfree是linux内核定义的函数，只能被内核态程序调用，用来给内核程序分配和释放内存。__kmalloc返回的地址是虚拟地址__，而且虚拟地址和实际物理地址都是连续的。kmalloc返回的虚拟地址通常与物理地址相差PAGE_OFFSET偏移量。
它们的接口声明如下：
{% highlight cpp %}
void* kmalloc(size_t size, int flags);  
void kfree(const void* ptr);  
{% endhighlight %}
具有与malloc/free类似的接口，除了kmalloc的第二个参数flags。这个参数通常用来控制内存分配的行为。

kmalloc内部是调用Linux内核中的slab内存分配器，是寻找可用的内存。

关于malloc和kmalloc的区别：
> There are two major differences:   
> kmalloc returns physically contiguous memory, malloc does not guarantee anything about the physical memory mapping.   
> The other main difference is that kmalloc'ed memory is reserved and locked, it cannot be swapped. malloc does not actually allocate physical memory. Physical memory gets mapped later, during use.   
> Memory is subject to fragmentation. You may have many megabytes of free memory left, but a 8K kmalloc (on a 4k page machine) can still fail. Things like the buddy allocator help but can't always guarantee large unfragmented blocks.   
> If you don't need contiguous mapping in kernel space, you can use vmalloc to avoid the fragmentation problem, at the cost of some MMU overhead.   
> Userspace memory (via malloc) is virtual - immediately after the call, the same non-writable zero page is mapped into every 4K block of your memory region. Only when you write to a page, a physical memory page is mapped for you at that location.   
> If swap is enabled, that page may be 'borrowed' from some other process, after the contents are deposited in the swap file.   

##vmalloc/vfree
vmalloc/vfree同样是linux内核定义的函数，只能被内核态程序调用。工作方式类似于kmalloc，只不过前者分配的内存虚拟地址是连续的，而物理地址则无需连续。vmalloc只能确保也在页在物理地址空间内是连续的。它通过分配非连续的物理内存块，再“修正”页表，把内存映射到逻辑地址空间的连续区域中，就能做到这点。由于vmalloc需要把物理上不连续的页转化为虚拟地址空间上连续的页，必须专门建立页表项，而且必须以一个页一个页地进行映射（这会导致比直接内存映射大得多的TLB抖动）。所以即使是分配一页的内存空间，也没必要用vmalloc来申请。  
它们的接口声明如下：
{% highlight cpp %}
void* vmalloc(unsigned long size);  
void vfree(void* ptr);  
{% endhighlight %}
在vmalloc内部是通过调用`__get_free_pages`来迭代分配页内存的（故而物理内存地址不连续），同时用kmalloc(GFP_KERNEL)来为页表分配内存，故而该函数可能睡眠。vfree是用可能睡眠。


##slab cache
slab allocator为linux内核提供小结构对象的分配提供缓存机制。每个slab cache挂接3个链表: full slabs, partial slabs, 和empty slabs.每个链表都是循环双向链表，链表元素为一个slab。而每个slab管理一页或多页（linux中通常是一页)的内存空间，这一页内存空间中存储着多个相同大小的结构体对象。不同slab cache统一管理不同size的结构体对象。譬如task\_struct\_cachep管理task_struct结构体对象的分配和回收。

申请分配一个slab会调用`__get_free_page`一次性获取一页的内存空间，然后对一页建立一个slab管理中心。

##\_\_get\_free\_page类函数
`__get_free_page`类函数采用buddy allocator system来分配内存，分配的内存对齐到PAGE_SIZE。
`alloc_page`函数返回一个page结构体指针，page结构体是与物理页相关，而并非与虚拟页相关。但是它结构体中有一成员void* virtual，来用描述物理页对应的虚拟页的地址。当获取page之后，可用通过调用page_address(page)获取虚拟地址，相对应地，通过`__get_free_page`返回的虚拟地址，然后调用函数`virt_to_page(addr)`就可以得到page。
