---
id: 21
title: Appending Text To The Titles Of All Web Pages
date: 2008-08-31T08:35:00+00:00
guid: http://www.btburnett.com/?p=21
permalink: /2008/08/appending-text-to-the-titles-of-all-web-pages.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/08/appending-text-to-titles-of-all-web.html
categories:
  - ASP.NET
---
<span style="font-size:85%;">Have you ever wanted to add a text string, like an application name, to the title of all your web pages. It&#8217;s easy to do using the control adapters available in ASP.NET 2.0 and later. First, create a new .browser file in the App_Browsers folder of your application, or add this entry to an existing file. Be sure to substitute MyNamespace for the correct namespace where the class is located.</p>

<pre class="csharpcode"><span class="kwrd">&lt;</span><span class="html">browsers</span><span class="kwrd">&gt;</span>

  <span class="kwrd">&lt;</span><span class="html">browser</span> <span class="attr">refID</span><span class="kwrd">="Default"</span><span class="kwrd">&gt;</span>
    <span class="kwrd">&lt;</span><span class="html">controlAdapters</span><span class="kwrd">&gt;</span>
      <span class="kwrd">&lt;</span><span class="html">adapter</span> <span class="attr">controlType</span><span class="kwrd">="System.Web.UI.HtmlControls.HtmlHead"</span>
               <span class="attr">adapterType</span><span class="kwrd">="MyNamespace.TitleAdapter"</span> <span class="kwrd">/&gt;</span>
    <span class="kwrd">&lt;/</span><span class="html">controlAdapters</span><span class="kwrd">&gt;</span>
  <span class="kwrd">&lt;/</span><span class="html">browser</span><span class="kwrd">&gt;</span>

<span class="kwrd">&lt;/</span><span class="html">browsers</span><span class="kwrd">&gt;</span></pre>

<p>
  Now create the TitleAdapter class that is referenced by the .browser file:
</p>

<pre class="csharpcode"><span class="kwrd">Imports</span> System.Web.UI.Adapters

<span class="kwrd">Public</span> <span class="kwrd">Class</span> TitleAdapter
    <span class="kwrd">Inherits</span> ControlAdapter

    <span class="kwrd">Private</span> <span class="kwrd">Shared</span> _titleAppendString <span class="kwrd">As</span> <span class="kwrd">String</span> = <span class="str">" - Your Application Name Here"</span>
    <span class="kwrd">Public</span> <span class="kwrd">Shared</span> <span class="kwrd">Property</span> TitleAppendString() <span class="kwrd">As</span> <span class="kwrd">String</span>
        <span class="kwrd">Get</span>
            <span class="kwrd">Return</span> _titleAppendString
        <span class="kwrd">End</span> <span class="kwrd">Get</span>
        <span class="kwrd">Set</span>(<span class="kwrd">ByVal</span> value <span class="kwrd">As</span> <span class="kwrd">String</span>)
            _titleAppendString = value
        <span class="kwrd">End</span> <span class="kwrd">Set</span>
    <span class="kwrd">End</span> <span class="kwrd">Property</span>

    <span class="kwrd">Protected</span> <span class="kwrd">Overrides</span> <span class="kwrd">Sub</span> Render(<span class="kwrd">ByVal</span> writer <span class="kwrd">As</span> System.Web.UI.HtmlTextWriter)
        <span class="kwrd">MyBase</span>.Render(<span class="kwrd">New</span> TitleAdapterHtmlTextWriter(writer, TitleAppendString))
    <span class="kwrd">End</span> <span class="kwrd">Sub</span>

<span class="kwrd">End</span> <span class="kwrd">Class</span></pre>

<p>
  And finally, create the HtmlTextWriter that is referenced by the control adapter:
</p>

