---
id: 24
title: Monitoring a WCF Service Host
date: 2008-10-13T08:22:00+00:00
guid: http://www.btburnett.com/?p=24
permalink: /2008/10/monitoring-a-wcf-service-host.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/10/monitoring-wcf-service-host.html
categories:
  - WCF
---
<span style="font-size:85%;">One problem I&#8217;ve been running into is my ServiceHosts failing out in my long running background services. I&#8217;ve written the following class to help monitor the service hosts and restart them if they fail. This is still experimental, and I&#8217;m not sure how well it&#8217;s going to work, so any feedback would be appreciated.</span>

The delegate is used to recreate the host whenever it needs to be created. Just pass in a delegate to a function that creates the ServiceHost and initializes all of its bindings, etc.<!-- code formatted by http://manoli.net/csharpformat/ -->

<pre class="csharpcode"><span class="kwrd">Public</span> <span class="kwrd">Delegate</span> <span class="kwrd">Function</span> ServiceHostCreateDelegate() <span class="kwrd">As</span> ServiceHost

<span class="kwrd">Public</span> <span class="kwrd">Class</span> ServiceHostMonitor
    <span class="kwrd">Implements</span> IDisposable

    <span class="kwrd">Private</span> _del <span class="kwrd">As</span> ServiceHostCreateDelegate
    <span class="kwrd">Private</span> <span class="kwrd">WithEvents</span> _host <span class="kwrd">As</span> ServiceHost
    <span class="kwrd">Public</span> <span class="kwrd">ReadOnly</span> <span class="kwrd">Property</span> Host() <span class="kwrd">As</span> ServiceHost
        <span class="kwrd">Get</span>
            <span class="kwrd">If</span> _host <span class="kwrd">Is</span> <span class="kwrd">Nothing</span> <span class="kwrd">Then</span>
                CreateHost()
            <span class="kwrd">End</span> <span class="kwrd">If</span>

            <span class="kwrd">Return</span> _host
        <span class="kwrd">End</span> <span class="kwrd">Get</span>
    <span class="kwrd">End</span> <span class="kwrd">Property</span>

    <span class="kwrd">Public</span> <span class="kwrd">ReadOnly</span> <span class="kwrd">Property</span> HostCreated() <span class="kwrd">As</span> <span class="kwrd">Boolean</span>
        <span class="kwrd">Get</span>
            <span class="kwrd">Return</span> _host IsNot <span class="kwrd">Nothing</span>
        <span class="kwrd">End</span> <span class="kwrd">Get</span>
    <span class="kwrd">End</span> <span class="kwrd">Property</span>

    <span class="kwrd">Private</span> _hostName <span class="kwrd">As</span> <span class="kwrd">String</span>
    <span class="kwrd">Public</span> <span class="kwrd">ReadOnly</span> <span class="kwrd">Property</span> HostName() <span class="kwrd">As</span> <span class="kwrd">String</span>
        <span class="kwrd">Get</span>
            <span class="kwrd">Return</span> _hostName
        <span class="kwrd">End</span> <span class="kwrd">Get</span>
    <span class="kwrd">End</span> <span class="kwrd">Property</span>

    <span class="kwrd">Public</span> <span class="kwrd">ReadOnly</span> <span class="kwrd">Property</span> State() <span class="kwrd">As</span> CommunicationState
        <span class="kwrd">Get</span>
            <span class="kwrd">Return</span> Host.State
        <span class="kwrd">End</span> <span class="kwrd">Get</span>
    <span class="kwrd">End</span> <span class="kwrd">Property</span>

    <span class="kwrd">Public</span> <span class="kwrd">Sub</span> <span class="kwrd">New</span>(<span class="kwrd">ByVal</span> createDelegate <span class="kwrd">As</span> ServiceHostCreateDelegate, <span class="kwrd">ByVal</span> hostName <span class="kwrd">As</span> <span class="kwrd">String</span>)
        <span class="kwrd">If</span> createDelegate <span class="kwrd">Is</span> <span class="kwrd">Nothing</span> <span class="kwrd">Then</span>
            <span class="kwrd">Throw</span> <span class="kwrd">New</span> ArgumentNullException(<span class="str">"createDelegate"</span>)
        <span class="kwrd">End</span> <span class="kwrd">If</span>
        <span class="kwrd">If</span> hostName <span class="kwrd">Is</span> <span class="kwrd">Nothing</span> <span class="kwrd">Then</span>
            <span class="kwrd">Throw</span> <span class="kwrd">New</span> ArgumentNullException(<span class="str">"hostName"</span>)
        <span class="kwrd">End</span> <span class="kwrd">If</span>

        _del = createDelegate
        _hostName = hostName
    <span class="kwrd">End</span> <span class="kwrd">Sub</span>

    <span class="kwrd">Protected</span> <span class="kwrd">Overrides</span> <span class="kwrd">Sub</span> Finalize()
        Dispose(<span class="kwrd">False</span>)
    <span class="kwrd">End</span> <span class="kwrd">Sub</span>

    <span class="kwrd">Protected</span> <span class="kwrd">Sub</span> CreateHost()
        <span class="kwrd">If</span> HostCreated <span class="kwrd">Then</span>
            Close()
        <span class="kwrd">End</span> <span class="kwrd">If</span>

        _host = _del.Invoke()
    <span class="kwrd">End</span> <span class="kwrd">Sub</span>

    <span class="kwrd">Public</span> <span class="kwrd">Sub</span> Open()
        <span class="kwrd">Try</span>
            CreateHost()
            Host.Open()

            Logger.LogMessage(<span class="str">"Opened Service Host "</span> & _hostName)
        <span class="kwrd">Catch</span> ex <span class="kwrd">As</span> Exception
            Logger.LogException(ex, <span class="str">"Error Opening Service Host "</span> & _hostName)
        <span class="kwrd">End</span> <span class="kwrd">Try</span>
    <span class="kwrd">End</span> <span class="kwrd">Sub</span>

    <span class="kwrd">Public</span> <span class="kwrd">Sub</span> Close()
        <span class="kwrd">Try</span>
            <span class="kwrd">If</span> HostCreated <span class="kwrd">Then</span>
                <span class="kwrd">If</span> State = CommunicationState.Opened <span class="kwrd">Then</span>
                    Host.Close()
                <span class="kwrd">End</span> <span class="kwrd">If</span>

                _host = <span class="kwrd">Nothing</span>
            <span class="kwrd">End</span> <span class="kwrd">If</span>
        <span class="kwrd">Catch</span> ex <span class="kwrd">As</span> Exception
            Logger.LogException(ex, <span class="str">"Error Closing Service Host "</span> & _hostName)
        <span class="kwrd">End</span> <span class="kwrd">Try</span>
    <span class="kwrd">End</span> <span class="kwrd">Sub</span>

    <span class="kwrd">Private</span> <span class="kwrd">Sub</span> _host_Faulted(<span class="kwrd">ByVal</span> sender <span class="kwrd">As</span> <span class="kwrd">Object</span>, <span class="kwrd">ByVal</span> e <span class="kwrd">As</span> System.EventArgs) <span class="kwrd">Handles</span> _host.Faulted
        Logger.LogError(<span class="str">"Service Host Faulted "</span> & _hostName)
        Open()
    <span class="kwrd">End</span> <span class="kwrd">Sub</span>

