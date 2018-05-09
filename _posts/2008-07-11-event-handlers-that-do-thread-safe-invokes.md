---
id: 17
title: Event Handlers That Do Thread-Safe Invokes
date: 2008-07-11T06:21:00+00:00
guid: http://www.btburnett.com/?p=17
permalink: /2008/07/event-handlers-that-do-thread-safe-invokes.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/07/event-handlers-that-do-thread-safe.html
categories:
  - .NET Framework
tags:
  - Threading
---
<div xmlns="http://www.w3.org/1999/xhtml">
  <small>I recently ran into a scenario where I wanted to write a class to do a low-level serial interface to a barcode scanner. When the barcode is scanned I wanted it to fire an event back to the Windows form. The complication was that the SerialPort class provided by .NET 2.0 fires the DataReceived event on an independent thread. In order to fire the event back to the form it needs to be synchronized to run on the thread of the form using the Invoke method.</p>

  <p>
    While this can be done using manual delegates and other methods, they don&#8217;t work very cleanly like an actual event handler does for simple development. The fix to this problem is to actually write a custom event handler. The custom event handler implements all of the back end handling that is normally hidden by VB. Here is the example code from the barcode scanner class:<br /></small>
  </p>

  <blockquote>
    <p>
      <small>Private _barcodeScannedList As List(Of EventHandler(Of BarcodeScannedEventArgs))<br />Public Custom Event BarcodeScanned As EventHandler(Of BarcodeScannedEventArgs)</p>

      <div style="margin-left: 2em">
        AddHandler(ByVal value As EventHandler(Of BarcodeScannedEventArgs))</p>

        <div style="margin-left: 2em">
          If _barcodeScannedList Is Nothing Then</p>

          <div style="margin-left: 2em">
            _barcodeScannedList = New List(Of EventHandler(Of BarcodeScannedEventArgs))
          </div>

          <p>
            End If<br />_barcodeScannedList.Add(value)</div>

            <p>
              End AddHandler
            </p>

            <p>
              RemoveHandler(ByVal value As EventHandler(Of BarcodeScannedEventArgs))
            </p>

            <div style="margin-left: 2em">
              If Not _barcodeScannedList Is Nothing Then</p>

              <div style="margin-left: 2em">
                _barcodeScannedList.Remove(value)
              </div>

              <p>
                End If</div>

                <p>
                  End RemoveHandler
                </p>

                <p>
                  RaiseEvent(ByVal sender As Object, ByVal e As BarcodeScannedEventArgs)
                </p>

                <div style="margin-left: 2em">
                  If Not _barcodeScannedList Is Nothing Then</p>

                  <div style="margin-left: 2em">
                    For Each handler As EventHandler(Of BarcodeScannedEventArgs) In _barcodeScannedList</p>

                    <div style="margin-left: 2em">
                      Dim safeInvoker As ISynchronizeInvoke = TryCast(handler.Target, ISynchronizeInvoke)<br />If safeInvoker Is Nothing Then</p>

                      <div style="margin-left: 2em">
                        handler.Invoke(sender, e)
                      </div>

                      <p>
                        Else
                      </p>

                      <div style="margin-left: 2em">
                        safeInvoker.Invoke(handler, New Object() {sender, e})
                      </div>

                      <p>
                        End If</div>

                        <p>
                          Next</div>

                          <p>
                            End If</div>

                            <p>
                              End RaiseEvent</div>

                              <p>
                                End Event</small>
                              </p></blockquote>

                              <p>
                                <small>The _barcodeScannedList simply stores a list of all of the event handlers that have been registered with the class. AddHandler and RemoveHandler do the process of actually adding and removing those event handlers from the list. The real meat is in the RaiseEvent handler. Instead of simply iterating through the list and firing all of the event delegates directly like the standard handler does, this handler tests to see if the target class of the delegate implements the ISynchronizeInvoke interface. System.Windows.Forms.Control does implement this interface, meaning that all Forms implement it. If this interface is found, then the interface&#8217;s Invoke method is used instead of the delegate&#8217;s Invoke method. This will synchronize the call onto the delegate&#8217;s target&#8217;s thread.<br /></small></div>
