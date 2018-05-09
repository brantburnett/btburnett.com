---
id: 18
title: Using Enumerations For Columns With LINQ
date: 2008-08-05T09:23:00+00:00
guid: http://www.btburnett.com/?p=18
permalink: /2008/08/using-enumerations-for-columns-with-linq.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/08/using-enumerations-for-columns-with.html
categories:
  - .NET Framework
  - LINQ
---
<div xmlns='http://www.w3.org/1999/xhtml'>
  <small>One of the great features of LINQ that DataSets didn&#8217;t support well is the ability to use enumerations for your columns instead of integers. It can make writing and working with your code a lot easier. This is accomplished by manually setting the <em>Type</em> of the column to the type of the enumeration in the visual designer.</p>

  <p>
    There are two important notes to remember about doing this. First, it takes extra work if you want to use an enumeration that is in a different namespace from the LINQ classes. You must not only include the full namespace, but also the global:: prefix. So &#8220;global::EnumNamespace.EnumTypeName&#8221; refers to the enumeration <i>EnumTypeName</i> in the <i>EnumNamespace</i> namespace. Secondly, if you want to update the data structure from the database, you have to remember to manually set the <i>Type</i> back to the enum type from integer.
  </p>

  <p>
    This system works great most of the time, including when performing LINQ queries in your code. However, the one place it doesn&#8217;t seem to work well is in the <i>Where</i> clause of a <i>LinqDataSource</i>. If you try to select on the column values of an enumeration column here, you&#8217;ll get various exceptions. If you try to refer to the enumeration constants, you get &#8220;No property or field &#8216;<i>x</i>&#8216; exists in type &#8216;<i>y</i>&#8216;. I&#8217;ve tried various namespace permutations with no success. If you try to use integers directly instead of enumeration members, you get &#8220;Argument types do not match&#8221;.
  </p>

  <p>
    The only solution I have found to the problem is to typecast the column and compare to integers. For example, &#8220;Int32(ColumnName) == 1&#8221;. Personally, I don&#8217;t like this solution much because it can cause huge problems if your constants ever change, but it works.
  </p>

  <p>
    Also, if you are dealing with a <i>Flags</i> enumeration, it gets even worse. I haven&#8217;t found any way to do bitwise operations in the Where clause, so you have to compare the column to all possible integers that contain the bits you are interested in.
  </p>

  <p>
    If anybody knows of a better way to address this issue, please post a comment and let me know. Thanks.<br /></small></div>