<span class="preproc">#Region</span> <span class="str">"IDisposable"</span>

    <span class="kwrd">Private</span> disposedValue <span class="kwrd">As</span> <span class="kwrd">Boolean</span> = <span class="kwrd">False</span>        <span class="rem">' To detect redundant calls</span>

    <span class="rem">' IDisposable</span>
    <span class="kwrd">Protected</span> <span class="kwrd">Overridable</span> <span class="kwrd">Sub</span> Dispose(<span class="kwrd">ByVal</span> disposing <span class="kwrd">As</span> <span class="kwrd">Boolean</span>)
        <span class="kwrd">If</span> <span class="kwrd">Not</span> <span class="kwrd">Me</span>.disposedValue <span class="kwrd">Then</span>
            <span class="kwrd">If</span> disposing <span class="kwrd">Then</span>
                Close()
            <span class="kwrd">End</span> <span class="kwrd">If</span>

            <span class="rem">' TODO: free shared unmanaged resources</span>
        <span class="kwrd">End</span> <span class="kwrd">If</span>
        <span class="kwrd">Me</span>.disposedValue = <span class="kwrd">True</span>
    <span class="kwrd">End</span> <span class="kwrd">Sub</span>

<span class="preproc">#Region</span> <span class="str">" IDisposable Support "</span>
    <span class="rem">' This code added by Visual Basic to correctly implement the disposable pattern.</span>
    <span class="kwrd">Public</span> <span class="kwrd">Sub</span> Dispose() <span class="kwrd">Implements</span> IDisposable.Dispose
        <span class="rem">' Do not change this code.  Put cleanup code in Dispose(ByVal disposing As Boolean) above.</span>
        Dispose(<span class="kwrd">True</span>)
        GC.SuppressFinalize(<span class="kwrd">Me</span>)
    <span class="kwrd">End</span> <span class="kwrd">Sub</span>
<span class="preproc">#End Region</span>

<span class="preproc">#End Region</span>

<span class="kwrd">End</span> Class</pre>

Please note that the Logger.LogError and Logger.LogMessage calls are for the custom error logging in my application. You should replace this with your own error logging as you see fit.