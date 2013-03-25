---
layout: post
title: "[wxWidget源码阅读] 理解wxWidget中event相关的宏"
description: ""
category: "wx"
tags: [程序开发, wxWidgets]
---
{% include JB/setup %}

##前言
wxWidget中的事件响应的方式，是基于事件表的方式，而不是采用虚函数的方式。

假如针对一个wxWindow类型的窗口，想要响应这个窗口的一些事件，通常我们会这样写：
MyWindow.h文件
{% highlight cpp %}
class MyWindow : public wxWindow
{
...
  DECLARE_EVENT_TABLE()
};
{% endhighlight %}
MyWindow.cpp文件：
{% highlight cpp %}
#include "MyWindow.h"
BEGIN_EVENT_TABLE( MyWindow, wxWindow )
   EVT_XXX_EVENT(wxMy_EVENT_ID, MyWindow::OnMyEventFunc)
END_EVENT_TABLE()
{% endhighlight %}
可以看到有很多宏替我们做了很多事情，但是到底做了什么事情呢？
##DECLARE_EVENT_TABLE
{% highlight cpp %}
#define DECLARE_EVENT_TABLE() \
    private: \
        static const wxEventTableEntry sm_eventTableEntries[]; \
    protected: \
        static const wxEventTable        sm_eventTable; \
        virtual const wxEventTable*      GetEventTable() const; \
        static wxEventHashTable          sm_eventHashTable; \
        virtual wxEventHashTable&        GetEventHashTable() const;
{% endhighlight %}
当我们在某个类中添加这个宏时，其实是对这个类添加了一个静态的事件响应表/数组（sm\_eventTableEntries），表中的每一项是事件类型与事件响应函数的组合。

##BEGIN\_EVENT\_TABLE
{% highlight cpp %}
#define BEGIN_EVENT_TABLE(theClass, baseClass) \
    const wxEventTable theClass::sm_eventTable = \
        { &baseClass::sm_eventTable, &theClass::sm_eventTableEntries[0] }; \
    const wxEventTable *theClass::GetEventTable() const \
        { return &theClass::sm_eventTable; } \
    wxEventHashTable theClass::sm_eventHashTable(theClass::sm_eventTable); \
    wxEventHashTable &theClass::GetEventHashTable() const \
        { return theClass::sm_eventHashTable; } \
    const wxEventTableEntry theClass::sm_eventTableEntries[] = { \
{% endhighlight %}
BEGIN\_EVENT\_TABLE其实是开始初始化这个静态的事件响应表，及其相关的静态函数。

##END_EVENT_TABLE
{% highlight cpp %}
#define END_EVENT_TABLE() DECLARE_EVENT_TABLE_ENTRY( wxEVT_NULL, 0, 0, 0, 0 ) };
{% endhighlight %}
可以看到END_EVENT_TABLE仅仅是插入一个空的EVENT ETNRY到表中，代表结束。

##DECLARE_EVENT_TABLE_ENTRY
事件响应表中的每一项其实都是结构体，包含着事件类型，来自于哪个窗口，对应的响应函数，以及其它参数。
{% highlight cpp %}
#define DECLARE_EVENT_TABLE_ENTRY(type, winid, idLast, fn, obj) \
    wxEventTableEntry(type, winid, idLast, fn, obj)
{% endhighlight %}

##如何添加/定义一个事件类型?
一个事件类型用ID来表示，INT类型。而且在这次事件链中必须是唯一的。声明一个事件类型借用宏DECLARE\_EXPORTED_EVENT\_TYPE，也仅仅是将全部事件整型值导出。通常声明在头文件中，定义在实现文件中。
譬如要定义一个事件类型wxEVT\_GRID\_CELL\_LEFT\_CLICK：
{% highlight cpp %}
#define BEGIN_DECLARE_EVENT_TYPES()
#define END_DECLARE_EVENT_TYPES()
#define DECLARE_EXPORTED_EVENT_TYPE(expdecl, name, value) \
    extern expdecl const wxEventType name;

BEGIN_DECLARE_EVENT_TYPES()
    DECLARE_EXPORTED_EVENT_TYPE(WXDLLIMPEXP_ADV, wxEVT_GRID_CELL_LEFT_CLICK, 1580)
    // more event type, add here.
END_DECLARE_EVENT_TYPES()
{% endhighlight %}
实现文件有如下定义：(请注意：wxEventType是int类型的typedef，也就是说，开始时所有的event type都是0)
{% highlight cpp %}
#define DEFINE_EVENT_TYPE(name) const wxEventType name = wxNewEventType();
DEFINE_EVENT_TYPE(wxEVT_GRID_CELL_LEFT_CLICK)
{% endhighlight %}
针对这个事件类型，定义它的事件响应函数，并将它们绑定起来形成event entry，插入到事件表中：
{% highlight cpp %}
#define wx__DECLARE_EVT2(evt, id1, id2, fn) \
    DECLARE_EVENT_TABLE_ENTRY(evt, id1, id2, fn, NULL),
#define wx__DECLARE_EVT1(evt, id, fn) \
    wx__DECLARE_EVT2(evt, id, wxID_ANY, fn)


typedef void (wxEvtHandler::*wxGridEventFunction)(wxGridEvent&);


#define wxGridEventHandler(func) \
    (wxObjectEventFunction)(wxEventFunction)wxStaticCastEvent(wxGridEventFunction, &func)


#define wx__DECLARE_GRIDEVT(evt, id, fn) \
    wx__DECLARE_EVT1(wxEVT_GRID_ ## evt, id, wxGridEventHandler(fn))


#define EVT_GRID_CMD_CELL_LEFT_CLICK(id, fn)     wx__DECLARE_GRIDEVT(CELL_LEFT_CLICK, id, fn)
{% endhighlight %}
定义一个定制的事件类型，编程思路差不多。

关于event entry结构体，有以下定义：
{% highlight cpp %}
// an entry from a static event table
struct WXDLLIMPEXP_BASE wxEventTableEntry : public wxEventTableEntryBase
{
    wxEventTableEntry(const int& evType, int winid, int idLast,
                      wxObjectEventFunction fn, wxObject *data)
        : wxEventTableEntryBase(winid, idLast, fn, data),
        m_eventType(evType)
    { }


    // the reference to event type: this allows us to not care about the
    // (undefined) order in which the event table entries and the event types
    // are initialized: initially the value of this reference might be
    // invalid, but by the time it is used for the first time, all global
    // objects will have been initialized (including the event type constants)
    // and so it will have the correct value when it is needed
    const int& m_eventType;


private:
    wxEventTableEntry& operator=(const wxEventTableEntry&);
};
{% endhighlight %}
m_eventType用了常量引用，一开始都是0，之后会是一个有效的值。（暂不清楚这个值是在哪里重新赋值的。）

