---
id: 7
title: Linking Stylesheets From Content Pages
date: 2008-04-02T09:12:00+00:00
guid: http://www.btburnett.com/?p=7
permalink: /2008/04/linking-stylesheets-from-content-pages.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/04/linking-stylesheets-from-content-pages_02.html
categories:
  - ASP.NET
  - CSS
---
<div xmlns='http://www.w3.org/1999/xhtml'>
  <small>Personally, I find stylesheets to be the best way to apply formating to web pages. Skins certainly have their uses, but they tend to pad out the HTML with lots of redundant information that can be avoided by referencing a stylesheet, thus reducing download times. Stylesheets can be easily added to themes, but sometimes you don&#8217;t want to reference all of your stylesheets on every page, which is what happens if you put stylesheets into a theme.</p>

  <p>
    When using master and content pages, this can be a bit tricky. The common solution isn&#8217;t all that difficult, you just have to add a content section inside the header of your master page.<br /></small>
  </p>

  <blockquote>
    <p>
      <small><font face='Courier New'><% Master Language=&#8221;VB&#8221; AutoEventWireup=&#8221;false&#8221; CodeBehind=&#8221;SystemMaster.Master.vb&#8221; Inherits=&#8221;MyWebApp.SystemMaster&#8221; %></p>

      <p>
        <!DOCTYPE html PUBLIC &#8220;-//W3C//DTD XHTML 1.0 Strict//EN&#8221; &#8220;http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd&#8221;>
      </p>

      <p>
        <html xmlns=&#8221;http://www.w3.org/1999/xhtml><br /><head runat=&#8221;server&#8221;><br /> <title>Untitled Page</title><br /> <asp:ContentPlaceHolder ID=&#8221;HeadEntries&#8221; runat=&#8221;server&#8221;></asp:ContentPlaceHolder><br /></head><br /><body><br /> <asp:ContentPlaceHolder ID=&#8221;Body&#8221; runat=&#8221;server&#8221;></asp:ContentPlaceHolder><br /></body><br /></html></font><br /></small>
      </p></blockquote>

      <p>
        <small>Then on your content page:<br /></small>
      </p>

      <blockquote>
        <p>
          <small><font face='Courier New'><%@ Page Language=&#8221;vb&#8221; AutoEventWireup=&#8221;false&#8221; MasterPageFile=&#8221;~/SystemMaster.Master&#8221; CodeBehind=&#8221;Page.aspx.vb&#8221; Inherits=&#8221;MyWebApp.Page&#8221; </font><br /><font face='Courier New'> title=&#8221;Untitled Page&#8221; %></font></p>

          <p>
            <font face='Courier New'><asp:Content ID=&#8221;HeadEntries&#8221; ContentPlaceHolderID=&#8221;HeadEntries&#8221; runat=&#8221;server&#8221;></font><br /><font face='Courier New'> <link rel=&#8221;stylesheet&#8221; type=&#8221;text/css&#8221; href=&#8221;styles/styles.css&#8221; /></font><br /><font face='Courier New'></asp:Content></font>
          </p>

          <p>
            <font face='Courier New'><asp:Content ID=&#8221;Body&#8221; ContentPlaceHolderID=&#8221;Body&#8221; runat=&#8221;server&#8221;></font><br /><font face='Courier New'> Insert Body Here&#8230;</font><br /><font face='Courier New'></asp:Content></font><br /></small>
          </p></blockquote>

          <p>
            <small>Now this works great for most scenarios, but what if you&#8217;re using URL rewriting in order to dynamically generate your pages based on the URL that was requested? In that case, the relative path to the stylesheet won&#8217;t work. If you know that your web application will always be the root application on the server, you can use a path that starts with /, but that isn&#8217;t always the case. What you really need to do is use an application relative path, starting with a ~/.</p>

            <p>
              Normally you can change your <b>link</b> tag to include <b>runat=&#8221;server&#8221;</b> and it will process application relative paths in the <b>href</b> attribute. However, this doesn&#8217;t work on content pages for some reason. Therefore, I&#8217;ve created the class below to help.<br /></small>
            </p>

            <blockquote>
              <p>
                <small><font face='Courier New'>Imports System<br />Imports System.ComponentModel<br />Imports System.Text<br />Imports System.Web<br />Imports System.Web.UI<br />Imports System.Web.UI.WebControls</p>

                <p>
                  <DefaultProperty(&#8220;href&#8221;), ToolboxData(&#8220;<{0}:SmartLink runat=&#8221;&#8221;server&#8221;&#8221;></{0}:SmartLink>&#8221;)> _<br />Public Class SmartLink<br /> Inherits WebControl
                </p>

                <p>
                  <Bindable(True), Category(&#8220;Behavior&#8221;), DefaultValue(&#8220;&#8221;), Localizable(True)> _<br /> Property href() As String<br /> Get<br /> Dim s As String = CStr(ViewState(&#8220;href&#8221;))<br /> If s Is Nothing Then<br /> Return String.Empty<br /> Else<br /> Return s<br /> End If<br /> End Get
                </p>

                <p>
                  Set(ByVal Value As String)<br /> ViewState(&#8220;href&#8221;) = Value<br /> End Set<br /> End Property
                </p>

                <p>
                  <Bindable(True), Category(&#8220;Behavior&#8221;), DefaultValue(&#8220;&#8221;), Localizable(True)> _<br /> Property rel() As String<br /> Get<br /> Dim s As String = CStr(ViewState(&#8220;rel&#8221;))<br /> If s Is Nothing Then<br /> Return String.Empty<br /> Else<br /> Return s<br /> End If<br /> End Get
                </p>

                <p>
                  Set(ByVal Value As String)<br /> ViewState(&#8220;rel&#8221;) = Value<br /> End Set<br /> End Property
                </p>

                <p>
                  <Bindable(True), Category(&#8220;Behavior&#8221;), DefaultValue(&#8220;&#8221;), Localizable(True)> _<br /> Property type() As String<br /> Get<br /> Dim s As String = CStr(ViewState(&#8220;type&#8221;))<br /> If s Is Nothing Then<br /> Return String.Empty<br /> Else<br /> Return s<br /> End If<br /> End Get
                </p>

                <p>
                  Set(ByVal Value As String)<br /> ViewState(&#8220;type&#8221;) = Value<br /> End Set<br /> End Property
                </p>

                <p>
                  Public Overrides Sub RenderBeginTag(ByVal writer As System.Web.UI.HtmlTextWriter)<br /> If rel <> String.Empty Then<br /> writer.AddAttribute(HtmlTextWriterAttribute.Rel, rel)<br /> End If<br /> If type <> String.Empty Then<br /> writer.AddAttribute(HtmlTextWriterAttribute.Type, type)<br /> End If<br /> If href <> String.Empty Then<br /> writer.AddAttribute(HtmlTextWriterAttribute.Href, ResolveClientUrl(href))<br /> End If<br /> writer.RenderBeginTag(HtmlTextWriterTag.Link)<br /> End Sub
                </p>

                <p>
                  End Class</font><br /></small>
                </p></blockquote>

                <p>
                  <small><br />Now you replace the <b>link</b> tag on the content page with this:<br /></small>
                </p>

                <blockquote>
                  <p>
                    <small><font face='Courier New'><cc1:SmartLink runat=&#8221;server&#8221; rel=&#8221;stylesheet&#8221; type=&#8221;text/css&#8221; href=&#8221;~/styles/style.css&#8221; /></font><br /></small>
                  </p>
                </blockquote>

                <p>
                  <small>That&#8217;s it, now you have an application relative stylesheet link on your content page that works with URL rewriting.</small></div>
