---
id: 5
title: Using UpdatePanels inside a Repeater
date: 2008-03-26T12:12:00+00:00
guid: http://www.btburnett.com/?p=5
permalink: /2008/03/using-updatepanels-inside-a-repeater.html
categories:
  - ASP.NET
---
Sometimes you might find it necessary to place an UpdatePanel control inside the ItemTemplate of a Repeater or other databound control. Normally you would probably just put the UpdatePanel outside of the repeater, but this can cause other problems. First of all, it makes the size of the update much larger. Secondly, I ran into a situation where I was using the [curvyCorners](http://www.curvycorners.net/) system for rounding the corners of DIVs that were inside my ItemTemplate.

First of all, it was problematic to run the javascript to round the corners again after the AJAX was done. You can do it using the ScriptManager.RegisterStartupScript method, so long as you place the recommended script code directly in the script block, and don't use the window.onload event. However, I found that it is VERY slow since it is running on several DIVs.

Therefore, I decided to use individual UpdatePanel controls inside each DIV that update just the data that needs updating. The problem I encountered is that I couldn't reference Container.DataItem for my data-binding inside the UpdatePanel like I could when it wasn't there.

The workaround I found was to use type-casting, and instead reference CType(Container, IDataItemContainer).DataItem. This fixed the problem without any side effects, at least so far. Each update panel is independently updated, and the rounded corners stay intact on my DIVs.

Note that I didn't have any problems using the Eval method, just DataBinder.GetPropertyValue and other DataBinder methods that require that you pass the DataItem as a parameter.
