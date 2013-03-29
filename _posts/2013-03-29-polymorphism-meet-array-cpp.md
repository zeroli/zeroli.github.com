---
layout: post
title: "C++中的多态遇上数组"
description: ""
category: cpp
tags: [C++]
---
{% include JB/setup %}

在C++中，当多态遇上数组会发生什么事情，比如下面的代码：  
{% highlight cpp %}
class A {
    public:
        A() {
                m_data = 10;
        }
    virtual void print() {

        printf("%d\n", m_data);
    }

    int m_data;

};

class B : public A {
    public:
        B() {
                m_data1 = 11;
    }
    virtual void print() {
        printf("%d\n", m_data1);
    }
    int m_data1;
};

int main()
{
   A* p = new B[2];
   p[1].print();
   delete [] p;
   return 0;

}
{% endhighlight %}

##问题:  
1. 用GCC来编译上面程序，能够通过吗？  
2. 如果编译通过了，程序能够正常运行吗？

##回答：  
1. GCC能够正确编译出可执行文件；  
2. 但是程序运行时会不会core dump，得看是32bit平台，还是64bit平台。32bit平台会crash，64bit不会。  

###简单分析：  
在32bit平台上，sizeof(A)=8, sizeof(B) = 12. p\[1\] 好比于p+1操作，故而p[1]将会(char\*)p + 8地址位置，而不是(char\*)p+12，第二个B所在的位置。而(char\*)p+8指向的位置是第一个B的数据内部，那里并没有vptr。  
在64bit平台上，sizeof(A)=16 (sizeof(vptr)=8, sizeof(int)=4, with 4 padding bytes), sizeof(B)=16. p\[1\]所指的位置正好是第二个B的头指针。因此能够正常操作那里的vptr。程序运行不会有问题。(但不建议这样的代码)  

###复杂分析：  
这个分析将从汇编代码的角度来阐述(平台：intel-i386)：  
直接贴出main函数的汇编代码，接下来将会分析各条指令的含义：  
{% highlight cpp %}
00000000 <main>:
   0:   55                      push   %ebp
   1:   89 e5                   mov    %esp,%ebp
   3:   83 ec 18                sub    $0x18,%esp
   6:   83 e4 f0                and    $0xfffffff0,%esp
   9:   b8 00 00 00 00          mov    $0x0,%eax
   e:   83 c0 0f                add    $0xf,%eax
  11:   83 c0 0f                add    $0xf,%eax
  14:   c1 e8 04                shr    $0x4,%eax
  17:   c1 e0 04                shl    $0x4,%eax
  1a:   29 c4                   sub    %eax,%esp
  1c:   83 ec 0c                sub    $0xc,%esp
  1f:   6a 18                   push   $0x18
  21:   e8 fc ff ff ff          call   22 <main+0x22>
                        22: R_386_PC32  _Znaj
  26:   83 c4 10                add    $0x10,%esp
  29:   89 45 f8                mov    %eax,0xfffffff8(%ebp)
  2c:   8b 45 f8                mov    0xfffffff8(%ebp),%eax
  2f:   89 45 f4                mov    %eax,0xfffffff4(%ebp)
  32:   c7 45 f0 01 00 00 00    movl   $0x1,0xfffffff0(%ebp)
  39:   83 7d f0 ff             cmpl   $0xffffffff,0xfffffff0(%ebp)
  3d:   75 02                   jne    41 <main+0x41>
  3f:   eb 17                   jmp    58 <main+0x58>
  41:   83 ec 0c                sub    $0xc,%esp
  44:   ff 75 f4                pushl  0xfffffff4(%ebp)
  47:   e8 fc ff ff ff          call   48 <main+0x48>
                        48: R_386_PC32  _ZN1BC1Ev
  4c:   83 c4 10                add    $0x10,%esp
  4f:   83 45 f4 0c             addl   $0xc,0xfffffff4(%ebp)
  53:   ff 4d f0                decl   0xfffffff0(%ebp)
  56:   eb e1                   jmp    39 <main+0x39>
  58:   8b 45 f8                mov    0xfffffff8(%ebp),%eax
  5b:   89 45 fc                mov    %eax,0xfffffffc(%ebp)
  5e:   83 ec 0c                sub    $0xc,%esp
  61:   8b 45 fc                mov    0xfffffffc(%ebp),%eax
  64:   83 c0 08                add    $0x8,%eax
  67:   8b 10                   mov    (%eax),%edx
  69:   8b 45 fc                mov    0xfffffffc(%ebp),%eax
  6c:   83 c0 08                add    $0x8,%eax
  6f:   50                      push   %eax
  70:   8b 02                   mov    (%edx),%eax
  72:   ff d0                   call   *%eax
  74:   83 c4 10                add    $0x10,%esp
  77:   83 7d fc 00             cmpl   $0x0,0xfffffffc(%ebp)
  7b:   74 0e                   je     8b <main+0x8b>
  7d:   83 ec 0c                sub    $0xc,%esp
  80:   ff 75 fc                pushl  0xfffffffc(%ebp)
  83:   e8 fc ff ff ff          call   84 <main+0x84>
                        84: R_386_PC32  _ZdaPv
  88:   83 c4 10                add    $0x10,%esp
  8b:   b8 00 00 00 00          mov    $0x0,%eax
  90:   c9                      leave
  91:   c3                      ret
{% endhighlight %}
上面代码中用到的symbol用c++filt解析之后如下：  
bash-3.00$ c++filt \_Znaj \_ZN1BC1Ev \_ZN1AC2Ev \_ZN1A5printEv \_ZN1B5printEv \_ZdaPv  
operator new\[\](unsigned int)  
B::B()  
A::A()  
A::print()  
B::print()  
operator delete\[\](void\*)  
汇编代码中相关常量的具体数值如下：  
0xfffffff8 = -0x18 = -8  
0xfffffff4 = -0x0c = -12  
0xfffffff0 = -0x10 = -16  
0xffffffff = -1  
0xfffffffc = -4  