<pre class="csharpcode"><span class="kwrd">Public</span> <span class="kwrd">Class</span> TitleAdapterHtmlTextWriter
    <span class="kwrd">Inherits</span> HtmlTextWriter

    <span class="kwrd">Public</span> <span class="kwrd">Sub</span> <span class="kwrd">New</span>(<span class="kwrd">ByVal</span> writer <span class="kwrd">As</span> HtmlTextWriter, <span class="kwrd">ByVal</span> appendString <span class="kwrd">As</span> <span class="kwrd">String</span>)
        <span class="kwrd">MyBase</span>.<span class="kwrd">New</span>(writer)
        InnerWriter = writer.InnerWriter

        _appendString = appendString
    <span class="kwrd">End</span> <span class="kwrd">Sub</span>

    <span class="kwrd">Public</span> <span class="kwrd">Sub</span> <span class="kwrd">New</span>(<span class="kwrd">ByVal</span> writer <span class="kwrd">As</span> System.IO.TextWriter, <span class="kwrd">ByVal</span> appendString <span class="kwrd">As</span> <span class="kwrd">String</span>)
        <span class="kwrd">MyBase</span>.<span class="kwrd">New</span>(writer)
        InnerWriter = writer

        _appendString = appendString
    <span class="kwrd">End</span> <span class="kwrd">Sub</span>

    <span class="kwrd">Private</span> _appendString <span class="kwrd">As</span> <span class="kwrd">String</span>

    <span class="kwrd">Private</span> _renderingTitle <span class="kwrd">As</span> <span class="kwrd">Boolean</span> = <span class="kwrd">False</span>
    <span class="kwrd">Private</span> _nestedCount <span class="kwrd">As</span> <span class="kwrd">Integer</span>

    <span class="kwrd">Public</span> <span class="kwrd">Overrides</span> <span class="kwrd">Sub</span> RenderBeginTag(<span class="kwrd">ByVal</span> tagName <span class="kwrd">As</span> <span class="kwrd">String</span>)
        <span class="kwrd">If</span> _renderingTitle <span class="kwrd">Then</span>
            _nestedCount += 1
        <span class="kwrd">ElseIf</span> <span class="kwrd">String</span>.Equals(tagName, <span class="str">"title"</span>, StringComparison.InvariantCultureIgnoreCase) <span class="kwrd">Then</span>
            _renderingTitle = <span class="kwrd">True</span>
            _nestedCount = 0
        <span class="kwrd">End</span> <span class="kwrd">If</span>

        <span class="kwrd">MyBase</span>.RenderBeginTag(tagName)
    <span class="kwrd">End</span> <span class="kwrd">Sub</span>

    <span class="kwrd">Public</span> <span class="kwrd">Overrides</span> <span class="kwrd">Sub</span> RenderBeginTag(<span class="kwrd">ByVal</span> tagKey <span class="kwrd">As</span> System.Web.UI.HtmlTextWriterTag)
        <span class="kwrd">If</span> _renderingTitle <span class="kwrd">Then</span>
            _nestedCount += 1
        <span class="kwrd">ElseIf</span> tagKey = HtmlTextWriterTag.Title <span class="kwrd">Then</span>
            _renderingTitle = <span class="kwrd">True</span>
            _nestedCount = 0
        <span class="kwrd">End</span> <span class="kwrd">If</span>

        <span class="kwrd">MyBase</span>.RenderBeginTag(tagKey)
    <span class="kwrd">End</span> <span class="kwrd">Sub</span>

    <span class="kwrd">Public</span> <span class="kwrd">Overrides</span> <span class="kwrd">Sub</span> RenderEndTag()
        <span class="kwrd">If</span> _renderingTitle <span class="kwrd">Then</span>
            <span class="kwrd">If</span> _nestedCount &gt; 0 <span class="kwrd">Then</span>
                _nestedCount -= 1
            <span class="kwrd">Else</span>
                _renderingTitle = <span class="kwrd">False</span>
                Write(_appendString)
            <span class="kwrd">End</span> <span class="kwrd">If</span>
        <span class="kwrd">End</span> <span class="kwrd">If</span>

        <span class="kwrd">MyBase</span>.RenderEndTag()
    <span class="kwrd">End</span> <span class="kwrd">Sub</span>

<span class="kwrd">End</span> <span class="kwrd">Class</span></pre>

<p>
  Note that you can edit the text string in the TitleAdapter class. Additionally, it is a static variable so you can alter it at startup from a database column in your Global.asax file.
</p>

<p>
  </span>
</p>
