---
id: 163
title: Correcting MVC 3 EditorFor Template Field Names When Using Collections
date: 2011-03-25T09:39:35+00:00
guid: http://btburnett.com/?p=163
permalink: /2011/03/correcting-mvc-3-editorfor-template-field-names-when-using-collections.html
---
I recently ran into a problem with ASP.NET MVC 3 and editor templates when dealing with models that contain collections. If you handle the collection directly in the view, it works fine:

```vb
@For i as integer = 0 To Model.Locations.Count-1
    Dim index As Integer = i

    @<div>
        @Html.EditorFor(Function(model) model.Locations(i).Name)
        @Html.EditorFor(Function(model) model.Locations(i).Description)
    </div>
Next
```

In the above example, the fields will receive the names &#8220;Locations[0].Name&#8221; and &#8220;Locations[0].Description&#8221;, and so on for each field. This is correct, and will result in the model begin parsed correctly by the DefaultModelBinder if it&#8217;s a parameter on your POST action.

However, what if you want to make an editor template that always gets used for a specific collection type?  Let&#8217;s say, for example, that the collection in the model is of type LocationCollection, and you make an editor template named LocationCollection:

```vb
@ModelType LocationCollection
@For i as integer = 0 To Model.Count-1
    Dim index As Integer = i
    @<div>
        @Html.EditorFor(Function(model) model(i).Name)
        @Html.EditorFor(Function(model) model(i).Description)
    </div>
Next
```

Then you reference it in your view like this:

```vb
@Model.EditorFor(Function(model) model.Locations)
```

In this case, the fields will actually have the incorrect names on them, the will be named &#8220;Locations.[0].Name&#8221; and &#8220;Locations.[0].Description&#8221;.  Notice the extra period before the array index specifier.  With these field names, the model binder won&#8217;t recognize the fields when they come back in the post.

The solution to this problem is a little cumbersome, due to the way the field names are built.  First, the prefix is passed down to the editor template in ViewData.TemplateInfo.HtmlFieldPrefix.  However, this prefix is passed down WITHOUT the period.  The period is added as a delimiter by the ViewData.TemplateInfo.GetFullHtmlFieldName function, which is used by all of the editor helpers, like Html.TextBox.

This means that we actually can&#8217;t fix it in the editor template shown above.  Instead, we need TWO editor templates.  The first one for LocationCollection, and then a second one for Location:

```vb
@ModelType LocationCollection
@For i as integer = 0 To Model.Count-1
    Dim index As Integer = i
    @Html.EditorFor(Function(model) model(index))
Next
```

```vb
@ModelType Location
@Code
    ViewData.TemplateInfo.FixCollectionPrefix()
End Code
<div>
    @Html.EditorFor(Function(model) model(i).Name)
    @Html.EditorFor(Function(model) model(i).Description)
</div>
```

By breaking this up into two editor templates, we can now correct the prefix in the second template.  The first editor template receives an HtmlFieldPrefix of &#8220;Locations&#8221;, so we can&#8217;t do anything with that.  However, the Location editor template receives an HtmlFieldPrefix of &#8220;Locations.[0]&#8221;, so all we need to do is remove the extra period and the problem is solved.  As you can see in my example above, I&#8217;m calling a method on TemplateInfo, FixCollectionPrefix.  This is a simple extension method which corrects prefix, which you just need implement somewhere in an imported namespace:

```vb
<Extension()>
Public Sub FixCollectionPrefix(templateInfo As TemplateInfo)
Dim prefix As String = templateInfo.HtmlFieldPrefix
If Not String.IsNullOrEmpty(prefix) Then
templateInfo.HtmlFieldPrefix = Regex.Replace(prefix, ".(\[\d+\])$", "$1")
End If
End Sub
```

And that&#8217;s all there is to it. I certainly hope that the MVC team fixes this problem internally for their next release, but until then we&#8217;ll just have to keep working around it.