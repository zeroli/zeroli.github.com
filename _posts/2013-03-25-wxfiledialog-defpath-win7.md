---
layout: post
title: "win7上的wxFileDialog的默认路径问题分析与解决"
description: ""
category: "wx"
tags: [程序开发, wxWidgets]
---
{% include JB/setup %}

##写在前面
wxFileDialog是[wxWidget](http://www.wxwidgets.org/)库的一个类, 这个类用来操作本地文件的打开与保存。在不同的平台上，wxFileDialog封装平台原生的API来实现，譬如win32平台，封装了GetOpenFileName和GetSaveFileName这2个API函数。我们只讨论win32平台。既然只是简单的封装这2个函数，那么wx在调用API之前必然要初始化一个结构体：OPENFILENAME。这个结构体有2个重要的变量：lpstrInitialDir和lpstrFile，前者是默认打开的目录路径，后者是默认打开的文件名。

##问题提出
假如我们要调用wxFileDialog来允许用户选择一个本地文件打开，通常会写下面的代码：
{% highlight cpp %}
wxFileDialog fileOpenDlg($parentWin, $title, $defaultDir, $defaultFile, $filterStr, wxOPEN);
fileOpenDlg.ShowModal();
{% endhighlight %}
如果我们希望打开时，定位到一个固定目录，那么我们会初始化$defaultDir。这一切都运行的不错，在winXP上。然而在win7上，运行结果却不是我们想要的（定位的目录，并不是我们期望的）。同样的代码，在不同的windows系统上运行的结果不同，主要是win7 OS对GetOpenFileName的实现算法进行了修改，相比于以前的系统。

##问题分析
前面提到win7上运行结果不同，是因为win7对API函数GetOpenFileName实现进行了更改，特别是结构体OPNEFILENAME里的成员变量lpstrInitialDir的处理。以下来自于MSDN的[说明](http://msdn.microsoft.com/en-us/library/windows/desktop/ms646839(v=vs.85).aspx)：
> 
> lpstrInitialDir   
> Type: LPCTSTR  
> The initial directory. The algorithm for selecting the initial directory varies on different platforms.  
> Windows 7:  
> If lpstrInitialDir has the same value as was passed the first time the application used an Open or Save As dialog box, the path > most recently selected by the user is used as the initial directory.  
> Otherwise, if lpstrFile contains a path, that path is the initial directory.  
> Otherwise, if lpstrInitialDir is not NULL, it specifies the initial directory.  
> If lpstrInitialDir is NULL and the current directory contains any files of the specified filter types, the initial directory is the current directory.  
> Otherwise, the initial directory is the personal files directory of the current user.  
> Otherwise, the initial directory is the Desktop folder.  
> Windows 2000/XP/Vista:  
> If lpstrFile contains a path, that path is the initial directory.   
> Otherwise, lpstrInitialDir specifies the initial directory.  
> Otherwise, if the application has used an Open or Save As dialog box in the past, the path most recently used is selected as the initial directory. However, if an application is not run for a long time, its saved selected path is discarded.  
> If lpstrInitialDir is NULL and the current directory contains any files of the specified filter types, the initial directory is the current directory.  
> Otherwise, the initial directory is the personal files directory of the current user.  
> Otherwise, the initial directory is the Desktop folder.  


其实在win7上，我们期望得到item3的结果，也就是我们设置了默认路径，就应该定位到那个路径，除非那个路径无效。然而第一项提到“如果这个默认路径与程序运行时__第一次__传给open/save as dialog的__有效的__的路径__相同__，那么win7 api就会采用__相同名字的程序__用户最近选择的路径（也就是上次成功打开/保存的那个路径）”。  
请看来自于微软一位Program Manager的解释：  
> Historically lpstrInitialDir would override "most recently used" folder. In general, this was not a good or expected user experience because the dialog did not remember the last place where the user saved, which is usually, the place where they want to save next time.  The new API (IFileDialog) allows applications to ask for a default folder (IFileDialog::SetDefaultFolder) which is prefferred for most cases, or explicitly set the folder (IFileDialog::SetFolder) which should be sparingly used in select cases.
 
> In Windows 7 we updated the legacy API beahvior (GetOpenFileName) to match the preferred behavior since “the file dialog forgot where I saved” was  was a significant source of negative feedback in Windows Vista.

> Let me know if you have any further questions.

> Thanks much,  
> David Washington  
> Program Manager   
> Microsoft


请注意：  
a) 第一次是指，不管该程序被运行多少次，只要第一次设置lpstrInitialDir为一个有效的路径，这个路径就会被记住，然后与后面传进来的lpstrInitialDir进行比较。哪怕程序实际上并没有打开那里面的任何文件，或者保存任何文件到那路径。  
b) 相同名字的程序，是指程序名字相同，不关心程序的路径。  

