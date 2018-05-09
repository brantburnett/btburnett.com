---
id: 7
title: Linking Stylesheets From Content Pages
date: 2008-04-02T09:12:00+00:00
guid: http://www.btburnett.com/?p=7
permalink: /2008/04/linking-stylesheets-from-content-pages.html
categories:
  - ASP.NET
  - CSS
---
Personally, I find stylesheets to be the best way to apply formating to web pages. Skins certainly have their uses, but they tend to pad out the HTML with lots of redundant information that can be avoided by referencing a stylesheet, thus reducing download times. Stylesheets can be easily added to themes, but sometimes you don&#8217;t want to reference all of your stylesheets on every page, which is what happens if you put stylesheets into a theme.

When using master and content pages, this can be a bit tricky. The common solution isn&#8217;t all that difficult, you just have to add a content section inside the header of your master page.

```html
<% Master Language="VB" AutoEventWireup="false" CodeBehind="SystemMaster.Master.vb" Inherits="MyWebApp.SystemMaster" %>

<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">

<html xmlns="http://www.w3.org/1999/xhtml">
  <head runat="server">
    <title>Untitled Page</title>
    <asp:ContentPlaceHolder ID="HeadEntries" runat="server"></asp:ContentPlaceHolder>
  </head>
  <body>
    <asp:ContentPlaceHolder ID="Body" runat="server"></asp:ContentPlaceHolder>
  </body>
</html>
```

Then on your content page:

```html
<%@ Page Language="vb" AutoEventWireup="false" MasterPageFile="~/SystemMaster.Master" CodeBehind="Page.aspx.vb" Inherits="MyWebApp.Page" title="Untitled Page" %>

<asp:Content ID="HeadEntries" ContentPlaceHolderID="HeadEntries" runat="server">
  <link rel="stylesheet" type="text/css" href="styles/styles.css" />
</asp:Content>

<asp:Content ID="Body" ContentPlaceHolderID="Body" runat="server">
  Insert Body Here
</asp:Content>
```

Now this works great for most scenarios, but what if you&#8217;re using URL rewriting in order to dynamically generate your pages based on the URL that was requested? In that case, the relative path to the stylesheet won&#8217;t work. If you know that your web application will always be the root application on the server, you can use a path that starts with /, but that isn&#8217;t always the case. What you really need to do is use an application relative path, starting with a ~/.

Normally you can change your **link** tag to include **runat="server"** and it will process application relative paths in the **href** attribute. However, this doesn&#8217;t work on content pages for some reason. Therefore, I&#8217;ve created the class below to help.

```vb
Imports System
Imports System.ComponentModel
Imports System.Text
Imports System.Web
Imports System.Web.UI
Imports System.Web.UI.WebControls

<DefaultProperty("href"), ToolboxData("<{0}:SmartLink runat=""server""></{0}:SmartLink>")> _
Public Class SmartLink
    Inherits WebControl

    <Bindable(True), Category("Behavior"), DefaultValue(""), Localizable(True)> _
    Property href() As String
        Get
            Dim s As String = CStr(ViewState("href"))
            If s Is Nothing Then
                Return String.Empty
            Else
                Return s
            End If
        End Get

        Set(ByVal Value As String)
            ViewState("href") = Value
        End Set
    End Property

    <Bindable(True), Category("Behavior"), DefaultValue(""), Localizable(True)> _
    Property rel() As String
        Get
            Dim s As String = CStr(ViewState("rel"))
            If s Is Nothing Then
                Return String.Empty
            Else
                Return s
            End If
        End Get

        Set(ByVal Value As String)
            ViewState("rel") = Value
        End Set
    End Property

    <Bindable(True), Category("Behavior"), DefaultValue(""), Localizable(True)> _
    Property type() As String
        Get
            Dim s As String = CStr(ViewState("type"))
            If s Is Nothing Then
                Return String.Empty
            Else
                Return s
            End I
        End Get

        Set(ByVal Value As String)
            ViewState("type") = Value
        End Set
    End Property

    Public Overrides Sub RenderBeginTag(ByVal writer As System.Web.UI.HtmlTextWriter)
        If rel <> String.Empty Then
            writer.AddAttribute(HtmlTextWriterAttribute.Rel, rel)
        End If
        If type <> String.Empty Then
            writer.AddAttribute(HtmlTextWriterAttribute.Type, type)
        End If
        If href <> String.Empty Then
            writer.AddAttribute(HtmlTextWriterAttribute.Href, ResolveClientUrl(href))
        End If
        writer.RenderBeginTag(HtmlTextWriterTag.Link)
    End Sub

End Class
```

Now you replace the **link** tag on the content page with this:

```html
<cc1:SmartLink runat="server rel="stylesheet" type="text/css" href="~/styles/style.css" />
```

That&#8217;s it, now you have an application relative stylesheet link on your content page that works with URL rewriting.
