---
id: 6
title: Using UpdatePanels Inside A Repeater On A Content Page
date: 2008-03-27T09:46:00+00:00
guid: http://www.btburnett.com/?p=6
permalink: /2008/03/using-updatepanels-inside-a-repeater-on-a-content-page.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/03/using-updatepanels-inside-repeater-on.html
categories:
  - AJAX
  - ASP.NET
---
<span style="font-size:85%;">In my previous post, I described how to use an <strong>UpdatePanel</strong> that is located inside a <strong>Repeater</strong> or other data-bound control while maintaining your databinding. In addition to the data-binding challenge, dynamically creating <strong>UpdatePanels</strong> your page can also pose other complications.</span>
<span style="font-size:85%;"></span>
<span style="font-size:85%;">Another issue that I&#8217;ve encountered is identifying controls that the <strong>AsyncPostBackTrigger</strong> on the <strong>UpdatePanel</strong> refer to if you are using master pages. So long as the control you are trying to trigger on is located in the same <strong>Content</strong> control of the content page, everything works just fine. However, if the <strong>UpdatePanel</strong> is located in one <strong>Content</strong> control while the control that triggers the asynchronous postback is in another <strong>UpdatePanel</strong>, you&#8217;ll get an exception when the page is loaded saying that it couldn&#8217;t find a control with the ID you specified.</span>
<span style="font-size:85%;"></span>
<span style="font-size:85%;">I&#8217;ve found various solutions for correcting this problem by dynamically adding the trigger in code, and using the control&#8217;s <strong>UniqueID</strong> to identify it. However, this becomes significantly more complicated when you are dealing with a dynamically added control. The best solution I&#8217;ve found so far is to use <strong>ScriptManager.RegisterAsyncPostBackControl</strong> to register the post back, and manually call <strong>Update</strong> on each <strong>UpdatePanel</strong> to perform the update. Below is the code that I&#8217;ve found fixes the problem.</span>

<span style=";font-family:courier new;font-size:85%;"  >Partial Public Class _Default</span>
<span style="margin-left: 2em;font-family:courier new;font-size:85%;"  >Inherits System.Web.UI.Page</span>
<span style=";font-family:courier new;font-size:85%;"  ></span>
<span style="margin-left: 2em;font-family:courier new;font-size:85%;"  >Dim DateUpdatePanels AS New List(Of UpdatePanel)</span>

<span style="margin-left: 2em;font-family:courier new;font-size:85%;"  >Protected Sub Page_Load(ByVal sender As Object, ByVal e As System.EventArgs) Handles Me.Load</span>
<span style="margin-left: 4em;font-family:courier new;font-size:85%;"  >ScriptManager.GetCurrent(Me).RegisterAsyncPostBackControl(edtDate)</span>
<span style="margin-left: 4em;font-family:courier new;font-size:85%;"  >DataBind()</span>
<span style="margin-left: 2em;font-family:courier new;font-size:85%;"  >End Sub</span>

<span style="margin-left: 2em;font-family:courier new;font-size:85%;"  >Private Sub Repeater_ItemDataBound(ByVal sender As Object, ByVal e As System.Web.UI.WebControls.RepeaterItemEventArgs) Handles Repeater.ItemDataBound</span>
<span style="margin-left: 4em;font-family:courier new;font-size:85%;"  >Dim pnl As UpdatePanel = e.Item.FindControl(&#8220;upnlShowTimes&#8221;)</span>
<span style="margin-left: 4em;font-family:courier new;font-size:85%;"  >If Not IsNothing(pnl) Then</span>
<span style="margin-left: 6em;font-family:courier new;font-size:85%;"  >DateUpdatePanels.Add(pnl)</span>
<span style="margin-left: 4em;font-family:courier new;font-size:85%;"  >End If</span>
<span style="margin-left: 2em;font-family:courier new;font-size:85%;"  >End Sub</span>

<span style="margin-left: 2em;font-family:courier new;font-size:85%;"  >Private Sub edtDate_TextChanged(ByVal sender As Object, ByVal e As System.EventArgs) Handles edtDate.TextChanged</span>
<span style="margin-left: 4em;font-family:courier new;font-size:85%;"  >For Each pnl As UpdatePanel In DateUpdatePanels</span>
<span style="margin-left: 6em;font-family:courier new;font-size:85%;"  >pnl.Update()</span>
<span style="margin-left: 4em;font-family:courier new;font-size:85%;"  >Next</span>
<span style="margin-left: 2em;font-family:courier new;font-size:85%;"  >End Sub</span>
<span style=";font-family:courier new;font-size:85%;"  ></span>
<span style=";font-family:courier new;font-size:85%;"  >End Class</span>
<span style="font-size:85%;"></span>
<span style="font-size:85%;">Note that if you define your <strong>ScriptManager</strong> on your content page instead of the master page, you can refer to it directly instead of using <strong>ScriptManager.GetCurrent(Me)</strong>. Also, you must be sure to set <strong>UpdateMode</strong> to <strong>Conditional</strong> on each <strong>UpdatePanel</strong>.</span>
