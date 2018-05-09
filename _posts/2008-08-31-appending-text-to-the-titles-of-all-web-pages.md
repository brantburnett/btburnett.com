---
id: 21
title: Appending Text To The Titles Of All Web Pages
date: 2008-08-31T08:35:00+00:00
guid: http://www.btburnett.com/?p=21
permalink: /2008/08/appending-text-to-the-titles-of-all-web-pages.html
categories:
  - ASP.NET
---
Have you ever wanted to add a text string, like an application name, to the title of all your web pages. It's easy to do using the control adapters available in ASP.NET 2.0 and later. First, create a new .browser file in the App_Browsers folder of your application, or add this entry to an existing file. Be sure to substitute MyNamespace for the correct namespace where the class is located.

```xml
<browsers>

  <browser refID="Default">
    <controlAdapters>
      <adapter controlType="System.Web.UI.HtmlControls.HtmlHead"
               adapterType="MyNamespace.TitleAdapter" />
    </controlAdapters>
  </browser>

</browsers>
```

Now create the TitleAdapter class that is referenced by the .browser file:

```vb
Imports System.Web.UI.Adapters

Public Class TitleAdapter
    Inherits ControlAdapter

    Private Shared _titleAppendString As String = " - Your Application Name Here"
    Public Shared Property TitleAppendString() As String
        Get
            Return _titleAppendString
        End Get
        Set(ByVal value As String)
            _titleAppendString = value
        End Set
    End Property

    Protected Overrides Sub Render(ByVal writer As System.Web.UI.HtmlTextWriter)
        MyBase.Render(New TitleAdapterHtmlTextWriter(writer, TitleAppendString))
    End Sub

End Class
```

And finally, create the HtmlTextWriter that is referenced by the control adapter:

```vb
Public Class TitleAdapterHtmlTextWriter
    Inherits HtmlTextWriter

    Public Sub New(ByVal writer As HtmlTextWriter, ByVal appendString As String)
        MyBase.New(writer)
        InnerWriter = writer.InnerWriter

        _appendString = appendString
    End Sub

    Public Sub New(ByVal writer As System.IO.TextWriter, ByVal appendString As String)
        MyBase.New(writer)
        InnerWriter = writer

        _appendString = appendString
    End Sub

    Private _appendString As String

    Private _renderingTitle As Boolean = False
    Private _nestedCount As Integer

    Public Overrides Sub RenderBeginTag(ByVal tagName As String)
        If _renderingTitle Then
            _nestedCount += 1
        ElseIf String.Equals(tagName, "title", StringComparison.InvariantCultureIgnoreCase) Then
            _renderingTitle = True
            _nestedCount = 0
        End If

        MyBase.RenderBeginTag(tagName)
    End Sub

    Public Overrides Sub RenderBeginTag(ByVal tagKey As System.Web.UI.HtmlTextWriterTag)
        If _renderingTitle Then
            _nestedCount += 1
        ElseIf tagKey = HtmlTextWriterTag.Title Then
            _renderingTitle = True
            _nestedCount = 0
        End If

        MyBase.RenderBeginTag(tagKey)
    End Sub

    Public Overrides Sub RenderEndTag()
        If _renderingTitle Then
            If _nestedCount > 0 Then
                _nestedCount -= 1
            Else
                _renderingTitle = False
                Write(_appendString)
            End If
        End If

        MyBase.RenderEndTag()
    End Sub

End Class
```

Note that you can edit the text string in the TitleAdapter class. Additionally, it is a static variable so you can alter it at startup from a database column in your Global.asax file.