分析如下：  
1) 程序首先会调用`operator new[](unsigned int) (_Znaj: line 22)`，在堆空间中构造2个B对象，在调用之前会调用`push   $0x18`指令来传入24 (2\*12)个字节的参数。这是函数operator new\[\]要求的。  
2) line29中%eax存的是源代码中p指针的值，代表2个B对象堆空间的首地址。它将会被保存到`0xfffffff8(%ebp) (=%ebp-8)`中，然后拷贝到`0xfffffff4(%ebp) (=%ebp-12)`的内存作为临时变量，调用`B::B() (_ZN1BC1Ev: line 48)`。因为在`call 48 <main+0x48>`之前总是调用`pushl  0xfffffff4(%ebp) (=%ebp-12)`。注意到line39的指令是一个比较指令，它会将-1 与 0xfffffff0(%ebp) (=%ebp-16)内存变量(=+1)做比较，从而循环2次来构造B对象，其间line4f的指令`addl   $0xc,0xfffffff4(%ebp)` 将那个内存地址往后推到下一个B对象地址头。
1和2，一起就完成了语句`A* p = new B[2];` 1就是函数`operator new [](unsigned int)`调用，2就是placement new语句(`A* p = new (%eax) A`).

3) 先看p\[1\]如何被计算出来:  
{% highlight cpp %}
  58:   8b 45 f8                mov    0xfffffff8(%ebp),%eax
  5b:   89 45 fc                mov    %eax,0xfffffffc(%ebp)
  5e:   83 ec 0c                sub    $0xc,%esp
  61:   8b 45 fc                mov    0xfffffffc(%ebp),%eax
  64:   83 c0 08                add    $0x8,%eax
  67:   8b 10                   mov    (%eax),%edx
{% endhighlight %}
我们知道之前有指令将%eax的值保存到了%ebp-8内存空间中，现在line58将它取出来放到eax中。接下来将这个值备份到%ebp-4内存空间中。注意到line64，常数8被增加到了eax中，这里的8就是sizeof(A)，而之前的%eax的值就是2个B堆空间的首地址，所以这条指令的意思就是将首地址增加8个字节，与我们之前分析的(char\*)p + 8刚好对应上，但是增加8个字节并不能定位到第二个B对象地址。**\[问题就是出现在这里\]**  
再来看看p\[1\].print是如何被调用的。  
{% highlight cpp %}
  69:   8b 45 fc                mov    0xfffffffc(%ebp),%eax
  6c:   83 c0 08                add    $0x8,%eax
  6f:   50                      push   %eax
  70:   8b 02                   mov    (%edx),%eax
  72:   ff d0                   call   *%eax
{% endhighlight %}
仍然是获取到堆空间的首地址，然后增加8个字节，存放到%eax中。由于print函数是虚函数，所以这个调用必然是在运行时决定的。line6f的push %eax指令是将偏移后的地址(期待是this指针)作为第一个参数压入堆栈。注意到C++对象的虚函数表指针被GCC放在对象内存空间的起始位置，而且B只有一个虚函数，故print函数的地址位于虚函数表的第一个，因而line70指令是将%edx(=%eax：B对象起始位置上的虚函数表指针)指向的虚函数表，也即print函数的地址存入%eax。line71指令中\*是代表运行时调用%eax指向的函数体，从而实现多态调用。由于前面增加8个字节来获取第一个B对象地址的操作不正确，故调用print函数会失败。

