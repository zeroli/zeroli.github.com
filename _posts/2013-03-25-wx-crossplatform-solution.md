---
layout: post
title: "[wxWidget源码阅读] 理解wxWidget跨平台的实现方法"
description: ""
category: "wx"
tags: [程序开发, wxWidgets]
---
{% include JB/setup %}

wxWidget为了做到跨平台，对于那么与平台相关的类，比如wxWindow，大都会先定义一个平台无关的基类$(class)Base，通常定义在include/wx/xxx.h文件中，实现则放在了src/common/yyy.cpp文件。  
请注意头文件和实现文件的文件名通常是不同的，不过也有规律可循，譬如wxWindowBase类声明在window.h中，定义在src/common/wincmn.cpp中（以cmn为后缀命名），再比如wxTopLevelWindowBase声明在include/wx/toplevel.h中，定义在src/common/toplvcmn.cpp中。  
然后根据不同的平台编译选项，都会有一个与平台相关的类被定义，比如wxWindowMSW是为Windows平台定义的wxWindow类，通过宏定义  
`#define wxWindowMSW wxWindow`  
来实现，之后代码中就可以随意使用wxWindow了。GTK版本类似，实际的类叫做wxWindowGTK。通常平台相关的类声明在include/wx/$platform/xxx.h和src/$platform/xxx.cpp文件中，2个文件的文件名是相同的。  

平台无关的类的头文件中，在文件末尾，都会根据平台编译选项来include对应平台相关的类的头文件，比如：
{% highlight cpp %}
#if defined(__WXPALMOS__)
    #ifdef __WXUNIVERSAL__
        #define wxWindowNative wxWindowPalm
    #else // !wxUniv
        #define wxWindowPalm wxWindow
    #endif // wxUniv/!wxUniv
    #include "wx/palmos/window.h"
#elif defined(__WXMSW__)
    #ifdef __WXUNIVERSAL__
        #define wxWindowNative wxWindowMSW
    #else // !wxUniv
        #define wxWindowMSW wxWindow
    #endif // wxUniv/!wxUniv
    #include "wx/msw/window.h"
#elif defined(__WXMOTIF__)
    #include "wx/motif/window.h"
#elif defined(__WXGTK20__)
    #ifdef __WXUNIVERSAL__
        #define wxWindowNative wxWindowGTK
    #else // !wxUniv
        #define wxWindowGTK wxWindow
    #endif // wxUniv
    #include "wx/gtk/window.h"
...
#endif
{% endhighlight %}

![](/assets/image/1345812204_8623.png)
