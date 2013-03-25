---
layout: post
title: "wxEvtHandler类相关"
description: ""
category: "wx"
tags: [程序开发, wxWidgets]
---
{% include JB/setup %}

push一个evtHandler到一个窗口的evtHandler chain中

{% highlight cpp %}
void wxWindowBase::PushEventHandler(wxEvtHandler *handler)
{
    wxEvtHandler *handlerOld = GetEventHandler();


    handler->SetNextHandler(handlerOld);


    if ( handlerOld )
        GetEventHandler()->SetPreviousHandler(handler);


    SetEventHandler(handler);
}
{% endhighlight %}
从以上过程可以看到，wxEvtHandler是一个双向链表节点。每个wxWindow维护着这样一个wxEvtHandler双向链表，GetEvtHandler总是返回列表头。注意，wxWindow本身就是一个wxEvtHandler，初始化，链表里就是this指针，或者说是窗口自己。


去除链表头节点
{% highlight cpp %}
wxEvtHandler *wxWindowBase::PopEventHandler(bool deleteHandler)
{
    wxEvtHandler *handlerA = GetEventHandler();
    if ( handlerA )
    {
        wxEvtHandler *handlerB = handlerA->GetNextHandler();
        handlerA->SetNextHandler((wxEvtHandler *)NULL);


        if ( handlerB )
            handlerB->SetPreviousHandler((wxEvtHandler *)NULL);
        SetEventHandler(handlerB);


        if ( deleteHandler )
        {
            delete handlerA;
            handlerA = (wxEvtHandler *)NULL;
        }
    }


    return handlerA;
}
{% endhighlight %}
// 去除一个wxEvtHandler时，如果没找到，则在debug模式下，就会报错。
{% highlight cpp %}
bool wxWindowBase::RemoveEventHandler(wxEvtHandler *handler)
{
    wxCHECK_MSG( handler, false, _T("RemoveEventHandler(NULL) called") );


    wxEvtHandler *handlerPrev = NULL,
                 *handlerCur = GetEventHandler();
    while ( handlerCur )
    {
        wxEvtHandler *handlerNext = handlerCur->GetNextHandler();


        if ( handlerCur == handler )
        {
            if ( handlerPrev )
            {
                handlerPrev->SetNextHandler(handlerNext);
            }
            else
            {
                SetEventHandler(handlerNext);
            }


            if ( handlerNext )
            {
                handlerNext->SetPreviousHandler ( handlerPrev );
            }


            handler->SetNextHandler(NULL);
            handler->SetPreviousHandler(NULL);


            return true;
        }


        handlerPrev = handlerCur;
        handlerCur = handlerNext;
    }


    wxFAIL_MSG( _T("where has the event handler gone?") );


    return false;
}

{% endhighlight %}

