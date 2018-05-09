---
id: 10
title: Odd WSHttpBinding Scenario
date: 2008-05-08T07:15:00+00:00
guid: http://www.btburnett.com/?p=10
permalink: /2008/05/odd-wshttpbinding-scenario.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/05/odd-wshttpbinding-scenario.html
categories:
  - WCF
---
<div xmlns="http://www.w3.org/1999/xhtml">
  <small>I was having lots of trouble recently with a WCF WSHttpBinding scenario on a server. It had been working, and the only thing I had changed recently was to the base address of some of the bindings. On top of that, everything was working in my test environment. When I did trace logging on the server, I wasn&#8217;t seeing any messages at all other than stating that listening had been started on the correct base addresses. On the client side, I was receiving the following message during the negotiation for the secure session (I&#8217;m using message encryption via certificates):<br /></small></p>

  <blockquote>
    <p>
      <small>An error occurred while receiving the HTTP response to http://xx.xx.xx.xx:yy/AdvWebClient/CapacityManager. This could be due to the service endpoint binding not using the HTTP protocol. This could also be due to an HTTP request context being aborted by the server (possibly due to the service shutting down). See server logs for more details.</small>
    </p>
  </blockquote>

  <p>
    <small><br />I replaced the ip and port numbers for security purposes.</p>

    <p>
      I worked on this for almost a day, restarting the windows service, changing configuration, staring at trace logs, etc. Finally, as a last ditch effort, I rebooted the server and everything started working. Apparently, there was some kind of hang up in the http.sys that was preventing it from processing the traffic. This is despite the listeners opening and closing (I watched using netstat) when I started and stopped the service.
    </p>

    <p>
      Hopefully this helps anyone else who encounters this problem. If anyone knows of a way to reset the http.sys system without rebooting the entire computer, please let me know. Thanks.
    </p>

    <p>
      </small></div>
