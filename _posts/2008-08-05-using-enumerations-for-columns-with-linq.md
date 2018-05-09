---
id: 18
title: Using Enumerations For Columns With LINQ
date: 2008-08-05T09:23:00+00:00
guid: http://www.btburnett.com/?p=18
permalink: /2008/08/using-enumerations-for-columns-with-linq.html
categories:
  - .NET Framework
---
One of the great features of LINQ that DataSets didn't support well is the ability to use enumerations for your columns instead of integers. It can make writing and working with your code a lot easier. This is accomplished by manually setting the <em>Type</em> of the column to the type of the enumeration in the visual designer.

There are two important notes to remember about doing this. First, it takes extra work if you want to use an enumeration that is in a different namespace from the LINQ classes. You must not only include the full namespace, but also the global:: prefix. So "global::EnumNamespace.EnumTypeName" refers to the enumeration *EnumTypeName* in the *EnumNamespace* namespace. Secondly, if you want to update the data structure from the database, you have to remember to manually set the *Type* back to the enum type from integer.

This system works great most of the time, including when performing LINQ queries in your code. However, the one place it doesn't seem to work well is in the *Where* clause of a *LinqDataSource*. If you try to select on the column values of an enumeration column here, you'll get various exceptions. If you try to refer to the enumeration constants, you get "No property or field "*x*" exists in type "*y*". I've tried various namespace permutations with no success. If you try to use integers directly instead of enumeration members, you get "Argument types do not match".

The only solution I have found to the problem is to typecast the column and compare to integers. For example, `Int32(ColumnName) == 1`. Personally, I don't like this solution much because it can cause huge problems if your constants ever change, but it works.

Also, if you are dealing with a *Flags* enumeration, it gets even worse. I haven't found any way to do bitwise operations in the Where clause, so you have to compare the column to all possible integers that contain the bits you are interested in.

If anybody knows of a better way to address this issue, please post a comment and let me know. Thanks.