4) line74到line88是`delete [] p;`的汇编代码。0xfffffffc(%ebp) (%ebp-4) 存储着指针p。首先判断p是否等于0，然后将其作为参数push到栈上，接着调用`operator delete[](void*)`函数。

###进一步：  
Q: 如果class A定义了`virtual ~A();`虚析构函数，`delete [] p;`调用会crash程序吗？  
A: 在32bit平台上，会crash，原因类似。虚析构函数也是通过虚函数表指针来定位的。我们知道`delete [] p;`操作会先调用class A的虚析构函数2次，然后再调用`operator delete [](void*)`来回收堆空间。那么在用p+1来定位到第二个A\*指针时，那么并没有一个虚函数表指针，因此无法定位到虚析构函数。  
以下是包含virtual ~A();之后的main函数汇编代码片段：  
{% highlight cpp %}
Disassembly of section .text:

00000000 <main>:
   0:   55                      push   %ebp
   1:   89 e5                   mov    %esp,%ebp
   3:   53                      push   %ebx
   4:   83 ec 34                sub    $0x34,%esp
   7:   83 e4 f0                and    $0xfffffff0,%esp
   a:   b8 00 00 00 00          mov    $0x0,%eax
   f:   83 c0 0f                add    $0xf,%eax
  12:   83 c0 0f                add    $0xf,%eax
  15:   c1 e8 04                shr    $0x4,%eax
  18:   c1 e0 04                shl    $0x4,%eax
  1b:   29 c4                   sub    %eax,%esp
  1d:   83 ec 0c                sub    $0xc,%esp
  20:   6a 1c                   push   $0x1c
  22:   e8 fc ff ff ff          call   23 <main+0x23>
                        23: R_386_PC32  _Znaj
  27:   83 c4 10                add    $0x10,%esp
  2a:   89 45 f4                mov    %eax,0xfffffff4(%ebp)
  2d:   8b 45 f4                mov    0xfffffff4(%ebp),%eax
  30:   83 c0 04                add    $0x4,%eax
  33:   89 45 f0                mov    %eax,0xfffffff0(%ebp)
  36:   8b 55 f0                mov    0xfffffff0(%ebp),%edx
  39:   c7 42 fc 02 00 00 00    movl   $0x2,0xfffffffc(%edx)
  40:   8b 45 f0                mov    0xfffffff0(%ebp),%eax
  43:   89 45 ec                mov    %eax,0xffffffec(%ebp)
  46:   8b 55 ec                mov    0xffffffec(%ebp),%edx
  49:   89 55 e8                mov    %edx,0xffffffe8(%ebp)
  4c:   c7 45 e4 01 00 00 00    movl   $0x1,0xffffffe4(%ebp)
  53:   83 7d e4 ff             cmpl   $0xffffffff,0xffffffe4(%ebp)
  57:   75 05                   jne    5e <main+0x5e>
  59:   e9 89 00 00 00          jmp    e7 <main+0xe7>		; 循环2次之后，直接跳转到e7
  5e:   83 ec 0c                sub    $0xc,%esp
  61:   ff 75 e8                pushl  0xffffffe8(%ebp)
  64:   e8 fc ff ff ff          call   65 <main+0x65>
                        65: R_386_PC32  _ZN1BC1Ev
  69:   83 c4 10                add    $0x10,%esp
  6c:   83 45 e8 0c             addl   $0xc,0xffffffe8(%ebp)
  70:   ff 4d e4                decl   0xffffffe4(%ebp)
  73:   eb de                   jmp    53 <main+0x53>
  
  ; 从这里到e7前的代码都不会执行到
  
  75:   89 45 dc                mov    %eax,0xffffffdc(%ebp)
  78:   8b 45 dc                mov    0xffffffdc(%ebp),%eax
  7b:   89 45 e0                mov    %eax,0xffffffe0(%ebp)
  7e:   83 7d ec 00             cmpl   $0x0,0xffffffec(%ebp)
  82:   74 3e                   je     c2 <main+0xc2>
  84:   b8 01 00 00 00          mov    $0x1,%eax
  89:   2b 45 e4                sub    0xffffffe4(%ebp),%eax
  8c:   89 45 d8                mov    %eax,0xffffffd8(%ebp)
  8f:   8b 45 d8                mov    0xffffffd8(%ebp),%eax
  92:   d1 e0                   shl    %eax
  94:   03 45 d8                add    0xffffffd8(%ebp),%eax
  97:   c1 e0 02                shl    $0x2,%eax
  9a:   8b 55 ec                mov    0xffffffec(%ebp),%edx
  9d:   01 c2                   add    %eax,%edx
  9f:   89 55 d8                mov    %edx,0xffffffd8(%ebp)
  a2:   8b 45 d8                mov    0xffffffd8(%ebp),%eax
  a5:   39 45 ec                cmp    %eax,0xffffffec(%ebp)
  a8:   74 18                   je     c2 <main+0xc2>
  aa:   83 6d d8 0c             subl   $0xc,0xffffffd8(%ebp)
  ae:   83 ec 0c                sub    $0xc,%esp
  b1:   8b 55 d8                mov    0xffffffd8(%ebp),%edx
  b4:   8b 02                   mov    (%edx),%eax
  b6:   ff 75 d8                pushl  0xffffffd8(%ebp)
  b9:   8b 00                   mov    (%eax),%eax
  bb:   ff d0                   call   *%eax
  bd:   83 c4 10                add    $0x10,%esp
  c0:   eb e0                   jmp    a2 <main+0xa2>
  c2:   8b 45 e0                mov    0xffffffe0(%ebp),%eax
  c5:   89 45 dc                mov    %eax,0xffffffdc(%ebp)
  c8:   8b 5d dc                mov    0xffffffdc(%ebp),%ebx
  cb:   83 ec 0c                sub    $0xc,%esp
  ce:   ff 75 f4                pushl  0xfffffff4(%ebp)
  d1:   e8 fc ff ff ff          call   d2 <main+0xd2>
                        d2: R_386_PC32  _ZdaPv
  d6:   83 c4 10                add    $0x10,%esp
  d9:   89 5d dc                mov    %ebx,0xffffffdc(%ebp)
  dc:   83 ec 0c                sub    $0xc,%esp
  df:   ff 75 dc                pushl  0xffffffdc(%ebp)
  e2:   e8 fc ff ff ff          call   e3 <main+0xe3>
                        e3: R_386_PC32  _Unwind_Resume
						
