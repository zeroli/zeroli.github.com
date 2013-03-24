---
layout: post
title: "调用栈 (Call Stack)"
description: ""
category: C++
tags: [程序开发]
---
{% include JB/setup %}

调用栈的英文叫做call stack，从其英文书名来看，知道它本身就是一个栈，故而它满足栈的先入后出的特性。

wiki上有篇文章讲述[call stack](http://en.wikipedia.org/wiki/Call_stack)  
关于栈的溢出(stack overflow)，有下面的定义：  
> Since the call stack is organized as a stack, the caller pushes the return address onto the stack, and the called subroutine, when it finishes, pops the return address off the call stack and transfers control to that address. If a called subroutine calls on to yet another subroutine, it will push another return address onto the call stack, and so on, with the information stacking up and unstacking as the program dictates. If the pushing consumes all of the space allocated for the call stack, an error called a stack overflow occurs, generally causing the program to crash.  

关于call stack的作用，它也谈到了：  
> Storing the return address When a subroutine is called, the location (address) of the instruction at which it can later resume needs to be saved somewhere. Using a stack to save the return address has important advantages over alternatives. One is that each task has its own stack, and thus the subroutine can be reentrant, that is, can be active simultaneously for different tasks doing different things. Another benefit is that recursion is automatically supported. When a function calls itself recursively, a return address needs to be stored for each activation of the function so that it can later be used to return from the function activation. This capability is automatic with a stack.
  
存储指令的返回地址，是call stack最重要的作用，简单，而且会带来很多其它的好处，譬如每个线程维护自己的调用栈，还有递归调用。
> Local data storage A subroutine frequently needs memory space for storing the values of local variables, the variables that are known only within the active subroutine and do not retain values after it returns. It is often convenient to allocate space for this use by simply moving the top of the stack by enough to provide the space. This is very fast compared to heap allocation. Note that each separate activation of a subroutine gets its own separate space in the stack for locals.

局部数据存储在栈这样的存储空间中，比堆存储要快得多，原因在于，只需要简单的移动栈顶的位置就可以了，当需要分配栈空间时。
> Parameter passing Subroutines often require that values for parameters be supplied to them by the code which calls them, and it is not uncommon that space for these parameters may be laid out in the call stack. Generally if there are only a few small parameters, processor registers will be used to pass the values, but if there are more parameters than can be handled this way, memory space will be needed. The call stack works well as a place for these parameters, especially since each call to a subroutine, which will have differing values for parameters, will be given separate space on the call stack for those values.

调用栈还可以用来传递调用函数的参数，通常是在参数较多的时候。
> Pointer to current instance Some object-oriented languages (e.g., C++), store the this pointer along with function arguments in the call stack when invoking methods. The this pointer points to the object instance associated with the method to be invoked.

它还可以用来传递C++中的this指针。
> Enclosing subroutine context Some programming languages (e.g., Pascal and Ada) support nested subroutines, allowing an inner routine to access the context of its outer enclosing routine, i.e., the parameters and local variables within the scope of the outer routine. Such static nesting can repeat - a function declared within a function declared within a function... The implementation must provide a means by which a called function at any given static nesting level can reference the enclosing frame at each enclosing nesting level. Commonly this reference is implemented by a pointer to the encompassing frame, called a "downstack link" or "static link", to distinguish it from the "dynamic link" that refers to the immediate caller (which need not be the static parent function). For example, languages often allow inner routines to call themselves recursively, resulting in multiple call frames for the inner routine's invocations, all of whose static links point to the same outer routine context. Instead of a static link, the references to the enclosing static frames may be collected into an array of pointers known as adisplay which is indexed to locate a desired frame. The Burroughs B6500 had such a display in hardware that supported up to 32 levels of static nesting.  

用来解决一些语言支持的闭包特性。  
除了以上的作用外，文章还谈到了其它一些作用。  

关于call stack的具体结构布局，不同的平台系统有不同的实现：  
一个call stack通常由一个或多个栈帧(stack frames)组成，假如一个函数DrawLine正在被另外一个函数DrawSquare调用，这时候的call stack可以表示如下：  
这里栈顶向上（高地址）增长，不同的硬件平台有不同的实现。  
> The stack frame at the top of the stack is for the currently executing routine. The stack frame usually includes at least the following items (in push order):

> the arguments (parameter values) passed to the routine (if any);
the return address back to the routine's caller (e.g. in the DrawLine stack frame, an address into DrawSquare's code); and
space for the local variables of the routine (if any).
The stack and frame pointers
The data stored in the stack frame may sometimes be accessed directly via the stack pointer register (SP, which indicates the current top of the stack). However, as the stack pointer is variable during the activation of the routine, memory locations within the stack frame are more typically accessed via a separate register which makes relative addressing simpler and also enables dynamic allocation mechanisms (see below). This register is often termed the frame pointer or stack base pointer (BP) and is set up at procedure entry to point to a fixed location in the frame structure (such as the return address).

Stack frame sizes
> As different routines have different parameters and local data, stack frames have varying sizes. Although they may often be fixed across all activations of a particular routine, many modern languages also support dynamicallocations on the stack, which means that the local data area will vary from activation to activation with a size that may be unspecified when the program is compiled. In this case, access via a frame pointer, rather than via the stack pointer, is usually necessary since the offsets from the stack top to values such as the return address would not be known at compile time. If the subroutine does not use dynamic stack allocation and does not call any further subroutines, the frame pointer is not needed, and the register may be used for other purposes.

Storing the address to the caller's frame
> In most systems a stack frame has a field to contain the previous value of the frame pointer register, the value it had while the caller was executing. For example, the stack frame of DrawLine would have a memory location holding the frame pointer value that DrawSquare uses (not shown in the diagram above). The value is saved upon entry to the subroutine and restored upon return. Having such a field in a known location in the stack frame enables code to access each frame successively underneath the currently executing routine's frame, and also allows the routine to easily restore the frame pointer to the caller's frame, just before it returns.

Lexically nested routines
> Further information: Nested function and Non-local variable.  
> Programming languages that support nested subroutines also have a field in the call frame that points to the stack frame of the latest activation of the procedure that most closely encapsulates the callee, i.e. the immediatescope of the callee. This is called an access link or static link (as it keeps track of static nesting during dynamic and recursive calls) and provides the routine (as well as any other routines it may invoke) access to the local data of its encapsulating routines at every nesting level. Some architectures, compilers, or optimization cases store one link for each enclosing level (not just the immediately enclosing), so that deeply nested routines that access shallow data do not have to traverse several links; this strategy is often called a display.[1] Access link(s) can be optimized away in cases where an inner function does not access any (non constant) local data in the encapsulation—pure functions, i.e. routines communicating via argument(s) and return value(s) only would be an example of this. Some historical computers, such as the Burroughs large systems, had special "display registers" to support nested functions while compilers for most modern machines (such as the ubiquitous x86) simply reserve a few words on the stack for the pointers, as needed.

Overlap
> For some purposes, the stack frame of a subroutine and that of its caller can be considered to overlap, the overlap consisting of the area where the parameters are passed from the caller to the callee. In some environments, the caller pushes each argument onto the stack, thus extending its stack frame, then invokes the callee. In other environments, the caller has a preallocated area at the top of its stack frame to hold the arguments it supplies to other subroutines it calls. This area is sometimes termed the outgoing arguments area or callout area. Under this approach, the size of the area is calculated by the compiler to be the largest needed by any called subroutine.


