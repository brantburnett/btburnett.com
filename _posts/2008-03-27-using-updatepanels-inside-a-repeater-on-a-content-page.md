---
id: 6
title: Using UpdatePanels Inside A Repeater On A Content Page
date: 2008-03-27T09:46:00+00:00
guid: http://www.btburnett.com/?p=6
permalink: /2008/03/using-updatepanels-inside-a-repeater-on-a-content-page.html
categories:
  - ASP.NET
---
In my previous post, I described how to use an **UpdatePanel** that is located inside a **Repeater** or other data-bound control while maintaining your databinding. In addition to the data-binding challenge, dynamically creating **UpdatePanels** your page can also pose other complications.

Another issue that I've encountered is identifying controls that the **AsyncPostBackTrigger** on the **UpdatePanel** refer to if you are using master pages. So long as the control you are trying to trigger on is located in the same **Content** control of the content page, everything works just fine. However, if the **UpdatePanel** is located in one **Content** control while the control that triggers the asynchronous postback is in another **UpdatePanel**, you'll get an exception when the page is loaded saying that it couldn't find a control with the ID you specified.

I've found various solutions for correcting this problem by dynamically adding the trigger in code, and using the control's **UniqueID** to identify it. However, this becomes significantly more complicated when you are dealing with a dynamically added control. The best solution I've found so far is to use **ScriptManager.RegisterAsyncPostBackControl** to register the post back, and manually call **Update** on each **UpdatePanel** to perform the update. Below is the code that I've found fixes the problem.

```vb
Partial Public Class _Default
    Inherits System.Web.UI.Page

    Dim DateUpdatePanels AS New List(Of UpdatePanel)

    Protected Sub Page_Load(ByVal sender As Object, ByVal e As System.EventArgs) Handles Me.Load
        ScriptManager.GetCurrent(Me).RegisterAsyncPostBackControl(edtDate)
        DataBind()
    End Sub

    Private Sub Repeater_ItemDataBound(ByVal sender As Object, ByVal e As System.Web.UI.WebControls.RepeaterItemEventArgs) Handles Repeater.ItemDataBound
        Dim pnl As UpdatePanel = e.Item.FindControl("upnlShowTimes")
        If Not IsNothing(pnl) Then
            DateUpdatePanels.Add(pnl)
        End If
    End Sub

    Private Sub edtDate_TextChanged(ByVal sender As Object, ByVal e As System.EventArgs) Handles edtDate.TextChanged
        For Each pnl As UpdatePanel In DateUpdatePanels
            pnl.Update()
        Next
    End Sub

End Class
```

Note that if you define your **ScriptManager** on your content page instead of the master page, you can refer to it directly instead of using **ScriptManager.GetCurrent(Me)**. Also, you must be sure to set **UpdateMode** to **Conditional** on each **UpdatePanel**.
