---
id: 17
title: Event Handlers That Do Thread-Safe Invokes
date: 2008-07-11T06:21:00+00:00
guid: http://www.btburnett.com/?p=17
permalink: /2008/07/event-handlers-that-do-thread-safe-invokes.html
categories:
  - .NET Framework
---
I recently ran into a scenario where I wanted to write a class to do a low-level serial interface to a barcode scanner. When the barcode is scanned I wanted it to fire an event back to the Windows form. The complication was that the SerialPort class provided by .NET 2.0 fires the DataReceived event on an independent thread. In order to fire the event back to the form it needs to be synchronized to run on the thread of the form using the Invoke method.

While this can be done using manual delegates and other methods, they don't work very cleanly like an actual event handler does for simple development. The fix to this problem is to actually write a custom event handler. The custom event handler implements all of the back end handling that is normally hidden by VB. Here is the example code from the barcode scanner class:

```vb
Private _barcodeScannedList As List(Of EventHandler(Of BarcodeScannedEventArgs))
Public Custom Event BarcodeScanned As EventHandler(Of BarcodeScannedEventArgs)
    AddHandler(ByVal value As EventHandler(Of BarcodeScannedEventArgs))
        If _barcodeScannedList Is Nothing Then
            _barcodeScannedList = New List(Of EventHandler(Of BarcodeScannedEventArgs))
        End If
        _barcodeScannedList.Add(value)
    End AddHandler

    RemoveHandler(ByVal value As EventHandler(Of BarcodeScannedEventArgs))
        If _barcodeScannedList IsNot Nothing Then
            _barcodeScannedList.Remove(value)
        End If
    End RemoveHandler

    RaiseEvent(ByVal sender As Object, ByVal e As BarcodeScannedEventArgs)
        If Not _barcodeScannedList Is Nothing Then
            For Each handler As EventHandler(Of BarcodeScannedEventArgs) In _barcodeScannedList
                Dim safeInvoker As ISynchronizeInvoke = TryCast(handler.Target, ISynchronizeInvoke)
                If safeInvoker Is Nothing Then
                    handler.Invoke(sender, e)
                Else
                    safeInvoker.Invoke(handler, New Object() {sender, e})
                End If
            Next
        End If
    End RaiseEvent</div>
End Event
```

The _barcodeScannedList simply stores a list of all of the event handlers that have been registered with the class. AddHandler and RemoveHandler do the process of actually adding and removing those event handlers from the list. The real meat is in the RaiseEvent handler. Instead of simply iterating through the list and firing all of the event delegates directly like the standard handler does, this handler tests to see if the target class of the delegate implements the ISynchronizeInvoke interface. System.Windows.Forms.Control does implement this interface, meaning that all Forms implement it. If this interface is found, then the interface's Invoke method is used instead of the delegate's Invoke method. This will synchronize the call onto the delegate's target's thread.
