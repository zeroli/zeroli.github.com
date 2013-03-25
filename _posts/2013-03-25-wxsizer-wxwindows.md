---
layout: post
title: "[wxWidget源码阅读] 理解wxWindow与wxSizer的关系"
description: ""
category: "wx"
tags: [程序开发, wxWidgets]
---
{% include JB/setup %}

##wxWindow与wxSizer

每个wxWindow都有一个m_contaningSizer（包含这个wxWindow窗口），还有一个m_windowSizer（这个窗口所包含的顶层wxSizer）。

设置包含这个window的sizer
{% highlight cpp %}
void wxWindowBase::SetContainingSizer(wxSizer* sizer)
{
// adding a window to a sizer twice is going to result in fatal and
// hard to debug problems later because when deleting the second
// associated wxSizerItem we're going to dereference a dangling
// pointer; so try to detect this as early as possible
wxASSERT_MSG( !sizer || m_containingSizer != sizer,
_T("Adding a window to the same sizer twice?") );


m_containingSizer = sizer;
}
{% endhighlight %}

设置这个窗口包含的顶层sizer
{% highlight cpp %}
void wxWindowBase::SetSizer(wxSizer *sizer, bool deleteOld)
{
if ( sizer == m_windowSizer)
return;

if ( m_windowSizer )
{
m_windowSizer->SetContainingWindow(NULL);

if ( deleteOld )
delete m_windowSizer;
}

m_windowSizer = sizer;
if ( m_windowSizer )
{
m_windowSizer->SetContainingWindow((wxWindow *)this);
}

SetAutoLayout(m_windowSizer != NULL);
}
{% endhighlight %}
窗口的Layout函数
layout这个窗口时，如果存在顶层sizer，就会调用这个sizer的Layout(in SetDimension函数中）。
{% highlight cpp %}
bool wxWindowBase::Layout()
{
// If there is a sizer, use it instead of the constraints
if ( GetSizer() )
{
int w = 0, h = 0;
GetVirtualSize(&w, &h);
GetSizer()->SetDimension( 0, 0, w, h );
}
#if wxUSE_CONSTRAINTS
else
{
SatisfyConstraints(); // Find the right constraints values
SetConstraintSizes(); // Recursively set the real window sizes
}
#endif


return true;
}
{% endhighlight %}
window的virtual size是client sizer和virtual Size变量的较大值。一般的window都是client size和virtual size变量一样大，除了scroll window类似的窗口。
{% highlight cpp %}
wxSize wxWindowBase::DoGetVirtualSize() const
{
// we should use the entire client area so if it is greater than our
// virtual size, expand it to fit (otherwise if the window is big enough we
// wouldn't be using parts of it)
wxSize size = GetClientSize();
if ( m_virtualSize.x > size.x )
size.x = m_virtualSize.x;


if ( m_virtualSize.y >= size.y )
size.y = m_virtualSize.y;


return size;
}
{% endhighlight %}