##问题解决
GetOpenFileName和GetSaveFileName这2个API函数被实现在comdlg32.dll中。关于它有一个专门的注册表项: `HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\ComDlg32\`
win7相比于winXP多了一个重要的folder key: FirstFolder。这个folder key就是用来存储前面提到的每个应用程序首次传给lpstrInitialDir的有效路径值。在这个folder key中有很多以阿拉伯数字命名的value，value对应的数值为二进制格式，而且是双字节存储。举个例子：
![](/assets/image/1345812150_5270.png)

数值17里面存储着某个应用程序全路径与它的第一个有效的lpstrInitialDir值的双字节ASCIC码。例如："E"对应45 00，":"对应3A 00。以低位字节打头。


一个案例：  
1. 初始化lpstrInitialDir到一个有效路径A，然后打开路径，cancel；  
2. 初始化lpstrInitialDir为空，然后保存一个文件到另一个目录B；  
3. go to 1。将会发现窗口定位到了目录B。  

前面提到，在win7上我们期待的是item3的效果，那么就需要越过item1，不让win7 API定位窗口默认目录到上次成功访问过的路径。我们发现在打开open/save之前，如果将这个程序对应的FirstFolder中对应的value删除，win7 api将会跳过item1，因为它没有发现第一次初始化值，也就不会用上次成功访问过的路径。  
所以方案就是  
a) 定位到folder key (HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\ComDlg32\FirstFolder)  
b) loop所有的values，获取它们的数值，将双字节的ASCII码转换成字符串  
c) 与当前程序路径进行比对，如果match，那就将这个value删除。  


但请注意，即使在打开文件打开对话框之前，将某个值删除了，打开之后，win7 api仍然会产生一个新的value。因为这时传给lpstrInitialDir是一个有效的路径，按照规则，该程序在FirstFolder里面没有了value，那么它就得再创建一个。但是没关系，下次再打开时，那个值仍然会被删除（有没点像打地鼠游戏？:)）。


一下是程序片段，如果发现传进来的lpstrInitialDir不空：
{% highlight cpp %}
void RemoveFirstFolderKeyForProgram()
{
    wxRegKey mainKey(wxRegKey::HKCU, "Software\\Microsoft\\Windows\\CurrentVersion\\Explorer\\ComDlg32\\FirstFolder");
    if (!mainKey.Exists()) return;
    if (!mainKey.Open()) return;


    wxString progPath = GetExecutablePath();
    long index = 0;
    wxString name;
    bool bCont = mainKey.GetFirstValue(name, index);
    while (bCont)
    {
        if (mainKey.GetValueType(name) == wxRegKey::Type_Binary)
        {
            wxMemoryBuffer valStr;
            if (mainKey.QueryValue(name, valStr))
            {	// the value data is double-bytes
             wxString val = ConvertBinDataToString((const char*)valStr.GetData(), valStr.GetDataLen(), 2);
             if (val.StartsWith(_T(progPath.c_str())))
             mainKey.DeleteValue(name);
            }
        }
        bCont = mainKey.GetNextValue(name, index);
    }
    mainKey.Close();
}

{% endhighlight %}