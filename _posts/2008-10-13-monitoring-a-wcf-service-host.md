---
id: 24
title: Monitoring a WCF Service Host
date: 2008-10-13T08:22:00+00:00
guid: http://www.btburnett.com/?p=24
permalink: /2008/10/monitoring-a-wcf-service-host.html
categories:
  - .NET Framework
---
One problem I've been running into is my ServiceHosts failing out in my long running background services. I've written the following class to help monitor the service hosts and restart them if they fail. This is still experimental, and I'm not sure how well it's going to work, so any feedback would be appreciated.

The delegate is used to recreate the host whenever it needs to be created. Just pass in a delegate to a function that creates the ServiceHost and initializes all of its bindings, etc.

```vb
Public Delegate Function ServiceHostCreateDelegate() As ServiceHost

Public Class ServiceHostMonitor
    Implements IDisposable

    Private _del As ServiceHostCreateDelegate
    Private WithEvents _host As ServiceHost
    Public ReadOnly Property Host() As ServiceHost
        Get
            If _host Is Nothing Then
                CreateHost()
            End If

            Return _host
        End Get
    End Property

    Public ReadOnly Property HostCreated() As Boolean
        Get
            Return _host IsNot Nothing
        End Get
    End Property

    Private _hostName As String
    Public ReadOnly Property HostName() As String
        Get
            Return _hostName
        End Get
    End Property

    Public ReadOnly Property State() As CommunicationState
        Get
            Return Host.State
        End Get
    End Property

    Public Sub New(ByVal createDelegate As ServiceHostCreateDelegate, ByVal hostName As String)
        If createDelegate Is Nothing Then
            Throw New ArgumentNullException("createDelegate")
        End If
        If hostName Is Nothing Then
            Throw New ArgumentNullException("hostName")
        End If

        _del = createDelegate
        _hostName = hostName
    End Sub

    Protected Overrides Sub Finalize()
        Dispose(False)
    End Sub

    Protected Sub CreateHost()
        If HostCreated Then
            Close()
        End If

        _host = _del.Invoke()
    End Sub

    Public Sub Open()
        Try
            CreateHost()
            Host.Open()

            Logger.LogMessage("Opened Service Host " & _hostName)
        Catch ex As Exception
            Logger.LogException(ex, "Error Opening Service Host " & _hostName)
        End Try
    End Sub

    Public Sub Close()
        Try
            If HostCreated Then
                If State = CommunicationState.Opened Then
                    Host.Close()
                End If

                _host = Nothing
            End If
        Catch ex As Exception
            Logger.LogException(ex, "Error Closing Service Host " & _hostName)
        End Try
    End Sub

    Private Sub _host_Faulted(ByVal sender As Object, ByVal e As System.EventArgs) Handles _host.Faulted
        Logger.LogError("Service Host Faulted " & _hostName)
        Open()
    End Sub

#Region "IDisposable"

    Private disposedValue As Boolean = False        ' To detect redundant calls

    ' IDisposable
    Protected Overridable Sub Dispose(ByVal disposing As Boolean)
        If Not Me.disposedValue Then
            If disposing Then
                Close()
            End If

            ' TODO: free shared unmanaged resources
        End If
        Me.disposedValue = True
    End Sub

#Region " IDisposable Support "
    ' This code added by Visual Basic to correctly implement the disposable pattern.
    Public Sub Dispose() Implements IDisposable.Dispose
        ' Do not change this code.  Put cleanup code in Dispose(ByVal disposing As Boolean) above.
        Dispose(True)
        GC.SuppressFinalize(Me)
    End Sub
#End Region

#End Region

End Class
```

Please note that the Logger.LogError and Logger.LogMessage calls are for the custom error logging in my application. You should replace this with your own error logging as you see fit.