; 开始执行delete [] p;
  e7:   8b 45 f0                mov    0xfffffff0(%ebp),%eax	; %ebp-16保存着p指针
  ea:   89 45 f8                mov    %eax,0xfffffff8(%ebp)
  ed:   83 7d f8 00             cmpl   $0x0,0xfffffff8(%ebp)	; 判断指针是否=0
  f1:   74 45                   je     138 <main+0x138>
  f3:   8b 45 f8                mov    0xfffffff8(%ebp),%eax
  f6:   83 e8 04                sub    $0x4,%eax        		; 将指针往前推4个字节(-4之后才是分配堆空间的真正起始地址)
  f9:   8b 00                   mov    (%eax),%eax				; 取出内容
  fb:   c1 e0 03                shl    $0x3,%eax				; * 8之后，应该是整个分配空间的大小
  fe:   8b 55 f8                mov    0xfffffff8(%ebp),%edx	
 101:   01 c2                   add    %eax,%edx				; 这样可以获取分配空间末尾的指针，放入%ebx
 103:   89 55 d4                mov    %edx,0xffffffd4(%ebp)
 106:   8b 45 d4                mov    0xffffffd4(%ebp),%eax
 109:   39 45 f8                cmp    %eax,0xfffffff8(%ebp)	; 判断首地址和末尾地址是否相等
 10c:   74 18                   je     126 <main+0x126>			; =?，跳转到126
 10e:   83 6d d4 08             subl   $0x8,0xffffffd4(%ebp)	; 否则，末尾地址减去8 (这个末尾地址-8，应该是从第二个对象开始析构)
 112:   83 ec 0c                sub    $0xc,%esp				
 115:   8b 55 d4                mov    0xffffffd4(%ebp),%edx	; 将减8之后的末尾地址存入%edx
 118:   8b 02                   mov    (%edx),%eax				; 取出那里的内容到%eax
 11a:   ff 75 d4                pushl  0xffffffd4(%ebp)			; 将减8之后的末尾地址，压入栈顶
 11d:   8b 00                   mov    (%eax),%eax				; %eax存着末尾地址-8指向的内容，应该是虚函数表指针。(%eax)是继续拿出第一个虚函数的地址
 11f:   ff d0                   call   *%eax					; 调用虚函数
 121:   83 c4 10                add    $0x10,%esp
 124:   eb e0                   jmp    106 <main+0x106>			; 跳转到106，继续执行第二个B的虚构函数
 126:   83 ec 0c                sub    $0xc,%esp
 129:   8b 45 f8                mov    0xfffffff8(%ebp),%eax	; 拿出p指针
 12c:   83 e8 04                sub    $0x4,%eax				; 往前4个字节，获取真正的分配空间首地址
 12f:   50                      push   %eax						; 压入栈顶，作为operator delete[](void*)的参数
 130:   e8 fc ff ff ff          call   131 <main+0x131>			; 
                        131: R_386_PC32 _ZdaPv		
 135:   83 c4 10                add    $0x10,%esp
 138:   b8 00 00 00 00          mov    $0x0,%eax
 13d:   8b 5d fc                mov    0xfffffffc(%ebp),%ebx
 140:   c9                      leave
 141:   c3                      ret
{% endhighlight %}
可以看到，`delete [] p;`的汇编语句是从第二个B对象开始析构：首先获取分配堆空间的末尾地址，然后减去sizeof(A)=8，对应以下语句：  
{% highlight cpp %}
 10e:   83 6d d4 08             subl   $0x8,0xffffffd4(%ebp)	; 否则，末尾地址减去8 (这个末尾地址-8，应该是从第二个对象开始析构)
{% endhighlight %}
 之后尝试着调用虚析构函数，失败了。  
从`delete []p;`的汇编代码，我们可以知道一点GLIBC的实现细节：A* p其实并不是分配堆空间的首地址，p-4(32bits平台)/p-8(64bits平台)才是首地址指针，\*(p-4)的值乘以8之后(fb: c1 e0 03 shl $0x3,%eax)，就可以得到p之后的堆空间字节大小。如下图所示：  

![](/assets/image/1357891587_5433.png)

从下面的代码可知， p-4也是传入`operator delete[]()`函数的参数。  
{% highlight cpp %}
129:   8b 45 f8                mov    0xfffffff8(%ebp),%eax	; 拿出p指针
12c:   83 e8 04                sub    $0x4,%eax				; 往前4个字节，获取真正的分配空间首地址
{% endhighlight %}